# Compartmentalization Evaluation Matrix — HFI

**System:** HFI (Hardware-assisted Fault Isolation) — a **proposed ISA extension** for in-process isolation
**Entry type:** 🔧 **PRIMITIVE (proposed hardware)** — an isolation mechanism, not a full system; evaluated in gem5 simulation, no shipping silicon. Score availability/validation accordingly.
**Paper:** S. Narayan, T. Garfinkel, M. Taram, J. Rudek, D. Moghimi, E. Johnson, C. Fallin, A. Vahldiek-Oberwagner, M. LeMay, R. Sahita, D. Tullsen, D. Stefan. *Going beyond the Limits of SFI: Flexible and Secure Hardware-Assisted In-Process Isolation with HFI.* ASPLOS 2023. DOI 10.1145/3582016.3582023.
**Source file:** `papers/HFI.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-08

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment, not a paper claim · **⚠️ Unclear** = not found, needs research.
>
> **⚠️ Maturity note (important):** HFI is a **hardware proposal** — a new x86-64 ISA extension that **does not exist in any shipping CPU**. It is evaluated via **gem5 cycle-accurate simulation + compiler-based emulation** (cross-validated, 1.62% geomean difference), not on real silicon. This affects availability/usage and validation cells, and is a different maturity class from the deployed software systems in this set. Compartment = an in-process **sandbox** defined by hardware "regions."

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — fast, scalable in-process sandboxing of untrusted Wasm modules **and unmodified native binaries** (libraries, FaaS tenants) (Abstract, §1, §3). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — sandboxes code (Wasm instances, native binaries/libraries) (§3). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Library / in-process sandbox** — a Wasm instance, a sandboxed library, or a FaaS function, in-process (§3.1). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Region / Object** — explicit "small" regions are **byte-granular** (fine-grain object sharing); implicit regions are power-of-2 sized/aligned (§3.2). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **Other — HFI's own new ISA "regions" primitive** (base/bound/permission registers), a *proposed* hardware extension (not MPK/MTE/CHERI/TEE, and explicitly not MMU-based) (§3, §4). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Complements/replaces SFI** — integrates with Wasm/SFI in "hybrid" mode (trusting a Wasm compiler/verifier); for "native" sandboxes uses **boundary wrappers** (springboards/trampolines to clear registers + switch stacks) (§3.3, §7). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — multiple sandboxes share the process address space, isolated by region registers (§3). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — a **trusted runtime** manages sandboxes (region setup, `hfi_enter`/`hfi_exit`, exit handler); plus minimal OS support to save HFI registers on context switch (§3.3, §3.3.3). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Data + control-flow isolation** (all memory accesses + instruction fetches checked, speculative *and* non-speculative); **complete OS-interface mediation** (syscalls/exits interposed, converted to jumps); **Spectre mitigation by construction** (region checks complete before physical-address resolution → no out-of-bounds/secret data affects cache/architectural state); near-zero-cost setup/teardown/resize/share (§3, §4). Two trust models: **native** (code untrusted) vs **hybrid** (code trusted via compiler/verifier). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — a bounds/permission violation or hardware trap makes HFI disable the sandbox, record the cause in an MSR, and generate a **SIGSEGV** delivered to the trusted runtime's handler, which decides the next action (§3.3.2). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — HFI supports **complete mediation** of all paths out of the sandbox (system calls, `hfi_exit`) directly in hardware (near-zero-cost syscall interposition via jump-to-handler); the runtime fully mediates sandbox↔OS/host interaction (§3.1, §4.4). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The trusted runtime**, the **CPU/hardware** (the HFI extension), the **OS kernel** (saves HFI registers), and — for **hybrid** sandboxes — the **trusted Wasm compiler/verifier**. Native sandboxed code is fully untrusted (§3.1, §3.3). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Hardware addition is minimal**: 8 instructions, ~44 internal 64-bit registers (with switch-on-exit), one 32-bit comparator, a few AND/equality gates/muxes (§4). **Software TCB (runtime) size is not quantified as LOC.** |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **✅ Spectre/speculative — by design** (a key differentiator): region checks finish before physical-address resolution (data) and before decode/execute (control), so secrets never reach cache/architectural state; plus serialized `hfi_enter`/`hfi_exit` (~30–60 cyc) and the switch-on-exit extension (§3.4, §4.2). **Caveat:** holistic Spectre security still required — HFI doesn't change prediction-hardware influence and the runtime must prevent speculative bypass of `hfi_enter` (§3.4). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — security evaluated empirically with exploit suites (TransientFail, Google SafeSide) in gem5; no formal proof (§5.3). |
| Experimental validation available | Yes (specify) / No | **Yes (simulation/emulation only)** — gem5 cycle-accurate simulation + compiler-based emulation, cross-validated (1.62% geomean diff); SPEC06, Firefox font/image rendering, Wasm FaaS scaling, NGINX/OpenSSL native sandbox, Spectre-PHT/BTB exploit tests (§5, §6). **No physical hardware.** |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **SPEC06** (not 2017 — SPEC17 exceeds Wasm's 4 GiB): HFI is **~3.2% faster** than guard pages (geomean; 92.5–107.5% of guard-page runtime), vs software bounds-checking's **+34.7%** slowdown. Native sandbox: **0% execution overhead by design** (§6.1, §6.4). Sim host modeled on Intel Skylake; emulation on i7-6700K (§5.2). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Software-managed**, ~function-call cost (low 10s of cycles) using zero-cost transitions; **+30–60 cyc** when serialized for Spectre (avoidable via switch-on-exit) (§3.3.1, §3.4). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Near-zero** — sandbox setup/teardown/resize is "only a few HFI instructions to update region registers." FaaS teardown: **23.1 µs** (HFI) vs 25.7 µs (stock Wasmtime); heap growth **370 ms (HFI) vs 10.92 s (mprotect)** — ~30× (§3, §6.2, §6.3). (HFI does isolation only; allocation is the developer's job.) |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **~Function-call cost** — cross-sandbox communication is a function call in the shared address space (no IPC); function chaining "as fast as a function call" vs 1000–10000× for IPC (§2, §3.3). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Near-zero-cost shared memory** between sandboxes (shared regions, byte-granular) (§3.2). |
| Memory overhead per domain | MB/domain | **Near-zero virtual-memory overhead** — eliminates Wasm's 8 GiB guard regions; on-chip state is constant (measured in bytes) regardless of sandbox count (§3, §4). |
| Domain count bounded by | Limiting factor; approx. domain count | **Effectively unbounded** — "imposes no limit on the number of concurrent sandboxes" (constant on-chip state). Demonstrated: **256,000 × 1-GiB** Wasm sandboxes in one process (vs ~16K with guard pages) (§3, §6.3.2). |
| Performance scales with domain count | Big O | **O(1) on-chip state** per executing sandbox → scales to arbitrary concurrency (vs systems that keep per-sandbox on-chip state and hit hard limits) (§3, §4). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Specialized — the proposed HFI hardware extension, which does NOT exist in any shipping CPU.** Designed for x86-64 with minimal additions; evaluated in gem5/emulation; refined with "architects at a major CPU vendor" (§4, §5). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Modified (minimally)** — the OS extends `xsave`/`xrstor` to save HFI registers on context switch ("a simple and minimal change") (§3.3.3). |
| What privileges does it need? | User / Root / Kernel access | **User** — "HFI does everything in userspace; there are no overheads from ring transitions or system calls when changing memory restrictions, or entering and leaving a sandbox" (§1, §3.1). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | A trusted runtime; for Wasm, HFI-modified Wasm2c (AOT) and Wasmtime (JIT) (§5.1). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Native: None** — unmodified binaries, no ABI/compiler/libc/binary-format changes. **Wasm: runtime/compiler integration** (no app change) (§3, §5.1). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes (native sandbox)** — sandboxes unmodified native binaries *and* dynamically-generated/JIT code (a key advantage over Wasm) (§1, §3). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any** — native binaries (C/C++, hand-written assembly, SIMD intrinsics, JIT code) via the native sandbox; Wasm via hybrid (§1, §3.3). |
| Other compatibility notes | [Describe] | Supports SIMD/assembly/JIT (which Wasm cannot); precise Wasm trap semantics; 64K-granular heap growth (§3, §3.2). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Zero for native binaries** (unmodified); Wasm toolchain integration is one-time framework work (done by the authors) (§3, §5.1). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | Native: none; Wasm: a modified compiler/runtime (§5.1). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Failure modes visibility | [crashes/logs/error codes/silent] | A bounds/permission violation or hardware trap → HFI disables the sandbox, records the cause in an MSR, and delivers a **SIGSEGV** to the runtime's handler (§3.3.2). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — MIT** [repo] (`github.com/PLSysSec/hfi-root`, GitHub SPDX = MIT). The paper itself is CC-BY 4.0. _(HFI is a hardware design — the artifacts are the simulator/emulator + benchmarks, not deployable silicon.)_ |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research / Experimental** — a hardware proposal evaluated in simulation; not shipping (§5). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — "seamlessly integrate with current SFI systems (e.g., WebAssembly)" (hybrid mode trusts a Wasm compiler/verifier); complements safe-language and Spectre-resistant-CPU approaches (Abstract, §3.3, §7). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — non-sandboxed code runs with **zero impact** (a µarch design goal); multiple processes can use HFI concurrently (§3.3.3, §4). |
| Can stack effectively | ✅ / ❌ | **✅** — arbitrary concurrent sandboxes; a runtime can run itself inside a hybrid sandbox to protect itself while running child sandboxes (switch-on-exit) (§3, §3.4). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Function calls** — `hfi_enter`/`hfi_exit` (software-managed, function-call-order); cross-sandbox is a function call in the shared address space (§3.3). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** — shared regions at near-zero cost; byte-granular explicit regions for object-level sharing (§3.2). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** — function-call-order enter/exit (§3.3). |
| Interaction security/validation | [Describe] | **Complete OS-interface mediation** (interposition) + region permission checks + optional Spectre serialization; argument validation is up to the runtime; trust model differs native vs hybrid (§3.1, §3.3). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **In-process sandbox** (library/component granularity) (§3). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Runtime** — the trusted runtime sets up region mappings and enter/exit; the programmer builds the runtime (§3.1, §3.3). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — region registers are configured at runtime (`hfi_set_region`); sandboxes are created, resized, and shared dynamically at near-zero cost (§3.2, §3.3). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | The runtime multiplexes sandboxes across cores; per-core HFI state; the OS saves HFI registers on context switch, so it composes with standard OS threads (§3.3, §3.3.3). |
| Process model | fork/exec / Custom / N/A | **In-process** sandboxing; multiple processes can each use HFI (§3.3.3). |
| POSIX compatibility | Full / Partial / Limited | **N/A / full (mediated)** — HFI is an isolation primitive, not an OS; native sandboxes run normal binaries whose syscalls are mediated by the runtime (§3.3). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | 6 implicit + 4 explicit regions per sandbox (finite; multiplexed by the runtime for more); implicit regions power-of-2 sized/aligned; explicit "small" regions can't cross a 4 GiB boundary; **requires the (nonexistent) hardware extension** (§3.2, §4). |
| Security caveats when layered | [Describe] | Holistic Spectre approach still required (HFI doesn't change prediction-hardware influence; runtime must prevent speculative bypass of `hfi_enter`); hybrid sandboxes trust the compiler/verifier; **evaluated only in simulation** (§3.4, §5). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud / Server (FaaS, edge) + Desktop (browser)** — Wasm FaaS (Fastly/Cloudflare), browser library sandboxing (Firefox), high-performance data planes (§1, §6). |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — high-concurrency FaaS; 256K sandboxes/process (§1, §6.3). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |

---

## Summary

> **What it is:** A proposed minimal x86-64 ISA extension that provides hardware-enforced in-process isolation via **regions** (base/bound/permission registers), entirely in userspace — designed to replace/complement software SFI (Wasm) *and* to sandbox unmodified native binaries — with near-zero overhead, arbitrary sandbox scalability, complete syscall interposition, and by-construction Spectre safety. Evaluated in gem5 + compiler emulation (no shipping silicon).
>
> **Who it's for:** Wasm-runtime / FaaS-platform / browser builders who need fast, scalable, Spectre-safe in-process isolation — and anyone wanting to sandbox unmodified native code or JIT.
>
> **What it protects:** Untrusted in-process code (Wasm modules, native libraries, FaaS tenants) — data + control-flow isolation, OS-interface mediation, and speculative (Spectre) safety.
>
> **What it costs (effort/money/performance):** Near-zero runtime (SPEC06 3.2% *faster* than guard pages; native sandbox 0%); the real cost is requiring a new hardware extension that doesn't exist, plus a minimal OS change and a trusted runtime.
>
> **What it needs (hardware/OS/expertise):** The HFI hardware extension (proposed), a minimal OS change (save HFI registers), and a trusted runtime; native binaries need no changes.
>
> **Key tradeoffs:** Solves SFI's core problems at once — performance, scaling to arbitrary sandboxes, **Spectre safety**, and **unmodified-native-binary + JIT compatibility** — with minimal hardware; but it requires a hardware change absent from every shipping CPU (simulation-only), hybrid (Wasm) mode trusts the compiler/verifier, and complete Spectre security still needs a holistic approach.
>
> **Additional Notes:** Uniquely in this set, HFI (1) addresses Spectre/speculative side channels **by design** (almost every other system lists them out of scope) and (2) sandboxes **unmodified native binaries + JIT code**. But it is a hardware *proposal*, not a usable system — a different maturity class from the deployed software systems; weight that heavily in any comparison.

---

## Sources

- **Primary:** `papers/HFI.pdf` — Narayan et al., *Going beyond the Limits of SFI... with HFI*, ASPLOS 2023, DOI 10.1145/3582016.3582023. All **(§N)** / Table / Figure citations reference this paper. (Read via extracted text — `pdftotext`.)
- **[repo]** `github.com/PLSysSec/hfi-root` — GitHub SPDX = **MIT** (simulator/emulator + benchmark artifacts). Paper licensed CC-BY 4.0.

### Cells flagged for further research
- **Security › TCB size** — hardware additions quantified; software-runtime TCB not given in LOC.
- **Usability › Debugging support** — not discussed.
- **Sys Design › Monitoring/orchestration, config complexity, maintenance** — N/A / not discussed (hardware primitive, not a managed system).
- **(maturity)** — HFI is a **hardware proposal evaluated only in simulation**; weight availability/usage/validation accordingly. Confirm with ORCA team how to score proposed-but-unbuilt hardware in the matrix.
- Inferences (required expertise, config complexity) marked `_Inferred:_` inline.
