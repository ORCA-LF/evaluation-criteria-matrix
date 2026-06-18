An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/capsicum.md)

## What it is
Capsicum is a capability and sandbox framework for UNIX, introduced by R. N. M. Watson, J. Anderson, B. Laurie, and K. Kennaway in "Capsicum: Practical Capabilities for UNIX" (USENIX Security 2010). It addresses the problem of self-compartmentalizing monolithic UNIX applications so that a vulnerability in one component cannot reach the rest of the system, applying privilege separation and least privilege without requiring special privilege to set up. It is built on FreeBSD and was mainlined into the FreeBSD base system in version 9.0.

- Compartment is a process in capability mode (entered via `cap_enter`), denied ambient authority and access to global namespaces and holding only explicitly delegated capabilities; a "logical application" is a group of such sandboxed processes.
- Mechanism: a pure OS-level technique with no hardware primitive. Capability mode plus capabilities (rights-masked file descriptors, around 60 rights masks) plus process descriptors (`pdfork`); cross-compartment calls go over RPC and message passing on UNIX domain sockets with hand-coded marshalling, alongside direct syscalls on delegated capabilities.
- Trust model: the TCB is the full FreeBSD kernel (capability checks in `fget` and `namei`), the `libcapsicum` runtime and the capability-aware runtime linker `rtld-elf-cap`, plus the CPU and hardware. Sandboxed component code is untrusted and can act only through the capabilities delegated to it.

## Key takeaways
- Capability-wrapped syscalls are cheap: roughly 0.05 to 0.07 microseconds added, with `fstat` +10.2%, `read` +4.4% to 8.9%, `write` +10.0%, `cap_new` +50.7%, and `cap_enter` itself at 1.22 microseconds. There is no SPEC measurement.
- Cross-compartment RPC over a UNIX domain socket costs about 309 microseconds per pingpong roundtrip (+15% vs fork). Sandbox creation is the expensive part: create plus RPC plus destroy is about 1,509 microseconds (1.5 ms, +85.9% vs fork plus exec), and a gzip sandbox launch is a constant 2.37 ms.
- Adoption effort is incremental and scales with depth: a bare `cap_enter` is a 2-line change, tcpdump about 10 lines, dhclient about 2 lines, Chromium about 100 lines, and gzip 409 lines (about 16%) for full RPC proxying. No special privilege is required, a key advantage over chroot and setuid sandboxes.
- It is production code (FreeBSD base system since 9.0) and composes with existing DAC/MAC; capability rights narrow monotonically on delegation. The main costs are that compartmentalization turns local development into distributed development (harder debugging, hand-coded marshalling), capability mode restricts the syscall surface, and there is no enforced IDL for argument validation.

## Where I need review
- Efficiency, Inter-domain throughput: Unknown. There is no bw_pipe figure; the design note is that file-descriptor (capability) delegation gives native UNIX I/O, with gzip converging within 5% of native by about 512 KB.
- Efficiency, Memory overhead per domain: Unknown. No MB-per-sandbox figure is given; `libcapsicum` adds about +9% to fork from extra VM mappings.
- Security, TCB approximate size: marked Large because the full FreeBSD kernel is in the TCB, but the Capsicum-specific delta is Unknown, not quantified as a single LOC figure.
- Efficiency, Domain count bounded by: Inferred to be bounded by OS process resources (each sandbox is a process); no specific cap is stated.
- Efficiency, Performance scales with domain count: Inferred linear, O(n) in process count; not stated as such in the paper.
- System Design, Primary deployment target: Inferred Desktop / Server from the example applications (browsers, network daemons, CLI tools).
- System Design, Deployment scale: Inferred Single device, since it is a per-host OS mechanism.
- Required expertise level: N/A, a self-report/operational dimension not assessable by an external evaluator from the paper.
- Monitoring/orchestration support: N/A, self-report/operational dimension.
- Initial configuration complexity: N/A, self-report/operational dimension.
- Ongoing maintenance burden: N/A, self-report/operational dimension.
- Note: Debugging support is normally an N/A self-report dimension but is kept and cited here because the paper explicitly discusses it (extended `procstat`, `ktrace`/DTrace integration, `ENOTCAPABLE` errno).

License: Open source, BSD (stated in the paper's availability section; Capsicum is part of the FreeBSD base system under 2-clause BSD since FreeBSD 9.0).
