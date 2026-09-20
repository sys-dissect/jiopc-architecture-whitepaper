# Defeating Cloud VDI Disconnect Killswitches: Non-Root XRDP Socket Latching, Protocol Reverse-Engineering & 24/7 Session Persistence

Hey r/linuxadmin / r/linux,

Recently, I’ve been reverse-engineering a locked-down enterprise cloud desktop environment (hosted on Azure instances with 8-vCPU Intel Xeon Platinum 8370C chips, 16 GB RAM, and persistent NFS mounts).

Like many corporate virtual desktop infrastructures (VDI) based on XRDP and proprietary brokers, it enforces an aggressive **15-minute disconnect killswitch**: closing your client, dropping a packet, or closing a browser tab starts a strict 900-second countdown. Once it hits zero, the X11 server is terminated with `SIGTERM`, all background tasks are killed, and the ephemeral compute node is recycled.

Standard sysadmin workarounds fail against this setup:
- `loginctl enable-linger $USER` preserves the systemd user instance, but cannot prevent the display server from dying and terminating all desktop-spawned processes.
- Cursor jigglers (`xdotool`, mouse movement) only address *connected* idle timeouts; they cannot prevent disconnect teardown because the server checks network sockets, not input events.
- Elevated privileges (`sudo`, root) are strictly absent.

Here is an architectural deep dive into how XRDP enforces disconnect timeouts at the binary and protocol levels, how we reverse-engineered the lifecycle, and how we engineered an unprivileged user-space daemon to latch the display socket, defeat the 900-second killswitch, and maintain 24/7 persistent sessions across node reassignments.

---

## 1. Anatomy of the Teardown: Why Standard Workarounds Fail

The session lifecycle is governed by a **two-tier killswitch**:

```
+--------------------------------------------------------------------------+
|                     Two-Tier Teardown Architecture                       |
|                                                                          |
|  [External Client]                                                       |
|         |                                                                |
|      (Drops)                                                             |
|         v                                                                |
|  +---------------+       Empty Socket       +-------------------------+  |
|  | xorgxrdp      | -----------------------> | Tier 1: Local Disconnect|  |
|  | Display :10   |                          | Timer (900s -> SIGTERM) |  |
|  +---------------+                          +-------------------------+  |
|         |                                                |               |
|   IPC Event                                              v               |
|         v                                   +-------------------------+  |
|  +---------------+     Disconnect Flag      | Xorg Terminates,        |  |
|  | Accops DVM /  | -----------------------> | GUI Process Tree Dies   |  |
|  | Cloud Broker  |                          +-------------------------+  |
|  +---------------+                                       |               |
|         |                                                v               |
|         | 900s Unreconnected                +-------------------------+  |
|         +---------------------------------> | Tier 2: Cloud Hypervisor|  |
|                                             | Destroys Ephemeral Node |  |
|                                             +-------------------------+  |
+--------------------------------------------------------------------------+
```

### Tier 1: Local Display Server (`xorgxrdp`)
`xorgxrdp` manages the X11 screen and listens on a UNIX domain stream socket located at `/var/run/xrdp/<UID>/xrdp_display_<DISPLAY_NUM>`.

When an external client disconnects:
1. `xorgxrdp` detects `POLLHUP`/`EOF` on the display socket.
2. It sets `clientCon = NULL` and arms an internal disconnect timer via the Xorg timer API:
   ```c
   rdpClientConDisconnect: engaging disconnect timer, exit after 900 seconds
   ```
3. Disassembly of `libxorgxrdp.so` reveals the exact mechanism inside `rdpDeferredDisconnectCallback`:
   ```assembly
   000000000000bdd0 <rdpDeferredDisconnectCallback>:
       cmpq   $0x0, 0x368(%rdx)      # Check if clientCon is active (connected)
       jne    bed8                   # If connected -> cancel & disengage timer
       imul   $0x3e8, 0x3d0(%rdx), %eax # timeout_ms = disconnect_timeout_s * 1000
       sub    0x3d4(%rdx), %esi      # elapsed = now - disconnect_time
       cmp    %eax, %esi             # elapsed > timeout_ms?
       ja     be60                   # If exceeded -> teardown!
       # Otherwise re-arm timer for 10,000 ms (0x2710) and return
       ...
       be60: call ErrorF             # "rdpDeferredDisconnectCallback: disconnect timeout exceeded, exiting"
       bebb: call getpid
       bec0: mov  $0xf, %esi         # SIGTERM (15)
       bec7: call kill               # kill(getpid(), SIGTERM)
   ```
   Every 10 seconds, `rdpDeferredDisconnectCallback` verifies whether a client is connected. If no client attaches before 900,000 ms (15 minutes) elapse, it issues a fatal `kill(getpid(), SIGTERM)` to the X server process itself.

