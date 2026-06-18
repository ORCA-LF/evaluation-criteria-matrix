An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/drawbridge.md)

## What it is
Drawbridge is the Windows 7 library OS from "Rethinking the Library OS from the Top Down" (ASPLOS 2011) by Porter, Boyd-Wickizer, Howell, Olinsky, and Hunt (Microsoft Research, Stony Brook, MIT). It refactors a full commercial OS personality (the Win32 API plus NT-kernel emulation) into a user-space library that runs inside each application's own address space, so each app can be isolated from the host and from other apps at roughly VM strength but at a fraction of the cost.

- Compartment is a Drawbridge process (picoprocess): one application plus its own private copy of the library OS, running in a single isolated address space.
- Mechanism: standard per-process hardware memory protection plus a narrow ABI (about 40 downcalls) mediated by a security monitor (dkmon) that virtualizes host resources and enforces a manifest-based whitelist; no specialized hardware primitive.
- Trust model: the security monitor (dkmon/dkpal), the host OS kernel, and the CPU are trusted. The library OS is untrusted and runs in the application's address space; the security boundary is the ABI. The application is untrusted.

## Key takeaways
- Isolation is demonstrated empirically rather than proven: registry-wiping malware affected only its own instance, a keylogger could not see other apps' keystrokes, and all five Keetch IE-protected-mode escape vectors were mitigated. There are no formal proofs; the argument is that the small ABI makes the boundary auditable.
- The trusted code added is small: the largest dkmon is about 17 KLoC, the ring-0 hvdkpal is about 6,000 LOC of C, and dkpal calls only 15 host syscalls. The host OS kernel remains trusted, so whole-system TCB is still large.
- Cost is low relative to a VM: about 16 MB working set plus 64 MB disk per app (versus 512 MB RAM and 4.2 to 4.8 GB disk for a Hyper-V Win7 guest), typically under 1% runtime overhead, sub-second to a few seconds app start (versus about 10 s for IIS under Hyper-V), and snapshots around 1 MB versus 150 to 190 MB for a VM. Applications run unmodified.
- Density is bound by host RAM, not per-VM OS state: one machine ran 104 Excel, 527 IE, and 287 IIS instances, versus 23, 22, and 21 respectively under Hyper-V.
- The one-time OS-refactoring cost was under 2 person-years (15,681 LOC changed, about 0.3% of 5.5 MLoC, plus roughly 36 KLoC new); adding PowerPoint support after Excel took 2 days.

## Where I need review
- Performance scales with domain count (Big O): inferred as memory O(n) in instances (about 16 MB each), based on the per-instance private library OS and the density measurements; not stated as a complexity in the paper.
- Interaction semantics (synchronous/asynchronous): inferred as both, from synchronous stream reads/writes plus the emulated NT async I/O model; not framed as a dimension in the paper.
- Privileges required: inferred as user-level for the process dkpal (uses standard cross-process APIs), with more privilege needed to install the monitor or the ring-0 hvdkpal.
- Primary usage / lineage: the influence on Bascule (2013), Haven (2014), and Graphene (2014) is an external inference that post-dates the 2011 paper, which traces only the Xax lineage.
- Domain creation cost (serialization/deserialization time): Unknown. The paper reports migration and snapshotting qualitatively and gives snapshot sizes, but no serialization or deserialization time.
- Inter-domain call latency and throughput (lat_pipe / bw_pipe): not reported. Instances do not share memory and communicate via I/O streams and RDP, which were not benchmarked as latency or bandwidth.
- Domain switch cost (lat_ctx): not reported as an isolated figure; Drawbridge threads live in the host scheduler, so context switching is the host's rather than an added layer.
- POSIX compatibility: N/A, because the personality is Win32 rather than POSIX.
- Required expertise level, Debugging support, Monitoring/orchestration, Initial configuration complexity, Ongoing maintenance burden: all N/A under the self-report policy, since an external paper-based evaluator cannot assess these operational dimensions.

License: Closed / proprietary research prototype (Microsoft Research; the paper states no productization plans and no public release). A later open-source "Drawbridge" SDK on GitHub is a separate, post-paper release and is not the paper artifact (paper §5; verify before citing the SDK as the artifact).
