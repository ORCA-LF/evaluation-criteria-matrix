# Compartmentalization Evaluation Matrix — ERIM

**Version:** v0.1
**System:** ERIM (in-process isolation via Intel MPK + binary inspection)
**Paper:** A. Vahldiek-Oberwagner, E. Elnikety, N. O. Duarte, M. Sammler, P. Druschel, D. Garg. ERIM: Secure, Efficient In-process Isolation with Protection Keys (MPK). USENIX Security 2019.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Secret protection** (primary) + **untrusted component isolation** — isolating sensitive state/data (crypto session keys, CPI safe region, a managed runtime from native libs) from untrusted co-linked code. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Hybrid** — a trusted *code* component **T** owns an exclusive *data* region **M_T**; isolation partitions both code (T vs U) and data (M_T vs M_U). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Function / Library** — T is a set of marked functions or a dynamic library. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Region** — M_T is a reserved address-space region tagged to an MPK domain; MPK is page-granular. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **MPK (Intel Memory Protection Keys)** — up to 16 domains via the per-core PKRU register; `WRPKRU` switches in userspace. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Boundary wrappers (call gates)** + **binary inspection / rewriting** (forbids exploitable `WRPKRU`/`XRSTOR`); explicitly **not SFI** and **does not require CFI**. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — MPK domains within one process. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — a small ERIM library (~**569 LoC**: call gates, init, an M_T memory allocator, binary inspection) linked into each process, plus interception via seccomp-bpf+ptrace or an optional LSM. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **In-process memory isolation of M_T from U** — confidentiality+integrity (or an integrity-only/write-protection mode) — enforced by two invariants: M_T inaccessible while in U, and access enabled only atomically with a control transfer to a designated T entry point. Achieved **without CFI** in U; binary inspection guarantees no exploitable PKRU-updating instruction is executable. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **Unknown** — ERIM is a confidentiality/integrity mechanism, not a crash-isolation/recovery one. A detected exploitation attempt or illegal M_T access **terminates the program** (call-gate check / page fault); the paper does not frame domain crash isolation or recovery. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — call gates enforce that control enters T only at designated entry points (control-flow safety), **but** they intentionally do **not** pass/validate parameters (shared via a buffer in M_U) and do **not** scrub registers/stack — T must do that itself. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **CPU/hardware (MPK)**, the **OS kernel**, the **ERIM library** (~569 LoC, incl. binary inspection) linked into each process, and the **trusted component T** (assumed vulnerability-free, no leaks, scrubs secrets). U is fully untrusted — may contain memory-corruption + control-flow-hijack bugs. The OS loader is trusted to run ERIM's init first. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **ERIM-specific: Small** — runtime library **569 LoC**; the binary rewriter (~**2,250 LoC**) is a build-time tool, not runtime TCB. **System TCB: Large** (includes the OS kernel). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — side-channel and rowhammer attacks, and microarchitectural leaks ... are beyond the scope of this work (compatible with existing defenses). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — an informal security argument is given; not formally verified. |
| Experimental validation available | Yes (specify) / No | **Yes** — three applications (NGINX session keys, Node.js+SQLite runtime isolation, CPI safe region), switch-cost microbenchmarks, SPEC CPU, and comparisons vs MPX-SFI / lwC / VT-x VMFUNC. Host: Dell PowerEdge R640, 16-core Intel Xeon Gold 6142 @ 2.6 GHz; some micro on i5-6500 @ 3.2 GHz. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **SPEC CPU (2006, not 2017):** geomean **4.3%**, below **6.7%** on all tests. App-level: NGINX **≤4.8%** isolating all session keys (≤5%); CPI within **2.5%** except perlbench/omnetpp/xalancbmk (up to 17.9%). vs others: **6.3× / 13.5× / 3×** lower overhead than MPX-SFI / lwC / VT-x respectively. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Userspace call-gate roundtrip = 2× `WRPKRU`** (11–260 cycles each) + register ops + 1 branch; adds **55–80 cycles** over a no-switch call. Table 2 comparison: VMFUNC EPT switch 332 cyc, lwC switch 6,050 cyc — ERIM is far cheaper. Overhead **0.07–1.0% per 100,000 switches/s** at 2.6 GHz. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | One extra MPK domain + an M_T memory pool created **once at process init**; creation latency not separately measured (one-time). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | Same as domain switch — a userspace call gate (~tens of cycles, no syscall). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Unknown** — data is shared zero-copy via a buffer in M_U (no marshalling), but no throughput/bandwidth figure is reported. |
| Memory overhead per domain | MB/domain | **Unknown** — M_T holds an application-defined memory pool; no MB/domain figure reported. |
| Domain count bounded by | Limiting factor; approx. domain count | **16 MPK domains** (fewer in practice — the kernel may reserve some); extensible beyond 16 via libmpk (left as future work). |
| Performance scales with domain count | Big O | Switch cost is **constant / O(1)** (a fixed `WRPKRU` pair); total overhead scales linearly with switch rate (0.07–1.0% per 100k switches/s). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Specialized (commodity)** — Intel **MPK** (x86, Skylake-SP and later). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock** — runs on stock Linux ≥ 4.6 (MPK support); interception via seccomp-bpf + ptrace, or an optional LSM / an 8-LoC seccomp-bpf kernel change for efficiency. **No kernel changes required**. |
| What privileges does it need? | User / Root / Kernel access | **User** — call gates and `WRPKRU` run entirely in userspace; no syscall for switching. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | The ERIM library (via `LD_PRELOAD` for the binary-only approach); optional binary rewriting to remove inadvertent `WRPKRU`/`XRSTOR`; **no compiler changes required**. Requires DEP/W^X. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Three options:** **None** (binary-only — wrap an unmodified dynamic library via `LD_PRELOAD`); **Annotations/API** (source — wrap functions with `ERIM_SWITCH_T/U` call-gate macros, e.g. OpenSSL+SQLite **437 LoC**); or **compiler pass** (e.g. CPI, **300 LoC** in LLVM). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** for the binary-only approach (unmodified dynamic library + `LD_PRELOAD`); **source changes** for finer/custom isolation. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** primarily (binary-level); also protects **managed runtimes** from native libraries (Node.js case study). Language-agnostic on the untrusted side (operates on x86 binaries). |
| Other compatibility notes | [Describe] | x86-only (ARM memory domains lack userspace switching); requires DEP/W^X; prototype is incompatible with apps already using MPK for other purposes (not fundamental). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Small** — hundreds of LoC per app (OpenSSL+SQLite 437 LoC; CPI LLVM pass 300 LoC); no person-days figure reported. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Build effort | [build changes / time / deps] | `LD_PRELOAD` (binary-only) or recompile; optional binary-rewriting tool (~2,250 LoC) to eliminate inadvertent PKRU-updating sequences. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Failure modes visibility | [crashes/logs/error codes/silent] | Exploitation attempts / illegal M_T access → **program termination** (call-gate check on line 12 of the gate, or a page fault); binary inspection **rejects** executable pages with unsafe `WRPKRU`/`XRSTOR`. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — license Unknown.** Paper: Available online at `gitlab.mpi-sws.org/`… (ERIM project, §6 footnote). A GitHub mirror (`vahldiek/erim`) reports SPDX **NOASSERTION** (no detected license). **License not clearly stated.** |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — prototype with three case studies. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **Partial** — composable with libmpk (for >16 domains, future work), compatible with existing side-channel defenses, and orthogonal to auto-partitioning tools (SeCage-style). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **Partial** — the prototype **conflicts with applications already using MPK** for other purposes (not fundamental; resolvable if ERIM's reserved domain is not reused). |
| Can stack effectively | ✅ / ❌ | **✅** — generalizes beyond two components to as many as Linux's MPK support allows (≤16), with arbitrary transitive pairwise trust relations. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Function calls** via userspace **call gates** (no syscall) — `WRPKRU` to enable M_T, jump to a designated T entry point, and back. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** — a designated buffer in M_U accessible to both U and T (zero-copy; the call gate itself passes no parameters). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** — control transfers into T and returns on the same thread. |
| Interaction security/validation | [Describe] | Control flow restricted to T's designated entry points (call gates + binary inspection); **no** parameter/register validation — T must scrub registers and stack before returning. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Library / Function** — T is a marked set of functions or a dynamic library. |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** (marks/wraps T functions with call-gate macros or via `LD_PRELOAD`) or **Compiler** (compiler-pass approach). |
| Boundaries flexible at runtime | ✅ / ❌ | **❌** — the T/U partition and the reserved M_T domain are fixed at build/init time (M_T domain created before `main()`); pages can be assigned into M_T at runtime by T, but the boundary itself is static. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Works with standard threads** (pthreads) — the kernel saves/restores PKRU on context switch; T must allocate a **private per-thread stack in M_T** to be safe. |
| Process model | fork/exec / Custom / N/A | **In-process** — ERIM isolates within a single process; orthogonal to fork/exec. |
| POSIX compatibility | Full / Partial / Limited | **Full** — a userspace library on stock Linux; no POSIX restrictions on the application. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | ≤16 MPK domains; conflicts with other MPK users (not fundamental); x86-only; requires DEP/W^X; needs a list of correct T entry points. |
| Security caveats when layered | [Describe] | T is assumed vulnerability-free and must scrub registers/stack and not call back into U with M_T enabled; binary inspection must catch **all** `WRPKRU`/`XRSTOR`; side channels out of scope; integrity-only mode is weaker than full isolation. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server / Other** — network-facing services (NGINX), language runtimes, and general in-process data isolation; not tied to one target. |
| Deployment scale | Single device / Cluster / Large scale | **Single device** — per-process, in-process isolation; not framed as a deployment-scale dimension. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |

---

## Summary

> **What it is:** An in-process isolation technique that combines Intel **MPK** (cheap userspace domain switches via the PKRU register) with **binary inspection/rewriting** that forbids exploitable `WRPKRU`/`XRSTOR` instructions — giving hardware-enforced isolation of a trusted component's data (M_T) from untrusted code **without requiring CFI**, even at very high switching rates (<1% overhead at 100,000 switches/s).
>
> **Who it's for:** Developers isolating sensitive data (crypto keys, CPI metadata) or hardening a runtime against native libraries, where switching is frequent enough that page-table/hypervisor-based isolation is too slow.
>
> **What it protects:** A trusted component T's data region M_T from the untrusted rest of the process U — confidentiality+integrity, or an integrity-only mode.
>
> **What it costs (effort/money/performance):** Near-zero in-component overhead; 55–80 cycles per switch; SPEC geomean 4.3%, NGINX ≤4.8%. Effort: none (binary-only) to a few hundred LoC (source/compiler).
>
> **What it needs (hardware/OS/expertise):** Intel MPK hardware, a stock Linux ≥4.6 host (no kernel changes required), DEP/W^X, and the ~569-LoC ERIM library.
>
> **Key tradeoffs:** Dramatically cheaper than SFI/lwC/VT-x at high switch rates and needs no CFI or compiler support — but caps at 16 MPK domains, is x86-only, conflicts with other MPK users, leaves side channels out of scope, and pushes register/stack scrubbing and entry-point correctness onto the trusted component.
>
> **Additional Notes:** A foundational MPK-isolation paper; the direct ancestor of later MPK systems in this set (CubicleOS, Pegasus, Hodor, FlexOS-MPK). Its key idea — binary inspection to make MPK safe without CFI — is what distinguishes it from MemSentry's MPK instance.
