---
layout: post
title: Container runtimes series
tags: container-runtimes docker
comments: true
description: Part 1 of the container runtimes series. Covers what containers are, the runtime stack, and OCI vs CRI fundamentals.
series: Container runtimes
series_order: 1
series_title: container runtimes fundamentals
---

{% include series-nav.html %}

# Introduction

Container runtimes are usually covered in a single chapter in Docker or Kubernetes books. Yet they are at the core of both, implementing the main functionality behind what we understand as "containerization" today. This series aim to provide them a more in-depth coverage.

In this series, I will explain what container runtimes are, how they evolved, and why each runtime exists.

In this first post, we will establish the core concepts and the main standards (OCI and CRI) used across the ecosystem.

# Containers

Before we attempt to define what a container runtime is, we need to define what a container is, so some backstory is important.

In 1979, `chroot` was introduced with the purpose of filesystem isolation. In 2000, FreeBSD Jails extended this model by adding process and network isolation. Later, in 2006, Google engineers built "process containers" (later renamed cgroups, and merged into the mainline kernel in 2008) to limit and isolate resource usage. Then, in 2008, LXC combined namespaces and cgroups into the first practical Linux container manager. This was the first time you could run something very close to what we call a container today.

In 2013, Docker wrapped LXC with a developer-friendly CLI and a layered image system.

From a high-level standpoint, a container is a lightweight, portable unit of software that packages an application and everything it needs to run: code, libraries, dependencies, and configuration. The point of that packaging is that the application runs consistently across different environments.

From a technical standpoint, a container is a process (or group of processes) on a host operating system that runs in isolation from other processes, enforced by kernel features such as:

- **Namespaces**: provide isolation of process IDs, filesystems, networks, and users.
- **cgroups** (control groups): manage and limit resource usage (CPU, memory, I/O).
- **Other security controls**: restrict what privileged operations the process can perform (capabilities, seccomp, and LSMs such as SELinux or AppArmor).

Note that none of these are "container" features. The kernel has no concept of a container. A container is a process that a runtime has placed inside a particular combination of these primitives, which is exactly why the runtime matters so much.

# Container runtimes

A container runtime is the low-level software responsible for creating and running containers on a host system. For example, when you run:

```console
$ docker run hello-world
```

a number of components are involved behind the scenes:

- **Docker CLI (`docker`)** – parses your command and sends it to the Docker daemon.
- **Docker daemon (`dockerd`)** – the high-level manager that handles images, networks, volumes, and container lifecycle.
- **containerd** – the core container lifecycle manager used by Docker. It manages pulling images, preparing the root filesystem, and handing off execution to a runtime.
- **containerd-shim** – a lightweight process that sits between containerd and the low-level runtime, keeping containers alive even if containerd restarts or dies.
- **Low-level runtime (`runc`)** – implements the OCI Runtime Specification. It sets up namespaces, cgroups, mounts, and capabilities, and finally executes the container process (in this case, `/hello`).

{% mermaid %}
flowchart TD
    A["docker (CLI)"] --> B["dockerd (daemon)"]
    B --> C["containerd"]
    C --> D["containerd-shim"]
    D --> E["runc (low-level runtime)"]
    E --> F["Linux kernel<br/>(namespaces, cgroups, capabilities)"]

    classDef client fill:#e3f2fd,stroke:#1e88e5,stroke-width:2px,color:#0d2b45
    classDef manager fill:#e8f5e9,stroke:#43a047,stroke-width:2px,color:#14321a
    classDef shim fill:#fff3e0,stroke:#fb8c00,stroke-width:2px,color:#3e2600
    classDef runtime fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px,color:#311b3f
    classDef kernel fill:#eceff1,stroke:#607d8b,stroke-width:2px,color:#22313a

    class A client
    class B,C manager
    class D shim
    class E runtime
    class F kernel
{% endmermaid %}

When Docker came about in 2013, it was marketed as "shipping apps as containers". But Docker itself did too many things at once:

- Image building and distribution
- Networking
- Orchestration APIs
- The low-level task of actually running containers

To make  it modular, the community split the runtime into its own component, responsible only for creating and running containers.

# OCI and CRI

The Container Runtime Interface (CRI) is a Kubernetes-specific gRPC API defined by the Kubernetes project. It sits between the kubelet and container runtimes, and it defines RPCs such as `RunPodSandbox`, `CreateContainer`, `StartContainer`, and `StopContainer`. Examples of runtimes that implement CRI directly are containerd (through its CRI plugin) and CRI-O.

On the other hand, `runc` does not implement CRI. Instead, `runc` implements the OCI Runtime Specification, a lower-level spec. OCI defines how to run a container from an OCI bundle (`config.json`, `rootfs`, and so on) and provides a standard CLI (`runc run`, `runc create`, and others).

In short:

| Interface | Used by | Scope |
|---|---|---|
| CRI | kubelet | Kubernetes runtime API |
| OCI Runtime Spec | low-level runtimes (`runc`, `crun`, `youki`) | How containers are executed on a host |

The two are complementary rather than competing: a CRI implementation like containerd is the thing that ends up calling an OCI runtime like `runc`.

The runtimes I plan to cover in this series are:

- `runc`: the reference OCI runtime, used by Docker and containerd
- NVIDIA Container Runtime (see also my write-ups on [CVE-2025-23266](/posts/nvidia-container-escape-cve-2025-23266/) and its [follow-up](/posts/nvidia-container-escape-cve-2025-23266-part-2/))
- Kata Containers
- gVisor
- Docker Desktop on Windows
- apple/container

In the next post, we will focus on `runc` and walk through its lifecycle: [Part 2 – runc overview](/posts/container-runtimes-runc-overview/).

## References

- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)
- [OCI runtime implementations](https://github.com/opencontainers/runtime-spec/blob/main/implementations.md)
- [Kubernetes CRI API](https://github.com/kubernetes/cri-api)
- [containerd](https://github.com/containerd/containerd)
- [CRI-O](https://github.com/cri-o/cri-o)
