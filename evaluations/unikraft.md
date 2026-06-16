# Compartmentalization Evaluation Matrix — Unikraft

**System:** Unikraft (micro-library OS / unikernel)
**Entry type:** ⬛ **BASELINE** — single-application unikernel with a single protection level (no intra-guest isolation by default); isolation is the hypervisor's (VM-granularity). Included as a contrast/substrate point; many isolation rows are N/A by design. _Notably the foundation that FlexOS, µFork, and CubicleOS (all in this matrix) build compartmentalization on._
**Paper:** S. Kuenzer, V.-A. Bădoiu, H. Lefeuvre, S. Santhanam, A. Jung, G. Gain, C. Soldani, C. Lupu, Ş. Teodorescu, C. Răducanu, C. Banu, L. Mathy, R. Deaconescu, C. Raiciu, F. Huici. *Unikraft: Fast, Specialized Unikernels the Easy Way.* EuroSys 2021. DOI 10.1145/3447786.3456248.
**Source file:** `papers/unikraft.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-10

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment, not a paper claim · **N/A** = not applicable / self-report policy · **Unknown** = genuinely unreported.
>
> **⚠️ Framing note:** Unikraft is **not** an internal-compartmentalization system. It is a *single-application* micro-library unikernel built for **specialization/performance**, with a **single address space** and **single protection level** (no user/kernel separation) by design (§2 Design Principles). Its isolation boundary is the **VM** (hypervisor). It explicitly *enables* compartmentalization as an extension — "this does not preclude compartmentalization (e.g., of micro-libraries), which can be achieved at reasonable cost [69]" (the paper's ref [69] is **Sung et al., "Intra-unikernel isolation with Intel Memory Protection Keys," VEE'20** — *not* FlexOS) — and notes "initial support for hardware compartmentalization … address-space isolation, e.g., Iso-Unik with Intel MPK" (§2, §Related Work). Read here as the **baseline/substrate** counterpart to OSv.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Other — single-application specialization unikernel.** Not internal isolation; isolation between workloads is the hypervisor's (separate VMs). Compartmentalization is an optional extension (§2, §Related Work). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the unit is the whole application+OS image (§2). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **VM** by default (one app per VM); micro-library granularity *if* compartmentalization extensions are used (§2). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A.** |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None by default** — isolation is the hypervisor's. "Initial support for hardware compartmentalization" (Intel MPK, address-space isolation à la Iso-Unik) is noted as a direction, not the core mechanism (§Related Work/Conclusion). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **None (by design)** — a modular **micro-library unikernel** with a single protection level; no internal software isolation by default (§2). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **VM** (§2). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — runs on a hypervisor (QEMU/KVM, Xen, Firecracker, Solo5); the Unikraft kernel = composed micro-libraries (allocators, schedulers, net stack, boot, musl libc + syscall shim) (§1, §3). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **VM-level isolation via the hypervisor only.** Inside the VM: single address space, single protection level, no user/kernel separation (§2). Unikraft *does* support some standard hardening — "stack and page protection, ASLR, etc." — which past unikernels lacked (§Related Work). No intra-guest memory isolation between components by default. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **Cross-VM: ✅** (hypervisor). **Intra-VM: ❌** — _Inferred from the single-protection-level design (§2):_ no internal fault containment; an app fault can corrupt the kernel/VM. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **❌ / N/A** by default — no internal protection boundary (single protection level); "syscalls" via the shim are function calls, not validated cross-domain crossings (§2, §3). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Hypervisor + CPU/hardware + the entire Unikraft image** (app + kernel share one protection domain). Specialization yields a "small Trusted Computing Base" relative to general-purpose OSes (§Related Work). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small per-image** (specialized: minimal config image **200 KB**; typical app images **~1 MB**) — argued to be a "small TCB" (§Related Work, §Eval). **No total TCB LOC figure** given. System TCB also includes the hypervisor. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — not addressed (not a security-isolation paper). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No.** |
| Experimental validation available | Yes (specify) / No | **Yes (performance, not isolation)** — nginx, SQLite, Redis vs Linux/other unikernels; image size, boot time, memory, throughput (§Eval). No isolation/attack evaluation (isolation is the hypervisor's). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC; Unikraft is faster than Linux.** **1.7×–2.7× performance improvement** vs Linux guests (nginx/SQLite/Redis) (Abstract, §Eval). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **N/A internally** — single protection level, no protection-domain switches (the design explicitly removes them) (§2). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **"Domain" = VM.** Boot **~1 ms on top of VMM time** (total boot **2–40 ms**) (Abstract, §Eval). No intra-VM domain creation (single app). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **N/A internally** (single app, single address space). Cross-VM communication via the network (§2). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **N/A internally.** App-level throughput is high (millions of req/s for tuned nginx) but that is application, not inter-domain, throughput (§Eval). |
| Memory overhead per domain | MB/domain | **< 10 MB RAM** to run typical app images (~1 MB image; min config 200 KB) — per *VM*, not per internal compartment (Abstract, §Eval). |
| Domain count bounded by | Limiting factor; approx. domain count | **One app per VM**; scaling = number of VMs (hypervisor/host-bound) (§2). |
| Performance scales with domain count | Big O | **N/A internally** (single app per VM). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — runs on QEMU/KVM, Xen, Firecracker, Solo5 (§1). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Custom guest OS** — Unikraft *is* the guest (replaces Linux); runs on stock hypervisors (§1). |
| What privileges does it need? | User / Root / Kernel access | Runs as an **unprivileged guest VM** (§2). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | A hypervisor; app built against Unikraft's micro-libraries (musl libc + syscall shim layer provided) (§1). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Refactoring/port, but eased** — apps are built with their native build system and linked into Unikraft; the **musl libc + syscall shim** layer reduces porting; many apps pre-ported (SQLite, nginx, Redis). Some porting effort still required (§1, §3). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Recompile / relink** against Unikraft (not unmodified Linux binaries by default) (§1). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++, Go, Python, Ruby, WebAssembly, Lua** + musl libc (§1). |
| Other compatibility notes | [Describe] | Single application, single address space; POSIX via musl + syscall shim (partial) (§1, §2). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Reduced vs classic unikernels** (native build + link, musl + shim), but **no person-days/LOC figure** for a typical port → **Unknown** (§1, §3). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | App compiled with native build system, object files linked into Unikraft; micro-libraries selected via Kconfig-style config (§1, §3). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Failure modes visibility | [crashes/logs/error codes/silent] | _Inferred:_ with a single protection level, an app fault can corrupt the kernel/VM; the paper does not detail failure handling (§2). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — BSD-3-Clause (main); per-library may differ** [repo]. Paper: "Linux Foundation open source project … www.unikraft.org" (Abstract); `COPYING.md` (`github.com/unikraft/unikraft`): "The main license of the project is the following BSD 3-clause license … each [library] might be individually licensed" (GitHub SPDX = NOASSERTION for that reason). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Open-source project** (Linux Foundation), positioned for production cloud/edge use; also an active research platform (Abstract, §1). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — explicitly designed to *host* compartmentalization extensions: micro-library compartmentalization "at reasonable cost" (paper's ref [69] = Sung et al., intra-unikernel MPK isolation, VEE'20 — not FlexOS) and "initial support for hardware compartmentalization" (MPK / Iso-Unik). Separately, FlexOS, µFork, and CubicleOS in this matrix are built on Unikraft (§2, §Related Work). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | _Inferred:_ **✅** — Unikraft VMs coexist with other VMs on a shared hypervisor (§1, §2). |
| Can stack effectively | ✅ / ❌ | _Inferred:_ **✅** for many Unikraft VMs side-by-side; internal composition is the modular micro-library model (§1). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Cross-VM via the network**; internally, micro-libraries interact via **direct function calls** through composable APIs (no internal protection boundary) (§2, §3). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Network (message passing)** across VMs; single shared address space internally (§2). |
| Interaction semantics | Synchronous / Asynchronous / Both | **N/A internally**; network I/O async/sync as usual. |
| Interaction security/validation | [Describe] | **N/A internally** (no internal boundary); cross-VM isolation is the hypervisor's (§2). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **VM** (isolation) / **micro-library** (modularity, not isolation by default) (§2). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | _Inferred:_ the **developer** selects/composes micro-libraries at build time (Kconfig-style); the operator deploys one app per VM (§1, §3). |
| Boundaries flexible at runtime | ✅ / ❌ | **❌** — composition is **build-time** (micro-library selection); one app per VM at runtime (§1, §3). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom / optional** — schedulers are pluggable micro-libraries; for RPC-style apps a single run-to-completion loop can replace threading entirely (§2). |
| Process model | fork/exec / Custom / N/A | **N/A — single application** (single address space); multi-process / `fork` is not the default model (µFork adds `fork` on top of Unikraft separately) (§2). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — via musl libc + a syscall shim layer (§1). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Single application, single address space by default; porting still required for some apps; compartmentalization is an add-on, not built-in (§1, §2). |
| Security caveats when layered | [Describe] | **No intra-VM isolation by default** — all security rests on the hypervisor; internal compartmentalization requires the extensions (FlexOS / MPK) (§2). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud / edge** — virtualized specialized unikernels (cloud workloads, NFV, FaaS lineage) (§1). |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Large scale** — many small, fast-booting VMs across a cloud (§1). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |

---

## Summary

> **What it is:** A modular **micro-library unikernel** that makes it easy to build a specialized, single-application OS image by composing OS primitives (allocators, schedulers, net stacks, boot code) as swappable micro-libraries behind performance-minded APIs — yielding tiny (~1 MB), fast-booting (~ms), high-performance VM images.
>
> **Who it's for:** Developers wanting unikernel-grade performance/specialization (cloud, edge, NFV, FaaS) without the usual per-application re-engineering.
>
> **What it protects:** Nothing *internally* by default — single address space, single protection level; isolation between workloads is the hypervisor's (separate VMs). It *enables* internal compartmentalization as an extension (FlexOS, MPK/Iso-Unik).
>
> **What it costs (effort/money/performance):** A performance *win* (1.7–2.7× over Linux, ~1 MB images, <10 MB RAM, ms boot); the cost is porting/relinking to Unikraft (eased by musl + syscall shim) and no internal protection by default.
>
> **What it needs (hardware/OS/expertise):** A hypervisor (QEMU/KVM, Xen, Firecracker, Solo5) and an app built/linked against Unikraft's micro-libraries.
>
> **Key tradeoffs:** Modular specialization gives small, fast, high-performance images and a small TCB — but it's single-application with VM-only isolation and no intra-guest protection unless compartmentalization extensions are added.
>
> **Additional Notes:** For the ORCA matrix, Unikraft is best read as a **baseline/substrate**: the high-performance unikernel foundation on top of which several *actual* compartmentalization systems in this set (FlexOS, µFork, CubicleOS) build their isolation.

---

## Sources

- **Primary:** `papers/unikraft.pdf` (read via `pdftotext`) — Kuenzer et al., *Unikraft: Fast, Specialized Unikernels the Easy Way*, EuroSys 2021, DOI 10.1145/3447786.3456248. Citations reference this paper by section/topic.
- **[repo]** `github.com/unikraft/unikraft` — `COPYING.md`: main license **BSD-3-Clause**, individual micro-libraries may be separately licensed (GitHub SPDX = NOASSERTION). Project: `www.unikraft.org` (Linux Foundation).

### Cells flagged for further research
- **Usability › Porting effort** — eased vs classic unikernels but no person-days/LOC figure → **Unknown**.
- **Security › TCB size** — "small TCB"/image sizes given, but no total LOC.
- **(structural)** — BASELINE: single-protection-level unikernel; many isolation rows N/A by design (substrate for FlexOS/µFork/CubicleOS).
- Self-report dims (debugging, expertise, config, maintenance, monitoring) → **N/A** by policy 2026-06-09.
- Rating/classification inferences (intra-VM crash isolation ❌, deployment scale, who-defines-boundaries, coexist) marked `_Inferred:_` inline.
