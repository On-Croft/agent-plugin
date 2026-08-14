# Changelog

All notable changes to the Croft Agent Plugin are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/) and the plugin uses
[Semantic Versioning](https://semver.org/).

## [1.0.0] - 2026-08-15

Initial release. Agent Plugin 1.0.0 compatible.

### Added
- `plugin.json` manifest (Agent Plugins 1.0.0).
- `mcp.json` declaring the Croft remote MCP server (`https://mcp.oncroft.net`,
  streamable-http, per-user OAuth).
- Skills:
  - `building-croft-apps` — the create → write_files → deploy → poll-status golden path and
    Croft's app conventions.
  - `croft-connectors` — calling external APIs through connectors (custom headers + raw
    bodies), never via a vendor SDK or a stored key.
  - `croft-shared-data` — sharing SQLite data between apps in a workspace.
