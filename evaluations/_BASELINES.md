# Baseline Comparison — Isolation Trio (VM / Container / Language-VM)

**Matrix version:** v0.1 · **Compiled:** 2026-06-10

> Side-by-side of the three **baseline / non-intra-process** reference points, spanning the isolation spectrum. These are the strong-isolation comparators against which the intra-process compartmentalization schemes (MPK/SFI/CHERI/TEE etc.) in `evaluations/` are measured.
>
> Cells here are **condensed** — full citations and caveats live in the per-system files: [[firecracker]], [[gvisor]], [[cloudflare-workers]].
>
> **Sourcing:** Firecracker = NSDI 2020 paper (`papers/firecracker.pdf`); gVisor = gvisor.dev docs + blog + repo; Cloudflare Workers = Cloudflare security-model docs + Spectre blog + `workerd` repo. **(§N)** = Firecracker paper; **[docs]/[blog]/[repo]** = web.

| | **Firecracker** | **gVisor** | **Cloudflare Workers** |
|---|---|---|---|
| **Isolation category** | Hardware VM (KVM) | Userspace kernel (container) | Language VM (V8 isolate) |
| **Compartment** | A MicroVM | A sandboxed container (Sentry-guarded) | A V8 isolate (one tenant Worker) |
| **Position on spectrum** | Strongest isolation, lowest density | Middle | Highest density, weakest isolation |

---

## Security

| **Dimension** | **Firecracker** | **gVisor** | **Cloudflare Workers** |
|---|---|---|---|
| Primary use case | Untrusted component isolation | Untrusted component isolation | Untrusted component isolation |
| Subject selection | Code-centric | Code-centric | Code-centric |
| Code granularity | **VM** | **Container** (process group) | **Function / isolate** |
| Data granularity | N/A | N/A | N/A |
| Hardware primitive | **HW virtualization (VT-x/KVM)** | None default; optional KVM platform | **MPK/PKU** (isolate heap keys) |
| Software technique | Minimal Rust VMM + jailer (seccomp/chroot/ns) | Userspace kernel in Go + syscall interception + seccomp | Language runtime (V8 isolate) + process sandbox |
| Isolation abstraction | **VM** | **Container** (host processes) | **Intra-Address-Space Domain** (isolates in one process) |
| Requires runtime support | ✅ VMM + KVM + jailer | ✅ Sentry + Gofer + runsc | ✅ V8 + workerd + sandbox + proxies |
| Security properties | Boundary moved to HW+tiny VMM; guest kernel untrusted; jailer whitelists 24 syscalls/30 ioctls | No syscall passes through; Sentry mediates ~240 syscalls (~100 reach host); Gofer mediates FS | Per-tenant heap isolation (MPK); no FS, no raw net; JS/WASM only; trust cordons |
| Crash isolated | ✅ (one process/VM) | ✅ _inf_ (one sandbox/container) | ✅ _inf_ (per-isolate; caveat: V8-escape hits co-tenants) |
| Interface safety | ✅ tiny device + KVM surface, seccomp-filtered | ✅ Sentry is full mediation point | ✅ curated APIs, no ambient FS/net |
| TCB includes | CPU, host kernel+KVM, Rust VMM, firmware (**guest kernel excluded**) | Sentry+Gofer (Go), runsc, host kernel (reduced surface), CPU | **V8** (critical), workerd, host kernel, CPU |
| TCB size | **Small (for a VMM)** — ~50k LOC Rust (96% < QEMU) + ~120k KVM | **Medium** — large Go reimpl; ~70 host syscalls; ⚠️ no LOC | **Large** — V8 (millions LOC); ⚠️ no LOC; <24h patch gap |
| Side-channel resistance | **Partial/operational** — deployer applies SMT-off, KPTI, IBPB/IBRS, L1TF, etc.; power/thermal deployer-owned | **None** — explicitly out of scope | **Partial** — Spectre mitigated not solved: frozen Date.now(), no timers/threads, dynamic process isolation, MPK, reshuffle |
| Formal validation | No | No | No |
| Experimental validation | Yes — boot/mem/IO/net vs QEMU & Cloud HV + prod (§5) | Yes — perf guide + syscall test suite + GKE/Cloud Run | Yes (operational) — global prod + documented Spectre-attack response |

---

## Runtime Efficiency

| **Dimension** | **Firecracker** | **gVisor** | **Cloudflare Workers** |
|---|---|---|---|
| General overhead | No SPEC; ~3% mem, negligible CPU | No SPEC; structural (syscall intercept) vs impl; near-native if compute-bound | No SPEC; per-process isolation = ~10× CPU (reason isolates used) |
| Domain switch (lat_ctx) | N/A — host scheduler | ⚠️ no figure; per-syscall userspace round-trip | very low _inf_ — sub-OS-switch, thousands/sec |
| Domain creation | **<125–150ms boot**; ≤150 MicroVMs/s/host | ⚠️ no figure; "mostly Docker itself" | **≈5ms** isolate start (secondary-sourced); near-zero cold start |
| Inter-domain call (lat_pipe) | N/A — cross-VM over TCP/IP | ⚠️ not reported | N/A — no direct call; bindings/RPC |
| Inter-domain throughput | Net ~15 Gb/s; 4k read ~13k IOPS | ⚠️ netstack-bound, underperforms runc | ⚠️ no shared-mem IPC by design |
| Memory per domain | **~3 MB/VM** (Cloud HV ~13, QEMU ~131) | "small, mostly fixed"; ⚠️ no MB figure | **"couple MB"/isolate** (~10× < Node) |
| Domain count bounded by | Host RAM (~3MB/VM); thousands/host | Host mem + Sentry overhead | Per-machine mem (couple-MB); **thousands/machine** |
| Scaling Big-O | Mem O(n) _inf_ | Mem O(n) _inf_ | Mem O(n) _inf_ |

