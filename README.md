# Simulated Distributed System

A hands-on learning lab for **distributed systems and cloud infrastructure**, built entirely with local Docker containers — no cloud bill required. One laptop impersonates a production cluster: load balancers, stateless API replicas, databases with replication and failover, caches, chaos experiments, and eventually local Kubernetes.

## Why this exists

Reading about distributed systems builds vocabulary; breaking them builds intuition. This project simulates multi-node environments locally so failure, latency, replication lag, and recovery can be experienced directly — killed containers, partitioned networks, promoted replicas, point-in-time recoveries. The goal is the engineering intuition expected of backend/infrastructure engineers at large-scale tech companies.

Everything here follows one methodology: **rebuild the machinery yourself, then compare notes with the real thing.** Containers are rebuilt from raw Linux primitives before using Docker in anger; RDS is reimplemented as containers + scripts before touching AWS.

## Repository structure

```
simulated_distributed_system/
├── CLAUDE.md            # Instructions for AI-assisted authoring (conventions, folder rules)
├── persona.md           # Who I am, my background, goals, and learning style
├── README.md
└── Documentation/
    ├── Docker_Practice/ # Step-by-step lectures: first principles, real commands, under-the-hood mechanics
    └── Plans/           # Brainstorms: project ideas, creative directions, study campaigns
```

### Document conventions

Files in `Documentation/` are named `NNN-Title.md` (three-digit sequence per folder) and begin with a creation timestamp line `YYYY_MM_DD_HH_MM-Title`, so documents read in chronological order.

## Current contents

### Docker_Practice — lectures

| # | Lecture | Covers |
|---|---------|--------|
| 001 | Docker From the Kernel Up | Namespaces, cgroups, OverlayFS, the engine chain (CLI → dockerd → containerd → runc), networking (veth/bridge/DNAT/DNS), volumes, lifecycle, a full Compose lab with failure experiments |
| 002 | Build a Container by Hand | Recreating `docker run` with zero Docker: `unshare`, `pivot_root`, hand-wired veth pairs, raw cgroupfs writes, manual OverlayFS mounts |
| 003 | Build Your Own RDS | Managed Postgres from first principles: parameter groups, streaming read replicas, sync vs async replication, failover + endpoint flips, PITR with WAL archiving, PgBouncer as RDS Proxy |

### Plans — ideas and campaigns

| # | Document | Theme |
|---|----------|-------|
| 001 | Distributed Systems Learning Ideas | The master roadmap: 9 phases from a single Compose stack to chaos engineering, observability, local Kubernetes, and consensus |
| 002 | The Chaos Arcade & Other Wild Ideas | Gamified failure drills, outage reenactments, Raspberry Pi edge nodes, an AI SRE agent, build-your-own-X ladder |
| 003 | RDS Study Campaign Ideas | How to learn a managed service you can't SSH into: build the control plane, race failovers, run disaster drills |

## Roadmap (abridged)

Foundations (Compose, LB, statelessness) → caching → async messaging → replication & consistency → chaos/failure injection → observability → local Kubernetes (kind/k3d) → cloud emulation (LocalStack) → consensus and build-your-own primitives. Full detail in `Documentation/Plans/001`.

## Requirements

- Docker Engine + Docker Compose (Linux, or Docker Desktop)
- Root access on a Linux box (or Pi) for the by-hand labs in Lecture 002
- Curiosity about what's under the hood
