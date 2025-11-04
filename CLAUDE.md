# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**ESPHome-based pool solar heating controller for ESP32-DevKit** that integrates with Home Assistant. Uses a **hybrid architecture** where control logic runs on the ESP32 (due to unreliable WiFi) while Home Assistant provides the web interface for configuration and monitoring.

**Location:** Buenos Aires, Argentina
**HA Server:** Proxmox VM at `hassio@192.168.1.7`
**MQTT Broker:** `192.168.1.8`
**ESP32 Device Name:** ESP32-pileta
**GitHub Repository:** https://github.com/igarreta/pool_heat_esp32.git

## Architecture

### Hybrid Control Strategy

**Critical Design Decision:** Logic must run on ESP32 due to unstable WiFi connection.

- **ESP32 Role:** Autonomous heating control logic, sensor reading, relay control, skimmer automation
- **Home Assistant Role:** Configuration interface (input entities), status display, optional override controls
- **Communication:** ESP32 pulls configuration from HA input entities, reports status via MQTT + HA API

**Current Status:** Production-ready heating control with comprehensive safety system. Skimmer automation implemented and ready for deployment.

### Hardware Configuration

**Board:** ESP32-DevKit (Arduino framework)

**Temperature Sensors** (Dallas DS18B20 on GPIO25, One-Wire):
- `0xd81c77d446f18a28` - Pool water temperature (SAG)
- `0x65011447d4f2aa28` - Roof box temperature
- `0xe0011448ab01aa28` - Solar heater temperature (SCL)

**Light Sensor:** ADC on GPIO32 - Roof illumination

**Control Outputs:**
- **GPIO2:** Pool circulation pump relay (1-hour auto-shutoff)
- **GPIO16:** Pool heater relay (8-hour auto-shutoff)
- **Virtual Switch:** "Calefaccion Completa" - activates both pump + heater (8-hour auto-shutoff)

**Warning:** GPIO2 is a strapping pin - monitor boot behavior if issues occur.

### Home Assistant Integration Entities

**Input Controls (HA → ESP32):**
- `input_boolean.activar_calefaccion_pileta` (IAC) - Master enable/disable heating
- `input_boolean.activar_skimmer_pileta` (IAS) - Master enable/disable skimmer
- `input_number.pileta_temp_objetivo` (ITO) - Target pool temperature
- `input_number.pileta_temp_max_diff` (IMX) - Temperature delta to turn ON heating
- `input_number.pileta_temp_min_diff` (IMI) - Temperature delta to turn OFF heating

**Control Entity (ESP32 exposes to HA):**
- `switch.pileta_exp32_calefaccion_completa_esp32` (SWA) - Heating system on/off

**Sensor Entities (ESP32 → HA):**
- `sensor.esp32_pileta_temperatura_agua` (SAG) - Current water temperature
- `sensor.esp32_pileta_temperatura_calefactor` (SCL) - Heater temperature
- `sensor.esp32_pileta_horas_bomba_hoy` - Daily pump runtime (hours)
- `sensor.esp32_pileta_horas_bomba_ayer` - Previous day pump runtime (hours)

### Control Logic (to be implemented on ESP32)

**Turn ON heating when ALL conditions met:**
1. IAC is enabled
2. SAG < (ITO - 0.5°C) — Dead zone prevents rapid cycling
3. SCL > (SAG + IMX) — Heater is sufficiently warmer than water

**Turn OFF heating when ANY condition met:**
1. SAG ≥ ITO — Target temperature reached
2. SCL ≤ (SAG + IMI) — Heater insufficient temperature
3. IAC is disabled

**Parameter Validation:** IMX ≥ (IMI + 1°C) to prevent logic conflicts
- Automatically enforced on ESP32
- Invalid values are corrected with warning logs
- Works bidirectionally (IMX or IMI changes trigger validation)

