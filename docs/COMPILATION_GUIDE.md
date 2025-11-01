# ESPHome Compilation Guide

## Quick Start - Recommended Method

### Using Home Assistant ESPHome Dashboard

This is the easiest and recommended method for compiling firmware within Home Assistant.

#### Step 1: Copy Files to ESPHome Directory
```bash
cp esp32-pileta-hybrid.yaml /config/esphome/
cp secrets.yaml /config/esphome/
```

Or via SCP from Windows:
```powershell
scp esp32-pileta-hybrid.yaml hassio@192.168.1.7:/config/esphome/
scp secrets.yaml hassio@192.168.1.7:/config/esphome/
```

#### Step 2: Access ESPHome Dashboard
1. Open Home Assistant in your browser
2. Go to **Settings** > **Add-ons**
3. Click on **ESPHome**
4. Click **Open Web UI**

#### Step 3: Compile and Install
1. You should see "esp32-pileta-hybrid" in the dashboard
2. Click the three dots menu on the device card
3. Choose **Install**
4. Select installation method:
   - **Wirelessly** (if device already has ESPHome)
   - **Plug into this computer** (USB connection)
   - **Manual download** (download .bin file)

#### Step 4: Monitor
After installation, click **Logs** to monitor the device.

---

## Alternative Methods

### Method A: ESPHome Web (Browser-Based)

**Best for**: First-time flashing via USB from any computer

**Requirements**:
- Chrome or Edge browser
- USB cable connected to ESP32
- No software installation needed

**Steps**:
1. Visit https://web.esphome.io/
2. Click **Connect**
3. Select USB port (e.g., COM3, /dev/ttyUSB0)
4. Click **Install** > **Prepare for first use**
5. When prompted, paste your configuration or upload YAML file
6. Wait for cloud compilation and automatic flashing

**Advantages**:
- No local setup required
- Works from any OS
- Cloud-based compilation (no local memory needed)
- Direct USB flashing

---

### Method B: Local Compilation (High-Spec Machine)

**Best for**: Development and testing with full control

**Requirements**:
- 8GB+ RAM
- Python 3.9+
- 20GB free disk space
- ESPHome CLI

**Setup (Windows)**:
```powershell
# Install Python from python.org if not already installed
# Then install ESPHome
pip install esphome

# Verify installation
esphome version
```

**Setup (Linux/Mac)**:
```bash
# Install ESPHome
pip3 install esphome

# Verify installation
esphome version
```

**Compile**:
```bash
cd pool_heat_esp32
esphome compile esp32-pileta-hybrid.yaml
```

**Upload via USB**:
```bash
esphome upload esp32-pileta-hybrid.yaml --device COM3
# Or on Linux/Mac:
esphome upload esp32-pileta-hybrid.yaml --device /dev/ttyUSB0
```

**Upload via OTA** (after initial USB flash):
```bash
esphome run esp32-pileta-hybrid.yaml
```

**View Logs**:
```bash
esphome logs esp32-pileta-hybrid.yaml
```

---

### Method C: Manual Flash with Pre-compiled Binary

**Best for**: Deploying known-good firmware to multiple devices

**Requirements**:
- esptool.py
- Pre-compiled .bin file

**Setup**:
```bash
pip install esptool
```

**Find Binary** (after successful compilation):
```bash
# Windows
dir /s *.bin

# Linux/Mac
find .esphome/build -name "*.bin"
```

**Flash (Windows)**:
```powershell
esptool.py --chip esp32 --port COM3 --baud 460800 --before default_reset --after hard_reset write_flash -z --flash_mode dio --flash_freq 40m --flash_size detect 0x0 firmware.bin
```

**Flash (Linux/Mac)**:
```bash
esptool.py --chip esp32 \
  --port /dev/ttyUSB0 \
  --baud 460800 \
  --before default_reset \
  --after hard_reset \
  write_flash -z \
  --flash_mode dio \
  --flash_freq 40m \
  --flash_size detect \
  0x0 firmware.bin
```

**Verify**:
```bash
esptool.py --port COM3 verify_flash 0x0 firmware.bin
```

---

## Troubleshooting Compilation

### Memory Errors During Build

**Symptom**: "memory allocation failed"

**Solutions**:
1. Use ESPHome Dashboard (optimized)
2. Use ESPHome Web (cloud compilation)
3. Compile on machine with more RAM
4. Close other applications
5. Clean cache: `rm -rf .esphome ~/.platformio` (Linux/Mac) or delete folders manually (Windows)

