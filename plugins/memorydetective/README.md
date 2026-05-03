# memorydetective (Claude Code plugin)

> One-command install of the [memorydetective](https://github.com/carloshpdoc/memorydetective) MCP server, plus a `/perf-investigate` skill for disciplined iOS leak + performance investigation.

## What this plugin gives you

When installed, this plugin auto-registers the `memorydetective` MCP server and adds a top-level skill:

- **28 MCP tools**: `analyzeMemgraph`, `classifyCycle`, `findRetainers`, `verifyFix`, `compareTracesByPattern` (new in v1.7), `analyzeHangs`, `analyzeAnimationHitches`, `analyzeAllocations`, `analyzeAppLaunch`, `analyzeTimeProfile`, `recordTimeProfile`, `captureMemgraph`, `renderCycleGraph`, `detectLeaksInXCUITest`, plus 5 SourceKit-LSP-backed Swift source-bridging tools, plus `logShow` / `logStream` for macOS unified logging
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

That's it. The MCP server is pulled from npm (`memorydetective@^1.6`) on first use.

## Use

Either invoke the skill directly:

```
/perf-investigate
```

…and the agent will route to the right playbook based on what you describe.

Or invoke a specific MCP prompt:

```
/investigate-leak                    # for memory leaks (provide memgraphPath)
/investigate-hangs                   # for main-thread hangs (provide tracePath)
/investigate-jank                    # for animation hitches (provide tracePath)
/investigate-launch                  # for slow app launch (provide tracePath)
/verify-cycle-fix                    # to verify a fix landed (provide before, after)
```

Or just talk to the agent in plain English:

> "I have a memgraph at `~/Desktop/myapp.memgraph`. What's leaking?"

The agent will pick the right tool chain.

## Requirements

- macOS with Xcode Command Line Tools (`xcode-select --install`)
- Node.js ≥ 20
- For Swift source-bridging tools: full Xcode (not just CLT), so `xcrun sourcekit-lsp` is available
- Memory Graph capture works on Mac apps and iOS Simulator. Physical iOS devices need to export from Xcode's Memory Graph Debugger button (`File → Export Memory Graph`).

## Versioning

This plugin tracks the MCP server's minor version. The `.mcp.json` in this plugin pulls `memorydetective@^1.7` via `npx -y`, so any 1.7.x patch/minor release is auto-resolved on first run.

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
