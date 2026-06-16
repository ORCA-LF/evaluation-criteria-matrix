# Compartmentalization Evaluation Matrix

**Version:** v0.1

> Complete this matrix for a single implementation or system.  
> Use ✅ / ❌ for checkboxes, categorical scales, and short text values as indicated.

---

## Security

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Isolation Model** | | |
| Primary use case | Untrusted component isolation / Secret protection / Fault isolation / Other: ___ | |
| Subject selection | Code-centric / Data-centric / Hybrid | |
| Code-centric granularity | Function / Library / Thread / Process / VM / N/A | |
| Data-centric granularity | Object / Region / Page / Allocation / File / N/A | |
| **Isolation Approach** | | |
| Hardware primitive(s) | MPK or PKU / MTE / CHERI / TEEs / Other: ___ / None | |
| Software technique(s) | SFI / Boundary wrappers/marshalling / Memory-safe language / Language runtime / Other: ___ / None | |
| Isolation abstraction | Process / Thread / Intra-Address Space Domain / VM / Container / Other: ___ | |
| Requires runtime software support | ✅ / ❌ (if yes, describe: ___) | |
| **Properties Enforced** | | |
| Security properties provided | [Describe: e.g., "Memory integrity and confidentiality, syscall filtering"] | |
| **Resilience** | | |
| Compartment crashes isolated | ✅ / ❌ | |
| **Interface Safety** | | |
| Provides means to facilitate boundary checking/validation | ✅ / ❌ / Partial | |
| **Trust & Threats** | | |
| TCB includes | Compiler / OS / Firmware / CPU / Other: ___ (list applicable) | |
| TCB approximate size | Small / Medium / Large / Very Large or [LOC/component count if known] | |
| Side-channel resistance | Cache-based / Timing / Speculative-execution / Other: ___ / None (describe mitigations: ___) | |
| **Validation** | | |
| Formal validation available | Yes (specify: ___) / No | |
| Experimental validation available | Yes (specify: ___) / No | |

---

## Runtime Efficiency

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| General runtime overhead | SPEC 2017 (platform/native) | |
| Domain switch cost | lmbench lat_ctx (platform/native) | |
| Domain creation cost | lmbench lat_proc exec (platform/native) | |
| Inter-domain call latency | lmbench lat_pipe (platform/native) | |
| Inter-domain throughput | lmbench bw_pipe (platform/native) | |
| Memory overhead per domain | MB/domain | |
| Domain count bounded by | Limiting factor: ___; Approximate domain count: ___ | |
| Performance scales with domain count | Big O | |

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
| Porting effort for typical app | [person-days] | |
| Required expertise level | Low / Medium / High / Expert | |
| Build effort | Changes to build process, build time overhead, or dependency management | |
| **Developer Experience** | | |
| Debugging support | Standard tools / Custom tools / Limited | |
| Failure modes visibility | How failures manifest (crashes / logs / error codes / silent failures) | |
| **Availability** | | |
| License | Open source / Closed source / Commercial / Other: ___ | |
| Primary usage | Production / Research / Internal tooling / Experimental / Other: ___ | |

---

## Composability

| **Dimension** | **Metric** | **Value / Notes** |
|----------------|------------|-------------------|
| **Cross-Framework Composability** | | |
| Integrates with other compartmentalization mechanisms | ✅ / ❌ (can be combined with other isolation mechanisms) | |
| Can coexist with other compartmentalization systems | ✅ / ❌ (side-by-side without interference) | |
| Can stack effectively | ✅ / ❌ (multiple instances of the same mechanism compose correctly) | |
| **Inter-Compartment Interactions** | | |
| How compartments invoke each other | Function calls / Message passing / RPC / Syscalls / Other: ___ | |
| How compartments share data | Shared memory / Message passing / Serialization / Other: ___ | |
| Interaction semantics | Synchronous / Asynchronous / Both | |
| Interaction security/validation | [Describe: e.g., "Capability-based interface", "Type-checked", "Manual validation", "None"] | |
| **Decomposition Model** | | |
| Decomposition boundary | Library / Service / Thread / Process / VM / Other: ___ | |
| Who defines boundaries | Programmer / Compiler / Runtime / Kernel / Hardware | |
| Boundaries flexible at runtime | ✅ / ❌ | |
| **System Integration** | | |
| Threading model | Works with standard threads / Requires custom threading / Other: ___ | |
| Process model | Works with fork/exec / Custom / N/A | |
| POSIX compatibility | Full / Partial / Limited | |
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
| **Operations** | | |
| Initial configuration complexity | Low / Medium / High | |
| Ongoing maintenance burden | Low / Medium / High | |

---

## Summary

> **What it is:**  
> **Who it's for:**  
> **What it protects:**  
> **What it costs (effort/money/performance):**  
> **What it needs (hardware/OS/expertise):**  
> **Key tradeoffs:**  
> **Additional Notes:**  
