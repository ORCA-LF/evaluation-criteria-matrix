# Compartmentalization Evaluation Matrix — OSv

**Version:** v0.1
**System:** OSv (single-application guest OS / unikernel for cloud VMs)
**Paper:** A. Kivity, D. Laor, G. Costa, P. Enberg, N. Har'El, D. Marti, V. Zolotarov. OSv — Optimizing the Operating System for Virtual Machines. USENIX ATC 2014.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Other — single-application VM.** Isolation between workloads is delegated to the hypervisor; OSv adds none. If several mutually-untrusting applications are to be run, they can be run in separate VMs. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the unit is the whole application+OS image. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **VM** — one application per VM. |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A.** |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None within OSv** — isolation is the hypervisor's (which uses CPU virtualization/MMU). OSv uses the MMU only for ordinary virtual memory, not for protection domains. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **None (by design)** — OSv is a library-OS/unikernel that provides **no internal software isolation** (single protection domain). Isolation relies entirely on the hypervisor, external to OSv. |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **VM**. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — OSv runs **only on a hypervisor** (KVM, Xen, VMware, VirtualBox, EC2, GCE); the OSv kernel (ELF loader, dynamic linker, scheduler, VFS/ZFS, network stack) is required. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **VM-level isolation via the hypervisor only.** Inside the VM there is **no protection**: the application and the OSv kernel share a **single address space and page tables**, with **no user/kernel separation** — system calls are ordinary function calls. OSv provides no memory protection between app and kernel. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **Cross-VM: ✅** (hypervisor). **Intra-VM: ❌** — From the single-address-space design: an application fault can corrupt the OSv kernel and bring down the whole VM; there is no internal fault containment. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **❌ / N/A** — no internal protection boundary. System calls are plain function calls and do not incur the cost of user-to-kernel parameter copying which is unnecessary in our single-application OS — i.e. no argument validation/marshalling. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Hypervisor + CPU/hardware + the entire OSv image** (kernel **and** application together, since they share one protection domain). There is no smaller in-VM TCB. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Large at system level** (hypervisor + whole guest image is one trust domain). The OSv kernel is described as relatively small / modern C++11, but **no LOC figure** is given. **Unknown — no exact size.** |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — not addressed (not a security paper). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No.** |
| Experimental validation available | Yes (specify) / No | **Yes for performance; NO security/isolation evaluation.** Benchmarks: Memcached (Memaslap), SPECjvm2008, Netperf TCP/UDP, context-switch microbenchmark, JVM balloon, boot time. No isolation/attack evaluation (isolation is the hypervisor's, not OSv's). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **OSv is faster than Linux** (it is an optimization, not an overhead). SPECjvm2008 weighted avg 1.046 vs Linux 1.041 (+0.5%); Memcached +22% throughput (+290% with a non-POSIX packet-filter API); Netperf TCP STREAM +24–25%, RR latency −37–47%. Host: 4-core 3.4 GHz i7-4770, 16 GB, KVM, Fedora 20. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Thread** context switch (Table 6): OSv 328 ns colocated / 1402 ns apart vs Linux 905 ns / 13148 ns — **3–10× faster**. _Note: these are same-protection-domain **thread** switches, not isolation-domain crossings (OSv has no protection domains)._ |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Domain = VM.** Sub-second boot; Memcached image boots in **0.6 s**, image size 11 MB. No intra-VM domain creation (single app). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **N/A internally** — syscalls are plain function calls (no crossing). Cross-VM communication is via the network stack. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **N/A internally.** Network: Memcached 406,750 tps with the packet-filter API (3.9× Linux). |
| Memory overhead per domain | MB/domain | Small images (Memcached image ≈ 11 MB); design goal is to *free* memory for the app (shrinker / JVM balloon). One app per VM — no per-compartment overhead in the isolation sense. |
| Domain count bounded by | Limiting factor; approx. domain count | **One app per VM**; multiple workloads = multiple VMs, bounded by hypervisor/host resources. |
| Performance scales with domain count | Big O | **N/A internally** (single app per VM). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — runs on KVM/Xen/VMware/VirtualBox and EC2/GCE; x86-64, preliminary ARM. |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Custom guest OS** — OSv *replaces* the guest Linux entirely; runs on stock hypervisors with no host changes. |
| What privileges does it need? | User / Root / Kernel access | Runs as an **unprivileged guest VM** (from the host's view). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | A hypervisor; the application packaged into an OSv image. |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None for unmodified single-process Linux executables** (OSv's ELF loader/dynamic linker runs standard dynamically-linked Linux binaries, resolving glibc/Linux ABI calls). **But `fork()`/`exec()` are unsupported** (no meaning in the single-app model). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — unmodified, dynamically-linked, **single-process** Linux binaries. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any** Linux executable; JVM languages (Java, Scala, JRuby) specifically optimized. |
| Other compatibility notes | [Describe] | No `fork`/`exec` (single application); optional non-POSIX APIs (network channels, shrinker, JVM balloon) for extra performance. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **~None** for compatible single-process apps (run unmodified); optional effort to adopt OSv-specific APIs for more performance. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Build effort | [build changes / time / deps] | Package the app into an OSv image; minimal. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Failure modes visibility | [crashes/logs/error codes/silent] | With no internal protection, an app fault can corrupt the kernel / crash the VM (from the single-address-space design); the paper does not detail failure handling. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source — BSD** (OSv is BSD-licensed open source, available at `github.com/cloudius-systems/osv`; `osv.io`). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Open-source project** backed by Cloudius Systems; described as a young project and a research/innovation platform. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **Partial** — composes with the hypervisor; runs on many hypervisors and clouds. It has no internal compartments to combine with other in-process mechanisms. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — OSv VMs coexist with other VMs (Linux, etc.) on a shared hypervisor. |
| Can stack effectively | ✅ / ❌ | **✅** for many OSv VMs side-by-side; no internal nesting (single app). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Cross-VM via the network** (TCP/IP stack); **no intra-VM compartments**. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Network (message passing)** across VMs; no internal sharing boundary (single address space). |
| Interaction semantics | Synchronous / Asynchronous / Both | **N/A internally**; network I/O is async/sync as usual. |
| Interaction security/validation | [Describe] | **N/A internally**; cross-VM isolation is the hypervisor's. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **VM**. |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | the **operator/deployment** — one app per VM; the hypervisor enforces the boundary. |
| Boundaries flexible at runtime | ✅ / ❌ | **❌ internally** — fixed at one app per VM; the unit of (de)composition is the VM, created/destroyed via the hypervisor. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom OSv scheduler** — lock-free, preemptive, tickless, O(lg N), FP-based fair accounting; standard threads within the single app; SMP-capable. |
| Process model | fork/exec / Custom / N/A | **N/A — single process** (no `fork`/`exec`). |
| POSIX compatibility | Full / Partial / Limited | **Partial** — single-process Linux ABI emulated; no `fork`/`exec`, no multi-process. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Single application only (no multi-process); `fork`/`exec` unsupported; isolation available only at VM granularity. |
| Security caveats when layered | [Describe] | **No intra-VM isolation** — an app fault can corrupt the OSv kernel; all security rests on the hypervisor; running mutually-untrusting components requires separate VMs. |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud** — IaaS VMs; designed with the sole purpose of running on virtual machines in the cloud. |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — many single-app VMs across a cloud. |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |

