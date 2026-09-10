# Docker Internal Architecture: How Docker Actually Works

## Table of Contents
1. [Introduction](#introduction)
2. [High-Level Architecture Overview](#high-level-architecture-overview)
3. [The Client-Server Model](#the-client-server-model)
4. [The Layered Component Stack](#the-layered-component-stack)
5. [dockerd — The Docker Daemon](#dockerd--the-docker-daemon)
6. [containerd — The Container Supervisor](#containerd--the-container-supervisor)
7. [containerd-shim](#containerd-shim)
8. [runc and the OCI Runtime Specification](#runc-and-the-oci-runtime-specification)
9. [Linux Namespaces in Depth](#linux-namespaces-in-depth)
10. [Control Groups (cgroups) in Depth](#control-groups-cgroups-in-depth)
11. [Union Filesystems and Storage Drivers](#union-filesystems-and-storage-drivers)
12. [Image Structure: Layers, Manifests, and Content Addressing](#image-structure-layers-manifests-and-content-addressing)
13. [Container Networking Internals](#container-networking-internals)
14. [The Full Lifecycle of `docker run`](#the-full-lifecycle-of-docker-run)
15. [Docker Desktop Architecture](#docker-desktop-architecture)
16. [Docker Desktop on macOS](#docker-desktop-on-macos)
17. [Docker Desktop on Windows](#docker-desktop-on-windows)
18. [Docker Desktop on Linux](#docker-desktop-on-linux)
19. [Docker Desktop Internal Components](#docker-desktop-internal-components)
20. [Security Model](#security-model)
21. [Common Questions (FAQ)](#common-questions-faq)
22. [Summary](#summary)

---

## Introduction

Knowing *how to use* Docker commands is one level of understanding. Knowing *what actually happens inside the machine* when you type `docker run nginx` is a much deeper level — and it's what separates a casual user from someone who can debug production issues, understand performance characteristics, and reason about security. This document goes underneath the Docker CLI and explains, layer by layer, exactly what components exist, how they talk to each other, and how a container is actually created by the Linux kernel. It then explains how **Docker Desktop** adapts all of this for macOS and Windows, where the required Linux kernel doesn't natively exist.

---

## High-Level Architecture Overview

Docker is not one single program — it's a **stack of cooperating components**, each with a specific responsibility, layered from the user-facing CLI down to the Linux kernel itself.

```
┌─────────────────────────────────────────────┐
│   docker CLI (client)                        │  <- what you type
└───────────────────┬───────────────────────────┘
                     │ REST API (over Unix socket / TCP)
┌────────────────────▼───────────────────────────┐
│   dockerd (Docker Daemon)                      │  <- manages images, volumes,
└────────────────────┬───────────────────────────┘     networks, builds, API
                     │ gRPC
┌────────────────────▼───────────────────────────┐
│   containerd                                    │  <- manages container lifecycle
└────────────────────┬───────────────────────────┘
                     │ spawns
┌────────────────────▼───────────────────────────┐
│   containerd-shim                               │  <- one per running container
└────────────────────┬───────────────────────────┘
                     │ invokes
┌────────────────────▼───────────────────────────┐
│   runc                                          │  <- actually creates the container
└────────────────────┬───────────────────────────┘
                     │ system calls
┌────────────────────▼───────────────────────────┐
│   Linux Kernel (namespaces + cgroups + more)    │  <- real isolation happens here
└──────────────────────────────────────────────────┘
```

Each layer exists for a reason, largely driven by Docker's history (see the companion document on Docker's evolution) — as Docker grew, functionality was progressively split into smaller, independently reusable, standardized components.

---

## The Client-Server Model

Docker follows a **client-server architecture**:

- The **Docker Client** is the `docker` command-line binary you interact with. It does not do any of the actual container work itself — it simply translates your commands into API calls.
- The **Docker Daemon** (`dockerd`) is a persistent background process running on the host. It receives those API calls and performs the actual work: pulling images, building images, creating networks, managing volumes, and delegating container execution further down the stack.

Communication between the client and daemon happens over the **Docker Engine API**, a REST API. By default, this happens over a local **Unix domain socket** (`/var/run/docker.sock` on Linux), though it can also be configured to listen over TCP (useful for remote Docker management, though this requires careful security configuration such as TLS, since anyone with access to the Docker socket effectively has root-level access to the host).

This client-server split is why the same `docker` CLI can control a **remote** Docker daemon (e.g., on a cloud server) just as easily as a local one — you simply point the client at a different daemon address.

---

## The Layered Component Stack

Historically, Docker was a single monolithic binary that did everything. Over time, Docker split its internals into distinct, independently useful layers — partly for better engineering, and partly to donate pieces to open, vendor-neutral foundations (see the Docker history document). The result is the four-layer execution stack: **dockerd → containerd → containerd-shim → runc**. Each is explained in detail below.

---

## dockerd — The Docker Daemon

`dockerd` is the core background service and is responsible for:
- Exposing the Docker Engine REST API that the CLI (and other tools) talk to
- **Image management** — pulling, building, tagging, and removing images
- **Volume management** — creating and managing persistent storage
- **Network management** — creating bridge, overlay, and other virtual networks
- **Build management** — handling `docker build` (using the **BuildKit** engine in modern Docker versions, which enables faster, cached, parallelized image builds)
- Delegating the actual running of containers to **containerd**

`dockerd` itself does not directly create containers — this responsibility was deliberately separated out into containerd so that container lifecycle management could be reused by other systems (like Kubernetes) independent of the full Docker daemon.

---

## containerd — The Container Supervisor

**containerd** is a separate daemon (donated by Docker to the CNCF, now a fully independent, widely-used project) responsible for the complete lifecycle of containers on a given host:

- Pulling and unpacking container images from a registry
- Managing container storage/snapshots
- Creating, starting, stopping, and deleting containers
- Managing low-level networking namespaces (though the higher-level network configuration is handled above it)
- Exposing a gRPC API used by `dockerd` (and by other systems, such as Kubernetes' `kubelet` via the **CRI**, the Container Runtime Interface)

Importantly, containerd is **not Docker-specific**. It is used directly by Kubernetes as one of its supported container runtimes, entirely without needing the rest of the Docker stack (`dockerd`) installed at all. This is a major reason why containerd was split out as its own project — to serve as shared, reusable infrastructure across the container ecosystem.

---

## containerd-shim

For every running container, containerd spawns a small, separate process called **containerd-shim** (specifically `containerd-shim-runc-v2` in modern setups). Its job is critical and easy to overlook:

- It becomes the direct **parent process** of the actual container process, **decoupling the container's lifetime from containerd's own lifetime**. This means containerd itself can be restarted or upgraded without killing all currently running containers — the shim keeps the container alive independently.
- It reports the container's exit status back to containerd.
- It keeps the container's standard input/output/error streams (stdin/stdout/stderr) open and available, even if no process is actively attached to them.
- It reaps "zombie" processes for the container (a Unix housekeeping responsibility that would otherwise fall to a missing init process).

This is why you'll often see a `containerd-shim` process for every single running container if you inspect process trees on a Docker host (e.g., using `ps aux` or `pstree`).

---

## runc and the OCI Runtime Specification

**runc** is the low-level component that does the actual, literal work of creating a container. It is a lightweight, portable command-line tool implementing the **OCI Runtime Specification** (a standard defined by the Open Container Initiative).

When invoked (by containerd-shim), runc:
1. Reads a **configuration file** (`config.json`) describing exactly how the container should be set up — which namespaces to create, what cgroup limits to apply, what the root filesystem is, what command to execute, what capabilities/security settings to apply, etc.
2. Makes the necessary **Linux system calls** to create new namespaces (`clone()`/`unshare()`), configure cgroups, set up the container's root filesystem (using `pivot_root` or similar), and apply security restrictions.
3. Executes the specified process inside this newly isolated environment.
4. **Exits** — runc's job is only to *create* the container and start the process; it does not stay running as a supervisor for the container's lifetime. This is exactly why containerd-shim exists: to sit above the (now-exited) runc and keep supervising the actual running process.

Because runc implements the open **OCI Runtime Spec**, other tools (e.g., **crun**, **gVisor's `runsc`**, or **Kata Containers**) can serve as drop-in alternatives to runc, offering different tradeoffs (e.g., gVisor provides stronger sandboxing/security by intercepting syscalls in userspace; Kata Containers runs each container inside a lightweight VM for stronger isolation).

---

## Linux Namespaces in Depth

Namespaces are the Linux kernel feature that gives each container the illusion of having its own isolated system, even though it's really sharing the host kernel with other containers. Each namespace type isolates a different resource:

| Namespace | Isolates | Effect Inside the Container |
|---|---|---|
| **PID** | Process IDs | The container sees its own process tree; its first process is PID 1, just like a normal Linux system |
| **NET** | Network stack | The container gets its own network interfaces, IP address, routing table, and port space |
| **MNT** | Mount points | The container sees its own isolated filesystem/directory structure |
| **UTS** | Hostname and domain name | The container can have its own hostname, separate from the host's |
| **IPC** | Inter-process communication | Prevents containers from being able to communicate via shared memory segments or semaphores with other containers |
| **USER** | User and group ID mappings | Allows a process to have root privileges (UID 0) *inside* the container while mapping to an unprivileged, non-root user *on the host* — a key security feature |
| **CGROUP** | Cgroup root directory view | Isolates the container's view of the cgroup hierarchy itself |

Namespaces answer the question: **"What can this process see and interact with?"** A process inside a container believes it's running on its own dedicated machine, when in reality the kernel is simply hiding everything outside its assigned namespaces.

---

## Control Groups (cgroups) in Depth

While namespaces control *visibility*, **cgroups** control *resource consumption*. Cgroups allow the kernel to group a set of processes together and apply limits, prioritization, and accounting to that group as a whole. Docker uses cgroups to enforce settings like:

- **CPU limits** — e.g., restricting a container to a maximum of 1.5 CPU cores (`--cpus="1.5"`)
- **Memory limits** — e.g., restricting a container to a maximum of 512MB of RAM (`--memory="512m"`), after which the kernel's Out-Of-Memory (OOM) killer may terminate a process inside the container
- **Block I/O limits** — restricting disk read/write throughput
- **Device access** — restricting which host devices a container is allowed to access

There are two major versions of the cgroups system in the Linux kernel:
- **cgroups v1** — the original implementation, with a separate hierarchy for each resource type (CPU, memory, etc.)
- **cgroups v2** — a redesigned, unified hierarchy that is simpler and more consistent; this is the modern default on most current Linux distributions and is what current versions of Docker prefer to use when available

Without cgroups, a single misbehaving container (e.g., one with a memory leak or infinite loop) could consume all of a host's resources and starve every other container and process on that machine. Cgroups exist specifically to prevent this ("noisy neighbor" problem).

---

## Union Filesystems and Storage Drivers

Docker images are built from a series of **read-only layers**, stacked on top of each other. When you run a container, Docker adds one additional **thin, writable layer** on top of those read-only image layers — this is where any changes the container makes to its filesystem at runtime are stored.

This layering is made possible by a **union filesystem** (also called a "storage driver" in Docker terminology). The most common modern storage driver is **overlay2**, based on the Linux kernel's OverlayFS.

How it works conceptually:
- Each image layer is stored as a separate directory on the host's disk.
- OverlayFS presents a single merged, unified view of all these layers stacked together, as if they were one filesystem.
- When a container modifies a file that exists in a read-only lower layer, the storage driver uses a technique called **copy-on-write (CoW)**: the file is copied up into the container's writable top layer first, and the modification happens there — leaving the original, shared, read-only layer completely untouched.

Benefits of this design:
- **Storage efficiency** — Multiple containers based on the same image (or images sharing common base layers) share those layers on disk instead of duplicating them.
- **Fast container startup** — Since containers don't need to copy the entire filesystem before starting, only reference existing layers plus a new empty writable layer, container creation is nearly instantaneous.
- **Disposability** — Deleting a container simply discards its thin writable layer; the underlying image layers remain untouched and reusable.

Other storage drivers exist (e.g., `aufs`, `devicemapper`, `btrfs`, `zfs`), but `overlay2` is the modern default and recommended choice for most Linux distributions today.

---

## Image Structure: Layers, Manifests, and Content Addressing

A Docker image is not a single file — it's a structured collection of components:

- **Layers** — Each instruction in a Dockerfile that modifies the filesystem (e.g., `RUN`, `COPY`, `ADD`) typically produces a new layer. Layers are stored as compressed tarballs (`.tar.gz`).
- **Manifest** — A JSON document describing the image: which layers make it up, in what order, and metadata like architecture (e.g., `amd64`, `arm64`) and OS.
- **Image ID / Digest** — Each layer and the overall image are identified by a **SHA-256 content hash**. This means images are **content-addressable**: the same content will always produce the same hash, and any change to the content produces a different hash. This allows Docker to detect whether a layer already exists locally (and skip re-downloading it) and ensures the integrity of pulled images (a corrupted or tampered layer would produce a mismatched hash).
- **Config file** — Contains metadata about how a container should be run from this image: the default command, environment variables, exposed ports, working directory, etc.

When you run `docker pull`, Docker downloads the manifest first, checks which layers you don't already have locally (by comparing hashes), and only downloads the missing ones — this is why pulling a new image that shares a base layer with an image you already have is often much faster than a full download.

---

## Container Networking Internals

By default, Docker creates a virtual network called the **bridge network** on the host. Here's what actually happens:

1. Docker creates a virtual Ethernet bridge on the host (commonly named `docker0`), which acts like a virtual network switch.
2. For each container, Docker creates a **veth (virtual Ethernet) pair** — essentially a virtual network cable with two ends. One end is placed inside the container's network namespace (appearing as `eth0` inside the container); the other end is attached to the `docker0` bridge on the host.
3. Each container is assigned its own private IP address (typically in a range like `172.17.0.0/16` by default).
4. Containers connected to the same bridge network can communicate with each other directly via their private IPs.
5. For a container to be reachable from outside the host (e.g., via `-p 8080:80`), Docker configures **NAT (Network Address Translation) rules using iptables** on the host, forwarding traffic arriving on the host's port 8080 to port 80 inside the specific container.
6. For containers to reach the outside internet, traffic from the container is also NAT'd through the host's own network interface.

Docker supports multiple network drivers beyond the default bridge:
- **host** — the container shares the host's network namespace directly (no isolation, but no NAT overhead either)
- **overlay** — used in multi-host setups (e.g., Docker Swarm or Kubernetes) to let containers on *different physical hosts* communicate as if on the same network, typically using VXLAN tunneling
- **macvlan** — assigns a container its own MAC address, making it appear as a physical device directly on the network
- **none** — disables networking entirely for the container

---

## The Full Lifecycle of `docker run`

Putting everything above together, here's what actually happens, step by step, when you run:

```bash
docker run -d -p 8080:80 nginx
```

1. The **Docker CLI** sends an HTTP request to the **Docker daemon (`dockerd`)** via the REST API (over the Unix socket).
2. `dockerd` checks if the `nginx` image exists locally. If not, it pulls the manifest and layers from the registry (Docker Hub by default), verifying content hashes as it goes.
3. `dockerd` passes the request to **containerd** via gRPC, instructing it to create a container from this image.
4. containerd prepares the container's filesystem using the storage driver (e.g., `overlay2`), stacking the pulled image layers and adding a new writable layer on top.
5. containerd spawns a new **containerd-shim** process for this container.
6. The shim invokes **runc**, passing it a generated OCI-compliant configuration (`config.json`).
7. **runc** makes the necessary Linux system calls: creating new namespaces (PID, NET, MNT, UTS, IPC), setting up cgroup limits, mounting the merged filesystem as the container's root, and configuring networking (attaching a veth pair to the `docker0` bridge).
8. runc starts the actual `nginx` process **inside** this newly isolated environment, then exits, leaving supervision to the shim.
9. `dockerd` configures the requested port mapping (`-p 8080:80`) by creating **iptables NAT rules** so that traffic to the host's port 8080 is forwarded into the container's port 80.
10. The container is now running, and `docker ps` will show it as `Up`.

Each of these ten steps corresponds directly to a component discussed earlier in this document — which is why understanding the architecture makes debugging (e.g., "why won't my container start," "why can't I reach this port") vastly easier.

---

## Docker Desktop Architecture

**Docker Desktop** is a separate application (distinct from the open-source Docker Engine itself) built by Docker, Inc., designed to provide an easy, integrated Docker experience on **macOS**, **Windows**, and (more recently) **Linux**. It bundles the Docker Engine, CLI, Docker Compose, Kubernetes (optional, built-in single-node cluster), a graphical user interface, and various developer conveniences (like file sharing and network integration with the host OS) into one installable package.

The fundamental challenge Docker Desktop solves: **Docker containers require a Linux kernel to run natively** (since namespaces and cgroups are Linux kernel features), but **macOS and Windows do not have a Linux kernel**. Docker Desktop's core job, therefore, is to transparently run a **lightweight Linux virtual machine** behind the scenes, and make interacting with containers inside that VM feel completely native and seamless to the user — so it *feels* like you're running Docker directly, even though there's a hidden VM layer underneath.

---

## Docker Desktop on macOS

On macOS, Docker Desktop runs a small, purpose-built Linux virtual machine using **Apple's native Virtualization.framework** (in current versions; older versions used HyperKit, a lightweight hypervisor toolkit derived from `xhyve`/`bhyve`). This Linux VM runs a minimal Linux distribution (historically based on **LinuxKit**, a toolkit Docker built for constructing minimal, purpose-specific Linux systems) whose entire job is to host the real `dockerd`, `containerd`, and `runc` stack described earlier in this document.

Key mechanisms Docker Desktop provides on macOS:
- **File sharing** — Since your project files live on macOS but containers run inside the Linux VM, Docker Desktop implements a file-sharing layer (historically using `osxfs`, more recently an improved system called **VirtioFS** for significantly better performance) so that a bind mount like `-v ./app:/app` transparently shares files between your Mac's filesystem and the container running inside the VM.
- **Networking** — Docker Desktop sets up networking so that `localhost:8080` on your Mac correctly reaches a container's port 8080 running inside the internal Linux VM, handling the necessary routing/proxying invisibly.
- **Resource allocation** — Through the Docker Desktop GUI, you can configure how much CPU, memory, and disk the underlying Linux VM is allowed to use.

---

## Docker Desktop on Windows

On Windows, Docker Desktop's architecture depends on the backend mode selected:

### WSL 2 backend (modern default)
Docker Desktop integrates with **WSL 2 (Windows Subsystem for Linux, version 2)**, which itself runs a real, lightweight Linux kernel inside a highly optimized, Microsoft-built virtual machine. Docker Desktop runs its Docker Engine components (`dockerd`, `containerd`, etc.) inside this WSL 2 environment. This approach offers:
- Better performance than the older Hyper-V approach (especially for file I/O)
- Tighter integration — you can access your Docker containers directly from within any WSL 2 Linux distribution's terminal
- Dynamic resource allocation — WSL 2 can grow and shrink its memory usage based on actual demand, rather than reserving a fixed amount upfront

### Hyper-V backend (legacy)
Before WSL 2 was mature, Docker Desktop used **Hyper-V** (Windows' native hypervisor) to run a dedicated Linux VM (again, based on LinuxKit), similar conceptually to the macOS approach. This mode still exists as an option but is largely superseded by the WSL 2 backend for most users today.

### Windows containers (a different mode entirely)
Docker Desktop on Windows can also run **native Windows containers** — these use the Windows kernel's own container support directly (no Linux VM involved) and run Windows-based images. This is a fundamentally different mode from running Linux containers, used specifically for legacy Windows applications, and requires switching Docker Desktop into "Windows containers" mode.

---

## Docker Desktop on Linux

On Linux, a Linux kernel is already natively present, so in principle a VM shouldn't be strictly necessary. However, **Docker Desktop for Linux** still runs its engine components inside a lightweight, managed VM (rather than using the host's Docker Engine directly), primarily for **consistency and isolation** — ensuring the exact same experience and version across macOS, Windows, and Linux, and isolating Docker Desktop's environment from the host's own configuration. This is different from simply installing the standalone open-source **Docker Engine** package directly on Linux (via `apt`/`dnf`/etc.), which runs natively on the host with no VM at all and is the more traditional, lightweight way to run Docker on Linux servers.

---

## Docker Desktop Internal Components

Beyond the core Docker Engine stack, Docker Desktop bundles several additional pieces:

- **Docker Desktop GUI** — a graphical application for managing containers, images, volumes, and settings visually, without needing the CLI.
- **Kubernetes (optional)** — Docker Desktop can spin up a single-node Kubernetes cluster locally with one click, useful for local development and testing of Kubernetes manifests before deploying to a real cluster.
- **Docker Compose** — bundled directly, so multi-container applications can be started with `docker compose up` out of the box.
- **Extensions** — Docker Desktop supports third-party and first-party "Extensions" that add extra functionality (e.g., visualizing logs, scanning images for vulnerabilities) directly within the GUI.
- **Resource management UI** — a settings panel for controlling the CPU, memory, disk, and swap allocated to the underlying VM (on macOS/Windows).
- **Dev Environments / Dashboard features** — additional productivity tooling for managing running containers, inspecting logs, and executing commands inside containers, all from the GUI.

---

## Security Model

A few important internal security considerations worth understanding:

- **Root inside, unprivileged outside** — By default, the main process inside a container may run as root (UID 0) *within* its own user namespace, but with **user namespace remapping** enabled, that root user is mapped to an unprivileged UID on the actual host — meaning even if a process somehow escaped basic container isolation, it wouldn't automatically have root privileges on the host itself. (Note: user namespace remapping is not always enabled by default and is a recommended hardening step.)
- **Capabilities** — Linux divides root's traditionally all-powerful privileges into fine-grained units called **capabilities** (e.g., `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`). By default, Docker containers run with a restricted subset of capabilities rather than full root capabilities, further limiting what a compromised container process could do.
- **Seccomp** — Docker applies a default **seccomp** (secure computing mode) profile that restricts which system calls a container's processes are allowed to make, blocking many dangerous or unnecessary syscalls by default.
- **Shared kernel risk** — Because all containers on a host share the same kernel, a critical kernel-level vulnerability could theoretically allow a process to escape container isolation entirely — this is the fundamental tradeoff containers make against the stronger isolation of full VMs (where each VM has its own separate kernel).
- **AppArmor / SELinux** — On supported Linux distributions, Docker can also apply mandatory access control profiles (AppArmor or SELinux) to further restrict container behavior beyond the default capability and seccomp restrictions.

---

## Common Questions (FAQ)

**Q: Why does Docker need containerd *and* runc — why not just one component?**
Separation of concerns. `runc` is a minimal, singular-purpose tool that just creates one container and exits. `containerd` handles everything around the full lifecycle (pulling images, managing storage, supervising many running containers, exposing an API). This split allows each piece to be simpler, more reliable, and independently reusable — for instance, Kubernetes uses containerd directly without needing Docker's full daemon at all.

**Q: If runc exits after starting the container, what keeps the container running?**
The container process itself keeps running as a normal Linux process (just one that's been placed into isolated namespaces with cgroup limits applied). The `containerd-shim` process sits above it as its parent, monitoring it and reporting its status — but the shim doesn't need to do any ongoing "work" to keep the container alive; the container process runs independently, just like any other Linux process, until it exits or is killed.

**Q: Does every container really have its own IP address?**
Yes, by default, each container attached to a bridge network gets its own private IP address, distinct from the host's IP and from other containers.

**Q: Is Docker Desktop the same thing as Docker Engine?**
No. **Docker Engine** is the core, free, open-source runtime (`dockerd` + containerd + runc + the CLI) that can be installed directly on Linux. **Docker Desktop** is a separate, packaged application (with its own licensing terms for larger businesses) that bundles Docker Engine, a management GUI, and (on macOS/Windows) a hidden Linux VM to run it, all set up for you automatically.

**Q: Why is Docker Desktop sometimes slower on Mac/Windows than native Linux?**
Because containers on macOS/Windows run inside a virtual machine, there is inherent overhead — particularly around file I/O when bind-mounting files from the host filesystem into a container, since that traffic must cross the VM boundary. This is precisely why technologies like VirtioFS (on macOS) and WSL 2 (on Windows) exist: to minimize that overhead as much as possible.

**Q: What happens to my containers if I quit Docker Desktop?**
Since containers run inside the internal Linux VM that Docker Desktop manages, quitting Docker Desktop stops that VM, which stops all running containers along with it. They are not deleted (unless explicitly removed) — they simply won't be running until Docker Desktop (and its VM) is started again.

---

## Summary

Underneath every `docker` command lies a carefully layered stack: the **CLI** talks to **dockerd**, which delegates container lifecycle management to **containerd**, which spawns a **containerd-shim** for each running container, which in turn invokes **runc** to perform the actual low-level work of creating Linux **namespaces** (for isolation) and configuring **cgroups** (for resource limits) via direct kernel system calls. Images themselves are built from content-addressed, stackable **layers**, combined at runtime using a **union filesystem** like `overlay2`. Networking between containers and the outside world is achieved through virtual Ethernet pairs, a bridge interface, and iptables NAT rules.

**Docker Desktop** exists to bring this fundamentally Linux-only technology to macOS and Windows by transparently running a minimal Linux virtual machine behind a polished, native-feeling interface — using Apple's Virtualization.framework or HyperKit on macOS, and WSL 2 or Hyper-V on Windows — while bundling Docker Engine, Compose, an optional local Kubernetes cluster, and a graphical management interface into a single, easy-to-install package.

Understanding this internal architecture — not just the commands, but the actual mechanics happening at the kernel level — is what enables confident troubleshooting, informed performance tuning, and a genuine understanding of what a "container" really is.
