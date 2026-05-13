# Incident Lab - VM Safe Shutdown and Restart Manual

This manual is for pausing the incident lab after completing steps **#1–#14** and continuing later from step **#15**.

**Main system:** Ubuntu 24.04.4 LTS VM, referred to as **WEB01**.  
**LXD container:** `iris01`, running DFIR-IRIS through Docker Compose.  
**Optional LXD container:** `attacker01`, only if you created it later or outside the guide.  
**Optional VM:** Kali Linux VM, only if you are using it.

The safest approach is to stop the DFIR-IRIS Docker containers gracefully, stop the LXD container, and then shut down Ubuntu normally. This preserves disk state and avoids an abrupt nested Docker shutdown.

---

## 1. Important safety notes

A powered-off VM cannot keep containers running in memory. Safe shutdown means:

1. DFIR-IRIS services are stopped cleanly.
2. The `iris01` LXD container is stopped cleanly.
3. Ubuntu is powered off through the guest OS, not by VMware forced power-off.
4. Later, Ubuntu, `iris01`, and DFIR-IRIS are started again.

$$\color{red}{\text{**Avoid these commands unless you intentionally want to remove lab data!**}}$$


```bash
docker compose down -v
# Do NOT use this for a normal break.
# down removes Compose containers and networks.
# -v also removes named volumes, which can destroy application data.
```

```bash
lxc delete iris01
# Do NOT use this for a normal break.
# This deletes the IRIS01 LXD container.
```

```bash
lxc storage volume delete docker iris01-docker
# Do NOT use this for a normal break.
# This deletes the Docker storage volume attached to IRIS01.
```

In VMware, avoid **Power Off** or **Reset** for a normal break. Use Ubuntu shutdown, or use VMware **Suspend** only as an optional short-term pause method.

---

## 2. Recommended shutdown procedure

Run this section from **WEB01**, not from inside `iris01`.

### 2.1 Record the current break point

```bash
date -Is
# Print the current timestamp in ISO-8601 format.
# -I selects ISO date output.
# s includes seconds.
```

```bash
hostname
# Print the current system hostname.
# Expected value in this lab: web01.
```

```bash
lxc list
# List all LXD instances and their current state.
# Use this to confirm whether iris01 is RUNNING before stopping it.
```

### 2.2 Check the DFIR-IRIS Docker Compose stack

```bash
lxc exec iris01 -- bash -lc 'cd /opt/iris-web && docker compose ps'
# Run a command inside the iris01 LXD container.
# exec executes a command in the container.
# -- separates LXD arguments from the command executed inside the container.
# bash -lc runs the quoted command through Bash as a login-style command string.
# cd /opt/iris-web enters the DFIR-IRIS project directory.
# && runs docker compose ps only if cd succeeds.
# docker compose ps shows the DFIR-IRIS Compose services and their status.
```

If this command fails because `iris01` is already stopped, skip to section **2.4**.

### 2.3 Stop DFIR-IRIS gracefully inside IRIS01

```bash
lxc exec iris01 -- bash -lc 'cd /opt/iris-web && docker compose stop --timeout 60'
# Stop the DFIR-IRIS Docker Compose containers without deleting them.
# docker compose stop stops existing containers but preserves containers, images, networks, and volumes.
# --timeout 60 gives containers up to 60 seconds to stop cleanly before Docker forces them to stop.
```

```bash
lxc exec iris01 -- bash -lc 'cd /opt/iris-web && docker compose ps'
# Verify the DFIR-IRIS containers are no longer running.
# Some services may show Exited, which is expected after docker compose stop.
```

### 2.4 Stop the IRIS01 LXD container

```bash
lxc stop iris01 --timeout 120
# Stop the iris01 LXD container gracefully.
# --timeout 120 gives the container up to 120 seconds to shut down before LXD forces the stop.
```

```bash
lxc list iris01
# Confirm that iris01 is now STOPPED.
```

