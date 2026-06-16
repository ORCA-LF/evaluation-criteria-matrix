# Compartmentalization Evaluation Matrix — Lind / lind-wasm (Grates · GrateOS)

**System:** Lind / lind-wasm — a WebAssembly-based POSIX sandbox whose compartment is a **cage**, plus **Grates**, composable userspace syscall-policy modules dispatched through the **3i** interface.
**Paper:** Anonymous. *Grates: Composable System Call Policies in Userspace.* (under submission; anonymized as "GrateOS"). `papers/grates-final-with-appendix.pdf`
**Source file:** `papers/grates-final-with-appendix.pdf` + project repo `github.com/Lind-Project/lind-wasm` (`/developer/nyu/lind/lind-wasm`)
**Matrix version:** v0.1
**Evaluated:** 2026-06-13

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from the lind-wasm repo · **_Inferred:_** = evaluator assessment · **⚠️ Unclear** = not found, needs research.
>
> _Naming: the paper anonymizes the implementation as **"GrateOS"**; the real project is **Lind / lind-wasm** [repo]. The paper's **3i** = the repo's `threei`; **RawPOSIX** = `rawposix`; **cage** = `src/cage` (with `vmmap` + `signal`); **fdtables** = `src/fdtables`._
>
> _Compartment = a **cage**: an isolated execution context with its own memory, threads, signal handlers, and file-descriptor namespace, behaving like a process. All cages share one host address space but each has a **disjoint WebAssembly linear memory** and a **per-cage programmable system-call table**. A **grate** is a cage that has used 3i to interpose on another cage's syscalls; grates compose by **linear sequencing**, **clamping** (conditional dispatch on a predicate), and **teeing** (duplicating a call across stacks) (§3)._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation + Other: composable syscall-policy enforcement** — run unmodified POSIX apps in isolated cages and route their syscalls through independently-written, reusable policy modules (tracing, allowlisting, filesystem confinement, write filtering, mTLS, replication) (Abstract, §1, §5.1). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a cage (an execution context); policy is attached at the syscall boundary, not to data objects (§3). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process** — a cage "behaves identically to a process from the application's perspective" (§3); grates are themselves cages (§3.2). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is per-cage (disjoint linear memory); no data-object tagging (§4.1). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** (prototype) — uses **software fault isolation via WebAssembly**; "the grate architecture does not rely on any specific isolation mechanism" and could run atop SFI, MPK, or processes (§3, §4.1, Appendix A; Table 2). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **SFI (WebAssembly) + Memory-safe language (Rust runtime) + syscall interposition (3i)** — apps compiled to wasm32 and linked against an interception-aware glibc that redirects syscalls into the per-cage 3i dispatch table (§4.1–4.4) [repo]. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Other — a "cage"** (an Intra-Address-Space SFI domain): a Wasm instance with disjoint linear memory + its own syscall table, process-equivalent semantics, all within one host address space (§3, §4.2). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the lind-wasm/GrateOS runtime: a modified **Wasmtime**, modified **glibc**, **3i** (`threei`), **RawPOSIX**, **fdtables**, **cage** (`vmmap`/`signal`), `lind-boot`; plus Binaryen passes (Asyncify, `fpcast-emu`, `wasm-opt`) (§4) [repo]. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Memory isolation between cages** — "each cage's linear memory is disjoint, so a buggy grate cannot address another grate's memory" (§4.1); **interposition completeness by construction** — "Wasm code cannot issue native syscall instructions," so no grate can be bypassed (§6); and **composable per-cage syscall policy** (clamping/teeing/sequencing) that kernel mechanisms can't express as independent modules (§3, §5.1). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | _Inferred:_ **✅** — cages have disjoint linear memory and Wasm traps on out-of-bounds/type violations; a fault is confined to the cage. Not framed as a crash-recovery experiment (§4.1–4.2). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — 3i is a single, uniform per-cage dispatch table (every handler has the same signature); cross-cage memory access uses `copy_data_between_cages` with **bounded copies validated against cage boundaries**, or zero-copy argument annotation that keeps pointers in the owning cage (§3.1). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The GrateOS/lind-wasm runtime** (modified Wasmtime + Rust components: 3i, **RawPOSIX** as the trusted terminal syscall shim, fdtables, cage), the **modified glibc**, the **host OS kernel**, and **CPU**. Individual grates are isolated policy modules, not part of each other's TCB (§3, §4.5). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Medium-Large, not quantified** — the Rust runtime + Wasmtime + modified glibc form the TCB; ⚠️ no single TCB-LOC figure. (Per-grate sizes 308–2,137 LoC are *policy* code, not TCB; Table 3.) (§4, §5.1). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | _Inferred:_ **None / out of scope** — not discussed in the paper (§ — absent). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal proofs; correctness argued empirically (§5.3). |
| Experimental validation available | Yes (specify) / No | **Yes** — SPEC CPU 2017 (Fig. 4), base-system microbenchmarks vs native Linux (Table 4), per-syscall overhead vs ptrace/eBPF/seccomp (Table 5), CPython/coreutils/PostgreSQL regression suites (Table 6), and real apps (NGINX, PostgreSQL, GCC/clang/tinycc, CPython, Perl, bash, coreutils, git, curl) (§5). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **SPEC CPU 2017: 1.09× (lbm) – 5.43× (gcc); 6 of 9 benchmarks within 3× of native** (Fig. 4). High-indirect-call workloads (gcc, perlbench) pay most (Wasm type checks + `fpcast-emu` ~20% on indirect calls); compute-bound near-native (§5.2). Host: Ubuntu 22.04 / Linux 6.11, 16-core Xeon Gold 5416S (§5.2). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Context switch: 2.70 µs vs 2.55 µs native** (Table 4) — near-native. Per-**grate** hop adds **~0.162 µs** (geteuid 0.207 µs via RawPOSIX → 0.369 µs through one grate), dominated by the Wasm trampoline, not 3i lookup (§5.2). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **fork: 10,117 µs vs 422 µs native (~25×); exec: 13,550 µs vs 449 µs** (Table 4) — dominated by Asyncify stack unwinding + Wasm module instantiation. New grate instantiation ~**10,187 µs** (compile a fresh Wasm module + reserve 4 GB VA + init workers) (§5.2). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **IPC round-trip (1 B): pipe 6.8 µs vs 5.7 µs; Unix socket 9.9 vs 7.4; TCP 21.1 vs 20.4** (Table 4) — near-native (calls forwarded to host kernel without interposition). The per-grate dispatch hop is ~0.162 µs (§5.2). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Not bw_pipe; file-read BW: 5,487 MB/s vs 6,566 MB/s native (64 MB, memory-backed)** (Table 4) — ~84% of native (§5.2). |
| Memory overhead per domain | MB/domain | **Each cage reserves a 4 GB virtual address window** (PROT_NONE; physical only on touch) — the dominant per-cage cost; not an MB-resident figure (§4.2, §5.2). |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: virtual address space (4 GB/cage).** ~**60 simultaneous cages** on a 256 GB / 48-bit host; up to **~16K** using the full 128 TB userspace VA range (Table 2, §5.2). ColorGuard ~15× denser; the memory64 proposal removes the 4 GB ceiling (§5.2). |
| Performance scales with domain count | Big O | _Inferred:_ **flat per hop** — composition adds ~0.162 µs/grate, "per-hop overhead is sub-microsecond, dominated by the Wasm trampoline rather than 3i dispatch" (§5.2 Key Takeaway). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity, any architecture** — Wasm SFI has no hardware dependency (Table 2; §4.1). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock** — "no kernel modifications or elevated privilege"; runs on an unmodified Linux (Ubuntu 22.04 / Linux 6.11 in eval) (Abstract, §5.2). |
| What privileges does it need? | User / Root / Kernel access | **User** — "zero-privilege deployment" (§4.1). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | **Modified glibc** + clang→wasm32 cross-compile toolchain + the Wasmtime-based runtime + Binaryen (`wasm-opt`/Asyncify/`fpcast-emu`) (§4) [repo]. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None (recompile only)** — "the application itself requires no modification; the deployer composes policies externally"; apps are cross-compiled to wasm32 and linked against the interception-aware glibc (§1, §4.1). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Recompile only** — unmodified POSIX source cross-compiled to wasm32 (not native binaries) (§4.1, §5.3). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** for applications (cross-compiled to wasm32); **grates** in "any language that compiles to Wasm" — current set in Rust, C, C++ (§3, §5.1). |
| Other compatibility notes | [Describe] | Broad POSIX (signals, fork, exec, threading, dynamic loading) sufficient for NGINX, PostgreSQL, GCC, CPython, coreutils, bash, git, curl; **path semantics don't fully compose** for descriptor-based ops (`fchdir`/`fchmod`/`fchown`) and `umask`-dependent behavior (§5.3, §7). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **~Zero for the app** (unmodified, recompiled). Writing a **grate** is 308–2,137 LoC (e.g. strace-grate 308, seccomp-grate 380, write-filter 519, imfs 1,574, ipc 2,137); none modified for composition (Table 3, §5.1). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | Cross-compile the app to wasm32 against the modified glibc; grates compile to Wasm; runtime applies Binaryen passes (§4) [repo]. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(A `strace-grate` exists as an observation tool, §5.1.)_ |
| Failure modes visibility | [crashes/logs/error codes/silent] | Policy denials surface as POSIX errnos (`EACCES`/`EPERM`/`ENOENT`); Wasm type/bounds violations trap; observation grates (strace-grate, attestation-grate) log calls (§5.1, §5.3). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — Apache 2.0** [repo] (`github.com/Lind-Project/lind-wasm`, root `LICENSE` = Apache 2.0; verified 2026-06-13). Paper is anonymized for review; source link in paper is an anonymized mirror. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — the Lind project (NYU); active open-source prototype; paper under submission (§4, [repo]). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — substrate-independent: "3i requires only memory boundary enforcement," so grate composition could run atop SFI, MPK, or processes (Table 2, §4.1); presented as **complementary to eBPF** (which works below the syscall boundary) (§6). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — runs as ordinary userspace; "other processes on the same host [are] entirely unaffected"; per-cage policy doesn't touch the host or other apps (§2.1, §3). |
| Can stack effectively | ✅ / ❌ | **✅✅ — the core contribution.** Grates compose by **linear sequencing + clamping + teeing**, "as independently written, reusable, per-process modules," with no coordination; ordering changes semantics (§3.2, §5.1). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Syscall dispatch via 3i** — `make_syscall` routes a call through the per-cage handler table to the next grate/handler (function-call-style across cages); `register_handler`/`copy_handler_table_to_cage`/`copy_data_between_cages` are themselves dispatched through `make_syscall` (Table 1, §3.1). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Bounded cross-cage copy or zero-copy annotation** — `copy_data_between_cages` (bounds-checked) when a grate needs the buffer; otherwise pointer arguments are annotated as owned by the originating cage and forwarded without copying (§3.1). |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Both** — synchronous syscall dispatch through the chain + asynchronous signal delivery (epoch-based) and teeing's best-effort secondary path (§3.2, §4.7). |
| Interaction security/validation | [Describe] | Cross-cage copies validated against cage boundaries; clamping enforced by an ancestor grate **interposing on `register_handler`**; `fdtables` translates (cage, virtual-fd) → backing resource per layer (§3.1–3.3). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process (cage) / policy module (grate)** — each grate is a standalone single-purpose module (§3.2). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Deployer + Programmer** — the **deployer** composes the policy chain externally on the command line (the declarative ground truth); **programmers** ("the same developers who write seccomp/eBPF") author individual grates (§1, §3). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — the grate chain is specified per launch; **inherited across fork/exec** via `copy_handler_table_to_cage`; dynamically-loaded modules inherit the chain; a deployer can query a cage's active handler table at runtime (§3.1, §3.2, §4.8). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom** — each guest thread maps to a native host thread (WASI-threads-like, GrateOS's own implementation); per-worker stack slots for re-entry (§4.3, §4.6). |
| Process model | fork/exec / Custom / N/A | **fork/exec (emulated)** — implemented via Binaryen's **Asyncify** stack capture/restore; `fork` clones the active call stack into a new cage, `exec` rewinds and replaces the module (§4.6). |
| POSIX compatibility | Full / Partial / Limited | **Partial (broad)** — signals, fork, exec, threads, dynamic loading, and most filesystem/IPC calls work (NGINX/PostgreSQL/GCC/CPython run); documented gaps in descriptor-based path semantics + `umask`/inode-level state (§5.3, §7). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | **Path semantics don't fully compose** — POSIX mixes path- and descriptor-based ops, so userspace FS interposition can't fully virtualize `fchdir`/`getcwd`/inode state (coreutils worst case ~77% of baseline); fork/exec/signal emulation is expensive (§5.3, §7). |
| Security caveats when layered | [Describe] | Wasm SFI adds compute overhead (a real tradeoff vs MPK substrates); RawPOSIX is the trusted terminal shim; side channels not addressed; path-semantics gaps can weaken filesystem-confinement grates on descriptor-based operations (§4.1, §5.3, §7). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server / Cloud / Desktop** — full web stack (NGINX + PostgreSQL), compiler toolchains, and standard Unix userland; deployable broadly thanks to zero-privilege, no-kernel-change design (§1, §5.3). |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Single device** — per-host cages + policy chains; not framed as a scale dimension (§5). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(Design intent: declarative command-line policy composition, §3.)_ |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |

