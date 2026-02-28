# Nextcloud Installation and Configuration

## Overview

Deploy **Nextcloud 26.0.4** on Apache with MariaDB, Redis caching, and optimized PHP settings for family cloud storage and automatic photo backup.

---

## Part 1: Install Dependencies

### Install Web Stack

```bash
sudo apt update
sudo apt install -y \
  apache2 \
  mariadb-server \
  libapache2-mod-php \
  php php-cli php-fpm php-mysql \
  php-gd php-curl php-xml \
  php-mbstring php-zip php-bcmath \
  php-intl php-imagick \
  php-redis php-bz2 php-common \
  php-ldap php-gmp php-apcu \
  unzip \
  redis-server

# Wait for completion
```

### Verify Services Running

```bash
# Apache
sudo systemctl status apache2

# MariaDB
sudo systemctl status mariadb

# Redis
sudo systemctl status redis-server
```

---

## Part 2: Download and Install Nextcloud

### Download Latest Nextcloud

```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/nextcloud-26.0.4.zip

# Verify download
ls -lh nextcloud-26.0.4.zip

# Extract
unzip nextcloud-26.0.4.zip
```

### Move to Storage Pool

```bash
# Move to MergerFS pool
sudo mv nextcloud /srv/data/nextcloud

# Verify location
ls -la /srv/data/nextcloud
```

---

## Part 3: File Permissions and Group Setup

Nextcloud requires both the `homeserver` user and `www-data` (Apache) to access files.

### Create Shared Group

```bash
# Create group
sudo groupadd nextcloud

# Add both users
sudo usermod -aG nextcloud homeserver
sudo usermod -aG nextcloud www-data

# Verify membership
groups homeserver
groups www-data

# Expected: nextcloud in both lists
```

### Set Ownership and Permissions

```bash
# Change ownership to homeserver, group to nextcloud
sudo chown -R homeserver:nextcloud /srv/data/nextcloud

# Enforce 2770 (setgid bit; new files inherit group)
sudo chmod -R 2770 /srv/data/nextcloud

# Recursively set directory and file permissions
sudo find /srv/data/nextcloud -type d -exec chmod 2770 {} \;
sudo find /srv/data/nextcloud -type f -exec chmod 660 {} \;

# Verify
ls -ld /srv/data/nextcloud

# Expected: drwxrwx--- with setgid bit
```

---

## Part 4: MariaDB Setup

### Secure MariaDB

```bash
sudo systemctl start mariadb

# Run security setup
sudo mysql_secure_installation

# Answer prompts:
# - Switch to unix_socket authentication? [Y] Press Y
# - Remove anonymous users? [Y] Press Y
# - Disable root login remotely? [Y] Press Y
# - Remove test database? [Y] Press Y
# - Reload privilege tables? [Y] Press Y
```

### Create Nextcloud Database and User

```bash
# Connect to MariaDB
sudo mysql -u root

# Inside MariaDB prompt, run (replace password!):
```

```sql
CREATE DATABASE nextcloud;

CREATE USER 'ncuser'@'localhost' IDENTIFIED BY 'SetAStrongPasswordHere!';

GRANT ALL PRIVILEGES ON nextcloud.* TO 'ncuser'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

**Note**: Save the password; you'll need it during Nextcloud setup.

---

## Part 5: Configure Apache

### Create Virtual Host Config

```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```

Paste:

```apache
Alias /nextcloud "/srv/data/nextcloud/"

<Directory /srv/data/nextcloud/>
    Require all granted
    Options FollowSymlinks MultiViews
    AllowOverride All
    <IfModule mod_dav.c>
        Dav off
    </IfModule>
</Directory>

<Directory /srv/data/nextcloud/data/>
    Require all denied
</Directory>

ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
```

Save (Ctrl+O, Enter, Ctrl+X).

### Enable Apache Modules

```bash
# Enable required modules
sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod env
sudo a2enmod dir
sudo a2enmod mime
sudo a2enmod setenvif
sudo a2enmod ssl

# Enable Nextcloud site
sudo a2ensite nextcloud

# Test config syntax
sudo apache2ctl configtest

# Expected: "Syntax OK"
```

### Restart Apache

```bash
sudo systemctl restart apache2

# Verify running
sudo systemctl status apache2
```

---

## Part 6: Redis Caching

### Install Redis Extension

```bash
# Already installed from Part 1, but verify
php --version
# Expected: PHP 8.X

# Find PHP version
PHP_VERSION=$(php -r 'echo PHP_VERSION;' | cut -d. -f1,2)
echo "PHP Version: $PHP_VERSION"
```

### Enable Redis in PHP

```bash
# Find PHP config file
php --ini | grep "Loaded Configuration File"

# Edit Apache PHP config (example: PHP 8.1)
sudo nano /etc/php/8.1/apache2/php.ini

# Uncomment or add:
# extension=redis.so

# Find the line (search for "redis" or ";extension=redis")
# If not present, near line 800, add:
# extension=redis.so
```

### Restart Apache to Load Redis

```bash
sudo systemctl restart apache2

# Verify Redis extension loaded
php -m | grep redis