**Safety Timers:**
- Pump auto-shutoff: 1 hour (only when NOT in heating mode)
- Heater auto-shutoff: 8 hours
- Combined mode auto-shutoff: 8 hours
- Heating mode flag (`bomba_modo_calefaccion`) prevents premature pump shutoff
- Independent watchdog timer provides backup 8-hour enforcement

**Daily Runtime Tracking:**
- Counter: `bomba_horas_hoy` tracks pump operation hours per day
- Updates every 60 seconds when pump is ON
- Resets automatically at midnight
- Exposed to HA as sensor: "Horas Bomba Hoy"
- Used for future skimmer automation logic

**Comprehensive Safety System (Phase 4A):**
- **Sensor range validation:** Water 0-50°C, heater 0-80°C
- **Sensor staleness detection:** 5-minute timeout triggers shutdown
- **Pump desync detection:** Verifies both pumps match expected state
- **Independent watchdog:** Backup 8-hour timer via millis() tracking
- **WiFi disconnect handling:** All parameters use restore_value for offline operation
- All safety violations force immediate heating shutdown with error logs

**ESP32 Disconnection Handling:**
- All critical parameters stored with `restore_value: yes`
- ESP32 continues autonomously with last-known-good values
- Parameters update automatically when WiFi/HA reconnects
- No action required - fully automatic

## Skimmer Automation (Phase 6 - COMPLETE)

**Purpose:** Ensure consistent daily filter runtime for pool water quality

**Current Situation:**
- Skimmer pump (`pileta_bomba_esp`) shared with heating system
- 1-hour auto-shutoff timer active when NOT in heating mode
- Daily runtime counter (`bomba_horas_hoy`) already tracking total pump hours
- Midnight reset via time synchronization already implemented

### Skimmer Requirements

**HA Input Entity (already created):**
- `input_boolean.activar_skimmer_pileta` - Master enable/disable for skimmer logic

**Schedule:**
- **7:00:** Run skimmer for 1 hour (skip if `bomba_horas_ayer` > 3.0 hours)
- **20:00:** Run skimmer for 1 hour (skip if `bomba_horas_hoy` > 2.5 hours)

**Fallback Mode (No Time Sync):**
- Run skimmer for 1 hour every 12 hours from boot
- Automatically disabled once HA time sync established
- Ensures basic water filtration even without HA connectivity

### Critical Constraints

**Heating Priority (Absolute):**
- Heating logic is **completely independent** and has **absolute priority**
- Skimmer NEVER starts if `bomba_modo_calefaccion == true`
- If heating starts while skimmer running, skimmer stops immediately (handled by existing 1-hour timer logic)
- Skimmer master switch OFF does NOT affect heating pump operation

**Safety:**
- Existing 1-hour auto-shutoff timer protects pump (lines 407-414)
- Skimmer runtime counts toward `bomba_horas_hoy` (already implemented)
- Heating 8-hour continuous limit remains unchanged

### Implementation Strategy

**New Globals Required:**
```yaml
- bomba_horas_ayer (float) - Previous day's runtime for 7:00 check
- skimmer_habilitado (bool) - Master switch state from HA
- time_synced (bool) - Track if HA time is valid
- skimmer_last_run (unsigned long) - Timestamp for fallback mode
```

**Logic Flow:**

1. **Boot:**
   - `time_synced = false`, `skimmer_last_run = 0`
   - Pull `skimmer_habilitado` from HA

2. **Fallback Mode (60s interval check):**
   - Active only when `time_synced == false`
   - If 12 hours elapsed AND `skimmer_habilitado` AND NOT `bomba_modo_calefaccion`:
     - Start pump for 1 hour
     - Update `skimmer_last_run = millis()`

3. **Time Sync Detection:**
   - Monitor `id(ha_time).now().is_valid()`
   - Once valid: `time_synced = true` (disables fallback mode)