---

## Summary

> **What it is:** A WebAssembly-based system (Lind / lind-wasm; anonymized in the paper as "GrateOS") that runs unmodified POSIX applications in isolated **cages** — Wasm instances with disjoint linear memory and a *per-cage programmable system-call table* — and a policy framework, **Grates**, that makes syscall policy **composable** the way Unix pipes make programs composable. Small, independent, reusable policy modules (a grate each: in-memory FS, strace, seccomp, mTLS, attestation, redaction…) are placed in sequence and combined via two primitives, **clamping** (conditional dispatch) and **teeing** (call duplication), all dispatched through the **3i** interface. No kernel changes, no elevated privilege.
>
> **Who it's for:** Deployers who want to compose multiple, independently-written syscall policies over unmodified apps — something seccomp/eBPF/ptrace/namespaces can't do as reusable modules — and systems developers who write those policy modules.
>
> **What it protects:** Each cage's memory from every other (disjoint Wasm linear memory; a buggy grate can't reach another's memory) and the host from untrusted app behavior, with interposition that is complete by construction (Wasm can't emit native syscalls) and policy that composes per-process and inherits across fork/exec.
>
> **What it costs (effort/money/performance):** No app changes (recompile to wasm32); composition itself is sub-µs/hop. The cost is the Wasm substrate: SPEC 1.09–5.43× (6/9 within 3×), near-native context switch/IPC/file-IO, but fork/exec ~25× native (Asyncify + module instantiation) and 4 GB VA per cage (~60–16K cages).
>
> **What it needs (hardware/OS/expertise):** Commodity hardware (any arch), a stock unmodified Linux at user privilege, a modified glibc + clang→wasm32 toolchain + the Wasmtime-based runtime.
>
> **Key tradeoffs:** Uniquely turns syscall policy into composable, reusable, per-process userspace modules with strong SFI isolation and zero-privilege deployment — at the price of Wasm compute overhead, expensive process/signal emulation, and POSIX path-semantics gaps for descriptor-based operations. The composition model is substrate-independent (could move to MPK/process for cheaper hops at some isolation/portability cost).
>
> **Additional Notes:** A full compartmentalization **System** with two faces: a userspace POSIX **isolation** substrate (Wasm cages — kin to the SFI line [[lfi]]/NaCl and the library-OS line [[graphene]]/[[osv]]/[[unikraft]]/[[kylinx]], and preserving fork like [[ufork]]) **and** a **policy-composition** framework (3i/grates) that the others lack. The paper explicitly positions against the matrix's MPK systems ([[erim]], [[smv]], [[cubicleos]], [[flexos]]) as alternative substrates, the partitioning systems ([[wedge]], [[arbiter]], [[salus]], [[breakapp]], [[rlbox]], [[natisand]]), the interposition-completeness work ([[k23]]), kernel-bypass ([[junction]], [[pegasus]]), and the isolation baselines ([[firecracker]], [[cloudflare-workers]]). Repo components: `cage`, `threei` (3i), `rawposix`, `fdtables`, modified `glibc`, `wasmtime`.

