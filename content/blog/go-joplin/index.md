---
title: "go-joplin: A Joplin Web Clipper Server in Go"
date: 2026-02-26T00:00:00-08:00
draft: false
tags: ["ai", "mcp", "joplin", "golang"]
categories: ["Projects"]
summary: "A lightweight, statically compiled Joplin Web Clipper server in Go—built to enable network-based MCP and AI agents without Node.js or stdio-only tooling."
---

I’ve been working on **go-joplin**: a Joplin Web Clipper server implementation in Go. It runs a local HTTP server that the Joplin Web Clipper browser extension talks to, and syncs your notes with either **Joplin Server** or an **S3-compatible bucket** (AWS S3, MinIO, etc.).

{{< figure src="featured.png" alt="go-joplin" href="featured.png" target="_blank" nozoom=true >}}

## Why I built it

Two main motivations drove this:

**1. MCP and AI agents.** I want an MCP (Model Context Protocol) so AI agents can work with my Joplin notes. Existing Joplin MCPs assume stdio and often require running Joplin or joplin-cli. I want something **lightweight and network-based**: an HTTP server that any agent (local or remote) can call, without tying the integration to a single process or stdio. go-joplin is that server—a small, focused service that exposes the clipper API over HTTP. A follow-up post will cover the MCP and agent integration built on top of it.

**2. Fewer runtime dependencies.** I prefer **statically compiled binaries** with no runtime stack to install. Reducing Node.js (and similar) dependencies on my machine is a goal. Go gives a single binary, no npm/node, and easy deployment. Same config story as the Joplin desktop app (it reads your existing Joplin config), so no extra ecosystem.

## What it does

- **Web Clipper API**: Notes, folders, tags, resources, search, and events endpoints compatible with the Joplin clipper.
- **Sync targets**:
  - **Joplin Server** (sync target 9): sync over HTTP with a Joplin Server instance.
  - **S3** (sync target 8): sync with any S3-compatible storage (AWS S3, MinIO, Backblaze B2, etc.), using the same object layout as Joplin’s built-in S3 sync.
- **End-to-end encryption**: Uses existing Joplin E2EE; no changes to the crypto model.

## Requirements and build

- Go 1.24 or later.
- For **Joplin Server**: server URL, username, password, and API token from your Joplin config.
- For **S3**: bucket name, region, endpoint URL (and optional force path style for MinIO). Credentials via environment variables (e.g. `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`).

Build from the project root:

```bash
go build -o go-joplin .
```

Or install into `$HOME/go/bin`:

```bash
go install .
```

## Configuration

go-joplin reads the Joplin desktop config (e.g. `~/.config/joplin-desktop/settings.json`). Override the path with `GOJOPLIN_CONFIG_PATH`.

- **Joplin Server**: In Joplin, set sync target to Joplin Server and configure `sync.9.path`, `sync.9.username`, `sync.9.password`. The clipper also needs `api.token` from Joplin’s Web Clipper auth.
- **S3**: In Joplin, set sync target to S3 and set `sync.8.path` (bucket), `sync.8.url` (endpoint), `sync.8.region`, and optionally `sync.8.forcePathStyle` for MinIO. S3 credentials can come from the Joplin config or from `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` (env overrides).

## Usage

**Serve (clipper + background sync):**

```bash
export AWS_ACCESS_KEY_ID=your_key      # when using S3
export AWS_SECRET_ACCESS_KEY=your_secret
./go-joplin serve --api-key YOUR_JOPLIN_API_KEY
```

The server listens on `localhost:41184` by default (`GOJOPLIN_PORT` or `--port` to override).

**One-shot sync (no HTTP server):**

```bash
./go-joplin sync
```

**Print resolved config:**

```bash
./go-joplin config
```

## Project layout

- `cmd/`: CLI commands (`serve`, `sync`, `config`, etc.)
- `internal/config`: Load and validate config from Joplin settings.
- `internal/store`: SQLite store (notes, folders, tags, resources, sync state).
- `internal/sync`: Sync engine; HTTP client for Joplin Server and S3 backend.
- `internal/s3`: S3 client (AWS SDK v2) for official S3 and S3-compatible endpoints.
- `internal/clipper`: Web Clipper HTTP API and handlers.
- `internal/e2ee`, `internal/models`: E2EE and shared models (crypto unchanged).

---

If you’re interested in a minimal, network-friendly way to run the Joplin clipper and sync—or in wiring AI agents to your notes—go-joplin is a step in that direction. A follow-up write-up on the MCP and agent integrations is in the works.

**Repository:** [github.com/jescarri/go-joplin](https://github.com/jescarri/go-joplin/tree/main)
