An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/endoprocess.md)

## What it is
Endoprocess is a programmable, extensible scheme for intra-process isolation, presented by Yang, Huang, Kaoudis, Vahldiek-Oberwagner, and Dautenhahn (Rice / Trail of Bits / Intel Labs) at NSPW 2023. This is largely a vision/organization paper paired with an initial prototype called Little Mac, so many quantitative cells are preliminary or unreported by design. The isolation problem it addresses is least-privilege confinement of mutually-distrusting or buggy components inside a single process, with the distinguishing goal of mediating OS interfaces (closing the /proc/self/mem-class bypasses that prior MPK systems such as ERIM and Hodor leave open).

- Compartment is a subprocess: a collection of code and data with assigned privileges, isolated within one process. Memory lives in subspaces (the smallest protection unit), and privileges attach via lexical scopes (static) and dynamic scopes (runtime time-slicing).
- Mechanism: a nested self-protecting security monitor, the endokernel, that virtualizes process interfaces (mmap, mprotect, syscalls, signals) for non-bypassable mediation, plus a compiler/translation layer for annotations. The Little Mac prototype uses Intel MPK/PKU (with an MPK-gated trampoline so only the monitor can issue syscalls); the design is portable to CHERI, Intel CET, VT-X/VMFUNC, or VDom.
- Trust model: the OS, the hardware (CPU/MPK), the endokernel/Little Mac monitor, and the compiler/translation layer are trusted and assumed bug-free. Subprocesses are mutually untrusted. Side channels are out of scope.

## Key takeaways
- The reported overheads are preliminary and app-level only: privilege-separated NGINX (virtual privilege rings) runs at under 10% overhead, with NGINX TLS throughput normalized to roughly 0.9 to 1.0, and Linux apps (curl, lighttpd, nginx, sqlite3, zip) up to tens of percent (std dev under 2.4%). There is no SPEC run and no native-baseline table.
- The advances over prior MPK compartmentalization are non-bypassable mediation of OS interfaces, programmability via the translation layer, and support for real-world functionality (threads, signals, process control) that earlier monitors broke.
- Adoption is annotation-based: developers label code and data into subprocesses and assign a virtual-ring level, then recompile with the annotation-lowering toolchain and link libsep. A coarse mode wraps a whole dynamic library with a default policy (for example OpenSSL) by changing its allocator. Targets C/C++.
- It is positioned to compose with and nest inside other systems (gVisor, unikernels, V8-isolate serverless runtimes) and to coexist with process-level MAC (seccomp/SELinux/AppArmor).

## Where I need review
- Compartment crashes isolated (_Inferred:_ yes): based on the claim that a compromised subprocess cannot touch other memory, inject code, read files, or send messages, and that the endokernel can route faults/signals to a handler subprocess. Not framed as a crash-recovery experiment.
- TCB approximate size: described as a small monitor plus a large trusted OS, but no LOC figure is given for Little Mac or the endokernel.
- Domain switch cost, domain creation cost, inter-domain call latency, inter-domain throughput, memory overhead per domain, and scaling Big-O: all not reported numerically (preliminary NSPW prototype; only app-level percentage overheads in Figs. 3 to 4).
- Domain count bounded by (_Inferred:_ the 16 hardware MPK PKEYs on Little Mac): inferred from the known MPK limit; the paper cites mitigations (libmpk, EPK/VDom) but states no concrete maximum.
- Privileges needed (_Inferred:_ user-level): MPK domain switches need no kernel entry and seccomp setup is user-configurable, but this is not explicitly stated.
- Deployment scale (_Inferred:_ single device): per-process intra-application isolation, not framed by the paper as a scale dimension.
- Required expertise and Debugging support (Usability), and Monitoring/orchestration, Initial configuration complexity, and Ongoing maintenance burden (System Design): all N/A as self-report/operational dimensions an external paper-based evaluator cannot assess.

License: Unknown (no license declared in the paper; the authors' endokernel GitHub org exists but the prototype repo endokernel/endokernel-paper-ver carries no LICENSE; the org's runq=Apache-2.0 and glibc=GPL-2.0 are upstream forks, not Little Mac's own license; verified via repo)
