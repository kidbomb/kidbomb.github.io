---
layout: post
title: Container runtimes - runc - Create and start stages
tags: container-runtimes security
comments: true
description: Part 4 of the container runtimes series. The rootfs, capabilities, seccomp, cgroups, OCI hooks, and what runc start actually triggers.
series: Container runtimes
series_order: 4
series_title: runc create and start stages
---

{% include series-nav.html %}

# Introduction

Part 3 left `runc init` in a very specific position: namespaces created, and the process blocked on a read from `/run/runc/mycontainer/exec.fifo`. Nothing has been `exec`'d yet. The container exists, and it is doing nothing.

That pause is the most interesting moment in the whole lifecycle, because it is the last point at which the container's security posture is still being assembled. This post covers what happens around it:

- what the **parent** `runc` process did while init was blocked (cgroups and state)
- the **filesystem** work: mount propagation, `pivot_root`, masked and read-only paths
- the **security context**: capabilities, `no_new_privs`, seccomp, and why the ordering matters
- the **OCI hooks**, and which of them run inside the container
- what `runc start` actually does

{% mermaid %}
flowchart TD
    A["runc create"] --> B["parent: create cgroup<br/>write state.json"]
    B --> C["init: mount propagation → private"]
    C --> D["init: mount /proc, /sys, /dev<br/>apply masked + readonly paths"]
    D --> E["init: pivot_root into rootfs"]
    E --> F["hook: createContainer<br/>(inside container namespaces)"]
    F --> G["init: drop capabilities<br/>set no_new_privs"]
    G --> H["init: block on exec.fifo"]
    H -. "state: created" .-> I["runc start"]
    I --> J["write to exec.fifo"]
    J --> K["hook: startContainer"]
    K --> L["apply seccomp filter"]
    L --> M["execve() the container process"]
    M --> N["state: running"]

    classDef parent fill:#e8f5e9,stroke:#43a047,stroke-width:2px,color:#14321a
    classDef init fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px,color:#311b3f
    classDef hook fill:#fce4ec,stroke:#d81b60,stroke-width:2px,color:#3f1220
    classDef start fill:#fff3e0,stroke:#fb8c00,stroke-width:2px,color:#3e2600

    class A,B parent
    class C,D,E,G,H init
    class F,K hook
    class I,J,L,M,N start
{% endmermaid %}

Everything below was produced with `runc` 1.4.3 (OCI spec 1.3.0) against the Alpine bundle we built in Part 3.

# What the parent process did

While `runc init` was busy creating namespaces, the parent `runc create` process was doing work of its own. Two pieces of it are visible from the outside.

**The cgroup already exists, before the container has started.** With the container in the `created` state:

```console
$ PID=$(sudo runc state mycontainer | jq -r .pid)
$ cat /proc/"$PID"/cgroup
0::/mycontainer
```

The init process is already confined, and `/sys/fs/cgroup/mycontainer` is already populated with the usual cgroup v2 interface files:

```console
$ ls /sys/fs/cgroup/mycontainer
cgroup.controllers  cgroup.freeze  cgroup.max.depth  cgroup.procs
cgroup.events       cgroup.kill    cgroup.max.descendants  cgroup.stat
```

This is a deliberate division of labor, and it is the one place where the "init does everything" mental model breaks down. Cgroups are created and applied by the **parent**, which then moves the init process into them. A process cannot reliably place itself into a restrictive cgroup, because the limits may take effect before it finishes setting itself up. Having the parent do it also means the limits are already in force during the rest of init's work, so a hook or a malicious image cannot exhaust host memory during setup.

Recall from Part 3 that the default spec has no `linux.cgroupsPath`. When it is unset, `runc` derives the path from the container ID, which is why this landed in `/mycontainer`.

**The state is on disk.** `runc create` writes a `state.json` under its root directory (`/run/runc/<id>/` by default). That file is what makes `runc state` and `runc list` work from any process, and it is what lets a shim reattach to a container it did not create.

# Filesystem setup

The rootfs work happens inside init, in the new mount namespace, and it proceeds in a careful order.

## Propagation first

The first mount-related thing `runc` does is set the propagation type of the whole tree, exactly like the `mount --make-rprivate /` we had to add by hand in Part 3. If it skipped this, every mount it subsequently made could propagate back to the host. This single step is the difference between a container and a very elaborate way to modify the host's mount table.

## Mounting the virtual filesystems

Next come the mounts from the spec. The default `runc spec` lists seven:

```console
$ jq -c '[.mounts[].destination]' config.json
["/proc","/dev","/dev/pts","/dev/shm","/dev/mqueue","/sys","/sys/fs/cgroup"]
```

Inside the running container, these show up as you would expect:

```
proc /proc proc rw,relatime 0 0
tmpfs /dev tmpfs rw,nosuid,size=65536k,mode=755,inode64 0 0
devpts /dev/pts devpts rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666 0 0
shm /dev/shm tmpfs rw,nosuid,nodev,noexec,relatime,size=65536k,inode64 0 0
mqueue /dev/mqueue mqueue rw,nosuid,nodev,noexec,relatime 0 0
```

