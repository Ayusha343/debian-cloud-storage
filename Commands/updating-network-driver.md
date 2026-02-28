# Network Driver and Connectivity Setup

## Overview

This guide covers diagnosing and resolving the **Realtek Ethernet NIC instability** and setting up a **TP-Link RTL8188EUS USB WiFi adapter** with a maintained DKMS driver for reliable cloud server networking.

---

## Part 1: Diagnosing Wired Ethernet Instability

### Problem Description

The integrated Realtek r8169/RTL8105e NIC exhibits:
- Autonegotiation issues (flaps between 100 Mb/s and 10 Mb/s)
- Continuous "Link is Down/Up" messages in kernel logs
- Eventually falls back to unstable 10 Mb/s connection
- Cable works fine on other devices (tested on laptop at 1 Gb/s)
- Router port swap does not help

### Root Cause

**PHY-level compatibility issue** between the NIC chipset and the router, causing negotiation to fail at higher speeds.

### Diagnostic Commands

Check current NIC status:

```bash
# List network interfaces
ip link show

# Check interface details (replace enp2s0 with your interface)
sudo ethtool enp2s0

# View full interface settings
sudo ethtool -s enp2s0 autoneg on

# Monitor link status in real-time
sudo dmesg -w | grep -i -e "link" -e "r8169" -e "enp2s0"

# Extract link history from kernel logs
sudo dmesg | grep -i -e "r8169" -e "link" -e "enp2s0" | tail -50
```

### Mitigation (Short-term)

**Keep autonegotiation ON**, even if speed is limited to 10 Mb/s, for stability:

```bash
# Enable autonegotiation with standard advertisement
sudo ethtool -s enp2s0 autoneg on

# Persist by adding to /etc/udev/rules.d/99-network.rules
echo 'ACTION=="add", SUBSYSTEM=="net", NAME=="enp2s0", RUN+="/sbin/ethtool -s %k autoneg on"' \
  | sudo tee /etc/udev/rules.d/99-network.rules
```

### Long-term Solution

**Replace with USB WiFi adapter** (see Part 2) or add a dedicated PCIe NIC:
- Small unmanaged switch between PC and router
- USB 3.0 Ethernet adapter (Realtek RTL8153, ASIX AX88179)
- PCIe 1 GbE NIC (Intel PRO/1000, Realtek RTL8111/8168)

---

## Part 2: TP-Link TL-WN725N WiFi Adapter Setup (RTL8188EUS)

### Hardware Identification

First, verify the adapter is detected:

```bash
# List USB devices
lsusb

# Expected output (look for this line):
# Bus 001 Device XXX: ID 0bda:8179 Realtek Semiconductor Corp. RTL8188EUS 802.11n Wireless Network Adapter

# Check kernel version
uname -r

# Expected: 6.1.x or later for Debian Bookworm
```

### Prerequisites

```bash
# Install build tools and firmware
sudo apt update
sudo apt install -y \
  git \
  build-essential \
  dkms \
  linux-headers-$(uname -r) \
  firmware-realtek \
  firmware-misc-nonfree \
  network-manager \
  wpasupplicant \
  wireless-tools
```

**Enable non-free firmware in APT sources:**

```bash
# Edit APT sources
sudo nano /etc/apt/sources.list

# Add "non-free-firmware" to the main Debian lines:
# deb http://deb.debian.org/debian/ bookworm main contrib non-free-firmware
# deb http://security.debian.org/debian-security bookworm-security main contrib non-free-firmware

# Save (Ctrl+O, Enter, Ctrl+X)

# Update
sudo apt update
```

---

## Part 2A: Confirm USB 2.0 Speed (480M)

**Critical Step**: The adapter must negotiate at USB 2.0 (480M), not USB 1.1 (12M).

```bash
# Check negotiated USB speed
sudo lsusb -t

# Expected output (look for the Realtek line):
# Port 4: Dev 3, If 0, Class=Vendor Specific, Driver=8188eu, 480M
#                                                            ^^^^^ This should be 480M

# If you see 12M, the adapter is at USB 1.1 speed
```

### Troubleshooting USB Speed

If USB speed is 12M instead of 480M:

