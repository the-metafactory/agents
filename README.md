# metafactory-agents

Manifests + persona files for the metafactory ecosystem's persona-driven agents. Source of truth from which `grove install agent` derives `bot.yaml` entries (Phase 5 of `meta-factory#390`, dogfooded in Phase 8 of `forge#6`).

## Layout

```
metafactory-agents/
├── README.md            ← this file
├── MIGRATING.md         ← step-by-step for migrating an existing agent
├── luna/
│   ├── arc-manifest.yaml
│   └── persona.md
├── echo/
│   ├── arc-manifest.yaml
│   └── persona.md
└── ivy/
    └── PLACEHOLDER.md   ← JC drives this when ready
```

(Forge has its own repo at `the-metafactory/forge` because Forge is a release agent with its own design + retro lifecycle. Luna/Echo/Ivy are simpler agents and live together here.)

## Status (2026-04-27)

| Agent | Manifest written | Installed via pipeline | Notes |
|---|---|---|---|
| Forge | ✅ (in `the-metafactory/forge`) | ✅ (Phase 7 dogfood) | Released as Forge v0.2.0 the same day |
| Luna | ✅ this repo | ⏸ blocked on grove#257 | Manifest validates; install conflicts with existing trustedAgentBots pin |
| Echo | ✅ this repo | ⏸ blocked on grove#257 | Same blocker as Luna |
| Ivy | placeholder | (JC-owned) | See `MIGRATING.md` for JC's recipe when ready |

## How to install once grove#257 lands

```bash
source ~/.config/grove/.luna-reinstall-backup/secrets.env  # or .echo-reinstall-backup/

cd /path/to/grove
bun src/cli/grove.ts install agent ./luna/arc-manifest.yaml \
  --discord-token "$DISCORD_TOKEN" \
  --discord-bot-id "$DISCORD_BOT_ID" \
  --discord-guild "$DISCORD_GUILD" \
  --discord-channel "$DISCORD_CHANNEL" \
  --skip-trusted-bot-pin \   # flag added by grove#257
  --yes
```

Then restart grove-bot. See `MIGRATING.md` for the full procedure.

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
