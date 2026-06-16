# Compartmentalization Evaluation Matrix — NatiSand

**System:** NatiSand (native-code sandboxing for JavaScript runtimes)
**Paper:** M. Abbadini, D. Facchinetti, G. Oldani, M. Rossi, S. Paraboschi. *NatiSand: Native Code Sandboxing for JavaScript Runtimes.* RAID 2023. DOI 10.1145/3607199.3607233.
**Source file:** `papers/natisand1.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-10

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment, not a paper claim · **N/A** = not assessable / not applicable · **Unknown** = a real value the paper/sources do not report.
>
> **⚠️ Framing note (important):** NatiSand is a **resource-confinement** sandbox, **not a memory-isolation** mechanism. It restricts what *system resources* (filesystem, IPC, network, syscalls) untrusted native code may touch — via Landlock (LSM), Seccomp, and eBPF — per **security context** (a policy-defined compartment). For **shared libraries** (`dlopen`/FFI) the native code runs **in the runtime's address space with no memory isolation**; only host-resource access is confined. For **executables** (`command`) it adds a separate process. Closest in this set to Capsicum / OS-sandboxing, not to MPK/SFI/CHERI memory isolation.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — confining untrusted/vulnerable **native code** (binary programs and shared libraries) invoked by JS/TS apps so it cannot abuse host resources (Abstract, §1, §3.1). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a native component (an executable or a shared library/function) (§3.1, §4.2). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Per-native-component** — **Process** for executables (`command`→clone+exec) and **Library/function** for shared libs (`dlopen`/FFI) (§2.1, §4.2). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **File / resource-level** — controls filesystem paths (Landlock), IPC channels, and network (eBPF); not memory/data isolation (§4.2). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — pure OS-level enforcement (§4). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — OS access control:** **Landlock** LSM (filesystem), **Seccomp-BPF** (syscalls), **eBPF** (IPC + network); plus OS **process isolation** for executables (§2.2, §4.2). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Process** (executables) + **thread-scoped "security context"** with policy-based ambient rights (shared libs, in-process; Landlock ambient rights are thread-based) (§4.2). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — a NatiSand component integrated into the JS runtime (Deno): policy parser, "sandboxer", eBPF frontend, and a context pool; relies on kernel Landlock/Seccomp/eBPF (§4.2, Fig. 3). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Per-component least-privilege confinement** of host resources — default-deny access to filesystem (Landlock), IPC (Seccomp+eBPF), and network (eBPF) — independent of how native code is invoked (§4.2). **Does NOT provide memory isolation:** a vulnerable shared library can still corrupt the runtime's in-process memory; the protection is at the system-resource boundary (§3.1, framing note). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | _Inferred:_ **Executables: ✅** (separate OS process). **Shared libraries: ❌** — run in the runtime's address space, so a fault there is not memory-isolated (§4.2). Not framed as a crash-recovery mechanism. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial (resource-level)** — enforcement happens at the OS resource boundary (LSM/Seccomp/eBPF checks on each access), not as argument/interface validation of the JS↔native call (`command`/`dlopen`) (§4.2). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **OS kernel** (Landlock/Seccomp/eBPF), the **JS engine + runtime** (V8 + Deno, trusted to render JS securely), the **NatiSand component**, and the developer's **policy**. Native code (binaries + shared libs) is **untrusted** — may be malicious (supply chain) or vulnerable (§3.1). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large** — includes the OS kernel + V8 engine + Deno runtime. **NatiSand-specific LOC: Unknown** (not reported in the paper). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | _Inferred:_ **None / out of scope** — not addressed (the mechanism is resource access control) (§3.1). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No.** |
| Experimental validation available | Yes (specify) / No | **Yes** — reproduced **32 high-severity CVEs** in native code used by popular packages and showed they are mitigated (§7.1, Table 3); extensive performance evaluation vs Minijail, Sandbox2, and a Wasm approach (§7.2). Paper open-access CC BY 4.0; software is MIT [repo]. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Overhead is concentrated on **short-lived executables** (process + sandbox setup); **lower than Minijail and Sandbox2**, and **much lower than a Wasm-based approach**; library-call and long-running overhead is low (≈5–10 ms-level latency differences per microservice) (§7.2). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **N/A (resource sandbox, not a memory domain).** Native invocation is a runtime op call: `command` (clone+exec) or `dlopen`/FFI (≈function call). No lat_ctx figure (§2.1). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Per-executable sandbox setup** (clone+exec + Landlock/Seccomp/eBPF configuration) adds latency, **noticeable for short-lived utilities**; a context pool is created at bootstrap to amortize. No single creation-latency figure reported (§4.2, §7.2). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **N/A** — JS→native via runtime ops; cross-process native↔native via OS IPC (itself confined by Seccomp/eBPF) (§4.2). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **N/A / Unknown** — no inter-domain bandwidth figure (resource sandbox). |
| Memory overhead per domain | MB/domain | **Unknown** — per-context state + eBPF maps; no MB/context figure reported. |
| Domain count bounded by | Limiting factor; approx. domain count | **Policy-defined security contexts**; limited by Landlock/eBPF resource limits. No specific cap reported (§4.2). |
| Performance scales with domain count | Big O | _Inferred:_ overhead is per native-invocation (executable launch / sandbox config); context pool amortizes setup. No asymptotic statement (§4.2, §7.2). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — no special hardware; testbed is a commodity AMD Ryzen 3900X / Ubuntu 22.04 (§7). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock** — mainline Linux **≥ 5.13** (Landlock) plus eBPF and Seccomp (all mainline features) (§2.2). |
| What privileges does it need? | User / Root / Kernel access | **User at runtime — no root** (an explicit design goal); loading eBPF programs needs capabilities (CAP_BPF, and CAP_NET_ADMIN for networking) at setup (§2.2, §4.1). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | NatiSand integrated into the JS runtime (done for Deno); a JSON policy file per app; an interactive CLI tool assists policy generation and fits CI/CD (§1, §4.1). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None** — "does not require changes to the application code" or to third-party modules; the developer only writes a JSON policy (§1, §4.1). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — sandboxes **unmodified** native binaries and shared libraries (§1, §4.1). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any native** code (binary programs + shared libraries, typically C/C++); host is a **JS/TS** runtime (§1). |
| Other compatibility notes | [Describe] | Integrated into **Deno**; designed to be generic across runtimes (Node.js, Bun); aligns with the runtimes' permission model; **stacks** with other LSMs (AppArmor/SELinux/SMACK), deny-takes-precedence (§4.2). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Minimal** — author a JSON policy (CLI-assisted), no code changes; supports incremental adoption (sandbox highest-risk components first) (§4.1). No person-days figure. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | One-time integration of NatiSand into the runtime; per-app effort is the policy file (§4). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Failure modes visibility | [crashes/logs/error codes/silent] | Disallowed resource access is **denied** by Landlock/Seccomp/eBPF (operation fails); on LSM-decision conflicts, **deny takes precedence** (§4.2). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — MIT** [repo] (`github.com/unibg-seclab/natisand`, GitHub SPDX = MIT). Paper text is CC BY 4.0 (§ Abstract: "available open source"). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — prototype integrated into Deno (§1, §6). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — LSMs operate in **stacking mode**, so NatiSand composes with AppArmor/SELinux/SMACK and the runtime's own permission model (§4.2). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — explicitly designed for stacking; deny-takes-precedence on conflicting LSM decisions (§4.2). |
| Can stack effectively | ✅ / ❌ | **✅** — multiple security contexts per app; stacks with other LSMs (§4.2). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | JS→native via runtime **op calls** (`command` for executables, `dlopen`/FFI for shared libraries); native↔native via OS **IPC** (confined by Seccomp/eBPF) (§2.1, §4.2). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | Executables: **IPC + filesystem** (access-controlled). Shared libraries: **share the runtime's address space** (in-process, no memory isolation) (§2.1, §4.2). |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Synchronous** native invocation (`command`/`dlopen`); not explicitly characterized (§2.1). |
| Interaction security/validation | [Describe] | Enforced as **resource ACLs** at the OS boundary (Landlock/Seccomp/eBPF) on each access; not argument-level validation (§4.2). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process** (executables) / **Library** (shared libs) — per native component (§4.2). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer/developer** — via the JSON policy file (CLI-assisted) (§4.1). |
| Boundaries flexible at runtime | ✅ / ❌ | _Inferred:_ contexts are **policy-defined at startup** (bootstrap imports security contexts; a context pool is created); incremental adoption supported, but the policy is fixed per run (§4.2). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | Landlock ambient rights are **thread-based**; NatiSand uses eBPF-aware threads + a context pool, integrating with the runtime's threading (§2.2, §4.2). |
| Process model | fork/exec / Custom / N/A | **Works with fork/exec** — executables launched via clone+exec, then confined (§2.1, §4.2). |
| POSIX compatibility | Full / Partial / Limited | **Full** (native code runs normally, just resource-restricted) (§4.2). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Shared libraries run **in-process** (no memory isolation); per-executable sandbox setup adds latency for short-lived utilities; eBPF program loading needs capabilities at setup (§3.1, §4.2, §7.2). |
| Security caveats when layered | [Describe] | **No memory isolation** for in-process shared libraries — a memory-corruption bug in a sandboxed library can still corrupt the runtime; protection covers host *resources*, not the runtime's memory; the JS engine + runtime permission system are trusted; side channels not addressed (§3.1). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server / Cloud** — backend web applications and microservices built on JS/TS runtimes (§1, §6). |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Single device** (per-host runtime); not framed as a scale dimension. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |

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

---

## Sources

- **Primary:** `papers/natisand1.pdf` — Abbadini et al., *NatiSand: Native Code Sandboxing for JavaScript Runtimes*, RAID 2023, DOI 10.1145/3607199.3607233. All **(§N)** citations reference this paper. (Read via `pdftotext` extraction.)
- **[repo]** `github.com/unibg-seclab/natisand` — GitHub SPDX = **MIT**. Paper text licensed CC BY 4.0.

### Cells flagged for further research
- **Security › TCB size (NatiSand LOC)** — Unknown (not reported).
- **Efficiency › Memory per domain, Inter-domain throughput, Domain creation latency** — Unknown / N/A (resource sandbox; no figures).
- Self-report dims (debugging, expertise, config, maintenance, monitoring) → **N/A** by policy 2026-06-09.
- **(structural)** — resource-confinement sandbox, not memory isolation; many memory-isolation rows are N/A by category. Confirm with ORCA team how this maps in the matrix.
- Rating/classification inferences (crash isolation, side channels, deployment scale, boundary flexibility, interaction semantics) marked `_Inferred:_` inline.
