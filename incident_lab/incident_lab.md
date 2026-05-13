# Ubuntu 24.04.4 Incident Lab — Step-by-Step Technical Implementation Guide

**Scope:** technical implementation only. The final report, screenshots, IRIS evidence write-up, and PDF packaging are intentionally left out of scope.

**Primary platform:** one Ubuntu 24.04.4 LTS VM in VMware.

**Required container inside WEB01:** an Ubuntu 24.04 LXD container for DFIR-IRIS.

**Default all-Ubuntu attack source:** an Ubuntu 24.04 LXD container named ATTACKER01, used to generate Nmap/Nikto traffic when Kali is not used.

**Optional external VM:** a Kali Linux VM in the same isolated lab network. Every Kali-related step is marked **[OPTIONAL — Kali VM]**.

**Important:** all commands are for an isolated training lab that you own/control. Do not expose the vulnerable Apache VM or DFIR-IRIS instance to the public Internet.

---

## 0. How to read this guide

Unless a heading says otherwise, run the commands on the main Ubuntu VM, referred to as **WEB01**.

Command-location labels used later:

| Label | Where the command is run |
|---|---|
| **WEB01** | Main Ubuntu 24.04.4 LTS VM |
| **IRIS01** | Ubuntu 24.04 LXD container that runs DFIR-IRIS |
| **ATTACKER01** | Default all-Ubuntu LXD attacker container used when Kali is not used |
| **KALI01** | Optional Kali Linux VM |

Default example addresses used in this guide:

| System | Example IP |
|---|---:|
| WEB01 | `192.168.52.134` |
| KALI01, if used | `192.168.56.20` |

Replace `192.168.52.134` with the real IP of your Ubuntu VM if you use another address.

---

## 1. Target lab design

The assignment is implemented as follows:

| Assignment component | Implementation |
|---|---|
| Apache web server | Installed on WEB01 |
| Course `index.html` replacement | `html.txt` copied to `/var/www/html/index.html` |
| Exposed backup script | `/var/www/html/backup.sh` |
| Exposed password file | `/var/www/html/.env` |
| `auditd` logging | Enabled on WEB01 |
| `sudo` forensic logging | Enabled on WEB01 |
| Bash history logging | Enabled on WEB01 |
| `osquery` | Installed on WEB01 in interactive mode |
| DFIR-IRIS | Installed inside IRIS01 LXD container |
| Nmap / Zenmap / Nikto attack simulation | KALI01 if available, otherwise ATTACKER01 LXD container |
| SSH login attempts from IRIS LXC | Performed from IRIS01 to WEB01 |

Recommended VMware network design:

| VMware network | Purpose |
|---|---|
| NAT | Package downloads, GitHub, Docker image pulls |
| Host-only / isolated lab network | Lab traffic between WEB01 and optional KALI01 |

**[OPTIONAL]** Take VMware snapshots at these points:

| Snapshot name | Suggested timing |
|---|---|
| `00-clean-install` | Fresh Ubuntu install |
| `01-web-configured` | Apache, auditd, sudo, Bash, osquery configured |
| `02-iris-configured` | DFIR-IRIS reachable |
| `03-before-attack` | Immediately before scans and SSH attempts |
| `04-after-attack` | After simulated compromise activity |

---

## 2. Prepare WEB01

### 2.1 Confirm Ubuntu version

```bash
lsb_release -a
# Print Ubuntu release information.
# -a shows all available distribution fields.
```

Expected release family: Ubuntu 24.04 LTS.

### 2.2 Set a clear hostname

```bash
sudo hostnamectl set-hostname web01
# Set the system hostname to web01.
```

```bash
hostnamectl
# Verify the current hostname and operating system metadata.
```

### 2.3 Set timezone

```bash
timedatectl
# Show current time, timezone, and NTP synchronization status.
```

```bash
sudo timedatectl set-timezone Europe/Riga
# Set the system timezone to Europe/Riga.
# Use another timezone only if your course requires it.
```

### 2.4 Identify network interfaces and IP addresses

```bash
ip -br address
# Show network interfaces and IP addresses in brief format.
# -br means brief output.
```

```bash
hostname -I
# Print IP addresses assigned to this VM.
# Use this to identify the WEB01 IP that Kali or attacker containers will target.
```

### 2.5 **[OPTIONAL]** Set a static lab IP on WEB01

Skip this subsection if your Ubuntu VM already has a stable IP address.

```bash
sudo nano /etc/netplan/01-lab.yaml
# Open a Netplan configuration file.
# Replace interface names in the example below with names from `ip -br address`.
```

Example content:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      dhcp4: false
      addresses:
        - 192.168.52.134/24
```

```bash
sudo netplan try
# Test the Netplan configuration with automatic rollback if connectivity breaks.
```

```bash
sudo netplan apply
# Apply the Netplan configuration permanently.
```

### 2.6 Update Ubuntu packages

```bash
sudo apt update
# Refresh package indexes from configured APT repositories.
```

```bash
sudo apt full-upgrade -y
# Upgrade installed packages, allowing dependency changes if needed.
# -y answers yes to prompts.
```

```
# remove packages installed automatically as dependencies but no longer needed:
sudo apt autoremove --purge --simulate
# --simulate show what would happen without changing the system.
sudo apt autoremove --purge
# --purge also remove related package configuration files.
```

**[CONDITIONAL — only if a kernel or core system package was upgraded]** Reboot before continuing.

```bash
sudo reboot
# Reboot WEB01 after major package upgrades.
```

### 2.7 Install base packages on WEB01

Might be already installed, could be checked before the installation!

```bash
sudo apt install -y software-properties-common
# Install add-apt-repository support before enabling Ubuntu Universe if needed.
# -y answers yes to prompts.
```

```bash
sudo add-apt-repository -y universe
# Enable the Ubuntu Universe repository, which is commonly needed for lab utilities such as sshpass.
# -y answers yes to prompts.
```

```bash
sudo apt update
# Refresh package indexes again after enabling Universe.
```

```bash
sudo apt install -y apache2 openssh-server openssh-client auditd audispd-plugins rsyslog cron ufw curl wget git unzip zip ca-certificates gnupg dirmngr rsync sshpass nano vim snapd btrfs-progs e2fsprogs
# Install the packages needed for the lab.
# -y answers yes to prompts.
# apache2 = web server.
# openssh-server = SSH service on WEB01.
# openssh-client = SSH client tools.
# auditd/audispd-plugins = Linux audit framework and dispatcher plugins.
# rsyslog = traditional log files such as /var/log/auth.log.
# cron = creates standard cron paths watched by the audit rules.
# ufw = firewall status/rule tool used by the conditional firewall step.
# curl/wget = HTTP download and testing tools.
# git = required for DFIR-IRIS repository cloning.
# unzip/zip = archive tools watched by auditd rules.
# ca-certificates/gnupg/dirmngr = repository signing and key handling tools.
# rsync/sshpass = used by the intentionally exposed backup script.
# nano/vim = editors watched by auditd rules.
# snapd = required for LXD snap installation.
# btrfs-progs = supports the Btrfs LXD storage pool used for nested Docker.
# e2fsprogs = provides chattr, which is included in the auditd watch rules.
```

### 2.8 Enable core services

```bash
sudo systemctl enable --now apache2
# Enable Apache at boot and start it immediately.
# --now starts the service immediately instead of only enabling it for next boot.
```

```bash
sudo systemctl enable --now ssh
# Enable OpenSSH Server at boot and start it immediately.
# On Ubuntu the OpenSSH Server systemd unit is named ssh.
```

```bash
sudo systemctl enable --now rsyslog
# Enable rsyslog at boot and start it immediately.
```

```bash
sudo systemctl enable --now cron
# Enable cron at boot and start it immediately.
```

```bash
sudo systemctl enable --now auditd
# Enable auditd at boot and start it immediately.
```

```bash
systemctl status apache2 ssh rsyslog cron auditd --no-pager
# Show service health without opening a pager.
# --no-pager prints directly to the terminal.
```

### 2.9 **[CONDITIONAL — only if UFW is active]** Allow SSH and HTTP

Most fresh Ubuntu desktop/server installations do not block inbound traffic unless UFW was enabled manually. Check first.

```bash
sudo ufw status verbose
# Show UFW firewall state and rules.
# verbose prints extra detail such as default policy.
```

Run the next two commands only if UFW status is `active`.

```bash
sudo ufw allow from 192.168.56.0/24 to any port 22 proto tcp
# Allow inbound SSH to WEB01 from the example isolated lab subnet only.
# from 192.168.56.0/24 limits the source network.
# to any port 22 targets WEB01's local SSH port.
# proto tcp limits the rule to TCP.
```

```bash
sudo ufw allow from 192.168.56.0/24 to any port 80 proto tcp
# Allow inbound HTTP to WEB01 from the example isolated lab subnet only.
# from 192.168.56.0/24 limits the source network.
# to any port 80 targets WEB01's local Apache port.
# proto tcp limits the rule to TCP.
```

---

## 3. Put the Digital Forensics course materials on WEB01

Make Digital Forensics course materials be available offline

```
mkdir -p ~/kiberacs_materials
# Create a directory to store the downloaded course materials.

