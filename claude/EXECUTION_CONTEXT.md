# Claude Code Remote Execution Context

This document describes the execution environment used by Anthropic's Claude Code Remote (claude.ai/code) based on runtime analysis performed on December 18, 2025.

## Overview

Claude Code runs inside a **gVisor (runsc)** sandbox with a custom process hierarchy and network egress control.

```
┌────────────────────────────────────────────────────────────────┐
│  gVisor Sandbox (runsc)                                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ process_api (Rust) - PID 1                               │  │
│  │  • WebSocket API :2024                                   │  │
│  │  • Process lifecycle, OOM monitoring                     │  │
│  └──────────────┬───────────────────────────────────────────┘  │
│                 │                                              │
│  ┌──────────────▼───────────────────────────────────────────┐  │
│  │ environment-manager (Go)                                 │  │
│  │  • Session orchestration                                 │  │
│  │  • Claude Code installation/launch                       │  │
│  │  • Embedded MCP server (codesign)                        │  │
│  └──────────────┬───────────────────────────────────────────┘  │
│                 │                                              │
│  ┌──────────────▼───────────────────────────────────────────┐  │
│  │ claude (Node.js)                                         │  │
│  │  • Claude Code CLI                                       │  │
│  │  • Spawns bash for tool execution                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Envoy Proxy Sidecar (21.0.0.183:15004)                   │  │
│  │  • JWT-authenticated egress                              │  │
│  │  • Domain allowlist enforcement                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

## Components

### 1. process_api (PID 1)

**Type**: Rust binary (stripped)
**Location**: `/process_api`
**Size**: ~2MB
**Framework**: Tokio + Hyper + Tungstenite (WebSocket)

**Command line**:
```bash
/process_api --addr 0.0.0.0:2024 \
  --max-ws-buffer-size 32768 \
  --cpu-shares 4096 \
  --oom-polling-period-ms 100 \
  --memory-limit-bytes 17179869184 \
  --block-local-connections
```

**Responsibilities**:
- Container init process
- WebSocket API on port 2024 for process management
- Resource limit enforcement (16GB memory, CPU shares)
- OOM monitoring and handling
- Blocks local network connections

### 2. environment-manager

**Type**: Go binary (with debug symbols)
**Location**: `/usr/local/bin/environment-manager`
**Size**: ~24.6MB
**Version**: `staging-29c451bde`

**Key Go dependencies**:
- `github.com/spf13/cobra` - CLI framework
- `github.com/google/uuid` - UUID generation
- `go.opentelemetry.io/otel` - Observability/tracing
- `github.com/mark3labs/mcp-go/mcp` - Model Context Protocol
- `github.com/cenkalti/backoff/v4` - Retry logic

**Command line**:
```bash
/usr/local/bin/environment-manager task-run \
  --stdin \
  --session <session_id> \
  --session-mode new \
  --upgrade-claude-code=False
```

**Responsibilities**:
- Session orchestration
- Claude Code installation and updates
- Git repository cloning and setup
- **Embedded MCP server** for code signing (port 63298)

### 3. Codesign MCP Server

**Embedded in**: environment-manager
**Port**: 63298 (localhost only)
**Protocol**: MCP (Model Context Protocol)
**Auth**: Bearer token via `CODESIGN_MCP_TOKEN` env var

**Tools exposed**:
```json
{
  "name": "sign_file",
  "description": "Sign a file's contents with SSH-style Ed25519 signature for git commits"
}
```

**Purpose**: Allows Claude Code to sign git commits without exposing the Ed25519 private key to the sandbox. The signing key is held in memory by environment-manager.

### 4. Claude Code CLI

**Type**: Node.js application
**Location**: `/opt/node22/lib/node_modules/@anthropic-ai/claude-code/cli.js`
**Symlink**: `/opt/node22/bin/claude`
**Runtime**: Node.js 22

## Network Architecture

### Container Network

The sandbox uses a **private 21.0.0.0/25 network** (not RFC 1918 private, but internally routed):

| Host | IP | Purpose |
|------|-----|---------|
| Container | 21.0.0.182 | Sandbox |
| Gateway | 21.0.0.183 | Envoy proxy sidecar |

### Egress Control

All outbound traffic is routed through an **Envoy proxy sidecar**:

- **Proxy**: `21.0.0.183:15004`
- **Auth**: JWT Bearer token (ES256 signed)
- **Allowlist**: ~200+ domains (package registries, APIs, etc.)

**JWT payload structure**:
```json
{
  "iss": "anthropic-egress-control",
  "organization_uuid": "<org_id>",
  "session_id": "<session_id>",
  "container_id": "<container_id>",
  "allowed_hosts": "github.com,api.anthropic.com,...",
  "exp": <timestamp>
}
```

### Hosts File

Static DNS entries for critical services:
```
127.0.0.1 localhost
127.0.0.1 runsc
160.79.104.10 api.anthropic.com
160.79.104.10 api-staging.anthropic.com
34.36.57.103 statsig.anthropic.com
...
```

## gVisor Details

**Kernel**: `Linux runsc 4.4.0`
**Runtime**: gVisor (Google's application kernel)

**Characteristics**:
- Userspace network stack (no real kernel netns)
- Synthetic `/proc` filesystem
- No `/sys/class/net` access
- Humorous dmesg messages ("Rewriting the kernel in Rust...")

## Configuration Files

### /root/.claude/settings.json
```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/stop-hook-git-check.sh"
          }
        ]
      }
    ]
  },
  "permissions": {
    "allow": ["Skill"]
  }
}
```

### /root/.gitconfig
```ini
[user]
    name = Claude
    email = noreply@anthropic.com
    signingkey = /home/claude/.ssh/commit_signing_key.pub
