# The Loam frontend: interface contract (draft)

Status: draft, 2026-10-26. The server and worker do not exist yet, so every endpoint and data shape
below is a proposal to validate in the spec. It states what the UI must be able to see and do.
Terms follow `CONTEXT.md`. Design decisions follow `design.md`.

## Surfaces
1. **Pool dashboard** (web). Served by the server on the LAN. Members see their own work and nodes;
   admins see everything and manage members.
2. **Owner panel** (web, local). Served by the worker on `127.0.0.1` of each node. It sets the node's
   owner policy and pauses it. It never goes through the server, because local policy always wins
   and must work with the server down.
3. **`loam top`** (terminal, Q32). One row per node (class, CPU, RAM, state, policy), then running
   tasks and app replicas with lease age. Built separately; the web dashboard shows the same data.

## Roles
| Role | Can |
| --- | --- |
| member | submit and cancel own tasks, deploy and stop own apps, see own tasks, apps, nodes and usage |
| admin | all of the above for everyone, plus issue join tokens, revoke members, nodes and credentials, view the whole ledger |
| owner (of a node) | on that machine only: owner panel (pause, schedule, caps, battery stop, uninstall) |

## Views and interactions
**Pool overview.** Counts of nodes (online, paused, unreachable), tasks (queued, running, failed in
the last day), apps (replicas healthy of wanted). Server state: up, or stopped with "apps still
serving" noted. Admin-only: members awaiting a join.

**Nodes.** List: name, owner, class (dedicated or opportunistic), state, free RAM and vCPU,
heartbeat age, policy state (paused, outside schedule, over cap). Detail: the same plus running
tasks and replicas, link latency to peers, recent refusals with reasons. Actions: admin revokes a
node. A member cannot change another owner's policy; the UI shows the policy read-only.

**Tasks.** List with filters (mine/all for admin, state, node). Submit form: image reference,
command, vCPUs, RAM, timeout, input file (size capped, 256 MB), network off by default with an
explicit opt-in, a placement-group field hidden until the research track needs it. Detail: state
timeline (queued, leased, refused, started, ended, result stored), attempts with node and reason,
console output, result. Actions: cancel, retry. States: queued, leased, running, succeeded,
failed, timed out, cancelled, waiting for a node.

**Apps.** List: name, hostname, replicas wanted and healthy, node of each replica, class. Deploy
form: image, command, vCPUs, RAM, hostname, port, replicas (stateless) or pinned node (stateful).
Detail: per-replica state (running, parked, waking, unhealthy), logs, route. Actions: scale, stop,
remove. Apps run only on dedicated nodes; the form must say so when none exist.

**Members (admin).** List with role and status. Invite: asks for an owner label, shows the one-time
join token and the exact command to run on the new machine, with expiry. Revoke member, revoke
node, revoke credential, each separate.

**Ledger.** Usage entries (member, node, task, CPU-seconds, GPU-seconds, time) and security events
(join, revoke, denied). Members see their own rows; admins see all. A "verify chain" action runs
the hash check and reports the first bad entry or "intact".

**Owner panel (local).** Pause toggle in the first position and always reachable. Schedule windows,
CPU cap, RAM cap (default 25%), stop on battery. Shows what is running here now and why anything
was refused. Uninstall instructions. Must work with the server stopped.

## Data shapes (draft)
```
Member   { id, name, role: "admin"|"member", status, created_at }
Node     { id, owner_member_id, name, class: "dedicated"|"opportunistic",
           state: "online"|"paused"|"unreachable"|"revoked", cpu_free, ram_free_mib,
           heartbeat_age_s, policy: { paused, in_window, cpu_cap, ram_cap_mib, battery_stop } }
Task     { id, member_id, spec: { image, command[], vcpus, ram_mib, timeout_s, network: bool,
           input_bytes }, state, attempts: [ { node_id, started_at, ended_at, reason } ],
           result?: { bytes, exit_marker, console_tail } }
App      { id, member_id, hostname, port, replicas_wanted, pinned_node_id?,
           replicas: [ { node_id, state } ] }
Usage    { seq, member_id, node_id, task_id, cpu_s, gpu_s, at, prev_hash, hash, sig }
JoinToken{ token, owner_label, expires_at, single_use: true }
```

## API sketch (REST + server-sent events)
```
GET  /v1/overview
GET  /v1/nodes            GET /v1/nodes/{id}        POST /v1/nodes/{id}/revoke      (admin)
GET  /v1/tasks            POST /v1/tasks            GET /v1/tasks/{id}   POST /v1/tasks/{id}/cancel|retry
GET  /v1/apps             POST /v1/apps             GET /v1/apps/{id}    POST /v1/apps/{id}/scale|stop   DELETE
GET  /v1/members (admin)  POST /v1/join-tokens (admin)   POST /v1/members/{id}/revoke
GET  /v1/ledger?since=    GET /v1/ledger/verify
GET  /v1/events           (SSE: node, task, app, member changes)
Worker local, 127.0.0.1:  GET/PUT /policy    POST /pause    POST /resume    GET /running
```
Errors carry `{ error, code }`. A refusal by an owner is data (`reason: "paused"|"outside_window"|
"over_ram_cap"|...`), not an error.

## Live updates and failure states
- The dashboard subscribes to `/v1/events` and falls back to polling every few seconds.
- Show heartbeat age on nodes and mark a node stale before it is declared unreachable.
- **Server stopped:** the dashboard cannot load; show a clear "server unreachable" screen that says
  deployed apps are still served by the ingress. The owner panel keeps working.
- **Node unreachable, lease expired, owner refusal, retry with backoff:** all appear in the task's
  attempt list with the reason, never as a silent re-run.
- Large results are not shown inline; show size and console tail. Results over the console stopgap
  limit say so until the copy-out feature exists.

## Constraints
- Closed group, one LAN, Linux donors only. No passwords or web accounts.
- Never show invented numbers. Empty states say what will appear and how to add the first item.
- Follow the `brand` skill (Outfit and Fragment Mono, glass over sky, DeepSeek blue). Build both
  themes, keyboard operable, usable at phone width.
- The owner panel's pause must be one click and never behind a menu.

## Open questions for the frontend work
1. **Browser authentication.** Members hold a client credential, which browsers handle poorly.
   Proposal: `loam login` mints a short-lived one-time URL that sets a session. Needs deciding in
   the spec.
2. **Who serves the dashboard**: the server (simplest) or a separate static host.
3. **Result viewing** while only console text exists.
4. **GPU fields** in node and task shapes: leave out until GPU batch work starts, or reserve now.
