# Compartmentalization Evaluation Matrix — CubicleOS

**Version:** v0.1
**System:** CubicleOS (compartmentalised library OS; compartment = a cubicle)
**Paper:** V. A. Sartakov, L. Vilanova, P. Pietzuch. CubicleOS: A Library OS with Software Componentisation for Practical Isolation. ASPLOS 2021. DOI 10.1145/3445814.3446731.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — mutually isolate library-OS/application components for data integrity + privacy, against malicious/vulnerable third-party components. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a component (cubicle). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Library / component** — a cubicle = a Unikraft component (VFS, RAMFS, LWIP, allocator, app…) compiled as a dynamic library; isolation expressed at the granularity of function calls. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Page** — windows and MPK operate at page granularity (developers must page-align structures to avoid unintended sharing). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **MPK (Intel Memory Protection Keys)** — one tag per cubicle, dynamically retagged via trap-and-map. A **minor MPK hardware modification** (tag-wide no-execute) is *proposed* for efficient CFI; the prototype works on stock MPK via load-time scanning. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Boundary wrappers** (auto-generated cross-cubicle call trampolines) + **CFI** + **load-time binary scanning** (SFI-like: rejects code containing `syscall`/`wrpkru`). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — cubicles share one address space, separated by MPK tags. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — CubicleOS's trusted runtime: component builder, cross-cubicle trampolines, memory monitor, cubicle loader; built on the Unikraft library OS, running on a host OS in a VM/container. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Spatial memory isolation** (cubicles own their pages; cross-cubicle access faults), **temporal memory isolation** (windows: dynamic, page-granular, zero-copy sharing), and **control-flow integrity** (cross-cubicle calls reach only public entry points) — giving data integrity + confidentiality between components. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | an unauthorized access raises a memory-protection fault (cross-cubicle corruption is prevented), but the paper describes **no crash-recovery / domain-restart** mechanism — the goal is confidentiality/integrity, not recovery. **Needs confirmation.** |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — cross-cubicle calls enforce CFI (only public entry points) and window ACLs gate memory access, **but** we do not target the validity of arguments or function call sequences used across isolation compartments. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Host OS kernel (Linux, assumed bug-free)**, the **compilation toolchain**, CubicleOS's **trusted runtime** (builder, trampolines, memory monitor, loader), the **trusted developer** who defines the isolation policy, and the **hardware + virtualization environment**. Third-party components/cubicles are untrusted. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **CubicleOS-specific TCB: small** — monitor/loader/trampolines ≈ **3,000 SLOC C + 110 SLOC ASM**, builder ≈ **640 SLOC Python**. **Full TCB is Large** (includes the host Linux kernel). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — we do not target malicious operators, side-channel attacks, or full protection against external input attacks (deferred to complementary work). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No.** |
| Experimental validation available | Yes (specify) / No | **Yes** — NGINX (8 partitions, 2× slowdown), SQLite (7 isolated + 4 shared cubicles, 1.7–8×), mechanism breakdown (trampolines/MPK/windows), and comparison vs Genode + seL4/Fiasco.OC/NOVA microkernels (3–5× faster on +1 compartment); developer effort measured. Artifact publicly available, Zenodo DOI 10.5281/zenodo.4321431. Host: Intel Xeon Silver 4210. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** SQLite **1.7–8×** vs non-isolated (avg 1.8× for OS-light queries, up to 8× for OS-heavy); NGINX **2×** with 8 partitions. Mechanism breakdown: trampolines +2–17%, MPK +50%–4×, windows +20%–1.2×. vs Unikraft baseline: CubicleOS-4 = 1.9× slower; vs microkernels, **3–5× faster**. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **No lat_ctx.** Permission switch via `wrpkru` ≈ **20 cycles** (vs `pkey_mprotect` ≈ 1,100 cycles, which CubicleOS avoids via trap-and-map); plus per-cubicle stack switch + first-access page-fault (trap-and-map). Trampolines add **2–17%** per query. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Unknown** — cubicles are loaded by the loader (dlopen-like, with code scanning); no load-latency figure reported. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Not lat_pipe.** Cross-cubicle call = trampoline + `wrpkru` (~20 cyc) + possible trap-and-map page fault on first window-page access. No isolated ns figure. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Zero-copy** via windows (no marshalling). High data volume hurts: NGINX files > 64 kB see throughput **halved** (2×) due to cross-cubicle data exchange. |
| Memory overhead per domain | MB/domain | **Unknown** — per-cubicle stacks + window-descriptor arrays + page-metadata maps, but **no MB/cubicle figure**. (Hard limit: 16 MPK tags → ≤16 cubicles without tag virtualization.) |
| Domain count bounded by | Limiting factor; approx. domain count | **16 cubicles** (MPK hardware tags, one per cubicle); extensible via tag virtualization (with overhead). Evaluated up to **8** (NGINX) / **11** (SQLite: 7 isolated + 4 shared). |
| Performance scales with domain count | Big O | Monitor lookups are **O(1)** (page-metadata map, cubicle-bitmask indexing); overall overhead scales with **cross-cubicle call frequency + shared data volume**. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Specialized (commodity-ish)** — Intel **MPK** (Skylake+); a *minor* MPK hardware modification is proposed for full CFI/no-execute but the prototype runs on **stock MPK** via load-time binary scanning. VT-x used for the microkernel baselines. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock host + custom library OS** — runs at user level on stock Linux (parses `/proc/self/maps`); CubicleOS itself is a modified **Unikraft** library OS. |
| What privileges does it need? | User / Root / Kernel access | **User-level** — MPK/`wrpkru` and window management run in user space; deployed in a VM/container. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | The trusted CubicleOS builder toolchain (LLVM 9.0 IR + GCC 8.3), Unikraft 0.4.0; each component compiled as a dynamic library. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Annotations / minor source changes** — developers add window-management calls (open/add/close) to grant cross-cubicle access; existing interfaces/organization preserved. SQLite 600–620 SLOC, NGINX 390–400 SLOC. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Source changes needed** — recompile each component as a dynamic library + add window management. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** — supports **type-unsafe** code (a contrast to language-based isolation). |
| Other compatibility notes | [Describe] | Retains **full POSIX compatibility** (Unikraft / feature-rich Linux apps) — a key design goal; developers must **page-align** structures to avoid unintended sharing via windows. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Small** — hundreds of SLOC per app (SQLite 600–620, NGINX 390–400), does not impact the organisation or interfaces; no person-days figure. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Build effort | [build changes / time / deps] | Components recompiled as dynamic libraries via the CubicleOS builder (extends Unikraft's build); cross-cubicle trampolines auto-generated. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Failure modes visibility | [crashes/logs/error codes/silent] | Unauthorized cross-cubicle access → page-fault → monitor checks the window ACL → grant (trap-and-map) or fault; load-time **rejection** of binaries containing `syscall`/`wrpkru`. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — MIT** (`github.com/lsds/CubicleOS`, GitHub SPDX = MIT; Zenodo DOI 10.5281/zenodo.4321431, branch `ASPLOS_AE`). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — prototype. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **Partial** — built on Unikraft; the paper suggests combining with SOAAP (annotation-based partitioning) to increase granularity. Incompatible with Intel SGX (MPK and SGX conflict). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — runs in a VM/container alongside other systems; many cubicles coexist. |
| Can stack effectively | ✅ / ❌ | **✅** — multiple cubicles compose (≤16 by MPK tags; more via tag virtualization). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Function calls** — cross-cubicle calls preserve ordinary function-call semantics via trusted trampolines (no message passing / IDL). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Zero-copy shared memory via windows** (dynamic, page-granular, owner-managed); plus **shared cubicles** (e.g. LIBC) whose static data is globally shared. |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** — execution migrates across cubicles on the same thread via the trampoline. |
| Interaction security/validation | [Describe] | **Window ACLs** (page-granular, owner-managed) + **CFI** (public entry points) + load-time integrity scanning; **does not** validate argument values or call sequences. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Library / component** (cubicle). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** defines the isolation policy (which components are isolated vs shared cubicles); the **builder** auto-identifies public entry points and generates trampolines. |
| Boundaries flexible at runtime | ✅ / ❌ | **Partial** — the **cubicle set is fixed at link/deploy time** (count known at link time, bitmask sizes fixed at deployment), but **windows (data sharing) are fully dynamic** at runtime; the loader supports dynamic (dlopen-like) loading. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom (Unikraft)** — user-level threads multiplexed onto a single host thread; per-cubicle stacks; per-thread MPK permissions. Multi-threading is limited (single host thread; MPK tag virtualization speculated to help). |
| Process model | fork/exec / Custom / N/A | **N/A — single-application unikernel** (Unikraft); the app is itself a cubicle; no multi-process model. |
| POSIX compatibility | Full / Partial / Limited | **Full** — retains Unikraft/Linux-application POSIX compatibility (an explicit design goal). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | ≤16 MPK tags (more via virtualization w/ overhead); **nested calls** need the owner to pre-open windows for all cubicles; developers must page-align structures; high-data-volume crossings halve throughput; the trusted builder conflicts with DevOps dynamic stacking. |
| Security caveats when layered | [Describe] | Argument validity / call sequences **not** validated; side channels / malicious operators / external-input attacks out of scope; stock MPK can't restrict `wrpkru` execution (handled by load-time scanning, not desirable) nor enforce tag-wide no-execute (needs the proposed hw modification for full CFI). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud** — library OSs deployed in containers, VMs, and TEEs. |
| Deployment scale | Single device / Cluster / Large scale | **Single device** (per-VM/container library OS); not framed as a scale dimension. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |

---

## Summary

> **What it is:** A compartmentalised library OS (built on Unikraft) that isolates each OS/application component in its own **cubicle** (an Intel-MPK-tagged dynamic library within one address space), while keeping ordinary function-call interfaces — data is shared **zero-copy** via page-granular **windows**, and control crosses cubicles through trusted **cross-cubicle call** trampolines that enforce CFI.
>
> **Who it's for:** Developers of library OSs / unikernels who want microkernel-style component isolation (data integrity + privacy) without rewriting interfaces around message passing, and while keeping full POSIX compatibility and type-unsafe C code.
>
> **What it protects:** Library-OS and application components from one another — a malicious/vulnerable component cannot read or corrupt another cubicle's memory except through explicitly opened windows.
>
> **What it costs (effort/money/performance):** Hundreds of SLOC per app to manage windows (SQLite ~620, NGINX ~390); runtime 1.7–8× (SQLite) / 2× (NGINX) vs non-isolated — but 3–5× faster than microkernels for the same partitioning.
>
> **What it needs (hardware/OS/expertise):** Intel MPK (commodity, with a minor hw modification proposed for full CFI), a stock Linux host, the Unikraft-based build with the trusted CubicleOS builder, and per-component window annotations.
>
> **Key tradeoffs:** Window-based zero-copy sharing keeps native call semantics and beats message-passing microkernels on performance — but it inherits MPK's limits (16 tags, no `wrpkru` control, no tag-wide no-execute without a hw change), pushes window/page-alignment management onto developers, doesn't validate cross-cubicle argument values, and leaves side channels out of scope.
>
> **Additional Notes:** A direct MPK counterpart to RedLeaf's language-based approach and a generalization of ERIM/Hodor (multiple untrusted components + zero-copy argument passing). Single-application unikernel model (Unikraft) — no multi-process.
