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
| "crashed in production", "hangs from real users", ".mxdiagnostic", "MetricKit payload", "TestFlight crash report" | **production-postmortem** | `analyzeMetricKitPayload` |
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

3. **`classifyCycle(path)`**: match against the 36-pattern catalog
   - Returns `primaryMatch` (highest-confidence pattern) + `allMatches` (everything that fired).
   - Each match carries: `patternId`, `name`, `confidence` (high/medium/low), `fixHint` (textual fix direction), `staticAnalysisHint` (which SwiftLint rule complements this OR explicit gap notice), and `fixTemplate` (Swift before/after code snippet. Adapt the type/method names to the user's codebase via the SourceKit-LSP tools).
   - **If `primaryMatch` is `null` AND `analyzeMemgraph` reported `leakCount: 0`:** this is the abandoned-memory shape. Skip to step 3b. v1.9 added a dedicated classifier for it.
   - **If `primaryMatch` is `null` BUT leaks exist:** the cycle is novel. Skip to step 4 with `findRetainers` to walk the chain manually.
   - **If `primaryMatch.confidence === "high"`:** treat the `fixHint` as authoritative direction. Lift the `fixTemplate` snippet as a starting point. DON'T paste it verbatim, adapt to the actual type/method names. Move to source location.

3b. **`analyzeAbandonedMemory(beforePath, afterPath)`** (when `leakCount: 0` on both sides)
   - The leak shape that `leaks(1)` cannot see: orphaned KVO observers, never-removed NotificationCenter blocks, runaway caches, singleton-retained payloads. The objects are technically reachable from KVO's global registry (or your singleton), so they don't count as "leaked" in the strict sense. They still grow without bound.
   - The pipeline is the verify-fix loop run in reverse: take a `before` snapshot of the suspected leaky state (e.g. after 10 cycles of the suspected flow), and an `after` snapshot of a clean baseline (e.g. cold launch). Compare them.
   - Returns `growthByClass[]` ranked by absolute delta. Each entry carries `classification` (`kvo-observer-orphaned` / `notificationcenter-observer-leaked` / `cache-too-aggressive` / `singleton-retains-payload` / `unknown-growth`), `confidence`, and a contextual `hint`. The classifier escalates when `NSKeyValueObservance` co-occurs with another large-delta class (the KVO-observer-orphan signal).
   - You can ALSO call `analyzeMemgraph` on a single `.memgraph` with `referenceTreeTopN: 20` to surface the top abandoned-memory classes from a SINGLE snapshot when you don't have a baseline yet.

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

3. **For sample-level hotspots:** `analyzeTimeProfile(tracePath)` returns top symbols. Note: `xctrace export --xpath '...time-profile'` SIGSEGVs on heavy unsymbolicated traces. The tool surfaces a structured workaround notice when this hits. Open the trace in Instruments first, then re-export from CLI.

4. **Classify what was blocking the main thread (v1.9):** chain the time-profile result back into `analyzeHangs` with `topFramesByHangStartNs: { '<startNs>': '<topFrame>' }` so each hang carries a `mainThreadViolations[]` entry classifying the work as `sync-io` (read/write/fsync/NSData blocking inits), `db-lock` (SQLite mutex / NSManagedObjectContext save), `network` (NSURLConnection sendSynchronousRequest, blocking nw_connection), or `lock-contention` (pthread / os_unfair_lock / dispatch_semaphore_wait / dispatch_sync). Each kind has a canonical fix chain (move to Task.detached / background DispatchQueue / async API).

5. **If hangs concentrate at a specific call site:** use `swiftSearchPattern` with patterns like `DispatchQueue\.main\.sync` or `Task\s*\{` to find synchronous main-thread offenders.

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

## Playbook F: production-postmortem (v1.18)

**Symptom shape:** "I have crashes / hangs from real production users, not from my sim" / "the user sent me a `.mxdiagnostic` file" / "I want to analyze TestFlight crash payloads."

**Why a separate playbook:** `captureMemgraph` and `recordTimeProfile` are local-capture tools, useless against problems that only happen on real-user devices. MetricKit (Apple's `MXMetricManager`) writes JSON payloads to the app's MetricKit directory on TestFlight / App Store builds, which the dev airdrops to their Mac. This playbook handles that lane.

**Steps:**

1. **`analyzeMetricKitPayload({ payloadPath })`** for a single file, or **`{ payloadDir }`** to aggregate across all `.mxdiagnostic` files in a directory (the typical case — devs accumulate payloads over weeks).
2. The result has 4 sections; prioritize in this order:
   - **`crashCluster[]`** — grouped by `groupBy` (default `exception-type`). Each entry has top-frame label + `affectedBuilds[]` so you can spot a regression introduced by a specific build version.
   - **`hangHotspots[]`** — sorted by `hangDurationMs` (after extracting the leading number from Apple's localized strings like `"5.4 sec"` or `"20秒"`). >1s = user-visible freeze.
   - **`cpuExceptions[]`** — sorted by `totalCPUTimeMs`. The "your code burned the battery in the background" lane.
   - **`diskWriteExceptionDiagnostics[]`** — `writesCausedMB` ranked. The "your code wrote 1 GB to disk in one session" lane.
3. Each entry carries raw `binaryUUID + offsetIntoBinaryTextSegment + binaryName` for downstream dSYM symbolication. We do NOT symbolicate in v1.18.
4. Follow `result.suggestedNextCalls`:
   - Retain-cycle-shaped top frame (`_objc_release`, `objc_msgSend`, `_dispatch_block_invoke`) → suggests `findCycles` on a memgraph of the same code path.
   - SQLite / network / lock top frame → suggests `analyzeHangs` with `includeStackClassification` against a trace of a repro scenario.

**What MetricKit does NOT cover:**

- Simulator builds. Apple does not generate `.mxdiagnostic` from the sim. This is local-only data from real devices.
- iOS 18 has a 24-48h delivery delay and a "new bundle id probation" window. Tool surfaces `payloadCount: 0` honestly when the directory is empty — do not invent diagnoses.

## Common pitfalls: don't fall into these

- **`captureMemgraph` on physical iOS devices.** Doesn't work. `leaks(1)` is Mac-only. Use Xcode's Memory Graph Debugger button + File → Export.
- **`xctrace --attach` for the Leaks template.** Silently produces empty data. Apple bug (`libmalloc not initialized`). Use the Time Profiler template + post-hoc `analyzeAllocations` instead, or capture `.memgraph` separately and run `analyzeMemgraph`.
- **`analyzeTimeProfile` on heavy unsymbolicated traces.** xctrace's export crashes with SIGSEGV. The tool returns a structured workaround notice. Open the trace in Instruments first to symbolicate, then re-export from CLI.
- **Assuming `[weak self]` fixes everything.** It doesn't. For `concurrency.async-sequence-on-self` (the `for await ... in seq { use(self) }` pattern), the iteration itself holds the actor isolation context. `[weak self]` is a no-op. The fix is to capture only the values you need *outside* the loop or `task.cancel()` in `deinit`.
- **Fixing the wrong cycle.** If multiple ROOT CYCLEs exist, prioritize by `transitiveBytes` (added in v1.4). The cycle that pins the most memory is the leverage. `analyzeMemgraph` returns this in `cycles[].transitiveBytes`.
- **Skipping `verifyFix`.** A fix that "looked right" is not a fix until the diff confirms the cycle is gone. Especially before merging or closing the ticket.

## Catalog reference (36 patterns)

The classifier covers the leak families that account for ~95% of real-world iOS retain cycles. Browse the live catalog as MCP resources at `memorydetective://patterns/{patternId}`. Every pattern has a markdown body with name, fix hint, and how to confirm via runtime evidence.

Categories:
- **SwiftUI**: including `.tag()` modifier, `_DictionaryStorage`/WeakBox, `ForEachState`, `@EnvironmentObject` back-refs, `@Observable`+`@State` modal leaks, `NavigationPath` retention, **v1.9: `swiftui.observable-write-on-every-render`** (mutating an `@Observable` inside `body`, scheduling infinite re-render; fix is to move the mutation to `.onChange` / `.task(id:)` or compute it as a derived property)
- **Combine**: `.sink` cancellable cycles, `.assign(to: \.x, on: self)`
- **Swift Concurrency**: `Task { }` capturing self, `AsyncStream` continuation, `AsyncSequence` on self (incl. `NotificationCenter.notifications(named:)`), Swift 6.2 `Observations { }` closure
- **UIKit / Foundation**: `Timer.scheduledTimer(target:selector:)`, `CADisplayLink`, `UIGestureRecognizer.addTarget`, `NSKeyValueObservation`, `URLSession` delegate, `NotificationCenter` block-form observer, `DispatchSource` event handler, `delegate` not declared `weak`, **v1.9: `uikit.viewcontroller-retained-after-pop`** (VC subclass alive in heap without parent/presenter edges; usually a closure / Combine sink / KVO observation retaining a popped VC)
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
Read & analyze (17)
  analyzeMemgraph, findCycles, findRetainers, countAlive, reachableFromCycle,
  diffMemgraphs, analyzeAbandonedMemory (v1.9: leakCount=0 family), verifyFix,
  classifyCycle, analyzeHangs (v1.9: optional mainThreadViolations[];
  v1.14: hang-risks schema; v1.17: supportStatus[] always carries both),
  analyzeAnimationHitches, analyzeTimeProfile, analyzeAllocations,
  analyzeAppLaunch, analyzeNetworkActivity (v1.14), analyzeMemoryFootprint
  (v1.15, VM resident/dirty/jetsam), analyzeEnergyImpact (v1.15, battery
  drain), analyzeLeakTimeline (v1.15, xctrace leaks as time series),
  logShow

Capture / record (4)
  recordTimeProfile (v1.17: bundleStatus on-disk viability field),
  recordViaInstrumentsApp (v1.16, macOS 26.x escape hatch; v1.17: catches
  saves outside watchDir via AppleScript document query),
  captureMemgraph, logStream

Verify-fix orchestration (3, v1.8)
  bootAndLaunchForLeakInvestigation, replayScenario (v1.15: screenshotDir
  per step), captureScenarioState

Discover (3)
  listTraceDevices, listTraceTemplates, inspectTrace (v1.11; v1.17:
  fault-tolerant fallback returns ok:true on wedged bundles)

Synthesize (1, v1.13)
  summarizeTrace (single-call cross-schema synthesis with pre-rendered
  markdown card; v1.15 chains analyzeNetworkActivity; v1.18 D-02:
  schemaDiscovery cache shaves 600-3000ms wall-clock vs v1.17)

Production diagnostics (1, v1.18)
  analyzeMetricKitPayload (42nd tool, [mg.production]). Post-mortem
  ingest of Apple MetricKit .mxdiagnostic JSON payloads from real-device
  TestFlight / App Store builds. Three input forms: payloadPath
  (single file), payloadDir (aggregate across all .mxdiagnostic files
  in a directory), payloadJson (raw, for in-memory callers). Outputs
  crashCluster (groupBy: "exception-type" | "binary" | "top-frame"),
  hangHotspots (with localized-duration parsing: "5.4 sec" / "20秒"),
  cpuExceptions, diskWriteExceptions. NO symbolication in v1 — ship
  raw binaryUUID + offsetIntoBinaryTextSegment. Simulator does NOT
  generate MetricKit (Apple-side); frame as post-mortem analyzer.
  Cross-tool chain hints: objc_release-style top frame → findCycles
  hint; sqlite top frame → analyzeHangs with mainThreadViolations
  classifier.

Render (1)
  renderCycleGraph (Mermaid + Graphviz DOT)

Ops (1, v1.9)
  cleanupTraces (dryRun-default preview/delete of .trace bundles under
  MEMORYDETECTIVE_TRACE_ROOT)

CI / test integration (3)
  detectLeaksInXCTest (v1.9, per-test unit-test gate),
  detectLeaksInXCUITest, compareTracesByPattern. Both detectLeaks* tools
  accept outputHtmlPath for self-contained HTML reports you can upload as
  a CI artifact.

Swift source bridging (5)
  swiftGetSymbolDefinition, swiftFindSymbolReferences, swiftGetSymbolsOverview,
  swiftGetHoverInfo, swiftSearchPattern

Pipeline awareness (1, meta)
  getInvestigationPlaybook
```

42 MCP tools total. v1.18 added analyzeMetricKitPayload (production
post-mortem lane) + audit-close trio (open-enum SupportStatusKind +
schemaDiscovery cache + real-Apple integration tests). v1.17 was
reliability-only. Highlights to teach the agent:
- `verifyFix.expectedAliveClasses` accepts per-entry { pattern, mode } where
  mode is "exact" | "substring" | "regex" (v1.17). Bare strings still mean
  substring for backwards compat.
- `countAlive` exposes `excludeFrameworkNoise` / `additionalNoisePatterns` /
  `unsuppressClassPatterns` / `noiseAuditMode` so the actionable view can
  be tuned per app (v1.17).
- Variable-size classes (NSData, NSString, CFData) report
  `instanceSizeBytesMin / Max / Median` (v1.17). Fixed-size classes still
  report a single `instanceSizeBytes` value.
- `analyzeMetricKitPayload` (v1.18) is the right tool when the user has
  `.mxdiagnostic` files from TestFlight / App Store. Distinct from
  `captureMemgraph` (local capture) and `recordTimeProfile` (local CLI
  recording). Inputs: payloadPath (single file), payloadDir (aggregate),
  or payloadJson (raw). Cross-tool chain hints fire automatically:
  retain-cycle-shaped top frames suggest `findCycles`; sqlite / network /
  lock top frames suggest `analyzeHangs` with `includeStackClassification`
  on the corresponding trace.
- `SupportStatusKind` is open (v1.18 D-01). Downstream code can author
  new kinds without a type bump. Internal call sites use
  `KnownSupportStatusKind` so typos in inline literals still fail-build.

Plus 6 MCP prompts (slash commands): `/investigate-leak`, `/investigate-hangs`, `/investigate-jank`, `/investigate-launch`, `/verify-cycle-fix`, `/summarize-trace`.

Plus 34 catalog resources at `memorydetective://patterns/{patternId}`.

## Environment flags worth knowing

The MCP server reads these on startup. The server logs the active redaction mode once on stderr so you can spot misconfigurations.

Every boolean below accepts the **strtobool** truthy set (v1.17+, case-insensitive): `1 / true / t / yes / y / on` (truthy) and `0 / false / f / no / n / off` (falsy). Unrecognized values emit a one-time stderr warning per variable and fall back to the default; pre-v1.17 only `1` worked.

| Variable | Default | When to set |
|---|---|---|
| `MEMORYDETECTIVE_REDACTION` | `balanced` | Set to `strict` for sensitive sessions (masks hostnames, IPv4, bundle ids in addition to home paths and token-shaped secrets). `off` only for local-only debugging. |
| `MEMORYDETECTIVE_ALLOW_LAUNCH` | unset | Boolean. `bootAndLaunchForLeakInvestigation` requires this to be truthy before running `xcodebuild` + `xcrun simctl launch`. Without it the tool returns `ok: false` with `state: "launchNotAllowed"`. |
| `MEMORYDETECTIVE_MAX_RECORDING_SECONDS` | `300` | Cap on `recordTimeProfile.durationSec`. Raise (max `3600`) when you genuinely need long captures. |
| `MEMORYDETECTIVE_TRACE_ROOT` | `~/Library/Application Support/memorydetective/traces` | Default directory for `.trace` bundles when `recordTimeProfile.output` is a relative path. Also the default scan path for `cleanupTraces`. |
| `MEMORYDETECTIVE_ALLOW_EXTERNAL_CLEANUP` | unset | Boolean. Required to be truthy for `cleanupTraces` to scan/delete outside `MEMORYDETECTIVE_TRACE_ROOT`. Default-deny on destructive disk operations outside the configured boundary. |
| `MEMORYDETECTIVE_SUPPRESS_PLATFORM_ADVISORY` | unset | Boolean. Set once you have an iOS 18 sim runtime installed and no longer need the macOS 26.x reminder banner. Also silences v1.17 stderr warnings for unrecognized boolean values and `schemaDiscovery` TOC fetch failures. |
| `MEMORYDETECTIVE_AUTO_OPEN_INSTRUMENTS` | unset | Boolean. Makes `recordTimeProfile` open Instruments.app on a timeout (the macOS 26.x regression). v1.17 probes `MANIFEST.plist` before opening so wedged 52K stub bundles do not surface a Document Missing Template Error dialog. |
| `MEMORYDETECTIVE_PREFLIGHT_XCTRACE` | unset (auto) | Boolean + `auto`. Controls the 2-second pre-flight probe on `recordTimeProfile` that detects the macOS 26.x wedge fast. Auto-enables on macOS 26.x simulator attach; truthy forces on; falsy forces off. |

## When `captureMemgraph` fails on macOS 26.x

Apple regressed `leaks --outputGraph` on macOS 26.x: it aborts with `Failed to get DYLD info for task` whenever the target was not launched with `MallocStackLogging=1`. As of v1.8 the plugin handles this end to end.

1. **Detect.** `captureMemgraph` returns `ok: false` with `workaroundNotice.issue === "minimal-corpse"`. The result also carries `suggestedNextCalls` pointing at the `recordTimeProfile` (Allocations) fallback.
2. **Recover via relaunch (preferred).** Call `bootAndLaunchForLeakInvestigation({ workspace, scheme, simulator: { name: "iPhone 15" } })`. It builds, boots, installs, and launches with `MallocStackLogging=1` propagated via `SIMCTL_CHILD_*`, and returns the host PID + simulator UDID. Re-call `captureMemgraph` with that PID.
3. **Fallback if relaunch is unavailable.** If you cannot rebuild the project (e.g. the user only has the running app), follow `suggestedNextCalls` to record an Allocations trace and inspect with `analyzeAllocations`. Or fall back to Xcode manual export: Debug -> View Memory Graph Hierarchy -> File -> Export Memory Graph, then pass the resulting `.memgraph` to `analyzeMemgraph`.

`getInvestigationPlaybook({ kind: "memgraph-leak" })` carries a `troubleshooting` field documenting these paths inline so you can branch deterministically.

## Verify-fix loop (v1.8)

When the user wants deterministic before/after evidence (typical after a refactor or a SwiftUI cycle fix):

1. **Set up the scenario.** Call `bootAndLaunchForLeakInvestigation` to get a clean process with `MallocStackLogging=1`.
2. **Amplify the suspected leak.** Call `replayScenario({ simulatorUDID, actions: [...], repeat: 5 })` to drive the UI through the leaking flow N times. Tap targets accept `label`, `elementId`, or `coords`. Soft dependency on Cameron Cooke's [axe](https://github.com/cameroncooke/AXe) CLI; the tool returns a structured install hint when missing.
3. **Capture before.** Call `captureScenarioState({ simulatorUDID, pid, outputDir, label: "before" })`. Writes `before.memgraph`, `before.png`, `before.ui.json`.
4. **User ships the fix and rebuilds.**
5. **Capture after.** Repeat steps 1-3 with `label: "after"`.
6. **Verdict.** Call `diffMemgraphs(before.memgraph, after.memgraph)` then `verifyFix(before, after, expectedPatternId: "<pattern>")`. Returns PASS / PARTIAL / FAIL with bytes freed.
