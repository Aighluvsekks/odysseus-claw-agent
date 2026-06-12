# Odysseus Claw Code Integration

This directory contains the Claw Code skill bundle for Odysseus.

## User Flow

1. Open Odysseus Settings > Integrations.
2. Add a Claw Agent.
3. Copy the full setup commands shown after the generated token.
4. Toggle the tools Claw is allowed to use.
5. Configure the terminal Claw Code session:

```bash
export ODYSSEUS_URL=http://your-odysseus-host:7000
export ODYSSEUS_API_TOKEN=ody_generated_token
mkdir -p ~/.claw
curl -fsSL -H "Authorization: Bearer $ODYSSEUS_API_TOKEN" "$ODYSSEUS_URL/api/claw/plugin.zip" -o /tmp/odysseus-claw-skill.zip
python3 -m zipfile -e /tmp/odysseus-claw-skill.zip ~/.claw/
```

Claw Code auto-loads anything under `~/.claw/skills/` (see its skill discovery
roots — `.claw/skills`, `.omc/skills`, `.agents/skills`, `.codex/skills`,
`.claude/skills` and their `~/` equivalents), so the `odysseus` skill is
available in any session that has `ODYSSEUS_URL` and `ODYSSEUS_API_TOKEN` in its
environment.

## What's in the bundle

- `skills/odysseus/SKILL.md` — the skill definition Claw Code reads.
- `skills/odysseus/scripts/odysseus_api.py` — small helper that calls the scoped
  `/api/codex/*` endpoints (these are the canonical scope-gated agent API; the
  `codex` path is historic and shared by all agent integrations).

## Scope enforcement

The token is scope-gated. Every tool surface is checked server-side in Odysseus,
so even if Claw tries to call a forbidden endpoint, it gets `403` until the
user enables the matching toggle in Settings > Integrations > Claw Agent.
