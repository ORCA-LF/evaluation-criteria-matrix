# Compartmentalization Evaluation Matrix — Asbestos

**Version:** v0.1
**System:** Asbestos — a DIFC operating system with labels + a lightweight "event process" abstraction for per-user data isolation; compartment = an event process (a labeled per-session context) / a labeled process
**Paper:** S. VanDeBogart, P. Efstathopoulos, E. Kohler, M. Krohn, C. Frey, D. Ziegler, F. Kaashoek, R. Morris, D. Mazières. Labels and Event Processes in the Asbestos Operating System. ACM TOCS 25(4), Article 11, Dec 2007 (journal version of the SOSP 2005 paper).

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation + Secret protection** — application-level **data isolation**: prevent exploited server code from accessing other users' secret data, via decentralized information flow control (§1, §3). Motivating target: a secure dynamic-content Web server (§1, §8). |
| Subject selection | Code-centric / Data-centric / Hybrid | **Hybrid** — labels apply to subjects (processes, event processes) and to messages/ports carrying data (§4, §5). |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **Process + event process** — the **event process** is a lightweight, isolated context (typically one per user session) within a base server process (§7, §8.3). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **Message / Object** — labels attach to inter-process messages and communication ports; discretionary labels apply to single messages (§2, §4, §5). |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None** — commodity x86 PC; standard virtual memory (event processes store per-session page-table differences) (§7, §10). |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other — kernel-enforced DIFC labels in a new OS + a message-passing IPC model + the event-process abstraction.** The kernel labels every message and updates the receiver's labels on delivery; event processes give cheap per-session isolation inside one server process (§4, §5, §7). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **Other — the event process** (a labeled per-session context within a base process), alongside ordinary labeled processes (§7, §8.3). |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — the Asbestos kernel (labels, tags, ports, event-process syscalls: allocate/remap/free memory, `ep_clean`, etc.) plus user-level services (a user-level network stack, the Asbestos Web server) (§4, §7, §8). |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | **DIFC** via per-process **tracking** and **clearance** labels; **decentralized** tags — "allocating a tag confers privilege for that tag on the allocating process," so any process may allocate arbitrarily many tags (§5). Prevents an exploited application from leaking one user's data to others (§1). Adds **discretionary labels** on single messages (§2). |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — isolation ensures "an exploit exposes only one user's data"; each user session is confined to its own event process (§1, §3, §8.3). |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — the kernel checks labels on every message send/receive and updates the receiver's tracking label (Eq 4), enforcing clearance bounds; but **label updates on receive are *implicit***, which is itself an information-flow concern the design only *partially* avoids (§2, §5). |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **The Asbestos kernel + CPU/hardware.** In the Web-server application, some components are **trusted** and some **semi-trusted** (hold privilege for specific tags) — per-session workers are isolated and largely untrusted (Fig 1, §8). |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Not stated in LOC in this paper.** _Inferred (external):_ the HiStar paper reports its own 15,200-line-C kernel as "roughly 45% fewer lines of C code than the Asbestos kernel," implying the Asbestos kernel is **≈27,000 lines of C** — an external inference, not an Asbestos claim; flagged. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **Partial** — the current design "avoids *some* implicit information flows," and could avoid all label-check implicit flows by not updating labels implicitly on receive (§2). The event-process implementation uses **a single thread per base process**, which "violates the isolation of individual event processes" (a blocking `recv` stalls its siblings — an acknowledged channel) (§7). Timing channels are a general DIFC limitation. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — no formal verification is claimed. |
| Experimental validation available | Yes (specify) / No | **Yes** — the Asbestos Web server compared against Apache and Mod-Apache (throughput, latency) and a memory-scaling study to >100,000 users (§10). Host: **2.8 GHz Pentium 4, 1 GB RAM**, gigabit-Ethernet LAN with a Linux HTTP client (§10). |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** Information-flow/isolation features add "moderate overhead"; the Asbestos Web server's throughput and latency are **competitive with Apache** on Linux, landing **between Apache-with-CGI-workers and Apache-with-a-linked-in module** (§10). With sixteen users it has a **smaller median and 90th-percentile latency than Apache** (§10, Fig 14). |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Not reported as lat_ctx** — inter-context communication is message passing over ports (§5). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **Not reported as lat_proc.** Event processes are deliberately cheap to create — the optimization is storing only **page-table differences** per event process (§7). |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | **Not reported** — communication is asynchronous labeled messages between ports, not a pipe RTT (§5). |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Not reported** as pipe bandwidth (Web-server request throughput is the reported metric) (§10). |
| Memory overhead per domain | MB/domain | **≈1.4 memory pages per user (event process)** for the test application — the headline result: comprehensive user isolation at ~1.4 pages/user for **more than 100,000 users** (Abstract, §7, §10.1). |
| Domain count bounded by | Limiting factor; approx. domain count | **Demonstrated to >100,000 concurrent users** (event processes); tags come from a large opaque space (61-bit IDs from an encrypted counter) so any process may allocate arbitrarily many (§5, §10). The practical scaling limit is **label size**, which grows with the number of sessions (§10). |
| Performance scales with domain count | Big O | Ideally throughput/latency are independent of session count, but "the size of various labels in the system will increase with" the number of sessions (§10); an optimized label representation keeps even **300,000-tag labels** fast (§ label impl). |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** — an x86 PC (§10). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Custom** — Asbestos is a new operating system (§4). |
| What privileges does it need? | User / Root / Kernel access | **It is the OS** — DIFC labels/tags are decentralized to applications; no central security-officer allocation required (§5). |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | Applications are written to the Asbestos label/port/event-process API; the Web server runs on a user-level network stack (§7, §8). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Rewrite / API changes** — applications are built to the Asbestos model; the evaluated Web server is a re-architected version of **OKWS** using labels + event processes (§8.3). |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Source changes needed** — a custom OS with a message-passing/label API, not binary-compatible with Unix (§4, §8). |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C** — kernel and services in C (§4, §8). |
| Other compatibility notes | [Describe] | Not a Unix; interaction is asynchronous labeled messages. The single-thread-per-base-process implementation limits event-process concurrency (§7). |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | No person-day figure; the Web server was rebuilt from OKWS to use labels + event processes (§8.3). |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension not assessable from the paper. |
| Build effort | [build changes / time / deps] | Applications built against the Asbestos API and services (§8). |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension not assessable from the paper. |
| Failure modes visibility | [crashes/logs/error codes/silent] | Messages that violate flow rules are governed by label checks; DIFC systems generally fail flows silently to avoid leaks (§5). Not detailed as an operational feature. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Open source** — the Asbestos source is available (historically via CVS); a **mixture of BSD/MIT-style licenses**, with at least one GPL component (an Ethernet driver). Not stated in the paper; verified via project/web sources (flagged for a precise per-file check). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — a prototype OS and Web server (§1, §8). |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | _Inferred:_ **❌** — Asbestos is a whole new OS, not designed to compose with external isolation frameworks (§4). |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **N/A** — it is the operating system, not a component run beside others. |
| Can stack effectively | ✅ / ❌ | **✅** — event processes compose within a base process (one per session, scaling to >100k), and labels/tags compose so multiple principals' policies coexist (§7, §8, §10). |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Message passing** — labeled messages over communication ports (§2, §5). |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Message passing** between processes; **event processes share the base process's address space** but are isolated by labels + per-session page-table differences (§5, §7). |
| Interaction semantics | Synchronous / Asynchronous / Both | **Asynchronous** — messages are unreliable in the model, but structured applications (e.g., the Web server) "can make delivery reliable in practice" (§4). |
| Interaction security/validation | [Describe] | The kernel enforces label checks on each message and updates the receiver's tracking label per the label-comparison rules (Eq 4), bounded by clearance (§5). |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **Event process (per session) / process** (§7, §8.3). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer** — unprivileged applications define their own isolation policies and allocate their own tags (§3, §5). |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — tags/labels and event processes are created and adjusted at runtime; however label changes on message receipt are **implicit** (§5, §7). |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom** — the current implementation uses **a single thread of execution per base process**, which limits event-process concurrency (§7). |
| Process model | fork/exec / Custom / N/A | **Custom** — base processes host many event processes; `ep_clean` releases an event process's memory except session data (a "cached session") (§7, §10.1). |
| POSIX compatibility | Full / Partial / Limited | **Limited** — a new message-passing/label OS, not a Unix (§4, §8). |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | Single-thread-per-base-process limits concurrency and leaks a channel (a blocking `recv` stalls sibling event processes); label size grows with session count; implicit label updates on receive (§7, §10). |
| Security caveats when layered | [Describe] | Implicit label changes on message receipt are a known information-flow weakness (later fixed by requiring explicit changes in HiStar/Flume); timing channels unaddressed (§2, §5). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Server** — a high-performance, data-isolating dynamic Web server is the driving application (§1, §8). |
| Deployment scale | Single device / Cluster / Large scale | **Single device, many users** — one machine supporting >100,000 isolated user sessions (§10). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension not assessable from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension not assessable from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension not assessable from the paper. |

