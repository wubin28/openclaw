# Sessions and Memory — OpenClaw Beginner Guide

> **How this guide was made**: Claude Code read the current OpenClaw source and
> reorganized the official documentation into a learning path suited for new users.
> Discrepancies found between the official docs and the actual code are called out
> explicitly in the [Appendix](#appendix-discrepancies-between-official-docs-and-code).

---

## Table of Contents

1. [What is a session?](#1-what-is-a-session)
2. [Multi-user safety: keeping DMs private](#2-multi-user-safety-keeping-dms-private)
3. [When does a session reset?](#3-when-does-a-session-reset)
4. [How memory works](#4-how-memory-works)
5. [Managing long conversations: pruning and compaction](#5-managing-long-conversations-pruning-and-compaction)
6. [Agents talking to agents: session tools](#6-agents-talking-to-agents-session-tools)
7. [Keeping storage under control: maintenance](#7-keeping-storage-under-control-maintenance)
8. [Quick reference: key config options](#8-quick-reference-key-config-options)
9. [Appendix: discrepancies between official docs and code](#appendix-discrepancies-between-official-docs-and-code)

---

## 1. What is a session?

### Pain point

You send a message to your agent, come back tomorrow, and it has no idea what you were talking about. Or you start a fresh chat but the agent drags in context from three days ago. When does a conversation "start" and "end"?

### Solution

OpenClaw groups incoming messages into **sessions** — named buckets of conversation history. Every message is routed to a session key, and the agent loads that session's transcript before replying. As long as two messages share the same key, the agent sees them as one continuous conversation.

### Concept: session keys

Session keys are strings that look like:

```
agent:<agentId>:main                          ← your own direct-chat session
agent:<agentId>:telegram:group:123456         ← a Telegram group
agent:<agentId>:discord:channel:789012        ← a Discord channel
cron:<jobId>                                  ← a scheduled job
hook:<uuid>                                   ← a webhook trigger
node-<nodeId>                                 ← a spawned node session
```

**Where does the state live?**

On the gateway host, under:

```
~/.openclaw/agents/<agentId>/sessions/
  sessions.json           ← index: sessionKey → { sessionId, updatedAt, … }
  <SessionId>.jsonl       ← transcript for each session
```

The gateway is the **sole source of truth**. UI clients (macOS app, WebChat) always query the gateway for session lists and token counts. They do not parse JSONL files themselves.

### Simple example

```jsonc
// ~/.openclaw/openclaw.json
{
  "session": {
    "mainKey": "main"   // the name of your primary direct-chat bucket (default: "main")
  }
}
```

The key that collects all your own DMs is `agent:<agentId>:main` by default. You rarely need to change `mainKey`.

---

## 2. Multi-user safety: keeping DMs private

### Pain point

You give several people access to your agent. Alice tells it something private. Bob asks "what were we just talking about?" — and the agent answers Bob using Alice's context, because by default all DMs share the same session bucket.

### Solution

Set `session.dmScope` so each sender gets their own session.

### Concept: `dmScope`

`dmScope` controls how direct messages are bucketed:

| Value | Session key pattern | Use when |
|---|---|---|
| `"main"` *(default)* | `agent:<id>:main` | Single-user; all DMs are one conversation |
| `"per-peer"` | `agent:<id>:direct:<peerId>` | Multiple users on the **same** account |
| `"per-channel-peer"` | `agent:<id>:<channel>:direct:<peerId>` | Multiple users, possibly on different channels |
| `"per-account-channel-peer"` | `agent:<id>:<channel>:<accountId>:direct:<peerId>` | Multiple accounts on the same channel |

**When to use `"per-channel-peer"`** — recommended for any setup where more than one person can send you DMs.

**When to use `"per-account-channel-peer"`** — when you run the same channel integration under multiple accounts (e.g., two Telegram phone numbers).

### Example: enabling secure DM mode

```jsonc
// ~/.openclaw/openclaw.json
{
  "session": {
    "dmScope": "per-channel-peer"
  }
}
```

You should enable this when:
- You have pairing approvals for more than one sender
- You use a DM allowlist with multiple entries
- You set `dmPolicy: "open"`
- Multiple phone numbers or accounts can message your agent

> **Note**: local CLI onboarding already writes `dmScope: "per-channel-peer"` when it is unset.
> Existing explicit values are never overwritten.

### Bonus: linking one person's identities across channels

If the same person contacts you on both Telegram and Discord and you want them to share a session:

```jsonc
{
  "session": {
    "dmScope": "per-channel-peer",
    "identityLinks": {
      "alice": ["telegram:123456789", "discord:987654321012345678"]
    }
  }
}
```

`identityLinks` maps the canonical identity name to a list of `provider:id` strings. When a message arrives from any of those IDs, OpenClaw replaces the `<peerId>` in the session key with the canonical name, so Alice gets one session no matter which platform she uses.

### Verifying your setup

```sh
openclaw security audit
```

---

## 3. When does a session reset?

### Pain point

You want your agent to start a fresh context every morning, but not forget things mid-conversation. Or you want idle group chats to clear their history automatically, but your own DM to persist for days.

### Solution

Configure a **reset policy**. A reset marks the current session as stale; the next incoming message creates a new session ID and starts with an empty context.

### Concept: reset policies

**Default**: daily reset at 4:00 AM **local time on the gateway host**.

```jsonc
// ~/.openclaw/openclaw.json
{
  "session": {
    "reset": {
      "mode": "daily",   // "daily" | "idle"
      "atHour": 4        // 0–23, local time on the gateway host
    }
  }
}
```

**Idle reset** — clear context after N minutes of inactivity:

```jsonc
{
  "session": {
    "reset": {
      "mode": "idle",
      "idleMinutes": 120
    }
  }
}
```

When both `mode: "daily"` and `idleMinutes` are set, whichever expires first wins.

**Per-type overrides** — different rules for DMs, groups, and threads:

```jsonc
{
  "session": {
    "resetByType": {
      "direct": { "mode": "idle", "idleMinutes": 240 },
      "group":  { "mode": "idle", "idleMinutes": 120 },
      "thread": { "mode": "daily", "atHour": 4 }
    }
  }
}
```

**Per-channel overrides** — override for a specific platform:

```jsonc
{
  "session": {
    "resetByChannel": {
      "discord": { "mode": "idle", "idleMinutes": 10080 }  // 1 week
    }
  }
}
```

`resetByChannel` takes precedence over `reset` and `resetByType`.

### Triggering a reset manually

In any chat, send `/new` or `/reset` as a standalone message. The session gets a fresh ID and the agent runs a short greeting turn. You can also pass a new model:

```
/new anthropic/claude-sonnet-4.6
```

---

## 4. How memory works

### Pain point

The model can only see what's in its context window. Once a session resets or compacts, it has no recollection of preferences or decisions you told it weeks ago. You want things to stick.

### Solution

OpenClaw stores memory as plain Markdown files in the agent **workspace**. Whatever is written to these files survives session resets and compactions. The model reads them at session start.

### Concept: the two memory layers

| File | Purpose | When read |
|---|---|---|
| `MEMORY.md` | Curated long-term facts, preferences, decisions | Every session start (main/private sessions only) |
| `memory/YYYY-MM-DD.md` | Daily append-only running notes | Today's + yesterday's file at session start |

Both files live under the workspace directory (default: `~/.openclaw/workspace`).

> **Group sessions**: memory files are loaded only for the main private session, never for group/channel contexts.

### Concept: memory tools

The agent has two tools for these files:

| Tool | What it does |
|---|---|
| `memory_search` | Semantic search over indexed snippets from all memory files |
| `memory_get` | Read a specific file or line range; returns `{ text: "", path }` gracefully when the file doesn't exist yet |

### When to write to which file

- **MEMORY.md** — user preferences, long-term decisions, facts that should survive forever
- **memory/YYYY-MM-DD.md** — today's running notes, task progress, temporary context

The model knows how to use these. If you want something to stick, just ask:

> "Remember that I prefer bullet points over prose."

The agent will write it into memory. You can also say:

> "Write into MEMORY.md that my timezone is Asia/Shanghai."

### Automatic memory flush before compaction

When a session is nearing auto-compaction (context window nearly full), OpenClaw silently triggers a "memory flush" turn. The agent is prompted to write any lasting notes to `memory/YYYY-MM-DD.md` before the context is summarized. The model usually replies `NO_REPLY` so you never see this turn.

This runs automatically and is enabled by default. It is skipped if the workspace is read-only.

**Configuration** (if you need to tune it):

```jsonc
{
  "agents": {
    "defaults": {
      "compaction": {
        "reserveTokensFloor": 20000,
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 4000,
          "systemPrompt": "Session nearing compaction. Store durable memories now.",
          "prompt": "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      }
    }
  }
}
```

> **Undocumented trigger**: the memory flush also fires when the `.jsonl` transcript file exceeds **2 MB** in size (`forceFlushTranscriptBytes`), regardless of token count. This is not shown in the official config docs.

### Vector search (optional)

OpenClaw can build a vector index over memory files for semantic recall. This requires additional setup (embedding provider config). See the Memory configuration reference for details.

---

## 5. Managing long conversations: pruning and compaction

### Pain point

After a long conversation, the model gets slower, more confused, or starts making mistakes. The context window is filling up with old tool outputs and stale messages that waste tokens.

OpenClaw has two complementary mechanisms to handle this:

---

### 5a. Session pruning (trim tool results, in-memory)

**Pain point**: your agent called a tool that returned a 50 KB JSON blob. That blob is now sitting in every subsequent LLM request, eating tokens and inflating cost, even though it's no longer relevant.

**Solution**: session pruning removes old, oversized tool results from the in-memory context *right before* each LLM call. It does **not** rewrite your on-disk `.jsonl` transcript.

**When it runs**: only in `"cache-ttl"` mode, and only when the last Anthropic call is older than the configured TTL (default: 5 minutes).

```jsonc
{
  "agents": {
    "defaults": {
      "contextPruning": {
        "mode": "cache-ttl",
        "ttl": "5m"
      }
    }
  }
}
```

Default is `"off"`. Only available for Anthropic API calls (and OpenRouter Anthropic models).

**What gets pruned**: only `toolResult` messages. User and assistant messages are never touched. The last `keepLastAssistants` (default: 3) assistant turns are protected; tool results before that cutoff are candidates for trimming.

Two levels of trimming:
- **Soft-trim**: keep head + tail, replace middle with `...` (only for oversized results)
- **Hard-clear**: replace entire result with `[Old tool result content cleared]`

**Restricting which tools get pruned**:

```jsonc
{
  "agents": {
    "defaults": {
      "contextPruning": {
        "mode": "cache-ttl",
        "tools": {
          "allow": ["exec", "read"],
          "deny": ["*image*"]
        }
      }
    }
  }
}
```

---

### 5b. Compaction (summarize and persist)

**Pain point**: the context window is nearly full, and you can't trim your way out — there are too many real conversation turns. You need to free up space permanently.

**Solution**: **compaction** summarizes older conversation history into a compact summary entry and rewrites it into the `.jsonl` transcript. The summary + recent messages are used for all future requests.

**Auto-compaction** triggers automatically when the session approaches the context window limit. You will see `🧹 Auto-compaction complete` in verbose mode.

**Manual compaction**:

```
/compact
/compact Focus on decisions and open questions
```

**Using a different model for compaction** (e.g., when your primary model is a small local model):

```jsonc
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "openrouter/anthropic/claude-sonnet-4-6"
      }
    }
  }
}
```

**Identifier preservation**: by default (`identifierPolicy: "strict"`), compaction summaries preserve opaque identifiers like IDs and hashes. You can disable this or provide custom instructions.

**Key difference between pruning and compaction**:

| | Session pruning | Compaction |
|---|---|---|
| Scope | In-memory only (per request) | Persistent (rewrites JSONL) |
| What it touches | Old tool results | Older conversation turns |
| Reversible? | Yes (original JSONL unchanged) | No (summary replaces old turns) |
| Trigger | Anthropic TTL expiry | Context window near-full |

---

## 6. Agents talking to agents: session tools

### Pain point

You want your agent to delegate subtasks to helper agents, or you want one session to send a message to another session, or you need to inspect what's happening in another session.

### Solution

OpenClaw provides five session tools available to agents:

| Tool | What it does |
|---|---|
| `sessions_list` | List sessions (by kind, activity window, etc.) |
| `sessions_history` | Fetch transcript for one session |
| `sessions_send` | Send a message into another session and optionally wait for a reply |
| `sessions_spawn` | Spawn an isolated child sub-agent session |
| `sessions_yield` | End the current turn; use after spawning sub-agents to receive their results as the next message |

> **Note**: `sessions_yield` is not mentioned in the official documentation but is a fully implemented and available tool. Its purpose is to cleanly end the parent agent's current turn so that sub-agent results can arrive as the next message.

### Concept: session key shortcuts

When using these tools, you can always use the **literal key `"main"`** to refer to your own primary direct-chat session. You don't need to construct the full `agent:<agentId>:main` string.

### Example: spawning a sub-agent

```
sessions_spawn(task="Summarize the last 10 emails and return a bullet list", label="email-summarizer")
```

The spawned session gets the key `agent:<agentId>:subagent:<uuid>`. After it completes, OpenClaw runs an announce step and posts the result back to your chat.

Then call `sessions_yield` to end your turn and receive the sub-agent's result cleanly:

```
sessions_yield(message="Waiting for email summarizer")
```

### Concept: visibility — who can see which sessions

By default, a session can only see itself and sessions it spawned (`"tree"` visibility). You can widen this:

```jsonc
{
  "tools": {
    "sessions": {
      "visibility": "tree"   // "self" | "tree" | "agent" | "all"
    }
  }
}
```

Sandboxed sessions are automatically clamped to `"tree"` even if you configure `"all"`.

### Controlling inter-session message delivery

You can block delivery to specific session types without listing individual IDs:

```jsonc
{
  "session": {
    "sendPolicy": {
      "rules": [
        { "action": "deny", "match": { "channel": "discord", "chatType": "group" } },
        { "action": "deny", "match": { "keyPrefix": "cron:" } }
      ],
      "default": "allow"
    }
  }
}
```

Runtime override (owner only, send as a standalone message):
- `/send on` — allow for this session
- `/send off` — deny for this session
- `/send inherit` — revert to config rules

### Ping-pong turns in `sessions_send`

When you use `sessions_send` to send to another agent, OpenClaw runs a reply-back loop between the two agents (up to `session.agentToAgent.maxPingPongTurns`, default 5). Either agent can reply `REPLY_SKIP` to stop the loop early.

---

## 7. Keeping storage under control: maintenance

### Pain point

You've been running your agent for months. The `sessions.json` file is huge, and the `~/.openclaw/agents/<agentId>/sessions/` directory is taking gigabytes of disk space.

### Solution

Configure `session.maintenance` to automatically prune old sessions and enforce disk budgets.

### Concept: maintenance modes

| Mode | Behavior |
|---|---|
| `"warn"` *(default)* | Reports what would be pruned; does not delete anything |
| `"enforce"` | Actually applies cleanup |

### Default maintenance settings

```jsonc
{
  "session": {
    "maintenance": {
      "mode": "warn",
      "pruneAfter": "30d",
      "maxEntries": 500,
      "rotateBytes": "10mb",
      "resetArchiveRetention": "30d"
    }
  }
}
```

### Cleanup order (when `mode: "enforce"`)

1. Prune sessions older than `pruneAfter`
2. Cap entry count to `maxEntries` (oldest first)
3. Archive transcripts for removed entries
4. Purge old `*.deleted.*` and `*.reset.*` archives past retention
5. Rotate `sessions.json` when it exceeds `rotateBytes`
6. If `maxDiskBytes` is set, enforce disk budget down to `highWaterBytes`

### Recommended production config

```jsonc
{
  "session": {
    "maintenance": {
      "mode": "enforce",
      "pruneAfter": "14d",
      "maxEntries": 500,
      "rotateBytes": "10mb",
      "maxDiskBytes": "1gb",
      "highWaterBytes": "800mb"
    }
  }
}
```

### CLI commands

```sh
# Preview what would be cleaned up (dry run):
openclaw sessions cleanup --dry-run

# Actually enforce cleanup now:
openclaw sessions cleanup --enforce

# Preview as JSON for scripting:
openclaw sessions cleanup --dry-run --json
```

### Performance note

Maintenance runs on every session-store write. Very large stores (high `maxEntries`, long `pruneAfter`, many transcript files) increase write latency. Set both time and count limits, not just one.

---

## 8. Quick reference: key config options

### Session routing

| Config key | Default | Description |
|---|---|---|
| `session.scope` | `"per-sender"` | Group/channel session isolation |
| `session.dmScope` | `"main"` | DM session isolation strategy |
| `session.mainKey` | `"main"` | Name of the primary direct-chat bucket |
| `session.identityLinks` | `{}` | Map canonical identity → list of `provider:id` strings |

### Session reset

| Config key | Default | Description |
|---|---|---|
| `session.reset.mode` | `"daily"` | `"daily"` or `"idle"` |
| `session.reset.atHour` | `4` | Daily reset hour (local time on gateway host, 0–23) |
| `session.reset.idleMinutes` | `0` (disabled) | Idle window in minutes |
| `session.resetByType` | — | Per-type overrides (`direct`, `group`, `thread`) |
| `session.resetByChannel` | — | Per-channel overrides |

### Session maintenance

| Config key | Default | Description |
|---|---|---|
| `session.maintenance.mode` | `"warn"` | `"warn"` or `"enforce"` |
| `session.maintenance.pruneAfter` | `"30d"` | Remove sessions older than this |
| `session.maintenance.maxEntries` | `500` | Max session count |
| `session.maintenance.rotateBytes` | `"10mb"` | Rotate `sessions.json` when it exceeds this |
| `session.maintenance.maxDiskBytes` | unset | Hard disk budget (optional) |
| `session.maintenance.highWaterBytes` | 80% of maxDiskBytes | Target size after budget enforcement |
| `session.maintenance.resetArchiveRetention` | same as `pruneAfter` | Archive retention for reset/deleted transcripts |

### Context pruning

| Config key | Default | Description |
|---|---|---|
| `agents.defaults.contextPruning.mode` | `"off"` | `"off"` or `"cache-ttl"` |
| `agents.defaults.contextPruning.ttl` | `"5m"` | Only prune if last Anthropic call older than this |
| `agents.defaults.contextPruning.keepLastAssistants` | `3` | Protect last N assistant turns from pruning |

### Compaction

| Config key | Default | Description |
|---|---|---|
| `agents.defaults.compaction.model` | (primary model) | Separate model for compaction summarization |
| `agents.defaults.compaction.reserveTokensFloor` | `20000` | Min tokens to reserve before compaction |
| `agents.defaults.compaction.identifierPolicy` | `"strict"` | `"strict"`, `"off"`, or `"custom"` |
| `agents.defaults.compaction.memoryFlush.enabled` | `true` | Run memory flush before compaction |
| `agents.defaults.compaction.memoryFlush.softThresholdTokens` | `4000` | Flush when this many tokens from compaction trigger |

### Session tools visibility

| Config key | Default | Description |
|---|---|---|
| `tools.sessions.visibility` | `"tree"` | `"self"`, `"tree"`, `"agent"`, or `"all"` |
| `agents.defaults.sandbox.sessionToolsVisibility` | `"spawned"` | Clamp for sandboxed sessions |

---

## 9. Inspecting sessions

### CLI

```sh
# Show store path and recent sessions:
openclaw status

# Dump all session entries (filter by activity):
openclaw sessions --json
openclaw sessions --json --active 60   # active in the last 60 minutes

# Fetch sessions from the running gateway (works for remote gateways too):
openclaw gateway call sessions.list --params '{}'
```

### In chat

| Command | What it shows |
|---|---|
| `/status` | Agent reachability, context usage, toggles |
| `/context list` | What's in the system prompt and workspace files |
| `/context detail` | Biggest context contributors |
| `/compact [instructions]` | Force compaction now |
| `/new` or `/reset` | Start a fresh session ID |
| `/stop` | Abort the current run and all sub-agents |

---

## Appendix: Discrepancies between official docs and code

The following were found by reading the current source. Where the docs and the code differ, trust the code.

### 1. `sessions_yield` tool is not documented

The official docs list four session tools: `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`. However, a fifth tool — **`sessions_yield`** — exists and is fully implemented (`src/agents/tools/sessions-yield-tool.ts`).

Its purpose (from the code description):
> "End your current turn. Use after spawning subagents to receive their results as the next message."

It accepts one optional parameter:
- `message` — a status message for the yield (default: `"Turn yielded."`)

This is most useful immediately after calling `sessions_spawn`: yield the turn so the sub-agent result arrives cleanly as the next inbound message.

### 2. Memory flush has an undocumented second trigger: `forceFlushTranscriptBytes`

The official docs describe only one trigger for the pre-compaction memory flush: when the session token estimate crosses `contextWindow - reserveTokensFloor - softThresholdTokens`.

The code (`src/auto-reply/reply/memory-flush.ts`) has a second trigger: when the `.jsonl` transcript file exceeds **`forceFlushTranscriptBytes`** (default: **2 MB**). This fires regardless of token count, so transcript-heavy sessions (many tool results, large outputs) may trigger a memory flush sooner than the token threshold alone would suggest.

You can configure this value if needed:

```jsonc
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "forceFlushTranscriptBytes": "2mb"
        }
      }
    }
  }
}
```

### 3. `sessions_history` and `sessions_list` have undocumented response limits

The implementation applies limits not mentioned in the official docs:
- **Per-message truncation**: 4,000 characters per message in `sessions_history`
- **Total response cap**: 80 KB across all messages in `sessions_history`
- **`messageLimit` max**: the `messageLimit` parameter in `sessions_list` is capped at **20** messages per session

These limits prevent accidentally overwhelming the model context when fetching history, but they mean you may get truncated results for long sessions.
