2026_07_05_16_06-Docker_From_The_Kernel_Up

# CONCEPT_ENGINEERING.md — Docker From the Kernel Up

*A full walkthrough of every concept you need for the Docker_Practice project, written for an embedded/firmware engineer moving into backend, distributed systems, and cloud infrastructure.*

---

## 1. Executive Overview

This project uses Docker containers to **simulate a distributed system on a single machine**. You will run a load balancer, multiple stateless API replicas, a relational database, and a cache — each in its own container, each with its own isolated view of the OS — and then practice the things that make distributed systems hard: failure, latency, state, and coordination.

Why Docker is the right lab bench: a container gives you a cheap, disposable "machine" in milliseconds. Where a VM boots a whole guest kernel, a container is **just a Linux process wearing a costume** — isolated by kernel namespaces, constrained by cgroups, and standing on a layered filesystem. That cheapness is what lets one laptop believably impersonate a 6-node production cluster.

The payoff for your career goals: every cloud platform you listed (Kubernetes, ECS, Cloud Run, Lambda's Firecracker cousins) is built on exactly the primitives you'll learn here. Kubernetes doesn't replace this knowledge — it *automates* it.

```
                        ┌──────────────────────────────────────┐
                        │      Your Laptop (one kernel)        │
                        │                                      │
   client ──► :8080 ──► │  [nginx LB] ──► [api-1]  [api-2]     │
                        │                    │        │        │
                        │                    ▼        ▼        │
                        │               [postgres] [redis]     │
                        │                                      │
                        │  every box = 1 container = 1 process │
                        └──────────────────────────────────────┘
```

---

## 2. Your Personal Mindset Shift

You come from embedded: one binary, one board, full control, direct hardware access, and if the process dies the product is broken. That background is a **massive head start** here, and also the source of the main mental shifts you need.

| Embedded instinct | Distributed/container reality |
|---|---|
| One long-lived process that must never crash | Fleets of short-lived processes that are *expected* to crash; the system heals around them |
| State lives in the device (flash, RAM, registers) | Processes are **stateless**; state is pushed out to databases/caches so any replica can serve any request |
| You configure the exact machine the code runs on | You ship the machine *with* the code (the image) so every environment is identical |
| Upgrades are risky, scheduled flash operations | Deploys are routine: kill old containers, start new ones, roll back by starting the old image |
| Debugging = JTAG, printf over UART, one target | Debugging = logs, metrics, traces aggregated across N ephemeral targets |
| Resource limits are physical (256KB RAM, that's it) | Resource limits are *policies* (cgroups) that you set and the kernel enforces |

Here's the good news you may not expect: **you already know most of Docker's foundations.** You work in Linux daily. Containers are not a new technology stack — they are `clone(2)`, `unshare(2)`, `setns(2)`, cgroups, and a union filesystem, packaged behind a good UX. Your C/Linux background means you can understand containers at a depth most backend developers never reach. This document leans into that.

The one genuinely new muscle is **designing for failure as a normal event**. In firmware, a watchdog reset is an incident. In distributed systems, a container dying is Tuesday — the design question is never "how do I prevent failure" but "what happens to in-flight requests, state, and clients *when* it fails."

---

## 3. The Cast of Characters

You said you like personified components — meet the crew. These names are used throughout the document.

| Character | Real component | Role |
|---|---|---|
| **The Landlord** | The Linux kernel | Owns the building. Every "isolated machine" is really just a room the Landlord partitioned. Namespaces are the walls he puts up; cgroups are the utility meters he installs. |
| **The Receptionist** | `docker` CLI | You never talk to the Landlord directly. You tell the Receptionist what you want; she translates it into a REST API call and passes it along. She does no real work herself. |
| **The General Manager** | `dockerd` (Docker daemon) | Listens on `/var/run/docker.sock`. Manages images, networks, volumes, the build system. Delegates the actual running of containers downward. |
| **The Site Foreman** | `containerd` | Supervises container lifecycles: pull image, create, start, stop, report status. Doesn't care about networks or builds — just containers. |
| **The Contractor** | `runc` | The only one who actually touches kernel syscalls. Reads an OCI spec (a JSON blueprint), makes the `clone`/`unshare`/`pivot_root` calls to build the room, starts your process inside it — then *leaves*. He doesn't stick around. |
| **The Warehouse** | Registry (Docker Hub, ECR, GHCR) | Stores images as stacks of content-addressed tarballs. You `pull` from and `push` to it. |
| **The Transparency Stack** | OverlayFS | Builds the container's filesystem out of read-only layers with one writable sheet on top. |
| **The Switchboard Operator** | Docker's embedded DNS + bridge network | Lets containers find each other by name (`postgres`, `redis`) instead of by IP. |

Keep this table handy — when a later section says "the Contractor exits after starting your process," you'll know exactly who did what.

---

## 4. Module 1 — What a Container Actually Is

### What problem is being solved?

"It works on my machine" and "these two apps need conflicting library versions on the same host." Before containers, the answers were VMs (heavyweight: minutes to boot, GBs of RAM each, a full guest kernel) or careful manual environment management (fragile). Containers solve isolation *without* a guest OS.

### The theory

A container is **a normal Linux process** (or process tree) that the Landlord has lied to, in six specific ways. Each lie is a **namespace** — a kernel feature that gives a process a private view of one global resource:

| Namespace | What the process is lied to about | Flag to `clone(2)` |
|---|---|---|
| PID | Process IDs. Your app sees itself as PID 1 and sees no host processes. | `CLONE_NEWPID` |
| NET | The network. Private interfaces, own IP, own routing table, own iptables. | `CLONE_NEWNET` |
| MNT | Mount points. A completely private filesystem tree. | `CLONE_NEWNS` |
| UTS | Hostname. `hostname` inside returns the container ID. | `CLONE_NEWUTS` |
| IPC | System V IPC / POSIX message queues. | `CLONE_NEWIPC` |
| USER | UID/GID mappings. Can be root *inside* while unprivileged outside. | `CLONE_NEWUSER` |

Isolation says what you can *see*. **Cgroups** (control groups) say what you can *use*: CPU shares, memory ceilings, block I/O, PIDs. The Landlord's utility meters. When a container exceeds its memory limit, the kernel's OOM killer kills it — same mechanism as on any Linux box, scoped to the cgroup.

The third leg is the filesystem (Module 5), built by the Transparency Stack.

$$\text{container} = \text{process} + \text{namespaces} + \text{cgroups} + \text{layered rootfs}$$

There is no "Docker layer" at runtime sitting between your process and the kernel. Your containerized process makes syscalls **directly** to the host kernel. That's why containers have near-zero performance overhead and why they boot in milliseconds — there is nothing to boot.

### Prove it to yourself (commands)

```bash
# Start a long-running container
docker run -d --name demo alpine sleep 3000

# 1) It's just a process. Find it FROM THE HOST:
ps aux | grep "sleep 3000"      # visible in the host's process table, real PID e.g. 41337

# 2) But inside, it thinks it's PID 1:
docker exec demo ps aux         # sleep is PID 1

# 3) Inspect its namespaces (host side; use the real PID from step 1):
sudo ls -l /proc/41337/ns/
# lrwxrwxrwx ... net -> 'net:[4026532281]'
# lrwxrwxrwx ... pid -> 'pid:[4026532284]'
# Compare with your shell's: ls -l /proc/$$/ns/  → different inode numbers = different namespaces

# 4) Enter just its network namespace with a host tool (nsenter):
sudo nsenter -t 41337 -n ip addr    # shows the CONTAINER's eth0, not the host's

# 5) See the cgroup limits:
docker run -d --name capped --memory=64m --cpus=0.5 alpine sleep 3000
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' capped).scope/memory.max
# → 67108864  (the Landlord's meter, set to 64 MiB)
```

**What you're practicing:** breaking the illusion. Once you've seen a container from both sides — a jailed PID 1 from inside, an ordinary process from outside — you'll never be confused by container behavior again. Networking issues, permission issues, OOM kills: all of them are ordinary Linux issues in a namespace.

### What should you pay attention to?

- **PID 1 responsibilities.** Inside the PID namespace, your app is init. PID 1 does not get default signal handlers — if it doesn't handle `SIGTERM`, `docker stop` waits 10s then `SIGKILL`s it. This is why graceful shutdown handling matters (and why `docker run --init` exists: it injects a tiny init, `tini`, to reap zombies and forward signals).
- **Shared kernel.** All containers share the host kernel. `uname -r` is identical in every container. This is the fundamental security difference vs VMs, and why you can't run a Windows container on a Linux host.

### Common mistakes

1. Thinking containers are "lightweight VMs." They are processes; the mental model drives every debugging decision.
2. Running as root inside containers in production (fine for this lab, flagged in real systems).
3. Ignoring signal handling, then wondering why every deploy drops requests for 10 seconds.

### Interview relevance

"What's the difference between a container and a VM?" is a warm-up question at every company on your target list. The senior answer names namespaces + cgroups + shared kernel, and mentions the security tradeoff. "What happens when a container hits its memory limit?" → cgroup OOM kill, exit code 137 (128 + SIGKILL(9)).

---

## 5. Module 2 — Images and Layers (The Transparency Stack)

### What problem is being solved?

You need to ship a filesystem — your binary plus every library, config file, and runtime it needs — and you need to do it **efficiently**. If ten services all use Debian + Python, shipping ten full copies of Debian wastes disk, bandwidth, and build time.

### The theory

An image is **a stack of read-only tarballs (layers) plus a JSON manifest**. Each layer contains only the *filesystem diff* introduced by one build step. Layers are **content-addressed**: their ID is the SHA-256 of their contents. Same contents → same hash → stored and downloaded exactly once, shared by every image that uses it. (If you've used Git, this is the same design philosophy as Git objects.)

At runtime, the Transparency Stack (**OverlayFS**, a union filesystem in the kernel) merges the layers:

```
        ┌─────────────────────────────┐
        │  writable container layer   │  ← "upperdir": all writes land here
        ├─────────────────────────────┤
        │  layer 3: COPY app.py /app  │  ┐
        ├─────────────────────────────┤  │
        │  layer 2: pip install flask │  ├ "lowerdir": read-only image layers
        ├─────────────────────────────┤  │
        │  layer 1: debian base       │  ┘
        └─────────────────────────────┘
              ▼ merged view = the container's /
```

Reads fall through the stack top-down until a file is found. Writes trigger **copy-on-write**: the file is copied from its read-only layer up into the writable layer, and the copy is modified. Deletes create a "whiteout" file in the upper layer that masks the file below — the original bytes never leave the image layer.

Consequences worth internalizing:

- **Immutability**: 50 containers from one image share the same read-only layers in the page cache. Starting container #50 costs one new empty upperdir — microseconds.
- **Ephemerality**: destroy the container, the upperdir is deleted. Anything written inside a container that isn't in a volume is gone. This is a *feature* — it's what forces statelessness.
- **First-write latency**: modifying a large file from a lower layer costs a full copy-up. Databases must never write to the overlay (Module 6 fixes this with volumes).

### Dockerfile → layers, and the build cache

Each Dockerfile instruction that changes the filesystem (`FROM`, `RUN`, `COPY`, `ADD`) produces one layer. The builder caches layers keyed on the instruction + the input files' checksums. **Cache invalidation cascades downward**: change one instruction and every layer after it rebuilds.

This yields the single most important Dockerfile rule — **order by change frequency, rarest first**:

```dockerfile
FROM python:3.12-slim              # changes ~never
WORKDIR /app
COPY requirements.txt .           # changes occasionally
RUN pip install --no-cache-dir -r requirements.txt   # cached unless requirements.txt changed
COPY . .                          # changes every commit — LAST, so only this layer rebuilds
CMD ["python", "app.py"]
```

If `COPY . .` came before `pip install`, every code edit would re-run dependency installation. This one ordering decision is the difference between 2-second and 3-minute rebuild loops.

### Prove it to yourself

```bash
docker pull python:3.12-slim
docker image inspect python:3.12-slim --format '{{json .RootFS.Layers}}' | tr ',' '\n'
# → list of sha256 digests: the layer stack

docker history python:3.12-slim         # which instruction created each layer, and its size

# Watch copy-on-write happen:
docker run -d --name cow python:3.12-slim sleep 3000
docker exec cow sh -c 'echo hacked > /etc/hostname'
docker diff cow                          # C /etc  |  C /etc/hostname  ← changes in the upperdir only
docker inspect cow --format '{{.GraphDriver.Data.UpperDir}}'
sudo ls $(docker inspect cow --format '{{.GraphDriver.Data.UpperDir}}')/etc   # your modified file, on the host
```

**What you're practicing:** seeing that an "image" is not a disk image at all — it's a Git-like content-addressed object store that the kernel assembles into a filesystem at runtime.

### Common mistakes

1. Putting `COPY . .` early → cache-busted builds.
2. Secrets in a layer (`COPY id_rsa`, `RUN export API_KEY=...`) — deleting them in a *later* layer does NOT remove them; the bytes live in the earlier tarball forever. Use build secrets or runtime env vars.
3. Bloated images: build tools left in the final image. Fix with **multi-stage builds** (compile in one stage, `COPY --from=builder` only the artifact into a slim final stage) — as a C/C++ developer you'll love this: your final image can be `FROM scratch` plus one static binary, ~5 MB total.

### Interview relevance

"How do Docker layers work?" / "Why is your image 2 GB and how would you shrink it?" / "A secret was committed into an image layer — is deleting it in the next layer enough?" (No.)

---

## 6. Module 3 — The Engine: What Actually Happens on `docker run`

### What problem is being solved?

You should be able to trace the full control flow of the platform you depend on — that's your own stated bar for understanding a system. Also practically: this architecture (and its OCI standards) is *why* Kubernetes can run containers without Docker.

### Control flow, end to end

```
you: docker run -d -p 8080:80 nginx
        │
        ▼
[Receptionist: docker CLI]
   Parses flags → HTTP POST /containers/create over the Unix socket
        │  /var/run/docker.sock
        ▼
[General Manager: dockerd]
   Image present? No → pulls layers from the Warehouse (registry),
   verifies each sha256, stores in /var/lib/docker/overlay2.
   Creates the veth pair + iptables DNAT rule for -p 8080:80 (Module 4).
        │  gRPC
        ▼
[Site Foreman: containerd]
   Prepares the OverlayFS snapshot (lowerdirs + fresh upperdir).
   Writes an OCI runtime spec: config.json describing namespaces,
   cgroups, mounts, env, capabilities — a JSON blueprint of the "room".
   Spawns a lightweight shim process per container.
        │  exec
        ▼
[Contractor: runc]
   clone(2) with CLONE_NEWPID|NEWNET|NEWNS|... → new namespaces
   writes cgroup limits into /sys/fs/cgroup/...
   pivot_root(2) into the merged overlay filesystem
   drops capabilities, applies seccomp profile
   execve("/docker-entrypoint.sh", ...)  ← becomes YOUR process
   ...and runc itself is gone. Nothing runs "under" your process.
        │
        ▼
[your nginx, PID 1 in its namespace, a direct child of the containerd-shim]
```

Two details worth savoring:

- **The shim** is why `docker` can be restarted/upgraded without killing your running containers: containers are parented to their shim, not to dockerd.
- **OCI standards** (image-spec + runtime-spec) mean the layers are swappable. Kubernetes talks to containerd directly (skipping dockerd entirely), and you can swap runc for `gVisor` (userspace-kernel sandbox) or `Firecracker`-style microVMs (what AWS Lambda uses) without changing images. Same blueprint, different contractor.

### Prove it to yourself

```bash
docker run -d --name web nginx
ps -ef --forest | grep -A2 containerd-shim   # shim → nginx parentage, dockerd is NOT the parent

# The Receptionist is optional — talk to the General Manager yourself:
curl --unix-socket /var/run/docker.sock http://localhost/v1.47/containers/json | python3 -m json.tool
```

### Interview relevance

"Kubernetes deprecated Docker — what did that actually mean?" (It dropped dockerd as the middleman and speaks to containerd; images are unaffected because OCI.) This is a classic senior-signal question.

---

## 7. Module 4 — Networking: How Containers Talk

### What problem is being solved?

Your simulated distributed system needs N services that can (a) find each other by *name*, (b) talk privately, and (c) selectively expose ports to the outside. Real clouds solve this with VPCs, subnets, and service discovery. Docker gives you the same concepts in miniature — and it builds them from Linux primitives you can inspect.

### The theory

When Docker creates a **bridge network**, three Linux mechanisms are wired together:

1. **A bridge device** (`docker0`, or per-network `br-xxxx`) — a virtual L2 switch living in the host kernel.
2. **veth pairs** — virtual Ethernet cables. Each has two ends: one end becomes `eth0` *inside* the container's network namespace; the other end is plugged into the bridge on the host. A packet written to one end pops out the other.
3. **iptables NAT rules** — for traffic leaving to the internet (masquerade/SNAT: container IP is rewritten to host IP) and for published ports coming in (`-p 8080:80` installs a DNAT rule: host:8080 → container_ip:80).

```
  [api-1 netns]              [api-2 netns]
   eth0 172.18.0.3            eth0 172.18.0.4
      │ veth pair                │ veth pair
      ▼                          ▼
  ────┴──────── br-a1b2c3 ───────┴────   (virtual switch, host kernel)
                    │
             iptables NAT ──► host eth0 ──► internet
             DNAT: host:8080 → 172.18.0.2:80  (published port)
```

The fourth piece is the **Switchboard Operator**: on every *user-defined* network (not the default bridge!), Docker runs an embedded DNS server at `127.0.0.11` inside each container. Container names and network aliases resolve to container IPs. This is **service discovery** in its smallest form — the same job Consul, or Kubernetes' CoreDNS + Services, do at scale.

Name-based discovery is what makes IP ephemerality survivable: containers get new IPs on every restart, so *nothing should ever hardcode a container IP*. Your API connects to `postgres:5432`, and the Switchboard resolves whatever the current IP is.

### Prove it to yourself

```bash
docker network create labnet                       # user-defined bridge → DNS enabled
docker run -d --name db  --network labnet postgres:16-alpine -e POSTGRES_PASSWORD=pw
docker run -d --name app --network labnet alpine sleep 3000

# Name resolution via the embedded DNS:
docker exec app cat /etc/resolv.conf               # nameserver 127.0.0.11
docker exec app nslookup db                        # → 172.x.x.x
docker exec app ping -c1 db

# See the plumbing from the host:
ip link | grep veth                                # the host ends of the veth cables
brctl show 2>/dev/null || ip link show type bridge # the virtual switch
sudo iptables -t nat -L DOCKER -n                  # DNAT rules for published ports

# Isolation: a network is a trust boundary
docker network create othernet
docker run -d --name outsider --network othernet alpine sleep 3000
docker exec outsider nslookup db                   # FAILS — different network, no route, no DNS
```

**What you're practicing:** VPC thinking. `labnet` vs `othernet` is exactly the private-subnet segmentation you'll later express as AWS security groups or Kubernetes NetworkPolicies. Only the load balancer publishes a port; databases stay unreachable from the host network — that's defense in depth.

### Common mistakes

1. Using the default `bridge` network and wondering why DNS doesn't work (name resolution is only on user-defined networks).
2. Hardcoding container IPs.
3. Confusing `-p` (publish to host — crossing the boundary) with inter-container traffic (same network — no publishing needed; containers reach each other on the *container* port directly).
4. `localhost` confusion: inside a container, `localhost` is *that container's* loopback in its own netns — not the host, not other containers. (From container to host machine: `host.docker.internal` on Docker Desktop.)

### Interview relevance

"How does `docker run -p` actually work?" (veth + bridge + iptables DNAT). "How do containers discover each other?" → embedded DNS, and the follow-up ladder goes DNS → Consul/etcd → Kubernetes Services, straight into distributed service-discovery design questions.

---

## 8. Module 5 — Storage: Where State Is Allowed to Live

### What problem is being solved?

Module 2 established that the container's writable layer dies with the container. But Postgres must keep its data across restarts, upgrades, and crashes. You need an escape hatch from ephemerality — a *deliberate* one, so that state is the exception, not the default.

### The theory

Three mount types, all implemented as bind mounts into the container's mount namespace before your process starts:

| Type | Syntax | Backing location | Use for |
|---|---|---|---|
| **Named volume** | `-v pgdata:/var/lib/postgresql/data` | `/var/lib/docker/volumes/pgdata/_data` (Docker-managed) | Databases, anything that must survive the container |
| **Bind mount** | `-v $(pwd)/src:/app/src` | Any host path you choose | Dev loops (live code reload), injecting configs |
| **tmpfs** | `--tmpfs /scratch` | RAM only | Secrets, scratch space that must never touch disk |

Critical mechanical detail: a volume mount **bypasses OverlayFS entirely**. Writes go straight to the backing filesystem — no copy-up penalty, no whiteouts. That's why databases *require* volumes, not just prefer them: overlay write performance and durability semantics are wrong for a database's fsync-heavy workload.

The architectural principle this enforces is the one from your mindset shift table: **containers are cattle, volumes are the herd's ledger**. Compute is disposable; state is explicitly designated and survives. In the cloud, the same split appears as EC2 instance vs EBS volume, or Kubernetes Pod vs PersistentVolume.

### Prove it to yourself

```bash
docker volume create pgdata
docker run -d --name pg -e POSTGRES_PASSWORD=pw -v pgdata:/var/lib/postgresql/data postgres:16-alpine

docker exec -it pg psql -U postgres -c "CREATE TABLE t(x int); INSERT INTO t VALUES (42);"

# Destroy the container COMPLETELY:
docker rm -f pg

# Resurrect with the same volume:
docker run -d --name pg2 -e POSTGRES_PASSWORD=pw -v pgdata:/var/lib/postgresql/data postgres:16-alpine
sleep 3 && docker exec pg2 psql -U postgres -c "SELECT * FROM t;"   # → 42. State survived; compute didn't.

docker volume inspect pgdata     # see the real host path
```

### Common mistakes

1. Running a database with no volume, losing data on `docker rm` (or on `docker compose down`).
2. Two containers writing to one volume without the application being designed for it — volumes provide shared bytes, **not** coordination. (Two Postgres instances on one data dir = corruption. This is your first taste of "shared state needs a coordination protocol.")
3. Bind-mounting over a path that had image content, then wondering where the image's files went (the mount shadows them).

---

## 9. Module 6 — Lifecycle: Limits, Health, Restarts, and Dying Well

### What problem is being solved?

A distributed system is defined by how it behaves when things go wrong. Docker gives you the per-node building blocks: resource ceilings (so one bad service can't starve the host), health checks (so "running" ≠ "working" can be detected), restart policies (self-healing), and clean shutdown (so failure doesn't corrupt work in flight).

### Resource limits (the Landlord's meters)

```bash
docker run -d --name limited --memory=256m --memory-swap=256m --cpus=1.5 myapi
```

- `--memory` writes to the cgroup's `memory.max`. Exceed it → kernel OOM-kills the process → exit code **137**. There is no swap-thrash grace period if `memory-swap` equals `memory` — it's a hard ceiling, like your MCU's physical RAM, except *you* chose the number.
- `--cpus=1.5` is implemented via the CFS scheduler's quota/period mechanism ($quota = 1.5 \times period$, period default 100ms). The process isn't pinned to cores; it's *throttled*: after consuming 150ms of CPU time in a 100ms window across all cores, it's paused until the next period. Throttling shows up as mysterious tail latency — a classic production gotcha.

### Health checks: running ≠ healthy

A process can be alive but useless (deadlocked, DB connection lost, event loop blocked). A health check is a probe Docker runs *inside* the container:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
  interval: 5s        # probe every 5s
  timeout: 3s         # probe itself must answer in 3s
  retries: 3          # 3 consecutive failures → status: unhealthy
  start_period: 10s   # grace period during boot; failures don't count yet
```

Design insight that scales all the way to Kubernetes: **the service defines what "healthy" means** (its `/health` endpoint should check its own critical dependencies), and **the platform defines what to do about it** (restart, remove from load balancing). Kubernetes splits this further into liveness (restart me) vs readiness (don't send me traffic) — hold that distinction now and K8s will feel obvious later.

### Restart policies: the smallest self-healing system

```bash
docker run -d --restart unless-stopped myapi
```

| Policy | Behavior |
|---|---|
| `no` | Default. Dead stays dead. |
| `on-failure[:N]` | Restart only on non-zero exit, up to N tries, with exponential backoff. |
| `always` / `unless-stopped` | Restart on any death — the watchdog-reset instinct you already have from embedded, made policy. |

This is **desired-state reconciliation** in embryo: you declare "this should be running," and an agent (dockerd) watches reality and corrects drift. Kubernetes is this same loop, generalized to a cluster.

### Dying well: the shutdown contract

`docker stop` sends `SIGTERM`, waits `--stop-timeout` (default 10s), then `SIGKILL`. The contract your app must honor on SIGTERM: stop accepting new work → finish or hand off in-flight work → flush → exit 0. Two classic bugs:

1. **The shell-form trap.** `CMD python app.py` (shell form) runs `/bin/sh -c "python app.py"` — *sh* is PID 1 and sh does not forward signals. Your app never sees SIGTERM and gets SIGKILLed at the deadline. Always use exec form: `CMD ["python", "app.py"]`.
2. **No handler at all.** Remember Module 1: PID 1 gets no default SIGTERM disposition. Handle it explicitly, or use `--init`.

### Interview relevance

"Your container exits with 137 — what happened?" (OOM kill or SIGKILL; check `docker inspect --format '{{.State.OOMKilled}}'`.) "Difference between liveness and readiness?" "Why did the deploy drop requests?" (SIGTERM not handled / shell-form CMD.) These are bread-and-butter SRE/backend interview probes.

---

## 10. Module 7 — Docker Compose: Declaring the Whole System

### What problem is being solved?

Modules 1–6 gave you `docker run` with eight flags per container, times six containers, in the right order, on the right networks. Unmanageable by hand. Compose replaces the *imperative* command sequence with a *declarative* file: you describe the desired end state; the tool computes how to get there. This declarative-over-imperative shift is arguably **the** central idiom of modern infrastructure (Compose → Kubernetes manifests → Terraform).

### The build: your simulated distributed system

Layout for `Docker_Practice/`:

```
Docker_Practice/
├── docker-compose.yml
├── api/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py            # tiny Flask/FastAPI app: /, /health, /whoami, /counter
└── nginx/
    └── nginx.conf        # upstream pool: api-1, api-2
```

`docker-compose.yml` — read the comments; every line practices a module from this document:

```yaml
services:
  lb:                                  # ── the only door into the system
    image: nginx:alpine
    ports: ["8080:80"]                 # Module 4: the ONLY published port (DNAT rule)
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro   # Module 5: read-only bind mount
    depends_on:
      api-1: { condition: service_healthy }           # Module 9: gate on health, not just "started"
      api-2: { condition: service_healthy }
    networks: [frontnet]

  api-1: &api                          # YAML anchor: define once, reuse for api-2
    build: ./api                       # Module 2: Dockerfile → layers, cache-ordered
    environment:
      - DATABASE_URL=postgresql://postgres:pw@postgres:5432/postgres   # Module 4: DNS name, never an IP
      - REDIS_URL=redis://redis:6379
    mem_limit: 256m                    # Module 6: cgroup memory.max
    cpus: 0.5                          # Module 6: CFS quota
    restart: unless-stopped            # Module 6: reconciliation loop
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 5s
      timeout: 3s
      retries: 3
      start_period: 10s
    networks: [frontnet, backnet]      # Module 4: LB-facing AND db-facing
  api-2: *api                          # identical replica — statelessness makes this possible

  postgres:
    image: postgres:16-alpine
    environment: [POSTGRES_PASSWORD=pw]
    volumes: [pgdata:/var/lib/postgresql/data]   # Module 5: state outlives compute
    networks: [backnet]                # Module 4: NOT on frontnet, NOT published → unreachable from LB/host

  redis:
    image: redis:7-alpine
    networks: [backnet]

networks:
  frontnet: {}                         # user-defined bridges → embedded DNS on both
  backnet: {}

volumes:
  pgdata: {}
```

`nginx/nginx.conf` core:

```nginx
events {}
http {
  upstream api_pool {
    server api-1:8000;      # the Switchboard resolves these names
    server api-2:8000;      # default algorithm: round-robin
  }
  server {
    listen 80;
    location / { proxy_pass http://api_pool; }
  }
}
```

In `app.py`, make `/whoami` return the container's hostname (`socket.gethostname()` — the UTS namespace name, i.e., the container ID) and `/counter` increment a value in Redis. Twenty lines of Python.

### Run it, then run the experiments

```bash
docker compose up -d --build
docker compose ps                    # note the health column
curl localhost:8080/whoami           # repeat: alternating hostnames = round-robin across replicas
```

**Experiment 1 — Statelessness (the foundation of horizontal scaling).**
Store a counter in an *instance variable* in app.py first. `curl /counter` repeatedly: values interleave incoherently (each replica has its own counter — state trapped in compute). Move the counter to Redis: now it's coherent regardless of which replica serves you. You have just experienced *why* shared state must be externalized — not read about it, felt it.

**Experiment 2 — Node failure.**
```bash
watch -n0.3 curl -s localhost:8080/whoami   # terminal 1: steady traffic
docker compose kill api-1                    # terminal 2: murder a node
```
Traffic continues via api-2 (nginx marks the dead upstream after failed attempts — some requests error first: *failure is detected, not prevented*). `restart: unless-stopped` resurrects api-1; the health check gates it; nginx folds it back in. You've watched detection → degradation → recovery, the heartbeat of every distributed system.

**Experiment 3 — The database is a single point of failure.**
`docker compose stop postgres`. Every replica's `/health` (if it checks the DB) goes unhealthy — replicas don't help when shared state is down. This is the motivation for Phase 4 of your learning plan (replication) and for the CAP conversations you want intuition for.

**Experiment 4 — OOM in the wild.**
Add an endpoint that allocates memory in a loop. Hit it; watch `docker compose ps` show exit 137 and the restart policy revive the container. Observe a real OOM kill + self-heal cycle end to end.

**Experiment 5 — Scale without config changes.**
`docker compose up -d --scale api-1=1` is clumsy with anchors — instead try removing api-2 and using `deploy: replicas` or just: `docker compose up -d --scale api=3` after collapsing api-1/api-2 into one service named `api` and pointing nginx at `server api:8000;`. The Switchboard then returns **multiple A records** for `api` — DNS-based load balancing, no LB config edits. Compare the two approaches; that comparison *is* a system design conversation.

### Common mistakes

1. `depends_on` without `condition: service_healthy` — "started" ≠ "accepting connections"; your API crashes on boot because Postgres isn't ready. (Also: apps should retry connections anyway — never trust startup order.)
2. Forgetting `docker compose down -v` keeps vs deletes volumes (`-v` deletes them — sometimes you want a factory reset, sometimes that's a disaster).
3. Editing code and re-running `up -d` without `--build`, then debugging stale images.

---

## 11. Module 8 — The Debugging Toolkit

Ephemeral, isolated processes need a different debugging reflex than JTAG-and-printf. Your toolbox, roughly in the order you'd reach for each:

```bash
docker compose ps                        # who's alive, who's healthy, exit codes
docker compose logs -f --tail=100 api-1  # stdout/stderr — containers log to stdout by convention,
                                         # the platform handles collection (12-factor principle)
docker inspect api-1                     # the full truth: config, IPs, mounts, State.OOMKilled, health log
docker stats                             # live per-container CPU/mem — the cgroup meters, read back
docker exec -it api-1 sh                 # shell inside the namespaces
docker diff api-1                        # what has this container written to its upperdir?
docker events                            # the daemon's live event stream (die/oom/kill/health_status)

# The embedded engineer's power moves (host side):
sudo nsenter -t <PID> -n tcpdump -i eth0        # sniff INSIDE a container's netns with host tools
sudo nsenter -t <PID> -n ss -tlnp               # what's actually listening in there
cat /sys/fs/cgroup/system.slice/docker-<id>.scope/memory.peak   # high-water mark vs the limit
```

Debugging heuristic, distilled: **is the problem inside the walls or in the walls themselves?** App bugs → `logs` + `exec`. Platform bugs (connectivity, ports, OOM, restarts) → `inspect`, `events`, iptables, and the namespace tools from Module 1. Knowing which side of the wall you're on is half the diagnosis.

Common pitfall: minimal images (`alpine`, distroless, `FROM scratch`) may lack `curl`, `ps`, even a shell. Trick: attach a tools container to the *same network namespace*: `docker run -it --rm --network container:api-1 nicolaka/netshoot` — full toolkit, the target's exact network view, zero changes to the target.

---

## 12. Mental Sandbox & Next Steps

Three design challenges, aligned with where you're heading. Do them on paper first, then in the lab.

**Challenge 1 — Sticky sessions vs statelessness.**
A teammate proposes storing user sessions in each API replica's memory and enabling nginx `ip_hash` so users always hit the same replica. It works in the demo. Enumerate everything that breaks: a replica dies (those users are logged out), scaling up (hash ring reshuffles), rolling deploys (every deploy logs everyone out), uneven load (one office building = one IP = one replica). Then design the fix (session store in Redis — Experiment 1 generalized) and articulate the tradeoff you accepted (a network hop per request; Redis as a new dependency to keep healthy). This exact scenario is a system-design interview staple.

**Challenge 2 — The deploy with zero dropped requests.**
Using only tools from this document (health checks, `SIGTERM` handling, nginx upstreams, `start_period`), design a rolling deploy of a new API image where `watch curl` never shows a single error. Sequence matters: start new replica → wait healthy → add to pool → drain old (stop sending, let in-flight finish) → SIGTERM → remove. Then implement it as a bash script against your compose stack. You will have re-derived what Kubernetes rolling updates do — which is the best possible preparation for learning them.

**Challenge 3 — Postgres has to survive a node loss.**
Sketch how you'd run a Postgres primary + streaming replica as two containers with two volumes, and what *failover* requires: detection (health check on the primary), promotion (`pg_ctl promote` on the replica), and redirection (how do the APIs learn the new primary's address? DNS alias flip? A proxy like pgbouncer/HAProxy?). Notice the hard part is not the copying of data — it's the *agreement about who is primary*. Sit with why automated failover is dangerous (split-brain: two primaries both accepting writes). You are now standing at the front door of consensus algorithms — Raft is the formal answer, and it's Phase 9 of your plan.

### Natural next topics (in order)

1. **Multi-stage builds & image hardening** — small attack surface, non-root user, `FROM scratch` for static binaries (your C background makes this fun).
2. **Failure injection** — Toxiproxy between api and postgres: add 500ms latency and watch your timeouts/retries actually matter. This builds the async/timeout intuition you flagged as a weakness.
3. **Observability** — Prometheus + Grafana as containers in this same compose file; instrument app.py with a request counter and latency histogram.
4. **kind/k3d** — re-deploy this exact system on local Kubernetes; every concept here maps 1:1 (health checks → probes, restart policy → controllers, networks → Services/NetworkPolicies, volumes → PVCs).

---

## 13. Quick-Reference: Concepts ↔ Commands

| Concept | One command that demonstrates it |
|---|---|
| Container = process | `ps aux \| grep <cmd>` on the host |
| Namespaces | `ls -l /proc/<pid>/ns/` |
| Cgroup limits | `docker run --memory=64m` + `cat .../memory.max` |
| Layers / CoW | `docker history <img>`, `docker diff <ctr>` |
| Engine chain | `ps -ef --forest \| grep shim` |
| Bridge + veth + DNAT | `ip link`, `iptables -t nat -L DOCKER -n` |
| Service discovery | `docker exec <ctr> nslookup <name>` |
| Volumes bypass overlay | `docker rm -f` + re-run with same `-v` |
| Health ≠ running | `docker compose ps` health column |
| Self-healing | `docker compose kill api-1` + watch it return |
| Graceful shutdown | exec-form CMD + SIGTERM handler + `docker stop` |

---

*Generated for the Docker_Practice project — companion to `Documentation/Plans/distributed_systems_learning_ideas.md` (this document deep-dives Phases 1–2 and the Docker foundations under all nine phases).*
