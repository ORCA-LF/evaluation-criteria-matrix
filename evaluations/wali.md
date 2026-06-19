# Compartmentalization Evaluation Matrix — WALI

**Version:** v0.1
**System:** WALI (The WebAssembly Linux Interface)
**Paper:** A. Ramesh, T. Huang, B. L. Titzer, A. Rowe. Stop Hiding The Sharp Knives: The WebAssembly Linux Interface. arXiv:2312.03858v1 (Dec 2023), CMU + Bosch Research.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — run untrusted, polyglot Wasm modules in an in-process memory sandbox at near-native speed (cloud/edge + cyber-physical/embedded), with WALI giving them a full Linux syscall interface. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a Wasm module. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Library / module** — a WebAssembly module/instance, in-process. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** (Wasm linear-memory sandbox; "multiple isolated memories" can give disjoint privileged memory). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — WebAssembly is software sandboxing (no HW primitive). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Language runtime / bytecode VM (WebAssembly)** — memory-safe linear-memory sandbox + **control-flow integrity (CFI)**; WALI itself is **boundary wrappers/marshalling** (a thin syscall translation layer between Wasm and Linux). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — in-process Wasm sandbox; multiple WALI processes can be co-located in one native process. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — a WebAssembly engine implementing WALI; reference implementation on **WAMR** (WebAssembly Micro Runtime, interpreter + AOT). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Wasm sandbox:** modules cannot access external state unless explicitly imported; **CFI** rules out remote-code-injection/ROP from memory-safety bugs. **WALI's own model is *relaxed*:** it exposes the raw Linux syscall surface (no capability confinement like WASI) — trading OS-level confinement for compatibility/portability; syscall-level security is **delegated to complementary mechanisms (seccomp/Draco)**. "Multiple isolated memories" allow a privileged memory disjoint from app memory. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — a Wasm trap/memory-safety violation is contained within the module's sandbox; not framed as a crash-recovery mechanism. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — the Wasm boundary enforces memory-safety + CFI and translates/validates syscall pointer arguments across the Wasm↔native boundary; but WALI deliberately does **not** confine *which* syscalls a module may invoke (relaxed model) — that is left to seccomp/Draco. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The Wasm engine** (WAMR) + the **Linux kernel** (full userspace syscall surface is exposed) + **CPU/hardware**. WALI **shrinks the engine's TCB** by pushing higher-level API implementations (e.g. WASI) *out* of the TCB — implemented as sandboxed layers over WALI. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **WALI itself is ≈2,000 lines of C** (with <100 lines of arch-specific code per platform). For contrast, a WASI implementation built *over* WALI (libuvwasi) is **>6,000 lines in the engine** — WALI keeps that OS-feature surface thin and pushes higher-level APIs out of the engine TCB. The engine (WAMR) + Linux kernel remain the TCB; **no total engine/TCB LOC** reported → **Unknown** for the absolute size. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — not addressed (Wasm does not prevent µarch side channels; not discussed). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** for WALI itself — though it builds on WebAssembly, which has a machine-checked formal semantics (cited); WALI is not formally verified. |
| Experimental validation available | Yes (specify) / No | **Yes** — reference implementation on WAMR; evaluated across many real applications (e.g. bash, Lua, others) with intrinsic (syscall) and extrinsic (virtualization) overhead measurements. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Near-native for AOT-compiled Wasm; **≈2× slower than Docker on average**; the signal-handling polling scheme adds **<10%** over WALI-without-polling. _(The paper notes Lua runs up to **60% slower than native in Docker** — cited as a reason such allocation-heavy apps are good WALI candidates, **not** a WALI overhead figure.)_ WALI's own overhead is dominated by the underlying syscall + engine. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Unknown (no lmbench).** A Wasm→host syscall is the native syscall cost plus argument translation; no isolated context-switch figure. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Unknown** — Wasm instance/process creation latency not reported as a figure (WALI models Linux process/thread creation via `clone`). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Unknown / N/A** — modules interact via Wasm imports/host functions and standard Linux IPC (which WALI exposes); no isolated figure. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Unknown** — no inter-module bandwidth figure. |
| Memory overhead per domain | MB/domain | **Unknown** — Wasm linear memory per module; no MB/module overhead figure reported. |
| Domain count bounded by | Limiting factor; approx. domain count | multiple Wasm modules / WALI processes can be co-located in one native process; bounded by host memory/process limits. No specific count. |
| Performance scales with domain count | Big O | **Unknown** — not characterized. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — portable across host ISAs (x86, ARM, etc.) via the Wasm engine; targets embedded/cyber-physical + cloud/edge. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock Linux** — WALI *is* the Linux userspace syscall interface for Wasm (non-Linux hosts — Windows/BSD/Darwin/QNX — are future work). |
| What privileges does it need? | User / Root / Kernel access | **User** — runs as a normal process hosting the Wasm engine. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | A WALI-supporting Wasm engine (WAMR reference) + a Wasm-enabled toolchain. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Recompile only** — "porting applications only requires recompilation with a Wasm-enabled toolchain … comparatively trivial effort," because standard libraries/ABIs already operate over the Linux syscall API. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Recompile to Wasm** (not unmodified native binaries). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Polyglot** — C/C++, Rust, Java, C#, … (anything compiling to Wasm); Lua/Python via runtimes. |
| Other compatibility notes | [Describe] | Faithfully models the Linux syscall interface — supports `mmap`, `fork`/`exec`, async I/O, signals, threads (which **WASI lacks**), giving strong Linux/POSIX compatibility. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Low ("trivial" recompile)**; no person-days figure → otherwise **Unknown** for a precise number. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Build effort | [build changes / time / deps] | Recompile with a Wasm toolchain; link against the WALI-enabled engine. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Failure modes visibility | [crashes/logs/error codes/silent] | Wasm traps on memory-safety/CFI violations; syscall errors surface as normal Linux errnos through WALI. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — MIT** (`github.com/arjunr2/WALI`; GitHub SPDX = MIT). Paper: "WALI is fully open source and available at github.com/arjunr2/WALI". |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** prototype (reference implementation on WAMR), targeting cyber-physical/embedded + cloud/edge deployment. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — explicitly **complementary to WASI** (WASI and other capability APIs are implemented *as sandboxed layers over* WALI) and to **seccomp/Draco** for syscall-layer security. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — runs as a normal Linux process; multiple WALI/Wasm modules co-locate in one native process or run as separate processes. |
| Can stack effectively | ✅ / ❌ | **✅** — higher-level APIs stack as sandboxed layers over WALI; multiple modules compose. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Syscalls** (Wasm module → Linux via WALI imports) + **host-function calls**; cross-module via Linux IPC primitives that WALI exposes. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | Via the Linux syscall interface (shared memory, pipes, sockets) that WALI exposes; Wasm linear memories are otherwise disjoint. |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — WALI models synchronous syscalls plus async I/O and signals. |
| Interaction security/validation | [Describe] | Wasm boundary enforces memory-safety + CFI + pointer-argument translation; syscall *authorization* is **not** confined by WALI (relaxed model) — delegated to seccomp/Draco. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Wasm module/instance**. |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Compiler/Runtime** — the Wasm toolchain produces sandboxed modules; the engine enforces the sandbox. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — Wasm modules/instances are created/loaded dynamically by the engine. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Standard** — WALI models Linux threads via the `clone` syscall (Wasm threads → native threads). |
| Process model | fork/exec / Custom / N/A | **fork/exec supported** — WALI faithfully models Linux processes (`fork`/`exec`/`clone`), unlike WASI. |
| POSIX compatibility | Full / Partial / Limited | **Strong (Linux-syscall-faithful)** — supports `mmap`, `fork`/`exec`, async I/O, signals, threads; effectively Linux-ABI compatible (Linux-only). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Linux-only (non-Linux hosts future work); performance depends on the underlying Wasm engine; relaxed security model needs complementary confinement. |
| Security caveats when layered | [Describe] | Exposing the **full raw syscall surface widens the OS attack surface** vs WASI capabilities — *must* be paired with seccomp/Draco or a capability layer for OS-level confinement. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Embedded / cyber-physical** (IoT, automotive, industrial) **+ Cloud/edge** — long-lived, polyglot, legacy-software-friendly deployments. |
| Deployment scale | Single device / Cluster / Large scale | **Single device → Large scale** (edge devices to cloud fleets); not framed as a scale dimension. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |

