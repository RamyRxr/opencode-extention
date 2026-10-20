# opencode-vscode — OpenCode, but inside VS Code

> A VS Code extension that runs OpenCode with a Claude-Code-style sidebar UI:
> your files stay visible in VS Code's native explorer on the left, the chat
> lives in the Secondary Side Bar on the right — no wasted empty panels
> anywhere.

**Status:** planning / pre-scaffold · **Last updated:** 2026-10-20

---

## 1. The idea (problem → solution)

### The problem
The current OpenCode interface (the terminal TUI) has two annoyances:

1. An **annoying empty bar on the right** — wasted screen space.
2. **No visibility of the project's files** — you can't browse the current
   folder's code while chatting with the agent.

### The solution
A real VS Code extension where:

- **Files**: VS Code's built-in Explorer shows the project tree (left side) —
  free, native, with git status and all VS Code features.
- **Chat**: OpenCode's agent UI runs in a **webview in the Secondary Side Bar**
  (right side) — the same place Copilot Chat lives.
- **Editor**: clicking a file in chat opens the real VS Code editor; tool-call
  diffs open in VS Code's real diff viewer.

Layout sketch:

```
VS Code window
├── Explorer (built-in)   ← project files, git status — left side
├── Editor area           ← real files, real diffs (vscode.diff)
└── Secondary Side Bar    ← OpenCode chat webview — right side
      ├── streaming messages + tool calls
      ├── session / model / agent switcher
      └── permission buttons (Allow once / Always / Reject)
```

## 2. Why a VS Code extension (research, 2026-10-20)

OpenCode is **client–server**: the TUI is just one client of a local HTTP API
(143 endpoints, OpenAPI spec at `/openapi.json`). Any UI can be a client —
that's what this extension will be.

Existing options and why none of them fit:

| Extension | What it does | Solves our complaints? |
|---|---|---|
| Official `sst-dev.opencode` | Launches the TUI in a split terminal | ❌ Same TUI, same empty bar |
| `bilalbentoumi.vsc-opencode` | Iframes the official web UI into a side panel | ⚠️ Someone else's layout, not customizable |
| `YasirArafat.opencode-ai-chat` | Custom chat sidebar (streaming, model picker) | ⚠️ Chat-only, shallow VS Code integration |

**The gap:** nobody has built the Claude-Code-style experience — native
sidebar chat with *deep* editor integration (files and diffs happening in VS
Code itself). That's this project.

## 3. Key decisions (and why)

### Decision 1 — Transport: HTTP API, not ACP
Use `@opencode/client` (TypeScript client generated from OpenCode's API
contract) + `Service.ensure()` from `@opencode/client/service` to
discover/start/auth the local background service.

- ✅ Same 143-endpoint API: sessions, streaming events, filesystem, VCS, PTY,
  permissions — everything the UI needs.
- ✅ Fastest path to build on; browser-compatible entrypoints mean the webview
  can stream directly.
- ❌ ACP (`opencode acp`, stdio JSON-RPC, what Zed uses) was considered and
  rejected for v1: more protocol plumbing, fewer features exposed.

### Decision 2 — Placement: Secondary Side Bar via one-time move
**Constraint (verified in official VS Code docs, 2026-10):** stable
`contributes.viewsContainers` only accepts `activitybar` and `panel`.
Direct `secondarySideBar` contribution exists only as a *proposed* API
(`contribSecondarySideBar`) — not stable, not safe to publish with.

**Our approach:**
1. Contribute the view container to the **Activity Bar** (stable API).
2. First run: show a guided step that opens the Secondary Side Bar
   (`workbench.action.toggleAuxiliaryBar`) and runs
   `workbench.action.moveView` so the user confirms placement once
   (or just drag the view there once).
3. VS Code **persists the layout across sessions** — after the first move it
   always opens in the right-hand bar.

### Decision 3 — File tree: do NOT rebuild one
Use VS Code's native Explorer for browsing. The extension only *integrates*:
- click file in chat → `vscode.window.showTextDocument` (jump to line)
- tool-call changes → `vscode.diff` (real diff editor)
- file mentions → context menu on explorer files → `@file` into the prompt
- OpenCode's `/api/fs/*` endpoints remain available for in-chat file lookup
  (quick-open style search) if we want it later.

