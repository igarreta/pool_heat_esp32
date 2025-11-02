# Home Assistant Debugging Procedure

## Connection
1. **Check if already connected via SSH**
   - Verify environment: `uname -a` should show Alpine Linux
   - Check current directory: `pwd` should be `/root`

2. **If not connected, establish SSH connection:**
   ```bash
   ssh -o "MACs=hmac-sha2-256-etm@openssh.com" hasssio@192.168.1.7
   ```
   - Prompt for password when requested
   - Connection should land in Home Assistant OS (Alpine Linux)

## Initial Resource Check

### 1. System Overview
```bash
# Check load averages and uptime
uptime

# Get detailed CPU/memory statistics
vmstat 1 3

# Check load average from proc
cat /proc/loadavg
```

**Normal values:**
- Load average < 2.0 for typical setup
- CPU idle (id) > 80%
- Low swap usage

### 2. Home Assistant Core Status
```bash
# Core resource usage
ha core stats

# Core logs (last 50 lines)
ha core logs | tail -50

# Check for errors
ha core logs | grep -i "error\|warn" | tail -30
```

### 3. Add-on Resource Analysis
```bash
# List all installed add-ons
ha addons --raw-json

# Check specific add-on stats
ha addons stats <addon_slug>

# Common add-ons to check:
ha addons stats a0d7b954_vscode         # VS Code Server
ha addons stats a0d7b954_nodered        # Node-RED
ha addons stats 5c53de3b_esphome        # ESPHome
ha addons stats a0d7b954_appdaemon      # AppDaemon
ha addons stats core_samba              # Samba
```

**Look for:**
- `cpu_percent` > 50% (investigate further)
- `memory_percent` > 50% (potential memory leak)
- High network activity if unexpected

### 4. Check All Add-ons at Once
```bash
for addon in a0d7b954_vscode a0d7b954_nodered 5c53de3b_esphome a0d7b954_appdaemon a0d7b954_tailscale core_samba cebe7a76_hassio_google_drive_backup; do 
  echo "=== $addon ==="
  ha addons stats $addon 2>/dev/null || echo "Not installed"
  echo ""
done
```

## Diagnosing High CPU/Memory

### Step 1: Identify the Culprit
1. **Check container stats** - Find which add-on/core is using resources
2. **Review logs** - Check for errors, warnings, or unusual activity
3. **Check timing** - When did the issue start? Recent changes?

### Step 2: Review Logs
```bash
# Add-on logs
ha addons logs <addon_slug> | tail -50

# Look for specific patterns
ha addons logs <addon_slug> | grep -E "error|Error|warn|Warn|exception"

# Supervisor logs
ha supervisor logs | grep -i "cpu\|memory\|<addon_name>" | tail -50
```

### Step 3: Common Issues

#### VS Code Server High CPU
**Cause:** Usually GitLens extension or language server indexing

**Solution:**
```bash
# Restart first
ha addons restart a0d7b954_vscode

# If persists, stop temporarily
ha addons stop a0d7b954_vscode

# Check if CPU drops
sleep 10 && vmstat 1 3

# If confirmed, disable problematic extensions or reinstall:
ha addons uninstall a0d7b954_vscode
ha addons install a0d7b954_vscode
```

#### LocalTuya Connection Failures
**Symptom:** Repeated connection failures in core logs

**Check:**
```bash
ha core logs | grep -i "localtuya"
```

**Action:** Verify device IPs are reachable, devices are online

#### Database/Recorder Issues
**Check:**
```bash
ha core logs | grep -i "recorder\|database"
```

**Action:** May need to purge old data or optimize database

## Remediation Steps

### For High CPU Add-on:

1. **Restart the add-on:**
   ```bash
   ha addons restart <addon_slug>
   sleep 30
   ha addons stats <addon_slug>
   ```

2. **If restart doesn't help, stop it:**
   ```bash
   ha addons stop <addon_slug>
   ```

3. **Verify system recovers:**
   ```bash
   vmstat 1 3
   uptime
   ```

4. **Review add-on configuration:**
   ```bash
   ha addons info <addon_slug>
   ```

5. **Check for updates:**
   ```bash
   ha addons update <addon_slug>
   ```

6. **Last resort - reinstall:**
   ```bash
   ha addons uninstall <addon_slug>
   ha addons install <addon_slug>
   ```

### For Home Assistant Core Issues:

1. **Check integration health:**
   ```bash
   ha core logs | grep -i "error\|exception" | tail -50
   ```

2. **Restart core if needed:**
   ```bash
   ha core restart
   ```

3. **Check for updates:**
   ```bash
   ha core update
   ```

## Host System Checks

### Disk Space
```bash
ha host info
# Look at disk_free and disk_used
```

### Running Jobs
```bash
ha jobs info
```

### Network Status
```bash
ha network info
```

## Important Notes

1. **SSH container limitations:** The SSH add-on runs in its own container and cannot see processes from other containers (HA Core, other add-ons). Always use `ha` commands to check resource usage.

2. **CPU measurements:**
   - `top`/`ps` inside SSH container: Shows only SSH container processes
   - `vmstat`: Shows overall VM CPU usage (more accurate)
   - `ha addons stats`: Shows per-container usage (most accurate for diagnosis)

3. **Proxmox vs Container:** Proxmox may show higher CPU than what's visible inside due to virtualization overhead and all containers combined.

4. **Persistent issues:** If an add-on keeps consuming high resources after restart, consider:
   - Checking for extension/plugin issues
   - Reviewing configuration
   - Checking for upstream bugs
   - Uninstalling and reinstalling as last resort

## Quick Reference Command Sheet

```bash
# Connection
ssh -o "MACs=hmac-sha2-256-etm@openssh.com" hasssio@192.168.1.7

# Quick health check
uptime && ha core stats && vmstat 1 2

# Find high CPU add-on
ha addons stats a0d7b954_vscode
ha addons stats a0d7b954_nodered
ha addons stats 5c53de3b_esphome

# Emergency stop problematic add-on
ha addons stop <addon_slug>

# Verify recovery
vmstat 1 3
```

## Monitoring After Fix

After resolving an issue, monitor for 2-4 hours:

```bash
# Check every 15-30 minutes
watch -n 900 'date && uptime && ha addons stats a0d7b954_vscode'

# Or manually:
date && uptime && ha addons stats <addon_slug>
```

Look for:
- Load average stable < 2.0
- CPU idle > 80%
- Add-on CPU < 10% when idle
- Memory not growing over time

## Log File Location Reference

From user rules:
- **Secrets/configs:** `~/etc/`
- **Global scripts:** `~/bin/` (in PATH)
- **Project logs:** `<project_directory>/log/`

Document findings:
```bash
echo "$(date): VS Code Server high CPU - GitLens extension issue. Resolved by uninstall/reinstall" >> ~/log/ha-debug-$(date +%Y%m%d).log
```
