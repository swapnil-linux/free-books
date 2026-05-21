# RH134 Chapter 17 - Managing Containers with Podman

> **Course:** Red Hat System Administration II (RH134)
> **Objective:** Find, run, and manage containers and container images by using the Podman container management tool.

---

## Windows vs Linux: Container Equivalents

| Windows Concept | Linux / Podman Equivalent |
|---|---|
| Hyper-V VM | Container (lighter, shares host kernel) |
| Docker Desktop (Windows) | Podman (daemonless, rootless-capable) |
| Docker CLI (`docker run`) | `podman run` (compatible syntax) |
| Docker Hub | `registry.access.redhat.com`, `quay.io` |
| `docker pull` | `podman pull` |
| `docker images` | `podman images` |
| `docker ps` | `podman ps` |
| `docker build` | `podman build` |
| Dockerfile | Containerfile (same format, different name) |
| `-p 8080:80` port mapping | `-p 8080:80` (identical syntax) |
| `docker exec -it CONTAINER bash` | `podman exec -it CONTAINER bash` |
| Windows Container Registry | Red Hat Registry / Quay.io |

---

## 1. Container Concepts

### VM vs Container

| | Virtual Machine | Container |
|---|---|---|
| What is isolated | Entire OS kernel + hardware | Filesystem, processes, network |
| Startup time | Minutes (kernel boot) | Milliseconds |
| Size | Gigabytes | Megabytes |
| Isolation method | Hypervisor | Linux namespaces + cgroups |
| Shares host kernel | No | Yes |
| Resource overhead | High | Very low |

### Key Terms

| Term | Meaning |
|---|---|
| **Container image** | Read-only package: app + all dependencies, layered filesystem |
| **Container** | Running instance of an image - adds a writable layer on top |
| **Registry** | Server that stores and distributes container images |
| **Tag** | Label on an image version (e.g. `latest`, `1.0`, `RHEL10`) |
| **Containerfile** | Text file with instructions to build a container image |
| **OCI** | Open Container Initiative - standard for image format and runtime |

> **Images are read-only. Containers are writable.** When you run a container, Podman creates a thin writable layer on top of the image. If you delete the container, that layer is gone. The image is unchanged. Multiple containers can run from the same image simultaneously.

### Podman: Daemonless by Design

- Docker requires a root-privileged background daemon (`dockerd`)
- Podman interacts directly with the kernel - no daemon required
- Containers run as child processes of the user who started them
- Supports **rootless containers** (run as a normal user, not root)
- Security benefit: no persistent privileged process to compromise

### Linux Kernel Features Behind Containers

| Feature | What it provides |
|---|---|
| **Namespaces** | Isolation - each container sees its own PID tree, network, filesystem, hostname |
| **cgroups** (control groups) | Resource limits - CPU, memory, disk I/O per container |

---

## 2. Container Image Registries

Red Hat provides multiple registries:

| Registry | Authentication | Content |
|---|---|---|
| `registry.redhat.io` | Required (Red Hat login) | Official Red Hat products |
| `registry.access.redhat.com` | Not required | UBI images, freely available |
| `registry.connect.redhat.com` | Required | Red Hat Partner Connect images |
| `quay.io` | Optional (for private repos) | Red Hat-hosted, public and private |
| `docker.io` (Docker Hub) | Optional (for private repos) | Community images |

> **Search for images:** https://catalog.redhat.com/search

### Logging In to a Registry

```bash
# Log in to a registry (credentials saved to ~/.config/containers/auth.json)
podman login registry.redhat.io
podman login registry.lab.example.com:5000

# With credentials inline
podman login -u USERNAME -p PASSWORD registry.redhat.io

# Log out
podman logout registry.redhat.io
```

---

## 3. Working with Container Images

### Searching and Pulling Images

```bash
# Search a registry for images
podman search registry.lab.example.com:5000/
podman search registry.access.redhat.com/ubi

# Search and list all tags for an image
podman search --list-tags registry.lab.example.com:5000/my_image

# Pull an image to local storage
podman pull registry.access.redhat.com/ubi10/ubi
podman pull registry.lab.example.com:5000/rhel10/httpd-24

# Pull a specific tagged version
podman pull registry.access.redhat.com/ubi10/ubi:latest
```

### Viewing Local Images

```bash
# List all local images
podman images
podman image list   # same command

# Inspect image metadata (full JSON output)
podman image inspect IMAGE
podman image inspect my_image:1.0

# Show image history (layers)
podman history IMAGE
```

