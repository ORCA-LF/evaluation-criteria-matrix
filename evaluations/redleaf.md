# Compartmentalization Evaluation Matrix — RedLeaf

**Version:** v0.1
**System:** RedLeaf (safe-language OS; compartment = a language-based domain)
**Paper:** V. Narayanan, T. Huang, D. Detweiler, D. Appel, Z. Li, G. Zellweger, A. Burtsev. RedLeaf: Isolation and Communication in a Safe Operating System. USENIX OSDI 2020.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Fault isolation** (primary) + **information hiding** — a domain is "a unit of information hiding and fault isolation"; also isolates mutually-distrusting kernel subsystems/extensions. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a domain (driver/subsystem/app). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Library / module** — a domain is an independently-compiled subsystem/driver/app loaded dynamically (no separate address space). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** for the isolation subject, but note **object-level ownership tracking** on a shared heap (`RRef<T>`). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — "does not rely on hardware mechanisms for isolation and instead uses only type and memory safety of the Rust language"; all domains run in **ring 0**, single address space. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Memory-safe language (Rust)** — ownership/type safety — plus **boundary wrappers/marshalling** (IDL-generated invocation proxies) and an ownership discipline (heap isolation, exchangeable types). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — language-based domains in one address space, ring 0. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — a microkernel (domain loading, scheduling, memory, interrupt dispatch), the IDL compiler, trusted crates, and a trusted compilation environment + Rust toolchain. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Heap isolation** (domains never hold pointers into another domain's private heap), **fault isolation** (a crashing domain is cleanly terminated, threads unwound, resources reclaimed without affecting others), **zero-copy cross-domain communication** with ownership tracking, **exchangeable-type** enforcement, **interface validation**, and **transparent device-driver recovery** (shadow drivers). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** (core contribution) — domains are dynamically loaded and cleanly terminated; "errors in one domain do not affect the execution of other domains." On crash, all threads are unwound, resources deallocated, and subsequent cross-domain calls return an `RpcResult` **error instead of panicking**; device drivers recover transparently. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the IDL compiler statically **validates all cross-domain interfaces** (exchangeable types, well-formedness) and generates **invocation proxies** that check domain liveness, manage ownership transfer, and support thread unwinding. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Rust compiler**, the **microkernel**, a **small set of trusted crates** (hardware-interface + device crates that use `unsafe` Rust), the **IDL compiler**, and the **trusted compilation environment**; plus **CPU/devices** (devices trusted to be non-malicious — relaxable via IOMMU). Domains (safe Rust) are outside the TCB. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small** — described as a "minimal" microkernel + "small set" of trusted crates; **Unknown — no exact LOC reported** in the paper. (check repo). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — "we do not protect against side-channel attacks ... beyond the scope of the current work"; speculative-execution mitigations were **disabled** in the evaluation. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — relies on Rust's safety (formally studied elsewhere by RustBelt, cited as a future direction for verifying the `unsafe` code); RedLeaf itself is not formally verified. |
| Experimental validation available | Yes (specify) / No | **Yes** — cross-domain invocation cost vs seL4/VMFUNC/MPK, Rust-vs-C overhead, Ixgbe driver vs DPDK, NVMe vs SPDK, Maglev / kv-store / httpd app benchmarks, transparent driver recovery. Host: CloudLab c220g2 (2× Intel E5-2660 v3 10-core Haswell, 160 GB). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Rust overhead: idiomatic Rust ≈ **25% slower** than C (hashtable), but **C-style Rust within 3–10 cycles** of C. Apps: **kv-store** at **61–86%** of the C-DPDK version; Maglev forwards **5.3 Mpps/core** as an Rv6 application; drivers match DPDK/SPDK. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Cross-domain invocation = 124 cycles** (vs seL4 834, VMFUNC call/reply 396, MPK ~99–105/syscall); **+17 cycles per `RRef` passed**; via a shadow domain 279/297 cycles. No page-table or privilege switch. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Unknown** — domains are compiled independently and loaded dynamically, but no domain-load latency figure is reported. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **124 cycles** (same as the cross-domain invocation), **synchronous** migrating-thread model (the thread continues on the same stack into the callee). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Zero-copy** via `RRef<T>` move semantics; line-rate 10 GbE (14.2 Mpps at batch 32). Single-domain `nullnet` 29.5 Mpps → 5.3 Mpps when accessed as an **Rv6 application** (rv6-domain, multiple crossings) at batch 1; the single-crossing config is `redleaf-domain`. Crossing cost amortizes away at batch 32. |
| Memory overhead per domain | MB/domain | **Unknown** — two-level allocation (microkernel coarse regions + per-domain slab/buddy), but no MB/domain figure. |
| Domain count bounded by | Limiting factor; approx. domain count | no hardware/address-space limit (domains are lightweight, share one address space, no page tables); bounded by memory. No specific count given. |
| Performance scales with domain count | Big O | Cross-domain calls are **constant ~124 cycles (O(1))** regardless of domain count. memory O(n) in domain count. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** x86-64 — notably needs **no** MPK/VT-x/SGX (the whole point); tested on Intel Haswell. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Custom** — RedLeaf is a from-scratch microkernel OS, running on bare metal (not Linux). |
| What privileges does it need? | User / Root / Kernel access | **It is the OS** — runs bare-metal with all code (microkernel + domains) in ring 0. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | The Rust toolchain, the RedLeaf IDL compiler, and a **trusted compilation environment** (domains must be built against the same IDL/compiler version + flags; the microkernel verifies a signed fingerprint at load). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Rewrite** — software is written in **safe Rust** as RedLeaf domains. **Rv6**, a POSIX-subset OS personality (xv6-like), is implemented as a collection of domains. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **No** — everything is Rust domains; Rv6 offers a POSIX-subset but is not Linux-binary compatible. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Rust** — safe Rust for domains; `unsafe` Rust restricted to trusted crates. |
| Other compatibility notes | [Describe] | **No full `fork`** ("we do not support the full semantics of the `fork` system call as we do not rely on address spaces") — uses `create` instead; POSIX-subset via Rv6. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Unknown** — requires writing/porting software as safe-Rust domains; no person-days/LOC porting figure given. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Build effort | [build changes / time / deps] | Domains compiled against the trusted compilation environment + IDL; microkernel verifies the fingerprint at load. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Failure modes visibility | [crashes/logs/error codes/silent] | Domain panics are **fault-isolated**; cross-domain calls return an `RpcResult` **error** (never corrupted data or a propagated panic); device drivers recover transparently via shadow domains. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Unknown — no license declared (deep search exhausted).** Code at `github.com/mars-research/redleaf` (paper footnote: `mars-research.github.io/redleaf`). Verified: recursive tree, README, root `Cargo.toml` (no `license=` field), and `main.rs` (no SPDX header) carry **no project license**; the only license files in the tree belong to **vendored deps** (`rust-elfloader`=Apache/MIT, `spin-rs`). Public-source resolution impossible → **license Unknown — not resolvable from public sources (author contact not pursued)** |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — an "experimentation platform for enabling future system architectures that leverage language safety". |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **Limited** — RedLeaf is a standalone OS; composing with external mechanisms is not a focus (IOMMU integration to harden device trust is mentioned as future work). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **N/A** at system level (RedLeaf is the whole OS); domains coexist within it. |
| Can stack effectively | ✅ / ❌ | **✅** — the entire OS is built as composed domains; domains create other domains (e.g. `init` creates driver domains via the `create` trait). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Typed Rust function/trait calls** across domains (RPC-style), mediated by IDL-generated **invocation proxies**; a **migrating-thread** model (the thread moves into the callee domain). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | A global **shared heap** with `RRef<T>` **remote references** (zero-copy, ownership-tracked); only **exchangeable types** may cross domains. |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** — migrating-thread model on the same stack. |
| Interaction security/validation | [Describe] | IDL-validated interfaces (exchangeable types, well-formedness) + proxy-enforced ownership/liveness + Rust type safety. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Domain** — a subsystem/driver/app-granularity module. |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** defines domains and IDL interfaces; the **IDL compiler + microkernel** enforce them. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — domains are dynamically loaded and cleanly terminated at runtime. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom** — microkernel-scheduled threads with a **migrating-thread** cross-domain model (continuations for unwinding); domains register interrupt-handler threads. |
| Process model | fork/exec / Custom / N/A | **Custom** — **no full `fork`** (no address spaces); uses `create`; Rv6 provides POSIX-subset "processes" as domains. |
| POSIX compatibility | Full / Partial / Limited | **Partial** — via Rv6 (xv6/POSIX-subset); no full `fork`. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Domains must be safe Rust; types must be compiled consistently across domains (trusted compilation environment); no full `fork`; `unsafe` Rust in extensions not yet addressed. |
| Security caveats when layered | [Describe] | Trusts the Rust compiler's correctness + the trusted crates' `unsafe` Rust (unverified); side channels out of scope; devices trusted (relaxable via IOMMU). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server / datacenter** — high-throughput data-plane focus (10 GbE Ixgbe, NVMe, kv-stores, Maglev load balancer); the paper does not name a single target explicitly. |
| Deployment scale | Single device / Cluster / Large scale | **Single device** — a from-scratch OS evaluated on one machine; not framed as a scale dimension. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |

---

## Summary

> **What it is:** A research OS written from scratch in Rust that uses **only language (type + memory) safety** — no hardware isolation — to provide lightweight, fine-grained isolation **domains** that can be dynamically loaded, cleanly terminated, and transparently recovered, with zero-copy cross-domain communication enforced by Rust's ownership discipline plus IDL-generated invocation proxies.
>
> **Who it's for:** Systems builders exploring fine-grained fault isolation inside an OS kernel / high-throughput data plane without the cost of hardware isolation boundaries.
>
> **What it protects:** Domains (drivers, kernel subsystems, apps) from one another — heap isolation + fault isolation so a crashing/misbehaving domain is cleanly terminated without corrupting others; enables transparent device-driver recovery.
>
> **What it costs (effort/money/performance):** Extremely cheap cross-domain calls (**124 cycles** vs seL4's 834); Rust overhead 0–25% (idiomatic), with C-style Rust ≈ C; but everything must be (re)written in safe Rust.
>
> **What it needs (hardware/OS/expertise):** Commodity x86-64 (no MPK/VT-x/SGX), the Rust toolchain + RedLeaf IDL + trusted compilation environment; software written as safe-Rust domains.
>
> **Key tradeoffs:** Language-based isolation delivers near-zero-cost fine-grained domains, fault isolation, and transparent recovery without any hardware mechanism — but the whole system must be safe Rust on a from-scratch OS (no Linux binaries, no full `fork`), the TCB trusts the (unverified) Rust compiler + `unsafe` trusted crates, and side channels are out of scope.
>
> **Additional Notes:** A strong data point that language safety can replace hardware isolation at very low cost — the opposite mechanism from MPK/SFI/TEE systems. Contrast with CubicleOS/FlexOS (next), which pursue similar intra-address-space isolation via hardware (MPK).
