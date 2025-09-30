# Evaluation Matrix for Compartmentalization Implementations

| **Category** | **Evaluation Dimensions** | **Example Questions / Considerations** |
|--------------|----------------------------|----------------------------------------|
| **Security** | Security posture | Does the design minimize attack surface? How strong are its guarantees? |
|              | TCB size / configuration | How large and complex is the trusted computing base? |
|              | Compromise / failure resilience | What happens when one compartment is compromised? Does it cascade? |
|              | Granularity of enforcement | Is isolation fine-grained (per thread, per function) or coarse-grained (per process, per VM)? |
|              | Vulnerabilities blocked | How many classes of known vulnerabilities are effectively mitigated? |
| **Runtime Efficiency** | Startup costs for new compartments | How fast can new compartments be launched? |
|              | Memory footprint | How much memory overhead per compartment or per system? |
|              | Inter-compartment communication | Latency and throughput for communication channels (pipes, domain sockets, shared memory, etc.). |
| **Usability / Adoptability** | Hardware changes | Does the system require new or specialized hardware? |
|              | OS / system changes | Are kernel patches, microkernel designs, or hypervisor modifications needed? |
|              | Application source rewriting | Must source code be modified to run? |
|              | Build toolchain changes | Are new compilers, linkers, or runtimes required? |
|              | Binary rewriting | Is lightweight binary instrumentation sufficient? |
| **Composability** | Application interactions | Can two apps interact without degrading each other’s security? |
|              | API completeness | Are APIs consistent and full-featured for real workloads? |
|              | Application decomposition | Does the system support breaking down apps into secure components? |
| **Adoption & Developer Effort** | Perspective of the adopter | Is the system intuitive and practical for real developers and operators? |
|              | Failure modes | What does failure look like—silent data corruption, crashes, security leaks? |
|              | Development effort | How long does it take to write or port applications to the model? |
|              | Build effort | How long do builds take with the required toolchain or transformations? |
| **System Designer Perspective** | Compatibility & extensibility | How easily does this approach integrate with other compartmentalization or security mechanisms? |
| **Cost & Resource Requirements** | Computational resources | Does it require heavy CPU/GPU cycles, or can it run on commodity machines? |
|              | Monetary cost | What is the cost to deploy at scale (licensing, hardware, operational)? |

