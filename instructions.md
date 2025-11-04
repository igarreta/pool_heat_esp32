# Pool Heat Automation Project

## Task Description
Create a Home Assistant automation to control pool solar heating system with intelligent temperature management and safety features.
The automation will run on an ESP32 due to unstable WiFi connection. 
Home Assistant will provide the fronting (web interface) to configure the automation. It will change the values of the input entities and provide the status of the system.

## Context
- **Location:** Buenos Aires, Argentina
- **Home Assistant:** Running on Proxmox server as a VM (HAOS) 
- **Control Method:** Automations and scripts (no Node-RED). Programming will be done via ESPHome Builder in Home Assistant
- **Hardware:** ESP32 with occasional WiFi connectivity issues

## Entities

### Heating Control
- **IAC**: `input_boolean.activar_calefaccion_pileta` - Master enable/disable heating
- **SWA**: `switch.pileta_exp32_calefaccion_completa_esp32` - Heating on/off switch
- **SAG**: `sensor.esp32_pileta_temperatura_agua` - Water temperature in pipe
- **SCL**: `sensor.esp32_pileta_temperatura_calefactor` - Roof heater temperature
- **ITO**: `input_number.pileta_temp_objetivo` - Target pool temperature
- **IMX**: `input_number.pileta_temp_max_diff` - Temp difference to turn ON heating
- **IMI**: `input_number.pileta_temp_min_diff` - Temp difference to turn OFF heating

### Skimmer Control
- **IAS**: `input_boolean.activar_skimmer_pileta` - Master enable/disable skimmer
- **Pump**: `pileta_bomba_esp` (GPIO2) - Shared pump for heating and skimmer
- **Runtime Today**: `sensor.esp32_pileta_horas_bomba_hoy` - Daily pump runtime
- **Runtime Yesterday**: `sensor.esp32_pileta_horas_bomba_ayer` - Previous day runtime

## Logic Requirements

### **Operating Window**
- Master control: Only when IAC is enabled
- Safety shutdown: To be implemented later

### **Turn ON Heating Conditions (ALL must be true):**
1. IAC is enabled
3. SAG < (ITO - 0.5°C) [Dead zone prevents rapid cycling]
4. SCL > (SAG + IMX) [Heater is sufficiently hot]
5. In this case turn ON pileta_calefaccion_completa

### **Turn OFF Heating Conditions (ANY is true):**
1. SAG ≥ ITO [Target temperature reached]
2. SCL ≤ (SAG + IMI) [Heater insufficient temperature]  
3. IAC is disabled
5. In this case turn OFF pileta_calefaccion_completa

### Safety timers
- pileta_calefaccion_completa is a template that turns ON pileta_bomba_esp and pileta_bomba_esp 
- pileta_bomba_esp has a 1 hour timer
- pileta_bomba_esp has a 8 hour timer
- Logic must account for these automatic shutdowns


### **Parameter Validation**
- Ensure IMX ≥ (IMI + 1°C) to prevent logic conflicts
- Alert if this condition is violated

### **ESP32 Disconnection Handling (Aggressive Approach)**
- System should be able to handle ESP32 disconnections. 
- At startup should try to retreive current values from Home Assistant
- If values are not available, should start with previous values (stored in helper entities)   
- When new values are received, should update helper entities
- At shutdown should stop both pumps

### **Runtime Tracking**
- Track daily pump runtime in `bomba_horas_hoy` (updates every 60s when pump ON)
- Store previous day's runtime in `bomba_horas_ayer` at midnight before reset
- Reset `bomba_horas_hoy` to 0 at midnight
- Exposed to HA as sensors for monitoring

### **Skimmer Automation Logic**

**Purpose:** Ensure consistent daily pool filtration for water quality

**Master Control:**
- Controlled by `input_boolean.activar_skimmer_pileta` (IAS)
- When IAS turns OFF, skimmer logic disabled (does NOT force pump off if heating active)

**Scheduled Mode (requires HA time sync):**
- **7:00 trigger:** Run pump for 1 hour IF:
  - IAS is enabled AND
  - `bomba_horas_ayer` ≤ 3.0 hours AND
  - `bomba_modo_calefaccion` is false (heating not active)

- **20:00 trigger:** Run pump for 1 hour IF:
  - IAS is enabled AND
  - `bomba_horas_hoy` ≤ 2.5 hours AND
  - `bomba_modo_calefaccion` is false (heating not active)

**Fallback Mode (no HA time sync):**
- Run pump for 1 hour every 12 hours from boot IF:
  - IAS is enabled AND
  - `bomba_modo_calefaccion` is false
- Automatically disabled once HA time sync established
- Uses `millis()` to track elapsed time

**Conflict Resolution:**
- Heating has **absolute priority**
- Skimmer NEVER starts if heating is active
- If heating starts while skimmer running, existing 1-hour timer allows heating flag to take over
- Skimmer runtime counts toward `bomba_horas_hoy`

**Safety:**
- Existing 1-hour auto-shutoff protects pump (when NOT in heating mode)
- Master switch OFF only disables skimmer scheduling, not pump operation
- No interference with heating 8-hour continuous limit

### **Critical Event Notifications (Phase 7)**

**Purpose:** Alert user of critical system failures via Pushover

**Notification Method:**
- ESP32 calls Home Assistant service: `notify.pushover`
- HA handles Pushover API communication
- Works over local network (not affected by internet issues)

**Events to Notify (Priority 0, No Daily Repeats):**

1. **Sensor Range Violations:**
   - Water temp < 0°C or > 50°C
   - Heater temp < 0°C or > 80°C
   - Message: "⚠️ Pool: [Sensor] fuera de rango (XX°C)"

2. **Sensor Staleness:**
   - No update for >5 minutes
   - Message: "⚠️ Pool: Sensor [name] sin actualizar (>5 min)"

3. **Pump Desync:**
   - Pumps don't match expected heating state
   - Message: "⚠️ Pool: Desincronización de bombas detectada"

4. **Runtime Watchdog:**
   - 8-hour continuous limit reached
   - Message: "⚠️ Pool: Límite de 8 horas alcanzado"

5. **Parameter Validation:**
   - IMX < IMI + 1°C auto-corrected
   - Message: "⚠️ Pool: Parámetros inválidos auto-corregidos"

**Notification Throttling:**
- Each notification type can only be sent once per day
- Flags reset at midnight
- Prevents notification spam

**Manual Test:**
- Test button exposed to HA: `button.esp32_pileta_test_pushover_notification`
- Sends test message to verify notification path
- Use after deployment or configuration changes

**Events NOT Notified:**
- WiFi disconnection (notifications wouldn't work anyway)
- Normal operation events
- Sensor readings
- Control logic evaluations
- Heating/skimmer cycles

## Requirements
- Minimize rapid on/off cycling with dead zones
- Robust error handling with delayed safety response
- Daily runtime logging for future integrations

## Files to Work With
- Main automations file location: /config/automations.yaml
- Helper entities configuration: /config/configuration.yaml (or via HA UI)

## Notes
- Dead zone of 0.5°C between turn-on and turn-off prevents oscillation
- IMX must be at least 1°C greater than IMI to prevent logic conflicts