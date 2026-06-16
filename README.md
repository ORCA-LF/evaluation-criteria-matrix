# Compartmentalization Evaluation Matrix

This repository provides a structured evaluation matrix for analyzing different **compartmentalization implementations**.  
It is designed to help researchers, practitioners, and system designers compare tradeoffs across security, performance, usability, and adoption.

- **`matrix.md`** — the blank evaluation matrix (v0.1), the canonical template.
- **`evaluations/`** — one filled matrix per system (`<system>.md`), 32 systems.
- **`evaluations/_BASELINES.md`** — side-by-side comparison of the three non-intra-process baselines: **Firecracker** (hardware VM), **gVisor** (userspace-kernel container), and **Cloudflare Workers** (language-VM / V8 isolate), spanning the isolation-vs-density spectrum.
- **`evaluations/REVIEW_NEEDED.md`** — central log of inferred/unreported cells flagged for review.


