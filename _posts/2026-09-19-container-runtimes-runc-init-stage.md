---
layout: post
title: Container runtimes - runc - Init stage
tags: container-runtimes security
comments: true
description: Part 3 of the container runtimes series. Deep dive into runc init internals, namespaces, OCI bundles, and exec FIFO synchronization.
series: Container runtimes
series_order: 3
series_title: runc init stage
---

{% include series-nav.html %}

# Introduction

In the previous post, we got a bird's-eye view of the `runc` lifecycle. This post goes deeper into the init phase, and focuses on the first thing it does: **namespace creation**. Filesystem setup, the security context, and cgroups are covered in Part 4.

# Linux namespaces

Not to be confused with Kubernetes namespaces, a Linux namespace is a kernel mechanism used to isolate resources for a process. In practical terms, that means a process can be isolated from other processes, mounts, networks, and users.

The kernel currently defines eight namespace types:

| Namespace | Isolates | Since |
|---|---|---|
| Mount (`mnt`) | mount points | Linux 2.4.19 |
| UTS (`uts`) | hostname and NIS domain name | Linux 2.6.19 |
| IPC (`ipc`) | System V IPC and POSIX message queues | Linux 2.6.19 |
| Process ID (`pid`) | process ID number space | Linux 2.6.24 |
| Network (`net`) | network devices, stacks, ports | Linux 2.6.29 |
| User (`user`) | user and group ID number space | Linux 3.8 |
| Cgroup (`cgroup`) | cgroup root directory | Linux 4.6 |
| Time (`time`) | boot and monotonic clocks | Linux 5.6 |

Most treatments of this topic stop at the first six, but `runc` creates a cgroup namespace by default too, as we will see when we inspect a generated spec.

A process does not really "enter" a namespace in one single way. There are three distinct operations:

- `clone()` creates a **new** process in new namespaces
- `unshare()` moves the **calling** process into new namespaces
- `setns()` joins an **existing** namespace

Creating most namespace types requires `CAP_SYS_ADMIN`. The exception is the user namespace: an unprivileged process can create one, and it holds a full capability set *inside* it. This is what makes rootless containers possible, and it is also why user namespaces are the most security-sensitive of the eight.

Let's dive into each one and run a few experiments. All examples are written in Go.

## Mount (mnt)

The mount namespace isolates the set of filesystem mount points seen by a group of processes. Processes in different mount namespaces can have completely different views of the filesystem hierarchy.

In container runtimes, this is combined with `pivot_root` (or `chroot`) so the process sees the container root filesystem instead of the host's (for example, layers under `/var/lib/docker/overlay2/` in Docker).

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "syscall"
)

func main() {
    cmd := exec.Command("/bin/bash")
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWNS, // Create new mount namespace
    }
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        fmt.Printf("Error: %v\n", err)
    }
}
```

There is a subtlety here that trips up almost everyone the first time. A new mount namespace starts as a *copy* of the parent's mount table, and by default those mounts are **shared**: a mount you create inside the namespace can propagate back out to the host. To get the isolation you probably expect, you have to change the propagation type first:

```bash
mount --make-rprivate /
```

We will see the consequences of forgetting this in the combined example below.

## Process ID (pid)

The PID namespace isolates the process ID number space, so processes in different PID namespaces can have the same PID. The first process in a new PID namespace becomes PID 1 (the init process) within that namespace, even though it has a different PID in the parent namespace.

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "syscall"
)

func main() {
    cmd := exec.Command("/bin/sh", "-c", "echo 'PID in namespace:' $$; sleep 30")
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWPID,
    }
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    fmt.Printf("Starting process with PID namespace isolation\n")
    if err := cmd.Start(); err != nil {
        fmt.Printf("Error starting: %v\n", err)
        return
    }

    fmt.Printf("Process PID in parent namespace: %d\n", cmd.Process.Pid)

    if err := cmd.Wait(); err != nil {
        fmt.Printf("Process finished with error: %v\n", err)
    }
}
```

