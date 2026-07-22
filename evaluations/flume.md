# Compartmentalization Evaluation Matrix — Flume

**Version:** v0.1
**System:** Flume — process-level Decentralized Information Flow Control (DIFC) via a user-level reference monitor on stock Unix; compartment = a confined process's DIFC label domain
**Paper:** M. Krohn, A. Yip, M. Brodsky, N. Cliffer, M. F. Kaashoek, E. Kohler, R. Morris. Information Flow Control for Standard OS Abstractions. SOSP 2007.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation + Secret protection (privacy) + Integrity** — DIFC: untrusted code computes on private data while small trusted code controls its release (secrecy), and trusted code protects itself from malicious input (integrity) (§1, §3.1). "Only bugs in the trusted code… can lead to security violations." |
| Subject selection | Code-centric / Data-centric / Hybrid | **Hybrid** — labels apply to subjects (processes) and to objects (files, pipes, sockets), tracking data flow (§3.1, §4). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process** — the paper's contribution is *process-level* DIFC on stock OSes (vs language-level Jif or new-kernel Asbestos/HiStar) (§1). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **File / Object** — files, pipes, sockets are objects with secrecy+integrity labels; each is exposed as a labeled **endpoint** (§3.3, §4.1). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — pure software on commodity hardware (§5). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — a user-level DIFC reference monitor + syscall interposition**: confined processes cannot issue most syscalls directly; an interposition layer (a **Linux LSM**, or **systrace** on OpenBSD) replaces them with IPC/RPC to the reference monitor, which enforces flow policy (§1, §5, §5.2). The **endpoint** abstraction attaches labels to each file descriptor (§4.1). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Process** — a confined OS process; processes are either *confined* (RM-controlled) or *unconfined* (normal Unix) (§5.1). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — a user-level **reference monitor** plus helpers: a **spawner**, user-space **file servers**, a **tag registry** (for clusters/NFS), and a **Flume libc** that redirects syscalls to the RM; plus a small in-kernel **LSM** (Linux) or systrace (OpenBSD) (§5, Fig 3–4). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **DIFC**: per-process secrecy label `Sp` and integrity label `Ip`; **decentralized privilege** via per-tag capabilities `t+`/`t-` (any process may allocate a tag and gets dual privilege) (§3.1–3.2). Enforces "no read up / no write down" for secrecy and the dual for integrity via **safe-message** and **safe-label-change** rules (Defs 1–5). **Only explicit** label changes are allowed (adopting HiStar's fix, since implicit changes are covert channels) (§2, §3.3). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — security depends only on the small trusted code (e.g., the FlumeWiki security module ≈1,000 lines); "bugs elsewhere in the application cannot" compromise security (§1). A confined process runs as an unprivileged user (`nobody`) and is constrained by the RM even if taken over (§5.2). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — the RM mediates all confined-process communication and drops unsafe messages per Definition 5; endpoints carry per-fd labels checked on every flow. But **file I/O is coarse-grained**: once the RM allows a file open, all subsequent reads/writes are permitted (safety then rests on immutable endpoint labels) (§4.2, §6.1). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The entire underlying OS (Linux/OpenBSD kernel + user space)** plus **Flume's own components** (reference monitor, spawner, file server, tag registry, LSM) and **CPU**. The paper flags that Flume's TCB is "many times larger than those of dedicated DIFC kernels," leaving it exposed to OS bugs (§1). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Flume-specific TCB ≈ 21,500 LoC (C++)** — RM ~14,000, tag registry ~3,500, file server ~2,500, spawner ~1,000, LSM ~500, LSM-framework patch <100 (§5). **Large in effect**, since the whole Linux/OpenBSD kernel + user space is also trusted (§1). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **Partial** — Flume "makes a concerted effort to close all known covert *storage* channels" (indicating where it falls short); the user-space design may expose channels that deeper kernel integration would close. **Covert *timing* channels are assumed absent and not addressed** ("present in all existing DIFC systems") (§1). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **Yes (paper proofs, not machine-checked)** — a formal model (Defs 1–5) with inductive arguments that safe endpoint messages imply safe process messages and that the example policies hold (§3–4). No mechanized verification. |
| Experimental validation available | Yes (specify) / No | **Yes** — **FlumeWiki** (a port of MoinMoin), syscall/IPC microbenchmarks, end-to-end throughput/latency vs unmodified MoinMoin, and cluster scaling (§7, §8). Linux implementation. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** FlumeWiki vs unmodified MoinMoin: **43% slower read throughput, 34% slower write throughput** (abstract: "roughly 30–40% slower"), "due primarily to Flume's user-space implementation" (§8.3, Fig 9). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **N/A** — processes, not intra-address-space domains. The relevant cost is RM-proxied IPC (below). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | Confined processes are created only via **`spawn`** (fork+exec combined; confined processes may not fork) (§5.2). No direct lat_proc figure; syscall interposition adds **35–286 µs** per interposed call (RPC alone ≈40 µs) (§8.2). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **IPC round-trip 33.8 µs (Flume) vs 4.1 µs (Linux) — 8.2×** (§8.2, Fig 8); interposed syscall latency overhead is a factor of **4–35×** (e.g., `stat` on an existing file 3.2 → 110.3 µs, 34.5×) (§8.2). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | IPC bandwidth measured over a 20 GB transfer and reported as a slowdown multiplier (RM proxies every message) (§8.2, Fig 8); exact MB/s not extracted here. |
| Memory overhead per domain | MB/domain | **Unknown** — not reported per confined process. |
| Domain count bounded by | Limiting factor; approx. domain count | Bounded by OS process limits; **tags drawn from a very large opaque space** (any process may allocate arbitrarily many) (§3.1). Cluster support scales to large user counts via a shared tag registry over NFS (§6.3). |
| Performance scales with domain count | Big O | Cluster read throughput **scales roughly linearly**: 4.3 req/s (1 node) → 15.5 req/s (4 nodes) (§8.3). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — runs on stock x86 Linux and OpenBSD (§5). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock + module** — a stock Linux kernel plus a small **LSM** and a <100-line getpid-style patch (or OpenBSD + systrace); the bulk (reference monitor, etc.) is user space (§5, §5.2). |
| What privileges does it need? | User / Root / Kernel access | Root/kernel access to install the LSM and run the reference-monitor service; **confined processes run as an unprivileged user (`nobody`)** (§5.2). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | **Flume libc** (redirects syscalls to the RM); reference monitor + spawner + user-space file servers; a **tag registry** for multi-machine/NFS deployments (§5, §6.3). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Refactoring (light) + a security module** — porting MoinMoin required ~1,000 new lines and changes to ~1,000 of its 91,000 lines (login, ACLs, file handling), plus a ~1,000-line isolated security module (§7). Unaware code runs confined; privilege-aware code uses the Flume API (§4, Fig 4). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Recompile/relink** against Flume libc to run confined; **source changes** where a process must exercise DIFC privilege (§5, §7). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any** — process-level and language-agnostic; demonstrated with **Python** (MoinMoin) and C/C++ (§7). |
| Other compatibility notes | [Describe] | Confined processes **cannot fork** (must `spawn`), cannot issue most syscalls directly, and hit coarse file-I/O granularity; uncontrolled channels (raw sockets, System V IPC) force an `e⊥` endpoint that blocks access to non-declassifiable secret data (§4.2, §5.2). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | No person-day figure; evidence: MoinMoin port changed ≈1,000 of 91,000 lines + a ~1,000-line security module (§7). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension not assessable from the paper. |
| Build effort | [build changes / time / deps] | Link against Flume libc; deploy the reference monitor + helpers (§5). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | Mixed by design: **unsafe messages are dropped *silently*** (to avoid leaks), which "can complicate application development and debugging," whereas **endpoint-label change failures are reported safely as errors** (depend only on the caller's local state) (§3.3, §4.2). |
| Failure modes visibility | [crashes/logs/error codes/silent] | **Silent message drops** for unsafe flows; **explicit error codes** for unsafe label/endpoint changes (§3.3, §4.1–4.2). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — research prototype** (MIT CSAIL, `flume.csail.mit.edu`). The specific license was **not confirmed** from the paper or a canonical repository (the "flume" name is heavily overloaded); flagged for verification. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — a prototype; FlumeWiki is a demonstration application (§7). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — builds *on* stock-OS mechanisms (LSM/systrace) and benefits from ongoing kernel improvements (hardware, NFS, RAID, SMP); explicitly contrasts itself with dedicated DIFC kernels that don't get those updates (§1, §2). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — confined and **unconfined** processes coexist on the same host; unconfined processes fall back to ordinary Unix access control (§5.1). |
| Can stack effectively | ✅ / ❌ | **✅** — DIFC tags/labels compose; mutually-distrusting users combine private data safely (the calendar / shared-secrets example) and deployments scale across a cluster via a shared tag registry (§3.3, §6.3). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Message passing / RPC** — pipes, sockets, and file descriptors proxied by the reference monitor (endpoints); confined processes reach the RM over a control socket (§4.2, §5.1). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Message passing (labeled endpoints) + labeled files** (§4.1–4.2). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — endpoints support reliable bidirectional communication (with the right capabilities) as well as one-way (rendered unreliable) pipes (§4.2). |
| Interaction security/validation | [Describe] | Kernel/RM enforces **safe-message** checks on endpoint labels (Def 5); the RM silently drops unsafe messages and safely reports unsafe configuration changes (§4.2). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process** — plus a per-file-descriptor **endpoint** sub-boundary (§4.1, §5.1). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** — applications allocate their own tags, set labels, and configure endpoints (decentralized), unlike SELinux's admin-set policy (§2, §3.2). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — labels, capabilities, and endpoint labels are adjusted at runtime, but only via **explicit** requests that the RM checks for safety (§3.3, §4.1). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Standard** — a process may hold multiple control sockets to support multithreading (§5.1). |
| Process model | fork/exec / Custom / N/A | **`spawn` (fork+exec combined)** — confined processes may not `fork`; `spawn` is the only way to create a confined process (§5.2). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — confined processes have a restricted syscall interface (three categories: direct / forwarded / forbidden), no `fork`, and coarse file I/O (§5.2, Fig 5). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | User-space overhead (30–43% slower); large TCB (whole underlying OS trusted); uncontrolled channels (System V IPC, raw sockets) force conservative `e⊥` restrictions until Flume "understands" them (§1, §4.2, §8). |
| Security caveats when layered | [Describe] | The entire OS is in the TCB, so an OS privilege-escalation bug breaks Flume's guarantees (explicit assumption); user-space implementation may leave some covert storage channels open; timing channels unaddressed (§1). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server** — Web services (the motivating and evaluated application is FlumeWiki) (§1, §7). |
| Deployment scale | Single device / Cluster / Large scale | **Single device + Cluster** — a shared tag registry over NFS lets multiple machines share file systems and users; read throughput scales roughly linearly to 4 nodes (§6.3, §8.3). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension not assessable from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension not assessable from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension not assessable from the paper. |

---

## Summary

> **What it is:** The first design and implementation of **process-level DIFC on stock operating systems** (Linux, OpenBSD). Flume runs as a **user-level reference monitor**: confined processes are blocked (via an LSM/systrace) from issuing most syscalls directly and instead route them as IPC to the monitor, which enforces per-process secrecy/integrity labels and decentralized `t+`/`t-` capabilities. Its **endpoint** abstraction attaches labels to ordinary file descriptors so DIFC works with pipes, sockets, and files (§1, §3, §4).
>
> **Who it's for:** Developers who want to retrofit strong information-flow policies onto existing applications with familiar tools — demonstrated by porting the 91,000-line MoinMoin wiki to **FlumeWiki**, moving security into a ~1,000-line isolated module and adding end-to-end integrity (§7).
>
> **What it protects:** Secrecy (untrusted code can compute on private data but cannot export it without a declassify capability) and integrity (endorsement), with policy delegated to applications rather than a central admin (§3).
>
> **What it costs (effort/money/performance):** ~30–43% slower than native (FlumeWiki: 43% read / 34% write), from user-space interposition; per-syscall overhead 35–286 µs and IPC round-trip 8.2× Linux (§8).
>
> **What it needs (hardware/OS/expertise):** Commodity hardware; a stock Linux/OpenBSD kernel plus a small LSM/systrace shim; a Flume libc and reference-monitor stack; light refactoring plus an isolated security module per app (§5, §7).
>
> **Key tradeoffs:** Gets DIFC onto *existing* OSes with existing languages and tools (and rides mainstream kernel improvements) — at the price of a **large TCB (the whole OS is trusted, ~21,500 LoC of Flume-specific code on top)**, user-space performance overhead, and unaddressed covert timing channels (§1, §5, §8).
>
> **Additional Notes:** Same DIFC lineage as **Asbestos** and **HiStar** (dedicated DIFC *kernels* with much smaller TCBs) and **Jif** (language-level DIFC, finer-grained but requires rewrites); Flume's labels split readership/ownership and transfer ownership as capabilities (§2). For the ORCA matrix the "compartment" is a **confined process's DIFC label domain**, an OS-level MAC/DIFC point rather than a memory-isolation sandbox — the "Flume" that `wedge` and others cite in the DIFC/partitioning family. License is a research-prototype (MIT CSAIL) not definitively confirmed — flagged.
