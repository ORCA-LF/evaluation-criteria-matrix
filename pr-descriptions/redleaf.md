An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/redleaf.md)

## What it is
RedLeaf is a research operating system written from scratch in Rust, presented by Narayanan, Huang, Detweiler, Appel, Li, Zellweger, and Burtsev at USENIX OSDI 2020 ("RedLeaf: Isolation and Communication in a Safe Operating System"). It addresses the problem of isolating mutually distrusting kernel subsystems, device drivers, and applications from one another so that a fault in one cannot corrupt the others, and it does so without using any hardware isolation mechanism. All code runs in a single address space in ring 0, isolated purely by the Rust language's type and memory safety plus an ownership discipline.

- Compartment is a domain: a lightweight language-based isolation unit (a driver, kernel subsystem, or application), independently compiled and dynamically loaded, sharing one address space with all other domains.
- Mechanism: no hardware primitive. The software technique is the memory-safe Rust language (ownership and type safety), plus boundary marshalling via IDL-generated invocation proxies, a global shared heap with ownership-tracked remote references (RRef<T>), and exchangeable-type enforcement.
- Trust model: the TCB includes the Rust compiler, the microkernel, a small set of trusted crates that use unsafe Rust for hardware and device interfaces, the IDL compiler, and the trusted compilation environment, plus the CPU and devices (devices are trusted to be non-malicious, which is relaxable via IOMMU). Domains, written in safe Rust, are outside the TCB.

## Key takeaways
- Cross-domain invocation is very cheap: 124 cycles, against seL4's 834, VMFUNC call/reply at 396, and MPK at roughly 99 to 105 cycles per syscall, with no page-table or privilege switch. Passing an RRef adds 17 cycles each, and a call through a shadow domain costs 279 or 297 cycles.
- Language-level overhead is modest: idiomatic Rust runs about 25% slower than C on a hashtable, but C-style Rust comes within 3 to 10 cycles of C. Application benchmarks show a kv-store at 61 to 86% of the C-DPDK version, Maglev forwarding 5.3 Mpps per core, and drivers matching DPDK and SPDK.
- Fault isolation and recovery are core results: a crashing domain is cleanly terminated, its threads unwound and resources reclaimed, subsequent cross-domain calls return an RpcResult error instead of panicking, and device drivers recover transparently via shadow domains.
- The cost is that everything must be (re)written in safe Rust on a from-scratch OS: no Linux binaries, no full fork (it uses create instead), only a POSIX subset via the Rv6 personality, and side channels are explicitly out of scope (speculative-execution mitigations were disabled during evaluation).

## Where I need review
- Security, TCB size: the paper describes a "minimal" microkernel and a "small set" of trusted crates but reports no exact LOC, so the value is Unknown.
- Efficiency, Domain creation cost: domains are loaded dynamically but no domain-load latency figure is reported, so it is Unknown.
- Efficiency, Memory overhead per domain: two-level allocation is described but no MB/domain figure is given, so it is Unknown.
- Efficiency, Domain count bounded by: inferred to have no hardware or address-space limit and to be bounded by memory; the paper gives no specific count.
- Efficiency, Performance scales with domain count: cross-domain calls are O(1) per the paper; the memory O(n) characterization is inferred.
- Usability, Porting effort: not quantified in the paper (no person-days or LOC), so it is Unknown.
- Usability, Required expertise and Debugging support: marked N/A as self-report/operational dimensions an external paper-based evaluator cannot assess.
- Composability, Integrates with other mechanisms: inferred as Limited, since RedLeaf is a standalone OS and external composition is not a focus.
- Composability, Can coexist with other systems: inferred as N/A at the system level, since RedLeaf is the whole OS.
- System Design, Primary deployment target: inferred as server/datacenter from the data-plane benchmarks; the paper names no single target.
- System Design, Deployment scale: inferred as single device, since it is evaluated on one machine.
- System Design, Monitoring/orchestration, Initial configuration complexity, and Ongoing maintenance burden: marked N/A as self-report/operational dimensions not assessable from the paper.

License: Unknown, not resolvable from public sources (repo github.com/mars-research/redleaf has no LICENSE file and GitHub reports no SPDX license; the only license files in the tree belong to vendored dependencies; author contact not pursued).