### Tier 2: Cloud Orchestrator / Broker Daemon
In parallel, a root-level daemon (`EDC.DVM.LINUXWebApi` / Accops DVM) communicates with the session broker. When the client disconnects, DVM logs a `Disconnect` event. If the cloud controller receives no `Reconnect` notification within 15 minutes, it sends a deallocation instruction to the Azure hypervisor, which destroys the virtual machine and recycles the compute instance.

Because both layers monitor **transport connection sockets**, no amount of desktop-level simulation (e.g. `xdotool mousemove`, audio playback scripts, or `loginctl enable-linger`) will clear the disconnect timers.

---

## 2. Phase 1: Unprivileged Shell Escape via Flatpak IPC

Before tackling the timers, we needed an unrestricted shell. The environment omitted standard terminal emulators (`gnome-terminal`, `xterm`) from `/usr/bin`, and client operating systems (macOS, Linux hosts) intercepted window manager hotkeys (`Alt+F2`, `Ctrl+Alt+T`).

However, system-wide Flatpaks remained present in `/var/lib/flatpak/app`. Specifically, **PuTTY** (`uk.org.greenend.chiark.sgtatham.putty`) was installed.

PuTTY ships with **`pterm`**—a standalone, pure-GTK X11 terminal emulator. Because the Flatpak runtime provides `flatpak-spawn`, invoking:
```bash
flatpak run --command=pterm uk.org.greenend.chiark.sgtatham.putty -e flatpak-spawn --host bash
```
breaks out of the container sandbox and attaches directly to an unrestricted host bash session.

By adding this command as a **Custom Action** in the graphical file manager (Thunar), an interactive rootless host shell can be spawned via a simple right-click anywhere in the desktop environment.

---

## 3. Phase 2: Reverse-Engineering the XRDP Display Protocol

To defeat Tier 1 (the 900-second Xorg killswitch), we investigated whether an unprivileged user process could attach to `/var/run/xrdp/<UID>/xrdp_display_10`.

Inspecting `libxorgxrdp.so`’s client connection parser revealed that a bare socket connection is insufficient: `xorgxrdp` requires an initial protocol handshake before it considers a client validly connected. If a client connects and writes nothing or closes prematurely, `xorgxrdp` drops the socket and resumes the countdown.

Tracing the binary revealed the required handshake:
1. **`XR_MSG_VERSION` (Type: 103, Msg: 301)**: A 26-byte little-endian struct initializing the client connection parameters.
2. **`XR_MSG_INVALIDATE` (Type: 103, Msg: 200)**: A 26-byte little-endian struct defining the bounding screen coordinates (`width` and `height`).

### Struct Layout:
```python
import struct

def build_version_message():
    # Length: 26, Type: 103, Msg: 301, Params: [0, 0, 0, 1], Pad: 0
    return struct.pack("<IHHIIIIH", 26, 103, 301, 0, 0, 0, 1, 0)

def build_invalidate_message(width, height):
    # Length: 26, Type: 103, Msg: 200, Param2: (width << 16) | height, Pad: 0
    param2 = ((width & 0xFFFF) << 16) | (height & 0xFFFF)
    return struct.pack("<IHHIIIIH", 26, 103, 200, 0, param2, 0, 0, 0)
```

When this 52-byte sequence is written to `/var/run/xrdp/<UID>/xrdp_display_10`, `xorgxrdp` disassembles the incoming buffer, populates `clientCon`, and executes:
```text
rdpDeferredDisconnectCallback: connected
rdpDeferredDisconnectCallback: disengaging disconnect timer
```
The local disconnect killswitch is immediately cancelled.

---

## 4. Phase 3: Handover Mechanics & Pointer Sanitization

A critical requirement was ensuring that our loopback latch **does not interfere with the user**. When a real user reconnects from an RDP client or web browser:

1. `xorgxrdp` only allows one active client connection at a time (`rdpClientConGotConnection`).
2. When the real incoming client establishes its socket, `xorgxrdp` marks our loopback socket for disconnection and closes the channel (`nbytes == 0`).
3. The daemon uses non-blocking `select()` to detect socket closure within milliseconds, immediately closes its end, and enters passive `STANDBY`:

```python
# Drain loopback socket until Xorg closes it (upon real client reconnection)
while True:
    r, _, _ = select.select([sock], [], [], 2.0)
    if r:
        nbytes = sock.recv_into(buf)
        if nbytes == 0:
            # Real client attached; yield display cleanly
            sanitize_pointer_state()
            break
```

### Pointer & Window Grab Sanitization:
During abrupt disconnects, X11 pointer grabs (state 6) or active XTest mouse buttons often remain latched in the window manager, causing the desktop to ignore mouse clicks or scrolling when the user returns.

To guarantee zero-friction handovers, the daemon executes a lightweight sanitization routine on every latch attach and yield:
```python
def sanitize_pointer_state():
    # Release stuck XTest mouse buttons
    subprocess.run(["xdotool", "mouseup", "1", "mouseup", "2", "mouseup", "3"], check=False)
    # Replace window manager cleanly to unfreeze any orphaned pointer grabs
    subprocess.Popen(["xfwm4", "--replace"], start_new_session=True)
```

