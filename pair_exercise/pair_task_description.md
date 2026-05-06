# Kali Pair Exercise Guide - Digital Forensics 

**Lab model:** each student prepares their own Kali VM as **Role A**, then students exchange VMs and perform **Role B** on the other student's VM.

**Scope:** isolated training VMs only. Do not use these activities on systems where you do not have written permission.

## General corrections applied (when compared with the original)

- Broken combined commands are split into separate commands.
- Task 3 no longer depends on `zerobank.vip`; it uses a local loopback HTTP capture.
- Kali/Debian package names are corrected, for example `libimage-exiftool-perl` for the `exiftool` command.
- Bash history is made reliable by forcing Bash and saving with `history -a`.
- `/var/tmp` is used for backup evidence that should survive VM exchange better than `/tmp`.
- Dotfile searches avoid broad exclude filters that can hide the target file.
- Suspicious process analysis is corrected: for shell scripts, `/proc/<PID>/exe` points to `bash`, so copy the script file itself.
- Linux memory forensics includes the Volatility 3 symbol-table caveat, a venv-based Volatility 3 install path, and a minimum `strings` triage path.

## Common notation

- Replace `<student_user>` with the real Linux username.
- Type commands one at a time; do not paste whole task blocks blindly.
- Leave running services/processes active until Role B documents them.
- Hash evidence files with `sha256sum <file>`.
- In Role B sections that investigate a user's home directory, set `TARGET_HOME="/home/<student_user>"`; use `TARGET_HOME="$HOME"` only if you are logged in as the account being investigated.

---

# Role A activities

## A1. Encrypted ZIP disguised as JPEG

**Goal:** create an encrypted ZIP, rename it to look like a JPEG, and delete the plaintext.

```bash
sudo apt update
sudo apt install -y zip p7zip-full
mkdir -p ~/task1
cd ~/task1
printf '%s\n' 'Meeting place: Brivibas iela 32, 21:00' > secret.txt
zip -e archive.zip secret.txt
mv archive.zip vacation_photo.jpg
mkdir -p ~/Pictures/old_2019
mv vacation_photo.jpg ~/Pictures/old_2019/
ls -la ~/Pictures/old_2019/
file ~/Pictures/old_2019/vacation_photo.jpg
rm ~/task1/secret.txt
rmdir ~/task1
```

Use a weak 3-digit ZIP password, for example `742`, but do not disclose it until debrief.

**Command/flag notes:** `apt update` refreshes package indexes; `apt install -y` installs and auto-confirms; `mkdir -p` creates parents; `printf` writes exact text; `zip -e` enables ZIP encryption; `file` checks content signatures, not extensions; `rm` removes the plaintext.

---

## A2. Multi-level steganography case

**Goal:** hide text in a JPEG, add a metadata password hint, generate an audio spectrogram password, and append an encrypted ZIP to the JPEG.

```bash
sudo apt update
sudo apt install -y zip steghide audacity libimage-exiftool-perl binwalk imagemagick fonts-dejavu-core sox git python3-pil
mkdir -p ~/task2
cd ~/task2
convert -size 1024x768 xc:lightblue -gravity center -pointsize 48 -annotate +0+0 'Vacation 2024' vacation_photo.jpg
printf '%s\n' 'Meeting: 15.03.2024, 22:00, Riga port, warehouse 7B' > secret.txt
steghide embed -cf vacation_photo.jpg -ef secret.txt -p 'Saulgriezi2024'
exiftool -overwrite_original -Comment='Remember the summer day when we celebrated Saulgriezus in 2024' vacation_photo.jpg
exiftool -overwrite_original -Artist='J.B.' -Copyright='Private collection' vacation_photo.jpg
convert -size 600x100 xc:black -font DejaVu-Sans -pointsize 48 -fill white -gravity center -annotate +0+0 'PASSWORD: TIGER42' message.bmp
rm -rf ~/spectrology
git clone https://github.com/solusipse/spectrology.git ~/spectrology
sed -i 's/data.tostring()/data.tobytes()/g' ~/spectrology/spectrology.py
python3 ~/spectrology/spectrology.py message.bmp -o hidden_message.wav -b 800 -t 8000 -p 30
sox -n -r 44100 -c 1 birds_intro.wav synth 3 sine 700 vol 0.02
sox birds_intro.wav hidden_message.wav birds_singing.wav
printf '%s\n' 'Account number: LV80HABA0551048436529' > final.txt
zip -e hidden_archive.zip final.txt
cat vacation_photo.jpg hidden_archive.zip > vacation_photo_final.jpg
mv vacation_photo_final.jpg vacation_photo.jpg
printf '%s\n' 'Case 2024 stego: analyze JPEG metadata/stego, audio spectrogram, and appended ZIP.' > readme.txt
zip case_2024_steg.zip vacation_photo.jpg birds_singing.wav readme.txt
sha256sum case_2024_steg.zip vacation_photo.jpg birds_singing.wav
```

When `zip -e hidden_archive.zip final.txt` asks for a password, enter `TIGER42`.

**Command/flag notes:** `convert` creates images; `steghide embed -cf` sets the cover file, `-ef` the embedded file, `-p` the passphrase; `exiftool -overwrite_original` edits metadata without leaving backup files; `sed -i` patches the old Spectrology Python method in place; `spectrology.py -o` sets output WAV, `-b/-t` set frequency range, `-p` sets pixels per second; `sox -n` generates audio from no input; `cat jpg zip > jpg` appends ZIP bytes while keeping the JPEG viewable.

---

## A3. HTTP credential capture without external dependency

**Goal:** create a `.pcapng` file containing a cleartext HTTP `POST` request with test credentials, then extract those credentials during investigation.

This version intentionally avoids a full web application. `curl` creates the HTTP request, `nc` listens on a TCP port and receives the raw bytes, and `tshark` captures the traffic. Netcat does not parse HTTP; it only sends and receives TCP data.

Use **two terminal tabs/windows**. Start capture first, then generate the HTTP request.

### Terminal 1: start packet capture

```bash
# Purpose: Refresh Kali's package index before installing tools.
# Flags: none.
sudo apt update
```

```bash
# Purpose: Install packet capture, HTTP client, and Netcat listener tools.
# Flags: -y automatically confirms the installation prompt.
sudo apt install -y tshark curl netcat-openbsd
```

```bash
# Purpose: Create the Task 3 working directory.
# Flags: -p creates missing parent directories and ignores the error if the directory already exists.
mkdir -p ~/task3
```

```bash
# Purpose: Remove old files so the result belongs only to this attempt.
# Flags: -f removes files without asking and ignores missing files.
sudo rm -f /tmp/login_capture.pcapng ~/task3/login_capture.pcapng ~/task3/nc_received_http.txt ~/task3/nc.pid
```

```bash
# Purpose: Start packet capture on localhost traffic and write the evidence file to /tmp.
# Flags:
#   -i lo captures the loopback interface used by 127.0.0.1 traffic.
#   -f 'tcp port 8099' is a capture filter; only TCP traffic using port 8099 is saved.
#   -w /tmp/login_capture.pcapng writes packets to a pcapng file.
# Action: leave this command running, then move to Terminal 2.
sudo tshark -i lo -f 'tcp port 8099' -w /tmp/login_capture.pcapng
```

