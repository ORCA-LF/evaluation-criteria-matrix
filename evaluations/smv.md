# Compartmentalization Evaluation Matrix — SMV (Secure Memory Views)

**System:** SMV — Secure Memory Views (intra-process least-privilege memory for multithreaded apps; compartment = an **SMV**, a thread container over a set of memory domains)
**Paper:** T. C-H. Hsu, K. Hoffman, P. Eugster, M. Payer. *Enforcing Least Privilege Memory Views for Multithreaded Applications.* CCS 2016. ACM.
**Source file:** `papers/smv.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-10

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment, not a paper claim · **⚠️ Unclear** = not found, needs research.
>
> _Compartment = a **Secure Memory View (SMV)**: a thread container holding a collection of **memory protection domains** (each a contiguous virtual-memory range). Threads bound to an SMV (**SMVthreads**) may access only the domains their SMV is granted, with per-domain Read/Write/Execute/Allocate privileges. Enforced by per-SMV page tables + privilege checks in the kernel page-fault handler — no special hardware. Positioned as a **"third-generation"** privilege separation technique (after process-split OpenSSH/Privtrans/[[wedge]]/[[salus]], and [[arbiter]]) that adds **concurrent execution + concurrent (parallel) tagging** (§1, Table 1)._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation + fault isolation** — "selectively isolate software components to restrict attack surface and prevent unintended cross-component memory corruption" in monolithic multithreaded apps; also **secret protection** (e.g. a worker thread denied the server's private key) (Abstract, §1, §3.6). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Hybrid** — code-centric subjects (SMVthreads) with privileges expressed over **data** (memory domains): per-(SMV, domain) Read/Write/Execute/Allocate (§3.1–3.2). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Thread** — the SMVthread; multiple threads can share one SMV to parallelize a workload (§3.3). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Region / Allocation** — a memory domain is a contiguous virtual range; `memdom_alloc` allocates within a domain; enforcement is at **page** granularity (page tables / PTE bits) (§3.1, §4.5). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — explicit "No hardware modifications (NH)" requirement; uses "standard hardware virtual memory protection mechanisms," runs on any commodity CPU regardless of brand/model (§1, §3.5). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — per-domain page tables + kernel page-fault reference monitor.** One `pgd_t` per SMV in the shared `mm_struct`; CR3 reloaded + TLB flushed on switch between SMVthreads; privilege checks added to the page-fault handler (§4.3, §4.5). User-space library auto-intercepts `pthread_create`/`malloc` (§3.4). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — all SMVthreads share one process `mm_struct`/address space, partitioned into memory domains with differing per-SMV privileges (§4.3). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — modified Linux kernel (4.4.5, x86-64) with SMV memory-management changes + an SMV loadable kernel module (Netlink) + the user-space SMV API library (glibc-compatible) (§4). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Default-distrust intra-process memory isolation**: "SMVthreads distrust other SMVthreads by default" and must be explicitly granted access; an unprivileged thread "cannot tamper with a memory protection domain even if there exists a defect in the code of the thread." Permission changes are master-thread-only and immutable by children. Invalid accesses are trapped and the offending thread is killed (SIGSEGV) (§2, §3.2–3.3, §4.5). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **Partial** — an invalid memory access is detected in the page-fault handler and the offending SMVthread receives SIGSEGV; **"the main process will crash to prevent further information leakage"** (fail-stop for the process, not per-thread recovery) (§5.3). Cross-thread *corruption* is contained (a bug stays within the component's memory view) (§3.6.1). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the SMV LKM "sanitizes all user space messages" and verifies the calling SMVthread has correct permissions for each requested change; the page-fault handler is the single reference monitor enforcing per-domain privileges; metadata lives in kernel space, immutable from user space (§4.1, §4.2, §4.5). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **OS (Linux kernel) + CPU/hardware.** Specifically the SMV LKM + SMV memory-management code in the kernel; the **SMV API user-space library is explicitly untrusted**; "We assume that the OS kernel is not compromised … and sound hardware" (§2, §5.4, Table 1). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small (SMV-specific) — `<1,800 LOC`** kernel code per Table 1 / abstract (Table 3: SMV LKM **443** + SMV MM **1,717** kernel LOC; SMV API **781** is untrusted user space); "less than 2000 LOC … could be formally verified." Whole Linux kernel still trusted (§5.4, Table 3). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None claimed** — out of scope. The paper defends a **TOCTTOU page-table-stealing** race (atomic fork; 1,023 malicious threads × 1M runs all used correct page tables) and argues PTE-exploitation is "highly unlikely without serious DRAM bugs such as rowhammer," but no cache/timing/speculative side-channel mitigation (§4.4, §5.4). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** (but argued feasible) — "Given the extremely small source code base (<2000 LOC), we believe that the SMV's TCB could be formally verified"; not actually done (§5.4). |
| Experimental validation available | Yes (specify) / No | **Yes** — LTP robustness suite (memory/FS/disk/scheduler/IPC, no crashes); a demonstrated blocked private-key access (CVE-2004-1097 class); the TOCTTOU security experiment; PARSEC 3.0 + Cherokee + Apache httpd + Firefox performance (§5.2–5.6). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC; PARSEC 3.0 instead** — overall ~**2%** runtime overhead (per-benchmark 0.39%–8.24%); real apps: **Cherokee <0.69%**, **Apache httpd <0.93%**, **Firefox <1.89%** (Abstract, §5.5–5.6, Fig. 6). Host: Intel i7-4790 (4 cores @ 2.8 GHz), 16 GB, Linux 4.4.5 / Ubuntu 14.04.2 (§5.1). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Not lat_ctx, but identified cost:** switching between SMVthreads forces a **page-table reload + full TLB flush** (the original kernel skips this for same-process threads). "Processors equipped with tagged TLBs could mitigate the flushing overhead" — but SMV doesn't require it (§4.3). No isolated µs figure. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | ⚠️ **No lat_proc figure.** SMVthread creation = `pthread_create` + a kernel signal to convert it (allocate private domain + page tables, set `using_smv`) (§4.4). Magnitude not isolated. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **N/A — no explicit cross-domain call.** SMVthreads interact via **shared memory domains**, not a call/IPC primitive; "an in-memory communication domain … so that all threads can exchange data without relying on expensive IPC" (§2, §3.6.1). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | ⚠️ **No bw_pipe figure.** Cross-component data sharing is direct shared-memory (a "secure communication domain"), so near-native, but not measured as bandwidth (§3.6.1). |
| Memory overhead per domain | MB/domain | ⚠️ **No MB/domain figure.** Per-SMV overhead = one `pgd_t` + kernel metadata (`memdom_struct`, `SMV_struct`) + SMV shadow page tables; not quantified in MB (§4.2, §4.3, §4.5). |
| Domain count bounded by | Limiting factor; approx. domain count | _Inferred:_ **Limiting factor: memory for per-SMV page tables/metadata + OS thread limits.** Test policy used N+1 domains for N workers; security test forked **1,023** SMVthreads. No hard max stated (§5.1, §5.4). |
| Performance scales with domain count | Big O | _Inferred:_ negligible/flat in practice (~2% even under "extremely intensive parallel alloc/free"); the per-switch TLB flush is the main per-domain cost. No asymptotic statement (§5.5). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — explicit NH goal; any x86-64 CPU, no special features (tagged TLB optional) (§1, §3.5, §4.3). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Modified Linux** (4.4.5, x86-64) — memory-management changes + an SMV loadable kernel module (§4). |
| What privileges does it need? | User / Root / Kernel access | _Inferred:_ **root/kernel to install** the modified kernel + load the LKM; apps run as **unprivileged** users; only the master thread (per-process) can change privileges (§2, §3.2). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | The user-space SMV API library; glibc-compatible (no custom libc); kernel-API access control delegated to AppArmor/SELinux/seccomp (orthogonal) (§1, §3.3). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Minimal API changes** — the SMV API auto-overrides `pthread_create`→`smvthread_create` and `malloc`→`memdom_alloc`. Measured diffs: **2 LOC** (PARSEC, Cherokee, Apache), **12 LOC** (Firefox, 13M LOC) (Abstract, §5.5, Table 1). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Source changes (tiny) + recompile** — a few LOC to init SMV + set up domains; pthreads still work unmodified for backward compatibility (but with full access) (§3.3, §5.5). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** — pthread-based applications (§3.4, §5.5). |
| Other compatibility notes | [Describe] | Mixing SMVthreads with plain pthreads is discouraged when isolation is needed (pthreads get unrestricted access); SMVthreads are glibc-compatible and cooperate with pthreads via standard synchronization (§3.3). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Very low (LOC-wise)** — 2–12 LOC via the "library-assisted" model (vs Arbiter's ">100 ΔLOC fully manual"); no person-days figure (Table 1, §5.5). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | Include the SMV header, link the SMV API library, recompile; run on the SMV-modified kernel with the LKM loaded (§4, §5.5). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Custom (SMV logs) + standard tools** — the SMV library "provides detailed memory logs" + stack traces (dmesg kernel log shows which SMVthread touched which domain, Listing 1); logs are unreadable to an attacker when debug mode is off. SMV deliberately keeps one `mm_struct` so **GDB/Valgrind keep working** (unlike `clone`-without-`CLONE_VM`) (§4.3, §5.3). _(Stated → kept, not N/A.)_ |
| Failure modes visibility | [crashes/logs/error codes/silent] | Invalid access → page-fault trap → **SIGSEGV, process crashes** to stop leakage; detailed kernel/library logs identify the offending SMVthread and domain (§5.3, Listing 1). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Public code, no declared license** — the repo `github.com/terry-hsu/smv` exists and is "publicly available" (§1 footnote) but carries **no LICENSE file** (GitHub detects none → "all rights reserved" by default; verified 2026-06-11). Same situation as [[erim]] / [[redleaf]] / [[ufork]]: open code, unlicensed. [repo] |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** prototype (CCS 2016), demonstrated on Cherokee, Apache httpd, Firefox, PARSEC (§5). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — explicitly complementary to kernel-API access control (AppArmor/SELinux/seccomp) for syscalls, which SMV does not cover (§1, §2). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — SMVthreads coexist with ordinary pthreads in the same process (backward compatible), and with normal processes on the host (§3.3). |
| Can stack effectively | ✅ / ❌ | **✅** — a dynamic set of SMVs/domains per process; multiple threads may share an SMV or be fully isolated; tested with 1,023 threads (§3.1, §5.4). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Shared-memory data exchange** (no dedicated cross-domain call) — threads cooperate through shared/communication memory domains; synchronization via standard pthreads primitives (§3.3, §3.6.1). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** — register a domain into multiple SMVs (possibly with different privileges, e.g. read-only one-way channels); a global communication domain avoids IPC (§3.2, §3.6.1). |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Both** — concurrent threads sharing memory with pthread synchronization (sync) and producer/consumer pipelines (async); not framed this way explicitly (§3.6.1). |
| Interaction security/validation | [Describe] | Every access validated by the page-fault handler against per-(SMV, domain) privileges; permission grants/revokes go through the kernel-sanitized API and are master-thread-only (§3.2, §4.1, §4.5). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Thread** — privileges attach to SMVs/SMVthreads over memory domains (§3). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** — defines domains, SMVs, and grant/revoke policy via the API (master thread); kernel enforces (§3.2, §3.4). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — domains/SMVs created, granted, revoked, and killed dynamically at runtime (`memdom_create`, `memdom_priv_grant/revoke`, `smv_kill`) — a **dynamic** security policy (Table 1, Table 2). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom over standard** — SMVthreads are pthreads converted in-kernel; glibc-compatible; interoperate with pthreads via standard synchronization (§3.3, §4.4). |
| Process model | fork/exec / Custom / N/A | **Standard single process, many threads** — all SMVthreads share one `mm_struct`; SMV deliberately avoids `clone`-without-`CLONE_VM` (which would break GDB/Valgrind and add sync overhead) (§4.3). |
| POSIX compatibility | Full / Partial / Limited | **Full (pthreads)** — pthread-compatible API and synchronization; backward compatible with unmodified pthread code (§3.3). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Mixing SMVthreads + pthreads when isolation is required is unsafe (pthreads have full access); per-switch TLB flush adds cost (mitigable only with tagged TLBs); kernel-API/syscall access is *not* covered (needs SELinux/seccomp) (§3.3, §4.3, §1). |
| Security caveats when layered | [Describe] | Whole Linux kernel + hardware trusted; an invalid access crashes the whole process (fail-stop); side channels (cache/timing/speculative) out of scope; security depends on a *correct* isolation setup by the programmer (§5.4). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server + Desktop** — multithreaded web servers (Cherokee, Apache httpd) and a desktop browser (Firefox) (§1, §3.6). |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Single device** — per-process intra-application isolation; not framed as a scale dimension (§3, §5). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |

---

## Summary

> **What it is:** A "third-generation" intra-process privilege-separation system for **multithreaded** applications. SMV divides a process's shared address space into **memory protection domains** and groups them into **Secure Memory Views**; each thread (SMVthread) runs in an SMV and can touch only the domains it is granted (Read/Write/Execute/Allocate). Enforcement is per-SMV page tables + privilege checks in the kernel page-fault handler — entirely in commodity-hardware virtual memory, no MPK/CHERI/TEE needed.
>
> **Who it's for:** Developers of monolithic multithreaded C/C++ software (web servers, browsers) who want to confine components/threads (e.g. keep a worker thread away from the private key) without rewriting to multi-process and without losing parallelism.
>
> **What it protects:** Sensitive in-process data and components from negligent or malicious cross-thread memory access — confining memory-safety bugs and blocking privilege escalation between threads sharing one address space.
>
> **What it costs (effort/money/performance):** Tiny diffs (2–12 LOC via library auto-interception) and ~2% overhead (PARSEC), <0.69–1.89% on Cherokee/Apache/Firefox — at the cost of a TLB flush per SMVthread switch and a modified Linux kernel.
>
> **What it needs (hardware/OS/expertise):** Commodity x86-64 (no special hardware), a modified Linux 4.4.5 kernel + SMV LKM, the SMV API library, and a programmer to define domains/views.
>
> **Key tradeoffs:** Solves the multithreaded gap that process-split schemes ([[wedge]], OpenSSH/Privtrans) and even the concurrent-but-slow Arbiter could not — full parallel execution *and* parallel memory tagging at ~2% — with a very small (<1,800 LOC) potentially-verifiable TCB, but the whole kernel stays trusted, an invalid access fail-stops the entire process, side channels are out of scope, and syscall/kernel-API confinement is left to SELinux/seccomp.
>
> **Additional Notes:** A full intra-address-space compartmentalization **System** (not a baseline). Page-table-per-domain is the same family as later MPK schemes ([[erim]], [[cubicleos]]) but predates/avoids PKU by using vanilla VM protection; its distinguishing contribution is **concurrency** (parallel execution + parallel tagging), which the matrix's other thread-level systems lack. Directly comparable to [[arbiter]] (the Wang et al. ATC'15 system) and [[wedge]]. The "Salus" SMV cites as prior work (a privilege-separation system, PolarSSL use case) **is** [[salus]] (`papers/salus1.pdf`, Strackx et al., kernel-supported secure process compartments). _(An earlier mis-filed `salus.pdf` — a CPU-FPGA TEE coincidentally also named "Salus" — was the wrong paper and has been removed.)_

---

## Sources

- **Primary:** `papers/smv.pdf` — Hsu, Hoffman, Eugster, Payer, *Enforcing Least Privilege Memory Views for Multithreaded Applications*, CCS 2016 (ACM), DOI 10.1145/2976749.2978327. All **(§N)** / Fig. / Table citations reference this paper.
- **Repo:** `github.com/terry-hsu/smv` (paper §1 footnote) — public; **license not yet verified**. **[repo]**

### Cells flagged for further research
- **Availability › License** — open-source repo confirmed (§1); exact license string not verified here.
- **Efficiency › Domain creation/switch latency, MB/domain, inter-domain throughput** — ⚠️ not reported numerically (TLB-flush cost noted qualitatively).
- **Usability › Required expertise** + **Sys Design › Monitoring, Config complexity, Maintenance burden** — N/A (self-report policy 2026-06-09).
- Inferences marked `_Inferred:_` inline: privileges-to-install, domain-count factor, scaling, deployment scale, interaction semantics.
- **TCB-LOC note:** abstract/Table 1 say "<1,800 LOC"; Table 3 lists LKM 443 + MM 1,717 (kernel) = 2,160, with §5.4 saying "<2000 LOC" — figures reproduced as stated (possible counting overlap between MM-integrated and standalone code).
