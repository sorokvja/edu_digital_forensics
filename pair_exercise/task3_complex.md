# Task 3 — HTTP Credential Capture and Analysis

## Goal

Create and investigate a `.pcapng` file that contains a cleartext HTTP `POST` request with test credentials.

Final evidence expected from Role A:

```text
~/task3/login_capture.pcapng
```

Expected credential data inside the capture:

```text
username=students_a&password=TestPass2024
```

## What was wrong in the previous sequence

The earlier sequence did not reliably match the desired outcome for three reasons:

1. `python3 -m http.server` is a static file server. It serves files, but it does not implement a real login `POST` handler.
2. `tshark` failed before the `curl` request was sent because it could not open the output file. After that failure, no capture process was running.
3. Traffic to `127.0.0.1` must be captured on the loopback interface `lo`. Capturing on `eth0` or another external interface will show nothing for this local test.

The corrected sequence below starts a POST-capable local HTTP server, verifies the server, starts capture before sending credentials, writes the capture to `/tmp` to avoid permission problems, then moves the finished `.pcapng` into `~/task3`.

---

# Role A activities — create the evidence capture

Run Role A commands in one terminal unless explicitly noted.

## A1. Install tools and prepare the workspace

```bash
# Purpose: Refresh the local APT package index before installing tools.
# Flags/options: none.
sudo apt update
```

```bash
# Purpose: Install command-line Wireshark tools and curl.
# Flags/options: DEBIAN_FRONTEND=noninteractive avoids installer prompts; -y automatically confirms installation.
sudo DEBIAN_FRONTEND=noninteractive apt install -y tshark curl
```

```bash
# Purpose: Create the Task 3 working directory and web-server directory.
# Flags/options: -p creates parent directories as needed and does not fail if the directory already exists.
mkdir -p ~/task3/www
```

```bash
# Purpose: Move into the web-server directory.
# Flags/options: none.
cd ~/task3/www
```

## A2. Create a POST-capable local HTTP login server

```bash
# Purpose: Create a minimal local HTTP server that serves /login.html and accepts POST /login.
# Flags/options: > writes to a file; <<'PY' starts a quoted heredoc, so shell variables are not expanded inside the Python code.
cat > ~/task3/www/login_lab_server.py <<'PY'
#!/usr/bin/env python3
from http.server import BaseHTTPRequestHandler, HTTPServer

HOST = "127.0.0.1"
PORT = 8099

LOGIN_PAGE = b"""<!doctype html>
<html>
  <head><title>Task 3 Login</title></head>
  <body>
    <h1>Task 3 Training Login</h1>
    <form method="POST" action="/login">
      <label>Username: <input name="username"></label><br>
      <label>Password: <input name="password" type="password"></label><br>
      <button type="submit">Login</button>
    </form>
  </body>
</html>
"""

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path in ("/", "/login.html"):
            self.send_response(200)
            self.send_header("Content-Type", "text/html; charset=utf-8")
            self.send_header("Content-Length", str(len(LOGIN_PAGE)))
            self.end_headers()
            self.wfile.write(LOGIN_PAGE)
        else:
            self.send_error(404, "Not found")

    def do_POST(self):
        if self.path == "/login":
            length = int(self.headers.get("Content-Length", "0"))
            self.rfile.read(length)  # Read the body so the HTTP transaction completes cleanly.
            response = b"Login received for Task 3 training capture.\n"
            self.send_response(200)
            self.send_header("Content-Type", "text/plain; charset=utf-8")
            self.send_header("Content-Length", str(len(response)))
            self.end_headers()
            self.wfile.write(response)
        else:
            self.send_error(404, "Not found")

    def log_message(self, fmt, *args):
        print(f"{self.client_address[0]} - {fmt % args}", flush=True)

HTTPServer((HOST, PORT), Handler).serve_forever()
PY
```

```bash
# Purpose: Make the local server file executable.
# Flags/options: +x adds execute permission for the owner/group/others according to the current umask.
chmod +x ~/task3/www/login_lab_server.py
```