Do not stop TShark yet. Continue with Terminal 2.

### Terminal 2: generate an HTTP POST request

```bash
# Purpose: Move into the Task 3 working directory.
# Flags: none.
cd ~/task3
```

```bash
# Purpose: Start a one-request Netcat listener on 127.0.0.1:8099.
# Flags and operators:
#   printf sends a minimal raw HTTP response: "200 OK" with a two-byte body, "OK".
#   \r\n means carriage-return + line-feed, the standard HTTP line ending.
#   | pipes that response into Netcat so curl receives a valid HTTP reply.
#   nc starts Netcat.
#   -l listens for an incoming TCP connection.
#   -s 127.0.0.1 binds the listener to localhost only.
#   -p 8099 selects TCP port 8099.
#   -w 10 exits after a short timeout if the connection becomes idle.
#   > saves the received HTTP request into nc_received_http.txt.
#   & runs the listener in the background.
printf 'HTTP/1.1 200 OK\r\nConnection: close\r\nContent-Length: 2\r\n\r\nOK' | nc -l -s 127.0.0.1 -p 8099 -w 10 > ~/task3/nc_received_http.txt &
```

```bash
# Purpose: Save the Netcat background process ID so we can wait for it later.
# Flags and variables:
#   $! is the PID of the most recent background process.
#   > writes the PID into nc.pid.
echo $! > ~/task3/nc.pid
```

```bash
# Purpose: Verify that port 8099 is listening before sending credentials.
# Flags:
#   -l shows listening sockets.
#   -t shows TCP sockets.
#   -n keeps numeric IP addresses and port numbers.
#   -p shows the owning process where permitted.
ss -ltnp | grep ':8099'
```

```bash
# Purpose: Send a cleartext HTTP POST request with test credentials.
# Flags:
#   -sS hides progress output but still shows errors.
#   --max-time 5 prevents hanging if the listener was not started correctly.
#   -X POST explicitly sets the HTTP method to POST.
#   -d sends form data in the request body; curl also sets application/x-www-form-urlencoded.
#   -o /dev/null discards the HTTP response body.
curl -sS --max-time 5 -X POST -d 'username=students_a&password=TestPass2024' http://127.0.0.1:8099/login -o /dev/null
```

```bash
# Purpose: Wait for Netcat to finish and flush nc_received_http.txt.
# Flags and operators:
#   $(cat ~/task3/nc.pid) reads the saved PID.
#   2>/dev/null hides harmless shell messages if the process already exited.
wait "$(cat ~/task3/nc.pid)" 2>/dev/null
```

```bash
# Purpose: Confirm that Netcat received the raw HTTP request.
# Flags: none.
cat ~/task3/nc_received_http.txt
```

Expected output includes:

```text
POST /login HTTP/1.1
Host: 127.0.0.1:8099
...
username=students_a&password=TestPass2024
```

### Terminal 1: stop capture and save evidence

Return to Terminal 1, stop TShark with **Ctrl+C**, then run these commands.

```bash
# Purpose: Change ownership of the capture from root back to the current Kali user.
# Flags: none.
# Variable: $USER expands to the current username.
sudo chown "$USER:$USER" /tmp/login_capture.pcapng
```

```bash
# Purpose: Move the completed evidence file into the Task 3 directory.
# Flags: none.
mv /tmp/login_capture.pcapng ~/task3/login_capture.pcapng
```

```bash
# Purpose: Confirm that the capture file exists and has a non-zero size.
# Flags: -l gives a long listing; -h shows a human-readable size.
ls -lh ~/task3/login_capture.pcapng
```

### Quick troubleshooting

```bash
# Purpose: Check that the capture contains packets on port 8099.
# Flags: -r reads the file; -Y applies a display filter.
tshark -r ~/task3/login_capture.pcapng -Y 'tcp.port == 8099'
```

If no packets appear, check these four points:

1. `tshark` must be running before `curl` is executed.
2. Localhost traffic must be captured on `lo`, not `eth0`.
3. The capture should first be written to `/tmp/login_capture.pcapng` to avoid path permission issues.
4. `ss -ltnp | grep ':8099'` must show Netcat listening before `curl` is run.

After the capture has been moved into `~/task3/login_capture.pcapng`, Role B can proceed with the task.

---

## A4. Deleted file traceable through Bash history

**Goal:** create a credentials file, leave an investigator-readable backup, delete the original, and reliably save the shell commands in Bash history.

Run all A4 commands in the Bash shell opened by the first command.

```bash
# Purpose: Start Bash so this task writes to .bash_history even if Kali's default shell is Zsh.
# Flags: none.
bash
```

```bash
# Purpose: Tell Bash which history file to update.
# Variable: $HOME expands to the current user's home directory.
export HISTFILE="$HOME/.bash_history"
```

```bash
# Purpose: Disable history filtering so commands are not skipped.
# Detail: An empty HISTCONTROL prevents ignorespace/ignoredups behavior from hiding useful evidence.
export HISTCONTROL=
```

```bash
# Purpose: Ask Bash to show timestamps when the history command is displayed.
# Detail: Bash stores timestamp markers in .bash_history and formats them when history is printed.
export HISTTIMEFORMAT='%F %T '
```

```bash
# Purpose: Create the directory where the original credentials file will exist briefly.
# Flags: -p creates missing parent directories and does not fail if the directory already exists.
mkdir -p ~/Documents/project
```

```bash
# Purpose: Create the first credential line in the original file.
# Operators: > creates the file or overwrites it if it already exists.
printf '%s\n' 'API_KEY=sk-prod-9f8e7d6c5b4a3210' > ~/Documents/project/credentials.txt
```

```bash
# Purpose: Append a second credential line to the same file.
# Operators: >> appends without overwriting the existing first line.
printf '%s\n' 'DB_PASS=Adm1n2024' >> ~/Documents/project/credentials.txt
```

```bash
# Purpose: List the project directory so the file path appears in command history.
# Flags: -l shows long metadata; -a includes hidden entries.
ls -la ~/Documents/project/
```

```bash
# Purpose: Display the file content so the sensitive path is visible in history.
# Flags: none.
cat ~/Documents/project/credentials.txt
```

```bash
# Purpose: Add realistic editor activity to history.
# Action: Press Ctrl+X to exit Nano without making changes.
# Flags: none.
nano ~/Documents/project/credentials.txt
```

```bash
# Purpose: Copy the original into a hidden backup location that should survive VM exchange better than /tmp.
# Flags: none.
cp ~/Documents/project/credentials.txt /var/tmp/.backup_creds_2024
```

```bash
# Purpose: Delete the original credentials file.
# Flags: none.
rm ~/Documents/project/credentials.txt
```

```bash
# Purpose: Remove the now-empty project directory.
# Flags: none.
rmdir ~/Documents/project
```

```bash
# Purpose: Confirm that the recent commands are present in this Bash session history.
# Flags: tail -20 prints only the last 20 history lines.
history | tail -20
```

