# Compartmentalization Evaluation Matrix — Pegasus

**Version:** v0.1
**System:** Pegasus (transparent kernel-bypass framework with in-process isolation)
**Paper:** D. Peng, C. Liu, T. Palit, A. Vahldiek-Oberwagner, M. Vij, P. Fonseca. Pegasus: Transparent and Unified Kernel-Bypass Networking for Fast Local and Remote Communication. EuroSys 2025. DOI 10.1145/3689031.3696083.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component / Fault isolation** — isolate co-located ("fused") programs so a faulty application does not affect others, enabling safe process fusion for kernel-bypass performance. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — isolates programs (vProcesses). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process** — a vProcess (a fused Linux program, with vThreads). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A.** |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **MPK (Intel/AMD)** + **μSwitch implicit kernel-context switching** (for per-domain kernel resources). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Boundary wrappers** (protected mode-switch gates) + **CFI** + **binary inspection** (eliminates WRPKRU/WRGSBASE in app code) + **per-domain Seccomp** filters. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — vProcesses share one address space, separated by MPK domains. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — a privileged user-space **monitor** (scheduling, memory, ELF loading, signals, network I/O), a μSwitch-modified kernel, DPDK/F-Stack, io_uring, Seccomp. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Memory isolation** between vProcesses (MPK) + **kernel-context isolation** (per-domain files/namespaces/Seccomp via μSwitch) + **CFI** via protected mode-switch gates (the only path to invoke the monitor) + **fault isolation** (monitor handles all real signals, so one program's fault/crash doesn't affect others). Each program accesses only its own resources; the monitor accesses all. **Functional isolation only — not performance isolation.** |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** (explicit) — a fault in one program only affects itself; the monitor catches SIGSEGV/real signals, terminates the faulty program if unhandled, and the monitor + other programs continue unaffected. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — mode-switch gates are the **only** way into the monitor (CFI), the monitor mediates all security-critical syscalls and validates signal frames, per-domain Seccomp restricts each program, and binary inspection blocks WRPKRU/WRGSBASE abuse. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Pegasus monitor** (user-space), the **Linux kernel** (with μSwitch modifications), and the **CPU/hardware** (MPK). Loaded programs are untrusted (may have arbitrary memory access, control-flow hijack, arbitrary syscalls). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large** — Pegasus is **25,946 LOC** C++/asm (the monitor is the trusted subset, not separately quantified) + the μSwitch kernel patch + the full Linux kernel. Monitor-only LOC not isolated. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — explicitly out of scope: the in-process isolation design is not resistant to side-channel attacks; mitigated only by the **same-tenant fusion** assumption. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No.** |
| Experimental validation available | Yes (specify) / No | **Yes** — Redis, Nginx, Memcached, Node.js, Envoy, Caddy, Istio (unmodified); local + remote throughput; vs Linux, Demikernel, F-Stack; Docker/K8s integration. Artifact: `github.com/rssys/pegasus-artifact`, Zenodo DOI 10.5281/zenodo.13714712. Host: 2× AMD EPYC 7543, 256 GiB, ConnectX-6 100 Gbps, Ubuntu 22.04. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC; Pegasus is faster than Linux** (a performance system). vs Linux: local throughput **+19–33%**, remote **+178–442%**, proxied web server **+222%**. vs others: Redis **+153% over Demikernel**, **−1.5% vs F-Stack** (both of which need app rewrites). |
| Domain switch cost | lmbench lat_ctx (platform/native) | Mode-switch gate = user-space context switch updating only a few registers + PKRU; faster than a kernel context switch. **No specific gate-latency figure reported.** (Baseline avoided: Linux futex wakeup ~1.37 µs.) |
| Domain creation cost | lmbench lat_proc exec (platform/native) | vProcess/vThread created via `clone` (supports pthread + vfork flags); user-space ELF loader. **No creation-latency figure.** |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Local TCP fast path** = copy into a shared ring buffer + user-space scheduling, with no syscall/context switch — avoiding the ~1.37 µs futex wakeup. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | Local +19–33% vs Linux; remote via DPDK/F-Stack +178–442%. |
| Memory overhead per domain | MB/domain | Each vProcess = one MPK domain + a reserved contiguous memory area (PROT_NONE, populated on demand). **No MB/domain figure.** |
| Domain count bounded by | Limiting factor; approx. domain count | **16 MPK domains (hardware) → ≈14 vProcesses** (domain 0 = monitor, domain 1 = monitor-writable/read-only shared data). |
| Performance scales with domain count | Big O | Per-CPU worker threads + CFS-like scheduler + load balancing; per-domain memory state. Scaling beyond ~14 fused programs requires multiple instances. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Specialized (commodity)** — **MPK** (Intel Skylake+ or AMD Zen3+; evaluated on AMD EPYC 7543) + a **DPDK-capable NIC** (ConnectX-6) for remote bypass. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Modified** — requires the **μSwitch kernel modifications** (implicit context switching via a shared-descriptor updated at syscall entry). |
| What privileges does it need? | User / Root / Kernel access | **Root to deploy** (kernel patch, DPDK, Seccomp); the **monitor is privileged in-process** (MPK domain 0); programs run unprivileged. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | μSwitch kernel, DPDK/F-Stack, io_uring, Seccomp, Intel XED; an OCI runtime (`runpc`) for Docker/Kubernetes. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None** — transparent, unmodified binaries at the Linux ABI level. **Assumptions:** PIE, no fixed-address page allocation, no `fork`. (Go programs built with `-buildmode=pie`.) |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — unmodified PIE binaries; fork-heavy apps (Apache, Bash) are **unsupported**. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any** — Redis (C), Nginx (C), Node.js (JS), Envoy (C++), Caddy/Istio (Go); any Linux binary meeting the assumptions. |
| Other compatibility notes | [Describe] | PIE required; no `fork`; no fixed-address `mmap`; some net tools (iproute2/iptables) unsupported under F-Stack. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **~None** — zero-effort deployment (unmodified binaries); just ensure PIE (Go: `-buildmode=pie`). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable from the paper. |
| Build effort | [build changes / time / deps] | None for the app (Go needs `-buildmode=pie`); deploy the monitor + μSwitch kernel + DPDK backend. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Partial** — Pegasus handles SIGTRAP/breakpoints (uses Intel XED); broader debugging support is not discussed. |
| Failure modes visibility | [crashes/logs/error codes/silent] | Disallowed syscall → SIGSYS → monitor; MPK violation → SIGSEGV → monitor (terminates the faulty program, others unaffected); gate tampering → vThread terminated. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — GPL-3.0** (`github.com/rssys/pegasus`, GitHub SPDX = GPL-3.0). Artifact: `github.com/rssys/pegasus-artifact`, Zenodo DOI 10.5281/zenodo.13714712. Paper licensed CC-BY 4.0. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — a deployment-oriented prototype (Docker/K8s integration, production-ready F-Stack backend). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — OCI runtime → Docker/Kubernetes integration; pluggable kernel-bypass backends (F-Stack/DPDK, io_uring fallback); built on μSwitch. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — Pegasus instances run as normal Linux processes/containers alongside others. |
| Can stack effectively | ✅ / ❌ | **✅** — up to ~14 vProcesses per instance; multiple instances per host / per K8s pod. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Monitor-mediated message passing** — a local TCP fast path over shared ring buffers; **direct invocations between programs are not allowed** (all go through the monitor). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** (shared ring buffers, monitor-mediated); a read-only shared domain (domain 1: monitor-writable, vProcess-read-only). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — cooperative scheduling for synchronous RPC-style communication + preemptive scheduling (timers/SIGALRM/SIGURG). |
| Interaction security/validation | [Describe] | The monitor mediates all cross-program communication + security-critical syscalls; mode-switch gates enforce CFI; per-domain Seccomp. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process** — a vProcess (a fused program). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | An **oracle (user/framework)** specifies which programs are symbiotic and co-loaded; with Kubernetes, all containers in a pod are auto-fused into one instance. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — vProcesses are created dynamically (`clone` / `vfork`+`exec`); per-pod instances are created on demand. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **User-space scheduler** (CFS-like), per-CPU pinned worker threads, vThreads; transparently supports **pthread** (via `clone` flags). |
| Process model | fork/exec / Custom / N/A | **`clone`/vThread/vProcess + `vfork`+`exec` supported; `fork` NOT supported** (CoW address-space duplication) — Apache/Bash excluded; multi-process fallback is future work. |
| POSIX compatibility | Full / Partial / Limited | **Partial (good)** — Linux ABI level, real apps run unmodified, but no `fork`, PIE required, some net tools unsupported. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | ≤16 MPK domains (≈14 vProcesses); no `fork`; PIE required; no fixed-address `mmap`; F-Stack lacks some Linux net features. |
| Security caveats when layered | [Describe] | Side channels out of scope (same-tenant only); **functional isolation only — no performance isolation**; WRPKRU/WRGSBASE abuse prevented by binary inspection (shares the MPK-system limitations of ERIM/μSwitch). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud / Server** — data center: service mesh, microservices, sidecars. |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — data center; Kubernetes pods. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **Good** — OCI runtime (`runpc`), Docker & Kubernetes integration (one instance per pod). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable from the paper. |

