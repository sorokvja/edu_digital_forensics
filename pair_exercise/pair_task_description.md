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
- Linux memory forensics includes the Volatility 3 symbol-table caveat and a minimum `strings` triage path.

## Common notation

- Replace `<student_user>` with the real Linux username.
- Type commands one at a time; do not paste whole task blocks blindly.
- Leave running services/processes active until Role B documents them.
- Hash evidence files with `sha256sum <file>`.

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

### Terminal 1: capture traffic

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
# Action: leave this running until the curl request has been sent in Terminal 2.
sudo tshark -i lo -f 'tcp port 8099' -w /tmp/login_capture.pcapng
```

Now move to [Terminal 2: generate a HTTP POST request](#terminal-2:-generate-a-http-post-request) 

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

### Terminal 2: generate a HTTP POST request

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

Role A now returns to Terminal 1, stops `tshark` with **Ctrl+C**, moves the capture into `~/task3/login_capture.pcapng`, and gives that file to Role B.

---

## A4. Deleted file traceable through Bash history

**Goal:** create credentials, copy a hidden backup, delete the original, and reliably save Bash history.

```bash
bash
export HISTFILE=~/.bash_history
export HISTCONTROL=
export HISTTIMEFORMAT='%F %T '
mkdir -p ~/Documents/project
printf '%s\n' 'API_KEY=sk-prod-9f8e7d6c5b4a3210' > ~/Documents/project/credentials.txt
printf '%s\n' 'DB_PASS=Adm1n2024' >> ~/Documents/project/credentials.txt
ls -la ~/Documents/project/
cat ~/Documents/project/credentials.txt
nano ~/Documents/project/credentials.txt
cp ~/Documents/project/credentials.txt /var/tmp/.backup_creds_2024
rm ~/Documents/project/credentials.txt
rmdir ~/Documents/project
history | tail -20
history -a
exit
```

**Command/flag notes:** `bash` forces Bash even if Kali uses another shell; `HISTFILE` sets the history file; empty `HISTCONTROL` avoids ignored commands; `HISTTIMEFORMAT` records timestamps; `>` overwrites, `>>` appends; `history -a` writes current session history immediately.

---

## A5. Hidden dotfile

**Goal:** hide sensitive text as a dotfile inside a hidden directory.

```bash
printf '%s\n' 'Bitcoin wallet: bc1q9h0v2xpkfslmnv3kxqsx2dy6qwx7d4f5cnz2pe' > ~/.config_backup_2019.dat
printf '%s\n' 'irrelevant' > ~/.cache_temp
printf '%s\n' 'more noise' > ~/.local_settings.bak
ls ~/
ls -la ~/
mkdir -p ~/.local/share/.hidden_data
mv ~/.config_backup_2019.dat ~/.local/share/.hidden_data/
stat ~/.local/share/.hidden_data/.config_backup_2019.dat
```

**Command/flag notes:** dot-prefixed names are hidden from plain `ls`; `ls -la` shows hidden files and metadata; `stat` shows size, permissions, owner, and timestamps.

---

## A6. Base64-encoded credential

**Goal:** place Base64-looking credentials in a config file and script comment.

```bash
mkdir -p ~/scripts
echo -n 'AdminPass!2024' | base64
printf '%s\n' '[server]' 'host=192.168.1.10' 'port=8443' '[auth]' 'user=admin' '# Encoded credential - do not modify' 'token=QWRtaW5QYXNzITIwMjQ=' '[logging]' 'level=INFO' 'path=/var/log/app.log' > ~/.app_config.ini
cat ~/.app_config.ini
printf '%s\n' '#!/usr/bin/env python3' '# Deployment helper' '# Legacy creds (TODO: migrate to vault): UGFyb2xlMTIzIQ==' 'import os' 'print("Deploying...")' > ~/scripts/deploy.py
chmod +x ~/scripts/deploy.py
```

**Command/flag notes:** `echo -n` avoids encoding a newline; `base64` encodes stdin; `chmod +x` makes a file executable.

---

## A7. Suspicious process from `/tmp`

**Goal:** run a harmless long-lived hidden script from a suspicious path.

```bash
printf '%s\n' '#!/bin/bash' 'while true; do sleep 60; done' > /tmp/.systemd-helper
chmod +x /tmp/.systemd-helper
nohup /tmp/.systemd-helper --beacon=192.168.1.50:4444 --interval=300 > /dev/null 2>&1 &
pgrep -af systemd-helper
```

Leave the process running for Role B.

**Command/flag notes:** `/tmp/.systemd-helper` is suspicious because it is hidden and in a temporary directory; `nohup` keeps it alive after terminal closure; `/dev/null 2>&1` hides output/errors; `pgrep -af` prints matching PIDs and full command lines.

---

## A8. Unauthorized Python HTTP server

**Goal:** share files through a local HTTP server and leave it running.

```bash
mkdir -p ~/share
cd ~/share
printf '%s\n' 'Confidential financial report 2024' > report.txt
printf '%s\n' 'API_KEY=xyz123' > .env
dd if=/dev/urandom of=data.bin bs=1K count=10 status=none
nohup python3 -m http.server 8080 --bind 127.0.0.1 > ~/share/http_server.log 2>&1 &
echo $! > ~/share/http_server.pid
ss -tlnp | grep ':8080'
```

Leave the server running for Role B.

**Command/flag notes:** `dd if=/dev/urandom` creates random bytes; `bs=1K count=10` creates 10 KiB; `python3 -m http.server 8080` serves the current directory; `ss -tlnp` shows listening TCP sockets with numeric ports and process info.

---

## A9. Large suspicious file

**Goal:** create unusually large files in temporary/download locations.

```bash
dd if=/dev/zero of=/var/tmp/cache.bin bs=1M count=100 status=progress
dd if=/dev/urandom of=/tmp/temp_data.dat bs=1M count=20 status=none
mkdir -p ~/Downloads
dd if=/dev/zero of=~/Downloads/.backup_2024.tar bs=1M count=80 status=progress
ls -lh /var/tmp/cache.bin /tmp/temp_data.dat ~/Downloads/.backup_2024.tar
```

**Command/flag notes:** `dd` copies raw bytes; `if=` is input, `of=` output; `/dev/zero` produces zero bytes; `/dev/urandom` produces random bytes; `bs=1M count=100` writes 100 MiB; `status=progress` shows progress; `ls -lh` shows human-readable sizes.

---

## A10. Linux memory dump case

**Goal:** create RAM artifacts, acquire a Linux memory dump, and leave it for Role B.

```bash
sudo apt update
sudo apt install -y volatility3 netcat-openbsd dwarf2json xz-utils
sudo apt install -y linux-headers-$(uname -r) lime-forensics-dkms
printf '%s\n' 'MyMemoryPassword2024' > /tmp/secret.txt
nc -lvnp 9999 > /tmp/nc_listener.log 2>&1 &
python3 -c "import time; password='MyMemoryPassword2024'; flag='FLAG{volatility_finds_me}'; time.sleep(3600)" &
pgrep -af 'nc|volatility_finds_me|MyMemoryPassword'
sudo modprobe lime path=/var/tmp/memdump.lime format=lime
sudo modprobe -r lime
ls -lh /var/tmp/memdump.lime
sha256sum /var/tmp/memdump.lime
```

If kernel headers fail to install, update Kali, reboot into the repository kernel, and retry the LiME install/acquisition:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

**Command/flag notes:** `lime-forensics-dkms` builds the LiME module; `linux-headers-$(uname -r)` targets the running kernel; `nc -lvnp 9999` means listen/verbose/no DNS/port 9999; `python3 -c` keeps strings in memory; `modprobe lime path=... format=lime` writes RAM to file; `modprobe -r lime` unloads LiME.

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

---

## B4. Recover deleted credentials from history and backup

```bash
mkdir -p ~/case4
cat ~/.bash_history | tail -100
grep -nE 'credentials|backup|rm |rmdir |cp |nano|cat ' ~/.bash_history
ls -la /var/tmp/.backup_creds_2024
cat /var/tmp/.backup_creds_2024
stat /var/tmp/.backup_creds_2024
sha256sum /var/tmp/.backup_creds_2024
```

Fallback if Role A used Zsh accidentally:

```bash
test -f ~/.zsh_history && grep -nE 'credentials|backup|rm |rmdir |cp |nano|cat ' ~/.zsh_history
```

**Command/flag notes:** `tail -100` shows the last 100 lines; `grep -nE` prints line numbers and uses extended regex; `ls -la` shows hidden metadata; `stat` gives ownership, permissions, and timestamps.

---

## B5. Find hidden dotfiles

```bash
mkdir -p ~/case5
ls -la ~/
find "$HOME" -type d -name '.*' -print 2>/dev/null
find "$HOME" -type f -name '.*' -print 2>/dev/null
find "$HOME" -path '*/.*' -type f -print 2>/dev/null | sort
find "$HOME" -path '*/.*' -type f -print0 2>/dev/null | xargs -0 file
cat ~/.local/share/.hidden_data/.config_backup_2019.dat
stat ~/.local/share/.hidden_data/.config_backup_2019.dat
sha256sum ~/.local/share/.hidden_data/.config_backup_2019.dat
```

**Command/flag notes:** `-name '.*'` finds dot-named entries; `-path '*/.*'` also finds files under hidden directories; `2>/dev/null` suppresses permission errors; `-print0 | xargs -0` handles spaces safely; avoid broad exclude filters in this task.

---

## B6. Find and decode Base64 credentials

```bash
mkdir -p ~/case6
grep -rInE '[A-Za-z0-9+/]{12,}={0,2}' "$HOME" 2>/dev/null | head -50
find "$HOME" -type f \( -name '*.ini' -o -name '*.conf' -o -name '*.cfg' -o -name '*.env' -o -name '*.py' \) -print 2>/dev/null
grep -nE '[A-Za-z0-9+/]{12,}={0,2}' ~/.app_config.ini
awk -F= '/^token=/{print $2}' ~/.app_config.ini | base64 -d; echo
grep -rInE '#.*[A-Za-z0-9+/]{12,}={0,2}' ~/scripts 2>/dev/null
grep -hoE '[A-Za-z0-9+/]{12,}={0,2}' ~/scripts/deploy.py | base64 -d; echo
```

**Command/flag notes:** `grep -rInE` recursively searches, ignores binary noise, prints line numbers, and uses regex; the regex targets Base64-looking strings; `awk -F=` splits on `=`; `base64 -d` decodes; `grep -h -o` prints only the matched token without filename.

---

## B7. Identify suspicious process

```bash
sudo apt update
sudo apt install -y lsof net-tools
mkdir -p ~/case7
ps auxf
ps aux | grep -E '/tmp/|/dev/shm/|/var/tmp/' | grep -v grep
ps -eo pid,ppid,user,etime,cmd | grep -E '/\.[^/ ]+' | grep -v grep
pgrep -af systemd-helper
PID=$(pgrep -f '/tmp/.systemd-helper' | head -n 1)
echo "$PID"
readlink -f /proc/$PID/exe
tr '\0' ' ' < /proc/$PID/cmdline; echo
readlink -f /proc/$PID/cwd
tr '\0' '\n' < /proc/$PID/environ | head -30
sudo lsof -p "$PID"
sudo ss -tulpn | grep "$PID"
sha256sum /tmp/.systemd-helper
cp /tmp/.systemd-helper ~/case7/suspicious_sample.sh
kill "$PID"
```

**Command/flag notes:** `ps auxf` shows a process tree; `ps -eo` selects columns; `pgrep -af` finds matching full command lines; `/proc/<PID>/cmdline` and `environ` are NUL-separated, so `tr '\0'` makes them readable; `lsof -p` lists open files; `ss -tulpn` lists TCP/UDP sockets and PIDs; copy the script path, not `/proc/<PID>/exe`, because the true executable is usually Bash.

---

## B8. Investigate Python HTTP server

```bash
sudo apt update
sudo apt install -y lsof curl
mkdir -p ~/case8
cd ~/case8
ss -tulnp
sudo lsof -iTCP:8080 -sTCP:LISTEN
PID=$(sudo lsof -tiTCP:8080 -sTCP:LISTEN | head -n 1)
echo "$PID"
ps -fp "$PID"
readlink -f /proc/$PID/cwd
tr '\0' ' ' < /proc/$PID/cmdline; echo
curl -s http://127.0.0.1:8080/ | tee index.html
grep -oE 'href="[^"]+"' index.html
curl -s -O http://127.0.0.1:8080/report.txt
cat report.txt
curl -s -O http://127.0.0.1:8080/.env
cat .env
sha256sum report.txt .env
sudo kill "$PID"
```

**Command/flag notes:** `ss -tulnp` shows TCP/UDP listening sockets with numeric ports and processes; `lsof -iTCP:8080 -sTCP:LISTEN` maps port to PID; `lsof -t` prints only PID; `/proc/<PID>/cwd` reveals the served directory; `curl -s -O` downloads quietly using the remote filename; `tee` displays and saves output.

---

## B9. Find large files

```bash
sudo apt update
sudo apt install -y xxd lsof
mkdir -p ~/case9
sudo find / -xdev -type f -size +50M -printf '%s %p\n' 2>/dev/null | sort -nr | head -20
du -ah /home /tmp /var/tmp 2>/dev/null | sort -h | tail -20
ls -lh /var/tmp/cache.bin
file /var/tmp/cache.bin
stat /var/tmp/cache.bin
xxd -l 128 /var/tmp/cache.bin
strings /var/tmp/cache.bin | head -20
sha256sum /var/tmp/cache.bin | tee ~/case9/cache.bin.sha256
sudo lsof -- /var/tmp/cache.bin
```

**Command/flag notes:** `find / -xdev` stays on the root filesystem; `-size +50M` finds files larger than 50 MiB; `-printf` prints size and path; `sort -nr` sorts largest first; `du -ah` shows disk usage; `xxd -l 128` shows the first 128 bytes; `lsof -- file` checks whether a process has the file open.

---

## B10. Analyze Linux memory dump

```bash
sudo apt update
sudo apt install -y volatility3 binutils dwarf2json xz-utils
mkdir -p ~/case10
cp /var/tmp/memdump.lime ~/case10/
cd ~/case10
sha256sum memdump.lime
strings memdump.lime | grep -iE 'password|flag|http|api_key' | sort -u | head -50
strings -t d memdump.lime | grep -i 'MyMemoryPassword'
strings memdump.lime | grep -E 'FLAG\{[^}]+\}'
vol -f memdump.lime banners.Banners
vol -f memdump.lime linux.pslist.PsList
vol -f memdump.lime linux.pstree.PsTree
vol -f memdump.lime linux.psaux.PsAux
vol -f memdump.lime linux.sockstat.Sockstat
vol -f memdump.lime linux.bash.Bash
```

Use `volatility3` instead of `vol` if the `vol` command is absent.

If Linux plugins fail with a missing symbol-table or ISF error, record the error and identify the kernel banner:

```bash
vol -f memdump.lime banners.Banners
vol isfinfo.IsfInfo | head -50
```

If the exact debug kernel file is available, generate and install a local ISF symbol table:

```bash
sudo find /usr/lib/debug/boot -name "vmlinux-$(uname -r)"
sudo test -f /boot/System.map-$(uname -r) && echo 'System.map exists'
sudo dwarf2json linux --elf /usr/lib/debug/boot/vmlinux-$(uname -r) --system-map /boot/System.map-$(uname -r) | xz -c > kali-$(uname -r).json.xz
mkdir -p ~/.local/share/volatility3/symbols/linux
cp kali-$(uname -r).json.xz ~/.local/share/volatility3/symbols/linux/
vol -f memdump.lime linux.pslist.PsList
```

**Command/flag notes:** `strings` is the minimum reliable triage path when Volatility symbols are missing; `strings -t d` prints decimal offsets; `banners.Banners` identifies Linux kernel banners; Linux plugins require an exact Volatility 3 ISF symbol table; `dwarf2json` creates ISF JSON; `xz -c` compresses it to `.json.xz` for Volatility.

---

## Minimal Role B report checklist

For every task, record:

- VM/user investigated and date/time.
- Commands executed and important output.
- Evidence path and SHA-256 hash.
- Finding summary: what was hidden, where it was found, and how it was verified.
- Any failed command and the correction used.
