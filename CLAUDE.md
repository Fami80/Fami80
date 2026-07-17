# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@shared/CLAUDE.md

# Arcads-specific session rules

## What this repo is

This is not application code — it's an **agent skill pack**. The "skills" are markdown instructions (`SKILL.md` + `reference.md` + prompting guides) that direct Claude Code / Cursor to call the [Arcads](https://arcads.ai/?via=claude-code) external API (`https://external-api.arcads.ai`) for AI video/image generation, plus a handful of bash/Python helper scripts. There is no application to build or deploy; the "product" is the agent's behavior when following these skills.

## Commands

- **First-time setup:** `./scripts/setup.sh` — creates `.env` from `.env.example`, validates and saves Arcads credentials, creates `MASTER_CONTEXT.md` from the template, syncs skills, verifies connectivity.
- **Verify API connectivity:** `./scripts/check-arcads-env.sh` — sends a test request to `/v1/products` with the configured auth.
- **Sync skill edits:** `./scripts/sync-skill.sh` — run after editing anything under `skills/` or `shared/skills/`. Copies canonical skill source into the gitignored `.claude/skills/` and `.cursor/skills/` runtime directories that Claude Code / Cursor actually load. **Edits to skill files have no effect until this runs** (the SessionStart hook also runs it automatically on every Claude Code session open).
- **No build, lint, or test suite.** There's no `package.json`/`pytest`/`Makefile` — verification means running the relevant shell script or exercising a skill's workflow against the live Arcads API.

## Architecture

**Two skill trees with different ownership — know which one you're editing:**

- `skills/` — canonical, hand-edited skills specific to *this* repo: `arcads-external-api` (the core API skill), `generate-youtube-thumbnail`, `chatgpt-image-ad`, `nano-banana-image-ad`, `image-ad-clone`.
- `shared/` — propagated verbatim from an upstream `gen-ai-core` repo (see `shared/README.md`). **Never hand-edit anything under `shared/`** — it's overwritten on the next upstream sync. This includes `shared/CLAUDE.md` (imported above), `shared/scripts/*`, and the cross-API skills (`pixar-style-ad`, `claymation-ad`, `caption-video`, `meta-ad-builder`, `image-ad-prompting`).

Both trees are copied by `scripts/sync-skill.sh` into `.claude/skills/` and `.cursor/skills/` — the gitignored, generated directories the editors actually read at runtime. If a skill change doesn't seem to take effect, check whether sync ran.

**Head/tail file split** — several top-level files are themselves generated from upstream with a repo-specific tail you *do* hand-edit:
- `AGENTS.md` (generated) ← edit `AGENTS.tail.md`
- `MASTER_CONTEXT.template.md` (generated) ← edit `MASTER_CONTEXT.template.tail.md`
- `.gitignore` (generated) ← edit `.gitignore.tail`

`CLAUDE.md` itself is **not** split this way — edit it directly (it only `@`-imports `shared/CLAUDE.md` for the universal cross-repo rules).

**Runtime state, not code:**
- `MASTER_CONTEXT.md` (gitignored; created from the template by setup) is the persistent memory layer. The agent reads it at the start of every session and writes learnings, credit costs, and a dated changelog back to it — this is the most important piece of state in the repo outside the skills themselves.
- `logs/arcads-api.jsonl` is an append-only audit log of every generation call (model, config, `creditsCharged`). It's the primary source for credit-cost estimates, checked before `MASTER_CONTEXT.md`'s manual cost table.
- `references/` (gitignored) holds user-dropped images the skills consume: `influencers/`, `products/`, `aesthetics/`.

**Request flow:** all generation goes through one API documented in `skills/arcads-external-api/reference.md`. That skill's `SKILL.md` has a decision tree routing a user's request to the right model/endpoint and the matching `prompting/prompt-library/*.md` formula — start there when tracing how a feature/workflow is supposed to behave.

## Key conventions enforced by the skills

Preserve these invariants if you're editing skill files — they're load-bearing for cost safety and output quality, not stylistic:

- **Credit costs are always estimates, confirmed before generation**, sourced in priority order: `logs/arcads-api.jsonl` → `MASTER_CONTEXT.md` rate table → ask the user. Never invented.
- **Dialogue approval is a separate gate from cost approval** for any speaking video — one confirmation never implies the other.
- **Generated still images get a mandatory QA pass** (anatomy/artifact check) with up to 2 regenerate retries before a defective frame is ever handed off.
- `.env` values containing special characters (`{`, `[`, `*`) must be single-quoted.
- `.env`, `MASTER_CONTEXT.md`, `references/`, `logs/`, and `outputs/` are gitignored by design — don't try to commit them.
- Every Meta ad created via the `meta-ad-builder` skill is created **PAUSED** — the skills never auto-launch ads.

## gstack (recommended)

This project uses [gstack](https://github.com/garrytan/gstack) for AI-assisted workflows.
Install it for the best experience:

```bash
git clone --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && ./setup --team
```

Skills like /qa, /ship, /review, /investigate, and /browse become available after install.
Use /browse for all web browsing. Use ~/.claude/skills/gstack/... for gstack file paths.
