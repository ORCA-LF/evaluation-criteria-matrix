An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the system's authors and maintainers. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/cloudflare-workers.md)

## What it is

Cloudflare Workers is an edge serverless platform that co-locates thousands of mutually-distrusting tenant Workers in a single OS process on each edge machine. There is no flagship paper, so this entry was built from Cloudflare's official security-model docs (developers.cloudflare.com/workers/reference/security-model), the "Mitigating Spectre" engineering blog (blog.cloudflare.com), and the open-source `workerd` runtime repo. The isolation problem it addresses is packing enormous numbers of small untrusted tenants per host economically while keeping them from reading each other's memory or reaching the host.

- Compartment is a V8 isolate: a JavaScript/WebAssembly sandbox with its own private heap, one isolate per tenant Worker, with thousands of isolates sharing one OS process and switching thousands of times per second.
- Mechanism: the primary boundary is the V8 language runtime (managed JS/WASM execution); this is hardened with Intel MPK/PKU keys protecting each isolate's V8 heap, a second-layer process sandbox (Linux namespaces plus seccomp), curated runtime APIs, and trust-tier cordons.
- Trust model: the TCB includes V8 (the JS/WASM engine), the `workerd` runtime, the host OS kernel, and the CPU/firmware. V8 is the critical and largest component. Tenant Worker code is untrusted; cross-tenant memory access is prevented by the isolate boundary plus MPK, and there is no filesystem, no raw network, and no native code.

## Key takeaways

- This is the lowest-overhead, weakest-isolation end of the baseline trio (with Firecracker for hardware VMs and gVisor for userspace-kernel containers): roughly a couple of megabytes per isolate, isolate startup around 5 ms, and inter-isolate switching cheaper than an OS context switch. Strict per-tenant process isolation was measured at about 10x the CPU cost, which is the reason isolates are used instead.
- The density comes at a steep compatibility cost: JS/TypeScript plus WebAssembly only, no native binaries, no threads or shared-memory concurrency, no filesystem, no raw sockets, and a Web-platform-plus-bindings API rather than POSIX.
- Side-channel handling is defense-in-depth rather than a guarantee. Spectre is mitigated (frozen `Date.now()`, no other timers, no concurrency, dynamic process isolation for suspicious Workers, MPK heap keys, periodic memory reshuffling) but Cloudflare explicitly states the goal is to make attacks too slow to be interesting, not impossible.
- The TCB is large (V8 is millions of LOC) and shared across co-tenants in a cordon, mitigated operationally by a sub-24-hour patch gap for V8 security fixes. A V8 isolate escape would threaten co-tenants until that patch lands.

## Where I need review

Because this entry is built from docs, a blog, and the repo rather than a paper, more cells than usual are qualitative or inferred. Please check these in particular:

- Domain creation cost (around 5 ms isolate startup): sourced from secondary third-party explainers; Cloudflare states it only qualitatively.
- Domain switch cost, inter-domain call latency, and inter-domain throughput: no numeric figures. Switching is described as sub-context-switch by design, and there is no shared-memory IPC between tenant isolates, so the lmbench-style metrics do not apply.
- TCB approximate size: marked "Large" on the basis that V8 is a large JIT engine, but there is no isolated LOC figure for the Workers-specific TCB.
- Compartment crashes isolated (_Inferred:_ yes): an isolate exception or OOM is confined to that isolate, with the caveat that an isolate escape or V8 RCE would threaten co-tenants.
- Privileges required (_Inferred:_): tenants need no host privileges on the managed platform; self-hosting `workerd` needs normal user/container privileges.
- Porting effort (_Inferred:_): low for greenfield edge functions, moderate-to-high to port existing Node/POSIX apps; no published figure.
- Performance scales with domain count (_Inferred:_): memory O(n) in isolate count; no asymptotic statement in the sources.
- Interaction semantics (_Inferred:_ both synchronous and asynchronous): synchronous service bindings plus asynchronous queues/events, though the docs do not frame it this way.
- Required expertise, Debugging support, Monitoring/orchestration, Initial configuration complexity, and Ongoing maintenance burden: all marked N/A under the self-report/operational policy, since an external evaluator cannot assess these.

License: Hybrid. The platform is commercial/closed (Cloudflare-operated), while the `workerd` runtime is open source under Apache 2.0 (source: github.com/cloudflare/workerd repo).