---

## Summary

> **What it is:** A transparent kernel-bypass framework that **fuses** co-located, frequently-communicating, same-tenant ("symbiotic") Linux programs into a single process — fast local communication via shared ring buffers + user-space scheduling, fast remote communication via DPDK — while preserving **fault isolation** between the fused programs using Intel/AMD **MPK** memory isolation + **μSwitch** kernel-context isolation + CFI-enforcing **protected mode-switch gates**, all behind a privileged user-space monitor, for unmodified Linux binaries at the ABI level.
>
> **Who it's for:** Data-center operators running service-mesh / microservice / sidecar architectures who want kernel-bypass performance for **both** local and remote communication without rewriting applications, while keeping the fused services isolated.
>
> **What it protects:** Each fused vProcess from the others — memory + kernel-context + fault isolation (one program's bug/crash doesn't affect others or the monitor). **Functional isolation only** (not performance, not side-channel).
>
> **What it costs (effort/money/performance):** A net performance *win* (+19–33% local, +178–442% remote, +222% proxied vs Linux); requires MPK hardware, a μSwitch kernel patch, and a DPDK NIC; PIE binaries, no `fork`.
>
> **What it needs (hardware/OS/expertise):** MPK (Intel Skylake+ / AMD Zen3+), a modified (μSwitch) Linux kernel, a DPDK-capable NIC; PIE binaries; root to deploy.
>
> **Key tradeoffs:** Combines kernel-bypass performance for *both* local and remote communication with **real in-process isolation** (the feature the concurrent Junction omits) and zero-effort deployment (OCI/Docker/K8s) — but it's limited to ~14 fused programs (MPK), requires PIE and no `fork` (Apache/Bash unsupported), needs a kernel patch + MPK hardware, and provides functional (not performance or side-channel) isolation for same-tenant programs only.
>
> **Additional Notes:** Closest comparison in this set is **Junction** (also kernel-bypass, Linux ABI) — Pegasus explicitly *adds* the in-process isolation Junction omits, and contrasts itself on exactly that point. Like Junction it is primarily a performance system, but isolation is a first-class MPK-enforced feature, so it maps onto the matrix more cleanly.
