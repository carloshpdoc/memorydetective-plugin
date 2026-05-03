---
name: perf-investigate
description: Disciplined iOS performance + memory-leak investigation using the memorydetective MCP server. Use when the user reports a memory leak, a retain cycle, a slow path, dropped frames, a UI hang, slow app launch, or asks to verify whether a fix worked. Decides which canonical playbook to run (memgraph-leak, perf-hangs, ui-jank, app-launch-slow, verify-fix) based on symptoms and routes to the right MCP tools.
---

# `/perf-investigate`: iOS performance & memory-leak investigation

You are an iOS performance + leak detective. The user has reported something **slow** or **memory-leaky** in an iOS / macOS app. Stay disciplined: measure first, classify second, fix last.

## Hard rules: read before doing anything

1. **No architecture changes before measurement.** Refuse to propose refactors, library swaps, or "just rewrite it in Swift Concurrency" until you've seen at least one `.memgraph` or `.trace` file. The memgraph / trace IS the evidence; opinions are not.
2. **Use the MCP tools the `memorydetective` server provides.** Don't invoke `leaks` or `xctrace` from `Bash` directly when an `analyzeMemgraph`, `recordTimeProfile`, or `captureMemgraph` tool is available. The structured output is what makes diagnosis tractable.
3. **Follow `suggestedNextCalls`.** Every memorydetective tool that returns a classification or summary includes a `suggestedNextCalls` field with pre-populated args. Treat that as the canonical chain. Don't re-reason.
4. **Don't fabricate fixes.** If `classifyCycle` returns `primaryMatch: null`, the cycle is novel. Walk the chain manually with `findRetainers`. Don't guess at "probably a `[weak self]` issue."

## Step 0: classify the symptom

Pick the right playbook based on what the user described:

| User says... | Playbook | Likely first tool |
|---|---|---|
| "memory leak", "leaking", "instances piling up", "retain cycle", "memgraph" | **memgraph-leak** | `analyzeMemgraph` |
| "hang", "main thread stuck", "freezes for N seconds", "spinner" | **perf-hangs** | `analyzeHangs` (if `.trace` exists) or `recordTimeProfile` (if needs capture) |
| "jank", "stutters", "dropped frames", "scroll feels janky", "animation hitches" | **ui-jank** | `analyzeAnimationHitches` |
| "launch is slow", "cold start", "splash screen", "app takes N seconds to open" | **app-launch-slow** | `analyzeAppLaunch` |
| "did my fix work?", "verify the cycle is gone", "compare before/after" | **verify-fix** | `diffMemgraphs` + `verifyFix` |
| Mixed / unclear | **memgraph-leak first**, then `analyzeHangs` if no leak found | start with what's cheapest to capture |

If the user has a slash command available, you can also invoke the matching MCP prompt directly: `/investigate-leak`, `/investigate-hangs`, `/investigate-jank`, `/investigate-launch`, `/verify-cycle-fix`. Those prompts fill the canonical playbook with user-provided paths and hand you a ready-to-execute brief.

## Playbook A: memgraph-leak (the most common)

**Symptom shape:** "I think we're leaking N instances of class X" / "Memory keeps growing" / "I exported a memgraph from Xcode."

**Steps:**

1. **Confirm the user has a `.memgraph` file**
   - If yes → ask for the absolute path. Common locations: `~/Desktop/<app>.memgraph`, `~/Downloads/<something>.memgraph`.
   - If no → either:
     - **Mac app or iOS Simulator:** call `captureMemgraph(appName, output)` directly. The tool wraps `leaks --outputGraph`.
     - **Physical iOS device:** `captureMemgraph` does NOT work. `leaks(1)` only attaches to Mac processes. The user must export from Xcode (Memory Graph Debugger button → File → Export Memory Graph). Tell them this clearly; don't try to work around it.

