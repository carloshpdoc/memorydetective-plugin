# Changelog

All notable changes to the `memorydetective` Claude Code plugin are recorded here.

The plugin tracks the MCP server's minor version. See the [memorydetective CHANGELOG](https://github.com/carloshpdoc/memorydetective/blob/main/CHANGELOG.md) for changes to the underlying server.

## [1.12.0] - 2026-05-14

Tracks `memorydetective@1.12.0` server release. The plugin's `.mcp.json` constraint bumps from `^1.11` to `^1.12`. Four-phase server patch: countAlive / findRetainers / verifyFix gain reference-tree integration (so the abandoned-memory agent chain works end-to-end on leakCount=0 memgraphs), and analyzeHangs gains auto cross-schema correlation via `includeStackClassification: true` (auto-populates mainThreadViolations[] without the caller building topFramesByHangStartNs manually).

### Changed

- `.mcp.json` MCP server constraint: `^1.11` -> `^1.12`. The `npx -y` resolver auto-pulls `memorydetective@1.12.x`.
- `.claude-plugin/plugin.json`: version `1.11.0 -> 1.12.0`, description refreshed for v1.12.
- `.claude-plugin/marketplace.json`: plugin description refreshed for v1.12.
- `plugins/memorydetective/README.md`: pin `^1.11 -> ^1.12`.
- Marketplace-level `README.md` (repo root): pin + manifest reference updated.

### Notes

- No SKILL.md changes in this release. The v1.10 playbook still applies; v1.12's new flags (`includeReferenceTree` on countAlive/findRetainers, `includeStackClassification` on analyzeHangs) are accessed via the same tool calls with new optional input parameters.
- No breaking changes for users. Re-running `/plugin install memorydetective@memorydetective-plugin` (or auto-update on session start) picks up the new constraint.

## [1.11.0] - 2026-05-14

Tracks `memorydetective@1.11.0` server release. The plugin's `.mcp.json` constraint bumps from `^1.10` to `^1.11`. Validation-driven patch: re-running v1.10 against the global npm binary surfaced 3 gaps (CLI human-output ignored abandoned-memory, diffMemgraphs had the same reference-tree blind spot v1.10 fixed only in analyzeMemgraph + analyzeAbandonedMemory, and there was no orientation tool for .trace bundles).

### Changed

- `.mcp.json` MCP server constraint: `^1.10` -> `^1.11`. The `npx -y` resolver auto-pulls `memorydetective@1.11.x`.
- `.claude-plugin/plugin.json`: version `1.10.0 -> 1.11.0`, description refreshed for v1.11 (inspectTrace, diffMemgraphs ref-tree, CLI surface).
- `.claude-plugin/marketplace.json`: plugin description refreshed for v1.11 capabilities.
- `plugins/memorydetective/README.md`: pin `^1.10 -> ^1.11`.
- Marketplace-level `README.md` (repo root): pin + manifest reference + tool count 34 -> 35.

### Notes

- No SKILL.md changes in this release. The v1.10 playbook continues to apply; v1.11's new inspectTrace is a discovery tool that complements the existing analyzer playbook (use it as the FIRST call on a .trace), and the diffMemgraphs reference-tree integration is accessed via the same tool call with new optional response fields.
- No breaking changes for users. Re-running `/plugin install memorydetective@memorydetective-plugin` (or auto-update on session start) picks up the new constraint.

## [1.10.0] - 2026-05-14

Tracks `memorydetective@1.10.0` server release. The plugin's `.mcp.json` constraint bumps from `^1.9` to `^1.10`. Notelet retro feedback loop: the v1.9 abandoned-memory pipeline left AVPlayerItem at raw rank ~82 in the heap (invisible at default `referenceTreeTopN: 20`) and the over-broad KVO co-occurrence classifier produced 25 false positives. v1.10 closes both gaps.

### Changed

- `.mcp.json` MCP server constraint: `^1.9` -> `^1.10`. The `npx -y` resolver auto-pulls `memorydetective@1.10.x`.
- `.claude-plugin/plugin.json`: version `1.9.0 -> 1.10.0`, description refreshed for v1.10 (actionable views + verify-fix-table).
- `.claude-plugin/marketplace.json`: plugin description refreshed for v1.10 capabilities.

### Notes

