An evaluation matrix is a structured rubric we fill in for each compartmentalization system, scoring it across a fixed set of dimensions (security, performance, usability, composability, and so on) so the systems can be compared side by side on the same terms. Each cell is either a cited fact from the paper, a value from the system's repo, an inferred judgment, or marked N/A or Unknown.

Review is welcome from anyone, and especially from the paper's authors. If you know the system well, please check the cells I inferred or characterized qualitatively and let me know where I got something wrong. The "Where I need review" section below points at the specific cells I am least sure about.

[View rendered matrix](../evaluations/asbestos.md)

## What it is
Asbestos is a decentralized information-flow-control (DIFC) operating system from VanDeBogart, Efstathopoulos, Kohler, Krohn, Frey, Ziegler, Kaashoek, Morris, and Mazières, "Labels and Event Processes in the Asbestos Operating System," ACM TOCS 25(4), 2007 (the journal version of the SOSP 2005 paper). Its two contributions are decentralized labels — tracking and clearance labels with self-contained tags any process can allocate — and the event process, a lightweight isolated context (typically one per user session) that lets a single server process handle many mutually-isolated users cheaply. Asbestos is the root of the DIFC-label lineage in this matrix: HiStar adopts its labels, and Flume brings the same model to stock Unix.

- Compartment is an event process (a labeled per-session context) or a labeled process. This is an OS-level MAC/DIFC point, not an in-process memory-isolation sandbox.
- Mechanism: a new OS kernel labels every inter-process message and updates the receiver's label on delivery; event processes provide cheap per-session isolation within one server process by storing only page-table differences. No special hardware (commodity x86).
- Trust model: the kernel plus CPU; in the Web-server application some components are trusted and some semi-trusted, while per-session workers are isolated and largely untrusted.

## Key takeaways
- Comprehensive user isolation at about 1.4 memory pages per user, demonstrated for more than 100,000 users, with throughput and latency competitive with a well-tuned Apache (between Apache-with-CGI and Apache-with-a-linked-module).
- The event process is the headline abstraction: many isolated per-session contexts inside a single base process, far cheaper than fork-accept designs.
- Known weakness, later fixed by HiStar and Flume: label changes happen implicitly on message receipt, which is an information-flow concern; the design only partially avoids it.
- The current implementation uses a single thread of execution per base process, which limits event-process concurrency and leaks a channel (a blocking recv stalls sibling event processes).

## Where I need review
- Security, TCB size: the paper gives no kernel LOC. I inferred approximately 27,000 lines of C from HiStar's statement that its own 15,200-line-C kernel has "roughly 45% fewer lines of C than the Asbestos kernel." This is an external inference, not an Asbestos claim, and should be confirmed.
- Security, License: open source, historically via CVS, described elsewhere as a mixture of BSD/MIT-style licenses with at least one GPL component (an Ethernet driver). Not stated in the paper; a precise per-file confirmation would help.
- Composability, Integrates with other mechanisms: Inferred No (Asbestos is a whole new OS).
- Isolation abstraction is recorded as "Other — the event process," which is a judgment about how the per-session context maps onto the matrix's fixed options.
- Self-report dimensions (debugging, required expertise, config complexity, maintenance, monitoring) are N/A per the matrix's external-evaluator policy.

License: open source, a mixture of BSD/MIT-style licenses with at least one GPL component (Ethernet driver); not stated in the paper and flagged for a precise per-file check.
