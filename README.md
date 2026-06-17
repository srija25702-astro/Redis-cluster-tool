# Redis-cluster-tool
Redis Cluster Lifecycle Tool

A CLI tool that wraps Ansible to provision, operate, and perform a zero-downtime rolling upgrade of a 6-node Redis Cluster (3 masters + 3 replicas) running inside containers.

Prerequisites

The tool checks for dependencies automatically before any command runs. You need:
Podman (preferred) or Docker Engine
Ansible 2.14+


If anything is missing, the tool prints exactly what to install and exits with a non-zero code. It does not auto-install anything without your consent.


Bringing Up the Container Infrastructure

The 6-node cluster runs inside containers that each expose SSH, simulating real servers. Your host machine acts as the Ansible control node.

With Podman (preferred)

bashcd infra/
podman-compose up -d

With Docker

bashcd infra/
docker compose up -d

This starts 6 containers on a static subnet (10.10.0.0/24):

ContainerIPSSH Portredis-node-110.10.0.112211redis-node-210.10.0.122212redis-node-310.10.0.132213redis-node-410.10.0.142214redis-node-510.10.0.152215redis-node-610.10.0.162216

Each container runs an SSH server. Redis is installed by Ansible — not baked into the image.

Rootless Podman note: Podman runs rootless by default, which means containers cannot bind to privileged ports or use the container network's IPs directly from the host. To work around this, each container maps its SSH port to a unique host port (2211–2216) and the Ansible inventory connects via ansible_host=127.0.0.1 with per-node ansible_port values. Redis cluster traffic between nodes uses the internal container network IPs (10.10.0.x) which are routable container-to-container — only the Ansible control path uses host-port mapping.

To tear down and reset:

bashcd infra/
podman-compose down   # or docker compose down


Running Each Command

All commands must be run from the submission/ directory.

Phase 1 — Provision a Redis Cluster

bash./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1

Installs Redis from source on all 6 nodes, configures cluster mode, starts Redis, forms the cluster, and prints the final topology.

Phase 2 — Seed and Verify Data

bash./redis-tool data seed --keys 1000
./redis-tool data verify

seed inserts 1000 deterministic key-value pairs (key format: key:0001, value: SHA-256 of the key name) distributed across all 3 masters via hash slots.

verify reads all 1000 keys back, recomputes expected values, and prints PASS — 1000/1000 keys verified or a failure report.

Phase 3 — Cluster Status

bash./redis-tool status

Prints a full cluster summary: role, version, slot ranges, key counts, replication topology, and per-node memory usage.

Phase 4 — Rolling Upgrade

bash./redis-tool upgrade --target-version 7.2.6 --strategy rolling

Performs a zero-downtime rolling upgrade. See Rolling Upgrade Strategy below for details.

Phase 5 — Full Verification

bash./redis-tool verify --full

Runs 5 comprehensive health checks post-upgrade: data integrity, version consistency, topology health, cluster state, and replication lag.


Rolling Upgrade Strategy

Why replicas first, then masters with failover

The strategy upgrades all replicas before touching any master. This matters because:


Replicas carry no write traffic — taking a replica offline briefly has zero client impact. There are no reads to redirect and no slots to reassign.
Masters must stay alive for the cluster to accept writes — upgrading a master directly means a brief window where that master's slot range is unavailable. Failover eliminates that window entirely.
Failover to an already-upgraded replica means the new master is already running 7.2.6 — by the time a master is demoted and taken offline for upgrade, its replacement is already healthy and serving traffic.


Step-by-step

Pre-flight:


Confirms cluster state is ok
Confirms all 6 nodes are reachable
Confirms current version differs from target (idempotent — exits cleanly if already on target)
Runs a full data verify to establish a pre-upgrade integrity baseline (1000/1000 keys must pass)


Replica upgrade (nodes 4, 5, 6 — one at a time):


Stop Redis on the replica
Install Redis 7.2.6 from source
Start Redis with the same cluster configuration
Poll until the replica rejoins the cluster and master_link_status is up
Confirm cluster_state:ok before moving to the next replica


Master upgrade (nodes 1, 2, 3 — one at a time):


Issue CLUSTER FAILOVER targeted at the replica of that master — the replica (already on 7.2.6) promotes itself to master
Poll until failover completes and the old master appears as a replica in CLUSTER NODES
Stop Redis on the demoted node (now a replica)
Install Redis 7.2.6 from source
Start Redis, wait for it to rejoin the cluster as a replica
Confirm cluster_state:ok before moving to the next master


Post-upgrade:


Runs data verify — all 1000 keys must still be present and correct
Runs cluster status — all nodes must show 7.2.6
Prints UPGRADE COMPLETE


Failure handling: any_errors_fatal: true is set on every Ansible play. If any step fails on any node, the entire run stops immediately. The cluster is left in its current state; no automatic rollback is attempted.


Assumptions and Trade-offs


Fixed topology: The infra is defined as a static 6-container setup (3 masters, 3 replicas). The --masters and --replicas-per-master flags are accepted for interface compatibility but do not dynamically resize the container count. Truly dynamic topology is the scope of the scale stretch goal (S1/S2).
Build from source: Redis is compiled from source on each node during provision and upgrade. This guarantees the exact version specified and avoids package repository availability concerns, at the cost of longer provisioning time (~5–10 min per full run).
Deterministic seed values: Keys use SHA-256(key_name) as their value. This allows data verify to recompute expected values independently without storing them anywhere — no external state needed.
Ansible role handlers: The roles/redis/handlers/ directory exists per the required role skeleton. Redis restarts are implemented as explicit ordered shell tasks rather than notify-triggered handlers, because the rolling upgrade requires precise sequencing (stop → wait → install → start → wait-for-sync → verify) that doesn't fit Ansible's end-of-play handler firing model.
Single control vantage point: Status, seed, verify, and upgrade plays run queries from redis-node-1 as the Ansible target. This is not "only checking 1 node" — each query explicitly addresses individual node IPs via redis-cli -h $ip -p $port, reading per-node INFO data from all 6 nodes. redis-node-1 is the SSH entry point, not the data scope. A separate pre-flight connectivity play runs across hosts: redis_cluster (all 6) in each playbook so all nodes appear in the Ansible PLAY RECAP.
--keys flag: The data seed and data verify commands respect the --keys flag via Ansible extra-vars ({{ num_keys | default(1000) }}). Omitting the flag defaults to 1000 keys.



Known Limitations


--masters / --replicas-per-master do not resize infra. The container topology is fixed at 6 nodes. These flags are accepted without error but only the default 3+3 layout is provisioned.
No automatic rollback. If an upgrade fails mid-way, the cluster is left in a mixed-version state. A rollback would require manually running ./redis-tool provision --version 7.0.15 after a full teardown, since Redis does not support in-place downgrade.
Rootless Podman port mapping. Because Podman runs rootless, the host cannot reach container IPs directly. Ansible connects via 127.0.0.1:221X port mapping. This is documented in the inventory and compose file. Redis inter-node traffic is unaffected (uses internal container IPs).
No structured logging. Operation output goes to stdout/stderr (capturable via tee). Per-run JSON logs to a logs/ directory are not implemented (stretch goal S5).
Scale commands not implemented. ./redis-tool scale --add-nodes and ./redis-tool scale --remove-node (stretch goals S1/S2) are not supported in this version.
