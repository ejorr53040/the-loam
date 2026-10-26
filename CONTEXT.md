# The Loam: domain glossary

**Pool**: the whole system: one server, many nodes, one group of members.

**Member**: a person in the pool. Has a name, a role (admin or member), a status and a client
credential. One member can own several nodes. Admins issue join tokens and revoke.

**Node**: a machine enrolled in the pool. Runs one **worker**. For the cirrocumulus surface it is
also one cirrocumulus Node. Linux x86_64 with KVM only. Has an owner (a member) and a class.

**Dedicated node**: always-on node. Runs apps and tasks. Never preempted.

**Opportunistic node**: someone's daily machine. Runs tasks only, under the owner's policy.

**Server**: the single control-plane process (SQLite). Holds the queue, the node registry, the
scheduler and the ledger. If it stops, running apps keep serving; new placements wait.

**Worker**: the pool agent on a node. It talks to the server over mTLS and to the local execution
surface. It touches only VMs it created.

**Surface**: the narrow interface a worker uses to run things (start, status, logs, stop, capacity).
cirrocumulus is the preferred and, for now, only implementation. A host-container GPU surface is
expected later.

**Task**: run-to-completion work. One-shot microVM. At-least-once, so it must be idempotent.
Has a timeout, up to 3 retries with backoff, and an input/output size cap. No network unless the
spec opts in.

**App**: long-running microVM. Stateless apps run as **replicas** started cold on several nodes.
Stateful apps are pinned to one node.

**Lease**: time-limited right of a worker to run a task. Expiry returns the task to the queue.
(Not the edge's in-flight "Lease" in cirrocumulus.)

**Owner policy**: a node's own limits (pause, schedule windows, CPU cap, RAM cap, battery stop).
The node enforces it and a refusal is a normal outcome. Local policy always wins.

**Reclaim**: the owner needs the machine back. A running task is killed after a short grace period
and requeued elsewhere.

**Ingress**: thin proxy that balances an app's replicas across nodes by forwarding to each node's
cirrocumulus edge over mTLS.

**Ledger**: append-only, hash-chained, server-signed record of usage and membership/security events.
Read-only. No credits. The scheduler may read it later for quotas.

**Placement group**: reserved job-spec field meaning "N workers placed together or none". For the
training research track. Unused for now.
