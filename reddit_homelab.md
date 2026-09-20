# Turning JioPC into a Persistent Cloud Homelab Node: 8 vCPU Xeon, 16GB RAM, Native Shell Access & Bypassing the 15-Minute Disconnect

Hey r/homelab,

If you are always on the hunt for compute nodes to expand your homelab cluster without racking up high cloud bills or electricity meters, you might have heard of **JioPC**—a cloud desktop service recently launched in India.

On paper, it is marketed as a simple virtual desktop for casual browsing. But once you tear down the underlying hypervisor, it turns out to be provisioned with substantial enterprise compute:
- **CPU**: 8 vCPUs (Intel Xeon Platinum 8370C @ 2.80GHz, Ice Lake-SP) with AVX-512 support.
- **Memory**: 16 GB RAM (DDR4).
- **Storage**: Enterprise NFSv4.1 network array (580+ MB/s sequential writes) + 2.65 GB/s `/dev/shm` RAM disk.
- **Network**: Multi-gigabit Azure cloud fabric datacenter pipe.

However, anyone who tried using it as a persistent homelab runner or remote dev server quickly ran into **three hard roadblocks**:
1. **No Direct Shell Access**: There is no inbound SSH, and terminal emulators (`gnome-terminal`, `xterm`) are omitted from the graphical environment.
2. **Double NAT / Firewall**: Inbound ports are completely blocked by carrier-grade NAT.
3. **The 15-Minute Death Clock**: Disconnecting your client, closing your browser tab, or dropping connection starts a strict 15-minute countdown. If a client does not reconnect within 15 minutes, the desktop session is torn down and background processes are killed.

Over the past two weeks, I reverse-engineered the platform's session lifecycle and built an open-source solution to transform this locked-down instance into a **fully persistent, 24/7 headless/hybrid homelab node**.

Here is the complete architectural guide, sysadmin setup, and homelab integration walkthrough.

*(Note: In accordance with subreddit guidelines against link-farming, the open-source GitHub daemon and one-liner setup scripts are shared in the first comment below).*

---

## 1. Step 1: Getting an Unrestricted Host Shell

JioPC omitted terminal launchers from the Start Menu, but the underlying system disk still houses pre-installed system Flatpaks (`/var/lib/flatpak/app`).

Specifically, **PuTTY** (`uk.org.greenend.chiark.sgtatham.putty`) is installed on the base image, which includes **`pterm`**—a pure GTK X11 terminal emulator. Because it has host execution capabilities, launching it with `flatpak-spawn --host bash` spawns a direct, unrestricted bash shell on the underlying host OS.

### 30-Second GUI Setup via File Manager (Thunar):
Since client-side operating systems (macOS, Fedora, Ubuntu) capture hotkeys like `Alt + F2` or `Ctrl + Alt + T`:
1. Open **File Manager** (double-click "Computer" on the desktop).
2. Go to **Edit → Configure custom actions...** and click **+** (Add).
3. Under **Basic**, set:
   - **Name**: `Terminal`
   - **Command**:
     ```bash
     flatpak run --command=pterm uk.org.greenend.chiark.sgtatham.putty -e flatpak-spawn --host bash
     ```
4. Under **Appearance Conditions**, check **Directories**. Click **OK** → **Close**.
5. **Right-click anywhere inside File Manager → click `Terminal`**.

You now have a full native bash terminal on your 8-core Xeon VM.

---

## 2. The Core Challenge: The 15-Minute Disconnect Timeout

### Why Naive Approaches Fail:
In a typical Linux server, keeping background tasks alive is trivial: you run `tmux`, `screen`, or execute `loginctl enable-linger $USER`.

On JioPC, this does not work. The infrastructure monitors **display socket connection states**:
1. When your RDP or browser client disconnects, the display server registers an empty client socket.
2. An internal session timer arms a 15-minute countdown.
3. Simultaneously, the cloud hypervisor detects the orphaned session. If no reconnection occurs before the 15-minute mark, the session is terminated and the compute node is deprovisioned.
4. Traditional mouse-jiggler scripts fail because they run *inside* the desktop, while the orchestrator monitors the network display socket.

---

## 3. The Architecture: Autonomous User-Space Session Keeper

To turn this into a persistent homelab worker, I engineered a lightweight Python daemon (`xrdp-session-keeper`) running under `systemd --user` (requires **zero root/sudo privileges**):

