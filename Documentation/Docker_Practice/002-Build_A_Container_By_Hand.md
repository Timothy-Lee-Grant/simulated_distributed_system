2026_07_05_16_44-Build_A_Container_By_Hand

# Lecture 002 — Build a Container by Hand (No Docker Allowed)

*A first-principles lab: recreate what `docker run` does using only raw Linux tools — `unshare`, `pivot_root`, `ip`, cgroupfs, and OverlayFS. Document 001 told you containers are "just processes wearing a costume." Today you sew the costume yourself.*

**Prerequisites:** a Linux machine or VM with root (a Raspberry Pi works beautifully — you have several), Docker installed only so we can steal a filesystem from it, and Document 001 read.

**The goal, stated precisely:** by the end, you will run a shell that (a) sees its own filesystem root, (b) believes it is PID 1, (c) has its own hostname, (d) has a private network with a working connection to the host, and (e) cannot use more than 64 MB of RAM. That is a container. No Docker process will be involved in running it.

---

## Step 0 — Get a Root Filesystem

### What we're doing

Every container needs a directory tree that will become its `/`. Docker gets this from image layers; we'll just grab Alpine Linux's userland (~3 MB) and unpack it into a plain directory.

```bash
mkdir -p ~/handmade/rootfs && cd ~/handmade

# Steal Alpine's filesystem out of a container image without running it:
docker export $(docker create alpine) | tar -C rootfs -xf -

ls rootfs
# bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

### What's going on

`docker create` prepares a container (assembles the overlay mount) but never starts a process. `docker export` streams the *merged* filesystem view as a tar. What lands in `rootfs/` is nothing magical — an ordinary directory containing a complete miniature Linux userland: BusyBox binaries, musl libc, `/etc` configs. Note what is **not** there: no kernel, no kernel modules, no bootloader. Containers bring userland only; the Landlord (host kernel) is shared. You know this split intimately from embedded — this is a rootfs image, exactly like the ones you flash, minus the kernel partition.

---

## Step 1 — The First Illusion: `chroot` (and Why It's Not Enough)

### What we're doing

The oldest isolation trick (1979, predates Linux itself): change what a process believes `/` is.

```bash
sudo chroot rootfs /bin/sh

/ # ls /          # Alpine's root, not your host's
/ # cat /etc/os-release    # Alpine Linux
/ # ps aux        # ERROR or empty — /proc isn't mounted; and we can still see a problem...
/ # exit
```

### What's going on, and why this fails as a container

`chroot(2)` changes one field in the process's `task_struct`: the pathname-resolution root. Every path lookup now starts at `rootfs/`. But *nothing else* changed:

- Mount the host's `/proc` inside and you see **all host processes** — no PID isolation.
- The process shares the host network completely.
- A root process can **escape**: the classic trick is `chroot("subdir")` followed by `chdir("../..")` repeatedly — because the kernel only blocks upward traversal *at* your root, and you've moved your root below your CWD. chroot is a lie told to path resolution, not a security boundary.

**What we're practicing:** understanding *why namespaces had to be invented*. chroot fakes the filesystem view; namespaces fake every other kernel-global view. Docker uses both — plus `pivot_root` instead of chroot, which we'll do in Step 3 because it actually detaches the old root rather than merely hiding it.

---

## Step 2 — Namespaces via `unshare`: PID and UTS

### What we're doing

`unshare(1)` wraps the `unshare(2)`/`clone(2)` syscalls from Document 001. We'll stack namespaces one at a time so you can feel each wall go up.

```bash
sudo unshare --pid --fork --mount-proc=$PWD/rootfs/proc \
     --uts --mount \
     chroot rootfs /bin/sh

/ # ps aux
# PID   USER   COMMAND
# 1     root   /bin/sh          ← you are PID 1. The host's thousands of processes: gone.
/ # hostname handmade-node-1    # allowed! it's OUR private UTS namespace
/ # hostname
# handmade-node-1
```

Meanwhile, on the host in another terminal:

```bash
hostname          # unchanged — the wall works both ways
ps aux | grep '/bin/sh'    # your "container" shell, visible as an ordinary host process
```

### What's going on

Flag by flag:

- `--pid --fork`: `unshare` creates a new PID namespace, but here's a subtlety worth savoring: **the caller doesn't join the new namespace — only its children do.** PID namespaces are special this way (a process's own PID can't change; it's baked into userspace assumptions). Hence `--fork`: unshare forks, and the *child* becomes PID 1 of the new namespace. In Docker, that child is your application. This is also why PID 1 duties (signal handling, zombie reaping, from 001 §Module 1) fall on your app.
- `--mount-proc`: `ps` doesn't use syscalls to list processes — it reads `/proc`. A new PID namespace with the *old* `/proc` mounted would still show host processes. Mounting a fresh `procfs` makes `/proc` reflect the new namespace. Lesson: **`/proc` is a window, and you must re-glaze the window when you build a new wall.**
- `--uts`: hostname/domainname become private. Two lines of kernel bookkeeping, but it's what lets every container believe it's a distinct "machine" — and it's where `socket.gethostname()` in your compose lab's `/whoami` endpoint gets its answer.
- `--mount`: private mount table, so our proc mount (and Step 3's surgery) doesn't leak to the host.

---

## Step 3 — `pivot_root`: Evicting the Old World Properly

### What we're doing

Replace chroot with what runc actually does. Run this sequence *inside* a fresh `unshare --mount --pid --fork --uts sh` on the host (not chrooted):

```bash
sudo unshare --mount --pid --fork --uts sh

