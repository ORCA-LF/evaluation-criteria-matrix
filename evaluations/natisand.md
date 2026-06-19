# Compartmentalization Evaluation Matrix — NatiSand

**Version:** v0.1
**System:** NatiSand (native-code sandboxing for JavaScript runtimes)
**Paper:** M. Abbadini, D. Facchinetti, G. Oldani, M. Rossi, S. Paraboschi. NatiSand: Native Code Sandboxing for JavaScript Runtimes. RAID 2023. DOI 10.1145/3607199.3607233.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — confining untrusted/vulnerable **native code** (binary programs and shared libraries) invoked by JS/TS apps so it cannot abuse host resources. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a native component (an executable or a shared library/function). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Per-native-component** — **Process** for executables (`command`→clone+exec) and **Library/function** for shared libs (`dlopen`/FFI). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **File / resource-level** — controls filesystem paths (Landlock), IPC channels, and network (eBPF); not memory/data isolation. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — pure OS-level enforcement. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — OS access control:** **Landlock** LSM (filesystem), **Seccomp-BPF** (syscalls), **eBPF** (IPC + network); plus OS **process isolation** for executables. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Process** (executables) + **thread-scoped security context** with policy-based ambient rights (shared libs, in-process; Landlock ambient rights are thread-based). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — a NatiSand component integrated into the JS runtime (Deno): policy parser, sandboxer, eBPF frontend, and a context pool; relies on kernel Landlock/Seccomp/eBPF. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Per-component least-privilege confinement** of host resources — default-deny access to filesystem (Landlock), IPC (Seccomp+eBPF), and network (eBPF) — independent of how native code is invoked. **Does NOT provide memory isolation:** a vulnerable shared library can still corrupt the runtime's in-process memory; the protection is at the system-resource boundary. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **Executables: ✅** (separate OS process). **Shared libraries: ❌** — run in the runtime's address space, so a fault there is not memory-isolated. Not framed as a crash-recovery mechanism. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial (resource-level)** — enforcement happens at the OS resource boundary (LSM/Seccomp/eBPF checks on each access), not as argument/interface validation of the JS↔native call (`command`/`dlopen`). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **OS kernel** (Landlock/Seccomp/eBPF), the **JS engine + runtime** (V8 + Deno, trusted to render JS securely), the **NatiSand component**, and the developer's **policy**. Native code (binaries + shared libs) is **untrusted** — may be malicious (supply chain) or vulnerable. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large** — includes the OS kernel + V8 engine + Deno runtime. **NatiSand-specific LOC: Unknown** (not reported in the paper). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None / out of scope** — not addressed (the mechanism is resource access control). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No.** |
| Experimental validation available | Yes (specify) / No | **Yes** — reproduced **32 high-severity CVEs** in native code used by popular packages and showed they are mitigated; extensive performance evaluation vs Minijail, Sandbox2, and a Wasm approach. Paper open-access CC BY 4.0; software is MIT. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Overhead is concentrated on **short-lived executables** (process + sandbox setup); **lower than Minijail and Sandbox2**, and **much lower than a Wasm-based approach**; library-call and long-running overhead is low (≈5–10 ms-level latency differences per microservice). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **N/A (resource sandbox, not a memory domain).** Native invocation is a runtime op call: `command` (clone+exec) or `dlopen`/FFI (≈function call). No lat_ctx figure. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Per-executable sandbox setup** (clone+exec + Landlock/Seccomp/eBPF configuration) adds latency, **noticeable for short-lived utilities**; a context pool is created at bootstrap to amortize. No single creation-latency figure reported. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **N/A** — JS→native via runtime ops; cross-process native↔native via OS IPC (itself confined by Seccomp/eBPF). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **N/A / Unknown** — no inter-domain bandwidth figure (resource sandbox). |
| Memory overhead per domain | MB/domain | **Unknown** — per-context state + eBPF maps; no MB/context figure reported. |
| Domain count bounded by | Limiting factor; approx. domain count | **Policy-defined security contexts**; limited by Landlock/eBPF resource limits. No specific cap reported. |
| Performance scales with domain count | Big O | overhead is per native-invocation (executable launch / sandbox config); context pool amortizes setup. No asymptotic statement. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — no special hardware; testbed is a commodity AMD Ryzen 3900X / Ubuntu 22.04. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock** — mainline Linux **≥ 5.13** (Landlock) plus eBPF and Seccomp (all mainline features). |
| What privileges does it need? | User / Root / Kernel access | **User at runtime — no root** (an explicit design goal); loading eBPF programs needs capabilities (CAP_BPF, and CAP_NET_ADMIN for networking) at setup. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | NatiSand integrated into the JS runtime (done for Deno); a JSON policy file per app; an interactive CLI tool assists policy generation and fits CI/CD. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None** — does not require changes to the application code or to third-party modules; the developer only writes a JSON policy. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — sandboxes **unmodified** native binaries and shared libraries. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any native** code (binary programs + shared libraries, typically C/C++); host is a **JS/TS** runtime. |
| Other compatibility notes | [Describe] | Integrated into **Deno**; designed to be generic across runtimes (Node.js, Bun); aligns with the runtimes' permission model; **stacks** with other LSMs (AppArmor/SELinux/SMACK), deny-takes-precedence. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Minimal** — author a JSON policy (CLI-assisted), no code changes; supports incremental adoption (sandbox highest-risk components first). No person-days figure. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Build effort | [build changes / time / deps] | One-time integration of NatiSand into the runtime; per-app effort is the policy file. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Failure modes visibility | [crashes/logs/error codes/silent] | Disallowed resource access is **denied** by Landlock/Seccomp/eBPF (operation fails); on LSM-decision conflicts, **deny takes precedence**. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — MIT** (`github.com/unibg-seclab/natisand`, GitHub SPDX = MIT). Paper text is CC BY 4.0 (available open source). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — prototype integrated into Deno. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — LSMs operate in **stacking mode**, so NatiSand composes with AppArmor/SELinux/SMACK and the runtime's own permission model. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — explicitly designed for stacking; deny-takes-precedence on conflicting LSM decisions. |
| Can stack effectively | ✅ / ❌ | **✅** — multiple security contexts per app; stacks with other LSMs. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | JS→native via runtime **op calls** (`command` for executables, `dlopen`/FFI for shared libraries); native↔native via OS **IPC** (confined by Seccomp/eBPF). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | Executables: **IPC + filesystem** (access-controlled). Shared libraries: **share the runtime's address space** (in-process, no memory isolation). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Synchronous** native invocation (`command`/`dlopen`); not explicitly characterized. |
| Interaction security/validation | [Describe] | Enforced as **resource ACLs** at the OS boundary (Landlock/Seccomp/eBPF) on each access; not argument-level validation. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process** (executables) / **Library** (shared libs) — per native component. |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer/developer** — via the JSON policy file (CLI-assisted). |
| Boundaries flexible at runtime | ✅ / ❌ | contexts are **policy-defined at startup** (bootstrap imports security contexts; a context pool is created); incremental adoption supported, but the policy is fixed per run. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | Landlock ambient rights are **thread-based**; NatiSand uses eBPF-aware threads + a context pool, integrating with the runtime's threading. |
| Process model | fork/exec / Custom / N/A | **Works with fork/exec** — executables launched via clone+exec, then confined. |
| POSIX compatibility | Full / Partial / Limited | **Full** (native code runs normally, just resource-restricted). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Shared libraries run **in-process** (no memory isolation); per-executable sandbox setup adds latency for short-lived utilities; eBPF program loading needs capabilities at setup. |
| Security caveats when layered | [Describe] | **No memory isolation** for in-process shared libraries — a memory-corruption bug in a sandboxed library can still corrupt the runtime; protection covers host *resources*, not the runtime's memory; the JS engine + runtime permission system are trusted; side channels not addressed. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server / Cloud** — backend web applications and microservices built on JS/TS runtimes. |
| Deployment scale | Single device / Cluster / Large scale | **Single device** (per-host runtime); not framed as a scale dimension. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |

---

## Summary

> **What it is:** A component for JavaScript runtimes (prototyped in **Deno**) that sandboxes untrusted **native code** — binary programs (`command`) and shared libraries (`dlopen`/FFI) — by confining each native component's access to the **filesystem (Landlock)**, **IPC and network (Seccomp + eBPF)** per a developer-provided JSON policy, with no application code changes and no root at runtime.
>
> **Who it's for:** Developers of JS/TS backends that call vulnerable third-party native modules and want to limit the host-resource damage those modules can do.
>
> **What it protects:** The host system's **resources** (files, IPC channels, network) from untrusted native code invoked by a JS app — per-native-component least privilege. (Not the runtime's in-process memory.)
>
> **What it costs (effort/money/performance):** Low overhead — lower than Minijail/Sandbox2, far lower than Wasm; concentrated on short-lived executables (process + sandbox setup, ms-level), negligible for library calls. Effort: a JSON policy, no code changes.
>
> **What it needs (hardware/OS/expertise):** Stock Linux ≥ 5.13 (Landlock) + eBPF + Seccomp; NatiSand integrated into the runtime; a JSON policy; no root at runtime.
>
> **Key tradeoffs:** Strong, OS-enforced, stackable resource confinement with zero code changes and easy policy — but it is **resource access control, not memory isolation**: shared libraries run in-process and can still corrupt the runtime's memory, and the JS engine + runtime remain trusted.
>
> **Additional Notes:** Best read alongside K23/Capsicum as an **OS-sandboxing / access-control** entry rather than a memory-compartmentalization one. Its distinctive contribution is bringing per-native-component, policy-driven confinement (Landlock+eBPF+Seccomp) into JS runtimes transparently.
