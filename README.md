# dev-command

One-command containerized development environments.

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

- [Docker](https://www.docker.com/)
- [docker-compose](https://docs.docker.com/compose/)
- [fzf](https://github.com/junegunn/fzf)
- [jq](https://jqlang.github.io/jq/)
- [curl](https://curl.se/)

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
└── container/  # (generated) Docker config
```

## Commands

| Command | Description |
|---------|-------------|
| `dev` | Start the dev container shell |
| `dev init` | Initialize a new container environment |
| `dev cli` | Build (if needed) and drop into container |
| `dev rebuild` | Rebuild the container image without cache |
| `dev clean` | Remove container, image, and `container/` folder |

## How It Works

1. **`dev init`** prompts you to choose a runtime (e.g., `node`, `python`, `ruby`, `ubuntu`) and a version
2. It creates a `container/` folder with a minimal `Dockerfile` and `compose.yml`
3. The `Dockerfile` uses your chosen base image
4. The `compose.yml` mounts your `repo/` folder into the container at `/app`
5. Running `dev` starts the container and drops you into a shell, unless you change the `CMD` entry in the Dockerfile

Key `compose.yml` settings:
- `network_mode: host` - Access localhost services from the container
- `user: "${HOST_UID}:${HOST_GID}"` - Run as you, not root
- `init: true` - Proper signal handling (Ctrl+C works)
- `volumes: ../repo:/app` - Your code is mounted, not copied

## Customization

### Changing the base image

Edit `container/Dockerfile` after running `dev init`:

```dockerfile
FROM node:18
WORKDIR /app
CMD ["bash"]
```

### Adding packages

Add `RUN` commands to the Dockerfile:

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

### Environment variables

The compose file forwards any `PORT` environment variable from your host. Add more in `container/compose.yml`:

```yaml
environment:
  - PORT
  - NODE_ENV=development
  - DATABASE_URL
```

### Using an existing Dockerfile

If the project already has a `Dockerfile` in the root, the script will detect it and use that instead.

## Troubleshooting

**"repo/ folder not found"**  
Run `dev` from the project root. You need a `repo/` folder at `$PWD/repo`.

**"Error: No such file or directory" for fzf/jq**  
Install the prerequisites.

**Permission denied when creating files**  
The container runs as your host user (via `HOST_UID`/`HOST_GID`), so file permissions should work. If you still have issues, check your Docker daemon is running.

**Container starts but immediately exits**  
Your container likely needs a persistent shell. Edit `container/Dockerfile` and ensure `CMD ["bash"]` (not `CMD ["node"]` or similar).

**Port already in use**  
`network_mode: host` means the container uses your host's network. If you get port conflicts, change the service port in your app's config.

