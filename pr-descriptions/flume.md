An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/flume.md)

## What it is
Flume is the first design and implementation of process-level decentralized information-flow control (DIFC) on stock operating systems, from Krohn, Yip, Brodsky, Cliffer, Kaashoek, Kohler, and Morris, "Information Flow Control for Standard OS Abstractions," SOSP 2007. Flume runs as a user-level reference monitor on Linux (and OpenBSD): a confined process is blocked from issuing most system calls directly and instead routes them as IPC to the monitor, which enforces per-process secrecy and integrity labels plus decentralized capabilities. Its endpoint abstraction attaches labels to ordinary file descriptors so DIFC works with pipes, sockets, and files. It shares the DIFC lineage of Asbestos and HiStar but, unlike those dedicated DIFC kernels, targets an existing OS.

- Compartment is a confined process's DIFC label domain. This is an OS-level MAC/DIFC point, not an in-process memory-isolation sandbox.
- Mechanism: a user-level reference monitor plus helpers (spawner, file servers, tag registry, Flume libc); system-call interposition is done with a small Linux Security Module (or systrace on OpenBSD).
- Trust model: the entire underlying OS kernel and user space is in the TCB, plus Flume's own components. The paper flags this TCB as "many times larger than those of dedicated DIFC kernels."

## Key takeaways
- The Flume-specific TCB is about 21,500 lines of C++ (reference monitor ~14,000, tag registry ~3,500, file server ~2,500, spawner ~1,000, LSM ~500, patch <100), but the whole Linux/OpenBSD kernel is also trusted.
- Demonstrated by porting the 91,000-line MoinMoin wiki to FlumeWiki, moving security into a ~1,000-line isolated module and adding an end-to-end integrity policy that could not exist without DIFC.
- Cost is user-space overhead: FlumeWiki is 43% slower on read throughput and 34% slower on write; per-interposed-syscall overhead is 35-286 us and IPC round-trip is 8.2x Linux.
- Only explicit label changes are allowed (adopting HiStar's fix), and covert timing channels are assumed absent (not addressed).

## Where I need review
- Security, License: marked "research prototype, license not confirmed." The "flume" name is heavily overloaded (Apache Flume, a Rust crate), so I could not confirm the specific license for the MIT CSAIL DIFC project (flume.csail.mit.edu). Author/maintainer confirmation would resolve this.
- Runtime Efficiency, Inter-domain throughput: the paper reports IPC bandwidth over a 20 GB transfer as a slowdown multiplier (Figure 8) but I did not transcribe an exact MB/s.
- Runtime Efficiency, Memory overhead per domain: Unknown (not reported per confined process).
- Usability, Debugging support: characterized as mixed — unsafe messages are dropped silently, whereas endpoint-label-change failures are reported as errors (Sections 3.3, 4.2).
- Self-report dimensions (required expertise, config complexity, maintenance, monitoring) are N/A per the matrix's external-evaluator policy.

License: open-source research prototype (MIT CSAIL, flume.csail.mit.edu); the specific license was not definitively confirmed and is flagged for verification.
