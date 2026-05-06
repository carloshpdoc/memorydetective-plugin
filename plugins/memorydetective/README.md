# memorydetective (Claude Code plugin)

> One-command install of the [memorydetective](https://github.com/carloshpdoc/memorydetective) MCP server, plus a `/perf-investigate` skill for disciplined iOS leak + performance investigation.

## What this plugin gives you

When installed, this plugin auto-registers the `memorydetective` MCP server and adds a top-level skill:

- **31 MCP tools**: `analyzeMemgraph`, `classifyCycle`, `findRetainers`, `verifyFix`, `compareTracesByPattern`, `analyzeHangs`, `analyzeAnimationHitches`, `analyzeAllocations`, `analyzeAppLaunch`, `analyzeTimeProfile`, `recordTimeProfile`, `captureMemgraph`, `renderCycleGraph`, `detectLeaksInXCUITest`, the v1.8 verify-fix trio (`bootAndLaunchForLeakInvestigation`, `replayScenario`, `captureScenarioState`) for the macOS 26.x `leaks --outputGraph` regression plus deterministic before/after snapshots, plus 5 SourceKit-LSP-backed Swift source-bridging tools, plus `logShow` / `logStream` for macOS unified logging
- **34 cycle patterns** in the classifier: SwiftUI (incl. Swift 6 / `@Observable` / SwiftData / NavigationStack), Combine, Swift Concurrency (incl. AsyncSequence-on-self), UIKit, Core Animation, Core Data, SwiftData (`ModelContext` + Actor), Coordinator pattern, RxSwift, Realm
- **Per-classification triple**: every cycle now ships `fixHint` (textual direction) + `staticAnalysisHint` (which SwiftLint rule complements this, or explicit gap notice) + `fixTemplate` (Swift before/after code snippet, new in v1.7)
- **5 MCP prompts** as slash commands: `/investigate-leak`, `/investigate-hangs`, `/investigate-jank`, `/investigate-launch`, `/verify-cycle-fix`
- **34 catalog resources** browsable at `memorydetective://patterns/{patternId}`, so an agent can read the catalog without burning a tool call
- **`/perf-investigate` skill**: disciplined investigation playbook routing to the right tools based on symptoms

## Install

```
/plugin marketplace add carloshpdoc/memorydetective-plugin
/plugin install memorydetective@memorydetective-plugin
```

That's it. The MCP server is pulled from npm (`memorydetective@^1.8`) on first use.

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

### Common gotchas

- **Physical iOS device + memgraph:** `leaks(1)` only attaches to local Mac processes (which includes the iOS Simulator). For physical devices, export from Xcode (`Debug > View Memory Graph > File > Export Memory Graph`) and pass the path to `analyzeMemgraph`.
- **`xctrace` SIGSEGV on heavy traces:** `analyzeTimeProfile` returns a structured workaroundNotice. Open the trace once in Instruments to symbolicate, close it, then retry.
- **First MCP call feels slow:** `npx -y memorydetective@^1.8` resolves the package on first invocation. Subsequent calls reuse the cached binary.

## Requirements

- macOS with Xcode Command Line Tools (`xcode-select --install`)
- Node.js ≥ 20
- For Swift source-bridging tools: full Xcode (not just CLT), so `xcrun sourcekit-lsp` is available
- Memory Graph capture works on Mac apps and iOS Simulator. Physical iOS devices need to export from Xcode's Memory Graph Debugger button (`File → Export Memory Graph`).

## Versioning

This plugin tracks the MCP server's minor version. The `.mcp.json` in this plugin pulls `memorydetective@^1.8` via `npx -y`, so any 1.8.x patch/minor release is auto-resolved on first run.

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