# Expected: redis in output
```

---

## Part 7: PHP Configuration Tuning

### Raise Upload Limits

```bash
# Edit PHP config
sudo nano /etc/php/8.1/apache2/php.ini

# Find and modify these lines (search with Ctrl+W in nano):
```

```
upload_max_filesize = 25G
post_max_size = 25G
max_execution_time = 3600
max_input_time = 3600
memory_limit = 512M
```

### Restart Apache

```bash
sudo systemctl restart apache2
```

### Verify Settings

```bash
# Check active settings
php -i | grep -E "upload_max_filesize|post_max_size|max_execution_time"
```

---

## Part 8: Nextcloud Manual Setup

### Access Nextcloud Web UI

Open browser to:

**Local Network:**
```
http://192.168.1.12/nextcloud
```

**Global (if Cloudflare Tunnel running):**
```
https://storagepaglu.pp.ua/nextcloud
```

### Initial Configuration

On the Nextcloud setup page:

1. **Admin Account:**
   - Username: `admin`
   - Password: (strong password, remember this)

2. **Database Settings:**
   - Database User: `ncuser`
   - Database Password: (the password you set in MariaDB)
   - Database Name: `nextcloud`
   - Localhost Database Host: `localhost`

3. **Recommended Apps:**
   - Can install later; leave as default

4. Click **"Finish Setup"**

### Post-Setup Verification

```bash
# Rescan files to show in Nextcloud
sudo -u homeserver php /srv/data/nextcloud/occ files:scan --all

# Expected: "____" rows scanned in X seconds
```

---

## Part 9: Nextcloud Configuration

### Configure Redis Cache

```bash
# Edit Nextcloud config
sudo nano /srv/data/nextcloud/config/config.php
```

Add before the closing `);`:

```php
'memcache.local' => '\\OC\\Memcache\\APCu',
'memcache.distributed' => '\\OC\\Memcache\\Redis',
'redis' => array(
  'host' => 'localhost',
  'port' => 6379,
  'timeout' => 0,
),
'filelocking.enabled' => true,
'memcache.locking' => '\\OC\\Memcache\\Redis',
```

### Set Trusted Domains

```bash
# Edit config
sudo nano /srv/data/nextcloud/config/config.php

# After 'datadirectory', add:
'trusted_domains' => array(
  '192.168.1.12',
  'storagepaglu.pp.ua',
  'localhost',
),
```

### Restart Apache to Apply Changes

```bash
sudo systemctl restart apache2
```

---

## Part 10: Create Family Member Accounts

### Via Web UI (Simplest)

1. Log in as `admin` to Nextcloud
2. Click profile (top right) → Settings → Users
3. Create new users:
   - Click "New User"
   - Username: (e.g., `brother`, `sister`)
   - Password: Send securely to family member
   - Groups: Leave blank or create as needed
   - Storage Quota: (e.g., 100 GB)
4. Share folders with new users

### Set Quotas

```bash
# Via command line (optional)
sudo -u www-data php /srv/data/nextcloud/occ user:list

# Set quota for user (500 GB)
sudo -u www-data php /srv/data/nextcloud/occ user:setting brother quota "500 GB"
```

---

## Part 11: Android App Setup

### Install Apps

From **Google Play Store**:
1. Install **Nextcloud** (main app)
2. Install **Nextcloud Memories** (Google Photos-like interface)

### Configure Auto Photo Backup

1. Open **Nextcloud** app
2. Log in with account (e.g., `homeserver`)
3. Server URL: `https://storagepaglu.pp.ua/nextcloud` (or local IP)
4. Settings → Auto Upload:
   - **Enable auto upload** ✓
   - Source folder: **DCIM/Camera**
   - Upload folder: **/Memories** (or desired folder)
   - **Upload only on Wi-Fi** ✓
   - **Upload while charging** (optional)

5. Open **Nextcloud Memories** app
6. Sign in with same account
7. View timeline of all backed-up photos

---

## Part 12: Troubleshooting

### Files Not Visible in Nextcloud

```bash
# Rescan files (when manually copied)
sudo -u homeserver php /srv/data/nextcloud/occ files:scan homeserver

# For all users
sudo -u homeserver php /srv/data/nextcloud/occ files:scan --all
```

### Permission Denied Upload

```bash
# Verify group permissions
ls -ld /srv/data/nextcloud

# Reapply if needed
sudo chown -R homeserver:nextcloud /srv/data/nextcloud
sudo chmod -R 2770 /srv/data/nextcloud
```

### Database Connection Error

```bash
# Verify MariaDB running
sudo systemctl status mariadb

# Test connection
sudo mysql -u ncuser -p nextcloud

# Enter password; if it connects, database is fine
EXIT;
```

### Nextcloud Too Slow

```bash
# Verify Redis running
sudo systemctl status redis-server

# Check Apache processes
ps aux | grep apache | head -5

# Increase Apache workers if needed
# (Advanced; see Apache docs)
```

---

## Related Guides

- [Storage Configuration](./Storage-Config.md) – MergerFS pool setup
- [SSH Remote Access](./SSH-Setup.md) – Command-line maintenance
- [Global Access](./Cloudflare-Tunnel.md) – Internet accessibility
