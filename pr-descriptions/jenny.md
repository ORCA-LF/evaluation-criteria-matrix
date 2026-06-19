An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/jenny.md)

## What it is

This PR adds `evaluations/jenny.md`, a filled-in evaluation matrix for Jenny (USENIX Security 2022; Schrammel, Weiser, Sadek, Mangard, Graz University of Technology, "Jenny: Securing Syscalls for PKU-based Memory Isolation Systems").

Jenny is a PKU-based in-process memory isolation system. Its contribution is securing the syscall interface that earlier PKU sandboxes (ERIM, Hodor, Donky) left exposed: it identifies new syscall-based attacks that break PKU isolation, derives per-domain filter rules, and designs a fast and secure syscall-interposition technique, plus secure multi-domain call gates and secure signal handling.

- Compartment is a PKU domain, page-granular and nestable (a parent domain constrains a child domain running 3rd-party code).
- Mechanism: Intel MPK plus W^X and binary scanning (ERIM's scanner), with a userspace monitor that filters syscalls and secure multi-domain call gates that protect the PKRU register. Builds on the Donky framework.
- Trust model: the in-process monitor, startup code and linker, the OS kernel, a tiny kernel module, and the CPU/MPK hardware are trusted; sandboxed code is untrusted.

## Key takeaways

- Low overhead: 0 to 5 percent for nginx (5 percent with the gzip and auth modules isolated under the pku-user-delegate technique). Like other PKU sandboxes it can sandbox unmodified binaries.
- It supports nested, per-domain, thread-specific syscall filters, and is the first PKU system with secure asynchronous signal handling and secure multi-domain x86-64 call gates.
- It is bounded by Intel MPK's 16 keys (Linux reserves up to 2): one key per domain, one for global monitor data, and one sysargs key per thread, which capped the case studies at 8 worker threads. Key virtualization scales further at the cost of only probabilistic isolation.

## Where I need review

- Crash isolation is recorded as "not framed": MPK contains a domain's memory accesses, but a fault terminates the process, and the paper treats core-dump leakage as an attack vector it mitigates rather than a recovery story. Please confirm that framing.
- Several efficiency cells (domain switch lat_ctx, inter-domain call latency and throughput, domain creation cost, MB per domain) have no absolute figure in the paper, so I recorded them qualitatively or as Unknown.
- Coexistence with other compartmentalization systems is marked Limited because Jenny owns the MPK keys and the PKRU register; please confirm.
- Usability self-report dimensions (required expertise, debugging support, config complexity, maintenance burden) are marked N/A, since they are not assessable from an external paper-based read.

License: Open source (released at github.com/IAIK/Jenny). The repository does not declare an explicit license (no LICENSE file, SPDX NOASSERTION), and the paper does not name one.