- No SKILL.md changes in this release. The v1.9 SKILL playbook continues to apply; the new actionable views and verify-fix-table output are accessed via the existing tool calls with refined response shapes + the new `outputFormat: "verify-fix-table"` value.
- No breaking changes for users. Re-running `/plugin install memorydetective@memorydetective-plugin` (or auto-update on session start) picks up the new constraint.

## [1.9.0] - 2026-05-14

Tracks `memorydetective@1.9.0` server release. The plugin's `.mcp.json` constraint bumps from `^1.8` to `^1.9`. SKILL.md gains the abandoned-memory playbook branch, the `mainThreadViolations[]` classification step in the perf-hangs playbook, and the env-flags reference.

### Changed

- `.mcp.json` MCP server constraint: `^1.8` -> `^1.9`. The `npx -y` resolver auto-pulls `memorydetective@1.9.x`.
- `skills/perf-investigate/SKILL.md`:
  - Tooling summary: Read & analyze `13 -> 14` (adds `analyzeAbandonedMemory`), new "Ops (1, v1.9)" section listing `cleanupTraces`, "CI / test integration" `2 -> 3` (adds `detectLeaksInXCTest`).
  - Memgraph-leak playbook (A) gains a new step `3b`: when `analyzeMemgraph` reports `leakCount: 0` and `classifyCycle` returns `primaryMatch: null`, route to `analyzeAbandonedMemory(before, after)` for the KVO-observer / NotificationCenter / cache / singleton-retained shapes that `leaks(1)` cannot see.
  - Perf-hangs playbook (B) gains a step for the new `mainThreadViolations[]` enrichment: chain a `topFramesByHangStartNs` map (built from `analyzeTimeProfile`) back into `analyzeHangs` to classify each top hang as `sync-io`, `db-lock`, `network`, or `lock-contention` with a canonical fix chain.
  - Catalog reference updated `34 -> 36` patterns, with `uikit.viewcontroller-retained-after-pop` and `swiftui.observable-write-on-every-render` added to the UIKit and SwiftUI category lists respectively.
  - New "Environment flags worth knowing (v1.9)" section: `MEMORYDETECTIVE_REDACTION`, `ALLOW_LAUNCH`, `MAX_RECORDING_SECONDS`, `TRACE_ROOT`, `ALLOW_EXTERNAL_CLEANUP`, `SUPPRESS_PLATFORM_ADVISORY`.
- `plugins/memorydetective/README.md`: tool count `31 -> 34`, pattern count `34 -> 36`, npm pin `^1.8 -> ^1.9`. New workflows F (abandoned-memory shape) and G (CI per-test leak detection). Existing workflows E (hangs / animation jank) expanded with the v1.9 `mainThreadViolations` enrichment.
- `.claude-plugin/plugin.json`: version `1.8.1 -> 1.9.0`, description refreshed for v1.9 capabilities.
- `.claude-plugin/marketplace.json`: plugin description refreshed for v1.9 capabilities.
- Marketplace-level `README.md` (repo root): tool count `31 -> 34`, pattern count `34 -> 36`, manifest reference updated to v1.9.0, npm pin `^1.8 -> ^1.9`, new bullet listing v1.9 highlights.

### Notes

- No breaking changes for users. Re-running `/plugin install memorydetective@memorydetective-plugin` (or auto-update on session start) picks up the new constraint and SKILL content.
- The `^1.9` constraint will continue to auto-resolve 1.9.x patch/minor releases without a plugin bump.

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

- No breaking changes for users. The plugin install command is unchanged. Re-running `/plugin install memorydetective@memorydetective-plugin` (or the auto-update on session start) picks up the new constraint and SKILL content.

## [1.6.0] - 2026-05-03

Initial public release of the Claude Code plugin wrapper for `memorydetective`.

### Added

- Plugin manifest declaring the `memorydetective` plugin, pulling the MCP server from npm via `npx -y memorydetective@^1.6`.
- `/perf-investigate` skill: disciplined iOS performance + memory-leak investigation playbook routing to the right MCP tools based on symptom (memgraph-leak, perf-hangs, ui-jank, app-launch-slow, verify-fix).
- Marketplace catalog (`.claude-plugin/marketplace.json`) for one-command install via `/plugin marketplace add`.

### Notes

- Plugin version is pinned to `1.6.0` because the npm `memorydetective` server is at `1.6.0` at release time. Patch + minor server releases are picked up automatically via the `^1.6` constraint in `.mcp.json`, with no plugin bump required.
- Major server releases (e.g. `2.0.0`) will require a plugin manifest bump.
