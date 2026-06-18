An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/osv.md)

## What it is
OSv is a single-application guest OS (a library-OS/unikernel) built to run one unmodified Linux application per VM in the cloud, presented by Kivity, Laor, Costa, Enberg, Har'El, Marti, and Zolotarov at USENIX ATC 2014. It addresses guest/hypervisor duplication and overhead rather than intra-guest isolation: it deliberately removes internal isolation (single address space, no kernel/user separation) to gain performance. We include it as a baseline contrast point, the strong-VM-isolation, zero-intra-VM-isolation end of the spectrum, so many compartmentalization rows are N/A by design.

- Compartment is the VM: one application plus the OSv kernel packaged into a single image, with isolation between workloads delegated to the hypervisor (mutually-untrusting apps run in separate VMs).
- Mechanism: no hardware protection primitive and no software isolation technique within OSv; the MMU is used only for ordinary virtual memory. All isolation is provided externally by the hypervisor (KVM, Xen, VMware, VirtualBox, EC2, GCE) using CPU virtualization.
- Trust model: the TCB is the hypervisor plus CPU/hardware plus the entire OSv image (kernel and application together, since they share one protection domain). There is no smaller in-VM TCB; inside the VM there is no user/kernel separation and "system calls" are ordinary function calls.

## Key takeaways
- OSv is an optimization rather than an overhead: SPECjvm2008 weighted average 1.046 vs Linux 1.041 (+0.5%), Memcached throughput +22% (and +290% with a non-POSIX packet-filter API), Netperf TCP STREAM +24 to 25%, and request/response latency 37 to 47% lower than Linux.
- Thread context switches are 3 to 10x faster than Linux (328 ns colocated / 1402 ns apart vs 905 ns / 13148 ns), but these are same-protection-domain thread switches, not isolation-domain crossings, since OSv has no protection domains.
- Images are small and boot fast: the Memcached image is about 11 MB and boots in roughly 0.6 s.
- The cost of stripping internal isolation is that an app fault can corrupt the OSv kernel and crash the whole VM (no intra-VM fault containment), only one process runs per VM (no fork/exec), and all security rests on the hypervisor. There is no security or isolation evaluation in the paper; experimental validation is performance-only.

## Where I need review
- Security › TCB size: Unknown. The OSv kernel is described as "relatively small" / modern C++11 but no LOC figure is given.
- Security › Compartment crashes isolated (intra-VM): Inferred from the single-address-space design (§2) that an app fault can corrupt the kernel and bring down the VM; the paper does not state this directly.
- Usability › Failure modes visibility: Inferred from the single-address-space design; the paper does not detail failure handling.
- Usability › Required expertise level: N/A. Self-report/operational dimension not assessable by an external evaluator from the paper.
- Usability › Debugging support: N/A (also not discussed in the paper). Self-report/operational dimension.
- Composability › Integrates with other mechanisms: Inferred Partial (composes with the hypervisor; no internal compartments to combine with in-process mechanisms).
- Composability › Can coexist with other systems: Inferred yes (OSv VMs coexist with other VMs on a shared hypervisor).
- Composability › Can stack effectively: Inferred yes for many OSv VMs side by side; no internal nesting.
- Composability › Who defines boundaries: Inferred to be the operator/deployment (one app per VM, hypervisor-enforced).
- Sys Design › Deployment scale: Inferred Large scale (many single-app VMs across a cloud).
- Sys Design › Monitoring/orchestration support: N/A (also not discussed in the paper). Self-report/operational dimension.
- Sys Design › Initial configuration complexity: N/A. Self-report/operational dimension.
- Sys Design › Ongoing maintenance burden: N/A. Self-report/operational dimension.

License: BSD (paper §8: "OSv is BSD-licensed open source, available at github.com/cloudius-systems/osv").
