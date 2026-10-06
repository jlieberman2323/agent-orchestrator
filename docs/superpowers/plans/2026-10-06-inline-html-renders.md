# Inline HTML Renders in Chat Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** An agent in an AO chat session can publish a self-contained HTML page (a chart, table, diagram, or mockup). The page renders inline in the thread above the agent's final reply, follows the app theme, and is sandboxed away from AO. This is AO's version of T3 Code's `html_render`.

**Architecture:**
1. The agent writes an HTML file and runs `ao render <file> --title …`.
2. The daemon injects a small theme, size, and link bootstrap after the doctype.
3. It stores an immutable copy as `<dataDir>/attachments/<sessionId>/render-<id>.html`.
4. It records a `system` conversation activity with `detail.event = "render"` on the turn that is running. This is the same discriminator pattern `steer` and `context.reset` use, so there is no migration and no new activity kind.
5. The renderer shows that activity as an `<iframe sandbox="allow-scripts allow-forms">` pointing at `GET /api/v1/sessions/{id}/renders/{renderId}`.
6. That route also sends `Content-Security-Policy: sandbox allow-scripts allow-forms`. The page therefore has an opaque origin wherever it is opened, and the daemon's CORS middleware refuses its `Origin: null` requests.
7. Before publishing, the agent can run `ao render --check <file>`. The daemon stores a temporary `check-` render, and the desktop app loads it in a hidden, in-memory-partition `WebContentsView` through the existing browser-runtime socket. The agent gets a PNG, the content height, and the console messages.

**Tech Stack:** Go (chi, cobra, existing sqlite store), React 19 + Tailwind v4 + shadcn primitives, react-i18next, Electron 33, vitest + Testing Library.

