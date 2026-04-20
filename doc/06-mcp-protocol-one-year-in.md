---
title: "The Model Context Protocol (MCP): One Year In — What Works, What Breaks"
slug: "mcp-protocol-one-year-in"
target_query: "model context protocol"
secondary_queries:
  - "MCP server tutorial"
  - "MCP Cursor integration"
  - "MCP vs LSP"
canonical_url: "https://agentbell.dev/blog/mcp-protocol-one-year-in"
author: "AgentBell Team"
date: "2026-05-05"
tags: ["mcp", "cursor", "vscode", "ai", "integration", "devtools"]
disclosure: "We ship @agentbell/mcp-server and integrate MCP across multiple IDEs. This is field notes, not the official spec."
---

# The Model Context Protocol (MCP): One Year In — What Works, What Breaks

The **Model Context Protocol (MCP)** is Anthropic's open standard for connecting AI assistants to **tools, data, and UI surfaces** — think "USB-C for agent integrations" rather than a single vendor API.

By early 2026, MCP has moved from **interesting spec** to **default integration path** for several major coding assistants. This article is **not** the spec — it is **field notes** from shipping multiple integrations: what is solid, what is fuzzy, and what I wish the ecosystem agreed on next.

> **Disclosure**: We maintain [`@agentbell/mcp-server`](https://www.npmjs.com/package/@agentbell/mcp-server) and wire MCP into Cursor, Windsurf, VS Code, and friends. Biases: we want MCP to succeed; we also hit real interoperability bugs.

---

## What MCP is (in one paragraph)

MCP separates concerns:

- **Host** — the IDE or client that runs the model (Cursor, Claude Desktop, VS Code extension, etc.).
- **Server** — a process that exposes **tools**, **resources**, and **prompts** over a structured protocol.
- **Transport** — how bytes move: classically **stdio** (spawn server, talk over pipes) or **HTTP/SSE** for remote servers.

The assistant decides when to call a tool; the server executes and returns structured results. **Security and policy** live at the host boundary — the server should not be a free pass to run arbitrary code without user consent.

---

## What went right (why MCP stuck)

### 1. One integration surface for "agent tools"

Before MCP, every IDE invented its own plugin shape. MCP gives **one mental model**: implement a server, register it in config, expose tools with JSON Schema.

### 2. JSON Schema for tools is LLM-friendly

Models are trained to reason about **function calling**; MCP's tool definitions map cleanly to that pattern.

### 3. Stdio transport is brutally simple for local dev

`npx @some-scope/mcp-server` → host spawns process → done. No port hunting for the common case.

### 4. Cross-editor momentum

Cursor, Windsurf, Zed-flavored setups, and VS Code MCP extensions mean **shipping one server** reaches **many hosts** — the whole point of a protocol.

### 5. Open ecosystem

Servers for GitHub, Postgres, filesystem, browsers — the long tail grows because the **interface** is shared.

---

## What breaks in practice (the honest part)

### 1. "MCP compatible" is not a binary

Hosts differ on:

- Which **transport** modes they support out of the box
- How they **sandbox** tool execution
- How they surface **permissions** (auto-approve vs prompt)
- How they handle **server crash / restart**

**Symptom**: The same `mcp-server` works in Host A, fails silently in Host B until you tweak `args`, `env`, or path quoting.

**Mitigation**: Maintain a **per-host config snippet** in docs; test on **two** hosts minimum before claiming support.

### 2. Schema edge cases

Strict JSON Schema validation across **Zod**, **Ajv**, and host-specific validators means **optional fields**, **oneOf**, and **nested objects** occasionally disagree.

**Symptom**: "Tool call failed: invalid arguments" with no useful line number.

**Mitigation**: Keep tool schemas **boring** — flat arguments, explicit required fields, string enums.

### 3. Stdio vs SSE — two worlds

Local stdio servers are easy. **Remote** SSE servers introduce auth, reconnects, and corporate proxies.

**Symptom**: "It works on my machine" locally; remote users timeout.

**Mitigation**: Document **both** paths; default to stdio for indie tools unless you truly need network.

### 4. Error surfaces to the user are still immature

When a tool throws, some hosts show a generic failure; others stream partial logs.

**Mitigation**: Return **structured errors** in tool responses (`{ error: { code, message, hint } }`) and log server-side to a file path users can find.

### 5. Version skew

The spec and host implementations move. A feature you read about in a blog post may not be in the host version your users run.

**Mitigation**: Pin **documented** minimum host versions in your README.

---

## Cursor vs Windsurf vs VS Code — what integrators should expect

Exact details change with releases; the **pattern** is stable:

| Concern | Typical pattern |
|---|---|
| **Config file location** | Different per product — document absolute paths |
| **Config format** | JSON vs TOML vs merged JSON — maintain adapters |
| **Tool approval UX** | Some hosts ask every call; some remember — behavior differs |
| **Multi-workspace** | Some hosts spawn one server per window; some share — affects state |

**Practical advice**: Treat "works in Cursor" as **necessary** for dev-tool adoption in 2026, not sufficient. Have a **second host** in CI or manual checklist.

---

## MCP vs LSP — stop conflating them

| | **LSP** | **MCP** |
|---|---|---|
| **Primary job** | Language intelligence in the editor (go-to-def, diagnostics) | **Agent** tool/data integration |
| **Who calls** | Editor features | The **model** / agent loop |
| **Shape** | Well-aged, language-specific | Younger, general-purpose tools |

They **compose**: LSP makes the editor smart; MCP makes the **agent** capable. Replacing one with the other is usually the wrong framing.

---

## Minimal MCP server mental model (for readers implementing one)

If you are building your first server:

1. **One tool, one job** — e.g. `notify_state` with explicit enum values.
2. **Stdio first** — ship `npx your-package` with a clear `command` + `args` block for hosts.
3. **Idempotent** — tool calls may retry; safe behavior matters.
4. **No secrets in logs** — tokens in env vars, never print them.
5. **Version your package** — users pin `@latest` until something breaks; semver matters.

Our own server exposes agent lifecycle hooks for IDEs; the **shape** is intentionally small — fewer schema footguns.

---

## What I want next from the ecosystem

1. **Interop test suite** — a reference host + golden tool calls hosts could run in CI.
2. **Standard user-visible error codes** — so UIs can map failures to actions ("re-auth", "retry", "open logs").
3. **Clear capability negotiation** — host advertises: stdio only, SSE ok, fs access level, etc.
4. **First-class "agent state" events** — optional extension: `idle` / `running` / `waiting` / `done` / `error` as a **recommended** tool surface so companions like menu bar apps can subscribe uniformly.

Until then, integrators document host quirks and ship defensive code.

---

## TL;DR

- MCP is **real adoption**, not vapor — especially for **coding assistants** that need tools.
- **Strength**: one server, many hosts; **weakness**: host-by-host behavioral differences and schema footguns.
- **Ship boring schemas**, **test two hosts**, **document config paths**, and treat MCP as **infrastructure**, not magic.
- If you run **multiple agents across editors**, MCP is part of the glue — alongside native hooks — which is why products like [AgentBell](https://agentbell.dev) fan in **both** MCP events and file-based IPC.

---

*Related: [Best way to track multiple AI coding agents](./01-track-multiple-ai-coding-agents.md) · [Claude Code notifications](./02-claude-code-notification-when-done.md)*
