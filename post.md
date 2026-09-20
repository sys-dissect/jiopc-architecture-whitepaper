# Guide: Accessing Terminal & Setting Up Developer Workflows on JioPC (After Software Center Updates)

Hey everyone,

Following the earlier architectural analysis of JioPC, a common question from developers setting up their instances was how to get a functional terminal environment now that IDEs like VSCodium or Code-OSS no longer appear in the Jio Software Center catalog.

Here is a breakdown of why this happens, what is actually on disk, and how to get a responsive terminal running directly in your browser or desktop in a few simple steps.

---

### 1. The Pre-Installed IDEs Are Still on Disk

If you checked the software store and couldn't find VSCodium or other IDEs, they haven't been removed from the system. They are pre-installed system-wide in the secondary application storage (`/mnt/sfdisk`), just omitted from the GUI catalog shortcuts.

Once you have command-line access, you can run:

```bash
flatpak list
```

You will see `com.vscodium.codium` is already installed. You can launch it directly with:

```bash
DISPLAY=:10.0 flatpak run com.vscodium.codium &
```

*Tip: If you want a GUI terminal window immediately without downloading anything, **PuTTY** is also pre-installed:*
```bash
DISPLAY=:10.0 flatpak run uk.org.greenend.chiark.sgtatham.putty &
```

---

### 2. Method A: Browser-Based Local Web Terminal (Zero Setup)

Since the default desktop does not include a dedicated desktop terminal emulator (like `xterm` or `gnome-terminal`), you can easily run a local web terminal daemon bound to localhost and view it inside Google Chrome.

#### Step 1: Create a launcher script
Create a file named `launch_terminal.sh` on your Desktop with the following contents:

```bash
#!/bin/bash
# Fetch open-source ttyd web terminal binary
wget -qO /tmp/ttyd https://github.com/tsl0922/ttyd/releases/download/1.7.7/ttyd.x86_64
chmod +x /tmp/ttyd

# Start ttyd on loopback port 9999 in background
killall -9 ttyd 2>/dev/null
nohup /tmp/ttyd -W -p 9999 bash >/tmp/ttyd.log 2>&1 &

# Open in Chrome
sleep 1
google-chrome http://127.0.0.1:9999 &
```

#### Step 2: Make executable and launch
1. Right-click `launch_terminal.sh` on your Desktop -> **Properties** -> **Permissions** -> check **Allow executing file as program**.
2. Double-click the file and click **Execute**.

Google Chrome will launch a tab with a full interactive bash shell connected to the 8-vCPU Xeon instance over local WebSockets.

---

### 3. Method B: Native Python Local Server (No External Binaries)

If you prefer not downloading external utilities, Python 3 is pre-installed (`Python 3.12`). You can run a simple user-space Python script to inspect system stats or stage dev tools directly from your user directory.

---

### 4. Essential Post-Setup Configurations

Once you are in your shell:

#### A. Keep Sessions Alive Across GUI Disconnects
By default, closing the browser window may terminate your session processes after an idle period. To allow background tasks and servers to stay running:

```bash
loginctl enable-linger $USER
```

#### B. Networking / Tailscale Notes
The `/dev/net/tun` device node exists with read/write permissions, but tenant accounts lack the kernel `CAP_NET_ADMIN` capability needed for `ioctl(TUNSETIFF)`. 

Run the daemon `tailscaled` with `--tun=userspace-networking --socket=$HOME/tailscaled.sock`. When bringing the interface up with the CLI client, disable MagicDNS so the system's local forward proxy routing is preserved:

```bash
tailscale --socket=$HOME/tailscaled.sock up --accept-dns=false
```

---

### Detailed Architecture Whitepaper

For anyone interested in the full compute, storage, and memory architecture benchmarks (AVX-512 throughput, sequential NFS write rates, and OpenVINO LLM inference benchmarks), the detailed technical whitepaper has been updated with full setup playbooks.

*(Repository link shared in the comments below to comply with subreddit posting guidelines).*