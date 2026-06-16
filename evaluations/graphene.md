# Compartmentalization Evaluation Matrix — Graphene

**System:** Graphene (Library OS) — original Linux version (later renamed **Gramine**; SGX support added separately)
**Paper:** C.-C. Tsai, K. S. Arora, N. Bandi, B. Jain, W. Jannen, J. John, H. A. Kalodner, V. Kulkarni, D. Oliveira, D. E. Porter. *Cooperation and Security Isolation of Library OSes for Multi-Process Applications.* EuroSys 2014.
**Source file:** `papers/2-graphene.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-08

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend** (no value is fabricated):
> - **(§N)** = directly stated in the paper, cited by section/table/figure.
> - **[repo]** = sourced from the project's git repo / site (not the paper).
> - **_Inferred:_** = a reasonable assessment by the evaluator, NOT a paper claim.
> - **⚠️ Unclear** = not stated in the paper or sources found; needs further research.
>
> _Note: this is the 2014 Linux Graphene paper. Graphene-SGX / Gramine (enclave isolation) is a separate later system and is **not** evaluated here._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — isolating mutually-distrusting multi-process applications on a shared host, with "security isolation comparable to running the application in a separate VM" (Abstract, §3). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a process group (sandbox of picoprocesses) (§3). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process** — the unit is a *picoprocess* (OS process with a narrowed host ABI); a **sandbox** = a set of mutually-trusting picoprocesses (§2, §3). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is process/sandbox-granular; file access is mediated but not the isolation subject. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — pure software isolation on commodity hardware (§6 setup; no HW primitive used). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other** — a **library OS** (`libLinux.so`) over a narrow host ABI (the **PAL**, 43 functions / ~50 host syscalls), with host enforcement via **seccomp-BPF** syscall filtering + an **AppArmor LSM reference monitor** mediating file/socket access (§2, §3, §3.1). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Process** (picoprocess), grouped into **sandboxes**; positioned as a lighter-weight alternative to a VM (§2, §3). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — `libLinux.so` (libOS), the PAL, a reference-monitor daemon + AppArmor LSM extension, and a kernel IPC module (§2, §3, Fig. 1, Table 3). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | Restricts host attack surface to **15% of the Linux system-call table** (§6.6); reference monitor mediates file (`open`) and network (`socket`) access per a per-app **manifest** (§3); **blocks all cross-sandbox communication** — signals/IPC cannot cross sandbox boundaries (§3, §6.6). Within a sandbox, picoprocesses mutually trust (all-or-nothing isolation) (§3). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅ (cross-sandbox)** — sandboxes are strictly isolated, and Graphene is "designed to tolerate disconnection of collaborating libOS instances... because of crashes or blocked RPCs" (§4.3). _Caveats (§4.3):_ leader recovery is **not implemented** (design only); resilience to arbitrary memory corruption of a picoprocess is **future work**; within a sandbox a corrupted picoprocess can corrupt shared libOS state. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — the reference monitor validates the sandbox↔host boundary (manifest-checked file/socket access, seccomp syscall filtering, §3.1); but inter-picoprocess RPC *within* a sandbox assumes mutual trust (no validation), and cross-sandbox is all-or-nothing blocked rather than checked (§3, §4.3). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Host OS kernel (Linux)**, the **reference monitor** (unprivileged daemon + AppArmor LSM extension + kernel IPC module), the **CPU/hardware**. **Explicitly NOT trusted:** `libLinux.so` and the **PAL** — "the adversary can control all code inside one or more picoprocesses, including `libLinux` and the PAL" (§3 threat model). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large** — includes the full host Linux kernel + AppArmor. Graphene-added trusted code: reference monitor **3,568 LOC**, AppArmor LSM isolation extension **888 LOC** (16.63% changed), kernel IPC module **1,131 LOC** (≈ **5.6 KLoC** added) (Table 3). The untrusted libOS is large by comparison (`libLinux` 31,112 LOC, PAL 11,644 LOC) but outside the TCB. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **Mostly None** — only a specific `/proc`-based info-leak (Memento-style) is addressed: blocked because `/proc` is implemented inside `libLinux` and the system `/proc` is inaccessible (§6.6). No general cache/timing/speculative resistance is claimed. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal verification presented. |
| Experimental validation available | Yes (specify) / No | **Yes** — startup/checkpoint/migration (Table 4), memory footprint vs KVM (Fig. 4), app benchmarks (Apache, lighttpd, bash, gcc/make — Table 5), LMBench microbenchmarks (Table 6), System V IPC microbenchmarks (Table 7), 48-core scalability (Fig. 5), four isolation attack experiments all blocked (§6.6), and a manual study of **291 Linux CVEs (2011–2013)** finding Graphene would prevent **147 (51%)** (§6.6, Table 8). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** App-level vs native Linux — Table 5 columns are **Linux / KVM / Graphene+RM** (no Graphene-without-RM column for these apps): bzip2 **5%** (KVM also 5%), gcc **29%** (the 8% is the **KVM baseline**, not Graphene), libLinux compile 20–30%; "compilation overheads are primarily from the reference monitor." Apache 25-conc +43% vs lighttpd 25-conc +18%; overall range **5–30%** (§6.3, Table 5). Host: Dell Optiplex 790, 4-core 3.4 GHz i7, 4 GB RAM, Ubuntu 12.04 / kernel 3.5 (§6). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Not measured as lat_ctx.** Related LMBench (Table 6): `fork+exit` 67 µs (Linux) → **463 µs (Graphene; the paper states ~5.9× slower, §6.4)**, 490 µs +RM. (The separate `fork+exec` row is 231 → 764 µs.) Cross-picoprocess signal: first ~2 ms, **~55 µs cached** (§4.1). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Picoprocess startup ≈ 641 µs (3.1× native Linux's 208 µs)**; vs KVM 3.3 s (Table 4). Internally a `vfork`+`exec` of a clean instance (§5). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Not lat_pipe.** Coordination is RPC over host byte streams; System V `msgsnd` inter-process 153 µs (Linux) → **761 µs (Graphene, +397%)** (Table 7); cached cross-picoprocess signal ~55 µs (§4.3). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Not bw_pipe.** Graphene Host RPC throughput tracks Linux pipes and scales without bottleneck up to 48 cores (Fig. 5, §6.5). |
| Memory overhead per domain | MB/domain | "hello world": Linux 352 KB vs **Graphene 1.4 MB** (≈ 1 MB extra); incremental cost per additional child ≈ **790 KB** (§6.2). |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: memory footprint.** Within the ~150 MB of the smallest usable Linux VM, one could run **12–188 libOSes** (§6.2). No hard architectural cap stated. |
| Performance scales with domain count | Big O | RPC substrate scales like Linux pipes to 48 cores with no bottleneck (Fig. 5). _Inferred:_ memory linear / O(n) in picoprocess count (from §6.2 per-process footprint); paper does not give an asymptotic statement. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — stock x86 Linux machine (§6). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Modified** — host Linux needs an **AppArmor LSM extension** (888 LOC) plus a **kernel IPC module** (1,131 LOC); runs on Linux 2.6/3.x without recompiling the kernel otherwise (§5, Table 3). |
| What privileges does it need? | User / Root / Kernel access | **Root/kernel access to deploy** (install the kernel IPC module + LSM extension); the reference monitor itself runs as an **unprivileged daemon**, and applications run as **User** (§3, §5). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | Each application requires a **manifest** describing permitted file-system view and network addresses (§3). Modified glibc shipped with Graphene (606 LOC / 0.07% changed) (§5, Table 3). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None** — runs **unmodified** Linux application binaries and libraries (Fig. 1, §6 "unmodified"). A manifest is authored per app but the app is not changed. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — unmodified binaries, including statically-linked ones (seccomp redirects in-binary syscalls back to `libLinux`) (§3.1). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any** — Linux-ABI-compatible libOS; language-agnostic for unmodified Linux binaries (§6). |
| Other compatibility notes | [Describe] | Implements **131 of ~300** Linux syscalls; ~100 more deemed implementable, <10 administrative calls (e.g. kernel-module load, reboot) intentionally unsupported, ~54 need PAL extensions (§5). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **~None for the application** (unmodified). Effort is writing a manifest. (Producing Graphene itself: libLinux 31 KLoC, PAL 11.6 KLoC — Table 3, but that is one-time framework effort, not per-app.) |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | **None for the app** (no recompile). Deploying Graphene requires building `libLinux`/PAL and installing the kernel module + LSM extension (§5). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Failure modes visibility | [crashes/logs/error codes/silent] | Disallowed syscalls raise **SIGSYS** (redirected to `libLinux` or terminate the app); reference-monitor policy violations are **denied**; picoprocess disconnection/crash is tolerated by collaborating libOSes (§3.1, §4.3). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — LGPLv3** [repo] (`github.com/gramineproject/graphene` `LICENSE.txt`). Paper does not state a license. Project later renamed **Gramine** (`gramineproject.io`). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — academic prototype (§6, "research prototype"). _(Note: the successor Gramine is used in production for SGX, but that is out of scope for this 2014 paper.)_ |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **Unknown** — not addressed. The reference monitor builds on AppArmor LSM, but composition with other isolation frameworks is not discussed. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | _Inferred:_ **✅** — Graphene picoprocesses run as ordinary host processes alongside normal Linux processes under the reference monitor (Fig. 1); not explicitly framed as a coexistence claim. |
| Can stack effectively | ✅ / ❌ | **✅** for multiple sandboxes / picoprocesses coexisting (the core multi-sandbox model, §3, §6.6). Nesting Graphene-within-Graphene is not discussed. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **RPC / message passing** over host-provided byte streams (pipe-like); an IPC helper thread per picoprocess exchanges coordination messages within a sandbox (§4, §4.1). Cross-sandbox invocation is blocked. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Message passing** (RPC), plus **bulk IPC** (copy-on-write page transfer) used for `fork`. **Shared memory is not permitted** by the host ABI (§4.3, §5). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — RPCs are made asynchronous wherever possible (e.g. message-queue sends), while signals are delivered synchronously (§4.1, §4.3). |
| Interaction security/validation | [Describe] | Intra-sandbox picoprocesses **mutually trust** (no validation); inter-sandbox messages are **blocked entirely** by the reference monitor — "all-or-nothing" isolation (§3, §4.3). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process** (picoprocess) and **sandbox** (group of picoprocesses) (§3). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | Defined by the **manifest/policy** (administrator/user) loaded into the reference monitor at launch, plus runtime `sandbox_create` calls by the application (§3, §6.6). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — a child can be launched in a new sandbox (picoprocess-creation flag), and a process can dynamically detach from its sandbox, splitting it; the reference monitor then severs bridging byte streams (§3). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Works with standard threads** — `libLinux` implements Linux threading; multithreaded apps run (e.g. lighttpd, 4 threads) (§5, §6.3). |
| Process model | fork/exec / Custom / N/A | **Works with fork/exec** — implements Unix `fork` (copy-on-write via checkpoint + bulk IPC), `exec`, signals, System V IPC, `wait`/exit notification (§2.1, §4, §5). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — broad multi-process POSIX coverage (fork, signals, System V IPC, semaphores) but **131/~300** Linux calls implemented (§5). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | No shared memory across picoprocesses (host ABI restriction); shared file seek cursors not implemented; idiosyncratic/admin syscalls unimplemented; RPC coordination adds overhead vs native shared-memory IPC (§4.3, §5). |
| Security caveats when layered | [Describe] | **All-or-nothing** isolation — controlled cross-sandbox communication is future work; within a sandbox a corrupted picoprocess can corrupt shared libOS state; leader recovery and resilience to arbitrary picoprocess memory corruption are future work (§3, §4.3). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | _Inferred:_ **Server / Cloud** — multi-process server workloads (Apache, lighttpd) and the "library OSes for the cloud" lineage (§1, §6.3); the paper does not name a single target explicitly. |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Single device** (single host; tested up to a 48-core machine). Cross-host **migration** of a picoprocess is supported via checkpoint/restore (§6.1). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |

---

## Summary

> **What it is:** A Linux-compatible **library OS** (`libLinux.so`) that runs unmodified multi-process Linux applications inside groups of **picoprocesses** (sandboxes) over a narrow host ABI (the PAL), cooperatively implementing shared POSIX abstractions (fork, signals, System V IPC) via RPC over host byte streams, with VM-comparable security isolation between mutually-distrusting sandboxes enforced by a reference monitor (seccomp-BPF + AppArmor LSM).
>
> **Who it's for:** Running mutually-distrusting / untrusted multi-process Linux applications (servers, shell scripts) with VM-like isolation but far lower memory footprint and faster startup.
>
> **What it protects:** The host kernel and other sandboxes from a compromised application — host syscall attack surface cut to 15% of the table, file/network access mediated by a manifest, and all cross-sandbox communication blocked.
>
> **What it costs (effort/money/performance):** 5–30% on compilation, up to ~2–6× on some IPC/fork-heavy microbenchmarks; ≈0.8–1.4 MB per picoprocess (3–20× *less* memory than KVM); requires a host kernel module + LSM extension and a per-app manifest.
>
> **What it needs (hardware/OS/expertise):** Commodity hardware; a modified Linux host (AppArmor LSM extension + IPC kernel module, root to install); a manifest per application; no application changes.
>
> **Key tradeoffs:** VM-comparable isolation at order-of-magnitude lower memory footprint and faster startup than KVM while running unmodified multi-process binaries — but isolation is **all-or-nothing** (no controlled cross-sandbox sharing yet), within-sandbox picoprocesses **mutually trust** each other, the host kernel must be modified, and the TCB still includes the full host kernel.
>
> **Additional Notes:** This is the 2014 Linux Graphene. The project was later renamed **Gramine** and gained Intel SGX support (Graphene-SGX) — a different threat model (enclave isolation) not covered by this paper.

---

## Sources

- **Primary:** `papers/2-graphene.pdf` — Tsai et al., *Cooperation and Security Isolation of Library OSes for Multi-Process Applications*, EuroSys 2014. All inline **(§N)** / Table / Figure citations reference this paper.
- **[repo]** `github.com/gramineproject/graphene` (formerly `oscarlab/graphene`) — `LICENSE.txt` confirms **LGPLv3**. Project renamed **Gramine**: `gramineproject.io`.

### Cells flagged for further research
- **Debugging support** — not discussed in paper; only inferred from "runs as ordinary Linux binary."
- **Integrates with other compartmentalization mechanisms** — not addressed.
- **Monitoring/orchestration support** — not addressed.
- **Ongoing maintenance burden** — not discussed; inferred only.
- Several rating-style cells (target environment, deployment scale, config complexity, required expertise, CPU/memory Big-O, coexistence) are evaluator **inferences**, marked inline.
