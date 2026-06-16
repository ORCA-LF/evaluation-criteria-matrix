# Compartmentalization Evaluation Matrix — Firecracker

**System:** Firecracker (lightweight VMM for serverless; compartment = a **MicroVM**, one Firecracker process per VM)
**Paper:** A. Agache, M. Brooker, A. Florescu, A. Iordache, A. Liguori, R. Neugebauer, P. Piwonka, D-M. Popa. *Firecracker: Lightweight Virtualization for Serverless Applications.* NSDI 2020. USENIX.
**Source file:** `papers/firecracker.pdf` (NSDI 2020, copied from Zotero); firecracker-microvm.github.io [repo]
**Matrix version:** v0.1
**Evaluated:** 2026-06-10

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment, not a paper claim · **⚠️ Unclear** = not found, needs research.
>
> _Compartment = a **MicroVM**: a minimal hardware-virtualized guest (KVM) with its own guest kernel, served by one dedicated Firecracker VMM process. The MicroVM is "a primary security boundary, with all components assuming that code running inside the MicroVM is untrusted" (§4.1.2)._
>
> _Entry type: **Baseline / virtualization-based isolation** (the strong-isolation reference point against which intra-process compartmentalization is compared)._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — safely co-locate workloads from many mutually-distrusting tenants on one host ("protected against privilege escalation, information disclosure, covert channels, and other risks") (§2, §4.1.2). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a whole guest (an entire OS + workload) running in a VM (§3). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **VM** — one MicroVM per slot, per single customer function / single concurrent invocation (§4.1.1, §4.1.2). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is whole-VM, not data-object granular. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **Other — hardware virtualization (Intel VT-x / KVM)**: each sandbox gets its own virtual hardware, page tables, and OS kernel (§2.1.3). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — a minimal VMM written in a memory-safe language (Rust)** running on KVM, plus a **seccomp-bpf + chroot + namespace jailer** wrapping the VMM (§2.1.3, §3, §3.4.1). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **VM** (MicroVM); one Firecracker host process per MicroVM (§3, §4.1.2). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the Firecracker VMM process + KVM in the host Linux kernel + the jailer; a minimized guest Linux kernel runs inside each MicroVM (§3, §3.4.1). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | Moves the security-critical interface "from the OS boundary to a boundary supported in hardware and comparatively simpler software"; the guest kernel is treated as untrusted and the VMM+KVM limit its access to the privileged domain/host kernel (§2.1, §2.1.3). Defense-in-depth: the jailer adds chroot, pid/net namespaces, dropped privileges, and a seccomp-bpf profile whitelisting **24 syscalls** (with arg filtering) and **30 ioctls** (22 required by KVM) (§3.4.1). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — one Firecracker process per MicroVM gives "a simple model for security isolation"; killing the process tears down that MicroVM only, and a guest fault stays in its VM (§3, §4.1.2). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the guest interacts with the host only through a deliberately tiny emulated-device surface (virtio net/block, serial, partial i8042) plus the KVM ioctl interface, and is further confined by the jailer's seccomp-bpf arg/ioctl filtering (§3.1, §3.4.1). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **CPU (VT-x), host Linux kernel + KVM (≈120k LOC of KVM), the Firecracker VMM (≈50k LOC Rust), and the host firmware.** Notably the **guest kernel is explicitly outside the TCB** (treated as untrusted) (§2.1.3, §2.1.2, §3). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small (for a VMM)** — Firecracker is **≈50k LOC of Rust (96% fewer lines than QEMU's >1.4M)**; KVM adds ≈120k LOC in the host kernel. "Minimizing the feature set of the VMM helps reduce surface area, and controls the size of the TCB" (§2.1.3). Whole-system TCB still includes the host kernel, so larger than an intra-process scheme. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **Partial / operational** — Firecracker does not itself stop µarch side channels; it ships **production guidance** and Lambda/Fargate enable: disable SMT/HyperThreading, KPTI, IBPB, IBRS, L1TF cache flush, SSB mitigation, disabled swap + samepage-merging, avoid file sharing (Flush+Reload/Prime+Probe), RowHammer-resistant hardware. Power/temperature side channels explicitly **not** addressed — left to the deployer (§3.4). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal proofs; the paper relies on Rust's memory safety, "multiple levels of automated tests," and auto-generated bindings (§2.1.3). |
| Experimental validation available | Yes (specify) / No | **Yes** — boot time (500 serial + 1000 concurrent MicroVMs), memory overhead vs VM size, block IO (fio) and network (iperf3) throughput/latency, all vs QEMU and Intel Cloud Hypervisor on an EC2 m5d.metal; plus production validation at Lambda/Fargate scale (trillions of requests/month). Data + scripts public ([repo] nsdi2020-data) (§5). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Reported: memory overhead ≈3MB/MicroVM and "negligible overhead on CPU"; production memory overhead as low as **3%** (§5.2, §5.4). CPU-bound SPEC-style benchmarks not run (paper targets serverless density/boot/IO). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **N/A / not lmbench.** VM scheduling is delegated to the host Linux scheduler; no lat_ctx-equivalent reported (§3). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **MicroVM boot (the creation cost):** boots to application code in **<125ms**; production "as little as 150ms"; pre-configured FC 99th-pct **146ms** (1000 MicroVMs, 50 launching concurrently) / **153ms** (100 concurrent); supports **up to 150 MicroVMs/sec/host** (§1, §5.1, §5.4). Not lat_proc, but this is the domain-creation analogue. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **N/A — strong isolation, no fast cross-VM call.** Inter-VM/control-plane traffic goes over **TCP/IP sockets** (MicroManager ↔ in-guest shim), deliberately loosely coupled; no lat_pipe figure (§4.1.2). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Not bw_pipe.** Guest↔host network (virtio, tap, 1500B MTU): FC ≈**15 Gb/s** (1 & 10 streams) vs loopback 44–47 Gb/s, Cloud HV ≈21–23, QEMU ≈19–30 (Table 1). Block IO: 4kB read ≈**13,000 IOPS / 52 MB/s** (HW capable of 340k IOPS); 4kB read latency only **49µs** over native (§5.3). |
| Memory overhead per domain | MB/domain | **≈3 MB per MicroVM** (smallest of the three VMMs; Cloud HV ≈13MB, QEMU ≈131MB), constant regardless of configured VM size; <5MB/container with the minimal guest config (§1, §5.2). |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: host RAM (and the ≈3MB/VM overhead + soft allocation).** Thousands of MicroVMs per host; "hundreds or thousands of MicroVMs" per Lambda worker; up to ~8000 128MB functions to fill 1TB RAM (§2, §4.1.2, §5.4). |
| Performance scales with domain count | Big O | _Inferred:_ memory **O(n)** in MicroVM count (fixed ≈3MB each); CPU/IO contention handled by the host scheduler + per-device rate limiters; no asymptotic statement in the paper (§3, §5.3). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** x86-64 with hardware virtualization (Intel VT-x / KVM); also runs on ARM64 [repo]. Evaluated on EC2 m5d.metal (Intel Xeon Platinum 8175M) (§5). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock Linux host with KVM** (Ubuntu 18.04, kernel 4.15-aws in the eval); the guest runs a **minimized (but unmodified-API) Linux kernel** (§3, §5). |
| What privileges does it need? | User / Root / Kernel access | **Root/host setup** to configure KVM, tap interfaces, cgroups and the jailer; the Firecracker VMM itself is then dropped into a restrictive sandbox (chroot, dropped privileges, seccomp) (§3.4.1). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | KVM enabled; TUN/TAP networking; cgroups; a guest kernel image + root filesystem. The jailer is recommended in production (§3, §3.4.1). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None** — a core design goal: run "arbitrary Linux binaries and libraries… without code changes or recompilation." The guest runs a full, unmodified Linux ABI (§2, §5.4). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — unmodified Linux binaries; "We have not found software that does not run in a MicroVM, other than software with specific hardware requirements" (§5.4). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any** — language-agnostic; anything that runs on Linux (the guest is a real Linux kernel + userland) (§4.1, §5.4). |
| Other compatibility notes | [Describe] | No BIOS, cannot boot arbitrary kernels, no legacy-device/PCI emulation, no VM migration, could not boot Windows without major changes; guests need virtio drivers (present in stock Linux) (§1.1). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **~Zero for the workload** — unmodified binaries run as-is. (Integration effort is at the platform/orchestration layer, not the app.) (§2, §5.4). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | No app rebuild. Operators build/ship a minimal guest kernel (4.0MB compressed, no modules) + rootfs; Firecracker is a single static Rust binary (§5.1). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Standard Linux tooling** (stated, and a deliberate design value): a Firecracker host shows all MicroVMs in `ps`, and `top`, `vmstat`, `kill` work as operators expect (§3). |
| Failure modes visibility | [crashes/logs/error codes/silent] | Per-MicroVM logs & metrics consumed by humans + automated alarming; MicroManager returns response payloads or **error details** on failure; standard Linux observability on the host (§3.3, §4.1.2). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — Apache 2.0** (released Dec 2018) (§1; [repo] github.com/firecracker-microvm/firecracker). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Production** — powers AWS Lambda and Fargate ("millions of workloads, trillions of requests per month") since 2018; also widely adopted by the open-source container community (§1, §4, §6). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — Firecracker "replaces QEMU, rather than Docker or Kubernetes"; orchestration/packaging are delegated to Kubernetes, Docker, containerd; integrated under Kata Containers et al. (§1.1, §3). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — hundreds/thousands of independent MicroVMs (one process each) coexist on a host alongside ordinary host processes and monitoring services (§4.1.2). |
| Can stack effectively | ✅ / ❌ | **✅** — the design is inherently many-independent-instances; soft-allocation/oversubscription tested at >20× (§4.2, §5.4). Nesting VMs-in-VMs is not a focus. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Message passing over TCP/IP** — the in-guest Lambda shim talks to the per-worker MicroManager via a TCP/IP socket across the MicroVM boundary (§4.1.2). MicroVMs are otherwise isolated and do not directly invoke each other. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Message passing** (payloads over TCP/IP); deliberately **no** filesystem passthrough/shared FS for security (block devices only) (§3.1, §4.1.2). |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Both** — request/response invoke (synchronous to the caller) over an async socket/event path; not framed this way in the paper (§4.1.1–4.1.2). |
| Interaction security/validation | [Describe] | The MicroManager↔shim protocol "is the boundary between the multi-tenant Lambda control plane and the single-tenant (single-function) MicroVM" — the explicit security-validated interface (§4.1.2). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **VM** (one MicroVM = one function slot) (§4.1.1). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Runtime/platform** — the Lambda control plane (Placement/Worker Manager/MicroManager) decides slot/MicroVM creation; the developer just supplies a function (§4.1.1). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — MicroVMs are created/destroyed on demand (≤150/s/host); slots transition init→idle→busy→dead; a warm pool hides boot latency (§4.1.1, §4.2, §5.4). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Standard** — the guest runs a normal Linux kernel; the host uses the standard Linux scheduler for VM threads; one Firecracker process per VM (§3, §4.1.2). |
| Process model | fork/exec / Custom / N/A | **Standard host fork/exec** (one VMM process per MicroVM); inside the guest, normal Linux process model (§4.1.2). |
| POSIX compatibility | Full / Partial / Limited | **Full (inside the guest)** — a complete unmodified Linux kernel/userland runs in the MicroVM (§2, §5.4). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | No device hotplug variety (virtio net/block, serial, partial i8042 only), no PCI/USB/GPU, no VM migration, no BIOS/arbitrary-kernel boot; cross-MicroVM comms must use networking (TCP/IP) — efficient host-bypass/shared paths sacrificed for isolation (§1.1, §3.1, §4.1.2). |
| Security caveats when layered | [Describe] | µarch side channels (Spectre/Meltdown/Zombieload/L1TF/RowHammer) and power/thermal channels are **not** mitigated by Firecracker alone — the deployer must apply the host-setup mitigations; SMT must be disabled (§3.4). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud** — serverless functions & containers (AWS Lambda, Fargate); "generally useful for containers, functions and other compute workloads" (§1, §4). |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — multi-tenant fleets serving trillions of requests/month; hundreds–thousands of MicroVMs per worker (§4, §4.1.2). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **Good** — REST API over a Unix socket for full lifecycle control; per-MicroVM logs/metrics for humans + automated alarming; orchestration delegated to Kubernetes/Docker/containerd (§3.2, §4.1.2). _(Paper explicitly states the value, so not N/A under the 2026-06-09 policy.)_ |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). _(Design intent is minimal config: simplest use needs only a guest kernel + one root block device, §3.2.)_ |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper (policy 2026-06-09). _(AWS uses immutable-infrastructure re-imaging for patching, §4.3 — operational context, not a burden rating.)_ |