This is crucial for containers because it allows each container to have its own init process (PID 1) and prevents processes inside the container from seeing or signaling processes outside their namespace.

One important caveat: a PID namespace on its own changes what PIDs processes *have*, not what `ps` *shows*. `ps` reads `/proc`, so without a matching mount namespace and a fresh `/proc` mount, you will still see the host's process list.

## Network (net)

Network namespaces provide isolation of network resources, including network devices, the IPv4 and IPv6 protocol stacks, IP routing tables, firewall rules, port numbers, and the `/proc/net` directory. Each network namespace has its own loopback device and can have its own network interfaces.

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "syscall"
)

func main() {
    cmd := exec.Command("/bin/bash", "-c", `
        echo "Network interfaces in namespace:"
        ip link show
        echo "Routing table:"
        ip route show
    `)

    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWNET,
    }
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        fmt.Printf("Error: %v\n", err)
    }
}
```

In a fresh network namespace, only the loopback interface exists, and it starts in the `DOWN` state:

```
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

Container runtimes like Docker create virtual ethernet pairs (`veth`) to connect containers to the host network. Note that `runc` itself does **not** do this: networking is left to the caller, which is why a raw `runc` container has no connectivity unless you set it up yourself.

## Inter-process communication (ipc)

IPC namespaces isolate System V IPC objects and POSIX message queues: semaphores, message queues, and shared memory segments. Processes in different IPC namespaces cannot communicate through these mechanisms.

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "syscall"
)

func main() {
    // Show IPC resources on the host
    out, err := exec.Command("ipcs").Output()
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("IPC resources on host:\n%s\n", out)

    cmd := exec.Command("/bin/bash", "-c", `
        echo "IPC resources in namespace:"
        ipcs
        echo "Creating a message queue..."
        ipcmk -Q
        ipcs -q
    `)

    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWIPC,
    }
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        fmt.Printf("Error: %v\n", err)
    }
}
```

Note the use of `Output()` rather than `Run()` for the host-side command. `Run()` discards the child's output unless you wire up `Stdout` yourself, so the host listing would silently print nothing.

The message queue created inside the namespace disappears when the last process in that namespace exits.

## UTS (UNIX Time-sharing System)

UTS namespaces isolate the system hostname and NIS domain name. This allows each namespace to have its own hostname, which is useful for containers that need to appear as separate systems.

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "syscall"
)

func main() {
    out, err := exec.Command("hostname").Output()
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("Host hostname: %s", out)

    cmd := exec.Command("/bin/bash", "-c", `
        echo "Original hostname in namespace: $(hostname)"
        hostname container-host
        echo "New hostname in namespace: $(hostname)"
    `)

    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS,
    }
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        fmt.Printf("Error: %v\n", err)
    }

    out, _ = exec.Command("hostname").Output()
    fmt.Printf("Host hostname after namespace change: %s", out)
}
```

A new UTS namespace **inherits** the parent's hostname rather than starting blank, which is why the first line inside the namespace still prints the host's name. Changing it afterwards has no effect on the host.

## User (user)

User namespaces isolate user and group ID number spaces. A process can have different user and group IDs inside and outside a user namespace. Most importantly, a process can have root privileges (UID 0) inside a user namespace while being an unprivileged user outside it.

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "syscall"
)

