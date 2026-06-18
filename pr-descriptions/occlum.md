An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/occlum.md)

## What it is

Occlum is a library OS for Intel SGX that enables multitasking inside a single enclave, presented in "Occlum: Secure and Efficient Multitasking Inside a Single Enclave of Intel SGX" by Shen, Tian, Chen, Chen, Wang, Xu, Xia, and Yan at ASPLOS 2020 (arXiv:2001.07450v1). The problem it addresses is that running each process in its own SGX enclave is expensive, so Occlum lets mutually distrusting processes share one enclave's address space while still being isolated from each other and from the LibOS.

- Compartment is an SIP (SFI-Isolated Process): a process that shares the single address space of one SGX enclave with other SIPs, partitioned into MMDSFI domains.
- Mechanism: Intel SGX provides the enclave boundary (confidentiality and integrity against the untrusted OS/host) and Intel MPX bound registers provide SFI domain bounds; the software technique is MMDSFI (MPX-based Multi-Domain SFI) plus an independent binary verifier and coarse-grained CFI, with the LibOS written in Rust.
- Trust model: the TCB is the CPU (SGX and MPX hardware), the Occlum LibOS (Rust), and the Occlum verifier, plus the Intel/Rust SGX SDK runtime. The LLVM-based toolchain is deliberately excluded (the verifier replaces trust in it); the untrusted OS/host and the SIPs themselves are outside the TCB.

## Key takeaways

- MMDSFI averages 36.6% overhead on SPECint2006 (CPU-intensive workloads); range-analysis optimizations cut store confinement from 10.1% to 4.3% and load confinement from 39.6% to 25.5%.
- Compared with per-process-enclave Graphene-SGX, Occlum is up to 500x faster on apps and 6,600x faster on microbenchmarks; Lighttpd runs at 9% over native Linux. SIP creation via spawn (no new enclave) is 97 us for a 14 KB binary, about 1.7 ms median, scaling with binary size.
- The TCB is roughly 15,000 LOC of Rust (LibOS) plus 2,000 LOC of Python (verifier); the ~3,000 LOC C++ toolchain stays out of the TCB because the verifier statically checks every ELF for MMDSFI compliance before loading.
- Domain count is independent of MPX bound registers (only 2 are used), so many SIPs coexist in one enclave; on SGX 1.0 the enclave pages and max domain count are fixed at compile time (relaxed on SGX 2.0). Side channels are explicitly out of scope. Porting requires recompilation with the Occlum toolchain and replacing fork with spawn (about 50 LOC for GCC, 150 LOC for Lighttpd).

## Where I need review

- Security › Compartment crashes isolated: inferred yes (an MMDSFI violation triggers a CPU exception caught by the LibOS, confining the fault); the paper does not present this as a crash-recovery experiment.
- Efficiency › Performance scales with domain count: inferred memory O(n) in domain count (pages preallocated per domain); no asymptotic statement in the paper.
- Efficiency › Domain switch cost: Unknown, no lat_ctx figure; syscalls are function calls into the shared LibOS via a trampoline and no SIP context-switch latency is isolated.
- Efficiency › Memory overhead per domain: Unknown, no MB/SIP figure (each domain preallocates code region, data region, and two 4 KB guard regions, sized at compile time).
- Composability › Integrates with other mechanisms: inferred Partial (built on the Intel/Rust SGX SDK; Iago defense delegated to orthogonal work; broader composition not evaluated).
- Composability › Can coexist with other systems: inferred yes (the enclave runs as a normal SGX user process alongside host processes).
- Composability › Interaction semantics: inferred Both (synchronous syscalls/IPC plus asynchronous signals); not explicitly characterized in the paper.
- Composability › Who defines boundaries: inferred Programmer + Runtime (the programmer structures the app into processes; the LibOS loads each verified binary into a domain).
- System Design › Deployment scale: inferred Single device (per-enclave on one host); not framed as a deployment-scale dimension in the paper.
- Usability › Required expertise level, Developer Experience › Debugging support, System Design › Monitoring/orchestration support, Initial configuration complexity, Ongoing maintenance burden: all N/A as self-report/operational dimensions an external paper-based evaluator cannot assess.
- Availability › License: BSD per the 2020 artifact appendix, but the current repo is unclassified (NOASSERTION); the current license should be verified directly.

License: BSD (paper artifact appendix A.2, Zenodo DOI 10.5281/zenodo.3565239; note the current repo github.com/occlum/occlum returns SPDX NOASSERTION).
