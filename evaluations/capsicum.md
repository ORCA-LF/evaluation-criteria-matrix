# Compartmentalization Evaluation Matrix — Capsicum

**Version:** v0.1
**System:** Capsicum (capability + sandbox framework for UNIX; compartment = a process in capability mode)
**Paper:** R. N. M. Watson, J. Anderson, B. Laurie, K. Kennaway. Capsicum: Practical Capabilities for UNIX. USENIX Security 2010.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — self-compartmentalizing monolithic UNIX apps (privilege separation / least privilege) to limit the impact of a vulnerability. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a process/component. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process** — a sandboxed process in capability mode; a "logical application" groups several. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Object** (secondary) — capabilities are per-object rights (≈60 rights masks) wrapping individual file descriptors (files, sockets, dirs). The *isolation subject* is still the process. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — a pure OS-level mechanism. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other** — OS **capability mode** (`cap_enter`) + **capabilities** (rights-masked file descriptors) + **process descriptors** (`pdfork`); cross-compartment via **RPC/message-passing** (UNIX domain sockets) with hand-coded marshalling (no enforced IDL). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Process**. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — kernel extensions (`cap_enter`, `cap_new`, `pdfork`, modified `namei`/`fget`), `libcapsicum`, and a capability-aware runtime linker (`rtld-elf-cap`). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Least privilege / ambient-authority removal**: capability-mode processes are denied access to global namespaces (file paths, PIDs, System V/POSIX IPC, sysctl, protocol addresses, NFS handles, etc.); **capabilities** restrict file-descriptor rights and are monotonically narrowed on delegation; directory capabilities delegate file-system subsets. A compromised component can act only through its delegated capabilities. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — OS process isolation; **process descriptors** (`pdfork`) tie sandbox lifecycle to a file descriptor (closing it terminates the process), and `SIGCHLD` masking lets libraries sandbox without disturbing the app; the whole logical application terminates together on Ctrl-C/segfault. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — the kernel enforces capability rights on **every** descriptor operation via `fget` (with a new rights argument; changing its signature lets the **compiler** flag missed paths) — strong on the OS-resource boundary. But there is **no enforced IDL**; inter-compartment RPC marshalling is hand-coded by the application, so argument validation is the developer's job. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The FreeBSD kernel** (capability checks in `fget`/`namei`), **`libcapsicum` + `rtld-elf-cap`** runtime, and the **CPU/hardware**. Sandboxed component code is untrusted. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large** — the full FreeBSD kernel is in the TCB. Capsicum-specific additions are described qualitatively (constraints applied at points of implementation; ≈30 of 3000 sysctls whitelisted) but the Capsicum-delta is **Unknown — not quantified as a single LOC**. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — not addressed (2010; pre-Spectre). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — compiler-assisted checking of `fget` call paths is used, but no formal proof. |
| Experimental validation available | Yes (specify) / No | **Yes** — adapted real apps (tcpdump, dhclient, gzip, Chromium); syscall + sandbox microbenchmarks; a comparison of Chromium's six sandbox mechanisms. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Capability-wrapped syscalls add ~0.05–0.07 µs: `fstat` +10.2%, `read` +4.4–8.9%, `write` +10.0%, `cap_new` +50.7%; `cap_enter` 1.22 µs. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Not lat_ctx.** Cross-compartment call = RPC over a UNIX domain socket; "pingpong" roundtrip **309 µs** (+15% vs fork). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Sandbox create+RPC+destroy ≈ 1,509 µs (1.5 ms)** (+85.9% vs fork+exec); `pdfork` 259 µs, `pdfork_exec` 863 µs; `cap_enter` alone 1.22 µs (negligible). gzip sandbox launch **2.37 ms** constant. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | RPC over UNIX domain socket; "pingpong" **309 µs**. Capability-wrapped direct syscalls (fast path) add only ~0.05 µs. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Unknown** — no bw_pipe figure. Design note: **file-descriptor (capability) delegation gives native UNIX I/O** (no proxying); RPC *buffer*-passing cost scales with file size (gzip converges within 5% of native by ~512 KB). |
| Memory overhead per domain | MB/domain | **Unknown** — no MB/sandbox figure. `libcapsicum` adds **+9% to fork** (extra VM mappings, avoidable via `vfork`). |
| Domain count bounded by | Limiting factor; approx. domain count | bounded by **OS process resources** (each sandbox is a process); no specific cap stated. |
| Performance scales with domain count | Big O | linear / O(n) in process count (OS processes); not stated as such. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — standard hardware running FreeBSD. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Modified → now Stock(FreeBSD)** — requires the Capsicum kernel extensions; **mainlined into FreeBSD 9.0** (2012), so stock on modern FreeBSD. |
| What privileges does it need? | User / Root / Kernel access | **User** — "No special privilege is required"; a key advantage over chroot/setuid sandboxes that need root. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | `libcapsicum` + the capability-aware runtime linker (`rtld-elf-cap`); FreeBSD. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Source changes (incremental)** — insert `cap_enter` and/or adopt `libcapsicum` + delegate capabilities; effort scales with depth: **tcpdump ~10 lines** for a sandbox (the bare `cap_enter` is a 2-line change) and **dhclient ~2 lines** (a similar `cap_enter` change), Chromium **~100 lines** (leveraging existing sandbox structure), gzip **409 lines (~16%)** for full RPC proxying. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Source changes needed** — the app must call `cap_enter` / use capabilities. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any** — an OS-level mechanism; C/C++ apps demonstrated. |
| Other compatibility notes | [Describe] | Extends (doesn't replace) UNIX APIs; in capability mode `fork`/`exec` are replaced by `pdfork`/`fexecve` (global-namespace-free); lazy/on-demand initialization (e.g. the DNS resolver) breaks after `cap_enter` and needs care. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Small-to-moderate, scales with depth** — ~10 lines tcpdump (2-line bare `cap_enter`) / ~2 lines dhclient → 100 lines (Chromium) → 409 lines/16% (gzip full proxying). No person-days figure. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Build effort | [build changes / time / deps] | Link against `libcapsicum`; use the capability-aware runtime linker. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Standard + custom** — extended **`procstat`** shows per-fd capability rights + capability-mode flag; **`ktrace`/DTrace** integration (`ENOTCAPABLE` errno; constraints applied in `namei` so paths appear in traces); `libcapsicum` API to run as a single process for debugging. But **distributed debugging across sandboxes remains hard** (gdb not extended). |
| Failure modes visibility | [crashes/logs/error codes/silent] | Capability failures return a distinct **`ENOTCAPABLE`** errno; blocked in `namei` so denied paths are visible in `ktrace`/DTrace. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — BSD** — "modifications to FreeBSD and the Chromium web browser are available under a BSD license"; Capsicum is part of the **FreeBSD base system** (2-clause BSD) since FreeBSD 9.0. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Production** — mainlined into FreeBSD 9+ (and the basis of later tools, e.g. Casper); research origin. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — designed to **supplement, not replace** DAC/MAC; can reinforce existing privilege separation (e.g. dhclient, Chromium). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — capability-mode processes coexist with normal UNIX processes on the same host. |
| Can stack effectively | ✅ / ❌ | **✅** — many sandboxes per logical application; capability rights **monotonically narrow** on delegation (a sub-capability is a subset). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **RPC / message-passing** over UNIX domain sockets, **plus** direct syscalls on delegated capabilities ("fast paths"). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **File-descriptor (capability) delegation** + **RPC** + **POSIX shared memory** (used by Chromium). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** — "RPCs are generally synchronous"; the sandbox thread stack is logically an extension of the host's. |
| Interaction security/validation | [Describe] | Capability rights checked in-kernel (`fget`); delegation is subset-only (monotonic); **no enforced IDL** — apps hand-code marshalling. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process** (sandbox); logical application = group of sandboxed processes. |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** — identifies "natural fault lines," inserts `cap_enter`/`libcapsicum`, and delegates capabilities. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — sandboxes created at runtime (`pdfork`+`cap_enter`); capabilities delegated dynamically (at creation or later over the IPC socket). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Standard UNIX** processes/threads. |
| Process model | fork/exec / Custom / N/A | **fork/exec supported**, but capability-mode variants **`pdfork` / `fexecve`** avoid the global namespaces that plain `fork`/`exec` rely on. |
| POSIX compatibility | Full / Partial / Limited | **Full outside capability mode**; **restricted inside** — global-namespace syscalls (path `open`, most `sysctl`, named System V IPC, etc.) are blocked/constrained in capability mode. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Capability mode blocks/constrains global-namespace syscalls; lazy/on-demand initialization breaks (e.g. DNS resolver); "logical application" lifecycle needs reference-cycle handling for clean termination. |
| Security caveats when layered | [Describe] | Leaked capabilities/memory if a sandbox is entered via bare `cap_enter` without flushing (`libcapsicum` flushes); two colluding sandboxes can race directory-tree rearrangement to defeat the ".." restriction (a conservative mitigation is chosen); no enforced IDL → marshalling bugs possible; side channels out of scope. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Desktop / Server** — general security-critical UNIX applications (browsers, network daemons, CLI tools). |
| Deployment scale | Single device / Cluster / Large scale | **Single device** — a per-host OS mechanism. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |

---

## Summary

> **What it is:** A lightweight OS capability + sandbox framework for UNIX (FreeBSD) that lets monolithic applications **self-compartmentalize** into "logical applications" — groups of processes running in a sandboxed **capability mode** that strips ambient authority and access to global namespaces, where rights are held as **capabilities** (rights-masked file descriptors) and delegated explicitly.
>
> **Who it's for:** Developers of security-critical UNIX applications (browsers, network daemons, parsers) wanting least-privilege compartmentalization with minimal code change and **no special privilege**.
>
> **What it protects:** The rest of the system and the user's resources from a compromised application component — a sandboxed component can act only through explicitly delegated capabilities.
>
> **What it costs (effort/money/performance):** A few % per capability-wrapped syscall; ~1.5–2.4 ms per sandbox creation; modest, incremental code changes (a few lines → ~16%).
>
> **What it needs (hardware/OS/expertise):** A Capsicum-enabled FreeBSD (mainline since 9.0) + `libcapsicum`; source changes to call `cap_enter`/use capabilities; no special privilege.
>
> **Key tradeoffs:** Blends capabilities into UNIX for easy, unprivileged, incremental adoption and native I/O performance (vs pure message-passing capability systems) — but compartmentalization still turns local development into distributed development (hard debugging, hand-coded marshalling), capability mode restricts the syscall surface, and there is no enforced IDL/argument validation.
>
> **Additional Notes:** A foundational, widely-deployed compartmentalization mechanism (FreeBSD base) and a common point of comparison for later work (RLBox, Wedge, etc.). Process-granular and OS-level — contrast with the intra-address-space MPK/SFI/CHERI systems in this set.
