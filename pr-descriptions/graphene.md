An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/graphene.md)

## What it is
Graphene is a Linux-compatible library OS that runs unmodified multi-process Linux applications on a shared host with isolation the authors describe as comparable to running the application in a separate VM. It was published as "Cooperation and Security Isolation of Library OSes for Multi-Process Applications" by Tsai, Arora, Bandi, Jain, Jannen, John, Kalodner, Kulkarni, Oliveira, and Porter at EuroSys 2014. The problem it addresses is isolating mutually-distrusting multi-process applications from each other and from the host kernel without the memory and startup cost of a full VM. (This is the 2014 Linux version; the project was later renamed Gramine and gained Intel SGX support, which is a separate system not covered here.)

- Compartment is a sandbox, that is, a group of mutually-trusting picoprocesses, where a picoprocess is an OS process running over a narrowed host ABI. Picoprocesses within a sandbox trust each other (all-or-nothing isolation); separate sandboxes are strictly isolated.
- Mechanism: pure software isolation on commodity hardware with no hardware primitive. A library OS (libLinux.so) runs over a narrow host ABI called the PAL (43 functions, around 50 host syscalls), with host enforcement via seccomp-BPF syscall filtering plus an AppArmor LSM reference monitor mediating file and socket access per a per-application manifest.
- Trust model: the TCB includes the host Linux kernel, the reference monitor (an unprivileged daemon plus an AppArmor LSM extension plus a kernel IPC module), and the CPU/hardware. Explicitly untrusted are libLinux.so and the PAL; the threat model assumes the adversary can control all code inside one or more picoprocesses, including libLinux and the PAL.

## Key takeaways
- Isolation comes with a measured cut in host attack surface to 15% of the Linux system-call table, file and network access mediated by a per-app manifest, and all cross-sandbox communication blocked. A manual study of 291 Linux CVEs from 2011 to 2013 found Graphene would prevent 147 of them (51%).
- The TCB is large because it includes the full host kernel plus AppArmor. Graphene's own added trusted code is around 5.6 KLoC (reference monitor 3,568 LOC, AppArmor LSM extension 888 LOC, kernel IPC module 1,131 LOC). The untrusted libOS is much larger (libLinux 31,112 LOC, PAL 11,644 LOC) but sits outside the TCB.
- Performance costs are moderate at the application level and higher on fork/IPC-heavy microbenchmarks: 5 to 30% on compilation workloads, fork+exit around 5.9x slower (67 to 463 microseconds), picoprocess startup around 641 microseconds (3.1x native Linux, versus 3.3 seconds for KVM), and System V msgsnd 153 to 761 microseconds. Memory cost is roughly 0.8 to 1.4 MB per picoprocess, which is 3 to 20x less than KVM.
- Isolation is all-or-nothing. Within a sandbox a corrupted picoprocess can corrupt shared libOS state, controlled cross-sandbox sharing is future work, and deployment requires modifying the host (kernel IPC module plus LSM extension, root to install) plus a manifest per app, though applications themselves run unmodified.

## Where I need review
- Performance scales with domain count (Runtime Efficiency): inferred as memory-linear / O(n) in picoprocess count from the per-process footprint in section 6.2; the paper gives no asymptotic statement.
- Can coexist with other compartmentalization systems (Composability): inferred as yes because picoprocesses run as ordinary host processes alongside normal Linux processes, but this is not stated as a coexistence claim.
- Integrates with other compartmentalization mechanisms (Composability): Unknown, not addressed in the paper.
- Primary deployment target (System Design and Operations): inferred as Server / Cloud from the multi-process server workloads and library-OS-for-the-cloud lineage; the paper names no single target.
- Deployment scale (System Design and Operations): inferred as Single device (tested up to a 48-core host), though cross-host migration via checkpoint/restore is supported.
- Required expertise level (Usability): N/A, a self-report/operational dimension an external paper-based evaluator cannot assess.
- Debugging support (Usability): N/A, self-report/operational dimension; only inferable from the fact that picoprocesses run as ordinary Linux binaries.
- Monitoring/orchestration support (System Design and Operations): N/A, not addressed in the paper.
- Initial configuration complexity (System Design and Operations): N/A, self-report/operational dimension.
- Ongoing maintenance burden (System Design and Operations): N/A, self-report/operational dimension; not discussed.

License: Open source, LGPLv3 (repo LICENSE.txt at github.com/gramineproject/graphene, formerly oscarlab/graphene; the paper does not state a license).
