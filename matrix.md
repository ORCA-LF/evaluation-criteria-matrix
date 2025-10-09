# Compartmentalization Evaluation: [Implementation Name]

> Complete this matrix for a single implementation or system.  
> Use ✅ / ❌ for checkboxes, *Low · Medium · High* for scales, and short text or numeric values as indicated.  
> Prefer concise, practitioner-readable entries.

---

## 🛡️ Security

| **Dimension** | **Field Type** | **Value** |
|----------------|----------------|------------|
| Threat model of underlying mechanism | Short text + checkboxes |  |
| Trust model coverage (Safebox / Sandbox / Mutual Distrust) | Checkboxes |  |
| Granularity of isolation | Scale |  |
| Failure taxonomy — location | Checkboxes |  |
| Failure taxonomy — class | Checkboxes |  |
| Blast radius / containment | Scale |  |
| Degradation behavior | Short text |  |
| Vulnerabilities blocked (classes) | Checkboxes |  |

---

## ⚙️ Runtime Efficiency

| **Dimension** | **Field Type** | **Value** |
|----------------|----------------|------------|
| Launch/startup cost (ms) | Metric |  |
| Compartment switch cost (µs) | Metric |  |
| Inter-compartment latency (µs) | Metric |  |
| Inter-compartment throughput (MB/s) | Metric |  |
| Per-syscall overhead (%) | Metric |  |
| Memory footprint (MB per compartment) | Metric |  |
| Power/runtime overhead | Scale |  |
| Scalability (scale-up / per-node) | Metric |  |
| Scalability (scale-out / across nodes) | Short text |  |

---

## 🧩 Usability / Adoptability

| **Dimension** | **Field Type** | **Value** |
|----------------|----------------|------------|
| Hardware changes required | ✅ / ❌ + notes |  |
| OS / hypervisor changes | ✅ / ❌ + notes |  |
| Operational access requirements | Checkboxes |  |
| Application source changes | Scale |  |
| Toolchain changes | Scale |  |
| Binary rewriting / instrumentation | ✅ / ❌ + notes |  |

---

## 🔗 Composability

| **Dimension** | **Field Type** | **Value** |
|----------------|----------------|------------|
| Secure app interactions | Checkboxes |  |
| API completeness | Scale |  |
| Application decomposition model | Short text |  |
| Composed protections caveats | Short text |  |

---

## 👩‍💻 Adoption & Developer Effort

| **Dimension** | **Field Type** | **Value** |
|----------------|----------------|------------|
| Developer effort (porting) | Scale + notes |  |
| Build effort | Metric / scale |  |
| Failure modes visibility | Short text |  |
| Usage model / licensing | Checkboxes + notes |  |

---

## 🧠 System Designer Perspective

| **Dimension** | **Field Type** | **Value** |
|----------------|----------------|------------|
| Compatibility & extensibility | Short text + checkboxes |  |

---

## 💰 Cost & Resources

| **Dimension** | **Field Type** | **Value** |
|----------------|----------------|------------|
| Compute / resource needs | Metrics |  |
| Monetary cost to deploy | Short text + metric |  |

---

## 📋 Adopter Summary

> **What it is:**  
> **Who it’s for:**  
> **What it protects:**  
> **What it costs:**  
> **What it needs:**  
> **How it scales:**  
> **Gotchas:**
