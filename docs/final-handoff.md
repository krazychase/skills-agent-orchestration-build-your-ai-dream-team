# Project Pulse final handoff

## Final result

Mona's **Project Pulse** dashboard is built. It is a static page that loads its data from a JSON file and shows one card per project. Each card lists the project's name, owner, status, recent activity, and priority, and can also show a summary, progress, and due date. The layout is a responsive grid. You start it from VS Code with the **Run Project Pulse Dashboard** launch configuration.

The work ran through GitHub Copilot CLI in a Codespace, using the custom agents defined in `.github/agents/`.

## Agent team and roles

| Agent | Role in this build |
| --- | --- |
| **Orchestrator** | Split the request into phases, gave each agent an explicit file scope, ran Designer and Coder in parallel because their files did not overlap, reviewed how the pieces fit together, and wrote this handoff. |
| **Planner** | Wrote `docs/project-pulse-plan.md`. It contains the shared contract: the data schema, the CSS class names, the status and priority values, and the launch settings. Designer and Coder both built against it. |
| **Designer** | Owned `app/styles.css`: dashboard layout, card styling, status and priority badges, the responsive grid, focus and contrast states, and the loading, error, and empty states. |
| **Coder** | Owned `app/index.html`, `app/project-data.json`, and `.vscode/launch.json`, and ran the validation checks below. |

## Files delivered

| File | Owner | Purpose |
| --- | --- | --- |
| `app/index.html` | Coder | Page markup with the "Project Pulse" title, the `styles.css` link, and an inline script that fetches `project-data.json` and builds the `.project-card` elements. |
| `app/styles.css` | Designer | All visual styling: `.dashboard`, `.project-card`, badges, the responsive grid (`@media` breakpoints at 480px and 768px), reduced motion, forced colors, and print styles. |
| `app/project-data.json` | Coder | Sample data: a top-level `projects` array with 6 projects. |
| `.vscode/launch.json` | Coder | The **Run Project Pulse Dashboard** launch configuration. |

Supporting docs: `docs/agent-team.md` and `docs/project-pulse-plan.md`.

## How to run

1. In VS Code, open **Run and Debug**.
2. Choose **Run Project Pulse Dashboard** and start it.
3. It runs `python3 -m http.server 5500` from `${workspaceFolder}/app`. When the server is ready, it opens `http://localhost:5500/index.html`.

If the launch configuration doesn't work, run this in a terminal:

```bash
cd app && python3 -m http.server 5500
```

Then open `http://localhost:5500/index.html`. In a Codespace, open forwarded port 5500.

## Dashboard validation results

Checks run on 2026-11-25:

| Check | Result |
| --- | --- |
| `python3 -m json.tool app/project-data.json` | ✅ Valid JSON |
| `python3 -m json.tool .vscode/launch.json` | ✅ Valid JSON (strict, no comments) |
| `app/index.html` contains `<title>Project Pulse</title>`, `styles.css`, `project-data.json`, `project-card`, `status`, `recentActivity`, `priority` | ✅ All present |
| `app/styles.css` contains `.dashboard`, `.project-card`, `border-radius`, `box-shadow`, responsive `@media` and `grid-template-columns` | ✅ All present |
| `app/project-data.json` has a top-level `projects` array, and each project has `name`, `owner`, `status`, `recentActivity`, `priority` | ✅ 6 of 6 projects have every required field |
| `.vscode/launch.json` has name `Run Project Pulse Dashboard`, command `python3 -m http.server 5500`, cwd `${workspaceFolder}/app`, uriFormat `http://localhost:%s/index.html` | ✅ All four match exactly |
| Served `app/` with `python3 -m http.server 5500` and requested files with curl | ✅ `index.html` 200 `text/html`, `styles.css` 200 `text/css`, `project-data.json` 200 `application/json`; `/` returns the dashboard, not a directory listing |
| Stopped the server | ✅ Port 5500 free afterward; no temp files left |
| `scripts/validate-exercise.sh` | ⚠️ 110 PASS, 2 FAIL (exit 1). Neither failure is in the app (see below). |

The two `validate-exercise.sh` failures:

- **"Learner answer files are tracked in the template"**: the check expects a clean template, but this repo has the learner's committed answer files (`app/*`, `docs/*`, `.vscode/launch.json`). This is expected after completing the exercise steps.
- **"README explains Project Pulse story"**: `README.md` doesn't mention Project Pulse. That is a template content issue and doesn't affect the dashboard.

## Final handoff notes

### Known limitations and risks

- **Not checked visually.** The dashboard hasn't been tested in a real browser or started through the VS Code debugger. Validation covered JSON parsing, required strings, and HTTP responses only.
- **"Overdue" depends on today's date.** A card is marked Overdue when its due date is before today and its status isn't completed. As of 2026-11-25, Billing Migration (due 2026-11-10, blocked) shows as Overdue. Other cards will become overdue as time passes.
- **Port 5500 must be free.** If another process is using it, the launch configuration fails to start.
- **The page needs an HTTP server.** Opening `app/index.html` directly from disk (`file://`) blocks the `fetch` of `project-data.json`, so the dashboard shows its error state.
- **`scripts/validate-exercise.sh` has 2 failures outside the app** (listed above).

### Suggested next steps

1. Run **Run Project Pulse Dashboard** and check the layout at desktop and mobile widths, plus the keyboard focus states.
2. Run an accessibility check (for example, Lighthouse or axe) on the rendered page.
3. Replace the sample data in `app/project-data.json` with Mona's real projects.
4. If wanted, add a sentence about Project Pulse to `README.md` so the README check passes.
