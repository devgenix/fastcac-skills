# FastCAC skills

Skills that teach AI apps and coding agents how to work with [FastCAC](https://fastcac.com), the
fast way to register a business with Nigeria's Corporate Affairs Commission (CAC).

| Skill | For | What it teaches |
|---|---|---|
| [`fastcac`](plugins/fastcac/skills/fastcac/SKILL.md) | Anyone using ChatGPT, Claude, Cursor or another AI app | How to use FastCAC's MCP server: check a business name, find the right line of business, explain CAC's rules and fees, and register a business name with you |
| [`fastcac-api`](plugins/fastcac-developers/skills/fastcac-api/SKILL.md) | Developers building on the [FastCAC API](https://docs.fastcac.com) | Everything to integrate: keys and test mode, errors, idempotency, webhooks and the wallet, then registering business names and searching, verifying and monitoring businesses (in `references/`, read when needed) |

## Install

Any agent the [skills CLI](https://github.com/vercel-labs/skills) supports (Claude Code, Cursor,
Codex, OpenCode and more):

```bash
npx skills add devgenix/fastcac-skills
```

Add `--skill fastcac` or `--skill fastcac-api` to install just one, or `-g` to install for every
project.

Claude Code, as a plugin (each also connects an MCP server):

```
/plugin marketplace add devgenix/fastcac-skills
/plugin install fastcac@fastcac              # using FastCAC from an AI app
/plugin install fastcac-developers@fastcac   # building on the FastCAC API
```

`fastcac-developers` also connects the docs search MCP server (`https://docs.fastcac.com/mcp`), so
your agent can read docs.fastcac.com while it works.

Claude (claude.ai and the desktop app): download
[`fastcac.zip`](https://github.com/devgenix/fastcac-skills/releases/latest/download/fastcac.zip),
then upload it under Customize > Skills (the "+" button). Code execution must be on, under
Settings > Capabilities.

## The MCP server

The `fastcac` skill works with FastCAC's MCP server at `https://api.fastcac.com/mcp`. The name,
fee and register tools are free and need no account; registering a business asks you to sign in
to FastCAC the first time. Setup for each app is at [fastcac.com/ai](https://fastcac.com/ai).

## Developer skills and docs.fastcac.com

`fastcac-api` is also served by docs.fastcac.com (`npx skills add https://docs.fastcac.com`). This
repository is the source. docs.fastcac.com can serve only one file per skill, so fastcac-developer-docs
builds a single `SKILL.md` from this folder with `scripts/build_skill.py`: change it here, then rebuild
it there in the same change. They never contain prices; they link
[fastcac.com/api/pricing](https://fastcac.com/api/pricing).

## Help

hello@fastcac.com, or WhatsApp +234 816 181 3856.
