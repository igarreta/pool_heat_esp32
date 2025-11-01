# Pool Heat ESP32 Project

## Project Overview
ESPHome-based pool heating controller using ESP32 for Home Assistant integration.

## Configuration Files

### Main Configuration
- **esp32-pileta-hybrid.yaml**: Main ESPHome configuration
  - Board: ESP32-DevKit
  - Framework: Arduino
  - Status: ✅ Validated successfully

### Secrets Management
- **secrets.yaml**: WiFi credentials storage
  - Location: `secrets.yaml`
  - Contains: `wifi_ssid` and `wifi_password`
  - Status: Template created - needs configuration

## Setup History

### Phase 1: Configuration Setup (2025-10-24)
1. ✅ Created `secrets.yaml` file with WiFi credential placeholders
2. ⏳ Update `secrets.yaml` with actual WiFi credentials
3. ⏳ Validate ESPHome configuration:
   ```bash
   esphome config esp32-pileta-hybrid.yaml
   ```

### Phase 2: Firmware Compilation
**Note**: Compilation on Home Assistant host failed due to memory constraints

**Error Details**:
- CMake/PlatformIO memory allocation failure (attempted 8GB allocation)
- Available system memory on HA: 3.8GB total, 2.5GB available
- Framework: ESP-IDF 5.4.2 with Arduino 3.2.1
- Error: "memory allocation of 8380369152 bytes failed"

**Root Cause**: Home Assistant environment has insufficient memory for ESP-IDF compilation

## Compilation Methods

### Method 1: Home Assistant ESPHome Dashboard (Recommended)
The ESPHome add-on in Home Assistant provides a web interface optimized for compilation.

**Steps**:
1. Open Home Assistant
2. Navigate to **Settings** → **Add-ons** → **ESPHome**
3. Click **Open Web UI**
4. Copy `esp32-pileta-hybrid.yaml` to `/config/esphome/` directory
5. Copy `secrets.yaml` to `/config/esphome/` directory
6. Click on the device in the dashboard
7. Click **Install** → **Manual download** (or **Wireless** if device is already online)
8. Download the compiled `.bin` file or install directly

**Advantages**:
- Optimized for Home Assistant environment
- Built-in OTA updates
- Automatic device discovery
- Log viewing

### Method 2: ESPHome Web (Browser-based)
Compile and install directly from your browser.

**Steps**:
1. Visit https://web.esphome.io/
2. Connect ESP32 via USB to your computer
3. Click **Connect**
4. Select your device port
5. Click **Install** → **Prepare for first use**
6. Upload your configuration files when prompted
7. Firmware will compile in the cloud and install directly

**Advantages**:
- No local installation required
- Works from any computer with Chrome/Edge browser
- USB connection ensures reliable flashing

### Method 3: Local Machine Compilation
If you have a local machine with more resources.

**Requirements**:
- Python 3.9 or later
- At least 8GB RAM recommended
- ESPHome CLI installed

**Installation**:
```bash
pip install esphome
```

**Compilation**:
```bash
esphome compile esp32-pileta-hybrid.yaml
```

**Upload via USB**:
```bash
esphome upload esp32-pileta-hybrid.yaml
```

**Upload via OTA** (after initial USB flash):
```bash
esphome run esp32-pileta-hybrid.yaml
```

### Method 4: Manual Flash with esptool
If you have a pre-compiled `.bin` file.

**Installation**:
```bash
pip install esptool
```

**Flash Command**:
```bash
esptool.py --port /dev/ttyUSB0 write_flash 0x0 firmware.bin
```

## Post-Installation Steps

### 1. Monitor Device Logs
```bash
esphome logs esp32-pileta-hybrid.yaml
```

### 2. Verify WiFi Connection
- Device should connect to configured WiFi network
- Check Home Assistant for automatic discovery

### 3. Home Assistant Integration
1. Navigate to **Settings** → **Devices & Services**
2. Look for **ESPHome** discovered device
3. Click **Configure**
4. Add to Home Assistant

### 4. Test Functionality
- Verify sensor readings
- Test control outputs
- Check entity states in Home Assistant

## Project Structure
```
pool_heat_esp32/
├── esp32-pileta-hybrid.yaml       # Main configuration (to be added)
├── secrets.yaml                    # WiFi credentials
├── README.md                       # This file
├── PROJECT_STATUS.md              # Current project status
├── docs/
│   ├── COMPILATION_GUIDE.md       # Detailed build instructions
│   └── SHELL_TIPS.md              # Shell tips and troubleshooting
└── log/                            # Log files
```

## Important Notes

### GPIO Warning
- **GPIO2** is a strapping PIN - use with care for I/O operations
- Current configuration uses GPIO2; ensure this doesn't conflict with boot mode

### Security
- `secrets.yaml` contains sensitive credentials
- Never commit `secrets.yaml` to version control
- Keep WiFi passwords secure

### Memory Requirements
- Direct compilation on Home Assistant host: ❌ Not recommended (insufficient memory)
- ESPHome Dashboard: ✅ Recommended (optimized for HA)
- Local compilation: ✅ Requires 8GB+ RAM
- Web-based compilation: ✅ No local requirements

## Troubleshooting

### Configuration Validation
```bash
esphome config esp32-pileta-hybrid.yaml
```

### Clean Build Cache
```bash
rm -rf .esphome/
```

### Check System Resources
```bash
free -h
df -h
```

### View Compilation Logs
```bash
cat log/compile.log
```

## Resources
- [ESPHome Documentation](https://esphome.io/)
- [ESPHome Web](https://web.esphome.io/)
- [Home Assistant ESPHome Integration](https://www.home-assistant.io/integrations/esphome/)
- [ESP32 Documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/)

## Hardware Configuration

### Sensors
- **3x Dallas DS18B20 Temperature Sensors** (One-Wire on GPIO25)
  - Water temperature: `0xd81c77d446f18a28` (30s updates)
  - Roof box temperature: `0x65011447d4f2aa28` (30s updates)  
  - Heater temperature: `0xe0011448ab01aa28` (30s updates)
- **Light Sensor** (ADC on GPIO32)
  - Roof illumination monitoring (60s updates)
- **WiFi Signal Strength** (60s updates)

### Control Outputs
- **GPIO2**: Pool circulation pump relay
  - Auto-shutoff: 1 hour after activation
- **GPIO16**: Pool heater relay
  - Auto-shutoff: 8 hours after activation
- **Virtual Switch**: Combined heating mode
  - Activates both pump + heater
  - Auto-shutoff: 8 hours after activation

### Communication
- **Home Assistant API**: Encrypted connection
- **MQTT**: Publishes sensor data to broker at `192.168.1.8`
  - Topics: `pool_heat/temp/water`, `pool_heat/temp/roof`, `pool_heat/temp/heater`, `pool_heat/ligth/roof`
- **Fallback WiFi AP**: "Esp32-Pileta Fallback Hotspot" (activates if main WiFi fails)

## Status Summary
- ✅ Project structure recreated
- ✅ Main configuration file restored (esp32-pileta-hybrid.yaml)
- ⏳ Need to update secrets.yaml with actual credentials
- 📋 Ready for validation and compilation

## Next Steps
1. ✅ ~~Add your ESP32 configuration file (`esp32-pileta-hybrid.yaml`)~~ COMPLETE
2. Update `secrets.yaml` with your WiFi credentials
3. Copy files to Home Assistant: `scp esp32-pileta-hybrid.yaml hassio@192.168.1.7:/config/esphome/`
4. Validate configuration: `esphome config esp32-pileta-hybrid.yaml`
5. Choose compilation method and flash firmware
