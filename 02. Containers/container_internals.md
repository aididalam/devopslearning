# Container Internals — Notes

## 1. The Core Mental Model

Do not treat **image**, **container**, and **runtime** as the same thing.

```text
Image       = packaged filesystem + metadata
Container   = isolated execution environment for processes
Runtime     = software that creates and starts containers
```

Layers:

```text
Container Image
    ↓ unpack / mount layers
OCI Bundle
    ├── config.json
    └── rootfs/
    ↓
OCI Runtime
    ↓
Container
    ↓
One or more processes running in isolation
```

Most important sentence:

> A container image is a packaging and distribution format; a container is a runtime execution environment.

---

# 2. Container vs Image

| Aspect       | Container Image                                   | Container                                               |
| ------------ | ------------------------------------------------- | ------------------------------------------------------- |
| Nature       | Static artifact                                   | Runtime execution environment                           |
| Mutability   | Immutable (read-only layers)                      | Has a writable layer on top                             |
| Contents     | Filesystem + metadata                             | One or more running processes                           |
| Purpose      | Packaging and distribution                        | Execution of a workload                                 |
| Lifecycle    | Built once, stored in a registry                  | Created from an image, runs, then is stopped or deleted |
| Persistence  | Changes require a new image build                 | Changes in the writable layer are lost on deletion      |
| Analogy      | Class definition / blueprint                      | Instance of that class / running program                |

---

## Container Image

A container image is a static artifact.

It usually contains:

| Component | Description |
| ----------------------- | ---------------------------------------------------- |
| Root filesystem content | Files and directories forming the container's `/`    |
| Metadata                | Image configuration, labels, and annotations         |
| Default command         | The default `CMD` to execute at runtime              |
| Environment variables   | Pre-set env vars declared via `ENV`                  |
| Working directory       | Default `WORKDIR` for processes                      |
| Exposed ports           | Ports declared with `EXPOSE`                         |
| Entrypoint              | Main executable defined via `ENTRYPOINT`             |
| Layer information       | References to the image's filesystem layers          |

An image is usually immutable and layered.

Example:

| Order      | Layer                    |
| ---------- | ------------------------ |
| 4 (top)    | Application Layer        |
| 3          | Dependency Layer         |
| 2          | Package Installation Layer |
| 1 (bottom) | Base OS Layer            |

## Container

A container is the runtime environment created from a filesystem and configuration.

It is not fundamentally the image itself.

A container may contain one or more processes.

| Level         | Definition                                                               |
| ------------- | ------------------------------------------------------------------------ |
| Beginner      | Isolated Linux process                                                   |
| More accurate | Standardized isolated execution environment for one or more processes    |

---

# 3. Running a Container Does Not Fundamentally Require an Image

At the low-level runtime layer, an OCI runtime such as `runc` does not need a Docker image directly.

It needs an **OCI bundle**.

```text
bundle/
├── config.json
└── rootfs/
```

## `config.json`

This file tells the runtime how the container should run.

It can define:

```text
process.args
root filesystem path
hostname
environment variables
mounts
namespaces
capabilities
resource limits
seccomp profile
```

Example concept:

```json
{
  "root": {
    "path": "rootfs"
  },
  "process": {
    "args": ["/app"]
  },
  "hostname": "my-container"
}
```

## `rootfs`

`rootfs` means **root filesystem**.

Inside the container, it appears as:

```text
/
```

So this host-side structure:

```text
rootfs/
└── app
```

appears inside the container as:

```text
/app
```

Key answer:

> An OCI runtime can run a container from a root filesystem and `config.json`. A Docker image is not strictly required at that stage.

---

# 4. Why Images Still Matter

Images are not mandatory for the lowest-level runtime, but they are essential in real-world systems.

Images solve:

```text
Packaging
Distribution
Reproducibility
Caching
Layer reuse
Versioning
Registry storage
Deployment consistency
```

Without images, every machine would need to manually prepare the correct root filesystem and runtime configuration.

Images make containers practical at scale.

---

# 5. Image Layers

Container images are usually built from immutable layers.

Each layer represents a filesystem change.

Example:

```text
Layer 4: Application files
Layer 3: Installed dependencies
Layer 2: Package index update
Layer 1: Base OS filesystem
```

Layers are often content-addressed.

