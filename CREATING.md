# Creating a metafactory agent — from zero to live in Discord

**Audience:** an operator who wants to bring a new agent (their own personality, scope, authority) into the metafactory ecosystem and have it respond to mentions in Discord through grove-bot.

**Source of truth for the design:** `forge/design/agent-platform.md` (manifest schema) + `forge/design/forge.md` (Forge as the canonical example).

**Status (2026-04-27):** Phase 9 of mf#392 just shipped. The end-to-end loop is plug-and-play:

```bash
arc install <git-url>           # operator runs once per agent repo
grove install agent <name>      # writes bot.yaml, materialises persona, scaffolds instance
# config-watcher picks up the bot.yaml change; no grove-bot restart needed
```

This guide takes you from "I have an idea for an agent" to "the bot is live on Discord."

---

## What is a metafactory agent

Three things, bundled:

1. **Manifest** (`arc-manifest.yaml`) — the canonical declaration. Identity, persona reference, guardrails, triggers, hooks, blueprints to compose, instantiation scope. The host (grove) reads this and derives its own `bot.yaml` from it. You don't hand-edit `bot.yaml` anymore.
2. **Persona** (`persona.md`) — the agent's voice, defaults, routing table for which blueprint to invoke when. This is what the Claude Code session reads at startup.
3. **Optional: scripts/ + bridge CLAUDE.md** — for agents that need scaffolding hooks or per-instance lifecycle logic. Not needed for simple chat-only agents.

The agent's name on disk and in the registry is `agent-<shortname>` (canonical convention since grove#262). For example: `agent-luna`, `agent-echo`, `agent-forge`. You write `name: agent-distiller` in the manifest; the host derives `shortName: distiller` for path/dir uses.

---

## Repo shape — pick one

### Shape A: standalone repo (forge-shape)

For agents with their own design lifecycle (own retro cadence, own design docs, own release notes — Forge is the canonical example):

```
the-metafactory/<your-agent>/
├── README.md
├── design/
│   └── <agent-name>.md          ← the design doc
└── agent/
    ├── arc-manifest.yaml         ← THE manifest
    ├── persona.md                ← THE persona
    ├── CLAUDE.md                 ← optional bridge file (per-instance)
    ├── scaffold-instance.sh      ← optional scaffolding script
    └── README.md
```

