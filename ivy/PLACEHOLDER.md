# Ivy — placeholder

**Ivy is JC's coordinator agent**, part of a separate operator team. The migration of Ivy from hand-configured `bot.yaml` entries to manifest-driven install via the Metafactory Agent install pipeline is **JC's call**, not Andreas/Luna's to drive autonomously.

This directory exists so Ivy has a slot in the `metafactory-agents` family alongside Luna / Echo / Forge once JC is ready.

## What lives here today

Just this `PLACEHOLDER.md` — the migration is JC-driven, on JC's timeline, against JC's instance of grove-bot.

## When JC is ready to migrate

See [`MIGRATING.md`](../MIGRATING.md) at the top of this repo (or this directory) — it captures the full migration recipe Andreas used for Luna + Echo + Forge. The steps generalize to any agent.

The Ivy-specific authoring artifacts (`arc-manifest.yaml`, `persona.md`) will land here once JC produces them.

## Ivy's current shape (for reference)

JC's `bot.yaml` entries today (per the team-identities convention):

- `discord` adapter for Ivy on JC's grove-bot instance
- Ivy's bot ID `1487024060411023491` allowlisted in `trustedAgentBots` for cross-adapter mentions (so Pilot can mention Luna/Echo via Ivy's CLI surface)
- Persona file at JC's host config path

Mapping these to a manifest follows the same shape as `../luna/arc-manifest.yaml` and `../echo/arc-manifest.yaml` — same 8 fields, JC-specific values.