---

## Summary

> **What it is:** A thin **WebAssembly system interface** that exposes Linux's userspace syscall API directly to sandboxed Wasm modules — letting entire Linux software stacks be recompiled to Wasm "as-is" and run portably across host ISAs, while inheriting WebAssembly's in-process memory sandbox and CFI.
>
> **Who it's for:** Developers virtualizing untrusted, polyglot, often-legacy software (cyber-physical/embedded, cloud/edge) who need broad Linux compatibility (mmap, fork/exec, signals, threads) that WASI doesn't provide.
>
> **What it protects:** Via WebAssembly — module memory isolation + control-flow integrity (no remote code injection). WALI itself does **not** confine the syscall surface (relaxed vs WASI capabilities); OS-level confinement is delegated to complementary seccomp/Draco.
>
> **What it costs (effort/money/performance):** Recompile-only porting ("trivial"); runtime near-native (AOT), ≈2× vs Docker on average and <10% for the signal-polling scheme; the security cost is a **wider OS attack surface** that must be re-confined externally.
>
> **What it needs (hardware/OS/expertise):** A WALI-enabled Wasm engine (WAMR reference) on stock Linux; a Wasm-enabled toolchain. Commodity hardware, any host ISA.
>
> **Key tradeoffs:** Maximizes Linux compatibility and portability and shrinks the engine TCB (API impls move out of the TCB, layered over WALI) — but deliberately trades WASI's capability confinement for raw syscall access, so the isolation is only as strong as WebAssembly's sandbox *plus* whatever OS-level confinement (seccomp) is layered on.
>
> **Additional Notes:** Like K23 and HFI, WALI is a **primitive**, not a full compartmentalization system — the isolation is WebAssembly's; WALI is the interface (and a security-relevant *de-confinement* tradeoff). Reference engine: WAMR.
