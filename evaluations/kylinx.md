# Compartmentalization Evaluation Matrix — KylinX

**Version:** v0.1
**System:** KylinX (dynamic library OS providing the pVM / process-like VM abstraction)
**Paper:** Y. Zhang, J. Crowcroft, D. Li, C. Zhang, H. Li, Y. Wang, K. Yu, Y. Xiong, G. Chen. KylinX: A Dynamic Library Operating System for Simplified and Efficient Cloud Virtualization. USENIX ATC 2018.

> Complete this matrix for a single implementation or system.
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other | **Untrusted component isolation** — strong isolation of mutually-untrusting tenant appliances in a multi-tenant public cloud. |
| Subject selection | Code-centric / Data-centric / Hybrid | **Code-centric** — the isolated subject is an appliance (application + its libraries) running as a pVM. |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | **VM** — the unit is a **pVM** (a lightweight, process-like paravirtualized Xen domain). |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | **N/A** — isolation is pVM/VM-granular. |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK/PKU / MTE / CHERI / TEEs / Other / None | **None directly** — isolation is provided by the **Xen type-1 hypervisor** (which itself leverages CPU privilege rings + MMU); KylinX adds no MPK/MTE/CHERI/SGX. SGX-based shielding from dom0/hypervisor is named as **future work**. |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other / None | **Other** — a **unikernel / library OS** (MiniOS-based) on a type-1 hypervisor; isolation boundary = the hardware-virtualized VM, plus dynamic-library-mapping restrictions (signature/version/loader). |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other | **VM** — the pVM (process-like VM), a paravirtualized Xen `domU`. |
| Requires runtime software support | ✅ / ❌ (describe) | **✅** — Xen hypervisor, a **modified dom0 toolstack** (dynamic page/library mapping in `libxc`/`xl`/`libchaos`), MiniOS-based LibOS, RedHat Newlib libc, lwIP TCP/IP, and LightVM's `noxs` for scaling. |
| **Properties Enforced** | | |
| Security properties provided | [Describe] | Strong **VM-level memory isolation** between pVMs via Xen (claimed equivalent to traditional VMs); inter-pVM communication (IpC) restricted to a **family of mutually-trusted pVMs** forked from the same root, rejecting communication between untrusted pVMs (all-or-nothing model, inherited from Graphene); dynamic library loading guarded by **RSA/SHA-1 signature, version, and loader restrictions**. Authors claim no new threats compared to the statically-sealed Unikernel. |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | **✅** — pVMs are isolated Xen domains, so a fault in one pVM is contained at the VM boundary; with loader restriction (i), a malicious library in one compromised application would not affect others. The paper does not frame this as a crash-recovery experiment. |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | **Partial** — provides **library-loading verification** (signature/version/loader restrictions) and **family-based trust checks** for IpC; but no general per-call data marshalling/validation across the isolation boundary. |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other (list) | **Xen hypervisor + control domain (dom0)** — KylinX treats both the hypervisor (with its toolstacks) and the control domain (dom0) as part of the TCB — plus **CPU/hardware** (assumed uncompromised). Other tenants' pVMs are untrusted. |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC] | **Per-appliance LibOS: Small** (MiniOS unikernel following the minimalism philosophy, links only requisite libraries). **System TCB: Large** — includes the full Xen hypervisor + Linux dom0. No exact LOC reported. |
| Side-channel resistance | Cache / Timing / Speculative / Other / None (mitigations) | **None** — not addressed. Hardware threats assumed out of scope; SGX-based protection from a malicious hypervisor/dom0 is future work. |
| **Validation** | | |
| Formal validation available | Yes (specify) / No | **No** — none presented. |
| Experimental validation available | Yes (specify) / No | **Yes** — prototype on Ubuntu 16.04 + Xen: standard boot, fork & recycling, memory footprint vs MiniOS/Docker, IpC latency, runtime library update, Redis & web-server apps. Testbed: two machines, each with one Intel 6-core Xeon E5-2640, 128 GB RAM, and one 1 GbE NIC. |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | **No SPEC.** App-level: Redis throughput higher than MiniOS, slightly below native Ubuntu (limited by `select`); web server higher than MiniOS, below Ubuntu. |
| Domain switch cost | lmbench lat_ctx (platform/native) | **Unknown** — no lat_ctx-style pVM context-switch figure reported (pVM scheduling is handled by Xen). |
| Domain creation cost | lmbench lat_proc exec (platform/native) | **pVM fork ≈ 1.3 ms** (vs Linux process fork ≈ 1 ms); full standard boot ≈ **124 ms** for one pVM (vs MiniOS 133 ms, Docker 210 ms); **recycling** reuses an in-memory domain to bypass domain creation. |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | IpC latency (µs): established (lineal) pVMs — pipe 55, msg_queue 43, kill 41, exit/wait 43, shared-mem 39 — **comparable to native Ubuntu IPC** (54/97/68/95/53). First-time (non-lineal) pVMs higher (232–256 µs) due to channel setup. |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | **Unknown** — no bw_pipe-style IpC throughput measured. |
| Memory overhead per domain | MB/domain | ≈ **6–7 MB per pVM** total footprint for the reduced-Redis workload at 1,000 pVMs — **lower** than statically-sealed MiniOS and Docker (shared libraries loaded once). (This is total footprint, not isolated overhead.) |
| Domain count bounded by | Limiting factor; approx. domain count | **Limiting factor: memory footprint** (cited as the single biggest factor limiting scalability/performance) plus the Xen control plane (XenStore), mitigated by LightVM `noxs`. **Tested up to 1,000 pVMs/machine**. |
| Performance scales with domain count | Big O | Boot time **superlinear** with stock XenStore but **linear** using LightVM `noxs`; memory linear / O(n) in pVM count. |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized | **Commodity** x86 capable of running Xen (Intel Xeon, 1 GbE in testbed). |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | **Custom** — requires the **Xen type-1 hypervisor** + a **modified dom0 toolstack** (dynamic-mapping extensions in `libxc`/`xl`/`libchaos`); the guest is a MiniOS unikernel, not a stock OS. |
| What privileges does it need? | User / Root / Kernel access | **Root / hypervisor (dom0) access** to deploy and manage pVMs. |
| Other deployment requirements | [e.g. custom libc, modified toolchain] | Xen, modified toolstack, RedHat Newlib libc, lwIP stack, and LightVM `noxs` (for scalable boot). |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | **Refactoring / source changes** — apps must be ported to MiniOS + Newlib and built as appliance images; documented adaptations: use `select` (no `epoll`), IPC limited to the Table 2 API, and (for the fork test) removing Redis's serialization code. |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | **Source changes needed** — recompiled against MiniOS/Newlib as a unikernel appliance; not unmodified Linux binaries. |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other | **C (primarily)** — source code (mainly C) compatibility; MiniOS is a C-style LibOS. MUSL libc support floated as future work. |
| Other compatibility notes | [Describe] | `epoll` unsupported (use `select`); IPC limited to pipe / signal / message-queue / shared-memory (Table 2); MiniOS is single-address-space with no kernel/user separation. |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] | **Unknown** — claimed minimum effort, but no person-days/LOC figure given; concrete adaptations (select, IPC, serialization) are required. |
| Required expertise level | Low / Medium / High / Expert | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Build effort | [build changes / time / deps] | App compiled as a shared library / appliance image against MiniOS + Newlib; appliances run on the modified Xen toolstack. |
| **Developer Experience** | | |
| Debugging support | Standard / Custom / Limited | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Failure modes visibility | [crashes/logs/error codes/silent] | Library signature/version verification failure → the linking procedure terminates; other failure-mode handling not described. |
| **Availability** | | |
| License | Open source / Closed / Commercial / Other | **Not found** — paper states no license; no official release located (GitHub: `kylinx/kylinx-tools` is empty; the only code, `LinuxSecurityModules/XenKylinx`, is a third-party MiniOS-on-Xen reimplementation, not the authors'). |
| Primary usage | Production / Research / Internal / Experimental / Other | **Research** — prototype. |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ | **Partial** — integrates with the Xen ecosystem and adopts LightVM `noxs`; composition with non-Xen isolation frameworks not addressed. |
| Can coexist with other compartmentalization systems | ✅ / ❌ | **✅** — pVMs are Xen domains and run alongside dom0 / other Xen domains; not framed as an explicit claim. |
| Can stack effectively | ✅ / ❌ | **✅** — many pVMs coexist (tested to 1,000). Nesting KylinX-in-KylinX not discussed. |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other | **Message passing + shared memory** — UNIX-style IpC (pipe, signal via kill/exit/wait, message queue, shared memory; Table 2) implemented over Xen **event channels + grant-table shared pages**. |
| How compartments share data | Shared memory / Message passing / Serialization / Other | **Shared memory** (Xen grant tables / shared pages) + **message passing**; copy-on-write page sharing for `fork`. |
| Interaction semantics | Synchronous / Asynchronous / Both | **Both** — asynchronous event-channel notification + synchronous pipe/queue read-write. |
| Interaction security/validation | [Describe] | **Family / all-or-nothing trust** — only pVMs in the same family (forked from a common root) may communicate; communication between untrusted pVMs is rejected. |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other | **VM** (pVM). |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | **Programmer + Runtime** — `fork()` calls create new pVMs; the dom0 toolstack instantiates/manages them. |
| Boundaries flexible at runtime | ✅ / ❌ | **✅** — dynamic pVM fork, dynamic library mapping & online update, and pVM recycling all occur at runtime. |
| **System Integration** | | |
| Threading model | Standard threads / Custom threading / Other | **Custom (MiniOS)** — MiniOS's simple non-preemptive scheduler; multithreaded apps run (web server uses worker threads) but without preemption. |
| Process model | fork/exec / Custom / N/A | **fork (pVM fork)** + UNIX-style IpC emulate multi-process; `fork` is a core contribution (MiniOS itself lacks it). |
| POSIX compatibility | Full / Partial / Limited | **Limited** — C-source subset via MiniOS/Newlib; `select` only (no `epoll`), restricted IPC API, single address space. |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe] | `epoll` unsupported; IPC limited to the Table 2 API; recycling is tentative and conditional — a recycled pVM cannot use event channels/shared memory and the app must be a self-contained shared library. |
| Security caveats when layered | [Describe] | **Recycling currently has no safeguards** against security threats between the new and old pVMs (future work); IpC trust is all-or-nothing within a family; hypervisor + dom0 are in the TCB (malicious-host protection via SGX is future work). |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other | **Cloud** — multi-tenant cloud virtualization; appliances such as big-data analysis and game servers. |
| Deployment scale | Single device / Cluster / Large scale | **Large scale** — dense per-machine appliance hosting (tested to 1,000 pVMs/machine; LightVM lineage cites 8K VMs/machine). |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |
| Ongoing maintenance burden | Low / Medium / High | **N/A** — self-report/operational dimension; not assessable by an external evaluator from the paper. |