**Spec:** the Design section of this document. It was derived from T3 Code (`~/Development/t3code` @ `442735897f`; PR pingdotgg/t3code#15968, scroll fix #16283) and checked against this branch's base, `origin/main` @ `64e8193e7`.

---

## Design

### T3 Code → AO mapping

| T3 Code piece | T3 file | AO equivalent (this plan) |
|---|---|---|
| `html_render` MCP tool on T3's own MCP server | `apps/server/src/mcp/toolkits/html/tools.ts`, `handlers.ts` | `ao render` CLI → `POST /api/v1/sessions/{id}/renders` (Task 5, Task 4). AO injects no MCP server into agents; they learn `ao` from the `using-ao` skill. |
| `html_preview` (PNG, `contentHeight`, console) + pinned Chrome-for-Testing download (~120 MB) + SOCKS public-only proxy (~1.3k loc) | `apps/server/src/htmlRender/{PreviewBrowser,headlessChrome,publicProxy}.ts` | `ao render --check` (Task 8). It runs in a hidden Electron view through the existing `ao browser` path, so it needs no download and no proxy. |
| Page stored as a thread attachment; bootstrap injected into `<head>` | `apps/server/src/htmlRender/HtmlRender.ts`, `packages/shared/src/htmlRender.ts` | `attachmentstore.PutRender` (Task 1) + `injectRenderBootstrap` (Task 2) |
| Tool result `{attachmentId,title,height,heights}` → timeline row `html-render` | `apps/web/src/session-logic.ts`, `components/chat/MessagesTimeline.*` | A `system` activity with `detail.event:"render"` (Task 3). It renders through a new `ActivityRow` branch (Task 6). `runsOf` already keeps evented activities out of tool-call runs (`ChatWorkspace.tsx:1593`). |
| Sandboxed iframe; theme passed in a `#t3-theme=` fragment and then via `ui/notifications/host-context-changed`; auto-height via `ui/notifications/size-changed`; links via `ui/open-link` (MCP Apps JSON-RPC over postMessage) | `apps/web/src/components/chat/HtmlRenderFrame.tsx`, `components/files/BrowserDocumentFrame.tsx` | `RenderFrame.tsx` (Task 6). It uses the same three protocol methods and an `#ao-theme=` fragment. |
| `Content-Security-Policy: sandbox …` on the asset route | `apps/server/src/http.ts` | Same header on the render route (Task 4) |
| Heights measured at 9 widths during publish | `HtmlRender.measure` | **Deferred.** The agent's `--height` is the first-paint box, and the page reports its real height after load. |
| Local image paths inlined as data URIs | `HtmlRender.inlineLocalImages` | **Deferred.** Phase 2 does it in the CLI, which reads files with the agent's own permissions. |

### Decisions

1. **CLI, not MCP.** `ChatStartConfig.MCPServers` exists, but nothing populates it. Every agent already gets the `using-ao` skill pointer (`session_manager/manager.go:4965`). A CLI works for every chat provider with no MCP plumbing, and it follows the shape `ao preview` already uses.
2. **An activity, not a table or a kind.** `UpsertActivity` allocates the sequence, binds the turn, fires the CDC trigger, and cascades on conversation delete. The `system` kind with a `detail.event` discriminator is the established pattern (`steer.go:640` `recordSteer`).
3. **Reuse `ErrNoActiveTurn`.** `awaitAcknowledgedTurn` (`controller.go:2334`) already answers "is a turn in flight". Steer reuses the same sentinel, so renders do too.
4. **Daemon origin plus a CSP sandbox, never the preview origin.** `ao-preview.<id>.localhost` is a loopback origin, and `corsMiddleware` deliberately lets it call the daemon API (`httpd/cors.go`). A render must have an opaque origin; `exactAllowedOrigins` excludes `null`, so the daemon answers it with 403. Render files therefore use a `render-` prefix that `attachmentstore.validateName` rejects. As a result `previewFile`, `MaterializeWorkspace` and `ImportWorkspace` can never serve or project them.
5. **Bootstrap goes after the doctype, not inside `<head>`.** Go's RE2 has no backreferences, so T3's raw-text scanner can't be ported. `golang.org/x/net/html` isn't a dependency either. Under the HTML parsing algorithm, a `<style>` or `<script>` right after the doctype opens the implied `<head>`. The page's own `<html>` start tag then merges its attributes, and its `<head>` start tag is ignored as a parse error. Leading BOM, whitespace and comments are skipped so the page never drops into quirks mode.
6. **The always-on prompt gets two sentences, and its cap rises from 220 to 260 words.** This was a user decision on 2026-10-06. `aoSkillPointer` is 213 words today, and the cap is enforced at `manager_test.go:5603`. The two sentences tell agents when to use `ao render` and where its guide is. They pass the ASD-STE100 structural linter. The detailed rules stay in `commands/render.md` and `ao render --help`.
7. **Theme is read from AO's semantic tokens at runtime.** These are `--color-bg-primary`, `--color-text-primary` and the rest, defined in `frontend/src/styles/tokens.css`, which DESIGN.md §6 names as the source of truth. `getComputedStyle` resolves their `var()` chains.
8. **Chat sessions only.** In a TUI session the CLI gets `SESSION_MODE_MISMATCH` and points the agent at `ao preview`.

### Accepted risks (the same ones T3 shipped with)

- Page scripts run in the renderer process inside an opaque-origin frame. They can fetch public URLs and other loopback/LAN services, but not the AO daemon (403 on `Origin: null`). The existing workspace preview is strictly more privileged.
- A page can take keyboard focus by script. Links open externally only while the frame has focus and user activation is live (`navigator.userActivation.isActive`).

### Decided with the user (2026-10-06)

- **Chart palette: five fixed colors per theme.** DESIGN.md §6 forbids one-off hex in app UI. AO's `--chart-1…5` tokens are grayscale, and the status colors carry session meaning. So `--chart-1` comes from `--color-brand-logo` and `--chart-2…6` are T3's five fixed hues per theme. These colors reach agent pages only, never AO chrome.
- **Execution: subagent-driven.**
- **Prompt: two sentences, cap raised to 260 words** (decision 6).
- **Self-check is in scope** (`ao render --check`), not Phase 2. It was offered as "Task 9". It became Task 8 so that real-app verification (Task 9) stays last.

---

## Global Constraints

- All AO state stays under `~/.ao`. Renders live in `<dataDir>/attachments/<sessionId>/`. Never use an OS app-data directory.
- No new SQLite migration. Don't hand-edit `backend/internal/storage/sqlite/gen/*`.
- The API is code-first. Edit `backend/internal/httpd/controllers/dto.go` and `backend/internal/httpd/apispec/specgen/build.go`, run `npm run api`, and commit `backend/internal/httpd/apispec/openapi.yaml` together with `frontend/src/api/schema.ts`.
- The iframe `sandbox` attribute is exactly `allow-scripts allow-forms`. Never add `allow-same-origin`, `allow-popups`, `allow-top-navigation` or `allow-modals`.
- The render route always sends `Content-Security-Policy: sandbox allow-scripts allow-forms`, `X-Content-Type-Options: nosniff` and `Content-Type: text/html; charset=utf-8`.
- Limits:
  - HTML ≤ 1 MiB (`1 << 20` bytes), checked in both the CLI and the service.
  - Title is trimmed, non-empty, and ≤ 200 runes.
  - Height is clamped to 80–2000 CSS px.
- UI is built from shadcn primitives (`components/ui/button.tsx`, `components/ui/dialog.tsx`, `components/ui/tooltip.tsx`) and Lucide icons at 14px, per DESIGN.md §9. Every user-visible string goes through `t()`, with keys in all 8 locale files under `frontend/src/renderer/i18n/`.
- `aoSkillPointer()` stays ≤ 260 words. Task 5 raises the cap at `manager_test.go:5603` from 220.
- Conventional commits, one PR, branch `feat/inline-html-renders`.

## Review Focus

1. **A render file requested through the workspace preview origin**, e.g. `/preview/files/.ao/attachments/render-x.html` on `ao-preview.<id>.localhost`, must be not-found. That origin can call the daemon API. Pinned by Task 1's test.
2. **A render URL opened top-level** (devtools, the user's browser) must still get an opaque origin from the CSP header, so a `fetch` to the daemon gets 403. Pinned by Task 4's test.
3. **`ao render` after the turn ended, or in a TUI session**, must give a clear error and leave no orphan file. Pinned by Task 3's test.
4. **Flipping light/dark while a thread is open** must restyle the page without reloading it, so the iframe `src` never changes. Pinned by Task 6's test.
5. **A page that starts with a BOM or `<!-- comment -->` before `<!doctype html>`** must stay in standards mode, with the bootstrap placed after the doctype. Pinned by Task 2's test.

---

## File map

| File | Responsibility |
|---|---|
| `backend/internal/attachmentstore/render.go` (new) | Render file names, `PutRender`, `OpenRender`, `RemoveRender` |
| `backend/internal/attachmentstore/store.go` (modify) | Extract `writeCanonical` from `PutCanonical` so `PutRender` shares it |
| `backend/internal/service/chat/render_bootstrap.js` (new) | In-page bootstrap: theme, size reporting, link routing |
| `backend/internal/service/chat/render_html.go` (new) | `injectRenderBootstrap`, default theme CSS |
| `backend/internal/service/chat/render.go` (new) | `PublishRender`, `Controller.recordRender`, `RenderFiles`, errors |
| `backend/internal/service/chat/service.go` (modify) | `Options.Renders` / `Service.renders` |
| `backend/internal/daemon/daemon.go:404` (modify) | Wire `Renders: attachmentstore.New(cfg.DataDir)` |
| `backend/internal/httpd/controllers/conversation_render.go` (new) | POST publish, GET sandboxed file |
| `backend/internal/httpd/controllers/conversations.go:78-86` (modify) | `Renders` field + two routes |
| `backend/internal/httpd/controllers/dto.go` (modify) | `PublishRenderRequest`, `PublishRenderResponse`, `RenderIDParam` |
| `backend/internal/httpd/apispec/specgen/build.go` (modify) | Two operations + schema names |
| `backend/internal/httpd/api.go:190` (modify) | Wire `Renders` |
| `backend/internal/cli/render.go` (new), `root.go:210` (modify) | `ao render` |
| `backend/internal/skillassets/using-ao/commands/render.md` (new), `SKILL.md` (modify) | Agent guide + catalog row |
| `backend/internal/session_manager/manager.go:4965`, `manager_test.go:5603` (modify) | "Showing pages in chat" section; cap 220 → 260 |
| `frontend/src/renderer/lib/render-frame.ts` (new) | Pure helpers: detail parsing, theme, protocol messages |
| `frontend/src/renderer/components/chat/RenderFrame.tsx` (new) | Inline frame, fit-to-content, expand dialog |
| `frontend/src/renderer/components/chat/ChatTimelineItems.tsx:850-864` (modify) | `ActivityRow` branch |
| `frontend/src/renderer/types/conversation.ts:397` (modify) | `"render"` event + `render?: RenderRef` |
| `frontend/src/renderer/i18n/*.json` (modify, 8 files) | `chat.render.expand` |
| `frontend/src/main/render-frame-guard.ts` (new), `frontend/src/main.ts:703` (modify) | Block a render frame from navigating away |
| `frontend/src/main/render-check.ts` (new), `frontend/src/main.ts:1327` (modify) | Hidden-view page check for `__render-check` |
| `backend/internal/service/chat/render.go`, `daemon/daemon.go`, `controllers/conversation_render.go` (modify, Task 8) | `CheckRender`, broker wiring, `POST …/renders/check` |

---

### Task 1: Store render files beside attachments, unreachable from the preview origin

**Files:**
- Create: `backend/internal/attachmentstore/render.go`
- Modify: `backend/internal/attachmentstore/store.go:94-125` (`PutCanonical`)
- Test: `backend/internal/attachmentstore/render_test.go`

**Interfaces:**
- Produces:
  - `func (s *Store) PutRender(ctx context.Context, id domain.SessionID, renderID string, data []byte) error`
  - `func (s *Store) OpenRender(ctx context.Context, id domain.SessionID, renderID string) (*os.File, fs.FileInfo, error)`
  - `func (s *Store) RemoveRender(ctx context.Context, id domain.SessionID, renderID string) error`
  - unexported `renderName(renderID string) string` → `"render-" + renderID + ".html"`

- [ ] **Step 1: Write the failing test**

```go
package attachmentstore

import (
	"context"
	"io"
	"os"
	"path/filepath"
	"testing"

	"github.com/aoagents/agent-orchestrator/backend/internal/domain"
)

func TestRenderRoundTripStaysOutOfWorkspaceAndPreview(t *testing.T) {
	ctx := context.Background()
	store := New(t.TempDir())
	session := domain.SessionID("proj-1")

	if err := store.PutRender(ctx, session, "r1", []byte("<p>hi</p>")); err != nil {
		t.Fatalf("PutRender: %v", err)
	}
	file, _, err := store.OpenRender(ctx, session, "r1")
	if err != nil {
		t.Fatalf("OpenRender: %v", err)
	}
	body, _ := io.ReadAll(file)
	_ = file.Close()
	if string(body) != "<p>hi</p>" {
		t.Fatalf("body = %q", body)
	}

	// The workspace preview origin may call the daemon API, so it must never serve a render.
	if _, ok := NameFromWorkspacePath(".ao/attachments/" + renderName("r1")); ok {
		t.Fatal("preview path resolved a render file")
	}
	if _, _, err := store.Open(ctx, session, renderName("r1")); err == nil {
		t.Fatal("attachment Open served a render file")
	}
	workspace := t.TempDir()
	if _, err := store.MaterializeWorkspace(ctx, session, workspace, nil); err != nil {
		t.Fatalf("MaterializeWorkspace: %v", err)
	}
	if _, err := os.Stat(filepath.Join(workspace, WorkspaceDir, renderName("r1"))); !os.IsNotExist(err) {
		t.Fatalf("render projected into the worktree: %v", err)
	}

	if err := store.RemoveRender(ctx, session, "r1"); err != nil {
		t.Fatalf("RemoveRender: %v", err)
	}
	if _, _, err := store.OpenRender(ctx, session, "r1"); err == nil {
		t.Fatal("render still readable after RemoveRender")
	}
}

func TestRenderIDsRejectTraversalAndDots(t *testing.T) {
	store := New(t.TempDir())
	for _, id := range []string{"", "../x", "a/b", `a\b`, "a.b", ".."} {
		if err := store.PutRender(context.Background(), "proj-1", id, []byte("x")); err == nil {
			t.Errorf("PutRender(%q) accepted", id)
		}
		if _, _, err := store.OpenRender(context.Background(), "proj-1", id); err == nil {
			t.Errorf("OpenRender(%q) opened something", id)
		}
	}
}
```

`MaterializeWorkspace` takes `beforeWrite func() error` (`store.go:213`) and treats `nil` as no hook (`store.go:259`).

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd backend && go test ./internal/attachmentstore/ -run Render -v`
Expected: FAIL to compile, with `store.PutRender undefined`.

- [ ] **Step 3: Extract the canonical writer in `store.go`**

Replace the body of `PutCanonical` (`store.go:94`) after its `validateName` check with a call to a new private helper, and add the helper below it:

```go
func (s *Store) PutCanonical(ctx context.Context, id domain.SessionID, name string, data []byte) error {
	if err := ctx.Err(); err != nil {
		return err
	}
	if err := validateSessionID(id); err != nil {
		return err
	}
	if err := validateName(name); err != nil {
		return err
	}
	return s.writeCanonical(ctx, id, name, data)
}

// writeCanonical stores bytes in a session's canonical directory only. Callers
// validate the name first: attachment names and render names follow different rules.
func (s *Store) writeCanonical(ctx context.Context, id domain.SessionID, name string, data []byte) error {
	if len(data) == 0 {
		return errEmpty
	}
	if len(data) > MaxFileBytes {
		return errTooLarge
	}
	if s.dataDir == "" {
		return errors.New("attachment data directory is empty")
	}
	sessionRoot, err := s.openCanonicalSession(ctx, id, true)
	if err != nil {
		return fmt.Errorf("open canonical attachment directory: %w", err)
	}
	defer func() { _ = sessionRoot.Close() }()
	if err := writeReaderAtomicRoot(ctx, sessionRoot, ".", name, bytes.NewReader(data), false); err != nil {
		return fmt.Errorf("write canonical attachment: %w", err)
	}
	return nil
}
```

- [ ] **Step 4: Write `render.go`**

```go
package attachmentstore

import (
	"context"
	"errors"
	"fmt"
	"io/fs"
	"os"

	"github.com/aoagents/agent-orchestrator/backend/internal/domain"
)

// Renders are agent-published HTML pages shown inline in chat. They share the
// session's canonical directory so RemoveSession deletes them with the session,
// but validateName rejects their "render-" prefix: a render is never projected
// into a worktree, never imported from one, and never served by the workspace
// preview origin, which may call the daemon API. Only the sandboxed render
// route reads them, through OpenRender.
func renderName(renderID string) string { return "render-" + renderID + ".html" }

func validateRenderID(renderID string) error {
	if len(renderID) > 200 || !validNamePart(renderID, false) {
		return fmt.Errorf("invalid render id %q", renderID)
	}
	return nil
}

// PutRender stores a render's bytes. Unlike Put it touches no worktree.
func (s *Store) PutRender(ctx context.Context, id domain.SessionID, renderID string, data []byte) error {
	if err := ctx.Err(); err != nil {
		return err
	}
	if err := validateSessionID(id); err != nil {
		return err
	}
	if err := validateRenderID(renderID); err != nil {
		return err
	}
	return s.writeCanonical(ctx, id, renderName(renderID), data)
}

// OpenRender opens a stored render for the sandboxed render route.
func (s *Store) OpenRender(ctx context.Context, id domain.SessionID, renderID string) (*os.File, fs.FileInfo, error) {
	if err := ctx.Err(); err != nil {
		return nil, nil, err
	}
	if s.dataDir == "" || validateSessionID(id) != nil || validateRenderID(renderID) != nil {
		return nil, nil, fs.ErrNotExist
	}
	sessionRoot, err := s.openCanonicalSession(ctx, id, false)
	if err != nil {
		return nil, nil, fs.ErrNotExist
	}
	defer func() { _ = sessionRoot.Close() }()
	return openRegularFile(ctx, sessionRoot, renderName(renderID))
}

// RemoveRender deletes a render whose timeline row was never recorded.
func (s *Store) RemoveRender(ctx context.Context, id domain.SessionID, renderID string) error {
	if s.dataDir == "" || validateSessionID(id) != nil || validateRenderID(renderID) != nil {
		return nil
	}
	sessionRoot, err := s.openCanonicalSession(ctx, id, false)
	if errors.Is(err, fs.ErrNotExist) {
		return nil
	}
	if err != nil {
		return err
	}
	defer func() { _ = sessionRoot.Close() }()
	if err := sessionRoot.Remove(renderName(renderID)); err != nil && !errors.Is(err, fs.ErrNotExist) {
		return err
	}
	return nil
}
```

- [ ] **Step 5: Run the package tests**

Run: `cd backend && go test ./internal/attachmentstore/ -v`
Expected: PASS, including the existing `PutCanonical` tests, which pin the refactor.

- [ ] **Step 6: Commit**

```bash
git add backend/internal/attachmentstore/
git commit -m "feat(attachments): store agent HTML renders outside the preview origin"
```

---

### Task 2: Theme/size bootstrap and injection after the doctype

**Files:**
- Create: `backend/internal/service/chat/render_bootstrap.js`
- Create: `backend/internal/service/chat/render_html.go`
- Test: `backend/internal/service/chat/render_html_test.go` (`package chat`, an internal test because the function is unexported)

**Interfaces:**
- Produces: `func injectRenderBootstrap(page string) string` (package `chat`). The page then carries `<style id="ao-theme">`, which the bootstrap rewrites from the `#ao-theme=<json>` fragment and from `ui/notifications/host-context-changed` messages. It posts `ui/notifications/size-changed {height}` and `ui/open-link {url}` to its parent.

- [ ] **Step 1: Write the failing test**

```go
package chat

import (
	"strings"
	"testing"
)

func TestInjectRenderBootstrapLandsAfterTheDoctype(t *testing.T) {
	cases := map[string]struct{ page, wantPrefix, wantSuffix string }{
		"doctype": {
			page:       "<!DOCTYPE html><html lang=\"en\"><head><title>x</title></head><body>b</body></html>",
			wantPrefix: "<!DOCTYPE html><meta name=\"viewport\"",
			wantSuffix: "<html lang=\"en\"><head><title>x</title></head><body>b</body></html>",
		},
		"bom and comment before doctype": {
			page:       "﻿<!-- made by an agent -->\n<!doctype html><p>x</p>",
			wantPrefix: "﻿<!-- made by an agent -->\n<!doctype html><meta name=\"viewport\"",
			wantSuffix: "<p>x</p>",
		},
		"no doctype": {
			page:       "<div>x</div>",
			wantPrefix: "<!doctype html><meta name=\"viewport\"",
			wantSuffix: "<div>x</div>",
		},
	}
	for name, tc := range cases {
		t.Run(name, func(t *testing.T) {
			got := injectRenderBootstrap(tc.page)
			if !strings.HasPrefix(got, tc.wantPrefix) {
				t.Fatalf("prefix = %.160q", got)
			}
			if !strings.HasSuffix(got, tc.wantSuffix) {
				t.Fatalf("page body not preserved verbatim: %.160q", got[len(got)-min(len(got), 160):])
			}
			for _, want := range []string{`<style id="ao-theme">`, "ui/notifications/size-changed", "ui/open-link", "ao-theme="} {
				if !strings.Contains(got, want) {
					t.Errorf("bootstrap missing %q", want)
				}
			}
		})
	}
}

func TestInjectRenderBootstrapKeepsThePagesViewport(t *testing.T) {
	got := injectRenderBootstrap(`<!doctype html><meta name="viewport" content="width=500">`)
	if n := strings.Count(got, `name="viewport"`); n != 1 {
		t.Fatalf("viewport metas = %d, want the page's own only", n)
	}
}

func TestRenderBootstrapCannotCloseItsOwnElements(t *testing.T) {
	if strings.Contains(strings.ToLower(renderBootstrapJS), "</script") {
		t.Fatal("bootstrap script would end its own <script> element")
	}
	if strings.Contains(strings.ToLower(renderDefaultThemeCSS+renderBaseCSS), "</style") {
		t.Fatal("bootstrap CSS would end its own <style> element")
	}
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd backend && go test ./internal/service/chat/ -run 'RenderBootstrap|InjectRender' -v`
Expected: FAIL to compile, with `undefined: injectRenderBootstrap`.

- [ ] **Step 3: Write `render_bootstrap.js`** (ES5, no `</script` anywhere)

```js
// Runs before the page's own styles and scripts. It hands the page the host
// theme, reports the page's content height, and asks the host to open links.
// The messages are the MCP Apps postMessage methods T3 Code also uses, so this
// host could later show upstream MCP apps.
(function () {
  var theme = document.getElementById("ao-theme");
  var framed = window.parent !== window;
  var seq = 0;
  function apply(t) {
    if (!theme || !t || typeof t !== "object" || !t.variables || typeof t.variables !== "object") return;
    var css = ":root{color-scheme:" + (t.appearance === "light" ? "light" : "dark") + ";";
    for (var k in t.variables) {
      if (/^--[a-z0-9-]+$/.test(k)) css += k + ":" + String(t.variables[k]).replace(/[;{}<>]/g, "") + ";";
    }
    theme.textContent = css + "}";
  }
  try {
    var m = /[#&]ao-theme=([^&]*)/.exec(location.hash);
    if (m) {
      apply(JSON.parse(decodeURIComponent(m[1])));
      history.replaceState(history.state, "", location.pathname + location.search);
    }
  } catch (e) {}
  window.addEventListener("message", function (e) {
    var d = e.data;
    var p = d && d.params;
    if (e.source === window.parent && d && d.jsonrpc === "2.0" &&
        d.method === "ui/notifications/host-context-changed" && p && p.styles) {
      apply({ appearance: p.theme, variables: p.styles.variables });
    }
  });
  if (!framed) return;
  document.addEventListener("click", function (e) {
    if (!e.isTrusted) return;
    var link = e.composedPath().find(function (n) { return n && n.matches && n.matches("a[href]"); });
    if (!link) return;
    var url;
    try { url = new URL(link.getAttribute("href"), document.baseURI); } catch (x) { return; }
    if (!/^https?:$/.test(url.protocol) || url.href.split("#")[0] === location.href.split("#")[0]) return;
    e.preventDefault();
    window.parent.postMessage({ jsonrpc: "2.0", id: "ao-link-" + (++seq), method: "ui/open-link", params: { url: url.href } }, "*");
  }, true);
  var last;
  function report() {
    var r = document.documentElement;
    var h = Math.ceil(r.scrollHeight > r.clientHeight ? r.scrollHeight : r.getBoundingClientRect().height);
    if (h === last) return;
    last = h;
    window.parent.postMessage({ jsonrpc: "2.0", method: "ui/notifications/size-changed", params: { height: h } }, "*");
  }
  var observer = window.ResizeObserver ? new ResizeObserver(report) : null;
  if (observer) observer.observe(document.documentElement);
  document.addEventListener("DOMContentLoaded", function () {
    if (observer && document.body) observer.observe(document.body);
    report();
  });
  window.addEventListener("load", report);
})();
```

- [ ] **Step 4: Write `render_html.go`**

The fallback values are AO's own `frontend/src/styles/tokens.css` values (dark `:root`, light `:root[data-theme="light"]`). They only apply when a page is opened without a host theme, for example a download.

```go
package chat

import (
	_ "embed"
	"regexp"
	"strings"
)

//go:embed render_bootstrap.js
var renderBootstrapJS string

// The variables agent pages style against (documented in commands/render.md).
// The host replaces this rule before first paint with the reader's live theme.
const renderDefaultThemeCSS = `:root{color-scheme:dark;--background:oklch(0.210 0.002 250);--foreground:oklch(0.970 0.002 250);--muted:oklch(0.295 0.002 250);--muted-foreground:oklch(0.720 0.002 250);--card:oklch(0.250 0.002 250);--card-foreground:oklch(0.970 0.002 250);--popover:oklch(0.300 0.002 250);--border:oklch(1 0 0 / 7%);--border-strong:oklch(1 0 0 / 4%);--primary:oklch(0.900 0.002 250);--primary-foreground:oklch(0.210 0.002 250);--accent:oklch(0.900 0.002 250);--accent-foreground:oklch(0.210 0.002 250);--success:#4ade80;--warning:#f06445;--destructive:oklch(0.704 0.191 22.216);--code:#79b0dc;--link:#79b0dc;--chart-1:#79b0dc;--chart-2:#2dd4bf;--chart-3:#fbbf24;--chart-4:#c084fc;--chart-5:#fb7185;--chart-6:#a3e635;--radius:0.5rem;--font-sans:ui-sans-serif,system-ui,sans-serif;--font-mono:ui-monospace,Menlo,Consolas,monospace}` +
	`@media (prefers-color-scheme: light){:root{color-scheme:light;--background:oklch(0.97 0 0);--foreground:oklch(0.18 0 0);--muted:oklch(0.935 0 0);--muted-foreground:oklch(0.5 0 0);--card:oklch(0.975 0 0);--card-foreground:oklch(0.18 0 0);--popover:oklch(0.99 0 0);--border:oklch(0.91 0 0);--border-strong:oklch(0.86 0 0);--primary:oklch(0.24 0 0);--primary-foreground:oklch(0.97 0 0);--accent:oklch(0.24 0 0);--accent-foreground:oklch(0.97 0 0);--success:#16a34a;--warning:#c2412d;--destructive:oklch(0.577 0.245 27.325);--code:#304c83;--link:#304c83;--chart-1:#304c83;--chart-2:#0d9488;--chart-3:#d97706;--chart-4:#9333ea;--chart-5:#e11d48;--chart-6:#65a30d}}`

// The frame grows to fit the page, so a scrollbar inside the reply would read
// as a box within the thread; it stays hidden. The page's own CSS overrides all of this.
const renderBaseCSS = `html{background:var(--background);color:var(--foreground);font-family:var(--font-sans);font-size:14px;line-height:1.5;-webkit-font-smoothing:antialiased;scrollbar-width:none}html::-webkit-scrollbar{display:none}body{margin:0}code,kbd,pre,samp{font-family:var(--font-mono)}`

var (
	// A BOM, whitespace, and comments may precede the doctype. Anything inserted
	// before the doctype would drop the page into quirks mode.
	leadingDoctype = regexp.MustCompile(`(?is)^\x{FEFF}?(?:\s|<!--.*?-->)*<!doctype[^>]*>`)
	pageViewport   = regexp.MustCompile(`(?i)<meta\s[^>]*name\s*=\s*["']?viewport`)
)

// injectRenderBootstrap places the theme style and bootstrap script ahead of
// the page's own markup. Right after the doctype the HTML parser opens the
// implied <head> for them; the page's own <html> start tag then only merges
// its attributes and its <head> start tag is ignored, so the page's styles and
// scripts still come after the bootstrap. This needs neither an HTML tokenizer
// (golang.org/x/net is not a dependency) nor T3's backreference regex, which
// RE2 cannot express.
func injectRenderBootstrap(page string) string {
	var b strings.Builder
	if !pageViewport.MatchString(page) {
		b.WriteString(`<meta name="viewport" content="width=device-width, initial-scale=1">`)
	}
	b.WriteString(`<style id="ao-theme">` + renderDefaultThemeCSS + `</style>`)
	b.WriteString(`<style>` + renderBaseCSS + `</style>`)
	b.WriteString(`<script>` + renderBootstrapJS + `</script>`)
	if loc := leadingDoctype.FindStringIndex(page); loc != nil {
		return page[:loc[1]] + b.String() + page[loc[1]:]
	}
	return "<!doctype html>" + b.String() + page
}
```

- [ ] **Step 5: Run the tests**

Run: `cd backend && go test ./internal/service/chat/ -run 'RenderBootstrap|InjectRender' -v`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add backend/internal/service/chat/render_bootstrap.js backend/internal/service/chat/render_html.go backend/internal/service/chat/render_html_test.go
git commit -m "feat(chat): theme and size bootstrap for agent HTML renders"
```

---

### Task 3: Publish a render onto the running turn

**Files:**
- Create: `backend/internal/service/chat/render.go`
- Modify: `backend/internal/service/chat/service.go`. Add `Renders RenderFiles` to `Options` (line 98), add `renders RenderFiles` to `Service` (line 37), and wire `renders: opts.Renders` in `New` (line 126).
- Modify: `backend/internal/daemon/daemon.go:404`. Add `Renders: attachmentstore.New(cfg.DataDir),` to `chatsvc.Options`.
- Modify: `backend/internal/service/chat/controller_test.go`. Add `renders *attachmentstore.Store` and `rendersDir string` to `harness` (line 3101). In `newHarnessWithConversationAndStoreForHarness`, set `h.rendersDir = t.TempDir()`, `h.renders = attachmentstore.New(h.rendersDir)`, and pass `Renders: h.renders` into `chatsvc.New`.
- Test: `backend/internal/service/chat/render_test.go` (`package chat_test`)

**Interfaces:**
- Consumes: `(*attachmentstore.Store).PutRender/RemoveRender` (Task 1); `injectRenderBootstrap` (Task 2); existing `Controller.awaitAcknowledgedTurn`, `Controller.sendMu`, `Store.UpsertActivity`, `ErrNoActiveTurn`, `ErrNotChatMode`.
- Produces:
  - `type RenderFiles interface { PutRender(ctx context.Context, id domain.SessionID, renderID string, data []byte) error; RemoveRender(ctx context.Context, id domain.SessionID, renderID string) error }`. This keeps the chat service free of storage-package imports, like its other dependencies.
  - `type RenderInput struct { HTML, Title string; Height int }`
  - `type RenderResult struct { RenderID, ActivityID, Path string }`
  - `var ErrRenderInvalid = errors.New("invalid render")`
  - `func (s *Service) PublishRender(ctx context.Context, id domain.SessionID, in RenderInput) (RenderResult, error)`
  - Activity detail JSON: `{"event":"render","render":{"id":"<renderID>","title":"…","height":N,"path":"/api/v1/sessions/<id>/renders/<renderID>"}}`, kind `system`, status `completed`, summary = title, providerItemId `render:<renderID>`.

- [ ] **Step 1: Write the failing test**

```go
package chat_test

import (
	"context"
	"encoding/json"
	"errors"
	"io"
	"os"
	"path/filepath"
	"strings"
	"testing"

	"github.com/aoagents/agent-orchestrator/backend/internal/domain"
	chatsvc "github.com/aoagents/agent-orchestrator/backend/internal/service/chat"
	"github.com/aoagents/agent-orchestrator/backend/internal/storage/sqlite/store"
)

type renderDetail struct {
	Event  string `json:"event"`
	Render struct {
		ID     string `json:"id"`
		Title  string `json:"title"`
		Height int    `json:"height"`
		Path   string `json:"path"`
	} `json:"render"`
}

func renderRows(s store.ConversationSnapshot) []domain.ConversationActivity {
	var rows []domain.ConversationActivity
	for _, a := range s.Activities {
		var d renderDetail
		if a.Kind == domain.ActivityKindSystem && json.Unmarshal(a.Detail, &d) == nil && d.Event == "render" {
			rows = append(rows, a)
		}
	}
	return rows
}

func TestPublishRenderLandsOnTheRunningTurn(t *testing.T) {
	h, _ := steerHarness(t)
	ctx := context.Background()

	result, err := h.svc.PublishRender(ctx, testSession, chatsvc.RenderInput{
		HTML: "<!doctype html><p>chart</p>", Title: "  Turns by day  ", Height: 5000,
	})
	if err != nil {
		t.Fatalf("PublishRender: %v", err)
	}
	wantPath := "/api/v1/sessions/" + string(testSession) + "/renders/" + result.RenderID
	if result.Path != wantPath {
		t.Errorf("path = %q, want %q", result.Path, wantPath)
	}

	file, _, err := h.renders.OpenRender(ctx, testSession, result.RenderID)
	if err != nil {
		t.Fatalf("stored page: %v", err)
	}
	page, _ := io.ReadAll(file)
	_ = file.Close()
	if !strings.Contains(string(page), `<style id="ao-theme">`) || !strings.HasSuffix(string(page), "<p>chart</p>") {
		t.Errorf("stored page lacks the bootstrap or the agent's body: %.200s", page)
	}

	snapshot := h.awaitSnapshot(t, func(s store.ConversationSnapshot) bool { return len(renderRows(s)) == 1 })
	row := renderRows(snapshot)[0]
	var d renderDetail
	_ = json.Unmarshal(row.Detail, &d)
	if d.Render.ID != result.RenderID || d.Render.Title != "Turns by day" || d.Render.Height != 2000 || d.Render.Path != wantPath {
		t.Errorf("detail = %+v", d)
	}
	if row.Summary != "Turns by day" || row.Status != domain.ActivityStatusCompleted {
		t.Errorf("row summary=%q status=%q", row.Summary, row.Status)
	}
	var running string
	for _, turn := range snapshot.Turns {
		if turn.ProviderTurnID == "provider-turn-1" {
			running = turn.ID
		}
	}
	if running == "" || row.TurnID != running {
		t.Errorf("render on turn %q, want the running turn %q", row.TurnID, running)
	}
}

func TestPublishRenderWithoutARunningTurnLeavesNoFile(t *testing.T) {
	h := newHarnessForHarness(t, domain.HarnessCodex)

	_, err := h.svc.PublishRender(context.Background(), testSession, chatsvc.RenderInput{
		HTML: "<p>x</p>", Title: "x", Height: 200,
	})
	if !errors.Is(err, chatsvc.ErrNoActiveTurn) {
		t.Fatalf("err = %v, want ErrNoActiveTurn", err)
	}
	entries, _ := os.ReadDir(filepath.Join(h.rendersDir, "attachments", string(testSession)))
	if len(entries) != 0 {
		t.Fatalf("orphan render files: %v", entries)
	}
}

func TestPublishRenderRejectsBadPagesBeforeStoring(t *testing.T) {
	h, _ := steerHarness(t)
	for name, in := range map[string]chatsvc.RenderInput{
		"empty html":  {HTML: "  ", Title: "x"},
		"no title":    {HTML: "<p>x</p>", Title: "   "},
		"over 1 MiB":  {HTML: strings.Repeat("a", 1<<20+1), Title: "x"},
	} {
		if _, err := h.svc.PublishRender(context.Background(), testSession, in); !errors.Is(err, chatsvc.ErrRenderInvalid) {
			t.Errorf("%s: err = %v, want ErrRenderInvalid", name, err)
		}
	}
	entries, _ := os.ReadDir(filepath.Join(h.rendersDir, "attachments", string(testSession)))
	if len(entries) != 0 {
		t.Fatalf("invalid pages were stored: %v", entries)
	}
}
```

`domain.ConversationActivity.Detail` is `[]byte` (`domain/conversation.go:668`), so `json.Unmarshal` reads it directly.

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd backend && go test ./internal/service/chat/ -run PublishRender -v`
Expected: FAIL to compile, with `h.svc.PublishRender undefined`.

- [ ] **Step 3: Write `render.go`**

```go
package chat

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"net/url"
	"strings"

	"github.com/aoagents/agent-orchestrator/backend/internal/domain"
)

const (
	maxRenderHTMLBytes  = 1 << 20
	maxRenderTitleRunes = 200
	minRenderHeight     = 80
	maxRenderHeight     = 2000
)

// ErrRenderInvalid reports a page the agent must fix before publishing. The
// wrapped message says what to change.
var ErrRenderInvalid = errors.New("invalid render")

// RenderFiles stores render pages; *attachmentstore.Store satisfies it.
type RenderFiles interface {
	PutRender(ctx context.Context, id domain.SessionID, renderID string, data []byte) error
	RemoveRender(ctx context.Context, id domain.SessionID, renderID string) error
}

// RenderInput is a self-contained HTML page an agent shows in its thread.
type RenderInput struct {
	HTML   string
	Title  string
	Height int
}

// RenderResult names the stored page and the timeline row that shows it.
type RenderResult struct {
	RenderID   string
	ActivityID string
	Path       string
}

// PublishRender stores an agent's HTML page and shows it in the turn the agent
// is running, above its final reply.
func (s *Service) PublishRender(ctx context.Context, id domain.SessionID, in RenderInput) (RenderResult, error) {
	title := strings.TrimSpace(in.Title)
	if runes := []rune(title); len(runes) > maxRenderTitleRunes {
		title = string(runes[:maxRenderTitleRunes])
	}
	switch {
	case strings.TrimSpace(in.HTML) == "":
		return RenderResult{}, fmt.Errorf("%w: the page is empty", ErrRenderInvalid)
	case len(in.HTML) > maxRenderHTMLBytes:
		return RenderResult{}, fmt.Errorf("%w: the page is %d bytes; the limit is %d", ErrRenderInvalid, len(in.HTML), maxRenderHTMLBytes)
	case title == "":
		return RenderResult{}, fmt.Errorf("%w: a title is required", ErrRenderInvalid)
	case s.renders == nil:
		return RenderResult{}, errors.New("render storage is not configured")
	}
	if _, err := s.requireChatSession(ctx, id); err != nil {
		return RenderResult{}, err
	}
	controller, err := s.Controller(id)
	if err != nil {
		return RenderResult{}, err
	}
	renderID := s.newID()
	if err := s.renders.PutRender(ctx, id, renderID, []byte(injectRenderBootstrap(in.HTML))); err != nil {
		return RenderResult{}, fmt.Errorf("store render: %w", err)
	}
	path := "/api/v1/sessions/" + url.PathEscape(string(id)) + "/renders/" + url.PathEscape(renderID)
	height := min(max(in.Height, minRenderHeight), maxRenderHeight)
	activityID, err := controller.recordRender(ctx, renderID, title, height, path)
	if err != nil {
		// Only the timeline row lets anything find the page, so an unrecorded page goes.
		if removeErr := s.renders.RemoveRender(context.WithoutCancel(ctx), id, renderID); removeErr != nil {
			s.log.Warn("render cleanup failed", "session", id, "render", renderID, "error", removeErr)
		}
		return RenderResult{}, err
	}
	return RenderResult{RenderID: renderID, ActivityID: activityID, Path: path}, nil
}

// recordRender attaches a published page to the turn in flight, as a system
// activity identified by its "render" discriminator, the way a steer is.
func (c *Controller) recordRender(ctx context.Context, renderID, title string, height int, path string) (string, error) {
	c.sendMu.Lock()
	defer c.sendMu.Unlock()
	providerTurnID, ok := c.awaitAcknowledgedTurn(ctx)
	if !ok {
		return "", ErrNoActiveTurn
	}
	detail, err := json.Marshal(map[string]any{
		"event": "render",
		"render": map[string]any{
			"id": renderID, "title": title, "height": height, "path": path,
		},
	})
	if err != nil {
		return "", fmt.Errorf("encode render detail: %w", err)
	}
	activityID := c.newID()
	if err := c.store.UpsertActivity(ctx, c.conversation.ID, providerTurnID, domain.ConversationActivity{
		ID:             activityID,
		Kind:           domain.ActivityKindSystem,
		Status:         domain.ActivityStatusCompleted,
		Summary:        title,
		Detail:         detail,
		ProviderItemID: "render:" + renderID,
	}, c.now()); err != nil {
		return "", fmt.Errorf("record render on turn %s: %w", providerTurnID, err)
	}
	return activityID, nil
}
```

- [ ] **Step 4: Wire `Options.Renders`** in `service.go` (struct field, `Options` field with comment `// Renders stores agent HTML renders. Nil refuses PublishRender.`, and `renders: opts.Renders` in `New`). Then wire `daemon.go:404`.

- [ ] **Step 5: Run the chat package with the race detector**

Run: `cd backend && go test -race ./internal/service/chat/ -run 'PublishRender|Steer' -v`
Expected: PASS. Then run `cd backend && go test ./internal/service/chat/ ./internal/daemon/` and expect PASS.

- [ ] **Step 6: Commit**

```bash
git add backend/internal/service/chat/ backend/internal/daemon/daemon.go
git commit -m "feat(chat): publish agent HTML renders onto the running turn"
```

---

### Task 4: HTTP routes: publish, and serve the page sandboxed

**Files:**
- Create: `backend/internal/httpd/controllers/conversation_render.go`
- Modify: `backend/internal/httpd/controllers/conversations.go:78-86`. Add `Renders *attachmentstore.Store` to `ConversationsController` and two routes in `Register`.
- Modify: `backend/internal/httpd/controllers/dto.go`. Add three types next to `ConversationTurnIDParam` (~line 2959).
- Modify: `backend/internal/httpd/apispec/specgen/build.go`. Add two operations after `steerSessionConversationTurn` (~line 1125), and add schema names next to line 203.
- Modify: `backend/internal/httpd/api.go:190`. Use `&controllers.ConversationsController{Svc: deps.Conversations, Renders: attachmentstore.New(cfg.DataDir)}`.
- Regenerate: `backend/internal/httpd/apispec/openapi.yaml`, `frontend/src/api/schema.ts`
- Test: `backend/internal/httpd/controllers/conversation_render_test.go` (`package controllers_test`)

**Interfaces:**
- Consumes: `chatsvc.Service.PublishRender`, `RenderInput`, `RenderResult`, `ErrRenderInvalid`, `ErrNoActiveTurn` (Task 3); `(*attachmentstore.Store).OpenRender` (Task 1).
- Produces (wire):
  - `POST /api/v1/sessions/{sessionId}/renders` takes `{"html","title","height"}`. It returns `201 {"renderId","activityId","path"}`, `400 RENDER_INVALID`, `409 RENDER_NO_ACTIVE_TURN`, or `409 SESSION_MODE_MISMATCH` (via `writeConversationError`).
  - `GET /api/v1/sessions/{sessionId}/renders/{renderId}` returns `200 text/html` with the CSP sandbox header, or `404 RENDER_NOT_FOUND`.

- [ ] **Step 1: Write the failing test**

```go
package controllers_test

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"

	"github.com/aoagents/agent-orchestrator/backend/internal/attachmentstore"
	"github.com/aoagents/agent-orchestrator/backend/internal/config"
	"github.com/aoagents/agent-orchestrator/backend/internal/domain"
	"github.com/aoagents/agent-orchestrator/backend/internal/httpd"
	chatsvc "github.com/aoagents/agent-orchestrator/backend/internal/service/chat"
)

type renderStub struct {
	*fakeConversationService
	input  chatsvc.RenderInput
	result chatsvc.RenderResult
	err    error
}

func (s *renderStub) PublishRender(_ context.Context, _ domain.SessionID, in chatsvc.RenderInput) (chatsvc.RenderResult, error) {
	s.input = in
	return s.result, s.err
}

func renderRouter(t *testing.T, dataDir string, svc *renderStub) *httptest.Server {
	t.Helper()
	log := slog.New(slog.NewTextHandler(io.Discard, nil))
	srv := httptest.NewServer(httpd.NewRouterWithControl(config.Config{DataDir: dataDir}, log, nil, httpd.APIDeps{
		Sessions:      newFakeSessionService(),
		Conversations: svc,
	}, httpd.ControlDeps{}))
	t.Cleanup(srv.Close)
	return srv
}

func TestRenderRouteServesTheStoredPageSandboxed(t *testing.T) {
	dir := t.TempDir()
	if err := attachmentstore.New(dir).PutRender(context.Background(), "proj-1", "r1", []byte("<p>chart</p>")); err != nil {
		t.Fatal(err)
	}
	srv := renderRouter(t, dir, &renderStub{fakeConversationService: &fakeConversationService{}})

	resp, err := http.Get(srv.URL + "/api/v1/sessions/proj-1/renders/r1")
	if err != nil {
		t.Fatal(err)
	}
	body, _ := io.ReadAll(resp.Body)
	_ = resp.Body.Close()
	if resp.StatusCode != http.StatusOK || string(body) != "<p>chart</p>" {
		t.Fatalf("status=%d body=%q", resp.StatusCode, body)
	}
	for header, want := range map[string]string{
		"Content-Security-Policy": "sandbox allow-scripts allow-forms",
		"Content-Type":            "text/html; charset=utf-8",
		"X-Content-Type-Options":  "nosniff",
	} {
		if got := resp.Header.Get(header); got != want {
			t.Errorf("%s = %q, want %q", header, got, want)
		}
	}

	// The page's own scripts run with an opaque origin; the daemon refuses it,
	// even for a "simple" no-cors POST that would otherwise reach a handler.
	req, _ := http.NewRequest(http.MethodPost, srv.URL+"/api/v1/sessions/proj-1/renders",
		strings.NewReader(`{"html":"<p>x</p>","title":"x"}`))
	req.Header.Set("Origin", "null")
	req.Header.Set("Content-Type", "text/plain")
	refused, err := http.DefaultClient.Do(req)
	if err != nil {
		t.Fatal(err)
	}
	_ = refused.Body.Close()
	if refused.StatusCode != http.StatusForbidden {
		t.Fatalf("Origin: null publish = %d, want 403", refused.StatusCode)
	}

	missing, err := http.Get(srv.URL + "/api/v1/sessions/proj-1/renders/..%2Fattachment-a.png")
	if err != nil {
		t.Fatal(err)
	}
	_ = missing.Body.Close()
	if missing.StatusCode != http.StatusNotFound {
		t.Fatalf("traversal-shaped id = %d, want 404", missing.StatusCode)
	}
}

func TestPublishRenderRouteMapsOutcomes(t *testing.T) {
	cases := []struct {
		name   string
		err    error
		status int
		code   string
	}{
		{"created", nil, http.StatusCreated, ""},
		{"invalid", fmt.Errorf("%w: the page is empty", chatsvc.ErrRenderInvalid), http.StatusBadRequest, "RENDER_INVALID"},
		{"no turn", chatsvc.ErrNoActiveTurn, http.StatusConflict, "RENDER_NO_ACTIVE_TURN"},
		{"tui session", chatsvc.ErrNotChatMode, http.StatusConflict, "SESSION_MODE_MISMATCH"},
	}
	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			svc := &renderStub{
				fakeConversationService: &fakeConversationService{},
				result:                  chatsvc.RenderResult{RenderID: "r1", ActivityID: "a1", Path: "/api/v1/sessions/proj-1/renders/r1"},
				err:                     tc.err,
			}
			srv := renderRouter(t, t.TempDir(), svc)
			resp, err := http.Post(srv.URL+"/api/v1/sessions/proj-1/renders", "application/json",
				strings.NewReader(`{"html":"<p>x</p>","title":"Chart","height":420}`))
			if err != nil {
				t.Fatal(err)
			}
			defer func() { _ = resp.Body.Close() }()
			var body struct {
				Code     string `json:"code"`
				RenderID string `json:"renderId"`
			}
			_ = json.NewDecoder(resp.Body).Decode(&body)
			if resp.StatusCode != tc.status || body.Code != tc.code {
				t.Fatalf("status=%d code=%q, want %d %q", resp.StatusCode, body.Code, tc.status, tc.code)
			}
			if tc.err == nil && (body.RenderID != "r1" || svc.input != (chatsvc.RenderInput{HTML: "<p>x</p>", Title: "Chart", Height: 420})) {
				t.Fatalf("renderId=%q input=%+v", body.RenderID, svc.input)
			}
		})
	}
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd backend && go test ./internal/httpd/controllers/ -run 'RenderRoute|PublishRenderRoute' -v`
Expected: FAIL. The routes 404/405 because nothing is registered, or compilation fails if `PutRender` isn't merged yet.

- [ ] **Step 3: Add the DTOs to `dto.go`**

```go
// PublishRenderRequest is a self-contained HTML page an agent shows inline in
// its chat thread.
type PublishRenderRequest struct {
	HTML   string `json:"html" description:"A complete, self-contained HTML document, at most 1 MiB."`
	Title  string `json:"title" description:"Short name for the page."`
	Height int    `json:"height,omitempty" description:"First-paint frame height in CSS pixels, clamped to 80-2000; the frame then fits the page."`
}

// PublishRenderResponse names the stored page and the timeline row showing it.
type PublishRenderResponse struct {
	RenderID   string `json:"renderId"`
	ActivityID string `json:"activityId"`
	Path       string `json:"path"`
}

// RenderIDParam names a published render.
type RenderIDParam struct {
	RenderID string `path:"renderId" description:"Render identifier returned when the page was published."`
}
```

- [ ] **Step 4: Write `conversation_render.go`**

```go
package controllers

import (
	"context"
	"errors"
	"net/http"

	"github.com/go-chi/chi/v5"

	"github.com/aoagents/agent-orchestrator/backend/internal/domain"
	"github.com/aoagents/agent-orchestrator/backend/internal/httpd/apispec"
	"github.com/aoagents/agent-orchestrator/backend/internal/httpd/envelope"
	chatsvc "github.com/aoagents/agent-orchestrator/backend/internal/service/chat"
)

const (
	publishRenderPath = "/api/v1/sessions/{sessionId}/renders"
	renderFilePath    = "/api/v1/sessions/{sessionId}/renders/{renderId}"
	// Scripts run, but the opaque origin keeps the page out of the app's
	// session and storage, and corsMiddleware refuses Origin: null, so even a
	// render opened top-level cannot call the daemon. No popups, no modals,
	// no top navigation.
	renderContentSecurityPolicy = "sandbox allow-scripts allow-forms"
)

type renderPublisher interface {
	PublishRender(context.Context, domain.SessionID, chatsvc.RenderInput) (chatsvc.RenderResult, error)
}

func (c *ConversationsController) publishRender(w http.ResponseWriter, r *http.Request) {
	svc, ok := c.Svc.(renderPublisher)
	if !ok {
		apispec.NotImplemented(w, r, "POST", publishRenderPath)
		return
	}
	var req PublishRenderRequest
	if !decodeConversationBody(w, r, &req) {
		return
	}
	result, err := svc.PublishRender(r.Context(), sessionID(r), chatsvc.RenderInput{
		HTML: req.HTML, Title: req.Title, Height: req.Height,
	})
	switch {
	case err == nil:
		envelope.WriteJSON(w, http.StatusCreated, PublishRenderResponse{
			RenderID: result.RenderID, ActivityID: result.ActivityID, Path: result.Path,
		})
	case errors.Is(err, chatsvc.ErrRenderInvalid):
		envelope.WriteAPIError(w, r, http.StatusBadRequest, "validation", "RENDER_INVALID", err.Error(), nil)
	case errors.Is(err, chatsvc.ErrNoActiveTurn):
		envelope.WriteAPIError(w, r, http.StatusConflict, "conflict", "RENDER_NO_ACTIVE_TURN",
			"a render is shown in the turn the agent is running, and no turn is in flight", nil)
	default:
		writeConversationError(w, r, err)
	}
}

func (c *ConversationsController) renderFile(w http.ResponseWriter, r *http.Request) {
	if c.Renders == nil {
		apispec.NotImplemented(w, r, "GET", renderFilePath)
		return
	}
	file, info, err := c.Renders.OpenRender(r.Context(), sessionID(r), chi.URLParam(r, "renderId"))
	if err != nil {
		envelope.WriteAPIError(w, r, http.StatusNotFound, "not_found", "RENDER_NOT_FOUND", "render not found", nil)
		return
	}
	defer func() { _ = file.Close() }()
	h := w.Header()
	h.Set("Content-Type", "text/html; charset=utf-8")
	h.Set("Content-Security-Policy", renderContentSecurityPolicy)
	h.Set("X-Content-Type-Options", "nosniff")
	h.Set("Referrer-Policy", "no-referrer")
	// A render never changes after publish; its id is its version.
	h.Set("Cache-Control", "private, max-age=31536000, immutable")
	http.ServeContent(w, r, "", info.ModTime(), file)
}
```

In `conversations.go`, add the field and the routes:

```go
type ConversationsController struct {
	Svc ConversationService
	// Renders serves agent HTML renders. Nil answers the render route 501.
	Renders *attachmentstore.Store
}
```

```go
	r.Post("/sessions/{sessionId}/renders", c.publishRender)
	r.Get("/sessions/{sessionId}/renders/{renderId}", c.renderFile)
```

- [ ] **Step 5: Register the operations in `specgen/build.go`**

```go
		{
			method: http.MethodPost, path: "/api/v1/sessions/{sessionId}/renders", id: "publishSessionRender", tag: "conversations",
			summary:    "Show an agent's self-contained HTML page inline in its chat thread",
			pathParams: []any{controllers.SessionIDParam{}},
			reqBody:    controllers.PublishRenderRequest{},
			resps: []respUnit{
				{http.StatusCreated, controllers.PublishRenderResponse{}},
				{http.StatusBadRequest, envelope.APIError{}},
				{http.StatusNotFound, envelope.APIError{}},
				{http.StatusConflict, envelope.APIError{}},
				{http.StatusInternalServerError, envelope.APIError{}},
				{http.StatusNotImplemented, envelope.APIError{}},
			},
		},
		{
			method: http.MethodGet, path: "/api/v1/sessions/{sessionId}/renders/{renderId}", id: "getSessionRender", tag: "conversations",
			summary:    "Serve a published render as a sandboxed HTML document",
			pathParams: []any{controllers.SessionIDParam{}, controllers.RenderIDParam{}},
			resps: []respUnit{
				{http.StatusOK, ""},
				{http.StatusNotFound, envelope.APIError{}},
				{http.StatusNotImplemented, envelope.APIError{}},
			},
			contentTypes: map[int]string{http.StatusOK: "text/html"},
		},
```

Schema names, next to `"ControllersSteerConversationRequest"`:

```go
	"ControllersPublishRenderRequest":  "PublishRenderRequest",
	"ControllersPublishRenderResponse": "PublishRenderResponse",
```

Wire `api.go:190` as listed under Files.

- [ ] **Step 6: Regenerate and run the HTTP suites**

Run: `npm run api && cd backend && go test ./internal/httpd/...`
Expected: PASS, including the spec-drift and route/spec parity tests. `git status` shows `openapi.yaml` and `frontend/src/api/schema.ts` modified.

- [ ] **Step 7: Commit**

```bash
git add backend/internal/httpd/ frontend/src/api/schema.ts
git commit -m "feat(api): publish and serve sandboxed agent HTML renders"
```

---

### Task 5: `ao render` CLI, the skill guide, and the prompt pointer

**Files:**
- Create: `backend/internal/cli/render.go`
- Modify: `backend/internal/cli/root.go:210`. Add `root.AddCommand(newRenderCommand(ctx))` after `newPreviewCommand`.
- Create: `backend/internal/skillassets/using-ao/commands/render.md`
- Modify: `backend/internal/skillassets/using-ao/SKILL.md`. Add a catalog row after `preview`.
- Modify: `backend/internal/session_manager/manager.go:4965-4978`. Add a "Showing pages in chat" section to `aoSkillPointer`.
- Modify: `backend/internal/session_manager/manager_test.go:5590-5605`. Raise the cap from 220 to 260 and assert the new sentence.
- Test: `backend/internal/cli/render_test.go` (`package cli`)

**Interfaces:**
- Consumes: `POST sessions/{id}/renders` (Task 4). Existing CLI helpers: `commandContext.postJSON`, `usageError`, and the test helpers `executeCLI`, `setConfigEnv`, `writeRunFileFor`, `previewCapture` (in `preview_test.go`).
- Produces: `ao render <file.html> --title <t> [--height N]`.

- [ ] **Step 1: Write the failing test**

```go
package cli

import (
	"encoding/json"
	"io"
	"net/http"
	"net/http/httptest"
	"os"
	"path/filepath"
	"strings"
	"testing"
)

func renderServer(t *testing.T, status int, respBody string) (*httptest.Server, *previewCapture) {
	t.Helper()
	capture := &previewCapture{}
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		body, err := io.ReadAll(r.Body)
		if err != nil {
			t.Fatalf("read body: %v", err)
		}
		capture.called, capture.body, capture.path, capture.method = true, string(body), r.URL.Path, r.Method
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(status)
		_, _ = io.WriteString(w, respBody)
	}))
	t.Cleanup(srv.Close)
	return srv, capture
}

func writePage(t *testing.T, body []byte) string {
	t.Helper()
	path := filepath.Join(t.TempDir(), "chart.html")
	if err := os.WriteFile(path, body, 0o600); err != nil {
		t.Fatal(err)
	}
	return path
}

func TestRenderPostsThePageToTheSession(t *testing.T) {
	t.Setenv("AO_SESSION_ID", "aa-47")
	cfg := setConfigEnv(t)
	srv, capture := renderServer(t, http.StatusCreated,
		`{"renderId":"r1","activityId":"a1","path":"/api/v1/sessions/aa-47/renders/r1"}`)
	writeRunFileFor(t, cfg, srv)
	page := writePage(t, []byte("<!doctype html><p>chart</p>"))

	out, errOut, err := executeCLI(t, Deps{ProcessAlive: func(int) bool { return true }},
		"render", page, "--title", "Turns by day", "--height", "420")
	if err != nil {
		t.Fatalf("render: %v\nstderr=%s", err, errOut)
	}
	if capture.method != http.MethodPost || capture.path != "/api/v1/sessions/aa-47/renders" {
		t.Fatalf("hit %s %s", capture.method, capture.path)
	}
	var req struct {
		HTML   string `json:"html"`
		Title  string `json:"title"`
		Height int    `json:"height"`
	}
	if err := json.Unmarshal([]byte(capture.body), &req); err != nil {
		t.Fatalf("decode body: %v", err)
	}
	if req.HTML != "<!doctype html><p>chart</p>" || req.Title != "Turns by day" || req.Height != 420 {
		t.Fatalf("request = %+v", req)
	}
	if !strings.Contains(out, "r1") {
		t.Fatalf("stdout = %q, want the render id", out)
	}
}

func TestRenderOutsideASessionDoesNotCallTheDaemon(t *testing.T) {
	t.Setenv("AO_SESSION_ID", "")
	cfg := setConfigEnv(t)
	srv, capture := renderServer(t, http.StatusCreated, `{}`)
	writeRunFileFor(t, cfg, srv)

	_, _, err := executeCLI(t, Deps{ProcessAlive: func(int) bool { return true }},
		"render", writePage(t, []byte("<p>x</p>")), "--title", "x")
	if err == nil || capture.called {
		t.Fatalf("err=%v called=%v; want a usage error and no request", err, capture.called)
	}
}

func TestRenderRefusesOversizedFilesBeforeSending(t *testing.T) {
	t.Setenv("AO_SESSION_ID", "aa-47")
	cfg := setConfigEnv(t)
	srv, capture := renderServer(t, http.StatusCreated, `{}`)
	writeRunFileFor(t, cfg, srv)

	_, _, err := executeCLI(t, Deps{ProcessAlive: func(int) bool { return true }},
		"render", writePage(t, make([]byte, 1<<20+1)), "--title", "x")
	if err == nil || capture.called {
		t.Fatalf("err=%v called=%v; want a size error and no request", err, capture.called)
	}
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd backend && go test ./internal/cli/ -run Render -v`
Expected: FAIL with `unknown command "render"`.

- [ ] **Step 3: Write `render.go`**

```go
package cli

import (
	"context"
	"errors"
	"fmt"
	"io"
	"net/url"
	"os"
	"strings"

	"github.com/spf13/cobra"
)

const maxRenderFileBytes = 1 << 20

type renderAPIRequest struct {
	HTML   string `json:"html"`
	Title  string `json:"title"`
	Height int    `json:"height"`
}

type renderAPIResponse struct {
	RenderID   string `json:"renderId"`
	ActivityID string `json:"activityId"`
	Path       string `json:"path"`
}

func newRenderCommand(ctx *commandContext) *cobra.Command {
	var title string
	var height int
	cmd := &cobra.Command{
		Use:   "render <file.html>",
		Short: "Show a self-contained HTML page inline in this chat session's thread",
		Long: "Show a chart, table, diagram, or mockup inline in the current chat session's\n" +
			"thread, above your final reply. Call it before that reply, and do not announce\n" +
			"or restate the page in it.\n\n" +
			"The file must be one self-contained HTML document (inline <style> and <script>,\n" +
			"at most 1 MiB). Remote https:// resources such as a CDN chart library load\n" +
			"as-is; local paths and relative URLs do not resolve, so embed images as data:\n" +
			"URIs. AO stores its own copy, so write the file outside the repository.\n\n" +
			"Style with the theme variables AO injects on :root, which follow light/dark\n" +
			"live: --background --foreground --muted --muted-foreground --card\n" +
			"--card-foreground --popover --border --border-strong --primary\n" +
			"--primary-foreground --accent --accent-foreground --success --warning\n" +
			"--destructive --code --link --chart-1..--chart-6 --radius --font-sans --font-mono.\n" +
			"Use a fluid width with no outer padding, card, or border; give charts fixed\n" +
			"pixel heights; never size html/body with 100vh. Chat sessions only: in a\n" +
			"terminal session use `ao preview` instead.",
		Example: `  ao render "$TMPDIR/turns-by-day.html" --title "Turns by day" --height 420`,
		Args:    cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			return ctx.publishRender(cmd.Context(), cmd.OutOrStdout(), args[0], title, height)
		},
	}
	cmd.Flags().StringVar(&title, "title", "", "short name for the page (required)")
	cmd.Flags().IntVar(&height, "height", 400, "first-paint frame height in CSS pixels, 80-2000; the frame then fits the page")
	_ = cmd.MarkFlagRequired("title")
	return cmd
}

func (c *commandContext) publishRender(ctx context.Context, out io.Writer, file, title string, height int) error {
	sessionID := strings.TrimSpace(os.Getenv("AO_SESSION_ID"))
	if sessionID == "" {
		return usageError{errors.New("ao render must run inside an AO chat session (AO_SESSION_ID is not set)")}
	}
	info, err := os.Stat(file)
	if err != nil {
		return usageError{fmt.Errorf("read %s: %w", file, err)}
	}
	if info.Size() > maxRenderFileBytes {
		return usageError{fmt.Errorf("%s is %d bytes; ao render accepts at most %d", file, info.Size(), maxRenderFileBytes)}
	}
	html, err := os.ReadFile(file)
	if err != nil {
		return usageError{fmt.Errorf("read %s: %w", file, err)}
	}
	var resp renderAPIResponse
	path := "sessions/" + url.PathEscape(sessionID) + "/renders"
	if err := c.postJSON(ctx, path, renderAPIRequest{HTML: string(html), Title: title, Height: height}, &resp); err != nil {
		return err
	}
	_, err = fmt.Fprintf(out, "Shown above your reply (render %s). Do not describe the page; add only what it does not say.\n", resp.RenderID)
	return err
}
```

- [ ] **Step 4: Write `commands/render.md`**

````markdown
# ao render

Show a self-contained HTML page inline in the current **chat** session's
thread, above your final reply. Use it when a chart, table, diagram, image
collage, or mockup says more than prose.

```bash
ao render "$TMPDIR/turns-by-day.html" --title "Turns by day" --height 420
```

## Rules

- Call it before your final reply. The reader already sees the page, so the
  reply must not announce it, say where it is, or restate it. Add only what the
  page does not say.
- One self-contained HTML file: inline `<style>` and `<script>`, at most 1 MiB.
  Remote `https://` resources (a CDN chart library, for example) load as-is.
  Local paths and relative URLs do not resolve; embed images as `data:` URIs.
- Write the file outside the repository (for example under `$TMPDIR`) so it does
  not appear in the diff. AO stores its own copy.
- `--height` is the first-paint frame height (80-2000). The frame then fits the
  page's real height.
- Chat sessions only. In a terminal session, open the file with `ao preview`.

## ao render or the Browser panel

Use `ao render` when the page is the answer: a chart, table, diagram, or
mockup that you make from data you already have. The page stays in the thread.

Use `ao preview` and `ao browser` when the page is the work: an app that runs,
a page that needs a server or a login, or a page that you must click through.
The Browser panel shows the live page, and the next preview replaces it.

If the user must still read the page after the dev server stops, use `ao render`.

## Layout

- The frame is borderless on the thread background, as wide as the reply column,
  and its left edge lines up with your text.
- Use a fluid width with no outer padding, card, border, or banner title. The
  page is part of your reply.
- Give charts fixed pixel heights. Do not size `html` or `body` with `100vh` or
  `height: 100%`: the frame grows to fit the page, and viewport heights make it
  grow again.
- Scripts run in a sandbox with no access to AO, cookies, or storage. Links open
  in the user's browser.

## Theme

AO injects its active theme as CSS custom properties on `:root`. They follow
light/dark mode live:

`--background` (identical to the thread), `--foreground`, `--muted`,
`--muted-foreground`, `--card`, `--card-foreground`, `--popover`, `--border`,
`--border-strong`, `--primary`, `--primary-foreground`, `--accent`,
`--accent-foreground`, `--success`, `--warning`, `--destructive`, `--code`,
`--link`, `--chart-1` … `--chart-6` (categorical series), `--radius`,
`--font-sans`, `--font-mono`.

The base stylesheet sets the page background, text color, and font from these,
sets `body` margin to 0, and hides the page scrollbar. Use the variables rather
than hard-coded colors so the page reads correctly in both themes.

## Checking a page

If the file is inside the workspace and the Browser panel is not showing an app
the user is working with, `ao preview <path>` then `ao browser screenshot` shows
roughly how it looks (without the theme variables).
````

`SKILL.md` row, after `preview`:

```markdown
| `render` | Show a self-contained HTML page inline in a chat thread | Answering with a chart, table, diagram, or mockup | [commands/render.md](commands/render.md) |
```

- [ ] **Step 5: Add the section to `aoSkillPointer` and raise its cap**

In `manager.go`, add `renderFile := filepath.ToSlash(filepath.Join(dir, "commands", "render.md"))` next to `previewFile`. Then append a section after the Browser panel paragraph. Change the current last line, ``"`ao browser` operates the same live page the user sees in that panel."``, to:

```go
		"`ao browser` operates the same live page the user sees in that panel.\n\n" +
		"## Showing pages in chat\n\n" +
		"In a chat session, use `ao render` when a chart, table, diagram, or mockup is clearer than text. " +
		"Read `" + renderFile + "` before you use `ao render`."
```

In `manager_test.go`, add `"use `ao render` when a chart, table, diagram, or mockup is clearer than text"` to the `want` list at line 5590. Raise the cap at line 5603:

```go
	if words := len(strings.Fields(m.aoSkillPointer())); words > 260 {
```

The new section is 30 words, so the pointer goes from 213 to about 243.

- [ ] **Step 6: Run the suites**

Run: `cd backend && go test ./internal/cli/ ./internal/skillassets/ ./internal/session_manager/`
Expected: PASS. If a CLI command-catalog golden or a docs test flags `render`, update it as that test instructs.

- [ ] **Step 7: Commit**

```bash
git add backend/internal/cli/ backend/internal/skillassets/ backend/internal/session_manager/manager.go
git commit -m "feat(cli): ao render shows an HTML page inline in a chat thread"
```

---

### Task 6: Renderer: themed sandboxed frame in the timeline

**Files:**
- Create: `frontend/src/renderer/lib/render-frame.ts`
- Create: `frontend/src/renderer/components/chat/RenderFrame.tsx`
- Modify: `frontend/src/renderer/components/chat/ChatTimelineItems.tsx:862`. Add one `else if` before `GenericActivityRow`.
- Modify: `frontend/src/renderer/types/conversation.ts:397`. Add `"render"` to `SystemEventDetail.event` and `render?: RenderRef`.
- Modify: all 8 `frontend/src/renderer/i18n/*.json`, adding `chat.render.expand`.
- Test: `frontend/src/renderer/lib/render-frame.test.ts`, `frontend/src/renderer/components/chat/RenderFrame.test.tsx`

**Interfaces:**
- Consumes: activity `detail.render = {id,title,height,path}` (Task 3); route `GET {base}{path}` (Task 4); bootstrap messages (Task 2).
- Produces:
  - `readRenderRef(detail): RenderRef | undefined`
  - `clampRenderHeight(n): number`
  - `readRenderTheme(root?): RenderTheme`
  - `renderThemeFragment(theme): string`
  - `renderThemeMessage(theme)`
  - `readRenderContentHeight(data): number | undefined`
  - `readRenderLinkRequest(data): string | undefined`
  - `<RenderFrame render={RenderRef} />`

- [ ] **Step 1: Write the failing tests**

`render-frame.test.ts`:

```ts
import { afterEach, describe, expect, it } from "vitest";
import {
	clampRenderHeight,
	readRenderContentHeight,
	readRenderLinkRequest,
	readRenderRef,
	readRenderTheme,
} from "./render-frame";

describe("render-frame helpers", () => {
	afterEach(() => {
		document.documentElement.removeAttribute("data-theme");
		document.documentElement.style.cssText = "";
	});

	it("accepts only a well-formed render reference with a render route path", () => {
		const ok = { event: "render" as const, render: { id: "r1", title: "Chart", height: 5000, path: "/api/v1/sessions/p-1/renders/r1" } };
		expect(readRenderRef(ok)).toEqual({ id: "r1", title: "Chart", height: 2000, path: "/api/v1/sessions/p-1/renders/r1" });
		expect(readRenderRef({ event: "render" as const, render: { ...ok.render, path: "https://evil.example/x" } })).toBeUndefined();
		expect(readRenderRef({ event: "steer" as const })).toBeUndefined();
		expect(clampRenderHeight(3)).toBe(80);
	});

	it("reads only the bootstrap's own protocol messages", () => {
		expect(readRenderContentHeight({ jsonrpc: "2.0", method: "ui/notifications/size-changed", params: { height: 412.4 } })).toBe(412.4);
		expect(readRenderContentHeight({ method: "ui/notifications/size-changed", params: { height: 412 } })).toBeUndefined();
		expect(readRenderLinkRequest({ jsonrpc: "2.0", id: 1, method: "ui/open-link", params: { url: "https://x.dev/a" } })).toBe("https://x.dev/a");
		expect(readRenderLinkRequest({ jsonrpc: "2.0", id: 1, method: "ui/open-link", params: { url: "javascript:alert(1)" } })).toBeUndefined();
	});

	it("maps AO tokens to the agent-facing variables and follows data-theme", () => {
		document.documentElement.style.setProperty("--color-bg-primary", "rgb(1, 2, 3)");
		document.documentElement.setAttribute("data-theme", "light");
		const theme = readRenderTheme();
		expect(theme.appearance).toBe("light");
		expect(theme.variables["--background"]).toBe("rgb(1, 2, 3)");
		expect(theme.variables["--chart-6"]).toBeDefined();
	});
});
```

`RenderFrame.test.tsx`:

```tsx
import { act, render as rtlRender, screen, waitFor } from "@testing-library/react";
import type { ReactElement } from "react";
import { afterEach, beforeEach, describe, expect, it } from "vitest";
import { setApiBaseUrl } from "../../lib/api-client";
import type { ConversationActivity } from "../../types/conversation";
import { TooltipProvider } from "../ui/tooltip";
import { ActivityRow } from "./ChatTimelineItems";

function render(ui: ReactElement) {
	return rtlRender(<TooltipProvider>{ui}</TooltipProvider>);
}

function renderActivity(height = 300): ConversationActivity {
	return {
		kind: "activity",
		id: "act-1",
		sequence: 7,
		revision: 0,
		activityKind: "system",
		status: "completed",
		summary: "Turns by day",
		createdAt: "2026-10-06T10:00:00Z",
		detail: { event: "render", render: { id: "r1", title: "Turns by day", height, path: "/api/v1/sessions/proj-1/renders/r1" } },
	};
}

function frame() {
	return screen.getByTitle("Turns by day") as HTMLIFrameElement;
}

function post(data: unknown, source: MessageEventSource | null) {
	act(() => {
		window.dispatchEvent(new MessageEvent("message", { data, source }));
	});
}

describe("render activity", () => {
	beforeEach(() => setApiBaseUrl("http://127.0.0.1:3001"));
	afterEach(() => {
		setApiBaseUrl(null);
		document.documentElement.removeAttribute("data-theme");
	});

	it("frames the page sandboxed, from the daemon, with the theme in the fragment", () => {
		render(<ActivityRow activity={renderActivity()} />);
		expect(frame().getAttribute("sandbox")).toBe("allow-scripts allow-forms");
		expect(frame().getAttribute("src")).toMatch(/^http:\/\/127\.0\.0\.1:3001\/api\/v1\/sessions\/proj-1\/renders\/r1#ao-theme=/);
		expect(frame().style.height).toBe("300px");
	});

	it("fits the page's reported height, clamped, and ignores other windows", () => {
		render(<ActivityRow activity={renderActivity()} />);
		const size = (height: number) => ({ jsonrpc: "2.0", method: "ui/notifications/size-changed", params: { height } });
		post(size(640), frame().contentWindow);
		expect(frame().style.height).toBe("640px");
		post(size(120), window);
		expect(frame().style.height).toBe("640px");
		post(size(9000), frame().contentWindow);
		expect(frame().style.height).toBe("2000px");
	});

	it("restyles on a theme flip without reloading the page", async () => {
		render(<ActivityRow activity={renderActivity()} />);
		const src = frame().getAttribute("src");
		const sent: unknown[] = [];
		Object.defineProperty(frame().contentWindow!, "postMessage", {
			configurable: true,
			value: (message: unknown) => sent.push(message),
		});
		act(() => document.documentElement.setAttribute("data-theme", "light"));
		await waitFor(() =>
			expect(sent).toContainEqual(
				expect.objectContaining({ method: "ui/notifications/host-context-changed", params: expect.objectContaining({ theme: "light" }) }),
			),
		);
		expect(frame().getAttribute("src")).toBe(src);
	});
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd frontend && npx vitest run --config vite.renderer.config.ts src/renderer/lib/render-frame.test.ts src/renderer/components/chat/RenderFrame.test.tsx`
Expected: FAIL. The module `./render-frame` is not found, and no iframe is titled "Turns by day".

- [ ] **Step 3: Extend the detail type** (`types/conversation.ts`)

```ts
/** An agent HTML page published with `ao render`, shown inline in its turn. */
export interface RenderRef {
	id: string;
	title: string;
	height: number;
	/** Daemon-relative route, `/api/v1/sessions/{id}/renders/{renderId}`. */
	path: string;
}
```

Add `| "render"` to `SystemEventDetail.event`, and add a field to `SystemEventDetail`:

```ts
	/** render */
	render?: RenderRef;
```

- [ ] **Step 4: Write `lib/render-frame.ts`**

```ts
import type { ConversationActivity, RenderRef } from "../types/conversation";

export const RENDER_MIN_HEIGHT = 80;
export const RENDER_MAX_HEIGHT = 2000;
const RENDER_PATH = /^\/api\/v1\/sessions\/[^/]+\/renders\/[^/]+$/;

export function clampRenderHeight(height: number): number {
	return Math.min(RENDER_MAX_HEIGHT, Math.max(RENDER_MIN_HEIGHT, Math.round(height)));
}

/** The page a `render` activity points at, or undefined for anything malformed. */
export function readRenderRef(detail: ConversationActivity["detail"]): RenderRef | undefined {
	const render = detail?.event === "render" ? detail.render : undefined;
	if (
		!render ||
		typeof render.id !== "string" ||
		typeof render.title !== "string" ||
		typeof render.height !== "number" ||
		!Number.isFinite(render.height) ||
		typeof render.path !== "string" ||
		!RENDER_PATH.test(render.path)
	) {
		return undefined;
	}
	return { id: render.id, title: render.title, height: clampRenderHeight(render.height), path: render.path };
}

export interface RenderTheme {
	appearance: "light" | "dark";
	variables: Record<string, string>;
}

// Agent-facing names (documented in commands/render.md) → AO's semantic tokens
// in frontend/src/styles/tokens.css. getComputedStyle resolves their var() chains.
const THEME_TOKENS: ReadonlyArray<readonly [string, string]> = [
	["--background", "--color-bg-primary"],
	["--foreground", "--color-text-primary"],
	["--muted", "--color-bg-tertiary"],
	["--muted-foreground", "--color-text-muted"],
	["--card", "--color-bg-secondary"],
	["--card-foreground", "--color-text-primary"],
	["--popover", "--color-bg-elevated"],
	["--border", "--color-border"],
	["--border-strong", "--color-border-strong"],
	["--primary", "--color-accent"],
	["--primary-foreground", "--color-accent-foreground"],
	["--accent", "--color-accent"],
	["--accent-foreground", "--color-accent-foreground"],
	["--success", "--color-success"],
	["--warning", "--color-warning"],
	["--destructive", "--color-danger"],
	["--code", "--color-text-markdown-code"],
	["--link", "--color-text-markdown-link"],
	["--chart-1", "--color-brand-logo"],
	["--font-sans", "--font-family-base"],
	["--font-mono", "--font-family-mono"],
	["--radius", "--radius-md"],
];

// ponytail: fixed categorical series for agent pages only (never AO chrome);
// AO's own --chart-* tokens are grayscale. Swap for tokens once design names some.
const CHART_SERIES = {
	dark: ["#2dd4bf", "#fbbf24", "#c084fc", "#fb7185", "#a3e635"],
	light: ["#0d9488", "#d97706", "#9333ea", "#e11d48", "#65a30d"],
} as const;

export function readRenderTheme(root: HTMLElement = document.documentElement): RenderTheme {
	const appearance = root.getAttribute("data-theme") === "light" ? "light" : "dark";
	const style = getComputedStyle(root);
	const variables: Record<string, string> = {};
	for (const [name, token] of THEME_TOKENS) {
		const value = style.getPropertyValue(token).trim();
		if (value) variables[name] = value;
	}
	CHART_SERIES[appearance].forEach((color, index) => {
		variables[`--chart-${index + 2}`] = color;
	});
	return { appearance, variables };
}

export function renderThemesEqual(left: RenderTheme, right: RenderTheme): boolean {
	return left.appearance === right.appearance && JSON.stringify(left.variables) === JSON.stringify(right.variables);
}

/** URL fragment that hands a render its theme before first paint. */
export function renderThemeFragment(theme: RenderTheme): string {
	return `#ao-theme=${encodeURIComponent(JSON.stringify(theme))}`;
}

/** The message a mounted render restyles from when the theme changes. */
export function renderThemeMessage(theme: RenderTheme) {
	return {
		jsonrpc: "2.0",
		method: "ui/notifications/host-context-changed",
		params: { theme: theme.appearance, styles: { variables: theme.variables } },
	} as const;
}

function rpc(data: unknown, method: string): Record<string, unknown> | undefined {
	if (typeof data !== "object" || data === null) return undefined;
	const message = data as Record<string, unknown>;
	if (message.jsonrpc !== "2.0" || message.method !== method) return undefined;
	return typeof message.params === "object" && message.params !== null ? (message.params as Record<string, unknown>) : undefined;
}

export function readRenderContentHeight(data: unknown): number | undefined {
	const height = rpc(data, "ui/notifications/size-changed")?.height;
	return typeof height === "number" && Number.isFinite(height) && height > 0 ? height : undefined;
}

export function readRenderLinkRequest(data: unknown): string | undefined {
	const url = rpc(data, "ui/open-link")?.url;
	return typeof url === "string" && /^https?:\/\//i.test(url) ? url : undefined;
}
```

- [ ] **Step 5: Write `components/chat/RenderFrame.tsx`**

```tsx
import { Maximize2 } from "lucide-react";
import { useEffect, useLayoutEffect, useRef, useState } from "react";
import { useTranslation } from "react-i18next";
import { getApiBaseUrl } from "../../lib/api-client";
import {
	clampRenderHeight,
	readRenderContentHeight,
	readRenderLinkRequest,
	readRenderTheme,
	renderThemeFragment,
	renderThemeMessage,
	renderThemesEqual,
	type RenderTheme,
} from "../../lib/render-frame";
import { cn } from "../../lib/utils";
import type { RenderRef } from "../../types/conversation";
import { Button } from "../ui/button";
import { Dialog, DialogContent, DialogTitle } from "../ui/dialog";
import { Tooltip, TooltipContent, TooltipTrigger } from "../ui/tooltip";

/** The app theme as handed to renders; follows data-theme flips on <html>. */
function useRenderTheme(): RenderTheme {
	const [theme, setTheme] = useState(readRenderTheme);
	useEffect(() => {
		const observer = new MutationObserver(() =>
			setTheme((current) => {
				const next = readRenderTheme();
				return renderThemesEqual(current, next) ? current : next;
			}),
		);
		observer.observe(document.documentElement, { attributes: true, attributeFilter: ["data-theme", "class", "style"] });
		return () => observer.disconnect();
	}, []);
	return theme;
}

/**
 * An agent's HTML page inline in its turn. The page runs in an opaque-origin
 * sandbox (no allow-same-origin), so it cannot reach the app's session,
 * storage, or the daemon. It reads the theme from its URL fragment before
 * first paint and restyles from posted messages after, so the src never changes.
 */
function RenderDocument({ render, fit, className }: { render: RenderRef; fit?: boolean; className?: string }) {
	const theme = useRenderTheme();
	const frameRef = useRef<HTMLIFrameElement>(null);
	const themeRef = useRef(theme);
	themeRef.current = theme;
	const [src] = useState(() => `${getApiBaseUrl()}${render.path}${renderThemeFragment(theme)}`);
	const [contentHeight, setContentHeight] = useState<number>();
	const postTheme = () => frameRef.current?.contentWindow?.postMessage(renderThemeMessage(themeRef.current), "*");
	useEffect(postTheme, [theme]);
	// Layout effect: a fast page can post its height before a passive effect runs.
	useLayoutEffect(() => {
		const onMessage = (event: MessageEvent) => {
			const frame = frameRef.current;
			if (!frame || event.source !== frame.contentWindow) return;
			const height = readRenderContentHeight(event.data);
			if (height !== undefined) {
				setContentHeight(height);
				return;
			}
			// Only while the reader is using this frame: a page can post on load.
			const url = readRenderLinkRequest(event.data);
			if (url && document.activeElement === frame && navigator.userActivation?.isActive !== false) {
				window.open(url, "_blank", "noopener,noreferrer");
			}
		};
		window.addEventListener("message", onMessage);
		return () => window.removeEventListener("message", onMessage);
	}, []);
	return (
		<iframe
			ref={frameRef}
			src={src}
			title={render.title}
			sandbox="allow-scripts allow-forms"
			loading="lazy"
			onLoad={postTheme}
			className={cn("block w-full border-0", className)}
			style={fit ? { height: clampRenderHeight(contentHeight ?? render.height) } : undefined}
		/>
	);
}

export function RenderFrame({ render }: { render: RenderRef }) {
	const { t } = useTranslation();
	const [expanded, setExpanded] = useState(false);
	return (
		<div className="group/render relative min-w-0">
			<RenderDocument render={render} fit />
			<Tooltip>
				<TooltipTrigger asChild>
					<Button
						variant="ghost"
						size="icon-sm"
						aria-label={t("chat.render.expand")}
						className="absolute end-1 top-1 opacity-0 transition-opacity group-hover/render:opacity-100 focus-visible:opacity-100"
						onClick={() => setExpanded(true)}
					>
						<Maximize2 className="size-3.5" />
					</Button>
				</TooltipTrigger>
				<TooltipContent>{t("chat.render.expand")}</TooltipContent>
			</Tooltip>
			<Dialog open={expanded} onOpenChange={setExpanded}>
				<DialogContent className="z-overlay flex h-[calc(100svh-6rem)] w-[calc(100vw-6rem)] max-w-none flex-col gap-2 p-2 pt-10">
					<DialogTitle className="sr-only">{render.title}</DialogTitle>
					{expanded ? <RenderDocument render={render} className="min-h-0 flex-1" /> : null}
				</DialogContent>
			</Dialog>
		</div>
	);
}
```

`components/ui/tooltip.tsx:51` exports `Tooltip, TooltipTrigger, TooltipContent, TooltipProvider`. `Button` takes `asChild` (`button.tsx:43`) and `size="icon-sm"` (`button.tsx:29`). The Dialog composition mirrors `ChatImage.tsx:124-133`.

- [ ] **Step 6: Branch `ActivityRow`** (`ChatTimelineItems.tsx`, before the `GenericActivityRow` fallback at line 864)

Above the `let content: ReactNode;` chain:

```tsx
	const renderRef = readRenderRef(activity.detail);
```

and in the chain, before the final `else`:

```tsx
	else if (renderRef) content = <RenderFrame render={renderRef} />;
```

Import `readRenderRef` from `../../lib/render-frame` and `RenderFrame` from `./RenderFrame`. A malformed render detail falls through to `GenericActivityRow`, matching the existing comment above `ActivityRow` that unknown rows still render.

- [ ] **Step 7: Add the i18n key to all 8 locales**, next to `chat.image.*`:

| file | value |
|---|---|
| `en.json` | `"chat.render.expand": "Expand page"` |
| `de.json` | `"chat.render.expand": "Seite vergrößern"` |
| `es.json` | `"chat.render.expand": "Ampliar página"` |
| `fr.json` | `"chat.render.expand": "Agrandir la page"` |
| `ja.json` | `"chat.render.expand": "ページを拡大"` |
| `ko.json` | `"chat.render.expand": "페이지 확대"` |
| `pt-BR.json` | `"chat.render.expand": "Ampliar página"` |
| `zh-CN.json` | `"chat.render.expand": "放大页面"` |

- [ ] **Step 8: Run the tests and the checks**

Run: `cd frontend && npx vitest run --config vite.renderer.config.ts src/renderer/lib/render-frame.test.ts src/renderer/components/chat/ src/renderer/i18n/ && npm run typecheck`
Expected: PASS, including the existing `ChatTimelineItems` tests and the i18n coverage test.

- [ ] **Step 9: Commit**

```bash
git add frontend/src/renderer/
git commit -m "feat(chat): show agent HTML renders inline, themed and sandboxed"
```

---

### Task 7: Electron: a render frame cannot navigate away from its page

A page with `allow-scripts` can still set `location` on its own frame. Without this guard, a page could swap itself for a look-alike login form inside the thread.

**Files:**
- Create: `frontend/src/main/render-frame-guard.ts`
- Modify: `frontend/src/main.ts:703`. Add a `will-frame-navigate` handler next to the existing `will-navigate`.
- Test: `frontend/src/main/render-frame-guard.test.ts`

**Interfaces:**
- Produces: `blocksRenderFrameNavigation(currentUrl: string, targetUrl: string): boolean`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, expect, it } from "vitest";
import { blocksRenderFrameNavigation } from "./render-frame-guard";

const page = "http://127.0.0.1:3001/api/v1/sessions/p-1/renders/r1";

describe("blocksRenderFrameNavigation", () => {
	it("lets a frame load its render and move within it", () => {
		expect(blocksRenderFrameNavigation("about:blank", `${page}#ao-theme=x`)).toBe(false);
		expect(blocksRenderFrameNavigation(page, `${page}#section`)).toBe(false);
	});

	it("blocks a render frame from leaving its page", () => {
		expect(blocksRenderFrameNavigation(page, "https://phish.example/login")).toBe(true);
		expect(blocksRenderFrameNavigation(page, "http://127.0.0.1:3001/api/v1/sessions/p-1/renders/r2")).toBe(true);
		expect(blocksRenderFrameNavigation(page, "not a url")).toBe(true);
	});

	it("leaves every other frame alone", () => {
		expect(blocksRenderFrameNavigation("https://docs.example/", "https://elsewhere.example/")).toBe(false);
	});
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd frontend && npx vitest run --config vite.renderer.config.ts src/main/render-frame-guard.test.ts`
Expected: FAIL because module not found. `vite.renderer.config.ts` has no `include` filter, so `npm test` also runs `src/main/*.test.ts`, as it does for `external-open.test.ts`.

- [ ] **Step 3: Write the guard and wire it**

```ts
const RENDER_PATH = /^\/api\/v1\/sessions\/[^/]+\/renders\/[^/]+$/;

/**
 * Whether a subframe showing an agent render may not navigate to targetUrl.
 * The render's scripts can set their own frame's location; a frame that is a
 * render stays on that render (hash changes only).
 */
