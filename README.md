# Compartmentalization Evaluation Matrix

This repository provides a structured evaluation matrix for analyzing and comparing compartmentalization and isolation systems. It is maintained as part of ORCA, the Open Robust Compartmentalization Alliance, a vendor-neutral group under the Linux Foundation working to make isolation and compartmentalization practical in real software.

Each system is scored against the same fixed set of dimensions, so that approaches as different as hardware VMs, language runtimes, software fault isolation, and kernel compartmentalization can be compared side by side on the same terms. The goal is to help researchers, practitioners, and system designers compare tradeoffs and pick what fits their needs.

## What the matrix covers

Every evaluation fills in the same rubric, grouped into five categories:

- **Security**: isolation model, hardware and software mechanism, trust model and TCB, threat coverage, and validation.
- **Runtime Efficiency**: general overhead, domain-switch and creation costs, memory per domain, and scaling.
- **Usability and Adoptability**: hardware and OS requirements, application changes, developer effort, and licensing.
- **Composability**: how compartments interact, share data, and stack with other mechanisms.
- **System Design and Operations**: deployment target, scale, and operational considerations.

The blank template is in [`matrix.md`](matrix.md).

## How cells are sourced

Cells generally fall into one of the following:

- a cited fact from the system's paper,
- a value taken from the system's repository or documentation,
- an inferred judgment by the evaluator, marked as such,
- **N/A** for dimensions an external evaluator cannot easily assess, for example self-reported operational qualities, or
- **Unknown** for a real value that the available sources do not report.

The aim is to source or flag values rather than guess at them, though some cells are still judgment calls. Each completed evaluation includes a short "Where I need review" summary that points at the cells which are inferred or uncertain, and these are the most useful places for feedback.

## Completed evaluations

- [Cerberus](evaluations/cerberus.md): a PKU/MPK sandbox hardening framework that makes existing in-process isolation schemes non-bypassable (EuroSys 2022).
- [ERIM](evaluations/erim.md): in-process isolation of a trusted component's data using MPK plus binary inspection, without requiring CFI (USENIX Security 2019).
- [FlexOS](evaluations/flexos.md): a library OS whose isolation mechanism and compartment boundaries are chosen at build time rather than fixed by design (ASPLOS 2022).
- [HAKC](evaluations/hakc.md): kernel compartmentalization for the Linux kernel using ARM pointer authentication and memory tagging, with no added runtime TCB (NDSS 2022).
- [HFI](evaluations/hfi.md): a proposed x86-64 ISA extension for hardware-enforced in-process isolation, with Spectre safety by construction (ASPLOS 2023).
- [K23](evaluations/k23.md): a general-purpose x86-64 system-call interposition primitive providing complete, tamper-resistant mediation of an application's syscall interface, the building block that syscall-based sandboxes depend on (ACM Middleware 2025).
- [LFI](evaluations/lfi.md): Lightweight Fault Isolation, a software fault isolation scheme for AArch64 and x86-64 (ASPLOS 2024).
- [Pegasus](evaluations/pegasus.md): transparent kernel-bypass networking that fuses same-tenant programs into one address space with MPK isolation (EuroSys 2025).
- [RLBox](evaluations/rlbox.md): retrofitting fine-grain isolation of untrusted C/C++ libraries, shipped in Firefox (USENIX Security 2020).

The following three are widely deployed isolation baselines, spanning the hardware-VM, userspace-kernel, and language-VM points in the design space:

- [Firecracker](evaluations/firecracker.md): a lightweight VMM that isolates each workload in a microVM (NSDI 2020).
- [gVisor](evaluations/gvisor.md): a userspace kernel that sandboxes containers behind a syscall-intercepting guard.
- [Cloudflare Workers](evaluations/cloudflare-workers.md): per-tenant isolation using V8 isolates for JavaScript and WebAssembly.

More evaluations are in progress and are added through pull requests.

## Contributing and review

Evaluations are added one per pull request. Authors of evaluated systems are encouraged to review their own entries and suggest corrections, either by commenting on the pull request or by opening an issue. Corrections to merged evaluations are equally welcome.
