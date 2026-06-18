An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/kylinx.md)

## What it is
KylinX is a dynamic library operating system that gives unikernels a process-like VM abstraction, which it calls a pVM. It was published by Zhang et al. at USENIX ATC 2018 ("KylinX: A Dynamic Library Operating System for Simplified and Efficient Cloud Virtualization"). The problem it addresses is multi-tenant cloud isolation: it tries to keep the strong VM-level isolation between mutually-untrusting tenant appliances while recovering the dynamism of ordinary processes (fork, inter-process communication, runtime library update, fast boot and recycling), which a statically sealed unikernel gives up.

- Compartment is a pVM, a process-like paravirtualized Xen domain running a MiniOS-based unikernel appliance, extended with process-style fork, inter-pVM communication, and dynamic library mapping.
- Mechanism: no dedicated hardware primitive; isolation comes from the Xen type-1 hypervisor (CPU rings plus MMU). The software technique is a unikernel/library OS (MiniOS plus Newlib libc and lwIP), with dynamic library mapping guarded by RSA/SHA-1 signature, version, and loader restrictions.
- Trust model: the TCB includes the Xen hypervisor and its toolstacks plus the control domain (dom0), and the CPU/hardware (assumed uncompromised). Other tenants' pVMs are untrusted. Inter-pVM communication is permitted only within a "family" of pVMs forked from a common root; communication between untrusted pVMs is rejected.

## Key takeaways
- Process-like dynamics at low cost: pVM fork is about 1.3 ms (versus about 1 ms for a Linux process fork), a full standard boot is about 124 ms for one pVM (versus 133 ms for MiniOS and 210 ms for Docker), and recycling reuses an in-memory domain to bypass domain creation entirely.
- Inter-pVM communication is competitive with native IPC. For established (lineal) pVMs the latencies (Table 3) are pipe 55, msg_queue 43, kill 41, exit/wait 43, and shared memory 39 microseconds, comparable to native Ubuntu IPC. First-time (non-lineal) pVMs pay 232 to 256 microseconds for channel setup.
- Memory footprint is the main scalability limit. The system was tested up to 1,000 pVMs per machine, with roughly 6 to 7 MB total footprint per pVM for a reduced-Redis workload at that count, lower than statically sealed MiniOS and Docker because shared libraries are loaded once.
- The cost is portability: applications must be ported to MiniOS/Newlib, are effectively C-only, must use select rather than epoll, and are limited to the pipe/signal/message-queue/shared-memory IPC API. The hypervisor plus dom0 sit in the TCB, and pVM recycling currently has no security safeguards between the old and new pVM.

## Where I need review
- License (Availability): no confirmed official licensed release was found. The paper states no license, the project page returned HTTP 403, the github.com/kylinx/kylinx-tools repo is empty, and the only code located (LinuxSecurityModules/XenKylinx) is a third-party reimplementation, not the authors'. Needs author contact or repo verification.
- Domain switch cost (Runtime Efficiency): Unknown. No lat_ctx-style pVM context-switch figure is reported; pVM scheduling is handled by Xen.
- Inter-domain throughput (Runtime Efficiency): Unknown. No bw_pipe-style IpC throughput is measured.
- Porting effort for typical app (Usability): Unknown. The paper claims "minimum effort" but gives no person-days or LOC figure, while still requiring concrete adaptations (select, restricted IPC, removing Redis serialization).
- Memory overhead per domain (Runtime Efficiency): inferred from Fig. 6 as about 6 to 7 MB per pVM total footprint; this is total footprint, not isolated overhead.
- Performance scaling Big O (Runtime Efficiency): inferred as memory linear / O(n) in pVM count from Fig. 6 (boot time is separately reported as superlinear with stock XenStore, linear with LightVM noxs).
- Compartment crashes isolated (Security): inferred yes, since pVMs are isolated Xen domains; the paper does not frame this as a crash-recovery experiment.
- Integrates with other mechanisms (Composability): inferred Partial; integrates with the Xen ecosystem and adopts LightVM noxs, but composition with non-Xen frameworks is not addressed.
- Can coexist with other systems (Composability): inferred yes; pVMs run alongside dom0 and other Xen domains, but this is not framed as an explicit claim.
- Who defines boundaries (Composability): inferred Programmer plus Runtime; fork() calls create pVMs and the dom0 toolstack instantiates and manages them.
- Required expertise level, Debugging support, Monitoring/orchestration support, Initial configuration complexity, Ongoing maintenance burden: all N/A as self-report/operational dimensions an external paper-based evaluator cannot assess (per project policy).

License: Not found (paper states no license; no official licensed release located, only an empty github.com/kylinx/kylinx-tools and a third-party reimplementation LinuxSecurityModules/XenKylinx).
