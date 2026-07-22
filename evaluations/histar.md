# Compartmentalization Evaluation Matrix — HiStar

**Version:** v0.1
**System:** HiStar — a DIFC (decentralized information-flow control) operating system; compartment = an information-flow label domain over kernel objects
**Paper:** N. Zeldovich, S. Boyd-Wickizer, E. Kohler, D. Mazières. Making Information Flow Explicit in HiStar. OSDI 2006.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation + Secret protection** — strict, decentralized information-flow control (DIFC) so untrusted code cannot leak private data, minimizing trusted code (§1). Motivating cases: an untrusted virus scanner, untrusted login, VPN isolation (§6). Also gives fault isolation: a bug in the untrusted user-level Unix library "only compromises threads that trigger the bug" (§5). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Hybrid** — Asbestos information-flow labels apply both to subjects (threads, gates, which may own categories) and to data objects (segments, containers, devices) (§2, §3). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Thread / Process** — threads and gates carry labels + clearance and can own categories; a "process" is a user-level convention built from kernel objects (§3.1, §5.2). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Object** — every kernel object has a label; a file is a segment object, a directory is a container (§3, §5.1). Any object may be labeled with arbitrarily many categories (§1). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — commodity x86-64 (AMD Opteron/Athlon64); the MMU/virtual memory backs per-thread address spaces, but protection is enforced by kernel labels, not a special hardware primitive (§3.4, §4). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — kernel-enforced DIFC (mandatory access control via Asbestos labels)**: a small fully-trusted kernel is the reference monitor that checks information flow on every syscall/page-fault; **gates** provide protected control transfer (§2, §3, §8). Not SFI / memory-safe-language. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Other — an information-flow label domain** enforced across six low-level kernel object types (threads, address spaces, segments, gates, containers, devices); processes/threads are user-level conventions assembled from these objects (§2, §3, §5.2). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the HiStar kernel (reference monitor + in-kernel drivers) plus an untrusted user-level Unix environment: a uClibc port with a ~10,000-LoC Linux-syscall-emulation layer, and daemons (netd for networking, per-user authentication) (§4, §5). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **End-to-end information-flow control**: "the contents of object A can only affect object B if, for every category c in which A is more tainted than B, a thread owning c takes part" (§3). **Decentralized untainting** — any thread may allocate categories and exclusively owns (`⋆`) them, unlike admin-assigned military labels (§2). Read/write categories give confidentiality + integrity (§2.1). **No superuser**; resource revocation is separated from access (§1, §8). Covert *storage* channels closed; *timing* channels not (§3, §9). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — the kernel is the only fully-trusted code; a vulnerability in the untrusted Unix library "only compromises threads that trigger the bug… an attacker can only exercise the privileges of the compromised thread" (§5). Invalid access → user-mode page-fault handler (default kills the process); if it cannot be invoked, the thread is halted (§3.4). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the kernel mediates every information flow on each syscall/page fault (§2, §3); **gates** give protected control transfer with a **verify label** that proves possession of categories *without granting them* across the call (§3.5). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The HiStar kernel (OS) + CPU/hardware.** The kernel — including in-kernel device drivers — is the only fully-trusted code; there is no superuser (§1, §4). The Unix library, `netd` (network stack), and application code are untrusted; a compromised `netd` "can only mount the equivalent of a network eavesdropping or packet tampering attack" (§5.7). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small (for an OS kernel) — 15,200 LoC C (5,700 lines contain a semicolon) + 150 LoC assembly**, ~45% fewer C lines than the Asbestos kernel; summary rounds to "less than 16,000 lines" (§4.1, §10). Breakdown: 3,400 arch-specific, 4,000 B+-tree/logging/persistence, 3,000 device drivers, 4,800 syscalls/containers/etc. (§4.1). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **Partial** — the low-level explicit-flow interface **closes known covert storage channels** (an improvement over Asbestos), and the single-level store reduces resource exhaustion to a disk-space channel via object quotas (§3). **Covert timing channels are explicitly unsolved** ("a more serious problem we do not know how to solve"), and there are no CPU quotas (§9). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal verification of the kernel or label model is claimed. |
| Experimental validation available | Yes (specify) / No | **Yes** — microbenchmarks (LFS small/large file, IPC pipe round-trip, fork/exec) and application benchmarks (kernel build, 100 MB `wget`, ClamAV scan) vs **Linux** (Fedora Core 5, kernel 2.6.16) and **OpenBSD 3.9**; host: 2.4 GHz AMD Athlon64 3400+, 1 GB RAM, 40 GB 7,200 RPM disk (§7). Real software ported: coreutils, ksh, gcc, gdb, links, OpenSSH, ClamAV, OpenVPN (§5, §6). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Application-level performance is competitive with Linux/OpenBSD (§7.2, Fig 13): build HiStar kernel **6.2 s** vs Linux 4.7 s / OpenBSD 6.0 s; 100 MB `wget` **9.1 s** ≈ Linux 9.0 s (all saturate 100 Mbps Ethernet); ClamAV scan of a 100 MB file **18.7 s = Linux**, and **identical (18.7 s) with the isolation wrapper** — no measurable isolation overhead. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **IPC round-trip 3.11 µs** over a Unix pipe (1M round-trips, 8-byte messages) vs Linux 4.32 µs / OpenBSD 2.13 µs — HiStar beats Linux here (§7.1, Fig 12). Gate calls are the protected-transfer primitive; `netd` uses shared memory + futexes to avoid gate-call overhead (§5.7). No lmbench `lat_ctx` figure. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **fork/exec 1.35 ms/iter** vs Linux 0.18 ms / OpenBSD 0.18 ms (~7.5× slower) — the same workload takes **317 syscalls on HiStar's low-level interface vs 9 on Linux**. The library `spawn` call is **0.47 ms (3× faster than fork+exec, 127 syscalls)** (§7.1, Fig 12). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | Gate = protected control transfer where the client donates the invoking thread's resources (§3.5). Measured IPC (pipe) round-trip **3.11 µs** (§7.1). No separate lmbench `lat_pipe` number. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Unknown** — not measured as pipe bandwidth. Network throughput saturates 100 Mbps Ethernet (`wget`) (§7.2). |
| Memory overhead per domain | MB/domain | **Unknown / not reported per domain.** Each kernel object carries a label, a quota, and 64 bytes of user-defined metadata (§3); category IDs are 61-bit. Label-comparison results for immutable labels are cached (§4). |
| Domain count bounded by | Limiting factor; approx. domain count | **Categories effectively unlimited** — 61-bit opaque category IDs; "over 60 years to exhaust the identifier space even allocating one billion per second," so any thread may allocate arbitrarily many (§2). Resource allocation is bounded by **object quotas** (a hierarchy under the administrator) and disk space via the single-level store (§3.3). |
| Performance scales with domain count | Big O | _Inferred:_ label operations grow with **label size** (the number of non-default categories), which is why the authentication design deliberately keeps labels small "improving the performance of label operations" (§6.2); no asymptotic bound is stated. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity x86-64** (AMD Opteron/Athlon64); the 64-bit address space is used to simplify the design (e.g., virtual memory for file descriptors) (§4). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Custom** — HiStar *is* the operating system, not a module or modification of an existing kernel (§3, §4). |
| What privileges does it need? | User / Root / Kernel access | **It is the OS (runs on bare metal); there is no superuser.** Administrative authority is simply write permission on the root container, which suffices to revoke resources (§1, §3.3, §8, §9). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | uClibc-based Unix library + Linux-syscall-emulation layer; a **single-level store** (whole-system state persisted to / restored from an on-disk snapshot on boot) (§3, §5). x86-64 only. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None → Source changes** — "code that does not interact with security aspects such as user management often requires no modification" (§5); security-aware code adds label/category logic, but that code is small (the ClamAV `wrap` is 110 lines, §1/§6.1). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Recompile only** (for most Unix software) — packages are ported and rebuilt against the HiStar uClibc; it is not binary-compatible with Linux executables (§5). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C / C++** — kernel in C; uClibc + ports in C; the authentication common library is 370 lines of C++ (§4.1, §5, §6.2). |
| Other compatibility notes | [Describe] | Unix-like environment via uClibc + ~10,000-LoC syscall emulation (§5). **Missing/changed** vs Unix: no file access times, no ACLs, no setuid, `chmod`/`chown`/`chgrp` revoke open fds and copy the file, no execute-without-read (§9). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | No person-day figure. Evidence: many Unix packages ported "with little or no source code modification" (§5); security-critical pieces are small — `wrap` 110 LoC (§6.1); authentication logging 58 / directory 188 / auth service 233 LoC (§6.2). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension not assessable from the paper. (The relevant skill is designing DIFC policies with categories/labels.) |
| Build effort | [build changes / time / deps] | Standard toolchain — built with GNU make 3.80 + GCC 3.4.5; the HiStar kernel itself builds in 6.2 s (§7.2). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Standard (gdb) via a per-process debug gate** — a process container exposes "a gate used by gdb for debugging"; gdb is among the ported tools (§5.2, §5). |
| Failure modes visibility | [crashes/logs/error codes/silent] | Invalid read/write → the file-system library coordinates with the user-mode page-fault handler to **return errors rather than SIGSEGV** (§5.1); an unhandled fault kills the process, or halts the thread if the handler can't run (§3.4). A crashed process can leave a **locked mutex** (e.g., the directory segment mutex), which is not currently recovered (§9). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — dual license** [repo `github.com/zeldovich/histar`, `COPYING`]: the **kernel (`kern/`) is GPLv2**; the **rest of the source is a BSD-style license** (Copyright 2005–2008 Zeldovich, Boyd-Wickizer, Stutsman). GitHub reports the repo license as NOASSERTION because it is mixed. Repo marked "not under active development." |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — a prototype OS; the repository is not under active development. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | _Inferred:_ **❌ / limited** — HiStar is a whole OS and is not designed to compose with external isolation frameworks. Its analogous strength is that new information-flow policies can be **overlaid on existing software with a localized change** (the VPN example applies a system-wide policy via one change, "difficult to achieve in a capability-based system") (§6.3). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **N/A** — HiStar replaces the operating system; it does not run side-by-side with other isolation systems on the same host. |
| Can stack effectively | ✅ / ❌ | **✅** — categories compose, and **non-hierarchical** categories let mutually-distrusting components coexist; two mutually-distrustful parties can safely combine privilege to create a shared object (login + the user's auth code jointly build the retry-count segment) (§6.2, §6.3, §9). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Gates (protected control transfer, used RPC-like)** — a client donates the invoking thread's resources across the gate; plus shared memory + futexes for fast paths (e.g., `netd`), and syscalls to the kernel (§3.5, §5.5, §5.7). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** (segments mapped into multiple address spaces; shared file-descriptor segments) **+ message passing via gates** (§3.5, §5.3, §5.7). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — synchronous gate calls (RPC-like), plus asynchronous **alerts** (`thread_alert`) that implement Unix signals (§3.4, §3.5, §5.6). |
| Interaction security/validation | [Describe] | Kernel checks information-flow labels on every gate call and access; gates carry a **verify label** to prove category possession without granting it across the call; thread labels are user-specified and kernel-verified (§3.5). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Thread/process + information-flow category** — the boundary is a set of taint categories the programmer allocates and applies (§2, §5.2). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** — applications allocate their own categories and set labels (decentralized), in contrast to SELinux where policy is centrally specified by the administrator (§2, §8). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — labels are dynamic: a thread can raise its own label (`self_set_label`), adjust clearance, and allocate categories at runtime, tainting itself to read more-tainted data up to its clearance (§2, §3.1). (Non-thread object labels are fixed at creation, but efficient re-labeled copies are supported, §3.) |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom** — threads are kernel objects with a label + clearance; user-level mutexes are built on a kernel futex primitive (§3.1, §4.1). |
| Process model | fork/exec / Custom / N/A | **fork/exec (emulated in the user library, 317 syscalls) + a faster `spawn`** (127 syscalls); processes are a user-space convention (§5, §5.2, §7.1). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — Unix-like via uClibc + syscall emulation, but lacks atime, ACLs, setuid, and changes some semantics (§5, §9). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Covert **timing** channels unsolved; no CPU quotas; a crashed process can leave a locked directory mutex (unrecovered); missing Unix features (atime/ACL/setuid); no upgrade-behind-a-gate mechanism yet (gate entries are fixed) (§9). |
| Security caveats when layered | [Describe] | The kernel is fully trusted (single point of trust). Protecting multiple objects with one category limits the granularity at which privileges can be enumerated (a capability-style critique); HiStar can emulate per-object capabilities by allocating a category pair per object, but the Unix library does not (§9). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server / Desktop** — a general-purpose Unix-like OS whose distinctive value is high-security applications: privilege-separated web services, VPN isolation, untrusted virus scanning, untrusted login (§6). |
| Deployment scale | Single device / Cluster / Large scale | **Single device** — a single-machine OS; the single-level store persists whole-system state to disk (§3). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension not assessable from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension not assessable from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension not assessable from the paper (the authors note they have "no experience administering a production HiStar system," §9). |

---

## Summary

> **What it is:** A DIFC (decentralized information-flow control) operating system that enforces strict, explicit information flow using Asbestos labels on six low-level kernel object types — threads, address spaces, segments, gates, containers, devices — with **no superuser** and a fully-trusted kernel of **under 16,000 lines** (15,200 C + 150 assembly). A Unix-like environment runs almost entirely as *untrusted* user-level library code (§1, §2, §4.1).
>
> **Who it's for:** Builders of high-assurance applications who want a small TCB and provable data-flow policies — the paper demonstrates an untrusted virus scanner, a login/authentication system with no fully-trusted component, VPN isolation, and privilege-separated web services (§6).
>
> **What it protects:** Confidentiality and integrity of data via per-category taint labels, with an end-to-end guarantee: object A can affect object B only if a thread owning every category in which A is more tainted than B takes part (§3). Categories are decentralized — any thread allocates and owns them (§2).
>
> **What it costs (effort/money/performance):** Application-level performance competitive with Linux/OpenBSD (virus scan and `wget` identical to Linux, *no* isolation-wrapper overhead); IPC round-trip 3.11 µs (faster than Linux); the main cost is `fork`/`exec` (~7.5× slower — 317 vs 9 syscalls), mitigated by a 3×-faster `spawn` (§7).
>
> **What it needs (hardware/OS/expertise):** Commodity x86-64; it is a *custom OS* (recompile Unix software against its uClibc); the security-critical skill is designing DIFC policies with categories and labels (§4, §5).
>
> **Key tradeoffs:** A tiny fully-trusted kernel, decentralized application-defined MAC, elimination of the superuser, and closure of covert *storage* channels — at the price of being a custom OS with compatibility gaps (no atime/ACL/setuid), unsolved covert *timing* channels, `fork`/`exec` overhead, and label-operation cost that grows with label size (§7, §9).
>
> **Additional Notes:** Direct successor to **Asbestos** (shares its labels) but adds system-wide persistence (single-level store), explicit hierarchical resource allocation (containers + quotas), and a lower-level interface that closes storage channels (§8). Compared to **EROS/KeyKOS** (capabilities), **SELinux** (centralized MAC), **Jif** (language-level DIFC), and **Plan 9** (no superuser) (§8). For the ORCA matrix, the "compartment" is an **information-flow label domain**, not a memory-isolation sandbox — an OS-level MAC/DIFC point that sits apart from the MPK/SFI in-process systems (it is the "HiStar/Asbestos" DIFC lineage that `wedge` cites as a partitioning relative). License is dual **GPLv2 (kernel) + BSD-style (userland)** [repo].