### 2.5 [OPTIONAL] Stop ATTACKER01 if it exists

Run this only if you have already created the all-Ubuntu attacker container named `attacker01`.

```bash
lxc list attacker01
# Check whether attacker01 exists and whether it is running.
```

```bash
lxc stop attacker01 --timeout 60
# Stop attacker01 gracefully.
# --timeout 60 gives the container up to 60 seconds to shut down before LXD forces the stop.
```

If `attacker01` does not exist, LXD will report that the instance was not found. That is harmless; continue with the shutdown.

### 2.6 Check host services before powering off

```bash
systemctl is-active apache2 ssh auditd rsyslog cron
# Print the active/inactive state of important WEB01 services.
# apache2 = web server.
# ssh = SSH server.
# auditd = Linux audit daemon.
# rsyslog = system log daemon.
# cron = scheduled task daemon.
```

This command may print multiple lines. `active` means the service is running. The command can return a non-zero status if any service is inactive; that does not stop you from shutting down.

### 2.7 Flush filesystem buffers

```bash
sync
# Flush pending filesystem writes to disk.
# This is a simple extra safety step before shutting down the VM.
```

### 2.8 Power off Ubuntu cleanly

```bash
sudo systemctl poweroff
# Ask systemd to shut down and power off Ubuntu cleanly.
# sudo runs the command with administrative privileges.
# systemctl controls systemd.
# poweroff stops services, unmounts filesystems, and powers off the VM.
```

Wait until VMware shows the Ubuntu VM as powered off.

---

## 3. [OPTIONAL] Take a VMware snapshot after shutdown

After Ubuntu is fully powered off, a VMware snapshot is a useful recovery point.

Suggested snapshot name:

```text
02-after-iris-configured-before-attack
```

Suggested snapshot description:

```text
Steps #1-#14 completed successfully. Apache, auditd, sudo logging, bash logging, osquery, LXD, IRIS01, Docker, and DFIR-IRIS are configured. Ready to continue with step #15.
```

Take the snapshot from the VMware interface, not from inside Ubuntu.

---

## 4. [OPTIONAL] VMware Suspend instead of shutdown

Use this only for a short break if you want to preserve the exact memory state of the VM.

Advantages:

- Faster resume.
- Containers and terminals usually resume where they were.

Disadvantages:

- Less clean than a real shutdown.
- Time jumps can confuse logs and sessions.
- Network leases or routes may need to refresh after resume.
- Not recommended before important evidence-generation steps.

If you use VMware Suspend, still check the lab after resuming with the validation commands in section **6**.

---

## 5. Restart procedure after the break

Power on the Ubuntu VM from VMware, log in to **WEB01**, and run the following commands from WEB01.

### 5.1 Confirm you are on WEB01

```bash
hostname
# Print the current hostname.
# Expected value in this lab: web01.
```

```bash
lsb_release -a
# Print Ubuntu release information.
# -a shows all available distribution fields.
```

### 5.2 Check core WEB01 services

```bash
systemctl is-active apache2 ssh auditd rsyslog cron
# Check whether important host services are active after boot.
# apache2 = web server.
# ssh = SSH server.
# auditd = Linux audit daemon.
# rsyslog = system log daemon.
# cron = scheduled task daemon.
```

If a service is inactive, start it with the matching command below.

```bash
sudo systemctl start apache2
# Start the Apache web server if it is not active.
```

```bash
sudo systemctl start ssh
# Start the SSH server if it is not active.
```

```bash
sudo systemctl start auditd
# Start the audit daemon if it is not active.
```

```bash
sudo systemctl start rsyslog
# Start the rsyslog daemon if it is not active.
```

```bash
sudo systemctl start cron
# Start the cron daemon if it is not active.
```

### 5.3 Confirm LXD is available

```bash
lxc storage list
# List LXD storage pools.
# This verifies that the LXD client can communicate with the LXD daemon.
```