1. **Move to rear I/O USB port** (rear ports are often USB 2.0 only)
2. **Use a short USB extension cable** (3–6 inches):
   - Reduces RF interference
   - Positions adapter away from PC case
   - Improves signal
3. **Avoid USB hubs** (use direct connection)
4. **Try different USB port** and reboot

After moving port/cable, verify again:

```bash
# Unplug and replug adapter
sudo lsusb -t

# Should now show 480M
```

---

## Part 2B: Install DKMS Driver (rtl8188eus from aircrack-ng)

**Why DKMS?** The kernel's built-in `r8188eu` and `rtl8xxxu` drivers often underperform with RTL8188EUS. The community-maintained DKMS driver from aircrack-ng provides better throughput and stability. DKMS automatically rebuilds the driver when the kernel is updated.

### Step 1: Remove Conflicting Modules

```bash
# Unload in-kernel modules if loaded
sudo modprobe -r r8188eu 2>/dev/null || true
sudo modprobe -r rtl8xxxu 2>/dev/null || true

# Blacklist them to prevent auto-load
echo "blacklist r8188eu" | sudo tee /etc/modprobe.d/blacklist-r8188eu.conf
echo "blacklist rtl8xxxu" | sudo tee /etc/modprobe.d/blacklist-rtl8xxxu.conf
```

### Step 2: Clone and Install DKMS Driver

```bash
# Clone repository
cd ~
git clone https://github.com/aircrack-ng/rtl8188eus.git
cd rtl8188eus

# View DKMS config to find version
cat dkms.conf | grep PACKAGE_VERSION

# Get the version string (e.g., "5.4.1")
VERSION=$(awk -F'"' '/PACKAGE_VERSION/ {print $2}' dkms.conf)

# Install via DKMS
sudo dkms add ./
sudo dkms build rtl8188eus/$VERSION
sudo dkms install rtl8188eus/$VERSION

# Load the new module
sudo modprobe 8188eu

# Verify it loaded (no output = success)
lsmod | grep 8188eu
```

### Step 3: Remove Windows Line Endings (if needed)

If you copied commands on Windows, remove carriage returns:

```bash
# Remove \r characters that break scripts
sudo sed -i 's/\r$//' /usr/local/bin/any-script.sh
```

---

## Part 2C: Verify and Tune WiFi Driver

### Confirm Driver in Use

```bash
# Find WiFi interface name
IFACE=$(iw dev | awk '/Interface/ {print $2; exit}')
echo "WiFi Interface: $IFACE"

# Confirm driver
sudo ethtool -i "$IFACE"

# Expected: driver: 8188eu (not rtl8xxxu or r8188eu)
```

### Disable WiFi Power Saving

Power saving causes disconnects and instability on a server. Disable it:

```bash
# Immediate disable
IFACE=$(iw dev | awk '/Interface/ {print $2; exit}')
sudo iw dev "$IFACE" set power_save off

# Verify disabled
sudo iw dev "$IFACE" get power_save

# Expected: Power save: off
```

### Persist Power Save Setting Across Reboots

```bash
# Create NetworkManager config
sudo mkdir -p /etc/NetworkManager/conf.d

# Write config
printf "[connection]\nwifi.powersave = 2\n" | \
  sudo tee /etc/NetworkManager/conf.d/wifi-powersave.conf

# Restart NetworkManager
sudo systemctl restart NetworkManager
```

### Auto-load Module at Boot

```bash
# Add to modules-load.d
echo "8188eu" | sudo tee /etc/modules-load.d/8188eu.conf

# Verify on next boot:
# lsmod | grep 8188eu
```

---

## Part 3: Connect to WiFi

### Manual Connection

```bash
# Enable WiFi
nmcli radio wifi on

# Scan networks
nmcli dev wifi list

# Connect (replace SSID and PASSWORD)
nmcli dev wifi connect "Your-SSID" password "Your-Password"

# Verify connection
nmcli device show wlan0 | grep -i "ip\|dns\|connection"
```

### Permanent Connection (Auto-Reconnect on Boot)

```bash
# Store connection permanently
nmcli connection modify "Your-SSID" connection.autoconnect yes

# Verify
nmcli connection show "Your-SSID"
```

