An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/firecracker.md)

## What it is
Firecracker is a lightweight Virtual Machine Monitor (VMM) written in Rust that runs on KVM to create MicroVMs, minimal hardware-virtualized guests built so a cloud provider can pack thousands of mutually-distrusting serverless workloads onto one host with VM-grade isolation but container-grade density and startup. It was presented at NSDI 2020 (Agache, Brooker, Florescu, Iordache, Liguori, Neugebauer, Piwonka, and Popa) and powers AWS Lambda and Fargate. The isolation problem it addresses is safe multi-tenant co-location: protecting the host and other tenants from untrusted guest code against privilege escalation, information disclosure, and covert channels.

- Compartment is a MicroVM, a minimal hardware-virtualized guest with its own guest kernel, served by one dedicated Firecracker VMM process per VM.
- Mechanism: hardware virtualization (Intel VT-x via KVM), each guest getting its own virtual hardware, page tables, and OS kernel, plus a minimal VMM in Rust and a jailer that wraps the VMM with chroot, pid/net namespaces, dropped privileges, and a seccomp-bpf profile (24 syscalls and 30 ioctls whitelisted).
- Trust model: the TCB includes the CPU (VT-x), the host Linux kernel plus KVM (about 120k LOC), the Firecracker VMM (about 50k LOC of Rust), and host firmware. The guest kernel is explicitly untrusted and outside the TCB.

## Key takeaways
- Per-MicroVM memory overhead is about 3MB and constant regardless of configured VM size, the smallest of the three VMMs compared (Cloud Hypervisor about 13MB, QEMU about 131MB); production memory overhead runs as low as 3%, and CPU overhead is reported as negligible.
- Boot to application code is under 125ms (production as little as 150ms; 99th percentile 146ms with 1000 MicroVMs, 50 launching concurrently), supporting up to 150 MicroVMs per second per host. The limiting factor on density is host RAM, roughly 8000 128MB functions to fill 1TB.
- IO and network throughput sit notably below bare metal: guest-to-host network about 15 Gb/s (versus 44 to 47 Gb/s loopback), and 4kB block reads about 13,000 IOPS / 52 MB/s on hardware capable of 340k IOPS. Adequate for serverless, not for IO-bound bare-metal workloads.
- The TCB is small for a VMM (96% fewer lines than QEMU's >1.4M) but the whole-system TCB still includes the host kernel, so it is larger than an intra-process scheme. There are no formal proofs; validation rests on Rust memory safety, automated tests, and large-scale production use, plus public benchmark data and scripts.

## Where I need review
- Runtime efficiency, Performance scales with domain count: marked _Inferred:_ as memory O(n) in MicroVM count (fixed about 3MB each), with CPU/IO contention handled by the host scheduler and per-device rate limiters. The paper makes no asymptotic statement.
- Composability, Interaction semantics: marked _Inferred:_ as Both, request/response invoke (synchronous to the caller) over an async socket path. The paper does not frame it this way.
- Runtime efficiency, Domain switch cost and Inter-domain call latency: marked N/A by design. VM scheduling is delegated to the host Linux scheduler and cross-VM communication goes over TCP/IP sockets, so no lmbench-equivalent figure is reported.
- Runtime efficiency, General runtime overhead (SPEC): not run; only memory, IO, and boot benchmarks are given.
- Usability, Required expertise level: marked N/A as a self-report/operational dimension not assessable by an external evaluator from the paper.
- System Design and Operations, Initial configuration complexity: marked N/A as a self-report/operational dimension (design intent is minimal config, a guest kernel plus one root block device).
- System Design and Operations, Ongoing maintenance burden: marked N/A as a self-report/operational dimension (AWS uses immutable-infrastructure re-imaging for patching, which is operational context rather than a burden rating).

License: Apache 2.0 (paper §1; repo github.com/firecracker-microvm/firecracker).
