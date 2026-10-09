# Polyglot-AI

**Status: discontinued.** The author discontinued this project after Anthropic introduced remote control and improved context compaction. The source remains available under the [MIT license](LICENSE); there is no commitment to ongoing maintenance, new features, or future releases.

[![Release](https://github.com/tugcantopaloglu/polyglot-ai/actions/workflows/release.yml/badge.svg)](https://github.com/tugcantopaloglu/polyglot-ai/actions/workflows/release.yml)

Polyglot-AI is a Rust platform that aggregates AI coding assistants into a terminal interface. The repository includes local and client-server binaries, plus a WebSocket bridge for the [historical Expo mobile client](https://github.com/tugcantopaloglu/polyglot-ai-mobile).

The sections below describe the retained implementation and historical setup. Compatibility with current third-party AI tools, native mobile environments, and deployments must be validated separately.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Installation](#installation)
  - [From Source](#from-source)
  - [From Releases](#from-releases)
  - [Docker](#docker)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Usage](#usage)
  - [Local Mode](#local-mode)
  - [Client-Server Mode](#client-server-mode)
  - [Multi-Model Mode](#multi-model-mode)
  - [Mobile Bridge](#mobile-bridge)
- [Supported Tools](#supported-tools)
- [Certificate Setup](#certificate-setup)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Overview

Polyglot-AI provides two modes of operation:

**Local Mode (`polyglot-local`)**: Standalone binary that runs AI tools directly on your machine. No server required. Best for individual developers.

**Client-Server Mode (`polyglot` + `polyglot-server`)**: Centralized server deployment where multiple clients connect over QUIC with mTLS authentication. Supports file synchronization, user management, and shared tool access. Best for teams or remote development.

Both modes share the same features: automatic tool rotation on rate limits, unified chat history, context transfer between sessions, and a terminal UI.

## Features

- **Multi-Tool Support**: Claude Code, Gemini CLI, OpenAI Codex, GitHub Copilot CLI, Cursor Agent, Perplexity, and Ollama
- **Automatic Failover**: When one tool hits a rate limit, automatically switches to the next available tool
- **Context Preservation**: Chat history and context transfer between tools and sessions
- **Multi-Model Queries**: Send the same prompt to multiple AI tools simultaneously and compare responses
- **Terminal UI**: Full-featured TUI with tabs, scrolling, and keyboard shortcuts
- **Simple CLI Mode**: Optional plain CLI mode without the TUI
- **Session History**: Searchable chat history with session resume
- **Plugin System**: Add custom AI tools via CLI commands, scripts, or HTTP APIs
- **File Sync**: Real-time or on-demand file synchronization in client-server mode
- **mTLS Security**: Mutual TLS authentication for client-server communication
- **Self-Updating**: Built-in update checker and installer

## Architecture

```
polyglot-ai/
|-- crates/
|   |-- common/     # Shared types, protocol definitions, utilities
|   |-- local/      # Standalone local binary (polyglot-local)
|   |-- client/     # Client binary for server mode (polyglot)
|   |-- server/     # Server binary (polyglot-server)
|   `-- bridge/     # WebSocket bridge (polyglot-bridge)
|-- config/         # Example configuration files
|-- docker/         # Dockerfiles for containerized deployment
`-- scripts/        # Installation and setup scripts
```


Communication between client and server uses QUIC (UDP-based) with MessagePack serialization for efficiency.

## Installation

### Requirements

- A Rust toolchain compatible with the checked-in `Cargo.lock` (the workspace does not declare a verified minimum Rust version)
- One or more AI CLI tools installed (see [Supported Tools](#supported-tools))
- OpenSSL (for certificate generation)

### From Source

```bash
git clone https://github.com/tugcantopaloglu/polyglot-ai.git
cd polyglot-ai

# Build all binaries
cargo build --locked --release

# Binaries are in target/release/
# - polyglot-local (standalone)
# - polyglot (client)
# - polyglot-server (server)
# - polyglot-bridge (mobile WebSocket bridge)
```

### From Releases

Historical pre-built local, client, and server binaries are available on the [Releases](https://github.com/tugcantopaloglu/polyglot-ai/releases) page. The release workflow does not package `polyglot-bridge`; build that binary from source. Existing releases do not imply continued support.

**Windows one-line install/update**

PowerShell:

```powershell
irm https://raw.githubusercontent.com/tugcantopaloglu/polyglot-ai/main/scripts/install.ps1 | iex
```

cmd.exe:

```cmd
powershell -NoProfile -ExecutionPolicy Bypass -Command "iwr https://raw.githubusercontent.com/tugcantopaloglu/polyglot-ai/main/scripts/install.ps1 -UseBasicParsing | iex"
```

Set `POLYGLOT_WITH_TOOLS=1` to also install AI CLI tools. Add `POLYGLOT_FORCE=1` (or `-Force`) to re-download even if the target version is already installed.

**Linux/macOS one-line install/update**

```bash
curl -fsSL https://raw.githubusercontent.com/tugcantopaloglu/polyglot-ai/main/scripts/install.sh | bash
```

Set `POLYGLOT_WITH_TOOLS=1` to also install AI CLI tools. Add `POLYGLOT_FORCE=1` to re-download even if the target version is already installed. Override `POLYGLOT_BIN_DIR` or `POLYGLOT_INSTALL_DIR` to change destinations.

Available binaries:

- `polyglot-local-x86_64-unknown-linux-gnu`
- `polyglot-local-aarch64-unknown-linux-gnu`
- `polyglot-local-x86_64-apple-darwin`
- `polyglot-local-aarch64-apple-darwin`
- `polyglot-local-x86_64-pc-windows-msvc.exe`

### Docker

```bash
# Server
docker build -f docker/Dockerfile.server -t polyglot-server .
docker run -p 4433:4433/udp -v ./certs:/app/certs -v ./data:/app/data polyglot-server

# Client
docker build -f docker/Dockerfile.client -t polyglot-client .
docker run -it -v ./certs:/app/certs -v ./workspace:/app/workspace polyglot-client
```

Or use Docker Compose:

The Compose file mounts certificates from `docker/certs/` and configuration from `docker/config/server.toml` (and `client.toml` for the optional client). Create those files from the certificate setup and configuration examples before starting it.

```bash
cd docker
docker-compose up -d
```

## Quick Start

### Local Mode (Recommended for Individual Use)

```bash
# Check which AI tools are available on your system
polyglot-local doctor

# Start the interactive TUI
polyglot-local

# Or run without TUI
polyglot-local --no-tui

# One-shot query
polyglot-local ask "explain this code" --tool claude
```

### Client-Server Mode

```bash
# From the repository root, copy and customize the example configuration
cp config/server.example.toml config/server.toml
cp config/client.example.toml config/client.toml

# 1. Generate certificates
./scripts/generate-certs.sh ./certs

# 2. Start the server
polyglot-server start -c config/server.toml

# 3. Connect with client
polyglot connect -c config/client.toml
```

### Mobile Bridge

Expo clients cannot speak QUIC directly. Use the WebSocket bridge to connect mobile apps to a Polyglot server:

Follow the [bridge setup and transport notes](crates/bridge/README.md) for certificates, access tokens, and local mode. The [mobile README](https://github.com/tugcantopaloglu/polyglot-ai-mobile/blob/main/README.md) explains connection profiles and client limitations.

```bash
# Build the bridge
cargo build --locked -p polyglot-bridge --release

# Start the bridge (WS on 8787, QUIC to server on 4433)
target/release/polyglot-bridge --listen 0.0.0.0:8787 --server 127.0.0.1:4433
```

Server mode requires a running Polyglot server and its CA, client certificate, and client key in `./certs/`, or explicit `--ca`, `--cert`, and `--key` paths. Point the mobile app at `ws://<host>:8787/ws`, or use `wss://...` with the bridge's `--tls` option or a TLS proxy. Set `--token` to require a shared access token; it does not encrypt a plain WebSocket connection.

Local-only mode (runs `polyglot-local` on the same machine):

```bash
target/release/polyglot-bridge --listen 0.0.0.0:8787 --mode local --local-bin polyglot-local
```

## Configuration

### Local Mode Configuration

Default location: `~/.config/polyglot-ai/local.toml`

Generate a config file:

```bash
polyglot-local init
```

Example configuration:

```toml
[tools]
default_tool = "claude"
rotation_strategy = "on_limit"  # or "round_robin", "priority"
switch_delay = 3

[tools.claude]
enabled = true
path = "claude"
args = []

[tools.gemini]
enabled = true
path = "gemini"
args = []

[tools.copilot]
enabled = true
path = "gh"
args = ["copilot"]

[tools.ollama]
enabled = true
path = "ollama"
args = ["run", "codellama"]

[ui]
tui_enabled = true
show_timestamps = true
theme = "default"
```

### Server Configuration

See [config/server.example.toml](config/server.example.toml) for all options.

Key settings:

```toml
[server]
bind_address = "0.0.0.0:4433"
max_connections = 100

[auth]
mode = "single_user"  # or "multi_user"
cert_path = "./certs/server.crt"
key_path = "./certs/server.key"
ca_path = "./certs/ca.crt"

[tools]
rotation_strategy = "on_limit"
default_tool = "claude"
```

### Client Configuration

See [config/client.example.toml](config/client.example.toml) for all options.

```toml
[connection]
server_address = "localhost:4433"
cert_path = "./certs/client.crt"
key_path = "./certs/client.key"
ca_path = "./certs/ca.crt"

[sync]
default_mode = "on_demand"
ignore_patterns = [".git", "node_modules", "target"]
```

## Usage

### Local Mode

**Keyboard Shortcuts:**

| Key         | Action                           |
| ----------- | -------------------------------- |
| F1          | Chat view                        |
| F2          | Tools view                       |
| F3          | Usage statistics                 |
| F4          | Chat history                     |
| F5          | Help                             |
| F6          | Multi-model selection            |
| F7          | About                            |
| Ctrl+M      | Toggle multi-model mode          |
| Ctrl+N      | New chat (with context transfer) |
| Ctrl+Q      | Quit                             |
| PageUp/Down | Scroll                           |

**Commands:**

```
/tools          List available tools
/switch <tool>  Switch to a specific tool (claude, gemini, codex, copilot, cursor, ollama)
/usage          Show usage statistics
/history        Show chat history
/search <query> Search chat history
/new            Start new chat with context transfer
/multi          Open multi-model selection
/single         Return to single-tool mode
/clear          Clear chat output
/update         Check for updates
/help           Show help
/quit           Exit
```

### Client-Server Mode

Same keyboard shortcuts and commands as local mode, plus:

```
/sync [path]    Sync files with server
```

### Multi-Model Mode

Query multiple AI tools simultaneously:

1. Press F6 or type `/multi`
2. Select tools with number keys (1-7)
3. Press Enter to confirm
4. Send your prompt - all selected tools respond in parallel
5. Responses display side-by-side
6. Type `/single` to return to normal mode

Or specify tools directly:

```
/multi claude gemini codex
```

## Supported Tools

These are the historical commands targeted by the adapters. Their presence does not establish compatibility with current vendor CLIs.

| Tool           | Historical command |
| -------------- | ------------------ |
| Claude Code    | `claude`           |
| Gemini CLI     | `gemini`           |
| OpenAI Codex   | `codex`            |
| GitHub Copilot | `gh copilot`       |
| Cursor Agent   | `cursor-agent`     |
| Perplexity     | `pplx`             |
| Ollama         | `ollama`           |

Historical tool installation scripts are retained below. Review their package names, versions, and commands against each vendor's current documentation before executing them.

```bash
# Linux/macOS
./scripts/install-tools.sh

# Windows (PowerShell)
.\scripts\install-tools.ps1
```

Verify installed tools:

```bash
polyglot-local doctor
```

## Certificate Setup

Client-server mode requires mTLS certificates.

### Using the Script

```bash
./scripts/generate-certs.sh ./certs
```

This generates:

- `ca.crt` / `ca.key` - Certificate Authority
- `server.crt` / `server.key` - Server certificate
- `client.crt` / `client.key` - Client certificate

### Using the CLI

```bash
# Generate CA and server certs
polyglot-server generate-certs -o ./certs

# Generate additional client certs
polyglot generate-certs -o ./client-certs --cn "client-2" --ca-cert ./certs/ca.crt --ca-key ./certs/ca.key
```

### Manual Setup with OpenSSL

See `scripts/generate-certs.sh` for the exact OpenSSL commands.

## Troubleshooting

### Double characters when typing (Windows)

This was a known issue with terminal echo on Windows. Compare your checkout or historical release with the console mode handling in `crates/local/src/main.rs`; no future fix release is promised.

### Tool not found

1. Check if the tool is installed: `which claude` (or the tool name)
2. Run `polyglot-local doctor` to see tool status
3. Verify the path in your config matches the actual binary location

### Connection refused (client-server mode)

1. Verify the server is running: `polyglot-server start`
2. Check firewall allows UDP port 4433
3. Verify certificates are correctly configured
4. Check the CA certificate matches between client and server

### Rate limit errors

Polyglot-AI automatically handles rate limits by switching tools. If all tools are rate limited:

1. Wait a few minutes for limits to reset
2. Add more tools to your configuration
3. Consider using Ollama for unlimited local inference

### Build errors

```bash
# Clean build
cargo clean
cargo build --locked --release
```

Record the compiler and dependency error before changing the lockfile. This discontinued project does not provide a verified minimum Rust version or a currently tested build matrix.

## Plugin System

Add custom AI tools without modifying code. Add to your config file:

```toml
# CLI tool
[[plugins]]
name = "my-tool"
display_name = "My Custom Tool"
plugin_type = "cli"
enabled = true
command = "/path/to/my-tool"
args = ["--prompt", "{prompt}"]
timeout = 120

# HTTP API
[[plugins]]
name = "my-api"
display_name = "My API"
plugin_type = "http"
enabled = true
command = "https://api.example.com/chat"
http_method = "POST"
headers = { "Authorization" = "Bearer TOKEN" }
body_template = '{"prompt": "{prompt}"}'
timeout = 60
```

## Updates

The updater queries existing GitHub releases. It does not indicate that new releases will be published.

Check for updates:

```bash
polyglot-local update --check-only
```

Install updates:

```bash
polyglot-local update
```

## License

MIT License. See [LICENSE](LICENSE) for details.

## Author

Tugcan Topaloglu ([@tugcantopaloglu](https://github.com/tugcantopaloglu))

## Contributing

The [issue tracker and pull requests](https://github.com/tugcantopaloglu/polyglot-ai) remain available, but review and maintenance are not guaranteed for this discontinued project.