That means they are identified by hashes such as:

```text
sha256:...
```

This allows:

```text
Deduplication
Caching
Integrity checking
Efficient storage
Efficient transfer
```

Useful phrase:

> Image layers are immutable filesystem diffs that can be reused across images.

---

# 6. Writable Layer and OverlayFS

When a container runs from an image, the image layers are normally read-only.

A writable layer is placed on top:

```text
Writable Container Layer
Read-only Image Layer 3
Read-only Image Layer 2
Read-only Image Layer 1
```

The container sees one unified filesystem.

Internally, this is commonly implemented using:

```text
OverlayFS
Union mount
Copy-on-write
```

## Copy-on-write

If a container modifies a file from a lower read-only layer, the file is copied into the writable layer first, then modified there.

Key sentence:

> The writable layer stores container-specific changes, while image layers remain immutable and reusable.

---

# 7. Building Images Uses Containers or Container-Like Environments

A common mental model is:

```text
Dockerfile → Image → Container
```

But internally, image building often uses containers:

```text
Dockerfile
    ↓
Base filesystem
    ↓
Temporary build container for RUN
    ↓
Command execution
    ↓
Filesystem diff
    ↓
New image layer
    ↓
Final image
```

This is especially important for `RUN` instructions.

---

# 8. Dockerfile Instruction Behavior

## `FROM`

Selects the base image.

```dockerfile
FROM debian:latest
```

This defines the initial filesystem and metadata foundation.

## `RUN`

Executes a command during image build.

```dockerfile
RUN apt-get update
```

This usually runs inside a temporary build container or isolated build environment.

The resulting filesystem change becomes a new image layer.

## `COPY`

Copies files from the build context into the image filesystem.

```dockerfile
COPY app.py /app.py
```

It does not execute the copied file.

## `CMD`

Sets the default command for container runtime.

```dockerfile
CMD ["python3", "/app.py"]
```

It does not run during image build.

## `ENTRYPOINT`

Defines the main executable for the container at runtime.

```dockerfile
ENTRYPOINT ["python3"]
```

## `ENV`

Sets environment variable metadata.

```dockerfile
ENV APP_ENV=production
```

---

# 9. Common Trap: `RUN` vs `CMD`

This is one of the most common questions.

```text
RUN = build-time command execution
CMD = runtime default command metadata
```

Example:

```dockerfile
FROM python:3.12
RUN pip install flask
COPY app.py /app.py
CMD ["python", "/app.py"]
```

What happens?

```text
RUN pip install flask
→ executes during build
→ creates an image layer

COPY app.py /app.py
→ copies app.py into the image filesystem

CMD ["python", "/app.py"]
→ saved as metadata
→ used when a container starts
```

Strong answer:

> `RUN` changes the image during build. `CMD` tells the container what to run by default at runtime.

---

# 10. Filesystem Diff

A filesystem diff is the change between two filesystem states.

For example:

```dockerfile
RUN apt-get install -y python3
```

Before:

```text
Python is not installed
```

After:

```text
Python binaries, libraries, and package metadata are added
```

The difference is saved as a layer.

```text
Before filesystem
    ↓
Command runs
    ↓
After filesystem
    ↓
Diff captured
    ↓
Image layer created
```

Useful phrase:

> Image building is essentially repeated filesystem preparation, command execution, diff capture, and layer creation.

---

# 11. Build Context

The **build context** is the set of files sent to the build engine.

Example:

```bash
docker build -t myapp .
```

Here, `.` is the build context.

`COPY` and `ADD` can access files from this context.

Example:

```dockerfile
COPY app.py /app.py
```

Docker can only copy `app.py` if it exists inside the build context.

Important point:

> The build context affects build performance and security because unnecessary files may be sent to the build engine.

Use `.dockerignore` to exclude unnecessary files.

---

# 12. `scratch`

`scratch` is an empty base image.

```dockerfile
FROM scratch
COPY my-static-binary /app
CMD ["/app"]
```

This is useful for minimal images, especially statically linked binaries.

Important meaning:

```text
FROM scratch = start from an empty filesystem
```

It does not provide:

```text
shell
package manager
standard libraries
debug tools
CA certificates
```

Warning:

> A scratch image is minimal, but debugging is harder because it lacks shell and common tools.