cd ~/kiberacs_materials
# Enter that directory.

wget --mirror --no-parent --accept=.txt,.html http://zerobank.vip:8099/kiberacs/
# Download the page and linked .txt/.html files under /kiberacs/.
# --mirror enables recursive retrieval with sensible defaults.
# --no-parent prevents wget from going above /kiberacs/.
# --accept limits downloads to .txt and .html content.

tar -czf kiberacs_materials.tar.gz .
# Pack everything into a single archive to feed to AI for an automated guide preparation.
```

If later on the archive will need to be unpacked:
```bash
mkdir -p ~/lab-materials
# Create a local directory for unpacked materials.
# -p avoids an error if the directory already exists.
```

```bash
tar -xzf ~/Downloads/kiberacs.tar.gz -C ~/lab-materials
# Extract the course archive into ~/lab-materials.
# -x extracts files.
# -z handles gzip compression.
# -f specifies the archive file.
# -C sets the extraction target directory.
```

```bash
find ~/lab-materials -type f -name html.txt -printf '%h\n' | head -n 1
# Find the directory that contains html.txt.
# -type f searches files only.
# -name html.txt matches the course HTML file.
# -printf '%h\n' prints the parent directory.
# head -n 1 prints only the first matching directory.
```

Set a shell variable for the course directory:

```bash
export COURSE_DIR="$(find "$HOME/lab-materials" -type f -name html.txt -printf '%h\n' | head -n 1)"
# Store the detected course-material directory in COURSE_DIR.
# $HOME expands to your home directory.
# The command substitution $(...) uses the output of find/head as the value.
```

```bash
ls -l "$COURSE_DIR"
# List the detected course directory to confirm that all expected .txt files are present.
# Quoting "$COURSE_DIR" protects paths that contain spaces.
```

```bash
test -n "$COURSE_DIR" && test -f "$COURSE_DIR/html.txt"
# Verify that COURSE_DIR is not empty and that html.txt exists.
# test -n checks that the variable has a value.
# test -f checks that the named path is a regular file.
# && runs the second test only if the first test succeeds.
```

---

## 4. Create the lab user `kiberacs`

The screenshot and course files use:

| Field | Value |
|---|---|
| User | `kiberacs` |
| Password | `Google2026` |

```bash
sudo adduser --disabled-password --gecos "" kiberacs
# Create the local user kiberacs.
# --disabled-password creates the account without asking for a password interactively.
# --gecos "" supplies empty user-profile fields non-interactively.
```

```bash
echo 'kiberacs:Google2026' | sudo chpasswd
# Set the password for kiberacs.
# chpasswd reads username:password pairs from standard input.
```

```bash
sudo usermod -aG sudo kiberacs
# Add kiberacs to the sudo group.
# -a appends the group instead of replacing existing groups.
# -G specifies the supplementary group list.
```

```bash
id kiberacs
# Verify the user exists and is a member of the expected groups.
```

---

## 5. Configure Apache web content

### 5.1 Back up the default Apache page

```bash
sudo cp /var/www/html/index.html /var/www/html/index.html.orig
# Save the default Apache page before replacing it.
```

### 5.2 Replace the Apache landing page with course HTML

```bash
sudo install -m 0644 "$COURSE_DIR/index.html" /var/www/html/index.html
# Copy index.html into the Apache document root as a new index.html.
# install copies a file and sets permissions in one operation.
# -m 0644 sets owner read/write and group/other read-only permissions.
```

```bash
curl -I http://127.0.0.1
# Request only HTTP headers from local Apache.
# -I sends an HTTP HEAD request.
```

```bash
curl http://127.0.0.1 | head -n 20
# Fetch the served page and print the first 20 lines.
# head -n 20 keeps the output short.
```

### 5.3 Create the intentionally exposed `.env` credential file

The uploaded course material places the leaked password in `/.env`; `backup.sh` references that file. This is intentionally insecure for the lab.

```bash
sudo nano /var/www/html/.env
# Create the intentionally exposed environment file under the Apache document root.
```

Paste this content:

```text
BACKUP_USER=webadmin
BACKUP_PASS=Google2026
BACKUP_HOST=192.168.1.50
BACKUP_PATH=/srv/backup/web/
```

```bash
sudo chmod 0644 /var/www/html/.env
# Make the file readable by Apache and by HTTP clients that request it.
# 0644 = owner read/write, group read, others read.
```

```bash
sudo chown root:root /var/www/html/.env
# Set root as owner and group for the file.
```

### 5.4 Create the intentionally exposed backup script

```bash
sudo nano /var/www/html/backup.sh
# Create the intentionally exposed backup script under the Apache document root.
```

Paste this content:

```bash
#!/bin/bash

set -euo pipefail

source /var/www/html/.env

DATE_STAMP=$(date +%F_%H-%M-%S)
ARCHIVE_FILE="/tmp/forensic_web_${DATE_STAMP}.tar.gz"
SOURCE_DIR="/var/www/html"

echo "[*] Creating archive: ${ARCHIVE_FILE}"
tar -czf "${ARCHIVE_FILE}" "${SOURCE_DIR}"

echo "[*] Sending archive to backup server ${BACKUP_HOST}"
sshpass -p "${BACKUP_PASS}" rsync -avz "${ARCHIVE_FILE}" "${BACKUP_USER}@${BACKUP_HOST}:${BACKUP_PATH}"

echo "[*] Removing temporary archive"
rm -f "${ARCHIVE_FILE}"

