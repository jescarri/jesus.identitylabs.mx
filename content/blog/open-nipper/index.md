---
title: "Open-Nipper: Building a Local-AI Native Agent Gateway in Go"
date: 2026-03-29
draft: false
tags: ["ai", "golang", "localai", "kubernetes", "mcp", "whatsapp", "agents", "rabbitmq"]
categories: ["Projects"]
summary: "Why I rewrote an AI agent gateway from scratch in Go - fighting context window limits, taming MCP tool bloat, and building something that actually fits in my homelab."
---

There's a particular kind of itch that only gets scratched by building the thing yourself. You hit a tool that _almost_ works. You fork it, patch it, fight its dependency tree, and realize that you've just adopted a second full-time job. At some point the sane decision is to open a new repository.

That's how **Open-Nipper** started.

{{< figure src="featured.png" alt="open-nipper" href="featured.png" target="_blank" nozoom=true >}}

---

## The Backstory: Why Not Just Use OpenClaw?

[OpenClaw](https://github.com/open-claw/openclaw) is impressive. It's polished, has a big feature set, and a large community. If you need a batteries-included AI agent platform and you're okay with running a Node.js + Python stack in a cloud environment, it might be exactly what you want.

It wasn't what I wanted. Here's why:

**The dependency tree.** OpenClaw brings along a _lot_. Node.js, multiple Python packages, various runtimes, npm modules. The surface area for "something broke" is enormous. I want to deploy an agent on my Kubernetes homelab and have it _stay deployed_.

**Memory footprint.** My cluster doesn't have unlimited RAM. A Go binary that idles at ~25MB is a lot more interesting to me than a Node process that considers 200MB a reasonable starting point.

**Context burn.** I tested OpenClaw against my local GPTOSS-120B model and it kept torching the context window. Every request arrived pre-loaded with the full catalog of MCP tools, system prompt boilerplate, and conversation history. My model, running on consumer hardware, was burning tokens on plumbing rather than thinking. The "dumb zone" (the region near the context limit where the model starts hallucinating and producing garbled output) arrived _way_ earlier than I expected.

More dependencies means more attack surface. I wanted to audit exactly what was running, not inherit the trust chain of a hundred transitive packages.

The real reason: I wanted to learn. I wanted to understand what an AI agent actually _is_ at the plumbing level - how tool calls work, how context is managed, how MCP servers connect. The only way to really understand it is to build it.

When I started, there were no serious Go alternatives. A few appeared a couple of weeks after my initial working version - which is validation enough to keep going. I had momentum, a working system, and a much clearer picture of what I actually needed.

---

## The Stack

Go, obviously. Single statically-linked binary for both the gateway and the agent. No runtime, no interpreter, no "did you run `npm install`?" ceremonies.

**[Eino SDK](https://github.com/cloudwego/eino)**, ByteDance's Go AI SDK, powers the agentic loop. It implements the ReAct pattern (`ChatModel -> Tools -> ChatModel` until complete) and provides adapters for OpenAI-compatible APIs, Ollama, MCP clients (both SSE and STDIO), and sandboxed tool execution. Eino is relatively young but it's well-structured and it let me wire together a real ReAct agent without building the loop from scratch.

**RabbitMQ** as the message bus between the Gateway and Agent processes. More on the complexity this introduced later.

**SQLite** for the Gateway's user/identity/allowlist datastore (lightweight, embedded, no server required).

**[Wuzapi](https://github.com/asternic/wuzapi)** for WhatsApp. Wuzapi provides a clean REST API over the WhatsApp Web protocol. The gateway registers as its webhook target and receives message events over HTTP.

**OpenTelemetry** for tracing and Prometheus metrics. Because if you can't observe it, you can't debug it, and debugging a local LLM agent that occasionally hallucinates its tool calls requires serious observability.

---

## Architecture: The Gateway / Agent Split

The most important architectural decision in Open-Nipper is the hard separation between the **Gateway** and the **Agent**.

```
Channels                 Gateway                    Agents
---------           ------------------           ----------
WhatsApp  --+       | Webhook/Adapter |       +-- Agent A (alice)
Slack     --+       | -> Router       |       +-- Agent B (bob)
MQTT      --+------>| -> RabbitMQ pub |------>+-- Agent C (carol)
RabbitMQ  --+       |                 |       |
Cron      --+       | Event consumer  |<------+
                    | -> Dispatcher   |
                    | -> Adapter      |------> Response to user
                    +-----------------+
```

**The Gateway is dumb by design.** It receives a WhatsApp message, validates the HMAC signature, resolves the user from the sender's phone number, checks the allowlist, normalizes the payload into a `NipperMessage`, and drops it onto a RabbitMQ exchange with a routing key based on the user ID. That's it. It doesn't know anything about AI, tools, or conversation state.

**The Agent is where everything interesting happens.** Each user gets their own queue (`nipper-agent-{userId}`). The agent process sits on that queue with `prefetch: 1`, consuming one message at a time. It loads the session transcript, runs the ReAct loop, calls tools, and publishes response events back to the Gateway. The Gateway picks those up and delivers them through whatever channel the message originally arrived on.

The Gateway and Agent are independently deployable, independently scalable, and independently restartable. If the agent crashes, messages accumulate in the queue and get processed when it comes back. If the Gateway restarts, the agent keeps consuming from RabbitMQ without missing a beat. If I want to point Alice at a different LLM tomorrow, I stop her agent, update the config, restart it. The Gateway never knew.

This is exactly what I needed for Kubernetes: the Gateway is just an API pod. The Agents are just consumer pods. Independent rollouts, clean topology.

**Agents are also polyglot.** The protocol is AMQP + JSON. If you want to write an agent in Python, TypeScript, or Rust - as long as it speaks AMQP and conforms to the `NipperMessage`/`NipperEvent` schema, it works. The Go agent is the reference implementation, but it's not the only possible one.

---

## The Hard Part: Fighting the Context Window

Running local models on your own hardware teaches you things the benchmarks don't mention.

Both GPTOSS-120B and Qwen3 advertise large context windows, and on paper, they're right. GPTOSS-120B has a 131K token training context and I run it at Q8_K_XL quantization, which preserves quality well. Qwen3 is similarly capable on paper. But there's a gap between what a model _technically_ supports and what it _reliably performs well within_ when serving a model on local consumer hardware, under real load, with a context already packed with tool schemas.

The practical effective window I was actually working with looked more like this, once you account for what's consuming that context:

- System prompt (safety preamble, formatting directives, instructions)
- MCP tool schemas (every tool you might ever want to call)
- Conversation history (every message in the current session)
- The current user message

With a typical home assistant setup - Home Assistant (20 tools), Joplin (13 tools), Google Workspace (10 tools) - the tool schemas alone burned **~12,000 tokens** before the first word of conversation history. A system prompt with detailed instructions added another ~2,000. By the time you got to a 5-turn conversation, you were already putting real pressure on quality.

And the models didn't handle high context fill gracefully in practice. They entered what I started calling the **Dumb Zone**: garbled output, malformed JSON tool calls, special tokens bleeding into responses (`<|channel|>`, `<|message|>` chat template artifacts), hallucinated function signatures. Quality degraded significantly before hitting the hard limit.

Debugging this was genuinely painful. The failures weren't consistent. Sometimes a request worked fine. Sometimes the agent would retry a tool call 8 times because it kept generating invalid JSON. Sometimes it would respond in the wrong language. The only signal was logs, and you had to instrument _everything_ to understand what was happening.

The fix required attacking the problem on multiple fronts.

### Lean MCP Tools

The biggest lever was **not binding all tools upfront**. Instead of loading the entire MCP catalog at startup and dumping all tool schemas into every request, Open-Nipper resolves tools on-demand per message.

```
User: "check my plants"
        |
        v
  1. Skill keyword matching: "plant" -> activates plant-care skill
     plant-care declares mcp_tools: [GetLiveContext]
        |
        v
  2. MCP tool keyword matching: "check" against tool descriptions
     -> no additional matches
        |
        v
  3. Agent binds: 13 native tools + GetLiveContext = 14 tools
     (instead of 13 native + 43 MCP = 56 tools)
        |
        v
  4. LLM runs with ~3,500 tokens of tool schemas
     (instead of ~12,000 tokens)
```

When the message is "tell me a joke," zero MCP tools are bound. Only the 13 native tools (bash, web search, web fetch, weather, datetime, skill_exec, search_tools, etc.) are included. The savings are significant:

| Scenario | Tool schema tokens |
|---|---|
| Legacy: all tools always bound | ~12,000 |
| Lean mode: keyword-matched only | ~3,500 |
| No match (simple requests) | ~800 |

The `ToolMatcher` interface is pluggable. The default `KeywordToolMatcher` splits tool names and descriptions into tokens, scores each tool against the user's message, and returns the top matches. It's fast, has no external dependencies, and works well for most cases.

For multilingual or semantic matching: "enciende la luz" should match `HassTurnOn: Turns on a device` even with zero keyword overlap. There's an `EmbeddingToolMatcher` that uses cosine similarity against tool description embeddings. A `HybridToolMatcher` composes both via reciprocal rank fusion:

```go
// HybridToolMatcher blends keyword and embedding scores using
// Reciprocal Rank Fusion: score = sum(1/(k + rank_i))
```

"Enciende la luz" hits the embedding matcher at 0.82 similarity and binds the right tools, even though there's no keyword overlap. Cross-language tool discovery, zero code changes needed to add new languages.

The `search_tools` native tool is the fallback path for the LLM itself: if the model needs a tool that wasn't pre-loaded, it can call `search_tools` with a natural language intent and get back matching tool names to request dynamically.

{{< figure src="tracing.png" alt="Agent Latency for turning off a light" href="tracing.png" target="_blank" nozoom=true >}}

### Skills: Config-Driven Prompt Injection

Skills are self-contained capability packages. Each skill lives in a directory with two files:

- `SKILL.md` - detailed instructions for the LLM (only loaded when needed)
- `config.yaml` - keywords, MCP tool dependencies, compact prompt hint

When the user's message matches a skill's keywords, two things happen:
1. The skill's MCP tool dependencies get added to the bound tool set
2. A compact `prompt_hint` (a few lines, not the full SKILL.md) gets injected into the system prompt

```yaml
# skills/plant-care/config.yaml
name: "plant-care"
keywords: ["plant", "soil", "moisture", "water", "garden"]
mcp_tools: ["GetLiveContext"]
prompt_hint: |
  Check soil moisture via Home Assistant MCP tools.
  Steps: 1) GetLiveContext 2) get_weather 3) Sensor <= 30 = needs water.
```

Adding a new skill requires zero code changes. Create the directory, write the YAML and the markdown, deploy. The keyword matching picks it up automatically on the next startup.

Before this, my system prompt was loading the full SKILL.md for every skill on every request - about 2,500 characters per skill. The compact `prompt_hint` brings that down to ~400 characters. For a set of 8 skills, that's the difference between ~20K characters and ~3,200 characters of skill context injected per request.

### System Prompt Surgery

The system prompt itself got a ~70% reduction through careful surgery:

- WhatsApp formatting directive: 28 lines -> 3 lines (a post-processor catches any Markdown that slips through delivery)
- Safety preamble: rewritten to be terse and active rather than verbose and passive
- Skill catalog: shows 1-line summaries when inactive, full `prompt_hint` only when keyword-matched

Every saved token is headroom before the Dumb Zone.

---

## Voice Messages from WhatsApp

Voice messages work, and the pipeline is simpler than you'd expect. When a WhatsApp voice message arrives, the Gateway receives a media attachment reference rather than text. Before the message hits the agent, an enrichment pipeline runs:

1. Download the audio from Wuzapi
2. Run it through a speech recognition model (OpenAI Whisper-compatible endpoint)
3. Substitute the transcription as the message text
4. Pass to the agent as a normal text message

From the agent's perspective, it never knows the message started as audio. From the user's perspective, they can send a voice note and get a response just as if they'd typed it. On a busy morning commute, that's useful.

The enrichment pipeline is composable. Audio transcription is one stage, but the same pipeline structure handles image description, link preview extraction, and future preprocessing steps.

---

## The Sandbox: Docker-out-of-Docker

One of the harder things to get right was the execution sandbox. The agent can run arbitrary shell commands. That's the point: install packages, run network scans, query APIs, do whatever the user asks. The constraint is that it should do this _inside an isolated container_, not on the host.

The solution is Docker-out-of-Docker (DooD):

```
+---------------------------------------------+
|  Host                                       |
|                                             |
|  docker daemon                              |
|    +-- agent container (has docker CLI)     |
|    |     +-- /var/run/docker.sock (mount)   |
|    +-- nipper-sandbox-xxxx                  |
|    +-- stdio-mcp-yyyy                       |
+---------------------------------------------+
```

The agent container ships with only the Docker CLI. The host's socket is bind-mounted in. When the agent needs to execute something, it calls `docker run` through the CLI, which talks to the host daemon via the socket. The sandbox container is a sibling on the host daemon, not a nested container. Simpler, lighter, no DinD daemon needed.

Sandbox containers get `--cap-drop ALL`, read-only root FS, memory and CPU limits, and `--no-new-privileges`. For scans that need raw sockets (nmap OS detection, ping sweeps), the sandbox config accepts an `extra_capabilities` list to grant `NET_RAW` selectively.

There's a subtle gotcha with DooD and skills: because sandbox containers are siblings resolved by the host daemon, any volume mount paths in the agent config must be paths that exist _on the host_, not inside the agent container. I burned a few hours on this before understanding what was happening. The workaround is mounting skills at the same absolute path on both the host and inside the agent container.

---

## What I Would Do Differently

**RabbitMQ was a mistake.** Or rather, it works, it's efficient, and the topology is elegant. But it's _yet another system I have to run on the cluster_. RabbitMQ needs persistent storage, health monitoring, connection management, and occasional restarts. Every time I want to spin up the stack on a fresh node, RabbitMQ is the annoying third wheel.

If I were starting today, I'd replace RabbitMQ with WebSocket channels for agent communication. The Gateway would maintain persistent WebSocket connections to registered agents. The protocol would be the same (send `NipperMessage`, receive `NipperEvent`), but the broker would be internal to the Gateway process. Less infrastructure, simpler deployment, same decoupling.

The trade-off is losing RabbitMQ's message durability and queue backpressure out of the box. For my use case (a personal assistant with 1-3 users), that's an acceptable trade. For a multi-tenant deployment serving thousands of users, RabbitMQ's guarantees matter more.

---

## What's Next

**Kubernetes pod sandbox.** Right now the sandbox is a Docker container via DooD. The next step is running the sandbox as a Kubernetes Pod, so the entire stack (gateway, agents, sandboxes) lives natively on the cluster. No Docker socket mount, no DooD gymnastics, just a clean Pod spec with proper resource quotas and network policies.

**Voice generation.** Speech recognition is in. Synthesizing voice responses is the natural next step. When the user sends a voice message, the agent should optionally respond with one. I want to explore voice cloning to give the agent a consistent voice identity.

Replace RabbitMQ with internal WebSocket channels: reduce the operational footprint from 3 processes to 2, sessions currently in local files move to pluggable storage (SQLite for local, Postgres or Redis for clustered deployments).

---

## Closing Thoughts

Open-Nipper isn't the most feature-rich AI agent platform. It's a tool built precisely to the shape of my actual needs: runs on my homelab, speaks to my phone over WhatsApp, uses my local models without burning their context windows, deploys cleanly on Kubernetes. Building it taught me more about AI agents than reading ever would - _why_ local models degrade near the context limit, _how_ tool schemas consume tokens, _what_ the ReAct loop actually looks like in code. These are things you only learn by instrumenting a live system and watching it behave badly.

The Lean MCP Tools system is the piece I'm most proud of. Reducing tool schema overhead by 65% while preserving full semantic and cross-language matching capability, and making it entirely config-driven with no code changes per new skill or tool - that's the kind of solution that comes from running the system, feeling the pain, and solving the actual problem.

If you're building with local AI and fighting the same context window battles, the code is at [github.com/jescarri/open-nipper](https://github.com/jescarri/open-nipper).

Go eat their tokens. Or rather, don't.
