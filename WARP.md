# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This is an **ESPHome-based pool heating controller** for ESP32-DevKit that integrates with Home Assistant. The project is currently in a **reconstruction phase** after the original configuration files were deleted from the Home Assistant environment. The main ESPHome configuration file (`esp32-pileta-hybrid.yaml`) needs to be restored or recreated.

## Architecture

### Hardware Platform
- **Board**: ESP32-DevKit
- **Framework**: Arduino on ESP-IDF (5.4.2 + Arduino 3.2.1)
- **GPIO Warning**: GPIO2 is used in configuration - this is a strapping pin and may affect boot sequence

### Integration Stack
- **ESPHome**: Firmware framework for ESP32
- **Home Assistant**: Primary integration target
- **ESPHome API**: Communication protocol between ESP32 and HA

### Configuration Structure
- `esp32-pileta-hybrid.yaml` - Main ESPHome device configuration (restored)
- `secrets.yaml` - WiFi credentials and sensitive data (template exists, never committed to git)
- Build artifacts stored in `.esphome/` (gitignored)
- Compiled firmware: `.bin` and `.elf` files (gitignored)

### Hardware Components

**Temperature Sensors (Dallas DS18B20 on One-Wire GPIO25)**:
- Water temperature: `0xd81c77d446f18a28` - Monitors pool water temp (30s updates)
- Roof box temperature: `0x65011447d4f2aa28` - Monitors solar collector/roof box (30s updates)
- Heater temperature: `0xe0011448ab01aa28` - Monitors heating element temp (30s updates)

**Light Sensor**:
- Roof illumination: ADC on GPIO32 - Measures ambient light levels (60s updates)

**Control Outputs**:
- **GPIO2**: Pool circulation pump relay (1-hour auto-shutoff timer)
- **GPIO16**: Pool heater relay (8-hour auto-shutoff timer)

**Virtual Controls**:
- Combined heating mode: Activates both pump + heater (8-hour auto-shutoff)

**Communication**:
- Home Assistant API with encryption
- MQTT broker at `192.168.1.8` for sensor data publishing
- Fallback AP: "Esp32-Pileta Fallback Hotspot" (if WiFi fails)

## Essential Commands

### ESPHome Operations

**Validate configuration:**
```powershell
esphome config esp32-pileta-hybrid.yaml
```

**Compile firmware:**
```powershell
esphome compile esp32-pileta-hybrid.yaml
```

**Upload via USB (first flash or recovery):**
```powershell
esphome upload esp32-pileta-hybrid.yaml --device COM3
```
*Note: Replace COM3 with actual port from Device Manager*

**Upload via OTA (after initial USB flash):**
```powershell
esphome run esp32-pileta-hybrid.yaml
```

**Monitor device logs:**
```powershell
esphome logs esp32-pileta-hybrid.yaml
```

**Clean build cache (if needed):**
```powershell
Remove-Item -Recurse -Force .esphome
```

### Manual Flashing with esptool

For manual firmware deployment with pre-compiled binaries:

```powershell
esptool.py --chip esp32 --port COM3 --baud 460800 --before default_reset --after hard_reset write_flash -z --flash_mode dio --flash_freq 40m --flash_size detect 0x0 firmware.bin
```

### File Transfer to Home Assistant

**Copy files to ESPHome directory on HA:**
```powershell
scp esp32-pileta-hybrid.yaml hassio@192.168.1.7:/config/esphome/
scp secrets.yaml hassio@192.168.1.7:/config/esphome/
```

## Development Workflow

### Current Project Phase: Configuration Reconstruction

The project requires the following steps to become operational:

1. **Restore/Create Main Configuration**
   - Recreate or obtain `esp32-pileta-hybrid.yaml`
   - Place in project root

2. **Configure Secrets**
   - Edit `secrets.yaml` with actual WiFi credentials
   - Format: 
     ```yaml
     wifi_ssid: "NetworkName"
     wifi_password: "Password"
     ```

3. **Validate Configuration**
   - Run: `esphome config esp32-pileta-hybrid.yaml`
   - Verify no YAML syntax errors

4. **Choose Compilation Method**
   - **Option A**: ESPHome Dashboard (recommended for HA integration)
   - **Option B**: ESPHome Web (browser-based, easiest for first-time)
   - **Option C**: Local compilation (requires 8GB+ RAM)

5. **Flash Device**
   - First flash: MUST use USB connection
   - Subsequent updates: OTA available

6. **Verify Integration**
   - Monitor logs for WiFi connection
   - Confirm Home Assistant auto-discovery
   - Add device via Settings → Devices & Services

## Critical Constraints

### Memory Limitations
- **Home Assistant Host**: Only 3.8GB total RAM (2.5GB available)
- **Compilation Requirement**: 8GB+ RAM for ESP-IDF builds
- **Impact**: Direct compilation on HA host fails with memory allocation errors
- **Solution**: Use ESPHome Dashboard (optimized), ESPHome Web (cloud compilation), or external build machine

