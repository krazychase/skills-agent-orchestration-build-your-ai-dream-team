# Project Pulse implementation plan

## 1. Summary

Project Pulse is a small static dashboard for Mona's team. It shows one card per project, and each card includes the project's name, owner, status, recent activity, and priority. Cards can also show a short summary, progress, and due date. All project data comes from `app/project-data.json`, and the dashboard opens from the VS Code **Run Project Pulse Dashboard** launch configuration.

**Sources checked for this plan:** `.github/project-pulse-brief.md`, `.github/steps/2-step.md`, `.github/steps/3-step.md`, `.github/workflows/2-step.yml`, `.github/workflows/3-step.yml`, `.github/workflows/4-step.yml`, `.github/agents/*.agent.md`, `docs/agent-team.md`, `scripts/validate-exercise.sh`, `.vscode/tasks.json`, and `.devcontainer/devcontainer.json`.

**Current repo state:** `app/` is empty. `.vscode/` contains only `tasks.json`, which must not be modified. `.vscode/launch.json` does not exist yet.

**Requirements from the Step 3 grader (`.github/workflows/3-step.yml`).** The grader only checks that files exist, that certain words appear (case-insensitive), and that two files parse as JSON. It does not run the app.

| File | Must exist | Must contain (case-insensitive) | Must parse as JSON |
| --- | --- | --- | --- |
| `app/index.html` | ✅ | `Project Pulse`, `styles.css`, `project-data.json`, `project-card`, `status`, `recentActivity`, `priority` | — |
| `app/styles.css` | ✅ | `.dashboard`, `.project-card`, `border-radius`, `box-shadow` | — |
| `app/project-data.json` | ✅ | `projects`, `name`, `owner`, `status`, `recentActivity`, `priority` | ✅ `python3 -m json.tool` |
| `.vscode/launch.json` | ✅ | `Run Project Pulse Dashboard`, `index.html` | ✅ `python3 -m json.tool` (strict JSON, no comments, no trailing commas) |

**Rendering code stays in `app/index.html`.** The JavaScript must be an inline `<script>` in `app/index.html`. The grader looks for the words `recentActivity`, `priority`, and `project-card` in `index.html` only. A separate `app.js` would not count, and it isn't one of the assigned files.

**Step 3 sets the launch command.** The Step 3 prompt asks for the command `python3 -m http.server 5500`, served from `app/`, with a `serverReadyAction` that opens `http://localhost:%s/index.html`.

**Implementation phases**

| Phase | Name | Mode | Owner(s) |
| --- | --- | --- | --- |
| 0 | Confirm the shared contract (this document, §3) | Sequential (gate) | Orchestrator |
| 1 | Build: styles, data, markup and rendering, launch config | **Parallel** (separate files) | Designer and Coder |
| 2 | Integration review and fixes | Sequential | Orchestrator, then Designer or Coder for fixes |
| 3 | Validation | Sequential | Coder runs the checks; Orchestrator verifies |

> Out of scope for this plan: `docs/final-handoff.md`, which is Step 4. It must later mention Project Pulse, Orchestrator, Planner, Coder, Designer, all four files, **Run Project Pulse Dashboard**, the word "validation", and the word "handoff".

---

## 2. File assignments

| File | Owning agent | Purpose | Notes |
| --- | --- | --- | --- |
| `app/index.html` | **Coder** | Page structure, the "Project Pulse" title, the `styles.css` link, and the inline script that loads `project-data.json` and builds `.project-card` elements | Designer may review it for accessibility (read-only) and sends change requests through the Orchestrator |
| `app/styles.css` | **Designer** | All visual styling: `.dashboard`, `.project-card`, status and priority badges, responsive grid, focus and contrast, loading/error/empty message styles | Coder must not edit it |
| `app/project-data.json` | **Coder** | Sample data with a top-level `projects` array that follows the §3 schema | Strict JSON |
| `.vscode/launch.json` | **Coder** | The "Run Project Pulse Dashboard" launch config: serves `app/` on port 5500 and opens `index.html` | Strict JSON, no comments. Do **not** touch `.vscode/tasks.json` |

