# Shell Tips and Common Issues

## Heredoc Quote Problems (bquote>)

### Problem: Stuck in bquote> or quote> prompt

**Symptom**: After running a command with heredoc or quotes, your shell shows:
```
bquote>
```
or
```
quote>
```

**Cause**: Unmatched quotes in the command. The shell is waiting for you to close the quote.

### Solution 1: Cancel the Command
Press **Ctrl+C** to abort and return to normal prompt.

### Solution 2: Prevent the Issue

#### Use Alternative Heredoc Delimiters
Instead of `EOF`, use a unique delimiter that won't conflict with content:

**Bad** (can cause issues if EOF appears in content):
```bash
cat > file.txt << EOF
content here
EOF
```

**Good** (unique delimiter):
```bash
cat > file.txt << 'ENDOFFILE'
content here
ENDOFFILE
```

**Best** (quoted delimiter prevents variable expansion issues):
```bash
cat > file.txt << 'END_OF_FILE_MARKER'
content here
END_OF_FILE_MARKER
```

#### Quote the Heredoc Delimiter
Always quote the delimiter to prevent variable expansion and quote issues:

```bash
# Good - prevents most issues
cat > file.txt << 'EOF'
content with $variables and "quotes"
EOF
```

```bash
# Also good - different delimiter
cat > file.txt << 'END'
content here
END
```

#### Alternative Methods to Avoid Heredocs

**Method 1: Use echo with quotes**
```bash
echo 'line 1
line 2
line 3' > file.txt
```

**Method 2: Use printf**
```bash
printf '%s\n' \
  'line 1' \
  'line 2' \
  'line 3' > file.txt
```

**Method 3: Multiple echo statements**
```bash
{
  echo 'line 1'
  echo 'line 2'
  echo 'line 3'
} > file.txt
```

**Method 4: Use tee (for complex content)**
```bash
tee file.txt > /dev/null << 'ENDOFFILE'
complex content here
ENDOFFILE
```

## Escaping Special Characters

### In Double Quotes
These need escaping: `$` `` ` `` `\` `"`

```bash
echo "Price: \$100"
echo "Command output: \`date\`"
echo "Quote: \"Hello\""
```

### In Single Quotes
Almost nothing needs escaping (except single quote itself):

```bash
echo 'Price: $100'  # $ is literal
echo 'Command: `date`'  # backtick is literal
echo 'Quote: "Hello"'  # double quote is literal
```

To include a single quote in single-quoted string:
```bash
echo 'It'\''s working'  # ends quote, adds escaped quote, starts quote again
# Or use double quotes for that part:
echo 'It'"'"'s working'
```

## Safe File Creation Commands

### For Small Files
```bash
# Using echo (single line)
echo 'content' > file.txt

# Using echo (multiple lines with \n)
echo -e 'line1\nline2\nline3' > file.txt

# Using printf
printf '%s\n' 'line1' 'line2' > file.txt
```

### For Multi-line Files
```bash
# Using heredoc with quoted delimiter (safest)
cat > file.txt << 'MARKER'
line 1
line 2
line 3
MARKER

# Using command grouping
{
  echo 'line 1'
  echo 'line 2'
  echo 'line 3'
} > file.txt
```

### For Files with Complex Content
```bash
# Create in text editor
nano file.txt
# or
vi file.txt

# Or copy from existing template
cp template.txt newfile.txt
```

## Debugging Shell Commands

### Check for Unmatched Quotes
```bash
# Look for unmatched quotes in your command history
history | tail -1
```

### Verify Command Before Running
```bash
# Add echo to see what will execute
echo cat > file.txt << 'EOF'
# Review output before running actual command
```

### Use Shell's Built-in Syntax Check
```bash
# For bash/zsh, check syntax without executing
bash -n script.sh
zsh -n script.sh
```

## Recovery Commands

### If Stuck in a Prompt