---

## Sources

- **Primary:** `papers/grates-final-with-appendix.pdf` — *Grates: Composable System Call Policies in Userspace* (anonymous submission, impl. anonymized as "GrateOS"). All **(§N)** / Table / Fig. citations reference this paper.
- **Project [repo]:** `github.com/Lind-Project/lind-wasm` (local `/developer/nyu/lind/lind-wasm`) — **Apache 2.0** (root `LICENSE`); README confirms components `fdtables`, `rawposix`, `threei` (= 3i), `typemap`, `cage` (`vmmap`/`signal`), `sysdefs`, `lind-boot`, modified `glibc`, `wasmtime`. Lind = "single-process sandbox … software fault isolation and a kernel microvisor."

### Cells flagged for further research
- **Security › TCB size (LOC)** — ⚠️ no single TCB-LOC figure (runtime + Wasmtime + modified glibc); per-grate LoC are policy code, not TCB.
- **Security › Side-channel resistance** — _Inferred_ out of scope (not discussed).
- **Efficiency › Inter-domain throughput (bw_pipe)** — file-read BW given (5,487 MB/s) instead; MB/domain is a 4 GB VA *reservation*, not resident memory.
- **Usability › Required expertise, Debugging** + **Sys Design › Monitoring, Config complexity, Maintenance burden** — N/A (self-report policy 2026-06-09).
- Inferences marked `_Inferred:_` inline: crash isolation, interaction semantics, deployment scale, scaling Big-O.
- **Naming:** paper "GrateOS"/"Grates" = repo **Lind / lind-wasm**; entry filed as `lind-wasm.md` (the user's project name).
