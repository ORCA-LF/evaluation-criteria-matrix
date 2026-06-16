# Compartmentalization Evaluation Matrix — Cerberus

**System:** Cerberus (a PKU-based **sandboxing framework** that hardens existing PKU memory-isolation schemes against bypass; compartment = the underlying scheme's trusted **M_T** vs untrusted **M_U** PKU domains)
**Paper:** A. Voulimeneas, J. Vinck, R. Mechelinck, S. Volckaert. *You Shall Not (by)Pass! Practical, Secure, and Fast PKU-based Sandboxing.* EuroSys 2022. (imec-DistriNet, KU Leuven)
**Source file:** `papers/cerberus.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-11

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment · **⚠️ Unclear** = not found, needs research.
>
> _Cerberus is **not a new isolation scheme** — it is a **bypass-resistant sandbox framework** for **PKU/MPK** memory-isolation schemes. The underlying scheme (e.g. [[erim]], Hodor, XOM-Switch) supplies the compartments — a trusted domain **M_T** and an untrusted domain **M_U**, each a set of pages tagged with a different PKU protection key. Cerberus's job is to stop an attacker who has seized **M_U** from tampering with the `PKRU` register (via unsafe `wrpkru`/`xrstor` instructions, kernel confused-deputy syscalls, relocation, threads, etc.) to unlock **M_T**. So most "isolation model" rows describe the hosted scheme; the Cerberus-specific contribution is the **sandbox/monitor** (§1, §5)._
>
> _Entry type: an MPK-family **hardening framework / meta-system**. Closely related to [[erim]], [[endoprocess]] (Endokernel), Hodor, Jenny, [[cubicleos]], [[flexos]]._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation / Secret protection** — keep an attacker who controls the untrusted domain **M_U** from accessing the trusted domain **M_T** (e.g. CPI safe regions, shadow stacks, OpenSSL keys, execute-only code) — by making the host PKU sandbox **non-bypassable** (§1, §6.1, §7.1). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Hybrid** — code is split into trusted/untrusted domains (code-centric) whose **data pages** are tagged with PKU keys (data-centric) (§2.1, §3). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Library / region** (inherited) — ERIM/Hodor isolate trusted vs untrusted *libraries/components*; Cerberus assumes two trust levels but can support more (§2.2, §3, §8). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Page** — PKU tags memory pages with one of 16 protection keys; M_T/M_U are page sets (§2). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **MPK/PKU** (Intel/AMD Memory Protection Keys for Userspace) — the `PKRU` register gates per-key data access (§2). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — a bypass-resistant sandbox**: static **binary rewriting** (ERIM's SBI, to remove unsafe `wrpkru`/`xrstor`) + **runtime instruction vetting** (Hodor-style hardware breakpoints + single-step, extended to `xrstor` + **x86 emulation** for multithreading) + **syscall interception** (a ptrace monitor + a minimal in-kernel syscall agent) (§5, §6). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — PKU memory domains within one process; trusted/untrusted code share the address space (§1, §2). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the Cerberus **monitor** (ptrace), **loader** (injects an `rdpkru` code page), **syscall agent** (in-kernel patch), and APIs/emulation engine; plus the underlying PKU isolation scheme (§5, §6). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Non-bypassable PKU isolation** — `M_U` cannot unlock `M_T` via: unsafe `wrpkru`/`xrstor` (vetted/emulated), `process_vm_readv/writev`/`ptrace`/`/proc/self/mem` (blocked), mutable file-backed & `MAP_SHARED`-executable mappings (forbidden, W^X enforced), `mremap` relocation (re-scanned), `seccomp`/`prctl` tricks (blocked), `pkey_mprotect` on trusted pages (forbidden from U), scan-time TOCTTOU races (per-thread monitor + critical sections), and the two new Hodor attacks. **Except signal-context attacks** (out of scope) (§4, §5.2, §7.2, Table 4). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | _Inferred:_ **✅ (fail-stop)** — on a detected tampering attempt the monitor terminates the program; the underlying PKU scheme confines `M_U` to its own domain. Not framed as crash-recovery (§5.2). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — call gates (inherited) are the trusted↔untrusted transition; Cerberus adds an **AllowList** of vetted safe instruction sequences (e.g. ERIM's call gates), reliable current-domain identification via `PKRU` readout, and complete syscall mediation of the boundary-relevant calls (§5.2, §5.3, §6.1). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **CPU (PKU) + OS kernel + the trusted code T + the Cerberus monitor/loader/syscall-agent.** "The kernel is considered part of the TCB"; PKU implementation trusted; T assumed free of exploitable bugs (§3). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Medium (framework) + Large (kernel)** — Cerberus ≈ **11 KLOC** total (monitor ≈1 KLOC, loader ≈650 LOC, APIs ≈9.4 KLOC, syscall agent ≈79 LOC, ≈229 LOC shell), + an emulation engine of **170 x86 instructions ≈2,752 LOC**; built on the ReMon MVEE; the whole Linux kernel stays trusted (§6). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None / out of scope** — "Attacks that target the underlying hardware such as transient execution and remote-fault injection are considered out of scope" (Spectre/Meltdown/Rowhammer/V0ltpwn) (§3). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal proofs; a systematic enumeration + empirical defeat of known + 2 new attacks (§7.2). |
| Experimental validation available | Yes (specify) / No | **Yes** — security: defeats all PKU-Pitfalls PoCs + the 2 new Hodor attacks except signal-context (Table 4); performance: nginx/lighttpd/redis with ERIM-CPI/SS/OpenSSL and XOM-Switch sandboxes (Tables 1–3) (§7). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC; server-app throughput.** Sandbox overhead is **low and dominated by the underlying scheme**: ERIM-CPI+Cerberus geomean **4.10%** (vs 3.87% no-sandbox), ERIM-SS+Cerberus **1.43%** (vs 0.71%), ERIM-OpenSSL+Cerberus **0.47%**, XOM-Switch+Cerberus **0.33%** (Tables 1–3). Host: 12-core Xeon Silver 4214 @ 2.20 GHz, 64 GB, Ubuntu 18.04 / Linux 5.3.18 (§7.1). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **N/A (inherited, fast)** — PKU domain switches use unprivileged `wrpkru` call gates (no syscall/TLB flush); Cerberus does not change the fast path. No lat_ctx figure (§2, §2.2.1). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **N/A** — domains are set up via `pkey_alloc`/`pkey_mprotect` by the host scheme (Cerberus monitors these); no creation-latency microbenchmark (§5.2). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **N/A (inherited)** — trusted↔untrusted transitions are call gates (`wrpkru`), near-native; not separately measured by Cerberus (§2.2.1). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **N/A** — same address space, no IPC; end-to-end server throughput is the reported metric (Tables 1–3). |
| Memory overhead per domain | MB/domain | ⚠️ **No MB/domain figure.** Cerberus adds a per-thread monitor + a one-page `rdpkru` code stub; the underlying PKU keys add no per-domain memory (§5.2). |
| Domain count bounded by | Limiting factor; approx. domain count | **Inherited: 16 hardware PKU keys** (the host scheme's limit; Cerberus's prototype uses 2 trust levels but can determine the current domain for >2) (§3, §8). |
| Performance scales with domain count | Big O | _Inferred:_ flat — overhead is per-syscall-interception + per-unsafe-instruction-vetting, not per-domain; servers saturated the NIC at 3 workers (§7.1). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — recent Intel/AMD x86 server/desktop CPUs with PKU (§2, §7.1). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Modified Linux (minor)** — a small kernel patch implementing the syscall agent + a new `prctl` option (Linux 5.3.18); ERIM's & Cerberus's sandboxes run mostly in user space (§6, Table 5 "Minor"). |
| What privileges does it need? | User / Root / Kernel access | _Inferred:_ **root to install** the kernel patch; the monitor uses ptrace (user-level); the protected app runs unprivileged (§5, §6). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | The ReMon-based monitor/loader; ERIM's SBI binary rewriter; for ERIM-CPI/SS host schemes, the ERIM/CPI/SS compiler passes (§6). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **For the sandbox: none** (binary rewriting + runtime monitoring, applied to app + system libs); **for the host isolation scheme:** whatever it needs (e.g. ERIM-CPI/SS = compiler passes; ERIM-OpenSSL/XOM-Switch = config/none) (§6.1, §7.1). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes (sandbox side)** — Cerberus rewrites/monitors existing binaries (app + `ld.so`/`libc`/`libm`); the underlying scheme may require recompilation (§6.1, §7.1.1). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C/C++** (evaluated on nginx/lighttpd/redis); the PKU model is language-agnostic at the binary level (§7.1). |
| Other compatibility notes | [Describe] | Restrictions on mutable-backed / `MAP_SHARED`-executable mappings could break old double-mapping JITs (modern JITs don't use it); multithreading handled via instruction emulation (§8). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Very low to build a sandbox for a scheme** — the ERIM sandbox use case = **≈55 LOC**; the XOM-Switch sandbox = **≈16 LOC** on top of the default Cerberus sandbox (§6.1). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | Apply the kernel patch; (optionally) run ERIM's SBI rewriter; extend the Cerberus loader/monitor via APIs; build on ReMon (§5.1, §6). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(Cerberus APIs do provide a "rich logging infrastructure" — §5.3.)_ |
| Failure modes visibility | [crashes/logs/error codes/silent] | A detected tampering attempt → monitor **terminates** the program; blocked syscalls/files denied; rich logging via the APIs (§5.2, §5.3). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — MIT** [repo] (`github.com/ku-leuven-msec/The-Cerberus-Project`, `LICENSE.md` = MIT, © 2020–2022 Voulimeneas/Vinck/Mechelinck/Volckaert; verified 2026-06-11. GitHub auto-classifies it as NOASSERTION only because of the file's "unless stated otherwise" preamble.) Paper states open source (§1). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** prototype (EuroSys 2022) (§6, §10). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅ (its whole purpose)** — Cerberus is a framework that *builds sandboxes for* other PKU isolation schemes ([[erim]], Hodor, XOM-Switch); reuses ERIM's SBI and Hodor's vetting idea; decouples sandbox development from the isolation scheme (§5.1, §6.1). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — runs as a ptrace monitor + kernel agent alongside normal processes; the same syscall agent serves all Cerberus-built sandboxes (§5.1). |
| Can stack effectively | ✅ / ❌ | **Partial** — prototype uses 2 trust levels but the framework "can determine the current executing domain" for >2 domains, enabling multi-level sandboxes (§8). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Function calls via call gates** (inherited from the PKU scheme) — `wrpkru`-bracketed transitions; Cerberus vets/allow-lists them (§2.2.1, §5.3). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** (same address space) — trusted code accesses both domains, untrusted only its own; Cerberus forbids dangerous shared/file-backed executable mappings (§2.1, §5.2). |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Synchronous** — call-gate transitions; not framed as a dimension (§2.2.1). |
| Interaction security/validation | [Describe] | Call gates allow-listed; current domain reliably identified via `PKRU` readout; all boundary-relevant syscalls intercepted; W^X enforced (§5.2, §5.3). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Library/component** (inherited — trusted vs untrusted) (§2.2). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer/Compiler (host scheme) + Sandbox developer (Cerberus APIs)** — the isolation scheme defines M_T/M_U; the developer writes a small Cerberus sandbox to make it non-bypassable (§6.1). |
| Boundaries flexible at runtime | ✅ / ❌ | _Inferred:_ **✅** — PKU domains/keys set up at runtime via `pkey_*` (monitored by Cerberus); domain identity tracked dynamically (§5.2). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Standard threads (explicitly supported)** — a unique monitor thread per app thread + shared critical sections + x86 emulation handle the multithreaded attacks (race scanning, thread-local breakpoints) that ERIM/Hodor mishandle (§5.2, §4.3.2). |
| Process model | fork/exec / Custom / N/A | **Standard** — the monitor re-initializes the syscall agent after `execve`/`fork`/`clone`; all fork/clone variants intercepted (§5.2). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — full POSIX except restricted `pkey_mprotect`/`mmap`(MAP_SHARED+exec)/`mremap`/`process_vm_*`/`ptrace`/`seccomp`/`/proc/self/mem` from untrusted code (§5.2). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | **Signal-context attacks not stopped** (out of scope; Endokernel-style fix proposed); mutable-backed/double-mapping restrictions can break old JITs; instruction emulation can be slow if many pages hold unsafe instructions; 16-key PKU limit (§4.2.7, §8). |
| Security caveats when layered | [Describe] | Trusted code T assumed bug-free; whole kernel + PKU trusted; transient-execution / fault-injection side channels out of scope; security depends on the host scheme correctly defining M_T (§3). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server** — high-performance server apps (nginx, lighttpd, redis) using PKU isolation (§7.1). |
| Deployment scale | Single device / Cluster / Large scale | _Inferred:_ **Single device** — per-process in-process isolation hardening; not framed as a scale dimension (§7). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |

---

## Summary

> **What it is:** A **framework for building bypass-resistant sandboxes** for PKU/MPK in-process memory-isolation schemes. PKU isolation ([[erim]], Hodor, XOM-Switch) is fast but bypassable: an attacker controlling the untrusted domain can flip the `PKRU` register via stray `wrpkru`/`xrstor` instructions or by turning the kernel into a confused deputy. Cerberus closes these holes by combining static binary rewriting (remove unsafe instructions), runtime instruction vetting (hardware breakpoints + single-step + x86 emulation, extended to `xrstor` and made multithread-safe), and complete interception of the dangerous syscalls (ptrace monitor + a minimal in-kernel syscall agent). The paper also finds **two new attacks on Hodor** and shows Cerberus stops them.
>
> **Who it's for:** Builders/users of PKU-based in-process isolation who need it to actually hold against a code-reuse attacker — without paying for CFI.
>
> **What it protects:** The trusted domain (CPI safe regions, shadow stacks, OpenSSL keys, execute-only code) from an attacker who has compromised the untrusted domain — by making the PKU sandbox non-bypassable (all known attacks except signal-context).
>
> **What it costs (effort/money/performance):** Tiny per-scheme sandbox code (≈16–55 LOC on top of the framework), a minor kernel patch, and **low overhead dominated by the underlying scheme** (Cerberus adds ~0.3–1.4% geomean on top of ERIM/XOM-Switch).
>
> **What it needs (hardware/OS/expertise):** Commodity Intel/AMD PKU CPU, a slightly-patched Linux, and the ReMon-based monitor/loader + ERIM's SBI rewriter.
>
> **Key tradeoffs:** The first PKU sandbox to handle unsafe instructions *completely* (incl. `xrstor` and multithreaded code) and to block the full known attack set at low cost and small per-scheme effort — but it's a hardening layer, not an isolation scheme; it doesn't stop signal-context attacks, trusts the whole kernel + PKU, leaves side channels out of scope, and inherits the 16-key PKU limit.
>
> **Additional Notes:** An MPK-family **hardening framework / meta-system** — distinct in the matrix because its "compartments" are inherited from the scheme it wraps ([[erim]] etc.), and its contribution is *bypass resistance*, not a new boundary. Directly complements [[erim]] (whose SBI it reuses) and [[endoprocess]] (Endokernel, the concurrent work it cites for signal handling); siblings in the PKU cluster include Hodor, Jenny, [[cubicleos]], [[flexos]]. From the same KU Leuven group as [[salus]] (different sub-group; unrelated mechanism). Honest about its one gap — signal-context attacks (§4.2.7).

---

## Sources

- **Primary:** `papers/cerberus.pdf` — Voulimeneas, Vinck, Mechelinck, Volckaert, *You Shall Not (by)Pass! Practical, Secure, and Fast PKU-based Sandboxing*, EuroSys 2022, DOI 10.1145/3492321.3519560. All **(§N)** / Table citations reference this paper.
- **Repo:** `github.com/ku-leuven-msec/The-Cerberus-Project` (§1) — open source; **license MIT** (`LICENSE.md`, verified 2026-06-11; GitHub shows NOASSERTION only due to the file's "unless stated otherwise" preamble). **[repo]**
- **Hosted/related schemes:** ERIM [70] ([[erim]]), Hodor [36], XOM-Switch [55], PKU Pitfalls [21], Endokernel [15] ([[endoprocess]]), Jenny [64].

### Cells flagged for further research
- **Availability › License** — RESOLVED: **MIT** confirmed via repo `LICENSE.md` (verified 2026-06-11); paper states open source (§1).
- **Efficiency › lmbench cells (switch/creation/call/throughput), MB/domain** — ⚠️ inherited from the PKU scheme / not separately measured (only end-to-end server overhead in Tables 1–3).
- **Usability › Required expertise, Debugging** + **Sys Design › Monitoring, Config complexity, Maintenance burden** — N/A (self-report policy 2026-06-09).
- Inferences marked `_Inferred:_` inline: crash isolation, privileges, scaling, boundaries-runtime-flexible, interaction semantics, deployment scale.
- **Category note:** a hardening framework, not a standalone isolation scheme — isolation-model rows describe the hosted scheme (ERIM/Hodor/XOM-Switch); Cerberus's contribution is the non-bypassable sandbox.