```bash
# Purpose: Remove stale files from previous failed attempts.
# Flags/options: -f removes without prompting and does not complain if files do not exist.
rm -f ~/task3/http_server.log ~/task3/http_server.pid ~/task3/dumpcap.log ~/task3/dumpcap.pid ~/task3/login_capture.pcapng
```

```bash
# Purpose: Remove an old temporary capture even if it is owned by root.
# Flags/options: -f removes without prompting and does not complain if the file does not exist.
sudo rm -f /tmp/task3_login_capture.pcapng
```

```bash
# Purpose: Start the local HTTP server in the background.
# Flags/options: > redirects standard output to a log file; 2>&1 sends standard error to the same log; & runs the server in the background.
python3 ~/task3/www/login_lab_server.py > ~/task3/http_server.log 2>&1 &
```

```bash
# Purpose: Save the server process ID so it can be stopped later.
# Flags/options: $! expands to the PID of the most recently started background process; > writes that PID to a file.
echo $! > ~/task3/http_server.pid
```

```bash
# Purpose: Give the server a moment to start.
# Flags/options: none.
sleep 1
```

```bash
# Purpose: Verify that the server is listening on TCP port 8099.
# Flags/options: -l shows listening sockets; -t shows TCP sockets; -n keeps numeric addresses/ports; -p shows the owning process where permitted.
ss -ltnp | grep ':8099'
```

```bash
# Purpose: Download the login page once to confirm that the server works before capturing.
# Flags/options: -sS runs silently but still shows errors; -o writes the response body to the named file.
curl -sS http://127.0.0.1:8099/login.html -o /tmp/task3_login_page.html
```

```bash
# Purpose: Confirm that the downloaded page contains an HTML form.
# Flags/options: -i makes grep case-insensitive.
grep -i '<form' /tmp/task3_login_page.html
```

Expected output should include something similar to:

```text
<form method="POST" action="/login">
```

## A3. Start packet capture before sending the credentials

```bash
# Purpose: Refresh sudo authentication before starting the capture in the background.
# Flags/options: -v validates sudo credentials and updates the cached sudo timestamp.
sudo -v
```

```bash
# Purpose: Start packet capture on the loopback interface and write a pcapng file to /tmp.
# Flags/options: -i lo captures loopback traffic; -f applies a capture filter before packets are saved; -a duration:20 stops automatically after 20 seconds; -w writes the capture file; > writes tool output to a log; 2>&1 merges errors into the log; & backgrounds the capture.
sudo dumpcap -i lo -f 'tcp port 8099' -a duration:20 -w /tmp/task3_login_capture.pcapng > ~/task3/dumpcap.log 2>&1 &
```

```bash
# Purpose: Save the capture process ID so it can be checked or waited on.
# Flags/options: $! expands to the PID of the most recently started background process; > writes that PID to a file.
echo $! > ~/task3/dumpcap.pid
```

```bash
# Purpose: Give dumpcap time to attach to the interface before traffic is generated.
# Flags/options: none.
sleep 2
```

```bash
# Purpose: Confirm that the capture process is still running before sending the POST request.
# Flags/options: -p selects the PID to inspect; -o selects output columns.
ps -p "$(cat ~/task3/dumpcap.pid)" -o pid,stat,cmd
```

```bash
# Purpose: If the previous command showed no running process, read the capture log before continuing.
# Flags/options: none.
cat ~/task3/dumpcap.log
```

Do not send the `curl` request until the capture process is running. If the capture process exited, fix the error first. A common fixed error is writing to `/tmp/task3_login_capture.pcapng` instead of directly to `~/task3/login_capture.pcapng`.

## A4. Generate the cleartext HTTP POST request

```bash
# Purpose: Send the test credentials in a deterministic HTTP POST request.
# Flags/options: --http1.1 forces HTTP/1.1; -sS hides progress but shows errors; -X POST explicitly sets the method; -H sets the Content-Type header; --data sends the form body and also implies POST; -o /dev/null discards the response body.
curl --http1.1 -sS -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'username=students_a&password=TestPass2024' \
  http://127.0.0.1:8099/login \
  -o /dev/null
```

