# Ubuntu 24.04 LTS Server Setup Guide

Complete step-by-step installation guide for a secure Ubuntu server with Tailscale, Python data science environment, VS Code, JupyterLab, and more.

## ⚠️ Important Security Notice

**CRITICAL: Change All Default Passwords!**
This guide uses placeholder values like `ChangeThisToYourSecurePassword` and `YOUR_TAILSCALE_IP`.
**You MUST replace these with your own secure values before use!**

**Before deploying to production:**
- Use strong, unique passwords (20+ characters recommended)
- Enable 2FA on your Tailscale account
- Review and understand each command before executing
- Keep your system updated with security patches

## 📄 License

MIT License - Free to use, modify, and distribute. See LICENSE file for details.

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Quick Reference](#quick-reference)
- [Installation Steps](#installation-steps)
- [Verification](#verification)
- [Usage](#usage)
- [Maintenance](#maintenance)
- [Troubleshooting](#troubleshooting)

---

## Overview

This guide will help you set up a secure Ubuntu 24.04 LTS server with:

- **Security**: Tailscale VPN + nftables firewall
- **Development**: VS Code Server + JupyterLab
- **Data Science**: Python with NumPy, Pandas, scikit-learn, PyTorch
- **Management**: Cockpit web interface
- **File Sharing**: Samba
- **Backups**: Automated hourly backups with 15-day retention

All services are accessible only via Tailscale for maximum security.

---

## Prerequisites

- Fresh Ubuntu 24.04 LTS installation
- Root or sudo access
- Tailscale account (free at [tailscale.com](https://tailscale.com))

---

## Quick Reference

Once installed, access your services at these URLs (replace `YOUR_TAILSCALE_IP` with your actual Tailscale IP):

| Service | URL | Default Port |
|---------|-----|--------------|
| VS Code | `http://YOUR_TAILSCALE_IP:8080` | 8080 |
| JupyterLab | `http://YOUR_TAILSCALE_IP:8888` | 8888 |
| Cockpit | `http://YOUR_TAILSCALE_IP:9090` | 9090 |
| Samba | `\\YOUR_TAILSCALE_IP\share` | 445 |

**Find your Tailscale IP:** Run `tailscale ip -4` on your server

---

## Installation Steps

### 1. Install Tailscale

Install Tailscale and enable SSH access through the Tailscale network:

```bash
# Download and install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Verify installation
tailscale version

# Connect to your Tailscale network with SSH enabled
sudo tailscale up --ssh

# Check status and note your Tailscale IP
tailscale status
```

### 2. Disable Traditional SSH (Optional but Recommended)

For maximum security, disable the traditional SSH service since Tailscale provides secure SSH access:

```bash
sudo systemctl stop ssh
sudo systemctl disable ssh
sudo systemctl stop ssh.socket
sudo systemctl disable ssh.socket
```

### 3. Install nftables Firewall

Install the modern nftables firewall:

```bash
sudo apt update
sudo apt install -y nftables
```

### 4. Configure Firewall Rules

Create a firewall configuration that only allows access through Tailscale:

```bash
sudo tee /etc/nftables.conf > /dev/null <<'EOF'
#!/usr/sbin/nft -f
flush ruleset
table inet filter {
  chain input {
    type filter hook input priority filter; policy drop;
    iif "lo" accept
    iifname "tailscale0" accept
    ct state established,related accept
    ct state invalid drop
    log prefix "nft-drop: " limit rate 5/minute
    drop
  }
  chain forward {
    type filter hook forward priority filter; policy drop;
    iifname "tailscale0" accept
    oifname "tailscale0" accept
    ct state established,related accept
  }
  chain output {
    type filter hook output priority filter; policy accept;
  }
}
table ip6 filter {
  chain input {
    type filter hook input priority filter; policy drop;
    iif "lo" accept
    iifname "tailscale0" accept
    ct state established,related accept
  }
  chain forward {
    type filter hook forward priority filter; policy drop;
    iifname "tailscale0" accept
    oifname "tailscale0" accept
    ct state established,related accept
  }
  chain output {
    type filter hook output priority filter; policy accept;
  }
}
EOF
```

### 5. Configure Firewall Service Dependencies

Ensure the firewall starts after Tailscale:

```bash
sudo mkdir -p /etc/systemd/system/nftables.service.d
sudo tee /etc/systemd/system/nftables.service.d/tailscale.conf > /dev/null <<'EOF'
[Unit]
After=tailscaled.service
Wants=tailscaled.service
EOF
```

### 6. Enable and Start Firewall

```bash
sudo systemctl daemon-reload
sudo systemctl enable nftables
sudo nft -f /etc/nftables.conf
sudo systemctl start nftables
```

### 7. Install Cockpit Web Interface

Install Cockpit for easy server management through a web interface:

```bash
sudo apt install -y cockpit
sudo systemctl enable --now cockpit.socket
```

### 8. Install Cockpit File Managers

#### Install cockpit-navigator (Feature-rich file manager)

```bash
wget https://github.com/45Drives/cockpit-navigator/releases/download/v0.5.10/cockpit-navigator_0.5.10-1focal_all.deb
sudo dpkg -i cockpit-navigator_0.5.10-1focal_all.deb
```

#### Install cockpit-files (Official lightweight file manager)

```bash
sudo apt install -y cockpit-files
```

#### Restart Cockpit

```bash
sudo systemctl restart cockpit.socket
```

### 9. Configure Samba File Sharing

#### Install Samba

```bash
sudo apt install -y samba samba-common-bin
```

#### Create Shared Folder

Replace `YOUR_USERNAME` with your actual Linux username:

```bash
sudo mkdir -p /srv/share
sudo chown -R $USER:$USER /srv/share
sudo chmod -R 755 /srv/share
```

#### Configure Samba Share

```bash
sudo tee -a /etc/samba/smb.conf > /dev/null <<EOF

[share]
   comment = Shared Projects Folder
   path = /srv/share
   browseable = yes
   read only = no
   create mask = 0755
   directory mask = 0755
   valid users = $USER
   force user = $USER
   force group = $USER
EOF
```

#### Set Samba Password

```bash
sudo smbpasswd -a $USER
sudo smbpasswd -e $USER
```

#### Start Samba Service

```bash
sudo systemctl restart smbd
sudo systemctl enable smbd
```

**Access via:**
- Windows: `\\YOUR_TAILSCALE_IP\share`
- Mac/Linux: `smb://YOUR_TAILSCALE_IP/share`

### 10. Set Up Automated Hourly Backups

#### Create Backup Script

```bash
sudo tee /usr/local/bin/hourly-backup.sh > /dev/null <<'EOF'
#!/bin/bash
BACKUP_DIR="/timeshift/snapshots"
LOGFILE="/var/log/hourly-backup.log"
RETENTION_DAYS=15
DATE=$(date '+%Y-%m-%d %H:%M:%S')
TIMESTAMP=$(date '+%Y-%m-%d_%H-%M-%S')

log() { echo "[$DATE] $1" >> $LOGFILE; }

log "===== Starting Hourly Backup ====="
log "Creating new snapshot..."

/usr/bin/timeshift --create --scripted --comments "Hourly auto-backup $TIMESTAMP" >> $LOGFILE 2>&1

if [ $? -eq 0 ]; then
    log "✅ Snapshot created successfully: $TIMESTAMP"
else
    log "❌ ERROR: Snapshot creation failed!"
    exit 1
fi

log "Checking for old snapshots (older than $RETENTION_DAYS days)..."

find $BACKUP_DIR -maxdepth 1 -type d -mtime +$RETENTION_DAYS 2>/dev/null | while read old_snapshot; do
    if [ -d "$old_snapshot" ] && [ "$old_snapshot" != "$BACKUP_DIR" ]; then
        chattr -R -i "$old_snapshot" 2>/dev/null
        log "🗑️  Deleting old snapshot: $(basename $old_snapshot)"
        rm -rf "$old_snapshot"
        [ $? -eq 0 ] && log "✅ Successfully deleted: $(basename $old_snapshot)" || log "❌ Failed to delete: $(basename $old_snapshot)"
    fi
done

SNAPSHOT_COUNT=$(ls -1 $BACKUP_DIR 2>/dev/null | wc -l)
BACKUP_SIZE=$(du -sh /timeshift 2>/dev/null | awk '{print $1}')
log "📊 Current snapshot count: $SNAPSHOT_COUNT"
log "💾 Backup directory size: $BACKUP_SIZE"
log "===== Backup Process Complete ====="
log ""
EOF

sudo chmod +x /usr/local/bin/hourly-backup.sh
```

#### Create Systemd Service and Timer

```bash
# Create service file
sudo tee /etc/systemd/system/hourly-backup.service > /dev/null <<'EOF'
[Unit]
Description=Hourly Timeshift Backup
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/hourly-backup.sh
User=root
StandardOutput=journal
StandardError=journal
EOF

# Create timer file
sudo tee /etc/systemd/system/hourly-backup.timer > /dev/null <<'EOF'
[Unit]
Description=Hourly Backup Timer
Requires=hourly-backup.service

[Timer]
OnCalendar=hourly
Persistent=true
AccuracySec=5m

[Install]
WantedBy=timers.target
EOF

# Enable and start the timer
sudo systemctl daemon-reload
sudo systemctl enable hourly-backup.timer
sudo systemctl start hourly-backup.timer
```

#### Monitor Backups

```bash
# View backup log in real-time
sudo tail -f /var/log/hourly-backup.log

# List all backups
sudo timeshift --list

# Check next scheduled backup
systemctl list-timers | grep hourly
```

### 11. Install Python Data Science Environment

#### Install System Dependencies

```bash
sudo apt update
sudo apt install -y \
    python3 python3-pip python3-venv python3-dev \
    build-essential git curl wget \
    libopenblas-dev liblapack-dev gfortran \
    libhdf5-dev libfreetype-dev libpng-dev \
    libjpeg-dev pkg-config libatlas3-base libgfortran5
```

#### Create Data Science Environment

```bash
# Create project structure
mkdir -p /srv/share/data_science/{projects,datasets,notebooks,scripts}
cd /srv/share/data_science

# Create virtual environment
python3 -m venv ds_env
source ds_env/bin/activate

# Upgrade pip
pip install --upgrade pip setuptools wheel

# Install data science packages
pip install jupyter jupyterlab notebook ipython
pip install numpy pandas scipy
pip install matplotlib seaborn plotly
pip install scikit-learn
pip install requests beautifulsoup4 openpyxl sqlalchemy statsmodels
pip install torch torchvision torchaudio

# Save requirements for reproducibility
pip freeze > requirements.txt
deactivate
```

### 12. Configure JupyterLab Service

```bash
# Activate environment
cd /srv/share/data_science
source ds_env/bin/activate

# Generate Jupyter configuration
jupyter lab --generate-config

# Set password (you'll be prompted)
jupyter lab password

deactivate

# Create systemd service
sudo tee /etc/systemd/system/jupyter.service > /dev/null <<EOF
[Unit]
Description=Jupyter Lab
After=network.target

[Service]
Type=simple
User=$USER
WorkingDirectory=/srv/share/data_science
ExecStart=/srv/share/data_science/ds_env/bin/jupyter lab --ip=0.0.0.0 --port=8888 --no-browser
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable jupyter
sudo systemctl start jupyter
```

**Access JupyterLab at:** `http://YOUR_TAILSCALE_IP:8888`

### 13. Install VS Code Server

```bash
# Install code-server
curl -fsSL https://code-server.dev/install.sh | sh

# Set your password (CHANGE THIS to your own secure password!)
PASSWORD="ChangeThisToYourSecurePassword"

# Create configuration
mkdir -p ~/.config/code-server
cat > ~/.config/code-server/config.yaml << EOF
bind-addr: 0.0.0.0:8080
auth: password
password: $PASSWORD
cert: false
user-data-dir: ~/.local/share/code-server
EOF

# Enable and start service
sudo systemctl enable --now code-server@$USER

# Install Python extensions
code-server --install-extension ms-python.python
code-server --install-extension ms-toolsai.jupyter

# Restart service
sudo systemctl restart code-server@$USER

echo "✅ VS Code Server installed!"
echo "Access: http://YOUR_TAILSCALE_IP:8080"
echo "Password: $PASSWORD"
```

**Access VS Code at:** `http://YOUR_TAILSCALE_IP:8080`

### 14. Configure VS Code for Python Development

```bash
# Create VS Code workspace settings
cd /srv/share/data_science
mkdir -p .vscode

# Create settings.json
cat > .vscode/settings.json << 'EOF'
{
    "python.defaultInterpreterPath": "${workspaceFolder}/ds_env/bin/python",
    "python.terminal.activateEnvironment": true,
    "python.analysis.extraPaths": ["${workspaceFolder}"],
    "python.autoComplete.extraPaths": ["${workspaceFolder}"],
    "jupyter.notebookFileRoot": "${workspaceFolder}",
    "files.watcherExclude": {
        "**/ds_env/**": true
    }
}
EOF

# Create launch.json for debugging
cat > .vscode/launch.json << 'EOF'
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Current File",
            "type": "debugpy",
            "request": "launch",
            "program": "${file}",
            "console": "integratedTerminal",
            "justMyCode": true,
            "python": "${workspaceFolder}/ds_env/bin/python"
        }
    ]
}
EOF
```

**In VS Code:**
1. Open Folder: `/srv/share/data_science`
2. Press `Ctrl+Shift+P` → "Python: Select Interpreter"
3. Choose: `/srv/share/data_science/ds_env/bin/python`

### 15. Enable Automatic Security Updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 16. Install fail2ban for Intrusion Prevention

Install and configure fail2ban to automatically ban IPs with suspicious activity:

```bash
# Install fail2ban
sudo apt install -y fail2ban

# Create local configuration
sudo tee /etc/fail2ban/jail.local > /dev/null <<'EOF'
[DEFAULT]
# Ban hosts for 1 hour
bantime = 3600
# Check for failures in last 10 minutes
findtime = 600
# Ban after 5 failed attempts
maxretry = 5

# Enable protection for SSH (Tailscale SSH)
[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s

# Protect against aggressive clients
[recidive]
enabled = true
filter = recidive
logpath = /var/log/fail2ban.log
action = %(action_mwl)s
bantime = 86400
findtime = 86400
maxretry = 3
EOF

# Enable and start fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Check status
sudo fail2ban-client status
```

**Monitor fail2ban activity:**

```bash
# View banned IPs
sudo fail2ban-client status sshd

# View fail2ban log
sudo tail -f /var/log/fail2ban.log

# Manually unban an IP if needed
sudo fail2ban-client set sshd unbanip IP_ADDRESS
```

---

## Verification

### Check All Services

Run this command to verify all services are running:

```bash
echo "=== System Status ==="
echo ""
echo "🔒 Firewall:   $(systemctl is-active nftables)"
echo "🌐 Tailscale:  $(systemctl is-active tailscaled)"
echo "🖥️  Cockpit:    $(systemctl is-active cockpit.socket)"
echo "📁 Samba:      $(systemctl is-active smbd)"
echo "📓 Jupyter:    $(systemctl is-active jupyter)"
echo "💻 VS Code:    $(systemctl is-active code-server@$USER)"
echo "💾 Backups:    $(systemctl is-active hourly-backup.timer)"
echo "🛡️  fail2ban:   $(systemctl is-active fail2ban)"
echo ""
echo "=== Access URLs ==="
echo "VS Code:    http://$(tailscale ip -4):8080"
echo "JupyterLab: http://$(tailscale ip -4):8888"
echo "Cockpit:    http://$(tailscale ip -4):9090"
echo "Samba:      \\\\$(tailscale ip -4)\\share"
```

### Monitor Firewall Activity

```bash
# View blocked connection attempts in real-time
sudo journalctl -kf | grep "nft-drop"
```

### Check Backup Status

```bash
# View backup log
sudo tail -f /var/log/hourly-backup.log

# List all backups
sudo timeshift --list
```

---

## Usage

### Accessing Your Server

**Via VS Code (Code Editing):**
```
http://YOUR_TAILSCALE_IP:8080
```

**Via JupyterLab (Data Science Notebooks):**
```
http://YOUR_TAILSCALE_IP:8888
```

**Via Cockpit (Server Management):**
```
http://YOUR_TAILSCALE_IP:9090
```

**Via Samba (File Sharing):**
- Windows: `\\YOUR_TAILSCALE_IP\share`
- Mac: `smb://YOUR_TAILSCALE_IP/share`
- Linux: `smb://YOUR_TAILSCALE_IP/share`

**Via SSH:**
```bash
ssh YOUR_USERNAME@YOUR_TAILSCALE_IP
# or use hostname if configured
ssh YOUR_USERNAME@YOUR_HOSTNAME
```

### Working with Python

```bash
# Navigate to data science folder
cd /srv/share/data_science

# Activate virtual environment
source ds_env/bin/activate

# Install new packages
pip install package_name

# Run Python scripts
python your_script.py

# Update requirements.txt
pip freeze > requirements.txt

# Deactivate virtual environment
deactivate
```

---

## Maintenance

### Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### Restart Services

```bash
# Restart individual services
sudo systemctl restart nftables
sudo systemctl restart smbd
sudo systemctl restart jupyter
sudo systemctl restart code-server@$USER
sudo systemctl restart cockpit.socket
```

### View Service Logs

```bash
# Firewall activity
sudo journalctl -kf | grep "nft-drop"

# Jupyter logs
sudo journalctl -u jupyter -f

# VS Code logs
sudo journalctl -u code-server@$USER -f

# Backup logs
sudo tail -f /var/log/hourly-backup.log
```

### Backup Management

```bash
# List all backups
sudo timeshift --list

# Create manual backup
sudo timeshift --create --comments "Manual backup before update"

# Restore from backup (interactive)
sudo timeshift --restore

# Check next scheduled backup
systemctl list-timers | grep hourly

# View backup timer status
systemctl status hourly-backup.timer
```

---

## Troubleshooting

### Can't Access Services

```bash
# Check Tailscale connection
tailscale status

# Verify Tailscale IP
tailscale ip -4

# Check firewall status
sudo systemctl status nftables
sudo nft list ruleset

# Check specific service
sudo systemctl status [service_name]
```

### Python Environment Issues

```bash
# Verify virtual environment
cd /srv/share/data_science
source ds_env/bin/activate
python --version
pip list

# Reinstall problematic package
pip install --force-reinstall package_name

# Recreate virtual environment if needed
deactivate
rm -rf ds_env
python3 -m venv ds_env
source ds_env/bin/activate
pip install -r requirements.txt
```

### VS Code Python Interpreter Not Found

```bash
# Verify interpreter path
ls -la /srv/share/data_science/ds_env/bin/python

# Recreate VS Code workspace settings
cd /srv/share/data_science
rm -rf .vscode
# Then recreate settings from step 14
```

### JupyterLab Won't Start

```bash
# Check service status
sudo systemctl status jupyter

# View detailed logs
sudo journalctl -u jupyter -n 50

# Restart service
sudo systemctl restart jupyter

# Reset Jupyter password if needed
cd /srv/share/data_science
source ds_env/bin/activate
jupyter lab password
deactivate
sudo systemctl restart jupyter
```

### Firewall Blocking Legitimate Traffic

```bash
# Temporarily disable firewall for testing
sudo systemctl stop nftables

# Check if issue persists, then re-enable
sudo systemctl start nftables

# View firewall rules
sudo nft list ruleset

# Check what's being blocked
sudo journalctl -kf | grep "nft-drop"
```

### fail2ban Issues

```bash
# Check fail2ban status
sudo systemctl status fail2ban

# View banned IPs
sudo fail2ban-client status sshd

# Unban a specific IP
sudo fail2ban-client set sshd unbanip IP_ADDRESS

# Restart fail2ban
sudo systemctl restart fail2ban

# View fail2ban logs
sudo tail -f /var/log/fail2ban.log
```

---

## Port Reference

| Service | Port | Protocol | Access |
|---------|------|----------|--------|
| Tailscale SSH | 22 | TCP | Tailscale only |
| VS Code | 8080 | TCP | Tailscale only |
| JupyterLab | 8888 | TCP | Tailscale only |
| Cockpit | 9090 | TCP | Tailscale only |
| Samba | 445, 139 | TCP | Tailscale only |

> **Note:** All ports are blocked from the public internet and only accessible through your Tailscale network.

---

## Security Summary

**🔒 Your Server is Protected By:**

- ✅ Tailscale zero-trust network (WireGuard encryption)
- ✅ nftables firewall (blocks all non-Tailscale traffic)
- ✅ fail2ban intrusion prevention (automatic IP banning)
- ✅ Traditional SSH disabled (Tailscale SSH only)
- ✅ Hourly automated backups (15-day retention)
- ✅ Automatic security updates
- ✅ All services accessible only via Tailscale

**🚫 What's Blocked:**

- All public internet traffic (except outbound)
- Port scanning attempts
- SSH brute-force attacks
- Unauthorized access attempts

**✅ What's Allowed:**

- Access from your Tailscale network devices only
- Established connections (responses to outbound requests)
- Localhost connections

---

## Installed Software Summary

| Category | Software | Purpose |
|----------|----------|---------|
| **Security** | Tailscale | Zero-trust VPN |
| | nftables | Modern firewall |
| | fail2ban | Intrusion prevention system |
| **Management** | Cockpit | Web-based server UI |
| | Cockpit Navigator | Advanced file manager |
| | Cockpit Files | Lightweight file browser |
| **Development** | VS Code Server | Browser-based IDE |
| | JupyterLab | Interactive notebooks |
| | Python 3 | Programming language |
| **File Sharing** | Samba | Cross-platform file sharing |
| **Backups** | Timeshift | System snapshots |
| | Custom script | Automated backup management |
| **Data Science** | NumPy, Pandas | Data manipulation |
| | Matplotlib, Seaborn | Data visualization |
| | scikit-learn | Machine learning |
| | PyTorch | Deep learning framework |

---

## Complete! 🎉

Your Ubuntu server is now configured with:

- 🔒 Military-grade security (Tailscale + firewall + fail2ban)
- 💻 Professional development environment (VS Code + JupyterLab)
- 📊 Complete data science stack (NumPy, Pandas, PyTorch)
- 💾 Automated hourly backups with 15-day retention
- 🖥️ Easy web-based server management (Cockpit)
- 📁 Cross-platform file sharing (Samba)

All services are accessible securely via Tailscale from anywhere in the world!

---

## 🔐 Additional Security Recommendations

For enhanced security, consider implementing these additional measures:

### 1. Enable 2FA on Tailscale Account (CRITICAL!)

Your Tailscale account is the master key to your entire server. Enable two-factor authentication:
- Go to https://login.tailscale.com/admin/settings/account
- Enable two-factor authentication
- Save backup codes in a secure location

### 2. Use Strong Passwords

- VS Code: 20+ characters
- JupyterLab: 20+ characters
- Samba: 20+ characters
- Use a password manager (Bitwarden, 1Password, KeePassXC)

### 3. Regular Security Audits

```bash
# Install and run Lynis security audit
sudo apt install -y lynis
sudo lynis audit system
```

### 4. Monitor Logs Regularly

```bash
# Set up log monitoring script
cat > ~/check-security.sh << 'EOF'
#!/bin/bash
echo "=== Recent fail2ban Activity ==="
sudo fail2ban-client status sshd
echo ""
echo "=== Recent Firewall Drops ==="
sudo journalctl -k --since "1 hour ago" | grep "nft-drop" | tail -20
echo ""
echo "=== Recent SSH Logins ==="
sudo journalctl -u sshd --since "24 hours ago" | grep "Accepted"
EOF
chmod +x ~/check-security.sh
```

### 5. Limit Sudo Access (Multi-user Systems)

```bash
# Create sudoers file for limited access
sudo visudo -f /etc/sudoers.d/limited_users
# Add: username ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart SERVICE_NAME
```

### 6. File Encryption for Sensitive Data

```bash
# For sensitive files, use gpg encryption
gpg --symmetric --cipher-algo AES256 sensitive_file.txt
```

### 7. Regular Backup Testing

Test your backups monthly to ensure they work:

```bash
# List available backups
sudo timeshift --list

# Test restore in a VM or test environment
# DO NOT test on production!
```

### 8. Network Segmentation (Advanced)

Use Tailscale ACLs to restrict access between devices:
- Go to https://login.tailscale.com/admin/acls
- Define which devices can access which services
- Example: Only allow your laptop to access Jupyter, not your phone

### 9. Update Schedule

```bash
# Check for updates weekly
sudo apt update && sudo apt list --upgradable

# Schedule automatic reboots for kernel updates
sudo systemctl edit apt-daily-upgrade.service
```

### 10. Incident Response Plan

Document what to do if compromised:
1. Disconnect from Tailscale: `sudo tailscale down`
2. Check active connections: `sudo netstat -tulpn`
3. Review recent commands: `history`
4. Check for unauthorized users: `cat /etc/passwd`
5. Restore from backup if needed

### 11. SMART Monitoring for NVMe Drives

Check what disks you have:

```bash
lsblk -d -o NAME,MODEL,SIZE,TYPE
```

Create the SMART monitoring script:

```bash
sudo tee /usr/local/bin/smart << 'EOF'
#!/usr/bin/env bash
set -u
mapfile -t DISKS < <(lsblk -dn -o NAME,TYPE | awk '$2=="disk"{print "/dev/"$1}')
if [ ${#DISKS[@]} -eq 0 ]; then
  echo "No disks found."
  exit 1
fi
for DISK in "${DISKS[@]}"; do
  SMART_OUT="$(sudo smartctl -a "$DISK" 2>/dev/null)"

  MODEL="$(echo "$SMART_OUT" | awk -F: '/^Model Number:/ {gsub(/^ +| +$/,"",$2); print $2; exit}')"
  [ -z "$MODEL" ] && MODEL="$(echo "$SMART_OUT" | awk -F: '/^Device Model:/ {gsub(/^ +| +$/,"",$2); print $2; exit}')"
  [ -z "$MODEL" ] && MODEL="$(lsblk -dn -o MODEL "$DISK" 2>/dev/null | sed 's/^ *//;s/ *$//')"
  [ -z "$MODEL" ] && MODEL="Unknown"

  SIZE="$(lsblk -dn -o SIZE "$DISK" 2>/dev/null | sed 's/^ *//;s/ *$//')"
  [ -z "$SIZE" ] && SIZE="N/A"

  TEMP=""
  HEALTH="Unknown"
  STATUS="yellow"

  if [[ "$DISK" == /dev/nvme* ]]; then
    TEMP="$(sudo nvme smart-log "$DISK" 2>/dev/null | awk -F: '/^temperature/ {gsub(/^ +| +$/,"",$2); match($2,/[0-9]+/); print substr($2,RSTART,RLENGTH); exit}')"
    [ -z "$TEMP" ] && TEMP="$(echo "$SMART_OUT" | awk -F: '/^Temperature:/ {gsub(/^ +| +$/,"",$2); match($2,/[0-9]+/); print substr($2,RSTART,RLENGTH); exit}')"
    if echo "$SMART_OUT" | grep -qiE 'PASSED|OK'; then
      HEALTH="Healthy"; STATUS="green"
    else
      HEALTH="Warning"; STATUS="yellow"
    fi
  else
    TEMP="$(echo "$SMART_OUT" | awk -F: '/^Temperature:/ {gsub(/^ +| +$/,"",$2); match($2,/[0-9]+/); print substr($2,RSTART,RLENGTH); exit}')"
    [ -z "$TEMP" ] && TEMP="$(echo "$SMART_OUT" | awk '
      /Temperature_Celsius/ {print $(NF-1); found=1; exit}
      /Airflow_Temperature_Cel/ {print $(NF-1); found=1; exit}
      END {if (!found) print ""}
    ')"
    if echo "$SMART_OUT" | grep -qiE 'PASSED|OK'; then
      HEALTH="Healthy"; STATUS="green"
    else
      HEALTH="Warning"; STATUS="yellow"
    fi
  fi

  TEMP_NUM="$(printf '%s' "$TEMP" | grep -oE '[0-9]+' | head -1)"
  if [ -n "$TEMP_NUM" ]; then
    TEMP="${TEMP_NUM} °C"
    if [ "$TEMP_NUM" -ge 70 ]; then
      STATUS="red"; HEALTH="Hot"
    elif [ "$TEMP_NUM" -ge 60 ] && [ "$STATUS" = "green" ]; then
      STATUS="yellow"
    fi
  fi
  [ -z "$TEMP_NUM" ] && TEMP="N/A"

  POWER_HOURS="$(echo "$SMART_OUT" | awk -F: '/^Power On Hours:/ {gsub(/^ +| +$|,/,"",$2); print $2; exit}')"
  [ -z "$POWER_HOURS" ] && POWER_HOURS="N/A"

  POWER_CYCLES="$(echo "$SMART_OUT" | awk -F: '/^Power Cycles:/ {gsub(/^ +| +$|,/,"",$2); print $2; exit}')"
  [ -z "$POWER_CYCLES" ] && POWER_CYCLES="N/A"

  PCT_USED="$(echo "$SMART_OUT" | awk -F: '/^Percentage Used:/ {gsub(/^ +| +$/,"",$2); print $2; exit}')"
  [ -z "$PCT_USED" ] && PCT_USED="N/A"

  SPARE="$(echo "$SMART_OUT" | awk -F: '/^Available Spare:/ {gsub(/^ +| +$/,"",$2); print $2; exit}')"
  [ -z "$SPARE" ] && SPARE="N/A"

  UNSAFE="$(echo "$SMART_OUT" | awk -F: '/^Unsafe Shutdowns:/ {gsub(/^ +| +$|,/,"",$2); print $2; exit}')"
  [ -z "$UNSAFE" ] && UNSAFE="N/A"

  case "$STATUS" in
    green)  DOT="[green]"  ;;
    yellow) DOT="[yellow]" ;;
    red)    DOT="[red]"    ;;
    *)      DOT="[?]"      ;;
  esac

  echo "$DOT  $DISK  $MODEL  ($SIZE)"
  echo "  Temp:             $TEMP"
  echo "  Health:           $HEALTH"
  echo "  Power On Hours:   $POWER_HOURS hrs"
  echo "  Power Cycles:     $POWER_CYCLES"
  echo "  Wear Level:       $PCT_USED"
  echo "  Available Spare:  $SPARE"
  echo "  Unsafe Shutdowns: $UNSAFE"
  echo
done
EOF
sudo chmod +x /usr/local/bin/smart
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/smart
```

Then just type `smart` to run it.

---

## Contributing

Found an issue or have suggestions? Please open an issue or submit a pull request on GitHub.

## Disclaimer

This guide is provided as-is without warranty. Always test in a non-production environment first. The author is not responsible for any damage or data loss resulting from following this guide.

## License

MIT License

Copyright (c) 2026

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