### Decision 4 — Stack
- **Language:** TypeScript (extension host + webview)
- **Bundling:** esbuild (standard for VS Code extensions, fast F5 loop)
- **Backend client:** `@opencode/client`
- **Webview UI:** start minimal (vanilla TS or small React+Vite build) —
  theme with `--vscode-*` CSS variables so it matches the user's theme exactly
- **Tested against:** OpenCode V2 API only

## 4. API surface we'll use

| Purpose | Endpoint / client call |
|---|---|
| Discover/start local service | `Service.ensure()` + `Service.headers(endpoint)` |
| Create session | `POST /api/session` with `location.directory = workspace folder` |
| Send prompt | `POST /api/session/{id}/prompt` |
| Live updates (streaming) | `GET /api/event` (async iterable) |
| Message history | `GET /api/session/{id}/message` |
| Session list / switch agent / model | `GET /api/session`, `POST .../agent`, `POST .../model` |
| Permission approve/reject | `GET /api/permission/request`, `POST .../reply` |
| Files (in-chat lookup) | `GET /api/fs/list`, `GET /api/fs/find`, `GET /api/fs/read/*` |
| Git status / diffs | `GET /api/vcs/status`, `GET /api/vcs/diff`, `GET /api/session/{id}/diff` |
| Interrupt / compact | `POST /api/session/{id}/interrupt`, `POST .../compact` |

**Gotchas to remember:**
- Event subscriptions are **live-only** (no replay) → load history first, then
  subscribe.
- `/api/fs/*` routes are marked experimental → schemas may shift slightly.
- Auth headers come from `Service.headers()` — never hardcode tokens.

## 5. Build order (milestones)

1. **Scaffold** — `package.json` (contributes: view container + views +
   commands), `src/extension.ts`, esbuild config, F5 debug loop works.
2. **Connect** — `Service.ensure()` → create session in workspace folder →
   prompt → event streaming into the webview.
3. **Minimal chat UI** — message list + input box, live streaming, markdown.
4. **File/diff integration** — click-to-open, `vscode.diff` for tool changes,
   explorer context-menu mentions. *(This is where it beats the TUI.)*
5. **Agent controls** — permission buttons, model/agent picker, session list,
   interrupt.
6. **Polish** — theming (`--vscode-*`), keybindings, status bar, first-run
   move-to-secondary-sidebar onboarding.

Definition of done for v1: chat in the Secondary Side Bar, files visible in
the Explorer, diffs opening in the real VS Code diff viewer.

## 6. Development & distribution

```bash
# dev loop
code .            # open this folder in VS Code
F5                # launches "Extension Development Host" window with the extension loaded
# (tsc + esbuild watchers rebuild on save; reload with "Developer: Reload Window")

# package for real use
npx vsce package  # produces opencode-vscode-x.y.z.vsix
code --install-extension opencode-vscode-x.y.z.vsix

# publish (optional, free publisher account needed)
npx vsce publish          # VS Marketplace
# Open VSX for VSCodium/Cursor users
```

Requirements: opencode CLI installed on PATH (the extension talks to the
service it manages), VS Code 1.90+.

## 7. Project standards (inherited from HQP)

- README with setup instructions (this file) + live link once shipped
- Tests for real functionality (not coverage-chasing, not zero)
- Clean commit history, no debug `console.log` left in
- No hardcoded secrets — env vars only
- Deployed/working, not just "runs locally" → here that means published as a
  real `.vsix` / Marketplace extension
- Own git repo for this project (not yet initialized — do it at scaffold time)

## 8. References

- OpenCode V2 docs: https://opencode.ai/v2/docs/
- JS client guide: https://opencode.ai/v2/docs/build/client
- HTTP API reference: https://opencode.ai/v2/docs/api
- VS Code contribution points: https://code.visualstudio.com/api/references/contribution-points
- VS Code sidebar UX guidelines: https://code.visualstudio.com/api/ux-guidelines/sidebars
- Secondary Side Bar placement issue (proposed API): microsoft/vscode#198087, #264346
- Webview view sample: https://github.com/microsoft/vscode-extension-samples/tree/main/webview-view-sample