export function blocksRenderFrameNavigation(currentUrl: string, targetUrl: string): boolean {
	let current: URL;
	try {
		current = new URL(currentUrl);
	} catch {
		return false;
	}
	if (!RENDER_PATH.test(current.pathname)) return false;
	try {
		const target = new URL(targetUrl);
		return target.origin !== current.origin || target.pathname !== current.pathname || target.search !== current.search;
	} catch {
		return true;
	}
}
```

In `main.ts`, after the `will-navigate` handler:

```ts
	shellWebContents.on("will-frame-navigate", (event) => {
		if (event.isMainFrame || !event.frame) return;
		if (blocksRenderFrameNavigation(event.frame.url, event.url)) event.preventDefault();
	});
```

- [ ] **Step 4: Run the tests and typecheck**

Run: `cd frontend && npx vitest run --config vite.renderer.config.ts src/main/render-frame-guard.test.ts && npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add frontend/src/main/render-frame-guard.ts frontend/src/main/render-frame-guard.test.ts frontend/src/main.ts
git commit -m "feat(desktop): keep agent render frames on their own page"
```

---

### Task 8: `ao render --check`: screenshot, console, and height from a hidden Electron view

The agent checks a page before it publishes it. This is AO's version of T3's `html_preview`. It uses the running desktop app instead of a downloaded Chrome:

1. The CLI posts the HTML to the daemon.
2. The daemon stores it as a temporary render `check-<id>`. Task 1 storage and the Task 2 bootstrap apply, so the check sees exactly what readers will see.
3. The daemon asks Electron, through the existing browser-runtime socket, to load that render URL in a hidden view.
4. Electron returns a PNG, the content height, and the console messages.
5. The daemon deletes the temporary file.

The check uses the app's current light/dark mode, because Electron views follow the app theme (`main.ts:2028`). It needs no running turn. It needs the desktop app, and without it the agent gets `503 RENDER_CHECK_UNAVAILABLE`.

**Files:**
- Modify: `backend/internal/service/chat/render.go`. Add `CheckRender`, the `RenderCheck` port, and the types.
- Modify: `backend/internal/service/chat/service.go`. Add a `renderCheck RenderCheck` field to `Service`. `SetRenderCheck` sets it.
- Modify: `backend/internal/daemon/daemon.go:404`. Wire `RenderCheck` to `browserBroker.Execute` (the broker is built at `daemon.go:206`).
- Modify: `backend/internal/httpd/controllers/conversation_render.go`, `conversations.go` (one route), `dto.go`, `apispec/specgen/build.go`
- Modify: `backend/internal/cli/render.go` (`--check`, `--width`, `--out`), `skillassets/using-ao/commands/render.md`
- Create: `frontend/src/main/render-check.ts`
- Modify: `frontend/src/main.ts:1327`. Route `__render-check` to `checkRender` before `browserViewHost`.
- Test: `backend/internal/service/chat/render_test.go`, `backend/internal/httpd/controllers/conversation_render_test.go`, `backend/internal/cli/render_test.go`, `frontend/src/main/render-check.test.ts`

**Interfaces:**
- Consumes: `(*attachmentstore.Store).PutRender/RemoveRender` (Task 1), `injectRenderBootstrap` (Task 2), `Service.renders`/`requireChatSession` (Task 3), `renderPublisher`/`writeConversationError`/`decodeConversationBody` (Task 4), `publishRender` command plumbing (Task 5). Existing: `(*browserruntime.Broker).Execute(ctx, domain.SessionID, string, map[string]interface{}) (browserruntime.Result, error)` (`broker.go:158`), `browserruntime.ErrUnavailable` (`broker.go:59`), `writeBrowserScreenshot(cmd, result map[string]any, target string, jsonOutput, annotate bool) error` (`cli/browser.go:990`).
- Produces (Go):
  - `type RenderCheck func(ctx context.Context, id domain.SessionID, args map[string]any) (any, error)`
  - `var ErrRenderCheckUnavailable = errors.New("render check needs the AO desktop app")`
  - `type RenderCheckInput struct { HTML string; Width int; BaseURL string }`
  - `type RenderConsoleMessage struct { Level, Text string }` (JSON `level`, `text`)
  - `type RenderCheckResult struct { PNG string; Width, Height, ContentHeight int; ConsoleMessages []RenderConsoleMessage }`
  - `func (s *Service) CheckRender(ctx context.Context, id domain.SessionID, in RenderCheckInput) (RenderCheckResult, error)`
- Produces (wire): `POST /api/v1/sessions/{sessionId}/renders/check` with `{"html","width"}`. It returns `200 {"screenshot":{"mimeType":"image/png","data","width","height"},"contentHeight","consoleMessages":[{"level","text"}]}`, `400 RENDER_INVALID`, `503 RENDER_CHECK_UNAVAILABLE`, or `409 SESSION_MODE_MISMATCH`.
- Produces (broker action): `__render-check` with args `{"url": "http://127.0.0.1:<port>/api/v1/sessions/<id>/renders/check-<id>", "width": N}`. The result is `{"data": base64 PNG, "width", "height", "contentHeight", "consoleMessages": [{"level": "debug"|"log"|"warning"|"error", "text"}]}`. Like `__destroy-session`, it is internal: it is not in the `service/browser` allowlist, so `ao browser` cannot send it.
- Produces (CLI): `ao render --check <file> [--width N] [--out file.png]`

- [ ] **Step 1: Write the failing Go service test** (append to `render_test.go`)

```go
func TestCheckRenderLoadsTheStoredPageAndDeletesIt(t *testing.T) {
	h := newHarnessForHarness(t, domain.HarnessCodex)
	var gotArgs map[string]any
	var pageDuringCheck []byte
	h.svc.SetRenderCheck(func(ctx context.Context, id domain.SessionID, args map[string]any) (any, error) {
		gotArgs = args
		renderID := strings.TrimPrefix(args["url"].(string), "http://127.0.0.1:3001/api/v1/sessions/"+string(id)+"/renders/")
		file, _, err := h.renders.OpenRender(ctx, id, renderID)
		if err != nil {
			t.Fatalf("page not stored during the check: %v", err)
		}
		pageDuringCheck, _ = io.ReadAll(file)
		_ = file.Close()
		return map[string]any{
			"data": "iVBORw0KGgo=", "width": 720.0, "height": 412.0, "contentHeight": 412.0,
			"consoleMessages": []any{map[string]any{"level": "error", "text": "Uncaught ReferenceError: d3 is not defined"}},
		}, nil
	})

	result, err := h.svc.CheckRender(context.Background(), testSession, chatsvc.RenderCheckInput{
		HTML: "<p>chart</p>", BaseURL: "http://127.0.0.1:3001",
	})
	if err != nil {
		t.Fatalf("CheckRender: %v", err)
	}
	if !strings.HasPrefix(gotArgs["url"].(string), "http://127.0.0.1:3001/api/v1/sessions/"+string(testSession)+"/renders/check-") || gotArgs["width"] != 720 {
		t.Fatalf("args = %v", gotArgs)
	}
	if !strings.Contains(string(pageDuringCheck), `<style id="ao-theme">`) {
		t.Fatal("the check did not see the bootstrapped page readers get")
	}
	if result.ContentHeight != 412 || result.PNG != "iVBORw0KGgo=" || len(result.ConsoleMessages) != 1 || result.ConsoleMessages[0].Level != "error" {
		t.Fatalf("result = %+v", result)
	}
	entries, _ := os.ReadDir(filepath.Join(h.rendersDir, "attachments", string(testSession)))
	if len(entries) != 0 {
		t.Fatalf("check left files behind: %v", entries)
	}
}