echo "[+] Backup completed successfully"
```

```bash
sudo chmod 0755 /var/www/html/backup.sh
# Make the script readable and executable.
# 0755 = owner read/write/execute, group read/execute, others read/execute.
```

Do not execute `backup.sh` during normal lab setup. It intentionally references a fake/offline backup destination so that the script and `.env` file can be discovered over HTTP.

```bash
sudo chown root:root /var/www/html/backup.sh
# Set root as owner and group for the script.
```

### 5.5 Validate that the exposed files are reachable

```bash
curl http://127.0.0.1/backup.sh
# Retrieve backup.sh over HTTP from local Apache.
```

```bash
curl http://127.0.0.1/.env
# Retrieve .env over HTTP from local Apache.
# This confirms the intentionally leaked password is web-accessible.
```

---

## 6. Configure SSH logging on WEB01

Use an SSH drop-in file instead of editing the main `/etc/ssh/sshd_config` directly.

```bash
sudo install -m 0755 -d /etc/ssh/sshd_config.d
# Ensure the OpenSSH server drop-in directory exists.
# -d creates a directory.
# -m 0755 sets directory permissions.
```

```bash
sudo nano /etc/ssh/sshd_config.d/00-forensic-logging.conf
# Create a high-priority OpenSSH server drop-in file.
# The 00- prefix makes it load early in the include directory.
```

Paste this content:

```text
SyslogFacility AUTH
LogLevel VERBOSE
PasswordAuthentication yes
PubkeyAuthentication yes
PermitRootLogin no
PrintLastLog yes
```

```bash
sudo /usr/sbin/sshd -t
# Validate the effective OpenSSH server configuration syntax.
# -t tests configuration and exits without starting a new daemon.
```

```bash
sudo systemctl restart ssh
# Restart the SSH service so the logging and password-authentication settings take effect.
```

```bash
sudo /usr/sbin/sshd -T | grep -Ei 'loglevel|passwordauthentication|permitrootlogin|syslogfacility'
# Print selected effective SSH daemon settings.
# -T prints the effective configuration.
# grep -E enables extended regex matching.
# -i makes matching case-insensitive.
```

```bash
sudo systemctl status ssh --no-pager
# Confirm that SSH restarted successfully.
# --no-pager prints directly to the terminal.
```

---

## 7. Configure `sudo` forensic logging

### 7.1 Create sudo logging policy

```bash
sudo nano /etc/sudoers.d/forensic-logging
# Create a dedicated sudoers include file for logging policy.
```

Paste this content:

```text
Defaults logfile="/var/log/sudo.log"
Defaults loglinelen=0
Defaults log_input,log_output
Defaults iolog_dir="/var/log/sudo-io"
Defaults use_pty
```

### 7.2 Set safe sudoers permissions and validate syntax

```bash
sudo chmod 0440 /etc/sudoers.d/forensic-logging
# Set standard sudoers include-file permissions.
# 0440 = owner read, group read, no write/execute for group/others.
```

```bash
sudo visudo -cf /etc/sudoers.d/forensic-logging
# Validate sudoers syntax without editing.
# -c checks syntax.
# -f selects the file to validate.
```

### 7.3 Create sudo I/O log directory

```bash
sudo mkdir -p /var/log/sudo-io
# Create the directory where sudo I/O session recordings are stored.
# -p avoids an error if the directory already exists.
```

```bash
sudo chmod 0700 /var/log/sudo-io
# Restrict sudo I/O recording directory access to root.
# 0700 = owner read/write/execute only.
```

```bash
sudo touch /var/log/sudo.log
# Create the sudo log file if it does not exist yet.
```

```bash
sudo chmod 0640 /var/log/sudo.log
# Set readable-by-root/group permissions for the sudo log.
# 0640 = owner read/write, group read, others no access.
```

### 7.4 Generate a test sudo event

```bash
sudo -k
# Invalidate cached sudo credentials.
# -k forces sudo to ask for authentication again next time.
```

```bash
sudo whoami
# Run a simple sudo command to generate a sudo log entry.
```

```bash
sudo tail -n 20 /var/log/sudo.log
# Show the last 20 sudo log lines.
# -n 20 limits output to 20 lines.
```

---

## 8. Configure Bash history logging

### 8.1 Create global Bash history settings

```bash
sudo nano /etc/profile.d/forensic-history.sh
# Create a global shell profile snippet for future login shells.
```

Paste this content:

```bash
if [ -n "$BASH_VERSION" ]; then
    export HISTSIZE=50000
    export HISTFILESIZE=100000
    export HISTCONTROL=
    export HISTIGNORE=
    export HISTTIMEFORMAT="%F %T "
    shopt -s histappend
    export PROMPT_COMMAND='history -a; history -n'
fi

