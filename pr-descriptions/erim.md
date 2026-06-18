An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/erim.md)

## What it is
ERIM is an in-process isolation technique presented by Vahldiek-Oberwagner, Elnikety, Duarte, Sammler, Druschel, and Garg at USENIX Security 2019. It addresses the problem of protecting a trusted component's sensitive data (crypto session keys, CPI safe-region metadata, a managed runtime) from untrusted code that is co-linked into the same address space, while keeping the cost of crossing the boundary low enough to tolerate very high switching rates. Its central idea is to make Intel MPK safe to use without CFI by inspecting binaries to forbid any exploitable WRPKRU or XRSTOR instruction.

- Compartment is a trusted component T with an exclusive data region M_T (an Intel MPK domain), isolated from the untrusted rest of the process U within a single address space.
- Mechanism: Intel MPK (up to 16 domains via the per-core PKRU register, switched in userspace with WRPKRU) combined with userspace call gates and build-time binary inspection/rewriting that forbids exploitable WRPKRU/XRSTOR occurrences. It is explicitly not SFI and does not require CFI.
- Trust model: the TCB includes the CPU/MPK hardware, the OS kernel, the small ERIM library (about 569 LoC, including binary inspection) linked into each process, and the trusted component T itself (assumed vulnerability-free, must scrub registers and stack and not leak secrets). U is fully untrusted and may contain memory-corruption and control-flow-hijack bugs. The OS loader is trusted to run ERIM's init first.

## Key takeaways
- Switching is cheap: a userspace call-gate roundtrip is two WRPKRU instructions (11 to 260 cycles each) plus register ops and one branch, adding 55 to 80 cycles over a no-switch call. The paper reports this as 6.3x, 13.5x, and 3x lower overhead than MPX-SFI, lwC, and VT-x respectively, and Table 2 shows a VMFUNC EPT switch at 332 cycles and an lwC switch at 6,050 cycles for comparison.
- Application overhead is low: SPEC CPU 2006 geomean 4.3% (under 6.7% on all tests), NGINX at most 4.8% when isolating all session keys, and CPI within 2.5% except for perlbench/omnetpp/xalancbmk (up to 17.9%). Per-switch overhead is roughly 0.07 to 1.0% per 100,000 switches/s at 2.6 GHz.
- Adoption can require no application changes: a binary-only path wraps an unmodified dynamic library via LD_PRELOAD. Finer isolation uses source annotations (OpenSSL+SQLite at 437 LoC) or a compiler pass (CPI at 300 LoC in LLVM). It runs on stock Linux 4.6 or later with no kernel changes required.
- Limits: caps at 16 MPK domains, is x86-only, conflicts with applications already using MPK, requires DEP/W^X, and leaves side channels out of scope. Register/stack scrubbing and entry-point correctness are pushed onto the trusted component.

## Where I need review
- Composability › Integrates with other compartmentalization mechanisms: marked Inferred (Partial). Based on libmpk for more than 16 domains being future work, compatibility with existing side-channel defenses, and orthogonality to auto-partitioning tools.
- Composability › Boundaries flexible at runtime: marked Inferred (No). The T/U partition and the reserved M_T domain are fixed at build/init time; pages can be assigned into M_T at runtime but the boundary itself is static.
- System Design › Primary deployment target: marked Inferred (Server / Other). The paper targets network-facing services, language runtimes, and general in-process isolation, not one specific target.
- System Design › Deployment scale: marked Inferred (Single device). It is per-process, in-process isolation and not framed as a deployment-scale dimension.
- Efficiency › Inter-domain throughput: Unknown. Data is shared zero-copy via a buffer in M_U, but no throughput or bandwidth figure is reported.
- Efficiency › Memory overhead per domain: Unknown. M_T holds an application-defined memory pool; no MB/domain figure is reported.
- Security › Compartment crashes isolated: Unknown. ERIM is a confidentiality/integrity mechanism, not a crash-isolation one; violations terminate the program and the paper does not frame domain crash isolation or recovery.
- Usability › Required expertise level: N/A, self-report/operational dimension not assessable by an external evaluator from the paper.
- Usability › Debugging support: N/A, self-report/operational dimension.
- System Design › Monitoring/orchestration support: N/A, self-report/operational dimension.
- System Design › Initial configuration complexity: N/A, self-report/operational dimension.
- System Design › Ongoing maintenance burden: N/A, self-report/operational dimension.

License: Open source, specific license unclear. The paper points to a project at gitlab.mpi-sws.org (§6 footnote); the GitHub mirror vahldiek/erim reports SPDX NOASSERTION (no detected license). (repo)
