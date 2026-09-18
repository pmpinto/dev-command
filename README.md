# dev-command

One-command containerized development environments. **Podman-native (v2).**

## The Problem

- Legacy projects need old versions of Node/Python/Ruby, new projects need the latest
- "Works on my machine" - local dependencies conflict across projects
- Running untrusted third-party code on your host machine is risky
- Forgetting how each project starts: `npm start`? `npm run dev`? `bundle exec rails server`?

## The Solution

`dev` gives you isolated, consistent, disposable development environments with a single command:

- **System-agnostic** - No local dependencies, run any runtime version
- **Consistent** - `dev` is the only command you need
- **Safe** - Code runs in containers, isolated from your host
- **Zero modifications** - No Dockerfiles in your project repo
- **Disposable** - Mess it up? `dev clean` and start fresh

## Comparison

| Feature | dev-command | devcontainers |
|---------|-------------|----------------|
| IDE required | No | Yes (VS Code, JetBrains) |
| Project modifications | No | Yes (`.devcontainer.json`) |
| Start command | `dev` | Varies by project |
| Customizable | Full Dockerfile control | Limited config |
| Learning curve | Low | Medium |

## Prerequisites

- [podman](https://podman.io/)
- [podman-compose](https://github.com/containers/podman-compose)
- [fzf](https://github.com/junegunn/fzf)
- [jq](https://jqlang.github.io/jq/)
- [curl](https://curl.se/)

Install on Arch: `sudo pacman -S podman podman-compose fzf jq curl`

## Why Podman (v2)?

dev-command ran on Docker through v1. Version 2 is **Podman-native** for a few reasons:

- **Rootless by default** - containers run as your user, no daemon with root privileges, no docker-group-with-passwordless-root security hole
- **Daemonless** - no always-on background service; `podman` activates on demand
- **No stale daemon bugs** - the "I stopped using docker for three months and now it can't connect to its own socket" failure mode is gone

The environment files are standard-compliant, so nothing meaningful changed:

- `compose.yml` is the engine-neutral Compose Specification - works identically
- The build file is now **`Containerfile`** (the OCI-canonical name). If a project's folder still has a `Dockerfile`, Buildah (podman's builder) reads that too - both are byte-for-byte the same format.

If you're migrating an existing project, the only real steps are: install podman + podman-compose, rename `container/Dockerfile` → `container/Containerfile` (optional, cosmetic), and keep using `dev` exactly as before.

## Installation

```bash
# Download the script
curl -fsSL https://raw.githubusercontent.com/pmpinto/dev-command/main/dev -o dev
chmod +x dev

# Add to PATH (or put it in ~/scripts and add to .zshrc/.bashrc)
export PATH="$HOME/scripts:$PATH"
```

## Quick Start

```bash
# 1. Create a project folder
mkdir my-project && cd my-project

# 2. Create the repo folder and add your code
mkdir repo
# ... put your code in repo/ ...

# 3. Initialize the container environment
dev init
# Select a runtime (e.g., node, python, ruby)
# Select a version

# 4. Start coding
dev
```

Your folder structure:
```
my-project/
├── repo/       # Your code lives here
└── container/  # (generated) container config
```

## Commands

| Command | Description |
|---------|-------------|
| `dev` | Start the dev container shell (builds if no image) |
| `dev init` | Initialize a new container environment |
| `dev build` | Build container image |
| `dev rebuild` | Rebuild the container image without cache |
| `dev run` | Run container (no build) |
| `dev cli` | Open shell in running container |
| `dev clean` | Remove container, image, and `container/` folder |

## How It Works

1. **`dev init`** prompts you to choose a runtime (e.g., `node`, `python`, `ruby`, `ubuntu`) and a version
2. It creates a `container/` folder with a minimal `Containerfile` and `compose.yml`
3. The `Containerfile` uses your chosen base image
4. The `compose.yml` mounts your `repo/` folder into the container at `/app`
5. Running `dev` starts the container and drops you into a shell, unless you change the `CMD` entry in the Containerfile

Generated folder structure:
```
my-project/
├── repo/           # Your code lives here
└── container/      # (generated) container config
    ├── compose.yml
    ├── Containerfile
    └── .env        # Port configuration (default: PORT=3000)
```

Key `compose.yml` security settings:
- **Bridged network isolation** (`networks: [dev-net]`) - Container has its own network stack, isolated from your host
- **Explicit port mapping** (`ports: "${PORT}:${PORT}"`) - Only explicitly mapped ports are accessible from your host
- **`host.docker.internal:host-gateway`** - Container can access your host's localhost services via `host.docker.internal:PORT`
- **`cap_drop: [ALL]`** - All Linux capabilities removed (no `ptrace`, no raw sockets, no file permission bypass)
- **`no-new-privileges: true`** - Prevents setuid privilege escalation
- **C2 domain sinkholing** - 8 known malicious domains sinkholed to `0.0.0.0` (blocks Shai-Hulud exfiltration)
- `user: "${HOST_UID}:${HOST_GID}"` - Run as you, not root
- `userns_mode: keep-id` - Maps your host user id into the container so bind-mount writes (`repo/`) work under rootless Podman
- `init: true` - Proper signal handling (Ctrl+C works)
- `volumes: ../repo:/app` - Your code is mounted, not copied

## Customization

### Changing the base image

Edit `container/Containerfile` after running `dev init`:

```dockerfile
FROM node:18
WORKDIR /app
CMD ["bash"]
```

### Adding packages

Add `RUN` commands to the Containerfile:

```dockerfile
FROM node:20
WORKDIR /app
RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*
CMD ["bash"]
```

### Setting a default command

Instead of dropping into a shell, start the project directly:

```dockerfile
FROM node:20
WORKDIR /app
CMD ["npm", "start"]
```

Now `dev` will run `npm start` instead of giving you a shell.

### Environment variables and Port Configuration

The compose file uses `container/.env` for configuration. The `PORT` variable controls both the port mapping and is passed to the container:

```
# container/.env (generated automatically)
PORT=3000
```

Change the `PORT` value in `.env` to use a different port. Add more environment variables in `container/compose.yml`:

```yaml
environment:
  - PORT
  - NODE_ENV=development
  - DATABASE_URL
```

**Note:** To access services running on your **host** from inside the container, use `host.docker.internal:PORT` instead of `localhost:PORT`.

### Using an existing Containerfile / Dockerfile

If the project already has a `Containerfile` (or a `Dockerfile`) in the container folder, that one is used. Podman's builder also auto-detects a `Dockerfile` with no changes needed.

## Multi-Container Projects

The generated `compose.yml` creates a dedicated network `dev-net` (named after your project, e.g., `my-project-net`). All services on this network can communicate with each other using their service names.

### Service Communication Patterns

| Direction | Pattern | Example |
|-----------|---------|---------|
| **Container → Container** | `http://<service-name>:<port>` | `http://api:3000` |
| **Container → Host** | `host.docker.internal:<port>` | `host.docker.internal:5432` (PostgreSQL on host) |
| **Host → Container** | `localhost:<mapped-port>` | `localhost:3000` (via ports mapping) |

### Example: app + API + Workers

Edit `container/compose.yml` after `dev init` to add more services:

```yaml
name: my-project-dev

networks:
  dev-net:
    name: my-project-net
    driver: bridge

services:
  app:
    build: .
    working_dir: /app
    user: "${HOST_UID}:${HOST_GID}"
    ports:
      - "${PORT}:${PORT}"
    networks:
      - dev-net
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - ../repo:/app
    environment:
      - PORT
    stdin_open: true
    tty: true
    init: true
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true

  api:
    build: .
    working_dir: /app
    user: "${HOST_UID}:${HOST_GID}"
    ports:
      - "3001:3001"
    networks:
      - dev-net
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - ../repo:/app
    init: true
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true

  workers:
    build: .
    working_dir: /app
    user: "${HOST_UID}:${HOST_GID}"
    networks:
      - dev-net
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - ../repo:/app
    init: true
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
```

Now the `app` service can access the API via `http://api:3001` (instead of `http://localhost:3001`), and both `app` and `api` can access the workers via `http://workers:<port>`.

## Security Hardening (Node Runtime)

When using the `node` runtime, the generated Containerfile includes npm config hardening:

```dockerfile
# Harden npm config - blocks postinstall hooks and fast-burst worms
RUN echo "ignore-scripts=true" >> /root/.npmrc && \
    echo "min-release-age=7" >> /root/.npmrc && \
    echo "ignore-scripts=true" >> /etc/npmrc && \
    echo "min-release-age=7" >> /etc/npmrc
```

**What this means:**
- `ignore-scripts=true` - Blocks all `postinstall`/`preinstall` hooks. This is the primary entry point for supply chain worms like Shai-Hulud.
- `min-release-age=7` - Blocks packages published in the last 7 days. The TanStack/Mini Shai-Hulud attack was exploited within 6 minutes of publishing.

**Tradeoff:** Packages with legitimate native builds (esbuild, sharp, prisma, swc, etc.) will fail to build automatically. You may need to run `npm rebuild <package>` manually after install.

## Troubleshooting

**"repo/ folder not found"**  
Run `dev` from the project root. You need a `repo/` folder at `$PWD/repo`.

**"Error: No such file or directory" for fzf/jq**  
Install the prerequisites.

**Permission denied when creating files**  
The container runs as your host user (via `HOST_UID`/`HOST_GID`), so file permissions should work. If you still have issues, check `podman info` runs cleanly.

**Container starts but immediately exits**  
Your container likely needs a persistent shell. Edit `container/Containerfile` and ensure `CMD ["bash"]` (not `CMD ["node"]` or similar).

**Cannot access app from host browser**  
Check the `PORT` in `container/.env` matches the port your app is listening on. The default is `PORT=3000`. If your app uses port `8080`, change the `.env` file.

**Port already in use**  
Change the `PORT` value in `container/.env` to use a different port.

**Cannot access host services from container**  
Use `host.docker.internal:PORT` instead of `localhost:PORT`.

**npm install fails for native modules (esbuild, sharp, prisma, etc.)**  
The `ignore-scripts=true` setting blocks automatic postinstall hooks. Try `npm rebuild <package>` to rebuild native modules manually.

