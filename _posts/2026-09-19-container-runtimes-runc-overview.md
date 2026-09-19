---
layout: post
title: Container runtimes - runc - Overview
tags: container-runtimes docker
comments: true
description: Part 2 of the container runtimes series. Introduces runc and breaks down the init, create, and start lifecycle stages.
series: Container runtimes
series_order: 2
series_title: runc overview
---

{% include series-nav.html %}

# What is runc?

[`runc`](https://github.com/opencontainers/runc) is an OCI (Open Container Initiative) compliant CLI to create and run containers on Linux. It was [introduced in 2015](https://www.docker.com/blog/runc/) by the Docker community and donated to the Open Container Project (later renamed the Open Container Initiative), with the goal of creating a lightweight, portable container runtime built from the low-level code Docker used to interact with Linux container primitives.

Its most common commands are:

- `runc spec`: creates a `config.json` file (the container spec)
- `runc create`: creates a container
- `runc start`: starts a previously created container
- `runc delete`: removes a stopped container
- `runc run`: creates and starts a container in one step

These commands orchestrate the overall container lifecycle, which we will break down in the next sections.

# runc stages

The OCI Runtime Specification defines the container lifecycle in terms of three operations: **create**, **start**, and **delete**. `runc` implements them, and `runc run` is simply a convenience wrapper around create plus start.

There is, however, a fourth piece that is not a CLI command at all: **`runc init`**. This is the process that actually sets up the isolation, and it is where most of the important work happens. It is easy to misread `runc` as running "init, then create, then start" in sequence. It does not. `runc init` is a child process forked *during* create, and it sits blocked partway through its own setup until `runc start` releases it:

{% mermaid %}
flowchart TD
    A["runc create"] --> B["fork + exec<br/>/proc/self/exe init"]
    B --> C["runc init<br/>(namespaces, rootfs, security context)"]
    C --> D["blocks on exec.fifo"]
    D -. "container state: created" .-> E["runc start"]
    E --> F["writes to exec.fifo"]
    F --> G["runc init unblocks<br/>and calls execve()"]
    G --> H["container state: running"]
    H --> I["runc delete<br/>(teardown)"]

    classDef create fill:#e8f5e9,stroke:#43a047,stroke-width:2px,color:#14321a
    classDef init fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px,color:#311b3f
    classDef start fill:#fff3e0,stroke:#fb8c00,stroke-width:2px,color:#3e2600
    classDef teardown fill:#eceff1,stroke:#607d8b,stroke-width:2px,color:#22313a

    class A,B create
    class C,D init
    class E,F,G,H start
    class I teardown
{% endmermaid %}

With that ordering in mind, the rest of this post walks through each phase: what `runc init` does, what create adds around it, and what start triggers.

## The init phase

The init phase is where `runc` prepares the container environment. This is the phase that requires the most attention from a security standpoint, since it establishes namespaces and security contexts before the container process starts.

{% mermaid %}
graph TD
    subgraph INIT["INIT PHASE"]
        direction LR
        B1["<b>Namespace Creation</b><br/>• PID • Mount • Network<br/>• IPC • UTS • User • Cgroup"]
        B1 --> B2["<b>Filesystem Setup</b><br/>• Rootfs preparation<br/>• Bind mounts<br/>• Device nodes<br/>• /proc, /sys"]
        B2 --> B3["<b>Security Context</b><br/>• SELinux/AppArmor<br/>• Capabilities<br/>• Seccomp profiles<br/>• User mappings"]
    end

    classDef initStage fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px,color:#311b3f

    class B1,B2,B3 initStage
    style INIT fill:#faf4fc,stroke:#ce93d8,stroke-width:1px,color:#311b3f
{% endmermaid %}

In this phase, the following steps are performed:

1. **Namespace creation**: `runc` creates new namespaces based on the container specification, using `clone()`, `unshare()`, and `setns()` with the appropriate flags (`CLONE_NEWPID`, `CLONE_NEWNS`, and so on).

2. **Filesystem setup**: the container's root filesystem is prepared, including:
   - setting up the rootfs from the container image
   - creating bind mounts for volumes
   - setting up device nodes
   - configuring `/proc`, `/sys`, and other special filesystems

3. **Security context**: security-related configurations are applied:
   - SELinux/AppArmor labels
   - capabilities are dropped or added
   - seccomp profiles are loaded
   - user namespace mappings are established

Cgroups are a deliberate exception here. They are **not** configured by `runc init`. The parent `runc` process creates the cgroup and moves the init process into it, because a process confined by a restrictive cgroup cannot reliably set up its own limits. We will come back to this in Part 4.

## The create phase

The create phase is where `runc` creates the container but does not start the user-specified process yet. At this point, the container exists in a `created` state: all namespaces and isolation are set up, but the main application has not started.

{% mermaid %}
graph TD
    subgraph CREATE["CREATE PHASE"]
        direction LR
        C1["<b>Fork Init Process</b><br/>• Process prepared<br/>• Not yet exec'd"]
        C1 --> C2["<b>Cgroups Applied</b><br/>• Created by parent runc<br/>• Init process moved in"]
        C2 --> C3["<b>Execute Hooks</b><br/>• createRuntime<br/>• createContainer"]
        C3 --> C4["<b>Container State: CREATED</b><br/>• All isolation ready<br/>• Main process not started<br/>• State persisted to disk"]
    end

    classDef createStage fill:#e8f5e9,stroke:#43a047,stroke-width:2px,color:#14321a

    class C1,C2,C3,C4 createStage
    style CREATE fill:#f4faf5,stroke:#a5d6a7,stroke-width:1px,color:#14321a
{% endmermaid %}

In this phase, the following steps are performed:

1. **Process preparation**: the init process is forked and configured, but the container command is not yet `exec`'d
2. **Cgroup setup**: control groups are created and applied to limit and isolate resource usage (CPU, memory, I/O)
3. **Runtime hooks**: OCI lifecycle hooks (`createRuntime` and `createContainer`) run during this phase
4. **State transition**: the container transitions to the `created` state
5. **State persistence**: container state is written to disk for management tools

The create phase matters because it gives tooling a window for inspection and additional setup before the container actually starts running. It is also the window in which the `createContainer` hook runs, which is exactly the mechanism abused by [CVE-2025-23266](/posts/nvidia-container-escape-cve-2025-23266/).

## The start phase

The start phase is where the container begins executing the user-specified command or entrypoint.

{% mermaid %}
graph TD
    subgraph START["START PHASE"]
        direction LR
        D1["<b>Signal Init Process</b><br/>• Write to exec.fifo<br/>• Unblocks runc init"]
        D1 --> D2["<b>execve()</b><br/>• Replace init with<br/>the container process"]
        D2 --> D3["<b>Execute poststart Hooks</b><br/>• Application-specific setup<br/>• Health checks"]
        D3 --> D4["<b>Container State: RUNNING</b><br/>• Main application executing<br/>• Monitoring begins"]
    end

    classDef startStage fill:#fff3e0,stroke:#fb8c00,stroke-width:2px,color:#3e2600

    class D1,D2,D3,D4 startStage
    style START fill:#fffaf3,stroke:#ffcc80,stroke-width:1px,color:#3e2600
{% endmermaid %}

In this phase, the following steps are performed:

1. **Unblocking init**: `runc start` writes to the container's `exec.fifo`, releasing the blocked `runc init` process
2. **Process execution**: the init process performs an `execve()` system call to replace itself with the container's main process
3. **Hooks**: the `startContainer` hook runs inside the container namespace just before `execve()`, and `poststart` hooks run on the host afterwards
4. **State transition**: the container state changes from `created` to `running`

### What's next?

In the next part of this series, we will explore the `runc` init phase and how each of these steps is implemented in practice: [Part 3 – runc init stage](/posts/container-runtimes-runc-init-stage/).
