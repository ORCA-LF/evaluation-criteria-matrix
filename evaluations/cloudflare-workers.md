# Compartmentalization Evaluation Matrix — Cloudflare Workers (V8 Isolates)

**System:** Cloudflare Workers (edge serverless on V8 isolates; compartment = a **V8 isolate**, one per tenant Worker)
**Paper:** *None* — no flagship paper. Evaluated from Cloudflare's official security-model docs, the "Mitigating Spectre" engineering blog, and the open-source `workerd` runtime.
**Source file:** N/A (web research — no PDF)
**Matrix version:** v0.1
**Evaluated:** 2026-06-10

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **[docs]** = developers.cloudflare.com/workers/reference/security-model · **[blog]** = blog.cloudflare.com (Spectre/security-model, sandbox-hardening) · **[repo]** = github.com/cloudflare/workerd · **_Inferred:_** = evaluator assessment · **⚠️ Unclear** = not found. (No **(§N)** — there is no paper.)
>
> _Compartment = a **V8 isolate**: a JavaScript/WebAssembly sandbox with its own private heap. "V8 executes code inside isolates, which prevent that code from accessing memory outside the isolate — even within the same process," so thousands of tenant isolates share one OS process and switch thousands of times/sec [docs][blog]._
>
> _Entry type: **Baseline / language-VM isolation** — the third isolation category (alongside the [[firecracker]] VM baseline and the [[gvisor]] container baseline). This is the "language VM isolation" point the Firecracker paper contrasts against (V8 isolates / JVM-style)._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — safely co-locate thousands of mutually-distrusting tenant Workers in one process on every edge machine [docs][blog]. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a tenant's Worker code running in its own isolate [docs]. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Function / isolate** — one isolate per Worker (per tenant deployment) [docs]. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is per-isolate (private heap), not data-object granular. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **MPK/PKU** — Cloudflare deploys internal V8 modifications using **memory protection keys**, giving each isolate a random key protecting its V8 heap data (a hardening layer on top of the software isolate boundary) [blog]. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Language runtime (V8 isolate)** as the primary boundary — managed JS/WASM execution; plus a **second-layer process sandbox** (Linux namespaces + seccomp) and curated APIs [docs][blog]. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Intra-Address Space Domain** — many isolates inside one shared OS process, each with a private heap; wrapped by a process-level sandbox [docs][blog]. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the V8 engine + the Workers runtime (`workerd`), the layer-2 process sandbox, the inbound/outbound proxy services, and the supervisor that manages cordons [docs][repo]. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Memory isolation between tenants** (isolates cannot read outside their heap; MPK-enforced); **no filesystem access** (layer-2 sandbox blocks "absolutely all filesystem-related system calls," empty filesystem); **no raw network** (outbound only via proxies that block internal-network access); **no native code** (JS/WASM only — no `RDTSC`/`CLFLUSH`); **trust-based cordons** keep e.g. Free-plan tenants out of Enterprise processes [docs][blog]. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | _Inferred:_ **✅** — an isolate exception/OOM is confined to that isolate; the runtime hosts thousands independently and can reschedule/restart. (Caveat: an isolate-escape or V8 RCE would threaten co-tenants in the same process — hence the layered defenses.) [docs] |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — "Cloudflare APIs define exactly what a Worker can and cannot do"; all I/O is mediated by curated runtime APIs and proxy services (no ambient filesystem/network) [docs]. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **V8 (the JS/WASM engine) + the `workerd` runtime + the host OS kernel + CPU/firmware.** V8 is the critical TCB component — an isolate boundary is only as strong as V8; the layer-2 sandbox + MPK + cordons exist precisely because V8 is a large, exposed TCB [docs][blog]. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large** — V8 is a very large, complex JIT engine (millions of LOC) in the TCB; mitigated operationally by a **<24-hour patch gap** for V8 security fixes. ⚠️ No isolated LOC figure for the Workers-specific TCB. [docs][blog] |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **Timing + Speculative (Spectre) — partial, defense-in-depth.** Mitigations: `Date.now()` frozen during execution + **no other timers** + **no concurrency/multithreading** (can't build a timer) [docs]; **dynamic process isolation** (Workers with suspicious CPU metrics rescheduled into isolated processes); **MPK** isolate-heap keys; periodic memory reshuffling/restarts; JS/WASM-only blocks native timing instructions. Cloudflare states Spectre "does not have an official solution… not even when using heavyweight virtual machines" — goal is to make attacks "so slow as to be uninteresting," not impossible [blog]. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal proofs; relies on V8's memory safety, layered sandboxing, and operational defenses [docs][blog]. |
| Experimental validation available | Yes (specify) / No | **Yes (engineering/operational, not a paper)** — production validation at global scale (Workers across 200+ locations, thousands of tenants/machine); public security-model writeups and the open-source `workerd` runtime; documented response to a real remote-Spectre research attack [docs][blog][repo]. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC** (managed JS/WASM platform). Strict per-tenant *process* isolation was measured at **~10× CPU cost** vs shared-process isolates — the core reason isolates are used instead [blog]. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Very low (by design), not lmbench.** Switching between isolates in one process is far cheaper than an OS context switch — the runtime "rapidly switches between guests thousands of times per second with minimal overhead." No µs figure [docs][blog]. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Isolate startup ≈ "under 5 ms"** (no per-process fork/exec) — effectively eliminates cold starts; "zero cold start" is a pre-warmed isolate. ⚠️ 5ms figure is from secondary sources; Cloudflare's own docs give it qualitatively [docs][secondary]. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **N/A — no fast cross-isolate call.** Isolates don't directly invoke each other; cross-Worker calls go via runtime bindings / service bindings / RPC over the platform, not a shared-memory pipe [docs]. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | ⚠️ **Not bw_pipe.** No shared-memory IPC between tenant isolates (would break isolation); no bandwidth figure [docs]. |
| Memory overhead per domain | MB/domain | **"A couple megabytes" per isolate** — "each guest cannot take more than a couple megabytes of memory"; isolates use "an order of magnitude less memory than a Node process." (Commonly cited as ~3MB.) [blog] |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: per-machine memory (couple-MB/isolate) + the V8/process model.** "Thousands of active tenants per machine"; a single runtime instance runs "hundreds or thousands of isolates" [docs][blog]. |
| Performance scales with domain count | Big O | _Inferred:_ memory **O(n)** in isolate count (couple-MB each); CPU is cooperatively scheduled within a process; no asymptotic statement [docs]. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** x86-64 edge servers; MPK hardening uses commodity Intel MPK [blog]. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock Linux** (namespaces + seccomp) running a **modified V8** (internal MPK patches). The OSS runtime `workerd` runs on stock Linux [docs][repo]. |
| What privileges does it need? | User / Root / Kernel access | _Inferred:_ Cloudflare-operated platform; tenants need no host privileges (they upload code). Self-hosting `workerd` needs normal user/container privileges to set up the sandbox [docs][repo]. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | **Cloudflare-hosted** (managed) — tenants deploy via Wrangler/the API; or self-host the open-source `workerd` runtime [docs][repo]. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **API changes / rewrite-to-platform** — code must target the Workers runtime API (Web/Service-Worker + Cloudflare bindings), not full Node.js/POSIX; existing server apps generally need adaptation [docs]. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **No native binaries** — JS/TS or anything compiled to **WebAssembly**; no arbitrary Linux binaries (a deliberate Spectre mitigation: no native code) [docs][blog]. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Managed: JavaScript/TypeScript + WebAssembly** (so Rust/C/C++/Go via WASM); not native execution [docs][blog]. |
| Other compatibility notes | [Describe] | No filesystem, no raw sockets, no threads/shared-memory concurrency, no native timers; partial Node.js-compat shims exist but it is not a full Node/POSIX environment [docs]. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | _Inferred:_ **Low for greenfield edge functions; moderate-to-high to port existing Node/POSIX apps** (API surface differs). No published figure [docs]. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | Deploy via Wrangler (bundles JS/WASM); no native build/link step for the tenant [docs]. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Failure modes visibility | [crashes/logs/error codes/silent] | Runtime exceptions surface as JS errors/HTTP 5xx; platform logs/observability (tail logs) available; CPU-limit overruns terminate the request [docs]. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Hybrid** — the platform is **commercial/closed (Cloudflare-operated)**; the runtime **`workerd` is open source (Apache 2.0)** [repo]. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Production** — Cloudflare's global edge serverless platform serving large-scale multi-tenant traffic [docs][blog]. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — isolates (layer 1) are deliberately composed with a process sandbox (namespaces/seccomp, layer 2) + MPK + cordons; isolates can host WASM sandboxes within [docs][blog]. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — thousands of isolates coexist per process; cordons place different trust tiers in different processes; Cloudflare also offers separate VM/container products alongside [docs][blog]. |
| Can stack effectively | ✅ / ❌ | **✅** — many isolates per process is the model; WASM nesting inside an isolate works [docs]. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **RPC / service bindings** — Workers invoke each other via platform bindings (service bindings, Durable Objects, queues), not direct in-memory calls [docs]. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Message passing / serialization** over platform primitives (KV, R2, D1, Durable Objects, queues); **no shared memory** between tenant isolates (would break isolation) [docs]. |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Both** — synchronous RPC-style service bindings + asynchronous queues/events; not framed this way in docs [docs]. |
| Interaction security/validation | [Describe] | All cross-Worker and external I/O is mediated by curated runtime APIs and inbound/outbound proxies that validate destinations [docs]. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Service / function** — one Worker (isolate) per deployable unit [docs]. |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer + Runtime** — the developer defines Workers; the runtime instantiates and schedules isolates and assigns cordons [docs]. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — isolates are created/destroyed on demand; Workers dynamically rescheduled into isolated processes when risk is detected [docs][blog]. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Other — no tenant concurrency**: multithreading is deliberately prohibited (a Spectre mitigation); execution is single-threaded per isolate [docs][blog]. |
| Process model | fork/exec / Custom / N/A | **Custom** — isolates within a shared process; no per-tenant fork/exec (that's the rejected 10×-cost model) [blog]. |
| POSIX compatibility | Full / Partial / Limited | **Limited** — no filesystem, no raw sockets, no threads; a Web-platform + bindings API, not POSIX [docs]. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | No shared memory / no threads between tenants; no native code; API surface ≠ Node/POSIX; cross-Worker communication only via platform primitives [docs]. |
| Security caveats when layered | [Describe] | The whole stack shares one V8/process TCB per cordon — a V8 isolate-escape or 0-day would threaten co-tenants until the <24h patch lands; Spectre is mitigated, not eliminated [docs][blog]. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud / Edge** — serverless functions at the network edge (200+ locations) [docs][blog]. |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — global multi-tenant edge fleet, thousands of tenants/machine [docs][blog]. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(Platform does provide tail logs/analytics, but no rating asserted.)_ |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |

---

## Summary

> **What it is:** Cloudflare's edge serverless platform isolates tenants using **V8 isolates** — lightweight JS/WASM sandboxes with private heaps that share a single OS process — so thousands of mutually-distrusting Workers run per machine and switch thousands of times/sec, with ~5ms startup and a couple MB each. It is wrapped in a second-layer process sandbox (namespaces + seccomp), MPK heap keys, trust-tier cordons, and Spectre defenses.
>
> **Who it's for:** Edge/serverless developers wanting near-zero cold start and global distribution, and a platform operator who must pack enormous numbers of tiny untrusted tenants per host economically.
>
> **What it protects:** Co-tenants from each other and the host from tenant code — via V8 memory isolation (MPK-hardened), a curated no-filesystem/no-raw-network API, and process-level sandboxing — while explicitly accepting that Spectre is mitigated, not solved.
>
> **What it costs (effort/money/performance):** Lowest overhead of the three baselines (couple-MB/isolate, ~5ms start, sub-OS-context-switch cost; strict per-process isolation would cost ~10× CPU) — but the tightest compatibility constraints: JS/WASM only, no native binaries, no threads, no filesystem, a non-POSIX API.
>
> **What it needs (hardware/OS/expertise):** Commodity x86-64 with MPK, stock Linux, a (modified) V8 / the `workerd` runtime; tenants write to the Workers API (managed) — no host privileges.
>
> **Key tradeoffs:** Extreme density and the weakest-isolation/lowest-overhead end of the spectrum, bought by trusting a very large shared V8 TCB and by heavy operational defenses (24h patch gap, dynamic process isolation, timing lockdown, cordons). The compatibility cost (managed JS/WASM, no native code, no POSIX) is the inverse of the VM/container baselines.
>
> **Additional Notes:** This is a **baseline / language-VM isolation** entry — the third reference point completing the trio with [[firecracker]] (hardware-VM) and [[gvisor]] (userspace-kernel container). Maps directly onto the "language VM isolation" category the Firecracker paper rejected for Lambda (needs arbitrary binaries) — useful as the density-vs-isolation extreme. No flagship paper; several efficiency cells (exact startup ms, MB/isolate, TCB-LOC) are reported only qualitatively by Cloudflare. The runtime `workerd` is open-source (Apache 2.0); the platform itself is commercial.

---

## Sources

- **[docs]** Cloudflare Workers — Security model: developers.cloudflare.com/workers/reference/security-model/ ; How Workers works: …/how-workers-works/
- **[blog]** "Mitigating Spectre and Other Security Threats: The Cloudflare Workers Security Model" (blog.cloudflare.com) — 10× process-isolation CPU cost, couple-MB/isolate, dynamic process isolation, MPK, Date.now() freeze, no-concurrency; "Safe in the sandbox: security hardening for Cloudflare Workers."
- **[repo]** github.com/cloudflare/workerd — the open-source Workers runtime, Apache 2.0.
- **[secondary]** ~5ms isolate startup / "~10× less memory than Node" figures (federicocalo.dev, third-party explainers) — corroborating Cloudflare's qualitative claims; flagged where used.
- **No flagship paper** — evaluated via web research.

### Cells flagged for further research
- **Efficiency › Domain creation (≈5ms)** — secondary-sourced; Cloudflare states qualitatively.
- **Efficiency › Domain switch (lat_ctx), Inter-domain call/throughput** — no numeric figures (sub-context-switch / no IPC by design).
- **Security › TCB approximate size** — V8 is "Large" but no isolated LOC figure.
- **Usability › Required expertise, Debugging** + **Sys Design › Monitoring, Config complexity, Maintenance burden** — N/A (self-report policy 2026-06-09).
- Inferences marked `_Inferred:_` inline: crash isolation, privileges, porting effort, memory Big-O, interaction semantics.
