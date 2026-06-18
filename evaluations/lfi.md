# Compartmentalization Evaluation Matrix — LFI

**System:** LFI (Lightweight Fault Isolation) — an SFI scheme for ARM64
**Paper:** Z. Yedidia. *Lightweight Fault Isolation: Practical, Efficient, and Secure Software Sandboxing.* ASPLOS 2024. DOI 10.1145/3620665.3640408.
**Source file:** `papers/lfi.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-08

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment, not a paper claim · **⚠️ Unclear** = not found, needs research.
>
> _Compartment = a **sandbox**: an aligned 4 GiB region in a shared address space, into which untrusted code is loaded after a static machine-code verifier confirms its loads/stores/jumps are guarded. Pure software fault isolation (SFI), no hardware isolation primitive._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — sandboxing untrusted code with low latency for cloud/serverless; a lighter "stores+jumps only" mode targets **fault isolation / software compartmentalization** (read-only cross-sandbox) (Abstract, §1, §6.1). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — sandboxes code (an untrusted program/module) (§3). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process-like sandbox** — a 4 GiB isolation domain holding a program, in a shared address space (§3, Fig. 1). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is per-4 GiB-region. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — classic SFI via software guard instructions; uses only **standard page protections (W⊕X), set once at init** (no per-switch hardware involvement) (§3). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **SFI** — guard instructions on loads/stores/jumps (exploiting ARM64 addressing modes for ~0–1 cycle guards) + a **static machine-code verifier** (§3, §4, §5.2). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — tens of thousands of sandboxes in one address space (§1, §3). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — a runtime (manages sandboxes, mediated runtime calls, scheduling, fast IPC), the static verifier, and the LFI assembly transformer (§5). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Full software isolation of loads, stores, and jumps** — a sandbox cannot read/write/jump outside its 4 GiB region (the static verifier guarantees the reserved-register + guard invariants); plus a "no-loads" mode (stores+jumps only) that permits read-only cross-sandbox access for compartmentalization; Spectre-breakout safety by construction (§3, §5.2, §6.1, §7.1). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | _Inferred:_ **✅** — out-of-bounds accesses hit unmapped guard regions and **trap** (page fault → signal to the runtime); the runtime mediates and schedules sandboxes (§3, §5.3). Not framed as an explicit recovery experiment. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the runtime mediates all runtime calls and **checks arguments** (e.g., can disallow access to certain directories); the static verifier rejects any binary using unsafe instructions/addressing (§5.2, §5.3). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Static verifier** (+ its disassembler/ELF-reader/instruction-def dependencies), the **runtime**, and the **OS/CPU**. The **compiler and the LFI assembly transformer are NOT trusted** — the verifier checks the final machine code (§5.1, §5.2). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small** — verifier **core = 300 LOC Rust** (excludes LLVM's ~2M LOC from the TCB), though external deps (Binary Ninja disassembler, ELF reader, instruction defs) are "larger than we'd like." The LFI transformer (~1,500 LOC) is outside the TCB (§5.1, §5.2). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **✅ (partial, Spectre)** — **sandbox-breakout mitigated by construction** (LFI relies on no fine-grained CFI; all jumps are data-flow-guarded, so speculative jumps stay in-sandbox). Cross-sandbox / host **poisoning** needs ARM's **CSV2_2** extension (Cortex-X2+, not universal) (§7.1). Stronger than most software systems in this set. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** formal proof — but the verifier's disassembler/instruction definitions are auto-generated from **Arm's Machine Readable Specification (MRS)**, a formal ISA spec (§5.2). |
| Experimental validation available | Yes (specify) / No | **Yes** — SPEC CPU 2017 (14 benchmarks), comparison vs Wasm engines (Wasmtime/Wasm2c/WAMR), vs hardware virtualization (KVM) + gVisor, context-switch microbenchmarks, and a Spectre analysis. Hosts: Apple M1 (Asahi Linux) + GCP T2A (§6). Artifact (MPL 2.0). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **SPEC 2017**: geomean **6.4% (Apple M1) / 7.3% (GCP T2A)** for full isolation (O2); **~1%** for the stores+jumps-only ("no loads") mode; worst single benchmark 17% (leela). vs Wasm: roughly **half** Wasm's overhead (WAMR 18–22%, Wasm2c-pinned 16%, Wasmtime 47–67%) (§6.1–6.2, Tables 4). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Cross-sandbox `yield` = 17–18 ns (~50 cycles)**; syscall 22–26 ns (**6× faster than Linux's** 129–160 ns); no hardware mode/pagetable switch (§5.3, §6.4, Table 5). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | Verification ~**34 MB/s** (each SPEC binary verifies < 0.3 s); **`fork` supported in a single address space** (CoW via Linux `memfd`). No explicit end-to-end creation-latency number (§5.2, §5.3). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | `yield` (microkernel-like IPC) **17–18 ns (~50 cycles)** — potentially faster than optimized microkernel IPC (~400-cycle hardware limit) (§5.3, §6.4). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | `pipe` round-trip **46–48 ns** vs Linux 1504–2494 ns and gVisor 22899 ns (§6.4, Table 5). |
| Memory overhead per domain | MB/domain | **4 GiB virtual address space per sandbox** (+ 48 KiB guard regions), but mostly **unmapped** — physical use = actual usage. Code size: +12.9% text / +8.3% binary (§3, §6.3). |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: virtual address space** (4 GiB/sandbox). **~64Ki (2¹⁶) sandboxes** in 48-bit userspace (default ~65,000); up to **128Ki** with kernel address space (§3). |
| Performance scales with domain count | Big O | Constant per-sandbox runtime state (reserved registers); _Inferred:_ memory O(n) in virtual space. Scales to tens of thousands of sandboxes (§3). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **ARM64 (AArch64) only** — commodity ARM (Apple M1, AWS Graviton); exploits ARM64 addressing modes. **Not x86.** SVE scatter/gather is disallowed by the verifier (§2, §6). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock Linux** — standard page protections + signals; no kernel modifications (Asahi Linux / standard Linux in eval) (§5.3, §6). |
| What privileges does it need? | User / Root / Kernel access | **User** — userspace sandboxing; the runtime is a normal process. (Kernel address space optionally gives 128Ki sandboxes) (§3, §5.3). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | The LFI runtime + static verifier + **SFI-instrumented runtime libraries** (musl-libc, compiler-rt, libc++, libc++abi, libunwind) (§5.1). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Recompile** through the LFI toolchain (Clang/GCC → assembly → LFI transformer → assembler), with `-ffixed-reg` flags + the SFI-instrumented musl toolchain. No source changes for most C/C++ programs (§5.1). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Recompile only** — code must be transformed + verified through the LFI toolchain; not arbitrary existing binaries (§5.1). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** (LLVM/musl toolchain provided); broader feature support than Wasm — **exceptions, SIMD, setjmp/longjmp** all work; Fortran-capable. perlbench/blender need glibc (musl limitation) (§5.1, §6.2). |
| Other compatibility notes | [Describe] | **Compiler-toolchain-independent** — operates on GNU assembly text from any toolchain (works even for hand-written `.s` files) (§5.1). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Recompile** via the LFI toolchain; no compiler modification needed and no source changes for compatible C/C++ — minimal, but not quantified in person-days (§5.1). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | Compile → LFI transform (~1,500 LOC tool) → assemble; link with SFI-instrumented runtime libraries (§5.1). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Failure modes visibility | [crashes/logs/error codes/silent] | Out-of-bounds access → **trap** (unmapped guard region → page-fault → runtime signal); the verifier **rejects** unsafe binaries at load (§3, §5.2, §5.3). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — MPL 2.0** (paper Artifact A.2: "Code licenses: MPL 2.0") [repo] `github.com/zyedidia/lfi` (+ `lfi-artifact`). Paper licensed CC-BY 4.0. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — prototype oriented toward cloud/serverless adoption (§1, §6). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | _Inferred:_ **✅** — composes with virtualization/containers (can run inside a VM) and can use ARM extensions (CSV2_2 for Spectre); future hardware-SFI support discussed (§6.4, §7.1, §7.3). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — tens of thousands of sandboxes + the runtime coexist in one process; runs as a normal Linux process (§3, §5.3). |
| Can stack effectively | ✅ / ❌ | **✅** — up to ~64Ki sandboxes compose in one address space (§3). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Function calls** — fast cross-sandbox `yield` (~50 cycles, microkernel-like IPC); runtime calls via an in-sandbox call table (no trampoline) (§4.4, §5.3). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Message passing** (pipes) + runtime-mediated; the **"no-loads" mode permits read-only cross-sandbox reads**; shared memory possible via the runtime (§5.3, §6.1). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** (`yield` / runtime calls); preemption is signal-driven (§5.3). |
| Interaction security/validation | [Describe] | The runtime mediates calls + checks arguments; the static verifier guarantees isolation invariants (§5.2, §5.3). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Sandbox** — a process-like 4 GiB region (§3). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Runtime/loader** loads verified ELFs into 4 GiB slots; the programmer structures programs into sandboxes (§5.3). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — sandboxes are loaded dynamically and `fork` is supported (CoW); the size is fixed at 4 GiB (§3, §5.3). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | Runtime-managed scheduling with **signal-based preemption** (`setitimer`); reserved registers per sandbox (§5.3). |
| Process model | fork/exec / Custom / N/A | **✅ Unix-like process model** — the runtime implements `open`/`read`/`write`/`fork`/`wait`/`pipe`/`mmap` within one Linux process; `fork` works in a single address space (§5.3). |
| POSIX compatibility | Full / Partial / Limited | **Partial (good)** — the runtime provides a small Unix-like OS; musl libc (some glibc-specific programs unsupported) (§5.3, §6). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | ARM64 only; **4 GiB max per sandbox** (SPECspeed >4 GiB excluded); SVE scatter/gather disallowed; glibc-specific programs unsupported (musl) (§2, §6). |
| Security caveats when layered | [Describe] | Full Spectre poisoning mitigation needs ARM CSV2_2 (Cortex-X2+, not universal); the verifier's external disassembler deps are in the TCB; the "no-loads" mode deliberately allows cross-sandbox reads (§5.2, §7.1). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud / Server** — cloud, serverless, edge; many short-lived untrusted programs with low latency (§1). |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — tens of thousands of sandboxes per process (§1, §3). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |

---

## Summary

> **What it is:** The first software fault isolation (SFI) scheme for **ARM64** — it sandboxes untrusted code by rewriting loads, stores, and jumps with cheap guard instructions (exploiting ARM64 addressing modes for ~0–1 cycle guards), supports **tens of thousands of 4 GiB sandboxes in one address space** with fast (~50-cycle) context switches, and uses a small **static machine-code verifier** to keep the TCB tiny (no trusted compiler).
>
> **Who it's for:** Cloud/serverless/edge platforms running many short-lived untrusted programs that need both low CPU overhead *and* fast context switches, on ARM64.
>
> **What it protects:** Each sandbox from the others — full software isolation of loads/stores/jumps within a 4 GiB region (or stores+jumps only in the lighter compartmentalization mode).
>
> **What it costs (effort/money/performance):** 6.4–7.3% runtime overhead on SPEC 2017 (~1% for stores+jumps only), ~13% code size; recompilation through the LFI toolchain; ARM64 only.
>
> **What it needs (hardware/OS/expertise):** Commodity ARM64, stock Linux, the LFI runtime + verifier + SFI-instrumented musl toolchain; ARM CSV2_2 for full Spectre mitigation.
>
> **Key tradeoffs:** Near-virtualization CPU overhead *with* microkernel-beating context-switch speed and a tiny verifier-based TCB, plus broad software support (exceptions/SIMD/setjmp) and Spectre-breakout safety by construction — but it is ARM64-only, capped at 4 GiB per sandbox, requires recompilation, and full Spectre-poisoning mitigation needs a recent ARM extension.
>
> **Additional Notes:** A strong, well-rounded SFI data point — directly comparable to **HFI** (proposed hardware) and **Occlum's MMDSFI** (x86/MPX), but pure-software on commodity ARM64. The "verifier keeps the compiler out of the TCB" idea mirrors Occlum; the toolchain-independence is a notable maintainability advantage.

---

## Sources

- **Primary:** `papers/lfi.pdf` — Z. Yedidia, *Lightweight Fault Isolation: Practical, Efficient, and Secure Software Sandboxing*, ASPLOS 2024, DOI 10.1145/3620665.3640408. All **(§N)** / Table / Figure citations reference this paper. (Read via extracted text — `pdftotext`.)
- License: paper Artifact A.2 = **MPL 2.0**; **[repo]** `github.com/zyedidia/lfi` (+ `github.com/zyedidia/lfi-artifact`). Paper licensed CC-BY 4.0.

### Cells flagged for further research
- **Usability › Debugging support** — not directly discussed.
- **Efficiency › Domain creation cost** — verification rate given, but no end-to-end sandbox-creation latency.
- **Sys Design › Monitoring/orchestration** — runtime scheduler only; no orchestration described.
- Rating/classification inferences (crash isolation, integrates-with-other, required expertise, config complexity, maintenance, memory Big-O) marked `_Inferred:_` inline.
