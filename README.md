# screendoc-mcp

> **Automatic, versioned PDF documentation for any app — driven by a Model Context Protocol (MCP) server.**
> Point it at a mobile or web project and it launches the real app, walks the UI, screenshots every screen and state, and builds a versioned, trackable PDF. Re-run after changes and it re-captures only what changed.

`screendoc-mcp` is an **MCP server** for AI coding agents (Claude Code, Claude Desktop, and any MCP-compatible client). Register it once and ask your agent to *"document this app"* — it detects the platform, maps the screen graph, captures the UI with [Maestro](https://maestro.mobile.dev/), and renders a polished PDF you can hand to a team, a client, or your future self.

---

## What it does

- **Documents the real app, not the code.** It drives the live UI (iOS simulator / Android emulator) and captures actual screens, popups, alerts, and empty/loading/error states.
- **Versioned & incremental.** Each run produces a numbered PDF tied to a git commit. `update_document` diffs since the last documented commit and re-captures **only affected screens**, adding change badges and a changelog.
- **Screenshot-driven visual docs.** The `visual-flow` format renders one screen per page — a device frame beside a source-linked breakdown (navigation, buttons, APIs, storage, deep links, analytics, and more).
- **Safe by design.** It works **only on its own documentation branch** (or a `.docmcp/` folder in non-git projects) and never touches other branches, stashes, or commits on your behalf.
- **Live progress & auto-cleanup.** Long runs stream per-screen progress; after a successful PDF, raw screenshots (already baked into the PDF) are cleaned up automatically to save space.

## Who it's for

Mobile and web teams using **Expo / React Native** (and, soon, Playwright-driven web) who want up-to-date visual documentation without maintaining it by hand — QA references, onboarding material, client deliverables, and design/engineering handoffs.

---

## Requirements

| Need | Why |
|------|-----|
| **Node.js ≥ 20** | Runs the MCP server. |
| **[Maestro](https://maestro.mobile.dev/)** | Drives the app UI and takes screenshots (iOS + Android). |
| **Google Chrome / Chromium / Edge** | Headless HTML→PDF rendering. |
| **iOS Simulator or Android Emulator** | A booted device for capture. |
| An **MCP client** (e.g. Claude Code) | To call the tools. |

Install Maestro:

```bash
curl -Ls https://get.maestro.mobile.dev | bash
```

## Install

```bash
git clone https://github.com/vivek-kubvt/screendoc-mcp.git
cd screendoc-mcp
npm install
npm run build
```

## Register with your MCP client

Point your client at the built server (`dist/server.js`):

```json
{
  "mcpServers": {
    "doc-mcp": {
      "command": "node",
      "args": ["/absolute/path/to/screendoc-mcp/dist/server.js"]
    }
  }
}
```

Tools accept an optional `projectRoot` argument; when omitted they operate on the server's working directory.

> **Naming:** the package/repo is `screendoc-mcp`, but the MCP **server id stays `doc-mcp`** (the key above), so tools are namespaced `mcp__doc-mcp__*`. Rename the id in `src/server.ts` if you prefer the `mcp__screendoc-mcp__*` namespace.

---

## Quick start

Ask your agent to run the pipeline, or call the tools directly:

```text
1. project_scan          → detect platform + build the screen graph
2. plan_capture          → generate editable Maestro flows (one per UI chunk)
3. run_flow chunk:"..."  → walk the UI and capture each chunk's screens
4. reconcile_capture     → check what landed; get exact fixes for anything missing
5. create_document       → build the versioned PDF (format: visual-flow for rich pages)
```

For the simplest path, `create_document` runs the **full pipeline end-to-end** (scan → capture → coverage gate → PDF). Later, after code changes:

```text
update_document          → re-capture only changed screens, rebuild with change badges + changelog
```

The generated PDF lands in `.docmcp/output/` as `‹project›-docs-v‹n›-‹date›-‹commit›.pdf`.

### Handy options

- `format: "visual-flow"` — rich, one-screen-per-page docs (vs. the default flowing layout).
- `keepCaptures: true` — keep raw screenshots instead of auto-deleting them after the PDF builds.
- `allowGaps: true` (default) — build even with uncaptured screens; gaps are always reported, never hidden.
- `allowDeepLinks: true` — opt into the legacy deep-link fallback (off by default; capture is Maestro-flow-first).

---

## Tools

| Tool | Purpose |
|------|---------|
| `doc_status` | Report documentation state: branch check, platform, coverage, last PDF. *(read-only)* |
| `project_scan` | Detect the platform and build the full screen graph. *(read-only preview)* |
| `plan_capture` | Group screens into chunks and scaffold one editable Maestro flow per chunk. |
| `run_flow` | Run a chunk's flow to capture its screens in one UI walk. |
| `reconcile_capture` | Diff expected screens vs. screenshots on disk; print exact fixes for gaps. |
| `capture_screen` | Re-capture a single screen (retakes / spot fixes). |
| `extract_annotations` | Source-analysis pass that pre-fills per-screen annotations (nav, APIs, storage…). |
| `save_recipe` | Store a coordinate-tap recipe to reach a tricky screen precisely. |
| `set_credential` | Store a secret (login, test data, API key) in the gitignored secrets file. |
| `create_document` | Full pipeline → build the versioned PDF. |
| `update_document` | Incremental refresh → re-capture changed screens + changelog. |
| `list_documents` | List every generated document with version, date, commit, and changed screens. |

---

## Output formats

Set the PDF's look with a named **format preset** (committed in `.docmcp/format.json` so every run renders identically):

- `default` (green), `brand` (indigo), `compact` (dense grayscale) — cover + table of contents + per-screen sections.
- `visual-flow` — one screen per page with a device frame and a source-linked breakdown, plus cross-cutting catalogs (APIs, data models, storage keys, deep links, analytics) aggregated across all screens.

Pin a preset with a bare name, or add shallow overrides:

```json
{
  "preset": "brand",
  "overrides": {
    "colors": { "accent": "#7A1FA2" },
    "cover":  { "eyebrow": "Acme Product Docs" }
  }
}
```

For `visual-flow`, the accent color is auto-detected from your project's palette (`app.json`, Tailwind config, theme files) unless overridden. A missing or malformed config falls back to `default` with a warning — a bad file never breaks a run.

The rich `visual-flow` sections are filled from per-screen annotation files at `.docmcp/annotations/<screen-id>.json`. Run `extract_annotations` to pre-fill the mechanical fields (with source line numbers), then author the semantic gaps (`purpose`, what each control does). See [`src/output/annotations.ts`](src/output/annotations.ts) for the full shape.

---

## How capture works

Capture is **Maestro-flow-first**: the app is documented by driving its real UI, not by deep-linking routes. `plan_capture` writes an editable `.docmcp/flows/<chunk>.yaml` per chunk (auth · one per tab · root modals) with `# TODO` markers where you fill taps; `run_flow` executes them and `reconcile_capture` tells you exactly what's missing and how to fix it (retry the flow, or drop a PNG manually). For screens that can't be reached generically, record a **coordinate-tap recipe** (`save_recipe`) — it always wins over the chunk flow. Credentials are injected via `{{secretKey}}` from the gitignored secrets store.

<details>
<summary><strong>Field notes (validated on iOS, Expo/RN)</strong></summary>

- **Custom-drawn RN controls** often aren't in the a11y tree — tap by point `%` instead of text.
- **First-launch takeovers** (onboarding, permission prompts) — dismiss with guarded `when: { visible }` blocks before asserting the target.
- **Live/WebRTC surfaces** render on a native overlay Maestro can't tap — capture with `xcrun simctl io <udid> screenshot` and document the limitation.
- **iOS has no hardware back** — tap the header chevron by point rather than `- back`.
- **Stop before irreversible actions** (send/commit/start-call) — capture the confirm screen, then back out.

</details>

---

## Develop

```bash
npm run build        # compile to dist/
npm run dev          # run the server via tsx (stdio)
npm run typecheck    # type-check without emitting
node scripts/mcp-smoke.mjs   # boot the server + call a tool over MCP
```

Releases use SemVer with git tags: `npm run release:patch|minor|major|rc` bumps `package.json` and tags `vX.Y.Z`. Branch flow: `feature/* → main → staging → production`.

## Contributing

Issues and PRs welcome. Please run `npm run typecheck && npm run build`, update `CHANGELOG.md` under **[Unreleased]**, and target `main` (see the [PR template](.github/pull_request_template.md)).

## License

Not yet licensed. Add a `LICENSE` file (MIT is a common choice for tooling) and set the `license` field in `package.json` to make it official.

---

## Keywords

MCP server · Model Context Protocol · automatic app documentation · screenshot documentation · PDF documentation generator · React Native docs · Expo documentation · iOS & Android screenshots · Maestro automation · visual documentation · Claude Code · AI documentation tool · versioned docs · screen graph · UI documentation

<sub>**Suggested GitHub repo topics:** `mcp` · `model-context-protocol` · `documentation` · `pdf-generation` · `react-native` · `expo` · `maestro` · `ios` · `android` · `screenshots` · `claude` · `automation` · `developer-tools`</sub>
