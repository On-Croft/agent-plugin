# Changelog

All notable changes to the Croft Agent Plugin are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/) and the plugin uses
[Semantic Versioning](https://semver.org/).

## [1.1.0] - 2026-08-19

### Added
- Cursor support: `.cursor-plugin/marketplace.json`, `.cursor-plugin/plugin.json`, and
  `.cursor-plugin/mcp.json` (the same Croft endpoint under Cursor's `type: http` spelling).
  Cursor requires a marketplace manifest before it will add a folder or repo as a plugin.
- Claude Code support: `.claude-plugin/marketplace.json` and `.claude-plugin/plugin.json`
  (reusing the root `mcp.json`).
- `assets/logo.svg` and `assets/logo.png` — the Croft brand icon, referenced by `logo` in
  the Cursor plugin and marketplace manifests. Cursor is the only host with a logo field.
- README: a one-click Cursor MCP install deeplink
  (`cursor://anysphere.cursor-deeplink/mcp/install`), flagged as MCP-server-only — Cursor
  has no plugin-install deeplink, so that route does not deliver `skills/`.

### Changed
- Version bumped to 1.1.0 across all manifests. The root `plugin.json`, `mcp.json`, and
  `skills/` are unchanged in content — every client shares the same skills and MCP server.

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
