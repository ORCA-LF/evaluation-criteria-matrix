An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/wali.md)

## What it is
WALI (The WebAssembly Linux Interface) is a thin system interface that exposes Linux's userspace syscall API directly to sandboxed WebAssembly modules, letting whole Linux software stacks be recompiled to Wasm and run portably across host ISAs. It comes from Ramesh, Huang, Titzer, and Rowe (CMU and Bosch Research), "Stop Hiding The Sharp Knives: The WebAssembly Linux Interface," arXiv:2312.03858v1 (Dec 2023). It is a primitive rather than a standalone isolation mechanism: the isolation is WebAssembly's in-process sandbox, and WALI is the syscall layer beneath it. Its deliberate design choice is to expose the raw syscall surface ("sharp knives") instead of WASI's capability model, which broadens the OS attack surface and relies on complementary OS confinement.

- Compartment is a WebAssembly module/instance, isolated in-process by the Wasm sandbox (linear-memory bounds plus control-flow integrity).
- Mechanism: no hardware primitive; software sandboxing via the WebAssembly bytecode VM (memory-safe linear memory plus CFI), with WALI acting as a boundary/marshalling layer that translates syscalls between Wasm and Linux. Requires a WALI-supporting Wasm engine (the reference implementation runs on WAMR).
- Trust model: the TCB includes the Wasm engine (WAMR), the Linux kernel (the full userspace syscall surface is exposed), and the CPU/hardware. WALI shrinks the engine's TCB by pushing higher-level APIs such as WASI out of the engine and implementing them as sandboxed layers over WALI. Wasm modules are untrusted; WALI itself does not confine which syscalls a module may invoke, leaving that to seccomp/Draco.

## Key takeaways
- WALI itself is about 2,000 lines of C, with under 100 lines of arch-specific code per platform. A WASI implementation built over WALI (libuvwasi) is over 6,000 lines in the engine, illustrating the design goal of keeping the OS-feature surface thin and out of the engine TCB. No total engine/TCB LOC is reported.
- Runtime cost is near-native for AOT-compiled Wasm, roughly 2x slower than Docker on average, and the signal-handling polling scheme adds under 10% over WALI without polling. No SPEC 2017 or lmbench figures are reported.
- The security tradeoff is explicit: exposing the full raw syscall surface widens the OS attack surface compared to WASI capabilities, so WALI must be paired with seccomp/Draco or a capability layer for OS-level confinement. The Wasm boundary still enforces memory safety, CFI, and pointer-argument translation.
- Porting is recompile-only ("trivial"), supports any language that compiles to Wasm (C/C++, Rust, Java, C#, and Lua/Python via runtimes), and provides strong Linux/POSIX compatibility (mmap, fork/exec, async I/O, signals, threads) that WASI lacks. It targets stock Linux only; non-Linux hosts are future work.

## Where I need review
- Compartment crashes isolated (Security, Resilience): marked _Inferred:_ true. A Wasm trap or memory-safety violation is contained in the module sandbox, but the paper does not frame this as a crash-recovery mechanism.
- Domain count bounded by (Runtime Efficiency): marked _Inferred:_. Multiple modules/WALI processes co-locate in one native process, bounded by host memory/process limits; no specific count is reported.
- Can coexist with other compartmentalization systems (Composability): marked _Inferred:_ true, reasoning that WALI runs as a normal Linux process and modules can co-locate or run separately.
- Boundaries flexible at runtime (Composability): marked _Inferred:_ true, since the engine loads/creates Wasm instances dynamically.
- Deployment scale (System Design): marked _Inferred:_ "single device to large scale"; the paper does not frame this as a scale dimension.
- TCB approximate size (Security): Unknown for the absolute engine/TCB size; only WALI's own ~2,000 LOC and the libuvwasi >6,000 LOC contrast are reported.
- Runtime Efficiency figures: domain switch cost, domain creation cost, inter-domain call latency, inter-domain throughput, memory overhead per domain, and performance scaling (Big O) are all Unknown, with no lmbench-style measurements in the paper.
- Porting effort for typical app (Usability): described as low/trivial, but no person-day figure is given, so the precise number is Unknown.
- Required expertise level, Debugging support, Monitoring/orchestration support, Initial configuration complexity, Ongoing maintenance burden: all N/A as self-report/operational dimensions an external evaluator cannot assess from the paper (policy 2026-06-09).
- Structural placement: WALI is treated as a primitive (like K23 and HFI) rather than a full compartmentalization system, since the isolation is WebAssembly's. This matrix placement should be confirmed with the ORCA team.

License: MIT (repo, github.com/arjunr2/WALI, GitHub SPDX = MIT; paper §1 states it is fully open source).
