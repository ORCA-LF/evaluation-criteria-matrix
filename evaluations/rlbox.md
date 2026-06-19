# Compartmentalization Evaluation Matrix — RLBox

**Version:** v0.1
**System:** RLBox
**Paper:** S. Narayan, C. Disselkoen, T. Garfinkel, N. Froyd, E. Rahm, S. Lerner, H. Shacham, D. Stefan. Retrofitting Fine Grain Isolation in the Firefox Renderer (Extended Version). USENIX Security 2020. arXiv:2003.00572v2.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — sandboxing untrusted third-party libraries (image/audio/video/font decoders) inside the Firefox renderer. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a library; policies refine by `<renderer, library, content-origin, content-type>`. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Library** — a fresh sandbox per library (optionally per content-origin/type). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** (per-object/per-origin *policies* exist but the isolation unit is the library; per-object used only for audio/video/fonts, too costly for images). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — backends are software (SFI, OS process, WebAssembly); no hardware isolation primitive required. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **SFI (NaCl-based)**, **OS process isolation (seccomp-bpf)**, **WebAssembly (Lucet)** as pluggable backends; **boundary wrappers/marshalling** via the `tainted` C++ type system + auto-generated trampolines. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** (SFI / Wasm) **or Process** (process sandbox) — pluggable. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — custom runtime per backend for symbol resolution + comms; SFI needs a modified NaCl toolchain; Wasm needs the Lucet runtime; ~100 LOC Clang plugin for struct reflection. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **One-way memory isolation** of the library (library may access only sandbox memory, not arbitrary renderer memory); **data-flow safety** via `tainted` types + static IFC (all sandbox data tainted, must be validated before use); **control-flow safety** via callback whitelisting (`sandbox_callback`) + RAII unregister; **pointer swizzling** + **ASLR preservation** (no pointer leaks into the sandbox by construction). Renderer can access sandbox memory; not bidirectional. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **Unknown** — the paper does **not** discuss crash isolation or recovery; its threat model is about preventing the library from *compromising* the renderer, not surviving a sandbox crash. **a process-backend crash would be confined to the sandbox process, while SFI/Wasm backends share the renderer process — but this is not stated.** |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the central contribution. `tainted<T>` forces validation via compile-time errors; `verify()`, `copyAndVerify()` (double-fetch-safe), `freeze()`/`unfreeze()`, and trampolines automate swizzling/bounds checks. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Compiler/toolchain** (RLBox C++ library + `tainted` type system, ~100 LOC Clang plugin, modified NaCl / Wasm toolchain), **OS** (seccomp-bpf for process backend), **CPU**, **the host renderer**, and **developer-written validators**. The sandboxed library is explicitly *outside* the TCB. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Medium** for the RLBox-specific layer: RLBox C++ library **< 3K LOC**; ~100 LOC Clang plugin; **35 unique validators, < 4 LOC each** across all ported libraries. The underlying NaCl/Wasm runtime is large. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — side channels and transient-execution (Spectre) attacks explicitly **out of scope**; authors note SFI/process make some harder (limit transient reads/control flow) but do not claim resistance. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — prototype is not formally verified; validators are hand-written and it is up to the user to get these checks correct. Authors contrast with Sammler et al.'s formally-modeled approach. |
| Experimental validation available | Yes (specify) / No | **Yes** — micro/macrobenchmarks, 11 real-world websites, image/video/audio decode benchmarks, Apache + Node.js case studies; reproduced real CVE exploits (CVE-2015-4506 VPX, CVE-2018-5148 OGG) against the unsandboxed code; **shipped in stock Firefox 74 (Linux) / 75 (Mac)**. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **Not measured (no SPEC).** App-level: Firefox page latency **3% (SFI) / 13% (process)**; image decode **23–32% JPEG (SFI)**, **< 41% (process)**; zlib decompress **< 1%**; bcrypt hashing **27%**; Apache tail latency **+10%**; libGraphite font code **85% (Wasm)**. Machine: Intel i7-6700K @ 4 GHz, 64 GB, Ubuntu 18.04.1. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Custom microbench, not lmbench.** Empty call 0.02 µs native → **0.22 µs SFI (~10×)**, **0.47 µs process (spinlock)**, **7.4 µs process (condvar)**. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Custom microbench.** Sandbox creation **≈ 1 ms (SFI)**, **≈ 2 ms (process)**; hidden via sandbox pre-allocation pools. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | Same as domain switch — **0.22 µs (SFI)** / **0.47 µs (process, spinlock)** per cross-boundary call. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Not measured directly.** Shared-memory marshalling; spinlocks for latency-sensitive paths, condition variables for bulk (e.g. 4K decode). |
| Memory overhead per domain | MB/domain | **≈ 1.6 MB / SFI sandbox**, **≈ 2.4 MB / process sandbox** (private copy of library code + libc, stack, heap). |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor:** pre-allocated thread-local storage + aligned free virtual address space (SFI). **≈ 250 concurrent SFI sandboxes** = the demonstrated maximum; conservative production thresholds for lazy reclamation are **10 sandboxes (image decoding — JPEG/PNG) and 50 (webpage decompression)**. |
| Performance scales with domain count | Big O | **Memory: linear / O(n)** in sandbox count — explicitly stated. **CPU:** measured 20–40% steady up to ~250 sandboxes; **roughly constant / O(1) over that range — the paper reports the measured range, not an asymptotic claim.** |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — x86-64, no special hardware (evaluated on stock Intel). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock** — process backend uses seccomp-bpf (standard Linux); SFI/Wasm need no kernel changes. |
| What privileges does it need? | User / Root / Kernel access | **User** — runs entirely in userspace; seccomp-bpf is unprivileged. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | **Modified NaCl toolchain** (compiler, LLVM, ELF loader, libc, NASM) for SFI, or **Wasm/Lucet toolchain**; ~100 LOC Clang plugin for struct reflection. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Host app: Refactoring** of the library interface to RLBox APIs (`sandbox_invoke`, `tainted`, validators) — ~25% LOC increase, ~180 LOC/library. **Library: None** (no library changes — a design goal). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Library: Recompile only** (with SFI/Wasm toolchain). **Host: Source changes needed** at the library interface. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** — RLBox is a C++ library leveraging the C++ type system; sandboxed libraries are C/C++. |
| Other compatibility notes | [Describe] | Prototype C++11; production C++17 (`if constexpr` for readable error messages). Wasm toolchain immaturity inflates some overheads. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **≈ 2 person-days per library** + **~180 LOC (25% increase)**, mostly mechanical; e.g. JPEG 720→1058 LOC, PNG 847→1317 LOC. |
| Required expertise level | Low / Medium / High / Expert | **Medium** — much is compiler-guided/mechanical, but writing validators requires understanding domain-specific invariants (hardest part of migration). |
| Build effort | [build changes / time / deps] | Per-library recompilation with the modified SFI (NaCl) or Wasm toolchain; integration of the RLBox runtime component. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Standard** C++ tooling; **compile-time type errors** actively guide migration step-by-step, with custom readable error messages in C++17. |
| Failure modes visibility | [crashes/logs/error codes/silent] | **Compile-time** type errors during migration; **runtime** validator failures abort (DIE!); sandbox memory violations → fault/abort. Unhandled double-fetches remain a possible silent risk. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — MIT** (`github.com/PLSysSec/rlbox/LICENSE`, © 2019 UCSD PLSysSec). Artifact at `usenix2020-aec.rlbox.dev`; merged into the Firefox codebase. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Production** — ships in stock Firefox 74 (Linux) / 75 (Mac); also a general-purpose research framework. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — mechanism-agnostic by design; pluggable SFI / process / Wasm backends behind one API; double-fetch detection tools can be used alongside. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — runs side-by-side with Firefox's existing process-level privilege separation / Site Isolation; many independent sandboxes coexist. |
| Can stack effectively | ✅ / ❌ | **✅** for multiple instances of the same mechanism (hundreds of sandboxes, up to ~250); nesting RLBox-within-RLBox not evaluated. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Function calls** — `sandbox_invoke()` (renderer→sandbox) and registered callbacks via auto-generated trampolines (sandbox→renderer). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** — objects allocated in sandbox shared memory (`sandbox_malloc`), marshalled lazily via `tainted` pointers (no eager copy). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** call/callback semantics; process backend switches spinlocks↔condition variables underneath but exposes synchronous calls. |
| Interaction security/validation | [Describe] | **Type-checked** via `tainted` IFC + manual validators; callbacks must be explicitly registered (capability-like) with RAII unregister. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Library** — the renderer↔library interface. |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** — developer chooses which library to sandbox and writes the interface. |
| Boundaries flexible at runtime | ✅ / ❌ | **❌** for the boundary *definition* (which library, fixed at source/compile time); sandbox *instances* are demonstrably created/destroyed on demand. The boundary-flexibility classification is the evaluator's, not a paper claim. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Works with standard threads** — NaCl modified to drop its unique-OS-thread requirement; callback state kept in TLS. |
| Process model | fork/exec / Custom / N/A | Process backend uses separate sandbox processes + shared memory; SFI/Wasm are single-process / **N/A**. |
| POSIX compatibility | Full / Partial / Limited | **Sandboxed code:** syscall surface is restricted — process backend uses seccomp-bpf, SFI/Wasm restrict syscalls. **host app retains Full POSIX (userspace lib on stock OS) and sandboxed code is therefore "Partial" — the paper does not frame this in POSIX-compatibility terms.** |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | SFI loads a private copy of code per sandbox → memory grows linearly; ~250-sandbox ceiling (TLS / aligned VA); immature Wasm toolchain inflates overhead (85% on libGraphite). |
| Security caveats when layered | [Describe] | Assurance is only as strong as the manually-written validators; unhandled double-fetch bugs may remain; side channels / Spectre out of scope; isolation is one-way (renderer can read sandbox memory). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Desktop** (the Firefox browser); also demonstrated on **Server** (Apache module, Node.js plugin). |
| Deployment scale | Single device / Cluster / Large scale | **Single device** (per-user browser process); not framed as a deployment-scale dimension in the paper. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |

---

## Summary

> **What it is:** A C++ framework for retrofitting fine-grain isolation of untrusted libraries, using the C++ type system (`tainted` types + static IFC) to make data and control flow across the library boundary explicit and checked, with pluggable software isolation backends (SFI/NaCl, OS process, WebAssembly).
>
> **Who it's for:** Developers of large existing C/C++ applications — especially browsers — who need to incrementally sandbox third-party libraries without rewriting them.
>
> **What it protects:** The host application (Firefox renderer) from memory-safety bugs / compromise in untrusted libraries. One-way: the library cannot read or corrupt host memory and cannot leak pointers (ASLR preserved).
>
> **What it costs (effort/money/performance):** ~2 person-days + ~180 LOC (25% increase) per library; runtime 3% (SFI) / 13% (process) page-load latency, 18–25% memory; ~1.6–2.4 MB per sandbox; ~10× per-cross-call overhead.
>
> **What it needs (hardware/OS/expertise):** Commodity x86-64, stock OS, a modified SFI (NaCl) or Wasm toolchain, a C++ codebase, and medium developer expertise (chiefly for writing validators).
>
> **Key tradeoffs:** Type-driven automation + incremental, compiler-guided migration — but the hand-written validators remain trust-critical and error-prone; side channels / Spectre and unhandled double-fetches are out of scope; isolation is one-way only.
>
> **Additional Notes:** Mechanism-agnostic design is a strength for composability. Production reality: NaCl is deprecated, so the shipping Firefox sandbox uses the WebAssembly backend. The Tor team integrated the patches into Tor Browser.