# (now inside the new namespaces, still seeing the host filesystem)
cd ~/handmade
mount --bind rootfs rootfs        # pivot_root demands the new root be a mount point; bind it to itself
cd rootfs
mkdir -p old_root
pivot_root . old_root             # THE MOVE: "." becomes /, the old / hangs at /old_root
cd /
mount -t proc proc /proc          # re-glaze the window
umount -l /old_root && rmdir /old_root    # evict the host filesystem entirely

ls /            # pure Alpine
ps aux          # PID 1: sh
```

### What's going on

`pivot_root(2)` swaps two entries in this mount namespace's mount tree: the new root becomes `/`, and the old root gets re-hung on a directory you name. Then you unmount the old root, and the host filesystem is **gone from this namespace's universe** — not hidden behind a pointer like chroot, but detached. There is no `cd ../..` escape because there is no "up" anymore. The `-l` (lazy) unmount detaches it even while pieces are technically busy.

This chroot-vs-pivot_root distinction is a genuinely senior piece of knowledge: chroot restricts *path lookup*; pivot_root restructures *the mount tree itself*.

---

## Step 4 — A Private Network, Wired by Hand

### What we're doing

Recreate Document 001's veth-pair diagram with our own hands: a network namespace, a virtual cable, and IPs on both ends. Use two terminals.

```bash
# ── Terminal A (host): create a named network namespace
sudo ip netns add handmade
sudo ip netns exec handmade ip addr        # only 'lo', and it's DOWN. A machine with no NIC.

# Manufacture a virtual cable and plug one end into the namespace:
sudo ip link add veth-host type veth peer name veth-ctr
sudo ip link set veth-ctr netns handmade

# Configure the host end:
sudo ip addr add 10.99.0.1/24 dev veth-host
sudo ip link set veth-host up

# Configure the container end (from outside, using netns exec):
sudo ip netns exec handmade ip addr add 10.99.0.2/24 dev veth-ctr
sudo ip netns exec handmade ip link set veth-ctr up
sudo ip netns exec handmade ip link set lo up
sudo ip netns exec handmade ip route add default via 10.99.0.1

# The moment of truth:
ping -c2 10.99.0.2                                  # host → "container"
sudo ip netns exec handmade ping -c2 10.99.0.1      # "container" → host
```

Want the internet too? That's two more lines — the same NAT Docker installs:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 10.99.0.0/24 -j MASQUERADE
sudo ip netns exec handmade ping -c2 8.8.8.8
```

### What's going on

A fresh network namespace is a machine with its cable unplugged — only a loopback, and even that starts down. A **veth pair** is two network devices sharing one virtual wire: a frame transmitted on one end is received on the other, exactly like the crossover cables you've used on the bench. Move one end into the namespace and you've physically (well, virtually) cabled the isolated machine to the host. The `MASQUERADE` rule is source-NAT: outbound packets get their source IP rewritten to the host's, and conntrack rewrites the replies back — precisely what your home router does, and precisely what Docker sets up per bridge network. The only piece we skipped versus Docker is the bridge itself (`br0`), which you'd add if you had *multiple* namespaces needing to talk to each other — Docker's `labnet` from 001 is exactly that: N veth pairs plugged into one virtual switch.

To run your shell inside this network: add `--net=/var/run/netns/handmade` to the unshare invocation, or `sudo ip netns exec handmade unshare --mount --pid --fork ...` — composing the namespaces.

---

## Step 5 — The Utility Meter: a Cgroup, Written Raw

### What we're doing

Impose Document 001's `--memory=64m` by writing to cgroupfs ourselves (cgroup v2 assumed — any modern distro):

