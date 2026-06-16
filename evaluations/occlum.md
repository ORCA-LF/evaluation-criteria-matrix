# Compartmentalization Evaluation Matrix — Occlum

**System:** Occlum (multi-process LibOS for Intel SGX; compartment = **SIP**, an SFI-Isolated Process)
**Paper:** Y. Shen, H. Tian, Y. Chen, K. Chen, R. Wang, Y. Xu, Y. Xia, S. Yan. *Occlum: Secure and Efficient Multitasking Inside a Single Enclave of Intel SGX.* ASPLOS 2020. arXiv:2001.07450v1.
**Source file:** `papers/6-occlum.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-08

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment, not a paper claim · **⚠️ Unclear** = not found, needs research.
>
> _Compartment = an **SIP** (SFI-Isolated Process): a process that shares the single address space of one SGX enclave with other SIPs, isolated by MMDSFI (MPX-based Multi-Domain SFI) and validated by an independent binary verifier._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Two levels:** **Secret protection** at the enclave boundary (SGX protects code/data from the untrusted OS/host) + **Untrusted component isolation** *inside* the enclave (isolating mutually-distrusting SIPs and protecting the LibOS from any SIP) — the paper's core contribution (§1, §3.1). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a process (SIP); each MMDSFI domain has a code region C and data region D (§3, §4.1). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process** — the SIP (§3). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is process-granular (each domain = one code region + one data region, but the unit is the SIP). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **TEEs (Intel SGX)** for the enclave boundary + **Intel MPX** (bound registers `bnd0`/`bnd1`) for SFI domain bounds. Note: MMDSFI uses only MPX *registers* (not MPX bound tables), so domain count is independent of register count (§2.3, §4). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **SFI** (MMDSFI — MPX-based Multi-Domain SFI) + **binary verification** (independent verifier) + coarse-grained **CFI**; the LibOS is written in **Rust** (memory-safe) (§4, §5, §8). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — all SIPs share the single address space of one enclave, partitioned into MMDSFI domains (§3, §4.1). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the Occlum LibOS (Rust), the Occlum LLVM-based toolchain (instruments code), the Occlum verifier, the Intel SGX SDK + Rust SGX SDK in-enclave runtime, and a modified musl libc (§6, §8). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Inter-process isolation** (an SIP protected from other SIPs) and **process-LibOS isolation** (LibOS protected from any SIP), via MMDSFI's memory-access policy + control-transfer policy (coarse CFI), guaranteed by the verifier (§3.1, §4–5). SGX provides enclave memory integrity + confidentiality vs untrusted OS/hardware (§2.1). Immune to code injection (data region non-executable; no RWX abuse); ROP made harder and gadget combinations covered by the verifier (§7). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | _Inferred:_ **✅** — an MMDSFI violation triggers a CPU exception (MPX bound-check fail / guard-region access) caught by the LibOS, confining the fault to the offending SIP (§4.1, §4.2). The paper does not present this as a crash-recovery experiment. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — an **independent binary verifier** statically checks every ELF for MMDSFI compliance in four stages before loading (disassembly, instruction-set, control-transfer, memory-access verification); the syscall trampoline checks return targets are valid `cfi_label`s; LibOS-entry assembly does sanity checks (§5, §6). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **CPU (SGX + MPX hardware)**, the **Occlum LibOS** (Rust), and the **Occlum verifier**. The **toolchain is deliberately excluded** from the TCB (the verifier replaces trust in it); the untrusted OS/host and the SIPs are outside the TCB (§5, §6). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Medium-small** — LibOS ≈ **15,000 LOC Rust** + verifier ≈ **2,000 LOC Python** in the TCB (the ≈3,000 LOC C++ toolchain is excluded via the verifier); plus the Intel/Rust SGX SDK runtime. Written from scratch → "small TCB / small attack surface" (§2.2, §8). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — "We do not consider side-channel attacks in this paper"; covert channels also out of scope. Acknowledged as a real SGX threat with orthogonal defenses (§3.1). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **Yes (partial)** — MMDSFI's correctness is **proved as theorems** in the paper: control-transfer compliance (Lemma 5.1, Theorem 5.2) and memory-access compliance (Theorem 5.3). These are pen-and-paper proofs of the SFI scheme, **not machine-checked** and not whole-system (§5). |
| Experimental validation available | Yes (specify) / No | **Yes** — app benchmarks (Fish, GCC, Lighttpd, §9.1), syscall microbenchmarks (process creation, pipe, file I/O, §9.2), MMDSFI overhead on SPECint2006 (§9.3), and the **RIPE** security benchmark showing prevention of code-injection + ROP attacks (§9.3). Artifact evaluated (BSD, Zenodo DOI 10.5281/zenodo.3565239). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **SPECint2006** (not 2017): MMDSFI averages **36.6%** overhead (CPU-intensive); range-analysis optimizations cut store-confinement 10.1%→4.3% and load-confinement 39.6%→25.5% (§9.3). vs Graphene-SGX: up to **500× faster (apps)** / **6,600× faster (micro)**; Lighttpd **9%** over Linux (Abstract, §9.1). Host: 2-core 3.5 GHz i7 (Kaby Lake), 32 GB, SGX 1.0, Ubuntu 16.04 / kernel 4.15 (§9). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Unknown (no lat_ctx).** SIPs map 1:1 to SGX threads scheduled by the host; syscalls are function calls into the shared LibOS via a trampoline (§6). No SIP context-switch latency isolated. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **SIP creation via `spawn`** (no new enclave): 14 KB binary **97 µs** (1.6× faster than Linux, 6,600× faster than Graphene-SGX's 0.64 s); median ≈ **1.7 ms** (10.5× slower than Linux, 390× faster than Graphene-SGX); large ≈ **63 ms** (13× faster than Graphene-SGX). Scales with binary size (§9.2). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Not lat_pipe.** IPC between SIPs is "simply copying data from one SIP to another, with no encryption" (cheap vs encrypted EIP IPC) (§3.2). No isolated µs figure. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | Pipe throughput **> 3× Graphene-SGX** and on par with Linux (§9.2, Fig. 6b). |
| Memory overhead per domain | MB/domain | **Unknown** — no MB/SIP figure. Each MMDSFI domain preallocates enclave pages (code region size + data region + two 4 KB guard regions), sized at compile time (§4.1, §6). |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: enclave EPC size + compile-time-prespecified max domain count** (SGX 1.0 cannot add enclave pages at runtime; relaxed on SGX 2.0). Crucially, domain count is **independent of MPX bound registers** (only 2 used) — unlike NaCl/PittSFIeld (§4.1, §2.3). |
| Performance scales with domain count | Big O | _Inferred:_ memory **O(n)** in domain count (pages preallocated per domain); no asymptotic statement in the paper (§4.1, §6). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Specialized** — Intel x86-64 CPU with **SGX + MPX** (SGX 1.0; i7 Kaby Lake recommended) (§9, Artifact A.3.2). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock + Intel SGX driver/SDK** — Ubuntu 16.04, kernel 4.15, untrusted host OS plus the Intel SGX driver/SDK (§9, Artifact A.3.3). |
| What privileges does it need? | User / Root / Kernel access | **User-level enclave**; root to install the SGX driver / set up the environment (Artifact A.2 "Root access to Ubuntu Linux"). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | Intel SGX SDK + Rust SGX SDK, the Occlum LLVM-7.0-based toolchain, and a modified musl libc (§8). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Recompile + small source changes** — recompile with the Occlum toolchain (source-level compatible) and replace `fork` with `spawn` (e.g. GCC ≈50 LOC, Lighttpd ≈150 LOC) (§3.3, §9.1). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Recompile (+ fork→spawn)** — not unmodified binaries; statically-linked ELFs only (shared-library support is future work) (§8, §6). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** (source-compatible via cross-compilers); Rust supported via the Rust SGX SDK (the LibOS itself is Rust) (§6, §8). |
| Other compatibility notes | [Describe] | `fork` unsupported → use `spawn`/`posix_spawn`; static linking only; no shared *memory* mappings (SGX cannot map one EPC page to multiple virtual addresses) (§3.3, §6). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Small** — chiefly the `fork`→`spawn` refactor (tens-to-hundreds of LOC: ≈50 GCC, ≈150 Lighttpd); "not as hard as one might expect." No person-days figure (§3.3, §9.1). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | Recompile with the Occlum LLVM toolchain (which inserts the MMDSFI instrumentation passes); link statically (§6, §8). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Failure modes visibility | [crashes/logs/error codes/silent] | MMDSFI violations → CPU exception (MPX bound-check fail / guard-region access) caught by the LibOS; the verifier **rejects** non-compliant binaries at load time (§4.1, §5). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — BSD** per the paper's artifact appendix (A.2 "Code licenses: BSD"; Zenodo DOI 10.5281/zenodo.3565239). _Note:_ the current repo `github.com/occlum/occlum` returns SPDX **NOASSERTION** (unclassified — license may have changed under later/Apache-incubation development). [repo] |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** prototype (§8) with strong industrial backing (Ant Financial Group co-authors) and an actively-maintained open-source project (§1, §8). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | _Inferred:_ **Partial** — built on the Intel/Rust SGX SDK; Iago-attack defense is delegated to orthogonal work (Sego) (§3.1). Broader composition not evaluated. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | _Inferred:_ **✅** — the enclave runs as a normal SGX user process alongside host processes; SIPs coexist within one enclave (§2.1, §3). |
| Can stack effectively | ✅ / ❌ | **✅** — MMDSFI supports an **unlimited number of domains** (no MPX-register constraint), so many SIPs coexist in one enclave (§3, §4). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Syscalls-as-function-calls** into the shared LibOS (via the trampoline), plus **IPC** between SIPs: signals, pipes, Unix domain sockets, implemented in the LibOS (§6). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | SIPs share the **single enclave address space** + shared LibOS data structures + a shared file-I/O cache and a **writable shared encrypted file system**; IPC is a direct SIP-to-SIP copy (no encryption) (§3.2, §6). |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Both** — synchronous syscalls/IPC plus asynchronous signals; not explicitly characterized in the paper (§6). |
| Interaction security/validation | [Describe] | SIP-to-SIP isolation is enforced by MMDSFI and **statically guaranteed by the verifier**; the LibOS mediates all syscalls and IPC (§4–6). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process** (SIP), intra-enclave (§3). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | _Inferred:_ **Programmer + Runtime** — the programmer structures the app into processes; the LibOS loads each verified binary into a domain (§4.1, §6). |
| Boundaries flexible at runtime | ✅ / ❌ | **Partial** — SIP *instances* are created dynamically and cheaply via `spawn`, but on **SGX 1.0** the enclave pages and the max domain count are **prespecified at compile time** (relaxed on SGX 2.0) (§4.1, §6, §9.2). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | LibOS threads (treated as SIPs sharing resources) **map 1:1 to SGX threads scheduled by the host**; `futex` synchronization relies on the host to sleep/wake but correctness lives in the LibOS (§6). |
| Process model | fork/exec / Custom / N/A | **Custom — `spawn`, not `fork`** (single address space makes `fork` incompatible); `posix_spawn` reimplemented over Occlum's spawn (§3.3, §6). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — musl libc; `spawn` instead of `fork`; static linking only; no shared memory mappings (§3.3, §6). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | No `fork`; static linking only; no shared memory mappings (SGX EPC restriction); SGX 1.0 page/domain preallocation fixes capacity at compile time (§3.3, §4.1, §6). |
| Security caveats when layered | [Describe] | Side channels out of scope (a real SGX threat); coarse CFI cannot fully prevent ROP (mitigated by verifier coverage + preserved isolation); network I/O insecure by default (needs TLS); Iago attacks not addressed (orthogonal) (§3.1, §6, §7). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud** — SGX-protected computation on untrusted public clouds; cloud-native multi-process apps with services like sshd/etcd/fluentd (§1, §2). |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Single device** (per-enclave on one host); not framed as a deployment-scale dimension in the paper. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |

---

## Summary

> **What it is:** A library OS for Intel SGX that enables secure *and* efficient multitasking **inside a single enclave** by running each process as an SFI-Isolated Process (SIP), isolated via a novel MPX-based Multi-Domain SFI (MMDSFI) scheme and guaranteed by an independent binary verifier — so processes share one enclave's address space (cheap `spawn`, cheap IPC, writable shared encrypted FS) instead of needing a separate enclave per process.
>
> **Who it's for:** Developers running multi-process legacy applications inside SGX enclaves on untrusted (cloud) hosts who need both strong isolation and good performance.
>
> **What it protects:** Enclave code/data from the untrusted OS/host (SGX), and — inside the enclave — each SIP from other SIPs and the LibOS from any SIP (MMDSFI + verifier).
>
> **What it costs (effort/money/performance):** ~36.6% CPU overhead from MMDSFI (SPECint2006), but up to 500× (apps) / 6,600× (micro) faster than per-process-enclave Graphene-SGX; `fork` must become `spawn`; static linking only.
>
> **What it needs (hardware/OS/expertise):** Intel SGX + MPX hardware, the Occlum toolchain/LibOS/verifier, recompilation, and small source changes (fork→spawn).
>
> **Key tradeoffs:** Single-enclave multitasking is dramatically faster than per-process-enclave LibOSes, the verifier keeps the large toolchain out of the TCB, and the SFI scheme is formally proven — but it requires SGX+MPX hardware, recompilation, spawn-not-fork, static linking only, and side channels remain explicitly out of scope.
>
> **Additional Notes:** Relies on Intel MPX (deprecated in later Intel CPUs/toolchains) — a portability concern for the SFI scheme going forward, though not discussed in the paper. The verifier design (excluding the toolchain from the TCB) is a notable, transferable idea.

---

## Sources

- **Primary:** `papers/6-occlum.pdf` — Shen et al., *Occlum...*, ASPLOS 2020, arXiv:2001.07450v1. All **(§N)** / Fig. / Theorem citations reference this paper.
- License: paper artifact appendix **A.2 (BSD)**, Zenodo DOI 10.5281/zenodo.3565239; artifact repo `github.com/occlum/reproduce-asplos20`. **[repo]** current project `github.com/occlum/occlum` → GitHub SPDX **NOASSERTION** (license unclassified; may differ from the 2020 artifact).

### Cells flagged for further research
- **Efficiency › Memory overhead per domain** — no MB/SIP figure.
- **Efficiency › Domain switch cost** — no lat_ctx figure.
- **Usability › Debugging support** — not discussed.
- **Sys Design › Monitoring/orchestration, Maintenance burden** — not discussed.
- **Availability › License** — BSD per 2020 artifact; current repo unclassified (verify current license).
- Rating/classification inferences (crash isolation, interaction semantics, deployment scale, config complexity, who-defines-boundaries, integrates/coexist, memory Big-O) marked `_Inferred:_` inline.