```bash
lxc list
# List LXD instances.
# After a clean shutdown, iris01 is usually STOPPED unless autostart was configured.
```

### 5.4 Start IRIS01

```bash
lxc start iris01
# Start the iris01 LXD container.
```

If LXD says the instance is already running, that is fine; continue.

```bash
lxc list iris01
# Confirm iris01 is RUNNING.
# The eth0 address is the LXD container address used to access DFIR-IRIS from WEB01.
```

### 5.5 Wait for IRIS01 networking

```bash
lxc exec iris01 -- ip -4 -o addr show dev eth0
# Show the IPv4 address assigned to iris01 on eth0.
# -4 limits output to IPv4.
# -o prints one-line output.
# addr show dev eth0 shows addresses only for eth0.
```

If this prints no IPv4 address, wait 10-20 seconds and run the same command again.

### 5.6 Check Docker inside IRIS01

```bash
lxc exec iris01 -- systemctl is-active docker
# Check whether the Docker service is active inside iris01.
```

If Docker is not active, start it:

```bash
lxc exec iris01 -- systemctl start docker
# Start Docker inside iris01.
```

### 5.7 Start DFIR-IRIS again

```bash
lxc exec iris01 -- bash -lc 'cd /opt/iris-web && docker compose up -d'
# Start the DFIR-IRIS Docker Compose stack inside iris01.
# bash -lc runs the quoted command through Bash.
# cd /opt/iris-web enters the DFIR-IRIS project directory.
# && runs docker compose up only if cd succeeds.
# up creates missing containers if needed and starts existing containers.
# -d runs the containers in detached/background mode.
```

```bash
lxc exec iris01 -- bash -lc 'cd /opt/iris-web && docker compose ps'
# Show the current DFIR-IRIS container status.
# Confirm that the main IRIS services are running or healthy.
```

### 5.8 Store the current IRIS01 eth0 IP address

```bash
export IRIS_IP="$(lxc exec iris01 -- ip -4 -o addr show dev eth0 | awk '{split($4,a,"/"); print a[1]}')"
# Store the current iris01 eth0 IPv4 address in the shell variable IRIS_IP.
# lxc exec iris01 -- runs the following command inside iris01.
# ip -4 -o addr show dev eth0 prints the IPv4 address on eth0 in one-line format.
# awk processes the fourth field, which contains address/prefix.
# split($4,a,"/") splits the address from the subnet prefix.
# print a[1] prints only the IPv4 address.
```

```bash
printf '%s\n' "$IRIS_IP"
# Print the detected IRIS01 IP address.
# Use this IP for browser access to DFIR-IRIS from WEB01.
```

### 5.9 Test DFIR-IRIS from WEB01

```bash
curl -k -I "https://${IRIS_IP}"
# Test the DFIR-IRIS HTTPS endpoint.
# -k allows curl to connect even if IRIS uses a self-signed lab certificate.
# -I requests only HTTP response headers.
```

A response with an HTTP status such as `200`, `301`, `302`, `401`, or `403` means the IRIS web service is responding. A browser login page should be available at:

```text
https://<IRIS_IP>
```

For example, if the IP is still `x.x.x.x`:

```text
https://x.x.x.x
```

Do not use Docker bridge addresses such as `172.x.x.x`, `172.x.x.x`, or `172.x.x.x` for browser access.

---

## 6. Validation checklist before continuing to step #15

Run these checks from **WEB01**.

### 6.1 Apache target check

```bash
curl -I http://127.0.0.1/
# Test the Apache web server locally on WEB01.
# -I requests only HTTP response headers.
```

```bash
curl -I http://127.0.0.1/backup.sh
# Confirm that the intentionally exposed backup script is reachable over HTTP.
# -I requests only HTTP response headers.
```