---

## 5. Phase 4: Neutralizing Tier 2 (Cloud Orchestrator Sync)

Defeating local Xorg was only half the battle. If the cloud broker still believed the session was disconnected, the virtual machine would be reclaimed after 15 minutes.

Scanning `/run` revealed a root-owned UNIX socket listening at `/run/accops/dvm/grpc.sock`. Crucially, this socket was provisioned with permissions `srw-rw-rw-` (world-writable).

By inspecting the gRPC service definition, we found the `/XrdpEvent.XrdpEventService/SendSessionChange` RPC endpoint. Constructing raw protobuf bytes allowed us to send unprivileged session state synchronizations directly to the broker:

```python
# Protobuf structure: username (tag 1), display (tag 2), pid (tag 3), state (tag 4)
# State 1 = Reconnect
req_bytes = (
    bytes([1 << 3 | 2]) + encode_varint(len(username)) + username.encode() +
    bytes([2 << 3 | 0]) + encode_varint(display_num) +
    bytes([3 << 3 | 0]) + encode_varint(os.getpid()) +
    bytes([4 << 3 | 0]) + encode_varint(1)
)

with grpc.insecure_channel("unix:/run/accops/dvm/grpc.sock") as channel:
    call_fn = channel.unary_unary("/XrdpEvent.XrdpEventService/SendSessionChange")
    call_fn(req_bytes, timeout=3.0)
```

When AUEMTray and DVM receive `Reconnect`, they immediately forward the event to the cloud controller:
```text
AUEMTray - [AUEMGrpcServer] Received session change event: Reconnect
AUEMTray - [ClientVDIInfoHelper] Syncing connection information with ARS
```
This resets the server-side cloud reclamation countdown. The daemon pulses this signal both during disconnects and periodically (every 60s) while connected to prevent stale state accumulation.

---

## 6. Architecture of the Finished Daemon (`xrdp-session-keeper`)

The final daemon runs entirely in user space under `systemd --user`:

```
+-----------------------------------------------------------------------+
|                       xrdp-session-keeper.py                          |
|                                                                       |
|  [Main Polling Loop]                                                  |
|         |                                                             |
|         +---> Check /proc/net/unix for ESTABLISHED socket on          |
|         |     /var/run/xrdp/<UID>/xrdp_display_<N>                    |
|         |                                                             |
|   +-----+----------------------------------+                          |
|   | Client Connected?                      | Client Disconnected?     |
|   v                                        v                          |
| [STANDBY MODE]                      [LATCH MODE]                      |
| - Poke audio idle socket (every 30s)- Connect to display socket       |
| - Pulse broker Reconnect (every 60s)- Send 52-byte handshake          |
| - Sleep 2s                          - Disengage 900s killswitch       |
|                                     - Emit cloud broker Reconnect     |
|                                     - Wait for real client reconnect  |
|                                     - Sanitize pointer grabs on yield |
+-----------------------------------------------------------------------+
```

### Systemd User Unit (`~/.config/systemd/user/xrdp-session-keeper.service`):
```ini
[Unit]
Description=XRDP Disconnect Timer Defeat Daemon (Session Keeper)
After=default.target

[Service]
Type=simple
ExecStart=%h/.local/venv/bin/python %h/bin/xrdp-session-keeper.py
Restart=always
RestartSec=3
KillMode=process

[Install]
WantedBy=default.target
```

Because user lingering (`loginctl enable-linger $USER`) is enabled, `systemd --user` runs continuously in the background regardless of whether an active GUI session exists. Across ephemeral node reassignments, the daemon auto-heals its linger state and re-attaches to the newly provisioned display socket.

---

## 7. Results & Verification

We battle-tested the setup against sustained disconnections:
- **Baseline Behavior**: Session killed and VM deprovisioned at $T + 900\text{ s}$ ($15\text{ min}$).
- **With Session Keeper Active**: Tested across **continuous 24+ hour disconnections**. Xorg uptime, running terminal processes, and background services (Tailscale userspace node, OpenSSH on port 2222, long-running compilation runs) remained completely uninterrupted.
- **Resource Footprint**: ~8.1 MB RSS memory, <0.1% CPU consumption.

---

## Key Takeaways for Sysadmins & VDI Engineers

1. **`enable-linger` is not a silver bullet**: Linger only controls `systemd-logind` session lifecycle. If the display server (`xorgxrdp`) maintains its own internal disconnect socket logic, it can and will terminate your sessions from within user space.
2. **IPC socket permissions matter**: Leaving orchestrator IPC sockets world-writable (`0666`) allows unprivileged users to synchronize broker states and override session lifecycle rules.
3. **Loopback socket latching works**: Emulating a minimal protocol client over the local UNIX socket is an effective, non-root technique to defeat display server idle/disconnect policies without binary patching.

The complete code, systemd definitions, and protocol handlers are open-sourced under MIT. 

Feedback, edge-case testing, and XRDP protocol insights are welcome!