### Tagging Images

```bash
# Assign a new name/tag to an existing image (does not copy - same image ID)
podman tag SOURCE_IMAGE TARGET_IMAGE

# Examples
podman tag my_image:1.0 my_image:stable
podman tag c6222576494f registry.example.com/myorg/myapp:1.0
podman tag localhost/my_image:1.1 registry.lab.example.com:5000/my_image:1.1
```

### Pushing Images to a Registry

```bash
# Push local image to a registry
podman image push SOURCE_IMAGE TARGET_REGISTRY/IMAGE:TAG

# Examples
podman image push localhost/my_image:1.0 registry.lab.example.com:5000/my_image:1.0
podman image push my-httpd registry.redhat.io/myorg/my-httpd
```

### Removing Images

```bash
# Remove a specific image (by name or ID)
podman rmi IMAGE
podman rmi IMAGE:TAG
podman rmi IMAGE1 IMAGE2    # multiple at once

# Remove all untagged/unreferenced images (dangling images)
podman image prune

# Remove ALL images not used by any container
podman image prune --all
```

---

## 4. Running Containers

### Basic `podman run` Syntax

```bash
podman run [OPTIONS] IMAGE [COMMAND [ARGS]]
```

### Key `podman run` Options

| Option | Purpose | Example |
|---|---|---|
| `-d` | Detached mode (run in background) | `podman run -d IMAGE` |
| `--name NAME` | Assign a name to the container | `--name my_webserver` |
| `-p HOSTPORT:CTRPORT` | Map host port to container port | `-p 8080:80` |
| `-e VAR=VALUE` | Set an environment variable | `-e MYSQL_ROOT_PASSWORD=secret` |
| `-v HOSTPATH:CTRPATH` | Mount a host directory into the container | `-v /data:/var/data` |
| `--rm` | Automatically remove container when it stops | `podman run --rm IMAGE` |
| `-it` | Interactive with a terminal (for shells) | `podman run -it IMAGE bash` |
| `--entrypoint` | Override the default entrypoint command | `--entrypoint /bin/bash` |

### Port Mapping Direction

```
-p  HOSTPORT  :  CONTAINERPORT
    8080             80

External client connects to HOST:8080
Traffic is forwarded to container port 80
```

> **Left = host (outside). Right = container (inside).** `-p 8080:80` means "connect to me on 8080, I will forward it to port 80 inside the container."

### Run Examples

```bash
# Run a one-shot command and exit
podman run registry.access.redhat.com/ubi10/ubi echo 'Hello Red Hat!'

# Run interactively (get a shell inside the container)
podman run -it registry.access.redhat.com/ubi10/ubi bash

# Run in the background (detached), expose a port, give it a name
podman run -d \
  --name my_webserver \
  -p 8080:8080 \
  registry.lab.example.com:5000/rhel10/httpd-24

# Run with environment variables
podman run -d \
  --name mydb \
  -e MYSQL_ROOT_PASSWORD=secretpassword \
  registry.access.redhat.com/rhel10/mysql-80

# Run and automatically remove when done (good for one-off tasks)
podman run --rm IMAGE command

# Run with a volume mount (host dir -> container dir)
podman run -d \
  --name myapp \
  -v /home/student/html:/var/www/html:Z \
  -p 8080:80 \
  my-httpd
```

> **The `:Z` on volume mounts** sets the correct SELinux context on the host directory so the container process can access it. Without `:Z`, SELinux may deny access even if permissions are correct.

---

## 5. Managing Running Containers

### Viewing Containers

```bash
# Show RUNNING containers only
podman ps

# Show ALL containers (running and stopped)
podman ps -a

# Custom output format (show specific fields)
podman ps -a --format "table {{.ID}}\t{{.Image}}\t{{.Status}}"
podman ps --format "{{.Names}}\t{{.Status}}\t{{.Ports}}"

# Detailed JSON info about a specific container
podman inspect CONTAINER
podman inspect my_webserver
```

### `podman ps` Output Columns

| Column | Meaning |
|---|---|
| `CONTAINER ID` | Short unique ID |
| `IMAGE` | Image the container was created from |
| `COMMAND` | Command running inside the container |
| `CREATED` | When the container was created |
| `STATUS` | Running / Exited (exit code) / Paused |
| `PORTS` | Port mappings (HOST -> CONTAINER) |
| `NAMES` | Name (auto-assigned or from `--name`) |

### Controlling Containers

