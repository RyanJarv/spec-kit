# Docker Sandbox Guide

Run Claude Code in isolated Docker sandbox environments using git worktrees, so you can work on multiple features in parallel without switching branches.

---

## Overview

The `specify sandbox` commands automate a full workflow:

1. Create a **git worktree** (isolated working directory on a new branch)
2. Build a **Docker sandbox image** from your Dockerfile
3. Launch a **sandboxed Claude Code** environment with auth provisioned

This keeps your main checkout untouched while Claude works in a sandboxed worktree.

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) with sandbox support (`docker sandbox` plugin)
- [Git](https://git-scm.com/downloads)
- A `CLAUDE_CODE_OAUTH_TOKEN` environment variable with your Claude auth token
- A `.specify/sandbox.json` configuration file (see [Configuration](#configuration))

Verify Docker is available:

```bash
specify check
```

Docker will appear in the tool check output.

---

## Configuration

Create `.specify/sandbox.json` in your project root:

```json
{
  "dockerfile": ".specify/sandbox.Dockerfile",
  "provision_hook": ".specify/hooks/sandbox-provision.sh",
  "image_name": "speckit-sandbox:latest"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `dockerfile` | Yes | Path (relative to repo root) to the Dockerfile for the sandbox image |
| `image_name` | No | Tag for the built image (default: `speckit-sandbox:latest`) |
| `provision_hook` | No | Script called after sandbox creation for custom provisioning |

### Creating a Dockerfile

The Dockerfile defines your sandbox environment. A minimal example:

```dockerfile
# .specify/sandbox.Dockerfile
FROM node:20-slim

# Install Claude Code
RUN npm install -g @anthropic-ai/claude-code

# Add any project-specific tools
RUN apt-get update && apt-get install -y git curl && rm -rf /var/lib/apt/lists/*
```

Customize this with your project's toolchain (language runtimes, build tools, linters, etc.).

---

## Commands

### `specify sandbox start`

Create a worktree, build the sandbox image, and start Claude.

```bash
specify sandbox start "Add user authentication" --short-name "user-auth"
```

**What it does:**

1. Validates `CLAUDE_CODE_OAUTH_TOKEN` is set
2. Builds the Docker image from your configured Dockerfile
3. Creates a git worktree at `../<repo>-<branch>` via `create-new-feature.sh --worktree`
4. Creates a Docker sandbox container mounting the worktree
5. Provisions the OAuth token into `/home/agent/.sandbox-env`
6. Runs the provision hook (if configured)
7. Starts Claude in the sandbox

**Options:**

| Option | Description |
|--------|-------------|
| `--short-name TEXT` | Custom short name for the branch |
| `--number TEXT` | Specify branch number manually |
| `--template TEXT` | Override Docker template image (skips Dockerfile build) |
| `--name TEXT` | Override sandbox container name |
| `--no-start` | Create sandbox but don't launch Claude |

**Examples:**

```bash
# Basic usage
specify sandbox start "Implement OAuth2 flow"

# With a short branch name and manual number
specify sandbox start "Implement OAuth2 flow" --short-name "oauth2" --number 5

# Create sandbox without starting Claude (for manual inspection)
specify sandbox start "Debug auth issue" --short-name "debug-auth" --no-start

# Use a pre-built template image instead of building from Dockerfile
specify sandbox start "Quick fix" --template my-image:latest --short-name "quick-fix"
```

### `specify sandbox shell`

Open a bash shell in an existing sandbox.

```bash
specify sandbox shell 001-user-auth
```

If no branch name is provided, it uses the current branch:

```bash
specify sandbox shell
```

**Options:**

| Option | Description |
|--------|-------------|
| `--name TEXT` | Sandbox name override (skip branch-based name derivation) |

### Sandbox naming convention

Sandbox containers are named `claude-<repo>-<branch>`. For example, if your repo is `my-project` and the branch is `001-user-auth`, the sandbox name is `claude-my-project-001-user-auth`.

---

## Git Worktrees

The `--worktree` flag on `create-new-feature.sh` (used internally by `sandbox start`) creates a separate working directory instead of switching branches:

```bash
# Direct usage (without sandbox)
bash .specify/scripts/bash/create-new-feature.sh --worktree "My feature" --short-name "my-feat"
```

This creates:
- A new branch `001-my-feat`
- A worktree directory at `../<repo>-001-my-feat/`
- A `specs/001-my-feat/spec.md` inside the worktree

Your main checkout stays on its current branch.

**Options:**

| Option | Description |
|--------|-------------|
| `--worktree` | Use `git worktree add` instead of `git checkout -b` |
| `--worktree-base-dir DIR` | Base directory for worktrees (default: parent of repo root) |

**List active worktrees:**

```bash
git worktree list
```

**Remove a worktree when done:**

```bash
git worktree remove ../my-project-001-my-feat
```

---

## Customizing the Sandbox

### Provision Hook

The provision hook is a script that runs after the sandbox container is created but before Claude starts. Use it to inject credentials, configure tools, or set up the environment.

The hook receives two arguments:

| Argument | Description |
|----------|-------------|
| `$1` | Sandbox container name |
| `$2` | Worktree directory path |

**Example: `.specify/hooks/sandbox-provision.sh`**

```bash
#!/usr/bin/env bash
SANDBOX_NAME="$1"
WORKTREE_DIR="$2"

# Provision GCP credentials
SA_KEY="$HOME/.config/gcloud/claude-code-deployer-key.json"
if [ -f "$SA_KEY" ]; then
  cat "$SA_KEY" | docker sandbox exec -i "$SANDBOX_NAME" bash -c '
    mkdir -p /home/agent/.config/gcloud
    cat > /home/agent/.config/gcloud/key.json
    echo "export GOOGLE_APPLICATION_CREDENTIALS=/home/agent/.config/gcloud/key.json" >> /home/agent/.profile
  '
fi

# Provision GitHub token
GH_TOKEN_FILE="$HOME/.config/gh/sandbox-token"
if [ -f "$GH_TOKEN_FILE" ]; then
  cat "$GH_TOKEN_FILE" | docker sandbox exec -i "$SANDBOX_NAME" gh auth login --with-token
fi

# Copy SSH keys for git operations
if [ -f "$HOME/.ssh/id_ed25519" ]; then
  cat "$HOME/.ssh/id_ed25519" | docker sandbox exec -i "$SANDBOX_NAME" bash -c '
    mkdir -p /home/agent/.ssh
    cat > /home/agent/.ssh/id_ed25519
    chmod 600 /home/agent/.ssh/id_ed25519
  '
fi
```

Make the hook executable:

```bash
chmod +x .specify/hooks/sandbox-provision.sh
```

### Custom Docker Image

You can build on top of any base image. Common patterns:

**Python project:**

```dockerfile
FROM python:3.12-slim
RUN npm install -g @anthropic-ai/claude-code
RUN pip install poetry
```

**Rust project:**

```dockerfile
FROM rust:1.77-slim
RUN apt-get update && apt-get install -y nodejs npm && rm -rf /var/lib/apt/lists/*
RUN npm install -g @anthropic-ai/claude-code
```

**With MCP servers:**

```dockerfile
FROM node:20-slim
RUN npm install -g @anthropic-ai/claude-code
RUN npm install -g @anthropic-ai/mcp-server-filesystem
```

### Using a Pre-built Image

If you don't want to build from a Dockerfile each time, use `--template` to skip the build step:

```bash
# Build once
docker build -f .specify/sandbox.Dockerfile -t my-sandbox:latest .

# Reuse for multiple features
specify sandbox start "Feature A" --template my-sandbox:latest --short-name "feat-a"
specify sandbox start "Feature B" --template my-sandbox:latest --short-name "feat-b"
```

---

## Manual Cleanup

Sandbox resources are not automatically cleaned up. When you're done with a feature:

```bash
# Remove the sandbox container
docker sandbox rm claude-my-project-001-my-feat

# Remove the worktree and branch
git worktree remove ../my-project-001-my-feat
git branch -d 001-my-feat
```

List active sandboxes and worktrees:

```bash
docker sandbox ls
git worktree list
```

---

## Troubleshooting

### "CLAUDE_CODE_OAUTH_TOKEN environment variable is not set"

Set the token before running sandbox commands:

```bash
export CLAUDE_CODE_OAUTH_TOKEN='your-token-here'
```

This is validated before any resources are created.

### "No Dockerfile configured"

Either create `.specify/sandbox.json` with a `dockerfile` field, or pass `--template` to use a pre-built image.

### "Docker build failed"

Check that your Dockerfile is valid:

```bash
docker build -f .specify/sandbox.Dockerfile -t test .
```

### "Sandbox creation failed"

Verify Docker Desktop is running and the sandbox plugin is available:

```bash
docker sandbox ls
```

### "Error: --worktree requires a git repository"

The `--worktree` flag (and `sandbox start`) requires a git repository. Initialize one first:

```bash
git init && git add . && git commit -m "Initial commit"
```

### Orphaned sandboxes after worktree cleanup

If you ran `git worktree prune` or manually deleted worktree directories, the associated Docker sandboxes still exist. Clean them up manually:

```bash
# List sandboxes
docker sandbox ls

# Remove orphans
docker sandbox rm claude-my-project-001-old-feature
```
