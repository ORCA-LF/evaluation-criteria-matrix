# Compartmentalization Evaluation Matrix — Drawbridge

**System:** Drawbridge (Windows 7 library OS; compartment = a **Drawbridge process** — an app + its private library OS in one isolated address space behind a narrow ABI)
**Paper:** D. E. Porter, S. Boyd-Wickizer, J. Howell, R. Olinsky, G. C. Hunt. *Rethinking the Library OS from the Top Down.* ASPLOS 2011. (Microsoft Research / Stony Brook / MIT)
**Source file:** `papers/asplos2011-drawbridge.pdf`
**Matrix version:** v0.1
**Evaluated:** 2026-06-10

> ✅ / ❌ for checkboxes; categorical scales and short text otherwise.
>
> **Sourcing legend:** **(§N)** = stated in paper · **[repo]** = from git repo/site · **_Inferred:_** = evaluator assessment · **⚠️ Unclear** = not found, needs research.
>
> _Compartment = a **Drawbridge process**: an application plus its **own private copy of the OS personality** (the Win32 API implementation, refactored into a user-space **library OS**) running in a single isolated address space. It reaches the host OS only through a deliberately **narrow ABI** (~40 downcalls) implemented by a **security monitor** (`dkmon`) that virtualizes host resources and enforces a manifest-based whitelist policy. Host OS keeps hardware + user services; the library OS holds application services (§1, §3, §4)._
>
> _A library-OS / picoprocess isolation **System** — the Windows analogue and predecessor of [[graphene]] (Linux library OS)._

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — strongly isolate each application from the host OS and from other apps, "at least as well as a VMM," at a fraction of the overhead; protect host system integrity from untrusted software (§1, §6). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is a whole application (+ its library OS) (§1, §3). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process** — a "Drawbridge process" (picoprocess); each app runs in its own address space with its own library OS (§1, §4). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is per-process; host resources are virtualized/whitelisted by URI, not data-tagged (§4). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **Other — standard hardware memory protection** (per-process address spaces); shares Xax's "hardware memory protection + a limited system-call table" approach. No specialized primitive (§7). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — library OS + narrow ABI + security monitor (reference monitor)**: the OS personality runs in-process as a library; a narrow ABI mediated by `dkmon` is the only path to host resources, gated by a manifest whitelist (§3, §4). _(The ABI count is **~40** by evaluator enumeration of the §4 call groups — ~38; the paper notes an early version had just 19 calls but states no single total.)_ |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Process (picoprocess / "extremely paravirtualized VM")** — a host OS process whose only host interface is the Drawbridge ABI; positioned between a heavyweight VM and a raw process (§4, §7). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the per-app **library OS** (refactored Win32 personality + NT-kernel emulation), the **security monitor `dkmon`**, and the **platform adaptation layer `dkpal`**; the host OS provides hardware/user services (§3–5). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **Strong host-from-app + app-from-app isolation**: a Drawbridge process gets no host handles, a private emulated NT kernel namespace, no shared kernel objects, and access only to manifest-whitelisted host resources (default: files in its own directory). Demonstrated: registry-wiping malware affected only itself; a keylogger couldn't see other apps' keystrokes; **all five Keetch IE-protected-mode escape vectors mitigated** (§4, §6). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — each app + library OS is a separate isolated address space; a child process inherits no memory/objects and a parent cannot modify a child's execution; a misbehaving/compromised app damages "at most the single application in the sandbox" (§4, §6, §8). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **✅** — the **narrow ABI is the validated boundary**: "the brevity of the Drawbridge ABI enables tractable coding-time and run-time review of its isolation boundary"; `dkmon` filters every I/O stream by URI against the manifest; no `ioctl`; apps cannot create host handles (§4). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The security monitor (`dkmon`/`dkpal`) + the host OS kernel + CPU.** The **library OS is *not* in the host's TCB** — it runs untrusted in the app's address space; the security boundary is the ABI (§4). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Small monitor + Large host OS** — the security monitor is the new trusted code: largest `dkmon` ≈ **17 KLoC**; the ring-0 `hvdkpal` ≈ **6,000 LOC C**; `dkpal` calls only **15 host syscalls**. The host OS kernel remains trusted (so whole-system TCB is large) (§4). |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — side channels are not discussed; the focus is integrity/isolation of host resources (§4, §6). |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal proofs; the argument is that a small, reviewable ABI makes the boundary *auditable* ("tractable … review"), plus empirical attack case studies (§4, §6). |
| Experimental validation available | Yes (specify) / No | **Yes** — runs unmodified Excel/PowerPoint/IE/IIS; integrity experiments (registry-wipe, keylogger); the Keetch IE-escape case study (all 5 vectors); scalability (100s of instances/host); servicing-footprint analysis; migration/snapshot experiments (§6). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Ongoing execution overhead vs native is small; "Ongoing execution overheads are only slightly higher with Hyper-V (… typically less than 1%)" — i.e. Drawbridge ≈ native, beating a VM. Threads use host NT threads directly, avoiding extra scheduling overhead (§5, §6). Host: dual 2.4 GHz Xeon E5530 quad-core, 16 GB, 64-bit Win7 (§6). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Not lat_ctx.** Drawbridge threads "reside in the host kernel's scheduling queues and avoid unnecessary scheduling overheads" — context switching is the host's, not an added layer; no isolated figure (§4). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **App start times** (Fig. 5, seconds): Drawbridge instances start in well under a second to a few seconds, far below a Hyper-V guest, which adds full-OS boot (e.g. IIS **10.0 s** under Hyper-V, §6). Snapshot **size** is also far smaller (Table 2: e.g. IIS 1.1 MB vs 193.4 MB VM; Excel <4 MB vs ~150 MB). The paper reports serialization/migration qualitatively but gives **no serialization/deserialization time** — that figure is **Unknown**. No lat_proc microbench (§6). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Not lat_pipe.** Drawbridge processes don't share memory; they communicate via **I/O streams** (`pipe:`/`tcp:`/`udp:`) and UI via **RDP**; cross-instance sharing reuses networking protocols. No latency figure (§4, §1). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | ⚠️ **No bw_pipe figure** — inter-process comms via I/O streams/RDP, not benchmarked as bandwidth (§4). |
| Memory overhead per domain | MB/domain | **≈ 16 MB working set added per app** (a private `win32k` + its font/graphics caches), + **64 MB disk** for the library OS (83 MB with DirectX 11; **< 16 MB** minimized for Reversi) — vs a Hyper-V Win7 guest's **512 MB RAM + 4.2–4.8 GB disk** (§1, §6). |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: host RAM (per-app ~16 MB), not per-VM OS state.** Measured on one machine: **104 Excel** (vs 23 Hyper-V, 142 native), **527 IE** (vs 22 Hyper-V, 138 Windows — Windows hits a per-session GDI-handle limit Drawbridge avoids), **287 IIS** (vs 21 Hyper-V) (§6). |
| Performance scales with domain count | Big O | _Inferred:_ memory **O(n)** in instances (~16 MB each, private library OS); scales far better than VMs because host OS state isn't replicated per instance (§6). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** x86-64 (runs as ordinary Windows processes, or as a ring-0 Hyper-V partition) (§6). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Stock host (no kernel changes for the default `dkpal`)** — runs on Windows 7 / Server 2008 R2 / MinWin / a pre-release Windows / ring-0 Hyper-V / (experimentally) Barrelfish; the same library OS runs unchanged across all (§4, §6). |
| What privileges does it need? | User / Root / Kernel access | _Inferred:_ **user-level** for the process `dkpal` (uses standard cross-process APIs); installing the monitor / ring-0 `hvdkpal` needs more privilege (§4). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | The Drawbridge library OS package (64 MB) + a per-app **manifest** (URI whitelist); apps are "sequenced" by capturing setup-time file-system/registry changes (App-V/ThinApp-style) (§4, §6). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **None** — "all application binaries are unmodified" (only Excel/PowerPoint licensing code disabled). 14,000+ Win32 functions supported (§6). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Yes** — unmodified Windows binaries (Excel, PowerPoint, IE, IIS, DirectX demos, CLR apps) (§6). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **Any Windows application** — native C/C++ plus the **.NET CLR** and **DirectX**; the personality is Win32, not language-specific (§1, §6). |
| Other compatibility notes | [Describe] | Holes: printing (drivers load in-process), multi-process apps sharing state through `win32k`, admin/debugger tools needing unfiltered host access, and rarely-used NT APIs that return failure — most apps tolerate these (§5 Limitations). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Zero for the application** (unmodified binaries). The **one-time OS-refactoring** effort: 15,681 LOC changed (0.3% of 5.5 MLoC) + ~36 KLoC new, **< 2 person-years** total; adding PowerPoint after Excel took **2 days** (§6). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Build effort | [build changes / time / deps] | No app rebuild; "sequence" the app (capture FS/registry deltas) into a Drawbridge package; ship the library OS image (§6). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(Paper notes serialization enables time-travel debugging / cloning as a future opportunity — §8.)_ |
| Failure modes visibility | [crashes/logs/error codes/silent] | Unimplemented NT APIs return failure (e.g. `STATUS_NOT_IMPLEMENTED`); most apps respond gracefully or don't use them; isolation violations are blocked at the ABI/manifest (§3, §4). |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Closed / proprietary (research prototype)** — Microsoft Research; "Microsoft has no plans to productize any of the concepts." No public release in the paper. _(The later open-source Drawbridge SDK exists separately; not the paper artifact.)_ ⚠️ verify (§5). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** prototype (ASPLOS 2011). _Inferred (external, post-dates this 2011 paper):_ influential lineage → Bascule (2013), Haven (2014), and [[graphene]] (2014). (The paper itself only traces the **Xax** lineage, §9.) |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **✅** — runs *on top of* a VMM (ring-0 Hyper-V partition) and on minimal/experimental kernels (MinWin, Barrelfish); the ABI cleanly layers the library OS over diverse hosts (§4, §6). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — Drawbridge processes run as ordinary host processes alongside native apps and VMs; 100s coexist per host (§6). |
| Can stack effectively | ✅ / ❌ | **✅** — every application can run in its own instance; the design is many-independent-instances (§6). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Message passing / networking** — I/O streams (`pipe:`/`tcp:`/`udp:`) for data, **RDP** for shared UI (screen/keyboard/clipboard); parent↔child via streams provided at creation (§1, §4). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Message passing / networking** — no shared memory between instances; resource sharing (clipboard, desktop) is via "the pragmatic reuse of networking protocols" (RDP), each gated by policy (§1, §4). |
| Interaction semantics | Synchronous / Asynchronous / Both | _Inferred:_ **Both** — synchronous stream reads/writes + asynchronous I/O (the NT async I/O model is emulated); not framed as a dimension (§5). |
| Interaction security/validation | [Describe] | Every host-resource access mediated by `dkmon` against the manifest whitelist; clipboard/network-server abilities must be explicitly whitelisted (e.g. IE on Drawbridge is denied web-server capability, blocking loopback spoofing) (§4, §6). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Process (picoprocess)** — one app + library OS per instance (§1, §4). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Runtime/packager + Policy author** — the OS refactoring defines the library/host split; per-app manifests define resource access; the developer writes no boundary code (§3, §4). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — instances created/destroyed dynamically; **process serialization/migration** lets a running app move across machines/host-OS releases and survive reboots (§4, §6). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Standard (host-backed)** — Drawbridge threads map to host NT threads/sync objects and live in the host scheduler; the ring-0 `hvdkpal` implements its own threading/futexes (§4). |
| Process model | fork/exec / Custom / N/A | **Custom — no `fork`** — `DkProcessCreate` spawns a child that inherits no memory/objects (the Windows API has no `fork`, which simplified the design) (§4, §5). |
| POSIX compatibility | Full / Partial / Limited | **N/A (Windows personality)** — the personality is Win32, not POSIX; the technique generalizes but the prototype is Windows (§1, §3). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Multi-process apps that share state through `win32k` aren't supported (would need shared address space or a shared `win32k` server); printing (in-process drivers); admin/debugger tools needing host access; rarely-used NT APIs return failure (§5). |
| Security caveats when layered | [Describe] | Host OS kernel stays trusted; side channels not addressed; correct isolation depends on the manifest whitelist being right; network sockets terminate on migration (apps must reconnect) (§4, §6). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Desktop + Server + Cloud** — rich desktop apps (Excel/PowerPoint/IE), server apps (IIS), and proposed cloud "virtual desktop" / per-app sandboxing at lower cost than VMs (§1, §6, §8). |
| Deployment scale | Single device / Cluster / Large scale | **Single device → Cloud** — 100s of instances per host; proposed for cloud hosting to lower per-sandbox cost vs VMMs (§6, §8). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator (policy 2026-06-09). _(Paper does report a servicing benefit: only 36% of security patches touch the library OS — §6.)_ |