```
+-------------------------------------------------------------------------+
|                          JioPC Compute Node                             |
|                                                                         |
|  +--------------------+        User Disconnects                         |
|  | Real Client Socket | ----------------------------+                   |
|  +--------------------+                             |                   |
|                                                     v                   |
|  +--------------------+                   +--------------------+        |
|  | xrdp-session-keeper| =================> |  Loopback Display  |        |
|  |   Daemon (User)    |  Auto-Latches     |    Socket Latch    |        |
|  +--------------------+                   +--------------------+        |
|           |                                         |                   |
|           | Periodic Session Heartbeat              v                   |
|           v                               +--------------------+        |
|  +--------------------+                   | Local Display & VM |        |
|  | Cloud Orchestrator |                   |    Stays Alive     |        |
|  | (Prevents Kill)    |                   |    24/7 / 365      |        |
|  +--------------------+                   +--------------------+        |
+-------------------------------------------------------------------------+
```

### How It Works:
1. **Loopback Display Latch**: The moment you disconnect your client, the daemon senses the severed connection and immediately binds a local loopback handler to the display server's UNIX socket. The display server sees an active client and suspends the 15-minute deprovisioning timer.
2. **Session Continuity Heartbeat**: It emits non-intrusive continuity pulses so the cloud hypervisor recognizes the instance as active and assigned to your account.
3. **Seamless Handover**: The second you reconnect from your actual browser or RDP client, the daemon detects the incoming client, releases the loopback latch, and hands back the display.
4. **Pointer Grab Sanitization**: The daemon resets X11 window manager grabs during transitions, ensuring mouse clicks, scrolling, and keyboard focus never freeze.

### Deployment (One-Liner):
Clone the repo and run the automated installer:
```bash
git clone <REPO_URL_IN_FIRST_COMMENT>
cd jiopc-session-keeper && ./install.sh
```
It sets up `systemd --user`, activates user linger, and starts the service. You can disconnect your browser or client indefinitely—your background jobs, containers, and server processes will stay running.

---

## 4. Turning the Instance into a Functional Homelab Node

With persistent uptime and a rootless bash environment, here is how to integrate it into your homelab:

### A. Ingress & Remote Access (Bypassing Carrier NAT)
Since you don't get a public IPv4, connect it directly to your homelab mesh network:

1. **Tailscale (Userspace Mode)**:
   You can run Tailscale without root using userspace networking:
   ```bash
   # Download static tailscale binary to ~/.local/bin
   tailscaled --tun=userspace-networking --socks5-server=localhost:1055 &
   tailscale up --authkey=tskey-auth-...
   ```
   Now the node is an addressable IP on your Tailscale mesh!

2. **Cloudflare Tunnels (`cloudflared`)**:
   Perfect for exposing local web dashboards, APIs, or dev instances to your domain without port forwarding:
   ```bash
   cloudflared tunnel run <YOUR_TUNNEL_NAME>
   ```

### B. Remote Development: Headless VS Code (`code-server`)
Instead of using the web GUI, turn the VM into a remote browser IDE:
```bash
curl -fsSL https://code-server.dev/install.sh | sh -s -- --prefix ~/.local
~/.local/bin/code-server --port 8080 --auth password
```
Pair this with Cloudflare Tunnels or Tailscale to access a full VS Code environment with 8 cores and 16 GB RAM from anywhere.

### C. Containerized Workloads (Rootless Podman / Proot)
Because sudo access is restricted, run containers via rootless Podman or `proot-distro` / `nix-portable`:
- Run automation workers, scrapers, bot runners, and cron jobs.
- Heavy build farm: Compile C/Rust projects leveraging all 8 vCPUs with native Ice Lake AVX-512 instructions (`-march=icelake-server -O3 -j8`).

---

## 5. Homelab Gotchas & Resource Guidelines

1. **0 MB Swap Alert**: The VM has 16 GB of physical RAM but **zero swap**. If your workload hits ~15 GB RSS, the Linux kernel OOM-killer will terminate the largest process instantly. Keep individual process memory footprints below 10 GB RSS or configure zram if needed.
2. **RAM Disk for Ephemeral I/O**: Utilize `/dev/shm` (in-memory `tmpfs` clocked at **2.65 GB/s**) for scratch files, build intermediates, or SQLite databases to avoid network NFS overhead.
3. **Pre-Installed Tooling**: VSCodium, Geany, Kate, and Code::Blocks are pre-installed in `/var/lib/flatpak/app` and can be launched or restored to the desktop anytime.

---

## Summary

With loopback display latching and user-space systemd automation, JioPC transforms from a restricted virtual desktop into an 8-core Xeon compute node capable of 24/7 background workloads.

*(Check the first comment for the open-source repository, daemon code, and systemd units).*

Happy homelabbing!
