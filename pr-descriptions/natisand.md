An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/natisand.md)

## What it is
NatiSand (Abbadini, Facchinetti, Oldani, Rossi, and Paraboschi, RAID 2023) is a component for JavaScript runtimes, prototyped in Deno, that sandboxes untrusted native code invoked by JS/TS applications. The problem it addresses is that JS backends routinely call vulnerable or malicious third-party native modules (binary programs and shared libraries) that run with the full authority of the runtime and can abuse host resources. NatiSand confines each native component's access to the filesystem, IPC, network, and syscalls per a developer-provided JSON policy, with no application code changes and no root at runtime. It is a resource-confinement sandbox, not a memory-isolation mechanism.

- Compartment is a policy-defined security context per native component: a separate OS process for executables (invoked via `command`, clone+exec) and a thread-scoped context for shared libraries (invoked via `dlopen`/FFI, running in the runtime's address space).
- Mechanism: no hardware primitive. Pure OS-level access control using Landlock (LSM) for the filesystem, Seccomp-BPF for syscalls, and eBPF for IPC and network, plus OS process isolation for executables.
- Trust model: the TCB includes the OS kernel (Landlock/Seccomp/eBPF), the V8 engine and Deno runtime, the NatiSand component, and the developer's policy. Native code (binaries and shared libraries) is untrusted. There is no memory isolation for in-process shared libraries, so the protection covers host resources, not the runtime's memory.

## Key takeaways
- It confines what host resources native code can touch (default-deny on filesystem, IPC, and network) but does not isolate memory: a vulnerable shared library still shares the runtime's address space and can corrupt it. Executables get crash isolation via a separate process; shared libraries do not.
- Overhead is low and concentrated on short-lived executables (process plus sandbox setup, roughly ms-level latency differences per microservice). The paper reports it is lower than Minijail and Sandbox2, and much lower than a Wasm-based approach; library-call and long-running overhead is low. A context pool is created at bootstrap to amortize setup.
- Adoption cost is minimal: no application or third-party module code changes, just a JSON policy (CLI-assisted), and it sandboxes unmodified native binaries and shared libraries. It needs stock Linux 5.13 or later (Landlock) plus eBPF and Seccomp, runs as a non-root user (loading eBPF needs CAP_BPF and CAP_NET_ADMIN at setup), and stacks with other LSMs (AppArmor/SELinux/SMACK) with deny taking precedence.
- Experimentally validated by reproducing 32 high-severity CVEs in native code used by popular packages and showing they are mitigated (Table 3). No formal validation.

## Where I need review
- Security, Compartment crashes isolated: inferred. Executables are isolated by a separate OS process; shared libraries are not, since they run in the runtime's address space. Not framed as a crash-recovery mechanism in the paper.
- Security, Side-channel resistance: inferred None / out of scope; the mechanism is resource access control and side channels are not addressed.
- Security, TCB approximate size (NatiSand-specific LOC): Unknown, not reported in the paper.
- Efficiency, Memory overhead per domain: Unknown; no MB-per-context figure reported.
- Efficiency, Inter-domain throughput: N/A / Unknown; no inter-domain bandwidth figure for a resource sandbox.
- Efficiency, Domain switch cost and Inter-domain call latency: N/A; native invocation is a runtime op call, not a memory-domain switch, so no lat_ctx or lat_pipe figures apply.
- Efficiency, Domain creation cost: per-executable sandbox setup adds latency, but no single creation-latency figure is reported.
- Efficiency, Performance scales with domain count: inferred; overhead is per native-invocation and the context pool amortizes setup, with no asymptotic statement in the paper.
- Composability, Interaction semantics: inferred Synchronous (`command`/`dlopen`); not explicitly characterized.
- Composability, Boundaries flexible at runtime: inferred; contexts are policy-defined at startup and fixed per run, though incremental adoption is supported.
- System Design, Deployment scale: inferred Single device (per-host runtime); not framed as a scale dimension.
- Self-report / operational dimensions marked N/A by policy: Required expertise level, Debugging support, Monitoring/orchestration support, Initial configuration complexity, and Ongoing maintenance burden, none assessable by an external paper-based evaluator.
- Structural note: this is a resource-confinement sandbox, not memory isolation, so many memory-isolation rows are N/A by category; worth confirming with the ORCA team how this maps in the matrix.

License: MIT (repo, github.com/unibg-seclab/natisand, GitHub SPDX; paper text is CC BY 4.0)
