An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/smv.md)

## What it is
SMV (Secure Memory Views) is an intra-process least-privilege memory system for multithreaded applications, presented by T. C-H. Hsu, K. Hoffman, P. Eugster, and M. Payer at CCS 2016. It addresses the problem that monolithic multithreaded programs (web servers, browsers) share one address space, so a defect in one thread can corrupt or read another thread's memory, including secrets like a private key. SMV partitions the shared address space into memory protection domains and lets the programmer confine each thread to only the domains it needs, while keeping full parallel execution.

- Compartment is a Secure Memory View (SMV): a thread container holding a set of memory protection domains, each a contiguous virtual range, with per-(SMV, domain) Read/Write/Execute/Allocate privileges. Threads bound to an SMV are SMVthreads.
- Mechanism: commodity virtual-memory protection only, no special hardware. One page-table directory (pgd_t) per SMV inside the shared mm_struct, CR3 reload plus full TLB flush on a switch between SMVthreads, and privilege checks added to the kernel page-fault handler. A user-space library auto-intercepts pthread_create and malloc.
- Trust model: the TCB is the OS kernel plus CPU/hardware. SMV-specific trusted code is the SMV loadable kernel module plus the in-kernel memory-management changes (Table 3: 443 + 1,717 kernel LOC). The user-space SMV API library is explicitly untrusted, and SMVthreads distrust each other by default.

## Key takeaways
- Low measured overhead: about 2% on PARSEC 3.0 (per-benchmark 0.39% to 8.24%), and under 0.69% on Cherokee, under 0.93% on Apache httpd, and under 1.89% on Firefox.
- Very small application changes via the library-assisted model: 2 LOC for PARSEC, Cherokee, and Apache, and 12 LOC for Firefox (about 13M LOC).
- Small, potentially verifiable security-relevant TCB (the paper cites under 1,800 / under 2,000 LOC for SMV-specific code), but the whole Linux kernel and hardware remain trusted. Failure is fail-stop: an invalid access traps in the page-fault handler, the offending SMVthread gets SIGSEGV, and the whole process crashes to prevent further leakage.
- Distinguishing contribution is concurrency: parallel execution plus parallel memory tagging, which earlier process-split schemes (wedge, OpenSSH/Privtrans) and the concurrent-but-slower Arbiter lacked. Side channels (cache/timing/speculative) are out of scope, and syscall/kernel-API confinement is left to SELinux/seccomp.

## Where I need review
- Efficiency, Domain switch cost: no isolated microsecond figure. The cost is identified qualitatively as a page-table reload plus full TLB flush per SMVthread switch (mitigable only with a tagged TLB).
- Efficiency, Domain creation cost: no lat_proc figure. Creation is pthread_create plus a kernel signal to convert the thread; magnitude not isolated.
- Efficiency, Inter-domain throughput: no bw_pipe figure. Sharing is direct shared memory (a secure communication domain), so near-native, but not measured as bandwidth.
- Efficiency, Memory overhead per domain: no MB/domain figure. Per-SMV cost is one pgd_t plus kernel metadata and shadow page tables, not quantified.
- Efficiency, Domain count bounded by: inferred. Limiting factor judged to be memory for per-SMV page tables/metadata plus OS thread limits; the security test forked 1,023 SMVthreads, and no hard maximum is stated.
- Efficiency, Performance scales with domain count: inferred. Judged effectively flat in practice (about 2% even under intensive parallel alloc/free), with no asymptotic statement in the paper.
- Usability, Privileges needed: inferred. Root/kernel access to install the modified kernel and load the LKM; apps then run unprivileged.
- Composability, Interaction semantics: inferred Both. Sync via pthread synchronization on shared memory and async producer/consumer pipelines, though the paper does not frame it this way.
- Sys Design, Deployment scale: inferred Single device, since the work is per-process intra-application isolation and is not framed as a scale dimension.
- Usability, Required expertise; Sys Design, Monitoring/orchestration, Initial configuration complexity, Ongoing maintenance burden: all N/A under the self-report policy, since an external paper-based evaluator cannot assess them.

License: Public code with no declared license. The repo github.com/terry-hsu/smv exists and is described as publicly available (paper §1 footnote) but carries no LICENSE file, so GitHub detects none (all rights reserved by default; verified 2026-06-11) (repo).
