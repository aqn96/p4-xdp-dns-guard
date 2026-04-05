# P4-XDP DNS Guard

A kernel-level DNS firewall built from scratch on a Raspberry Pi 5, using P4 and eBPF/XDP to intercept and block malicious DNS queries before they leave the machine.

**Author:** Andrew Nguyen (@aqn96)  
**Hardware:** Raspberry Pi 5 (8GB, aarch64)  
**OS:** Raspberry Pi OS 64-bit (Debian 12 Bookworm)  
**Status:** Active — Phase 1 in progress  
**Related:** Runs on the same Pi as [rpi5-openclaw-assistant](https://github.com/aqn96/rpi5-openclaw-assistant)

---

## What This Is

The Pi runs 24/7 as an unattended AI assistant. The existing security stack (Tailscale + Fail2Ban) protects inbound SSH access but does nothing about outbound DNS queries. A compromised process could beacon out to a malicious domain and nothing would catch it.

This project adds a kernel-level DNS firewall that intercepts DNS packets at the network driver, checks the queried domain against a blocklist, and drops matching packets — all before they leave the machine.

---

## Architecture

The system is split into two planes:

```
┌─────────────────────────────────────────────────────┐
│                  DATA PLANE (Kernel)                │
│                                                     │
│  eth0 (bcmgenet driver)                             │
│       │                                             │
│       ▼                                             │
│  TC eBPF Hook                                       │
│       │                                             │
│       ▼                                             │
│  sentry.p4 → compiled to eBPF bytecode              │
│  ┌──────────────────────────────────────────────┐   │
│  │ Parser: Ethernet → IPv4 → UDP → Port 53?     │   │
│  │ Match:  Look up domain in BPF Hash Map       │   │
│  │ Action: PASS / DROP / notify userspace       │   │
│  └──────────────────────────────────────────────┘   │
│       │                                             │
│  BPF Hash Map                                       │
│  { domain_hash → BLOCKED | UNKNOWN }                │
└─────────────────────┬───────────────────────────────┘
                      │ perf ring buffer
┌─────────────────────▼───────────────────────────────┐
│               CONTROL PLANE (Userspace)             │
│                                                     │
│  loader.c   — loads eBPF into kernel, attaches      │
│               to eth0, creates BPF map              │
│                                                     │
│  manager.c  — listens for unknown domain alerts,    │
│               checks blocklist.txt, updates map     │
└─────────────────────────────────────────────────────┘
```

---

## Project Structure

```
p4-xdp-dns-guard/
├── README.md
├── LICENSE
├── src/
│   ├── sentry.p4       # Phase 1 — P4 packet parser
│   ├── loader.c        # Phase 2 — eBPF loader and map setup
│   ├── manager.c       # Phase 3 — domain classification logic
│   └── Makefile        # Build system
├── docs/
└── notes/
    ├── architecture.md # Design decisions and system diagrams
    └── learnings.md    # Concepts learned during the build
```

---

## Build Phases

| Phase | File | Goal | Status |
|-------|------|------|--------|
| 1 | `sentry.p4` | Parse Ethernet → IPv4 → UDP → identify Port 53 | 🔄 In progress |
| 2 | `loader.c` | Load eBPF into kernel, attach to eth0, create BPF map | ⏳ Pending |
| 3 | `manager.c` | Listen for unknown domains, check blocklist, update map | ⏳ Pending |

---

## Toolchain

| Tool | Version | Purpose |
|------|---------|---------|
| clang | 14.0.6 | Compiles eBPF C code to BPF bytecode |
| llvm | 14 | Backend for clang |
| libbpf | 1.5 | Userspace API for loading eBPF programs |
| bpftool | 7.5.0 | Kernel eBPF inspector and debugger |
| p4c-ebpf | latest | Compiles P4 to C-based eBPF |
| linux-headers | $(uname -r) | Kernel data structure definitions |

---

## Installation

```bash
# Install eBPF toolchain
sudo apt update && sudo apt install -y clang llvm libelf-dev libbpf-dev \
    linux-headers-$(uname -r) bpftool

# Install p4c build dependencies
sudo apt install -y cmake g++ libboost-all-dev libgc-dev libfl-dev \
    libprotobuf-dev protobuf-compiler python3-pip

# Build p4c from source (~30 min on Pi 5)
git clone --recurse-submodules https://github.com/p4lang/p4c.git
cd p4c && mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DENABLE_EBPF=ON
make -j4
sudo make install
```

---

## Known Limitations

- **bcmgenet driver** — Pi 5's ethernet driver has no native XDP support. Using TC eBPF mode instead, which is fully supported and sufficient for DNS traffic
- **DNS over HTTPS (DoH)** — encrypted DNS on port 443 cannot be inspected. This MVP only handles plaintext DNS on port 53
- **Static blocklist** — `blocklist.txt` is a local file for the MVP. Future plan: pull from a live threat feed

---

## Notes

Detailed architecture decisions and concept explanations are in the `notes/` folder:
- [Architecture & Design Decisions](notes/architecture.md)
- [Learnings & Concepts](notes/learnings.md)