### Configuration Errors

**Symptom**: YAML validation fails

**Check**:
```bash
esphome config esp32-pileta-hybrid.yaml
```

**Common Issues**:
- Missing secrets.yaml
- Incorrect indentation (use spaces, not tabs)
- Invalid GPIO pins
- Typos in component names
- Special characters in strings (use quotes)

### USB Connection Issues

**Windows**:
- Install USB drivers for your ESP32 (CH340, CP2102, etc.)
- Check Device Manager for COM port
- Try different USB ports
- Use quality USB cable (data, not charge-only)

**Linux**:
- Add user to dialout group: `sudo usermod -a -G dialout $USER`
- Check permissions: `ls -l /dev/ttyUSB*`
- May need: `sudo chmod 666 /dev/ttyUSB0`

**Mac**:
- Install USB drivers if needed
- Check: `ls /dev/tty.*`
- Port usually shows as `/dev/tty.usbserial-*`

### Build Takes Too Long

**Normal**: First build downloads dependencies (5-15 minutes)

**Speed Up**:
- Keep .esphome directory (incremental builds)
- Use wired internet connection
- Use ESPHome Dashboard (pre-cached dependencies)
- Use SSD instead of HDD

---

## Post-Installation Verification

### Check WiFi Connection
```bash
esphome logs esp32-pileta-hybrid.yaml
```

Look for:
- "WiFi connected"
- IP address assignment
- "API server started"

### Verify Home Assistant Integration
1. Go to **Settings** > **Devices & Services**
2. Look for "ESPHome" integration
3. Device should auto-discover
4. Click **Configure** to add

### Test Functionality
- Check sensor readings appear in HA
- Test switches/controls
- Verify entity states update
- Monitor for errors in logs

---

## Optimization Tips

### Reduce Build Time
Add to your YAML:
```yaml
esphome:
  platformio_options:
    build_flags:
      - -DCORE_DEBUG_LEVEL=0
    build_unflags:
      - -Os
    build_flags:
      - -O2
```

### Reduce Memory Usage
Use specific framework version:
```yaml
esp32:
  framework:
    type: arduino
    version: recommended
```

### Enable Faster OTA
```yaml
ota:
  safe_mode: false
  num_attempts: 3
```

---

## Backup and Recovery

### Backup Current Firmware (Windows)
```powershell
esptool.py --port COM3 read_flash 0x0 0x400000 backup.bin
```

### Backup Current Firmware (Linux/Mac)
```bash
esptool.py --port /dev/ttyUSB0 \
  read_flash 0x0 0x400000 backup.bin
```

### Restore from Backup
```bash
esptool.py --port COM3 write_flash 0x0 backup.bin
```

### Factory Reset
```bash
esptool.py --port COM3 erase_flash
```

---

## Platform-Specific Notes

### Windows
- Use PowerShell or Command Prompt
- COM ports: COM1, COM3, COM5, etc.
- Install Python from python.org
- May need admin rights for driver installation

### Linux
- Use terminal
- Ports: /dev/ttyUSB0, /dev/ttyACM0
- May need sudo for USB access
- Driver usually built into kernel

### macOS
- Use Terminal
- Ports: /dev/tty.usbserial-*, /dev/tty.SLAB_USBtoUART
- May need to install drivers
- Use homebrew for Python: `brew install python`

---

## Resources

- [ESPHome Documentation](https://esphome.io/)
- [ESPHome Web Tool](https://web.esphome.io/)
- [PlatformIO Docs](https://docs.platformio.org/)
- [ESP32 Technical Reference](https://www.espressif.com/en/products/socs/esp32)
- [esptool.py Documentation](https://docs.espressif.com/projects/esptool/)

---

## Common Error Messages

### "Could not find COM port"
- Check USB connection
- Install drivers
- Try different port
- Check Device Manager (Windows)

### "Failed to connect"
- Device may be in wrong boot mode
- Hold BOOT button while connecting
- Check baud rate
- Try lower baud rate: `--baud 115200`

### "YAML validation error"
- Check indentation (2 spaces per level)
- Verify all quotes are matched
- Check for special characters
- Validate with: `esphome config file.yaml`

### "Out of memory"
- Close other applications
- Use ESPHome Web instead
- Compile on different machine
- Clean build cache

---

**Pro Tip**: Start with ESPHome Web for your first flash, then use OTA updates for subsequent changes!
