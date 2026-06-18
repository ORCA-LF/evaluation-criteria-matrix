An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the system's authors and maintainers. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/gvisor.md)

## What it is
gVisor is Google's userspace application kernel for containers. Because there is no single flagship paper, this entry was built from the official gvisor.dev documentation (architecture and security guides, performance guide), the gVisor security-basics blog, and the GitHub repo. The isolation problem it addresses is running untrusted container workloads in multi-tenant environments without giving them direct access to the host kernel: instead of passing syscalls through to the host, gVisor intercepts them and services them in a separate userspace kernel, shrinking the reachable host-kernel attack surface.

- Compartment is a sandboxed container: one sandbox per container, with the untrusted workload running against the Sentry rather than the host kernel.
- Mechanism: a from-scratch re-implementation of the Linux syscall ABI in memory-safe Go (the Sentry), plus syscall interception (the default Systrap platform, or ptrace, or the optional KVM platform that uses hardware virtualization), a Gofer process that mediates filesystem access, and seccomp-bpf confinement of the Sentry and Gofer. The default interception path uses no special hardware primitive.
- Trust model: the TCB includes the Sentry and Gofer (Go), the runsc OCI runtime, the host Linux kernel (a reduced but non-zero surface remains reachable), and the CPU/firmware. The untrusted container workload is outside the TCB; no syscall is passed straight through to the host.

## Key takeaways
- The Sentry exposes roughly 240 syscalls to the workload, of which about 100 can reach the host, while the Sentry's own host footprint is restricted to about 70 syscalls via seccomp (the Gofer even fewer). This is the concrete sense in which the host-kernel attack surface is reduced versus a plain runc container.
- Overhead is structural rather than incidental: every intercepted syscall pays a userspace round-trip into the Sentry, and the reimplemented netstack and VFS make syscall-heavy and IO/network-heavy workloads noticeably slower than runc, while compute-bound workloads run near-native. The docs frame this qualitatively and do not publish SPEC, lmbench, or a single MB-per-sandbox figure.
- Porting effort is near zero: gVisor runs unmodified Linux container images as a drop-in runsc OCI runtime under Docker, containerd, and Kubernetes, subject to partial Linux ABI coverage (some syscalls, ioctls, raw sockets, and parts of /proc are deliberately unimplemented).
- Hardware side channels are explicitly out of scope, and there is no formal validation; assurance rests on Go memory safety, the structural no-passthrough design, and an extensive syscall-compatibility test suite. It is in production across GKE Sandbox, Cloud Run, and App Engine.

## Where I need review
This entry is doc-based rather than paper-based, so several efficiency cells are qualitative or platform-dependent where a paper would normally give hard numbers.

- TCB approximate size (Security): marked Medium with syscall counts but no authoritative LOC figure for the Sentry/Gofer codebase.
- Domain switch cost, domain creation cost, inter-domain call latency, inter-domain throughput, and memory overhead per domain (Runtime Efficiency): no lmbench or MB-per-sandbox numbers in the docs; described qualitatively and as platform-dependent.
- Compartment crashes isolated (Security): inferred from one-sandbox-per-container design, not stated as a recovery experiment.
- Privileges required (Usability): inferred as container-runtime privileges to set up namespaces/seccomp, with the sandbox itself running unprivileged.
- Domain count bounded by and performance scales with domain count (Runtime Efficiency): inferred as host memory plus per-sandbox Sentry overhead, memory O(n) in sandbox count; no asymptotic statement in the docs.
- Can stack effectively (Composability): inferred yes from many coexisting sandboxes and the Systrap platform being suited to run inside a VM.
- Interaction semantics (Composability): inferred Both (synchronous syscall emulation plus async I/O); not explicitly characterized.
- Required expertise level and Debugging support (Usability), and Initial configuration complexity and Ongoing maintenance burden (System Design and Operations): N/A under the self-report policy, since an external evaluator cannot assess these operational dimensions.

License: Apache 2.0 (gVisor GitHub repo, github.com/google/gvisor).
