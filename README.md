# memorydetective-plugin

> Claude Code plugin marketplace for [memorydetective](https://github.com/carloshpdoc/memorydetective): iOS leak hunting + performance investigation as a one-command install.

## Install

```
/plugin marketplace add carloshpdoc/memorydetective-plugin
/plugin install memorydetective@memorydetective-plugin
```

That's it. You get:

- The `memorydetective` MCP server auto-registered (pulled from npm via `npx`)
- 28 MCP tools + a `/perf-investigate` skill for disciplined investigation
- 5 slash commands (`/investigate-leak`, `/investigate-hangs`, `/investigate-jank`, `/investigate-launch`, `/verify-cycle-fix`)
- 34 cycle-pattern resources browsable at `memorydetective://patterns/{patternId}`, every classification carrying `fixHint` + `staticAnalysisHint` + `fixTemplate` (Swift before/after snippet)

Full docs: [plugins/memorydetective/README.md](./plugins/memorydetective/README.md).

## Why a plugin AND an npm package?

Both are first-class. Plugin is for Claude Code's native UX (zero-friction install + slash commands). The npm package is universal: install it directly to use with Cursor, Cline, Claude Desktop, Continue, or any MCP-compatible client.

The plugin wraps the npm package; it doesn't fork it. Single source of truth on npm, single repo for code.

## Repo layout

```
memorydetective-plugin/
├── .claude-plugin/
│   └── marketplace.json           ← marketplace catalog (lists 1 plugin)
└── plugins/
    └── memorydetective/
        ├── .claude-plugin/
        │   └── plugin.json        ← plugin manifest (v1.7.0)
        ├── .mcp.json              ← MCP server registration (npx pulls from npm)
        ├── skills/
        │   └── perf-investigate/
        │       └── SKILL.md       ← /perf-investigate skill body
        ├── README.md
        ├── LICENSE
        └── CHANGELOG.md
```

## Versioning

Plugin tracks the MCP server's minor version. `memorydetective-plugin@1.7.x` pulls `memorydetective@^1.7` via `npx`. Patch releases are auto-resolved without bumping the plugin. Major server bumps (e.g. `2.0.0`) require a plugin manifest update.

## Source

- Plugin (this repo): https://github.com/carloshpdoc/memorydetective-plugin
- MCP server: https://github.com/carloshpdoc/memorydetective
- npm: https://www.npmjs.com/package/memorydetective

## License

Apache-2.0. See [LICENSE](./LICENSE).