[gpg]
    format = ssh
[gpg "ssh"]
    program = /tmp/code-sign
[commit]
    gpgsign = true
[http]
    proxyAuthMethod = basic
```

Note: `/tmp/code-sign` is a symlink to `environment-manager`, which handles signing via its embedded MCP server.

## Environment Variables

### Key Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_REMOTE=true` | Indicates remote execution |
| `CLAUDE_CODE_ENTRYPOINT=remote` | Entry point type |
| `CLAUDE_CODE_VERSION` | CLI version |
| `IS_SANDBOX=yes` | Sandbox indicator |
| `CODESIGN_MCP_PORT` | MCP server port (63298) |
| `CODESIGN_MCP_TOKEN` | MCP auth token |
| `HTTP_PROXY` / `HTTPS_PROXY` | Egress proxy with JWT |

### Proxy Configuration

```bash
HTTP_PROXY=http://container_<id>:jwt_<token>@21.0.0.183:15004
HTTPS_PROXY=http://container_<id>:jwt_<token>@21.0.0.183:15004
NO_PROXY=localhost,127.0.0.1,169.254.169.254,metadata.google.internal,...
```

## Port Summary

| Port | Bind Address | Service | Process |
|------|--------------|---------|---------|
| 2024 | 0.0.0.0 | Container control API | process_api |
| 63298 | 127.0.0.1 | Codesign MCP | environment-manager |
| 15004 | 21.0.0.183 | Egress proxy | Envoy (sidecar) |

## Base Image

**OS**: Ubuntu 24.04.3 LTS (Noble Numbat)

**Pre-installed**:
- Node.js 20, 21, 22
- Python 3.10, 3.11, 3.12, 3.13
- Ruby 3.1.6, 3.2.6, 3.3.6
- Go
- Rust (rustup)
- Java 21 (OpenJDK)
- Maven, Gradle
- Various dev tools (git, curl, jq, etc.)

## Files in This Directory

| File | Description |
|------|-------------|
| `environment-manager.xz` | Session orchestrator + MCP server (Go, xz -9e compressed) |
| `process_api.xz` | Container init process (Rust, xz -9e compressed) |
| `config/settings.json` | Claude Code settings |
| `config/gitconfig` | Git configuration with signing setup |
| `config/stop-hook-git-check.sh` | Pre-stop hook for uncommitted changes |
| `scripts/check-tools` | Tool version checker script |
| `scripts/use-node-22` | Node.js version switcher |
| `scripts/use-python` | Python version switcher |
| `scripts/create-venv-py3.11` | Python venv creator |

## Recreating the Environment

To recreate this environment for self-hosted runners:

1. **Base**: Start with Ubuntu 24.04 container
2. **Runtime**: Install gVisor (runsc) or use standard Docker with seccomp
3. **Init**: Use `process_api` as PID 1 (or create equivalent)
4. **Orchestrator**: Run `environment-manager task-run` with appropriate flags
5. **Network**: Configure egress proxy (Envoy) with JWT auth
6. **MCP**: The codesign MCP server is embedded in environment-manager

### Self-Hosted Mode

environment-manager supports self-hosted mode with additional flags:
```bash
--organization-id <org_id>
--environment-id <env_id>
--local-append-system-prompt <prompt>
--session-mode resume-cached  # default for self-hosted
```

## Security Notes

- **No private keys on disk**: Ed25519 signing key held in memory by environment-manager
- **Egress control**: All outbound traffic authenticated and allowlisted
- **gVisor isolation**: Application-level kernel provides syscall filtering
- **Process isolation**: Claude Code cannot access environment-manager internals
