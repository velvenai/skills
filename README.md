# Velven skills

Agent Skills (skills.sh format), plugins and the MCP server for [Velven](https://velven.ai), the community
marketplace and host for spaces built with AI.

- `onboard/` teaches a coding agent to publish a space on Velven: `velven publish` makes a private preview, `--prod`
  puts it live after Velven's safety check, and it works before the creator has an account (an unlisted page with a
  claim token to keep it). It fills `velven.json` from the project, links an already deployed site as the side path,
  claims a space that is already on Velven, and adds sign-in, leaderboards, saves, achievements and rooms through the
  Velven SDK.
- `plugins/velven/` is the same skill with Velven's MCP server, as a plugin for Claude Code and Codex.

## Install

Skill only, for any agent that reads skills.sh skills:

```
npx skills add velvenai/skills
```

Claude Code, the plugin (the skill and the MCP server):

```
/plugin marketplace add velvenai/skills
/plugin install velven@velven
```

Codex: add this repository as a plugin marketplace (`.agents/plugins/marketplace.json`) and install `velven`.

Gemini CLI, the MCP server: `gemini extensions install https://github.com/velvenai/skills`.

## The MCP server

`https://mcp.velven.ai/mcp`, Streamable HTTP. Tools: `publish` (one HTML page or a few files, 3 MB in all: a preview,
or live with `prod: true`), `my_spaces`, `versions`, `rollback` and `search_docs`. Without signing in, `publish` makes an
unlisted page and answers its claim token; the account tools ask you to sign in to Velven (OAuth). Any client that takes
a remote MCP server adds it by that address, for example:

```
claude mcp add --transport http velven https://mcp.velven.ai/mcp
```

`server.json` is its entry in the MCP Registry (`io.github.velvenai/velven`).

Docs: https://velven.ai/docs
