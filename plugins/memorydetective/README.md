# memorydetective (Claude Code plugin)

> One-command install of the [memorydetective](https://github.com/carloshpdoc/memorydetective) MCP server, plus a `/perf-investigate` skill for disciplined iOS leak + performance investigation.

## What this plugin gives you

When installed, this plugin auto-registers the `memorydetective` MCP server and adds a top-level skill:

- **42 MCP tools** (memgraph + trace + MetricKit + verify-fix + Swift source bridging + ops). The full list lives in the [memorydetective README](https://github.com/carloshpdoc/memorydetective#api). v1.18 added `analyzeMetricKitPayload` (post-mortem production diagnostics from `.mxdiagnostic` payloads). v1.16 added `recordViaInstrumentsApp` (macOS 26.x escape hatch). v1.15 added three trace tools (`analyzeMemoryFootprint` / `analyzeEnergyImpact` / `analyzeLeakTimeline`). v1.14 added `analyzeNetworkActivity`. v1.13 added `summarizeTrace`. v1.11 added `inspectTrace`. v1.17 was reliability-only (no surface changes, 24 new tests across the audit punch list).
- **36 cycle patterns** in the classifier: SwiftUI (incl. Swift 6 / `@Observable` / SwiftData / NavigationStack / the v1.9 `observable-write-on-every-render` shape), Combine, Swift Concurrency (incl. AsyncSequence-on-self), UIKit (incl. the v1.9 `viewcontroller-retained-after-pop` shape), Core Animation, Core Data, SwiftData (`ModelContext` + Actor), Coordinator pattern, RxSwift, Realm
- **Per-classification triple**: every cycle ships `fixHint` (textual direction) + `staticAnalysisHint` (which SwiftLint rule complements this, or explicit gap notice) + `fixTemplate` (Swift before/after code snippet, v1.7+)
- **7 MCP prompts** as slash commands: `/investigate-leak`, `/investigate-hangs`, `/investigate-jank`, `/investigate-launch`, `/verify-cycle-fix`, `/summarize-trace`, `/investigate-metrickit` (v1.18+, post-mortem from `.mxdiagnostic` payloads)
- **34 catalog resources** browsable at `memorydetective://patterns/{patternId}`, so an agent can read the catalog without burning a tool call
- **`/perf-investigate` skill**: disciplined investigation playbook routing to the right tools based on symptoms

## Install

```
/plugin marketplace add carloshpdoc/memorydetective-plugin
/plugin install memorydetective@memorydetective-plugin
```

That's it. The MCP server is pulled from npm (`memorydetective@^1.18`) on first use.

## Use

### Three ways to invoke

**1. The disciplined skill.** Walks you through symptom triage and routes to the right playbook:

```
/perf-investigate
```

**2. A specific slash command** when you already know the symptom class:

```
/investigate-leak       # memory leak (provide memgraphPath)
/investigate-hangs      # main-thread hangs (provide tracePath)
/investigate-jank       # animation hitches (provide tracePath)
/investigate-launch     # slow app launch (provide tracePath)
/verify-cycle-fix       # confirm a fix actually closed the cycle (before + after)
```

**3. Plain English.** Best for ad-hoc work; the agent picks the tool chain:

> "I have a memgraph at `~/Desktop/myapp.memgraph`. What's leaking?"

### Optional: install `axe` for UI driving

Only required if you want to use `replayScenario` or the UI-tree sub-capture of `captureScenarioState`. Everything else works without it.

```bash
brew install cameroncooke/axe/axe
```

When `axe` is missing, those two tools return a structured `workaroundNotice` with this exact install hint instead of throwing.

### Common workflows

#### A. Classify a memgraph you already have

You exported from Xcode (`Debug > View Memory Graph > File > Export Memory Graph`). Drop the path on the agent:

> "I have a leak at `~/Desktop/leak.memgraph`. Classify it and tell me where to fix."

What runs: `analyzeMemgraph` → `classifyCycle` → `swiftSearchPattern` / `swiftGetSymbolDefinition` → fix suggestion adapted to your codebase via the per-classification `fixTemplate` snippet.

#### B. Capture a fresh memgraph from a running simulator app

> "Capture a memgraph from `DemoApp` running on the simulator and classify it."

What runs: `captureMemgraph({ appName: "DemoApp", output: ... })` → same chain as workflow A.

#### C. macOS 26.x: `leaks --outputGraph` is broken

If your app was launched without `MallocStackLogging=1`, `leaks` aborts with `Failed to get DYLD info for task`. v1.8 detects this and self-heals. From your side you do nothing different:

> "Capture a memgraph from `DemoApp`."

What runs:
1. `captureMemgraph` returns `workaroundNotice: { issue: "minimal-corpse" }` with `suggestedNextCalls` pointing at the relaunch path
2. Agent calls `bootAndLaunchForLeakInvestigation({ workspace: "MyApp.xcworkspace", scheme: "DemoApp", simulator: { name: "iPhone 15" } })` which builds, boots, installs, and launches with `MallocStackLogging=1` propagated via `SIMCTL_CHILD_*`
3. Agent re-tries `captureMemgraph` with the new host PID
4. Capture succeeds, classification proceeds normally

#### D. Verify-fix loop (deterministic before/after snapshots)

> "Build my app, repeat the carousel flow 5 times to amplify the leak, capture `before`. I'll ship the fix; then capture `after` and compare."

What runs:
1. `bootAndLaunchForLeakInvestigation({ workspace, scheme, simulator })` → app up with the right env
2. `replayScenario({ simulatorUDID, actions: [...], repeat: 5 })` → drives the UI through the leaking flow
3. `captureScenarioState({ simulatorUDID, pid, outputDir, label: "before" })` → writes `before.memgraph` + `before.png` + `before.ui.json`

You ship the fix and rebuild. Steps 1-3 again with `label: "after"`, then:

4. `diffMemgraphs(before.memgraph, after.memgraph)` → instances + bytes deltas
5. `verifyFix(before, after, expectedPatternId: "swiftui.tag-index-projection")` → PASS / PARTIAL / FAIL

#### E. Hangs or animation jank, not leaks

> "Profile `DemoApp` for 60 s on my iPhone 17 Pro Max and tell me where the hangs are."

What runs: `listTraceDevices` → `recordTimeProfile({ template: "Time Profiler", deviceId, attachAppName, durationSec: 60, output })` → `analyzeHangs`. Reports user-visible hangs with the longest at the top.

For animation hitches, swap to `template: "Animation Hitches"` and `analyzeAnimationHitches`.

For v1.9, you can also re-call `analyzeHangs` with a `topFramesByHangStartNs` map (built from a chained `analyzeTimeProfile`) to enrich each top hang with a `mainThreadViolations[]` classification: `sync-io` (read/write/fsync, NSData blocking inits), `db-lock` (SQLite mutex / NSManagedObjectContext save), `network` (NSURLConnection sendSynchronousRequest, blocking nw_connection), or `lock-contention` (pthread / os_unfair_lock / dispatch_semaphore_wait / dispatch_sync).

#### F. Abandoned-memory shape (leakCount: 0 but heap grows)

The leaks you can't see with `leaks(1)` alone. Orphaned KVO observers, NotificationCenter blocks that were never removed, caches that never evict, singleton-retained payloads. The objects are reachable from KVO's global registry (or your singleton), so they don't count as "leaked" in the strict sense.

> "I have two memgraphs at `~/Desktop/before.memgraph` and `~/Desktop/after.memgraph`. Both report leakCount: 0 but the heap clearly grew. What's accumulating?"

What runs: `analyzeAbandonedMemory(before, after)` → returns `growthByClass[]` ranked by delta, each entry classified as `kvo-observer-orphaned`, `notificationcenter-observer-leaked`, `cache-too-aggressive`, `singleton-retains-payload`, or `unknown-growth`. The classifier escalates when `NSKeyValueObservance` co-occurs with another large-delta class (the KVO-observer-orphan signal).

If you only have ONE memgraph and want to know what is accumulating at the top: pass `referenceTreeTopN: 20` to `analyzeMemgraph` to surface the top classes by live instance count from a single snapshot.

#### G. CI: gate a PR on per-test leak detection

> "Wire this unit-test scheme into GitHub Actions so the build fails when a new ROOT CYCLE appears."

What runs: see the [5-minute CI recipe](https://github.com/carloshpdoc/memorydetective#add-memorydetective-to-your-ci-in-5-minutes) in the upstream README. `detectLeaksInXCTest --outputHtmlPath` writes a self-contained HTML report you can upload as a workflow artifact; the exit code is non-zero when new cycles outside the allowlist appear.

### Common gotchas

- **Physical iOS device + memgraph:** `leaks(1)` only attaches to local Mac processes (which includes the iOS Simulator). For physical devices, export from Xcode (`Debug > View Memory Graph > File > Export Memory Graph`) and pass the path to `analyzeMemgraph`.
- **`xctrace` SIGSEGV on heavy traces:** `analyzeTimeProfile` returns a structured workaroundNotice. Open the trace once in Instruments to symbolicate, close it, then retry.
- **First MCP call feels slow:** `npx -y memorydetective@^1.18` resolves the package on first invocation. Subsequent calls reuse the cached binary.

## Requirements

- macOS with Xcode Command Line Tools (`xcode-select --install`)
- Node.js ≥ 20
- For Swift source-bridging tools: full Xcode (not just CLT), so `xcrun sourcekit-lsp` is available
- Memory Graph capture works on Mac apps and iOS Simulator. Physical iOS devices need to export from Xcode's Memory Graph Debugger button (`File → Export Memory Graph`).

## Versioning

This plugin tracks the MCP server's minor version. The `.mcp.json` in this plugin pulls `memorydetective@^1.18` via `npx -y`, so any 1.18.x patch release is auto-resolved on first run.

When the MCP server bumps to a new major (e.g. `2.0.0`), this plugin will publish a matching plugin version with the updated constraint.

See the upstream [CHANGELOG](https://github.com/carloshpdoc/memorydetective/blob/main/CHANGELOG.md) for what's in each release.

## Both surfaces (npm and plugin) are first-class

This plugin wraps the MCP server; it doesn't replace it. If you don't use Claude Code, install via npm directly:

```bash
npm install -g memorydetective
```

…then add to your MCP client config (Cursor, Cline, Claude Desktop, Continue, etc.):

```jsonc
{
  "mcpServers": {
    "memorydetective": { "command": "memorydetective" }
  }
}
```

The plugin route is for Claude Code users who want the slash command + zero-friction install. The npm route is universal.

## License

Apache-2.0. See [LICENSE](./LICENSE).

## Source

Plugin: https://github.com/carloshpdoc/memorydetective-plugin
MCP server: https://github.com/carloshpdoc/memorydetective
