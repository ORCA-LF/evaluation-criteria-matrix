# Compartmentalization Evaluation Matrix — gVisor

**System:** gVisor (Google's userspace application kernel / container sandbox; compartment = a **sandboxed container**, guarded by the Sentry)
**Paper:** *None* — gVisor has no single flagship paper. Evaluated from official docs (gvisor.dev architecture/security guides), the gVisor security-basics blog, and the GitHub repo.
**Source file:** N/A (web research per CLAUDE.md "REQUESTED" instruction — no PDF)
**Matrix version:** v0.1
**Evaluated:** 2026-06-10

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **[docs]** = gvisor.dev documentation · **[repo]** = GitHub repo · **_Inferred:_** = evaluator assessment · **⚠️ Unclear** = not found, needs research. (No **(§N)** citations — there is no paper.)
>
> _Compartment = a **sandboxed container**: the untrusted workload runs against the **Sentry**, a from-scratch userspace re-implementation of the Linux kernel ABI written in Go. The Sentry intercepts every application syscall and implements it in userspace; a separate **Gofer** process mediates filesystem access. Both are confined to the host by seccomp-bpf._
>
> _Entry type: **Baseline / container-isolation** (a userspace-kernel sandbox — the "secure container runtime" reference point), runnable via the `runsc` OCI runtime under Docker/Kubernetes._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — sandbox untrusted container workloads so they cannot exploit the host kernel; "limits the host kernel surface accessible to the application" [docs][repo]. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a whole container/workload [docs]. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Container** (process group) — one sandbox per container; ≈"Process/Container" granularity [docs]. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is whole-sandbox, not data-object granular. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **Mostly None / optional virtualization** — default interception is software (Systrap/seccomp-bpf or ptrace); the optional **KVM platform** uses hardware virtualization to let the Sentry act as guest-kernel + VMM [docs]. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — userspace kernel re-implementation in a memory-safe language (Go)** + **syscall interception** (Systrap/ptrace/KVM platforms) + **seccomp-bpf** confinement of the Sentry/Gofer [docs]. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Container** sandbox, implemented as host processes (Sentry, Gofer) with restricted privileges; "similar to virtual machines… two separate kernels" but without a full guest OS [docs]. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the **Sentry** (guest kernel in userspace, ≈Linux syscall ABI in Go), the **Gofer** (filesystem proxy), and the **`runsc`** OCI runtime; a syscall-interception platform (Systrap default / ptrace / KVM) [docs]. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **No syscall is passed through to the host** — every supported call has an independent Sentry implementation; the Sentry cannot open new files / create sockets / etc. on the host (the Gofer mediates files). Two confinement layers: the workload is trapped by the Sentry, and the Sentry itself is confined by seccomp-bpf to a small host-syscall set. Defense-in-depth engineering rules: only universal functionality implemented; unsafe code quarantined to `unsafe.go`; no CGo (pure-Go binary) [docs][blog]. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | _Inferred:_ **✅** — each container runs in its own Sentry/sandbox; a crash is confined to that sandbox (one sandbox per container) [docs]. (Not stated as a recovery experiment.) |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the Sentry is a complete mediation point: it parses/validates every syscall and its arguments in userspace before deciding whether any (heavily restricted) host action occurs; the Gofer validates all filesystem access [docs]. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The Sentry + Gofer (Go), the `runsc` runtime, the host Linux kernel (small reduced surface still reachable), CPU/firmware.** The host kernel remains in the TCB but its exposed surface is deliberately minimized vs a normal container [docs]. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Medium** — the Sentry is a substantial from-scratch kernel re-implementation (exposes ≈**240 syscalls** to the workload, of which ≈**100** can reach the host) but written in memory-safe Go; the Sentry's *own* host footprint is restricted to ≈**70 syscalls** via seccomp, the Gofer even fewer. ⚠️ No authoritative LOC figure for the TCB. [docs][blog] |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None (explicitly out of scope)** — gVisor "does not defend against hardware side channels"; side-channel/network attacks are outside its primary defense scope [docs]. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal proofs; relies on Go memory safety, the structural "no passthrough" design, and extensive syscall-compatibility testing [docs]. |
| Experimental validation available | Yes (specify) / No | **Yes (engineering, not a paper)** — published performance guide (syscall latency, memory density, network/IO benchmarks vs runc) and a large open syscall test suite; widely deployed in Google Cloud (GKE Sandbox / Cloud Run) [docs][repo]. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Docs frame overhead as **structural** (syscall interception, Sentry memory) vs **implementation** (network stack, VFS). Rule of thumb: "small operations impose a large overhead… the more work an application does the smaller the impact of structural costs." CPU-bound code with few syscalls runs near-native [docs]. |
| Domain switch cost | lmbench lat_ctx (platform/native) | ⚠️ **No lat_ctx figure.** Every intercepted syscall pays a userspace round-trip into the Sentry; magnitude depends heavily on platform (ptrace slowest; Systrap/KVM faster). Not quantified in lmbench terms [docs]. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | ⚠️ **No precise figure.** Container startup carries overhead, but "most of the time overhead… is associated [with] Docker itself"; using `runsc do` / the OCI runtime directly reduces it. No ms value given [docs]. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | ⚠️ **Not reported as lat_pipe.** In-sandbox IPC is Sentry-implemented; cross-sandbox is ordinary networking. No isolated latency figure [docs]. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | ⚠️ **Not bw_pipe.** Network throughput (iperf) underperforms runc, especially for "minimal work per request" synthetic benchmarks — "mostly bound by implementation costs" of the userspace netstack [docs]. |
| Memory overhead per domain | MB/domain | **"A small, mostly fixed amount of memory overhead" per sandbox** (Sentry process), plus per-instance cost shown in Node/Ruby density benchmarks vs runc. ⚠️ No single authoritative MB/sandbox number [docs]. |
| Domain count bounded by | Limiting factor; approx. domain count | _Inferred:_ **host memory + per-sandbox Sentry overhead** (one Sentry+Gofer process set per container). No explicit max count [docs]. |
| Performance scales with domain count | Big O | _Inferred:_ memory **O(n)** in sandbox count (fixed-ish Sentry overhead each); no asymptotic statement in docs. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — builds/runs on x86-64 and ARM64; software platforms (Systrap/ptrace) need no virtualization; the optional KVM platform uses commodity VT-x [docs][repo]. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock Linux** — runs on an unmodified host kernel as userspace processes (no kernel module); Systrap/ptrace use standard seccomp/ptrace, KVM uses the stock KVM subsystem [docs]. |
| What privileges does it need? | User / Root / Kernel access | _Inferred:_ container-runtime privileges to set up namespaces/seccomp (typical OCI runtime install); the sandbox itself runs **unprivileged** with dropped capabilities [docs]. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | Install `runsc` as an OCI/Docker runtime or enable GKE Sandbox; no guest image or custom libc — the app runs against the Sentry's Linux ABI [docs]. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None** — runs unmodified container images against the Sentry's reimplemented Linux ABI [docs]. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — unmodified Linux binaries, subject to compatibility gaps (the Sentry implements a large but not 100% subset of Linux; some syscalls/`/proc`/ioctls/raw sockets are unimplemented by design) [docs]. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any** — language-agnostic; anything that runs as a Linux container, within ABI-coverage limits. (gVisor *itself* is written in Go.) [docs][repo] |
| Other compatibility notes | [Describe] | Specialized/less-universal APIs are deliberately omitted (raw sockets, many ioctls); apps needing those may not run. The netstack and VFS are reimplemented, so behavior can differ subtly from the host kernel [docs]. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **~Zero for the workload** — drop-in `runsc` runtime, no app changes (compatibility permitting) [docs]. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | No app rebuild; operators install the `runsc` binary and register it as a runtime [docs]. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(Docs do mention `runsc` debug logging/strace-like tooling, but no explicit rating.)_ |
| Failure modes visibility | [crashes/logs/error codes/silent] | Unimplemented syscalls return errors (e.g. `ENOSYS`) rather than passing to the host; `runsc` provides debug logs of intercepted syscalls [docs]. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — Apache 2.0** [repo]. |
| Primary usage | Production / Research / Internal / Experimental / Other | **Production** — Google's container sandbox, used in GKE Sandbox, Cloud Run, and App Engine; actively maintained open-source project [docs][repo]. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — implements the OCI runtime interface (`runsc`), so it composes with Docker, containerd, and Kubernetes/GKE as a drop-in secure runtime; can layer on Linux namespaces/cgroups/seccomp [docs][repo]. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — runs side-by-side with `runc` and other runtimes on the same host; per-container choice of runtime [docs]. |
| Can stack effectively | ✅ / ❌ | _Inferred:_ **✅** — many independent sandboxes coexist; can run inside a VM (Systrap platform is explicitly "well-suited to run inside a virtual machine") [docs]. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Networking / IPC** — separate sandboxes communicate like separate containers (network sockets); within a sandbox the Sentry provides the syscall/IPC surface. The Sentry↔Gofer link is a dedicated socket [docs]. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Message passing / networking** between sandboxes; filesystem sharing is mediated by the **Gofer** over a socket; the sandbox starts in an empty mount namespace [docs]. |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Both** — synchronous syscall emulation + async I/O/networking; not explicitly characterized [docs]. |
| Interaction security/validation | [Describe] | All workload→host interaction is mediated and re-implemented by the Sentry; all filesystem access is mediated by the Gofer; both confined by seccomp [docs]. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Container** (per-sandbox) [docs]. |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Runtime** — the container runtime/orchestrator decides what runs in a gVisor sandbox; the developer ships an ordinary container image [docs]. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — sandboxes are created/destroyed per container on demand by the orchestrator [docs]. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Standard** (from the app's view) — the Sentry implements Linux threading; underlying scheduling differs by platform [docs]. |
| Process model | fork/exec / Custom / N/A | **Standard fork/exec** semantics, re-implemented inside the Sentry [docs]. |
| POSIX compatibility | Full / Partial / Limited | **Partial** — a large but incomplete subset of the Linux/POSIX ABI; deliberately omits some syscalls/ioctls/raw-sockets and exposes a reduced `/proc` [docs]. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | ABI-coverage gaps (some syscalls/ioctls unimplemented); reimplemented netstack/VFS impose throughput/latency costs; system-call-heavy and IO/network-heavy workloads see the largest overhead [docs]. |
| Security caveats when layered | [Describe] | Hardware side channels explicitly **out of scope**; the host kernel surface is reduced but not eliminated (≈100 of the Sentry's syscalls can still reach the host) — a kernel bug in that residual surface remains reachable [docs][blog]. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud / Server** — sandboxing untrusted containers in multi-tenant cloud (GKE Sandbox, Cloud Run, App Engine) [docs][repo]. |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — deployed across Google Cloud's container platforms [docs]. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **Good** — OCI/`runsc` runtime integrates with Kubernetes/Docker/containerd monitoring & orchestration; runtime-level metrics/debug logs available [docs]. _(Stated via OCI integration, so reported rather than N/A.)_ |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |

---

## Summary

> **What it is:** A userspace "application kernel" for containers: gVisor's **Sentry** re-implements the Linux syscall ABI from scratch in memory-safe Go and intercepts every syscall an untrusted container makes, so the workload talks to the Sentry instead of the host kernel. A separate **Gofer** mediates filesystem access, and both are confined to the host by seccomp-bpf. It ships as the `runsc` OCI runtime.
>
> **Who it's for:** Cloud/container operators who want a stronger isolation boundary than plain Linux containers (runc) for untrusted workloads, without paying for a full guest kernel/VM per container.
>
> **What it protects:** The host kernel (and other tenants) from untrusted container code — by ensuring no syscall is passed straight through, drastically shrinking the reachable host-kernel attack surface (the Sentry itself is limited to ~70 host syscalls; ~100 of the ~240 it exposes can reach the host).
>
> **What it costs (effort/money/performance):** Near-zero porting effort (drop-in runtime) but real runtime overhead — every syscall is intercepted, and the reimplemented netstack/VFS make syscall-heavy and IO/network-heavy workloads noticeably slower than runc; compute-bound workloads run near-native. A small, mostly fixed memory overhead per sandbox.
>
> **What it needs (hardware/OS/expertise):** Commodity x86-64/ARM64, a stock Linux host (no kernel module), and the `runsc` runtime — no application changes, within ABI-compatibility limits.
>
> **Key tradeoffs:** Stronger isolation than runc with full container-image compatibility and no special hardware, at the price of per-syscall interception overhead, partial Linux ABI coverage, and **no defense against hardware side channels**. It sits between plain containers and full VMs ([[firecracker]]): a userspace kernel rather than a hardware-virtualized guest.
>
> **Additional Notes:** This is a **baseline / container-isolation** reference entry (a secure container runtime), not an intra-process compartmentalization technique. No flagship paper — evaluated entirely from gvisor.dev docs, the security-basics blog, and the repo; several efficiency cells (lat_ctx, MB/sandbox, TCB-LOC) are genuinely unreported as hard numbers. Contrast with [[firecracker]] (the VM-based baseline) — both were requested as the non-intra-process reference points for the matrix.

---

## Sources

- **gVisor docs [docs]:** Security Model (gvisor.dev/docs/architecture_guide/security/), Performance Guide (…/performance/), Introduction (…/intro/), platform docs.
- **Blog [blog]:** "gVisor Security Basics — Part 1" (gvisor.dev/blog/2019/11/18/…) — syscall-count figures (~240 exposed, ~100 reach host, ~70 Sentry host syscalls).
- **Repo [repo]:** github.com/google/gvisor — Apache 2.0; written in Go; x86-64/ARM64; `runsc` OCI runtime; production use (GKE Sandbox, Cloud Run, App Engine).
- **No flagship paper** — evaluated via web research per CLAUDE.md "REQUESTED" instruction.

### Cells flagged for further research
- **Efficiency › Domain switch (lat_ctx), Domain creation, Inter-domain call/throughput, Memory MB/sandbox** — ⚠️ no hard numbers in docs (described qualitatively / platform-dependent).
- **Security › TCB approximate size** — ⚠️ no authoritative LOC; syscall counts (~240/~100/~70) given instead.
- **Usability › Required expertise, Debugging support** + **Sys Design › Config complexity, Maintenance burden** — N/A (self-report policy 2026-06-09).
- Inferences marked `_Inferred:_` inline: crash isolation, privileges, memory Big-O, stacking, interaction semantics.