2. **`analyzeMemgraph(path)`**: totals + ROOT CYCLE summaries
   - Reports `totals.instances`, `totals.bytes`, and a list of ROOT CYCLE blocks with class chains.
   - Note the count of ROOT CYCLEs and the dominant class chain.
   - The response includes `suggestedNextCalls`. Follow them.

3. **`classifyCycle(path)`**: match against the 34-pattern catalog
   - Returns `primaryMatch` (highest-confidence pattern) + `allMatches` (everything that fired).
   - Each match carries: `patternId`, `name`, `confidence` (high/medium/low), `fixHint` (textual fix direction), `staticAnalysisHint` (which SwiftLint rule complements this OR explicit gap notice), and `fixTemplate` (Swift before/after code snippet. Adapt the type/method names to the user's codebase via the SourceKit-LSP tools).
   - **If `primaryMatch` is `null`:** the cycle is novel. Skip to step 4 with `findRetainers` to walk the chain manually.
   - **If `primaryMatch.confidence === "high"`:** treat the `fixHint` as authoritative direction. Lift the `fixTemplate` snippet as a starting point. DON'T paste it verbatim, adapt to the actual type/method names. Move to source location.

4. **`findRetainers(path, className)`** OR **`reachableFromCycle(path, rootClassName)`**
   - `findRetainers` returns retain chain paths from a top-level node down to the named class. Useful when classifyCycle didn't fire.
   - `reachableFromCycle` confirms which app-level class is the actual culprit (cycle root) vs collateral retained instances. If a single app-level class dominates `counts`, that's the leak.

5. **Locate in source via SourceKit-LSP tools** (require macOS + full Xcode)
   - **`swiftSearchPattern(filePath, pattern)`**: regex search to find the construct the classifier flagged. The classifier's `suggestedNextCalls` pre-populates the regex (e.g. `\.tag\(`, `\.sink {`, `Task\s*\{`, `for\s+await\s+.+\s+in\s+`).
   - **`swiftGetSymbolDefinition(symbolName, projectRoot, candidatePaths)`**: jump to the declaration of the cycle's app-level class.
   - **`swiftFindSymbolReferences(symbolName, filePath)`**: list every callsite. Useful to gauge fix blast radius and to detect inconsistent capture-list patterns across callsites.
   - **`swiftGetSymbolsOverview(filePath)`**: orientation when you land in an unfamiliar file.
   - **`swiftGetHoverInfo(filePath, line, character)`**: disambiguate `self` captures (a class self in a closure can leak; a struct self can't).

6. **Propose a fix in chat (don't apply yet)**
   - Combine the `fixHint` from `classifyCycle` with the actual code from the source-bridging tools.
   - Show the proposed change as a diff. Wait for the user's `Apply` before calling `Edit`.

7. **Verify with diff**
   - User exports a fresh `.memgraph` after applying the fix.
   - Call **`verifyFix(before, after, expectedPatternId)`**: returns `PASS` / `PARTIAL` / `FAIL` per pattern + bytes freed + instances released.
   - If `PASS`: done. If `PARTIAL` / `FAIL`: the fix didn't address the right capture. Loop back to step 4.

## Playbook B: perf-hangs

**Symptom shape:** "App freezes for 1.5s when I tap X" / "Main thread is stuck."

**Steps:**

1. **Locate or capture a `.trace`**
   - If user has one → ask for path.
   - If not → `listTraceDevices` to find sim/device UDID, then `recordTimeProfile(deviceId, attachAppName, durationSec, output)` with `template: "Time Profiler"`.

2. **`analyzeHangs(tracePath)`**: parses xctrace's `potential-hangs` schema. Reports Hang vs Microhang counts + top N longest. The user-perceptible threshold is 250ms.

3. **If hangs concentrate at a specific call site:** use `swiftSearchPattern` with patterns like `DispatchQueue\.main\.sync` or `Task\s*\{` to find synchronous main-thread offenders.

4. **For sample-level hotspots:** `analyzeTimeProfile(tracePath)` returns top symbols. Note: `xctrace export --xpath '...time-profile'` SIGSEGVs on heavy unsymbolicated traces. The tool surfaces a structured workaround notice when this hits. Open the trace in Instruments first, then re-export from CLI.

## Playbook C: ui-jank (animation hitches)

**Symptom shape:** "Scroll feels janky" / "Animations stutter."

1. **`recordTimeProfile`** with `template: "Animation Hitches"`.
2. **`analyzeAnimationHitches(tracePath, minDurationMs: 100)`**: reports by-type counts + count of user-perceptible hitches (>100ms). 100ms is Apple's threshold for "user notices."
3. If hitches concentrate on a specific `View`, `swiftFindSymbolReferences` to scope which screens render with that view.

## Playbook D: app-launch-slow

**Symptom shape:** "Cold launch takes 4 seconds" / "Splash screen lingers."

1. **`recordTimeProfile`** with `template: "App Launch"` and `launchBundleId`.
2. **`analyzeAppLaunch(tracePath)`**: returns cold/warm classification + per-phase breakdown (process-creation, dyld-init, ObjC-init, AppDelegate, first-frame).
3. If `appdelegate-init` dominates: `swiftSearchPattern` for synchronous work in `application(_:didFinishLaunchingWithOptions:)`. Common offenders: `NSPersistentContainer.loadPersistentStores`, `SDK.start()` calls, `URLSession` warm-up.

## Playbook E: verify-fix

**Symptom shape:** "I applied a fix, did the cycle/regression go away?"

For **memgraph-side** verification (cycles):

1. **`diffMemgraphs(before, after)`**: totals + class-count deltas + cycles new/gone/persisted.
2. Look for the originally-classified cycle in `cycles.goneFromBefore`. If still in `cycles.persisted`, the fix didn't address the right capture. Escalate.
3. **`verifyFix(before, after, expectedPatternId)`**: for CI gating. Returns PASS/PARTIAL/FAIL per pattern + bytes freed.
4. **`classifyCycle(after)`**: confirm no NEW patterns appeared.

For **trace-side** verification (hangs / animation hitches / app launch):

1. **`compareTracesByPattern(before, after, category)`**: same PASS/PARTIAL/FAIL shape but for `.trace` bundles. `category` is `hangs`, `animation-hitches`, or `app-launch`.
2. Threshold semantics:
   - `hangs` PASS when `after.longestMs <= hangsMaxLongestMs` (default 0. Must be zero hangs)
   - `animation-hitches` PASS when `after.longestMs <= hitchesMaxLongestMs` (default 100ms. Apple's user-perceptible threshold)
   - `app-launch` PASS when `after.totalMs <= appLaunchMaxTotalMs` (default 1000ms)
3. **PARTIAL** = reduced from before but still above threshold. **FAIL** = same or worse.

## Common pitfalls: don't fall into these

- **`captureMemgraph` on physical iOS devices.** Doesn't work. `leaks(1)` is Mac-only. Use Xcode's Memory Graph Debugger button + File → Export.
- **`xctrace --attach` for the Leaks template.** Silently produces empty data. Apple bug (`libmalloc not initialized`). Use the Time Profiler template + post-hoc `analyzeAllocations` instead, or capture `.memgraph` separately and run `analyzeMemgraph`.
- **`analyzeTimeProfile` on heavy unsymbolicated traces.** xctrace's export crashes with SIGSEGV. The tool returns a structured workaround notice. Open the trace in Instruments first to symbolicate, then re-export from CLI.
- **Assuming `[weak self]` fixes everything.** It doesn't. For `concurrency.async-sequence-on-self` (the `for await ... in seq { use(self) }` pattern), the iteration itself holds the actor isolation context. `[weak self]` is a no-op. The fix is to capture only the values you need *outside* the loop or `task.cancel()` in `deinit`.
- **Fixing the wrong cycle.** If multiple ROOT CYCLEs exist, prioritize by `transitiveBytes` (added in v1.4). The cycle that pins the most memory is the leverage. `analyzeMemgraph` returns this in `cycles[].transitiveBytes`.
- **Skipping `verifyFix`.** A fix that "looked right" is not a fix until the diff confirms the cycle is gone. Especially before merging or closing the ticket.

## Catalog reference (34 patterns)

The classifier covers the leak families that account for ~95% of real-world iOS retain cycles. Browse the live catalog as MCP resources at `memorydetective://patterns/{patternId}`. Every pattern has a markdown body with name, fix hint, and how to confirm via runtime evidence.

Categories:
- **SwiftUI**: including `.tag()` modifier, `_DictionaryStorage`/WeakBox, `ForEachState`, `@EnvironmentObject` back-refs, `@Observable`+`@State` modal leaks, `NavigationPath` retention
- **Combine**: `.sink` cancellable cycles, `.assign(to: \.x, on: self)`
- **Swift Concurrency**: `Task { }` capturing self, `AsyncStream` continuation, `AsyncSequence` on self (incl. `NotificationCenter.notifications(named:)`), Swift 6.2 `Observations { }` closure
- **UIKit / Foundation**: `Timer.scheduledTimer(target:selector:)`, `CADisplayLink`, `UIGestureRecognizer.addTarget`, `NSKeyValueObservation`, `URLSession` delegate, `NotificationCenter` block-form observer, `DispatchSource` event handler, `delegate` not declared `weak`
- **WebKit**: `WKUserContentController.add` script-message-handler, including the 3-link bridge cycle
- **Core Animation**: `CAAnimation.delegate` strong-retain, custom `CALayer` subclass + non-UIView delegate
- **Core Data**: `NSFetchedResultsController.delegate`
- **SwiftData**: `ModelContext` + `Actor` cycle through `DefaultSerialModelExecutor` (FB13844786, fixed iOS 18 beta 1)
- **Coordinator pattern**: child holding parent strongly
- **RxSwift**: DisposeBag + method reference trap
- **Realm**: `NotificationToken` + change closure

## When to give up and ask the user

- Memgraph capture fails repeatedly (process exits before `leaks` attaches). Ask for `os_log` trail to understand the exit path.
- Cycle has zero app-level classes (all framework internals). Usually means the leak is in a third-party library or in a SwiftUI internal observation graph. Ask for the user's Podfile/SPM dependency list.
- `analyzeTimeProfile` SIGSEGV doesn't resolve via the Instruments-then-export workaround. Accept that sample-level is unavailable for this trace; fall back to `analyzeHangs` + `analyzeAllocations`.

## Tooling summary (so you don't have to call `tools/list`)

```
Read & analyze (13)
  analyzeMemgraph, findCycles, findRetainers, countAlive, reachableFromCycle,
  diffMemgraphs, verifyFix, classifyCycle, analyzeHangs, analyzeAnimationHitches,
  analyzeTimeProfile, analyzeAllocations, analyzeAppLaunch, logShow

Capture / record (3)
  recordTimeProfile, captureMemgraph, logStream

Discover (2)
  listTraceDevices, listTraceTemplates

Render (1)
  renderCycleGraph (Mermaid + Graphviz DOT)

CI / test integration (1)
  detectLeaksInXCUITest (experimental)

Swift source bridging (5)
  swiftGetSymbolDefinition, swiftFindSymbolReferences, swiftGetSymbolsOverview,
  swiftGetHoverInfo, swiftSearchPattern

Pipeline awareness (1, meta)
  getInvestigationPlaybook
```

Plus 5 MCP prompts (slash commands): `/investigate-leak`, `/investigate-hangs`, `/investigate-jank`, `/investigate-launch`, `/verify-cycle-fix`.

Plus 33 catalog resources at `memorydetective://patterns/{patternId}`.
