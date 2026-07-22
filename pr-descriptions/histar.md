An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/histar.md)

## What it is
HiStar is a decentralized information-flow-control (DIFC) operating system from Zeldovich, Boyd-Wickizer, Kohler, and Mazières, "Making Information Flow Explicit in HiStar," OSDI 2006. It enforces strict, decentralized information flow using Asbestos labels applied to six low-level kernel object types — threads, address spaces, segments, gates, containers, and devices. HiStar has no superuser and no fully trusted code other than a kernel of under 16,000 lines; a Unix-like environment runs almost entirely as untrusted user-level library code. It is the direct successor to Asbestos (it adopts Asbestos's labels) and the predecessor to Flume.

- Compartment is an information-flow label domain over kernel objects, not a memory-isolation sandbox. This is an OS-level MAC/DIFC point rather than an in-process MPK/SFI scheme.
- Mechanism: a small fully-trusted kernel acts as a reference monitor, checking information flow on every system call and page fault; gates provide protected control transfer. No special hardware (commodity x86-64).
- Trust model: the kernel (including in-kernel device drivers) is the only fully-trusted code; the Unix library, netd network stack, and applications are untrusted.

## Key takeaways
- Tiny TCB for an OS: 15,200 lines of C (5,700 with a semicolon) plus 150 lines of assembly, roughly 45% fewer C lines than the Asbestos kernel.
- Application-level performance is competitive with Linux and OpenBSD: a ClamAV scan of a 100 MB file is identical to Linux and shows no measurable overhead from the isolation wrapper; IPC round-trip (3.11 us) actually beats Linux.
- The main cost is process creation: fork/exec is ~7.5x slower (317 syscalls on HiStar's low-level interface vs 9 on Linux), mitigated by a spawn call that is 3x faster than fork+exec.
- Closes known covert storage channels by design; covert timing channels are explicitly unsolved, and there are no CPU quotas.

## Where I need review
- Security, TCB size: classified "Small (for an OS kernel)" from the 15,200-line figure (Section 4.1) — a judgment call on the categorical scale.
- Runtime Efficiency, Memory overhead per domain and Inter-domain throughput: both Unknown; the paper reports neither per-object memory overhead nor a pipe-bandwidth number.
- Runtime Efficiency, Performance scales with domain count: Inferred — label-operation cost grows with label size (Section 6.2 keeps labels small deliberately); no asymptotic bound is stated.
- Composability, Integrates with other mechanisms: Inferred No/limited (HiStar is a whole OS; the analog is overlaying new policy on existing software, Section 6.3).
- Self-report dimensions (required expertise, config complexity, maintenance, monitoring) are N/A per the matrix's external-evaluator policy; Debugging is kept as "Standard (gdb via a per-process debug gate)" per Section 5.2.

License: dual license — the kernel (kern/) is GPLv2 and the rest is a BSD-style license (Copyright 2005-2008 Zeldovich, Boyd-Wickizer, Stutsman); repo github.com/zeldovich/histar, COPYING. GitHub reports the repo license as NOASSERTION because it is mixed.
