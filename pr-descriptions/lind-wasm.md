An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/lind-wasm.md)

## What it is
Lind / lind-wasm is a WebAssembly-based POSIX sandbox that runs unmodified POSIX applications inside isolated execution contexts, paired with Grates, a framework that makes system-call policy composable in userspace. It is described in the anonymous submission "Grates: Composable System Call Policies in Userspace" (under review, implementation anonymized as "GrateOS"; the real project is the NYU Lind project, repo Lind-Project/lind-wasm). The problem it addresses is that existing syscall-policy mechanisms (seccomp, eBPF, ptrace, namespaces) cannot be written as independent, reusable modules that stack together, and they often require kernel changes or elevated privilege. Grates lets a deployer compose separately-written policy modules over an unmodified app with no kernel modification and no elevated privilege.

- Compartment is a cage: an isolated execution context with its own memory, threads, signal handlers, and file-descriptor namespace that behaves like a process. All cages share one host address space but each has a disjoint WebAssembly linear memory and a per-cage programmable system-call table. A grate is a cage that has interposed on another cage's syscalls via the 3i interface.
- Mechanism: software fault isolation via WebAssembly (no hardware primitive in the prototype), plus a Rust runtime and syscall interposition through the 3i dispatch table. Apps are compiled to wasm32 and linked against an interception-aware glibc that routes syscalls into the per-cage 3i table. The architecture is substrate-independent and could instead run atop MPK or processes.
- Trust model: the TCB includes the GrateOS/lind-wasm runtime (modified Wasmtime plus the Rust components 3i, RawPOSIX as the trusted terminal syscall shim, fdtables, and cage), the modified glibc, the host OS kernel, and the CPU. Individual grates are isolated policy modules and are not part of each other's TCB.

## Key takeaways
- Composition is the core contribution and is cheap: grates compose by linear sequencing, clamping (conditional dispatch), and teeing (call duplication) as independently written reusable modules, and each per-grate hop adds roughly 0.162 microseconds (geteuid goes from 0.207 microseconds via RawPOSIX to 0.369 microseconds through one grate), dominated by the Wasm trampoline rather than 3i lookup.
- General compute overhead comes from the Wasm substrate: SPEC CPU 2017 ranges from 1.09x (lbm) to 5.43x (gcc), with 6 of 9 benchmarks within 3x of native; high-indirect-call workloads pay the most.
- Common operations are near native, but process creation is not: context switch is 2.70 microseconds vs 2.55 native, IPC and file IO are close to native (file-read bandwidth 5,487 MB/s vs 6,566 native), while fork is about 25x native (10,117 microseconds vs 422) and exec is similar, due to Asyncify stack unwinding plus Wasm module instantiation.
- Scale is bounded by virtual address space: each cage reserves a 4 GB virtual window, giving roughly 60 simultaneous cages on a 256 GB / 48-bit host and up to about 16K using the full 128 TB userspace VA range. Isolation is enforced by construction (Wasm code cannot emit native syscalls, so no grate can be bypassed), with documented gaps in POSIX path semantics for descriptor-based operations.

## Where I need review
- Security › Compartment crashes isolated (Inferred): set to yes based on disjoint linear memory and Wasm traps on out-of-bounds/type violations; the paper does not frame this as a crash-recovery experiment.
- Security › TCB approximate size (Unknown LOC): no single TCB-LOC figure exists; the runtime plus Wasmtime plus modified glibc form the TCB, and the per-grate sizes (308 to 2,137 LoC) are policy code, not TCB.
- Security › Side-channel resistance (Inferred): marked none / out of scope because side channels are not discussed in the paper.
- Efficiency › Inter-domain throughput (substituted metric): bw_pipe is not reported; file-read bandwidth (5,487 MB/s) is given instead. Memory overhead per domain is a 4 GB VA reservation, not resident memory.
- Efficiency › Performance scales with domain count (Inferred): characterized as flat per hop (about 0.162 microseconds/grate) from the paper's per-hop measurement.
- Composability › Interaction semantics (Inferred): marked both based on synchronous syscall dispatch plus asynchronous signal delivery and teeing's secondary path.
- System Design › Deployment scale (Inferred): marked single device; the paper does not frame deployment as a scale dimension.
- Usability › Required expertise and Debugging support, and System Design › Monitoring/orchestration, Initial configuration complexity, and Ongoing maintenance burden: all marked N/A under the self-report policy, since these are operational dimensions an external paper-based evaluator cannot assess.

License: Apache 2.0 (repo Lind-Project/lind-wasm, root LICENSE; verified 2026-06-13)