func main() {
    cmd := exec.Command("/bin/bash", "-c", `
        echo "UID in namespace: $(id -u)"
        echo "GID in namespace: $(id -g)"
        echo "User: $(whoami)"
        cat /proc/self/uid_map
        cat /proc/self/gid_map
    `)

    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUSER,
        UidMappings: []syscall.SysProcIDMap{
            {ContainerID: 0, HostID: os.Getuid(), Size: 1},
        },
        GidMappings: []syscall.SysProcIDMap{
            {ContainerID: 0, HostID: os.Getgid(), Size: 1},
        },
    }
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        fmt.Printf("Error: %v\n", err)
    }
}
```

The `UidMappings` field is what makes this work: it writes `/proc/<pid>/uid_map` on our behalf, mapping container UID 0 to our real host UID. Without a mapping, every UID in the namespace resolves to the overflow ID (`nobody`, 65534).

The privileges you gain are real, but they are scoped: you hold capabilities **over resources owned by the namespace**, not over the host. Being root inside the namespace does not let you read a host file owned by the real root.

## Putting it all together

Now that we have explored each namespace individually, let's see how they work together. In real environments these namespaces are created together, in a single `clone()` call.

Here is an example that creates a process in all six of the classic namespaces:

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
    "syscall"
)

func main() {
    fmt.Println("Creating a fully isolated container-like environment...")

    cmd := exec.Command("/bin/bash", "-c", `
        echo "=== CONTAINER ENVIRONMENT ==="
        echo "Hostname: $(hostname)"
        echo "PID: $$"
        echo "User: $(whoami) (UID: $(id -u))"
        echo ""

        echo "=== PROCESS ISOLATION ==="
        ps aux | head -5
        echo ""

        echo "=== NETWORK ISOLATION ==="
        ip link show
        echo ""

        echo "=== FILESYSTEM ISOLATION ==="
        ls -la / | head -5
    `)

    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWPID | // Process isolation
            syscall.CLONE_NEWNS | // Mount isolation
            syscall.CLONE_NEWNET | // Network isolation
            syscall.CLONE_NEWIPC | // IPC isolation
            syscall.CLONE_NEWUTS | // Hostname isolation
            syscall.CLONE_NEWUSER, // User isolation

        // Map current user to root inside the namespace
        UidMappings: []syscall.SysProcIDMap{
            {ContainerID: 0, HostID: os.Getuid(), Size: 1},
        },
        GidMappings: []syscall.SysProcIDMap{
            {ContainerID: 0, HostID: os.Getgid(), Size: 1},
        },
    }

    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        fmt.Printf("Error: %v\n", err)
    }
}
```

Build and run it:

```bash
go build -o container-like container-like.go
./container-like
```

On my Ubuntu laptop, this is the result:

```
Creating a fully isolated container-like environment...
=== CONTAINER ENVIRONMENT ===
Hostname: filipe-rodrigues-zbook-fury
PID: 1
User: root (UID: 0)

=== PROCESS ISOLATION ===
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
nobody         1  0.0  0.0  25476 16612 ?        Ss   Sep17   0:46 /sbin/init splash
nobody         2  0.0  0.0      0     0 ?        S    Sep17   0:00 [kthreadd]
nobody         3  0.0  0.0      0     0 ?        S    Sep17   0:00 [pool_workqueue_release]
nobody         4  0.0  0.0      0     0 ?        I<   Sep17   0:00 [kworker/R-rcu_gp]

=== NETWORK ISOLATION ===
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

=== FILESYSTEM ISOLATION ===
ls: cannot open directory '/': Permission denied
```

This output is worth reading carefully, because two parts of it are **not** what the section headings claim.

### Why "process isolation" still lists host processes

Our process is PID 1, and yet `ps` happily lists `/sbin/init`, `kthreadd`, and every kernel worker on the machine. The PID namespace is working fine. The problem is that `ps` does not ask the kernel for a process list, it reads `/proc`, and `/proc` is still the host's.

`CLONE_NEWNS` gave us a new mount namespace, but a new mount namespace is a *copy* of the host's mount table. Nothing remounted `/proc`. To see real process isolation we need to mount a fresh `procfs`, and that requires `CAP_SYS_ADMIN` in the namespace plus a propagation change so the mount does not leak back to the host:

```go
cmd := exec.Command("/bin/bash", "-c", `
    mount --make-rprivate /
    mount -t proc proc /proc
    echo "=== PROCESS ISOLATION ==="
    ps -eo pid,user,comm
    echo "=== NETWORK ISOLATION ==="
    ip -o link show
`)

cmd.SysProcAttr = &syscall.SysProcAttr{
    Cloneflags: syscall.CLONE_NEWPID | syscall.CLONE_NEWNS | syscall.CLONE_NEWNET |
        syscall.CLONE_NEWIPC | syscall.CLONE_NEWUTS,
}
```

