# Croft Agent Plugin

A portable [Agent Plugin 1.0.0](https://agent-plugins.org/specification) for
**[Croft](https://oncroft.net)** — a passwordless PaaS where you build, deploy, and
maintain small web apps by talking to an AI assistant.

The plugin gives any compatible agent two things:

1. **The Croft MCP server** (`mcp.json`) — the tools to create apps, write files, deploy,
   manage connectors and shared data, and more, on the user's Croft workspace.
2. **Skills** (`skills/`) — portable instructions that teach the agent Croft's golden path
   so it builds apps the right way from the first turn, even before it reads the live
   conventions.

## Layout

```
croft-agent-plugin/
├── plugin.json                     # Agent Plugin 1.0.0 manifest
├── mcp.json                        # Croft MCP server (remote, streamable-http)
└── skills/
    ├── building-croft-apps/        # the create → write → deploy → verify golden path
    ├── croft-connectors/           # calling external APIs via connectors (no SDKs, no keys)
    └── croft-shared-data/          # sharing SQLite data between apps
```

## The MCP server

`mcp.json` declares Croft's remote MCP endpoint:

```json
{
  "mcpServers": {
    "croft": { "type": "streamable-http", "url": "https://mcp.oncroft.net" }
  }
}
```

**Authentication is per-user OAuth**, not a static key — which is why no credentials appear
in `mcp.json` (the spec forbids credentials in package data anyway). On first connect the
agent's client performs OAuth 2.1 (PKCE + dynamic client registration) against
`https://mcp.oncroft.net`; the user signs in and authorises their workspace, and every
request is then scoped to that workspace with the user's role. Only workspace builders
(owner/admin/creator) can use the build tools.

> In agents that authenticate the MCP server per end-user (e.g. Microsoft Copilot's **User**
> auth mode), each user connects with their own Croft identity and role — which is the
> intended setup. Avoid shared/maker-credential modes.

## The skills

| Skill | Use it when |
|-------|-------------|
| `building-croft-apps` | Creating, changing, deploying, or fixing any Croft app — the required `create_app → write_files → deploy → poll status` workflow and Croft's app conventions. |
| `croft-connectors` | The app must call an external API (Stripe, Azure DevOps, a CRM, email…) — always through a connector, never a vendor SDK or a stored key. |
| `croft-shared-data` | Two apps in the workspace should reuse the same data (e.g. a shared contacts list), or one app needs to read another's data. |

The skills summarise Croft's `get_platform_conventions`, which the agent should still call at
runtime — it returns the authoritative, live golden path plus the workspace's actual
connectors and shared apps.

## Installing

Drop this directory into your agent client's plugin location, or point the client at this
repository. Any Agent Plugin 1.0.0-compatible client will pick up the Croft MCP server and
the skills automatically.

## License

MIT — see [LICENSE](LICENSE).
