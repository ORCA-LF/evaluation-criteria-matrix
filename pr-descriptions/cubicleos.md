An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/cubicleos.md)

## What it is
CubicleOS is a compartmentalized library OS, presented by V. A. Sartakov, L. Vilanova, and P. Pietzuch in "CubicleOS: A Library OS with Software Componentisation for Practical Isolation" (ASPLOS 2021). It is built on the Unikraft library OS and addresses the problem of mutually isolating library-OS and application components, so that a malicious or vulnerable third-party component cannot read or corrupt another component's memory, while keeping ordinary function-call interfaces and full POSIX compatibility rather than rewriting everything around message passing.

- Compartment is a cubicle: a library-OS or application component (VFS, RAMFS, LWIP, allocator, the app itself) compiled as a dynamic library and isolated in its own Intel-MPK tag within a single address space.
- Mechanism: Intel MPK (one tag per cubicle, retagged via trap-and-map) plus software boundary wrappers (auto-generated cross-cubicle call trampolines), control-flow integrity to public entry points, and load-time binary scanning that rejects code containing syscall or wrpkru. Data is shared zero-copy via page-granular windows. A minor MPK hardware modification (tag-wide no-execute) is proposed for full CFI, but the prototype runs on stock MPK.
- Trust model: the TCB includes the host Linux kernel (assumed bug-free), the compilation toolchain, CubicleOS's trusted runtime (component builder, cross-cubicle trampolines, memory monitor, cubicle loader), the trusted developer who defines the isolation policy, and the hardware and virtualization environment. Third-party components (cubicles) are untrusted.

## Key takeaways
- The CubicleOS-specific TCB is small: the monitor, loader, and trampolines are about 3,000 SLOC of C plus 110 SLOC of assembly, and the builder is about 640 SLOC of Python (Table 1). The full TCB is large because it includes the host Linux kernel.
- Runtime overhead is 1.7 to 8 times versus non-isolated for SQLite (about 1.8 times average for OS-light queries, up to 8 times for OS-heavy ones) and 2 times for NGINX with 8 partitions. For the same partitioning, CubicleOS is 3 to 5 times faster than seL4/Fiasco.OC/NOVA microkernels on adding a compartment. A permission switch via wrpkru is about 20 cycles, versus about 1,100 cycles for pkey_mprotect, which CubicleOS avoids via trap-and-map.
- Porting effort is small in source terms: hundreds of SLOC per app to manage windows (SQLite 600 to 620, NGINX 390 to 400), without changing component interfaces or organization. The system supports type-unsafe C/C++ and keeps full POSIX compatibility.
- The main limits come from MPK: at most 16 cubicles without tag virtualization (8 evaluated for NGINX, 11 for SQLite), and high cross-cubicle data volume hurts (NGINX files over 64 kB see throughput halved). Argument values and call sequences are not validated, and side channels are out of scope.

## Where I need review
- Security, Compartment crashes isolated: inferred. An unauthorized access raises a memory-protection fault, but the paper describes no crash-recovery or domain-restart mechanism, so fault containment here is my reading rather than a stated claim.
- Composability, Integrates with other compartmentalization mechanisms: inferred as partial. Built on Unikraft; the paper suggests combining with SOAAP and notes incompatibility with Intel SGX, but integration is not directly evaluated.
- Composability, Can coexist with other compartmentalization systems: inferred as yes, on the basis that it runs in a VM or container alongside other systems and many cubicles coexist.
- System Design, Deployment scale: inferred as single device (per-VM/container library OS); the paper does not frame this as a scale dimension.
- Efficiency, Domain creation cost: Unknown. Cubicles are loaded by a dlopen-like loader with code scanning, but no load-latency figure is reported.
- Efficiency, Memory overhead per domain: Unknown. Per-cubicle stacks, window-descriptor arrays, and page-metadata maps exist, but no MB-per-cubicle figure is given.
- Usability, Required expertise level: N/A. Self-report/operational dimension not assessable by an external paper-based evaluator.
- Usability, Debugging support: N/A. Self-report/operational dimension; not discussed in the paper.
- System Design, Monitoring/orchestration support: N/A. Self-report/operational dimension; not discussed (a DevOps tension with the trusted builder is noted).
- System Design, Initial configuration complexity: N/A. Self-report/operational dimension.
- System Design, Ongoing maintenance burden: N/A. Self-report/operational dimension.

License: MIT (repo github.com/lsds/CubicleOS, GitHub SPDX = MIT; artifact at Zenodo DOI 10.5281/zenodo.4321431, branch ASPLOS_AE).
