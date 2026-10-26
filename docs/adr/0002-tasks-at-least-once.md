# 0002: Tasks are at-least-once and must be idempotent

Status: accepted 2026-10-25

Workers vanish (unplugged, reclaimed by their owner, crashed). Exactly-once needs fencing and
commit protocols that a small closed pool does not justify.

Decision: a lease that expires returns the task to the queue. A task may run more than once. Output
is named by task and attempt and the server accepts the first completion; later ones are ignored.

Consequences: task authors must write idempotent tasks. Reclaim is simple (kill and requeue) and
needs no checkpointing.