Note `nosuid`, `nodev`, and `noexec` appearing throughout. These are not decoration. A writable `/dev/shm` without `noexec` is a convenient place to drop a payload.

Mounting a **fresh** `/proc` is what makes the PID namespace observable, which is the fix we worked through at length in Part 3. `runc` gets this right by default.

## Masked and read-only paths

`/proc` is a problem. Some of its entries leak host state or allow host manipulation even from inside a fully namespaced container, so the spec carries two lists that `runc` applies after mounting it:

```console
$ jq -c '.linux.maskedPaths' config.json
["/proc/acpi","/proc/asound","/proc/kcore","/proc/keys","/proc/latency_stats",
 "/proc/timer_list","/proc/timer_stats","/proc/sched_debug","/sys/firmware","/proc/scsi"]

$ jq -c '.linux.readonlyPaths' config.json
["/proc/bus","/proc/fs","/proc/irq","/proc/sys","/proc/sysrq-trigger"]
```

Masking is implemented by bind-mounting `/dev/null` over the file (or an empty read-only `tmpfs` over a directory). You can see the result from inside the container:

```console
/ # ls -l /proc/kcore
crw-rw-rw-    1 root     root        1,   3 Sep 19 16:10 /proc/kcore
```

`/proc/kcore` is normally a window into all of physical memory. Here it is character device 1,3, which is `/dev/null`. `/proc/sysrq-trigger`, one line further down that list, would otherwise let a container reboot the host.

The read-only paths are enforced with a read-only bind mount, visible in the init process's `mountinfo`:

```
2779 6714 0:100 /sys /proc/sys ro,relatime - proc proc rw
```

And from inside:

```console
/ # touch /proc/sys/probe
touch: /proc/sys/probe: No such file or directory
```

## pivot_root

Finally, `runc` changes the root. It prefers `pivot_root()` over `chroot()`, and the distinction matters for security rather than convenience.

`chroot()` only changes the process's idea of `/`. The old root is still mounted and still reachable, and the ways out of a `chroot` are well documented and easy. `pivot_root()` moves the old root to a new mount point and then unmounts it, so the host filesystem is genuinely gone from the container's mount namespace, not merely hidden.

If the spec sets `root.readonly` (the default spec does), the rootfs is then remounted read-only:

```console
$ jq -c '.root' config.json
{"path":"rootfs","readonly":true}
```

```console
/ # touch /probe
touch: /probe: Read-only file system
```

# The security context

With the filesystem in place, init narrows what the process is allowed to do. Inspecting the blocked init process from the host shows the result:

```console
$ grep -E '^(Name|CapEff|CapBnd|NoNewPrivs|Seccomp)' /proc/"$PID"/status
Name:	runc:[2:INIT]
CapEff:	0000000020000420
CapBnd:	0000000020000420
NoNewPrivs:	1
Seccomp:	0
```

## Capabilities

The default spec grants three capabilities:

```console
$ capsh --decode=0000000020000420
0x0000000020000420=cap_kill,cap_net_bind_service,cap_audit_write
```

Three, out of the 41 the kernel currently defines (`cat /proc/sys/kernel/cap_last_cap` returns 40). This is a much narrower default than people assume, and it is worth contrasting with Docker, which grants fourteen:

```console
$ docker run --rm alpine sh -c 'grep CapEff /proc/self/status'
CapEff:	00000000a80425fb

$ capsh --decode=00000000a80425fb
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,
cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,
cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
```

The important field is `CapBnd`, the bounding set. Effective capabilities can be regained by a `setuid` binary; capabilities removed from the **bounding set** cannot be recovered by any means, for this process or any of its descendants. `runc` shrinks the bounding set, which is what makes the restriction permanent.

## no_new_privs

```
NoNewPrivs:	1
```

Once this bit is set it cannot be unset, and it is inherited across `execve()`. It means a `setuid` binary inside the container gains nothing: running `/bin/su` will not give you UID 0. It is a single bit that neutralizes an entire category of privilege escalation, and it is also a prerequisite for applying an unprivileged seccomp filter.

## Seccomp, and why it comes last

```
Seccomp:	0
```

Zero means **disabled**. This surprises people, so it is worth stating plainly: `runc spec` generates no seccomp profile at all. The well-known default profile that blocks several dozen syscalls belongs to **Docker**, not to `runc`. A container you launch with bare `runc` and a default spec has the full host syscall surface available.

When a profile *is* configured, its position in the sequence is deliberate: the filter is installed at the very end of init, immediately before `execve()`. The reason is simple. Init's own setup work needs syscalls (`mount`, `pivot_root`, `setns`, `capset`) that no sane container profile would allow. Installing the filter early would block `runc` from finishing the job. So the filter goes on last, guarding only the container process.

This creates a narrow but real window: every hook that runs before that point runs *without* the seccomp filter that the container itself will be subject to.

# OCI hooks

