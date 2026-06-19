# Compartmentalization Evaluation Matrix — Firecracker

**Version:** v0.1
**System:** Firecracker (lightweight VMM for serverless; compartment = a MicroVM, one Firecracker process per VM)
**Paper:** A. Agache, M. Brooker, A. Florescu, A. Iordache, A. Liguori, R. Neugebauer, P. Piwonka, D-M. Popa. Firecracker: Lightweight Virtualization for Serverless Applications. NSDI 2020. USENIX.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — safely co-locate workloads from many mutually-distrusting tenants on one host (protected against privilege escalation, information disclosure, covert channels, and other risks). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a whole guest (an entire OS + workload) running in a VM. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **VM** — one MicroVM per slot, per single customer function / single concurrent invocation. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is whole-VM, not data-object granular. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **Other — hardware virtualization (Intel VT-x / KVM)**: each sandbox gets its own virtual hardware, page tables, and OS kernel. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — a minimal VMM written in a memory-safe language (Rust)** running on KVM, plus a **seccomp-bpf + chroot + namespace jailer** wrapping the VMM. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **VM** (MicroVM); one Firecracker host process per MicroVM. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the Firecracker VMM process + KVM in the host Linux kernel + the jailer; a minimized guest Linux kernel runs inside each MicroVM. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | Moves the security-critical interface from the OS boundary to a boundary supported in hardware and comparatively simpler software; the guest kernel is treated as untrusted and the VMM+KVM limit its access to the privileged domain/host kernel. Defense-in-depth: the jailer adds chroot, pid/net namespaces, dropped privileges, and a seccomp-bpf profile whitelisting **24 syscalls** (with arg filtering) and **30 ioctls** (22 required by KVM). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — one Firecracker process per MicroVM gives a simple model for security isolation; killing the process tears down that MicroVM only, and a guest fault stays in its VM. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the guest interacts with the host only through a deliberately tiny emulated-device surface (virtio net/block, serial, partial i8042) plus the KVM ioctl interface, and is further confined by the jailer's seccomp-bpf arg/ioctl filtering. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **CPU (VT-x), host Linux kernel + KVM (≈120k LOC of KVM), the Firecracker VMM (≈50k LOC Rust), and the host firmware.** Notably the **guest kernel is explicitly outside the TCB** (treated as untrusted). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small (for a VMM)** — Firecracker is **≈50k LOC of Rust (96% fewer lines than QEMU's >1.4M)**; KVM adds ≈120k LOC in the host kernel. Minimizing the feature set of the VMM helps reduce surface area, and controls the size of the TCB. Whole-system TCB still includes the host kernel, so larger than an intra-process scheme. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **Partial / operational** — Firecracker does not itself stop µarch side channels; it ships **production guidance** and Lambda/Fargate enable: disable SMT/HyperThreading, KPTI, IBPB, IBRS, L1TF cache flush, SSB mitigation, disabled swap + samepage-merging, avoid file sharing (Flush+Reload/Prime+Probe), RowHammer-resistant hardware. Power/temperature side channels explicitly **not** addressed — left to the deployer. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal proofs; the paper relies on Rust's memory safety, multiple levels of automated tests, and auto-generated bindings. |
| Experimental validation available | Yes (specify) / No | **Yes** — boot time (500 serial + 1000 concurrent MicroVMs), memory overhead vs VM size, block IO (fio) and network (iperf3) throughput/latency, all vs QEMU and Intel Cloud Hypervisor on an EC2 m5d.metal; plus production validation at Lambda/Fargate scale (trillions of requests/month). Data + scripts public (nsdi2020-data). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Reported: memory overhead ≈3MB/MicroVM and negligible overhead on CPU; production memory overhead as low as **3%**. CPU-bound SPEC-style benchmarks not run (paper targets serverless density/boot/IO). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **N/A / not lmbench.** VM scheduling is delegated to the host Linux scheduler; no lat_ctx-equivalent reported. |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **MicroVM boot (the creation cost):** boots to application code in **<125ms**; production as little as 150ms; pre-configured FC 99th-pct **146ms** (1000 MicroVMs, 50 launching concurrently) / **153ms** (100 concurrent); supports **up to 150 MicroVMs/sec/host**. Not lat_proc, but this is the domain-creation analogue. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **N/A — strong isolation, no fast cross-VM call.** Inter-VM/control-plane traffic goes over **TCP/IP sockets** (MicroManager ↔ in-guest shim), deliberately loosely coupled; no lat_pipe figure. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Not bw_pipe.** Guest↔host network (virtio, tap, 1500B MTU): FC ≈**15 Gb/s** (1 & 10 streams) vs loopback 44–47 Gb/s, Cloud HV ≈21–23, QEMU ≈19–30. Block IO: 4kB read ≈**13,000 IOPS / 52 MB/s** (HW capable of 340k IOPS); 4kB read latency only **49µs** over native. |
| Memory overhead per domain | MB/domain | **≈3 MB per MicroVM** (smallest of the three VMMs; Cloud HV ≈13MB, QEMU ≈131MB), constant regardless of configured VM size; <5MB/container with the minimal guest config. |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: host RAM (and the ≈3MB/VM overhead + soft allocation).** Thousands of MicroVMs per host; hundreds or thousands of MicroVMs per Lambda worker; up to ~8000 128MB functions to fill 1TB RAM. |
| Performance scales with domain count | Big O | memory **O(n)** in MicroVM count (fixed ≈3MB each); CPU/IO contention handled by the host scheduler + per-device rate limiters; no asymptotic statement in the paper. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** x86-64 with hardware virtualization (Intel VT-x / KVM); also runs on ARM64. Evaluated on EC2 m5d.metal (Intel Xeon Platinum 8175M). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock Linux host with KVM** (Ubuntu 18.04, kernel 4.15-aws in the eval); the guest runs a **minimized (but unmodified-API) Linux kernel**. |
| What privileges does it need? | User / Root / Kernel access | **Root/host setup** to configure KVM, tap interfaces, cgroups and the jailer; the Firecracker VMM itself is then dropped into a restrictive sandbox (chroot, dropped privileges, seccomp). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | KVM enabled; TUN/TAP networking; cgroups; a guest kernel image + root filesystem. The jailer is recommended in production. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None** — a core design goal: run arbitrary Linux binaries and libraries… without code changes or recompilation. The guest runs a full, unmodified Linux ABI. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — unmodified Linux binaries; We have not found software that does not run in a MicroVM, other than software with specific hardware requirements. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any** — language-agnostic; anything that runs on Linux (the guest is a real Linux kernel + userland). |
| Other compatibility notes | [Describe] | No BIOS, cannot boot arbitrary kernels, no legacy-device/PCI emulation, no VM migration, could not boot Windows without major changes; guests need virtio drivers (present in stock Linux). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **~Zero for the workload** — unmodified binaries run as-is. (Integration effort is at the platform/orchestration layer, not the app.) |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Build effort | [build changes / time / deps] | No app rebuild. Operators build/ship a minimal guest kernel (4.0MB compressed, no modules) + rootfs; Firecracker is a single static Rust binary. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **Standard Linux tooling** (stated, and a deliberate design value): a Firecracker host shows all MicroVMs in `ps`, and `top`, `vmstat`, `kill` work as operators expect. |
| Failure modes visibility | [crashes/logs/error codes/silent] | Per-MicroVM logs & metrics consumed by humans + automated alarming; MicroManager returns response payloads or **error details** on failure; standard Linux observability on the host. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — Apache 2.0** (released Dec 2018). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Production** — powers AWS Lambda and Fargate (millions of workloads, trillions of requests per month) since 2018; also widely adopted by the open-source container community. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — Firecracker replaces QEMU, rather than Docker or Kubernetes; orchestration/packaging are delegated to Kubernetes, Docker, containerd; integrated under Kata Containers et al. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — hundreds/thousands of independent MicroVMs (one process each) coexist on a host alongside ordinary host processes and monitoring services. |
| Can stack effectively | ✅ / ❌ | **✅** — the design is inherently many-independent-instances; soft-allocation/oversubscription tested at >20×. Nesting VMs-in-VMs is not a focus. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Message passing over TCP/IP** — the in-guest Lambda shim talks to the per-worker MicroManager via a TCP/IP socket across the MicroVM boundary. MicroVMs are otherwise isolated and do not directly invoke each other. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Message passing** (payloads over TCP/IP); deliberately **no** filesystem passthrough/shared FS for security (block devices only). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — request/response invoke (synchronous to the caller) over an async socket/event path; not framed this way in the paper. |
| Interaction security/validation | [Describe] | The MicroManager↔shim protocol is the boundary between the multi-tenant Lambda control plane and the single-tenant (single-function) MicroVM — the explicit security-validated interface. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **VM** (one MicroVM = one function slot). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Runtime/platform** — the Lambda control plane (Placement/Worker Manager/MicroManager) decides slot/MicroVM creation; the developer just supplies a function. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — MicroVMs are created/destroyed on demand (≤150/s/host); slots transition init→idle→busy→dead; a warm pool hides boot latency. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Standard** — the guest runs a normal Linux kernel; the host uses the standard Linux scheduler for VM threads; one Firecracker process per VM. |
| Process model | fork/exec / Custom / N/A | **Standard host fork/exec** (one VMM process per MicroVM); inside the guest, normal Linux process model. |
| POSIX compatibility | Full / Partial / Limited | **Full (inside the guest)** — a complete unmodified Linux kernel/userland runs in the MicroVM. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | No device hotplug variety (virtio net/block, serial, partial i8042 only), no PCI/USB/GPU, no VM migration, no BIOS/arbitrary-kernel boot; cross-MicroVM comms must use networking (TCP/IP) — efficient host-bypass/shared paths sacrificed for isolation. |
| Security caveats when layered | [Describe] | µarch side channels (Spectre/Meltdown/Zombieload/L1TF/RowHammer) and power/thermal channels are **not** mitigated by Firecracker alone — the deployer must apply the host-setup mitigations; SMT must be disabled. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud** — serverless functions & containers (AWS Lambda, Fargate); generally useful for containers, functions and other compute workloads. |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — multi-tenant fleets serving trillions of requests/month; hundreds–thousands of MicroVMs per worker. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **Good** — REST API over a Unix socket for full lifecycle control; per-MicroVM logs/metrics for humans + automated alarming; orchestration delegated to Kubernetes/Docker/containerd. _(Paper explicitly states the value, so not N/A under the policy.)_ |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. _(Design intent is minimal config: simplest use needs only a guest kernel + one root block device.)_ |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. _(AWS uses immutable-infrastructure re-imaging for patching — operational context, not a burden rating.)_ |

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
> **Additional Notes:** This is a **baseline / virtualization-based** reference entry, not an intra-process compartmentalization technique — its compartment is a whole VM. It is the canonical strong security, low overhead point that intra-address-space schemes (MPK/SFI/CHERI etc.) are measured against. Started from Google's crosvm; shares the rust-vmm crate base with Intel Cloud Hypervisor. Contrast with gvisor, the other requested baseline, which intercepts syscalls in a userspace kernel rather than running a full guest kernel under KVM.