```bash
# Purpose: Wait until dumpcap reaches its 20-second auto-stop and finalizes the pcapng file.
# Flags/options: wait waits for the background process whose PID is stored in the file.
wait "$(cat ~/task3/dumpcap.pid)"
```

```bash
# Purpose: Show capture-tool output, including packet count or errors.
# Flags/options: none.
cat ~/task3/dumpcap.log
```

```bash
# Purpose: Change ownership of the root-created capture to the current user.
# Flags/options: chown USER:GROUP changes file owner and group; $USER expands to the current username.
sudo chown "$USER:$USER" /tmp/task3_login_capture.pcapng
```

```bash
# Purpose: Move the finished evidence file into the Task 3 directory.
# Flags/options: none.
mv /tmp/task3_login_capture.pcapng ~/task3/login_capture.pcapng
```

```bash
# Purpose: Confirm that the final pcapng exists and is not empty.
# Flags/options: -l uses long listing format; -h prints human-readable sizes.
ls -lh ~/task3/login_capture.pcapng
```

## A5. Verify that the capture contains the POST credentials

```bash
# Purpose: Count packets related to TCP port 8099.
# Flags/options: -r reads a capture file; -Y applies a display filter; wc -l counts output lines.
tshark -r ~/task3/login_capture.pcapng -Y 'tcp.port == 8099' | wc -l
```

The count should be greater than `0`. If it is `0`, the POST was not captured. Check that capture used `-i lo`, the server URL used `127.0.0.1`, and the `curl` command was run while `dumpcap` was active.

```bash
# Purpose: Extract the HTTP POST request and form fields from the capture.
# Flags/options: -r reads the capture; -d tcp.port==8099,http forces HTTP decoding on non-standard port 8099; -Y filters to POST requests; -T fields prints selected fields only; each -e selects one field.
tshark -r ~/task3/login_capture.pcapng \
  -d tcp.port==8099,http \
  -Y 'http.request.method == "POST"' \
  -T fields \
  -e frame.time \
  -e tcp.stream \
  -e http.host \
  -e http.request.uri \
  -e urlencoded-form.key \
  -e urlencoded-form.value
```

Expected evidence should show the `/login` request and the values `students_a` and `TestPass2024`.

```bash
# Purpose: Raw fallback check for cleartext strings in the pcapng file.
# Flags/options: grep -i ignores case; -E enables extended regular expressions.
strings ~/task3/login_capture.pcapng | grep -iE 'username|password|students_a|TestPass|/login'
```

## A6. Stop the local server and prepare the handoff

```bash
# Purpose: Stop the local HTTP server after the capture is complete.
# Flags/options: kill sends the default SIGTERM signal to the PID stored in the file.
kill "$(cat ~/task3/http_server.pid)"
```

```bash
# Purpose: Calculate a SHA-256 hash of the evidence file for integrity documentation.
# Flags/options: none.
sha256sum ~/task3/login_capture.pcapng
```

Role A hands off this file to Role B:

```text
~/task3/login_capture.pcapng
```

---

# Role B activities — investigate the capture

## B1. Prepare a case folder and preserve a working copy

```bash
# Purpose: Create a separate analysis folder.
# Flags/options: -p creates parent directories as needed and does not fail if the folder already exists.
mkdir -p ~/case3
```

```bash
# Purpose: Copy the received capture into the analysis folder.
# Flags/options: none.
cp ~/task3/login_capture.pcapng ~/case3/
```

```bash
# Purpose: Move into the case folder.
# Flags/options: none.
cd ~/case3
```

```bash
# Purpose: Record the evidence hash before analysis.
# Flags/options: none.
sha256sum login_capture.pcapng
```

```bash
# Purpose: Confirm the file type.
# Flags/options: none.
file login_capture.pcapng
```

## B2. Confirm that packets were captured

```bash
# Purpose: Count packets related to TCP port 8099.
# Flags/options: -r reads the capture; -Y applies a display filter; wc -l counts matching packet lines.
tshark -r login_capture.pcapng -Y 'tcp.port == 8099' | wc -l
```

```bash
# Purpose: Display the first few packets involving TCP port 8099.
# Flags/options: -r reads the capture; -Y filters packets; head limits output to the first lines.
tshark -r login_capture.pcapng -Y 'tcp.port == 8099' | head
```