---

## Summary

> **What it is:** A new **DIFC operating system** whose two contributions are (1) decentralized **labels** — tracking + clearance labels with self-contained tags any process can allocate — that control system-wide information flow, and (2) the **event process**, a lightweight isolated context (typically one per user session) that lets a single server process handle many mutually-isolated users cheaply (§5, §7).
>
> **Who it's for:** Builders of servers that must isolate per-user data even when the application code is exploited — the flagship is the **Asbestos Web server** (a re-architected OKWS) that keeps one user's data out of reach of code serving another (§1, §8).
>
> **What it protects:** Application-defined data-isolation policies (per-user secrecy) enforced by the kernel on every message, so "an exploit exposes only one user's data" (§1, §3).
>
> **What it costs (effort/money/performance):** Moderate overhead — throughput and latency competitive with Apache (between Apache-CGI and Apache-module), and only **≈1.4 memory pages per user** to isolate more than 100,000 users (§10).
>
> **What it needs (hardware/OS/expertise):** A commodity x86 PC; it is a *custom OS*, so applications are written to its label/port/event-process API (§4, §8, §10).
>
> **Key tradeoffs:** Cheap, fine-grained per-session isolation at Web scale with application-defined policy — but a custom OS (no Unix compatibility), **implicit label updates on message receipt** (a covert-channel weakness later fixed by HiStar/Flume), a single-thread-per-base-process implementation that limits event-process concurrency, and labels that grow with session count (§2, §7, §10).
>
> **Additional Notes:** Asbestos is the **root of the DIFC-label lineage** in this matrix — **HiStar** adopts its labels but adds persistence, explicit resource allocation, and explicit (non-implicit) label changes to close storage channels; **Flume** brings the same model to stock Unix at process granularity. For the ORCA matrix the "compartment" is an **event process / labeled process**, an OS-level MAC/DIFC point rather than a memory-isolation sandbox. TCB size is not given in LOC (HiStar implies ≈27K C lines — external inference, flagged); license is a BSD/MIT mixture (+GPL driver), flagged for a precise check.
