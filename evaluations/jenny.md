# Compartmentalization Evaluation Matrix — Jenny

**Version:** v0.1
**System:** Jenny (PKU-based in-process memory isolation with secure userspace syscall filtering; builds on the Donky framework)
**Paper:** D. Schrammel, S. Weiser, R. Sadek, S. Mangard. Jenny: Securing Syscalls for PKU-based Memory Isolation Systems. USENIX Security 2022. Graz University of Technology.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — confine untrusted or vulnerable in-process code (e.g. nginx modules, a compression library like zlib) so it cannot escape the PKU sandbox. The paper's specific focus is making the syscall interface of PKU sandboxes secure. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — code is partitioned into PKU domains (a library or module per domain), each with private memory. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Library / Module** — a domain typically wraps a library or module; domains can be nested (a parent domain constrains a child domain running 3rd-party code). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Page** — Intel MPK tags pages with protection keys; domain-private memory is page-granular. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **MPK/PKU** — Intel MPK (16 keys, per-thread PKRU policy register). base-donky* filters additionally assume Donky's proposed hardware changes; base-mpk runs on commodity Intel MPK. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Boundary wrappers (secure multi-domain call gates) + binary scanning + W^X + a userspace syscall monitor.** Uses ERIM's binary scanner to remove unsafe WRPKRU/XRSTOR; not SFI. Like other PKU sandboxes it can sandbox unmodified binaries. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — nested PKU domains within a single process. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — an in-process userspace monitor enforces syscall filters and PKRU call gates at runtime. The fastest interposition mode (pku-user-delegate) adds a tiny kernel module (319 LoC) and a 33-LoC kernel patch; other modes use stock kernel features (seccomp-user, ptrace). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Inter-domain memory isolation (MPK) plus comprehensive, PKU-aware syscall filtering** that is per-domain, nested, stateful, and thread-specific; **secure asynchronous signal handling**; and protection of the PKRU register via **secure multi-domain call gates**. The paper identifies and blocks new syscall-based PKU escapes (e.g. process_vm_writev, ptrace, /proc/self/mem, userfaultfd, brk/sbrk re-mapping, core-dump leakage, exec, arch_prctl, personality). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **Not framed as crash isolation.** MPK contains a domain's memory accesses, but a fault terminates the process rather than recovering the domain; the paper instead treats crash/core-dump behavior as an attack vector (an untrusted domain triggering a core dump to leak other domains' memory) and mitigates it by disabling core dumps. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — secure multi-domain call gates validate every PKRU transition, the in-process monitor mediates and filters all syscalls, binary scanning removes unsafe WRPKRU/XRSTOR, and per-thread sysargs protection keys guard filtered syscall pointer arguments against TOCTOU/Boomerang attacks. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The in-process sandboxing code (PKU monitor, startup code, linker), the OS kernel, a tiny kernel module, and the CPU/MPK hardware.** Sandboxed code is fully untrusted (may be malicious or contain exploitable bugs). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small added components** — kernel module 319 LoC plus a 33-LoC kernel patch (for the pku-user-delegate mode); the userspace monitor builds on the Donky framework and is not separately quantified in LOC. The full OS kernel remains in the TCB. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — hardware flaws, side channels, and fault attacks are explicitly out of scope. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — security is argued via a systematic analysis of the Linux 5.4 syscall interface, identification of new attacks, and derived filter rules; no formal proof. |
| Experimental validation available | Yes (specify) / No | **Yes** — microbenchmarks (getpid, open) across interposition techniques, application benchmarks, and an nginx case study with isolated modules (zlib/gzip, auth/localstorage); also ffmpeg. Open-sourced at github.com/IAIK/Jenny. Host: Intel Xeon 4208 at a fixed 1.8 GHz, Linux 5.4.0. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Application-level overhead is 0–5% for nginx; the nginx case study with the gzip+auth modules isolated measures 5% with the pku-user-delegate technique, and near-native when no modules are isolated. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Not lat_ctx.** A domain switch is a userspace PKRU change via a secure call gate (WRPKRU); pku-user-delegate avoids extra domain-switch overhead. The paper reports relative microbenchmark runtimes (getpid, Figure 4) rather than an absolute lat_ctx figure; no absolute cycle/µs number is given in the text. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Unknown** — no domain-creation latency is reported. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Not lat_pipe.** Cross-domain calls use WRPKRU-based secure call gates within one address space; syscalls are routed to the in-process monitor. No absolute lat_pipe figure is reported. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Unknown** — no bandwidth/throughput figure is reported (data is shared in-process via page-granular protection keys). |
| Memory overhead per domain | MB/domain | **No MB/domain figure.** Each domain consumes at least one of the 16 MPK protection keys plus its private memory; the paper quantifies key usage, not per-domain MB. |
| Domain count bounded by | Limiting factor; approx. domain count | **Bounded by the 16 Intel MPK keys** (Linux reserves up to 2). Jenny uses one private key per domain, one key for read-only global monitor data, and one sysargs key per thread; the nginx and ffmpeg case studies were limited to 8 worker threads. The case study used 14 keys (2 monitor, 3 domains, 9 threads). Keys can be virtualized for more domains at a loss of security (probabilistic isolation). |
| Performance scales with domain count | Big O | **Cost tracks the number of domain switches**, not domain count directly (e.g. 2 switches per gzip request); the hard scaling limit is MPK key exhaustion, mitigable via key virtualization. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — Intel MPK (Skylake-SP server CPUs and later) for base-mpk. The base-donky* filter set additionally assumes Donky's proposed hardware changes. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Module + small patch** for the fastest mode (319-LoC kernel module plus a 33-LoC kernel patch for pku-user-delegate); other interposition modes (seccomp-user, ptrace, seccomp-ptrace) use stock Linux features. Evaluated on Linux 5.4.0. |
| What privileges does it need? | User / Root / Kernel access | **User** for the monitor; **root** to load the kernel module used by the pku-user-delegate mode. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | Builds on the Donky software framework; W^X enforcement; ERIM's binary scanner. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None to run an unmodified binary in a sandbox; source changes to actually compartmentalize an app** (assign domains and install per-domain filters). The nginx case study added 167 and removed 36 LoC in nginx 1.20.0. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — like other PKU sandboxes it can sandbox unmodified binaries; finer compartmentalization needs source changes. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** (binary/MPK level); the mechanism is largely language-agnostic at the binary level. |
| Other compatibility notes | [Describe] | Case studies limited to 8 worker threads due to per-thread sysargs keys; the 16-key MPK limit; syscall analysis based on Linux 5.4. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | Small code delta (nginx: +167/−36 LoC); no person-days figure reported. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable from the paper. |
| Build effort | [build changes / time / deps] | Link against the Donky/Jenny framework and load the kernel module; not quantified. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable from the paper. |
| Failure modes visibility | [crashes/logs/error codes/silent] | A filter violation denies the syscall or raises a signal; a PKU memory violation faults to the monitor. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source (released)** at github.com/IAIK/Jenny; the repository does not declare an explicit license (no LICENSE file; SPDX NOASSERTION). The paper states it is open-sourced but does not name a license. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** prototype (USENIX Security 2022). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — built on Donky, reuses ERIM's binary scanner, and supports multiple syscall-interposition backends (seccomp-bpf, ptrace, seccomp-ptrace, seccomp-trap, seccomp-user, Linux syscall user dispatch, and its own pku-user-delegate). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **Limited** — it consumes Intel MPK keys and owns the PKRU register, so it conflicts with other MPK users (only 16 keys total); it can run alongside non-MPK isolation. |
| Can stack effectively | ✅ / ❌ | **✅** — nested PKU domains where a parent recursively constrains the syscalls of its child domains is a core feature. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Function calls via secure multi-domain call gates** (WRPKRU-based) within one address space; syscalls are routed to the in-process monitor. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** — page-granular sharing controlled by MPK protection keys (domain-private vs shared). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — synchronous call gates plus securely handled asynchronous signals. |
| Interaction security/validation | [Describe] | Secure call gates validate PKRU transitions; the monitor filters syscalls per domain; per-thread sysargs keys protect syscall pointer arguments against TOCTOU/Boomerang attacks. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Library / module.** |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** assigns domains and filter rules; enforced by the runtime monitor and MPK hardware. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — custom filter rules can be installed per (sub-)domain at runtime via an API, and filtering is nested/dynamic. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Standard threads, with limits** — multi-threading is supported (an improvement over ERIM), but per-thread sysargs keys capped the case studies at 8 worker threads. |
| Process model | fork/exec / Custom / N/A | **Restricted** — fork/clone are filtered as dangerous and exec is incompatible with the in-process monitor (it replaces the running program). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — many syscalls are filtered or denied for security; signals are supported securely. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | 16 MPK keys total; per-thread sysargs keys limit worker-thread count (8 in case studies); exec unsupported; depends on the Donky framework. |
| Security caveats when layered | [Describe] | Key virtualization scales domain count at the cost of only probabilistic isolation; the base-donky* filter set is only fully secure with Donky's proposed hardware changes. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server / Cloud** — in-process containers and web servers (nginx); browser sandboxing is also discussed. |
| Deployment scale | Single device / Cluster / Large scale | **Single device** — per-process in-process isolation; not framed as a deployment-scale dimension. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable from the paper. |

---

## Summary

> **What it is:** Jenny is a PKU-based in-process memory isolation system whose contribution is making the syscall interface of PKU sandboxes secure. It identifies new syscall-based attacks that break PKU isolation, derives comprehensive per-domain filter rules, compares syscall-interposition techniques, and designs a fast and secure one (pku-user-delegate), plus secure multi-domain call gates and secure signal handling.
> **Who it's for:** Builders of in-process sandboxes and in-process containers (web servers, cloud runtimes, browsers) who want fine-grained, per-library isolation with low overhead on commodity Intel MPK.
> **What it protects:** Untrusted in-process code (vulnerable or malicious libraries/modules) from escaping the PKU sandbox via the syscall interface, signals, or the PKRU register.
> **What it costs (effort/money/performance):** 0–5% overhead for nginx; small source changes to compartmentalize an app (nginx +167/−36 LoC); bounded by the 16 MPK keys (case studies capped at 8 worker threads).
> **What it needs (hardware/OS/expertise):** Intel MPK; the Donky framework; for the fastest interposition mode a 319-LoC kernel module plus a 33-LoC kernel patch (other modes use stock Linux features).
> **Key tradeoffs:** The first comprehensive secure-syscall story for PKU sandboxes (nested filtering, secure signals, secure multi-domain call gates) at low overhead, but constrained by MPK's 16-key limit (and per-thread sysargs keys), no crash-recovery story, side channels out of scope, and dependence on the Donky framework; the base-donky* filter set is only fully secure with proposed hardware changes.
> **Additional Notes:** Part of the MPK/PKU in-process isolation line (ERIM, Hodor, Donky, Cerberus); it specifically hardens the syscall and signal interfaces that ERIM/Hodor/Donky left exposed, and builds directly on Donky's software framework.
