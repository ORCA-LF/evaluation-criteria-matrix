# Compartmentalization Evaluation Matrix — Arbiter

**System:** Arbiter (fine-grained data-object privilege separation for multithreaded apps; compartment = an **Arbiter thread**, a labeled principal thread over the ASMS)
**Paper:** J. Wang, X. Xiong, P. Liu. *Between Mutual Trust and Mutual Distrust: Practical Fine-grained Privilege Separation in Multithreaded Applications.* USENIX ATC 2015.
**Source file:** `papers/atc15-paper-wang-jun.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-10

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment, not a paper claim · **⚠️ Unclear** = not found, needs research.
>
> _Compartment = an **Arbiter thread**: a normal-looking thread carrying a **label + ownership** (DIFC-inspired) that determines, per data object, whether it has read/write/no access. All Arbiter threads share one process but each has its **own page tables** over a special **Arbiter Secure Memory Segment (ASMS)** whose physical pages are shared but whose per-thread protection bits differ ("same accessibility, same page"). A separate **Security Manager** process mediates all allocation/thread-creation/policy operations via RPC (§2.4, §3)._
>
> _Solves the "**2nd privilege-separation problem**": mutually-distrusting principal threads inside one multithreaded process (vs the "1st problem" — splitting a monolith into privileged/unprivileged compartments — addressed by OpenSSH/Privtrans/[[wedge]]) (§1.1, Table 1)._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** between mutually-distrusting principal threads — "the compromise or malfunction of one thread does not lead to data contamination or data leakage of another thread"; protects per-connection/per-user data (Abstract, §1). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Hybrid** — code-centric subjects (Arbiter threads) with privileges defined over **data objects** via an "accessibility" vector (which threads can access an object, in what mode) (§3.1). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Thread** — the Arbiter thread (principal) (§2.4). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Object / Allocation** — per-data-object accessibility, achieved by a permission-oriented allocator that co-locates same-accessibility objects on a page ("same accessibility, same page"); enforcement is at **page** granularity (§3.3, §3.5). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — uses standard **page-table protection bits** (Present, Read/Write) + MMU; explicitly contrasts with Loki/MMP/Mondrian, which "require architectural changes to commodity CPUs" (§1.1, §3.5). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — per-thread page tables over a shared segment (ASMS) + a page-fault reference monitor + an out-of-process Security Manager**; a label-based (DIFC-inspired, non-tracking) policy; user-level permission-oriented allocator (§3, §4). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — threads share one process address space but get distinct page-table views of the shared ASMS physical pages (§2.4, §3.3). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — modified Linux kernel (ASMS management syscalls + page-fault handler additions), the user-level ASMS allocator library, and a separate **Security Manager** process that starts and arbitrates the application (§3, §4). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Data-object-level read/write confinement** across distrusting threads. Arbiter threads cannot allocate/deallocate or change ASMS permissions themselves — only the Security Manager can (mediated by RPC with Unix-socket credential checks). Three intent categories supported: object private to creator, shared by a subset, or shared by all (Categories 1–3). Invalid access → MMU trap → SIGSEGV (§2.1, §3.3, §3.5, §4.2). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅ (fail-stop)** — an illegal ASMS access traps to the page-fault handler and "result[s] in a SIGSEGV signal sent to the faulting thread"; the paper notes a signal handler *could* handle violations more gracefully (drop the connection), but their prototype just crashes (§4.2, §6.1). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the **Security Manager** authenticates each request (verifies caller PID via Unix-socket credentials) and authorizes it against the label rules before any allocation/thread-creation/policy change; a `get_privilege` API lets an innocent thread verify a requester's permission to defeat confused-deputy/forged-reference attacks (§4.1, §6.1). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **OS (Linux kernel) + CPU + the Arbiter Security Manager + ASMS library + ASMS kernel management/page-fault code** (the shaded TCB in Fig. 2). "We assume that the kernel is inside TCB" and the app is independently confined by SELinux/AppArmor at the OS level (§2.2, Fig. 2). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large** — whole Linux kernel trusted, plus the Security Manager + ASMS components; ⚠️ **no LOC figure** for the Arbiter-specific TCB is given (contrast with [[smv]], which reports <1,800 LOC for the same idea) (§2.2, Fig. 2). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None claimed** — out of scope. The paper does enumerate and defend **direct bypass counterattacks** (calling `mprotect`/`munmap`+`mmap` on ASMS → forbidden/denied; `fork`/`pthread_create` to escape → ASMS not mapped; child `ab_register` as rogue Security Manager → different physical mapping; forged reference → `get_privilege`) (§6.1). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal proofs; security argued via threat-model attack simulations on the three ported apps + counterattack enumeration (§6.1). |
| Experimental validation available | Yes (specify) / No | **Yes** — attack simulations (Memcached data theft/overwrite, Cherokee format-string + Heartbleed-style overread, FUSE) all defeated; microbenchmarks; application throughput/CPU/RSS overhead on Memcached, Cherokee, FUSE (§6). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Application throughput reduction **< 10%** (Memcached ~5.6%, Cherokee 1.8–3.0%, FUSE 7.4%); CPU utilization **1.29–1.55×** (Table 5; abstract states 1.37–1.55×). Host: Dell T310, Intel Xeon X3440 quad-core 2.53 GHz, 4 GB, 32-bit Ubuntu 10.04.3, kernel 2.6.32 (§6.3–6.4). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Not lat_ctx, but identified:** Arbiter uses **Option 1** (a new address space per Arbiter thread), so a context switch between Arbiter threads "will lead to TLB flush … just like the context switch between two processes." A partial-flush optimization (Option 2) was deemed too invasive (§3.6). No isolated µs figure. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **`ab_pthread_create` = 145.33 µs vs 91.45 µs native (1.59×)** (Table 4); cost grows with allocated ASMS size (permission reconfiguration of ASMS on each new thread) (§6.2, Fig. 4b). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **N/A for data sharing** (shared memory, below). The relevant managed cost is the **Security Manager RPC round-trip = 5.84 µs** (`ab_null`), paid on every alloc/free/thread-create/policy op (Table 4, §6.2). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | ⚠️ **No bw_pipe figure.** Threads share data through ASMS shared physical pages (near-native reads/writes once mapped, MMU-checked); not measured as bandwidth (§3.3, §4.2). |
| Memory overhead per domain | MB/domain | **RSS overhead 3.9–6.2%** per app (Memcached 6.2%, Cherokee 5.2%, FUSE 3.9%, Table 6) — driven by "same accessibility, same page" page-internal waste + per-thread page tables; **not** an MB/thread figure (§6.4). |
| Domain count bounded by | Limiting factor; approx. domain count | _Inferred:_ **Limiting factor: per-thread page tables + TLB pressure + the single-lock allocator** (allocation/deallocation serialized). `ab_malloc` time rises **~5.7% per additional thread** (allocations propagate to all Arbiter threads). No hard max stated (§6.2, §7). |
| Performance scales with domain count | Big O | _Inferred:_ memory-allocation cost **~O(n)** in thread count (propagation to every Arbiter thread); CPU overhead "roughly correlated with the number of labeled objects." No asymptotic claim (§6.2, §6.4). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** x86 — standard page-table protection bits / MMU, no architectural changes (§1.1, §3.5). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Modified Linux** (kernel 2.6.32) — new ASMS syscalls (`absys_sbrk/mmap/mprotect`, `ab_register`, `ab_pthread_create` over `clone`) + page-fault handler additions + `get_futex_key` change (§4). |
| What privileges does it need? | User / Root / Kernel access | _Inferred:_ **kernel/root to install** the modified kernel; the app + Security Manager run as a normal (unprivileged) user; the attacker is assumed to *not* have root (§2.2, §4). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | The ASMS allocator library (built on uClibc 0.9.32 in the prototype); the app must be launched **by a Security Manager** (which `fork`/`exec`s it) (§4.3). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Source annotations / small API changes** — programmer annotates threads/data with labels and replaces thread-creation/allocation calls with the Arbiter API. Measured: **0.5% of LOC on average** (Memcached 100 LOC/0.5%, Cherokee 188/0.3%, FUSE 129/1.6%) (Abstract, §5.4, Table 3). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Source changes needed** — manual labeling + Arbiter-API substitution + recompile; the API is C-Standard/Pthreads-syntax-compatible to ease incremental adoption (`ab_malloc(L=NULL)` behaves like libc `malloc`) (§5.4, Appendix A.1). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** — pthread-based legacy applications (§5). |
| Other compatibility notes | [Describe] | Futex identification had to be fixed in-kernel because Arbiter threads have different `mm_struct`s; the API mirrors Pthreads/libc so adoption is incremental (§4.3, Appendix A.1). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Low (LOC-wise), understanding-dominated** — "most of our time is spent on understanding the source code and data sharing semantics"; 0.5% avg LOC change; no person-days figure (§5.4). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(Paper notes correct label assignment can be hard in complex/dynamic deployments — §7.)_ |
| Build effort | [build changes / time / deps] | Recompile against the Arbiter API/ASMS library; run on the modified kernel; launch via the Security Manager (§4.3). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(Paper suggests adding a policy debugging/model-checking tool for label correctness, but none is provided — §7.)_ |
| Failure modes visibility | [crashes/logs/error codes/silent] | Invalid access → **SIGSEGV / program crash**; a signal handler could instead drop the connection (not implemented); RPC returns a "security violation" indication on a denied request (§6.1, §4.1). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Not found** — no license or repository in the paper, and no public Arbiter repo located (GitHub search, 2026-06-11). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** prototype (ATC 2015), demonstrated on Memcached, Cherokee, FUSE (§5, §6). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — explicitly **complementary** to OS-level access control (SELinux/AppArmor/Capsicum), which it assumes confines the app, and to DIFC systems like Flume (which it says "can be complementary") (§1.1, §2.2, §6.1). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | _Inferred:_ **✅** — runs as an ordinary multithreaded process under a Security Manager alongside normal processes; relies on OS access control coexisting (§2.2). |
| Can stack effectively | ✅ / ❌ | **✅** — a dynamic set of labeled threads/objects per process; demonstrated with multiple worker threads + labeled-object policies (§5). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Shared memory** for thread-to-thread data; **RPC to the Security Manager** (Unix socket) for all privileged operations (alloc/thread-create/policy) (§2.4, §4.1). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** — ASMS physical pages mapped into multiple threads' page tables with per-thread permissions (e.g. RW for owner, R for others) (§3.3, Table 2). |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Both** — synchronous RPC to the Security Manager + concurrent threads sharing memory with pthread synchronization (futex) (§4.1, §4.3). |
| Interaction security/validation | [Describe] | Security Manager authenticates (PID via socket creds) + authorizes (label RULE 1 data-flow, RULE 2 thread-creation, RULE 3 allocation) every request; ASMS cannot be manipulated via normal `mmap`/`mprotect` (§3.4, §4.1). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Thread** (labeled principal) (§2.4). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** — annotates labels/ownership in source; the Security Manager + kernel enforce; labels can be inherited on thread creation (§3.4, §4.3). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — a **dynamic (label) policy**: permissions are derived at runtime; categories/labels created, granted, and revoked during execution (Table 1, §3.4). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom over standard** — `ab_pthread_create` built on `clone`, Pthreads-compatible API; futex handling patched so synchronization keeps working across differing `mm_struct`s (§4.3). |
| Process model | fork/exec / Custom / N/A | **Custom** — the Security Manager `fork`/`exec`s the app and arbitrates; Arbiter threads share one process but have separate page tables (§4.3). |
| POSIX compatibility | Full / Partial / Limited | **Partial/Full (pthreads)** — Pthreads/libc-syntax-compatible API enabling incremental adaptation; some kernel internals (futex keying) changed (Appendix A.1, §4.3). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Two principals served by the *same* thread cannot be separated (need per-principal "virtual" threads — future work); the user-level allocator uses a **single lock** (serialized alloc/free) limiting scalability; TLB flush per Arbiter-thread switch (§7). |
| Security caveats when layered | [Describe] | Whole kernel trusted; invalid access fail-stops the process; side channels out of scope; **security depends on correct programmer label assignment** (hard in complex/dynamic deployments) (§7). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server** — multi-principal multithreaded servers: in-memory KV store (Memcached), web server (Cherokee), userspace FS (FUSE) (§5). |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Single device** — per-process intra-application separation; not framed as a scale dimension (§5). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |

---

## Summary

> **What it is:** A 2015 system for **fine-grained, data-object-level privilege separation among mutually-distrusting threads** in one multithreaded process. Arbiter gives each thread its own page tables over a shared "Arbiter Secure Memory Segment" (ASMS), co-locates objects of identical accessibility on the same page ("same accessibility, same page"), and uses page-table protection bits as the reference monitor — so a compromised worker thread cannot read or write another principal's in-process data. A DIFC-inspired label model and an out-of-process Security Manager (RPC-mediated) define and enforce policy.
>
> **Who it's for:** Developers of multi-principal multithreaded servers (KV stores, web servers, userspace filesystems) who need to relax the "threads are mutually trusted" assumption without rewriting to multi-process or to a type-safe language.
>
> **What it protects:** Per-thread/per-connection/per-user data objects from compromise (buffer overflow, shellcode, ROP) or logic bugs (Heartbleed-style overreads) in sibling threads.
>
> **What it costs (effort/money/performance):** ~0.5% LOC to annotate, <10% app throughput loss and 1.29–1.55× CPU, plus 3.9–6.2% RSS — but raw memory-op microbenchmarks are 2–4× (RPC to the Security Manager) and the allocator is single-locked.
>
> **What it needs (hardware/OS/expertise):** Commodity x86 (standard page protection), a modified Linux 2.6.32 kernel, the ASMS library, a Security Manager, and a programmer to assign labels correctly.
>
> **Key tradeoffs:** First systematic solution to the "mutually-distrusting threads" problem with object granularity on commodity hardware and tiny source diffs — but the whole kernel is trusted (no TCB-LOC reported), an invalid access fail-stops the process, per-thread page tables force TLB flushes, the allocator serializes, two principals on one thread can't be separated, and correctness hinges on the programmer's label assignment.
>
> **Additional Notes:** A full intra-process compartmentalization **System** (not a baseline) and the **direct predecessor of [[smv]]**, which critiques Arbiter's 200–400% memory-op overhead, global-barrier tagging, and >100-LOC manual effort and replaces them with a library-assisted, fully-concurrent design at ~2%. The page-table-per-thread-over-shared-segment mechanism is the shared idea; Arbiter adds the DIFC-style label model + out-of-process Security Manager. Sits in the same lineage as [[wedge]] (1st PS problem) and the MPK systems [[erim]]/[[cubicleos]].

---

## Sources

- **Primary:** `papers/atc15-paper-wang-jun.pdf` — Wang, Xiong, Liu, *Between Mutual Trust and Mutual Distrust: Practical Fine-grained Privilege Separation in Multithreaded Applications*, USENIX ATC 2015. All **(§N)** / Fig. / Table citations reference this paper. System name **"Arbiter"** confirmed both in this paper and in [[smv]] Table 1.
- **Availability:** the paper states **no repository or license** — availability **Unknown**.

### Cells flagged for further research
- **Availability › License** — ⚠️ Unknown: no repo/license in the paper.
- **Security › TCB size (LOC)** — ⚠️ no LOC given for the Arbiter TCB (contrast [[smv]] <1,800 LOC).
- **Efficiency › Domain switch (lat_ctx), MB/thread, inter-domain throughput (bw_pipe)** — ⚠️ not reported numerically (TLB-flush + RSS% noted; RPC round-trip 5.84 µs given).
- **Usability › Required expertise, Debugging** + **Sys Design › Monitoring, Config complexity, Maintenance burden** — N/A (self-report policy 2026-06-09).
- Inferences marked `_Inferred:_` inline: privileges-to-install, domain-count factor, scaling, deployment scale, coexistence, interaction semantics.
- **CPU-overhead note:** Table 5 lists 1.29× (Cherokee)–1.55× (Memcached); the abstract states "1.37–1.55×" — both reproduced as stated.