---

## Summary

> **What it is:** A minimal, open-source Virtual Machine Monitor (VMM) written in ~50k lines of Rust that runs on KVM to create lightweight MicroVMs — purpose-built so cloud providers can pack thousands of mutually-distrusting serverless workloads onto one host with VM-grade isolation but container-grade density and startup.
>
> **Who it's for:** Infrastructure/serverless platform builders (AWS Lambda & Fargate in the paper; the broader container community) who need strong multi-tenant isolation without the overhead of a full QEMU/VM.
>
> **What it protects:** The host (and other tenants) from untrusted guest code — the guest kernel itself is treated as untrusted, and the security boundary is moved to hardware virtualization + a tiny VMM, hardened further by the jailer (chroot, namespaces, dropped privileges, seccomp whitelisting 24 syscalls / 30 ioctls).
>
> **What it costs (effort/money/performance):** ≈3MB memory overhead per MicroVM and ~negligible CPU overhead; <125–150ms boot; up to 150 MicroVMs/sec/host. Block IO and network throughput are notably below bare metal (≈15 Gb/s net; ≈13k IOPS 4k) — adequate for serverless, not for IO-bound bare-metal workloads. Near-zero porting effort: unmodified Linux binaries run as-is.
>
> **What it needs (hardware/OS/expertise):** Commodity x86-64/ARM64 with hardware virtualization, a stock Linux host with KVM, a minimized guest kernel image — no application changes.
>
> **Key tradeoffs:** Strong, hardware-backed isolation with full Linux compatibility and tiny per-VM overhead, achieved by deliberately dropping features (no PCI/USB/GPU, no migration, no BIOS, limited device model) and accepting lower IO/network throughput. Side channels are an explicit, deployer-owned residual risk, not solved by Firecracker itself.
>
> **Additional Notes:** This is a **baseline / virtualization-based** reference entry, not an intra-process compartmentalization technique — its compartment is a whole VM. It is the canonical "strong security, low overhead" point that intra-address-space schemes (MPK/SFI/CHERI etc.) are measured against. Started from Google's crosvm; shares the rust-vmm crate base with Intel Cloud Hypervisor. Contrast with [[gvisor]], the other requested baseline, which intercepts syscalls in a userspace kernel rather than running a full guest kernel under KVM.

---

## Sources

- **Primary:** Agache et al., *Firecracker: Lightweight Virtualization for Serverless Applications*, NSDI 2020 (USENIX). Local: `papers/firecracker.pdf` (copied from Zotero; extracted via `pdftotext -layout`). All **(§N)** / Fig. / Table citations reference this paper.
- **License / repo:** Apache 2.0, github.com/firecracker-microvm/firecracker; firecracker-microvm.github.io. Public eval data: github.com/firecracker-microvm/nsdi2020-data. **[repo]**

### Cells flagged for further research
- **Efficiency › Domain switch / inter-domain call latency** — N/A by design (VM scheduling delegated to host; cross-VM comms over TCP/IP); no lmbench-equivalent reported.
- **Efficiency › General runtime overhead (SPEC)** — not run; only memory/IO/boot benchmarks given.
- **Usability › Required expertise** + **Sys Design › Config complexity, Maintenance burden** — N/A (self-report policy 2026-06-09).
- Inferences marked `_Inferred:_` inline: memory Big-O, interaction semantics (sync/async).
