# Compartmentalization Evaluation Matrix — Junction

**System:** Junction (kernel-bypass cloud OS / runtime)
**Paper:** J. Fried, G. I. Chaudhry, E. Saurez, E. Choukse, Í. Goiri, S. Elnikety, R. Fonseca, A. Belay. *Making Kernel Bypass Practical for the Cloud with Junction.* NSDI 2024.
**Source file:** `papers/4-junction.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-08

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment, not a paper claim · **⚠️ Unclear** = not found, needs research.
>
> _Framing note: Junction is primarily a **kernel-bypass performance** system; its compartmentalization role is **instance-level isolation** for dense multi-tenant cloud hosting (an alternative to per-tenant VMs/containers). The compartment = an **instance** (a Linux process running Junction's library-OS kernel + one or more uProcs in a shared address space)._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — strong memory isolation between mutually-untrusting tenant instances and the host kernel in a multi-tenant cloud (§1, §4.1). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is an instance (a process running tenant code) (§3). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process** — an instance is a single Linux process (a `kProc`) acting as an isolated container; it may host multiple uProcs/binaries in one shared address space (§3). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is instance/process-granular. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None for isolation** — isolation uses OS process + seccomp. (Kernel-bypass NIC features and Intel **UIPI** are used for *performance*, not isolation — §3, §5, Table 3.) |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other** — a kernel-bypass **library OS** (the per-instance "Junction kernel" providing OS abstractions in userspace) + **seccomp-BPF** restricting the host syscall interface to ~11 calls + Linux **process isolation** + a per-instance `chroot` jail + read-only page-cache sharing (§3, §4.2, Table 2). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Process / Container** — instance = an "isolated container" implemented as one Linux process (§3). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the Junction kernel (library OS, ~12K LOC C++23) atop Caladan's runtime (~14K LOC), a modified `mlx5` NIC driver (~5K LOC), and a custom UIPI kernel module (~500 LOC) (§7). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | Memory isolation between instances and the host kernel; host syscall attack surface restricted to **11 allowed syscalls** (Table 2, Table 5); each instance gets its own NIC queues, pinned memory, and doorbell page; a uProc that corrupts memory "can only harm its own availability" (§4.1). Reduces host-kernel syscall attack surface by **69–87%** vs two security-focused library OSes (Drawbridge 36, gVisor 57/64 → Junction 11) (§1, §8.5). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅ (cross-instance)** — the Junction kernel "has fate sharing with the uProcs it handles, but is strictly isolated from other instances," so a corrupted uProc harms only its own instance's availability (§4.1). _Within_ an instance, uProcs share an address space and fate-share (no mutual isolation). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — the host boundary is constrained by the seccomp filter; Junction deliberately **omits** TOCTOU mitigation and syscall-argument copying (it assumes a correctly-written program won't trigger UB), but still performs standard argument checks (e.g. fd validity) that Linux programs depend on (§6.2). Intra-instance uProc↔Junction-kernel calls fate-share, so are not security-validated (§4.1). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Host OS kernel (Linux)**, the **NIC** (trusted; provides network virtualization/scheduling across instances), the **CPU/hardware**. The per-instance Junction kernel is **not** trusted by other instances or the host — it sits inside the untrusted boundary and only its own uProcs trust it (§4.1, §4.2). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large** — the full host Linux kernel remains in the TCB, though the *exercised* host-kernel attack surface is drastically reduced (**4 unique / 11 allowed syscalls**, Tables 5–6). Authors note a future minimal/formally-verified host kernel (uKVM-style) as a way to shrink this (§9). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **Transient-execution mitigations are intentionally NOT applied** — not required for isolation given Junction's fate-sharing model (§4.1, §6.2). For cross-tenant microarchitectural interference, Junction builds on **Caladan**, which controls some forms (caches, memory bandwidth) (§8.6). No general speculative/side-channel resistance is claimed. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — none presented (a formally-verified minimal host kernel is floated as future work, §9). |
| Experimental validation available | Yes (specify) / No | **Yes** — extensive: performance vs eRPC/Demikernel/Caladan (Fig. 3–4); density to **3,500 instances** (Fig. 5); compatibility across nginx/Node/Python/Go/Rocket/Tomcat vs gVisor & Firecracker (Fig. 6); attack-surface syscall comparison (Tables 5–6); design-factor analysis (Fig. 7–8). Host: Intel Xeon 6354 18-core, 64 GB, 200 GbE Mellanox ConnectX-6, Linux 6.2 (§8.1). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC; Junction is faster than native Linux for its workloads.** Throughput **1.6–7.0× higher** than Linux while using **1.2–3.8× fewer cores** across seven apps; matches/exceeds eRPC, Demikernel, Caladan (Abstract, §8.2). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Unknown (no lmbench).** Syscalls are converted to function calls into the Junction kernel via a patched glibc trampoline (§6.2); uThread context switches happen in userspace. No lat_ctx figure given. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Unknown** — no instance-creation latency reported (serverless cold-start is cited as motivation but not measured). uProc creation uses `vfork()`+`execve()` (§6.1). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Not lat_pipe.** Cross-instance communication is **loopback networking through the NIC**; intra-instance uProcs use IPC (pipes, loopback) (§3). End-to-end p99 tail latency stays **< 350 µs at 3,500 instances** (35× better than Linux) (§8.3). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Not bw_pipe.** Network throughput examples: Masstree 21 MOp/s (vs eRPC 19.3), memcached 14–16 M req/s (§8.2, Fig. 3–4). |
| Memory overhead per domain | MB/domain | **0.47 MB per instance per core**; **~33 MB per 8-threaded instance**; ~29 MB single-core base overhead (Table 1, §8.6, Fig. 8a). |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: memory** (pinned NIC buffers + per-instance state). **Max 3,500 instances on 128 GB RAM** (Table 1, §8.3). |
| Performance scales with domain count | Big O | Scheduler scales to thousands of instances via NIC notifications (+4×) and a hierarchical timer wheel (+1.69×) (§8.6, Fig. 8b). _Inferred:_ memory linear / O(n) in instance count (from per-instance footprint, Fig. 5). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Specialized (recent commodity)** — modern x86 (Intel/AMD); **Mellanox ConnectX-5+** NIC for buffer/polling density optimizations; **Intel UIPI** (Sapphire Rapids) for best signal/preemption performance, with `tgkill`/`rt_sigreturn` host fallback on older CPUs (§9, Table 3). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Module/Modified** — stock Linux (6.2) plus a custom **UIPI kernel module** (~500 LOC) and a **modified NIC kernel module/driver** (§7). |
| What privileges does it need? | User / Root / Kernel access | **Root/kernel access to deploy** (load kernel modules, configure kernel-bypass NIC); instances themselves run as **seccomp-restricted user processes** (§4.2). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | Patched glibc per instance (replaced by the ELF loader at startup); per-instance `chroot` jail; the Caladan runtime; a kernel-bypass-capable NIC providing per-instance virtualization (§4.2, §6.1, §6.2). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None** — runs unmodified Linux binaries ("No porting", Table 1). _(Go optionally gets a custom OS target for best performance, but unmodified Go still works via the seccomp fallback — §6.)_ |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — unmodified Linux binaries (§1, §6). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any (cloud runtimes)** — C, C++, Rust, Go, Java, Node.js, Python all demonstrated (§6, §8.4). |
| Other compatibility notes | [Describe] | **126 Linux syscalls** implemented — sufficient for common cloud runtimes; desktop features (graphics, input, sound) intentionally unsupported (§6, §7). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **~Zero** — no application porting required (Table 1, headline result). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | **None for the app** (no recompile). Junction itself depends on Caladan + modified NIC driver + UIPI module (§7). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Standard** — "Junction can take full advantage of existing debugging and profiling tools; and the control plane and management functions of a standard Linux environment" (§3). |
| Failure modes visibility | [crashes/logs/error codes/silent] | Disallowed syscalls are blocked/trapped by the seccomp filter (signal); a uProc fault harms only its own instance's availability (§4.1, §4.2). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — MIT** [repo] (`github.com/JunctionOS/junction`, `LICENSE.md`; GitHub SPDX = MIT). Stated open-source in paper (§1). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research / Experimental** — research prototype targeting cloud/serverless; co-authored with Azure Research & Microsoft Research (authors, §1). Not stated to be in production. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | _Inferred:_ **Partial ✅** — built on Caladan; can be combined with Linux cgroups for memory limits and NIC-offloaded bandwidth allocation (§8.6). Composition with other isolation frameworks (e.g. nesting in a VM) is not evaluated. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — "Junction can coexist with normal Linux processes that do not use Junction" (§3). |
| Can stack effectively | ✅ / ❌ | **✅** for many instances coexisting (up to 3,500) (§8.3). Nesting Junction-in-Junction not discussed. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Message passing / networking** — uProcs in different instances communicate only via **loopback networking through the NIC**; uProcs within one instance use standard IPC (pipes) (§3). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | Across instances: **message passing** (network) + **read-only page-cache sharing** of common binaries/libraries (§3, §4.2, Table 7). Within an instance: a **shared address space** among uProcs (§3, §6.1). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — asynchronous network I/O plus synchronous in-process calls (§3, §6). |
| Interaction security/validation | [Describe] | Cross-instance: isolated by separate address spaces + per-instance NIC resources (mediated by the trusted NIC). Within an instance: **no isolation** between uProcs (shared address space, mutual trust) (§4.1, §6.1). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process / Container** (the instance). Sub-instance uProcs are *not* an isolation boundary (shared address space) (§3, §6.1). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | _Inferred:_ the **operator/control plane** — an instance is the deployment unit launched per tenant/function (§3, §8.3); the paper does not frame "who defines boundaries" explicitly. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — instances are created/destroyed dynamically (serverless model, §2, §8.3); uProcs added within an instance via `vfork`+`execve` (§6.1). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom user-level threading** (uThreads with work stealing, mapped onto kThreads), but it transparently supports **unmodified** pthreads / Go / Java threading by shifting threading into userspace (§3, §6.1). |
| Process model | fork/exec / Custom / N/A | **Custom** — single-address-space OS; multiprocess support via `vfork`+`execve` that loads each uProc at a different offset in the shared address space (no page-table clone) (§6.1). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — 126 Linux syscalls; broad coverage for cloud runtimes but not a complete POSIX/Linux surface (no graphics/input/sound) (§6, §7). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | uProcs in the same instance are **not** mutually isolated (shared address space); at most one uProc per instance can be non-PIC; density bounded by memory; UIPI requires Sapphire Rapids; best density needs ConnectX-5+ NICs (§6.1, §9). |
| Security caveats when layered | [Describe] | For isolation between workloads they must run in **separate instances** (§4.2); the full host kernel remains in the TCB; transient-execution mitigations are intentionally omitted, relying on the isolation model (§4.1, §6.2). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud** — explicitly targets cloud applications: microservices, serverless, in-memory stores (§1, §2). |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — designed to densely pack thousands of instances per machine across a cloud fleet (§1, §8.3). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **Good** — instances are normal Linux processes, so the standard Linux control plane / management + profiling tools apply; cgroups usable for resource limits (§3, §8.6). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |

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

---

## Sources

- **Primary:** `papers/4-junction.pdf` — Fried et al., *Making Kernel Bypass Practical for the Cloud with Junction*, NSDI 2024. All **(§N)** / Table / Figure citations reference this paper.
- **[repo]** `github.com/JunctionOS/junction` — `LICENSE.md`; GitHub API SPDX = **MIT**.

### Cells flagged for further research
- **Efficiency › Domain switch cost** — no lmbench/lat_ctx figure.
- **Efficiency › Domain creation cost** — no instance-creation latency reported.
- **Sys Design › Ongoing maintenance burden** — not discussed.
- Rating/classification inferences (config complexity, required expertise, deployment-scale wording, who-defines-boundaries, integrates-with-other-mechanisms, memory Big-O) marked `_Inferred:_` inline.
