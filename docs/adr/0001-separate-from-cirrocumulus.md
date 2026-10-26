# 0001: The Loam is a separate system that drives cirrocumulus

Status: accepted 2026-10-25

cirrocumulus is a single-machine system: one agent per host, local Unix socket, no node identity,
no scheduler. Its own roadmap (M8) planned a hub-and-spoke server inside it. A pool of several
people's machines is a different product, and a personal cluster does not make sense yet.

Decision: The Loam is its own project and repo. It uses cirrocumulus through the public socket API
as its preferred execution surface. cirrocumulus is never exposed to the network; only the worker
is. Anything The Loam needs from cirrocumulus must stand on its own as a single-machine feature
(for example "run to completion with an exit code", "copy a file out of an ended VM"). Nothing
cluster-shaped goes into cirrocumulus.

Consequences:
- Cluster CA, join, scheduling, ledger and ingress are The Loam's.
- The Loam owns its own client types and a contract test against the real agent socket.
- The worker is in the `cirro` group, which is full control of that host's VMs, so it only touches
  VMs it created (reserved `loam-` name prefix) and enforces its own budget.
- cirrocumulus M8's server/join/cluster stubs should be marked not planned. Not yet done: wait for
  the shared understanding to be confirmed.