export TMOUT=0
```

Why the Bash check matters: `/etc/profile.d/*.sh` can be sourced by non-Bash shells. The `if [ -n "$BASH_VERSION" ]` guard prevents `shopt` errors in non-Bash shells.

```bash
sudo chmod 0644 /etc/profile.d/forensic-history.sh
# Make the profile snippet readable by all users and writable only by root.
```

### 8.2 Add similar settings for root Bash sessions

```bash
sudo nano /root/.bashrc
# Open root's Bash startup file.
```

Append this block at the end:

```bash
export HISTSIZE=50000
export HISTFILESIZE=100000
export HISTCONTROL=
export HISTIGNORE=
export HISTTIMEFORMAT="%F %T "
shopt -s histappend
export PROMPT_COMMAND='history -a; history -n'
```

```bash
sudo touch /root/.bash_history
# Create root's Bash history file if it does not exist yet.
```

```bash
sudo chmod 0600 /root/.bash_history
# Restrict root history file permissions.
# 0600 = owner read/write only.
```

The settings apply automatically to new login shells. Log out and log back in, or start a new terminal, before testing Bash history behavior.

---

## 9. Configure `auditd`

This ruleset is course-aligned but adjusted for Ubuntu 24.04.4 so the watches load cleanly. It keeps the important forensic keys from the course material, including identity, privilege, SSH, webroot, command execution, deletion, permissions, cron, package management, and network tools.

### 9.1 Create paths required by audit watches

```bash
sudo mkdir -p /root/.ssh /var/spool/cron /etc/NetworkManager /etc/network
# Create directories referenced by audit watch rules.
# -p creates parents as needed and avoids errors if directories already exist.
```

```bash
sudo touch /root/.bash_history /etc/security/opasswd /var/log/sudo.log /var/log/auth.log /var/log/syslog /var/log/apache2/access.log /var/log/apache2/error.log
# Create files referenced by audit watch rules if they do not already exist.
```

```bash
sudo chmod 0600 /root/.bash_history /etc/security/opasswd
# Restrict sensitive placeholder files.
# 0600 = owner read/write only.
```

```bash
sudo chmod 0640 /var/log/sudo.log
# Set conservative permissions on the sudo log.
# 0640 = owner read/write, group read, others no access.
```

### 9.2 Create the audit rules file

```bash
sudo nano /etc/audit/rules.d/forensics.rules
# Create the course-aligned persistent audit rules file.
```

Paste this content:

```text
-D
-b 8192
-f 1
--backlog_wait_time 60000

-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/security/opasswd -p wa -k identity

-w /etc/sudoers -p wa -k privilege
-w /etc/sudoers.d/ -p wa -k privilege
-w /usr/bin/sudo -p x -k sudo_cmd
-w /usr/bin/su -p x -k su_cmd

-w /etc/ssh/sshd_config -p wa -k ssh_config
-w /etc/ssh/sshd_config.d/ -p wa -k ssh_config
-w /etc/ssh/ssh_config -p wa -k ssh_config
-w /etc/ssh/ssh_config.d/ -p wa -k ssh_config
-w /root/.ssh/ -p wa -k ssh_keys
-w /home/ -p wa -k home_ssh_watch

-w /etc/pam.d/ -p wa -k pam
-w /etc/rsyslog.conf -p wa -k logging
-w /etc/rsyslog.d/ -p wa -k logging
-w /etc/systemd/journald.conf -p wa -k logging
-w /var/log/auth.log -p wa -k authlog
-w /var/log/syslog -p wa -k syslog
-w /var/log/sudo.log -p wa -k sudo_log

-w /var/www/html/ -p wa -k webroot
-w /etc/apache2/ -p wa -k apache_config
-w /var/log/apache2/access.log -p wa -k apache_access
-w /var/log/apache2/error.log -p wa -k apache_error

-w /root/.bash_history -p wa -k shell_history
-w /home/ -p wa -k user_homes

-w /etc/crontab -p wa -k cron
-w /etc/cron.d/ -p wa -k cron
-w /etc/cron.daily/ -p wa -k cron
-w /etc/cron.hourly/ -p wa -k cron
-w /etc/cron.monthly/ -p wa -k cron
-w /etc/cron.weekly/ -p wa -k cron
-w /var/spool/cron/ -p wa -k cron

-w /etc/systemd/system/ -p wa -k systemd
-w /usr/lib/systemd/system/ -p wa -k systemd
-w /lib/systemd/system/ -p wa -k systemd
-w /etc/init.d/ -p wa -k init_scripts

-w /etc/hosts -p wa -k network_config
-w /etc/hostname -p wa -k network_config
-w /etc/resolv.conf -p wa -k network_config
-w /etc/netplan/ -p wa -k network_config
-w /etc/NetworkManager/ -p wa -k network_config

-w /usr/bin/apt -p x -k package_mgmt
-w /usr/bin/apt-get -p x -k package_mgmt
-w /usr/bin/dpkg -p x -k package_mgmt
-w /usr/bin/snap -p x -k package_mgmt

-w /usr/bin/nano -p x -k editor_use
-w /usr/bin/vim -p x -k editor_use
-w /usr/bin/vi -p x -k editor_use
-w /usr/bin/cp -p x -k file_copy
-w /usr/bin/mv -p x -k file_move
-w /usr/bin/rm -p x -k file_remove
-w /usr/bin/touch -p x -k file_touch
-w /usr/bin/chmod -p x -k perm_mod_cmd
-w /usr/bin/chown -p x -k perm_mod_cmd
-w /usr/bin/chattr -p x -k perm_mod_cmd

-w /usr/bin/tar -p x -k archive_cmd
-w /usr/bin/gzip -p x -k archive_cmd
-w /usr/bin/zip -p x -k archive_cmd
-w /usr/bin/unzip -p x -k archive_cmd
-w /usr/bin/curl -p x -k network_tool
-w /usr/bin/wget -p x -k network_tool
-w /usr/bin/scp -p x -k network_tool
-w /usr/bin/rsync -p x -k network_tool
-w /usr/bin/ssh -p x -k network_tool
-w /usr/bin/sshpass -p x -k network_tool

-w /usr/bin/mount -p x -k mount_cmd
-w /usr/bin/umount -p x -k mount_cmd
-w /usr/sbin/insmod -p x -k kernel_module
-w /usr/sbin/rmmod -p x -k kernel_module
-w /usr/sbin/modprobe -p x -k kernel_module

-a always,exit -F arch=b64 -S adjtimex,settimeofday,clock_settime -k time_change
-a always,exit -F arch=b32 -S adjtimex,settimeofday,clock_settime -k time_change
-w /etc/localtime -p wa -k time_change
-w /etc/timezone -p wa -k time_change

-a always,exit -F arch=b64 -S execve -k cmd_exec
-a always,exit -F arch=b32 -S execve -k cmd_exec

-a always,exit -F arch=b64 -S unlink,unlinkat,rename,renameat,rmdir -k file_delete
-a always,exit -F arch=b32 -S unlink,unlinkat,rename,renameat,rmdir -k file_delete

-a always,exit -F arch=b64 -S chmod,fchmod,fchmodat,chown,fchown,fchownat,lchown -k perm_change
-a always,exit -F arch=b32 -S chmod,fchmod,fchmodat,chown,fchown,fchownat,lchown -k perm_change

-a always,exit -F arch=b64 -S open,openat,creat,truncate,ftruncate -F dir=/var/www/html -F perm=wa -k web_write
-a always,exit -F arch=b64 -S open,openat,creat,truncate,ftruncate -F dir=/etc -F perm=wa -k etc_write
-a always,exit -F arch=b64 -S open,openat,creat,truncate,ftruncate -F dir=/home -F perm=wa -k home_write

-a always,exit -F arch=b64 -S open,openat,creat,truncate,ftruncate -F auid>=1000 -F auid!=4294967295 -k user_file_open
-a always,exit -F arch=b32 -S open,openat,creat,truncate,ftruncate -F auid>=1000 -F auid!=4294967295 -k user_file_open
-a always,exit -F arch=b64 -S unlink,unlinkat,rename,renameat,rmdir,mkdir,mkdirat -F auid>=1000 -F auid!=4294967295 -k user_file_delete
-a always,exit -F arch=b32 -S unlink,unlinkat,rename,renameat,rmdir,mkdir,mkdirat -F auid>=1000 -F auid!=4294967295 -k user_file_delete

-a always,exit -F arch=b64 -S kill,tkill,tgkill -k signal_send
-a always,exit -F arch=b32 -S kill,tkill,tgkill -k signal_send

-w /usr/sbin/reboot -p x -k system_power
-w /usr/sbin/shutdown -p x -k system_power
-w /usr/bin/systemctl -p x -k systemctl_cmd

-e 1
```

### 9.3 Load and verify audit rules

```bash
sudo augenrules --load
# Compile and load persistent audit rules from /etc/audit/rules.d/.
```

Do not restart `auditd` with `systemctl restart auditd` on Ubuntu; loading with `augenrules --load` is enough and avoids service-control refusal behavior.

```bash
sudo auditctl -s
# Show audit subsystem status.
# Useful fields include enabled, backlog, lost, and failure.
```

```bash
sudo auditctl -l | head -n 40
# List active audit rules and show the first 40 lines.
# -l lists loaded rules.
# head -n 40 keeps output readable.
```

Generate one harmless webroot event first, then search for it.

```bash
sudo touch /var/www/html/audit-test.tmp
# Create a temporary file in the Apache webroot to generate an audit event.
```

```bash
sudo rm -f /var/www/html/audit-test.tmp
# Remove the temporary audit test file.
# -f avoids an error if the file is already absent.
```

```bash
sudo ausearch -k webroot -ts recent -i
# Search recent audit events with key webroot.
# -k selects an audit key.
# -ts recent limits the search to recent events.
# -i interprets numeric IDs into readable names where possible.
```

---

## 10. Install `osquery` on WEB01

Use `osqueryi` interactively for forensic queries. Keep `osqueryd` disabled in this lab so `auditd` remains the primary event logger.

### 10.1 Add the osquery APT repository

```bash
sudo install -m 0755 -d /etc/apt/keyrings
# Create the APT keyring directory if it does not exist.
# -d creates a directory.
# -m 0755 sets standard directory permissions.
```

```bash
sudo rm -f /etc/apt/keyrings/osquery.gpg
# Remove any old osquery keyring before recreating it.
# -f avoids an error if the file does not exist.
```

```bash
curl -fsSL https://pkg.osquery.io/deb/pubkey.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/osquery.gpg
# Download the osquery repository public key and convert it into a binary keyring.
# curl -f fails on HTTP errors.
# -sS keeps output quiet but still shows errors.
# -L follows redirects.
# gpg --dearmor converts armored key data into APT keyring format.
# -o sets the output file.
```

```bash
sudo chmod a+r /etc/apt/keyrings/osquery.gpg
# Make the keyring readable by APT.
# a+r adds read permission for all classes.
```

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/osquery.gpg] https://pkg.osquery.io/deb deb main" | sudo tee /etc/apt/sources.list.d/osquery.list
# Add the osquery APT repository.
# dpkg --print-architecture inserts the system architecture, usually amd64.
# signed-by limits trust to the osquery keyring.
# tee writes the repository line to the target file with sudo privileges.
```

### 10.2 Install osquery

```bash
sudo apt update
# Refresh package indexes after adding the osquery repository.
```

```bash
sudo apt install -y osquery
# Install osquery.
# -y answers yes to prompts.
```

```bash
sudo systemctl disable --now osqueryd
# Disable and stop the osquery daemon for this lab.
# --now stops it immediately as well as disabling future automatic starts.
```

### 10.3 Test useful osquery queries

```bash
sudo osqueryi "SELECT name, version, platform, codename FROM os_version;"
# Show operating system version information.
```

```bash
sudo osqueryi "SELECT hostname, cpu_brand, physical_memory FROM system_info;"
# Show host, CPU, and memory information.
```

```bash
sudo osqueryi "SELECT username, uid, gid, directory, shell FROM users;"
# List local user accounts.
```

```bash
sudo osqueryi "SELECT user, tty, host, time FROM logged_in_users;"
# Show currently logged-in users.
```

```bash
sudo osqueryi "SELECT pid, name, path FROM processes ORDER BY pid LIMIT 20;"
# List the first 20 processes by PID.
# LIMIT 20 keeps the output short.
```

```bash
sudo osqueryi "SELECT pid, port, protocol, address FROM listening_ports WHERE port IN (22,80);"
# Show listeners for SSH and HTTP.
# IN (22,80) filters to ports 22 and 80.
```

```bash
sudo osqueryi "SELECT id, active_state, sub_state FROM systemd_units WHERE id IN ('ssh.service','apache2.service','auditd.service');"
# Show state of the key systemd services used in this lab.
```

```bash
sudo osqueryi "SELECT * FROM crontab;"
# Show cron entries visible to osquery.
```

---

## 11. Install and initialize LXD on WEB01

DFIR-IRIS will run inside an Ubuntu 24.04 LXD container named **IRIS01**. Docker will run inside that container.

### 11.1 Install LXD

```bash
sudo snap install lxd
# Install LXD from the snap package.
```

If LXD is already installed, refresh it instead:

```bash
sudo snap refresh lxd
# Refresh the installed LXD snap.
```

### 11.2 Allow your user to manage LXD

```bash
getent group lxd | grep -qwF "$USER" || sudo usermod -aG lxd "$USER"
# Add the current user to the lxd group only if not already a member.
# getent group lxd prints the group entry.
# grep -q makes grep quiet.
# -w matches a whole word.
# -F treats the pattern as a fixed string.
# || runs usermod only if the grep check fails.
# usermod -aG appends the lxd group to the user's supplementary groups.
```

```bash
newgrp lxd
# Start a new shell with lxd group membership active immediately.
```

### 11.3 Initialize LXD

```bash
lxc profile list
# Check whether LXD already responds.
# If this prints a profile table, LXD is already initialized and you can skip `lxd init --minimal`.
```

**[CONDITIONAL — only if LXD is not initialized yet]** Run minimal initialization.

```bash
lxd init --minimal
# Initialize LXD with a minimal default configuration.
# --minimal avoids the interactive setup wizard.
```

```bash
lxc storage list
# List configured LXD storage pools.
```

```bash
lxc network list
# List configured LXD networks.
```

### 11.4 Create a Btrfs storage pool for nested Docker

```bash
lxc storage list
# Check existing storage pools before creating the Docker-specific pool.
# If a pool named docker already exists from a previous attempt, skip the next create command or use another pool name consistently.
```

```bash
lxc storage create docker btrfs size=25GiB
# Create a Btrfs LXD storage pool named docker.
# btrfs is used because Docker works well with Btrfs-backed storage inside LXD.
# size=25GiB creates a 25 GiB loop-backed pool, matching the course disk-size requirement.
```

```bash
lxc storage list
# Confirm the docker storage pool was created.
```

Show the default profile:

```bash
lxc profile show default
# Show the default LXD profile.
# In your case, it is likely missing a root disk device.
```

Creating the default profile if it is absent:
```bash
lxc storage create default dir
# Create a storage pool named default for LXD container root disks.
# dir uses WEB01's existing filesystem instead of creating another loop-backed disk image.
# Your existing docker Btrfs pool will remain dedicated to /var/lib/docker inside IRIS01.
```

```bash
lxc profile device add default root disk path=/ pool=default
# Add the missing root disk device to the default profile.
# default = profile name.
# root = device name.
# disk = LXD disk-device type.
# path=/ makes this the container root filesystem.
# pool=default stores container root volumes in the default storage pool.
```

Now check whether you already have an LXD bridge network:
```bash
lxc network list
# List LXD-managed networks.
# If lxdbr0 is already listed, do not create it again.
```

Only run this next command if lxdbr0 is not listed:
```bash
lxc network create lxdbr0 ipv4.address=auto ipv4.nat=true ipv6.address=none
# Create a NATed LXD bridge network.
# ipv4.address=auto lets LXD choose a private IPv4 subnet.
# ipv4.nat=true allows containers to reach the Internet through WEB01.
# ipv6.address=none disables IPv6 for this lab bridge.
```

Check the default profile again:
```bash
lxc profile show default
# Check whether the profile already has an eth0 NIC device.
```

Only run this next command if the profile has no eth0 device:
```bash
lxc profile device add default eth0 nic name=eth0 network=lxdbr0
# Add a network interface to the default profile.
# eth0 = LXD device name.
# nic = network-interface device type.
# name=eth0 sets the interface name inside containers.
# network=lxdbr0 connects containers to the LXD bridge.
```
The profile should now contain at least a root disk and an eth0 NIC. LXD’s own profile example shows this same structure: root as a disk with path: / and a storage pool, plus eth0 as a NIC.

---

## 12. Create the IRIS01 LXD container

### 12.1 Create IRIS01 with course-aligned CPU and RAM limits

```bash
lxc list iris01
# Check whether an IRIS01 container already exists.
# If it exists from a previous attempt, either reuse it or delete it deliberately before recreating it.
```

```bash
lxc init ubuntu:24.04 iris01 -s default -c limits.cpu=4 -c limits.memory=3GiB
# Create an Ubuntu 24.04 LXD container named iris01 without starting it.
# -s default explicitly stores the root disk in the default storage pool.
# -c applies instance configuration keys.
# limits.cpu=2 caps the container at four vCPUs.
# limits.memory=4GiB caps the container memory at 3 GiB.
```

### 12.2 Attach the 25 GiB Btrfs pool to `/var/lib/docker`

```bash
lxc storage volume create docker iris01-docker
# Create a custom storage volume named iris01-docker inside the docker storage pool.
```

```bash
lxc storage volume list docker
# Confirm that iris01-docker already exists in the docker storage pool.
```

```bash
lxc config device add iris01 docker disk pool=docker source=iris01-docker path=/var/lib/docker
# Attach the existing iris01-docker custom volume to IRIS01.
# docker = LXD device name.
# disk = disk-device type.
# pool=docker selects your Btrfs storage pool.
# source=iris01-docker selects the custom volume.
# path=/var/lib/docker mounts it where Docker stores images, containers, and volumes.
```
That custom volume is separate from the root disk; adding it as a disk device is the correct mechanism for mounting a custom LXD storage volume inside a container.

### 12.3 Enable nesting options required by Docker inside LXD

```bash
lxc config set iris01 security.nesting=true security.syscalls.intercept.mknod=true security.syscalls.intercept.setxattr=true
# Enable nested container support for Docker inside IRIS01.
# security.nesting=true allows nested container behavior.
# security.syscalls.intercept.mknod=true lets LXD intercept/emulate mknod calls used by Docker layers.
# security.syscalls.intercept.setxattr=true lets LXD intercept/emulate setxattr calls used by Docker layers.
```

### 12.4 Start and inspect IRIS01

```bash
lxc start iris01
# Start the IRIS01 container.
```

```bash
lxc list iris01
# Confirm that IRIS01 is RUNNING and has an IPv4 address.
```

### 12.5 Test Ubuntu repository access, outbound networking and DNS inside IRIS01:

```bash
lxc exec iris01 -- apt update
# -- separates lxc arguments from the command executed inside the container.
```

```bash
lxc exec iris01 -- ping -c 2 192.168.52.134
# Test connectivity from IRIS01 to WEB01.
# -- separates lxc arguments from the command run inside the container.
# ping -c 2 sends two ICMP echo requests and stops.
```

```bash
lxc exec iris01 -- ping -c 2 google.com
# Test DNS and internet access
```

---

## 13. Install Docker inside IRIS01

Open a root shell inside the container:

```bash
lxc exec iris01 -- bash
# Open an interactive Bash shell inside IRIS01 as root.
# -- separates lxc arguments from the command executed inside the container.
```

The remaining commands in this section are run **inside IRIS01**.

### 13.1 Update IRIS01 and install prerequisites

```bash
apt update
# Refresh package indexes inside IRIS01.
```

```bash
apt install -y ca-certificates curl gnupg git openssh-client iputils-ping
# Install tools needed for Docker repository setup, Git clone, SSH tests, and ping.
# -y answers yes to prompts.
```

### 13.2 Remove conflicting Docker packages if any are installed

```bash
dpkg-query -W -f='${binary:Package}\n' docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc 2>/dev/null | xargs --no-run-if-empty apt remove -y
# Remove old/conflicting Docker-related packages only if they are installed.
# dpkg-query -W queries package database entries by package name.
# -f='${binary:Package}\n' prints only package names, one per line.
# 2>/dev/null hides messages for package names that are not known on this Ubuntu release.
# | sends the listed package names to xargs.
# xargs --no-run-if-empty prevents apt from running with an empty package list.
# apt remove -y removes the packages non-interactively.
```

### 13.3 Add Docker's official APT repository

```bash
install -m 0755 -d /etc/apt/keyrings
# Create Docker's APT keyring directory.
# -d creates a directory.
# -m 0755 sets standard permissions.
```

```bash
rm -f /etc/apt/keyrings/docker.asc
# Remove an old Docker key file if it exists.
# -f avoids an error if the file does not exist.
```

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
# Download Docker's official Ubuntu APT signing key.
# -f fails on HTTP errors.
# -sS keeps output quiet but still shows errors.
# -L follows redirects.
# -o writes to the specified file.
```

```bash
chmod a+r /etc/apt/keyrings/docker.asc
# Make the Docker key readable by APT.
# a+r grants read permission to all classes.
```

```bash
cat >/etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
# Create Docker's APT source file in modern .sources format.
# > replaces the target file with the here-document content.
# <<EOF starts a here-document and the final EOF line ends it.
# The command substitution reads the Ubuntu codename from /etc/os-release.
# dpkg --print-architecture inserts the container architecture, usually amd64.
```

### 13.4 Install Docker Engine and Compose plugin

```bash
apt update
# Refresh package indexes after adding Docker's repository.
```

```bash
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# Install Docker Engine, Docker CLI, containerd, Buildx, and Docker Compose plugin.
# -y answers yes to prompts.
```

```bash
systemctl status docker --no-pager
# Confirm Docker daemon status.
# --no-pager prints directly to the terminal.
```

```bash
docker version
# Print Docker client and server version information.
```

```bash
docker compose version
# Print the Docker Compose plugin version.
```

```bash
docker run hello-world
# Pull and run Docker's hello-world image to verify Docker works inside IRIS01.
```

---

## 14. Install DFIR-IRIS inside IRIS01

Continue running commands **inside IRIS01**.

### 14.1 Clone the DFIR-IRIS repository

```bash
cd /opt
# Change to /opt, a common location for manually installed applications.
```

```bash
git clone https://github.com/dfir-iris/iris-web.git
# Clone the DFIR-IRIS iris-web repository.
```

```bash
cd /opt/iris-web
# Enter the cloned DFIR-IRIS project directory.
```

### 14.2 Check out a stable tagged version

```bash
git checkout v2.4.27
# Switch to the stable tagged DFIR-IRIS release used by the current quick-start documentation.
```

```bash
git status --short --branch
# Confirm the repository is on the selected tag.
# --short prints compact status.
# --branch includes branch/tag information.
```

### 14.3 Prepare the environment file

```bash
cp .env.model .env
# Create the working environment file from the project template.
```

**[OPTIONAL — lab convenience]** Set a known initial IRIS administrator password before the first start.

```bash
printf '\nIRIS_ADM_PASSWORD=Google2026\n' >> .env
# Append a lab-only initial administrator password to the IRIS environment file.
# This affects only the first administrator creation.
# printf prints exactly the requested line with a leading newline.
# >> appends to .env instead of replacing it.
```

### 14.4 Pull and start IRIS containers

```bash
docker compose pull
# Pull the container images required by DFIR-IRIS.
```

```bash
docker compose up -d
# Start DFIR-IRIS containers in detached/background mode.
# -d means detached.
```

```bash
docker compose ps
# Show the running DFIR-IRIS Docker services.
```

**[OPTIONAL — only if no fixed IRIS admin password was configured]** Run this only if you did not set `IRIS_ADM_PASSWORD` before the first start:

```bash
docker compose logs app | grep "WARNING :: post_init :: create_safe_admin"
# Search application logs for the generated administrator password line.
# grep filters log output to matching lines.
```

Exit the IRIS01 shell when installation is complete:

```bash
exit
# Leave the IRIS01 container shell and return to WEB01.
```

### 14.5 Test IRIS from WEB01

```bash
lxc list iris01
# Show IRIS01's IP address.
```

```bash
export IRIS_IP="$(lxc list iris01 -c 4 --format csv | awk '{print $1}')"
# Store the first IRIS01 IP address in IRIS_IP.
# -c 4 prints only the IPv4/IPv6 address column.
# --format csv removes the table formatting.
# awk '{print $1}' keeps only the first address if multiple values are shown.
```

```bash
printf '%s\n' "$IRIS_IP"
# Print the detected IRIS01 IP address.
```

```bash
curl -k -I "https://${IRIS_IP}"
# Test the IRIS HTTPS endpoint from WEB01.
# -k allows the self-signed certificate.
# -I requests only HTTP headers.
# Quoting the URL protects against empty or unexpected variable content.
```

Open the IRIS web interface from a browser inside the Ubuntu VM using the printed IP address:

```text
https://IRIS_IP_FROM_THE_COMMAND_ABOVE
```

Default username is normally:

```text
administrator
```

Use `Google2026` if you appended `IRIS_ADM_PASSWORD=Google2026` before first start. Otherwise use the generated password found in the container logs.

---

## 15. Generate reconnaissance traffic: choose one attack source

You need one source that is not the normal WEB01 desktop shell to generate scan and web-access evidence.

Use **Path A** as the default all-Ubuntu implementation. Use **Path B** only if you also deploy Kali in the same isolated VMware network.

| Path | Status | Use when |
|---|---|---|
| Path A — ATTACKER01 LXD container | Default all-Ubuntu path | You want the lab to stay inside the Ubuntu 24.04.4 VM |
| Path B — Kali VM | **[OPTIONAL — Kali VM]** | You want to match the screenshot literally with Kali, Zenmap, and Nikto |

Run only one path unless you intentionally want extra log evidence from both sources.

---

## 16. Path A — default all-Ubuntu attack source: ATTACKER01 LXD container

Run this section on **WEB01** if you are not using Kali.

### 16.1 Create ATTACKER01

```bash
lxc list attacker01
# Check whether an ATTACKER01 container already exists.
# If it exists from a previous attempt, either reuse it or delete it deliberately before recreating it.
```

```bash
lxc launch ubuntu:24.04 attacker01
# Create and start an Ubuntu 24.04 LXD container named attacker01.
# launch combines image download, instance creation, and start.
```

```bash
lxc exec attacker01 -- apt update
# Refresh package indexes inside ATTACKER01.
# -- separates lxc arguments from the command run inside the container.
```

```bash
lxc exec attacker01 -- apt install -y nmap nikto curl openssh-client iputils-ping
# Install attacker-side tools inside ATTACKER01.
# -y answers yes to prompts.
# nmap performs network/service scanning.
# nikto performs web-server checks.
# curl retrieves exposed web files.
# openssh-client provides ssh/scp client tools.
# iputils-ping provides ping for connectivity testing.
```

### 16.2 Verify ATTACKER01 can reach WEB01

```bash
lxc exec attacker01 -- ping -c 2 192.168.52.134
# Send two ICMP echo requests from ATTACKER01 to WEB01.
# -c 2 stops after two packets.
# Replace 192.168.52.134 if WEB01 uses another lab IP.
```

### 16.3 Run Nmap from ATTACKER01

```bash
lxc exec attacker01 -- nmap -Pn -sV 192.168.52.134
# Scan WEB01 from ATTACKER01 and detect service versions.
# -Pn skips host discovery and treats the host as online.
# -sV probes open ports to identify service versions.
```

```bash
lxc exec attacker01 -- nmap -Pn -p 22,80 -sV 192.168.52.134
# Focus the scan on SSH and HTTP.
# -p 22,80 restricts the scan to ports 22 and 80.
# -sV performs service-version detection.
```

### 16.4 Run Nikto from ATTACKER01

```bash
lxc exec attacker01 -- nikto -h http://192.168.52.134
# Run Nikto against Apache on WEB01.
# -h specifies the target host or URL.
```

### 16.5 Retrieve exposed files from ATTACKER01

```bash
lxc exec attacker01 -- curl http://192.168.52.134/backup.sh
# Retrieve the exposed backup script from WEB01.
```

```bash
lxc exec attacker01 -- curl http://192.168.52.134/.env
# Retrieve the exposed credential file from WEB01.
# This should reveal the lab password Google2026.
```

---

## 17. **[OPTIONAL — Kali VM]** Path B — Kali attack source in the same network

Run this section only if you deploy a Kali VM named **KALI01** in the same isolated VMware network as WEB01.

### 17.1 Configure Kali networking

Run on **KALI01**:

```bash
ip -br address
# Show Kali network interfaces and assigned IP addresses in brief format.
# -br means brief output.
```

**[CONDITIONAL — only if Kali does not already have a lab-network IP]** Add a temporary static address.

```bash
sudo ip address add 192.168.56.20/24 dev <lab_nic>
# Add a temporary IP address to the selected Kali lab interface.
# Replace <lab_nic> with the actual interface name.
# /24 sets the subnet mask to 255.255.255.0.
```

```bash
sudo ip link set <lab_nic> up
# Bring the selected Kali lab interface up.
# Replace <lab_nic> with the actual interface name.
```

### 17.2 Install required Kali tools

```bash
sudo apt update
# Refresh Kali package indexes.
```

```bash
sudo apt install -y nmap zenmap nikto firefox-esr
# Install Nmap, Zenmap GUI, Nikto, and Firefox ESR.
# -y answers yes to prompts.
```

### 17.3 Verify Kali can reach WEB01

```bash
ping -c 2 192.168.52.134
# Send two ICMP echo requests to WEB01.
# -c 2 stops after two packets.
```

### 17.4 Run Nmap or Zenmap from Kali

CLI Nmap scan:

```bash
nmap -Pn -sV 192.168.52.134
# Scan WEB01 and detect service versions.
# -Pn skips host discovery and treats the host as online.
# -sV probes open ports to identify service versions.
```

Focused CLI scan:

```bash
nmap -Pn -p 22,80 -sV 192.168.52.134
# Scan only SSH and HTTP.
# -p 22,80 restricts the scan to ports 22 and 80.
# -sV performs service-version detection.
```

Zenmap GUI equivalent:

```bash
zenmap
# Start the Zenmap GUI from Kali.
```

In Zenmap, set the target to `192.168.52.134` and use a command equivalent to:

```text
nmap -Pn -sV 192.168.52.134
```

### 17.5 Run Nikto from Kali

```bash
nikto -h http://192.168.52.134
# Scan the Apache web server for common web weaknesses.
# -h specifies the target host or URL.
```

### 17.6 Retrieve exposed files from Kali

```bash
curl http://192.168.52.134/backup.sh
# Retrieve the exposed backup script from WEB01.
```

```bash
curl http://192.168.52.134/.env
# Retrieve the exposed .env file from WEB01.
# This reveals the lab password Google2026.
```

---

## 18. Simulate SSH activity from IRIS01 to WEB01

This step matches the screenshot instruction to connect from the IRIS LXC container to the Apache server over SSH and generate failed plus successful login evidence.

Run on WEB01:

```bash
lxc exec iris01 -- ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no kiberacs@192.168.52.134
# Start an SSH login from IRIS01 to WEB01 as kiberacs.
# -o PreferredAuthentications=password asks SSH to prefer password authentication.
# -o PubkeyAuthentication=no disables key authentication for this attempt.
```

At the prompt:

1. Accept the host key if asked.
2. Enter a wrong password two or three times.
3. Start the SSH command again if the session closes.
4. Enter the correct password: `Google2026`.

This should generate failed and successful SSH evidence in `/var/log/auth.log` on WEB01.

---

## 19. Simulate sudo activity after successful SSH login

After you successfully SSH into WEB01 from IRIS01 as `kiberacs`, run the commands below **one at a time** in that SSH session.

```bash
sudo -i
# Start an interactive root shell through sudo.
# -i simulates an initial login shell as root and should trigger sudo logging and I/O recording.
```

```bash
echo "SUPER SECRET DATA" > /home/kiberacs/forensic-test.txt
# Create a forensic test file in the kiberacs home directory.
```

```bash
whoami
# Confirm the current user context.
```

```bash
cd /home/kiberacs
# Change to the kiberacs home directory.
```

```bash
cat forensic-test.txt
# Read the test file to create command history and audit activity.
```

```bash
rm forensic-test.txt
# Delete the test file to trigger file deletion audit events.
```

```bash
touch hacked.txt
# Create a visible compromise marker file.
```

```bash
echo "hacked" > hacked.txt
# Write content into the marker file.
```

```bash
cp /var/www/html/index.html /var/www/html/index.html.before-compromise
# Save a copy of the course-provided Apache page before modifying it.
# This command is run from the root shell opened through sudo, so no sudo prefix is needed here.
```

```bash
sed -i '1i<!-- LAB COMPROMISE MARKER -->' /var/www/html/index.html
# Insert a compromise marker as the first line of index.html.
# -i edits the file in place.
# 1i inserts text before line 1.
```

```bash
exit
# Leave the root shell and return to the kiberacs SSH session.
```

```bash
exit
# Close the SSH session and return to IRIS01 or WEB01, depending on where your terminal is.
```

---

## 20. Verify Apache `index.html` modification

Run these verification commands on **WEB01** after the SSH/sudo session from the previous section has modified the file.

```bash
head -n 5 /var/www/html/index.html
# Show the first five lines of the Apache page on disk.
# The compromise marker should appear on the first line.
```

```bash
curl http://127.0.0.1 | head -n 5
# Confirm the modified page is served locally.
# head -n 5 shows only the first five lines.
```

```bash
curl http://192.168.52.134 | head -n 5
# Confirm the modified page is served on the lab IP.
# Replace the IP if your WEB01 address is different.
```

---

## 21. Validate evidence on WEB01

### 21.1 Apache access log

```bash
sudo tail -n 80 /var/log/apache2/access.log
# Show the last 80 Apache access log lines.
# -n 80 limits output to 80 lines.
```

Useful filter:

```bash
sudo grep -E 'backup\.sh|\.env|Nikto|nmap|GET / ' /var/log/apache2/access.log | tail -n 80
# Filter Apache access logs for backup.sh, .env, Nikto, nmap, and root-page requests.
# grep -E enables extended regular expressions.
# tail -n 80 shows the most recent 80 matching lines.
```

### 21.2 SSH and authentication log

```bash
sudo tail -n 100 /var/log/auth.log
# Show the most recent authentication events.
# -n 100 limits output to 100 lines.
```

Useful filter:

```bash
sudo grep -Ei 'failed password|accepted password|invalid user|session opened|session closed|sudo' /var/log/auth.log | tail -n 100
# Filter authentication logs for failed logins, accepted logins, sessions, and sudo.
# -E enables extended regex.
# -i makes matching case-insensitive.
```

### 21.3 Sudo log and I/O replay

```bash
sudo tail -n 80 /var/log/sudo.log
# Show the most recent sudo command log entries.
```

```bash
sudo sudoreplay -l
# List available sudo I/O replay sessions.
# -l lists recorded sessions instead of replaying one.
```

```bash
sudo sudoreplay <session_id>
# Replay a selected sudo I/O session.
# Replace <session_id> with an ID shown by `sudo sudoreplay -l`.
```

### 21.4 Auditd searches

```bash
sudo ausearch -k webroot -ts recent -i
# Search recent events related to the webroot watch.
```

```bash
sudo ausearch -k web_write -ts recent -i
# Search recent write events under /var/www/html.
```

```bash
sudo ausearch -k file_delete -ts recent -i
# Search recent file deletion events.
```

```bash
sudo ausearch -k sudo_cmd -ts recent -i
# Search recent executions of the sudo binary.
```

```bash
sudo ausearch -k cmd_exec -ts recent -i | tail -n 120
# Search recent command execution audit records and show the last 120 lines.
```

```bash
sudo aureport -x --summary
# Summarize executed programs from audit data.
# -x reports executable activity.
# --summary groups the output.
```

### 21.5 Bash history

Bash history is flushed most reliably after the shell exits. If files are empty, close the SSH/root shells and check again.

```bash
sudo tail -n 80 /home/kiberacs/.bash_history
# Show recent kiberacs Bash history entries.
```

```bash
sudo tail -n 80 /root/.bash_history
# Show recent root Bash history entries.
```

### 21.6 Osquery spot checks

```bash
sudo osqueryi "SELECT * FROM last WHERE username='kiberacs' ORDER BY time DESC LIMIT 10;"
# Query recent login records for kiberacs.
# ORDER BY time DESC sorts newest first.
# LIMIT 10 keeps output short.
```

```bash
sudo osqueryi "SELECT pid, port, protocol, address FROM listening_ports WHERE port IN (22,80);"
# Confirm SSH and HTTP listeners.
```

```bash
sudo osqueryi "SELECT id, active_state, sub_state FROM systemd_units WHERE id IN ('ssh.service','apache2.service','auditd.service','rsyslog.service');"
# Confirm key service states through osquery.
```

---

## 22. Minimal DFIR-IRIS use for this technical phase

Do this through the IRIS web interface after confirming the platform works.

1. Create one case for the simulated Apache compromise.
2. Add asset `WEB01`.
3. Add IP `192.168.52.134` or your actual WEB01 IP.
4. Add notes for these events:
   - Nmap or Zenmap scan.
   - Nikto scan.
   - HTTP access to `/backup.sh`.
   - HTTP access to `/.env`.
   - Failed SSH attempts from IRIS01.
   - Successful SSH login as `kiberacs`.
   - `sudo -i` activity.
   - `forensic-test.txt` creation/deletion.
   - `hacked.txt` creation.
   - `index.html` modification.
5. Keep detailed screenshots for the later report phase.

---

## 23. **[OPTIONAL]** Rollback and cleanup

Run these only after the lab evidence has been collected.

### 23.1 Restore the original Apache page

```bash
sudo cp /var/www/html/index.html.orig /var/www/html/index.html
# Restore the original Apache default page saved earlier.
```

### 23.2 Remove intentionally exposed files

```bash
sudo rm -f /var/www/html/backup.sh /var/www/html/.env
# Remove the intentionally exposed lab files.
# -f avoids errors if a file is already absent.
```

### 23.3 Stop containers

```bash
lxc stop iris01
# Stop the DFIR-IRIS LXD container.
```

```bash
lxc stop attacker01
# Stop the optional Ubuntu attacker container if you created it.
```

### 23.4 Restart containers later

```bash
lxc start iris01
# Start the DFIR-IRIS LXD container again.
```

```bash
lxc start attacker01
# Start the optional Ubuntu attacker container again.
```

---

## 24. Troubleshooting checks

### 24.1 SSH password login does not work

```bash
sudo /usr/sbin/sshd -T | grep -Ei 'passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|loglevel'
# Show effective SSH authentication and logging settings.
# -T prints effective configuration.
# grep -E uses extended regex; -i ignores case.
```

```bash
sudo tail -n 80 /var/log/auth.log
# Review recent SSH/authentication errors.
```

### 24.2 Audit rules did not load as expected

```bash
sudo augenrules --load
# Reload persistent audit rules.
```

```bash
sudo auditctl -l | grep -E 'webroot|cmd_exec|sudo_cmd|file_delete'
# Confirm important rules are active.
# grep -E searches for multiple rule keys.
```

```bash
sudo auditctl -s
# Check audit enabled state, backlog, and lost event counters.
```

### 24.3 Docker fails inside IRIS01

Run on WEB01:

```bash
lxc config show iris01 --expanded | grep -E 'security.nesting|intercept.mknod|intercept.setxattr|limits.cpu|limits.memory'
# Confirm IRIS01 has the required nested Docker LXD settings.
# --expanded includes profile-derived settings.
```

```bash
lxc exec iris01 -- df -h /var/lib/docker
# Confirm /var/lib/docker is mounted and has available space inside IRIS01.
# df -h shows filesystem usage in human-readable units.
```

Run inside IRIS01 if Docker still fails:

```bash
systemctl status docker --no-pager
# Show Docker daemon status inside IRIS01.
```

```bash
journalctl -u docker -n 100 --no-pager
# Show the latest 100 Docker service log lines.
# -u docker filters to the Docker unit.
# -n 100 limits output.
```

---

## 25. Source references used for version-sensitive steps

These references are for the technical implementation choices in this guide:

- DFIR-IRIS Quick Start: `https://docs.dfir-iris.org/latest/getting_started/`
- Docker Engine on Ubuntu: `https://docs.docker.com/engine/install/ubuntu/`
- LXD installation: `https://documentation.ubuntu.com/lxd/latest/installing/`
- Docker inside LXD containers: `https://ubuntu.com/tutorials/how-to-run-docker-inside-lxd-containers`
- osquery Linux installation notes: `https://osquery.readthedocs.io/en/stable/installation/install-linux/`
- Kali nmap / zenmap package reference: `https://www.kali.org/tools/nmap/`
- Kali nikto package reference: `https://www.kali.org/tools/nikto/`