func TestCheckRenderWithoutTheDesktopAppSaysSo(t *testing.T) {
	h := newHarnessForHarness(t, domain.HarnessCodex)
	h.svc.SetRenderCheck(func(context.Context, domain.SessionID, map[string]any) (any, error) {
		return nil, chatsvc.ErrRenderCheckUnavailable
	})
	_, err := h.svc.CheckRender(context.Background(), testSession, chatsvc.RenderCheckInput{HTML: "<p>x</p>", BaseURL: "http://127.0.0.1:3001"})
	if !errors.Is(err, chatsvc.ErrRenderCheckUnavailable) {
		t.Fatalf("err = %v, want ErrRenderCheckUnavailable", err)
	}
	if _, err := h.svc.CheckRender(context.Background(), testSession, chatsvc.RenderCheckInput{HTML: "<p>x</p>", Width: 100}); !errors.Is(err, chatsvc.ErrRenderInvalid) {
		t.Fatalf("width 100: err = %v, want ErrRenderInvalid", err)
	}
}
```

`SetRenderCheck` is a post-construction setter, like the existing `SetReportCoordinator` (`service.go:66`). The harness does not have to change its `chatsvc.New` call.

- [ ] **Step 2: Run it to verify it fails**

Run: `cd backend && go test ./internal/service/chat/ -run CheckRender -v`
Expected: FAIL to compile, with `h.svc.SetRenderCheck undefined`.

- [ ] **Step 3: Implement in `render.go`**

```go
const (
	defaultRenderCheckWidth = 720
	minRenderCheckWidth     = 240
	maxRenderCheckWidth     = 1600
	renderCheckAction       = "__render-check"
)

