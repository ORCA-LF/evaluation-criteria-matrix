# Compartmentalization Evaluation Matrix — Salus

**Version:** v0.1
**System:** Salus (kernel-enforced intra-address-space process compartments; compartment = a secure compartment — a public/private memory region with PC-based access control)
**Paper:** R. Strackx, P. Agten, N. Avonds, F. Piessens. Salus: Kernel Support for Secure Process Compartments. EAI Endorsed Transactions on Security and Safety, Vol. 2, Issue 3, e1 (2015). (iMinds-DistriNet, KU Leuven) — extended version of the SecureComm 2013 paper.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Secret protection + untrusted component isolation** — strongly isolate sensitive data (e.g. crypto keys) *and* confine vulnerabilities so an attacker would need to exploit vulnerabilities in multiple compartments to reach her goal. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Hybrid** — code-centric compartments (public-section code) whose private data section is protected; access governed by which code (PC) is executing. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Function / Module** — a compartment is a code+data region with entry-point functions (e.g. one class → one compartment); granularity chosen by the developer. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Region (page-aligned)** — public/private sections are page-aligned so the MMU can enforce them; private section is the protected data unit. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None — standard MMU** (without relying on any non-standard hardware support); a pure-HW implementation exists in the authors' prior work but Salus deliberately targets commodity PCs. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — program-counter-based access control via the page-fault handler + capabilities (unforgeable references) + syscall filtering**; a C compiler/linker emits entry-point/return-entry-point handling. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — all compartments share one process address space; a lightweight programming model that enables legacy applications to be ported easily. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — a **modified Linux kernel** (page-fault handler + new syscalls + conflicting-syscall checks); plus the Salus C compiler/linker for compartment plumbing. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **(1) Strong data isolation** — private section accessible only from its own public section, entered only at entry points. **(2) Vulnerability containment** — an exploited compartment cannot affect others unless they trust it. **(3) Caller/callee authentication** — via a signed security report (hash of public section + layout). **(4) Unforgeable references** — guessing a compartment's address is insufficient; a (location, nonce) capability is required. **(5) Irrevocable syscall restriction** — dropped syscalls can't be regained, inherited by children. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — non-hierarchical isolation: the exploitation of a compartment must not affect the security of compartments other than those that explicitly trust the compromised compartment; for memory-safe compartments, related work proves **full source-code abstraction** (attacker limited to the public interface). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — entry-point enforcement + caller/callee authentication + unforgeable references + callee-side argument type-checking are the validated boundary; the security report (signed hash + layout) lets compartments verify each other and integrate code from different vendors. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **OS (the Salus-modified Linux kernel) + CPU/MMU + the trusted loader + crypto primitives.** The Salus compiler aids correctness but each compartment is mutually distrusting; kernel-level and physical attacks are considered out of scope. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large (whole Linux kernel) + small Salus delta** — the modification is a page-fault-handler change + 6 new syscalls + a handful of conflicting-syscall checks; **no LOC figure** given for the Salus delta. Per-app trusted compartments can be several orders of magnitude smaller than the risky ones. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None / out of scope** — defends in-application memory scanning (via unforgeable references) and assumes a Dolev-Yao model for crypto, but no cache/timing/speculative side-channel defense; kernel-level + physical attacks excluded. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No direct proof, but strong adjacent results** — Salus's PC-based protected-module model is the target of the authors' secure compilation / full abstraction proofs [27–30]; Salus itself is argued, not machine-verified. |
| Experimental validation available | Yes (specify) / No | **Yes** — security analysis (memory-safe vs unsafe compartments, ROP/ASLR benefits) + microbenchmarks (compartment vs function vs syscall) + macrobenchmarks (PolarSSL SSL server per-connection compartments; gzip parser isolation); SPECint2006 to show legacy apps are unaffected. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **SPECint2006: within ±0.4%** of the vanilla kernel for legacy (non-compartmentalized) apps — the kernel change is near-free until compartment boundaries are crossed. Host: Dell Latitude E6510, Intel Core i5 560M @ 2.67 GHz, 4 GiB, Ubuntu 12.04 / modified Linux 3.6.0-rc5. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Not lat_ctx, but measured: a compartment call = 4,024,227 CPU cycles** — **677× a function call** (5,944 cyc) and **~20× a system call** (193,970 cyc) — the cost of MMU access-right modification on boundary crossing. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Not isolated.** `salus_create` sets MMU rights for the new region (must not overlap existing compartments/kernel, not file-mapped); no creation-latency microbenchmark. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **= the compartment-call cost above (~4.0M cycles)** — inter-compartment calls are same-address-space calls through entry points (args via CPU registers, separate private stacks), so this *is* the inter-domain call latency. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **No bw_pipe figure.** Data crosses via registers / shared unprotected memory (no marshalling — a stated advantage over process-split schemes); not measured as bandwidth. |
| Memory overhead per domain | MB/domain | **No MB/domain figure.** Per-compartment cost = a `comp_struct` (sections, ID, saved SP, syscall-privilege list) + page-aligned public/private sections + a private call stack; not quantified. |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: address space + per-crossing cost** (the 677× call overhead pushes toward fewer, larger compartments). SSL benchmark creates one compartment per connection; no hard max stated. |
| Performance scales with domain count | Big O | overhead is **proportional to the number of compartment-boundary crossings**, not compartment count per se — there is a trade-off … between a low number of compartment transitions and small compartments. gzip overhead 21.9%→−0.5% as I/O dominates. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — any recent commodity computer with a standard MMU; no special hardware. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Modified Linux** (3.6.0-rc5 prototype: page-fault handler + 6 new syscalls + conflicting-syscall checks). |
| What privileges does it need? | User / Root / Kernel access | **kernel/root to install** the modified kernel; applications + compartments run at **user level** (the kernel is the only added trusted component). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | The Salus C compiler/linker (for entry points / return entry points / cross-compartment annotations); SSL benchmark used PolarSSL + a subset of diet libc. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Annotations + incremental refactoring** — developers annotate functions (entry point / unprotected / in another compartment); legacy apps are partitioned incrementally by first keeping data in unprotected memory, then moving it into a compartment once only one compartment accesses it. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Source changes + recompile** with the Salus compiler; **unmodified legacy apps run unaffected** (but unprotected) — SPECint within ±0.4%. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** — a C compiler/linker is provided; OO languages map naturally (one class → one compartment). |
| Other compatibility notes | [Describe] | **No multithreading** in the prototype (needs per-thread page tables — acknowledged, not fundamental); private sections are non-executable by default (hinders JITed code unless opted in); `fork`/`clone`/`mmap`/`mprotect`/`personality`/`madvise` required extra checks. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Moderate, tool-assisted** — incremental partitioning with compiler help, but manual inspection of code is still required to ensure sensitive data isn't touched from unprotected memory; no person-days figure. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator. |
| Build effort | [build changes / time / deps] | Compile with the Salus C compiler/linker; run on the modified kernel; annotate entry points. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator. _(Paper suggests a dev-time tool that logs access-right violations instead of stopping the app, citing [31]=Wedge — not implemented in Salus.)_ |
| Failure modes visibility | [crashes/logs/error codes/silent] | Invalid entry/access → MMU page fault → Salus denies (memory access violation); authentication/nonce mismatches return error values; revoked syscalls fail. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Paper: CC BY 3.0** (open access, copyright notice). **Code: Not found** — no repository stated and none located; KU Leuven research prototype. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** prototype (KU Leuven; Intel Labs URO + EU FP7 funded); extends the SecureComm 2013 paper. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — explicitly composes with other defenses: high-overhead countermeasures (bounds checkers) can be applied *only* to likely-vulnerable compartments; can host a NaCl-style sandbox in a compartment; a custom loader can add fine-grained ASLR. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — non-compartmentalized legacy apps run unaffected on the same modified kernel (±0.4%); compartmentalized and normal processes coexist. |
| Can stack effectively | ✅ / ❌ | **✅** — arbitrary numbers of compartments per process; **non-hierarchical** isolation lets compartments from different (possibly mutually-distrusting) vendors coexist, each choosing which security-report issuers to trust. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Function calls through entry points** — same-address-space calls; arguments/return addresses passed in CPU registers; a return entry point handles the call-site return. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared (unprotected) memory + register-passed args** — unprotected memory is read/write accessible from any compartment; **no marshalling** required (a stated advantage over process-split/NaCl). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** — blocking inter-compartment calls returning via the return entry point. |
| Interaction security/validation | [Describe] | Caller/callee authentication via signed security reports; unforgeable-reference nonce checked per entry point; callees type-check arguments; re-authentication via unique-until-reboot compartment IDs. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Module/compartment** (function-group with a public interface). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer (+ compiler)** — developer annotates/partitions; the Salus compiler emits the plumbing; the kernel enforces. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — compartments created/destroyed dynamically at any point (at the time a new plugin is loaded); attackers are even *allowed* to create compartments; syscall privileges drop dynamically (irrevocably). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Single-threaded (prototype limitation)** — shared page tables break the model for multiple threads; per-thread page tables would fix it but aren't implemented. |
| Process model | fork/exec / Custom / N/A | **Standard process + intra-process compartments** — `fork`/`vfork`/`clone` are guarded (CLONE_VM / VM_DONTCOPY) so compartments aren't mapped into children. |
| POSIX compatibility | Full / Partial / Limited | **Partial** — runs Linux/POSIX C apps but with restricted/guarded `mmap`/`mprotect`/`fork`/`personality`/`madvise` semantics for compartmentalized processes, no multithreading. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | No multithreading; high per-crossing cost (677× a function call) favors coarse compartments; private sections non-executable by default (JIT needs opt-in); several syscalls need special-casing. |
| Security caveats when layered | [Describe] | Whole Linux kernel trusted; kernel-level + physical + side-channel attacks out of scope; a secure *interface* is still hard to design (an authenticated caller can still abuse a compartment's public functions — the DigiNotar problem). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Desktop + Server** — consumer/desktop security apps (browsers, banking) and network servers (SSL CA / web server). |
| Deployment scale | Single device / Cluster / Large scale | **Single device** — per-process intra-application compartmentalization; not framed as a scale dimension. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator. |

---

## Summary

> **What it is:** A Linux-kernel modification that partitions a process into mutually-distrusting **secure compartments** sharing one address space. Each compartment has a public section (code + entry points) and a private section (secrets) protected by **program-counter-based access control** enforced with the *standard MMU* (no special hardware): the private section is reachable only while that compartment's own code runs. On top of strong data isolation, Salus adds **irrevocable system-call restriction**, **caller/callee authentication** (signed security reports), and **unforgeable references** (nonce capabilities) — so an attacker must compromise *multiple* compartments, with the right capabilities, to reach a target.
>
> **Who it's for:** Developers hardening security-sensitive C/C++ applications (crypto/CA services, SSL servers, browsers) who want fine-grained least-privilege and vulnerability containment without per-component processes/marshalling and without special hardware.
>
> **What it protects:** Sensitive data *and* the rest of the app from a compromised component — confining an exploit to its compartment, blocking guessing-based access (unforgeable references), and shrinking each compartment's kernel reach (syscall dropping).
>
> **What it costs (effort/money/performance):** Near-zero for legacy code (SPECint ±0.4%), but a **compartment call is ~677× a function call (~20× a syscall)** due to MMU right-switching — so coarse, crossing-light partitioning matters; real-app overhead was 12.7% (per-connection SSL server) and 21.9%→−0.5% (gzip parser, shrinking as I/O dominates). Annotation + incremental refactoring effort.
>
> **What it needs (hardware/OS/expertise):** Commodity hardware (standard MMU), a Salus-modified Linux kernel, and the Salus C compiler/linker — no special hardware, no marshalling.
>
> **Key tradeoffs:** Stronger than pure protected-module/TEE schemes (it *also* contains vulnerabilities via syscall dropping + auth + capabilities) and lighter to adopt than process-split schemes (shared address space, no marshalling, incremental porting) — but the per-crossing cost is high, the prototype is single-threaded, the whole kernel is trusted, and kernel/physical/side-channel attacks are out of scope.
>
> **Additional Notes:** A protected-module / privilege-separation **System** (intra-address-space), in the authors' **Fides/Sancus** lineage and the **direct relative of Wedge** (sthreads+callgates) — Salus adds unforgeable references, caller/callee authentication, and irrevocable syscall restriction, and the paper explicitly contrasts itself with Wedge, Capsicum, Privtrans, and NaCl. **This is the Salus that Smv and Arbiter cite as prior work** (Table 1, dynamic security policy); the `_BASELINES.md` trio and the SGX entries are unrelated. _(An earlier mis-filed `salus.pdf` — a 2024 Alibaba CPU-FPGA TEE coincidentally also named Salus — was the wrong paper and has been deleted; this `salus1.pdf` is the correct one.)_
