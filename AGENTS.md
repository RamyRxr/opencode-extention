# AGENTS.md — project memory for the OpenCode VS Code extension

> **Read this first when a new chat starts in this folder.** It contains the
> full context of the planning conversation from 2026-10-20 between Ramy and
> the assistant. If Ramy asks "what are we doing here?" / "what did we talk
> about?" / "what is this project?" — everything you need is below.
> Detailed version of the same plan lives in `README.md`.

## What this project is

A **VS Code extension** that provides a better interface for running OpenCode
— Ramy's own words: *"a better interface to run opencode in it, because the
current opencode interface has an annoying empty bar on the right, and I
can't see the current folder's code files."*

The chosen direction (Ramy picked it): **an extension just like the Claude
Code VS Code extension** — agent chat in a sidebar, results/interactions
displayed in VS Code's **Secondary Side Bar** (the right-hand panel where
Copilot Chat lives), files visible in VS Code's native Explorer on the left.

## The conversation, condensed

1. **Ramy's problem with the current OpenCode TUI:**
   - An annoying empty/wasted bar on the right side of the interface.
   - No way to see the current project's files while chatting with the agent.
2. **Feasibility was confirmed:** OpenCode is client–server. The TUI is just
   one client of a local HTTP API (143 endpoints). Building a custom client
   is fully supported via `@opencode/client`.
3. **Options presented:** web app / desktop app (Tauri/Electron) / embedded
   SDK. Ramy answered: *"more like extension in vscode, just like the claude
   extension."*
4. **Secondary Side Bar requirement:** Ramy wants results displayed in the
   secondary bar. Verified in official VS Code docs (2026-10): stable API
   cannot contribute views there directly (only `activitybar` / `panel`;
   `contribSecondarySideBar` is a *proposed* API). Solution: contribute to
   the Activity Bar, then one-time move (drag or guided
   `workbench.action.moveView`) — VS Code persists the layout across
   sessions.
5. **This folder was then created** with `README.md` (detailed plan) and this
   `AGENTS.md` (memory). **No code has been written yet** — this is the
   pre-scaffold planning stage.

## Decisions already made — don't re-litigate without asking Ramy

1. **Transport = HTTP API** with `@opencode/client` + `Service.ensure()` /
   `Service.headers(endpoint)` from `@opencode/client/service` (discovers or
   starts the local OpenCode background service and supplies auth).
   Rejected for v1: ACP (`opencode acp` — stdio JSON-RPC, Zed-style) and
   embedding the official web UI in an iframe (someone else's layout).
2. **UI shell = `WebviewView`** (sidebar webview), contributed via a view
   container in `contributes.viewsContainers.activitybar`, moved to the
   Secondary Side Bar once by the user or a first-run guided command.
   Do NOT depend on proposed API `viewsContainers.secondarySideBar`.
3. **No custom file tree** — use VS Code's native Explorer (it directly fixes
   complaint #2). Integrate instead: click file in chat →
   `vscode.window.showTextDocument`; tool diffs → `vscode.diff`; explorer
   context menu → `@file` mention in prompt.
4. **Stack:** TypeScript, esbuild bundling, `@opencode/client`, webview UI
   styled with `--vscode-*` CSS variables (matches any theme). OpenCode V2
   docs are the only source of truth for API questions.

## Research findings (verified 2026-10-20 — landscape, don't redo)

| Extension | What it does | Verdict |
|---|---|---|
| Official `sst-dev.opencode` | Opens the TUI in a split terminal | Doesn't solve our complaints |
| `bilalbentoumi.vsc-opencode` | Iframes official web UI in side panel | Not customizable |
| `YasirArafat.opencode-ai-chat` | Chat-only sidebar | Shallow VS Code integration |

**The gap we're filling:** Claude-Code-style depth — native chat sidebar +
real editor integration (files/diffs in VS Code itself). No one has shipped
that for OpenCode.

Key API endpoints planned: `session.create` / `session.prompt`,
`event.subscribe` (live-only, no replay → load history before subscribing),
`permission` reply endpoints, `/api/fs/list|find|read`, `/api/vcs/status|diff`,
`session/{id}/interrupt|compact`. The `/api/fs/*` routes are experimental.

Useful entry-point snippet:

```ts
import { OpenCode } from "@opencode/client"
import { Service } from "@opencode/client/service"

const endpoint = await Service.ensure()
const client = OpenCode.make({ baseUrl: endpoint.url, headers: Service.headers(endpoint) })
const session = await client.session.create({ location: { directory: workspaceFolder } })
await client.session.prompt({ sessionID: session.id, text: "..." })
for await (const event of client.event.subscribe()) { /* stream to webview */ }
```

## Current state & next step

- ✅ Idea validated, direction chosen, research done, folder + README + this
  memory file created.
- ⬜ **Next action: scaffold the extension** (milestone 1 in README §5):
  `package.json` with view container/views/commands, `src/extension.ts`,
  esbuild config, F5 debug loop. HQP convention: initialize a git repo for
  this project at scaffold time.
- Milestones after that: connect → minimal chat UI → file/diff integration →
  agent controls (permissions, model/agent picker) → polish/onboarding.

## Standards for this project (from the HQP parent folder)

Note: HQP's root `CLAUDE.md` is NOT auto-loaded by OpenCode V2 (it only reads
`AGENTS.md`), so the relevant standards are restated here.

- Explain the *why* behind tool/architecture choices — Ramy is learning
  web development and wants decisions to double as learning.
- Prefer well-established, well-documented tech over bleeding-edge options.
- README with setup instructions; tests for real functionality (not
  coverage-chasing, not zero); clean commit history; no leftover
  `console.log`/debug code; no hardcoded secrets (env vars only).
- This is a real, shippable HQP project → done means a working `.vsix` /
  Marketplace presence, not just "runs on my machine".
- Each HQP project is its own folder and its own git repo.

## Docs (source of truth)

- OpenCode V2: https://opencode.ai/v2/docs/ (client guide:
  /docs/build/client, API: /docs/api) — do not use V1 docs or web search for
  V2 API questions.
- VS Code extension API: https://code.visualstudio.com/api/references/contribution-points
  (viewsContainers = `activitybar` + `panel` only) and UX guidelines:
  /api/ux-guidelines/sidebars (Secondary Side Bar placement rules).
