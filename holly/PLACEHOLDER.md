# Holly — placeholder

**Holly is JC's second agent**, part of a separate operator team from Andreas/Luna. The migration of Holly from hand-configured `bot.yaml` entries to manifest-driven install via the Metafactory Agent install pipeline is **JC's call**, on JC's timeline, against JC's instance of grove-bot.

This directory exists so Holly has a slot in the `metafactory-agents` family alongside Luna / Echo / Forge / Ivy once JC is ready.

## When JC is ready to migrate

Same recipe as Luna + Echo. See [`MIGRATING.md`](../MIGRATING.md) — the section "For JC migrating Ivy + Holly" walks through the JC-specific steps.

Holly's authoring artifacts (`arc-manifest.yaml`, `persona.md`) will land here once JC produces them.

## What we know about Holly

JC has surfaced Holly as a second agent in JC's operator team. Specific role + scope is JC's to determine and document in the manifest. The migration mechanics are the same as Luna's + Echo's — only the values differ.

If Holly's authority on JC's instance is asymmetric (broad on Holly's own adapter; narrow when @-mentioning peers), declare `mentionRole: <role>` in Holly's manifest the same way Luna's manifest does. If symmetric, omit the field — defaults to `agent-holly`.

## References

- [`../luna/arc-manifest.yaml`](../luna/arc-manifest.yaml) — closest shape if Holly is coordinator-class
- [`../echo/arc-manifest.yaml`](../echo/arc-manifest.yaml) — closest shape if Holly is reviewer-class
- [`../ivy/PLACEHOLDER.md`](../ivy/PLACEHOLDER.md) — Holly's sibling placeholder for Ivy
