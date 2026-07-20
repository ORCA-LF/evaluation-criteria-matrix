# Compartmentalization Evaluation Matrix — FlexOS

**Version:** v0.1
**System:** FlexOS (flexible-isolation LibOS; compartment = a configurable compartment of Unikraft libraries)
**Paper:** Lefeuvre et al. FlexOS: Towards Flexible OS Isolation. ASPLOS 2022. DOI 10.1145/3503222.3507759.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation + Secret protection (flexible)** — isolate any subset of OS/app components (network stack, parsers, app) considered untrusted; there is no single trust model for FlexOS. Per the author it can also do secret protection, but is **not** aimed at fault isolation (in the MPK backend a fault in one compartment usually halts the system). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — each library is present only once and maps to a specific set of privileges. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Library (configurable)** — the Unikraft micro-library is the minimal granularity; compartments are configurable groups of libraries (merge/split at build time). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Byte** — data shared at the granularity of a byte via `__shared` annotations (hardware enforcement is page-granular for MPK). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **Configurable: MPK (Intel) + EPT/VM + CHERI** backends implemented — CHERI in follow-up work ([Lefeuvre et al., workshop 2023](https://dl.acm.org/doi/abs/10.1145/3623759.3624550)), per the author, **not** merely sketched; SGX/TrustZone fit the model. Selected per build. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Boundary wrappers / marshalling** — inlined call gates for MPK, plus marshalling in the VM/EPT backend (per the author) + **configurable hardening**: CFI, KASan, UBSan, stack protector (per-component). SFI and memory-safe languages (Rust) are listed as further hardening options. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Other** — per the author, FlexOS has its own compartment abstraction that matches none of the fixed options; under the hood it is enforced by MPK domains, VMs, or other mechanisms (Intra-Address-Space Domain for MPK, VM for EPT). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — a runtime: the FlexOS OS itself (the isolation backend, core libraries exposing hooks, and the Unikraft base). Per the author, the build-time toolchain (Coccinelle, Cscope) is **not** part of the runtime — it is a deployment/build requirement. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Configurable spatial memory isolation** between compartments (via the chosen mechanism); **data ownership** (private by default; `__shared` whitelists per-library, ACL-style); **register-set isolation** at gates (save + zero registers); **incomplete CFI** (gates enforce legal entry points, which is essential for isolation; but per the author MPK cannot enforce execute permissions, so another compartment's code *can* be executed without faulting as long as it does not touch that compartment's private data — the EPT backend enforces execute permissions on pages and is therefore stronger); **per-component software hardening**. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **❌** — per the author, FlexOS does not target resilience; in the MPK backend a fault in one compartment usually halts the whole system. No built-in transparent recovery, though "start a safer configuration of the same software" on crash is a discussed use case one could build around FlexOS. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — call gates enforce entry points (incomplete CFI) + register isolation; the EPT RPC server validates legal entry points. But interfaces are assumed to correctly check arguments and be free of confused-deputy/Iago situations — argument validity is **assumed** (made probabilistically harder by variable interface size, not eliminated). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Core libraries** (early boot code, memory manager, scheduler, first-level interrupt-handler context switch), the **isolation backend**, the **hardware**, and the **compiler**. The rest of the toolchain (Coccinelle) is **NOT** in the TCB (compile-time checks detect invalid transformations); isolated app/kernel libraries are untrusted. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small** — around 3000 LoC in the case of Intel MPK, and even less for VM/EPT. Per the author, a **toy** implementation of the scheduler was formally verified in Dafny — **not** the performance-focused scheduler used in the evaluation or in production Unikraft. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None (active)** — FlexOS is not immune to hardware vulnerabilities by design, but its flexibility lets you **rebuild with a different mechanism** if one breaks (Meltdown/KPTI response use case). No active side-channel mitigation. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **Yes (partial)** — a **toy** implementation of the scheduler is formally verified in **Dafny** (not the one used in the evaluation or in production, per the author); the framework supports **incremental verification** (verify + isolate individual components, preserving proven properties when mixed with unverified ones). |
| Experimental validation available | Yes (specify) / No | **Yes** — 2×80 configs for Redis + Nginx, SQLite vs Linux/SeL4-Genode/CubicleOS/Unikraft, iPerf network throughput, gate-latency + DSS microbenchmarks, and the partial-safety-ordering exploration. Artifact public (BSD-3-clause, Zenodo DOI 10.5281/zenodo.5748505). Host: Intel Xeon Silver 4114. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC; configurable.** Redis 0–4.1× across 80 configs (292K → 1.2M req/s); Nginx similar; SQLite MPK3 **2×**, EPT2 ≈ Linux; **FlexOS NONE ≈ Unikraft baseline** ("only pay for what you get"). Cross-system numbers (vs SeL4 ≈3.1× / CubicleOS ≈order of magnitude) are the paper's reference points only — per the author they are **not apples-to-apples** (these systems have different security properties) and deployers should run their own comparison. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Gate latencies (cycles):** function call ≈ **2**; **MPK-light ≈ 62** (raw `wrpkru`, 80% faster than a normal MPK gate); **MPK-dss ≈ 108**; **EPT ≈ 462** — close to a Linux syscall *with* KPTI (**470**), so EPT2 configs perform almost identically to Linux; a Linux syscall *without* KPTI is **146**. Per the author, these gate costs are very hardware-dependent. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **N/A at runtime** — compartments are instantiated at **build time** (no code is loaded after compilation); EPT generates one VM image per compartment. Reconfiguration = a rebuild (seconds to create a new binary). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | Same as gate latency: **62 cyc (MPK-light) → 462 cyc (EPT)** per cross-compartment call. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | Shared-memory based (MPK shared heap + DSS; EPT shared-memory RPC). Batching: MPK reaches ≈ baseline from 128 B payloads, EPT from 256 B. No direct bytes/sec sharing-throughput figure was measured, per the author. |
| Memory overhead per domain | MB/domain | **MPK:** DSS doubles stack size (small — 8-page stacks; a Redis instance with 8 threads = **288 KB** DSS overhead) + per-compartment data/rodata/bss + private heap. **EPT:** a **full VM image per compartment** (TCB duplicated). |
| Domain count bounded by | Limiting factor; approx. domain count | **MPK: ≤15 compartments** (16 keys, ≥1 reserved for the shared domain); **EPT:** bounded by VM/vCPU/core count. |
| Performance scales with domain count | Big O | Inlined build-time gates → no runtime abstraction overhead; overall cost depends on **which** components are isolated and their **communication patterns** (cut placement), not just compartment count. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Depends on backend** — MPK needs Intel **Skylake+** (Xeon Scalable); EPT needs commodity **VT-x** (widely available); CHERI needs **ARM Morello** (implemented in follow-up work, per the author). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Custom LibOS** (Unikraft-based), running in a **QEMU/KVM VM**; the EPT backend needs a small (~90 LoC) KVM patch for inter-VM shared memory. |
| What privileges does it need? | User / Root / Kernel access | Runs as a **VM guest** (unikernel); MPK isolation is user-level within the guest. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | The FlexOS toolchain (Coccinelle; **Cscope is only a helper**, not a requirement, per the author), per-backend runtime, and a declarative config file. The **Wayfinder** platform is likewise **not required** — just useful for evaluating many configurations. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Annotations** — call-gate insertion is **automated**; **shared-data marking (`__shared`) is manual**. Function pointers across components need manual annotation. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Source changes needed** — annotate shared data and rebuild via the toolchain; static linking (no code loaded after compile). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C** (Unikraft C codebase; Coccinelle C transforms). Memory-safe languages (Rust) are mentioned as a *possible* hardening direction, not implemented. |
| Other compatibility notes | [Describe] | Extends a **POSIX**-interface OS (Unikraft); deeply-entangled components (e.g. ramfs/vfscore) resist clean splitting without redesign. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **10 minutes (time subsystem, no shared data) to 2–5 days (filesystem, network stack)** per component; similar to other compartmentalization frameworks. Patch sizes + shared-var counts in Table 1. |
| Required expertise level | Low / Medium / High / Expert | **High (author-provided)** — per the author, FlexOS is a custom OS that must be configured, software must be ported, and interfaces manually hardened. |
| Build effort | [build changes / time / deps] | Toolchain transforms source (Coccinelle) and builds the image; a single image builds in about a minute (per the author). Large multi-config sweeps take longer, but the "6–12 h / 160 images" artifact figure likely bundled other work and should not be read as per-image cost. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Standard** — GDB and all usual debugging toolchains are supported, not significantly different from Unikraft; source-to-source transforms are inspectable with file-comparison tools. |
| Failure modes visibility | [crashes/logs/error codes/silent] | A memory-access violation **crashes** with a report pointing to the offending symbol (used to guide `__shared` annotation); a compromised compartment that ROPs elsewhere crashes on accessing the target's private data. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — BSD-3-clause**; `github.com/project-flexos`; Zenodo DOI 10.5281/zenodo.5748505. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — proof-of-concept prototype. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅ (the central idea)** — designed to swap/abstract isolation mechanisms (MPK/EPT/CHERI/SGX) behind one API; adding a backend should not involve any rewrite. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **Partial** — a single image uses **one isolation mechanism per build** (but per-component hardening is mixable); runs as a VM guest alongside others. Heterogeneous-hardware deployment (pick best mechanism per machine) is a use case. |
| Can stack effectively | ✅ / ❌ | **✅** — compartments merge/split freely, and software hardening stacks (e.g. CFI + ASan per component). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Configurable** — **inlined call gates** (MPK, System V ABI) **or RPC** (EPT, shared-memory busy-wait). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** — MPK shared heap + **Data Shadow Stacks (DSS)** for stack data; EPT shared regions mapped at identical addresses; `__shared` byte-granular annotations. |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** — function-call gates (MPK) and busy-wait RPC (EPT). |
| Interaction security/validation | [Describe] | Data ownership (private-by-default + whitelisted `__shared`), entry-point CFI + register isolation. Per the author, argument validity itself is **not** assumed; rather, compartment **interfaces are assumed to have been hardened**. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Library / compartment** (configurable grouping of Unikraft libraries). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** — a declarative config file (which libraries → which compartment, mechanism, hardening); the build-time toolchain instantiates it. |
| Boundaries flexible at runtime | ✅ / ❌ | **❌ at runtime** — the entire premise is **build-time** (not design-time *and* not runtime) configuration; no code is loaded after compilation. Reconfiguration = a cheap rebuild. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Unikraft scheduler** — per-thread-per-compartment stacks via a stack registry (MPK); the EPT RPC server keeps a thread pool for multithreaded loads. |
| Process model | fork/exec / Custom / N/A | **N/A — single-application unikernel** (Unikraft); no multi-process model. |
| POSIX compatibility | Full / Partial / Limited | **Partial** — extends Unikraft's POSIX interface. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | MPK ≤15 compartments; EPT cost higher + TCB **duplicated** per compartment; function pointers need manual annotation; deeply-entangled components resist clean splitting; **one isolation mechanism per image build**. |
| Security caveats when layered | [Describe] | Compartment interfaces are **assumed to be hardened** against argument-validity / confused-deputy / Iago issues (made harder by variable interface size, not eliminated); MPK can't restrict `wrpkru` execution → relies on static W⊕X analysis (safe because no code loaded post-compile); MPK CFI is incomplete; side channels not actively addressed. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud** — cloud applications (Redis, Nginx, SQLite); LibOS/unikernel in VMs. |
| Deployment scale | Single device / Cluster / Large scale | **Single device** (per-VM unikernel); use cases mention load-balancer triage and blue-green deployments at fleet level. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | Per the author, **Unikraft has monitoring/orchestration support**, but this was **not explored with FlexOS**. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **High (author-provided)** — per the author, adopting FlexOS is quite disruptive compared to e.g. Linux. |
| Ongoing maintenance burden | Low / Medium / High | **Medium (author-provided)** — per the author, new versions of a given piece of software still need to be adapted. |

---

## Summary

> **What it is:** A LibOS (built on Unikraft) whose **isolation strategy is chosen at build/deployment time, not design time** — compartment granularity, hardware isolation mechanism (MPK / EPT / CHERI), data-sharing strategy, and per-component software hardening are all configurable via abstract call gates + `__shared` annotations that the toolchain instantiates through source-to-source transformation. Ships with a "partial safety ordering" technique to navigate the huge configuration space.
>
> **Who it's for:** OS builders/operators who want to specialize the safety/performance trade-off per application or use-case — or react to a newly-broken hardware mechanism — without redesign.
>
> **What it protects:** Configurable — any subset of OS/app components (network stack, parsers, app) can be isolated as untrusted compartments, with private-by-default data and whitelisted sharing.
>
> **What it costs (effort/money/performance):** Configurable, 0–4.1× (Redis/Nginx); gate latency ~62 cyc (MPK-light) to ~462 cyc (EPT); porting = source annotations (10 min–5 days per component).
>
> **What it needs (hardware/OS/expertise):** Intel MPK (Skylake+) or VT-x (EPT) — or CHERI (ARM Morello); a Unikraft-based build with the FlexOS toolchain (Coccinelle); runs in a VM.
>
> **Key tradeoffs:** Unmatched flexibility — swap isolation mechanism/granularity/hardening at build time with near-zero runtime abstraction cost (inlined gates) plus a tool to navigate the space — but it's a single-app unikernel (no multi-process), boundaries are fixed per build (not runtime), only one isolation mechanism per image, MPK caps at 15 compartments, and argument-validity / confused-deputy is assumed rather than enforced.
>
> **Additional Notes:** Outperforms CubicleOS by an order of magnitude on SQLite (CubicleOS runs on Unikraft's Ring-3 `linuxu` debug platform using `pkey_mprotect` syscalls; FlexOS runs in a real VM with inlined gates). The "configurable at build time" stance is the key differentiator from every fixed-mechanism system in this set.