---

## Summary

> **What it is:** The first refactoring of a full commercial OS (Windows 7) into a **library OS**. Each application runs in its own isolated address space with a *private* copy of the Win32 personality (API implementation + NT-kernel emulation) and reaches the host only through a deliberately tiny ABI (~40 calls) mediated by a security monitor that whitelists host resources. It delivers VM-grade isolation at a fraction of the cost.
>
> **Who it's for:** Anyone needing strong per-application isolation, mobility, or OS-version independence for unmodified Windows apps — desktop sandboxing, app compatibility across OS releases, live migration, and cheap cloud per-app sandboxes.
>
> **What it protects:** The host OS (and other apps) from untrusted application code — a Drawbridge process gets no host handles, a private kernel namespace, and only manifest-whitelisted resources (demonstrated to contain registry-wiping malware, a keylogger, and all five IE-protected-mode escape vectors).
>
> **What it costs (effort/money/performance):** ~16 MB working set + 64 MB disk per app (vs 512 MB / 4.8 GB for a Win7 VM), <1% runtime overhead, sub-second-to-few-second app start (vs ~10 s for a Hyper-V guest) and tiny snapshots (~1 MB vs ~150–190 MB VM) — and zero app changes (unmodified binaries). The one-time OS-refactoring cost was <2 person-years.
>
> **What it needs (hardware/OS/expertise):** Commodity x86-64, a stock host OS (no kernel changes for the default monitor), the 64 MB library OS image, and a per-app manifest.
>
> **Key tradeoffs:** VMM-level isolation + application mobility + independent OS evolution (one library OS ran across 4 host OSes) at order-of-magnitude lower overhead and a small, auditable trusted ABI — but the host kernel stays trusted, side channels are unaddressed, multi-process/shared-`win32k` apps and printing aren't supported, and it's a closed research prototype.
>
> **Additional Notes:** A library-OS / **picoprocess** isolation **System** — the Windows analogue and direct ancestor of [[graphene]] (Linux library OS), and a sibling of the VM baselines ([[firecracker]]) and userspace-kernel sandbox ([[gvisor]]), but lighter than a VM and heavier/coarser than the intra-process schemes ([[erim]]/[[smv]]/[[arbiter]]). Its enduring contribution is the **narrow, host-independent ABI** as a small, reviewable isolation boundary, plus the demonstration that a *commercial* OS can be refactored this way. Shares Xax's "HW memory protection + limited syscall table" lineage.

---

## Sources

- **Primary:** `papers/asplos2011-drawbridge.pdf` — Porter, Boyd-Wickizer, Howell, Olinsky, Hunt, *Rethinking the Library OS from the Top Down*, ASPLOS 2011, DOI 10.1145/1950365.1950399. All **(§N)** / Fig. / Table citations reference this paper.
- **Availability:** Microsoft Research prototype; paper states no productization plans and no public release — **treated as closed/proprietary** for the paper artifact (a later "Drawbridge" SDK on GitHub is a separate, post-paper release; verify before citing as the artifact).

### Cells flagged for further research
- **Availability › License** — ⚠️ paper artifact closed/proprietary; a later GitHub SDK exists separately (verify).
- **Efficiency › Domain switch (lat_ctx), inter-domain call/throughput** — ⚠️ not reported (host-backed threads; comms via streams/RDP). MB/domain (~16 MB) and start/migration times *are* given.
- **Composability › POSIX compatibility** — N/A (Win32 personality, not POSIX).
- **Usability › Required expertise, Debugging** + **Sys Design › Monitoring, Config complexity, Maintenance burden** — N/A (self-report policy 2026-06-09).
- Inferences marked `_Inferred:_` inline: privileges, scaling Big-O, interaction semantics.
