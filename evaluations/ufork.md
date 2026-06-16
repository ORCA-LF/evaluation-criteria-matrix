# Compartmentalization Evaluation Matrix — µFork (uFork)

**System:** µFork — POSIX `fork` within a single-address-space OS (compartment = a **µprocess**)
**Paper:** J. A. Kressel, H. Lefeuvre, P. Olivier. *µFork: Supporting POSIX fork Within a Single-Address-Space OS.* ACM SOSP 2025. arXiv:2509.09439v2.
**Source file:** `papers/ufork.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-09

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment, not a paper claim · **⚠️ Unclear** = not found, needs research.
>
> _Compartment = a **µprocess**: an emulated POSIX process living in a single shared address space, isolated from other µprocesses and the kernel via CHERI hardware capabilities. The paper's primary thrust is enabling `fork`; isolation is a co-equal design requirement (R3)._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation / Fault isolation** — provides POSIX-process-equivalent isolation between µprocesses (and µprocess↔kernel) in a single address space. **Parameterized (R4):** adversarial fault isolation (e.g. privilege separation, qmail), non-adversarial fault isolation (e.g. Nginx), or disabled (trusted, e.g. Redis snapshot) (§3.1, §3.6). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a µprocess (§3). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process** — the µprocess (an emulated POSIX process) (§3.4). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is process-granular (CHERI capabilities bound each µprocess's memory region). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **CHERI** — hardware capabilities (intra-address-space isolation) + memory tagging (pointer identification/relocation) + sealed capabilities (trapless syscalls) + fault-on-capability-load (for CoPA). Prototype on ARM Morello (§2.4, §4). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other** — capability-based (CHERI) isolation, position-independent code (PIC/PIE) to minimize relocations, and a novel **Copy-on-Pointer-Access (CoPA)** copy strategy; not SFI/managed-language (§3.4, §3.8). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — µprocesses share a single address space (the defining SASOS property) (§3). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — µFork is implemented as a kernel library in Unikraft (CHERI-ported); the kernel performs `fork`, pointer relocation, CoPA, and capability management. Runs on CHERI hardware (Morello) + bhyve/CheriBSD (§4). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Inter-µprocess isolation:** capabilities bound each µprocess to its own region; **monotonicity** prevents privilege escalation; CoPA relocation prevents capability leakage across µprocesses. **µprocess↔kernel isolation:** sealed capabilities restrict kernel entry to the syscall handler; the CHERI system-permission bit blocks privileged instructions; syscall args validated + by-reference buffers copied (TOCTTOU protection). Equivalent to POSIX process isolation (§4.2–4.4). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — provides POSIX-process-equivalent fault isolation; a bad reference outside a µprocess's memory triggers a hardware fault. Isolation level is **parameterized**: adversarial (privilege separation), non-adversarial fault isolation (catch bugs), or disabled (§3.6). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — the **kernel boundary** is validated (syscall argument sanitization + TOCTTOU copy of by-reference buffers, both toggleable for non-adversarial models). **Cross-µprocess application interfaces** are the app's responsibility ("we assume that user code using the `fork` API for isolation properly sanitizes cross-µprocess interfaces"), as with POSIX (§3.3, §4.4). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The OS kernel (including the µFork implementation)** + **CPU (CHERI hardware)**. The kernel distrusts all user code (µprocesses) (§3.3). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small–Medium** — µFork adds **~3,000 LoC**; total kernel changes **7 KLoC** (incl. 4,230 LoC for the CHERI + bhyve Unikraft port). Full TCB = the Unikraft SASOS kernel + µFork; **no total kernel LOC given** (§4). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — "We consider side channels, including transient execution attacks, out of scope" (§3.3). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — relies on CHERI's capability model (formally studied elsewhere); µFork itself is not formally verified (§4). |
| Experimental validation available | Yes (specify) / No | **Yes** — real apps (Redis snapshots, MicroPython/Zygote FaaS, Nginx) + microbenchmarks (fork latency, memory, Unixbench IPC) + CoPA vs CoA vs full-copy, vs CheriBSD and Nephele. Host: ARM Morello, 4× ARMv8.2-A @ 2.5 GHz, 16 GB (§5). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC; net win for fork-heavy workloads.** FaaS throughput **+24%** vs CheriBSD; Nginx (single-core) **+9%** requests; Redis save **1.4–1.9× faster** (§5.1). CHERI pure-capability mode has documented overheads, projected reducible to **1.8–3%** in future hardware (§5). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Trapless, single-privilege-level** user/kernel transitions via sealed capabilities (no costly trap). Unixbench Context1 (pipe IPC, 100k increments) **245 ms vs 419 ms** on CheriBSD (§5.2). No raw cycle count given. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Fork a µprocess in 54 µs** — 3.7× faster than CheriBSD's traditional fork (197 µs) and 198× faster than Nephele's VM-based fork (10.7 ms). Unixbench Spawn (1000 forks) **56 ms vs 198 ms** CheriBSD (Abstract, §5.2). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Fast SASOS IPC** (single address space, no page-table switch). Unixbench Context1 pipe benchmark 245 ms vs 419 ms CheriBSD (§5.2). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Unknown** — no bw_pipe-style throughput figure; shared memory (`shm_open`) is supported (described as straightforward) (§3.7, §5.2). |
| Memory overhead per domain | MB/domain | **~0.13 MB per minimal µprocess** (vs Nephele 1.6 MB, CheriBSD 0.29 MB). Forked Redis process **6 MB (CoPA)** vs 56 MB on CheriBSD at a 100 MB DB (§5.1, §5.2). |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: virtual address space** — µprocesses occupy contiguous VA regions (**fragmentation** is a noted limitation, mitigable via compaction/size-classes) — plus memory. 64-bit VA space expected ample; no specific count given (§6). |
| Performance scales with domain count | Big O | _Inferred:_ memory O(n) in µprocess count; CoPA reduces per-fork cost (up to 89× vs synchronous copy) (§5.1). No asymptotic statement. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Specialized — requires CHERI hardware** (prototype on ARM Morello; CHERI also on RISC-V — CHERIoT, Codasip X730) (§2.4, §4.1). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Custom** — µFork is built into Unikraft (a SASOS/unikernel), CHERI-ported; runs bare-metal or on the bhyve hypervisor on CheriBSD (§4, §5). |
| What privileges does it need? | User / Root / Kernel access | **It is the OS** — µprocesses and kernel run at the **same privilege level (EL1)**; isolation comes from CHERI capabilities, not privilege rings (§4.4). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | CHERI pure-capability toolchain; applications + kernel compiled as **PIC/PIE**; Unikraft; 16-byte CHERI pointer alignment (§3.7, §4.1). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None for µFork itself** — "Applications which run on Unikraft do not require porting to work with µFork"; unmodified `fork`-based apps run (Redis, Nginx, MicroPython). They must, however, be compiled PIC/PIE in CHERI pure-capability mode (§4). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Recompile only** — CHERI pure-capability, PIC/PIE; no application *source* changes, but not unmodified native binaries (CHERI requires recompilation) (§4). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++ + language runtimes** compilable to CHERI pure-cap (e.g. MicroPython demonstrated) (§5.1). |
| Other compatibility notes | [Describe] | Needs PIC/PIE; **SMP is limited** (Unikraft "big kernel lock", serializes kernel code — community work in progress); shared memory / shared libraries supportable but described as straightforward/future (§4.5, §3.7). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **~Zero for the application** (unmodified). Retrofitting µFork into a SASOS ≈ **7 KLoC** kernel work ("relatively modest"), plus per-OS process-state engineering (fd tables, task structs, PIDs, scheduling) (§4, §4.5). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | Compile in CHERI pure-capability mode, PIC/PIE; extended Unikraft build/linker scripts to place kernel + apps in contiguous VA regions (§4.4). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Failure modes visibility | [crashes/logs/error codes/silent] | Capability bounds/tag violations trigger **hardware faults** (CHERI); CoPA capability-load faults drive copying; isolation violations fault (§4.2–4.3). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Unknown — code unlicensed (deep search exhausted).** Repo at `github.com/flexcap-project/ufork` (§8). Verified 2026-06-09: every license file in the tree is a **vendored third-party dep** (micropython, lwip, newlib, compiler-rt); µFork's own code and README carry **no license**. The **paper** is CC BY 4.0, but that does not cover the code → **license Unknown — not resolvable from public sources (author contact not pursued)** [repo] |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — a prototype (§4, §8). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | _Inferred:_ **Partial** — built on CHERI + Unikraft; isolation is **parameterized (R4)** and could in principle use other intra-address-space mechanisms (MPK, MTE, Mondrian discussed as alternatives but not implemented) (§3.6, §6, §7). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | _Inferred:_ **✅** — µprocesses coexist in one address space; the SASOS runs bare-metal or virtualized under bhyve (§4, §5). |
| Can stack effectively | ✅ / ❌ | **✅** — many µprocesses coexist; `fork` builds µprocess hierarchies (parent/child) (§3.5). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **POSIX IPC** (pipes, signals, shared memory) + **syscalls** into the kernel; benefits from fast SASOS IPC (§3.7, §5.2). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** (`shm_open`, mapping the same physical pages into µprocess VA regions) + **message passing** (pipes); CoPA shares pages copy-on-pointer-access (§3.7, §3.8). |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Both** — synchronous syscalls/pipes + asynchronous signals (standard POSIX); not explicitly characterized (§3.7). |
| Interaction security/validation | [Describe] | Kernel validates syscall args + TOCTTOU-copies by-reference buffers; CHERI bounds enforce isolation; application is responsible for sanitizing cross-µprocess interfaces (as with POSIX) (§3.3, §4.4). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process** (µprocess) (§3.4). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** — via standard POSIX `fork` (§3.5). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — µprocesses are created dynamically at runtime via `fork` (§3.5). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | Each µprocess may have **many threads** (a thread is associated with a PID); **SMP via a big kernel lock** in Unikraft (immature; community work ongoing) (§3.4, §4.5). |
| Process model | fork/exec / Custom / N/A | **Full POSIX `fork`** (the central contribution) + standard process semantics; per-µprocess fd tables, task structs, PIDs, scheduling (§3.4, §4.5). |
| POSIX compatibility | Full / Partial / Limited | **Strong** — transparent POSIX `fork`; overall POSIX coverage is that of Unikraft's compatibility layer (broad but not complete) (§4). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Virtual-address fragmentation (contiguous regions); SMP immature; CHERI hardware-maturity overheads; static heaps in the prototype (§4.5, §5, §6). |
| Security caveats when layered | [Describe] | Side channels out of scope; disabling isolation (R4) removes protection — appropriate only for trusted threat models; apps must sanitize cross-µprocess interfaces (§3.3, §3.6). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud / FaaS / edge** — SASOS use-cases: Function-as-a-Service, edge computing, confidential computing (§1, §5.1). |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Large scale** — FaaS/cloud with many lightweight µprocesses; not framed as a scale dimension. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |

---

## Summary

> **What it is:** A single-address-space OS design (prototyped in Unikraft on CHERI / ARM Morello) that supports **transparent POSIX `fork`** by copying a parent µprocess's memory to a new location within the *same* address space — using CHERI capabilities for intra-address-space isolation (µprocess↔µprocess and µprocess↔kernel), CHERI memory tags to identify/relocate pointers, and a novel **Copy-on-Pointer-Access (CoPA)** strategy.
>
> **Who it's for:** Builders of lightweight SASOSes/unikernels (FaaS, edge, confidential computing) who need POSIX `fork` and multiprocess support *with* process-grade isolation, without giving up single-address-space lightweightness.
>
> **What it protects:** µprocesses from each other and from the kernel — equivalent to POSIX process isolation — with **parameterized** levels (adversarial, non-adversarial fault isolation, or disabled).
>
> **What it costs (effort/money/performance):** A net win for `fork` (54 µs, 3.7× faster than CheriBSD's fork, 198× faster than Nephele; ~0.13 MB/process); cost is the CHERI hardware requirement and CHERI recompilation, plus CHERI pure-cap overheads (projected reducible to 1.8–3% in future hardware).
>
> **What it needs (hardware/OS/expertise):** CHERI hardware (ARM Morello / CHERI RISC-V), a CHERI-ported SASOS (Unikraft), and PIC/PIE pure-capability compilation; no application source changes.
>
> **Key tradeoffs:** Brings *true* POSIX `fork` + process isolation into a single address space with order-of-magnitude better fork latency and memory than prior approaches — but depends on CHERI hardware (still maturing), needs recompilation, has SMP/fragmentation limitations in the prototype, and side channels are out of scope.
>
> **Additional Notes:** A flagship CHERI compartmentalization data point; closely related to FlexOS (build-time isolation schemas) and CompartOS/CHERIoT in the CHERI lineage. The parameterized-isolation idea (R4) — pay for isolation only when the threat model needs it — is notable and transferable.

---

## Sources

- **Primary:** `papers/ufork.pdf` — Kressel, Lefeuvre, Olivier, *µFork: Supporting POSIX fork Within a Single-Address-Space OS*, SOSP 2025, arXiv:2509.09439v2. All **(§N)** / Figure citations reference this paper.
- **[repo]** `github.com/flexcap-project/ufork` (stated open source, §8) — **no LICENSE file**; GitHub reports no SPDX license. Paper text licensed CC BY 4.0.
- _Note: this PDF triggered a "request too large" error on image-based reading; content was extracted via `pdftotext` instead._

### Cells flagged for further research
- **Availability › License** — code repo has no license file (paper is CC BY 4.0, but code license unstated); verify with authors.
- **Efficiency › Inter-domain throughput** — no bw_pipe figure.
- **Usability › Debugging support** — not discussed.
- **Sys Design › Monitoring/orchestration, Maintenance** — not discussed.
- **Security › TCB total size** — µFork adds ~3 KLoC (7 KLoC total kernel changes) but no full kernel LOC.
- Rating/classification inferences (interaction semantics, deployment scale, config complexity, integrate/coexist, required expertise, memory Big-O) marked `_Inferred:_` inline.
