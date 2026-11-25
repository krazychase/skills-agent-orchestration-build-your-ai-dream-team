# Agent team

I am using GitHub Copilot CLI in a Codespace to orchestrate the work of building Mona's Project Pulse dashboard. The custom agents below are defined in `.github/agents/`.

| Agent | Target model | Definition | Responsibility |
| --- | --- | --- | --- |
| Orchestrator | Claude Opus 5.5 (copilot) | `.github/agents/orchestrator.agent.md` | Breaks the request into tasks, delegates to Planner, Coder, and Designer with explicit file scopes, runs work in parallel only when scopes don't overlap, verifies the integrated result, and reports. Does not implement or run git operations. |
| Planner | Claude Opus 5.5 (copilot) | `.github/agents/planner.agent.md` | Researches the repo and docs, then returns an ordered plan with file assignments, dependencies, parallel/sequential work, edge cases, validation expectations, and open questions. Writes no code. |
| Coder | GPT-5.5 (copilot) | `.github/agents/coder.agent.md` | Implements the dashboard logic and support files (such as `.vscode/launch.json` with `cwd` `${workspaceFolder}/app`, opening `index.html`) within its assigned scope, and validates before reporting. |
| Designer | Gemini 3.7 Flash (copilot) | `.github/agents/designer.agent.md` | Handles UI/UX, accessibility, and styling: project cards, status badges, priority treatment, responsive layout, and CSS hooks like `.dashboard` and `.project-card`. |

None of the agents stage, commit, or push; I control all git operations through Copilot CLI prompts.
