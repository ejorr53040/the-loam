# The Loam

A volunteer compute pool for a small, closed group. Members lend idle CPU and GPU time on their
own machines; the pool runs apps (as microVMs) and batch tasks on them. Distributed ML training is a
research track, not built yet.

The Loam is a **separate system from cirrocumulus**. It drives cirrocumulus as its preferred
execution surface and never lives inside it. cirrocumulus stays a single-machine system.

Status: design grilled 2026-10-25, no code yet.

- `CONTEXT.md`: the glossary. Use these words.
- `docs/design.md`: every decision made so far, the build order, the open items.
- `docs/frontend-interface.md`: what the UI must show and do (draft contract).
- `docs/architecture.html`: the interactive architecture diagram.
- `docs/adr/`: decisions that are hard to reverse.
