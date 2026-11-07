# FastCAC skills

Skills that teach AI apps and coding agents how to work with [FastCAC](https://fastcac.com), the
fast way to register a business with Nigeria's Corporate Affairs Commission (CAC).

| Skill | For | What it teaches |
|---|---|---|
| [`fastcac`](plugins/fastcac/skills/fastcac/SKILL.md) | Anyone using ChatGPT, Claude, Cursor or another AI app | How to use FastCAC's MCP server: check a business name, find the right line of business, explain CAC's rules and fees, and register a business name with you |

## Install

Any agent the [skills CLI](https://github.com/vercel-labs/skills) supports (Claude Code, Cursor,
Codex, OpenCode and more):

```bash
npx skills add devgenix/fastcac-skills
```

Add `--skill fastcac` to install just that one, or `-g` to install it for every project.

Claude Code, as a plugin that also connects FastCAC's MCP server:

```
/plugin marketplace add devgenix/fastcac-skills
/plugin install fastcac@fastcac
```

Claude (claude.ai and the desktop app): download
[`fastcac.zip`](https://github.com/devgenix/fastcac-skills/releases/latest/download/fastcac.zip),
then upload it under Customize > Skills (the "+" button). Code execution must be on, under
Settings > Capabilities.

## The MCP server

The `fastcac` skill works with FastCAC's MCP server at `https://api.fastcac.com/mcp`. The name,
fee and register tools are free and need no account; registering a business asks you to sign in
to FastCAC the first time. Setup for each app is at [fastcac.com/ai](https://fastcac.com/ai).

## Help

hello@fastcac.com, or WhatsApp +234 816 181 3856.
