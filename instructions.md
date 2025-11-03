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
- **IAC**: `input_boolean.activar_calefaccion_pileta` - Master enable/disable control
- **SWA**: `switch.pileta_exp32_calefaccion_completa_esp32` - Heating on/off switch
- **SAG**: `sensor.esp32_pileta_temperatura_agua` - Water temperature in pipe
- **SCL**: `sensor.esp32_pileta_temperatura_calefactor` - Roof heater temperature  
- **ITO**: `input_number.pileta_temp_objetivo` - Target pool temperature
- **IMX**: `input_number.pileta_temp_max_diff` - Temp difference to turn ON heating
- **IMI**: `input_number.pileta_temp_min_diff` - Temp difference to turn OFF heating

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
- Track daily heating runtime in hours/minutes
- Store in helper entity for future skimmer automation logic
- Reset counter at midnight

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