```bash
# Purpose: Immediately append the current Bash session history to ~/.bash_history.
# Flags: -a appends new history lines to the history file now, without waiting for shell exit.
history -a
```

```bash
# Purpose: Leave the Bash subshell after the evidence has been written.
# Flags: none.
exit
```

**Expected result:** `~/Documents/project/credentials.txt` is gone, but `/var/tmp/.backup_creds_2024` remains and `.bash_history` contains the creation, copy, and deletion commands.

---

## A5. Hidden dotfile

**Goal:** hide sensitive text as a dotfile inside a hidden directory.

```bash
# Purpose: Create a sensitive file with a dot-prefixed name in the user's home directory.
# Operators: > creates or overwrites the file.
printf '%s\n' 'Bitcoin wallet: bc1q9h0v2xpkfslmnv3kxqsx2dy6qwx7d4f5cnz2pe' > ~/.config_backup_2019.dat
```

```bash
# Purpose: Create harmless dotfile noise so the target is not the only hidden file.
# Operators: > creates or overwrites the file.
printf '%s\n' 'irrelevant' > ~/.cache_temp
```

```bash
# Purpose: Create a second harmless dotfile-like backup for noise.
# Operators: > creates or overwrites the file.
printf '%s\n' 'more noise' > ~/.local_settings.bak
```

```bash
# Purpose: Show that normal ls output does not display dotfiles.
# Flags: none.
ls ~/
```

```bash
# Purpose: Show that dotfiles become visible when hidden entries are included.
# Flags: -l shows long metadata; -a includes dot-prefixed entries.
ls -la ~/
```

```bash
# Purpose: Create a hidden directory deeper in the user's home tree.
# Flags: -p creates missing parent directories and ignores existing directories.
mkdir -p ~/.local/share/.hidden_data
```

```bash
# Purpose: Move the sensitive dotfile into the deeper hidden directory.
# Flags: none.
mv ~/.config_backup_2019.dat ~/.local/share/.hidden_data/
```

```bash
# Purpose: Confirm the hidden file's metadata and final path.
# Flags: none.
stat ~/.local/share/.hidden_data/.config_backup_2019.dat
```

**Expected result:** the real target is `~/.local/share/.hidden_data/.config_backup_2019.dat`.

---

## A6. Base64-encoded credential

**Goal:** place Base64-looking credentials in a config file and in a script comment.

```bash
# Purpose: Create a scripts directory for the secondary artifact.
# Flags: -p creates the directory if it does not exist.
mkdir -p ~/scripts
```

```bash
# Purpose: Show the Base64 encoding of the chosen password.
# Detail: printf avoids adding an extra newline before encoding.
printf '%s' 'AdminPass!2024' | base64
```

```bash
# Purpose: Create a realistic INI-style config file containing the encoded credential.
# Operators: > writes the generated lines into ~/.app_config.ini.
printf '%s\n' '[server]' 'host=192.168.1.10' 'port=8443' '[auth]' 'user=admin' '# Encoded credential - do not modify' 'token=QWRtaW5QYXNzITIwMjQ=' '[logging]' 'level=INFO' 'path=/var/log/app.log' > ~/.app_config.ini
```

```bash
# Purpose: Display the config file to verify that the token was written correctly.
# Flags: none.
cat ~/.app_config.ini
```

```bash
# Purpose: Create a Python-looking deployment helper with a second Base64 value in a comment.
# Operators: > writes the generated lines into deploy.py.
printf '%s\n' '#!/usr/bin/env python3' '# Deployment helper' '# Legacy creds (TODO: migrate to vault): UGFyb2xlMTIzIQ==' 'import os' 'print("Deploying...")' > ~/scripts/deploy.py
```

```bash
# Purpose: Mark the helper script as executable.
# Flags: +x adds execute permission.
chmod +x ~/scripts/deploy.py
```

```bash
# Purpose: Confirm script permissions and path.
# Flags: -l shows long metadata; -h shows human-readable sizes when relevant.
ls -lh ~/.app_config.ini ~/scripts/deploy.py
```

**Expected result:** Role B should decode `QWRtaW5QYXNzITIwMjQ=` to `AdminPass!2024` and `UGFyb2xlMTIzIQ==` to `Parole123!`.

---

## A7. Suspicious process from `/tmp`

**Goal:** run a harmless long-lived hidden script from a suspicious temporary path.

```bash
# Purpose: Create a harmless script that stays alive by sleeping repeatedly.
# Operators: > writes the generated script to /tmp/.systemd-helper.
printf '%s\n' '#!/bin/bash' 'while true; do sleep 60; done' > /tmp/.systemd-helper
```

```bash
# Purpose: Make the hidden script executable.
# Flags: +x adds execute permission.
chmod +x /tmp/.systemd-helper
```

```bash
# Purpose: Start the hidden script with suspicious-looking arguments and detach it from the terminal.
# Flags and operators:
#   nohup keeps the process running if the terminal closes.
#   --beacon and --interval are harmless arguments used only to look suspicious.
#   > /dev/null discards standard output.
#   2>&1 sends standard error to the same destination as standard output.
#   & runs the process in the background.
nohup /tmp/.systemd-helper --beacon=192.168.1.50:4444 --interval=300 > /dev/null 2>&1 &
```

```bash
# Purpose: Confirm that the suspicious process is running.
# Flags: -a prints the full command line; -f matches against the full command line.
pgrep -af systemd-helper
```

Leave the process running for Role B.

**Expected result:** Role B should find a process launched from `/tmp/.systemd-helper` with suspicious arguments. The process is harmless; it only sleeps.

---

## A8. Unauthorized Python HTTP server

**Goal:** share files through a local Python HTTP server and leave it running.

```bash
# Purpose: Create the directory that will be shared by the HTTP server.
# Flags: -p creates the directory if it does not exist.
mkdir -p ~/share
```

```bash
# Purpose: Move into the directory that Python will serve.
# Flags: none.
cd ~/share
```

```bash
# Purpose: Create a visible file that should be found through the web server.
# Operators: > creates or overwrites report.txt.
printf '%s\n' 'Confidential financial report 2024' > report.txt
```

```bash
# Purpose: Create a hidden environment-style file that is also exposed by the server.
# Operators: > creates or overwrites .env.
printf '%s\n' 'API_KEY=xyz123' > .env
```

```bash
# Purpose: Create a small binary noise file.
# Flags:
#   if=/dev/urandom reads random bytes.
#   of=data.bin writes to data.bin.
#   bs=1K sets a 1 KiB block size.
#   count=10 writes 10 blocks, so the result is about 10 KiB.
#   status=none hides dd progress output.
dd if=/dev/urandom of=data.bin bs=1K count=10 status=none
```

```bash
# Purpose: Start a local-only HTTP server from the current directory and keep it running in the background.
# Flags and operators:
#   -m http.server runs Python's built-in static HTTP server module.
#   8080 sets the listening TCP port.
#   --bind 127.0.0.1 restricts access to localhost inside the VM.
#   > ~/share/http_server.log writes standard output to a log file.
#   2>&1 sends standard error to the same log file.
#   & backgrounds the server.
nohup python3 -m http.server 8080 --bind 127.0.0.1 > ~/share/http_server.log 2>&1 &
```

