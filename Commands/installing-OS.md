# Installing Debian 12.6

## Goal
Download and install Debian 12.6 Bookworm on the recycled PC and configure it according to cloud server requirements.

---

## Part 1: Download and Create Bootable USB

### Prerequisites
- Another PC with Windows or Linux
- USB drive with at least 4–5 GB free space
- Internet connection

### Steps

1. **Download Ventoy**
   - Visit: https://sourceforge.net/projects/ventoy/
   - Extract to a temporary folder on your Windows PC

2. **Prepare the USB**
   - Insert USB drive (minimum 4–5 GB)
   - Run `Ventoy2Disk.exe`
   - Select the USB drive and install Ventoy onto it
   - Wait for completion (non-destructive, formats the drive)

3. **Download Debian ISO**
   - Download Debian 12.6 DVD ISO from: https://www.debian.org/download
   - Or from archive: https://archive.org/download/debian-12-bookworm-collection/
   - File: `debian-12.6-amd64-DVD-1.iso` (~3.8 GB)

4. **Copy ISO to USB**
   - After Ventoy installation, mount the USB
   - Copy the Debian ISO file directly to the USB root directory
   - Eject safely

---

## Part 2: Installing Debian on the Server

### BIOS Configuration

1. **Insert USB** into the target server PC
2. **Enter BIOS** by spamming `Delete` key during startup
3. **Set Boot Order**:
   - Navigate to Advanced BIOS Settings
   - Set USB as 1st Bootable Device
   - Save and exit (press `F10`)

### Installation Steps

1. **Boot from USB**
   - Select the Debian 12.6 ISO from Ventoy GUI
   - Continue to Debian installer

2. **Choose Installation Type**
   - Select: **Graphical installation**
   - Press Enter

3. **Select Desktop Environment**
   - Choose: **XFCE** (lightweight, ideal for older hardware)
   - Keep SSH enabled during installation ✓

4. **Install Boot Loader**
   - Select: **GRUB** bootloader to MBR
   - Confirm installation to boot sector

---

## Part 3: Disk Partitioning

During installation, when prompted for disk partitioning, create the following layout on the 128 GB SSD:

| Partition | Size | Format | Mount Point | Flags |
|-----------|------|--------|-------------|-------|
| /dev/sda1 | 100 GB | ext4 | / (root) | - |
| /dev/sda2 | 3 GB | linux-swap | [SWAP] | - |
| /dev/sda3 | 23 GB | ext4 | /home | - |

**Steps:**
1. Use entire disk: `/dev/sda` (128 GB SSD)
2. Select "Automatic partitioning" or manually create the three partitions above
3. Confirm and continue

**Note**: The remaining drive space remains unallocated for future use.

---

## Part 4: User and Network Configuration

1. **Root Password**
   - Set a strong password for `root` account
   - Remember this for future `sudo` operations

2. **Regular User**
   - Create user: `homeserver`
   - Set a strong password
   - This account owns the Nextcloud data

3. **Network**
   - Hostname: `homeserver` (or any name you prefer)
   - Domain name: Leave blank
   - DHCP will auto-assign IP address

4. **Timezone**
   - Select your timezone

---

## Part 5: Post-Installation System Update

After first boot, open terminal and run:

```bash
# Update package lists
sudo apt update

# Full system upgrade
sudo apt full-upgrade -y

# Install essential packages
sudo apt install -y \
  htop \
  curl \
  unzip \
  git \
  ufw \
  smartmontools \
  rsync
```

**Wait for completion** (~5–15 minutes depending on hardware and internet speed).

### What each package does:
- **htop**: System monitoring and process viewer
- **curl**: Command-line download tool
- **unzip**: Archive extraction utility
- **git**: Version control (for driver installations later)
- **ufw**: Uncomplicated Firewall
- **smartmontools**: Hard drive health monitoring
- **rsync**: Fast file backup/sync tool

---

## Part 6: Verify Installation

```bash
# Check Debian version
lsb_release -a

# Expected output: Debian GNU/Linux 12 (Bookworm)

# Check kernel
uname -r

# Expected: 6.1.x or higher

# Check disk layout
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

# Check free space
df -h

# Check RAM
free -h
```

---

## Part 7: Optional - Install XFCE Desktop (if not selected)

If you skipped XFCE during installation:

```bash
sudo apt install -y xfce4 xfce4-terminal xfce4-goodies lightdm
sudo systemctl set-default graphical.target
```

To disable XFCE desktop to save RAM for server operation:

```bash
sudo systemctl set-default multi-user.target  # CLI only
sudo systemctl set-default graphical.target   # GUI enabled
```

---

## Troubleshooting

### USB not detected in BIOS
- Try different USB port (rear I/O preferred)
- Use short USB cable, avoid power hubs
- Recreate Ventoy USB and re-copy ISO

### Installation hangs during disk format
- Wait 5–10 minutes (large disk formatting can be slow)
- If truly frozen, reboot and try again

### No internet after installation
- Check DHCP is enabled in BIOS
- Run: `sudo systemctl restart networking`
- Verify ethernet is connected

### Keyboard/mouse not responding
- Try USB 2.0 ports (usually darker on PC)
- Use wired input devices, not wireless during installation

---

## Next Steps

After successful Debian installation:

1. Configure storage drives → See [Docs/Server-guide.md](../Docs/Server-guide.md#setting-up-drives-and-pools)
2. Install network drivers → See [Commands/updating-network-driver.md](./updating-network-driver.md)
3. Configure SSH → See [Docs/Server-guide.md](../Docs/Server-guide.md#installing-ssh)
4. Install Nextcloud → See [Docs/Server-guide.md](../Docs/Server-guide.md#installing-nextcloud)
