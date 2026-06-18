An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/breakapp.md)

## What it is
BreakApp (Vasilakis, Karel, Roessler, Dautenhahn, DeHon, and Smith, NDSS 2018) is a drop-in replacement for a language runtime's module system that turns module boundaries into isolation boundaries automatically. The problem it addresses is supply-chain and dependency risk: in ecosystems like Node.js/npm, third-party code can make up roughly 99.9% of shipped code, and a single malicious or buggy package runs with full application privileges. At each import/require, BreakApp can spawn the imported module into its own compartment, rewrite the returned value so member and function accesses become RPC proxies, and preserve single-runtime semantics so the rest of the app is unchanged.

- Compartment is a module run in its own configurable "box," chosen per import: an in-runtime software sandbox (SBX, e.g. a fresh V8 context), an OS process (PROC), or a container (LXC).
- Mechanism: no hardware primitive; automated source/AST transformation of import into a compartment spawn plus generated RPC proxies and interposition, layered on a memory-safe managed runtime, with OS process and container isolation backing the heavier box types.
- Trust model: the TCB includes the language runtime (V8/JIT), the built-in libraries (fs, net, ...), BreakApp itself, the OS (for PROC and LXC), and the CPU. Third-party modules are untrusted. Package-manager install-script attacks and most side channels are out of scope.

## Key takeaways
- Applications need no changes to adopt it: no annotations, no tracing or inference pre-runs, and no manual rewriting. Optional one-line per-import policies (BOX, IPC, CONTEXT, TRUST, DOUBT, REPLICAS, ON_FAIL, TIMER) tune the security versus performance trade-off, and a vanilla require ignores them.
- Isolation strength and cost are coupled to the box type. Per-additional-compartment startup is 1.5 ms for SBX (5x the 0.3 ms baseline), 72 ms for PROC (240x), and 666 ms for LXC (2220x). Cross-boundary call latency ranges from 6.5 ns for an in-runtime function call up to tens of milliseconds for TCP, including serialization.
- In a realistic HTTP application the compartmentalization overhead is only 2 to 15% of overall latency even for heavier boxes, because I/O and HTTP handling dominate; the proxy and interposition cost is described as barely visible.
- BreakApp's own code is small (about 2K LOC of JavaScript, plus about 2K for the NaCL crypto library via Emscripten), but the effective TCB is large because the V8/Node runtime and built-ins are trusted. It is a research prototype, evaluated against 6 real CVE-class vulnerabilities plus 6 hypothetical ones and an applicability study over 15 real apps.

## Where I need review
- Runtime Efficiency > Memory overhead per domain: Unknown. No MB-per-compartment figure is reported; only startup, throughput, and latency are quantified (Tables VII and VIII).
- Runtime Efficiency > Performance scales with domain count: Inferred. I read startup as O(n) (fixed per-box cost times count) with cross-boundary throughput degrading as the compartment count rises.
- Usability > What privileges does it need: Inferred. I read it as user-level for SBX and PROC, with container privileges for LXC (Docker), and no kernel modification.
- Composability > POSIX compatibility: Inferred as N/A / language-level, since BreakApp operates at the language module layer rather than the POSIX API.
- System Design > Deployment scale: Inferred as single device (a per-application process tree); the paper does not frame scale as a dimension.
- Usability > Required expertise level: N/A (self-report/operational dimension, not assessable by an external evaluator).
- Usability > Debugging support: N/A (self-report/operational dimension). BreakApp does provide monitoring, ON_FAIL actions, and violation reporting.
- System Design > Initial configuration complexity: N/A (self-report/operational dimension).
- System Design > Ongoing maintenance burden: N/A (self-report/operational dimension).

License: GPL-2.0 (the npm package breakapp declares GPL-2.0, verified 2026-06-11; the wrapped core @andromeda/breakapp and the nvasilakis/breakapp wrapper repo carry no separate license field).
