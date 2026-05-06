# Changelog

All notable changes to the `memorydetective` Claude Code plugin are recorded here.

The plugin tracks the MCP server's minor version. See the [memorydetective CHANGELOG](https://github.com/carloshpdoc/memorydetective/blob/main/CHANGELOG.md) for changes to the underlying server.

## [1.8.1] - 2026-05-06

Documentation-only patch. No server bump, no MCP server constraint change (`.mcp.json` stays at `memorydetective@^1.8`).

### Changed

- `plugins/memorydetective/README.md` "Use" section expanded from a 4-line invocation note to a structured walkthrough with: 3 invocation modes (skill / slash command / plain English), an explicit "Optional: install axe" subsection clarifying the soft-dependency boundary, 5 concrete copy-pasteable workflows (classify existing memgraph, capture fresh, macOS 26.x auto-recovery, verify-fix loop, hangs/jank), and a "Common gotchas" subsection for the physical-device limitation, xctrace SIGSEGV workaround, and first-call latency.
- Marketplace-level `README.md` (repo root) stale 1.7 references fixed (28 -> 31 tools, `^1.7` -> `^1.8`, plugin manifest reference). Caught during post-release double-check; the v1.8.0 sync touched the plugin sub-README but missed the parent.
- Sub-plugin `README.md` Versioning section pin reference fixed (`^1.7` -> `^1.8`). Caught during the same double-check.

### Notes

- No server-side changes. Existing v1.8.0 plugin installs continue to work unchanged. Re-running `/plugin install memorydetective@memorydetective-plugin` (or auto-update on session start) picks up the updated README content.
- No MCP tool changes, no SKILL.md changes, no constraint changes.

## [1.8.0] - 2026-05-06

Tracks `memorydetective@1.8.0` server release. The plugin's `.mcp.json` constraint bumps from `^1.7` to `^1.8`. SKILL.md gains the macOS 26.x troubleshooting workflow plus the new verify-fix orchestration playbook (bootAndLaunchForLeakInvestigation -> replayScenario -> captureScenarioState -> diffMemgraphs).

### Changed

- `.mcp.json` MCP server constraint: `^1.7` -> `^1.8`. The `npx -y` resolver auto-pulls `memorydetective@1.8.x`.
- `skills/perf-investigate/SKILL.md`:
  - Tooling summary updated: 28 -> 31 tools. New "Verify-fix orchestration (3)" section listing `bootAndLaunchForLeakInvestigation`, `replayScenario`, `captureScenarioState`. CI / test integration count fixed `1 -> 2` (compareTracesByPattern was stale at v1.7).
  - Memgraph-leak playbook gains a "When capture fails on macOS 26.x" branch: detect `workaroundNotice.issue === "minimal-corpse"`, then call `bootAndLaunchForLeakInvestigation` with `MallocStackLogging=1` to relaunch and retry, fall back to Xcode manual export if the relaunch path is unavailable.
  - Verify-fix playbook gains the canonical loop using the new tools: `bootAndLaunchForLeakInvestigation` -> `replayScenario(repeat: 5)` -> `captureScenarioState(label: "before")` -> ship fix -> repeat -> `diffMemgraphs`.
  - Catalog resources count updated 33 -> 34 (was stale).
- `README.md`: tool count `28 -> 31`, mention of the new macOS 26.x fix and verify-fix loop.
- `.claude-plugin/plugin.json` description: refreshed for v1.8.0 capabilities.
- `.claude-plugin/marketplace.json` plugin description: refreshed for v1.8.0 capabilities.

### Notes

- No breaking changes for users. Re-running `/plugin install memorydetective@memorydetective-plugin` (or auto-update on session start) picks up the new constraint and SKILL content.

## [1.7.0] - 2026-05-03

Tracks `memorydetective@1.7.0` server release. The plugin's `.mcp.json` constraint bumps from `^1.6` to `^1.7`. SKILL.md updated to reference the new fields and tool the upstream server ships.

### Changed

- `.mcp.json` MCP server constraint: `^1.6` → `^1.7`. The `npx -y` resolver auto-pulls `memorydetective@1.7.x`.
- `skills/perf-investigate/SKILL.md`:
  - Catalog reference updated `33 → 34` patterns. Adds the `swiftdata.modelcontext-actor-cycle` row in the catalog summary.
  - `classifyCycle` step in the memgraph-leak playbook now mentions the new `fixTemplate` field and instructs the agent to lift the Swift snippet as a starting point (adapt, don't paste).
  - `verify-fix` playbook gains a trace-side path using `compareTracesByPattern` for hangs / animation-hitches / app-launch regressions.
- `README.md`: tool count `27 → 28`, pattern count `33 → 34`, mention of the per-classification triple (`fixHint` + `staticAnalysisHint` + `fixTemplate`).

### Notes

- No breaking changes for users — the plugin install command is unchanged. Re-running `/plugin install memorydetective@memorydetective-plugin` (or the auto-update on session start) picks up the new constraint and SKILL content.

## [1.6.0] — 2026-05-03

Initial public release of the Claude Code plugin wrapper for `memorydetective`.

### Added

- Plugin manifest declaring the `memorydetective` plugin, pulling the MCP server from npm via `npx -y memorydetective@^1.6`.
- `/perf-investigate` skill — disciplined iOS performance + memory-leak investigation playbook routing to the right MCP tools based on symptom (memgraph-leak, perf-hangs, ui-jank, app-launch-slow, verify-fix).
- Marketplace catalog (`.claude-plugin/marketplace.json`) for one-command install via `/plugin marketplace add`.

### Notes

- Plugin version is pinned to `1.6.0` because the npm `memorydetective` server is at `1.6.0` at release time. Patch + minor server releases are picked up automatically via the `^1.6` constraint in `.mcp.json` — no plugin bump required.
- Major server releases (e.g. `2.0.0`) will require a plugin manifest bump.
