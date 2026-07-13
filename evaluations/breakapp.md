# Compartmentalization Evaluation Matrix — BreakApp

**Version:** v0.1
**System:** BreakApp (automated module-boundary compartmentalization; compartment = a module spawned into a configurable box — sandbox / process / container)
**Paper:** N. Vasilakis, B. Karel, N. Roessler, N. Dautenhahn, A. DeHon, J. M. Smith. BreakApp: Automated, Flexible Application Compartmentalization. NDSS 2018.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — isolate risky **third-party modules** (which can be ~99.9% of shipped code) at their natural boundaries to contain supply-chain/dependency vulnerabilities. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a module/package; data crosses the boundary only via the module's API (RPC). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Library/Module** — package-level (default) or file-level modules; the `LEVEL`/`GROUP` policies tune granularity. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is module-granular; cross-module data is copied/proxied, not page/region-tagged. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — relies on language/runtime + OS isolation, not hardware primitives (PROC/LXC use the OS kernel). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Boundary wrappers/marshalling + Language runtime** — automated source/AST transformation of `import` into compartment-spawn + **RPC proxies/interposition**, layered on a memory-safe managed runtime; OS process/container isolation for heavier boxes. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Other — configurable (BOX)**: Intra-runtime sandbox (SBX) / Process (PROC) / Container (LXC), chosen per import; heavier compartment types … positively correlated with better isolation. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the BreakApp module-system replacement (must load first) + the language runtime (V8/Node.js) + OS process/Docker support for PROC/LXC; a trusted mini standard library to locate/load modules. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Confidentiality + integrity + availability at module boundaries**: blocks a malicious module from reading/writing global state, traversing the module cache, patching built-ins, inspecting the call stack, exfiltrating via env/fs/net, and (with PROC/replicas) DoS-ing the whole app. The mitigations are **per-box-type**: SBX stops globals/introspection/eval/context attacks; PROC adds stack-inspection, module-cache, process-args, DoS, unsafe-extension defenses; LXC adds fs/net and process-env. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅ (PROC/LXC) / Partial (SBX)** — BreakApp **monitors compartment health** and the `ON_FAIL` policy can log/restart/kill/replicate a faulting compartment; children get `SIGHUP` on parent exit. PROC/LXC give true crash isolation; SBX shares the heap + event queue so a crash/DoS is less contained (mitigated by moving to PROC). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the module API *is* the validated boundary: all cross-compartment interaction is mediated by generated RPC stubs/interposition proxies; `CONTEXT` whitelists exactly which state may be shared; `TRUST`/`DOUBT` whitelist/blacklist modules; BreakApp self-protects (immutable core, source-scan + interposition against tampering). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The language runtime (V8/JIT) + built-in libraries (`fs`, `net`, …) + BreakApp itself + the OS (for PROC/LXC) + CPU.** We assume that the core language runtime and built-in libraries … can be trusted; a minimal trusted (ideally formally-verified) stdlib is needed to locate/load modules. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large** — the V8/Node runtime + built-ins dominate the TCB; **BreakApp's own code is small (~2K LOC JS**, +2K for the NaCL crypto lib via Emscripten). The paper suggests a trusted (say, formally verified) version of a mini standard library to shrink the load-time TCB. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **Partial (timing only, opt-in)** — the `TIMER` policy sets a **minimum response time** for cross-boundary calls to blunt timing attacks (e.g. `fernet`); `ENCRYPT` can encrypt the channel. SBX shares the call stack/event queue (a leakage vector that requires moving to PROC). No cache/speculative defenses. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal proofs; the core transformation is given as a small functional algorithm but not verified. |
| Experimental validation available | Yes (specify) / No | **Yes** — mitigates 6 real CVE-class vulnerabilities + 6 hypothetical ones with specific policies; applicability study over 15 real apps; microbenchmarks (startup, throughput/latency, proxy cost); an end-to-end DoS-mitigation experiment on Ghost. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** In a realistic HTTP app, compartmentalization (IPC request/response) is **2–15% of overall latency** even for heavier box types (HTTP handling dominates); the proxy/interposition overhead is barely visible. Host: 160-core Intel Xeon E7-8860 @ 2.27 GHz, 512 GB, Node.js 6.9.1, Docker 17.06. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Not lat_ctx; cross-boundary call latency instead** (5 compartments): in-runtime **Function 6.5 ns**, **Pipe 1.3–1.4 ms**, **UDS 17.8–73.8 ms**, **TCP 17.7–36.6 ms** (includes serialize/deserialize). Choice of `IPC` (TCP/UDS/FIFO) trades throughput vs latency. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Per-additional-compartment startup**: **SBX 1.5 ms (5×)**, **PROC 72 ms (240×)**, **LXC 666 ms (2220×)** vs Standard 0.3 ms baseline. Parallel spawn: 500 compartments in 2.82 s, 5K in 24.3 s (good elasticity). Dominated by module files read from disk. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Pipe ≈ 1.3–1.4 ms** at 5 compartments (rising to ~11–13 ms at 50, ~155 ms at 500); UDS/TCP higher. This is the explicit cross-compartment call cost. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Measured (5 cmpts):** in-runtime Function **192.3 GB/s**, **Pipe 18.3 GB/s**, **UDS 149.5 MB/s**, **TCP 158.1 MB/s**; degrades with compartment count (Pipe 3.6 GB/s at 500). |
| Memory overhead per domain | MB/domain | **No MB/compartment figure.** SBX shares the heap/event-queue; PROC = a new process; LXC = a container — qualitatively increasing footprint, but not quantified per-domain. |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: startup cost + system resources** (esp. LXC: 500 sequential = 332 s). Real apps need ~2–65 compartments at package-trust boundaries, ~326 avg at per-package, 1–2 orders more at file level. |
| Performance scales with domain count | Big O | startup ≈ **O(n)** per-compartment (fixed per-box cost ×count); cross-boundary throughput degrades with count. More compartments → finer security but higher startup/comms cost. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — no special hardware. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock** — runs on an unmodified OS + the language runtime; PROC uses ordinary OS processes, LXC uses Docker. |
| What privileges does it need? | User / Root / Kernel access | **user-level** for SBX/PROC; container privileges for LXC (Docker). No kernel modification. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | BreakApp must be loaded **before any other module**; built on Andromeda + Node.js; a trusted mini stdlib to load modules; `breakapp --create-map` post-install hook to canonicalize the dependency tree. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None required** — does not require any annotations, … tracing/inference (pre-)runs, … or manual rewriting. Optional per-import policies fine-tune behavior; defaults apply otherwise. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes (existing source modules), no recompile** — transformations happen at runtime on `import`; policies are backward- and forward-compatible (ignored by a vanilla `require`). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **JavaScript** (prototype); technique is language-independent and a good fit for interpreted langs (Lua/Python/Ruby); compiled langs (C/Rust) are feasible, albeit nontrivial (split across compile/link/runtime). Native C modules can be isolated in PROC (can't forge pointers). |
| Other compatibility notes | [Describe] | Maintains single-runtime semantics via heavy machinery: distributed references, copy-propagation of primitives, GC-event propagation, reference-equality preservation, class-hierarchy redirection, stream/console redirection. Replication (`REPLICAS`) needs care for stateful modules. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Very low** — zero changes to run with defaults; optional one-line per-import policies. No person-days figure; the design goal is low developer effort. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator. |
| Build effort | [build changes / time / deps] | `npm install -g breakapp`; optionally run `breakapp -c` as a post-install hook to build the file→package map; no app rebuild. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator. _(BreakApp does provide monitoring + `ON_FAIL` actions + violation reporting.)_ |
| Failure modes visibility | [crashes/logs/error codes/silent] | Exceptions in a child are serialized and re-thrown in the parent; violations trigger `ON_FAIL` (log / email admin / kill / restart / replica); health monitoring detects crashed/unresponsive compartments. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — GPL-2.0** — the npm package `breakapp` (the installable artifact the paper points to via `npm install -g breakapp`) declares **GPL-2.0**. _Note:_ the wrapped core `@andromeda/breakapp` and the author's `nvasilakis/breakapp` wrapper repo carry no separate license field. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** prototype (NDSS 2018), evaluated on real npm apps (Ghost, etc.). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — composes with OS sandboxing (SELinux/AppArmor/Docker) as the LXC/PROC substrate; complementary to package-auditing services (which can drive `TRUST`/`DOUBT` choices). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — a single BreakApp instance owns the module system; later imports of BreakApp are no-ops returning the singleton, avoiding version conflicts. |
| Can stack effectively | ✅ / ❌ | **✅** — hundreds–thousands of compartments per app; nested module boundaries handled (class-hierarchy redirection avoids nested boundary-crossings). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **RPC** — function/method calls across boundaries become generated RPC stubs (sync calls block; async register continuations) over the chosen `IPC` channel. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Serialization / message passing** — primitives are deep-copied (with change propagation); objects are remote-referenced via an ID→pointer translation table; explicit `CONTEXT` can share selected pointers/globals. |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — synchronous blocking calls yield to the module scheduler; asynchronous calls register continuations; message ordering preserved via sequence numbers. |
| Interaction security/validation | [Describe] | All cross-boundary traffic mediated by BreakApp; constructors/`new`, reference-equality, GC events handled specially; `CONTEXT` is the explicit sharing whitelist; channels can be `ENCRYPT`-ed. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Module/Library** — at `import` boundaries (package- or file-level). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Runtime (automatic at module boundaries) + Programmer (optional policies)** — boundaries are discovered automatically; the developer optionally tunes via `LEVEL`/`GROUP`/`TRUST`/`DOUBT`. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — compartments spawned dynamically at `import`; policies generated at runtime (from env/args/preprocessing); replicas added on demand; future work explores dynamic split/coalesce. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Other — module scheduler + event-driven** — synchronous cross-boundary calls yield to a module scheduler; built on Node's cooperative-concurrency/event-loop model. |
| Process model | fork/exec / Custom / N/A | **Configurable** — SBX (no new process), PROC (new OS process), LXC (container); a non-blocking-I/O vs blocking-`require` conflict is resolved by polling for the child's channel file. |
| POSIX compatibility | Full / Partial / Limited | **N/A / language-level** — BreakApp operates at the language module layer, not the POSIX API; underlying boxes run on stock POSIX OS. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | LXC startup is very expensive (2220×/compartment); SBX shares heap/event-queue (weaker isolation, stack/timing leakage); replication risks state inconsistency for stateful modules; policy **conflicts** between library and app authors need `COMPOSE` resolution. |
| Security caveats when layered | [Describe] | Trusts the whole language runtime + built-ins; package-manager (install-script) attacks are **out of scope**; native/unsafe C modules nullify language safety unless boxed in PROC; security depends on the user choosing appropriate policies for their threat model. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server / Cloud** — Node.js server applications with deep third-party dependency trees (blogging platforms, web servers). |
| Deployment scale | Single device / Cluster / Large scale | **Single device** (per-application process tree); not framed as a scale dimension. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **Good** — BreakApp interposes on inter-compartment communication, tracking load/call-frequency per channel and monitoring compartment health, taking curative actions (restart/kill/spawn). _(Stated in paper → reported, not N/A.)_ |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator. _(Design intent: zero-config defaults, optional policy tuning.)_ |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator. |

---

## Summary

> **What it is:** A drop-in replacement for a language runtime's **module system** that automatically turns `import` boundaries into isolation boundaries. At each import BreakApp can spawn the module into its own compartment, rewrite its interface into RPC proxies, and transparently preserve single-runtime semantics (references, GC, exceptions, streams) — so existing apps get compartmentalization with **no annotations, no rewrites, no pre-runs**. A rich per-import policy language (BOX/IPC/CONTEXT/LEVEL/TRUST/DOUBT/REPLICAS/ON_FAIL/TIMER…) tunes the security↔performance trade-off, including the **box type**: in-runtime sandbox (SBX), OS process (PROC), or container (LXC).
>
> **Who it's for:** Developers of applications built from large, untrusted third-party dependency trees (the paper's exemplar: Node.js/npm, where third-party code can be ~99.9% of shipped code) who want to retrofit least-privilege isolation cheaply.
>
> **What it protects:** The application and its modules from malicious/buggy third-party modules — blocking global-state reads/writes, module-cache tampering, built-in patching, stack inspection, env/fs/net exfiltration, and DoS — with the strength dialed by box type.
>
> **What it costs (effort/money/performance):** Near-zero developer effort; per-compartment startup of 1.5 ms (SBX) / 72 ms (PROC) / 666 ms (LXC); cross-boundary calls of ns (in-runtime) to tens of ms (TCP); but only 2–15% of real HTTP-app latency since I/O dominates.
>
> **What it needs (hardware/OS/expertise):** Commodity hardware, a stock OS + the language runtime (Node/V8), Docker for LXC — and that BreakApp loads first and the runtime/built-ins are trusted.
>
> **Key tradeoffs:** Uniquely automates compartmentalization at module granularity with a flexible, decoupled policy layer (security assumptions at *use* time, not *develop* time) — but the whole language runtime is trusted, isolation strength and cost are coupled to the chosen box (SBX cheap/weak ↔ LXC strong/expensive), package-manager attacks and most side channels are out of scope, and security depends on the user picking the right policies.
>
> **Additional Notes:** A full compartmentalization **System** (not a baseline), distinctive for **automation + flexibility** rather than a single fixed mechanism — it can *use* sandboxes, processes (Firecracker-style is the heavier analogue), or containers (Gvisor-adjacent) as its substrate. Conceptual sibling of SOAAP/Wedge (assisted decomposition) but automated; module-boundary focus contrasts with the thread-level memory systems Smv/Arbiter and the MPK systems Erim/Cubicleos. Built on the authors' Andromeda runtime + Node.js.
