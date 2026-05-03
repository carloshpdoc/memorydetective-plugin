# Changelog

All notable changes to the `memorydetective` Claude Code plugin are recorded here.

The plugin tracks the MCP server's minor version. See the [memorydetective CHANGELOG](https://github.com/carloshpdoc/memorydetective/blob/main/CHANGELOG.md) for changes to the underlying server.

## [1.6.0] — 2026-05-03

Initial public release of the Claude Code plugin wrapper for `memorydetective`.

### Added

- Plugin manifest declaring the `memorydetective` plugin, pulling the MCP server from npm via `npx -y memorydetective@^1.6`.
- `/perf-investigate` skill — disciplined iOS performance + memory-leak investigation playbook routing to the right MCP tools based on symptom (memgraph-leak, perf-hangs, ui-jank, app-launch-slow, verify-fix).
- Marketplace catalog (`.claude-plugin/marketplace.json`) for one-command install via `/plugin marketplace add`.

### Notes

- Plugin version is pinned to `1.6.0` because the npm `memorydetective` server is at `1.6.0` at release time. Patch + minor server releases are picked up automatically via the `^1.6` constraint in `.mcp.json` — no plugin bump required.
- Major server releases (e.g. `2.0.0`) will require a plugin manifest bump.
