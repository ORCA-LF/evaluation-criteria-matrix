# Compartmentalization Evaluation Matrix

> Complete this matrix for a single implementation or system.  
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Approach** | | |
| Hardware primitive(s) | Page tables / MPK / MTE / CHERI / Virtualization / Other: ___ / None | |
| Software technique(s) | SFI / Type system / Language runtime / Other: ___ / None | |
| Isolation abstraction | Process / Thread / Protection domain / VM / Container / Other: ___ | |
| Mechanism requires additional software for safety | ✅ / ❌ (if yes, describe: ___) | |
| **Isolation Model** | | |
| Subject selection | Code-centric / Data-centric / Hybrid | |
| Finest isolation granularity | Function / Library / Thread / Process / VM / Other: ___ | |
| Primary use case | Untrusted library isolation / Secret/key protection / Fault isolation / Supply chain defense / Other: ___ | |
| **Properties Enforced** | | |
| Security properties provided | [Describe: e.g., "Memory integrity and confidentiality, syscall filtering"] | |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | |
| **Interface Safety** | | |
| Boundaries are checked/validated | ✅ / ❌ / Partial | |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other: ___ (list applicable) | |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC/component count if known] | |
| Side-channel resistance | None / Partial / Strong (if Partial/Strong, describe mitigations: ___) | |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | [%] or Low / Medium / High | |
| Domain switch cost | [µs] or <1µs / 1-10µs / 10-100µs / >100µs | |
| Domain creation cost | [ms] or <1ms / 1-10ms / >10ms | |
| Inter-domain call latency | [µs] or Low / Medium / High | |
| Inter-domain throughput | [MB/s] or Low / Medium / High | |
| Memory overhead per domain | [KB/MB] or <10KB / 10-100KB / >100KB | |
| Maximum domains (practical limit) | [number] or <10 / 10-100 / 100-1000 / >1000 | |
| Performance scales with domain count | Well / Moderately / Poorly | |

---

## Usability / Adoptability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Deployment Assumptions** | | |
| What hardware does it need? | Commodity / Specialized (specify: ___) | |
| What OS/kernel does it need? | Stock / Module / Modified / Custom | |
| What privileges does it need? | User / Root / Kernel access | |
| Other deployment requirements | [e.g., "Custom libc", "Modified toolchain"] | |
| **Software Compatibility** | | |
| What application changes are needed? | None / Annotations / API changes / Refactoring / Rewrite | |
| Can it run existing binaries? | Yes / Recompile only / Source changes needed | |
| What languages does it support? | C/C++ / Rust / Go / Managed / Any / Other: ___ | |
| Other compatibility notes | [Describe any other constraints] | |
| **Developer Effort** | | |
| Porting effort for typical app | [person-days] or Low / Medium / High | |
| Required expertise level | Low / Medium / High / Expert | |
| Build effort | Changes to build process, build time overhead, or dependency management | |
| **Developer Experience** | | |
| Debugging support | Standard tools / Custom tools / Limited | |
| Failure modes visibility | How failures manifest (crashes / logs / error codes / silent failures) | |
| **Availability** | | |
| Usage model / licensing | Open source / Commercial / Research / Restricted / Other: ___ | |
| Ease of access | Easy to obtain / Requires approval / Difficult / Other: ___ | |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Works with VMs/containers | ✅ / ❌ / Partial | |
| Works with other compartmentalization systems | ✅ / ❌ (if no, specify which conflict: ___) | |
| Can be layered/stacked | ✅ / ❌ | |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other: ___ | |
| How compartments share data | Shared memory / Message copying / Serialization / Other: ___ | |
| Interaction semantics | Synchronous / Asynchronous / Both | |
| Interaction security/validation | [Describe: e.g., "Capability-based interface", "Type-checked", "Manual validation", "None"] | |
| **Decomposition Model** | | |
| Typical compartment boundary | Library / Service / Thread / Process / VM / Other: ___ | |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | |
| Boundaries flexible at runtime | ✅ / ❌ | |
| **System Integration** | | |
| Threading model | Works with standard threads / Requires custom threading / Other: ___ | |
| Process model | Works with fork/exec / Custom / N/A | |
| POSIX compatibility | Full / Partial / Limited | |
| Standard tool compatibility | Debuggers/profilers work: ✅ / ❌ / Partial | |
| **Composition Limitations** | | |
| Known limitations when composed | [Describe: e.g., "Performance degrades 2x when inside VMs", "Cannot nest with other MPK users"] | |
| Security caveats when layered | [Describe: e.g., "Outer layer can bypass inner protections"] | |

---

## System Design & Operations

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Target Environment** | | |
| Primary deployment target | Embedded / IoT / Cloud / Server / Desktop / Other: ___ | |
| Deployment scale | Single device / Cluster / Large scale | |
| **Integration** | | |
| Monitoring/orchestration support | Good / Limited / None | |
| CI/CD compatibility | Good / Limited / Custom | |
| **Operations** | | |
| Configuration complexity | Low / Medium / High | |
| Maintenance burden | Low / Medium / High | |

---

## Cost & Resources

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| CPU/memory overhead | Low / Medium / High | |
| Requires specialized hardware | ✅ / ❌ (if yes: ___) | |
| License/operational costs | Free / Moderate / High / Enterprise | |

---

## Summary

> **What it is:**  
> **Who it's for:**  
> **What it protects:**  
> **What it costs (effort/money/performance):**  
> **What it needs (hardware/OS/expertise):**  
> **Key tradeoffs:**