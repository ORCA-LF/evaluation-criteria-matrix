# Compartmentalization Evaluation Matrix — Endoprocess (Little Mac)

**System:** Endoprocess (programmable subprocess isolation via a nested in-process monitor; compartment = a **subprocess**, mediated by the **endokernel**; prototype = **Little Mac**)
**Paper:** F. Yang, W. Huang, K. Kaoudis, A. L. Vahldiek-Oberwagner, N. Dautenhahn. *Endoprocess: Programmable and Extensible Subprocess Isolation.* NSPW 2023 (New Security Paradigms Workshop). (Rice / Trail of Bits / Intel Labs)
**Source file:** `papers/endoprocess.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-10

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment · **⚠️ Unclear** = not found, needs research.
>
> _Compartment = a **subprocess**: a collection of code + data with assigned privileges, isolated within one process by a nested self-protecting monitor, the **endokernel**, which virtualizes process interfaces (`mmap`, `mprotect`, syscalls, signals) for non-bypassable mediation. Memory lives in **subspaces** (the smallest protection unit); privileges attach via **lexical scopes** (static) and **dynamic scopes** (runtime time-slicing). A **translation layer** maps custom application-level policy abstractions down to the subprocess interface (§3)._
>
> _**Venue caveat:** NSPW is a "new security paradigms" workshop; this is largely a **vision/organization paper** with an **initial prototype (Little Mac)** — many quantitative cells are preliminary or unreported by design (§1, §4)._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — least-privilege confinement of mutually-distrusting or buggy components inside one process; "prevent these intra-process privilege escalations" (§2.1–2.2). Also secret protection (e.g. confine OpenSSL key access to OpenSSL) (§2.4, §6). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Hybrid** — code is partitioned into subprocesses (code-centric) whose privileges are expressed over data **subspaces** and system objects (data-centric) (§3.3). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Function / Library** — subprocesses are defined by labeling lexical scopes; granularity ranges from a whole dynamic library down to annotated functions (e.g. the HTTP parser) (§3.3, §5, §6.1). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Region (subspace) / Allocation** — a subspace is a virtual-address segment (code/stack/heap) and is "the smallest unit of memory protection"; system objects (files/sockets) are also belongs-to-assigned (§3.3). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **MPK/PKU** in the Little Mac prototype (Intel MPK domains for subprocess heaps + protecting the monitor); explicitly **portable** to CHERI, Intel CET (for the trampoline), VT-X/VMFUNC, or VDom (§4, §6.2, §7). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — a nested self-protecting security monitor (endokernel) + syscall/system-object virtualization + an MPK-protected trampoline** (only the monitor can issue `syscall`, enforced via MPK + seccomp/`dispatch`); plus a compiler/translation layer for annotations (§3.2, §4). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — subprocesses share one process address space, partitioned into subspaces with per-subprocess privileges; the endokernel is nested in-process to avoid supervisor context switches (§3.2). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the in-process **endokernel/Little Mac** monitor + its syscall-monitoring trampoline + the translation layer/`libsep` runtime libraries; minimal OS support (seccomp/`dispatch` to confine stray `syscall` instructions) (§3, §4). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Non-bypassable intra-process least-privilege**: memory isolation between subprocesses *and* of the monitor, with **complete mediation even through OS interfaces** (the paper's key advance — closing `/proc/self/mem`-style bypasses that ERIM/Hodor ignore). Subprocess privileges are lexically + dynamically scoped; pointer sharing acts as **sealed capabilities** (a shared pointer can't be re-delegated without explicit policy). Can also enforce CPI/isolated-secrets (§2.4, §3.2–3.3). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | _Inferred:_ **✅** — a compromised subprocess "cannot interact with any other memory in the process, nor inject code, nor read files, nor send messages"; the endokernel mediates faults/signals and can route a segfault to a configured handler subprocess (§2.1, §3.3). Not framed as a crash-recovery experiment. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the endokernel is a **non-bypassable reference monitor**: all subprocess switches go through statically-declared entry points + an MPK-gated trampoline; all syscalls/system-objects are virtualized and policy-checked; the translation layer + `libsep` give validated high-level policy abstractions (virtual privilege rings) (§3.2–3.4, §4). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **OS + hardware (CPU/MPK) + the endokernel/Little Mac monitor + the compiler/translation layer** — "The operating system, hardware platform, and monitor are part of the Trusted Computing Base, and assumed bug free" (§2.2). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small monitor + Large OS** — the endokernel is a deliberately minimal microkernel-style monitor ("does not have any additional default abstractions"), but **no LOC figure** is given for Little Mac; the whole OS kernel remains trusted (§3.1, §6.1). ⚠️ TCB-LOC unreported. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — "Side-channel attacks are considered out of scope" (§2.2). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal proofs; a design/paradigm paper with an initial prototype (§1, §4). |
| Experimental validation available | Yes (specify) / No | **Yes (preliminary)** — Little Mac micro/app overheads on curl, lighttpd, nginx, sqlite3, zip (Fig. 3) and privilege-separated NGINX (HTTP parser sandboxed + OpenSSL safeboxed) over TLS (Figs. 3–4); illustrative use cases (§4, §6). Not a full systematic evaluation (NSPW). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Linux apps (curl/lighttpd/nginx/sqlite3/zip) under seccomp/`dispatch`/CET variants: overhead up to ~tens-of-% (Fig. 3, std dev < 2.4%); **privilege-separated NGINX (virtual privilege rings) < 10%** overhead; NGINX TLS throughput normalized to ~0.9–1.0 (Figs. 3–4). Preliminary, no native-baseline table (§4). |
| Domain switch cost | lmbench lat_ctx (platform/native) | ⚠️ **Not reported numerically.** Switches use MPK (no kernel entry) via the endokernel trampoline + entry points; designed to avoid "costly context-switching" of OS/page-table approaches, but no lat_ctx figure (§3.2, §7). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | ⚠️ **Not reported.** Subprocesses are created by labeling lexical scopes (static) or creating dynamic scopes at runtime; no creation-latency figure (§3.3). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | ⚠️ **Not reported.** Cross-subprocess invocation is a mediated context-switch through declared entry points (privileges activate/deactivate); no µs figure (§3.3). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | ⚠️ **Not bw_pipe.** Data shared via message passing (copy) or **shared subspaces** (zero-copy, direct pointers) — near-native for shared memory, but not measured as bandwidth (§3.3). |
| Memory overhead per domain | MB/domain | ⚠️ **Not reported.** Per-subprocess overhead = subspaces (code/stack/heap) + MPK domain + monitor records; no MB/domain figure (§3.3, §4). |
| Domain count bounded by | Limiting factor; approx. domain count | _Inferred:_ **Limiting factor on Little Mac: the 16 hardware MPK PKEYs** (a known MPK limit). The paper cites mitigations (libmpk multiplexing, EPK/VDom) and portability to other mechanisms to exceed it; no concrete max stated (§7). |
| Performance scales with domain count | Big O | ⚠️ **Not stated** — no asymptotic analysis (preliminary prototype). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — Intel MPK (widely available) for Little Mac; portable to CHERI/CET/VT-X/VDom (§4, §6.2). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock + minimal support** — Linux with seccomp/`dispatch` to confine stray `syscall` instructions; "require only minimal OS modifications"; POSIX virtualized (prior work), extensible to Windows (§3.2, §4, §6.2). |
| What privileges does it need? | User / Root / Kernel access | _Inferred:_ **User-level** — MPK domain switches need no kernel entry; seccomp setup is user-configurable. Not explicitly stated (§3.2, §4). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | A compiler/translation toolchain that lowers the annotation dialect to subprocess enforcement; `libsep` runtime libraries; subprocess-aware allocator changes for some libraries (§3.4, §5). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Annotations** — developers label code/data into subprocesses ("boxes") and assign a virtual-ring level; a crude mode just wraps a whole dynamic library with default policy (e.g. OpenSSL) by changing its allocator (§5, §6.1). Annotations are no-ops if no mechanism is present (like CFI markers) (§5). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Source changes (annotations) + recompile** — though "transparently execute unmodified applications" is a stated goal where policy can be applied at library boundaries; full benefit needs annotation (§3.1, §5). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** — retains "C/C++ ABI semantics" for pointer sharing; envisioned to augment managed/JS runtimes (V8 isolates, Lambdas) too (§3.3, §6.1). |
| Other compatibility notes | [Describe] | Explicitly aims to support commonly-neglected functionality — multithreading, process control, **signals**, `rseq` — that prior MPK monitors break; sealed-capability pointer semantics preserve normal C sharing (§2.4, §3.3, §4). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Low (annotation-based), tool-assisted** — virtual privilege rings + translation layer minimize manual cross-context policy; automated marker-generation tools "in development." No person-days/LOC figure (§3.4, §4, §5). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(A core design goal is "least-effort programmability" — §3.1.)_ |
| Build effort | [build changes / time / deps] | Compile with the annotation-lowering toolchain + link `libsep`; configure seccomp/`dispatch`; some allocator changes (§3.4, §4, §5). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Failure modes visibility | [crashes/logs/error codes/silent] | Denied accesses trap to the endokernel; segfaults/signals can be routed to a configured handler subprocess or default-handled; sealed-capability misuse is blocked at switch time (§3.3). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | ⚠️ **No declared license** — the paper states no repo; the authors' `endokernel` GitHub org exists (`endokernel/endokernel-paper-ver`) but that prototype repo carries **no LICENSE** (the org's `runq`=Apache-2.0 and `glibc`=GPL-2.0 are *upstream forks*, not Little Mac's own license; verified 2026-06-11). No clearly-licensed release of the prototype. [repo] |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research / Experimental** — an NSPW 2023 vision paper with an early prototype; future work outlines automation + richer scenarios (§1, §8). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — explicitly proposed to *augment* other systems: nest inside [[gvisor]] (replace its separate monitor process), inside [[unikraft]]-style unikernels, and inside V8-isolate serverless runtimes ([[cloudflare-workers]]/Lambda) to add non-bypassable subprocess isolation (§6.1). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — composes with seccomp/SELinux/AppArmor at the process level (process MAC stays; endokernel adds the subprocess layer) (§2.3, §3.2). |
| Can stack effectively | ✅ / ❌ | **✅** — three virtual privilege rings + arbitrary additional contexts; dynamic scopes nest/time-slice within a subprocess (§3.3, §4). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Function calls through mediated entry points** — entering a subprocess's lexical scope via statically-declared entry points activates its privileges and deactivates the caller's (§3.3). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Both** — message passing (microkernel-style copy) **or** shared **subspaces** (zero-copy, shared via normal pointers acting as sealed capabilities) (§3.3). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — synchronous scope-switch calls + asynchronous interrupts/signals that the endokernel multiplexes/blocks per subprocess policy (§3.3). |
| Interaction security/validation | [Describe] | Every switch/syscall mediated by the endokernel; sealed-capability pointers can't be re-delegated without explicit policy; dynamic scopes auto-revoke privileges (and invalidate their pointers) on destruction (§3.3). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Library / Function (subprocess)** — lexical scopes aligned to static program scopes; dynamic libraries or annotated functions (§3.3, §5). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer (annotations) + Compiler/translation layer** — developer labels boxes/rings; the toolchain lowers to subprocess enforcement; the endokernel enforces (§3.4, §5). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — **dynamic scopes** are created/destroyed at runtime to raise/lower privilege and time-slice (e.g. per-user-session in a DB), the paper's "most novel" feature (§3.3). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Standard threads (explicitly supported)** — Little Mac ensures "threads do not break our trampoline"; multithreading is a deliberately-supported feature prior monitors neglected (§2.4, §4). |
| Process model | fork/exec / Custom / N/A | **Standard, virtualized** — process creation/destruction and `exec` are among the OS interfaces the endokernel virtualizes/belongs-to-assigns (§3.2–3.3). |
| POSIX compatibility | Full / Partial / Limited | **Partial→Full (virtualized)** — POSIX/Linux interfaces are virtualized (from the authors' prior endokernel work); aims to preserve app-visible semantics (§3.2). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Little Mac bounded by 16 MPK PKEYs (mitigable via libmpk/EPK/VDom); some libraries need allocator changes; augmenting JIT/managed runtimes (V8) needs integration work (§5, §6.1, §7). |
| Security caveats when layered | [Describe] | OS + hardware + monitor trusted and assumed bug-free; side channels out of scope; security depends on the developer correctly annotating components/policies (§2.2). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server / Cloud** — multiplexed servers (NGINX), multi-user DBs, containerized microservices, serverless (Lambda/Workers), unikernels (§2.1, §6). |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Single device** — per-process intra-application isolation; not framed as a scale dimension (§6). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |

---

## Summary

> **What it is:** A microkernel-inspired organization for **intra-process** isolation. The **endokernel** — a self-protecting monitor nested *inside* the process — exports a universal **subprocess** abstraction over existing process interfaces and, crucially, **virtualizes the OS/system-call interface so the monitor cannot be bypassed** (closing the `/proc/self/mem`-class holes that prior MPK systems like ERIM/Hodor leave open). A translation layer lets developers express high-level, programmable policies (e.g. "virtual privilege rings": sandbox / mainbox / safebox) that lower to subprocess enforcement. The prototype, **Little Mac**, uses Intel MPK.
>
> **Who it's for:** Developers of single-process applications that mix mutually-distrusting or risky components (servers, multi-user apps, heterogeneous-language runtimes, serverless, unikernels) who want fine-grained, programmable, non-bypassable least-privilege without rewriting to multi-process.
>
> **What it protects:** In-process secrets, code, and resources from a compromised component — preventing intra-process privilege escalation through both memory *and* OS interfaces, with sealed-capability pointer sharing and runtime-revocable dynamic scopes.
>
> **What it costs (effort/money/performance):** Annotation effort (tool-assisted) + a recompile; preliminary overheads of <10% for privilege-separated NGINX and up to tens-of-% on some Linux apps. Bounded on Little Mac by the 16 MPK keys.
>
> **What it needs (hardware/OS/expertise):** Commodity Intel MPK (or CHERI/CET/VT-X/VDom), stock Linux with minimal seccomp/`dispatch` support, the annotation toolchain — and a developer to mark components.
>
> **Key tradeoffs:** Its distinctive advances over prior MPK compartmentalization are **(1) non-bypassable mediation of OS interfaces**, **(2) programmability/extensibility via a translation layer**, and **(3) support for real-world functionality (threads, signals, process control)** that earlier monitors broke — but it is an early NSPW prototype: TCB-LOC, switch/creation/memory costs, and license are unreported, the OS+hardware+monitor are trusted, side channels are out of scope, and security hinges on correct developer annotations.
>
> **Additional Notes:** A research-vision intra-process compartmentalization **System** in the MPK family — directly building on and critiquing [[erim]] (and Hodor/Cerberus/Jenny/Enclosure), distinguished by the **endokernel** (from Dautenhahn's Nested-Kernel lineage) and its OS-interface virtualization. Conceptually adjacent to the thread/data systems [[smv]]/[[arbiter]] and complementary to (proposed to nest inside) [[gvisor]], unikernels ([[unikraft]]), and [[cloudflare-workers]]-style isolates. Because it's an NSPW paradigm paper, treat its evaluation cells as **preliminary**, not a measured system characterization.

---

## Sources

- **Primary:** `papers/endoprocess.pdf` — Yang, Huang, Kaoudis, Vahldiek-Oberwagner, Dautenhahn, *Endoprocess: Programmable and Extensible Subprocess Isolation*, NSPW 2023, DOI 10.1145/3633500.3633507. All **(§N)** / Fig. citations reference this paper.
- **Availability:** no repository or license stated in the paper — **Unknown**.
- **Lineage:** builds on the authors' prior Endokernel/Nested-Kernel work and the MPK-isolation literature (ERIM, Hodor, Cerberus, Jenny, Enclosure, PKU Pitfalls).

### Cells flagged for further research
- **Availability › License** — ⚠️ Unknown: no repo/license in the paper.
- **Security › TCB size (LOC)** — ⚠️ no LOC for Little Mac/endokernel.
- **Efficiency › Domain switch/creation/call latency, MB/domain, inter-domain throughput, scaling Big-O** — ⚠️ not reported (preliminary NSPW prototype; only app-level % overheads in Figs. 3–4).
- **Usability › Required expertise, Debugging** + **Sys Design › Monitoring, Config complexity, Maintenance burden** — N/A (self-report policy 2026-06-09).
- Inferences marked `_Inferred:_` inline: crash isolation, privileges, domain-count bound (16 PKEYs), deployment scale.
- **Venue caveat:** NSPW vision paper — quantitative cells are preliminary, not a full evaluation.