`grove install agent <name>` knows about both shapes and resolves to `<pkgDir>/<name>/agent/arc-manifest.yaml` automatically (grove#264).

### Shape B: flat in this repo (`the-metafactory/agents`)

For lighter agents that don't need their own design machinery — Luna, Echo, the future Distiller:

```
the-metafactory/agents/
├── luna/
│   ├── arc-manifest.yaml
│   └── persona.md
├── echo/
│   ├── arc-manifest.yaml
│   └── persona.md
└── <your-agent>/
    ├── arc-manifest.yaml
    └── persona.md
```

`grove install agent <name>` resolves to `<pkgDir>/<name>/arc-manifest.yaml` (root shape).

**Recommendation:** start with Shape B unless you know upfront you want an independent design + release cadence. Moving B → A later is mechanical (mv + add the `agent/` wrapper).

---

## Step 1 — Create the Discord bot identity

Before writing any YAML, create the Discord bot. The manifest needs the bot ID; the install command needs the token, the guild ID, and the channel ID.

1. <https://discord.com/developers/applications> → **New Application** → name it after your agent (e.g. "Distiller").
2. **Bot** tab → **Add Bot** → copy the **token** (you'll only see it once; store in a password manager or `~/.config/grove/.<name>-secrets.env` with `chmod 600`).
3. **OAuth2 → URL Generator** → scopes: `bot`, `applications.commands`. Bot permissions: at minimum `Send Messages`, `Read Message History`, `Mention @everyone, @here, and All Roles` (off — agents don't @everyone).
4. Visit the generated URL → invite the bot to your server.
5. Note the **bot user ID** (right-click the bot in Discord with Developer Mode on → Copy ID), the **guild ID** (server icon → Copy ID), and the **channel ID** for the agent's home channel.

Result: `BOT_TOKEN`, `BOT_ID`, `GUILD_ID`, `CHANNEL_ID`.

---

## Step 2 — Write the manifest

Copy `luna/arc-manifest.yaml` as a starting point. Adjust the eight required field blocks:

```yaml
schema: pai/v1
namespace: metafactory
type: agent
name: agent-<shortname>           # e.g. agent-distiller
version: 0.1.0
tier: custom

description: "<one-line elevator pitch>"

author:
  name: <your name>
  github: <your gh login>

# ─── Identity ────────────────────────────────────────────────────────────
identity:
  displayName: <Capitalized>      # e.g. Distiller
  shortName: <lowercase>          # e.g. distiller
  oneLine: <one-line for /help>
  channels:
    discord:
      botId: "<from Step 1>"
      preferredChannels: ["#backlog", "#planning"]    # for routing hints
      dmEnabled: false             # default-deny; override per agent

# ─── Persona ─────────────────────────────────────────────────────────────
persona:
  file: persona.md                # relative to the manifest

# ─── Cross-adapter authority ─────────────────────────────────────────────
# Required when this agent will be mentioned by other bots (Pilot loop, etc).
# The role MUST exist on every Discord-instance's roles[] in the host's bot.yaml.
mentionRole: agent-restricted    # or agent-distiller (a role you also define)

# ─── Blueprints (arc bundles to compose) ─────────────────────────────────
# Empty for now if your agent just chats. Add as you build out workflows.
blueprints: []

# ─── Guardrails (declarative authority) ──────────────────────────────────
guardrails:
  allowedDirs:
    - "~/Developer/<your-project>"
  readOnlyDirs:
    - "~/Developer/compass"        # SOPs are read-only for everyone
  allowedSkills:
    - <skill-name>                 # e.g. backlog-create — empty list = all denied
  disallowedTools:
    - Edit                         # for review-only / chat-only agents
    - Write
    - NotebookEdit
  bashAllowlist:
    rules:
      - pattern: "^gh\\s+(pr|issue|api|search|label)\\b"
      - pattern: "^git\\s+(status|log|diff|show|fetch)\\b"

# ─── Triggers (when this agent activates) ────────────────────────────────
triggers:
  - type: mention
    surface: discord
    channelPattern: "*"            # any channel, or "#backlog|#planning"
  - type: cli
    command: <name>                # for `<name> <subcommand>` shortcuts

# ─── Hooks (lifecycle) — Phase 10 work; leave empty for now ──────────────
hooks: {}

# ─── Roster (sibling agents this one knows about) ────────────────────────
roster:
  - name: agent-luna
    role: coordinator
  - name: agent-echo
    role: reviewer

# ─── Instance state ──────────────────────────────────────────────────────
instanceStateSpec:
  blueprint: AgentState
  version: ">=0.2.0"

instantiation:
  scope: per-host                  # one instance per host (grove deployment)
```

**Naming reminder:** `name` is canonical (`agent-distiller`). `identity.shortName` is the stripped form (`distiller`). The host respects this asymmetry — `name` shows up in identity prose, `shortName` shows up in path/dir derivations.

---

## Step 3 — Write the persona

`persona.md` is read at session startup. It defines the agent's voice and the routing table for which blueprint to invoke when.

Minimum structure (mirror `luna/persona.md` or `echo/persona.md`):

```markdown
# <Name> — <one-line role>

You are <Name>, the metafactory ecosystem's <role>. Speak in first person.
Address the operator by name.

## Self-identity

<Name>'s Discord user ID is set by the operator at install time
(`identity.channels.discord.botId` in the manifest). When you see your
own @-mention in incoming messages, that is you — recognize self-mentions
and respond. The operator's manifest is your source of truth for who you
are; do not invent or hard-code IDs.

## Introducing yourself

When asked who you are, keep it tight:

> *I'm <Name> — <one-line role>. I <what you do>, defer to the operator
> at every irreversible step.*

## Voice

- **<adjective>.** <reason>
- ...

## What I do (routing table)

When the work calls for...

- **<task category>** → invoke `<Blueprint>/<Workflow>`. Specifically: ...
- ...

## Hard rules

- Never <X> without operator confirmation.
- Always <Y>.
- ...
```

Look at `luna/persona.md` (coordinator), `echo/persona.md` (reviewer), and Forge's persona (release agent) for three different voice templates.

---

## Step 4 — Test the manifest locally (before publishing)

```bash
cd /path/to/grove
bun src/cli/grove.ts install agent /path/to/your/agent-<name>/arc-manifest.yaml \
  --dry-run \
  --discord-token "$BOT_TOKEN" \
  --discord-bot-id "$BOT_ID" \
  --discord-guild "$GUILD_ID" \
  --discord-channel "$CHANNEL_ID"
```

`--dry-run` validates the manifest, computes the bot.yaml diff, and prints what would change without writing. Iterate on the manifest until the diff matches what you want.

---

## Step 5 — Publish (commit + push)

If using **Shape B** (this repo), open a PR adding the `<name>/` directory. After merge, the manifest is publicly available and `arc install` works against `https://github.com/the-metafactory/agents` (or, for now, against your fork until we wire up subdirectory installs).

If using **Shape A** (standalone repo), the repo IS the bundle.

---

## Step 6 — Install into grove

```bash
arc install https://github.com/<owner>/<repo>     # if not already installed
grove install agent <name> \
  --discord-token "$BOT_TOKEN" \
  --discord-bot-id "$BOT_ID" \
  --discord-guild "$GUILD_ID" \
  --discord-channel "$CHANNEL_ID" \
  --yes
```

What `grove install agent <name>` does (per grove#264):

1. **Resolves** the name → `~/.config/metafactory/pkg/repos/<name>/arc-manifest.yaml` (root shape) or `<pkgDir>/<name>/agent/arc-manifest.yaml` (forge-shape).
2. **Validates** the manifest schema.
3. **Computes a bot.yaml diff** — adds the `agent-<name>` role, a `discord-<name>` adapter (token + guild + channel), and a `trustedAgentBots` row binding the bot ID to the role.
4. **Applies atomically** with backup at `~/.config/grove/bot.yaml.backup-<timestamp>`.
5. **Materialises the persona** at `~/.config/grove/personas/<name>.md` (tier-3 default per grove#263) and scaffolds `~/.config/grove/agents/<name>/` (CLAUDE.md, dashboard.md, context/, retros/).
6. `config-watcher` picks up the bot.yaml change within ~1s; grove-bot reloads without restart.

---

## Step 7 — Verify it's live

In the Discord channel from Step 1:

```
@<Your-Agent> ping
```

Expected: the bot mentions you back with a short greeting from its persona. Check:

- `~/.config/grove/agents/<name>/dashboard.md` regenerates after the response (proves AgentState hooks fire if you wired them).
- The grove dashboard at <https://grove.meta-factory.ai> shows the agent card.
- `discord post --thread <id> "test"` from another agent works if you set up trustedAgentBots correctly.

If the bot doesn't respond, check stderr — the most common failures:

- **Wrong bot ID** in the manifest → grove-bot ignores its own messages but not the new bot's. Fix the `botId` and `grove install agent` again.
- **Token mismatch** → grove-bot can't connect. The instance's `token` is per-adapter; double-check Step 6's `--discord-token`.
- **Channel not in `agentChannelId`** → the bot connects but doesn't see messages. Update the channel on the manifest or pass `--discord-channel`.

---

## Step 8 — Round-trip-export to inspect

After install, `grove agent export <name>` reverse-engineers the manifest from your bot.yaml. The output should match what you wrote (modulo placeholders for fields bot.yaml doesn't carry: `blueprints[]`, `hooks{}`, `roster[]`).

```bash
bun src/cli/grove.ts agent export <name> > /tmp/round-trip.yaml
diff /path/to/your/agent-<name>/arc-manifest.yaml /tmp/round-trip.yaml
```

This is your sanity check that the host correctly understood your manifest. Per grove#262's canonical contract, the exported `name` is `agent-<shortname>` regardless of input shape.

---

## Conventions & constraints (the "musts")

These are non-negotiables learned from shipping Luna, Echo, Forge, and the Phase 9 migration:

- **Canonical naming.** `agent-<shortname>` everywhere. Use `agent-naming.ts` primitives in any tooling — never re-implement `stripAgentPrefix` or the regex.
- **Round-trip parity.** Install + uninstall + scaffold + export agree on the same name/path derivations. If they drift, you'll see operator copy/paste failures.
- **Atomic file ops.** Persona installs and bridge-file writes use `atomicWriteWithBackup` (grove#263). Don't bake new partial-write paths.
- **Persona 3-tier precedence** — tier-1 instance override → tier-2 manifest bundle → tier-3 Discord-instance default. Bundle bumps don't silently overwrite operator edits (bundle-only no-plant rule).
- **Path-traversal hardening.** Any name → path resolution validates the joined path is under its root, rejects `..`.
- **No auto-shell-out.** Operator confirms every irreversible step. No CLI silently fetches things.
- **Multi-host roles[].** If you set `mentionRole: agent-restricted`, the role MUST exist on every Discord-instance's `roles[]` in the host's bot.yaml. Single-host today; design for multi-host.

---

## Going further

- **Compose blueprints.** Most agents start with `blueprints: []` for chat-only behavior. As you add workflows, list them under `blueprints[]` with version ranges; arc resolves on install.
- **Add a scaffold script.** For agents that need per-instance setup beyond the default scaffold (e.g. seeding a SQLite schema, generating a config file from environment), add `scripts/scaffold-instance.sh` and let the bundle's onInstall hook invoke it.
- **Custom triggers.** Beyond `mention` and `cli`, you can register `cron` triggers for periodic work (Forge runs a weekly retrospective on Mondays) and `webhook` triggers (Phase 10).

---

## When you get stuck

- Manifest validation failures: `bun src/cli/grove.ts install agent <path> --dry-run` shows the schema errors.
- Bot won't respond: check `grove-bot` stderr (`journalctl --user -u grove-bot` or wherever it's run).
- Want to migrate an existing hand-configured agent: see `MIGRATING.md` in this repo.
- Want to see how `Luna`, `Echo`, `Forge` are built: read their manifests and personas. They're the reference set.

---

## Related

- `forge/design/agent-platform.md` — manifest schema + four-layer split
- `forge/design/forge.md` — Forge as the canonical example
- `the-metafactory/grove#262` — canonical naming convention (merged 2026-04-27)
- `the-metafactory/grove#263` — per-agent persona resolution (merged 2026-04-27)
- `the-metafactory/grove#264` — `grove install agent <name>` registry-resolved (merged 2026-04-27)
- `arc#103` — `arc install <git-url>` for `type: agent` (merged 2026-04-27)
- `the-metafactory/forge#12` — Forge's agent-lifecycle workflows (open — once shipped, Forge automates Steps 5-6 of this guide)
