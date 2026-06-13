# Odysseus Agent Integrations

This directory contains the skill / plugin bundles that let third-party coding-agent
CLIs talk to a running Odysseus instance. Each subdirectory targets one agent host.

| Subdir   | Target product                                                   | Source of truth                          | Extracts to                    |
|----------|------------------------------------------------------------------|------------------------------------------|--------------------------------|
| `claude/`| **Claude Code** — the Anthropic Claude CLI / SDK / IDE extensions| `anthropics/claude-code`                 | `~/.claude/skills/odysseus/`   |
| `claw/`  | **Claw Code** — the Rust port of the `claude` CLI by ultraworkers| `ultraworkers/claw-code`                 | `~/.claw/skills/odysseus/`     |
| `codex/` | **Codex CLI** — OpenAI's `codex` CLI plugin marketplace          | `openai/codex`                           | `~/plugins/odysseus/` (registered via `codex plugin add`) |

These are **not** Odysseus itself, **not** Claude Desktop add-ons, **not** model routers,
and **not** npm packages. They are skill bundles consumed by the listed agent CLIs.

## What ships in each bundle

The spec file is `SKILL.md` (Markdown with a YAML frontmatter block — not a standalone
`skill.yaml`). The frontmatter declares the skill `name` and `description`; the body
documents the surfaces (todos, calendar, memory, email, documents, cookbook) and the
exact `/api/codex/*` endpoints they map to.

```
<agent>/
├── README.md                              ← per-agent install flow
└── skills/odysseus/
    ├── SKILL.md                           ← the skill definition the agent reads
    └── scripts/odysseus_api.py            ← thin HTTP helper the SKILL invokes
```

`codex/` is the exception — Codex's plugin marketplace format puts `scripts/` at the
bundle root alongside `.codex-plugin/plugin.json`, with `SKILL.md` still under
`skills/odysseus/`:

```
codex/
├── README.md
├── .codex-plugin/plugin.json              ← marketplace manifest
├── scripts/odysseus_api.py                ← top-level (referenced by plugin.json)
└── skills/odysseus/SKILL.md
```

## How they all talk to Odysseus

All three bundles call the **same** scope-gated runtime API at `/api/codex/*`. The
`codex` path in the URL is historic — every agent integration shares it. What differs
between bundles is only:

1. **Download URL** — `/api/claude/plugin.zip`, `/api/claw/plugin.zip`, or
   `/api/codex/plugin.zip`. These shims live in `routes/codex_routes.py` and
   each one streams its own subdirectory as a zip.
2. **Extract path** — see the table above.
3. **Setup terminology** — the UI shows agent-specific install commands but the
   underlying token + scopes are identical.

## Token and scope model

The user generates one `ApiToken` row in **Odysseus → Settings → Integrations → Add
Integration → \<agent\> Agent**. The token:

- Is a single bearer string prefixed `ody_`.
- Carries a comma-separated `scopes` field (e.g. `todos:read,todos:write,memory:read`).
- Is checked server-side on every `/api/codex/*` call. A request outside the token's
  scopes gets `403`, regardless of which agent is calling.

Scope toggles are user-facing in the Settings UI. There is no DB-level discriminator
distinguishing "Claude Agent" from "Claw Agent" tokens — they're just `ApiToken` rows
whose `name` happens to start with `"Claude Agent"`, `"Claw Agent"`, or
`"Codex Agent"`. The UI uses that name prefix to group cards.

## When to use which

| You want to use Odysseus data from… | Install      |
|-------------------------------------|--------------|
| Claude Code (CLI, SDK, VS Code)     | Claude Agent |
| Claw Code (Rust `claw` binary)      | Claw Agent   |
| OpenAI Codex CLI                    | Codex Agent  |

The three integrations are independent. Installing one does not affect the others.
You can run all three on the same machine pointing at the same Odysseus instance,
each with its own token and its own scope toggles.

## Installing the host agent

The bundles assume the host agent is already installed. They do **not** install it.

| Agent       | Real install (do not trust drive-by curl-pipe-bash snippets)                                    |
|-------------|------------------------------------------------------------------------------------------------|
| Claude Code | See `anthropics/claude-code` README. Typical path: `npm i -g @anthropic-ai/claude-code`.       |
| Claw Code   | `git clone https://github.com/ultraworkers/claw-code && cd claw-code/rust && cargo build --workspace` — build from source only. Do **not** `cargo install claw-code` — it installs a deprecated stub. |
| Codex CLI   | See `openai/codex` README.                                                                     |

## Forbidden bypass

Even if you have shell access to the Odysseus container, agents must not import app
internals, query SQLite directly, call MCP helper modules, or read app state through
any path other than the scoped `/api/codex/*` HTTP API. That contract is enforced in
each `SKILL.md` under the **Forbidden Bypass Pattern** / **Safety** section. If a
toggle is off, the agent must surface that to the user, not work around it.