---

## Part 4: Performance Tuning

### Router-Side Settings (2.4 GHz)

For best performance with the N150 adapter:

1. **Fix WiFi channel** to 1, 6, or 11 (non-overlapping)
   - Don't use "Auto" channel
   - Use WiFi analyzer to find least congested channel

2. **Set channel width to 20 MHz** (not 40 MHz)
   - More stable than wide channels on 2.4 GHz
   - Better range and compatibility

3. **Reduce transmit power** (if available)
   - Prevents over-power interference
   - Typically `HIGH` or `MEDIUM` is fine

### Expected Performance

With DKMS driver and proper tuning:

```
Real-world throughput: 25–60 Mb/s (N150 typical)
```

This is a significant improvement over the 10 Mb/s wired baseline. Test:

```bash
# Simple speed test (requires internet)
sudo apt install -y speedtest-cli

# Run test
speedtest-cli --simple

# Expected: 25–60 Mb/s download (varies with network/ISP)
```

---

## Part 5: Persistence Across Reboots and Kernel Updates

### Verify DKMS Installation

```bash
# Check installed DKMS modules
dkms status

# Expected output:
# rtl8188eus/X.X.X: installed for 6.1.0-xx-amd64
```

### Rebuild After Kernel Update

When Debian updates the kernel, DKMS automatically rebuilds the driver. Verify:

```bash
# After any `sudo apt full-upgrade`, check:
dkms status

# If not installed for new kernel:
cd ~/rtl8188eus
VERSION=$(awk -F'"' '/PACKAGE_VERSION/ {print $2}' dkms.conf)
sudo dkms install rtl8188eus/$VERSION

# Reload module
sudo modprobe 8188eu
```

### Post-Reboot Verification

After system reboot, verify everything persists:

```bash
# Driver loaded?
lsmod | grep 8188eu

# Power save still off?
IFACE=$(iw dev | awk '/Interface/ {print $2; exit}')
sudo iw dev "$IFACE" get power_save

# USB at 480M?
sudo lsusb -t | grep Realtek

# Connected to WiFi?
nmcli connection show --active
```

---

## Part 6: Rollback (if ever needed)

If you need to revert to in-kernel driver:

```bash
# Stop WiFi and unload module
nmcli radio wifi off
sudo modprobe -r 8188eu

# Remove DKMS installation
VERSION=$(awk -F'"' '/PACKAGE_VERSION/ {print $2}' ~/rtl8188eus/dkms.conf)
sudo dkms remove rtl8188eus/$VERSION --all

# Remove blacklist
sudo rm -f /etc/modprobe.d/blacklist-r8188eu.conf
sudo rm -f /etc/modprobe.d/blacklist-rtl8xxxu.conf

# Remove module auto-load
sudo rm -f /etc/modules-load.d/8188eu.conf

# Re-enable in-kernel driver
sudo modprobe r8188eu || sudo modprobe rtl8xxxu

# Enable WiFi again
nmcli radio wifi on
```

---

## Troubleshooting

### No WiFi networks visible
```bash
nmcli radio wifi off
sleep 2
nmcli radio wifi on
nmcli dev wifi rescan
```

### Frequent disconnects
- Disable power saving (already done above)
- Check USB speed is 480M (not 12M)
- Try different WiFi channel on router
- Check RF interference (microwave, cordless phone, neighboring networks)

### Very slow speeds
- Router speed test from laptop (verify ISP)
- Check WiFi signal strength: `nmcli device wifi list`
- Change to less congested WiFi channel
- Move adapter away from PC using USB extension

### Connection works but then stops
- WiFi power save re-enabled: `sudo iw dev wlan0 set power_save off`
- Check NetworkManager: `sudo systemctl restart NetworkManager`
- Relevant logs: `sudo journalctl -u NetworkManager -f`

---

## Related Configuration

After network is stable, proceed with:

1. [Installing SSH](../Docs/Server-guide.md#installing-ssh)
2. [Setting up storage](../Docs/Server-guide.md#setting-up-drives-and-pools)
3. [Installing Nextcloud](../Docs/Server-guide.md#installing-nextcloud)
