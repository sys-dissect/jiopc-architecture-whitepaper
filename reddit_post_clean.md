# Reverse-Engineering JioPC (Part 2): Native Terminal Access, Unhiding Pre-Installed IDEs & Persistent Sessions

Hey r/developersIndia,

Two weeks ago, I shared an architectural deep dive analyzing the compute, storage, and display stack of JioPC (8-core Intel Xeon Platinum 8370C on Azure, 581 MB/s NFS array, AVX-512 support).

Following that post, three practical questions came up from developers trying to set up persistent developer environments on their instances:

1. **Terminal Access Was Obscured**: Standard terminals (`gnome-terminal`, `xterm`) are omitted from `/usr/bin`, and client-side operating systems (like Fedora, Ubuntu GNOME, or macOS) capture hotkeys like `Alt + F2` before they can reach the remote VM.
2. **IDEs Disappeared from the Store**: VSCodium and other developer tools no longer appear in the JioStore GUI catalog.
3. **The 15-Minute Disconnect Timeout**: Disconnecting your client, closing the browser tab, or a temporary network drop triggers an automatic 15-minute countdown that terminates your desktop session and resets your environment. Standard user-space commands like `loginctl enable-linger` or cursor-moving scripts cannot prevent this teardown.

Here is a straightforward, solution-oriented guide on how to get a native GTK terminal using only standard GUI menus (zero hotkeys, zero downloads), how to unhide and launch all pre-installed IDEs, and how to keep your desktop session and background tasks running permanently across disconnections.

*(Note: In accordance with subreddit rules, the open-source repository link and one-line setup script are shared in the first comment below).*

---

## 1. Native Terminal Access (100% GUI — No Hotkeys, No Downloads)

### The Pre-Installed Tool: `pterm`
While desktop catalog shortcuts were removed, the underlying system-wide Flatpak storage (`/var/lib/flatpak/app`) still contains pre-installed packages. 

Specifically, **PuTTY** (`uk.org.greenend.chiark.sgtatham.putty`) is present on disk. PuTTY includes **`pterm`**—a standalone, pure-GTK X11 terminal emulator. When spawned with `-e flatpak-spawn --host bash`, it attaches directly to an unrestricted host Linux bash shell without needing SSH, web wrappers, or external IDEs.

### 30-Second Setup via File Manager (Thunar):
Since keyboard shortcuts like `Alt + F2` are captured by host operating systems on Linux/Mac, use this standard mouse-driven configuration:

1. Open **File Manager** (double-click "Computer" or "Downloads" on the desktop).
2. In the top menu bar, click: **Edit → Configure custom actions...**
3. Click the **+** (Add) button on the right.
4. Under the **Basic** tab:
   - **Name**: `Terminal`
   - **Command**:
     ```bash
     flatpak run --command=pterm uk.org.greenend.chiark.sgtatham.putty -e flatpak-spawn --host bash
     ```
5. Under the **Appearance Conditions** tab:
   - Check **Directories** (or leave all checked).
6. Click **OK**, then click **Close**.

👉 **Done!** Now, simply **right-click anywhere inside the File Manager and select `Terminal`**. A native GTK terminal immediately opens on the host.

*(Optional)* Once your terminal is open, run this one-liner to place a permanent **Terminal** shortcut directly on your desktop:
```bash
cat << 'EOF' > ~/Desktop/terminal.desktop
[Desktop Entry]
Version=1.0
Name=Terminal
Exec=flatpak run --command=pterm uk.org.greenend.chiark.sgtatham.putty -e flatpak-spawn --host bash
Icon=utilities-terminal
Type=Application
EOF
chmod +x ~/Desktop/terminal.desktop
```

---

## 2. Unhiding the Pre-Installed IDEs (Already on Disk)

Even though the JioStore catalog no longer lists IDEs, **they are already installed on the system disk** in the system-wide Flatpak repository (`/var/lib/flatpak/app`). They simply lack exported `.desktop` launchers in the Start Menu.

The following full IDEs are already present out of the box:

| Application | Flatpak ID | Description |
| :--- | :--- | :--- |
| **VSCodium** | `com.vscodium.codium` | Full, telemetry-free VS Code build (v1.104). |
| **Code - OSS** | `com.visualstudio.code-oss` | Open-source VS Code base (v1.74). |
| **Geany** | `org.geany.Geany` | Fast, lightweight IDE with integrated VTE terminal dock (v2.1). |
| **Kate** | `org.kde.kate` | Advanced KDE editor with integrated Konsole terminal (`F4`). |
| **Code::Blocks** | `org.codeblocks.codeblocks` | Full C/C++ development IDE with GCC/GDB tooling (v25.03). |
| **pgAdmin 4** | `org.pgadmin.pgadmin4` | Complete PostgreSQL database management and query GUI. |