4. **Scheduled Mode (requires time sync):**
   - **7:00 trigger:** Skip if `bomba_horas_ayer > 3.0` OR `bomba_modo_calefaccion` OR NOT `skimmer_habilitado`
   - **20:00 trigger:** Skip if `bomba_horas_hoy > 2.5` OR `bomba_modo_calefaccion` OR NOT `skimmer_habilitado`
   - Start pump for 1 hour (existing timer handles shutoff)

5. **Midnight Reset (enhanced):**
   - Store: `bomba_horas_ayer = bomba_horas_hoy`
   - Reset: `bomba_horas_hoy = 0.0`

### Expected Behavior

**Normal Operation (time synced):**
- 7:00: Pump runs if yesterday's total < 3 hours and heating not active
- 20:00: Pump runs if today's total < 2.5 hours and heating not active
- Heating can interrupt skimmer at any time
- Skimmer defers to heating automatically

**Fallback Operation (no time sync):**
- Pump runs 1 hour every 12 hours from boot
- Stops immediately if heating starts
- Transitions to scheduled mode once time syncs

**Master Switch OFF:**
- Skimmer scheduling completely disabled
- Does NOT force pump off (respects heating mode)

**Complexity:** ~70 additional lines of code, no interference with existing heating logic

## Critical Event Notifications (Phase 7 - Next Implementation)

**Purpose:** Alert user of critical system failures that require attention

