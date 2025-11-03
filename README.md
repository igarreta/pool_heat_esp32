# Pool Heat ESP32 Project

ESPHome-based solar pool heating controller for ESP32-DevKit with Home Assistant integration.

## Overview

**Architecture:** Hybrid control system
- **ESP32:** Autonomous heating control logic (due to unreliable WiFi)
- **Home Assistant:** Configuration interface and monitoring dashboard
- **Communication:** MQTT + Home Assistant API

**Hardware:** ESP32-DevKit (Arduino framework)
**Location:** Buenos Aires, Argentina
**Home Assistant:** `hassio@192.168.1.7` (Proxmox VM)
**MQTT Broker:** `192.168.1.8`

## Hardware Configuration

### Sensors
- **3x Dallas DS18B20 Temperature Sensors** (One-Wire on GPIO25)
  - Water temperature: `0xd81c77d446f18a28` (30s updates)
  - Roof box temperature: `0x65011447d4f2aa28` (30s updates)
  - Heater temperature: `0xe0011448ab01aa28` (30s updates)
- **Light Sensor** (ADC on GPIO32) - Roof illumination (60s updates)
- **WiFi Signal Strength** (60s updates)

### Control Outputs
- **GPIO2:** Pool circulation pump relay
  - 1-hour auto-shutoff (only when NOT in heating mode)
  - Heating mode flag prevents premature shutoff
- **GPIO16:** Pool heater relay (8-hour auto-shutoff)
- **Virtual Switch:** "Calefaccion Completa (ESP32)"
  - Combined heating mode - activates pump + heater together
  - 8-hour auto-shutoff with independent watchdog backup
  - Manages heating mode flag for pump protection

**⚠️ GPIO2 Warning:** Strapping pin - monitor boot behavior

## Features

### Autonomous Heating Control
- **Automatic Operation:** ESP32 evaluates heating conditions every 30 seconds
- **Temperature Control:** Turns ON when water below target and heater sufficiently hot
- **Dead Zone Logic:** 0.5°C hysteresis prevents rapid cycling
- **Safety Shutdown:** Automatic stop when target reached or heater insufficient

### Configuration Interface (Home Assistant)
- **IAC (input_boolean):** Master enable/disable switch
- **ITO (input_number):** Target pool temperature setpoint
- **IMX (input_number):** Temperature delta to turn ON heating
- **IMI (input_number):** Temperature delta to turn OFF heating

### Daily Runtime Tracking
- **Pump Hours Counter:** Tracks daily operation time
- **Automatic Reset:** Midnight reset via time synchronization
- **HA Integration:** "Horas Bomba Hoy" sensor for monitoring
- **Future Use:** Enables skimmer automation based on filter runtime

### Comprehensive Safety System
- ✅ **Parameter Validation:** Enforces IMX ≥ IMI + 1°C automatically
- ✅ **Sensor Range Validation:** Water 0-50°C, heater 0-80°C
- ✅ **Staleness Detection:** 5-minute timeout for frozen sensors
- ✅ **Pump Desync Detection:** Verifies both pumps match expected state
- ✅ **Independent Watchdog:** Backup 8-hour timer prevents runaway operation
- ✅ **WiFi Resilience:** Continues with last-known values during disconnection
- All safety violations trigger immediate shutdown with error logging

### Communication
- **Home Assistant API:** Encrypted connection
- **MQTT Topics:**
  - `pool_heat/temp/water`
  - `pool_heat/temp/roof`
  - `pool_heat/temp/heater`
  - `pool_heat/light/roof`
- **Fallback WiFi AP:** "Esp32-Pileta Fallback Hotspot" (password: KHIUDZeptBI2)

## Project Structure

```
pool_heat_esp32/
├── esp32-pileta.yaml           # Main ESPHome configuration
├── instructions.md             # Control logic requirements
├── README.md                   # This file
├── PROJECT_STATUS.md           # Development status and session memory
├── CLAUDE.md                   # Claude Code guidance
├── WARP_HA_DEBUG.md           # Home Assistant debugging procedures
├── .gitignore                  # Protects secrets and build artifacts
├── LICENSE                     # MIT License
├── docs/
│   ├── COMPILATION_GUIDE.md   # Detailed compilation instructions
│   └── SHELL_TIPS.md          # Shell troubleshooting
└── log/                        # Log output directory
```

## Getting Started

### 1. Copy Configuration to Home Assistant

```powershell
# From Windows PowerShell
scp esp32-pileta.yaml hassio@192.168.1.7:/config/esphome/
```

Or copy YAML content via clipboard and paste into ESPHome Dashboard.

### 2. Configure WiFi Credentials

- WiFi secrets are managed in ESPHome Dashboard UI
- Navigate to: **Settings → Add-ons → ESPHome → Open Web UI**
- Edit device configuration to add WiFi SSID and password

### 3. Compile and Flash Firmware

1. Open ESPHome Dashboard in Home Assistant
2. Click on device card
3. Click **Install** and choose method:
   - **Wirelessly** (if already flashed)
   - **Plug into this computer** (USB for first flash)
   - **Manual download** (download .bin file)

### 4. Monitor and Test

- View logs in ESPHome Dashboard
- Verify sensor readings in Home Assistant
- Test control outputs

## Important Notes

### Security
- WiFi credentials managed in ESPHome Dashboard (not in Git)
- API encryption key and OTA password stored in YAML
- Never commit sensitive credentials

### Compilation
- **Do NOT compile on HA host directly** (memory constraints)
- **Use ESPHome Dashboard exclusively**
- First flash requires USB connection
- Subsequent updates can use OTA

### Connectivity
- ESP32 WiFi is 2.4GHz only
- WiFi connection is unstable by design - ESP32 operates autonomously
- Fallback AP activates if main WiFi fails

## Troubleshooting

### Configuration Validation
```bash
esphome config esp32-pileta.yaml
```

### WiFi Connection Issues
- Verify 2.4GHz network
- Check credentials in ESPHome Dashboard
- Monitor logs for connection status
- Use fallback AP if needed

### GPIO2 Boot Issues
- GPIO2 is a strapping pin
- If boot fails, may need to reassign pin
- Monitor serial output during boot

### Home Assistant Debugging
See `WARP_HA_DEBUG.md` for comprehensive HA troubleshooting procedures.

## Resources

- [ESPHome Documentation](https://esphome.io/)
- [ESPHome Web Tool](https://web.esphome.io/)
- [Home Assistant ESPHome Integration](https://www.home-assistant.io/integrations/esphome/)
- [ESP32 Documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/)
- [GitHub Repository](https://github.com/igarreta/pool_heat_esp32)

## License

MIT License - See LICENSE file for details