// ErrRenderCheckUnavailable reports that no desktop app is connected to load the page.
var ErrRenderCheckUnavailable = errors.New("render check needs the AO desktop app")

// RenderCheck asks the desktop app to load a render URL in a hidden view. The
// daemon wires it to the browser-runtime broker.
type RenderCheck func(ctx context.Context, id domain.SessionID, args map[string]any) (any, error)

// RenderCheckInput is a page an agent wants to see before it publishes it.
// BaseURL is the daemon origin the desktop app can load, e.g. http://127.0.0.1:3001.
type RenderCheckInput struct {
	HTML    string
	Width   int
	BaseURL string
}

// RenderConsoleMessage is one console line the page wrote while it loaded.
type RenderConsoleMessage struct {
	Level string `json:"level"`
	Text  string `json:"text"`
}

// RenderCheckResult is what the page looked like at the requested width.
type RenderCheckResult struct {
	PNG             string                 `json:"data"`
	Width           int                    `json:"width"`
	Height          int                    `json:"height"`
	ContentHeight   int                    `json:"contentHeight"`
	ConsoleMessages []RenderConsoleMessage `json:"consoleMessages"`
}

// SetRenderCheck installs the desktop-app page loader after daemon wiring.
func (s *Service) SetRenderCheck(check RenderCheck) {
	s.renderCheck = check
}

