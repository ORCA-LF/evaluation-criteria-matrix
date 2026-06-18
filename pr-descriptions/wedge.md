An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/wedge.md)

## What it is
Wedge is a default-deny privilege-separation system for legacy C applications, from Bittau, Marchenko, Handley, and Karp at NSDI 2008 (USENIX). It addresses the problem of stopping exploited network-facing code from leaking sensitive data (SSL private and session keys, user credentials) in monolithic servers, where coarse process-level privilege separation leaves too much accessible to a compromised component. It adds OS primitives so an application can be split into fine-grained compartments that inherit nothing by default, plus a dynamic-analysis tool (Crowbar) to make that partitioning tractable.

- Compartment is an sthread: a thread of control plus a default-deny policy naming the memory tags, file descriptors, and callgates it may access. It is implemented as a Linux process variant that inherits only the regions and FDs its policy names.
- Mechanism: standard hardware page protection enforces per-tag memory access, with no specialized primitive. Software techniques are OS-level isolation (sthreads as process variants), SELinux for syscall restriction, memory tagging at allocation time, and callgates as kernel-validated upcalls into privileged code. Partitioning is assisted by Crowbar, a Pin-based dynamic memory-access analysis used only at development time.
- Trust model: the TCB is the whole Linux kernel plus the CPU. The Wedge-specific kernel support code for sthreads and callgates is the auditable core (less than 2000 lines), and programmer-written callgate bodies are also trusted (they must not be exploitable or leak via their return value). Untrusted code runs in sthreads holding only a subset of the parent's privileges.

## Key takeaways
- Partitioning is manual but small in code terms and heavily tool-assisted: Apache/OpenSSL needed about 1700 LOC changed (0.5%) and OpenSSH about 564 LOC (2%). Enforcing one Apache boundary meant identifying 222 heap objects and 389 globals, which Crowbar surfaces for the programmer.
- Performance cost depends on workload. Partitioned Apache runs at 27% of native throughput in the worst case (all sessions cached) up to 69% (no session cache) with recycled callgates. OpenSSH adds negligible latency. Sthread creation is roughly fork latency, about 8x a pthread, and recycled callgates are about 8x faster than standard callgates.
- The Wedge-specific trusted core is small and auditable (under 2000 lines), but the whole Linux kernel remains trusted, so the system-wide TCB is large.
- It defeated real attacks in the case studies: private-key and session-key disclosure on Apache/OpenSSL, including a man-in-the-middle attack that coarse process-level privilege separation does not stop. Side and covert channels (including a residual timing channel in the SSL defense) and DoS are out of scope.

## Where I need review
- Efficiency, Domain count bounded by: inferred as OS process limits plus memory, since sthreads are processes. The paper gives no maximum count, only that each Apache request creates 2 sthreads and invokes about 8 callgates.
- Efficiency, Performance scales with domain count: inferred as O(n) in compartment instances per request from the paper's statement that overhead is proportional to sthreads and callgates created and invoked; no asymptotic claim is made.
- Efficiency, Memory overhead per domain: Unknown. No MB-per-sthread figure is reported; the paper describes private stack plus COW global data and pre-main image.
- Efficiency, Inter-domain throughput (bw_pipe): Unknown. Not measured; recycled callgates pass arguments via shared memory but throughput is not reported.
- Efficiency, Domain switch/creation/call latency: given only qualitatively (about fork, about 8x a pthread). The absolute microsecond values are in Fig. 7, which did not come through the text extraction.
- Usability, Required privileges: inferred as kernel/root to install (kernel modification), with sthreads running unprivileged.
- Usability, Required expertise level: N/A. Self-report/operational dimension, not assessable by an external evaluator.
- Composability, Can coexist with other systems: inferred yes, since sthreads are ordinary Linux processes that run alongside normal processes and standard fork/pthreads.
- System Design, Deployment scale: inferred single device, since the paper frames this as per-host application partitioning and does not treat scale as a dimension.
- System Design, Monitoring/orchestration, Initial configuration complexity, Ongoing maintenance burden: N/A. Self-report/operational dimensions, not assessable by an external evaluator.

License: Unknown (paper §9 states public release at http://nrg.cs.ucl.ac.uk/wedge/, a now-defunct 2008 UCL URL, with no license stated and no current source located).