If packets exist but the `http` filter shows nothing, the traffic is probably not being decoded as HTTP because port `8099` is non-standard. Use `-d tcp.port==8099,http` in the next commands.

## B3. Extract the submitted HTTP credentials with tshark

```bash
# Purpose: Extract the HTTP POST request and URL-encoded form values.
# Flags/options: -r reads the capture; -d tcp.port==8099,http forces HTTP decoding on port 8099; -Y filters to HTTP POST requests; -T fields outputs only selected fields; each -e names a field to print.
tshark -r login_capture.pcapng \
  -d tcp.port==8099,http \
  -Y 'http.request.method == "POST"' \
  -T fields \
  -e frame.time \
  -e tcp.stream \
  -e http.host \
  -e http.request.uri \
  -e urlencoded-form.key \
  -e urlencoded-form.value
```

Expected finding:

```text
username = students_a
password = TestPass2024
URL      = http://127.0.0.1:8099/login
Method   = POST
```

```bash
# Purpose: Identify the TCP stream number for the POST request.
# Flags/options: -r reads the capture; -d forces HTTP decoding; -Y filters to POST requests; -T fields outputs selected fields; -e tcp.stream prints the stream number.
tshark -r login_capture.pcapng \
  -d tcp.port==8099,http \
  -Y 'http.request.method == "POST"' \
  -T fields \
  -e tcp.stream
```

```bash
# Purpose: Follow TCP stream 0 as ASCII to view the raw HTTP request and response.
# Flags/options: -r reads the capture; -d forces HTTP decoding; -q suppresses normal packet output; -z follow,tcp,ascii,0 prints TCP stream 0 in ASCII.
tshark -r login_capture.pcapng \
  -d tcp.port==8099,http \
  -q \
  -z follow,tcp,ascii,0
```

If the previous command reports a different stream number in your capture, replace the final `0` with that stream number.

## B4. Optional Wireshark GUI workflow

```bash
# Purpose: Open the capture in Wireshark for graphical analysis.
# Flags/options: & starts Wireshark in the background so the terminal remains usable.
wireshark login_capture.pcapng &
```

In Wireshark:

1. Use display filter: `tcp.port == 8099`.
2. If HTTP is not decoded automatically, use **Analyze → Decode As... → TCP port 8099 → HTTP**.
3. Use display filter: `http.request.method == "POST"`.
4. Right-click the POST packet and choose **Follow → TCP Stream** or **Follow → HTTP Stream**.
5. Document the username, password, request URI, timestamp, and evidence hash.

## B5. Fallback string search

```bash
# Purpose: Search the pcapng for cleartext credential strings if protocol decoding fails.
# Flags/options: grep -i ignores case; -E enables extended regular expressions.
strings login_capture.pcapng | grep -iE 'username|password|students_a|TestPass|/login'
```

## Common errors and fixed actions

| Symptom | Likely cause | Fixed action |
|---|---|---|
| `Permission denied` when writing the capture | Capture tool cannot write to the selected path | Write to `/tmp/task3_login_capture.pcapng`, then `sudo chown` and `mv` it to `~/task3`. |
| Capture file is missing | Capture process exited before the POST request | Check `cat ~/task3/dumpcap.log`; verify `ps -p "$(cat ~/task3/dumpcap.pid)"` before sending the POST. |
| Packet count is `0` | Wrong interface or request sent outside capture window | Use `-i lo`, use URL `http://127.0.0.1:8099/login`, and send `curl` while `dumpcap` is running. |
| `http.request.method == "POST"` shows nothing but packets exist | Port `8099` is not decoded as HTTP automatically | Add `-d tcp.port==8099,http` to `tshark` commands or use Wireshark **Decode As**. |
| Server does not start | Port `8099` is already in use | Run `ss -ltnp | grep ':8099'`, identify the old PID, and stop it with `kill <PID>`. |
| Browser/curl shows `connection refused` | Local server is not running | Start `python3 ~/task3/www/login_lab_server.py` and verify with `ss -ltnp | grep ':8099'`. |