---

## Summary

> **What it is:** A single-application guest OS (library-OS/unikernel) built specifically to run one unmodified Linux application per VM in the cloud, eliminating the guest/hypervisor duplication by **removing internal isolation** (single address space, no kernel/user separation) for performance.
>
> **Who it's for:** Cloud operators running focused single-application VMs who want lower OS overhead, smaller images, and faster boot than a general-purpose Linux guest.
>
> **What it protects:** Nothing *internally* — OSv provides no intra-VM compartmentalization. Isolation between workloads is delegated entirely to the hypervisor (run mutually-untrusting apps in separate VMs).
>
> **What it costs (effort/money/performance):** A net performance *win* vs Linux (+22% Memcached throughput, −37–47% network latency, 3–10× faster context switches); the cost is loss of multi-process support (no `fork`/`exec`) and no internal protection.
>
> **What it needs (hardware/OS/expertise):** A hypervisor (KVM/Xen/VMware/VirtualBox/EC2/GCE) and an unmodified single-process Linux app packaged into an OSv image.
>
> **Key tradeoffs:** Stripping the guest's internal isolation buys significant performance and small, fast-booting images — but the entire app+kernel become a single trust domain (no intra-VM protection), only one process runs per VM (no `fork`/`exec`), and all security rests on the hypervisor.
>
> **Additional Notes:** For the ORCA matrix, OSv is best read as a **contrast/baseline**: maximal performance with VM-only isolation and *zero* intra-VM compartmentalization — the opposite design point from intra-address-space systems like RLBox, Occlum, or CubicleOS.