No other files should be created or modified in Step 3. Agents must not stage, commit, or push; the learner handles git.

---

## 3. Shared contract

Phase 0 locks this contract. Designer and Coder both build against it so they can work in parallel without reading each other's files.

### 3.1 Data schema: `app/project-data.json`

The file has one top-level object with a `projects` array. It is UTF-8, strict JSON, with no comments and no trailing commas.

| Field | Type | Required | Allowed values / format | Rendered as |
| --- | --- | --- | --- | --- |
| `id` | string | recommended | kebab-case, unique (e.g. `"pulse-api"`) | `data-project-id` on the card |
| `name` | string | **yes** | non-empty | `h2.project-card__name` |
| `owner` | string | **yes** | person or team name | `.project-card__owner` |
| `status` | string | **yes** | `"on-track"`, `"at-risk"`, `"blocked"`, `"completed"`, `"planning"` | `.status-badge.status-badge--{status}` with a readable label |
| `priority` | string | **yes** | `"high"`, `"medium"`, `"low"` | `.priority-badge.priority-badge--{priority}` with a readable label |
| `recentActivity` | string | **yes** | one-sentence update | `.project-card__activity` |
| `summary` | string | optional | short, friendly description for contributors | `.project-card__summary` |
| `progress` | number | optional | integer 0–100 | `<progress class="progress-bar">` plus `.progress-label` |
| `dueDate` | string | optional | ISO date `YYYY-MM-DD` | `<time class="project-card__due" datetime="…">` |
| `lastUpdated` | string | optional | ISO date `YYYY-MM-DD` | shown next to recent activity |

**Display labels** (the Coder maps machine values to these labels):

- Status: `on-track` → "On track", `at-risk` → "At risk", `blocked` → "Blocked", `completed` → "Completed", `planning` → "Planning"
- Priority: `high` → "High priority", `medium` → "Medium priority", `low` → "Low priority"

Unknown values render as "Unknown" with the modifier `--unknown`.

**Example shape** (one entry shown; sample data should have 5–6 projects):

```json
{
  "projects": [
    {
      "id": "pulse-api",
      "name": "Pulse API",
      "owner": "Mona",
      "status": "on-track",
      "priority": "high",
      "summary": "REST API that powers the Project Pulse dashboard.",
      "recentActivity": "Merged pagination support for the projects endpoint.",
      "lastUpdated": "2026-11-20",
      "progress": 72,
      "dueDate": "2026-12-15"
    }
  ]
}
```

**Sample data coverage:** use at least 5 projects, with every status value and every priority value appearing at least once. Include at least one project with no optional fields (to test fallbacks) and one with `progress` of 100 and status `completed`.

### 3.2 Markup contract: classes and ids in `app/index.html`

Classes use BEM naming. The Designer styles only these hooks, and the Coder outputs exactly these hooks.