```bash
# Stop a running container gracefully (SIGTERM then SIGKILL)
podman stop CONTAINER
podman stop my_webserver

# Stop all running containers
podman stop -a

# Start a stopped container
podman start CONTAINER

# Restart (stop then start)
podman restart CONTAINER

# Send a specific signal
podman kill --signal SIGHUP CONTAINER

# Remove a stopped container (frees storage)
podman rm CONTAINER
podman rm my_webserver

# Remove a running container (force)
podman rm -f CONTAINER

# Remove all stopped containers
podman rm -a
```

### Interacting with Running Containers

```bash
# Execute a command inside a running container
podman exec CONTAINER COMMAND
podman exec my_webserver ls /var/www/html

# Get an interactive shell inside a running container
podman exec -it my_webserver bash
podman exec -it my_webserver /bin/sh   # if bash not available

# View container logs (stdout/stderr)
podman logs CONTAINER
podman logs my_webserver

# Follow logs live (like tail -f)
podman logs -f my_webserver

# Show last N lines of logs
podman logs --tail 20 my_webserver
```

---

## 6. Building Container Images with a Containerfile

### Containerfile Format

```dockerfile
# Comments start with #

# Every Containerfile must start with FROM
FROM BASE_IMAGE

# Run commands during build (each RUN creates a new layer)
RUN command
RUN dnf install -y httpd && dnf clean all

# Copy files from build context (local directory) into image
COPY source destination
COPY index.html /var/www/html/index.html

# Copy files (can unpack archives and fetch URLs)
ADD archive.tar.gz /destination/

# Set environment variables (available during build and at runtime)
ENV MYVAR=value

# Document which port the container listens on (does not actually publish)
EXPOSE 80

# Set default command when container starts (shell form)
CMD echo "Hello World"

# Set default command (exec form - preferred, avoids shell interpretation)
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]

# Set executable (CMD provides default arguments to ENTRYPOINT)
ENTRYPOINT ["/usr/sbin/httpd"]
```

### Common Containerfile Instructions

| Instruction | Purpose |
|---|---|
| `FROM` | Base image to build on (required, must be first) |
| `RUN` | Execute a command during build (creates a layer) |
| `COPY` | Copy local files into the image |
| `ADD` | Like COPY, but can unpack archives and fetch URLs |
| `ENV` | Set environment variables |
| `EXPOSE` | Document container listening port (metadata only) |
| `CMD` | Default command when container starts |
| `ENTRYPOINT` | Main executable (CMD provides args to this) |
| `LABEL` | Add metadata key-value pairs |
| `USER` | Set the user for subsequent RUN/CMD/ENTRYPOINT |
| `WORKDIR` | Set the working directory |

### Building an Image

```bash
# Build from a Containerfile in the current directory
# -t = tag (name:version for the resulting image)
# .  = build context (where to find COPY sources and the Containerfile)
podman build -t IMAGE:TAG .
podman build -t my-httpd:1.0 .

# Build from a specific Containerfile in a different directory
podman build -t my-httpd:1.0 /home/student/myapp/.

# Build specifying the Containerfile explicitly
podman build -t my-httpd:1.0 -f /path/to/Containerfile .

# Build without using cache (forces fresh layer builds)
podman build --no-cache -t my-httpd:latest .
```

### A Complete Containerfile + Build Example

```dockerfile
# Containerfile for a simple web server
FROM registry.access.redhat.com/ubi10/ubi

# Install httpd (clean up after to reduce layer size)
RUN dnf install -y httpd && dnf clean all

# Copy custom web content
COPY index.html /var/www/html/index.html

# Document the port
EXPOSE 80

# Start httpd in the foreground (daemon would exit immediately)
ENTRYPOINT ["/usr/sbin/httpd", "-DFOREGROUND"]
```

```bash
# Build it
podman build -t my-httpd:1.0 .

# Verify the image exists
podman images

# Test it locally
podman run -d --name test-web -p 8080:80 my-httpd:1.0
curl http://localhost:8080

# Clean up test container
podman stop test-web && podman rm test-web
```

---

## 7. Complete Workflow: Build, Tag, Push, Run

```bash
# 1. Write a Containerfile
vim Containerfile

# 2. Build the image
podman build -t my_image:1.0 .

# 3. Test it locally
podman run my_image:1.0

# 4. Tag for the registry
podman tag localhost/my_image:1.0 registry.example.com:5000/my_image:1.0

# 5. Log in to registry
podman login registry.example.com:5000

# 6. Push to registry
podman image push localhost/my_image:1.0 registry.example.com:5000/my_image:1.0

# 7. Verify it's in the registry
podman search --list-tags registry.example.com:5000/my_image

# 8. Remove local copy
podman rmi my_image:1.0

# 9. Pull and run from registry (simulating another machine)
podman run -d --name my_app -p 8080:80 registry.example.com:5000/my_image:1.0

# 10. Verify
podman ps
curl http://localhost:8080
```