---

## Usability / Adoptability

| **Dimension** | **Firecracker** | **gVisor** | **Cloudflare Workers** |
|---|---|---|---|
| Hardware needed | Commodity x86-64/ARM64 + virt | Commodity x86-64/ARM64 | Commodity x86-64 + MPK |
| OS/kernel needed | Stock Linux + KVM (minimal guest kernel) | Stock Linux (no module) | Stock Linux + modified V8 |
| Privileges | Root/host setup; VMM then sandboxed | Runtime privileges; sandbox unprivileged _inf_ | Cloudflare-hosted; tenant none _inf_ |
| App changes | **None** (unmodified Linux binaries) | **None** (unmodified images, ABI-limits) | **API changes / rewrite-to-platform** |
| Run existing binaries | ✅ unmodified | ✅ unmodified (ABI gaps) | ❌ no native; JS/WASM only |
| Languages | **Any** (full Linux guest) | **Any** (within ABI coverage) | **JS/TS + WASM** (Rust/C/Go via WASM) |
| Porting effort | ~zero for workload | ~zero (drop-in runsc) | Low greenfield / mod-to-high to port _inf_ |
| Required expertise | N/A (policy) | N/A (policy) | N/A (policy) |
| Debugging | Standard Linux tools (§3) | N/A (policy) | N/A (policy) |
| License | **Apache 2.0** | **Apache 2.0** | Platform commercial; **workerd Apache 2.0** |
| Primary usage | **Production** (Lambda/Fargate) | **Production** (GKE Sandbox/Cloud Run) | **Production** (Cloudflare edge) |

---

## Composability

| **Dimension** | **Firecracker** | **gVisor** | **Cloudflare Workers** |
|---|---|---|---|
| Integrates w/ other mechanisms | ✅ replaces QEMU; K8s/containerd orchestrate | ✅ OCI runsc; namespaces/cgroups/seccomp | ✅ isolate + process sandbox + MPK + cordons |
| Can coexist | ✅ many independent VMs | ✅ side-by-side with runc | ✅ thousands/process; cordons by trust |
| Can stack | ✅ many instances | ✅ runs inside a VM (Systrap) | ✅ many isolates; WASM nesting |
| Invoke each other | Message passing (TCP/IP) | Networking / IPC | RPC / service bindings |
| Share data | Message passing (no shared FS) | Message passing; Gofer-mediated FS | Message passing/serialization (KV/R2/DO); no shared mem |
| Interaction semantics | Both _inf_ | Both _inf_ | Both _inf_ |
| Decomposition boundary | **VM** | **Container** | **Service / function** |
| Who defines boundaries | Runtime/platform | Runtime | Programmer + Runtime |
| Boundaries runtime-flexible | ✅ | ✅ | ✅ (dynamic re-isolation) |
| Threading model | Standard | Standard (Sentry-impl) | **No tenant concurrency** (Spectre) |
| Process model | Host fork/exec (1 proc/VM) | Standard (Sentry-reimpl) | Custom (isolates in shared proc) |
| POSIX compat | **Full** (inside guest) | **Partial** | **Limited** (Web API + bindings) |

---

## System Design & Operations

| **Dimension** | **Firecracker** | **gVisor** | **Cloudflare Workers** |
|---|---|---|---|
| Deployment target | Cloud (serverless/containers) | Cloud / Server | Cloud / Edge (200+ locations) |
| Deployment scale | **Large scale** | **Large scale** | **Large scale** |
| Monitoring/orchestration | Good (REST API + logs; K8s/containerd) | Good (OCI integration) | N/A (policy) |
| Config complexity | N/A (policy) | N/A (policy) | N/A (policy) |
| Maintenance burden | N/A (policy) | N/A (policy) | N/A (policy) |

---

## Reading the trio

- **Compatibility falls as density rises.** Firecracker runs *any unmodified Linux binary* (full guest kernel, in the TCB-excluded sense) at ~3MB/VM; gVisor runs unmodified images but against a *partial* reimplemented ABI; Cloudflare runs only *JS/WASM* against a non-POSIX API at ~couple-MB/isolate. The Firecracker paper rejected language-VM isolation (Cloudflare's model) for Lambda precisely because Lambda must run arbitrary binaries (§2.1.2).
- **TCB shape inverts.** Firecracker shrinks the *VMM* TCB and pushes the guest kernel out of it; gVisor reimplements the kernel in memory-safe Go (medium TCB, reduced host surface); Cloudflare trusts a *large* shared V8 and compensates operationally (<24h patch gap, MPK, cordons, dynamic process isolation).
- **Side channels are unsolved across all three** — Firecracker delegates to deployer mitigations, gVisor declares them out of scope, Cloudflare mitigates-but-cannot-eliminate Spectre. None claims immunity.
- **Cross-compartment sharing is deliberately weak** in all three (message passing / mediated FS / platform primitives) — none offers fast shared-memory cross-tenant calls, since that would erode the isolation that is their entire purpose. This is the key contrast with intra-process schemes (MPK/SFI/CHERI) that *do* optimize cheap cross-domain calls.

---

## Sources

Per-system files carry full citations: [[firecracker]] (`papers/firecracker.pdf`, NSDI 2020), [[gvisor]] (gvisor.dev docs + security-basics blog + repo), [[cloudflare-workers]] (Cloudflare security-model docs + "Mitigating Spectre" blog + `workerd` repo). Flagged/inferred cells (`_inf_`, ⚠️) are logged per-system in `evaluations/REVIEW_NEEDED.md`.
