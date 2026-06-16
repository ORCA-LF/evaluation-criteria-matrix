# Compartmentalization Evaluation Matrix — RLBox

**System:** RLBox
**Paper:** S. Narayan, C. Disselkoen, T. Garfinkel, N. Froyd, E. Rahm, S. Lerner, H. Shacham, D. Stefan. *Retrofitting Fine Grain Isolation in the Firefox Renderer* (Extended Version). USENIX Security 2020. arXiv:2003.00572v2.
**Source file:** `papers/rlbox.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-08

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend** (no value is fabricated):
> - **(§N)** = directly stated in the paper, cited by section.
> - **[repo]** = sourced from the project's git repo / site (not the paper).
> - **_Inferred:_** = a reasonable assessment by the evaluator, NOT a paper claim.
> - **⚠️ Unclear** = not stated in the paper or sources found; needs further research.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — sandboxing untrusted third-party libraries (image/audio/video/font decoders) inside the Firefox renderer (§1, §2). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a library; policies refine by `<renderer, library, content-origin, content-type>` (§2). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Library** — a fresh sandbox per library (optionally per content-origin/type) (§2). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** (per-object/per-origin *policies* exist but the isolation unit is the library; per-object used only for audio/video/fonts, too costly for images — §2). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — backends are software (SFI, OS process, WebAssembly); no hardware isolation primitive required (§6, §9). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **SFI (NaCl-based)**, **OS process isolation (seccomp-bpf)**, **WebAssembly (Lucet)** as pluggable backends; **boundary wrappers/marshalling** via the `tainted` C++ type system + auto-generated trampolines (§4, §6, §9). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** (SFI / Wasm) **or Process** (process sandbox) — pluggable (§6). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — custom runtime per backend for symbol resolution + comms; SFI needs a modified NaCl toolchain; Wasm needs the Lucet runtime; ~100 LOC Clang plugin for struct reflection (§6.1, §6.2, §9). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **One-way memory isolation** of the library (library may access only sandbox memory, not arbitrary renderer memory); **data-flow safety** via `tainted` types + static IFC (all sandbox data tainted, must be validated before use); **control-flow safety** via callback whitelisting (`sandbox_callback`) + RAII unregister; **pointer swizzling** + **ASLR preservation** (no pointer leaks into the sandbox by construction) (§4.2–§4.4). Renderer can access sandbox memory; not bidirectional. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **Unknown** — the paper does **not** discuss crash isolation or recovery; its threat model (§2) is about preventing the library from *compromising* the renderer, not surviving a sandbox crash. _Inferred:_ a process-backend crash would be confined to the sandbox process, while SFI/Wasm backends share the renderer process — but this is not stated. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the central contribution. `tainted<T>` forces validation via compile-time errors; `verify()`, `copyAndVerify()` (double-fetch-safe), `freeze()`/`unfreeze()`, and trampolines automate swizzling/bounds checks (§4.2–§4.4). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Compiler/toolchain** (RLBox C++ library + `tainted` type system, ~100 LOC Clang plugin, modified NaCl / Wasm toolchain), **OS** (seccomp-bpf for process backend), **CPU**, **the host renderer**, and **developer-written validators**. The sandboxed library is explicitly *outside* the TCB (§2, §4). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Medium** for the RLBox-specific layer: RLBox C++ library **< 3K LOC** (§9.1); ~100 LOC Clang plugin (§6.1); **35 unique validators, < 4 LOC each** across all ported libraries (§7.3). The underlying NaCl/Wasm runtime is large. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — side channels and transient-execution (Spectre) attacks explicitly **out of scope**; authors note SFI/process make some harder (limit transient reads/control flow) but do not claim resistance (§2). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — prototype is not formally verified; validators are hand-written and "it is up to the user to get these checks correct" (§4.3). Authors contrast with Sammler et al.'s formally-modeled approach (§8). |
| Experimental validation available | Yes (specify) / No | **Yes** — micro/macrobenchmarks, 11 real-world websites, image/video/audio decode benchmarks, Apache + Node.js case studies (§7); reproduced real CVE exploits (CVE-2015-4506 VPX, CVE-2018-5148 OGG) against the unsandboxed code (§7.1); **shipped in stock Firefox 74 (Linux) / 75 (Mac)** (§9.2). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **Not measured (no SPEC).** App-level: Firefox page latency **3% (SFI) / 13% (process)**; image decode **23–32% JPEG (SFI)**, **< 41% (process)**; zlib decompress **< 1%**; bcrypt hashing **27%**; Apache tail latency **+10%**; libGraphite font code **85% (Wasm)** (§7.2, §7.4, §7.5, §9.2). Machine: Intel i7-6700K @ 4 GHz, 64 GB, Ubuntu 18.04.1. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Custom microbench, not lmbench.** Empty call 0.02 µs native → **0.22 µs SFI (~10×)**, **0.47 µs process (spinlock)**, **7.4 µs process (condvar)** (§7.2). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Custom microbench.** Sandbox creation **≈ 1 ms (SFI)**, **≈ 2 ms (process)**; hidden via sandbox pre-allocation pools (§7.2). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | Same as domain switch — **0.22 µs (SFI)** / **0.47 µs (process, spinlock)** per cross-boundary call (§7.2). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Not measured directly.** Shared-memory marshalling; spinlocks for latency-sensitive paths, condition variables for bulk (e.g. 4K decode) (§6.2). |
| Memory overhead per domain | MB/domain | **≈ 1.6 MB / SFI sandbox**, **≈ 2.4 MB / process sandbox** (private copy of library code + libc, stack, heap) (§7.5.3). |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor:** pre-allocated thread-local storage + aligned free virtual address space (SFI). **≈ 250 concurrent SFI sandboxes** = the demonstrated maximum; conservative production thresholds for lazy reclamation are **10 sandboxes (image decoding — JPEG/PNG) and 50 (webpage decompression)** (§6.3, §7.5.3). |
| Performance scales with domain count | Big O | **Memory: linear / O(n)** in sandbox count — explicitly stated (§7.5.3). **CPU:** measured 20–40% steady up to ~250 sandboxes (§7.5.3, Fig. 6); _Inferred:_ roughly constant / O(1) over that range — the paper reports the measured range, not an asymptotic claim. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — x86-64, no special hardware (§7, evaluated on stock Intel). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock** — process backend uses seccomp-bpf (standard Linux); SFI/Wasm need no kernel changes (§6.2). |
| What privileges does it need? | User / Root / Kernel access | **User** — runs entirely in userspace; seccomp-bpf is unprivileged (§6.2). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | **Modified NaCl toolchain** (compiler, LLVM, ELF loader, libc, NASM) for SFI, or **Wasm/Lucet toolchain**; ~100 LOC Clang plugin for struct reflection (§6.1, §6.2, Appendix B). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Host app: Refactoring** of the library interface to RLBox APIs (`sandbox_invoke`, `tainted`, validators) — ~25% LOC increase, ~180 LOC/library. **Library: None** (no library changes — a design goal) (§4, §5, §7.3). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Library: Recompile only** (with SFI/Wasm toolchain). **Host: Source changes needed** at the library interface (§5). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** — RLBox is a C++ library leveraging the C++ type system; sandboxed libraries are C/C++ (§4.1). |
| Other compatibility notes | [Describe] | Prototype C++11; production C++17 (`if constexpr` for readable error messages) (§9.1). Wasm toolchain immaturity inflates some overheads (§9.1). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **≈ 2 person-days per library** + **~180 LOC (25% increase)**, mostly mechanical; e.g. JPEG 720→1058 LOC, PNG 847→1317 LOC (§7.3, Fig. 3). |
| Required expertise level | Low / Medium / High / Expert | **Medium** — much is compiler-guided/mechanical, but writing validators requires understanding domain-specific invariants ("hardest part of migration") (§7.3). |
| Build effort | [build changes / time / deps] | Per-library recompilation with the modified SFI (NaCl) or Wasm toolchain; integration of the RLBox runtime component (§6.1–§6.2). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Standard** C++ tooling; **compile-time type errors** actively guide migration step-by-step (§5), with custom readable error messages in C++17 (§9.1). |
| Failure modes visibility | [crashes/logs/error codes/silent] | **Compile-time** type errors during migration; **runtime** validator failures abort ("DIE!"); sandbox memory violations → fault/abort. Unhandled double-fetches remain a possible silent risk (§4.3, §8). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — MIT** [repo] (`github.com/PLSysSec/rlbox/LICENSE`, © 2019 UCSD PLSysSec). Artifact at `usenix2020-aec.rlbox.dev`; merged into the Firefox codebase (§ Availability, §9.2). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Production** — ships in stock Firefox 74 (Linux) / 75 (Mac); also a general-purpose research framework (§9.2, §10). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — mechanism-agnostic by design; pluggable SFI / process / Wasm backends behind one API; double-fetch detection tools can be used alongside (§6, §8). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — runs side-by-side with Firefox's existing process-level privilege separation / Site Isolation; many independent sandboxes coexist (§2, §8). |
| Can stack effectively | ✅ / ❌ | **✅** for multiple instances of the same mechanism (hundreds of sandboxes, up to ~250); nesting RLBox-within-RLBox not evaluated (§7.5.3). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Function calls** — `sandbox_invoke()` (renderer→sandbox) and registered callbacks via auto-generated trampolines (sandbox→renderer) (§4.2, §4.4). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** — objects allocated in sandbox shared memory (`sandbox_malloc`), marshalled lazily via `tainted` pointers (no eager copy) (§4.3). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** call/callback semantics; process backend switches spinlocks↔condition variables underneath but exposes synchronous calls (§6.2). |
| Interaction security/validation | [Describe] | **Type-checked** via `tainted` IFC + manual validators; callbacks must be explicitly registered (capability-like) with RAII unregister (§4.2–§4.4). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Library** — the renderer↔library interface (§4). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** — developer chooses which library to sandbox and writes the interface (§5). |
| Boundaries flexible at runtime | ✅ / ❌ | _Inferred:_ **❌** for the boundary *definition* (which library, fixed at source/compile time); sandbox *instances* are demonstrably created/destroyed on demand (§2, §6.3). The boundary-flexibility classification is the evaluator's, not a paper claim. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Works with standard threads** — NaCl modified to drop its unique-OS-thread requirement; callback state kept in TLS (§6.1, Appendix B). |
| Process model | fork/exec / Custom / N/A | Process backend uses separate sandbox processes + shared memory; SFI/Wasm are single-process / **N/A** (§6.2). |
| POSIX compatibility | Full / Partial / Limited | **Sandboxed code:** syscall surface is restricted — process backend uses seccomp-bpf, SFI/Wasm restrict syscalls (§6.2, this part cited). _Inferred:_ host app retains Full POSIX (userspace lib on stock OS) and sandboxed code is therefore "Partial" — the paper does not frame this in POSIX-compatibility terms. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | SFI loads a private copy of code per sandbox → memory grows linearly; ~250-sandbox ceiling (TLS / aligned VA); immature Wasm toolchain inflates overhead (85% on libGraphite) (§7.5.3, §9.1). |
| Security caveats when layered | [Describe] | Assurance is only as strong as the manually-written validators; unhandled double-fetch bugs may remain; side channels / Spectre out of scope; isolation is one-way (renderer can read sandbox memory) (§2, §4.3, §8). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Desktop** (the Firefox browser); also demonstrated on **Server** (Apache module, Node.js plugin) (§7.6). |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Single device** (per-user browser process); not framed as a deployment-scale dimension in the paper. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |

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
> **Additional Notes:** Mechanism-agnostic design is a strength for composability. Production reality: NaCl is deprecated, so the shipping Firefox sandbox uses the WebAssembly backend (§9). The Tor team integrated the patches into Tor Browser (§2).

---

## Sources

- **Primary:** `papers/rlbox.pdf` — Narayan et al., *Retrofitting Fine Grain Isolation in the Firefox Renderer* (Extended Version), USENIX Security 2020, arXiv:2003.00572v2. All inline **(§N)** citations reference this paper.
- **[repo]** `github.com/PLSysSec/rlbox` — LICENSE confirms **MIT**, © 2019 UCSD PLSysSec (header-only C++ library). Project site: `rlbox.dev`.
- Artifact: https://usenix2020-aec.rlbox.dev (cited in paper § Availability).
- CVEs reproduced as motivation: CVE-2015-4506 (libvpx), CVE-2018-5148 (libvorbis/OGG) (§7.1).

### Cells flagged for further research
- **Compartment crashes isolated** (Security/Resilience) — not discussed in paper; only inferred.
- **POSIX compatibility** — only the syscall-restriction part is cited; the POSIX framing is inferred.
- **Monitoring/orchestration support** — not addressed in paper.
- Several rating-style cells (config complexity, maintenance burden, boundary flexibility, deployment scale, CPU-scaling Big-O) are evaluator **inferences**, marked inline.