---

# 13. `docker commit`

`docker commit` creates an image from a container’s current filesystem state.

Flow:

```bash
docker run -it debian bash
apt-get update
apt-get install -y python3
exit
docker commit <container-id> my-python-image
```

This works, but it is usually discouraged for production image builds.

Why use it?

```text
Poor reproducibility
Manual steps are hidden
Harder to review
Harder to automate
Harder to rebuild consistently
```

Better:

```text
Use Dockerfile / Buildfile / automated build definition
```

---

# 14. Reproducibility

Reproducibility means:

> The same build instructions should produce the same or predictable output.

Dockerfiles help because the build process is written down explicitly.

Bad:

```text
Run container
Manually install packages
Commit container
```

Better:

```dockerfile
FROM debian
RUN apt-get update && apt-get install -y python3
COPY app.py /app.py
CMD ["python3", "/app.py"]
```

---

# 15. Buildah, Kaniko, and Similar Builders

## Buildah

Buildah makes the image-building process more explicit.

Conceptual flow:

```text
buildah from fedora
    ↓
working container
    ↓
buildah run ...
    ↓
filesystem changes
    ↓
buildah commit
    ↓
image
```

Buildah shows clearly that building images can involve a working container.

## Kaniko-style Build

Kaniko-style builders may not start Docker daemon-managed temporary containers in the same traditional way.

Instead, they often modify a filesystem directly while running inside a containerized environment.

Conceptual flow:

```text
Builder runs inside a container
    ↓
Dockerfile instructions interpreted
    ↓
Filesystem modified
    ↓
Diffs/layers produced
    ↓
Image created
```

Key idea:

> Even when the build tool does not visibly create temporary containers, it still needs a safe isolated environment to avoid damaging the host filesystem.

---

# 16. OCI: Open Container Initiative

OCI stands for Open Container Initiative.

OCI defines open standards for container ecosystems.

Important OCI specs:

| Spec                           | Purpose                                       |
| ------------------------------ | --------------------------------------------- |
| **Runtime Specification**      | How to run a container                        |
| **Image Specification**        | How container images are formatted            |
| **Distribution Specification** | How images are distributed through registries |

---

# 17. OCI Runtime View of a Container

OCI defines a container as an environment for executing processes with:

```text
Configurable isolation
Resource limitations
Platform-specific behavior
Lifecycle operations
```

This is more general than saying:

```text
Container = Linux process
```

Better:

```text
Container = standardized execution environment for processes
```

---

# 18. OCI Runtime Lifecycle

A runtime usually supports lifecycle operations like:

```text
create
start
kill
delete
state
```

## `create`

Prepares the container environment.

This may include:

```text
Preparing namespaces
Setting up mounts
Applying cgroup configuration
Preparing root filesystem
Setting hostname
Preparing process configuration
```

The user process may not be running yet.

## `start`

Starts the configured process inside the prepared container environment.

## `kill`

Sends a signal to the container process.

## `delete`

Cleans up container resources.

Flow:

```text
OCI Bundle
    ↓
create
    ↓
environment prepared
    ↓
start
    ↓
process runs
    ↓
kill / exits
    ↓
delete
    ↓
resources cleaned up
```

---

# 19. Linux Container Building Blocks

Linux containers commonly use these kernel features:

| Feature               | Purpose                                      |
| --------------------- | -------------------------------------------- |
| **PID namespace**     | Isolates process IDs                         |
| **Network namespace** | Isolates network interfaces, routes, ports   |
| **Mount namespace**   | Isolates filesystem mount points             |
| **UTS namespace**     | Isolates hostname/domain name                |
| **IPC namespace**     | Isolates IPC resources                       |
| **User namespace**    | Maps users/groups between host and container |
| **Cgroups**           | Limits and accounts resources                |
| **Capabilities**      | Breaks root privileges into smaller units    |
| **Seccomp**           | Filters system calls                         |

Important:

> Linux containers share the host kernel.

That is why they are lightweight, but also why isolation is not identical to a full virtual machine.

---

# 20. Namespace vs Cgroup

## Namespace

Namespaces answer:

```text
What can the process see?
```

Examples:

```text
Process list
Network interfaces
Hostname
Mount points
User IDs
```

## Cgroup

