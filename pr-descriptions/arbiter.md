An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/arbiter.md)

## What it is
Arbiter is a 2015 system for fine-grained, data-object-level privilege separation among mutually-distrusting threads inside a single multithreaded process, from J. Wang, X. Xiong, and P. Liu, published at USENIX ATC 2015 ("Between Mutual Trust and Mutual Distrust: Practical Fine-grained Privilege Separation in Multithreaded Applications"). It addresses what the paper calls the second privilege-separation problem: protecting per-connection or per-user data when worker threads in one process distrust each other, so that the compromise or malfunction of one thread cannot read or corrupt another principal's in-process data.

- Compartment is an Arbiter thread, a normal-looking thread that carries a DIFC-inspired label plus ownership determining its per-object read/write/no access.
- Mechanism: standard x86 page-table protection bits and the MMU (no special hardware), with each thread holding its own page tables over a shared Arbiter Secure Memory Segment (ASMS), a permission-oriented allocator that co-locates same-accessibility objects on a page, a page-fault reference monitor, and an out-of-process Security Manager that mediates all allocation, thread-creation, and policy operations over RPC.
- Trust model: the TCB includes the Linux kernel, the CPU, the Arbiter Security Manager, the ASMS library, and the ASMS kernel management and page-fault code (the shaded TCB in Fig. 2). The application worker threads are untrusted with respect to each other, and the app is also assumed to be confined at the OS level by SELinux/AppArmor.

## Key takeaways
- Source effort is small: about 0.5% of LOC on average to annotate (Memcached 100 LOC/0.5%, Cherokee 188/0.3%, FUSE 129/1.6%), with the porting time dominated by understanding the code rather than editing it.
- Application-level overhead is modest while microbenchmarks are not: throughput loss is under 10% (Memcached ~5.6%, Cherokee 1.8-3.0%, FUSE 7.4%), CPU utilization is 1.29-1.55x, and RSS overhead is 3.9-6.2%, but the Security Manager RPC round-trip is 5.84 us on every alloc/free/thread-create/policy op and thread creation is 145.33 us versus 91.45 us native (1.59x).
- Enforcement is fail-stop on commodity hardware: an illegal ASMS access traps to the page-fault handler and sends SIGSEGV to the faulting thread; the paper notes a handler could instead drop the connection, but the prototype crashes.
- The TCB is large (the whole kernel is trusted), and no LOC figure is given for the Arbiter-specific TCB, in contrast to smv which reports under 1,800 LOC for the same core idea.

## Where I need review
- License (Availability): Unknown. The paper states no repository or license, and no public Arbiter repo was located (GitHub search, 2026-06-11).
- TCB approximate size (Security): no LOC figure is reported for the Arbiter TCB; characterized as Large because the whole kernel plus the Security Manager and ASMS components are trusted.
- Domain switch cost, inter-domain throughput, and memory overhead per domain (Runtime Efficiency): not reported numerically. A per-Arbiter-thread context switch causes a TLB flush like a process switch, but there is no lat_ctx us figure, no bw_pipe bandwidth, and RSS is given as a percentage rather than MB/thread.
- Domain count bounded by (Runtime Efficiency): inferred. Limiting factor taken to be per-thread page tables, TLB pressure, and the single-lock allocator; no hard maximum is stated.
- Performance scales with domain count (Runtime Efficiency): inferred. Memory-allocation cost taken as roughly O(n) in thread count (allocations propagate to every Arbiter thread); the paper makes no asymptotic claim.
- Privileges needed (Usability): inferred that kernel/root is needed to install the modified kernel while the app and Security Manager run unprivileged.
- Can coexist with other systems and interaction semantics (Composability): inferred. Coexistence inferred from running as an ordinary process under OS access control; interaction semantics inferred as both synchronous (Security Manager RPC) and asynchronous (concurrent threads sharing memory).
- Deployment scale (System Design): inferred Single device, since the system is per-process intra-application separation and not framed as a scale dimension.
- Required expertise and Debugging support (Usability): N/A, self-report/operational dimensions not assessable by an external paper-based evaluator.
- Monitoring/orchestration support, Initial configuration complexity, and Ongoing maintenance burden (System Design and Operations): N/A, self-report/operational dimensions not assessable by an external paper-based evaluator.

License: Unknown (paper states no repository or license; no public repo found via GitHub search, 2026-06-11)