Note that `CLONE_NEWUSER` is gone from the flag list. Run this one with `sudo`, and the process list finally matches the promise:

```
=== PROCESS ISOLATION ===
    PID USER     COMMAND
      1 root     bash
      5 root     ps
=== NETWORK ISOLATION ===
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
```

Two processes. That is the entire visible world.

The `mount --make-rprivate /` line is not optional. Without it, on a distribution where `/` is mounted shared (which is the default on systemd systems), the fresh `/proc` mount propagates back to the host and you have just broken process listing for the entire machine until you unmount it.

### Why listing `/` is denied

The second surprise is `ls: cannot open directory '/': Permission denied`, while `ls /etc` and `ls /tmp` work fine inside the same namespace. That combination is a strong hint that this is not ordinary DAC permissions. `/` is `drwxr-xr-x root root`, so our UID can read it.

The culprit is AppArmor. Since Ubuntu 23.10, creating an unprivileged user namespace transitions the process into a restricted profile:

```console
$ cat /proc/sys/kernel/apparmor_restrict_unprivileged_userns
1

$ unshare -U cat /proc/self/attr/current
unprivileged_userns (enforce)
```

And that profile, in `/etc/apparmor.d/unprivileged_userns`, contains this rule:

```
allow file rwlkm /**,
```

The AppArmor pattern `/**` matches a slash followed by one or more characters. It matches `/etc` and `/tmp/foo`, but it does **not** match the root directory `/` itself. There is no rule covering `/`, so access to it is denied. It is a one-character gap in a profile, and it is the entire explanation.

The same profile also carries `audit deny capability`, which is why the privileged version above had to drop `CLONE_NEWUSER`: inside an unprivileged user namespace on this host, AppArmor strips the `CAP_SYS_ADMIN` we would need to mount `/proc`, and the mount fails with a confusing `mount: /proc: cannot mount proc read-only`.

This is a good illustration of a point worth internalizing early: namespaces are only one of several layers deciding what a process can do. On a modern distribution, an LSM is quietly making decisions alongside them.

# Creating a container with runc

Let's compare our hand-rolled example with what `runc` actually does. For that, we will:

- create a container bundle;
- create a container based on it.

A container bundle (or OCI bundle) defines how a container and its configuration data are stored on a local filesystem so they can be consumed by a compliant runtime. It must contain at least:

- a `config.json` file
- a directory that serves as the container root filesystem

There are two common ways to build one.

**Method 1: using Docker export (requires Docker)**

```bash
mkdir -p /tmp/mycontainer/rootfs
cd /tmp/mycontainer

# Export a container filesystem using Docker
docker export $(docker create busybox) | tar -C rootfs -xf -

# Generate an OCI spec
runc spec
```

**Method 2: direct download (no Docker required)**

```bash
mkdir -p /tmp/mycontainer/rootfs
cd /tmp/mycontainer

# Download and extract an Alpine Linux root filesystem
wget https://dl-cdn.alpinelinux.org/alpine/v3.23/releases/x86_64/alpine-minirootfs-3.23.6-x86_64.tar.gz
tar -C rootfs -xzf alpine-minirootfs-3.23.6-x86_64.tar.gz

# Generate an OCI spec
runc spec
```

Either way, you end up with a `rootfs` directory holding the container's filesystem and a `config.json` describing how to run it:

```
/tmp/mycontainer
├── config.json
└── rootfs
    ├── bin
    ├── dev
    ├── etc
    └── ...
```

## Creating our first container with runc

Open `config.json` and set `process.terminal` to `false`, or use `jq`:

```bash
jq '.process.terminal = false' config.json > config.tmp && mv config.tmp config.json
```

This simplifies things, since terminal support requires setting up a console socket. Then run:

```bash
sudo runc create mycontainer
```

Running under `sudo` is necessary here because the default spec produced by `runc spec` does not include a user namespace. (`runc spec --rootless` generates one that does, but as we saw above, AppArmor will block it on recent Ubuntu.)