| Prompt | Meaning | Solution |
|--------|---------|----------|
| `>` | Incomplete command | Ctrl+C or complete the command |
| `>>` | Heredoc continuation | Type delimiter or Ctrl+C |
| `quote>` | Unmatched quote | Type closing quote or Ctrl+C |
| `dquote>` | Unmatched double quote | Type `"` or Ctrl+C |
| `bquote>` | Unmatched backtick | Type `` ` `` or Ctrl+C |
| `heredoc>` | Heredoc input | Type delimiter or Ctrl+C |

### Universal Escape
**Ctrl+C** - Cancels current command and returns to prompt

### If Shell Becomes Unresponsive
1. Try Ctrl+C (once)
2. Try Ctrl+D (logout/exit)
3. Try Ctrl+Z (suspend) then `kill %1`
4. Close terminal and reconnect

## Best Practices for This Project

### When Creating Documentation Files

**Use quoted heredoc delimiters:**
```bash
cat > README.md << 'ENDOFFILE'
# Documentation content
...
ENDOFFILE
```

**Or use multiple echo statements:**
```bash
{
  echo '# Documentation Title'
  echo ''
  echo '## Section 1'
  echo 'Content here'
} > README.md
```

### When Editing Configuration Files

**Prefer text editors:**
```bash
nano secrets.yaml
vi esp32-pileta-hybrid.yaml
```

**Or use simple echo for key-value pairs:**
```bash
echo 'wifi_ssid: "MyNetwork"' > secrets.yaml
echo 'wifi_password: "MyPassword"' >> secrets.yaml
```

### When Running ESPHome Commands

**Use full paths and redirect output:**
```bash
esphome config /root/projects/pool_heat_esp32/esp32-pileta-hybrid.yaml 2>&1 | tee log/output.log
```

## PowerShell (Windows) Specific Tips

### Creating Multi-line Files in PowerShell

**Method 1: Here-String (PowerShell equivalent of heredoc)**
```powershell
@"
line 1
line 2
line 3
"@ | Out-File -FilePath "file.txt" -Encoding UTF8
```

**Method 2: Array of strings**
```powershell
$content = @(
    "line 1",
    "line 2",
    "line 3"
)
$content | Out-File -FilePath "file.txt" -Encoding UTF8
```

**Method 3: Set-Content**
```powershell
Set-Content -Path "file.txt" -Value "line 1`nline 2`nline 3"
```

### PowerShell Quoting Rules
- Single quotes `'`: Literal strings (no variable expansion)
- Double quotes `"`: Allow variable expansion with `$variable`
- Backtick `` ` ``: Escape character (like `\` in bash)

## Quick Reference

### Safe Heredoc Template (Bash/Zsh)
```bash
cat > filename << 'MARKER'
content
MARKER
```

### Safe Multi-line Echo (Bash/Zsh)
```bash
echo 'line1
line2
line3' > filename
```

### Safe Command Grouping (Bash/Zsh)
```bash
{
  command1
  command2
  command3
} > filename
```

### Safe File Creation (PowerShell)
```powershell
@"
content line 1
content line 2
"@ | Out-File "filename.txt"
```

### Check Current Shell State
```bash
echo $SHELL      # Current shell (Linux/Mac)
set -o          # Shell options
echo $PS1       # Primary prompt
```

```powershell
$PSVersionTable  # PowerShell version info
Get-Host         # Host information
```

## Additional Resources

### For Bash/Zsh Users
- ZSH Manual: `man zshall`
- Bash Manual: `man bash`
- Shell tutorial: https://www.shellscript.sh/
- Quoting guide: https://mywiki.wooledge.org/Quotes

### For PowerShell Users
- PowerShell Docs: https://docs.microsoft.com/powershell/
- About Quoting: `Get-Help about_Quoting_Rules`
- About Special Characters: `Get-Help about_Special_Characters`

---

**Remember**: When in doubt, use **Ctrl+C** to cancel and start over!

## Common Scenarios and Solutions

### Scenario 1: Creating YAML file with secrets

**Problem**: Special characters in password causing issues

**Solution**:
```bash
# Use single quotes to prevent interpretation
cat > secrets.yaml << 'END'
wifi_ssid: "MyNetwork"
wifi_password: "P@ssw0rd$pecial!"
END
```

Or in PowerShell:
```powershell
@'
wifi_ssid: "MyNetwork"
wifi_password: "P@ssw0rd$pecial!"
'@ | Out-File secrets.yaml -Encoding UTF8
```

### Scenario 2: Running command with output redirection

**Problem**: Command produces no output or errors

**Solution**:
```bash
# Capture both stdout and stderr
esphome config file.yaml 2>&1 | tee output.log
```

PowerShell:
```powershell
esphome config file.yaml 2>&1 | Tee-Object output.log
```

### Scenario 3: Copying files between Windows and Linux

**Via SCP**:
```powershell
# From Windows to Linux
scp file.yaml user@192.168.1.7:/path/to/destination/

# From Linux to Windows (run on Linux)
scp file.yaml user@windowspc:/c/Users/user/Documents/
```

**Via WSL** (Windows Subsystem for Linux):
```bash
# Access Windows files from WSL
cd /mnt/c/Users/YourName/Documents

# Access WSL files from Windows
# In Explorer: \\wsl$\Ubuntu\home\username
```

---

**Pro Tip**: Keep a backup of your configuration files before experimenting with shell commands!
