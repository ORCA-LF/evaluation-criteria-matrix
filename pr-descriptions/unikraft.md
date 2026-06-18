An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/unikraft.md)

## What it is
Unikraft is a modular micro-library unikernel for building specialized, single-application OS images by composing OS primitives (allocators, schedulers, network stacks, boot code) as swappable micro-libraries behind performance-minded APIs. It comes from Kuenzer et al., "Unikraft: Fast, Specialized Unikernels the Easy Way," EuroSys 2021. It is not itself an internal-compartmentalization system; it addresses specialization and performance, running each application as its own VM with a single address space and a single protection level. It is included here as a baseline and substrate, since FlexOS, uFork, and CubicleOS in this matrix build their compartmentalization on top of it.

- Compartment is the VM by default (one application per VM); micro-library granularity only if compartmentalization extensions are added.
- Mechanism: no internal hardware or software isolation primitive by default. Isolation between workloads is the hypervisor's (separate VMs on QEMU/KVM, Xen, Firecracker, or Solo5). The paper notes initial support for hardware compartmentalization via Intel MPK (Iso-Unik style) as a direction, not the core mechanism.
- Trust model: the TCB is the hypervisor plus the CPU/hardware plus the entire Unikraft image, since the application and kernel share one protection domain. There is no user/kernel separation inside the VM and no intra-guest memory isolation between components by default.

## Key takeaways
- Performance is the point: the paper reports a 1.7x to 2.7x improvement over Linux guests on nginx, SQLite, and Redis. Images are small (minimal config 200 KB, typical app images around 1 MB), run in under 10 MB of RAM, and boot in roughly 1 ms on top of the VMM (total boot 2 to 40 ms).
- Isolation is VM-granularity only. Cross-VM crashes are contained by the hypervisor, but there is no intra-VM fault containment; an application fault can corrupt the kernel and VM. Many isolation rows are N/A by design.
- It is a composition substrate. The paper explicitly designs Unikraft to host compartmentalization extensions (its reference [69] is Sung et al., intra-unikernel MPK isolation, VEE'20), and FlexOS, uFork, and CubicleOS in this matrix are built on it.
- Adoption cost is porting/relinking against Unikraft's micro-libraries, eased by a musl libc plus syscall shim layer, with several apps pre-ported (SQLite, nginx, Redis).

## Where I need review
- Security, intra-VM crash isolation (marked No, Inferred): inferred from the single-protection-level design; the paper does not separately characterize internal fault containment.
- Security, TCB size: image sizes and a "small TCB" claim are given, but no total TCB LOC figure is reported.
- Usability, porting effort for a typical app (Unknown): described as reduced versus classic unikernels, but no person-days or LOC figure is given.
- Composability, can coexist with other compartmentalization systems (Yes, Inferred): inferred from many Unikraft VMs coexisting on a shared hypervisor.
- Composability, can stack effectively (Yes, Inferred): inferred for many VMs side by side; internal composition is the modular micro-library model.
- Composability, who defines boundaries (Inferred): inferred that the developer selects and composes micro-libraries at build time and the operator deploys one app per VM.
- System Design, deployment scale (Large scale, Inferred): inferred from many small, fast-booting VMs across a cloud.
- Usability, failure modes visibility (Inferred): inferred that an app fault can corrupt the kernel/VM under a single protection level; the paper does not detail failure handling.
- Self-report and operational dimensions set to N/A by policy: required expertise level, debugging support, monitoring/orchestration support, initial configuration complexity, and ongoing maintenance burden. These are not assessable by an external paper-based evaluator.

License: Open source, BSD-3-Clause (main project; individual micro-libraries may be separately licensed) (repo github.com/unikraft/unikraft, COPYING.md; GitHub SPDX = NOASSERTION for that reason).
