An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/salus.md)

## What it is
Salus (Strackx, Agten, Avonds, and Piessens, iMinds-DistriNet / KU Leuven, EAI Endorsed Transactions on Security and Safety, Vol. 2(3), e1, 2015, an extended version of the SecureComm 2013 paper) is a Linux-kernel modification that partitions a single process into mutually-distrusting compartments sharing one address space. It addresses the problem of protecting sensitive data (such as crypto keys) and containing exploited vulnerabilities within an application, without resorting to per-component processes with their marshalling cost and without special hardware. Do not confuse this with other systems also named "Salus".

- Compartment is a secure compartment: a virtual-memory region split into a public section (code plus world-readable data, immutable after init, entered only at marked entry points) and a private section (sensitive data, readable and writable only while that compartment's own public-section code is executing). All compartments of a process share one address space, granularity is developer-chosen at the function/module level (for example one class maps to one compartment).
- Mechanism: program-counter-based access control enforced by the standard MMU plus a modified page-fault handler (no MPK, MTE, CHERI, or TEE). On top sit irrevocable syscall restriction, caller/callee authentication via a signed security report (a hash of the public section plus layout), and unforgeable references implemented as (location, nonce) capabilities. A Salus C compiler/linker emits the entry-point and return-entry-point plumbing.
- Trust model: the TCB is the Salus-modified Linux kernel plus the CPU/MMU, the trusted loader, and the crypto primitives. Compartments are mutually distrusting and run at user level. Kernel-level, physical, and side-channel attacks are out of scope.

## Key takeaways
- Legacy, non-compartmentalized code runs essentially unaffected: SPECint2006 is within plus or minus 0.4% of the vanilla kernel, since the kernel change is near-free until a compartment boundary is crossed.
- Boundary crossings are expensive. A compartment call measured 4,024,227 CPU cycles, about 677 times a function call (5,944 cycles) and about 20 times a system call (193,970 cycles), driven by the cost of MMU access-right modification. This favors coarse, crossing-light partitioning.
- Real-application overhead depends on how often boundaries are crossed: 12.7% for a PolarSSL SSL server with one compartment per connection, and 21.9% falling to -0.5% for a gzip parser as I/O comes to dominate.
- Adoption is incremental and tool-assisted (annotate functions, move data into a compartment once only one compartment touches it), but the prototype is single-threaded (shared page tables), private sections are non-executable by default (JIT needs opt-in), and several syscalls (fork/clone/mmap/mprotect/personality/madvise) need special-casing.

## Where I need review
- Efficiency, Domain count bounded by (_Inferred:_): I judged the limiting factor to be address space plus per-crossing cost, since no hard maximum is stated.
- Efficiency, Performance scales with domain count (_Inferred:_): I characterized overhead as proportional to the number of boundary crossings rather than compartment count; the paper states the trade-off but gives no Big-O.
- Usability, What privileges does it need (_Inferred:_): I read this as kernel/root to install the modified kernel, with apps and compartments at user level.
- Composability, Interaction semantics (_Inferred:_): I marked synchronous (blocking calls returning via the return entry point).
- System Design, Deployment scale (_Inferred:_): I marked single device, since the system is framed as per-process intra-application compartmentalization.
- Security, TCB approximate size (Unknown LOC): no LOC figure is given for the Salus kernel delta (page-fault handler change, 6 new syscalls, conflicting-syscall checks).
- Efficiency, Domain creation cost / Memory overhead per domain / Inter-domain throughput (Unknown): not reported (compartment-call latency is reported, about 4.0M cycles).
- Availability, code/repo license (Unknown): the paper is open access under CC BY 3.0, but no code repository was stated or located.
- N/A self-report and operational dimensions: Required expertise level, Debugging support, Monitoring/orchestration support, Initial configuration complexity, and Ongoing maintenance burden are all set N/A per the self-report policy, as an external paper-based evaluator cannot assess them.

License: paper open access under CC BY 3.0 (copyright header); code/prototype availability not stated, KU Leuven research prototype.
