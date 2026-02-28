# SSH Setup and Remote Access

## Overview

Secure, key-based SSH access with hardened security settings for remote server management from Windows using WSL and VS Code.

---

## Part 1: SSH Server Installation

### Install and Enable

```bash
sudo apt update
sudo apt install -y openssh-server

# Enable and start
sudo systemctl enable --now ssh

# Verify status
systemctl status ssh

# Note the server's LAN IP
hostname -I
# or
ip -4 addr | grep inet
```

---

## Part 2: SSH Key Authentication (Client Setup)

### Generate SSH Keys (on laptop/Windows via WSL)

```bash
# In WSL bash terminal
ssh-keygen -t ed25519 -C "laptop-key"

# Press Enter for default location (~/.ssh/id_ed25519)
# Enter strong passphrase (remember it)

# Verify keys created
ls -la ~/.ssh/id_ed25519*
```

### Copy Public Key to Server

```bash
# From WSL, copy your public key to server
ssh-copy-id -p 22 homeserver@192.168.1.12

# When prompted, enter homeserver's password
# Keys are now authorized
```

### Test Key-Based Login

```bash
# Connect without password (passphrase only)
ssh homeserver@192.168.1.12

# Should connect directly without password prompt
# You may be asked for SSH key passphrase (normal, one-time)

# Exit
exit
```

---

## Part 3: SSH Hardening

### Change Default Port

```bash
# Backup original config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# Edit SSH config
sudo nano /etc/ssh/sshd_config
```

Find and modify these lines:

```ssh
# Line ~15: Change port from 22
Port 2222

# Line ~60: Disable password auth (use keys only)
PasswordAuthentication no

# Line ~33: Permit root login
PermitRootLogin no
```

Save (Ctrl+O, Enter, Ctrl+X).

### Restart SSH and Update Firewall

```bash
# Restart SSH on new port
sudo systemctl restart ssh

# Allow new port in firewall
sudo ufw allow 2222/tcp
sudo ufw reload

# Verify SSH running on new port
sudo ss -tlnp | grep sshd

# Expected: tcp  0  0 0.0.0.0:2222  0.0.0.0:*  LISTEN
```

### Block ICMP Pings (Optional Security)

Makes server invisible to ping sweeps:

```bash
# Add to sysctl
echo "net.ipv4.icmp_echo_ignore_all=1" | sudo tee -a /etc/sysctl.conf

# Apply immediately
sudo sysctl -p

# Verify (should get no response)
ping -c 1 192.168.1.12
```

### Test New Port

```bash
# From WSL, test login on new port
ssh -p 2222 homeserver@192.168.1.12

# Should connect with key, no password

# Exit
exit
```

---

## Part 4: VS Code Remote-SSH Integration

### Install Extension

1. Open VS Code (Windows side, NOT WSL)
2. Press `Ctrl+Shift+X` to open Extensions
3. Search for `Remote - SSH`
4. Install the Microsoft extension

### Configure Remote.SSH Path

1. Press `Ctrl+,` (Settings)
2. Search for `Remote.SSH: Path`
3. Set value to: `C:\Windows\System32\wsl.exe ssh`

This makes VS Code use WSL's SSH binary instead of Windows' (if installed).

### Create SSH Config Entry

Edit `/home/username/.ssh/config` inside WSL:

```bash
# Open in editor (from WSL bash)
nano ~/.ssh/config
```

Paste:

```ssh
Host OurServer
    HostName 192.168.1.12
    Port 2222
    User homeserver
    IdentityFile ~/.ssh/id_ed25519
```

Save (Ctrl+O, Enter, Ctrl+X).

### Set SSH Config Permissions

```bash
# SSH requires strict permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config ~/.ssh/id_ed25519

# Verify
ls -la ~/.ssh/
```

### Connect from VS Code

1. In VS Code, press `F1`
2. Type: `Remote-SSH: Connect to Host...`
3. Select `OurServer` from dropdown (or enter name)
4. VS Code installs remote server component
5. Once connected, you can:
   - Browse remote files in Explorer
   - Edit files directly
   - Open integrated terminal (remote)
   - Run extensions remotely

---

## Part 5: Advanced SSH Configuration

### SSH as Root (Optional, Not Recommended)

If you need direct root SSH access:

```bash
sudo nano /etc/ssh/sshd_config
```

Change:
```
PermitRootLogin yes
```

Then:
```bash
sudo systemctl restart ssh

# Connect as root
ssh -p 2222 root@192.168.1.12
```

**Warning**: Prefer using `sudo` as `homeserver` when possible.

### SSH Key Passphrase Management

Avoid entering passphrase every time:

```bash
# Start SSH agent (in WSL bash)
eval "$(ssh-agent -s)"

# Add key to agent
ssh-add ~/.ssh/id_ed25519

# Enter passphrase once; agent remembers for this session

# Verify agent has key
ssh-add -l
```

For permanent WSL setup, add to `~/.bashrc`:

```bash
# Start SSH agent if not already running
if [ -z "$SSH_AGENT_PID" ]; then
    eval "$(ssh-agent -s)" > /dev/null
    ssh-add ~/.ssh/id_ed25519 2>/dev/null
fi
```

### SSH Timeout and Keep-Alive

To prevent SSH disconnects during idle time, add to `~/.ssh/config`:

```ssh
Host OurServer
    HostName 192.168.1.12
    Port 2222
    User homeserver
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
    ServerAliveCountMax 5
```

---

## Part 6: Common SSH Commands

```bash
# Connect normally
ssh -p 2222 homeserver@192.168.1.12

# Execute command and exit
ssh -p 2222 homeserver@192.168.1.12 "df -h"

# Copy file from server to local
scp -P 2222 homeserver@192.168.1.12:/srv/data/file.txt ./

# Copy file from local to server
scp -P 2222 ./file.txt homeserver@192.168.1.12:/srv/data/

# Mount remote directory locally (via sshfs)
sudo apt install -y sshfs  # Debian install
mkdir ~/server-mount
sshfs -p 2222 homeserver@192.168.1.12:/srv/data ~/server-mount
# Access as /home/username/server-mount

# Unmount when done
fusermount -u ~/server-mount
```

---

## Troubleshooting

### "Connection refused" on port 2222
```bash
# Verify SSH running on new port
sudo ss -tlnp | grep sshd

# Restart if needed
sudo systemctl restart ssh

# Check firewall allows port
sudo ufw status verbose
```

### "Permission denied (publickey)"
```bash
# Keys not authorized; re-copy public key
ssh-copy-id -p 2222 homeserver@192.168.1.12

# Verify authorized_keys on server
cat ~/.ssh/authorized_keys
```

### "Too many authentication failures"
```bash
# SSH agent has too many keys; specify explicitly
ssh -i ~/.ssh/id_ed25519 -p 2222 homeserver@192.168.1.12

# Or clear agent
ssh-add -D
ssh-add ~/.ssh/id_ed25519
```

### VS Code can't connect
```bash
# Verify WSL SSH works first
ssh -p 2222 homeserver@192.168.1.12

# Check extension installed correctly
Command Palette > Remote-SSH: Reload Window

# Check output: View > Output > Remote - SSH
```

---

## Related Guides

- [Network Setup](../Commands/updating-network-driver.md) – WiFi connectivity
- [Cockpit Web UI](./Nextcloud.md#web-management-cockpit) – browser-based management
- [Nextcloud Android SSH](./Nextcloud.md) – remote access from phone
