# Compartmentalization Evaluation: [Implementation Name]

> Complete this matrix for a single implementation or system.  
> Use ✅ / ❌ for checkboxes, *Low · Medium · High* for scales, and short text or numeric values as indicated.  
> Each row includes a “Considerations / Metrics” column describing what should be measured or evaluated.

---

## Security

| **Dimension** | **Considerations / Metrics** | **Value / Notes** |
|----------------|------------------------------|-------------------|
| Threat model (trusted vs untrusted components) | Which components are trusted/untrusted; typical attacker capabilities; assumptions and exclusions. |  |
| Underlying enforcement mechanism(s) | Mechanism(s) providing isolation (hardware, SFI, microkernel, etc.); enforcement granularity. |  |
| Security guarantees | Types of properties enforced (memory safety, control-flow integrity, syscall mediation, policy enforcement). |  |
| Failure scope and resilience | Where failures occur (component, runtime, kernel, hardware) and how the system recovers or degrades. |  |
| Compromise propagation / containment | Degree to which a compromise in one domain affects others; measurable blast radius. |  |
| Trust model coverage | Which trust relations are supported: Safebox, Sandbox, or Mutual Distrust. |  |
| Granularity of isolation | Finest supported isolation level (function, thread, process, VM, etc.) and enforcement method. |  |

---

## Runtime Efficiency

| **Dimension** | **Considerations / Metrics** | **Value / Notes** |
|----------------|------------------------------|-------------------|
| General runtime overhead | Overall performance delta vs native; aggregate application throughput or latency change (%). |  |
| Compartment switch cost (µs) | Time to switch between isolated contexts (µs); may dominate total overhead in fine-grained models. |  |
| Compartment startup cost (ms) | Initialization time to create a new compartment or domain (ms). |  |
| Inter-compartment latency (µs) | Round-trip latency for message or syscall crossing between compartments. |  |
| Inter-compartment throughput (MB/s) | Sustained communication throughput between compartments. |  |
| Memory footprint (MB per compartment) | Static and dynamic memory overhead per compartment. |  |
| Scalability (scale-up) | Maximum number of compartments supported efficiently on a single node. |  |
| Scalability-performance coupling | Relationship between compartment count and performance degradation. |  |

---

## Usability / Adoptability

| **Dimension** | **Considerations / Metrics** | **Value / Notes** |
|----------------|------------------------------|-------------------|
| Hardware changes required | Specialized CPU or hardware support (MPK, MTE, CHERI, etc.) required to run the system. |  |
| OS / hypervisor changes | Kernel, microkernel, or hypervisor modifications required. |  |
| Operational access requirements | Privileges needed (root, kernel module, hypervisor control, etc.). |  |
| Application source changes | Source-level changes or code refactoring needed for compatibility. |  |
| Toolchain changes | Custom compiler, linker, or runtime dependencies. |  |
| Binary rewriting / instrumentation | Binary rewriting required (static, dynamic, or loader-based). |  |

---

## Composability

| **Dimension** | **Considerations / Metrics** | **Value / Notes** |
|----------------|------------------------------|-------------------|
| Directionality (protects others vs protects self) | Whether isolation primarily protects other components, the compartment itself, or both. |  |
| Cross-framework composability | Ability to compose with other isolation mechanisms (kernel, hypervisor, userspace frameworks). |  |
| Secure app interactions | Support for IPC, shared memory, message passing, or capability-based interfaces. |  |
| Application decomposition model | Typical compartmentalization unit (per-lib, per-service, per-thread, etc.). |  |
| Composed protections caveats | Limitations when multiple layers of compartmentalization interact. |  |

---

## Adoption & Developer Effort

| **Dimension** | **Considerations / Metrics** | **Value / Notes** |
|----------------|------------------------------|-------------------|
| Developer effort (porting) | Approximate developer time or complexity to port an existing app. |  |
| Build effort | Changes to build process, build time overhead, or dependency management. |  |
| Failure modes visibility | How failures manifest to developers (crashes, logs, error codes). |  |
| Usage model / licensing | Open-source, commercial, or restricted licensing; ease of access. |  |

---

## System Designer Perspective

| **Dimension** | **Considerations / Metrics** | **Value / Notes** |
|----------------|------------------------------|-------------------|
| Compatibility & extensibility | Integrations with orchestration tools, CI/CD, monitoring, and packaging. |  |
| Target environment | Intended use domain: embedded, IoT, cloud, edge, workstation, etc. |  |

---

## Cost & Resources

| **Dimension** | **Considerations / Metrics** | **Value / Notes** |
|----------------|------------------------------|-------------------|
| Compute / resource requirements | CPU, memory, and storage overhead compared to baseline. |  |
| Monetary cost to deploy | Hardware cost, license fees, or operational expense implications. |  |

---

## Adopter Summary

> **What it is:**  
> **Who it’s for:**  
> **What it protects:**  
> **What it costs:**  
> **What it needs:**  
> **How it scales:**  
> **Gotchas:**
