# P4-XDP DNS Guard — Learnings & Concepts

**Running log of concepts learned during this build.**  
Updated as each phase progresses.

---

## System Fundamentals

### `uname -m`
Prints the machine hardware architecture. Used to verify the Pi is running
64-bit OS before installing p4c.
- `aarch64` = 64-bit ARM (Raspberry Pi 5) ✅
- `armv7l` = 32-bit ARM (older Pi setups)
- `x86_64` = standard 64-bit Intel/AMD (most laptops)

### apt vs pip vs conda
- `apt` installs system-level tools (compilers, libraries, kernel headers).
  These live in `/usr/bin`, `/usr/lib`, `/usr/include` and are available to
  every user and every environment on the machine.
- `pip` installs Python packages into a Python environment.
- `conda` manages isolated Python environments so different projects don't
  conflict.
- For this project: everything is `apt`. No virtual environment needed because
  we are writing C and P4, not Python.

---

## Toolchain Dependencies

### `clang`
A C compiler. The only compiler that can target the BPF virtual machine inside
the Linux kernel. Used to compile eBPF C code into BPF bytecode. gcc cannot
do this.

### `llvm`
The backend that clang sits on top of. Clang handles parsing and analysis;
LLVM handles the actual code generation (producing BPF bytecode). Not invoked
directly — it's a required dependency for clang to function.

### `libelf-dev`
eBPF programs are packaged into `.o` files in ELF format (the same format as
any Linux binary). This library lets loader.c read and parse those `.o` files
to extract the eBPF bytecode for loading into the kernel.

### `libbpf-dev`
The main library used in loader.c. Provides the API for interacting with the
kernel's BPF subsystem — functions like `bpf_object__load()` and
`bpf_program__attach()`. Abstracts away raw system calls so you can load and
manage eBPF programs in clean C code.

### `linux-headers-$(uname -r)`
Header files for the exact running kernel. eBPF code needs to reference kernel
data structures (like what a network packet looks like in memory). These headers
define those structures. The `$(uname -r)` substitution automatically selects
the headers that match the current kernel version.

### `bpftool`
Debugger and inspector for the entire project. Used to:
- Verify eBPF programs loaded correctly into the kernel
- Dump BPF bytecode for inspection
- Inspect BPF Hash Map contents (see what domains are blocked)
- Troubleshoot why a program was rejected by the kernel verifier

---

## p4c Build Dependencies

### `cmake`
A build system generator. Reads `CMakeLists.txt` and generates the actual
Makefile. p4c uses it to manage its complex multi-file C++ build. You run
`cmake` once to configure, then `make` to compile.

### `g++`
The C++ compiler. p4c itself is written in C++ so you need this to compile
the compiler. Different role from clang: g++ builds p4c, clang builds your
eBPF programs.

### `libboost-all-dev`
A large collection of C++ utility libraries. p4c uses Boost internally for
string manipulation, graph traversal (parsing P4 is essentially walking an
abstract syntax tree), and data structures. Acts as p4c's standard toolkit.

### `libgc-dev`
A garbage collector library. p4c uses this to manage memory automatically
while compiling P4 code. Without it, p4c would need to manually track and
free every allocation made during the compilation process.

### `libfl-dev`
Flex is a lexer generator. When p4c reads sentry.p4 it first tokenizes the
file — splits raw text into meaningful chunks like `header`, `ethernet_h`,
`{`, `bit<48>`. libfl-dev provides the lexer that handles this first stage
of compilation.

### `libprotobuf-dev` + `protobuf-compiler`
Protocol Buffers is Google's data serialization format. p4c uses it internally
to represent compiled P4 programs as structured data passed between compiler
stages. Also used in p4c's IR (Intermediate Representation).

### `python3-pip`
p4c's build process runs Python helper scripts during compilation. pip ensures
those scripts can install any Python packages they need at build time.

---

## Git & GitHub

### PAT (Personal Access Token) Scopes
GitHub fine-grained PATs have per-permission scopes. The existing openclaw
PAT only had read access, which is why `git push` returned a 403 error even
though `git pull` worked. Fix: create a new token with Contents set to
Read and Write for the target repository.

### Branch naming
GitHub defaults to `master` for older repos, `main` for newer ones. When
`git pull origin main` fails with "couldn't find remote ref main", the branch
is likely named `master`. Use `git branch -M main` to rename it locally,
then `git push -u origin main` to push under the new name.

---

## eBPF & Kernel Concepts

### Two Planes
- **Data Plane** — lives in the kernel. Processes every packet at line rate.
  Must be simple and fast. Written in P4/eBPF.
- **Control Plane** — lives in userspace. Makes intelligent decisions. Can be
  slow and complex. Written in C (manager.c).

### BPF Hash Map
Shared memory between the kernel eBPF program and userspace C code. The kernel
reads from it to make instant drop/pass decisions. Userspace writes to it when
a domain is classified. Once a domain is in the map as BLOCKED, every future
query for that domain is dropped in the kernel without touching userspace.

### XDP vs TC
- **XDP (eXpress Data Path)** — hooks inside the network driver, before the
  kernel stack. Fastest possible interception point. Requires driver support.
- **TC (Traffic Control)** — hooks inside the kernel network subsystem, after
  the driver. Slightly later but supported on all drivers.
- Pi 5 uses `bcmgenet` driver which has no native XDP support → using TC mode.

### Port 53
The standard port for DNS (Domain Name System) queries. UDP protocol.
All plaintext DNS traffic uses this port. DNS over HTTPS (DoH) uses port 443
and is encrypted — cannot be inspected by this MVP.

---

## P4 Language Concepts
*(populated as Phase 1 progresses)*

### Headers
...

### Parsers
...

### State Machines
...
