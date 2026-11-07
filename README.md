# FastCAC skills

Skills that teach AI apps and coding agents how to work with [FastCAC](https://fastcac.com), the
fast way to register a business with Nigeria's Corporate Affairs Commission (CAC).

| Skill | For | What it teaches |
|---|---|---|
| [`fastcac`](plugins/fastcac/skills/fastcac/SKILL.md) | Anyone using ChatGPT, Claude, Cursor or another AI app | How to use FastCAC's MCP server: check a business name, find the right line of business, explain CAC's rules and fees, and register a business name with you |
| [`fastcac-api-basics`](plugins/fastcac-developers/skills/fastcac-api-basics/SKILL.md) | Developers building on the [FastCAC API](https://docs.fastcac.com) | Keys and test/live mode, errors, idempotency, rate limits, pagination, the wallet, webhooks and signature checks, going live |
| [`fastcac-registrations`](plugins/fastcac-developers/skills/fastcac-registrations/SKILL.md) | Developers | Registering a business name by API: name checks, NIN lookups, documents, submit, CAC queries, the certificate, testing every outcome |
| [`fastcac-business-data`](plugins/fastcac-developers/skills/fastcac-business-data/SKILL.md) | Developers | Searching, looking up, verifying and monitoring businesses on CAC's register, and NIN lookups |

## Install

Any agent the [skills CLI](https://github.com/vercel-labs/skills) supports (Claude Code, Cursor,
Codex, OpenCode and more):

```bash
npx skills add devgenix/fastcac-skills
```

Add `--skill <name>` to install just one (for example `--skill fastcac`, or the three developer
skills with `--skill fastcac-api-basics fastcac-registrations fastcac-business-data`),
or `-g` to install for every project.

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

The three developer skills are also served by docs.fastcac.com (`npx skills add https://docs.fastcac.com`).
This repository is the source: change them here first, then copy them to `.mintlify/skills/` in
fastcac-developer-docs in the same change. They never contain prices; they link
[fastcac.com/api/pricing](https://fastcac.com/api/pricing).

## Help

hello@fastcac.com, or WhatsApp +234 816 181 3856.