Hooks are the extension point of the runtime spec, and they are where a surprising amount of real-world container security goes wrong. There are five, and the two properties that matter are *when* they run and *in which namespaces*:

| Hook | Runs during | Namespace | Notes |
|---|---|---|---|
| `prestart` | create | runtime (host) | Deprecated in favour of `createRuntime` |
| `createRuntime` | create | runtime (host) | After namespaces exist, before `pivot_root` |
| `createContainer` | create | **container** | Sees the container's mounts; before `pivot_root` |
| `startContainer` | start | **container** | After `pivot_root`, immediately before `execve()` |
| `poststart` | start | runtime (host) | After the container process is running |

`createContainer` and `startContainer` run **inside the container's namespaces**. They execute a binary from the host filesystem, with host privileges, in a context that the container's configuration partly controls. They also run before the seccomp filter is installed.

That combination is not hypothetical. It is exactly the shape of [CVE-2025-23266](/posts/nvidia-container-escape-cve-2025-23266/), where the NVIDIA Container Toolkit's `createContainer` hook inherited environment variables from the container image, allowing `LD_PRELOAD` to be set for a host binary running with full privileges. The [follow-up post](/posts/nvidia-container-escape-cve-2025-23266-part-2/) traces exactly how that environment inheritance happens.

If you write a hook, the rules follow directly from the table: never trust anything originating from the image, including the environment, and assume the container may have influenced every path you touch.

# The start stage

After all of that, `runc start` is almost an anticlimax. It writes a single byte to the FIFO, and init, which has been blocked this whole time, wakes up and calls `execve()`.

The cleanest way to see it is to watch a single PID across the transition. Create a container whose command is `sleep 120`:

```console
$ sudo runc state mycontainer | jq -c '{status,pid}'
{"status":"created","pid":32}

$ grep ^Name /proc/32/status
Name:	runc:[2:INIT]
```

Now start it:

```console
$ sudo runc start mycontainer

$ sudo runc state mycontainer | jq -c '{status,pid}'
{"status":"running","pid":32}

$ grep -E '^(Name|CapEff|NoNewPrivs)' /proc/32/status
Name:	sleep
CapEff:	0000000020000420
NoNewPrivs:	1
```

The PID never changed. `runc:[2:INIT]` **became** `sleep`. There was no fork at start time; `execve()` replaced the program image of the process that had been sitting there since create, and every namespace, capability set, and cgroup membership established during init carried straight over.

Inside the container, that process is PID 1 and the world is empty:

```console
$ sudo nsenter -t 32 -p -m -- ps -o pid,comm
PID   COMMAND
    1 sleep
    7 ps
```

One thing to watch out for while experimenting: if you leave the default `process.args` of `["sh"]` with `terminal` set to `false` and no stdin attached, `sh` reads EOF and exits immediately, so the container goes straight from `created` to `stopped` and `runc start` looks like it did nothing. Use a long-running command such as `sleep` to observe the `running` state.

You can also check resource accounting through the cgroup the parent set up:

```console
$ sudo runc events --stats mycontainer | jq -c '{pids: .data.pids}'
{"pids":{"current":1}}
```

## Teardown

When the process exits, the container becomes `stopped` but the state directory and cgroup remain until you remove them:

```bash
sudo runc delete mycontainer
```

`poststop` hooks run here. Note that a stopped container still occupies its cgroup and its entry under `/run/runc`, which is why orchestrators that crash mid-lifecycle tend to leave debris behind.

# What this means in practice

Walking through the sequence, a few things stand out:

1. **The isolation is assembled incrementally, not atomically.** There is no single instant where a container "becomes" isolated. There is a sequence of steps, and hooks run partway through it.

2. **`runc`'s defaults are narrower than most people think in some places and wider in others.** Three capabilities and `no_new_privs` are good defaults. No seccomp profile at all is not, and the syscall filtering you may be assuming comes from Docker, not from the runtime.

3. **Hooks are the soft underbelly.** Two of the five run inside the container's namespaces, before seccomp, with host privileges. Every container escape that involves a hook follows from that one fact.

4. **Masked and read-only paths are doing quiet, load-bearing work.** Fifteen entries across those two lists are what stand between a namespaced process and reading physical memory (`/proc/kcore`) or rebooting the host (`/proc/sysrq-trigger`).

### What's next?

That completes the `runc` walkthrough. `runc` is the reference implementation, but it shares the host kernel, and everything above is ultimately a set of kernel-enforced restrictions on a normal process.

The next posts in this series look at runtimes that take a different approach to that problem: Kata Containers, which puts each container in a lightweight VM, and gVisor, which puts a user-space kernel in between.

## References

- [OCI Runtime Specification – config.json](https://github.com/opencontainers/runtime-spec/blob/main/config.md)
- [OCI Runtime Specification – Linux-specific configuration](https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md)
- [OCI Runtime Specification – lifecycle and hooks](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md)
- [runc source](https://github.com/opencontainers/runc)
- [capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [pivot_root(2)](https://man7.org/linux/man-pages/man2/pivot_root.2.html)
