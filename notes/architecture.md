# P4-XDP DNS Guard — Architecture & Design Decisions

**Project:** Kernel-level DNS firewall using P4/eBPF/XDP  
**Hardware:** Raspberry Pi 5 (8GB, aarch64)  
**OS:** Raspberry Pi OS 64-bit (Debian 12 Bookworm)  
**Author:** Andrew Nguyen (@aqn96)  
**Status:** Active — Phase 1 in progress

---

## 1. Problem Statement

The Pi runs 24/7 as an unattended AI assistant (~800 miles from the operator).
The existing security stack (Tailscale + Fail2Ban) protects inbound SSH access but
does nothing about outbound DNS queries. A compromised process on the Pi could
beacon out to a malicious domain and the current stack would never know.

The goal is to intercept DNS queries at the kernel level — before they leave the
machine — and block known bad domains without any userspace overhead.

---

## 2. High-Level Architecture

The system is split into two planes:

```
┌─────────────────────────────────────────────────────┐
│                  DATA PLANE (Kernel)                │
│                                                     │
│  Network Driver (eth0)                              │
│       │                                             │
│       ▼                                             │
│  eBPF/XDP Hook                                      │
│       │                                             │
│       ▼                                             │
│  sentry.p4 (compiled to eBPF bytecode)              │
│  ┌──────────────────────────────────────────────┐   │
│  │ Parser: Ethernet → IPv4 → UDP → Port 53?     │   │
│  │ Match:  Look up domain in BPF Hash Map       │   │
│  │ Action: PASS / DROP / notify userspace       │   │
│  └──────────────────────────────────────────────┘   │
│       │                                             │
│       ▼                                             │
│  BPF Hash Map (shared memory)                       │
│  { domain_hash → BLOCKED | UNKNOWN }                │
└─────────────────────┬───────────────────────────────┘
                      │ perf ring buffer notification
┌─────────────────────▼───────────────────────────────┐
│               CONTROL PLANE (Userspace)             │
│                                                     │
│  manager.c                                          │
│  ┌──────────────────────────────────────────────┐   │
│  │ Listener: waits for unknown domain alerts    │   │
│  │ Logic:    checks domain against blocklist.txt│   │
│  │ Writer:   updates BPF Hash Map if blocked    │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
│  loader.c                                           │
│  ┌──────────────────────────────────────────────┐   │
│  │ Loads eBPF bytecode into kernel              │   │
│  │ Attaches program to eth0                     │   │
│  │ Creates and pins the BPF Hash Map            │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## 3. Design Decisions

### 3.1 Why P4 + eBPF instead of iptables/nftables?

iptables and nftables can block IPs but not domain names — DNS operates at a
higher level. A domain name must be extracted from the DNS packet payload, which
requires deep packet inspection. eBPF can do this; iptables cannot without
additional plugins.

P4 gives us a structured, readable way to describe the packet parsing logic.
It compiles down to eBPF bytecode via p4c-ebpf, so we get the readability of
P4 with the performance of eBPF.

### 3.2 Why TC (Traffic Control) instead of native XDP?

The Pi 5 uses the `bcmgenet` ethernet driver. This driver does not support
native XDP mode (where eBPF runs in the driver itself, before the kernel stack).
XDP would fall back to generic mode automatically.

TC eBPF hooks at a slightly later point in the stack (after the driver, inside
the kernel network subsystem) but is fully supported on all drivers including
`bcmgenet`. For DNS traffic (small, infrequent packets) the performance
difference is negligible.

### 3.3 Why a BPF Hash Map for state?

The data plane (kernel) and control plane (userspace) cannot share normal memory.
BPF Maps are a kernel-managed shared memory mechanism specifically designed for
this. The kernel-side eBPF program reads from the map to make fast drop/pass
decisions. The userspace manager.c writes to the map when it classifies a domain
as blocked. This means a domain only needs to be classified once — every
subsequent query hits the map directly in the kernel and is dropped instantly
without waking userspace.

### 3.4 Why separate loader.c and manager.c?

Single responsibility principle. loader.c has one job: get the eBPF program into
the kernel and set up the map. manager.c has one job: make blocking decisions and
update the map. Keeping them separate makes each easier to debug and replace
independently.

### 3.5 Cloud Grounding (Mode A) stays intact

The existing OpenClaw setup uses Gemini for web research — Google's crawlers visit
external sites, not the Pi. The DNS guard adds a complementary layer: even if a
locally running process tries to query a malicious domain directly, the kernel
drops the packet before it leaves the machine.

---

## 4. Phases

| Phase | File | Goal |
|-------|------|------|
| 1 | `sentry.p4` | Parse Ethernet → IPv4 → UDP → identify Port 53 |
| 2 | `loader.c` | Load eBPF into kernel, attach to eth0, create BPF map |
| 3 | `manager.c` | Listen for unknown domains, check blocklist, update map |

---

## 5. Known Limitations

- **bcmgenet driver** — no native XDP, using TC mode instead
- **DNS over HTTPS (DoH)** — cannot inspect encrypted DNS (port 443). This MVP
  only handles plaintext DNS (port 53)
- **Gemini quota** — the morning news cron uses 1 of 20 daily Gemini requests.
  The DNS guard is kernel-level and has no impact on this
- **blocklist.txt** — static file for MVP. Future: pull from a threat feed API
