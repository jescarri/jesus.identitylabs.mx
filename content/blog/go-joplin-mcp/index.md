---
title: "go-joplin: MCP, RAG Search, and a Month of Changes"
date: 2026-03-23T22:00:00-07:00
draft: false
tags: ["ai", "mcp", "joplin", "golang", "rag"]
categories: ["Projects"]
summary: "A month after the initial release: go-joplin now ships a full MCP server, semantic RAG search via sqlite-vec, a mutation allow-list for safe agent writes, and smaller list responses."
---

A month ago I wrote about [go-joplin](https://github.com/jescarri/go-joplin): a Go-based Joplin Web Clipper server built to let AI agents talk to Joplin over HTTP. The original post ended with "a follow-up write-up on the MCP and agent integrations is in the works."

That work shipped. Here's what changed.

{{< figure src="featured.png" alt="go-joplin" href="featured.png" target="_blank" nozoom=true >}}

## MCP is live

The main promise from the first post: a Model Context Protocol server so AI agents can read and write Joplin notes without stdio. It's there now.

The MCP server runs as an SSE endpoint at `/mcp` on the same port as the clipper, protected by Bearer token auth. It registers 13 tools out of the box:

`list_notes`, `get_note`, `create_note`, `update_note`, `search_notes`, `list_folders`, `get_folder`, `create_folder`, `list_tags`, `get_note_tags`, `list_resources`, `trigger_sync`, `get_capabilities`

Any client that speaks MCP over SSE (Claude Desktop, cursor, custom agents) can connect to it directly.

One practical addition: an SSE heartbeat that sends keep-alive frames on a timer. Without it, reverse proxies and load balancers kill idle SSE connections after 30-60 seconds, which breaks long agent sessions silently.

## Semantic search with sqlite-vec

`search_notes` now hits a vector index built on [sqlite-vec](https://github.com/asg017/sqlite-vec).

By default, `search_notes` still uses Joplin's FTS4 keyword search. When you set `GOJOPLIN_RAG_ENABLED=true`, it switches to the vector index and requires an OpenAI-compatible embedding endpoint - LocalAI, ollama, vLLM, or OpenAI itself. The index is stored locally in sqlite-vec; only the embedding calls go to whichever endpoint you configure.

Keyword search on notes is frustrating. You write a note about "Kubernetes node pressure eviction" and later search for "pod OOMKilled troubleshooting". Those don't overlap lexically, but they're the same problem. Vector search finds them.

IndexAll runs at startup with configurable worker concurrency and logs progress so you can see it working rather than guessing whether the process hung.

## Token efficiency

Two changes cut LLM context use significantly for agent workflows.

**Slim list responses.** List endpoints previously returned full note objects including body text, encryption metadata, sync fields, and timestamps. Now:

- `list_notes` and `search_notes` return `{id, title, parent_id, is_todo, updated_time}`. No body.
- `list_folders`, `list_tags`, and `list_resources` follow the same pattern - ID, title, and type-specific fields only.

Roughly 70-80% smaller payloads on list calls. Use `get_note` when you need content.

**Tool filtering.** Not every agent needs all 13 tools. A note-capture agent only needs `create_note` and `list_folders`. Register only what the agent uses:

```bash
GOJOPLIN_MCP_ENABLED_TOOLS=create_note,list_folders,list_tags
```

Fewer tools in the context window means fewer tokens spent on tool descriptions on every call.

## Mutation allow-list

By default, all write operations through MCP are denied. Read access works; writes to notes, folders, and tags require explicit opt-in.

```bash
GOJOPLIN_MCP_ALLOW_FOLDERS=work,inbox   # only these folders are writable
GOJOPLIN_MCP_ALLOW_TAGS=ai-generated    # only this tag can be applied
GOJOPLIN_MCP_ALLOW_CREATE_TAG=false     # agents cannot create new tags
GOJOPLIN_MCP_ALLOW_CREATE_FOLDER=false  # agents cannot create new folders
```

Use `*` to allow all writes for a resource type. The `get_capabilities` tool (and the `joplingo://capabilities` MCP resource) expose the current policy to the agent at runtime, so it knows what it can and cannot do before attempting a write.

This is the thing I wanted most before connecting an agent with write access to my notes: a fence with explicit holes rather than full trust by default.

## YAML config

```yaml
# config.yaml.example
sync:
  target: 8   # S3
  "8":
    path: my-bucket
    url: https://s3.us-east-1.amazonaws.com
    region: us-east-1
api:
  token: "${GOJOPLIN_API_TOKEN}"
```

Native YAML config for deployments that have no Joplin desktop - containers, VMs, CI. Secrets stay in environment variables; the YAML holds structure. Pass `--config config.yaml` or set `GOJOPLIN_CONFIG_PATH`. If the path ends in `.yaml` or `.yml`, the YAML loader runs; otherwise it expects Joplin's JSON format.

## Observability

OpenTelemetry tracing and Prometheus metrics ship on a separate port (9091 by default). The `http_request_duration_seconds` histogram breaks down by method and route. Sync backends and E2EE operations are traced. Example recording rules for p50/p95/p99 latency are in `docs/prometheus-recording-rules.yaml`.

---

The follow-up MCP work is done, the agent integration is usable, and semantic search over your notes is now local and free. What's left is better embedding model options and tighter Joplin Server auth handling. That's the next milestone.

**Repository:** [github.com/jescarri/go-joplin](https://github.com/jescarri/go-joplin/tree/main)