Check the container state with:

```bash
sudo runc state mycontainer
```

Which returns:

```json
{
  "ociVersion": "1.3.0",
  "id": "mycontainer",
  "pid": 34,
  "status": "created",
  "bundle": "/tmp/mycontainer",
  "rootfs": "/tmp/mycontainer/rootfs",
  "created": "2026-09-19T16:09:19.142537522Z",
  "owner": ""
}
```

At this point the `runc init` process is running and waiting for the `start` command. You can confirm that by peeking into its namespaces from the host:

```bash
PID=$(sudo runc state mycontainer | jq -r .pid)
sudo nsenter -t "$PID" -p -m -i -u -- ps -o pid,comm
```

Which returns:

```
PID   COMMAND
    1 runc:[2:INIT]
    7 ps
```

`runc:[2:INIT]` is the init process, sitting at PID 1 inside the container, halfway through its own setup. You can also see the FIFO it is blocked on:

```bash
sudo ls -l /proc/"$PID"/fd | grep exec.fifo
```

```
l--------- 1 root root 64 Sep 19 16:09 7 -> /run/runc/mycontainer/exec.fifo
```

## Inspecting the config.json file

At this stage, the most relevant fields in `config.json` are:

- `process.args`: what will be `exec()`'d when `runc start` unblocks init
- `process.env`: environment variables for the container process
- `process.capabilities`: the capability sets granted to the container process
- `linux.namespaces`: which namespaces are created
- `mounts`: bind mounts and virtual filesystems (`proc`, `sysfs`, and so on)

You can inspect them with:

```bash
jq '{
    process: {args: .process.args, env: .process.env},
    namespaces: [.linux.namespaces[].type],
    mounts: [.mounts[].destination]
}' config.json
```

Which gives:

```json
{
  "process": {
    "args": ["sh"],
    "env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "TERM=xterm"
    ]
  },
  "namespaces": ["pid", "network", "ipc", "uts", "mount", "cgroup"],
  "mounts": ["/proc", "/dev", "/dev/pts", "/dev/shm", "/dev/mqueue", "/sys", "/sys/fs/cgroup"]
}
```

Two details in that namespace list are easy to miss. There is a **cgroup** namespace, which most introductions to this topic never mention. And there is **no user namespace**, which is precisely why we needed `sudo`.

You may also expect to find `linux.cgroupsPath` here. It is absent from the default spec: when it is not set, `runc` derives the cgroup path from the container ID, which is why the container we just created lands in `/sys/fs/cgroup/mycontainer`.

## The runc init process

When you run `runc create mycontainer`, `runc` does not create the namespaces in the current process. Instead, the work is split across two processes:

1. **Parent process (`runc create`)**: prepares the container configuration, sets up the cgroup, and forks a child.

2. **Child process (`runc init`)**: the forked process re-executes the `runc` binary with the `init` argument:

   ```bash
   /proc/self/exe init
   ```

   Although `runc init` is a real command, it is not meant to be invoked by hand.

3. **Bootstrap with nsexec**: before the Go runtime starts, `runc` runs C code from `libcontainer/nsenter/nsexec.c` that creates the namespaces and sets up the initial environment.

### How nsexec works in runc

