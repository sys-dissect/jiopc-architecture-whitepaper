🔗 **Project Repositories & Implementation Details:**

* **GitHub Repository (Daemon & Systemd Units)**: https://github.com/sys-dissect/jiopc-session-keeper
* **Technical Whitepaper & Disassembly Notes**: https://github.com/sys-dissect/jiopc-architecture-whitepaper
* **License**: MIT (100% User-Space / Non-Root)

---

### Technical Summary & Source Code Layout
* `bin/xrdp-session-keeper.py`: Core daemon handling `/var/run/xrdp` UNIX socket latching, 52-byte synthetic handshake generation, and `/run/accops/dvm/grpc.sock` protobuf state synchronization.
* `bin/keep-awake.sh`: Connected audio idle socket keep-alive (`xrdp_idle_timeout_data_flow_*`).
* `systemd/xrdp-session-keeper.service`: Systemd user service unit with automatic linger persistence.

Feel free to inspect the implementation, review the protocol handlers, or open an issue on GitHub!