Cgroups answer:

```text
How much can the process use?
```

Examples:

```text
CPU
Memory
Block I/O
PIDs
```

Useful phrase:

> Namespaces isolate visibility; cgroups control resource usage.

---

# 21. OCI Is Not Linux-Only

Although Linux containers are the most common, OCI is broader.

OCI can describe container execution environments across different platforms and implementations.

Possible implementations include:

```text
Linux namespace/cgroup containers
VM-backed containers
Sandboxed containers
Windows containers
```

So this statement is incomplete:

```text
Container = Linux process
```

Better:

```text
Container = OCI-standard execution environment that may be implemented using Linux primitives, VMs, or sandboxing.
```

---

# 22. VM-Backed Containers

A VM-backed container uses virtual machine isolation instead of only Linux namespaces and cgroups.

Conceptual model:

```text
Application
    ↓
Container root filesystem
    ↓
Guest kernel / VM
    ↓
Hypervisor
    ↓
Host kernel / hardware
```

Compared with normal Linux containers:

```text
Linux container:
App → namespaces/cgroups → host kernel

VM-backed container:
App → guest kernel → hypervisor → host
```

Why use it?

```text
Stronger isolation
Better for untrusted workloads
Useful in multi-tenant platforms
Useful in serverless environments
```

Trade-off:

```text
Usually more overhead than plain runc containers
```

---

# 23. Kata Containers

Kata Containers combine:

```text
Container-like workflow
+
VM-level isolation
```

Useful phrase:

> Kata provides OCI-compatible containers backed by lightweight virtual machines.

It is useful when you want the operational model of containers but stronger isolation boundaries.

---

# 24. MicroVM and Firecracker

A **MicroVM** is a very small, lightweight virtual machine designed for fast startup and low overhead.

Firecracker is a well-known MicroVM technology.

Conceptual purpose:

```text
VM-like isolation
+
Container-like efficiency
```

Useful for:

```text
Serverless workloads
Multi-tenant workloads
Untrusted code execution
Fast isolated workloads
```

Important distinction:

> Firecracker itself is not the same thing as a container image or a container runtime. It provides a lightweight VM foundation that can be used by higher-level systems.

---

# 25. gVisor

gVisor provides a sandboxed container runtime approach.

It inserts a user-space kernel-like layer between the application and the host kernel.

Conceptual model:

```text
Application
    ↓
gVisor user-space kernel layer
    ↓
Host kernel
```

Its OCI runtime is commonly known as:

```text
runsc
```

Why use it?

```text
Extra isolation
Reduced direct exposure to host kernel
Useful for untrusted workloads
```

Trade-off:

```text
Can add performance overhead
May not support every Linux syscall exactly like the host kernel
```

---

# 26. runc vs gVisor vs Kata

| Runtime          | Isolation Model            | Notes                                                  |
| ---------------- | -------------------------- | ------------------------------------------------------ |
| `runc`           | Linux namespaces + cgroups | Fast, common, shares host kernel                       |
| `runsc` / gVisor | User-space kernel sandbox  | More isolation, possible overhead                      |
| Kata runtime     | VM-backed isolation        | Stronger boundary, more overhead than plain containers |

Useful comparison:

```text
runc  = standard Linux container
gVisor = sandboxed container
Kata  = VM-backed container
```

---

# 27. Docker, containerd, runc: Who Does What?

## Docker

High-level developer-facing tool.

Handles:

```text
CLI
Builds
Image management
Container management
Networking UX
Volume UX
```

## containerd

Container lifecycle and image management daemon.

Handles:

```text
Pulling images
Managing snapshots
Preparing runtime bundles
Calling low-level runtimes
```

## runc

Low-level OCI runtime.

Handles:

```text
Create container
Start process
Apply namespaces/cgroups/mounts
Manage lifecycle according to OCI runtime spec
```

Flow:

```text
docker
    ↓
containerd
    ↓
runc
    ↓
Linux container process
```

---

# 28. CRI and Kubernetes Context

In Kubernetes, the kubelet does not normally talk directly to Docker.

Modern Kubernetes uses the **Container Runtime Interface**, or CRI.

Flow:

```text
kubelet
    ↓ CRI
containerd / CRI-O
    ↓
OCI runtime
    ↓
container process
```