| Hook | Element | Purpose |
| --- | --- | --- |
| `<html lang="en">` | html | Page language |
| `<title>Project Pulse</title>` | title | Exact title (graded) |
| `<link rel="stylesheet" href="styles.css">` | link | Relative path (graded) |
| `.skip-link` (href `#project-list`) | a | Keyboard skip link, visible on focus |
| `main.dashboard#dashboard` | main | Main layout wrapper (**`.dashboard`** is graded in CSS) |
| `.dashboard__header` | header | Title area |
| `h1.dashboard__title` | h1 | Text: "Project Pulse" |
| `p.dashboard__subtitle` | p | One-line description of the dashboard |
| `section.dashboard__summary#pulse-summary` (with `aria-label="Status summary"`) | section | Count of projects per status |
| `.summary-stat`, `.summary-stat--{status}` | div | One counter tile per status |
| `.summary-stat__value`, `.summary-stat__label` | span | Counter number and label |
| `p.status-message#status-message` (`role="status"`, `aria-live="polite"`) | p | Loading, error, and empty messages |
| `.status-message--loading`, `.status-message--error`, `.status-message--empty` | modifiers | State styling. The element gets the `hidden` attribute after a successful render |
| `ul.project-grid#project-list` (`aria-label="Projects"`) | ul | Responsive card grid |
| `li.project-grid__item` | li | Grid cell |
| `article.project-card` | article | **Graded hook.** Attributes: `data-status`, `data-priority`, `data-project-id` |
| `.project-card__header` | header | Groups name and badges |
| `h2.project-card__name` | h2 | Project name |
| `.project-card__badges` | div | Wraps both badges |
| `.status-badge` + `.status-badge--{on-track\|at-risk\|blocked\|completed\|planning\|unknown}` | span | Status badge; always includes text, never color alone |
| `.priority-badge` + `.priority-badge--{high\|medium\|low\|unknown}` | span | Priority badge with text |
| `.project-card__owner` | p | "Owner: …" |
| `.project-card__summary` | p | Optional summary |
| `.project-card__activity` | p/div | "Recent activity: …" |
| `.project-card__label` | span | Small labels such as "Owner", "Recent activity", "Due" |
| `.project-card__progress` | div | Wraps the progress bar and its label |
| `progress.progress-bar` (`max="100"`, labelled) | progress | Progress bar |
| `.progress-label` | span | e.g. "72% complete" |
| `.project-card__footer` | footer | Due date and last-updated date |
| `time.project-card__due` | time | Due date with a `datetime` attribute |
| `.project-card__updated` | time/span | Last updated date |
| `.is-overdue` | modifier on `.project-card` | Due date is in the past and status is not `completed` |

**Rule for the Designer:** if styling needs a hook that is not in this table, ask the Orchestrator. Don't assume the Coder will add it.

**Rule for the Coder:** output every class in this table exactly as spelled, even if a value is missing. Use fallback text instead of leaving an element out, except optional sections (summary, progress, due date), which can be omitted.

---

## 4. Designer responsibilities (`app/styles.css` only)

1. **Polished first view.** The page should clearly look like a Project Pulse dashboard, not a plain HTML page. Use a styled header, a summary strip, and a card grid on a soft background.
2. **Required selectors.** Include `.dashboard` and `.project-card` literally, and use `border-radius` and `box-shadow` (all four are graded).
3. **Project cards.** Rounded corners, a soft shadow (slightly raised on hover or focus-within), consistent padding, and a clear order: name → badges → owner → summary → activity → progress → footer.
4. **Status badges.** One pill-shaped color per `.status-badge--*` modifier. Each badge keeps its text label so status is never shown by color alone. Optionally add a colored top or left accent on the card keyed to `[data-status="…"]`.
5. **Priority treatment.** `.priority-badge--high` stands out most (for example a strong color or icon-like marker), `medium` is moderate, and `low` is quiet. Must stay readable in grayscale.
6. **Summary strip.** Style the `.summary-stat` tiles to match the status colors.
7. **Responsive layout.** Use a CSS grid, for example `repeat(auto-fill, minmax(18rem, 1fr))`: one column on phones (≤ 480px), two or more on tablet and desktop. Cap the content width (around 1200px) and center it. Long words and URLs must not overflow cards.
8. **Accessibility.**
   - Text contrast at least 4.5:1, and at least 3:1 for badge borders and UI elements.
   - Visible `:focus-visible` outlines.
   - `.skip-link` hidden until focused.
   - A `prefers-reduced-motion` rule that turns off hover transitions.
   - Base font size of at least 16px.
   - Readable line height (about 1.5).
9. **Typography.** System font stack, clear sizes for the title, card names, and body text, and tabular numbers for counts and percentages.
10. **State styles.** Give `.status-message--loading`, `--error` (clearly an error, high contrast), and `--empty` distinct looks. `[hidden]` must stay hidden.
11. **Progress bar.** Style `progress.progress-bar` in a way that works in both WebKit/Blink and Firefox (`::-webkit-progress-value`, `::-moz-progress-bar`).
12. **Overdue cards.** Give `.project-card.is-overdue` a subtle extra cue on the due date.
13. **Optional:** dark mode via `prefers-color-scheme`, keeping the same contrast targets.
14. **Report back:** design decisions, tradeoffs, which contract hooks were used, and contrast pairs checked.

---

## 5. Coder responsibilities

### 5.1 `app/project-data.json`