```bash
# Purpose: Save the server PID for optional later cleanup.
# Variable: $! is the PID of the most recent background process.
echo $! > ~/share/http_server.pid
```

```bash
# Purpose: Verify that TCP port 8080 is listening.
# Flags:
#   -l shows listening sockets.
#   -t shows TCP sockets.
#   -n keeps numeric addresses and ports.
#   -p shows process information where permitted.
ss -ltnp | grep ':8080'
```

Leave the server running for Role B.

**Expected result:** Role B should identify a Python process listening on `127.0.0.1:8080`, determine that `~/share` is the served directory, and download `report.txt` and `.env`.

---

## A9. Large suspicious file

**Goal:** create unusually large files in temporary and download locations.

```bash
# Purpose: Create a 100 MiB zero-filled file that looks like a cache artifact.
# Flags:
#   if=/dev/zero reads zero bytes.
#   of=/var/tmp/cache.bin writes to the target file.
#   bs=1M uses 1 MiB blocks.
#   count=100 writes 100 blocks, producing about 100 MiB.
#   status=progress shows progress while writing.
dd if=/dev/zero of=/var/tmp/cache.bin bs=1M count=100 status=progress
```

```bash
# Purpose: Create a smaller random-looking noise file in /tmp.
# Flags:
#   if=/dev/urandom reads pseudo-random bytes.
#   of=/tmp/temp_data.dat writes to the target file.
#   bs=1M uses 1 MiB blocks.
#   count=20 writes about 20 MiB.
#   status=none hides dd progress output.
dd if=/dev/urandom of=/tmp/temp_data.dat bs=1M count=20 status=none
```

```bash
# Purpose: Ensure the Downloads directory exists before creating the hidden large file.
# Flags: -p creates the directory if it does not exist.
mkdir -p ~/Downloads
```

```bash
# Purpose: Create an 80 MiB hidden file in Downloads.
# Flags:
#   if=/dev/zero reads zero bytes.
#   of=~/Downloads/.backup_2024.tar writes to a hidden dotfile.
#   bs=1M uses 1 MiB blocks.
#   count=80 writes about 80 MiB.
#   status=progress shows progress while writing.
dd if=/dev/zero of=~/Downloads/.backup_2024.tar bs=1M count=80 status=progress
```

```bash
# Purpose: Confirm that the expected files exist and show their sizes.
# Flags: -l gives long output; -h shows human-readable sizes.
ls -lh /var/tmp/cache.bin /tmp/temp_data.dat ~/Downloads/.backup_2024.tar
```

**Expected result:** Role B should find `/var/tmp/cache.bin` and `~/Downloads/.backup_2024.tar` when searching for files larger than 50 MiB.

---

## A10. Linux memory dump case

**Goal:** create simple RAM artifacts, acquire a Linux memory dump with LiME, and leave the dump for Role B.

Task 10 is more sensitive to Kali kernel/package state than the other tasks. Run it on a VM with enough free disk space for a RAM-sized dump.

```bash
# Purpose: Refresh Kali's package index before installing kernel/module tools.
# Flags: none.
sudo apt update
```

```bash
# Purpose: Install Netcat and the LiME DKMS module for the currently running kernel.
# Flags and substitutions:
#   -y automatically confirms installation prompts.
#   linux-headers-$(uname -r) installs headers matching the running kernel version.
#   lime-forensics-dkms builds the LiME kernel module through DKMS.
sudo apt install -y netcat-openbsd linux-headers-$(uname -r) lime-forensics-dkms
```

```bash
# Purpose: Create a plaintext artifact on disk and in file cache.
# Operators: > creates or overwrites the file.
printf '%s\n' 'MyMemoryPassword2024' > /tmp/secret.txt
```

```bash
# Purpose: Start a harmless Netcat listener so memory contains a network-process artifact.
# Flags and operators:
#   -l listens for inbound TCP connections.
#   -v prints verbose status to the log.
#   -n disables DNS lookups.
#   -p 9999 selects local TCP port 9999.
#   > /tmp/nc_listener.log writes output to a log file.
#   2>&1 sends errors to the same log file.
#   & backgrounds the listener.
nc -lvnp 9999 > /tmp/nc_listener.log 2>&1 &
```

```bash
# Purpose: Start a Python process that keeps a password and flag string in memory.
# Flags and operators:
#   -c runs the Python code supplied on the command line.
#   & backgrounds the process so the terminal remains usable.
python3 -c "import time; password='MyMemoryPassword2024'; flag='FLAG{volatility_finds_me}'; time.sleep(3600)" &
```

```bash
# Purpose: Confirm that the memory-artifact processes are active before dumping RAM.
# Flags: -a prints full command lines; -f matches the full command line.
pgrep -af 'nc|volatility_finds_me|MyMemoryPassword'
```

```bash
# Purpose: Remove an old dump from a previous run so this attempt produces a fresh file.
# Flags: -f ignores the error if the file does not exist.
sudo rm -f /var/tmp/memdump.lime
```

```bash
# Purpose: Load the LiME module and write RAM to a local file.
# Parameters:
#   path=/var/tmp/memdump.lime tells LiME where to save the dump.
#   format=lime stores the dump with LiME headers, which Volatility can read.
sudo modprobe lime path=/var/tmp/memdump.lime format=lime
```

```bash
# Purpose: Unload the LiME module after acquisition finishes.
# Flags: -r removes/unloads the named module.
sudo modprobe -r lime
```

```bash
# Purpose: Make the dump readable for Role B after VM exchange.
# Mode: 0644 lets the owner write and everyone read; use only inside this training VM.
sudo chmod 0644 /var/tmp/memdump.lime
```

```bash
# Purpose: Confirm the dump exists and show its size.
# Flags: -l gives long output; -h shows human-readable size.
ls -lh /var/tmp/memdump.lime
```

```bash
# Purpose: Record evidence integrity for the memory dump.
# Flags: none.
sha256sum /var/tmp/memdump.lime
```

If `linux-headers-$(uname -r)` or `lime-forensics-dkms` fails, update Kali, reboot into the repository kernel, and retry A10:

