An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/ufork.md)

## What it is
µFork (uFork) is a single-address-space OS design that supports transparent POSIX `fork`, presented by J. A. Kressel, H. Lefeuvre, and P. Olivier at SOSP 2025 (arXiv:2509.09439v2). The problem it addresses is that single-address-space OSes (SASOSes, such as unikernels) are lightweight but cannot ordinarily provide POSIX `fork` or process-grade isolation, since all code shares one address space. µFork copies a parent's memory to a new location within that same address space and uses CHERI hardware capabilities to keep the copies isolated, so fork-based applications run unmodified while still getting process-equivalent isolation.

- Compartment is a µprocess: an emulated POSIX process living in a single shared address space, isolated from other µprocesses and from the kernel.
- Mechanism: CHERI hardware capabilities for intra-address-space isolation, memory tags for pointer identification and relocation, and sealed capabilities for trapless syscalls; software side uses position-independent code (PIC/PIE) and a novel Copy-on-Pointer-Access (CoPA) strategy. Prototyped in Unikraft on ARM Morello.
- Trust model: the TCB is the Unikraft SASOS kernel (including the µFork implementation) plus the CHERI CPU hardware. All user code (µprocesses) is untrusted; the kernel distrusts every µprocess. Side channels and transient execution are out of scope.

## Key takeaways
- Fork is fast and cheap: a µprocess forks in 54 µs, 3.7x faster than CheriBSD's traditional fork (197 µs) and 198x faster than Nephele's VM-based fork (10.7 ms). Memory per minimal µprocess is about 0.13 MB, versus 1.6 MB for Nephele and 0.29 MB for CheriBSD.
- Fork-heavy workloads see net gains: FaaS throughput +24% versus CheriBSD, Nginx (single core) +9% requests, Redis save 1.4 to 1.9x faster, and a forked Redis process at a 100 MB DB uses 6 MB with CoPA versus 56 MB on CheriBSD.
- User/kernel transitions are trapless and single-privilege-level via sealed capabilities; the Unixbench pipe IPC benchmark runs 245 ms versus 419 ms on CheriBSD.
- The cost is the CHERI hardware requirement and recompilation: applications need no source changes but must be compiled PIC/PIE in CHERI pure-capability mode. CHERI pure-cap overheads are projected reducible to 1.8 to 3% in future hardware. Prototype limitations include SMP serialized behind a big kernel lock and virtual-address fragmentation.

## Where I need review
- Availability › License: code repo (github.com/flexcap-project/ufork) has no LICENSE file; every license file in the tree is a vendored third-party dep, and µFork's own code carries none. The paper is CC BY 4.0, but that does not cover the code. Marked Unknown after a deep search; author contact not pursued.
- Efficiency › Inter-domain throughput: marked Unknown; no bw_pipe-style throughput figure is reported.
- Efficiency › Performance scales with domain count: inferred as memory O(n) in µprocess count, with CoPA reducing per-fork cost (up to 89x versus synchronous copy); the paper gives no asymptotic statement.
- Composability › Integrates with other compartmentalization mechanisms: inferred Partial; MPK, MTE, and Mondrian are discussed as alternatives under the parameterized-isolation design (R4) but not implemented.
- Composability › Can coexist with other compartmentalization systems: inferred yes, since µprocesses coexist in one address space and the SASOS runs bare-metal or under bhyve.
- Composability › Interaction semantics: inferred Both (synchronous syscalls/pipes plus asynchronous signals); not explicitly characterized.
- Sys Design › Deployment scale: inferred Large scale (FaaS/cloud with many lightweight µprocesses); not framed as a scale dimension in the paper.
- Usability › Required expertise level: N/A, a self-report/operational dimension not assessable by an external paper-based evaluator.
- Developer Experience › Debugging support: N/A, self-report/operational dimension; also not discussed in the paper.
- Sys Design › Monitoring/orchestration support: N/A, self-report/operational dimension; not discussed.
- Sys Design › Initial configuration complexity: N/A, self-report/operational dimension.
- Sys Design › Ongoing maintenance burden: N/A, self-report/operational dimension.
- Security › TCB approximate size: µFork adds about 3,000 LoC (7 KLoC total kernel changes including the CHERI + bhyve Unikraft port), but the paper gives no full kernel LOC, so the total TCB size is not precisely stated.

License: Unknown (repo github.com/flexcap-project/ufork has no LICENSE file; the paper text is CC BY 4.0 but does not cover the code).