// CheckRender shows the agent its page as readers will see it: the same
// stored, bootstrapped document, loaded by the desktop app in a hidden view.
func (s *Service) CheckRender(ctx context.Context, id domain.SessionID, in RenderCheckInput) (RenderCheckResult, error) {
	width := in.Width
	if width == 0 {
		width = defaultRenderCheckWidth
	}
	switch {
	case strings.TrimSpace(in.HTML) == "":
		return RenderCheckResult{}, fmt.Errorf("%w: the page is empty", ErrRenderInvalid)
	case len(in.HTML) > maxRenderHTMLBytes:
		return RenderCheckResult{}, fmt.Errorf("%w: the page is %d bytes; the limit is %d", ErrRenderInvalid, len(in.HTML), maxRenderHTMLBytes)
	case width < minRenderCheckWidth || width > maxRenderCheckWidth:
		return RenderCheckResult{}, fmt.Errorf("%w: width must be %d-%d", ErrRenderInvalid, minRenderCheckWidth, maxRenderCheckWidth)
	case s.renders == nil || s.renderCheck == nil:
		return RenderCheckResult{}, ErrRenderCheckUnavailable
	}
	if _, err := s.requireChatSession(ctx, id); err != nil {
		return RenderCheckResult{}, err
	}
	renderID := "check-" + s.newID()
	if err := s.renders.PutRender(ctx, id, renderID, []byte(injectRenderBootstrap(in.HTML))); err != nil {
		return RenderCheckResult{}, fmt.Errorf("store render check: %w", err)
	}
	defer func() {
		if err := s.renders.RemoveRender(context.WithoutCancel(ctx), id, renderID); err != nil {
			s.log.Warn("render check cleanup failed", "session", id, "render", renderID, "error", err)
		}
	}()
	pageURL := strings.TrimRight(in.BaseURL, "/") +
		"/api/v1/sessions/" + url.PathEscape(string(id)) + "/renders/" + url.PathEscape(renderID)
	value, err := s.renderCheck(ctx, id, map[string]any{"url": pageURL, "width": width})
	if err != nil {
		return RenderCheckResult{}, err
	}
	encoded, err := json.Marshal(value)
	if err != nil {
		return RenderCheckResult{}, fmt.Errorf("encode render check result: %w", err)
	}
	var result RenderCheckResult
	if err := json.Unmarshal(encoded, &result); err != nil || result.PNG == "" {
		return RenderCheckResult{}, fmt.Errorf("desktop app returned an unreadable render check: %v", err)
	}
	return result, nil
}
```

The JSON round trip turns the broker's `float64` numbers into `int` fields. Go encodes `720.0` as `720`.

Add `renderCheck RenderCheck` to `Service` (`service.go:37`). In `daemon.go`, right after `chatSvc := chatsvc.New(...)` (line 404):

```go
	chatSvc.SetRenderCheck(func(ctx context.Context, id domain.SessionID, args map[string]any) (any, error) {
		result, err := browserBroker.Execute(ctx, id, "__render-check", args)
		if errors.Is(err, browserruntime.ErrUnavailable) {
			return nil, chatsvc.ErrRenderCheckUnavailable
		}
		return result.Value, err
	})
