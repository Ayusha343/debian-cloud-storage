# Global Internet Access via Cloudflare Tunnel

## Overview

Enable permanent **global HTTPS access** to Nextcloud using **Cloudflare Tunnel** without port forwarding, port mapping, or dynamic DNS management. The server maintains an outbound connection to Cloudflare, routing all incoming traffic through a stable persistent tunnel.

---

## Part 1: Domain Setup

### Problem Statement

- **Dynamic IP**: ISP provides changing IP addresses
- **No Port Forwarding**: ISP (CGNAT) blocks port forwarding
- **Persistence Required**: URL must remain stable across reboots
- **Solution**: Named Cloudflare Tunnel bound to a domain

### Register Domain

Multiple options for cheap domains compatible with Cloudflare:

| Domain | Registrar | Cost | Notes |
|--------|-----------|------|-------|
| storagepaglu.pp.ua | Regery.com | ₹120/year | .pp.ua (free second-level, Ukraine) |
| yourdomain.com | Namecheap | ₹400/year | Standard .com |
| yourdomain.tk | Freenom | Free | Free .tk/.ml/.ga (requires renewal) |

**Chosen:** `storagepaglu.pp.ua` from Regery.com for affordability.

### Point Nameservers to Cloudflare

1. Register at [Cloudflare.com](https://www.cloudflare.com) (free account)
2. Add domain in Cloudflare dashboard
3. On Cloudflare, note the **two nameservers** provided (e.g., `ns1.cloudflare.com`)
4. Update your domain registrar's nameservers to point to Cloudflare
5. Wait 24–48 hours for propagation (check: `nslookup storagepaglu.pp.ua`)

---

## Part 2: Install and Authenticate Cloudflared

### Add Cloudflared Repository

```bash
# Add Cloudflare GPG key
curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | \
  sudo tee /usr/share/keyrings/cloudflare-main.gpg > /dev/null

# Add repository
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] \
https://pkg.cloudflare.com/cloudflared any main' | \
  sudo tee /etc/apt/sources.list.d/cloudflared.list

# Update and install
sudo apt update
sudo apt install -y cloudflared
```

### Authenticate Cloudflared

```bash
# Login to Cloudflare account
cloudflared tunnel login

# Browser opens; log in with your Cloudflare account
# Cloudflare asks permission; approve to download certificate
# Certificate saved to: ~/.cloudflared/<UUID>.json
```

---

## Part 3: Create and Configure Named Tunnel

### Create Named Tunnel

```bash
# Create a persistent named tunnel (not ephemeral)
cloudflared tunnel create homeserver-tunnel

# Output shows:
# Tunnel UUID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# Credentials file: ...credentials.json
# DNS records: to route domain to tunnel
```

### Route Domain to Tunnel

```bash
# Bind domain to this tunnel
cloudflared tunnel route dns homeserver-tunnel storagepaglu.pp.ua

# Expected output: Successfully created DNS record
```

### Create Configuration File

```bash
# Create config directory (if not exists)
mkdir -p ~/.cloudflared

# Create config file
nano ~/.cloudflared/config.yml
```

Paste (replace UUID with your actual tunnel UUID):

```yaml
tunnel: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
credentials-file: /home/homeserver/.cloudflared/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.json

ingress:
  - hostname: storagepaglu.pp.ua
    service: http://localhost:80
  - service: http_status:404
```

Save (Ctrl+O, Enter, Ctrl+X).

### Set Correct Permissions

```bash
chmod 600 ~/.cloudflared/config.yml
chmod 600 ~/.cloudflared/*.json
```

### Test Tunnel Manually

```bash
# Run tunnel in foreground (for testing)
cloudflared tunnel run homeserver-tunnel

# You should see:
# Connection established successfully
# Serving tunnel at: https://storagepaglu.pp.ua

# Test in browser:
# https://storagepaglu.pp.ua/nextcloud

# If working, press Ctrl+C to stop
```

---

## Part 4: Install as System Service

### Register and Install Service

```bash
# Install cloudflared as system service
sudo cloudflared service install

# Expected: Service already installed or successfully installed
```

### Start and Enable Service

```bash
# Enable on boot
sudo systemctl enable cloudflared

# Start service
sudo systemctl start cloudflared

# Verify running
sudo systemctl status cloudflared

# Check logs
sudo journalctl -u cloudflared -f

# Expected: no errors, connection established
```

---

## Part 5: Verify Global Access

### Test Public URL

From any device/network:

```
https://storagepaglu.pp.ua/nextcloud
```

Should show Nextcloud login page.

### Verify HTTPS and Certificates

```bash
# Check certificate (padlock in browser)
# Should show verified Cloudflare certificate (no warning)

# SSL Labs test (optional)
# https://www.ssllabs.com/ssltest/analyze.html?d=storagepaglu.pp.ua
```

---

## Part 6: Architecture and How It Works

### Connection Flow

```
┌──────────────────────────────────────────────────────────┐
│                    INTERNET USER                         │
│              Anywhere on Earth                           │
│                                                          │
│  Visits: https://storagepaglu.pp.ua/nextcloud           │
└────────────────────────┬─────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│              CLOUDFLARE EDGE NETWORK                     │
│                                                          │
│  - Global presence (~200+ data centers)                 │
│  - DDoS protection                                      │
│  - SSL/TLS termination (HTTPS)                          │
│  - Caches DNS, routes traffic                           │
└────────────────────────┬─────────────────────────────────┘
                         ↓
                  [Persistent Tunnel]
              (Outbound WebSocket connection)
                         ↓
┌──────────────────────────────────────────────────────────┐
│                  HOME SERVER                            │
│              192.168.1.12 (Private)                     │
│                                                          │
│  ┌────────────────────────────────────────────┐         │
│  │  cloudflared daemon                        │         │
│  │  (maintains tunnel connection)             │         │
│  └────────────────────────────────────────────┘         │
│                 ↓                                        │
│  ┌────────────────────────────────────────────┐         │
│  │  Apache HTTP Server (port 80/443)          │         │
│  │  ├─ Nextcloud                             │         │
│  │  ├─ MariaDB                               │         │
│  │  └─ Redis Cache                           │         │
│  └────────────────────────────────────────────┘         │
│                                                          │
│  Network: TP-Link WiFi Adapter                          │
│  Storage: MergerFS pool (/srv/data)                     │
└──────────────────────────────────────────────────────────┘
```

### Key Advantages

✓ **No inbound ports** opened on router  
✓ **No port forwarding** configuration needed  
✓ **No DDNS** client/updates required  
✓ **Dynamic IP** works automatically  
✓ **Persistent URL** survives reboots  
✓ **HTTPS automatic** via Cloudflare  
✓ **DDoS protection** included  

---

## Part 7: Monitoring and Maintenance

### Check Tunnel Status

```bash
# View service status
sudo systemctl status cloudflared

# View real-time logs
sudo journalctl -u cloudflared -f

# List active tunnels
cloudflared tunnel list

# View tunnel config
cloudflared tunnel info homeserver-tunnel
```

### Monitor Connection

```bash
# Persistent monitoring
watch -n 5 'sudo systemctl status cloudflared --no-pager'

# Verify DNS resolution
nslookup storagepaglu.pp.ua

# Check tunnel dashboard
# Requires: https://dash.cloudflare.com/
```

### Keep Cloudflared Updated

```bash
# Cloudflared is auto-updated via apt
sudo apt update
sudo apt upgradable

# If update available:
sudo apt upgrade -y

# Restart service
sudo systemctl restart cloudflared
```

---

## Part 8: Troubleshooting

### Tunnel Not Connecting

```bash
# Check logs for errors
sudo journalctl -u cloudflared -n 30

# If auth error:
cloudflared tunnel login  # Re-authenticate

# If config error:
nano ~/.cloudflared/config.yml  # Verify syntax
```

### Domain Not Resolving

```bash
# Wait 24–48 hours after nameserver change

# Check propagation
nslookup storagepaglu.pp.ua

# Force DNS refresh
sudo systemctl restart systemd-resolved

# Alternative:
dig storagepaglu.pp.ua @1.1.1.1
```

### HTTPS Certificate Error

```bash
# Cloudflare typically handles this
# If persistent, re-install service:
sudo cloudflared service uninstall
sudo cloudflared service install
sudo systemctl restart cloudflared
```

### Tunnel Keeps Disconnecting

```bash
# Check network stability
ping -c 100 8.8.8.8

# Monitor WiFi signal
nmcli device show wlan0 | grep signal

# If weak, restart WiFi:
nmcli radio wifi off; sleep 2; nmcli radio wifi on

# Check system logs for network issues
dmesg | tail -20
```

---

## Part 9: Scaling and Advanced Configuration

### Multiple Services (Optional)

To tunnel multiple services (e.g., Apache on 80, SSH on 2222):

```yaml
tunnel: your-tunnel-uuid
credentials-file: /path/to/credentials.json

ingress:
  - hostname: storagepaglu.pp.ua
    service: http://localhost:80
  - hostname: ssh.storagepaglu.pp.ua
    service: ssh://localhost:2222
  - service: http_status:404
```

### Custom SSL Certificate (Advanced)

By default, Cloudflare provides free SSL certificate. For custom cert:

```bash
# Generate self-signed cert (example)
openssl req -new -x509 -days 365 -nodes -out cert.pem -keyout key.pem

# Update Apache vhost to use cert
# (See Apache SSL configuration docs)
```

### Rate Limiting and Rules (Optional)

In Cloudflare dashboard:

1. Domain Settings → Firewall Rules
2. Create rule: Limit requests per IP
3. Example: "Block if >100 req/min from same IP"

---

## Part 10: Backup and Disaster Recovery

### Backup Tunnel Configuration

```bash
# Backup credentials and config
cp -r ~/.cloudflared ~/backups/cloudflared-backup-$(date +%Y%m%d)

# Store securely (USB, cloud storage, etc.)
```

### Restore Tunnel After Disk Failure

```bash
# Restore files
cp ~/backups/cloudflared-backup-*/. ~/.cloudflared/

# Re-install service
sudo cloudflared service install

# Start
sudo systemctl start cloudflared
```

---

## Related Guides

- [SSH Remote Access](./SSH-Setup.md) – Command-line tunneling alternative
- [Nextcloud Setup](./Nextcloud.md) – Local access
- [Network Configuration](../Commands/updating-network-driver.md) – Internet connectivity
- [Maintenance](../README.md#maintenance-schedule) – Regular checks
