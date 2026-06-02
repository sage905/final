# Fix: Mono/Unity game servers segfault under Pelican Wings

**Applies to:** Pelican Wings (rootful Podman) on Ubuntu 24.04 / kernel 6.8.
**Symptom:** Rust (and other Unity/Mono-based) dedicated servers crash
immediately on startup inside their container.

## Symptom

The game-server console shows a fatal signal during garbage-collector thread
suspension, then a segfault:

```
Caught fatal signal - signo:11 code:128 errno:0 addr:(nil)
Obtained 22 stack frames.
#1  ... in GC_suspend_all
#2  ... in GC_stop_world
...
pthread_kill failed at suspend
Segmentation fault (core dumped)
Main game process exited with code 139
```

`strace` on the process shows the GC's thread-suspend signal being rejected:

```
tgkill(1, 49, SIGPWR) = -1 EACCES (Permission denied)
```

## Root cause

Pelican Wings **hardcodes** `no-new-privileges` on every game-server container.
In [`environment/docker/container.go`](https://github.com/pelican-dev/wings/blob/v1.0.0-beta25/environment/docker/container.go):

```go
SecurityOpt:    []string{"no-new-privileges"},
ReadonlyRootfs: true,
```

Unity bundles Mono, whose Boehm GC suspends managed threads by sending signals
with `pthread_kill`/`tgkill`. Under `no-new-privileges` + kernel 6.8, the
kernel's signal permission check returns `EACCES`, the GC aborts, and the
process segfaults. (This overlaps Unity bug **UUM-72029**, fixed in Unity
6000.0.20f1+, but the container environment triggers the same Boehm-GC path
regardless of Unity version.)

Confirmed: running the identical container **without** `no-new-privileges`
starts the server successfully.

### Things that do NOT work

- **`+gc.incremental 0`** — disables Unity's *incremental* GC; the crash is in
  the separate bundled *Boehm* GC.
- **Custom seccomp profile** — `EACCES` here is from the kernel's signal
  permission check, not a seccomp syscall filter.
- **`/etc/containers/containers.conf` → `no_new_privileges = false`** — only
  affects containers Podman creates itself. Wings passes the flag directly in
  the OCI container spec via the Docker-compatible API, bypassing
  `containers.conf` entirely.

The only fix on the Podman side is to remove the hardcoded `SecurityOpt` and
recompile Wings.

## Fix: build a patched Wings binary

Wings requires the **Go 1.25** toolchain; Ubuntu 24.04's apt `golang` (1.22) is
too old, so install Go from upstream. Run as root on the Wings host:

```bash
# 1. Go 1.25 toolchain
cd /tmp
curl -fsSL https://go.dev/dl/go1.25.0.linux-amd64.tar.gz -o go.tgz
rm -rf /usr/local/go && tar -C /usr/local -xzf go.tgz
export PATH=$PATH:/usr/local/go/bin
go version                       # expect go1.25.0

# 2. Source at the exact tag you run (match your installed Wings version)
apt-get install -y git
cd /usr/local/src
git clone --depth 1 --branch v1.0.0-beta25 https://github.com/pelican-dev/wings
cd wings

# 3. Remove the hardcoded SecurityOpt
sed -i 's/\[\]string{"no-new-privileges"}/[]string{}/' environment/docker/container.go
grep -n 'SecurityOpt' environment/docker/container.go   # verify -> []string{}

# 4. Build
CGO_ENABLED=0 go build -o wings_patched .
./wings_patched version

# 5. Swap in, keeping a backup
systemctl stop wings
cp -a /usr/local/bin/wings /usr/local/bin/wings.bak.$(date +%Y%m%d)
install -m 0755 wings_patched /usr/local/bin/wings
systemctl start wings
systemctl status wings --no-pager
```

Start the game server again and confirm it gets past `GC_suspend_all`.

## ⚠️ This patch is reverted by the Ansible role

The `sage.final.pelican_wings` role installs the **upstream release** binary,
and with the default `pelican_wings_version: latest` it re-downloads with
`force: true` on every run — overwriting this patched binary.

Until the fix is baked into the role, either:

- exclude this host from the `pelican_wings` play, or
- re-apply the patch after any role run that touches the binary.

A durable approach (build-from-source gated behind a role flag, pinned to a
version tag) is the intended follow-up.

## Security note

Removing `no-new-privileges` is a genuine relaxation: a setuid binary inside a
container could regain privileges. It is mitigated here by Wings' other
hardening on the same container — `ReadonlyRootfs: true`, an empty effective
capability set, and a restrictive capability bounding set (`CapDrop` includes
`setuid`/`setpcap`/`sys_ptrace`/etc.). The relaxation applies to all containers
on the node, not just Mono-based ones.