**Notification Strategy:**
- **Method:** ESP32 → Home Assistant → Pushover (via `notify.pushover` service)
- **Why HA instead of direct API:**
  - Works over local network (WiFi issues don't block notifications)
  - Simpler ESP32 code (no HTTPS/token management)
  - Flexible (easy to change notification service later)
  - Already connected via HA API

**Critical Events to Notify:**

1. **Sensor Range Violations** (esp32-pileta.yaml:268-284)
   - Water temp < 0°C or > 50°C
   - Heater temp < 0°C or > 80°C
   - Message: "⚠️ Pool: [Sensor] fuera de rango (XX°C)"
   - Priority: 0 (normal)

2. **Sensor Staleness** (esp32-pileta.yaml:286-304)
   - No sensor update for >5 minutes
   - Message: "⚠️ Pool: Sensor [name] sin actualizar (>5 min)"
   - Priority: 0

3. **Pump Desync Detection** (esp32-pileta.yaml:306-314)
   - Pumps don't match expected heating state
   - Message: "⚠️ Pool: Desincronización de bombas detectada"
   - Priority: 0

4. **Runtime Watchdog Triggered** (esp32-pileta.yaml:316-325)
   - 8-hour continuous runtime limit reached
   - Message: "⚠️ Pool: Límite de 8 horas alcanzado"
   - Priority: 0

5. **Parameter Validation Auto-Correction** (esp32-pileta.yaml:146-177)
   - IMX < IMI + 1°C automatically corrected
   - Message: "⚠️ Pool: Parámetros inválidos auto-corregidos (IMX ajustado)"
   - Priority: 0

**Notification Throttling:**
- Do NOT repeat same notification within the same day
- Use global flags to track sent notifications
- Reset flags at midnight

**Events NOT Notified:**
- ❌ WiFi disconnection (notifications wouldn't work anyway)
- ❌ Normal operation events (too frequent)
- ❌ Sensor readings (continuous data)
- ❌ Control logic evaluations (every 30s)
- ❌ Heating/skimmer cycles (optional for future)

**Manual Test Function:**
- **Test Button:** `button.esp32_pileta_test_pushover_notification`
- Exposed to Home Assistant UI
- Sends test message: "Pool System Test - This is a test notification from ESP32-pileta"
- Use to verify notification path after deployment or configuration changes

**Implementation Details:**
```yaml
# Add to esp32-pileta.yaml
button:
  - platform: template
    name: "Test Pushover Notification"
    on_press:
      - homeassistant.service:
          service: notify.pushover
          data:
            title: "Pool System Test"
            message: "Test notification from ESP32-pileta"
            data:
              priority: 0

# Add to each critical error:
- homeassistant.service:
    service: notify.pushover
    data:
      title: "Pool System Alert"
      message: !lambda |-
        return "Sensor agua fuera de rango: " + String(temp_agua) + "°C";
      data:
        priority: 0
```

**New Globals Required:**
- `notif_sensor_range_sent` (bool) - Track if sensor range notification sent today
- `notif_sensor_stale_sent` (bool) - Track if staleness notification sent today
- `notif_pump_desync_sent` (bool) - Track if desync notification sent today
- `notif_watchdog_sent` (bool) - Track if watchdog notification sent today
- `notif_param_validation_sent` (bool) - Track if validation notification sent today
- All reset to false at midnight

**Prerequisites:**
- Home Assistant must have Pushover integration configured
- Service name: `notify.pushover` must be functional
- Test with button before adding to critical errors

## Essential Commands

### Git Workflow

**IMPORTANT:** Commit and push changes regularly to maintain project history.

**Check status:**
```bash
git status
```

**Stage and commit changes:**
```bash
git add .
git commit -m "Descriptive commit message"
```

**Push to GitHub:**
```bash
git push origin main
```

**Complete workflow after making changes:**
```bash
git add .
git commit -m "Update ESP32 control logic" && git push origin main
```

### ESPHome Dashboard Workflow (Recommended)

**Copy configuration to Home Assistant:**
```powershell
# From Windows PowerShell
scp esp32-pileta.yaml hassio@192.168.1.7:/config/esphome/
```

**Alternative:** Copy YAML content via clipboard and paste into ESPHome Dashboard.

**Configure secrets:**
- WiFi credentials are managed directly in ESPHome Dashboard UI
- Edit device configuration in the dashboard
- Secrets are stored securely by Home Assistant

**Compile and flash:**
1. Open Home Assistant web interface
2. Navigate to **Settings → Add-ons → ESPHome → Open Web UI**
3. Click on the device card
4. Click **Install** → Choose method:
   - **Wirelessly** (if device already has ESPHome)
   - **Plug into this computer** (USB connection)
   - **Manual download** (download .bin file)
5. Monitor logs after installation

### Validation (if ESPHome CLI installed locally)

```bash
esphome config esp32-pileta.yaml
```

### Home Assistant Debugging

**SSH connection:**
```bash
ssh -o "MACs=hmac-sha2-256-etm@openssh.com" hassio@192.168.1.7
```

**Check add-on resource usage:**
```bash
ha addons stats 5c53de3b_esphome  # ESPHome add-on
ha addons stats a0d7b954_vscode   # VS Code Server
```

**Check HA core status:**
```bash
ha core stats
ha core logs | tail -50
```

**System health check:**
```bash
uptime && vmstat 1 3
```

See `WARP_HA_DEBUG.md` for comprehensive HA debugging procedures.

## Development Workflow

### Current State
- Main ESP32 configuration: `esp32-pileta.yaml`
- Secrets managed in ESPHome Dashboard
- Heating control logic: ✅ COMPLETE (Phases 1-4A)
- Daily runtime tracking: ✅ COMPLETE
- Safety system: ✅ COMPLETE (Phase 4A)
- Skimmer automation: ✅ COMPLETE (Phase 6)
- Critical notifications: ⏳ PLANNED (Phase 7 - Next)

### Typical Workflow

1. **Edit** `esp32-pileta.yaml` to modify ESP32 logic (or edit directly in ESPHome Dashboard)
2. **Test locally** if possible with validation
3. **Commit changes** to Git with descriptive message
4. **Push to GitHub** to maintain backup and history
5. **Copy** file to HA via SCP or clipboard
6. **Configure secrets** in ESPHome Dashboard UI if needed
7. **Compile** via ESPHome Dashboard in HA
8. **Flash** firmware (USB first time, then OTA)
9. **Monitor** logs in ESPHome Dashboard
10. **Test** control logic via HA interface
11. **Document** any issues or changes in commit messages

### Important Constraints

**Version Control:**
- **Always commit and push changes regularly**
- Use descriptive commit messages
- Never commit sensitive data (WiFi credentials, API keys)
- `.gitignore` protects secrets and build artifacts

**Security:**
- WiFi credentials and secrets are managed securely in ESPHome Dashboard
- API encryption keys and OTA passwords are stored in the YAML configuration
- Never commit actual WiFi credentials to Git

**Hardware:**
- GPIO2 is a strapping pin - may affect boot sequence if misconfigured
- ESP32 WiFi is 2.4GHz only

**Connectivity:**
- WiFi connection is unstable - design logic to handle disconnections
- ESP32 must operate autonomously
- Store fallback values locally

**Compilation:**
- Do NOT compile on HA host directly (memory constraints)
- Use ESPHome Dashboard exclusively

## File Structure

```
pool_heat_esp32/
├── esp32-pileta.yaml           # Main ESPHome configuration
├── instructions.md             # Detailed requirements for automation logic
├── README.md                   # Project documentation
├── CLAUDE.md                   # This file
├── WARP_HA_DEBUG.md           # Home Assistant debugging guide
├── .gitignore                  # Protects build artifacts and secrets
├── LICENSE                     # MIT License
├── docs/
│   ├── COMPILATION_GUIDE.md   # Detailed compilation instructions (all methods)
│   └── SHELL_TIPS.md          # Shell troubleshooting
└── log/                        # Log output directory
```

## Key Design Patterns

### ESP32 Autonomy
The ESP32 must operate independently due to WiFi instability:
- Pull configuration parameters from HA on startup
- Store last-known-good values locally
- Continue operation if HA becomes unavailable
- Re-sync when connection restored
- Fail-safe: turn off pumps on shutdown

### Dead Zone Logic
Prevent rapid on/off cycling:
- Turn ON threshold: SAG < (ITO - 0.5°C)
- Turn OFF threshold: SAG ≥ ITO
- This 0.5°C dead zone prevents oscillation

### Safety First
Multiple layers of automatic shutdowns:
- Individual relay timers (1h pump when skimmer, 8h heater)
- Combined heating mode timer (8h)
- Master disable switches (IAC for heating, IAS for skimmer)
- Heating has absolute priority over skimmer

## Troubleshooting

### WiFi Connection Issues
- Verify 2.4GHz WiFi network
- Check WiFi credentials in ESPHome Dashboard
- Monitor logs in ESPHome Dashboard
- Fallback AP activates if main WiFi fails:
  - SSID: "Esp32-Pileta Fallback Hotspot"
  - Password: "KHIUDZeptBI2"

### Configuration Errors
- Validate YAML syntax in ESPHome Dashboard
- Check indentation (spaces, not tabs)
- Verify WiFi secrets are configured in dashboard

### GPIO2 Boot Issues
- GPIO2 is a strapping pin
- If boot fails consistently, may need to change pin assignment
- Monitor serial output during boot

### Home Assistant Add-on Issues
- See `WARP_HA_DEBUG.md` for comprehensive debugging procedures
- Check resource usage: `ha addons stats`
- Restart add-on if needed: `ha addons restart 5c53de3b_esphome`

## Platform Notes

**Development Environment:**
- OS: Windows
- Shell: PowerShell 5.1
- Working Directory: `C:\Users\RamonSantiagoIgarret\bin\pool_heat_esp32`

**Deployment Target:**
- Home Assistant: `hassio@192.168.1.7`
- ESPHome Directory: `/config/esphome/`
- Access: SSH or ESPHome Dashboard Web UI

**Version Control:**
- Git repository: https://github.com/igarreta/pool_heat_esp32.git
- Branch: main
- Commit and push changes regularly
