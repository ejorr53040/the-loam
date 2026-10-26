# The Loam: design so far

Grilled 2026-10-25. Everything below is agreed unless listed under "Open".

## Purpose
Several people lend idle CPU and GPU to one pool over a LAN. The pool runs (1) apps in microVMs
sharded across devices and (2) batch tasks. (3) Distributed ML training is research only: kept in
mind, not implemented.

## Scope and trust
- Closed group, invite only. 3 to 10 machines.
- Donors: Linux x86_64 with KVM only (cirrocumulus constraint).
- Success test: a 200-task batch finishes faster than on the best single machine, with one machine
  unplugged mid-run.
- Same LAN first. mTLS between nodes and server. An overlay (WireGuard or an existing NAT-traversal
  tool) is a later, separate decision when a second home network joins.

## Contributor promises
Never slower while they use it, one-click pause and uninstall, jobs cannot reach their files or LAN.
- Tasks run at the lowest cgroup CPU weight; RAM capped (default 25% of host; refuse placements
  over it).
- Owner policy knobs: pause, schedule windows, CPU cap, RAM cap, stop on battery. GPU-busy and
  thermal checks arrive with GPU batch. Real input-idle detection is later, if needed.
- Local policy always wins. A refused lease is normal and the server re-places the work.

## Node classes
Dedicated (always-on; apps and tasks; never preempted) and opportunistic (daily machines; tasks
only, under owner policy). Chosen by the owner at enrollment. Machines that cannot run microVMs
(the Pi, no-KVM laptops) are not pool compute; the Pi hosts the server and may be a cirrocumulus
Machine.

## Control plane
- One `loam` server, SQLite, backups, on the always-on machine (the Pi). Replication is later.
- If the server stops, running apps keep serving; no new placements.
- Scheduler is a pure scoring function over free RAM, free vCPU, hard filters and spread.

## Apps
Stateless apps run as replicas started cold (parked snapshots only wake on identical hardware, so
they do not move between different donors). Stateful apps are pinned to one node. The ingress
balances across replicas by forwarding to each node's cirrocumulus edge over mTLS; edges unchanged.

## Tasks
One-shot microVM per task. CPU first. At-least-once and idempotent (ADR 0002). Timeout, up to 3
retries with backoff so a busy node is not retried at once. Inputs and outputs go through the
server, capped (256 MB per task) for now; a chunked store is the later upgrade. No network unless
the spec opts in. On reclaim, kill after a grace period and requeue.
GPU batch runs on the host in a container, outside microVMs, only when the owner is not using the
card. GPU inside a microVM is research.

## Training research track
Reserve a placement-group field in the job spec and record link latency and bandwidth between
peers. Nothing else until research says what is feasible over home links.

## Identity, members, ledger
- Admin issues one-time join tokens that carry an owner label. No web accounts.
- Records: Member (id, name, role, status, credential), Node (id, owner, class, certificate
  fingerprint, status), Usage entry (member, node, task, resources used, time).
- Members authenticate the CLI with a client credential that is separate from node certificates.
  Revoke member, node and credential independently. Only admins issue tokens or revoke. A member
  sees their own usage; the admin sees all.
- Ledger: append-only, hash-chained, server-signed. Holds node-signed usage reports and membership
  and security events (join, revoke, denied requests). Read-only. No credits. Per-owner concurrency
  cap. The scheduler may read it for quotas later.

## Boundary with cirrocumulus (ADR 0001)
- cirrocumulus stays single-node and Unix-socket only. Only the worker faces the network.
- Surface interface: start, status, logs, stop, advertise capacity. One implementation now.
- Worker touches only `loam-` VMs and enforces its own budget.
- Gaps in cirrocumulus today: a finished VM has an end reason (exited, crashed, stopped, parked)
  and a readable console log, but no numeric exit code and no output extraction. Stopgap: infer
  success from the end reason and console output. Ask cirrocumulus for "run to completion with an
  exit code" and "copy a file out" only as single-machine features judged on their own merit.
- The Loam has its own small client types for the HTTP+JSON socket API and a contract test against
  the real agent. It does not depend on cirrocumulus crates.

## Build order
0. Surface client and contract test: start/stop/logs/list through the real agent socket; `loam-`
   prefix and budget.
1. Identity: pool CA, `loam server init`, `worker join`, member and node records, mTLS.
2. Heartbeats and capacity: node registry, a multi-node `loam top`.
3. Placement and ingress: the scoring function, apps on two nodes, route cache so traffic keeps
   flowing with the server stopped.
4. Tasks: leases, retries, results (stopgap result path).
5. Owner policy and reclaim.
6. Ledger.
7. Demo: the 200-task test above.
GPU, training and NAT traversal stay off the list until slice 7 is done.

## Open (not grilled yet)
- Shared understanding has not been confirmed by Ethan; no code should start before it is.
- (Resolved) `cirro run` sends the command as argv to guest-init over vsock, and the guest powers
  off when it exits, so the host only sees an end reason and the console log. Result stopgap: the
  task wrapper prints an exit marker and a small result on the console and the worker parses it.
  The 256 MB cap needs the file copy-out feature in cirrocumulus; the console path is for small
  results only.
- What the multi-node `loam top` shows, and the Pi's exact role as server host.
- The ADR in cirrocumulus marking M8's cluster plans as not planned (waiting on confirmation).
- (Resolved) Diagram redrawn: `docs/architecture.html` (open in a browser). Published copy:
  https://claude.ai/artifact/PseQPvYnDXW9BXLZd6ohzb