```

- [ ] **Step 4: Run the Go service tests**

Run: `cd backend && go test -race ./internal/service/chat/ -run 'CheckRender|PublishRender' -v`
Expected: PASS.

- [ ] **Step 5: Write the failing HTTP test** (append to `conversation_render_test.go`)

```go
type renderCheckStub struct {
	*renderStub
	checkInput chatsvc.RenderCheckInput
	checkErr   error
}

func (s *renderCheckStub) CheckRender(_ context.Context, _ domain.SessionID, in chatsvc.RenderCheckInput) (chatsvc.RenderCheckResult, error) {
	s.checkInput = in
	return chatsvc.RenderCheckResult{PNG: "iVBORw0KGgo=", Width: 720, Height: 412, ContentHeight: 412,
		ConsoleMessages: []chatsvc.RenderConsoleMessage{{Level: "error", Text: "boom"}}}, s.checkErr
}

func TestRenderCheckRouteReturnsTheScreenshotAndNamesTheOrigin(t *testing.T) {
	for _, tc := range []struct {
		name   string
		err    error
		status int
		code   string
	}{
		{"ok", nil, http.StatusOK, ""},
		{"no desktop app", chatsvc.ErrRenderCheckUnavailable, http.StatusServiceUnavailable, "RENDER_CHECK_UNAVAILABLE"},
	} {
		t.Run(tc.name, func(t *testing.T) {
			svc := &renderCheckStub{renderStub: &renderStub{fakeConversationService: &fakeConversationService{}}, checkErr: tc.err}
			log := slog.New(slog.NewTextHandler(io.Discard, nil))
			srv := httptest.NewServer(httpd.NewRouterWithControl(config.Config{DataDir: t.TempDir()}, log, nil, httpd.APIDeps{
				Sessions: newFakeSessionService(), Conversations: svc,
			}, httpd.ControlDeps{}))
			t.Cleanup(srv.Close)

			resp, err := http.Post(srv.URL+"/api/v1/sessions/proj-1/renders/check", "application/json",
				strings.NewReader(`{"html":"<p>x</p>","width":390}`))
			if err != nil {
				t.Fatal(err)
			}
			defer func() { _ = resp.Body.Close() }()
			var body struct {
				Code       string `json:"code"`
				Screenshot struct {
					MimeType string `json:"mimeType"`
					Data     string `json:"data"`
				} `json:"screenshot"`
				ContentHeight int `json:"contentHeight"`
			}
			_ = json.NewDecoder(resp.Body).Decode(&body)
			if resp.StatusCode != tc.status || body.Code != tc.code {
				t.Fatalf("status=%d code=%q, want %d %q", resp.StatusCode, body.Code, tc.status, tc.code)
			}
			if tc.err != nil {
				return
			}
			if body.Screenshot.MimeType != "image/png" || body.Screenshot.Data != "iVBORw0KGgo=" || body.ContentHeight != 412 {
				t.Fatalf("body = %+v", body)
			}
			if svc.checkInput.Width != 390 || svc.checkInput.BaseURL != srv.URL {
				t.Fatalf("input = %+v", svc.checkInput)
			}
		})
	}
}
```

- [ ] **Step 6: Implement the route**

DTOs in `dto.go`:

```go
// RenderCheckRequest is a page an agent wants to see before it publishes it.
type RenderCheckRequest struct {
	HTML  string `json:"html" description:"A complete, self-contained HTML document, at most 1 MiB."`
	Width int    `json:"width,omitempty" description:"Viewport width in CSS pixels, 240-1600. Defaults to 720."`
}

// RenderCheckScreenshot is the page as the desktop app drew it.
type RenderCheckScreenshot struct {
	MimeType string `json:"mimeType"`
	Data     string `json:"data" description:"Base64 PNG."`
	Width    int    `json:"width"`
	Height   int    `json:"height"`
}

// RenderConsoleMessage is one console line the page wrote while it loaded.
type RenderConsoleMessage struct {
	Level string `json:"level" enum:"debug,log,warning,error"`
	Text  string `json:"text"`
}

// RenderCheckResponse reports how the page rendered.
type RenderCheckResponse struct {
	Screenshot      RenderCheckScreenshot  `json:"screenshot"`
	ContentHeight   int                    `json:"contentHeight" description:"Height the page needs at this width, in CSS pixels."`
	ConsoleMessages []RenderConsoleMessage `json:"consoleMessages"`
}
```

Handler in `conversation_render.go`:

```go
type renderChecker interface {
	CheckRender(context.Context, domain.SessionID, chatsvc.RenderCheckInput) (chatsvc.RenderCheckResult, error)
}

func (c *ConversationsController) checkRender(w http.ResponseWriter, r *http.Request) {
	svc, ok := c.Svc.(renderChecker)
	if !ok {
		apispec.NotImplemented(w, r, "POST", "/api/v1/sessions/{sessionId}/renders/check")
		return
	}
	var req RenderCheckRequest
	if !decodeConversationBody(w, r, &req) {
		return
	}
	// The desktop app loads the page from the origin the CLI reached: the
	// loopback listener (preview hosts never reach this route).
	result, err := svc.CheckRender(r.Context(), sessionID(r), chatsvc.RenderCheckInput{
		HTML: req.HTML, Width: req.Width, BaseURL: "http://" + r.Host,
	})
	switch {
	case err == nil:
		messages := make([]RenderConsoleMessage, 0, len(result.ConsoleMessages))
		for _, m := range result.ConsoleMessages {
			messages = append(messages, RenderConsoleMessage{Level: m.Level, Text: m.Text})
		}
		envelope.WriteJSON(w, http.StatusOK, RenderCheckResponse{
			Screenshot:      RenderCheckScreenshot{MimeType: "image/png", Data: result.PNG, Width: result.Width, Height: result.Height},
			ContentHeight:   result.ContentHeight,
			ConsoleMessages: messages,
		})
	case errors.Is(err, chatsvc.ErrRenderInvalid):
		envelope.WriteAPIError(w, r, http.StatusBadRequest, "validation", "RENDER_INVALID", err.Error(), nil)
	case errors.Is(err, chatsvc.ErrRenderCheckUnavailable):
		envelope.WriteAPIError(w, r, http.StatusServiceUnavailable, "unavailable", "RENDER_CHECK_UNAVAILABLE",
			"render check needs the AO desktop app; open it, or publish without a check", nil)
	default:
		writeConversationError(w, r, err)
	}
}
```

Route in `Register`, before the `{renderId}` GET:

```go
	r.Post("/sessions/{sessionId}/renders/check", c.checkRender)
```

Spec operation (next to `publishSessionRender`) and schema names:

```go
		{
			method: http.MethodPost, path: "/api/v1/sessions/{sessionId}/renders/check", id: "checkSessionRender", tag: "conversations",
			summary:    "Screenshot an agent's HTML page in the desktop app before it is published",
			pathParams: []any{controllers.SessionIDParam{}},
			reqBody:    controllers.RenderCheckRequest{},
			resps: []respUnit{
				{http.StatusOK, controllers.RenderCheckResponse{}},
				{http.StatusBadRequest, envelope.APIError{}},
				{http.StatusNotFound, envelope.APIError{}},
				{http.StatusConflict, envelope.APIError{}},
				{http.StatusServiceUnavailable, envelope.APIError{}},
				{http.StatusInternalServerError, envelope.APIError{}},
				{http.StatusNotImplemented, envelope.APIError{}},
			},
		},
```

```go
	"ControllersRenderCheckRequest":    "RenderCheckRequest",
	"ControllersRenderCheckResponse":   "RenderCheckResponse",
	"ControllersRenderCheckScreenshot": "RenderCheckScreenshot",
	"ControllersRenderConsoleMessage":  "RenderConsoleMessage",
```

Run: `npm run api && cd backend && go test ./internal/httpd/... ./internal/service/chat/ ./internal/daemon/`
Expected: PASS.

- [ ] **Step 7: Write the failing Electron test** (`frontend/src/main/render-check.test.ts`)

```ts
import { describe, expect, it, vi } from "vitest";
import { checkRender } from "./render-check";

const url = "http://127.0.0.1:3001/api/v1/sessions/p-1/renders/check-id-001";

function fakes(options: { loadError?: Error } = {}) {
	const listeners = new Map<string, (...args: unknown[]) => void>();
	const permissionRequest = vi.fn();
	const contents = {
		session: {
			setPermissionRequestHandler: (handler: (...args: unknown[]) => void) => permissionRequest.mockImplementation(handler),
			setPermissionCheckHandler: vi.fn(),
		},
		setWindowOpenHandler: vi.fn(),
		on: (event: string, listener: (...args: unknown[]) => void) => listeners.set(event, listener),
		loadURL: vi.fn(async () => {
			if (options.loadError) throw options.loadError;
			listeners.get("console-message")?.({}, 3, "Uncaught ReferenceError: d3 is not defined", 1, url);
		}),
		executeJavaScript: vi.fn(async () => 412),
		debugger: {
			attach: vi.fn(),
			sendCommand: vi.fn(async () => ({ data: "iVBORw0KGgo=" })),
			detach: vi.fn(),
		},
		close: vi.fn(),
	};
	const view = { webContents: contents, setBounds: vi.fn() };
	// A function, not an arrow: checkRender calls it with `new` (vitest 4 rejects arrow constructors).
	const WebContentsView = vi.fn(function () {
		return view;
	});
	const window = { contentView: { addChildView: vi.fn(), removeChildView: vi.fn() } };
	return { contents, view, WebContentsView, window, permissionRequest };
}

describe("checkRender", () => {
	it("loads only daemon render-check URLs", async () => {
		const f = fakes();
		await expect(checkRender(f as never, { url: "https://example.com/", width: 720 })).rejects.toThrow(/render-check URL/);
		await expect(checkRender(f as never, { url: url.replace("check-", ""), width: 720 })).rejects.toThrow(/render-check URL/);
		expect(f.WebContentsView).not.toHaveBeenCalled();
	});

	it("returns the screenshot, height, and console, in a sandboxed throwaway view", async () => {
		const f = fakes();
		const result = await checkRender(f as never, { url, width: 390 });
		expect(result).toEqual({
			data: "iVBORw0KGgo=",
			width: 390,
			height: 412,
			contentHeight: 412,
			consoleMessages: [{ level: "error", text: "Uncaught ReferenceError: d3 is not defined" }],
		});
		const prefs = f.WebContentsView.mock.calls[0][0].webPreferences;
		expect(prefs).toMatchObject({ sandbox: true, contextIsolation: true, nodeIntegration: false });
		expect(prefs.partition).toMatch(/^ao-render-check-/);
		const decide = vi.fn();
		f.permissionRequest({}, "media", decide);
		expect(decide).toHaveBeenCalledWith(false);
		expect(f.window.contentView.removeChildView).toHaveBeenCalledWith(f.view);
		expect(f.contents.close).toHaveBeenCalled();
	});

	it("removes the view when the page fails to load", async () => {
		const f = fakes({ loadError: new Error("ERR_CONNECTION_REFUSED") });
		await expect(checkRender(f as never, { url, width: 720 })).rejects.toThrow(/ERR_CONNECTION_REFUSED/);
		expect(f.window.contentView.removeChildView).toHaveBeenCalledWith(f.view);
		expect(f.contents.close).toHaveBeenCalled();
	});
});
```

- [ ] **Step 8: Run it to verify it fails**

Run: `cd frontend && npx vitest run --config vite.renderer.config.ts src/main/render-check.test.ts`
Expected: FAIL because module `./render-check` not found.

- [ ] **Step 9: Write `render-check.ts`**

```ts
import { randomUUID } from "node:crypto";
import type { BaseWindow, WebContentsView } from "electron";