---

## Summary

> **What it is:** A dynamic library OS that gives unikernels a **process-like VM (pVM)** abstraction — extending MiniOS on Xen with dynamic page mapping (pVM `fork` + UNIX-style inter-pVM communication) and dynamic library mapping (runtime library linking, online library update, fast recycling) — to combine unikernel-grade strong (hypervisor) isolation with process-like flexibility and efficiency.
>
> **Who it's for:** Cloud providers/tenants who want single-purpose appliances with VM-strength isolation but the dynamism of processes (fork, IPC, runtime library update, fast boot/recycling).
>
> **What it protects:** Tenant pVMs from one another via Xen type-1 hypervisor isolation, plus library-loading integrity via signature/version/loader restrictions.
>
> **What it costs (effort/money/performance):** pVM fork ≈1.3 ms (≈ Linux process fork), boot ≈124 ms, IpC ≈39–55 µs (≈ Linux IPC) — but apps must be **ported** to MiniOS/Newlib (C, `select` not `epoll`, limited IPC).
>
> **What it needs (hardware/OS/expertise):** Commodity x86 with the Xen hypervisor + a modified dom0 toolstack; a C source port to the unikernel; root/hypervisor access to deploy.
>
> **Key tradeoffs:** VM-strength isolation *and* process-like dynamics at low memory footprint — but applications must be ported to a unikernel with limited POSIX, the hypervisor + dom0 sit in the TCB, IpC is all-or-nothing within trusted families, and pVM recycling currently lacks security safeguards.
>
> **Additional Notes:** An extended journal version exists (ACM TOCS 2020, *KylinX: Simplified Virtualization Architecture for Specialized Virtual Appliances with Strong Isolation*) — not evaluated here. SGX-based protection from a malicious hypervisor/dom0 is listed as future work.