```bash
curl -I http://127.0.0.1/.env
# Confirm that the intentionally exposed .env file is reachable over HTTP.
# -I requests only HTTP response headers.
```

### 6.2 SSH check

```bash
systemctl is-active ssh
# Confirm that the SSH server is active on WEB01.
```

```bash
ss -tulpn | grep ':22'
# Show listening processes on TCP/UDP sockets and filter for port 22.
# -t shows TCP sockets.
# -u shows UDP sockets.
# -l shows listening sockets.
# -p shows the owning process.
# -n shows numeric addresses and ports.
# grep ':22' filters for SSH port 22.
```

### 6.3 Audit logging check

```bash
sudo auditctl -s
# Show audit subsystem status.
# sudo is required because audit status is privileged.
# -s prints status information.
```

```bash
sudo auditctl -l
# List loaded audit rules.
# -l lists current rules.
```

### 6.4 Bash and sudo logging check

```bash
tail -n 5 ~/.bash_history
# Show the last five commands from the current user's Bash history file.
# -n 5 limits output to five lines.
```

```bash
sudo ls /var/log/sudo-io
# List the sudo I/O logging directory if sudo I/O logging is enabled.
```

### 6.5 DFIR-IRIS check

```bash
lxc exec iris01 -- bash -lc 'cd /opt/iris-web && docker compose ps'
# Confirm that the DFIR-IRIS Docker Compose stack is running after restart.
```

```bash
curl -k -I "https://${IRIS_IP}"
# Confirm that the DFIR-IRIS web endpoint still responds.
# -k allows a self-signed certificate.
# -I requests only HTTP response headers.
```

When these checks pass, continue with step **#15** in the main lab guide.

---

## 7. [OPTIONAL] Enable IRIS01 autostart on future WEB01 boots

Manual start is safer while learning because you see every step. Use this optional section only after you have successfully tested the manual restart procedure above.

```bash
lxc config set iris01 boot.autostart true
# Configure iris01 to start automatically when LXD starts.
# boot.autostart=true enables automatic startup for this instance.
```

```bash
lxc config set iris01 boot.autostart.delay 10
# Add a 10-second delay after starting iris01 before LXD starts the next autostart instance.
# This is useful when multiple containers exist.
```

```bash
lxc config show iris01 | grep -A5 'boot.autostart'
# Display the autostart-related configuration for iris01.
# grep filters the output to the boot.autostart lines and a few following lines.
# -A5 prints five lines after a match.
```

This only autostarts the LXD container. It does not guarantee that the DFIR-IRIS Docker Compose stack starts unless Docker or a systemd unit inside `iris01` starts it. The recommended approach for this lab is still to run:

```bash
lxc exec iris01 -- bash -lc 'cd /opt/iris-web && docker compose up -d'
# Manually start or confirm the DFIR-IRIS Docker Compose stack after WEB01 boots.
```

---

## 8. Quick command summary

### Shutdown summary

```bash
lxc exec iris01 -- bash -lc 'cd /opt/iris-web && docker compose stop --timeout 60'
# Stop DFIR-IRIS Docker Compose containers cleanly.
```

```bash
lxc stop iris01 --timeout 120
# Stop the IRIS01 LXD container cleanly.
```

```bash
sync
# Flush pending writes to disk.
```

```bash
sudo systemctl poweroff
# Power off Ubuntu cleanly.
```

### Startup summary

```bash
lxc start iris01
# Start the IRIS01 LXD container.
```

```bash
lxc exec iris01 -- bash -lc 'cd /opt/iris-web && docker compose up -d'
# Start DFIR-IRIS Docker Compose services.
```

```bash
export IRIS_IP="$(lxc exec iris01 -- ip -4 -o addr show dev eth0 | awk '{split($4,a,"/"); print a[1]}')"
# Store the current IRIS01 eth0 IP address.
```

```bash
curl -k -I "https://${IRIS_IP}"
# Verify that DFIR-IRIS is reachable.
```