- Write the sample data following §3.1, with the coverage listed there.
- Validate it with `python3 -m json.tool app/project-data.json`.

### 5.2 `app/index.html`

1. **Page shell.** Use semantic HTML5: `<!DOCTYPE html>`, `lang="en"`, `meta charset`, `meta viewport`, `<title>Project Pulse</title>`, and `<link rel="stylesheet" href="styles.css">`. Then the skip link, `main.dashboard`, the header with `h1` "Project Pulse", the summary section, the status message, and the `#project-list` grid, all exactly as listed in §3.2.
2. **Inline script** at the end of `<body>` (or with `defer`). It contains a constant `DATA_URL = "project-data.json"` (relative path), and the literal field names `status`, `recentActivity`, and `priority` appear in the code.
3. **Loading flow:**
   1. Show the loading message ("Loading projects…") with `.status-message--loading`.
   2. Fetch `DATA_URL` with `{ cache: "no-store" }`.
   3. If `!response.ok`, show an error that includes the HTTP status, e.g. "Could not load project-data.json (HTTP 404)."
   4. Parse the JSON. If parsing fails, show "project-data.json is not valid JSON."
   5. If the top-level `projects` key is missing or isn't an array, show an error saying the `projects` array is missing.
   6. If `projects` is an empty array, show the empty state ("No projects to show yet.").
   7. Otherwise, render the cards and summary counts, then set `hidden` on the status message. Optionally announce "Showing N projects."
4. **The `file://` limitation.** Browsers block `fetch()` of local JSON when a page is opened from `file://`, so the dashboard must be served over HTTP.
   - Check `location.protocol === "file:"` before fetching. If true, show a clear error telling the user to run **Run Project Pulse Dashboard** or `python3 -m http.server 5500` from `app/`.
   - Also treat a network `TypeError` from `fetch` as an error with the same hint.
5. **Rendering rules:**
   - Build elements with `document.createElement` and `textContent`. Never put JSON values into `innerHTML`, which prevents HTML injection.
   - Map status and priority values to the display labels in §3.1. Unknown values become "Unknown" with the modifier `--unknown`.
   - Show missing required fields as "Not specified" (or "No recent activity" for `recentActivity`), and log `console.warn` naming the project index and field.
   - Clamp `progress` to 0–100, and skip the progress section if it isn't a number.
   - Parse ISO dates as dates (avoid timezone off-by-one) and display them in a readable format. Add `.is-overdue` when the due date is past and status is not `completed`.
   - Label the progress bar: `aria-label="{name} progress"`, or connect it to `.progress-label` with `aria-labelledby`.
   - Keep the data order as given. Optional enhancement: sort by priority (high first). If added, the sort must be deterministic.
6. **Summary counts:** fill `#pulse-summary` with one `.summary-stat` per status value, including zero counts, so the layout stays stable.
7. **No external dependencies.** No CDNs, no frameworks, no build step.

### 5.3 `.vscode/launch.json`

Strict JSON (no comments, no trailing commas) with these exact values:

| Key | Value |
| --- | --- |
| `version` | `"0.2.0"` |
| `configurations[0].name` | `"Run Project Pulse Dashboard"` |
| `configurations[0].type` | `"node-terminal"` (built-in JS debugger terminal; runs an arbitrary command and supports `serverReadyAction`) |
| `configurations[0].request` | `"launch"` |
| `configurations[0].command` | `"python3 -m http.server 5500"` |
| `configurations[0].cwd` | `"${workspaceFolder}/app"` |
| `configurations[0].serverReadyAction.action` | `"openExternally"` |
| `configurations[0].serverReadyAction.pattern` | `"Serving HTTP on .* port ([0-9]+)"` (captures the port from Python's startup line) |
| `configurations[0].serverReadyAction.uriFormat` | `"http://localhost:%s/index.html"` |

This opens `http://localhost:5500/index.html` (the dashboard), not a directory listing. Validate it with `python3 -m json.tool .vscode/launch.json`. Do not edit `.vscode/tasks.json`.

---

## 6. Dependencies

1. **§3 shared contract → everything.** The Designer's selectors and the Coder's markup and data all depend on the class names and the schema. Phase 0 must lock it before Phase 1.
2. **`app/project-data.json` → `app/index.html` rendering.** The script reads the schema's field names. Both files belong to the Coder, so write the data first, then the HTML.
3. **`app/index.html` ↔ `app/styles.css`.** These are tied only through the contract, not by reading each other's files. They can be built in parallel, and their visual result is checked together in Phase 2.
4. **`.vscode/launch.json` → runtime check.** It depends only on the folder layout (`app/index.html` existing at run time). It doesn't depend on CSS or data content.
5. **Phase 2 integration → Phase 3 validation.** Final checks need all four files.
6. **Step 4 handoff (`docs/final-handoff.md`) → Phase 3 results.** That later step reports the validation outcomes.

---

## 7. Parallel work decisions

**Runs in parallel (Phase 1):**

| Stream | Agent | Files (exclusive) | Why it is safe |
| --- | --- | --- | --- |
| A | Designer | `app/styles.css` | No overlap with Coder files; builds only against the §3.2 hooks |
| B | Coder | `app/project-data.json`, then `app/index.html`, plus `.vscode/launch.json` | No overlap with Designer files; builds against the §3.1 schema and §3.2 hooks |

**Runs sequentially:**

- **Phase 0 before Phase 1.** The contract must be fixed first, or class names will drift and the CSS won't match the markup.
- **Within the Coder stream:** data, then HTML and script, because the script depends on the field names. `launch.json` has no dependency and can be written at any point in that stream.
- **Phase 2 after Phase 1.** Integration review needs both streams done.
  - The Orchestrator checks the rendered page against the contract.
  - CSS fixes go to the Designer, and HTML, JS, data, or launch fixes go to the Coder, one agent per file.
  - If both files need changes, run the fixes in parallel only if neither change touches the contract. Otherwise update the contract first, then run them one after the other.
- **Designer accessibility review of `app/index.html`** (optional) is read-only. Any changes go back to the Coder as requests, so two agents never edit the same file.
- **Phase 3 after Phase 2.** Validation runs against the final files.

---

## 8. Validation expectations

Run these after Phase 2. The Coder runs the commands and the Orchestrator confirms the results.

### 8.1 Static checks (mirror the grader)

- [ ] `python3 -m json.tool app/project-data.json` exits 0.
- [ ] `python3 -m json.tool .vscode/launch.json` exits 0 (proves there are no comments or trailing commas).
- [ ] `app/index.html` contains (case-insensitive): `Project Pulse`, `styles.css`, `project-data.json`, `project-card`, `status`, `recentActivity`, `priority`.
- [ ] `app/styles.css` contains `.dashboard`, `.project-card`, `border-radius`, and `box-shadow`.
- [ ] `app/project-data.json` has a top-level `projects` array, and every entry has `name`, `owner`, `status`, `recentActivity`, and `priority`. A quick Python check is enough.
- [ ] `.vscode/launch.json` contains `Run Project Pulse Dashboard`, `index.html`, `${workspaceFolder}/app`, `python3 -m http.server 5500`, and `http://localhost:%s/index.html`.
- [ ] `.vscode/tasks.json` is unchanged.
- [ ] Every class and id in §3.2 that the CSS targets also appears in the HTML or script, and the reverse: no unused required hooks.

### 8.2 Runtime checks

- [ ] From `app/`, run `python3 -m http.server 5500`. Then `curl -s http://localhost:5500/index.html` returns HTML containing "Project Pulse", and `curl -sI http://localhost:5500/project-data.json` returns 200.
- [ ] In VS Code: Run and Debug → **Run Project Pulse Dashboard** opens a browser tab at `/index.html` that shows the dashboard, not a directory listing.
- [ ] Every project in the JSON renders as an `article.project-card` with name, owner, status badge, priority badge, and recent activity. Check in DevTools that `document.querySelectorAll('.project-card').length` equals `projects.length`.
- [ ] Summary counts match the data.
- [ ] No console errors on a normal load.

### 8.3 Error and edge states

Test against a temporary copy (e.g. `/tmp/pulse-test/`), not the committed files.

- [ ] Opening `index.html` via `file://` shows the "serve over HTTP" error message instead of a blank page.
- [ ] Missing `project-data.json` (404) shows an error that includes the HTTP status.
- [ ] Invalid JSON shows the parse error message.
- [ ] `{"projects": []}` shows the empty state.
- [ ] `{}` (no `projects` key) shows the missing-array error.
- [ ] Unknown status or priority values render as "Unknown" badges, and missing optional fields don't break the layout.

### 8.4 Responsive and accessibility checks

- [ ] At 375px, 768px, and 1280px wide, the grid changes between 1 and 3+ columns, with no horizontal scroll and no overflowing text.
- [ ] Keyboard: the skip link appears on first Tab, and focus outlines are visible.
- [ ] Semantics: one `h1`, an `h2` per card, a `main` landmark, a list for the grid, and `role="status"` plus `aria-live` on the message.
- [ ] Text contrast is at least 4.5:1 (spot-check badges and muted text with DevTools or Lighthouse). Badges remain clear in grayscale.
- [ ] Optional: Lighthouse accessibility score of 90 or higher.

---

## 9. Edge cases and open questions

### Edge cases to handle

- **The page is opened via `file://`.** Detect this and explain the fix: use the launch config or the HTTP server.
- **Port 5500 is already in use** (for example by Live Server or a previous run). `http.server` fails with "Address already in use". The learner should stop the other process or the earlier debug session. The `serverReadyAction` pattern never matches in that case, so no browser opens; this is expected and should be documented.
- **Codespaces port forwarding.** `openExternally` on `localhost:5500` should be forwarded automatically. If the browser doesn't open, use the Ports panel and open port 5500 at `/index.html`.
- **Python output buffering.** If the "Serving HTTP" line doesn't appear in time, `serverReadyAction` never fires. `node-terminal` runs in a terminal, so output is line-buffered and this should not happen. If it does, the fallback is `python3 -u -m http.server 5500`, but that changes the command Step 3 asked for, so confirm with the learner first.
- **Browser caching** of an old `project-data.json`. Handled by `cache: "no-store"`.
- **Data problems.** Status or priority in unexpected casing (e.g. "At Risk"): normalize by trimming, lowercasing, and replacing spaces with hyphens before mapping. Also handle:
  - Duplicate `id`s
  - `progress` outside 0–100, or given as a string
  - An invalid `dueDate` string: show it as "Not specified"
  - Very long names or activity text
  - HTML-like text in data values: `textContent` keeps it safe
- **Timezones in date display.** Avoid `new Date("YYYY-MM-DD")`, which is read as UTC and can show the previous day in western timezones. Split the parts manually or use local date construction.
- **CSS limits.** Browsers without `:has()` or container queries: don't depend on them for core layout. Native `progress` styling differs between browsers; see Designer item 11.

### Open questions

1. **Status values:** is the proposed set (`on-track`, `at-risk`, `blocked`, `completed`, `planning`) acceptable, or does Mona's team use different labels? The grader only checks that the key `status` exists, so either works. Answer: Ok as-is
2. **Priority versus risk:** the brief says "priority or risk level". This plan uses `priority` (`high`/`medium`/`low`) only. Should we add a separate `risk` field? Answer: Ok as-is
3. **Single `recentActivity` string versus a list** of dated events: the plan uses a single string plus an optional `lastUpdated` for simplicity. Should it be an array? Answer: Ok as-is
4. **Card sort order:** keep the data order (the current default) or sort by priority or due date? Answer: Ok as-is
5. **Dark mode:** include it now (optional Designer item 13) or defer? Answer: defer
6. **Launch `type`:** `node-terminal` is chosen because it is built into VS Code and supports `command` plus `serverReadyAction`. If the learner's environment lacks the built-in JS debugger, a fallback such as a `tasks.json`-backed config would require editing `.vscode/tasks.json`, which is outside the assigned scope and needs approval. Answer: Ok as-is
7. **Designer review of `app/index.html` markup:** should the Orchestrator schedule the optional read-only accessibility pass in Phase 2, or skip it to save time? Answer: Skip it