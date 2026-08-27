# Build `wb-tool-gateway` — Controlled Tool Execution plugin

You are building **one** plugin inside a larger multi-agent build: the
Sovereign AI Workbench, a set of Cordis plugins mounted on top of the
DeepSeek Harness agent runtime. Eleven other agents are independently
building the other plugins; you will never see their code and they will
never see yours. The only thing that lets your work integrate correctly
later is that you follow the frozen contract below exactly, without
improvising names.

## Required reading, in this order

1. `workbench/DESIGN.md` — read in full once for context, then read §6.7
   ("`wb-tool-gateway` — Controlled Tool Execution") closely; it is your
   contract card. Also read §4, §7 in full (especially `WbToolManifest`,
   `WbToolRiskLevel`, `WbToolNetworkAccess`), §7.5 (the frozen `wb_*` tool
   name table — the tools that will register manifests with you), and §12.
2. `workbench/AGENTS.md` — general build process, §4 ("If your plugin
   registers a model-facing tool" — read this even though you don't register
   tools yourself, because you consume manifests from plugins that do), §9
   "done" checklist.
3. `workbench/packages/wb-types/src/index.ts` — frozen shared types. Import
   `WbToolManifest`, `WbToolRiskLevel`, `WbToolNetworkAccess`; never redefine
   them.
4. `docs/cookbook/adding-a-tool.md` (repo root) — read in full. You need to
   understand the tool registration API (`ctx.tools.register`,
   `defineTool`) well enough to know what a tool "name" is and how a
   registry/directory-style plugin conventionally sits alongside it in this
   repo.
5. `docs/testing.md` (repo root) — "Unit" tier and "Prefer the real
   implementation over a mock" sections.
6. Skim `packages/extensions/tool-cordis/tests/cordis-lifecycle.spec.ts` for
   Cordis testing idioms.

## Your role

Own the **tool manifest directory** — structured metadata about every tool
in the system — so `wb-policy` (a sibling plugin) has something to evaluate
instead of guessing from a tool's name string, and so adding a 13th tool
later never requires touching `wb-policy`'s code.

- Package: `@mrpl/dsh-workbench-tool-gateway`, at
  `workbench/packages/wb-tool-gateway/`.
- Provides: `ctx.wbToolGateway` implementing `WbToolGatewayService` from
  `wb-types` (`registerManifest(manifest)`, `getManifest(toolId)`).
- This is a **directory, not an executor** — you never intercept or run a
  tool call yourself (that's `wb-policy`'s job, hooking the harness's
  `tools/pre-execute`). You only answer "what kind of thing is this tool."
- For harness-native tools that will never call `registerManifest`
  themselves (`dsh-tool-fs`, `dsh-tool-web`, `dsh-tool-bash`, and whatever
  else you find registered by default in this repo's base bundle), ship a
  `Config`-driven static table mapping their well-known tool names to a
  manifest, so they are governed too — `DESIGN.md` §6.7 requires this
  explicitly (the capability matrix in `Plugin_design_idea` covers
  Files/Python/Web/DB for every agent, not just workbench-added tools).
  Find the actual registered names of these harness-native tools in this
  repo (search `packages/fs`, `packages/web`, `packages/shell` or similar)
  rather than guessing them.

## Dependencies you consume

None at the service level — you are a leaf directory. `registerManifest`
callers (`wb-vision`, `wb-artifacts`, siblings you won't see built) and the
reader (`wb-policy`, also a sibling) only need your public interface, which
is already frozen in `wb-types`. You do not need to fake anything to test
your own plugin in isolation — write test manifests inline.

## Non-goals — do not build these

- No ALLOW/DENY decision logic of any kind — that belongs entirely to
  `wb-policy`. If you find yourself writing an `if (riskLevel === ...)
  deny()`, stop — that's the wrong plugin.
- No tool execution, no `tools/pre-execute` hook of your own — you are read
  as data by whatever plugin *does* hook that (`wb-policy`), you don't hook
  it yourself.
- Do not silently invent a manifest for an unknown tool — `getManifest` for
  an unregistered, non-static-table tool name returns `undefined`; let the
  caller (`wb-policy`) decide what to do with that.

## Workflow — tests first, then implementation, then verification

**Step 1 — write failing tests before any implementation.** At minimum:
- `registerManifest(manifest)` followed by `getManifest(manifest.toolId)`
  returns the same manifest.
- `getManifest` for a name that was never registered and isn't in the
  static harness-native table returns `undefined` — not a default/guessed
  manifest.
- Every harness-native tool name in your static table resolves to a
  manifest with `riskLevel`/`dataClassificationCeiling`/`networkAccess`
  values that make sense for what that tool actually does (e.g. a raw
  filesystem write tool should not default to `networkAccess: 'external'`,
  a web-search tool should not default to `riskLevel: 'local'`) — write one
  assertion per harness-native tool you cover, naming it explicitly so a
  reviewer can see your reasoning per tool.
- `registerManifest` called twice for the same `toolId` with different
  content: decide and test one explicit behavior (last-write-wins, or throw
  on duplicate) — do not leave this undefined behavior; document your choice
  in the README.
- A manifest with a `toolId` that doesn't match the harness's own registered
  tool name for that tool is still stored as given (you don't cross-validate
  against the live tools registry — that's out of scope; note this
  explicitly if you considered and rejected doing so).
- HMR-safety test per `docs/testing.md`: dispose the fiber, assert
  previously `registerManifest`'d entries contributed *by that fiber* are
  gone (but note in your test whether static-table entries, which aren't
  fiber-scoped the same way, should survive — decide and assert one way).

**Step 2 — implement** the minimum plugin code to pass those tests.

**Step 3 — expand tests** for edge cases found while implementing.

**Step 4 — verify**, from `workbench/packages/wb-tool-gateway/`:
```
pnpm install
pnpm run typecheck
pnpm run lint
pnpm run test
pnpm run build
```

**Step 5 — self-check** against `AGENTS.md` §9, then write `README.md` per
`AGENTS.md` §8 (service API, the full static harness-native tool table with
your reasoning per entry, duplicate-registration behavior, and a
"Deviations" section for any harness-native tool name you couldn't confirm
from this repo's source and had to infer).

## If you hit a gap

Do not invent a workbench-wide name to fill it. Note it under "Deviations"
in your `README.md` and append a dated bullet to `DESIGN.md` §12.
