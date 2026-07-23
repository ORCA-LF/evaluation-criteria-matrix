# Compartmentalization Evaluation Matrix — Unikraft

**Version:** v0.1
**System:** Unikraft (micro-library OS / unikernel)
**Paper:** S. Kuenzer, V.-A. Bădoiu, H. Lefeuvre, S. Santhanam, A. Jung, G. Gain, C. Soldani, C. Lupu, Ş. Teodorescu, C. Răducanu, C. Banu, L. Mathy, R. Deaconescu, C. Raiciu, F. Huici. Unikraft: Fast, Specialized Unikernels the Easy Way. EuroSys 2021. DOI 10.1145/3447786.3456248.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Other — single-application specialization unikernel.** Not internal isolation; isolation between workloads is the hypervisor's (separate VMs). Compartmentalization is an optional extension. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the unit is the whole application+OS image. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **VM** by default (one app per VM); micro-library granularity *if* compartmentalization extensions are used. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A.** |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None by default** — isolation is the hypervisor's. "Initial support for hardware compartmentalization" (Intel MPK, address-space isolation à la Iso-Unik) is noted as a direction, not the core mechanism. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **None (by design)** — a modular **micro-library unikernel** with a single protection level; no internal software isolation by default. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **VM**. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — runs on a hypervisor (QEMU/KVM, Xen, Firecracker, Solo5); the Unikraft kernel = composed micro-libraries (allocators, schedulers, net stack, boot, musl libc + syscall shim). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **VM-level isolation via the hypervisor only.** Inside the VM: single address space, single protection level, no user/kernel separation. Unikraft *does* support some standard hardening — "stack and page protection, ASLR, etc." — which past unikernels lacked. No intra-guest memory isolation between components by default. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **Cross-VM: ✅** (hypervisor). **Intra-VM: ❌** — from the single-protection-level design: no internal fault containment; an app fault can corrupt the kernel/VM. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **❌ / N/A** by default — no internal protection boundary (single protection level); "syscalls" via the shim are function calls, not validated cross-domain crossings. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Hypervisor + CPU/hardware + the entire Unikraft image** (app + kernel share one protection domain). Specialization yields a "small Trusted Computing Base" relative to general-purpose OSes. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small per-image** (specialized: minimal config image **200 KB**; typical app images **~1 MB**) — argued to be a "small TCB". **No total TCB LOC figure** given. System TCB also includes the hypervisor. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — not addressed (not a security-isolation paper). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No.** |
| Experimental validation available | Yes (specify) / No | **Yes (performance, not isolation)** — nginx, SQLite, Redis vs Linux/other unikernels; image size, boot time, memory, throughput. No isolation/attack evaluation (isolation is the hypervisor's). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC; Unikraft is faster than Linux.** **1.7×–2.7× performance improvement** vs Linux guests (nginx/SQLite/Redis). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **N/A internally** — single protection level, no protection-domain switches (the design explicitly removes them). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **"Domain" = VM.** Boot **~1 ms on top of VMM time** (total boot **2–40 ms**). No intra-VM domain creation (single app). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **N/A internally** (single app, single address space). Cross-VM communication via the network. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **N/A internally.** App-level throughput is high (millions of req/s for tuned nginx) but that is application, not inter-domain, throughput. |
| Memory overhead per domain | MB/domain | **< 10 MB RAM** to run typical app images (~1 MB image; min config 200 KB) — per *VM*, not per internal compartment. |
| Domain count bounded by | Limiting factor; approx. domain count | **One app per VM**; scaling = number of VMs (hypervisor/host-bound). |
| Performance scales with domain count | Big O | **N/A internally** (single app per VM). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — runs on QEMU/KVM, Xen, Firecracker, Solo5. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Custom guest OS** — Unikraft *is* the guest (replaces Linux); runs on stock hypervisors. |
| What privileges does it need? | User / Root / Kernel access | Runs as an **unprivileged guest VM**. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | A hypervisor; app built against Unikraft's micro-libraries (musl libc + syscall shim layer provided). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Refactoring/port, but eased** — apps are built with their native build system and linked into Unikraft; the **musl libc + syscall shim** layer reduces porting; many apps pre-ported (SQLite, nginx, Redis). Some porting effort still required. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Recompile / relink** against Unikraft (not unmodified Linux binaries by default). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++, Go, Python, Ruby, WebAssembly, Lua** + musl libc. |
| Other compatibility notes | [Describe] | Single application, single address space; POSIX via musl + syscall shim (partial). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Reduced vs classic unikernels** (native build + link, musl + shim), but **no person-days/LOC figure** for a typical port → **Unknown**. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Build effort | [build changes / time / deps] | App compiled with native build system, object files linked into Unikraft; micro-libraries selected via Kconfig-style config. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Failure modes visibility | [crashes/logs/error codes/silent] | with a single protection level, an app fault can corrupt the kernel/VM; the paper does not detail failure handling. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — BSD-3-Clause (main); per-library may differ**. Paper: "Linux Foundation open source project … www.unikraft.org" (Abstract); `COPYING.md` (`github.com/unikraft/unikraft`): "The main license of the project is the following BSD 3-clause license … each [library] might be individually licensed" (GitHub SPDX = NOASSERTION for that reason). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Open-source project** (Linux Foundation), positioned for production cloud/edge use; also an active research platform. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — explicitly designed to *host* compartmentalization extensions: micro-library compartmentalization "at reasonable cost" (paper's ref [69] = Sung et al., intra-unikernel MPK isolation, VEE'20 — not FlexOS) and "initial support for hardware compartmentalization" (MPK / Iso-Unik). Separately, FlexOS, µFork, and CubicleOS in this matrix are built on Unikraft. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — Unikraft VMs coexist with other VMs on a shared hypervisor. |
| Can stack effectively | ✅ / ❌ | **✅** for many Unikraft VMs side-by-side; internal composition is the modular micro-library model. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Cross-VM via the network**; internally, micro-libraries interact via **direct function calls** through composable APIs (no internal protection boundary). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Network (message passing)** across VMs; single shared address space internally. |
| Interaction semantics | Synchronous / Asynchronous / Both | **N/A internally**; network I/O async/sync as usual. |
| Interaction security/validation | [Describe] | **N/A internally** (no internal boundary); cross-VM isolation is the hypervisor's. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **VM** (isolation) / **micro-library** (modularity, not isolation by default). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | the **developer** selects/composes micro-libraries at build time (Kconfig-style); the operator deploys one app per VM. |
| Boundaries flexible at runtime | ✅ / ❌ | **❌** — composition is **build-time** (micro-library selection); one app per VM at runtime. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom / optional** — schedulers are pluggable micro-libraries; for RPC-style apps a single run-to-completion loop can replace threading entirely. |
| Process model | fork/exec / Custom / N/A | **N/A — single application** (single address space); multi-process / `fork` is not the default model (µFork adds `fork` on top of Unikraft separately). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — via musl libc + a syscall shim layer. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Single application, single address space by default; porting still required for some apps; compartmentalization is an add-on, not built-in. |
| Security caveats when layered | [Describe] | **No intra-VM isolation by default** — all security rests on the hypervisor; internal compartmentalization requires the extensions (FlexOS / MPK). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud / edge** — virtualized specialized unikernels (cloud workloads, NFV, FaaS lineage). |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — many small, fast-booting VMs across a cloud. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |

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
