An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/lfi.md)

## What it is
LFI (Lightweight Fault Isolation) is a software fault isolation (SFI) scheme for ARM64, presented by Z. Yedidia in "Lightweight Fault Isolation: Practical, Efficient, and Secure Software Sandboxing" at ASPLOS 2024 (DOI 10.1145/3620665.3640408). It addresses the problem of running many untrusted programs in one address space with low CPU overhead and fast context switches, targeting cloud, serverless, and edge settings, on commodity ARM64 rather than specialized hardware.

- Compartment is a sandbox: an aligned 4 GiB region in a shared address space, holding a process-like untrusted program, with tens of thousands of such sandboxes coexisting in one address space.
- Mechanism: pure software fault isolation with no hardware isolation primitive. Guard instructions are added to loads, stores, and jumps (exploiting ARM64 addressing modes for roughly 0 to 1 cycle guards), a static machine-code verifier confirms the invariants on the final binary, and standard page protections (W xor X) are set once at init.
- Trust model: the TCB is the static verifier (core is 300 LOC of Rust, plus its disassembler, ELF reader, and instruction-definition dependencies), the runtime, and the OS and CPU. The compiler and the LFI assembly transformer (about 1,500 LOC) are not trusted, because the verifier checks the final machine code. This keeps LLVM's roughly 2M LOC out of the TCB.

## Key takeaways
- Runtime overhead on SPEC CPU 2017 is a geomean of 6.4% on Apple M1 and 7.3% on GCP T2A for full isolation (O2), and about 1% for the lighter stores-and-jumps-only ("no loads") mode, with the worst single benchmark at 17% (leela). This is roughly half the overhead of the Wasm engines compared against (WAMR 18 to 22%, Wasm2c-pinned 16%, Wasmtime 47 to 67%).
- Context switches are cheap: cross-sandbox yield is 17 to 18 ns (about 50 cycles) with no hardware mode or page-table switch, and a runtime-mediated syscall is 22 to 26 ns, about 6x faster than Linux's 129 to 160 ns. Pipe round-trip is 46 to 48 ns versus 1504 to 2494 ns on Linux and 22899 ns on gVisor.
- Each sandbox reserves 4 GiB of virtual address space (plus 48 KiB guard regions) but most of it stays unmapped, so physical use tracks actual usage. Code grows about 12.9% (text) and 8.3% (binary). The domain count is bounded by virtual address space: about 64Ki sandboxes in 48-bit userspace, up to 128Ki using kernel address space.
- Spectre sandbox-breakout is mitigated by construction because LFI uses no fine-grained CFI and all jumps are data-flow-guarded, but full cross-sandbox or host poisoning mitigation needs ARM's CSV2_2 extension (Cortex-X2 and later, not universal). The scheme is ARM64 only and requires recompilation through the LFI toolchain with an SFI-instrumented musl toolchain.

## Where I need review
- Security › Compartment crashes isolated (Inferred): marked yes because out-of-bounds accesses hit unmapped guard regions and trap to the runtime, but the paper does not frame this as an explicit recovery experiment.
- Runtime Efficiency › Performance scales with domain count (Inferred): per-sandbox runtime state is constant, and I inferred memory is O(n) in virtual space.
- Composability › Integrates with other compartmentalization mechanisms (Inferred): marked yes based on running inside a VM or container and using ARM CSV2_2, plus discussion of future hardware-SFI support.
- Runtime Efficiency › Domain creation cost (Unknown end-to-end): the verification rate is given (about 34 MB/s, each SPEC binary under 0.3 s) but there is no end-to-end sandbox-creation latency.
- Usability › Required expertise level (N/A): self-report or operational dimension, not assessable by an external paper-based evaluator.
- Usability › Debugging support (N/A): self-report or operational dimension, and not directly discussed in the paper.
- System Design › Monitoring/orchestration support (N/A): self-report or operational dimension; only a runtime scheduler is described, no orchestration.
- System Design › Initial configuration complexity (N/A): self-report or operational dimension.
- System Design › Ongoing maintenance burden (N/A): self-report or operational dimension.

License: MPL 2.0 (paper Artifact A.2 "Code licenses: MPL 2.0", confirmed at repo github.com/zyedidia/lfi; paper text licensed CC-BY 4.0).