export type RenderCheckMessage = { level: "debug" | "log" | "warning" | "error"; text: string };
export type RenderCheckResult = {
	data: string;
	width: number;
	height: number;
	contentHeight: number;
	consoleMessages: RenderCheckMessage[];
};

type RenderCheckDeps = {
	WebContentsView: typeof WebContentsView;
	window: Pick<BaseWindow, "contentView">;
};

// Only the daemon's own temporary render-check pages; never an arbitrary URL.
const RENDER_CHECK_URL = /^http:\/\/(?:127\.0\.0\.1|localhost):\d+\/api\/v1\/sessions\/[^/]+\/renders\/check-[A-Za-z0-9_-]+$/;
// Chromium console levels 0-3: verbose (console.debug), info (console.log), warning, error.
const LEVELS = ["debug", "log", "warning", "error"] as const;
const MAX_MESSAGES = 20;
const MAX_MESSAGE_CHARS = 500;
const MAX_CAPTURE_HEIGHT = 2_000;
const LOAD_TIMEOUT_MS = 15_000;
// ponytail: fixed settle for CDN scripts and first animation frames; wait on network idle if pages race it.
const SETTLE_MS = 300;

function renderCheckError(code: string, message: string): Error & { code: string } {
	return Object.assign(new Error(message), { code });
}

function wait(ms: number, signal?: AbortSignal): Promise<void> {
	return new Promise((resolve, reject) => {
		const timer = setTimeout(resolve, ms);
		signal?.addEventListener("abort", () => {
			clearTimeout(timer);
			reject(renderCheckError("BROWSER_COMMAND_CANCELED", "render check canceled"));
		}, { once: true });
	});
}

/**
 * Loads an agent's page in a throwaway hidden view, the way readers will see
 * it, and returns a screenshot, the content height, and console output. The
 * view never joins the user's Browser panel: in-memory partition, sandboxed,
 * no permissions, no popups, no navigation away. CDP captures it while it is
 * offscreen.
 */
export async function checkRender(
	deps: RenderCheckDeps,
	args: Record<string, unknown>,
	signal?: AbortSignal,
): Promise<RenderCheckResult> {
	const { url, width } = args;
	if (typeof url !== "string" || !RENDER_CHECK_URL.test(url)) {
		throw renderCheckError("INVALID_ARGUMENT", "render check needs a daemon render-check URL");
	}
	if (typeof width !== "number" || !Number.isInteger(width) || width < 240 || width > 1_600) {
		throw renderCheckError("INVALID_ARGUMENT", "render check width must be an integer from 240 to 1600");
	}
	const view = new deps.WebContentsView({
		webPreferences: {
			contextIsolation: true,
			nodeIntegration: false,
			sandbox: true,
			backgroundThrottling: false,
			// No "persist:" prefix: Electron keeps this partition in memory only.
			partition: `ao-render-check-${randomUUID()}`,
		},
	});
	const contents = view.webContents;
	const consoleMessages: RenderCheckMessage[] = [];
	contents.session.setPermissionRequestHandler((_contents, _permission, decide) => decide(false));
	contents.session.setPermissionCheckHandler(() => false);
	contents.setWindowOpenHandler(() => ({ action: "deny" }));
	contents.on("will-navigate", (event) => event.preventDefault());
	contents.on("console-message", (_event, level, message) => {
		if (consoleMessages.length >= MAX_MESSAGES) return;
		consoleMessages.push({ level: LEVELS[level] ?? "log", text: message.slice(0, MAX_MESSAGE_CHARS) });
	});
	view.setBounds({ x: -10_000, y: -10_000, width, height: 800 });
	deps.window.contentView.addChildView(view);
	try {
		await Promise.race([
			contents.loadURL(url),
			wait(LOAD_TIMEOUT_MS, signal).then(() => {
				throw renderCheckError("BROWSER_COMMAND_FAILED", `render check page did not load within ${LOAD_TIMEOUT_MS} ms`);
			}),
		]);
		await wait(SETTLE_MS, signal);
		const contentHeight = Number(
			await contents.executeJavaScript(
				"Math.ceil(Math.max(document.documentElement.scrollHeight, document.documentElement.getBoundingClientRect().height))",
			),
		);
		const height = Math.min(Math.max(contentHeight, 1), MAX_CAPTURE_HEIGHT);
		view.setBounds({ x: -10_000, y: -10_000, width, height });
		contents.debugger.attach("1.3");
		try {
			const shot = (await contents.debugger.sendCommand("Page.captureScreenshot", {
				format: "png",
				clip: { x: 0, y: 0, width, height, scale: 1 },
			})) as { data: string };
			return { data: shot.data, width, height, contentHeight, consoleMessages };
		} finally {
			contents.debugger.detach();
		}
	} finally {
		deps.window.contentView.removeChildView(view);
		contents.close();
	}
}
```

The load-timeout `wait` keeps a 15 s timer after a successful load. That is acceptable, because it only rejects into a settled race. If lint objects, swap it for a `setTimeout` handle that `finally` clears.

- [ ] **Step 10: Route the action in `main.ts`** (`connectBrowserRuntime(..., { execute })`, line 1327)

```ts
		execute: (command, signal) => {
			// A render check uses a throwaway hidden view, never the session's
			// Browser panel, so it does not need (or disturb) the view host.
			if (command.action === "__render-check") {
				if (!mainWindow) {
					throw Object.assign(new Error("AO window is unavailable"), { code: "BROWSER_TARGET_UNAVAILABLE" });
				}
				return checkRender({ WebContentsView, window: mainWindow }, command.args ?? {}, signal);
			}
			const host = browserViewHost;
```

Import `checkRender` from `./main/render-check`.

Run: `cd frontend && npx vitest run --config vite.renderer.config.ts src/main/render-check.test.ts && npm run typecheck`
Expected: PASS. `command.args` is optional (`browser-runtime-link.ts:15`), hence the `?? {}`.

- [ ] **Step 11: Write the failing CLI test** (append to `cli/render_test.go`)

```go
func TestRenderCheckWritesTheScreenshotAndPrintsConsole(t *testing.T) {
	t.Setenv("AO_SESSION_ID", "aa-47")
	cfg := setConfigEnv(t)
	srv, capture := renderServer(t, http.StatusOK,
		`{"screenshot":{"mimeType":"image/png","data":"iVBORw0KGgo=","width":390,"height":412},"contentHeight":412,`+
			`"consoleMessages":[{"level":"error","text":"Uncaught ReferenceError: d3 is not defined"}]}`)
	writeRunFileFor(t, cfg, srv)
	out := filepath.Join(t.TempDir(), "check.png")

	stdout, errOut, err := executeCLI(t, Deps{ProcessAlive: func(int) bool { return true }},
		"render", "--check", writePage(t, []byte("<p>chart</p>")), "--width", "390", "--out", out)
	if err != nil {
		t.Fatalf("render --check: %v\nstderr=%s", err, errOut)
	}
	if capture.path != "/api/v1/sessions/aa-47/renders/check" || !strings.Contains(capture.body, `"width":390`) {
		t.Fatalf("hit %s with %s", capture.path, capture.body)
	}
	if png, err := os.ReadFile(out); err != nil || len(png) == 0 {
		t.Fatalf("screenshot not written: %v", err)
	}
	for _, want := range []string{out, "412", "console.error: Uncaught ReferenceError: d3 is not defined"} {
		if !strings.Contains(stdout, want) {
			t.Errorf("stdout missing %q:\n%s", want, stdout)
		}
	}
}
```

Run: `cd backend && go test ./internal/cli/ -run RenderCheck -v`
Expected: FAIL, because `--check` is an unknown flag.

- [ ] **Step 12: Implement `--check` in `cli/render.go`**

Replace the `MarkFlagRequired("title")` line and the `RunE` body. Change the `--title` usage to `"short name for the page (required unless --check)"`. Add three flags and a check path:

```go
	var check bool
	var width int
	var out string
	// RunE:
		RunE: func(cmd *cobra.Command, args []string) error {
			if check {
				return ctx.checkRender(cmd, args[0], width, out)
			}
			if strings.TrimSpace(title) == "" {
				return usageError{errors.New("--title is required unless --check is set")}
			}
			return ctx.publishRender(cmd.Context(), cmd.OutOrStdout(), args[0], title, height)
		},
	// flags:
	cmd.Flags().BoolVar(&check, "check", false, "screenshot the page in the AO desktop app instead of publishing it")
	cmd.Flags().IntVar(&width, "width", 720, "with --check: viewport width in CSS pixels, 240-1600; use 390 for phones")
	cmd.Flags().StringVar(&out, "out", "", "with --check: PNG path to write (default: a new file in the temp directory)")
```

```go
type renderCheckAPIRequest struct {
	HTML  string `json:"html"`
	Width int    `json:"width"`
}

type renderCheckAPIResponse struct {
	Screenshot struct {
		Data   string `json:"data"`
		Width  int    `json:"width"`
		Height int    `json:"height"`
	} `json:"screenshot"`
	ContentHeight   int `json:"contentHeight"`
	ConsoleMessages []struct {
		Level string `json:"level"`
		Text  string `json:"text"`
	} `json:"consoleMessages"`
}

func (c *commandContext) checkRender(cmd *cobra.Command, file string, width int, out string) error {
	sessionID := strings.TrimSpace(os.Getenv("AO_SESSION_ID"))
	if sessionID == "" {
		return usageError{errors.New("ao render must run inside an AO chat session (AO_SESSION_ID is not set)")}
	}
	html, err := readRenderFile(file)
	if err != nil {
		return err
	}
	var resp renderCheckAPIResponse
	path := "sessions/" + url.PathEscape(sessionID) + "/renders/check"
	if err := c.postJSON(cmd.Context(), path, renderCheckAPIRequest{HTML: html, Width: width}, &resp); err != nil {
		return err
	}
	if out == "" {
		out = filepath.Join(os.TempDir(), fmt.Sprintf("ao-render-check-%d.png", time.Now().UnixNano()))
	}
	shot := map[string]any{"data": resp.Screenshot.Data, "width": resp.Screenshot.Width, "height": resp.Screenshot.Height}
	if err := writeBrowserScreenshot(cmd, shot, out, false, false); err != nil {
		return err
	}
	w := cmd.OutOrStdout()
	if _, err := fmt.Fprintf(w, "Content height: %d px at width %d.\n", resp.ContentHeight, width); err != nil {
		return err
	}
	for _, m := range resp.ConsoleMessages {
		if _, err := fmt.Fprintf(w, "console.%s: %s\n", m.Level, m.Text); err != nil {
			return err
		}
	}
	return nil
}
```

Extract the stat, size-check and read steps from `publishRender` (Task 5) into `readRenderFile(file string) (string, error)`, and call it from both paths. `writeBrowserScreenshot` already prints the saved path, refuses to overwrite, and writes mode 0600 (`cli/browser.go:990`).

Append this to the CLI `Long` help, after `"terminal session use `ao preview` instead."`:

```go
			"\n\nRun `ao render --check <file>` first: the AO desktop app returns a\n" +
			"screenshot, the page's content height, and its console messages.",
```

Add a "Check before you publish" section to `commands/render.md`, after "Rules":

```markdown
## Check before you publish

Run `ao render --check <file>` first. It loads the page in the AO desktop app
the way readers see it and writes a PNG. It prints the page's content height
and its console messages. Read the PNG. Fix every `console.error` line. Use the
content height as `--height`. Use `--width 390` to check a phone layout. The
check needs the desktop app. Without it, publish without a check.
```

Replace the old "Checking a page" section (the `ao preview` workaround) with this one.

- [ ] **Step 13: Run all affected suites**

Run: `cd backend && go test ./internal/cli/ ./internal/service/chat/ ./internal/httpd/... ./internal/daemon/ ./internal/skillassets/ && cd ../frontend && npx vitest run --config vite.renderer.config.ts src/main/ && npm run typecheck`
Expected: PASS.

- [ ] **Step 14: Commit**

```bash
git add backend/ frontend/src/main/ frontend/src/main.ts frontend/src/api/schema.ts
git commit -m "feat(cli): ao render --check screenshots a page in a hidden desktop view"
```

---

### Task 9: Real-app verification and the CI suites

**Files:** none. If a fix is needed, the changes belong to the task that owns the code.

- [ ] **Step 1: Run the full local CI suites** with the commands in `AGENTS.md` and `.github/workflows/`:

```bash
npm run lint                 # backend go test ./... + golangci-lint
npm run api && git diff --exit-code backend/internal/httpd/apispec/openapi.yaml frontend/src/api/schema.ts
npm run frontend:typecheck
cd frontend && npm test
```

Expected: all green. Report any job that cannot run locally by name, and verify it in CI instead.

- [ ] **Step 2: Launch the real desktop app on scratch data.** Use the `ao-desktop-dev` skill with an isolated `AO_DATA_DIR`, never `~/.ao`.

- [ ] **Step 3: Exercise the feature end to end.**
  1. Start a Codex chat session and ask: "Render a bar chart of the files in this repo by extension, inline in the thread."
  2. Expect the agent to write a file under `$TMPDIR`, run `ao render --check` and read the PNG, then run `ao render`. Expect it to reply without restating the chart. The chart should appear above the reply, borderless, aligned with the reply text, with no scrollbar.
  3. Repeat with a Claude chat session, which goes through the ACP driver.
  4. Run `ao render --check` on a page that calls an undefined function. Expect a `console.error: Uncaught ReferenceError` line, and expect the Browser panel to stay untouched, with no activity badge. Quit the desktop app, run the check again, and expect `RENDER_CHECK_UNAVAILABLE`.
  5. Toggle light/dark. Expect the chart to restyle in place with no reload flash.
  6. Hover the chart and press the expand control. The dialog shows the page full size, and Esc restores focus to the control.
  7. Click an `https://` link inside a page. It opens in the system browser, and the thread does not navigate.
  8. In the frame's devtools console, run `location = "https://example.com"`. Expect the frame to stay put (Task 7). Then run `fetch("http://127.0.0.1:<port>/api/v1/sessions")`. Expect it to fail with 403 or a CORS error.
  9. Check that the frame's `--background` matches the reply column's surface. If the thread sits on a different surface than `--color-bg-primary`, change that one mapping in `render-frame.ts` and the Go fallback.
  10. Run `ao render page.html --title x` in a TUI session. Expect a `SESSION_MODE_MISMATCH` error.
  11. Delete the session permanently. Expect `<AO_DATA_DIR>/attachments/<sessionId>/` to be gone.
- [ ] **Step 4: Show it.** Run `ao preview` on a screenshot or the running app, per CLAUDE.md, and tell the user to check the Browser tab.
- [ ] **Step 5: Open the PR** using `.agents/skills/pr-description/SKILL.md`. Include the Phase 2 list below as intentional omissions, and link the PR to this thread.

---

## Phase 2 (deferred, each its own PR)

1. **Local image inlining in the CLI.** Absolute image paths become `data:` URIs. The CLI reads them with the agent's own permissions and checks magic bytes, like T3's `isImageBytes`. It raises the page cap to match `MaxFileBytes`.
2. **Height measured at publish**, to remove the small resize after load. It reuses Task 8's hidden view.
3. **More actions in the expand dialog:** open in browser, view source, save.
4. **Mobile/LAN clients.** The route is already behind `authMiddleware` on the LAN listener; mobile needs its own WebView frame.
5. **Cloud sessions.** `CloudSessionChatSurface` builds its activities from cloud CP events and has no render route.
6. **Hosting upstream MCP Apps.** The bootstrap already speaks `ui/*`.
7. **An agent-requested scrolling frame**, i.e. T3's "a height below `contentHeight` caps the frame".
8. **Render cleanup on rollback.** Files currently survive a rolled-back turn until the session is deleted.