### How to Launch Them:

#### Option A: Directly from your Terminal
Once you open a terminal via the `pterm` method above, you can launch any IDE instantly:
```bash
flatpak run com.vscodium.codium &
flatpak run org.geany.Geany &
flatpak run org.kde.kate &
flatpak run org.codeblocks.codeblocks &
```

#### Option B: Permanently Restore All IDE Icons to Desktop & Start Menu
Run this one-liner in your terminal to automatically copy their official desktop launchers into your user directories:
```bash
mkdir -p ~/.local/share/applications ~/Desktop
cp /var/lib/flatpak/app/*/current/active/export/share/applications/*.desktop ~/.local/share/applications/ 2>/dev/null || true
cp /var/lib/flatpak/app/*/current/active/export/share/applications/*.desktop ~/Desktop/ 2>/dev/null || true
chmod +x ~/Desktop/*.desktop
```
Instantly:
- Clickable icons for **VSCodium**, **Geany**, **Kate**, and **Code::Blocks** appear on your **Desktop**.
- They are also restored into the **Start Menu** under the **Development** category!

#### Option C: Without Terminal via File Manager
If you prefer not opening a terminal first, you can use the same Thunar Custom Action method from Step 1:
- Set Command to: `flatpak run com.vscodium.codium`
- Right-click anywhere in File Manager to launch VSCodium directly!

---

## 3. Solving the 15-Minute Session Timeout

### Why Basic Workarounds Failed
Many developers noticed that even with `loginctl enable-linger $USER` enabled, disconnecting for more than 15 minutes caused the graphical session and background jobs to terminate. 

This happens because the cloud environment monitors **active connection sockets**, not just local user idle time:
1. When your RDP or browser connection closes, the local display server detects an empty connection socket and arms an internal 15-minute countdown.
2. In parallel, the cloud orchestrator detects the disconnected state and schedules the compute node for deprovisioning if no reconnection occurs within 15 minutes.
3. Because both the display server and the orchestrator rely on network socket state, cursor-jiggler scripts and user linger flags are bypassed.

---

### The Solution: Autonomous User-Space Session Keeper

To maintain session continuity without requiring elevated privileges, we developed a lightweight Python daemon that runs under `systemd --user`:

1. **Loopback Display Latch**:
   The moment your client disconnects, the daemon attaches a local loopback handler to the display server's UNIX socket, satisfying its connection check and disengaging the local 15-minute teardown timer.
2. **Session Continuity Heartbeat**:
   The daemon emits background session continuity signals to ensure the remote orchestrator keeps your compute node alive and assigned to your account.
3. **Seamless Handover**:
   The instant you reconnect from your real client, the daemon immediately yields the display socket, providing a smooth transition back to your active session.
4. **Pointer & Window Manager Sanitization**:
   Across disconnect and reconnect cycles, the daemon automatically refreshes window manager grabs to prevent mouse clicks and scrolling from freezing.
5. **Idle Screen Suppression**:
   While actively connected, it sends periodic keep-alives to prevent unwanted screen blanking during long reading or compile sessions.

---

## 4. Quick Installation & Verification

Once you have opened a terminal using the File Manager method above, you can install the daemon with a single command (requires **zero sudo / root**):

```bash
# Clone the repository (direct link in the first comment below):
git clone <REPO_URL_FROM_FIRST_COMMENT>
cd jiopc-session-keeper && ./install.sh
```

The installer configures `systemd --user`, sets up user lingering, and enables the service immediately.

### How to Test It:
1. Start a long-running task, build, or keep an editor window open.
2. Close your RDP client or browser tab.
3. Wait **25 to 30 minutes** (well past the default 15-minute threshold).
4. Log back in — your desktop session, windows, running terminals, and node uptime will be completely intact!
5. Check daemon status anytime:
   ```bash
   systemctl --user status xrdp-session-keeper
   tail -n 30 ~/.local/state/session-keeper.log
   ```

---

The code is completely open-source under MIT, non-root, and audited for privacy. **See the first comment below for the GitHub repository.**

If you are using JioPC for cloud builds, development, or long-running workloads, test this out and let us know your experience in the comments!
