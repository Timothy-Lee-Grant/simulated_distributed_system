2026_07_05_16_48-Build_Your_Own_RDS_Simulating_Managed_Postgres

# Lecture 003 — Build Your Own RDS: Simulating Managed Postgres in Docker

*AWS RDS is not a database. It's a **control plane** wrapped around an ordinary database engine. That means every feature on the RDS marketing page is something you can rebuild locally with Postgres containers — and once you've built it, RDS stops being magic and becomes a checklist of automation you happen not to have to write yourself.*

**Prerequisites:** Lectures 001–002, Docker Compose basics, and a healthy suspicion of the word "managed."

---

## Part 0 — What RDS Actually Is (Demystification First)

When you click "Create database" in the AWS console, roughly this happens: AWS provisions a VM (you never get SSH), attaches network block storage (EBS), installs a stock Postgres/MySQL, applies your **parameter group** as the engine config, wires a **DNS endpoint** (a CNAME — never a raw IP; remember Lecture 001's "nothing should ever hardcode an IP"), schedules **automated backups** (storage snapshots + WAL archiving), and optionally keeps a **standby replica** in another datacenter with synchronous replication (**Multi-AZ**), promoting it automatically if the primary dies and flipping the CNAME to point at it.

Every bolded term above is one Part of this lecture. The translation table we'll build:

| RDS feature | What it really is | Our local simulation |
|---|---|---|
| DB instance | VM + block storage + stock engine | a Postgres container + named volume |
| Parameter group | managed `postgresql.conf` | `-c` flags / mounted config file |
| Endpoint | a CNAME AWS controls | Docker network **alias** we can re-point |
| Read replica | async streaming replication | second container + `pg_basebackup` |
| Multi-AZ standby | **sync** replication + auto-failover | `synchronous_standby_names` + our own promote script |
| Automated backup + PITR | base snapshot + WAL archive | `pg_basebackup` + `archive_command` + `recovery_target_time` |
| RDS Proxy | managed connection pooler | a PgBouncer container |
| Performance Insights | wait-event sampling dashboard | `pg_stat_activity` + postgres_exporter/Grafana |

---

## Part 1 — The "DB Instance" and Parameter Groups

```bash
docker network create rdsnet

docker run -d --name pg-primary --network rdsnet \
  --network-alias db.mylab.internal \
  -e POSTGRES_PASSWORD=pw \
  -v pgprimary:/var/lib/postgresql/data \
  postgres:16-alpine \
  -c max_connections=200 \
  -c shared_buffers=256MB \
  -c wal_level=replica \
  -c log_min_duration_statement=250
```

**What's going on.** The trailing `-c key=value` args are passed to the `postgres` binary and override `postgresql.conf` — this *is* a parameter group: a set of engine parameters applied at the control-plane level rather than by editing files on the box. RDS's distinction between *dynamic* parameters (apply immediately) and *static* ones (require reboot) exists here too: `ALTER SYSTEM SET work_mem='8MB'; SELECT pg_reload_conf();` applies live, but changing `shared_buffers` needs a container restart — same rule, same reason (it's allocated at startup).

The `--network-alias db.mylab.internal` is the star of this lecture: an extra DNS name for this container on `rdsnet`. Your apps will connect to `db.mylab.internal:5432` and **never** to `pg-primary`. That indirection is exactly what an RDS endpoint is, and it's what makes failover invisible to clients (Part 4).

Practice the "no SSH" discipline too: resist `docker exec` for admin tasks; do everything through `psql` over the network, like a real RDS customer:

```bash
docker run -it --rm --network rdsnet postgres:16-alpine \
  psql -h db.mylab.internal -U postgres -c "SHOW max_connections;"
```

---

## Part 2 — The Read Replica (Streaming Replication From First Principles)

**The theory.** Postgres durability rests on the **WAL** (write-ahead log): every change is appended to a sequential log *before* the data pages are touched — the same journaling idea as flash filesystems in your embedded world. Replication is the observation that this log is also a perfect change feed: ship the WAL stream to a second server that continuously replays it, and you get a byte-identical, read-only copy. Async by default: the primary does not wait for the replica (fast, but a crash can lose the last few transactions on the replica side — that's why RDS read replicas can serve *stale* reads).

```bash
# 1) Create a replication role on the primary:
docker exec pg-primary psql -U postgres -c \
  "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replpw';"

# 2) Take a base backup INTO a fresh volume, with replication pre-configured:
docker run --rm --network rdsnet -v pgreplica:/var/lib/postgresql/data \
  -e PGPASSWORD=replpw postgres:16-alpine \
  pg_basebackup -h db.mylab.internal -U replicator \
    -D /var/lib/postgresql/data -X stream -R

# 3) Start the replica on that volume:
docker run -d --name pg-replica --network rdsnet \
  --network-alias db-ro.mylab.internal \
  -v pgreplica:/var/lib/postgresql/data postgres:16-alpine
```

**What `-X stream -R` did:** copied the primary's entire data directory, then wrote two things into it — a `standby.signal` file ("boot as a replica, keep replaying WAL forever instead of opening for writes") and a `primary_conninfo` line ("here's who to stream WAL from"). That's the entire secret of replica creation; RDS's "Create read replica" button does this snapshot-then-stream dance on EBS.

**Watch it work:**

```bash
docker exec pg-primary psql -U postgres -c \
  "CREATE TABLE readings(id serial, v int); INSERT INTO readings(v) VALUES (7);"
docker exec pg-replica psql -U postgres -c "SELECT * FROM readings;"   # it's there
docker exec pg-replica psql -U postgres -c "INSERT INTO readings(v) VALUES (8);"
# ERROR: cannot execute INSERT in a read-only transaction   ← replicas are read-only

# The primary's view of its followers (state, replayed LSN, lag):
docker exec pg-primary psql -U postgres -x -c "SELECT * FROM pg_stat_replication;"
# The replica's lag, self-reported:
docker exec pg-replica psql -U postgres -c \
  "SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;"
```

Note the second alias: `db-ro.mylab.internal` — you've just built RDS's **reader endpoint**. Point your app's read-heavy queries there and you've implemented read scaling.

---

## Part 3 — Multi-AZ Semantics: Sync vs Async (The Distinction That Matters)

RDS read replicas and Multi-AZ standbys use the same mechanism (WAL shipping) with one flipped switch — and that switch is a profound distributed-systems tradeoff:

```bash
docker exec pg-primary psql -U postgres -c \
  "ALTER SYSTEM SET synchronous_standby_names = '*'; SELECT pg_reload_conf();"
```

Now every `COMMIT` on the primary **blocks until the standby confirms it has durably received the WAL**. Zero data loss on failover (this is why Multi-AZ exists) — but you've bought it with latency on every write, and with a scarier failure mode. Feel it:

```bash
docker stop pg-replica
docker exec pg-primary psql -U postgres -c "INSERT INTO readings(v) VALUES (9);"
# ...HANGS. The primary would rather stall than acknowledge a write it can't guarantee.
```

(Ctrl-C, `docker start pg-replica`, and it heals.) You have just experienced CAP-theorem consistency-over-availability as a *feeling* rather than a diagram. RDS Multi-AZ handles this by failing over instead of hanging forever; standalone Postgres just waits. This experiment — sync standby dies, writes freeze — is the single best intuition-builder for "why is synchronous replication expensive?"

---

## Part 4 — Failover: Promotion and the Endpoint Flip

The two-step dance RDS performs during Multi-AZ failover (typically 60–120s):

```bash
# Disaster strikes:
docker rm -f pg-primary

# Step 1 — PROMOTE: tell the replica to stop replaying and open for writes
docker exec pg-replica psql -U postgres -c "SELECT pg_promote();"
docker exec pg-replica psql -U postgres -c "SELECT pg_is_in_recovery();"   # f = now a primary

# Step 2 — FLIP THE ENDPOINT: move the writer alias to the survivor
docker network disconnect rdsnet pg-replica
docker network connect --alias db.mylab.internal --alias db-ro.mylab.internal rdsnet pg-replica

# Clients reconnect to db.mylab.internal and land on the new primary. No config changed anywhere.
docker run -it --rm --network rdsnet postgres:16-alpine \
  psql -h db.mylab.internal -U postgres -c "INSERT INTO readings(v) VALUES (10); SELECT * FROM readings;"
```

**What you're practicing:** the anatomy of every managed-database failover — *detect, promote, redirect*. Detection is the part we hand-waved (we knew the primary died because we killed it); doing detection *correctly* is the hard problem, because a slow primary and a dead primary look identical from outside (Lecture 001's health checks, and the split-brain warning from the Mental Sandbox — if the old primary comes back and still thinks it's the boss while clients write to the new one, you get divergent histories; this is why real systems use fencing or consensus, and why Raft is on your roadmap).

Also now obvious: why RDS failover still drops connections (TCP sessions die with the old primary; clients must reconnect and DNS must re-resolve — hence "keep DNS TTLs low" in every RDS runbook) and why applications need retry logic *no matter how managed the database is*.

---

## Part 5 — Automated Backups and Point-in-Time Recovery

**The theory.** Backup = **base snapshot + continuous WAL archive**. Restore-to-time = restore snapshot, replay archived WAL, stop replaying at the target timestamp. RDS's "restore to any second in the last 7 days" is exactly this.

```bash
# Archive WAL segments to a second volume as they're completed:
docker volume create walarchive
# (recreate pg-primary with: -v walarchive:/archive \
#   -c archive_mode=on -c "archive_command=cp %p /archive/%f")

# Base backup = the snapshot:
docker exec pg-primary pg_basebackup -U postgres -D /tmp/base -Ft -z
```

Now the classic drill — **the 14:07 disaster**: insert rows, note the time, `DROP TABLE readings;` at 14:07, then restore the base backup into a fresh volume with `recovery_target_time = '14:06:50'` set and a `recovery.signal` file. Postgres replays archived WAL and **stops just before the drop**. Your table is back; the mistake never happened.

The lesson that generalizes: `pg_dump` is a *logical* copy (portable, but restore-to-a-second is impossible); snapshot+WAL is *physical* and gives you a time machine. RDS automated backups are the physical kind; that's why PITR works. (Also connects straight to Plans/002's event-sourcing "Time Machine" idea — WAL *is* event sourcing at the storage layer.)

---

## Part 6 — RDS Proxy Is Just a Pooler

Postgres forks **one OS process per connection** (~5–10MB each; you can watch them: `docker exec pg-replica ps aux | grep postgres`). 500 lambda invocations = 500 processes = a dead database. RDS Proxy exists purely to multiplex many client connections onto few server connections:

```bash
docker run -d --name pool --network rdsnet --network-alias db-pooled.mylab.internal \
  -e DB_HOST=db.mylab.internal -e DB_USER=postgres -e DB_PASSWORD=pw \
  -e POOL_MODE=transaction -e DEFAULT_POOL_SIZE=20 \
  edoburu/pgbouncer
```

Point apps at `db-pooled.mylab.internal:5432`; PgBouncer holds ~20 real connections and juggles hundreds of clients across them (`transaction` mode: a client borrows a server connection only for the duration of each transaction). Experiment: open 200 client connections through the pooler, then check `pg_stat_activity` on the server — ~20 rows. Bonus insight: because the pooler re-resolves DNS on new server connections, it also *smooths failover* — another reason RDS Proxy cuts failover time.

---

## Part 7 — What RDS Does That We Didn't

Honest accounting, so you know what you're paying AWS for: hardware provisioning and replacement, storage that grows online (EBS), *trustworthy* failure detection (their detector has seen millions of failures; yours has seen one), patching with orchestrated failovers, backup storage lifecycle, encryption/IAM integration, and — in Aurora — a redesigned storage layer where replication happens *below* the engine (6 copies across 3 AZs at the storage tier, WAL-only writes). Aurora is a great "spot the architectural difference" study once this lecture feels easy.

### Interview relevance

This lecture *is* an interview bank: sync vs async replication tradeoffs (Part 3's hang), how PITR works (Part 5), why connection pooling matters for serverless (Part 6), what happens during Multi-AZ failover and why clients still see errors (Part 4), read replica lag and read-your-own-writes anomalies (Part 2 — try to catch one: write to the writer endpoint, instantly read from the reader). "How would you build RDS?" is a real system-design prompt at AWS/Azure — you'll have actually done it.

**Next lecture candidates:** 004 — Put it all in one compose file with your Lecture-001 lab in front of it; or 004 — Aurora anatomy: what changes when replication moves into the storage layer.