`runc` uses a C constructor (`__attribute__((constructor))`) to run C code before the Go runtime starts. The `nsenter` package is imported for its side effects in [`init.go`](https://github.com/opencontainers/runc/blob/main/init.go), so every time `runc init` is invoked, that code runs:

```go
// In runc's init.go
import _ "github.com/opencontainers/runc/libcontainer/nsenter"
```

Why go to this trouble? Because the Go runtime is multi-threaded, and several of the operations `runc` needs, including `setns()` for certain namespace types and `unshare(CLONE_NEWUSER)`, are only valid in a single-threaded process. By the time Go's runtime has started, that window is gone. The C constructor runs while the process is still single-threaded.

The code in `nsexec.c` runs in three stages, communicating over a socket pair:

1. **Stage 0 (parent)**: reads the configuration passed by the parent `runc` process over a pipe, and clones stage 1.
2. **Stage 1 (child)**: creates the user namespace if one is configured, waits for the parent to write the UID/GID mappings, then `unshare()`s the remaining namespaces and clones stage 2.
3. **Stage 2 (init)**: this is the process that ends up as PID 1 inside the new PID namespace. It hands control to the Go code, which continues the setup.

{% mermaid %}
flowchart TD
    A["runc create<br/>(parent)"] -->|"config over pipe"| B["Stage 0: PARENT<br/>reads config, clones stage 1"]
    B --> C["Stage 1: CHILD<br/>creates user namespace"]
    C -->|"asks parent to<br/>write uid_map/gid_map"| B
    C --> D["Stage 1: CHILD<br/>unshare() remaining namespaces"]
    D --> E["Stage 2: INIT<br/>lives in the new PID namespace"]
    E --> F["Go runtime starts<br/>continues setup"]
    F --> G["blocks on exec.fifo<br/>visible as runc:[2:INIT]"]

    classDef parent fill:#e8f5e9,stroke:#43a047,stroke-width:2px,color:#14321a
    classDef child fill:#e3f2fd,stroke:#1e88e5,stroke-width:2px,color:#0d2b45
    classDef init fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px,color:#311b3f

    class A,B parent
    class C,D child
    class E,F,G init
{% endmermaid %}

The stage numbering is not just internal trivia: it is the `2` you saw earlier in the process name `runc:[2:INIT]`.

The second clone is not an accident of implementation. A process that calls `unshare(CLONE_NEWPID)` does not itself move into the new PID namespace, only its children do. Forking again is the only way to get a process that actually *lives* in the new PID namespace.

```c
// Simplified from nsexec.c
clone(child_func, stack, CLONE_NEWNS | CLONE_NEWPID |
      CLONE_NEWNET | CLONE_NEWIPC | CLONE_NEWUTS |
      CLONE_NEWUSER, NULL);
```

### The exec FIFO synchronization

This is the mechanism that makes `created` a real, observable state rather than an instant on the way to `running`.

1. **FIFO creation**: during `runc create`, a named pipe is created at `/run/runc/<container-id>/exec.fifo`.

2. **Init process waits**: after `runc init` has set up the namespaces, it **blocks** opening this FIFO for reading:

   ```go
   // Simplified from runc source - init process waits here
   fd, err := os.OpenFile(execFifoPath, os.O_RDONLY, 0)
   if err != nil {
       return err
   }
   data := make([]byte, 1)
   fd.Read(data) // BLOCKS until runc start writes to the FIFO
   ```

   Opening a FIFO for reading blocks until a writer appears, which means init is parked before it has done anything irreversible.

3. **Create returns**: `runc create` exits successfully, but the container sits in the `created` state. All namespaces are set up; the main process has not started.

4. **Start triggers execution**: when you run `runc start`, it writes to the FIFO:

   ```go
   // Simplified - start process unblocks init
   fd, err := os.OpenFile(execFifoPath, os.O_WRONLY, 0)
   if err != nil {
       return err
   }
   fd.Write([]byte("0")) // Unblocks the waiting init process
   ```

5. **Init continues**: the `runc init` process unblocks and proceeds to `exec()` the container's main command.

For exact implementation details, see `libcontainer/process_linux.go` and `libcontainer/init_linux.go` in the `runc` source tree.

### What's next?

In the next part of this series, we will pick up exactly where init is blocked, and cover the rest of the story: the rootfs and `pivot_root`, the security context (capabilities, seccomp, LSM labels), cgroups, the OCI hooks, and what `runc start` actually triggers. See [Part 4 – runc create and start stages](/posts/container-runtimes-runc-create-start-stages/).

If you want a real-world example of hook behavior leading to impact, see [NVIDIA Container Escape CVE-2025-23266](/posts/nvidia-container-escape-cve-2025-23266/).
