# Croft Agent Plugin

[![Agent Plugin 1.0.0](https://img.shields.io/badge/Agent%20Plugin-1.0.0-1f6ae0)](https://agent-plugins.org/specification)
[![Version](https://img.shields.io/badge/version-1.1.0-1f6ae0)](CHANGELOG.md)
[![License: MIT](https://img.shields.io/badge/License-MIT-1f6ae0)](LICENSE)

> One step to make any compatible AI agent a Croft builder — the tools **and** the know-how.

A portable [Agent Plugin 1.0.0](https://agent-plugins.org/specification) for
**[Croft](https://oncroft.net)** — a passwordless PaaS where you build, deploy, and
maintain small web apps by talking to an AI assistant.

It installs natively in **Cursor** and **Claude Code**, and in any other Agent Plugin
1.0.0-compatible client. Point a client at this plugin and your agent gets two things:

1. **The Croft MCP server** (`mcp.json`) — the tools to create apps, write files, deploy,
   manage connectors and shared data, and more, on the user's Croft workspace.
2. **Skills** (`skills/`) — portable instructions that teach the agent Croft's golden path,
   so it builds apps the right way from the very first turn, even before it reads the live
   conventions.

Most MCP setups hand an agent some tools and hope it figures out the rest. This plugin
gives it the tools *and* the conventions — so it behaves like a Croft expert from message
one.

## What your agent can do once installed

- **Build and ship apps** — the full `create_app → write_files → deploy → poll status`
  path, following Croft's conventions (Node/Hono/SQLite, `$PORT`, `/healthz`, no auth code,
  secrets, file uploads, public paths, scheduled tasks).
- **Reach outside tools** — call external APIs (Stripe, a CRM, email, Azure DevOps, …)
  through **connectors**, never a vendor SDK or a key stored in code.
- **Share data between apps** in the workspace (e.g. one shared contacts list).

## Install

There are no credentials to configure in any client (see [Authentication](#authentication)).

**Cursor (recommended)** — add this repository as a plugin marketplace, then install the
`croft` plugin. This is the route that installs the MCP server **and** the skills:

```
Cursor Settings → Plugins → Add marketplace → https://github.com/on-croft/agent-plugin
```

You can also point Cursor at a local clone of the folder — it reads
`.cursor-plugin/marketplace.json`.

**Cursor, one click, MCP server only** — this
[install link](https://cursor.com/docs/mcp/install-links) adds the Croft MCP server to
Cursor without the marketplace step:

[![Install MCP Server](https://cursor.com/deeplink/mcp-install-dark.svg)](cursor://anysphere.cursor-deeplink/mcp/install?name=croft&config=eyJ0eXBlIjoiaHR0cCIsInVybCI6Imh0dHBzOi8vbWNwLm9uY3JvZnQubmV0In0=)

> **It installs the tools, not the know-how.** Cursor's deeplinks only cover MCP servers —
> there is no plugin-install deeplink — so this route skips `skills/` entirely. Your agent
> gets Croft's tools but not its conventions, which is exactly the gap this plugin exists to
> close. Prefer the marketplace install above; use the link when you only want the server.

<details>
<summary>How the link is built</summary>

`cursor://anysphere.cursor-deeplink/mcp/install?name=<name>&config=<base64>`, where the
config is the base64 of a single server entry:

```json
{"type":"http","url":"https://mcp.oncroft.net"}
```

```bash
printf '{"type":"http","url":"https://mcp.oncroft.net"}' | base64
# eyJ0eXBlIjoiaHR0cCIsInVybCI6Imh0dHBzOi8vbWNwLm9uY3JvZnQubmV0In0=
```

Regenerate it if the endpoint ever changes.

</details>

**Claude Code** — add the marketplace and install:

```bash
claude plugin marketplace add on-croft/agent-plugin
claude plugin install croft@croft
```

**Any other Agent Plugin 1.0.0 client** — point it at this repository, or drop the
directory into the client's plugin location. It discovers the root `plugin.json`,
`mcp.json`, and `skills/` automatically.

```
https://github.com/on-croft/agent-plugin
```

## What's inside

```
agent-plugin/
├── plugin.json                     # Agent Plugin 1.0.0 manifest
├── mcp.json                        # Croft MCP server (remote, streamable-http)
├── .cursor-plugin/
│   ├── marketplace.json            # Cursor marketplace manifest (this repo = one plugin)
│   ├── plugin.json                 # Cursor plugin manifest
│   └── mcp.json                    # same server, Cursor's `type: http` spelling
├── .claude-plugin/
│   ├── marketplace.json            # Claude Code marketplace manifest
│   └── plugin.json                 # Claude Code plugin manifest (points at ../mcp.json)
├── assets/
│   ├── logo.svg                    # Croft brand icon — shown by Cursor's marketplace
│   └── logo.png                    # 1024px raster of the same icon
└── skills/
    ├── building-croft-apps/        # the create → write → deploy → verify golden path
    ├── croft-connectors/           # calling external APIs via connectors (no SDKs, no keys)
    └── croft-shared-data/          # sharing SQLite data between apps
```

The same `skills/` directory and the same MCP endpoint serve every client — the
client-specific directories only carry each host's manifest format. Keep the `version`
field in sync across the four manifests when releasing.

`assets/logo.svg` is Croft's brand icon, copied from the marketing site. Only Cursor renders
it (via `logo` in both of its manifests) — the Agent Plugin 1.0.0 spec and Claude Code have
no logo field. It's the light tile, which carries its own background so it reads on light
and dark marketplace themes alike.

## The MCP server

`mcp.json` declares Croft's remote MCP endpoint:

```json
{
  "mcpServers": {
    "croft": { "type": "streamable-http", "url": "https://mcp.oncroft.net" }
  }
}
```

Cursor spells the same transport `"type": "http"`, so `.cursor-plugin/mcp.json` carries an
identical server under that name. Both point at `https://mcp.oncroft.net`.

## Authentication

**Per-user OAuth — never a static key**, which is why no credentials appear in `mcp.json`
(the spec forbids credentials in package data anyway). On first connect the agent's client
performs OAuth 2.1 (PKCE + dynamic client registration) against `https://mcp.oncroft.net`;
the user signs in and authorises their workspace, and every request is then scoped to that
workspace with the user's role. Only workspace builders (owner/admin/creator) can use the
build tools.

> In agents that authenticate the MCP server per end-user (e.g. Microsoft Copilot's **User**
> auth mode), each user connects with their own Croft identity and role — which is the
> intended setup. Avoid shared/maker-credential modes.

## The skills

| Skill | Use it when |
|-------|-------------|
| `building-croft-apps` | Creating, changing, deploying, or fixing any Croft app — the required `create_app → write_files → deploy → poll status` workflow and Croft's app conventions. |
| `croft-connectors` | The app must call an external API (Stripe, Azure DevOps, a CRM, email…) — always through a connector, never a vendor SDK or a stored key. |
| `croft-shared-data` | Two apps in the workspace should reuse the same data (e.g. a shared contacts list), or one app needs to read another's data. |

The skills summarise Croft's `get_platform_conventions`, which the agent should still call
at runtime — it returns the authoritative, live golden path plus the workspace's actual
connectors and shared apps.

## Learn more

- **Croft** — [oncroft.net](https://oncroft.net)
- **Set it up in your assistant** — [oncroft.net/connect/agent-plugin](https://oncroft.net/connect/agent-plugin/)
- **Agent Plugin 1.0.0 spec** — [agent-plugins.org/specification](https://agent-plugins.org/specification)
- **Cursor plugins** — [cursor.com/docs/plugins](https://cursor.com/docs/plugins)
- **Claude Code plugins** — [code.claude.com/docs/en/plugins-reference](https://code.claude.com/docs/en/plugins-reference)
- **Changelog** — [CHANGELOG.md](CHANGELOG.md)

## License

MIT — see [LICENSE](LICENSE).