```bash
# Purpose: Update the VM to a repository-supported kernel/header set.
# Flags: full-upgrade allows package removals/installations needed for the upgrade; -y auto-confirms.
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

**Expected result:** `/var/tmp/memdump.lime` exists, has a large non-zero size, and contains searchable strings such as `MyMemoryPassword2024` and `FLAG{volatility_finds_me}`.

---

# Role B activities

## B1. Investigate encrypted ZIP disguised as JPEG

```bash
sudo apt update
sudo apt install -y p7zip-full john
mkdir -p ~/case1
find /home -type f -exec file {} + 2>/dev/null | grep -i 'Zip archive data'
cp /home/<student_user>/Pictures/old_2019/vacation_photo.jpg ~/case1/evidence.zip
cd ~/case1
file evidence.zip
7z l -slt evidence.zip | grep -i 'Encrypted'
7z l evidence.zip
zip2john evidence.zip > hash.txt
john --format=PKZIP --mask='?d?d?d' hash.txt
john --show --format=PKZIP hash.txt
7z x evidence.zip -p<recovered_password>
cat secret.txt
sha256sum evidence.zip secret.txt
```

**Command/flag notes:** `find /home -type f` searches files; `-exec file {} +` checks batches; `7z l -slt` shows technical archive fields; `zip2john` prepares a crackable hash; `john --mask='?d?d?d'` tries exactly three digits; `7z x -p<password>` extracts with the recovered password.

---

## B2. Investigate multi-level steganography

```bash
sudo apt update
sudo apt install -y unzip steghide libimage-exiftool-perl binwalk p7zip-full sox audacity
mkdir -p ~/case2
cp /home/<student_user>/task2/case_2024_steg.zip ~/case2/
cd ~/case2
unzip case_2024_steg.zip
file vacation_photo.jpg birds_singing.wav
sha256sum vacation_photo.jpg birds_singing.wav
exiftool vacation_photo.jpg
steghide info vacation_photo.jpg
steghide extract -sf vacation_photo.jpg -p 'Saulgriezi2024'
cat secret.txt
sox birds_singing.wav -n spectrogram -o spectrogram.png
xdg-open spectrogram.png
binwalk vacation_photo.jpg
binwalk -e vacation_photo.jpg
find . -type f -iname '*.zip' -print
7z x ./_vacation_photo.jpg.extracted/*.zip -pTIGER42
cat final.txt
```

If `binwalk -e` does not extract the appended ZIP, use the ZIP offset shown by `binwalk`:

```bash
binwalk vacation_photo.jpg
dd if=vacation_photo.jpg of=appended.zip bs=1 skip=<ZIP_OFFSET_FROM_BINWALK> status=none
7z x appended.zip -pTIGER42
cat final.txt
```

**Command/flag notes:** `unzip` opens the case package; `exiftool` shows the metadata hint; `steghide extract -sf` extracts from the stego file; `sox input.wav -n spectrogram -o image.png` creates a spectrogram image; `binwalk -e` extracts embedded files; `dd bs=1 skip=<offset>` copies bytes starting at the embedded ZIP offset.

---

## B3. Analyze HTTP capture

### Wireshark method

```bash
# Purpose: Open the capture file in Wireshark.
# Operator: & starts Wireshark in the background.
wireshark ~/task3/login_capture.pcapng &
```

In Wireshark:

1. Use display filter `tcp.port == 8099`.
2. Find the packet containing `POST /login`.
3. Right-click the packet and choose **Follow → TCP Stream**.
4. Record the cleartext form data:

```text
username=students_a&password=TestPass2024
```

If Wireshark does not label port `8099` as HTTP, use **Analyze → Decode As... → HTTP** for TCP port `8099`.

### TShark method

```bash
# Purpose: Extract decoded HTTP POST data from the capture.
# Flags:
#   -r reads an existing capture file.
#   -d tcp.port==8099,http forces TShark to decode TCP port 8099 as HTTP.
#   -Y keeps only packets matching the display filter.
#   -T fields prints selected fields only.
#   -e selects one field to print.
tshark -r ~/task3/login_capture.pcapng -d tcp.port==8099,http -Y 'http.request.method == "POST"' -T fields -e frame.time -e http.request.uri -e http.file_data
```

```bash
# Purpose: Fallback method; search readable strings in the pcapng file.
# Flags:
#   -i makes grep case-insensitive.
#   -E enables extended regular expressions.
strings ~/task3/login_capture.pcapng | grep -iE 'username|password|students_a|TestPass2024'
```

Document:

- File: `login_capture.pcapng`
- URL: `http://127.0.0.1:8099/login`
- Method: `POST`
- Username: `students_a`
- Password: `TestPass2024`
- Finding: credentials were visible because HTTP does not encrypt the request body.

---

## B4. Recover deleted credentials from history and backup

**Goal:** use Bash history to identify a deleted credentials file, then recover the remaining hidden backup.

```bash
# Purpose: Define which user's home directory is being investigated.
# Action: Replace <student_user> with the real username, or use "$HOME" if you are logged in as that user.
TARGET_HOME="/home/<student_user>"
```

```bash
# Purpose: Create a case directory for copied evidence and notes.
# Flags: -p creates the directory if it does not exist.
mkdir -p ~/case4
```

```bash
# Purpose: Confirm that the Bash history file exists for the target user.
# Flags: -l gives long metadata; -h shows human-readable sizes.
ls -lh "$TARGET_HOME/.bash_history"
```

```bash
# Purpose: View the end of the target user's Bash history.
# Flags: -100 prints the last 100 lines.
tail -100 "$TARGET_HOME/.bash_history"
```

```bash
# Purpose: Search history for commands related to the file creation, backup, editing, and deletion.
# Flags:
#   -n prints line numbers.
#   -E enables extended regular expressions.
grep -nE 'credentials|backup|rm |rmdir |cp |nano|cat ' "$TARGET_HOME/.bash_history"
```

```bash
# Purpose: Confirm that the hidden backup file exists in /var/tmp.
# Flags: -l gives long metadata; -a would show hidden entries if listing a directory; here it is harmless.
ls -la /var/tmp/.backup_creds_2024
```

```bash
# Purpose: Copy the backup into the investigator's case directory before reading it.
# Flags: none.
cp /var/tmp/.backup_creds_2024 ~/case4/backup_creds_2024
```

```bash
# Purpose: Read the recovered credential content from the evidence copy.
# Flags: none.
cat ~/case4/backup_creds_2024
```

```bash
# Purpose: Document ownership, permissions, size, and timestamps for the original backup.
# Flags: none.
stat /var/tmp/.backup_creds_2024
```

```bash
# Purpose: Record evidence integrity for both original and copied files.
# Flags: none.
sha256sum /var/tmp/.backup_creds_2024 ~/case4/backup_creds_2024
```

Fallback if Role A accidentally used Zsh instead of Bash:

```bash
# Purpose: Search Zsh history only if it exists.
# Operators: && runs grep only when the history file exists.
# Flags: grep -nE prints line numbers and uses extended regex.
test -f "$TARGET_HOME/.zsh_history" && grep -nE 'credentials|backup|rm |rmdir |cp |nano|cat ' "$TARGET_HOME/.zsh_history"
```

Document the deleted original path, the backup path, the recovered `API_KEY`, and the recovered `DB_PASS`.

---

## B5. Find hidden dotfiles

**Goal:** identify dot-prefixed files and hidden directories, then recover the target wallet text.

```bash
# Purpose: Define which user's home directory is being investigated.
# Action: Replace <student_user> with the real username, or use "$HOME" if you are logged in as that user.
TARGET_HOME="/home/<student_user>"
```

```bash
# Purpose: Create a case directory for Task 5 evidence.
# Flags: -p creates the directory if it does not exist.
mkdir -p ~/case5
```

```bash
# Purpose: Inspect the target user's home directory, including hidden entries.
# Flags: -l gives long metadata; -a includes dot-prefixed files and directories.
ls -la "$TARGET_HOME"
```

```bash
# Purpose: Find hidden directories under the target home.
# Flags and filters:
#   -type d limits results to directories.
#   -name '.*' matches names beginning with a dot.
#   -print prints matching paths.
#   2>/dev/null suppresses permission-denied noise.
find "$TARGET_HOME" -type d -name '.*' -print 2>/dev/null
```

```bash
# Purpose: Find files whose own names start with a dot.
# Flags and filters:
#   -type f limits results to regular files.
#   -name '.*' matches dot-prefixed filenames.
find "$TARGET_HOME" -type f -name '.*' -print 2>/dev/null
```

```bash
# Purpose: Find all files located anywhere under hidden paths, not only files whose own names start with a dot.
# Detail: This catches files inside hidden directories as well as dotfiles themselves.
# Flags: sort makes the result easier to review.
find "$TARGET_HOME" -path '*/.*' -type f -print 2>/dev/null | sort
```

```bash
# Purpose: Identify file types for hidden-path files while safely handling spaces in filenames.
# Flags and operators:
#   -print0 separates paths with NUL bytes.
#   xargs -0 reads NUL-separated paths safely.
find "$TARGET_HOME" -path '*/.*' -type f -print0 2>/dev/null | xargs -0 file
```

```bash
# Purpose: Copy the target hidden file into the case directory.
# Flags: none.
cp "$TARGET_HOME/.local/share/.hidden_data/.config_backup_2019.dat" ~/case5/config_backup_2019.dat
```

```bash
# Purpose: Read the recovered hidden file content.
# Flags: none.
cat ~/case5/config_backup_2019.dat
```

```bash
# Purpose: Document file metadata for the original hidden file.
# Flags: none.
stat "$TARGET_HOME/.local/share/.hidden_data/.config_backup_2019.dat"
```

```bash
# Purpose: Record evidence integrity for original and copied files.
# Flags: none.
sha256sum "$TARGET_HOME/.local/share/.hidden_data/.config_backup_2019.dat" ~/case5/config_backup_2019.dat
```

Avoid broad `grep -v` exclude filters in this task because they can accidentally hide the target path.

---

## B6. Find and decode Base64 credentials

**Goal:** locate Base64-looking strings in configuration/script files and decode the credentials.

```bash
# Purpose: Define which user's home directory is being investigated.
# Action: Replace <student_user> with the real username, or use "$HOME" if you are logged in as that user.
TARGET_HOME="/home/<student_user>"
```

```bash
# Purpose: Create a case directory for decoded values and copied evidence.
# Flags: -p creates the directory if it does not exist.
mkdir -p ~/case6
```

```bash
# Purpose: Search the target home for Base64-looking strings.
# Flags:
#   -r searches recursively.
#   -I ignores binary files.
#   -n prints line numbers.
#   -E enables extended regular expressions.
#   head -50 limits noisy output.
grep -rInE '[A-Za-z0-9+/]{12,}={0,2}' "$TARGET_HOME" 2>/dev/null | head -50
```

```bash
# Purpose: List likely configuration and script files for focused review.
# Filters:
#   -type f limits results to files.
#   -name matches common config/script extensions.
# Operators: \( and \) group the name conditions for find.
find "$TARGET_HOME" -type f \( -name '*.ini' -o -name '*.conf' -o -name '*.cfg' -o -name '*.env' -o -name '*.py' \) -print 2>/dev/null
```

```bash
# Purpose: Copy the main config file into the case directory before decoding.
# Flags: none.
cp "$TARGET_HOME/.app_config.ini" ~/case6/app_config.ini
```

```bash
# Purpose: Show the Base64-looking token line in the copied config.
# Flags: -n prints line numbers; -E enables extended regex.
grep -nE '[A-Za-z0-9+/]{12,}={0,2}' ~/case6/app_config.ini
```

```bash
# Purpose: Decode the token value while preserving Base64 padding characters.
# Flags and operators:
#   sed removes only the leading token= text.
#   base64 -d decodes Base64 input.
#   echo prints a final newline after decoded output.
sed -n 's/^token=//p' ~/case6/app_config.ini | base64 -d; echo
```

```bash
# Purpose: Copy the secondary script artifact into the case directory.
# Flags: none.
cp "$TARGET_HOME/scripts/deploy.py" ~/case6/deploy.py
```

```bash
# Purpose: Find Base64-looking values that appear in script comments.
# Flags: -n prints line numbers; -E enables extended regex.
grep -nE '#.*[A-Za-z0-9+/]{12,}={0,2}' ~/case6/deploy.py
```

```bash
# Purpose: Extract only the Base64 token from the script and decode it.
# Flags:
#   -h suppresses filenames.
#   -o prints only the matching token.
#   -E enables extended regex.
#   base64 -d decodes the extracted value.
grep -hoE '[A-Za-z0-9+/]{12,}={0,2}' ~/case6/deploy.py | base64 -d; echo
```

```bash
# Purpose: Record evidence integrity for copied artifacts.
# Flags: none.
sha256sum ~/case6/app_config.ini ~/case6/deploy.py
```

Document that Base64 is encoding, not encryption.

---

## B7. Identify suspicious process

**Goal:** find a suspicious process running from `/tmp`, inspect its process metadata, preserve the script, and stop it after documentation.

```bash
# Purpose: Refresh package indexes before installing helper tools.
# Flags: none.
sudo apt update
```

```bash
# Purpose: Install tools for open-file and legacy network checks.
# Flags: -y automatically confirms installation prompts.
sudo apt install -y lsof net-tools
```

```bash
# Purpose: Create a case directory for the suspicious sample.
# Flags: -p creates the directory if it does not exist.
mkdir -p ~/case7
```

```bash
# Purpose: Show a process tree for broad situational awareness.
# Flags:
#   a shows processes from all users with a terminal.
#   u shows user-oriented details.
#   x includes processes without a terminal.
#   f draws process hierarchy.
ps auxf
```

```bash
# Purpose: Search for processes launched from suspicious writable temporary locations.
# Flags: grep -E enables extended regex; grep -v grep removes the grep process itself.
ps aux | grep -E '/tmp/|/dev/shm/|/var/tmp/' | grep -v grep
```

```bash
# Purpose: Search for command lines containing hidden dot-prefixed executable paths.
# Flags:
#   -e selects all processes.
#   -o chooses displayed columns.
#   grep -E enables extended regex.
ps -eo pid,ppid,user,etime,cmd | grep -E '/\.[^/ ]+' | grep -v grep
```

```bash
# Purpose: Find the exact suspicious process by its expected name.
# Flags: -a prints the full command line; -f matches against the full command line.
pgrep -af systemd-helper
```

```bash
# Purpose: Save the suspicious process ID into a shell variable.
# Detail: head -n 1 selects one PID if more than one match exists.
PID=$(pgrep -f '/tmp/.systemd-helper' | head -n 1)
```

```bash
# Purpose: Print the selected PID so it can be recorded in notes.
# Flags: none.
echo "$PID"
```

```bash
# Purpose: Identify the interpreter or executable used by the process.
# Detail: for a shell script, this usually points to bash, not to the script file.
# Flags: -f resolves the final absolute path.
readlink -f /proc/$PID/exe
```

```bash
# Purpose: Read the full command line from /proc.
# Detail: cmdline is NUL-separated, so tr converts NUL bytes to spaces.
tr '\0' ' ' < /proc/$PID/cmdline; echo
```

```bash
# Purpose: Identify the process working directory.
# Flags: -f resolves the final absolute path.
readlink -f /proc/$PID/cwd
```

```bash
# Purpose: Review the first environment variables for context.
# Detail: environ is NUL-separated, so tr converts NUL bytes to newlines.
tr '\0' '\n' < /proc/$PID/environ | head -30
```

```bash
# Purpose: List files opened by the suspicious process.
# Flags: -p selects the target PID.
sudo lsof -p "$PID"
```

```bash
# Purpose: Check whether the process owns network sockets.
# Flags:
#   -t shows TCP sockets.
#   -u shows UDP sockets.
#   -l shows listening sockets.
#   -p shows process information.
#   -n keeps numeric addresses and ports.
# Note: no output is acceptable here because this lab process only sleeps.
sudo ss -tulpn | grep "$PID"
```

```bash
# Purpose: Hash the suspicious script file itself.
# Flags: none.
sha256sum /tmp/.systemd-helper
```

```bash
# Purpose: Preserve the suspicious script for later review.
# Flags: none.
cp /tmp/.systemd-helper ~/case7/suspicious_sample.sh
```

```bash
# Purpose: Stop the suspicious process after evidence has been documented.
# Flags: none.
kill "$PID"
```

Do not copy `/proc/<PID>/exe` as the sample in this task; for a Bash script it points to the Bash interpreter. Copy `/tmp/.systemd-helper` instead.

---

## B8. Investigate Python HTTP server

**Goal:** identify the local HTTP service, map it to a process and directory, download exposed files, and stop the server after documentation.

```bash
# Purpose: Refresh package indexes before installing helper tools.
# Flags: none.
sudo apt update
```

```bash
# Purpose: Install tools for process-to-port mapping and HTTP downloads.
# Flags: -y automatically confirms installation prompts.
sudo apt install -y lsof curl
```

```bash
# Purpose: Create and enter a case directory for downloaded HTTP evidence.
# Flags: -p creates the directory if it does not exist.
mkdir -p ~/case8
cd ~/case8
```

```bash
# Purpose: Show listening TCP and UDP sockets.
# Flags:
#   -t shows TCP sockets.
#   -u shows UDP sockets.
#   -l shows listening sockets.
#   -n keeps numeric addresses and ports.
#   -p shows process information where permitted.
ss -tulnp
```

```bash
# Purpose: Map TCP port 8080 to the listening process.
# Flags:
#   -iTCP:8080 selects TCP sockets using port 8080.
#   -sTCP:LISTEN limits output to listening sockets.
sudo lsof -iTCP:8080 -sTCP:LISTEN
```

```bash
# Purpose: Save the server PID into a shell variable.
# Flags:
#   -t prints only the PID.
#   head -n 1 selects one PID if there is more than one result.
PID=$(sudo lsof -tiTCP:8080 -sTCP:LISTEN | head -n 1)
```

```bash
# Purpose: Print the selected PID for documentation.
# Flags: none.
echo "$PID"
```

```bash
# Purpose: Show process details for the server.
# Flags: -f shows a full-format process listing; -p selects the PID.
ps -fp "$PID"
```

```bash
# Purpose: Identify the directory being served by Python http.server.
# Flags: -f resolves the final absolute path.
readlink -f /proc/$PID/cwd
```

```bash
# Purpose: Read the full server command line.
# Detail: cmdline is NUL-separated, so tr converts NUL bytes to spaces.
tr '\0' ' ' < /proc/$PID/cmdline; echo
```

```bash
# Purpose: Download and save the directory listing.
# Flags and operators:
#   -s makes curl silent.
#   tee displays output and writes it to index.html.
curl -s http://127.0.0.1:8080/ | tee index.html
```

```bash
# Purpose: Extract linked filenames from the saved directory listing.
# Flags:
#   -o prints only matching text.
#   -E enables extended regex.
grep -oE 'href="[^"]+"' index.html
```

```bash
# Purpose: Download the visible report file using its remote filename.
# Flags:
#   -s makes curl silent.
#   -O saves the output with the remote filename.
curl -s -O http://127.0.0.1:8080/report.txt
```

```bash
# Purpose: Read the downloaded report.
# Flags: none.
cat report.txt
```

```bash
# Purpose: Download the hidden .env file exposed by the server.
# Flags:
#   -s makes curl silent.
#   -O saves the output with the remote filename.
curl -s -O http://127.0.0.1:8080/.env
```

```bash
# Purpose: Read the downloaded .env file.
# Flags: none.
cat .env
```

```bash
# Purpose: Record evidence integrity for the downloaded files.
# Flags: none.
sha256sum report.txt .env index.html
```

```bash
# Purpose: Stop the unauthorized HTTP server after documentation.
# Flags: none.
sudo kill "$PID"
```

---

## B9. Find large files

**Goal:** locate large unusual files, inspect their content type and metadata, and hash the primary evidence.

```bash
# Purpose: Refresh package indexes before installing helper tools.
# Flags: none.
sudo apt update
```

```bash
# Purpose: Install a hex viewer and open-file checker.
# Flags: -y automatically confirms installation prompts.
sudo apt install -y xxd lsof
```

```bash
# Purpose: Create a case directory for Task 9 notes and hashes.
# Flags: -p creates the directory if it does not exist.
mkdir -p ~/case9
```

```bash
# Purpose: Search common user/temp locations for files larger than 50 MiB.
# Flags and filters:
#   -type f limits results to regular files.
#   -size +50M finds files larger than 50 MiB.
#   -printf '%s %p\n' prints size in bytes and path.
#   sort -nr sorts largest first.
#   head -20 limits output.
sudo find /home /tmp /var/tmp -type f -size +50M -printf '%s %p\n' 2>/dev/null | sort -nr | head -20
```

```bash
# Purpose: Review disk usage in the same high-interest locations.
# Flags:
#   -a shows files as well as directories.
#   -h shows human-readable sizes.
#   sort -h sorts human-readable sizes.
#   tail -20 shows the largest entries at the end.
du -ah /home /tmp /var/tmp 2>/dev/null | sort -h | tail -20
```

```bash
# Purpose: Confirm the primary suspicious file size.
# Flags: -l gives long output; -h shows human-readable size.
ls -lh /var/tmp/cache.bin
```

```bash
# Purpose: Identify the file type from content/signature, not filename.
# Flags: none.
file /var/tmp/cache.bin
```

```bash
# Purpose: Document ownership, permissions, size, and timestamps.
# Flags: none.
stat /var/tmp/cache.bin
```

```bash
# Purpose: Inspect the first 128 bytes as hexadecimal.
# Flags: -l 128 limits output to 128 bytes.
xxd -l 128 /var/tmp/cache.bin
```

```bash
# Purpose: Look for readable strings in the primary file.
# Detail: zero-filled files may produce no useful string output, which is still a finding.
# Flags: head -20 limits output.
strings /var/tmp/cache.bin | head -20
```

```bash
# Purpose: Record evidence integrity and save the hash to the case directory.
# Operators: tee writes output to a file and also displays it.
sha256sum /var/tmp/cache.bin | tee ~/case9/cache.bin.sha256
```

```bash
# Purpose: Check whether any running process currently has the file open.
# Flags: -- marks the end of options so the filename is treated only as a path.
sudo lsof -- /var/tmp/cache.bin
```

Document the full path, size in bytes, human-readable size, timestamps, file type, and SHA-256 hash.

---

## B10. Analyze Linux memory dump

**Goal:** preserve the memory dump, perform minimum string triage, then attempt Volatility 3 analysis if symbols are available.

```bash
# Purpose: Refresh package indexes before installing analysis prerequisites.
# Flags: none.
sudo apt update
```

```bash
# Purpose: Install Python venv support, symbol-generation tools, compression tools, and binutils.
# Flags: -y automatically confirms installation prompts.
sudo apt install -y python3 python3-venv python3-pip binutils dwarf2json xz-utils
```

```bash
# Purpose: Create an isolated Python virtual environment for Volatility 3.
# Flags: -m venv runs Python's venv module.
python3 -m venv ~/vol3
```

```bash
# Purpose: Upgrade pip inside the virtual environment.
# Detail: Using the venv avoids Kali/Debian externally-managed Python restrictions.
~/vol3/bin/pip install --upgrade pip
```

```bash
# Purpose: Install Volatility 3 inside the virtual environment.
# Flags: none.
~/vol3/bin/pip install volatility3
```

```bash
# Purpose: Verify that Volatility 3 runs.
# Flags: --help prints usage; head -20 limits output.
~/vol3/bin/vol --help | head -20
```

```bash
# Purpose: Create and enter a case directory for memory evidence.
# Flags: -p creates the directory if it does not exist.
mkdir -p ~/case10
cd ~/case10
```

```bash
# Purpose: Copy the memory dump into the case directory.
# Detail: sudo is used because memory dumps are commonly root-owned.
# Flags: none.
sudo cp /var/tmp/memdump.lime ~/case10/
```

```bash
# Purpose: Give the investigator user ownership of the copied dump.
# Variable: $USER expands to the current username.
sudo chown "$USER:$USER" ~/case10/memdump.lime
```

```bash
# Purpose: Record evidence integrity for the copied dump.
# Flags: none.
sha256sum memdump.lime
```

```bash
# Purpose: Run minimum reliable string triage for likely secrets and indicators.
# Flags and operators:
#   grep -i ignores case.
#   grep -E enables extended regex.
#   sort -u sorts and removes duplicates.
#   head -50 limits output.
strings memdump.lime | grep -iE 'password|flag|http|api_key' | sort -u | head -50
```

```bash
# Purpose: Locate the known password string with decimal offsets.
# Flags: -t d prints decimal offsets for strings.
strings -t d memdump.lime | grep -i 'MyMemoryPassword'
```

```bash
# Purpose: Locate CTF-style flag strings.
# Flags: grep -E enables extended regex.
strings memdump.lime | grep -E 'FLAG\{[^}]+\}'
```

```bash
# Purpose: Identify Linux kernel banners in the memory image.
# Flags: -f selects the memory dump file.
~/vol3/bin/vol -f memdump.lime banners.Banners
```

Attempt Linux plugins after the banner command. These usually require an exact Volatility 3 Linux symbol table for the dumped kernel.

```bash
# Purpose: Attempt to list processes from the memory dump.
# Flags: -f selects the memory dump file.
~/vol3/bin/vol -f memdump.lime linux.pslist.PsList
```

```bash
# Purpose: Attempt to show a process tree.
# Flags: -f selects the memory dump file.
~/vol3/bin/vol -f memdump.lime linux.pstree.PsTree
```

```bash
# Purpose: Attempt to show process command lines.
# Flags: -f selects the memory dump file.
~/vol3/bin/vol -f memdump.lime linux.psaux.PsAux
```

```bash
# Purpose: Attempt to show socket information from the memory dump.
# Flags: -f selects the memory dump file.
~/vol3/bin/vol -f memdump.lime linux.sockstat.Sockstat
```

```bash
# Purpose: Attempt to recover Bash command history from memory.
# Flags: -f selects the memory dump file.
~/vol3/bin/vol -f memdump.lime linux.bash.Bash
```

If Linux plugins fail with a missing-symbol-table or ISF error, record the error and keep the `strings` findings. Then check what symbol files Volatility can currently see:

```bash
# Purpose: List Volatility's known ISF symbol information.
# Flags: head -50 limits output.
~/vol3/bin/vol isfinfo.IsfInfo | head -50
```

Optional symbol-table generation path, only if the exact unstripped debug kernel file exists for the dumped kernel. The commands below use `$(uname -r)` because Role B is normally analyzing the same VM immediately after acquisition; if the banner shows a different kernel version, replace `$(uname -r)` with the dumped kernel version.

```bash
# Purpose: Check whether the exact debug vmlinux file exists.
# Detail: Most systems do not have this unless a debug-symbol package was installed.
# Flags: -f tests for a regular file.
sudo test -f "/usr/lib/debug/boot/vmlinux-$(uname -r)" && echo 'debug vmlinux exists'
```

```bash
# Purpose: Check whether the matching System.map file exists.
# Flags: -f tests for a regular file.
sudo test -f "/boot/System.map-$(uname -r)" && echo 'System.map exists'
```

```bash
# Purpose: Generate a compressed Volatility 3 Linux ISF file from debug symbols.
# Flags and operators:
#   dwarf2json linux selects Linux ISF generation.
#   --elf provides the unstripped debug kernel file.
#   --system-map provides kernel symbol addresses.
#   xz -c compresses output to stdout.
#   > writes the compressed ISF file.
sudo dwarf2json linux --elf "/usr/lib/debug/boot/vmlinux-$(uname -r)" --system-map "/boot/System.map-$(uname -r)" | xz -c > "kali-$(uname -r).json.xz"
```

```bash
# Purpose: Create a local symbol directory for this case.
# Flags: -p creates missing parent directories.
mkdir -p ~/case10/symbols/linux
```

```bash
# Purpose: Copy the generated ISF file into the local Linux symbols directory.
# Flags: none.
cp "kali-$(uname -r).json.xz" ~/case10/symbols/linux/
```

```bash
# Purpose: Retry process listing while explicitly pointing Volatility to the local symbol directory.
# Flags:
#   -s ~/case10/symbols sets an additional symbols directory.
#   -f selects the memory dump file.
~/vol3/bin/vol -s ~/case10/symbols -f memdump.lime linux.pslist.PsList
```

Document whether the analysis succeeded through Volatility plugins or through minimum `strings` triage only.

---

## Minimal Role B report checklist

For every task, record:

- VM/user investigated and date/time.
- Commands executed and important output.
- Evidence path and SHA-256 hash.
- Finding summary: what was hidden, where it was found, and how it was verified.
- Any failed command and the correction used.
