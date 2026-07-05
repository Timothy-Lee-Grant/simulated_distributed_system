2026_07_05_16_48-RDS_Study_Campaign_Ideas

# The RDS Study Campaign — Ideas for Learning a Managed Service You Can't SSH Into

*Brainstorm document. The question: how do you build deep understanding of AWS RDS — a service whose whole pitch is hiding the machinery — using the local-Docker methodology? Answer: rebuild the machinery, then compare notes with AWS. Companion lecture: Docker_Practice/003.*

---

## The core insight for studying ANY managed service

A managed service = **engine + control plane + SLA**. The engine (Postgres) is open source — you can hold it. The control plane (provisioning, failover, backups) is automation — you can rebuild a toy version. The SLA is the business wrapper. So the methodology for RDS generalizes to ElastiCache (Redis + control plane), MSK (Kafka + control plane), OpenSearch... Learn it once here, reuse the recipe for the whole AWS catalog. That's the real prize.

## Idea 1 — "You Are the Control Plane" 🎛️

Don't just run the Lecture-003 commands — **wrap them in your own mini-RDS API.** A ~300-line FastAPI service with endpoints mirroring the real AWS API: `CreateDBInstance` (launches a tuned Postgres container + volume + alias), `CreateDBSnapshot`, `RestoreDBInstanceToPointInTime`, `FailoverDBCluster`, `DescribeDBInstances` (status: creating → available → failing-over, driven by health checks). Under the hood it shells out to the Docker API. This is the project: you stop being an RDS *user* and become an RDS *implementer*. Bonus rung: make its request/response shapes match real AWS closely enough that `boto3` pointed at your endpoint half-works. ("I wrote a boto3-compatible RDS control plane over Docker" — an absurdly good line.)

## Idea 2 — The Failover Championship 🏁

Failover has a *time budget*: detection + promotion + redirection + client reconnect. Instrument each phase and race yourself. Scoreboard per run: seconds of write unavailability, transactions lost (should be 0 with sync standby), stale reads served. Iterate: cron health-checker → tighter probe intervals → PgBouncer in front (smooths reconnects) → automated promote script. Real RDS Multi-AZ does it in 60–120s, Aurora in <30s. Can your laptop beat AWS? (Yes — and understanding *why that's cheating* — no fencing, one machine, trivial detection — teaches you what the hard 90% actually is.)

## Idea 3 — The 14:07 Disaster Drills 🔥

A recurring PITR game (Lecture 003 Part 5) with escalating scenarios: (a) dropped table — restore to one minute prior; (b) bad deploy wrote garbage for 20 minutes — restore to before, then *replay the good writes* that came after (much harder — now you need logical extraction); (c) the "oops, wrong environment" classic — someone truncated prod thinking it was staging. Write a one-page runbook after each drill. Interviewers love recovery stories, and DBAs are made in exactly these fires — yours are controlled burns.

## Idea 4 — Replica Lag Observatory 🔭

Make lag *visible*: Grafana panel of `pg_stat_replication` LSN deltas while you run write bursts (`pgbench` is perfect). Then hunt the classic anomaly: **read-your-own-writes violation** — write a row via the writer endpoint, immediately read via the reader endpoint, catch the miss. Then fix it three different ways (session pinning, read-after-write goes to writer, bounded-staleness check) and compare. This single anomaly is the gateway drug to consistency models — and it's a top-5 system design interview topic ("user updates profile, refreshes, sees old data — why?").

## Idea 5 — Parameter Group Wind Tunnel 🌬️

RDS gives you ~400 knobs and most engineers touch none. Pick the ten that matter (`shared_buffers`, `work_mem`, `max_connections`, `effective_cache_size`, `random_page_cost`, `wal_compression`, ...), and for each: run `pgbench` before/after a deliberately *bad* value and a tuned value, with the cgroup memory limits from Lecture 001 making the container behave like a small instance class. You're literally simulating "what does upgrading from db.t3.micro to db.r6g.large buy me?" — capacity planning intuition, earned empirically on a laptop.

## Idea 6 — The Two-Pizza Comparison Study 🍕

After each local build, read how AWS actually did it and write a short "my version vs theirs" note: my endpoint flip = Route53 CNAME update (what does DNS TTL do to my failover time?); my `archive_command` = S3 WAL archiving (what's the durability difference?); my health-check cron = their distributed failure detector (what happens when the *detector* is partitioned, not the database?). Each gap you find is a distributed-systems lesson AND ready-made "questions to ask" material for interviews. Aurora deserves its own note: replication moved *below* the engine into a shared storage layer — the single most interesting database architecture decision of the last decade.

## Idea 7 — The AI DBA 🤖 (Chaos Arcade crossover)

Extend Plans/002's AI SRE with database tools: `get_replication_status`, `get_slow_queries` (from `pg_stat_statements`), `promote_replica`, `restore_pitr(time)`. Then run RDS-flavored chaos: kill the primary, inject replica lag, exhaust connections. Grade the agent: does it check `pg_stat_replication` before promoting? Does it refuse to promote when it can't confirm the primary is truly dead (split-brain caution)? Teaching an agent to be *appropriately scared* of promotion is a beautiful exercise in encoding operational judgment — and it merges your AI-engineering track with your infrastructure track in one artifact.

## Idea 8 — Pi-RDS: Multi-AZ on Actual Hardware 🍓

The embedded-bridge idea, RDS edition: primary on one Raspberry Pi, sync standby on another, your laptop as a client. Now "AZ failure" = pulling a power cable, and network partition = yanking Ethernet mid-transaction. Watch the sync-commit hang from Lecture 003 Part 3 happen over a *real* network with real packet loss. Nobody's resume says "I ran Multi-AZ Postgres on Raspberry Pis and physically induced partitions." Yours could.

---

## Suggested campaign order

1. **Lecture 003 end-to-end** (the foundation — one focused weekend).
2. **Idea 3** drills + **Idea 4** observatory (operational feel, ~an evening each).
3. **Idea 1** control plane (the centerpiece project, 2–3 weekends — this is the portfolio piece).
4. **Idea 6** notes throughout, **Idea 7/8** when the AI-agent and Pi tracks mature.

Then, and only then, spin up a real RDS free-tier instance for a week and audit it with expert eyes: find the `pg_stat_replication` view they expose, notice which parameters are locked, time an actual failover with `dig` watching the CNAME. You'll be *interrogating* the product instead of touring it.