---

## Quick Reference: All Commands

```bash
# --- Registry ---
podman login REGISTRY                   # Authenticate
podman logout REGISTRY                  # Log out
podman search REGISTRY/                 # Search a registry
podman search --list-tags REGISTRY/IMG  # List all tags

# --- Images ---
podman pull IMAGE                       # Download image
podman images                           # List local images
podman image inspect IMAGE              # Detailed image info
podman history IMAGE                    # Show build layers
podman tag SOURCE NEWTAG                # Tag/rename image
podman rmi IMAGE                        # Delete local image
podman image prune                      # Delete dangling images
podman image prune --all                # Delete unused images

# --- Build ---
podman build -t NAME:TAG .              # Build from Containerfile
podman build -t NAME:TAG DIR/           # Build from specific directory
podman build --no-cache -t NAME:TAG .   # Build ignoring cache

# --- Push ---
podman image push SRC REGISTRY/DEST:TAG  # Push to registry

# --- Run ---
podman run IMAGE                        # Run (foreground, auto-remove on exit)
podman run -d IMAGE                     # Run in background
podman run --name NAME IMAGE            # Run with a name
podman run -p 8080:80 IMAGE            # Run with port mapping
podman run -e VAR=VAL IMAGE            # Run with env variable
podman run -v /host:/ctr:Z IMAGE       # Run with volume mount
podman run --rm IMAGE CMD              # Run and auto-remove
podman run -it IMAGE bash              # Interactive shell

# --- Manage containers ---
podman ps                              # Running containers
podman ps -a                           # All containers (including stopped)
podman stop NAME                       # Stop gracefully
podman start NAME                      # Start a stopped container
podman restart NAME                    # Restart
podman rm NAME                         # Remove stopped container
podman rm -f NAME                      # Force remove running container
podman rm -a                           # Remove all stopped containers

# --- Interact ---
podman exec NAME COMMAND               # Run command in running container
podman exec -it NAME bash              # Interactive shell in running container
podman logs NAME                       # View stdout/stderr
podman logs -f NAME                    # Follow live logs
podman inspect NAME                    # Detailed container info
```

---

## Key Paths

| Path | Purpose |
|---|---|
| `~/.config/containers/auth.json` | Registry login credentials |
| `~/.local/share/containers/` | User container and image storage (rootless) |
| `/etc/containers/registries.conf` | System-wide registry configuration |

---

## Things to Remember

1. **`podman ps` shows only running containers. `podman ps -a` shows all.** Stopped containers still exist and use storage until you `podman rm` them.

2. **Port mapping is `HOST:CONTAINER`. Left is outside, right is inside.** `-p 8080:80` means external clients hit port 8080, which forwards to port 80 inside the container.

3. **Images are read-only. Containers add a writable layer.** Deleting a container (`podman rm`) does not free image storage. Use `podman rmi` to remove images.

4. **`podman rmi` fails if a container (even stopped) is using the image.** Remove the container first with `podman rm`, then remove the image.

5. **Add `:Z` to volume mounts when SELinux is enforcing.** `-v /host/dir:/ctr/dir:Z` sets the SELinux context on the host directory so the container process can access it.

6. **Each `RUN` instruction in a Containerfile creates a new image layer.** Chain commands with `&&` in a single `RUN` to reduce layer count: `RUN dnf install -y httpd && dnf clean all`.

7. **Layer caching speeds up builds.** Put rarely-changing instructions (FROM, OS packages) near the top. Put frequently-changing instructions (COPY your application code) near the bottom.

8. **`CMD` vs `ENTRYPOINT`:** `ENTRYPOINT` is the executable; `CMD` provides default arguments. If both are set, `CMD` is passed as arguments to `ENTRYPOINT`. Either can be overridden at `podman run` time.

9. **`EXPOSE` in a Containerfile is documentation only.** It does not publish ports. You must still use `-p` in `podman run` to actually expose a port to the host.

10. **Podman is daemonless and supports rootless containers.** Unlike Docker, Podman does not require a privileged background daemon. You can run containers as a normal user without `sudo`. This is the key security advantage for production environments.
