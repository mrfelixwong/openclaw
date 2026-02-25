# OpenClaw Deployment Progress (Fly.io + Telegram + OpenAI)

## Status: ✅ COMPLETE (Multi-agent Telegram + Memory Search working, WhatsApp pending)

---

## What's Running

- **App**: `openclaw-fw` on Fly.io, region `sjc`
- **Machine**: `78175e1a03d148`, `shared-cpu-2x`, 2GB RAM
- **URL**: https://openclaw-fw.fly.dev/
- **LLM**: OpenAI GPT-4o
- **Telegram bots** (multi-account):
  - `@sep_baba_bot` (Baba Shiv) — ✅ agent `baba`, workspace `/data/workspace-baba`
  - `@sep_barnett_bot` (Bill Barnett) — ✅ agent `barnett`, workspace `/data/workspace-barnett`
- **Allowed Telegram users**: `5422228671`, `8527086317`
- **Memory search**: ✅ enabled (OpenAI embeddings, session history recall)
- **CI/CD**: GitHub Action auto-deploys on push to `main` (fly.toml, Dockerfile, src/\*\*)
- **Repo**: [`mrfelixwong/openclaw`](https://github.com/mrfelixwong/openclaw) (fork of openclaw/openclaw)
- **WhatsApp**: ⏳ pending (datacenter IP blocked)
- **Cost**: ~$10-15/month

---

## Completed Steps

### ✅ 1. Fly.io CLI installed and authenticated

```bash
brew install flyctl && fly auth login
```

### ✅ 2. Repo cloned, app created, volume provisioned

- App: `openclaw-fw`, region: `sjc`
- Volume: `openclaw_data`, 1GB

### ✅ 3. fly.toml configured

Key settings added during deployment:

- `NODE_COMPILE_CACHE = "/data/.node_compile_cache"` — reduces cold start from 6min → 38sec
- `[[http_service.checks]] grace_period = "600s"` — allows slow Node startup
- `[[restart]] policy = "always"` — auto-restart on crash
- `auto_stop_machines = false` — keep alive for persistent connections

### ✅ 4. Secrets set

```bash
fly secrets set OPENCLAW_GATEWAY_TOKEN=<token>
fly secrets set OPENAI_API_KEY=sk-...
```

### ✅ 5. Deployed

```bash
fly deploy
```

### ✅ 6. Config written to /data/openclaw.json

Current config (as of 2026-02-25):

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "openai/gpt-4o" },
      "memorySearch": {
        "provider": "openai",
        "sources": ["memory", "sessions"],
        "experimental": { "sessionMemory": true }
      },
      "tools": {
        "alsoAllow": ["memory_search", "memory_get"]
      }
    },
    "list": [
      { "id": "barnett", "workspace": "/data/workspace-barnett" },
      { "id": "baba", "workspace": "/data/workspace-baba" }
    ]
  },
  "bindings": [
    { "agentId": "barnett", "match": { "channel": "telegram", "accountId": "barnett" } },
    { "agentId": "baba", "match": { "channel": "telegram", "accountId": "baba" } }
  ],
  "gateway": {
    "controlUi": {
      "allowedOrigins": ["https://openclaw-fw.fly.dev"]
    }
  },
  "channels": {
    "whatsapp": {
      "dmPolicy": "allowlist",
      "allowFrom": ["+14252462059"],
      "groupPolicy": "allowlist"
    },
    "telegram": {
      "accounts": {
        "baba": {
          "botToken": "<redacted>",
          "dmPolicy": "allowlist",
          "allowFrom": [5422228671, 8527086317],
          "streaming": "partial"
        },
        "barnett": {
          "botToken": "<redacted>",
          "dmPolicy": "allowlist",
          "allowFrom": [5422228671, 8527086317],
          "streaming": "partial"
        }
      }
    }
  }
}
```

### ✅ 7. Device pairing bypassed

Manually wrote approved device to `/data/devices/paired.json` (CLI couldn't connect due to `--bind lan`).

### ✅ 8. Telegram connected (multi-account)

- Created two bots via @BotFather:
  - `@sep_baba_bot` (Baba Shiv) — bound to agent `baba`
  - `@sep_barnett_bot` (Bill Barnett) — bound to agent `barnett`
- Each agent has its own workspace with SOUL.md, IDENTITY.md, lecture notes, etc.
- Both use `streamMode: "partial"` for streaming preview edits
- Allowlist: Telegram IDs `5422228671`, `8527086317`

### ✅ 9. End-to-end smoke test passed (Telegram)

- Both bots receive messages, route to correct agents, and respond via GPT-4o
- Agent workspaces: `/data/workspace-barnett/`, `/data/workspace-baba/`
- Session files: `/data/agents/{barnett,baba}/sessions/`

### ✅ 10. Memory search enabled (2026-02-25)

- **Backend**: OpenAI embeddings (`text-embedding-3-small`) via existing `OPENAI_API_KEY`
- **Sources**: workspace `MEMORY.md` files + session transcripts (experimental `sessionMemory`)
- **Tools**: `memory_search` and `memory_get` added via `alsoAllow` (not in default `messaging` tool profile)
- **Index**: SQLite at `/data/memory/{agentId}.sqlite`
- **Workspace files**: Created `MEMORY.md` in both `/data/workspace-baba/` and `/data/workspace-barnett/`
- **Gateway fix**: Added `gateway.controlUi.allowedOrigins` (required by newer upstream builds for non-loopback `--bind lan`)

### ✅ 11. GitHub fork + CI/CD (2026-02-25)

- Forked `openclaw/openclaw` → [`mrfelixwong/openclaw`](https://github.com/mrfelixwong/openclaw)
- Remotes: `origin` = fork, `upstream` = openclaw/openclaw
- GitHub Action (`.github/workflows/fly-deploy.yml`): auto-deploys on push to `main` when `fly.toml`, `Dockerfile`, or `src/**` change
- `FLY_API_TOKEN` secret set on the GitHub repo (deploy-scoped token for `openclaw-fw`)

### Debugging notes (2026-02-23)

- `sendMessage ok` logs do NOT appear in fly stdout when `streamMode: "partial"` is active — responses are delivered via draft-stream (`editMessageText`) which bypasses the `runtime.log` path. This is a logging gap, not a delivery failure.
- WhatsApp health-monitor restart loop is expected (datacenter IP blocked) and does not affect Telegram.

---

## Pending

### ⏳ WhatsApp — blocked by datacenter IP issue

**Root cause**: WhatsApp's server-side IP check blocks QR linking from Fly.io datacenter IPs (confirmed Baileys issue [#2248](https://github.com/WhiskeySockets/Baileys/issues/2248)).

**Workaround in progress**: Pair locally (residential IP), upload creds to Fly.io.

- Local gateway running at `http://localhost:3001/?token=localtest`
- Data dir: `/tmp/openclaw-local-data/`
- **Wait ~24 hours** from last failed attempt before retrying (WhatsApp rate-limits failed pairings)
- After successful local pairing, upload `/tmp/openclaw-local-data/credentials/whatsapp/default/` to Fly.io VM

**Upload command** (after successful local pairing):

```bash
# Encode and upload creds
CREDS=$(cat /tmp/openclaw-local-data/credentials/whatsapp/default/creds.json | base64)
fly ssh console -a openclaw-fw -C "mkdir -p /data/credentials/whatsapp/default && sh -c \"echo $CREDS | base64 -d > /data/credentials/whatsapp/default/creds.json\""
fly machine restart 78175e1a03d148 -a openclaw-fw --skip-health-checks
```

---

## Troubleshooting Reference

| Issue                                 | Fix                                                                                                                     |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Machine stops every 5 min             | Was free trial — fixed by adding credit card                                                                            |
| Node cold start 6+ min                | Fixed: `NODE_COMPILE_CACHE=/data/.node_compile_cache` in fly.toml                                                       |
| Health check timeout                  | Fixed: `grace_period = "600s"` in `[[http_service.checks]]`                                                             |
| Machine doesn't restart on exit       | Fixed: `[[restart]] policy = "always"` in fly.toml                                                                      |
| Config write via heredoc corrupted    | Use base64: `echo "$CONFIG" \| base64 \| fly ssh console -C "sh -c 'base64 -d > /data/openclaw.json'"`                  |
| CLI can't connect to gateway          | Gateway uses `--bind lan`; CLI defaults to 127.0.0.1:18789. Write paired.json manually.                                 |
| WhatsApp "couldn't link device"       | Fly.io datacenter IP blocked by WhatsApp. Pair from residential IP (localhost).                                         |
| gateway.port in config breaks binding | Don't set `port` in config JSON — use CLI flag `--port 3000` only                                                       |
| No `sendMessage ok` in fly logs       | Expected with `streamMode: "partial"` — delivery uses draft-stream editMessageText path which doesn't log to fly stdout |
| WhatsApp health-monitor restart loop  | Expected — datacenter IP blocked by WhatsApp, does not affect Telegram                                                  |
| Gateway fails: `allowedOrigins`       | Newer upstream builds require `gateway.controlUi.allowedOrigins` for non-loopback bind. Add to openclaw.json.           |
| memory_search not called by agent     | Tool is only in `coding` profile. Add `tools.alsoAllow: ["memory_search", "memory_get"]` to agent defaults.             |
| OpenAI rate limit on startup          | Session memory indexing + embedding calls can spike on cold start. Wait 1 min or use `gpt-4o-mini` to reduce pressure.  |

## Next: Stanford GSB Demo Prep

See demo ideas discussed — showcase: live Telegram → GPT-4o loop, custom persona via system prompt, multi-user allowlist, ownership/control framing.