### Compilation Method Recommendations
1. **ESPHome Dashboard** - Best for HA environment, optimized for resource constraints
2. **ESPHome Web** - Cloud-based compilation, no local requirements, https://web.esphome.io/
3. **Local Compilation** - Only on machines with 8GB+ RAM and 20GB+ disk space

### Security Considerations
- `secrets.yaml` is gitignored and MUST NEVER be committed
- Contains WiFi credentials and potentially OTA passwords
- Use strong passwords for OTA updates

### Hardware Constraints
- Initial firmware flash REQUIRES USB connection (USB-to-Serial driver needed)
- GPIO2 is a strapping pin - may affect boot if not configured properly
- First boot requires holding BOOT button during flash if device is unresponsive

## File Structure

```
pool_heat_esp32/
├── esp32-pileta-hybrid.yaml    # Main ESPHome config (✅ restored)
├── secrets.yaml                 # WiFi credentials template (not committed)
├── README.md                    # Comprehensive project documentation
├── PROJECT_STATUS.md            # Current phase and next steps
├── WARP.md                      # This file
├── LICENSE                      # MIT License
├── .gitignore                   # Protects secrets and build artifacts
├── docs/
│   ├── COMPILATION_GUIDE.md    # Detailed build instructions (all methods)
│   └── SHELL_TIPS.md           # Shell troubleshooting for heredoc issues
└── log/                         # Log output directory
```

## Platform-Specific Notes

### Development Environment
- **OS**: Windows
- **Shell**: PowerShell 5.1
- **Working Directory**: `C:\Users\RamonSantiagoIgarret\bin\pool_heat_esp32`

### Deployment Target
- **Home Assistant**: `hassio@192.168.1.7`
- **ESPHome Directory**: `/config/esphome/`
- **Access Method**: SSH or ESPHome Dashboard Web UI

### USB Ports (Windows)
- Ports typically appear as: `COM3`, `COM4`, `COM5`, etc.
- Check Device Manager → Ports (COM & LPT) to identify ESP32 port
- May require CH340 or CP2102 USB-to-Serial drivers

### PowerShell Tips
- Use `Out-File` with `-Encoding UTF8` for YAML files
- Here-strings: `@' ... '@` for multi-line content
- File paths: Use double quotes and backslashes or forward slashes

## Alternative Workflows

### Using ESPHome Dashboard (Recommended)
1. Access HA: Settings → Add-ons → ESPHome → Open Web UI
2. Copy config files to `/config/esphome/`
3. Click Install → Choose method (Wireless/USB/Manual download)
4. Monitor logs directly in dashboard

### Using ESPHome Web (Easiest for First Flash)
1. Visit https://web.esphome.io/
2. Connect ESP32 via USB
3. Click Connect → Select COM port
4. Install → Prepare for first use
5. Upload configuration when prompted
6. Cloud compilation and automatic flashing

## Project Context

### Current Status
- **Phase**: Configuration Complete, Ready for Deployment (Phase 2 of 5)
- **Configuration**: ✅ Main YAML restored with full hardware setup
- **Git Repository**: https://github.com/igarreta/pool_heat_esp32.git
- **License**: MIT

### Historical Notes
- Original setup: 2025-10-24 on Home Assistant
- Memory constraint identified during compilation attempts
- Recovery: 2025-11-01 on local Windows machine
- Documentation preserved and enhanced
- Configuration restored: 2025-11-01 from backup

### Next Immediate Steps
1. ✅ ~~Obtain or recreate `esp32-pileta-hybrid.yaml`~~ (COMPLETE)
2. Update `secrets.yaml` with actual credentials
3. Copy files to Home Assistant: `scp esp32-pileta-hybrid.yaml hassio@192.168.1.7:/config/esphome/`
4. Validate configuration via ESPHome Dashboard
5. Compile and flash firmware

## Troubleshooting Quick Reference

### Configuration Errors
- Check YAML indentation (spaces, not tabs)
- Validate: `esphome config esp32-pileta-hybrid.yaml`
- Ensure `secrets.yaml` exists and has correct format

### Compilation Failures
- Memory errors: Use ESPHome Dashboard or Web instead of local compilation
- Clean cache: Delete `.esphome` directory
- Check available disk space (20GB+ recommended)

### USB Connection Issues
- Install USB-to-Serial drivers (CH340/CP2102)
- Try different USB ports or cables (must be data cable, not charge-only)
- Hold BOOT button during flash if device is unresponsive
- Check Device Manager for COM port assignment

### WiFi Connection Problems
- Verify credentials in `secrets.yaml`
- Check WiFi network is 2.4GHz (ESP32 limitation)
- Monitor logs: `esphome logs esp32-pileta-hybrid.yaml`
- Ensure strong signal at device location

## Additional Resources

- ESPHome Documentation: https://esphome.io/
- ESPHome Web Tool: https://web.esphome.io/
- Home Assistant ESPHome Integration: https://www.home-assistant.io/integrations/esphome/
- ESP32 Technical Reference: https://docs.espressif.com/projects/esp-idf/
- Detailed compilation guide: `docs/COMPILATION_GUIDE.md`
- Shell troubleshooting: `docs/SHELL_TIPS.md`
