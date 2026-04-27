# Migrating an existing agent to the Metafactory Agent install pipeline

**Audience**: operators with a hand-configured agent in `~/.config/<host>/bot.yaml` who want to switch to manifest-driven install via `grove install agent`.

**Source of truth for the design**: `forge/design/agent-platform.md` (manifest schema) + `forge/design/forge.md` (Forge as the canonical example).

This guide captures the recipe Andreas used to migrate Luna + Echo on 2026-04-27. Forge was the first agent installed via the new pipeline (Phase 7 of `meta-factory#390`); Luna + Echo are the second cohort (Phase 8 of `forge#6`).

---

## Prerequisites

- Grove host has the install pipeline shipped (grove#252 + grove#254 merged — bundled into v0.26.0+ if released as such; otherwise main).
- The agent you're migrating has a working `bot.yaml` configuration and a Discord bot identity (token, bot ID, guild, channel).
- You have access to the agent's persona file at `~/.config/<host>/personas/<name>.md`.

---

## Step 1 — Save the secrets out-of-band

`bot.yaml` carries operator-supplied secrets that **don't belong in a manifest** (token most importantly). Before any `bot.yaml` mutation:

```bash
mkdir -p ~/.config/grove/.<name>-reinstall-backup
grep -A6 "instanceId: discord-<name>" ~/.config/grove/bot.yaml \
  > ~/.config/grove/.<name>-reinstall-backup/discord-block.yaml.txt

# Also extract token + ids for the install command
TOKEN=$(awk "/instanceId: discord-<name>/,/personaFile:/" ~/.config/grove/bot.yaml \
  | grep token: | sed 's/.*token: //')
BOT_ID="<from trustedAgentBots or manifest.identity if you already have one>"
GUILD="<from bot.yaml guildId>"
CHANNEL="<from bot.yaml agentChannelId>"

cat > ~/.config/grove/.<name>-reinstall-backup/secrets.env <<EOF
DISCORD_TOKEN=$TOKEN
DISCORD_BOT_ID=$BOT_ID
DISCORD_GUILD=$GUILD
DISCORD_CHANNEL=$CHANNEL
EOF
chmod 600 ~/.config/grove/.<name>-reinstall-backup/secrets.env
```

Also backup the full bot.yaml and persona file so you have a clean rollback point.

## Step 2 — Reverse-engineer (or hand-write) the manifest

`grove agent export <name>` reverse-engineers a manifest from existing `bot.yaml` entries — but only works for agents whose bot ID is in `trustedAgentBots` with an `agent-<name>` role on someone else's adapter. Agents with their **own** discord adapter (Luna, Echo, Forge, Ivy) need hand-written manifests today.

The manifest captures eight fields per the design (`forge/design/agent-platform.md` lines 159–171):

```yaml
schema: pai/v1
namespace: metafactory
type: agent
name: <name>
version: 0.1.0
tier: custom
description: "..."
author: { name: ..., github: ... }

identity:
  displayName: ...
  shortName: <name>
  oneLine: ...
  channels:
    discord:
      botId: "<derived from token's first base64-decoded segment>"
      preferredChannels: ["#..."]
      dmEnabled: true

persona:
  file: persona.md

blueprints: []                                  # arc bundle composition; empty if none yet
guardrails:
  allowedDirs: [...]
  readOnlyDirs: [...]
  allowedSkills: [...]
  disallowedTools: [...]                        # restrictive role: [Edit, Write, NotebookEdit, Task]
  bashAllowlist:
    rules: [...]
triggers:
  - { type: mention, surface: discord, channelPattern: "*" }
  - { type: cli, command: <name> }
hooks: {}                                       # gated on grove#245 (host-runs-hooks)
roster: [{ name: ..., role: ... }, ...]
instanceStateSpec:
  blueprint: AgentState
  version: ">=0.2.0"
instantiation:
  scope: per-host
```

Reference `forge/agent/arc-manifest.yaml` for a full canonical example.

## Step 3 — Validate the manifest (no install yet)

```bash
cd /path/to/grove
bun src/cli/grove.ts install agent /path/to/<name>/arc-manifest.yaml \
  --discord-token "$TOKEN" \
  --discord-bot-id "$BOT_ID" \
  --discord-guild "$GUILD" \
  --discord-channel "$CHANNEL" \
  --dry-run
```

You should see `✓ manifest validated: name=<name> version=X.Y.Z tier=custom` and a unified diff showing the proposed `bot.yaml` changes.

If validation fails, the error tells you exactly which field is wrong. Common issues:

- `identity.displayName` / `identity.shortName` / `identity.oneLine` required
- `triggers[].type` must be one of `mention | cli | cron | webhook` (no `dm` — DM is part of `identity.channels.discord.dmEnabled`)
- Tier must be one of `custom | community | verified | official`

## Step 4 — Apply the install

Once the dry-run looks right:

```bash
bun src/cli/grove.ts install agent /path/to/<name>/arc-manifest.yaml \
  --discord-token "$TOKEN" \
  --discord-bot-id "$BOT_ID" \
  --discord-guild "$GUILD" \
  --discord-channel "$CHANNEL" \
  --yes
```

The install will:

1. Validate the manifest schema
2. Compute the `bot.yaml` diff (target instance derived from `botId`)
3. Apply atomically with backup-restore semantics (writes `bot.yaml.backup-<timestamp>`)
4. Copy the bundle persona to `~/.config/<host>/personas/<name>.md` (with three-way merge if an operator override exists)
5. Scaffold instance state at `~/.config/<host>/agents/<name>/`

## Step 5 — Restart grove-bot

```bash
pkill -f "grove-bot start"
bun ~/bin/grove-bot start --config ~/.config/grove/bot.yaml > ~/.config/grove/logs/grove-bot.log 2>&1 &
```

Watch the log for `connected as <name>#XXXX` to confirm the bot reconnected with the new config.

## Step 6 — Smoke test

Mention the migrated agent in its channel; verify it responds with the manifest-derived authority. For agents like Echo that participate in the pilot loop, also verify cross-adapter mentions still work (Luna → Echo for review pings).

## Known limitations

- **Conflict on existing trustedAgentBots pin**: if the agent's bot ID is already pinned in `trustedAgentBots` with a different role, install refuses to overwrite (operator-tuned safety). Tracked at [grove#257](https://github.com/the-metafactory/grove/issues/257). Workaround: rename the manifest-derived role OR remove the pin manually before install OR wait for the issue to land a `--skip-trusted-bot-pin` flag.
- **Per-instance trust mapping**: `trustedAgentBots` is global today (one role per bot across all instances). For agents that need different authority on different adapters (Luna runs as `agent-restricted` when pinging Echo, but should run as `agent-luna` on her own adapter), the global pin is too coarse. Tracked at [grove#246](https://github.com/the-metafactory/grove/issues/246) (architectural roadmap).
- **Host-runs-hooks**: lifecycle hook invocation (`onStart`, `onMessageAccepted`, etc.) needs the runtime to invoke blueprint workflows on the agent's behalf. Not yet built. Tracked at [grove#245](https://github.com/the-metafactory/grove/issues/245). Until it lands, `hooks: {}` is the right manifest value.
- **`grove agent export`**: only works for agents whose bot ID is in `trustedAgentBots` with an `agent-<name>` role. Doesn't reverse-engineer agents with their own adapter. Future enhancement.

## Reverting

If something goes wrong:

```bash
# Restore the pre-install bot.yaml
cp ~/.config/grove/bot.yaml.backup-<timestamp> ~/.config/grove/bot.yaml

# Restart grove-bot
pkill -f "grove-bot start" && bun ~/bin/grove-bot start --config ~/.config/grove/bot.yaml > /tmp/grove-bot.log 2>&1 &
```

The persona file's safe-rename backup (if you're using uninstall) is at `~/.config/grove/personas/<name>.md.backup-<timestamp>` — `cp` it back if needed.

---

## For JC migrating Ivy + Holly

JC operates two agents — **Ivy** (coordinator, peer to Luna) and **Holly** (TBD by JC). Same recipe for both, JC-side:

### Per-agent steps

1. **Hand-write the manifest.** Pick the closest shape from `../luna/arc-manifest.yaml` (coordinator) or `../echo/arc-manifest.yaml` (reviewer). For each agent:
   - Update `name`, `identity.displayName`, `identity.shortName`, `identity.oneLine`
   - Set `identity.channels.discord.botId` (JC's bot ID for that agent)
   - Set `guardrails.allowedDirs` etc. to match what's in JC's existing `bot.yaml` for that agent
   - **`mentionRole`**: if the agent's authority on its own adapter differs from what it should have when @-mentioning peers (almost always YES for coordinator-class agents), declare `mentionRole: agent-restricted` (or whatever narrow role JC has pinned for the agent in `trustedAgentBots`). Omit only for symmetric release-class agents like Forge.
2. **Save secrets out-of-band** — token + bot ID + guild + channel for each agent into a `~/.config/grove/.<name>-reinstall-backup/secrets.env` file.
3. **Dry-run** the install to see the bot.yaml diff: `grove install agent <ivy-manifest> --discord-token X --discord-bot-id Y --discord-guild Z --discord-channel W --dry-run`.
4. **Apply** with `--yes` (or skip the flag for interactive confirm).
5. **Hot-reload** is automatic — AP-104 (grove#243) means `bot.yaml` changes to `discord[].roles[]` and `trustedAgentBots[]` no longer require a grove-bot restart. Confirm via the bot logs that the role applied.
6. **Smoke test**: ping the migrated agent in its channel; verify it responds with the right authority.

### Holly-specific note

Holly's role/scope is for JC to determine. If Holly is a peer to Ivy (coordinator-class), copy Luna's manifest shape. If reviewer-class, copy Echo's. The mechanics are identical — only the manifest values differ.

### What's preserved across the migration

- The bot tokens (operator-supplied; `--discord-token` keeps them out of the manifest)
- Operator-tuned `trustedAgentBots` pins (the install respects existing pins via `mentionRole` semantics — no `--update-trusted-bot-pin` needed unless JC explicitly wants to change a pin)
- Persona file content (three-way merge if JC has local overrides)
- Instance state at `~/.config/grove/agents/<name>/` (preserved by default; pass `--remove-instance-state` to uninstall to wipe)

The manifest-driven config makes Ivy + Holly survive operator-config moves cleanly — no more hand-edited bot.yaml drift.

---

## Repo home for the manifests

These manifests can live in this `metafactory-agents` directory (on disk during migration), and ultimately in `the-metafactory/metafactory-agents` once that repo is bootstrapped — same convention as the satellite skill bundles (`agent-state`, `release-manager`).

For now, this directory is the working source of truth.
