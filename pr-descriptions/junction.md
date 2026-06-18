An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/junction.md)

## What it is
Junction is a kernel-bypass cloud OS/runtime from Fried et al., "Making Kernel Bypass Practical for the Cloud with Junction" (NSDI 2024), co-authored with Azure Research and Microsoft Research. It runs unmodified Linux applications inside densely packed instances, with the goal of giving cloud operators kernel-bypass performance plus VM-like multi-tenant isolation at much higher density. Its compartmentalization role is instance-level isolation: keeping mutually untrusting tenant instances, and the host kernel, isolated from each other as an alternative to per-tenant VMs or containers.

- Compartment is an instance: a single Linux process (a kProc) running Junction's userspace library OS (the "Junction kernel") plus one or more uProcs that share one address space.
- Mechanism: no hardware primitive for isolation; isolation comes from Linux process separation, a seccomp-BPF filter that restricts the host syscall interface to about 11 calls, a per-instance chroot jail, and read-only page-cache sharing. Kernel-bypass NIC features and Intel UIPI are used for performance, not isolation.
- Trust model: the TCB includes the full host Linux kernel, the NIC (trusted to virtualize and schedule the network across instances), and the CPU/hardware. The per-instance Junction kernel is not trusted by other instances or the host; only its own uProcs trust it. uProcs within one instance share an address space and fate-share, so they are not isolated from each other.

## Key takeaways
- Performance is a net win rather than a tax: throughput is 1.6 to 7.0x higher than Linux while using 1.2 to 3.8x fewer cores across seven apps, and end-to-end p99 tail latency stays under 350 us at 3,500 instances (35x better than Linux).
- The host syscall attack surface is cut to 11 allowed syscalls (4 unique), 69 to 87 percent smaller than security-focused library OSes (Drawbridge 36, gVisor 57/64).
- Density is bounded by memory: about 0.47 MB per instance per core, around 33 MB per 8-threaded instance, reaching a maximum of 3,500 instances on 128 GB RAM.
- Isolation is coarse and deliberate: only cross-instance, with no mutual isolation between uProcs in the same instance, the full host kernel left in the TCB, and transient-execution mitigations intentionally omitted in favor of the fate-sharing model.

## Where I need review
- Efficiency, Domain switch cost: Unknown. No lmbench/lat_ctx figure is reported; syscalls become function calls into the Junction kernel and uThread switches happen in userspace.
- Efficiency, Domain creation cost: Unknown. No instance-creation latency is reported, though serverless cold-start motivates the work.
- Efficiency, Performance scales with domain count: Inferred that memory is linear, O(n), in instance count, from the per-instance footprint (Fig. 5). The paper does not state a Big O.
- Composability, Integrates with other compartmentalization mechanisms: Inferred "Partial." Built on Caladan and combinable with cgroups, but composition with other isolation frameworks (e.g. nesting in a VM) is not evaluated.
- Composability, Who defines boundaries: Inferred to be the operator/control plane, since an instance is the per-tenant/function deployment unit. The paper does not frame this explicitly.
- Usability, Required expertise level: N/A. Self-report/operational dimension, not assessable by an external evaluator from the paper.
- Operations, Initial configuration complexity: N/A. Self-report/operational dimension, not assessable from the paper.
- Operations, Ongoing maintenance burden: N/A. Self-report/operational dimension, not discussed in the paper.

License: MIT (repo github.com/JunctionOS/junction, LICENSE.md; GitHub SPDX = MIT; stated open-source in paper §1).
