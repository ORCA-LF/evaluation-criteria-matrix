# Compartmentalization Evaluation Matrix — HAKC

**System:** HAKC (Hardware-Assisted Kernel Compartmentalization; compartment = a two-level **Clique** ⊂ **Compartment**, enforced by ARM PAC + MTE)
**Paper:** D. McKee, Y. Giannaris, C. Ortega Perez, H. Shrobe, M. Payer, H. Okhravi, N. Burow. *Preventing Kernel Hacks with HAKC.* NDSS 2022. (Purdue / MIT CSAIL / EPFL / MIT Lincoln Laboratory)
**Source file:** `papers/hakc.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-11

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment · **⚠️ Unclear** = not found, needs research.
>
> _Compartment = a **two-level structure**: a **Clique** is a partition of code + data (each function/object in exactly one Clique, assigned an MTE **color** Cc); a **Compartment** groups ≥1 Clique (≤16 Cliques, the MTE color count) with a globally-unique ID. Data is owned by exactly one Clique; a **data-access policy** (which Cliques a Clique may touch) and a **control-transition policy** (which Compartments/entry-Cliques it may call) are enforced **at runtime by ARM Pointer Authentication (PAC) + Memory Tagging Extension (MTE)**. On cross-Compartment control flow, data ownership is transferred (pointers recolored + resigned) and restored on return (§IV)._
>
> _A **kernel compartmentalization System** (primarily for Loadable Kernel Modules). Key idea: combine MTE's 16 colors with per-Compartment PAC contexts to get **≈4·10¹⁵ compartments** despite only 16 hardware tags, with **no added software TCB** during compartment execution (§IV)._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation / fault isolation** — apply least privilege to monolithic-kernel code (esp. buggy LKMs) so a data-only or code-reuse exploit in one module can't reach data/code "irrelevant to its task," approximating microkernel security without the redesign (Abstract, §I). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Hybrid** — code is partitioned into Cliques/Compartments (code-centric) and every data object is *owned* by exactly one Clique with an access policy (data-centric) (§IV). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Function (Clique) / Module (Compartment)** — a developer assigns individual functions + globals to Cliques; finer than per-module (HAKC supports intra-module compartmentalization) (§IV-A, §V). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Object / Allocation** — global, stack, and dynamically-allocated objects each belong to a Clique; MTE colors 16-byte-aligned regions ≥16 bytes (§II-A, §V). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **MTE + Other (ARM PAC)** — Memory Tagging Extension provides per-region colors; Pointer Authentication cryptographically signs pointers with a 64-bit context (QARMA cipher). HAKC combines them: PAC context = compile-time Compartment ID + runtime MTE color (ARMv8.5-A) (§II-A, §V-A). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — an LLVM compiler pass** that signs/colors pointers, inserts data-access + control-transition checks, and does ownership transfers on annotated sources (no runtime monitor) (§V-B, §V-D). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — partitions the single kernel address space into Clique/Compartment domains via pointer signing + tagging (§IV-B). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅ (load-time + compiler), but no runtime monitor** — the kernel module loader colors Clique code/data at load; enforcement is then **pure hardware (PAC/MTE)** while compartmentalized code runs — "HAKC does not rely on any additional TCB" during execution (§IV-B, §V-B). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Fine-grained data-access + forward-edge control-transition enforcement**: every pointer deref is checked (PAC auth from candidate context) to belong to the executing Compartment and an allowed Clique; indirect-call targets must be valid Compartment entry-Cliques; cross-Compartment data ownership is transferred. Prevents data-only attacks (e.g. submember corruption) and arbitrary code execution **even with bugs, without violating memory-safety or CFI** (§I, §IV, §VI-E). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅ (containment)** — "the damage the bug causes is contained to the compartmentalized code"; a corrupted pointer can't be dereferenced into an invalid Clique. _Caveat:_ non-pointer data *within* a Clique can still be corrupted → typically DoS, not privilege escalation (§VII-A). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the Compartment-transition mechanism *is* the validated boundary: indirect targets checked against the Compartment's entry tokens (Tn*), data ownership transferred via recolor+resign; a Clique-access policy gates intra-Compartment calls (§IV-A, §V-A). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **CPU/SoC (PAC + MTE implementation) + the core kernel outside the victim LKM + the compiler (LLVM pass).** No hypervisor/microkernel/monitor layer — "rooting trust in hardware avoids … growing the TCB." IO devices are **untrusted** (§III, §IV-B). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **No added runtime TCB (the headline claim) + Large core kernel** — enforcement is hardware; the trusted base is the core kernel + SoC, *not* a new software layer. ⚠️ No LOC quantification (the point is *zero added* software TCB during execution) (§I, §IV-B). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None / out of scope** — "Spectre, Meltdown, or Rowhammer," DMA, and hardware glitching are explicitly out of scope (§III). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — security argued via a CVE-mitigation analysis (567 CVEs) + two real-CVE case studies (CVE-2017-9074, CVE-2019-14815); the paper *suggests* formally verifying MM+IPC and HAKC-protecting the rest as future work (§II-B, §VI-E, §VII-A). |
| Experimental validation available | Yes (specify) / No | **Yes, but via instruction analogs** — no MTE hardware existed (June 2021) and PAC was only on locked-down Apple A12, so performance uses **PAC/MTE instruction analogs** (cycle/footprint-equivalent stubs, PARTS-style) on a Raspberry Pi 4; functional correctness via emulation. Worst-case estimate (§VI-A). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC; ApacheBench on `ipv6.ko` (55k LOC): 1.6%–24% overhead** (worst case ~20% fewer req/s + transfer rate; 10 MB payload only 2–4%). **Estimated via instruction analogs** (no real PAC/MTE silicon), claimed worst-case. Host: Raspberry Pi 4 8GB, Debian 5.10.24 / Linux 5.10 (§VI-B, §VI-A). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Not lat_ctx.** A Compartment transition = validate target + **recolor & resign all pointer arguments** (the dominant cost); reported as HAKC-operations/KB rather than a µs figure (§IV-A, §VI-B, Fig. 8). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **N/A** — Cliques/Compartments are colored by the kernel module loader at **LKM load time** (one-shot), not created per-call; no creation-latency figure (§V-B). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Not lat_pipe.** Cross-Compartment calls are signed indirect calls + data-ownership transfer; intra-Compartment Clique calls need only the access-policy check; captured in the throughput overhead, not isolated (§IV-A). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **N/A** — same address space, no IPC; ApacheBench transfer-rate is the reported end-to-end metric (Fig. 7). |
| Memory overhead per domain | MB/domain | ⚠️ **No MB/domain figure.** Overhead is the PAC signatures (in pointer bits) + MTE 16-byte-aligned coloring; no per-Compartment memory cost reported (§II-A). |
| Domain count bounded by | Limiting factor; approx. domain count | **Cliques/Compartment bounded by 16 (MTE colors); total compartments = 2^(64−Ntag)·Ntag ≈ 4·10¹⁵** — the two-level color-reuse design overcomes the 16-tag limit by orders of magnitude (§IV-A, §V-A). |
| Performance scales with domain count | Big O | **Linear in *related* Compartments** — two LKMs computing on the same data path compound at **14–19% per compartment**; unrelated compartments hypothesized to take the max, not the sum (future work) (§VI-C, §VII-B). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Specialized — ARMv8.5-A with PAC + MTE.** "HAKC is not tied to any specific architecture" (needs address-space partitioning + pointer-metadata association), but the prototype targets ARM; at publication no MTE silicon existed → evaluated on analogs (§IV-B, §VI-A). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Modified Linux** (5.10) — kernel module loader colors Cliques at load; LLVM-built kernel; runs **bare metal** (or in a VM with the HW features), no hypervisor needed (§V-B, §III). |
| What privileges does it need? | User / Root / Kernel access | **Kernel** — it *is* a kernel compartmentalization mechanism; the attacker is assumed *non-root* and unable to modify LKMs (§III). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | The HAKC LLVM pass; annotated kernel/LKM sources; kernel-loader support for HAKC ELF sections (§V-B). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Annotations** — developer assigns Cliques/Compartments via macros, marks kernel→Clique data transfers, and transfers dynamically-allocated data; "lightweight (largest single annotation … 74 lines)." The LLVM pass auto-inserts checks/transitions/signing (§V-B, Table III). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Source annotations + recompile** with the HAKC LLVM pass (skipped on un-annotated sources) (§V-B). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C** — the Linux kernel / LKMs; LLVM-based (§V-B). |
| Other compatibility notes | [Describe] | HAKC does **not** solve general pointer aliasing — only transferred pointers + struct-member/current-stack pointers are resigned; the programmer must invalidate lingering aliases of same-color data (§V). Page-level W^X interacts with recoloring (can't recolor read-only/code) (§V-B). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Low (annotation-based)** — simplest case is a single macro per source file; partition further only where wanted; no person-days figure. Optimal partitioning is NP-complete (Partition Problem) → heuristics future work (§V-B, §V-C). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | Build the kernel/LKM with LLVM + the HAKC pass (scheduled late, after inlining); annotate sources; loader handles coloring (§V-B, §V-D). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Failure modes visibility | [crashes/logs/error codes/silent] | A policy violation → PAC authentication fails → the (now-invalid) pointer cannot be dereferenced (fault); non-pointer corruption may cause DoS (§IV-B, §VII-A). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — GPL-2.0** [repo] (`github.com/mit-ll/HAKC`, GitHub SPDX = GPL-2.0; verified 2026-06-11). Paper states open-source release (§IX); license not in paper. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** prototype (NDSS 2022; DARPA/MIT-LL/NSF/ERC-funded) (§I, Acks). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — complementary to memory safety (CVE analysis: 229 compartmentalization-mitigable vs 193 memory-safety-mitigable, only 71 overlap) and can be *combined* with PAC-based CFI like PACStack; works alongside the kernel's page-level enforcement (§II-B, §VIII-C, §V-B). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — compartmentalized and unmodified LKMs coexist; multiple HAKC LKMs run together (ipv6.ko + nf_tables.ko) (§VI-C). |
| Can stack effectively | ✅ / ❌ | **✅** — two-level Cliques-in-Compartments + color reuse gives ≈4·10¹⁵ compartments; multiple compartmentalized LKMs compose (overheads compound only when related) (§IV-A, §VI-C). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Function calls (direct + signed indirect)** — same address space; indirect-call targets validated against entry tokens; direct cross-Compartment calls still transfer data ownership (§IV-A, §V-A). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory with ownership transfer** — data is owned by one Clique; crossing a Compartment boundary **recolors + resigns** the pointers to transfer ownership (and restores on return) (§IV-A, Fig. 5). |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Synchronous** — call/return ownership transfer; not framed as a dimension (§IV-A). |
| Interaction security/validation | [Describe] | Every deref PAC-authenticated against a candidate context (Compartment ID + Clique-access tokens + runtime color); indirect targets checked against all valid entry tokens; computationally hard (QARMA) to forge (§V-A, §VI-E). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Function/module (Clique/Compartment)** — intra- and inter-module (§IV-A, §VIII-A). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer (annotations) + Compiler (LLVM pass) + Hardware (enforcement)**; policy can also come from static/dynamic analysis (future work) (§IV-A, §V-C). |
| Boundaries flexible at runtime | ✅ / ❌ | **Mostly static** — Compartment IDs are baked into instructions as immediates for performance, making membership "permanent," but "HAKC can be extended to dynamically change compartmentalization policies" (§V-B). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | _Inferred:_ **Standard kernel threading** — no special threading model; per-pointer checks are thread-agnostic (PAC/MTE are architectural) (§IV–V). |
| Process model | fork/exec / Custom / N/A | **N/A** — kernel-internal compartmentalization, not a process model (§IV). |
| POSIX compatibility | Full / Partial / Limited | **N/A** — operates inside the kernel; doesn't change the user/POSIX ABI (§IV). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Related LKMs on one data path compound overhead (14–19%/compartment); general pointer aliasing unsolved (programmer must invalidate); ≤16 Cliques/Compartment; optimal partitioning NP-complete (§V, §VI-C, §VII). |
| Security caveats when layered | [Describe] | Core kernel + SoC trusted; **confused-deputy** via kernel-signed invalid data is possible (no practical solution short of formal verification); non-pointer data corruption (DoS) within a Clique; side channels / DMA / glitching out of scope (§III, §VII-A). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server / Desktop / Embedded** — commodity monolithic Linux kernels (esp. LKMs: networking, drivers); ARM target suits mobile/embedded too (§I, §VI). |
| Deployment scale | Single device | _Inferred:_ **Single device** — per-kernel compartmentalization; not framed as a scale dimension (§VI). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |

---

## Summary

> **What it is:** A compiler+hardware scheme that compartmentalizes a commodity monolithic kernel — especially its buggy Loadable Kernel Modules — using **ARM Pointer Authentication + Memory Tagging**. Code and data are partitioned into **Cliques** grouped into **Compartments**; every pointer dereference and indirect call is checked in hardware against a PAC context computed from the Compartment ID plus the runtime MTE color, and data ownership is transferred (pointers recolored/resigned) when control crosses a Compartment. Its trick is composing 16 MTE colors with per-Compartment PAC contexts to yield ≈4·10¹⁵ compartments — and crucially it adds **no software TCB** during execution (no hypervisor/microkernel/monitor).
>
> **Who it's for:** Kernel developers who want microkernel-like least-privilege isolation (especially around drivers/network stacks) on commodity hardware without redesigning the kernel or trusting a new software layer.
>
> **What it protects:** Kernel code/data from data-only and code-reuse exploits in buggy modules — confining a compromised module to exactly the data its access policy allows, even when the bug violates neither memory safety nor CFI.
>
> **What it costs (effort/money/performance):** Lightweight source annotations + an LLVM rebuild; estimated 1.6–24% overhead on `ipv6.ko` (worst ~20%), linear 14–19%/compartment when modules share a data path, and no user-noticeable difference in real web browsing. (Performance is measured via PAC/MTE *instruction analogs* — no real MTE silicon existed at publication.)
>
> **What it needs (hardware/OS/expertise):** ARMv8.5-A PAC+MTE (or any HW giving address-space partitioning + pointer-metadata association), an LLVM-built Linux 5.10 with annotated sources, bare metal — no hypervisor.
>
> **Key tradeoffs:** The first bare-metal, commodity-hardware kernel compartmentalization with **zero added runtime TCB**, fine-grained intra-module boundaries, and a clever escape from the 16-tag limit — but it needs (then-unavailable) MTE hardware so numbers are analog-estimated, leaves side channels/DMA/glitching and kernel-confused-deputy out of scope, doesn't solve general aliasing, allows non-pointer DoS, and optimal partitioning is NP-complete (heuristics future work). Complementary to memory safety, not a replacement.
>
> **Additional Notes:** A kernel-compartmentalization **System** in the intra-address-space family — the ARM PAC+MTE counterpart to the MPK kernel/userspace schemes ([[erim]], [[cerberus]], Hodor) and a relative of [[redleaf]] (Rust kernel isolation), Nested Kernel, and [[flexos]]; contrasts with CHERI (fixed capabilities, misses data-only attacks) and Mondrix (simulated, inter-module only). Distinctive in **rooting trust in hardware** to avoid the "turtles all the way down" monitor/hypervisor TCB. Same author cluster as [[endoprocess]] / much of the MPK-isolation line (Payer). The instruction-analog evaluation is the main caveat for its quantitative cells.

---

## Sources

- **Primary:** `papers/hakc.pdf` — McKee, Giannaris, Ortega Perez, Shrobe, Payer, Okhravi, Burow, *Preventing Kernel Hacks with HAKC*, NDSS 2022, DOI 10.14722/ndss.2022.24026. All **(§N)** / Table / Fig. citations reference this paper.
- **Repo:** `github.com/mit-ll/HAKC` (§II-A, §IX) — open source; **exact license not verified here**. **[repo]**
- **Hardware refs:** ARM PAC (ARMv8.3) + MTE (ARMv8.5-A); PARTS [50] (PAC analogs); QARMA cipher.

### Cells flagged for further research
- **Availability › License** — open-source repo confirmed (§IX); exact license string not verified.
- **Security › TCB size (LOC)** — by design *no added* runtime software TCB; core kernel + SoC trusted; not LOC-quantified.
- **Efficiency › Domain switch (lat_ctx), inter-domain call latency, MB/domain** — ⚠️ not isolated (overhead given as ApacheBench % + HAKC-ops/KB); **all numbers are instruction-analog estimates, not real MTE hardware** (§VI-A).
- **Composability › threading model, interaction semantics** — _Inferred_; **POSIX/process model** N/A (kernel-internal).
- **Usability › Required expertise, Debugging** + **Sys Design › Monitoring, Config complexity, Maintenance burden** — N/A (self-report policy 2026-06-09).
- **MEASUREMENT CAVEAT:** performance evaluated via PAC/MTE instruction analogs on a Raspberry Pi 4 (no MTE silicon at publication) — treat efficiency cells as estimates.
