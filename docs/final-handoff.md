# Project Pulse final handoff

Project Pulse is a static dashboard for Mona's team. It shows each project's name, owner, status, recent activity, priority, and an optional summary. It has no build step and no external dependencies. The work was committed as `2057f4c` "Build the Project Pulse dashboard" at the learner's request. No agent staged, committed, or pushed anything.

## Agent team and roles

Roles are defined in [`docs/agent-team.md`](agent-team.md).

| Agent | Role in this project |
|-------|----------------------|
| **Orchestrator** | Split the work into phases (contract, build, review, fixes, validation, report), gave each specialist its own file scope, and passed the Designer's review notes to the Coder. Wrote no files. |
| **Planner** | Researched the repo and wrote [`docs/project-pulse-plan.md`](project-pulse-plan.md). The plan covers the data and markup contracts, file ownership, dependencies, edge cases, validation expectations, and open questions. Wrote no code. |
| **Designer** | Owned `app/styles.css`. Reviewed `app/index.html` for accessibility and recommended `role="alert"` for error messages and styled `.field-missing` placeholders for missing values. Fixed a flex layout bug in the message area and styled `.summary-item--unknown`. |
| **Coder** | Built `app/project-data.json`, `app/index.html`, and `.vscode/launch.json`. Applied the Designer's review fixes and ran the validation checklist. |

## Deliverables

| File | Description |
|------|-------------|
| `app/index.html` | Page shell with one inline script. The script fetches `project-data.json`, checks the data, and renders `.project-card` elements using `textContent` only. It also handles the loading, empty, and error states. |
| `app/styles.css` | All visual design: tokens, responsive card grid, status and priority badges, priority accents, message states, `:focus-visible`, and reduced-motion support. |
| `app/project-data.json` | Six sample projects under a top-level `projects` array. Together they use every status and priority value. |
| `.vscode/launch.json` | The **Run Project Pulse Dashboard** launch configuration. It serves `app/` and opens `index.html`. |

## How to run

1. In VS Code, open **Run and Debug** (Ctrl/Cmd+Shift+D).
2. Select **Run Project Pulse Dashboard** and press Start.
3. A terminal runs `python3 -m http.server 5500` with `cwd` set to `${workspaceFolder}/app`. This serves `app/` on port 5500.
4. When the server prints its "Serving HTTP" line, `serverReadyAction` opens `http://localhost:5500/index.html`, so you see the dashboard, not a directory listing.
5. Stop the debug session to shut down the server.

## Dashboard validation results

Validation ran on 2026-11-04 from the repo root, following the plan's "Validation expectations" section. The schema check was a one-off Python snippet and was not saved to the repo.

### Static checks

| Check | Result |
|-------|--------|
| All four files exist | ✅ Pass |
| `python3 -m json.tool app/project-data.json` | ✅ Pass |
| `python3 -m json.tool .vscode/launch.json` | ✅ Pass |
| Schema: `projects` list has at least 5 items (6 found); each has non-empty `name`, `owner`, `status`, `recentActivity`, `priority`; all values are valid; every status and priority value is used | ✅ Pass |
| `app/index.html` keyphrases: Project Pulse, styles.css, project-data.json, project-card, status, recentActivity, priority | ✅ Pass (7/7) |
| `app/styles.css` keyphrases: .dashboard, .project-card, border-radius, box-shadow, @media, :focus-visible, prefers-reduced-motion | ✅ Pass (7/7) |
| `app/project-data.json` keyphrases: projects, name, owner, status, recentActivity, priority | ✅ Pass (6/6) |
| `.vscode/launch.json` keyphrases: Run Project Pulse Dashboard, index.html, `python3 -m http.server 5500`, `${workspaceFolder}/app`, `http://localhost:%s/index.html`, serverReadyAction | ✅ Pass (6/6) |
| `index.html` has `lang="en"` and the viewport meta tag | ✅ Pass |
| `index.html` has exactly one `<h1>` | ✅ Pass (1) |
| `index.html` has no `innerHTML` | ✅ Pass (0 occurrences) |
| `git diff --stat -- .vscode/tasks.json` is empty | ✅ Pass |

### Runtime checks

Port 5500 was already in use by an earlier `python3 -m http.server 5500` process (PID 26771). That process was started from a VS Code terminal and serves `app/`, so it was most likely the learner's own launch session. Validation did not stop it. The runtime checks ran against a separate server on port **5501**, started from `app/`.

| Check | Result |
|-------|--------|
| `GET /index.html` returns 200 | ✅ Pass |
| Response contains "Project Pulse" | ✅ Pass |
| Response does not contain "Directory listing for" | ✅ Pass |
| `GET /project-data.json` parses with `python3 -m json.tool` | ✅ Pass |
| `GET /styles.css` returns 200 | ✅ Pass |
| Validation server stopped and port 5501 is free | ✅ Pass |
| Port 5500 is free | ⚠️ Not applicable: the learner's server still holds it, and validation left it running on purpose |

**Summary:** every automated check passed. Validation found no defects.

### Manual checks still outstanding

No agent did these. They need a person in VS Code and a browser.

- [ ] Launching **Run Project Pulse Dashboard** opens `…/index.html` in a browser automatically.
- [ ] Visual check: the title, one card per project, distinct and labeled status and priority badges, and high-priority cards that stand out.
- [ ] Responsive check: one column with no horizontal scroll at about 375px, and multiple columns at 1280px or wider.
- [ ] Keyboard check: Tab shows visible focus styles.
- [ ] Opening `app/index.html` through `file://` shows the friendly error message.
- [ ] A temporary unknown status (for example `"paused"`) shows the neutral `unknown` badge. Revert it afterwards.
- [ ] Stopping the debug session shuts down the server on port 5500.

## Final handoff notes

### Known risks

- **Port 5500 already in use:** `http.server` exits with "Address already in use" and the browser never opens. This is the current state of the workspace, because a server from an earlier session is still running on 5500. To fix it, stop the earlier debug session or terminal (or `kill <PID>`), then launch again. The port is fixed at 5500 on purpose so the setup stays predictable.
- **Codespaces port forwarding:** `openExternally` depends on Codespaces forwarding the port automatically. If no browser tab opens, open port 5500 from the **Ports** panel and add `/index.html` to the URL.
- **Opening through `file://`:** browsers block `fetch` for local files. The page catches this and shows a friendly error that tells the learner to use **Run Project Pulse Dashboard**. It does not show a blank page.
- **`scripts/validate-exercise.sh`:** this is the template maintainers' self-check. Once learner files are committed, it fails on purpose, so don't treat its exit code as the pass gate for the dashboard.

### Open questions from the plan

1. Sort order: keep the JSON order (the current setup), or sort by priority or status?
2. Status values: are `on-track`, `at-risk`, `blocked`, and `done` enough, or should values such as `planning` or `paused` be added?
3. `recentActivity`: keep it as free text, or change it to structured data with a date so the dashboard can show relative times and sort by recency?
4. Should the optional `summary` field become required?
5. Keep the status overview strip? It is included now.
6. Launch type: keep `node-terminal`, which needs no extra extension, or switch to `debugpy`?
7. Dark mode: `color-scheme: light dark` is declared. Should there be a full `prefers-color-scheme: dark` palette pass?

### Suggested next steps

1. Stop the leftover server on port 5500, then do the manual checks above through **Run Project Pulse Dashboard**.
2. Answer the open questions, especially sort order and status values, before adding more data.
3. Consider adding a small automated check, such as a CI job that runs the schema and keyphrase checks, so later data edits are validated.
4. Replace the sample projects with Mona's team's real project data.
