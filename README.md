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
└── distiller/           ← scaffolded; JC fills TODOs before install
    ├── arc-manifest.yaml
    └── persona.md
```

(Forge has its own repo at `the-metafactory/forge` because Forge is a release agent with its own design + retro lifecycle. Luna/Echo/Ivy/Holly/Distiller are simpler agents and live together here.)

## Status (2026-04-27)

| Agent | Manifest written | Installed via pipeline | Notes |
|---|---|---|---|
| Forge | ✅ (in `the-metafactory/forge`) | ✅ (Phase 7 dogfood) | Released as Forge v0.2.0 same day; manifest migrated to new schema (state, installScope) |
| Luna | ✅ this repo (canonical `agent-luna`) | ✅ via `grove install agent` | `mentionRole: agent-restricted` captures Luna's narrow cross-adapter authority |
| Echo | ✅ this repo (canonical `agent-echo`) | ✅ via `grove install agent` | Same `mentionRole` shape; review-only across the board |
| Ivy | scaffold (JC fills TODOs) | (JC-owned, JC's host) | botId 1487024060411023491 known from Andreas's `trustedAgentBots`. See `ivy/arc-manifest.yaml` |
| Holly | scaffold (JC fills TODOs) | (JC-owned, JC's host) | Role + scope TBD by JC. See `holly/arc-manifest.yaml` |
| Distiller | scaffold (JC fills TODOs) | (JC-owned, JC's host) | New agent: project-manager backlog distillation. Tracker: [#2](https://github.com/the-metafactory/agents/issues/2) |

## How to install (current)

After grove#258 (mentionRole + schema renames) merged, the install for migrating agents is one command — no flag needed because the manifest's `mentionRole` declares the cross-adapter role explicitly:

```bash
LUNA_TOKEN=$(grep -A1 "instanceId: discord-luna" ~/.config/grove/bot.yaml | grep token: | sed 's/.*token: //')

bun /path/to/grove/src/cli/grove.ts install agent ./luna/arc-manifest.yaml \
  --discord-token "$LUNA_TOKEN" \
  --discord-bot-id "1487180524542890144" \
  --discord-guild "1487023327791808592" \
  --discord-channel "1487029848164536361" \
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