```bash
# ── Terminal A (host):
sudo mkdir /sys/fs/cgroup/handmade
echo 64M  | sudo tee /sys/fs/cgroup/handmade/memory.max
echo 50000 100000 | sudo tee /sys/fs/cgroup/handmade/cpu.max   # 0.5 CPU: 50ms quota per 100ms period

# Find your container shell's host PID, then imprison it:
ps aux | grep '/bin/sh'     # say it's 52001
echo 52001 | sudo tee /sys/fs/cgroup/handmade/cgroup.procs

# ── Inside the container shell: try to allocate ~100MB
/ # head -c 100m /dev/zero | tail
# Killed          ← the OOM killer, scoped to our cgroup

# ── Host: read the meter
cat /sys/fs/cgroup/handmade/memory.peak     # high-water mark before the kill
cat /sys/fs/cgroup/handmade/memory.events   # oom_kill 1
```

### What's going on

Cgroups are configured **entirely through a filesystem interface** — `echo` and `cat`, no syscalls, no libraries. `memory.max` arms a per-group limit; when the group's resident pages hit it and reclaim fails, the kernel OOM-kills within the group. `cpu.max` is the CFS bandwidth mechanism from 001: $\text{quota}/\text{period} = 50000/100000 = 0.5$ CPUs — the process isn't slowed, it's *paused* whenever it exhausts its 50ms allowance in each 100ms window. Docker's `--memory` and `--cpus` flags are, literally, ergonomic wrappers around these two `echo` commands. You have now personally been the Landlord installing the meter.

---

## Step 6 — OverlayFS by Hand: the Transparency Stack, Assembled

### What we're doing

Build the layered filesystem from 001 §Module 2 with one mount command:

```bash
cd ~/handmade
mkdir -p lower upper work merged
echo "from the read-only image layer" > lower/base.txt

sudo mount -t overlay overlay \
     -o lowerdir=lower,upperdir=upper,workdir=work merged

cat merged/base.txt                    # visible via the merged view
echo "modified!" > merged/base.txt     # write through the merged view...
cat lower/base.txt                     # "from the read-only image layer" — UNTOUCHED
cat upper/base.txt                     # "modified!" — the copy-up landed here
rm merged/base.txt
ls -la upper/                          # base.txt is now a character device 0,0 — the "whiteout"
```

### What's going on

You just watched copy-on-write and whiteouts happen at the file level: writes copy up into `upperdir`, deletes are recorded as whiteout markers that mask the lower file, and the lower layer stays byte-for-byte pristine — which is why one image can back fifty containers. Stack more `lowerdir`s with colons (`lowerdir=layer3:layer2:layer1`, leftmost wins) and you have a Docker image. Point Step 3's `pivot_root` at a `merged/` dir instead of a plain `rootfs/` and your handmade container gains a real layered root — that's the last assembly step runc performs.

---

## Step 7 — The Full Assembly, and What We Skipped

One command, composing everything (with the netns and cgroup from Steps 4–5 already created):

```bash
sudo ip netns exec handmade unshare --mount --uts --ipc --pid --fork sh -c '
  cd ~/handmade/rootfs &&
  mount --bind . . && cd . &&
  mkdir -p old_root && pivot_root . old_root &&
  mount -t proc proc /proc &&
  umount -l /old_root && rmdir /old_root &&
  hostname handmade-node-1 &&
  exec /bin/sh'
```

Compare with 001 §Module 3's control-flow diagram: you have just performed, by hand, essentially every line of the Contractor's (runc's) job. What production runc adds that we skipped — each one a worthwhile future lecture:

| Skipped | What it does | Why it matters |
|---|---|---|
| User namespaces | root inside ≠ root outside | The big rootless-container security win |
| Capabilities | drop `CAP_SYS_ADMIN` etc. from PID 1 | "root" inside a container is *already* a diminished root |
| seccomp | syscall allowlist (Docker blocks ~44 of ~350) | shrinks the kernel attack surface |
| `/dev` population | tmpfs with only null, zero, urandom... | handmade container has a stale exported /dev |
| The shim | holds stdio, reports exit status | why `docker logs` works after dockerd restarts |

## The takeaway

Docker is ~15 lines of shell, industrialized. Every flag you pass to `docker run` now has a face: `-m` is an echo into cgroupfs, `-p` is an iptables rule, `--hostname` is a UTS namespace write, the image is a stack of overlay lowerdirs, and the container itself is `unshare` + `pivot_root` + `exec`. When something breaks in your compose lab — or in production at Microsoft — you can now reason *beneath* the tool.

**Cleanup:** `sudo ip netns del handmade; sudo umount ~/handmade/merged; sudo rmdir /sys/fs/cgroup/handmade; sudo iptables -t nat -D POSTROUTING -s 10.99.0.0/24 -j MASQUERADE`

**Next lecture candidates:** 003 — Multi-stage builds & a `FROM scratch` static C binary image (your home turf); or 003 — User namespaces and rootless containers.
