# Compartmentalization Evaluation Matrix — Junction

**Version:** v0.1
**System:** Junction (kernel-bypass cloud OS / runtime)
**Paper:** J. Fried, G. I. Chaudhry, E. Saurez, E. Choukse, Í. Goiri, S. Elnikety, R. Fonseca, A. Belay. Making Kernel Bypass Practical for the Cloud with Junction. NSDI 2024.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — strong memory isolation between mutually-untrusting tenant instances and the host kernel in a multi-tenant cloud. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is an instance (a process running tenant code). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process** — an instance is a single Linux process (a `kProc`) acting as an isolated container; it may host multiple uProcs/binaries in one shared address space. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is instance/process-granular. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None for isolation** — isolation uses OS process + seccomp. (Kernel-bypass NIC features and Intel **UIPI** are used for *performance*, not isolation.) |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other** — a kernel-bypass **library OS** (the per-instance "Junction kernel" providing OS abstractions in userspace) + **seccomp-BPF** restricting the host syscall interface to ~11 calls + Linux **process isolation** + a per-instance `chroot` jail + read-only page-cache sharing. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Process / Container** — instance = an "isolated container" implemented as one Linux process. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the Junction kernel (library OS, ~12K LOC C++23) atop Caladan's runtime (~14K LOC), a modified `mlx5` NIC driver (~5K LOC), and a custom UIPI kernel module (~500 LOC). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | Memory isolation between instances and the host kernel; host syscall attack surface restricted to **11 allowed syscalls**; each instance gets its own NIC queues, pinned memory, and doorbell page; a uProc that corrupts memory "can only harm its own availability". Reduces host-kernel syscall attack surface by **69–87%** vs two security-focused library OSes (Drawbridge 36, gVisor 57/64 → Junction 11). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅ (cross-instance)** — the Junction kernel "has fate sharing with the uProcs it handles, but is strictly isolated from other instances," so a corrupted uProc harms only its own instance's availability. _Within_ an instance, uProcs share an address space and fate-share (no mutual isolation). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — the host boundary is constrained by the seccomp filter; Junction deliberately **omits** TOCTOU mitigation and syscall-argument copying (it assumes a correctly-written program won't trigger UB), but still performs standard argument checks (e.g. fd validity) that Linux programs depend on. Intra-instance uProc↔Junction-kernel calls fate-share, so are not security-validated. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Host OS kernel (Linux)**, the **NIC** (trusted; provides network virtualization/scheduling across instances), the **CPU/hardware**. The per-instance Junction kernel is **not** trusted by other instances or the host — it sits inside the untrusted boundary and only its own uProcs trust it. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large** — the full host Linux kernel remains in the TCB, though the *exercised* host-kernel attack surface is drastically reduced (**4 unique / 11 allowed syscalls**). Authors note a future minimal/formally-verified host kernel (uKVM-style) as a way to shrink this. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **Transient-execution mitigations are intentionally NOT applied** — not required for isolation given Junction's fate-sharing model. For cross-tenant microarchitectural interference, Junction builds on **Caladan**, which controls some forms (caches, memory bandwidth). No general speculative/side-channel resistance is claimed. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — none presented (a formally-verified minimal host kernel is floated as future work). |
| Experimental validation available | Yes (specify) / No | **Yes** — extensive: performance vs eRPC/Demikernel/Caladan; density to **3,500 instances**; compatibility across nginx/Node/Python/Go/Rocket/Tomcat vs gVisor & Firecracker; attack-surface syscall comparison; design-factor analysis. Host: Intel Xeon 6354 18-core, 64 GB, 200 GbE Mellanox ConnectX-6, Linux 6.2. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC; Junction is faster than native Linux for its workloads.** Throughput **1.6–7.0× higher** than Linux while using **1.2–3.8× fewer cores** across seven apps; matches/exceeds eRPC, Demikernel, Caladan. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Unknown (no lmbench).** Syscalls are converted to function calls into the Junction kernel via a patched glibc trampoline; uThread context switches happen in userspace. No lat_ctx figure given. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Unknown** — no instance-creation latency reported (serverless cold-start is cited as motivation but not measured). uProc creation uses `vfork()`+`execve()`. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Not lat_pipe.** Cross-instance communication is **loopback networking through the NIC**; intra-instance uProcs use IPC (pipes, loopback). End-to-end p99 tail latency stays **< 350 µs at 3,500 instances** (35× better than Linux). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Not bw_pipe.** Network throughput examples: Masstree 21 MOp/s (vs eRPC 19.3), memcached 14–16 M req/s. |
| Memory overhead per domain | MB/domain | **0.47 MB per instance per core**; **~33 MB per 8-threaded instance**; ~29 MB single-core base overhead. |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: memory** (pinned NIC buffers + per-instance state). **Max 3,500 instances on 128 GB RAM**. |
| Performance scales with domain count | Big O | Scheduler scales to thousands of instances via NIC notifications (+4×) and a hierarchical timer wheel (+1.69×). memory linear / O(n) in instance count (from per-instance footprint). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Specialized (recent commodity)** — modern x86 (Intel/AMD); **Mellanox ConnectX-5+** NIC for buffer/polling density optimizations; **Intel UIPI** (Sapphire Rapids) for best signal/preemption performance, with `tgkill`/`rt_sigreturn` host fallback on older CPUs. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Module/Modified** — stock Linux (6.2) plus a custom **UIPI kernel module** (~500 LOC) and a **modified NIC kernel module/driver**. |
| What privileges does it need? | User / Root / Kernel access | **Root/kernel access to deploy** (load kernel modules, configure kernel-bypass NIC); instances themselves run as **seccomp-restricted user processes**. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | Patched glibc per instance (replaced by the ELF loader at startup); per-instance `chroot` jail; the Caladan runtime; a kernel-bypass-capable NIC providing per-instance virtualization. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None** — runs unmodified Linux binaries ("No porting"). *(Go optionally gets a custom OS target for best performance, but unmodified Go still works via the seccomp fallback.)* |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — unmodified Linux binaries. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any (cloud runtimes)** — C, C++, Rust, Go, Java, Node.js, Python all demonstrated. |
| Other compatibility notes | [Describe] | **126 Linux syscalls** implemented — sufficient for common cloud runtimes; desktop features (graphics, input, sound) intentionally unsupported. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **~Zero** — no application porting required. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Build effort | [build changes / time / deps] | **None for the app** (no recompile). Junction itself depends on Caladan + modified NIC driver + UIPI module. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Standard** — "Junction can take full advantage of existing debugging and profiling tools; and the control plane and management functions of a standard Linux environment". |
| Failure modes visibility | [crashes/logs/error codes/silent] | Disallowed syscalls are blocked/trapped by the seccomp filter (signal); a uProc fault harms only its own instance's availability. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — MIT** (`github.com/JunctionOS/junction`, `LICENSE.md`; GitHub SPDX = MIT). Stated open-source in paper. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research / Experimental** — research prototype targeting cloud/serverless; co-authored with Azure Research & Microsoft Research. Not stated to be in production. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **Partial ✅** — built on Caladan; can be combined with Linux cgroups for memory limits and NIC-offloaded bandwidth allocation. Composition with other isolation frameworks (e.g. nesting in a VM) is not evaluated. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — "Junction can coexist with normal Linux processes that do not use Junction". |
| Can stack effectively | ✅ / ❌ | **✅** for many instances coexisting (up to 3,500). Nesting Junction-in-Junction not discussed. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Message passing / networking** — uProcs in different instances communicate only via **loopback networking through the NIC**; uProcs within one instance use standard IPC (pipes). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | Across instances: **message passing** (network) + **read-only page-cache sharing** of common binaries/libraries. Within an instance: a **shared address space** among uProcs. |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — asynchronous network I/O plus synchronous in-process calls. |
| Interaction security/validation | [Describe] | Cross-instance: isolated by separate address spaces + per-instance NIC resources (mediated by the trusted NIC). Within an instance: **no isolation** between uProcs (shared address space, mutual trust). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process / Container** (the instance). Sub-instance uProcs are *not* an isolation boundary (shared address space). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | the **operator/control plane** — an instance is the deployment unit launched per tenant/function; the paper does not frame "who defines boundaries" explicitly. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — instances are created/destroyed dynamically (serverless model); uProcs added within an instance via `vfork`+`execve`. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom user-level threading** (uThreads with work stealing, mapped onto kThreads), but it transparently supports **unmodified** pthreads / Go / Java threading by shifting threading into userspace. |
| Process model | fork/exec / Custom / N/A | **Custom** — single-address-space OS; multiprocess support via `vfork`+`execve` that loads each uProc at a different offset in the shared address space (no page-table clone). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — 126 Linux syscalls; broad coverage for cloud runtimes but not a complete POSIX/Linux surface (no graphics/input/sound). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | uProcs in the same instance are **not** mutually isolated (shared address space); at most one uProc per instance can be non-PIC; density bounded by memory; UIPI requires Sapphire Rapids; best density needs ConnectX-5+ NICs. |
| Security caveats when layered | [Describe] | For isolation between workloads they must run in **separate instances**; the full host kernel remains in the TCB; transient-execution mitigations are intentionally omitted, relying on the isolation model. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud** — explicitly targets cloud applications: microservices, serverless, in-memory stores. |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — designed to densely pack thousands of instances per machine across a cloud fleet. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **Good** — instances are normal Linux processes, so the standard Linux control plane / management + profiling tools apply; cgroups usable for resource limits. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |

---

## Summary

> **What it is:** A kernel-bypass cloud OS/runtime that runs unmodified Linux applications inside densely-packed **instances**, each a Linux process hosting a userspace "Junction kernel" (library OS) plus one or more uProcs in a shared address space, isolated from other instances and the host by a strict 11-syscall seccomp boundary.
>
> **Who it's for:** Cloud operators wanting kernel-bypass performance (high throughput, low tail latency) with VM-like multi-tenant isolation but far higher density and no application porting.
>
> **What it protects:** The host kernel and other tenant instances from a compromised/malicious instance — strong memory isolation, host syscall attack surface cut to 11 calls (69–87% smaller than security-focused library OSes).
>
> **What it costs (effort/money/performance):** Performance is a *net win* (1.6–7.0× throughput, 1.2–3.8× fewer cores than Linux); cost is in deployment — recent NICs (ConnectX-5+), ideally UIPI-capable CPUs, and kernel modules. ~0.47 MB/instance/core, ~33 MB per 8-thread instance.
>
> **What it needs (hardware/OS/expertise):** Modern x86 + kernel-bypass NIC, a stock Linux host with custom UIPI + NIC kernel modules (root to deploy), and the Caladan runtime; no application changes.
>
> **Key tradeoffs:** Kernel-bypass performance *and* strong cross-instance isolation with a tiny host attack surface, at unmodified-binary compatibility — but uProcs **within** an instance are mutually trusting (not isolated), the full host kernel stays in the TCB, transient-execution defenses are deliberately omitted, and it depends on specialized NIC/CPU features.
>
> **Additional Notes:** Junction sits at the boundary of "compartmentalization" — its primary contribution is performance/density; isolation is an enabling property for safe cloud deployment. The matrix's per-call/per-domain efficiency rows (lmbench-style) map poorly because Junction's whole point is avoiding kernel crossings; those cells are flagged rather than guessed.
