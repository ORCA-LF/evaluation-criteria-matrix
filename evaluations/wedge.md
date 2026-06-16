# Compartmentalization Evaluation Matrix — Wedge

**System:** Wedge (default-deny privilege separation for legacy apps; compartment = an **sthread**)
**Paper:** A. Bittau, P. Marchenko, M. Handley, B. Karp. *Wedge: Splitting Applications into Reduced-Privilege Compartments.* NSDI 2008. USENIX.
**Source file:** `papers/wegge.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-10

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment, not a paper claim · **⚠️ Unclear** = not found, needs research.
>
> _Compartment = an **sthread**: a thread of control plus a default-deny security policy specifying which memory tags, file descriptors, and callgates it may access. Implemented as a variant of a Linux process that inherits only the memory regions and FDs named in its policy — not the parent's full address space (§3.1, §4.1). Two helper abstractions: **tagged memory** (regions tagged at allocation; access granted per-tag) and **callgates** (code that runs with different — usually higher — privilege than its caller, as a separate sthread) (§3.2, §3.3)._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Secret protection** (primary) — prevent leakage of sensitive data (SSL private/session keys, user credentials) from exploited network-facing code — combined with **untrusted component isolation** of the network-facing parser (§1, §2, §5). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Hybrid** — code-centric compartments (sthreads/callgates) whose privileges are expressed over **data** (memory tags): "access rights to memory are granted to sthreads in terms of these tags" (§3.1, §3.2). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Thread** — the sthread (a thread of control with its own policy); callgates also run as sthreads (§3.1, §3.3). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Region (tagged) / Allocation** — memory is tagged at `smalloc`-allocation time; access is per-tag; enforcement is at **page** granularity via hardware page protection; globals tagged by carving page-aligned ELF sections (§3.2, §4.1). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **Other — standard hardware page protection** ("We use the standard hardware page protection mechanism to enforce the access permissions for tagged memory") (§3.2). No specialized primitive. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — OS-level isolation primitives** (sthreads = process variants) + **SELinux** for syscall restriction + **memory tagging**; partitioning **assisted by Crowbar**, a Pin-based dynamic memory-access analysis (cb-log/cb-analyze) (§3, §4). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Process** — "We implement sthreads as a variant of Linux processes"; each sthread has its own memory map (§4.1). (Programming interface resembles pthreads, but the isolation unit is a process.) |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — modified Linux kernel (2.6.19) providing sthread/callgate/tag system calls + the userland `smalloc` allocator (dlmalloc-derived, hooked via `LD_PRELOAD`) + SELinux; Crowbar (Pin-based) is dev-time only (§4). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Default-deny confidentiality/integrity of tagged memory**: an sthread can access only the tags/FDs/callgates its policy names; "computations performed on an sthread's stack or heap are by default strongly isolated." A child can hold only a **subset** of its parent's privileges (monotonic). Callgates give a narrow, tamper-proof interface to privileged code/data with a kernel-protected trusted argument (§3.1–3.3). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — sthreads are separate processes with separate memory maps; a protection violation terminates the offending sthread, and "the caller cannot tamper with the callee (and vice-versa)"; workers are isolated from the master and from each other (§4.1, §5.2). (A missing privilege causes a protection violation + crash under default-deny — §5.1.) |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — **callgates** are the validated boundary: kernel checks the entry point is valid and the caller is authorized, retrieves kernel-stored permissions + trusted argument, and validates the argument's access permissions are a subset of the sthread's (§3.3, §4.1). **Crowbar** further helps the programmer enumerate the correct privileges (§3.4). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **OS (the whole Linux kernel) + CPU**, with the Wedge-specific **kernel support code for sthreads/callgates** as the audited core; the programmer-written **callgate bodies are trusted** (must not be exploitable / must not leak via return value). Compiler trusted for legacy app compilation (§7). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Wedge-specific core is Small** — "the kernel support code for sthreads and callgates… is of manageable size—less than 2000 lines" and auditable; **but the rest of the Linux kernel must also be trusted** (so whole-system TCB is Large) (§7). Per-app trusted callgate code is small (e.g. OpenSSH private-key callgate = **280 LOC**) (§5.2). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None / partial-by-design** — Wedge is not a DIFC system, so it avoids the general covert-channel problem, but "Wedge still may disclose sensitive information through covert channels" from privileged compartments; the SSL-read defense explicitly leaves only a **timing covert channel** (modulating packet inter-arrival). No DoS protection; text segment is readable (§5.1.2, §7). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal proofs; security argued via threat-model case analysis on Apache/OpenSSL and OpenSSH (§5). |
| Experimental validation available | Yes (specify) / No | **Yes** — two real applications partitioned (Apache/OpenSSL, OpenSSH) defeating private-key/session-key disclosure incl. a man-in-the-middle attack; microbenchmarks of primitives; Crowbar trace-overhead; end-to-end throughput/latency (§5, §6). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC as a perf target** (SPECint2006 used only to measure Crowbar dev-tool trace overhead, not runtime isolation). App-level: partitioned Apache reaches **27% of native throughput** (all-sessions-cached, worst case) to **69%** (no session cache) with recycled callgates; OpenSSH adds "negligible latency" (§6, Table 2). Host: 8-core 2.66 GHz Xeon, 4 GB (Apache on 1-core 2.2 GHz Opteron, 2 GB) (§6). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Not lat_ctx.** Sthread creation is **~8× slower than pthread** creation and **similar to `fork`** (Fig. 7); recycled callgates context-switch via a `futex` wake + shared-memory copy and are **~8× faster** than standard callgates. No isolated context-switch latency reported (§6, §4.1). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Sthread creation ≈ `fork` latency**, ~8× a pthread; for parents with large page tables, sthread creation can be *faster* than fork (only policy-named page-table entries + FDs are copied) (Fig. 7, §6). Absolute µs in Fig. 7 (not in extracted text). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Not lat_pipe.** A standard **callgate invocation** = create + destroy an sthread (dominant cost); **recycled callgates** amortize this (~8× faster) by reusing a long-lived sthread woken via futex (§3.3, §6). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | ⚠️ **No bw_pipe figure.** Recycled callgates pass arguments via memory shared between caller and underlying sthread; throughput not measured (§4.1). |
| Memory overhead per domain | MB/domain | ⚠️ **No MB/sthread figure.** Each sthread gets a private stack, a private (COW) copy of global data, and a COW snapshot of the pre-`main` memory image; only policy-named regions/FDs are mapped (§4.1). |
| Domain count bounded by | Limiting factor; approx. domain count | _Inferred:_ **Limiting factor: OS process limits + memory** (sthreads are processes). Each Apache request creates **2 sthreads and invokes ~8 callgates**; no max-count stated (§6). |
| Performance scales with domain count | Big O | _Inferred:_ overhead "directly proportional to the number of sthreads and callgates created and invoked per request" — i.e. **O(n)** in compartment instances per request; no asymptotic statement (§6). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** x86 — uses only standard hardware page protection (§3.2, §6). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Modified Linux** (kernel 2.6.19, with new sthread/callgate/tag syscalls) + **SELinux** for syscall restriction (§4, §4.1). |
| What privileges does it need? | User / Root / Kernel access | _Inferred:_ kernel modification → **kernel/root to install**; sthreads run as unprivileged users; changing an sthread's UID/root follows UNIX semantics (only root can) (§3.1). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | `LD_PRELOAD` `smalloc` allocator; ELF-section placement for tagged globals (`BOUNDARY_VAR`); code must be compiled with **frame pointers** + debug symbols for Crowbar (§4.1, §4.2). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Refactoring (API changes)** — programmer restructures the app into sthreads/callgates, tags memory (`smalloc`/`smalloc_on`/`BOUNDARY_VAR`), and writes security policies. Measured: Apache **~1700 LOC changed (0.5%)**, OpenSSH **564 LOC (2%)** (§5.1, §5.2). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Source changes needed** — manual partitioning required; binary-only libraries are a noted limitation for tagging (§3.2, §5). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** — legacy C applications; relies on the C allocator hooks and frame-pointer/debug-symbol compilation (§4). |
| Other compatibility notes | [Describe] | No write-only memory permissions (CPU limitation → use read-write); `smalloc_on` not signal/thread-safe by default (programmer must save/restore); binary-only libraries hard to tag (§3.1, §4.1). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Moderate, tool-assisted** — no person-days figure, but small diffs (0.5–2% of LOC) heavily aided by Crowbar (e.g. enforcing one Apache boundary required identifying **222 heap objects + 389 globals**) (§5.1, §5.2). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(Paper notes security decisions must stay with the programmer, not be automated — §7.)_ |
| Build effort | [build changes / time / deps] | Compile with frame pointers + debug symbols; link the `smalloc` preload; tagged globals need ELF-section handling; dev-time Pin/Crowbar tracing (§4). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Custom — Crowbar** (cb-log/cb-analyze): logs every memory read/write with full backtraces (function/file/line) and original allocation sites, queryable to find which code needs which memory; an **sthread-emulation library** lets the app run without protection faults to reveal all needed privileges (§3.4, §4.2). _(Stated in paper → kept, not N/A.)_ |
| Failure modes visibility | [crashes/logs/error codes/silent] | A missing privilege → **hardware protection violation that crashes the sthread** under default-deny; Crowbar's emulation library surfaces these as logged violations instead of crashes during development (§5.1, §3.4). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Not found** — paper §9 says "publicly available" at `http://nrg.cs.ucl.ac.uk/wedge/` (a 2008 UCL URL, now defunct); no license stated and no current source located. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** prototype (NSDI 2008), demonstrated on Apache/OpenSSL and OpenSSH (§5, §9). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — explicitly composes with **SELinux** (per-sthread syscall policy) and UNIX UID/filesystem-root isolation; the paper suggests Crowbar-style tools could also aid Asbestos/HiStar/Flume partitioning (§3.1, §8). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | _Inferred:_ **✅** — sthreads are ordinary Linux processes running alongside normal processes; coexists with standard `fork`/pthreads (handled correctly by Crowbar) (§4.1, §4.2). |
| Can stack effectively | ✅ / ❌ | **✅** — "any number of sthreads and callgates may exist within an application, interconnected in whatever pattern the programmer specifies," sharing disjoint memory regions at single-byte granularity (§8). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Callgate invocation** (a blocking call into a privileged sthread, RPC-like with a return value) + shared **tagged memory** for data; recycled callgates signal via `futex` (§3.3, §4.1). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory via tags** — granting a tag's permission to multiple sthreads shares those regions; callgate arguments are passed by `smalloc`'d pointers (§3.2, §3.3). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** — "the caller blocks until the callgate terminates, and then collects any return values" (§3.3). |
| Interaction security/validation | [Describe] | Kernel-validated: callgate permissions stored in-kernel (untamperable), entry point + authorization + argument-permission-subset checked on every invocation; trusted argument cannot be altered by the caller (§3.3, §4.1). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Thread/Process** — the sthread (a process-backed compartment) (§3.1, §4.1). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** — manually structures sthreads/callgates and writes policies; **Crowbar assists** but takes no security decisions ("the tool alone guarantees no security properties") (§3.4, §7). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — sthreads and callgates are created/destroyed dynamically at runtime (e.g. one worker sthread per connection; per-client tags); privileges are monotonically reducible (§5.1, §3.1). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom — sthreads** (pthread-like API but process-backed); standard pthreads still supported and handled by Crowbar (§4.1, §4.2). |
| Process model | fork/exec / Custom / N/A | **Custom (fork-derived)** — sthreads are a `fork` variant that inherits only policy-named regions/FDs, not the full address space (§4.1). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — runs legacy POSIX C apps with source changes; sthread API mirrors pthreads; standard `fork`/pthreads coexist (§4.1). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | No write-only permissions; `smalloc_on` not signal/thread-safe by default; binary-only libraries hard to tag; recycled callgates trade isolation for speed (a reused callgate exploited across principals can leak one caller's args to another) (§3.3, §3.2, §4.1). |
| Security caveats when layered | [Describe] | Callgates must not be exploitable and must not leak via return value or side channel; covert/timing channels remain from privileged compartments; no DoS protection; whole Linux kernel still trusted (§7). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server** — networked server applications (SSL web server, SSH login server) (§1, §5). |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Single device** — per-host application partitioning; not framed as a scale dimension (§5). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(Paper notes refactoring can break sthreads by touching new memory; re-run Crowbar — §3.4.)_ |

---

## Summary

> **What it is:** An early (2008) default-deny privilege-separation system for legacy C applications. Wedge adds three OS primitives — **sthreads** (process-backed compartments that inherit *nothing* by default), **tagged memory** (per-tag access grants enforced by hardware page protection), and **callgates** (narrow, kernel-validated upcalls into privileged code) — plus **Crowbar**, Pin-based dynamic-analysis tools that tell the programmer which code needs which memory, making default-deny partitioning tractable.
>
> **Who it's for:** Developers of complex, monolithic, security-sensitive servers (web/SSH) who want fine-grained least-privilege compartments without rewriting from scratch.
>
> **What it protects:** Sensitive data (SSL private/session keys, credentials, user data) from exploited network-facing code — demonstrated to stop key disclosure and even a man-in-the-middle attack on Apache/OpenSSL that defeats coarse process-level privilege separation.
>
> **What it costs (effort/money/performance):** Manual refactoring (0.5–2% of app LOC, heavily Crowbar-assisted), sthread creation ≈ `fork` (~8× a pthread, amortized by recycled callgates), and 31–73% throughput loss on partitioned Apache depending on workload; negligible latency on interactive OpenSSH.
>
> **What it needs (hardware/OS/expertise):** Commodity x86 (standard page protection), a **modified Linux 2.6.19 kernel** + SELinux, frame-pointer/debug-symbol compilation, and a programmer who reasons about the partition (Crowbar assists but does not decide).
>
> **Key tradeoffs:** Much finer-grained, tighter, default-deny partitioning than prior privilege-separation (Privtrans/Privman/OKWS) and a small auditable Wedge-specific TCB (<2000 LOC), but the whole Linux kernel stays trusted, partitioning is manual, side/covert channels and DoS are out of scope, and recycled callgates trade isolation for throughput.
>
> **Additional Notes:** A foundational intra-application privilege-separation **System** (not a baseline). Conceptual ancestor of later memory-view / least-privilege-thread work — directly comparable to the multithreaded-privilege-separation cluster ([[smv]], the Wang et al. ATC'15 system) and to callgate/PKU schemes like [[erim]]. The callgate-as-validated-boundary and "tools to make default-deny usable" ideas are its durable contributions.

---

## Sources

- **Primary:** `papers/wegge.pdf` — Bittau, Marchenko, Handley, Karp, *Wedge: Splitting Applications into Reduced-Privilege Compartments*, NSDI 2008 (USENIX). All **(§N)** / Fig. / Table citations reference this paper.
- **Availability:** paper §9 states public release at `http://nrg.cs.ucl.ac.uk/wedge/` (UCL Network Research Group); no license stated — **license Unknown** (likely defunct academic URL).

### Cells flagged for further research
- **Availability › License** — ⚠️ Unknown: "publicly available" (§9) but no license; 2008 UCL URL likely dead.
- **Efficiency › Memory MB/sthread, Inter-domain throughput (bw_pipe)** — ⚠️ not reported.
- **Efficiency › Domain switch/creation/call latency** — given only qualitatively (≈fork, ~8× pthread; Fig. 7 absolute µs not in extracted text).
- **Usability › Required expertise** + **Sys Design › Monitoring, Config complexity, Maintenance burden** — N/A (self-report policy 2026-06-09).
- Inferences marked `_Inferred:_` inline: privileges, domain-count factor, scaling Big-O, deployment scale, coexistence.
