# metafactory-agents

Manifests + persona files for the metafactory ecosystem's persona-driven agents. Source of truth from which `grove install agent` derives `bot.yaml` entries (Phase 5 of `meta-factory#390`, dogfooded in Phase 8 of `forge#6`).

## Layout

```
metafactory-agents/
├── README.md            ← this file
├── CREATING.md          ← keys-handover for new agent operators (post-Phase-9)
├── MIGRATING.md         ← step-by-step for migrating an existing agent
├── luna/
│   ├── arc-manifest.yaml
│   └── persona.md
├── echo/
│   ├── arc-manifest.yaml
│   └── persona.md
├── ivy/                 ← scaffolded; JC fills TODOs before install
│   ├── arc-manifest.yaml
│   └── persona.md
├── holly/               ← scaffolded; JC fills TODOs before install
│   ├── arc-manifest.yaml
│   └── persona.md
└── juniper/            ← PM-style backlog agent (botanical sibling); runs the `distiller` skill
    ├── arc-manifest.yaml
    └── persona.md
```

(Forge has its own repo at `the-metafactory/forge` because Forge is a release agent with its own design + retro lifecycle. Luna/Echo/Ivy/Holly/Juniper are simpler agents and live together here.)

## Status (2026-04-27)

| Agent | Manifest written | Installed via pipeline | Notes |
|---|---|---|---|
| Forge | ✅ (in `the-metafactory/forge`) | ✅ (Phase 7 dogfood) | Released as Forge v0.2.0 same day; manifest migrated to new schema (state, installScope) |
| Luna | ✅ this repo (canonical `agent-luna`) | ✅ via `grove install agent` | `mentionRole: agent-restricted` captures Luna's narrow cross-adapter authority |
| Echo | ✅ this repo (canonical `agent-echo`) | ✅ via `grove install agent` | Same `mentionRole` shape; review-only across the board |
| Ivy | ✅ this repo (canonical `agent-ivy`) | ✅ via `grove install agent` (JC-owned host) | Coordinator on JC's side. Operator role + dm.operatorRole installed via grove#266 fix |
| Holly | ✅ this repo (canonical `agent-holly`) | ✅ via `grove install agent` (JC-owned host) | Adversarial reviewer on JC's side. Reviewer-class shape, no DM |
| Juniper | ✅ this repo (canonical `agent-juniper`) | (JC-owned, JC's host) — pending Discord app creation | PM-style backlog agent. Reads threads, runs `Skill("distiller")`, files GitHub issues with two-phase confirmation |

## How to install (current)

After grove#258 (mentionRole + schema renames) merged, the install for migrating agents is one command — no flag needed because the manifest's `mentionRole` declares the cross-adapter role explicitly:

```bash
LUNA_TOKEN=$(grep -A1 "instanceId: discord-luna" ~/.config/grove/bot.yaml | grep token: | sed 's/.*token: //')

bun /path/to/grove/src/cli/grove.ts install agent ./luna/arc-manifest.yaml \
  --discord-token "$LUNA_TOKEN" \
  --discord-bot-id "YOUR-BOT-ID" \
  --discord-guild "YOUR-GUILD-ID" \
  --discord-channel "YOUR-CHANNEL-ID" \
  --yes
```

Hot-reload is automatic via AP-104 (`config-watcher` includes `trustedAgentBots` + `discord[].roles[]` in `SAFE_FIELDS`). No grove-bot restart needed.

See `MIGRATING.md` for the full procedure including troubleshooting.

## Why this matters

Hand-written `bot.yaml` entries are operator drift waiting to happen. Manifests are the canonical, reviewable, version-controlled source of truth. Operators who clone this repo + install pipeline get the agent's full authority + persona + identity reproducibly, without needing to know the implementation details of `bot.yaml`.

When Phase 8 completes (this migration + the architectural fixes that unblock it), every metafactory agent will be defined here (or in their own design repo for the heavier ones like Forge) and installed via the same single command. Plug-and-play.

## Related

- `forge/design/agent-platform.md` — manifest schema + four-layer split + install flow
- `forge/design/forge.md` — Forge as the canonical example agent
- `the-metafactory/grove#252` — install pipeline (merged 2026-04-27)
- `the-metafactory/grove#254` — uninstall pipeline (merged 2026-04-27)
- `the-metafactory/grove#257` — trustedAgentBots self-skip on migration (open, blocks Luna/Echo install)
- `the-metafactory/forge#6` — Phase 8 migration tracking issue