Useful phrase:

> Kubernetes relies on CRI-compatible runtimes, which eventually use OCI runtimes to create containers.

---

# 29. Common Questions and Good Answers

## Q1. Is a container an image?

No.

An image is a static packaged artifact. A container is a runtime execution environment created from filesystem content and configuration.

---

## Q2. Can you run a container without an image?

At the low-level runtime layer, yes.

An OCI runtime can run a container using:

```text
config.json + rootfs
```

This is an OCI bundle.

In real-world workflows, images are normally used to package and distribute that filesystem and metadata.

---

## Q3. Why do we need images then?

Images solve packaging, distribution, caching, reproducibility, and layer reuse.

They make containers practical across machines, teams, CI/CD pipelines, and production clusters.

---

## Q4. What is an OCI bundle?

An OCI bundle is the runtime-ready filesystem structure used by OCI runtimes.

```text
bundle/
├── config.json
└── rootfs/
```

`config.json` contains runtime configuration.

`rootfs` contains the container filesystem.

---

## Q5. What is the difference between `RUN` and `CMD`?

```text
RUN = executes during image build and creates a layer
CMD = sets default command metadata for runtime
```

---

## Q6. What happens during `docker build` for a `RUN` instruction?

A filesystem is prepared from the previous layer, a command is executed inside an isolated build environment, the filesystem diff is captured, and that diff becomes a new image layer.

---

## Q7. What is a writable layer?

It is the top layer added when a container runs.

It stores container-specific changes while image layers remain read-only.

---

## Q8. What is OverlayFS?

OverlayFS is a Linux filesystem mechanism used to present multiple layers as one unified filesystem.

It supports copy-on-write behavior for container filesystems.

---

## Q9. What is `scratch`?

`scratch` is an empty base image.

It is useful for minimal images, especially statically linked binaries.

---

## Q10. What is the difference between namespace and cgroup?

```text
Namespace = controls what a process can see
Cgroup    = controls how much resource a process can use
```

---

## Q11. What is OCI?

OCI stands for Open Container Initiative.

It defines standards for container runtime, image format, and distribution.

---

## Q12. Is OCI Linux-only?

No.

Linux containers are common, but OCI can describe container execution environments across multiple implementations, including Linux containers, VM-backed containers, and sandboxed runtimes.

---

## Q13. What is a VM-backed container?

A container that uses a virtual machine for stronger isolation instead of relying only on Linux namespaces and cgroups.

---

## Q14. What is gVisor?

gVisor is a sandboxed container runtime approach that places a user-space kernel-like layer between the application and the host kernel.

---

## Q15. What is Kata Containers?

Kata Containers provide container-like workflows with VM-level isolation.

---

# 30. One-Minute Revision

```text
Image:
Packaged filesystem + metadata.

Container:
Isolated execution environment for one or more processes.

OCI Bundle:
config.json + rootfs.

Runtime:
Creates and starts containers from OCI bundles.

runc:
Low-level OCI runtime using Linux namespaces and cgroups.

Docker/containerd:
Higher-level tools that manage images, snapshots, bundles, and runtime invocation.

RUN:
Executes during build and creates filesystem diff/layer.

CMD:
Runtime default command metadata.

Image layer:
Immutable filesystem diff.

Writable layer:
Container-specific top layer.

OverlayFS:
Combines read-only image layers and writable layer.

Namespace:
Controls what a process can see.

Cgroup:
Controls how much resource a process can use.

Kata:
VM-backed container.

gVisor:
User-space kernel sandbox.

Firecracker:
MicroVM foundation for lightweight VM isolation.
```

---

# 31. Finally

> A container image is a static, layered package containing filesystem content and metadata. A container is the runtime execution environment created from filesystem content and configuration. At the OCI runtime level, a Docker image is not strictly required; the runtime needs an OCI bundle containing `config.json` and `rootfs`. Images are still essential because they provide packaging, distribution, caching, reproducibility, and layer reuse. During image builds, `RUN` instructions are executed in temporary isolated environments, their filesystem diffs are captured, and those diffs become immutable image layers. OCI standardizes this ecosystem by defining image, runtime, and distribution specifications, while runtimes such as `runc`, gVisor, and Kata implement different isolation models using Linux primitives, sandboxing, or virtual machines.
