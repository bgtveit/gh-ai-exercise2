# Project Pulse implementation plan

## Summary

Project Pulse is a small static dashboard for Mona's team. It shows contributors which projects are active, who owns each one, its current status, recent activity, priority or risk level, and a short summary. It uses a polished card layout with status badges and readable spacing.

The deliverable is four files with no build step, no npm, and no external CDN or font dependencies:

| File | Purpose | Owner |
|------|---------|-------|
| `app/project-data.json` | Project data with a top-level `projects` array | **Coder** |
| `app/index.html` | Semantic page shell plus an inline script that fetches `project-data.json` and renders `.project-card` elements | **Coder**, built to the Designer's markup and accessibility contract |
| `app/styles.css` | All visual design: layout, cards, badges, priority treatment, responsive behavior, focus and contrast | **Designer** |
| `.vscode/launch.json` | **Run Project Pulse Dashboard** launch configuration. It serves `app/` on port 5500 and opens `index.html` | **Coder** |

Each file has exactly one owner. The Orchestrator coordinates, writes no files, and passes the Designer's review notes on `app/index.html` to the Coder. No agent stages, commits, or pushes. The learner controls git through Copilot CLI prompts.

### Research basis

- `.github/project-pulse-brief.md` sets the required files, the top-level `projects` array, and the five required fields: `name`, `owner`, `status`, `recentActivity`, `priority`. It also says the launch must show `index.html`, not a directory listing.
- `.github/steps/3-step.md` (the Step 3 prompt) sets:
  - the exact title "Project Pulse"
  - the `project-card` class
  - the `.dashboard` and `.project-card` selectors
  - `border-radius`, `box-shadow`, and a responsive layout
  - the launch name "Run Project Pulse Dashboard"
  - the command `python3 -m http.server 5500`
  - a `serverReadyAction` that opens `http://localhost:%s/index.html`
- `.github/workflows/3-step.yml` checks literal keyphrases:
  - `app/index.html` must contain `Project Pulse`, `styles.css`, `project-data.json`, `project-card`, `status`, `recentActivity`, and `priority`.
  - `app/styles.css` must contain `.dashboard`, `.project-card`, `border-radius`, and `box-shadow`.
  - `app/project-data.json` must contain `projects`, `name`, `owner`, `status`, `recentActivity`, and `priority`.
  - `.vscode/launch.json` must contain `Run Project Pulse Dashboard` and `index.html`.
  - Both JSON files must pass `python3 -m json.tool`.
- `.github/agents/coder.agent.md` requires `.vscode/launch.json` to be strict JSON with no comments, with `cwd` set to `${workspaceFolder}/app`. The launch must open `index.html` and use deterministic names, commands, ports, and URLs.
- `.github/agents/designer.agent.md` requires a polished dashboard with cards, badges, priority treatment, spacing, responsive layout, `.dashboard` and `.project-card` hooks, rounded corners, shadows, contrast, and clear typography.
- `.vscode/tasks.json` already exists and must not be modified. `.vscode/launch.json` is a new file next to it.
- `.devcontainer/devcontainer.json` uses the `universal:2` image, so `python3` is available. The Python VS Code extension is not explicitly installed. That is why the launch configuration below uses the built-in `node-terminal` debug type.

---

## Shared contracts (decided here so phases can run in parallel)

### Contract A: data schema for `app/project-data.json`

```json
{
  "projects": [
    {
      "name": "string, required, non-empty",
      "owner": "string, required, non-empty (display name or @handle)",
      "status": "one of: on-track | at-risk | blocked | done",
      "recentActivity": "string, required, one short sentence",
      "priority": "one of: high | medium | low",
      "summary": "string, optional, one or two contributor-friendly sentences"
    }
  ]
}
```

- The top-level value is an object. The only required key is `projects`, which holds an array.
- `status` and `priority` are lowercase slugs. They are machine values that map directly to CSS modifier classes. Display labels are produced in the HTML script:
  - `on-track` → "On track"
  - `at-risk` → "At risk"
  - `blocked` → "Blocked"
  - `done` → "Done"
  - `high` → "High", `medium` → "Medium", `low` → "Low"
- The file holds 5 or 6 projects. Together they must include every status value and every priority value at least once, so every badge style can be checked visually. At least one project should have long text to exercise wrapping.
- Use 2-space indentation and UTF-8. No trailing commas or comments.

### Contract B: markup and class names shared by `app/index.html` and `app/styles.css`

The Coder must produce this structure exactly. The Designer styles exactly these hooks.

```
<body>
  <main class="dashboard">
    <header class="dashboard-header">
      <h1 class="dashboard-title">Project Pulse</h1>
      <p class="dashboard-subtitle">…</p>
    </header>
    <section class="dashboard-summary" aria-label="Status overview">   (counts per status; optional but recommended)
      <div class="summary-item summary-item--{status}"> <span class="summary-count">N</span> <span class="summary-label">On track</span> </div>
    </section>
    <p class="dashboard-message" role="status" aria-live="polite">…</p>   (loading / empty / error; hidden when cards render)
    <ul class="project-grid" role="list">
      <li class="project-grid-item">
        <article class="project-card project-card--priority-{high|medium|low|unknown}" aria-labelledby="project-{index}-name">
          <header class="project-card-header">
            <h2 class="project-name" id="project-{index}-name">…</h2>
            <span class="status-badge status-badge--{on-track|at-risk|blocked|done|unknown}">On track</span>
          </header>
          <p class="project-owner"><span class="field-label">Owner</span> …</p>
          <p class="project-summary">…</p>   (only if summary present)
          <div class="project-activity"><span class="field-label">Recent activity</span> <p>…</p></div>
          <p class="project-priority"><span class="field-label">Priority</span> <span class="priority-badge priority-badge--{high|medium|low|unknown}">High</span></p>
        </article>
      </li>
    </ul>
    <footer class="dashboard-footer">…</footer>   (optional, e.g. "Data from project-data.json")
  </main>
</body>
```

Additional state classes: `.dashboard-message--error` and `.dashboard-message--empty`.

Rules that both agents rely on:

- Badges always contain visible text. Color is never the only signal.
- `unknown` is the fallback modifier for any `status` or `priority` value outside the enum.
- `{index}` is the 0-based array position, which keeps IDs deterministic.

---

## Designer responsibilities

1. **Own `app/styles.css`.** Build the whole visual system against Contract B. Use only the class names it lists, and add no selectors that depend on markup outside the contract.
2. **Required selectors and properties:** `.dashboard`, `.project-card`, `border-radius`, and `box-shadow` must appear literally, because the workflow checks for them.
3. **Visual design:**
   - Use CSS custom properties in `:root` for colors, spacing, radius, and shadow.
   - Use a system font stack (no web fonts) with a clear type scale: h1 larger than h2 larger than body text.
   - Build the card grid with `display: grid; grid-template-columns: repeat(auto-fill, minmax(18rem, 1fr)); gap: …`.
   - Give cards rounded corners, a soft shadow, a subtle hover or `:focus-within` lift, and generous padding.
4. **Status badges:** pill-shaped, with a distinct color per `--on-track`, `--at-risk`, `--blocked`, `--done`, and `--unknown`. Text contrast must be at least 4.5:1.
5. **Priority treatment:** a `.priority-badge` per level, plus a card accent such as a colored left border or top bar keyed off `.project-card--priority-high`, `--medium`, `--low`, and `--unknown`. High priority must stand out without relying on color alone; the text label is always present.
6. **Responsive layout:** a single column below about 600px, with header and padding that scale down. No horizontal scrolling at 320px width.
7. **Accessibility in CSS:**
   - Visible `:focus-visible` outlines.
   - A `@media (prefers-reduced-motion: reduce)` block that disables transitions.
   - `overflow-wrap: anywhere` on names, owners, and activity text.
   - Readable line-height of about 1.5.
8. **State styling:** `.dashboard-message`, `--error`, and `--empty` should look intentional and clearly readable, not like a bare text line.
9. **Review `app/index.html` in Phase 2** for semantics, heading order, `lang`, viewport meta, aria usage, and contrast in context. Return findings to the Orchestrator for the Coder to apply. The Designer does **not** edit `app/index.html`.

## Coder responsibilities

1. **Own `app/project-data.json`.** Write 5 or 6 realistic sample projects that follow Contract A exactly.
2. **Own `app/index.html`:**
   - Use `<!DOCTYPE html>`, `<html lang="en">`, `<meta charset="utf-8">`, and `<meta name="viewport" content="width=device-width, initial-scale=1">`.
   - Set `<title>Project Pulse</title>` and make the `<h1>` read exactly "Project Pulse".
   - Add `<link rel="stylesheet" href="styles.css">` with a relative path.
   - Put the rendering logic in one inline `<script>`. No separate JS file, because only four files are assigned. The script should:
     - Call `fetch('project-data.json')` with a relative URL. Check `response.ok` and throw an explicit error otherwise.
     - Validate the shape: the top-level object must have a `projects` array, and each item must have the 5 required fields.
     - Build DOM nodes with `document.createElement` and `textContent`. Never use `innerHTML` with data, so there is no injection risk.
     - Map `status` and `priority` slugs to labels and modifier classes, with an `unknown` fallback.
     - Render cards in the same order as the JSON, for deterministic output.
     - Render the optional status-count summary.
     - Handle loading, empty, and error states through `.dashboard-message`.
   - The literal strings `project-card`, `status`, `recentActivity`, and `priority` will naturally appear in the script, which satisfies the workflow keyphrase checks.
3. **Own `.vscode/launch.json`.** It must be strict JSON with no comments and no trailing commas. Recommended content:
   ```json
   {
     "version": "0.2.0",
     "configurations": [
       {
         "name": "Run Project Pulse Dashboard",
         "type": "node-terminal",
         "request": "launch",
         "command": "python3 -m http.server 5500",
         "cwd": "${workspaceFolder}/app",
         "env": {
           "PYTHONUNBUFFERED": "1"
         },
         "serverReadyAction": {
           "action": "openExternally",
           "pattern": "Serving HTTP on .* port ([0-9]+)",
           "uriFormat": "http://localhost:%s/index.html"
         }
       }
     ]
   }
   ```
   - Why `node-terminal`: it is built into VS Code's JavaScript debugger, so no Python extension is needed. It runs the exact command from the Step 3 prompt in a terminal.
   - `serverReadyAction` reads the port from the `http.server` startup line and opens `index.html` directly, not the directory listing.
   - `PYTHONUNBUFFERED` makes sure the "Serving HTTP" line is flushed right away so the pattern matches.
4. **Apply the Designer's review findings** to `app/index.html` in Phase 2.
5. **Run the validation checklist** below and report results, including any remaining risk.

---

## Ordered implementation steps

| # | Phase | Step | Agent | Files (exclusive write scope) | Depends on |
|---|-------|------|-------|-------------------------------|------------|
| 0 | Contract | Orchestrator confirms Contracts A and B from this plan and passes them to both specialists. Any change to a contract must happen here, before Phase 1. | Orchestrator | none | the plan |
| 1a | Build (parallel) | Create sample data that follows Contract A. | Coder | `app/project-data.json` | 0 |
| 1b | Build (parallel, after 1a within the Coder track) | Create the page shell and render script that follow Contracts A and B. | Coder | `app/index.html` | 0, 1a (needs real data to test against) |
| 1c | Build (parallel, any time within the Coder track) | Create the launch configuration. | Coder | `.vscode/launch.json` | 0 |
| 1d | Build (parallel with 1a–1c) | Create the full stylesheet following Contract B, including all state, badge, and priority modifiers. | Designer | `app/styles.css` | 0 |
| 2a | Integration review | Run the dashboard and review the rendered page plus `app/index.html` markup for accessibility and hierarchy. Report findings and any CSS fixes needed. | Designer | `app/styles.css` (only CSS fixes) | 1a–1d |
| 2b | Integration fixes | Apply the Designer's markup and accessibility findings to `app/index.html`. | Coder | `app/index.html` | 2a |
| 3 | Validation | Run every check in "Validation expectations" and report pass or fail for each. | Coder (runs commands); Orchestrator (verifies the integrated result) | none (read-only) | 2b |
| 4 | Report | Summarize what each agent did, the validation results, and any open risks. No git operations. | Orchestrator | none | 3 |

## Dependencies

1. **Data schema before rendering.** `app/index.html` reads field names and enum values from `app/project-data.json`. Contract A removes the *design-time* dependency. The Coder should still write 1a before testing 1b so the renderer runs against real data. Both are in the Coder's track, so this ordering costs nothing.
2. **Class-name contract between HTML and CSS.** `app/styles.css` and `app/index.html` depend on each other only through Contract B. Because the contract is fixed in Phase 0, the two can be built at the same time. Any class name added outside the contract needs both agents to agree, so it goes to Phase 2.
3. **Launch configuration and the app directory.** `.vscode/launch.json` depends only on the `app/` path, the `index.html` filename, and port 5500, all decided in this plan. It can be written at any time, but it can only be *exercised* once `app/index.html` exists (Phase 3).
4. **Review before fixes.** Step 2b depends on 2a's findings, and 2a needs every Phase 1 file to exist.
5. **Validation is last.** Phase 3 needs every file in its final state.

## Parallel work decisions

**Can run in parallel (Phase 1):**

- The **Designer** on `app/styles.css` and the **Coder** on `app/project-data.json`, `app/index.html`, and `.vscode/launch.json`.
  - Reason: the write scopes do not overlap; each file has one owner. The only cross-file coupling (class names, data fields) is fixed by Contracts A and B before Phase 1 starts, so neither agent waits on the other's output.
- Within the Coder track, `.vscode/launch.json` has no data dependency on the other files and can be written in any order.

**Must run sequentially:**

- `app/project-data.json`, then testing of `app/index.html` (1a then 1b). The renderer must be checked against real data; same agent, so they run in order.
- Phase 1, then Phase 2a. The Designer's integration review needs the rendered page, which needs all Phase 1 files.
- 2a, then 2b. Fixes to `app/index.html` come from the review findings. `app/index.html` has only one owner (the Coder), so the Designer reports rather than edits. That keeps overlapping scopes in separate phases, as `orchestrator.agent.md` requires.
- Phase 2, then Phase 3, then Phase 4. Validation and reporting need the final integrated files.

**Explicitly not parallel:**

- No two agents ever write the same file in the same phase.
- If the Designer finds a need for a new class hook during Phase 1, they record it for Phase 2. They do not ask the Coder to change `app/index.html` mid-phase.

---

## Edge cases to handle

| Case | Expected behavior | Owner |
|------|-------------------|-------|
| Page opened via `file://` (fetch blocked by the browser) | Catch the fetch error and show `.dashboard-message--error` with the text: "Could not load project data. Open the dashboard with the **Run Project Pulse Dashboard** launch configuration (it serves the app over http://localhost:5500)." Do not show a blank page. | Coder (logic), Designer (style) |
| `project-data.json` missing or 404 | `response.ok` is false, so throw an explicit error that includes the HTTP status and show the error message. | Coder |
| Invalid JSON | `response.json()` rejects, so show the error state. Log the details with `console.error`. | Coder |
| JSON valid but no `projects` key, or `projects` is not an array | Treat it as a data error with a clear message such as "project-data.json must contain a top-level \"projects\" array." | Coder |
| `projects` is an empty array | Show `.dashboard-message--empty`: "No projects to show yet." Hide the grid. Summary counts show zero or are hidden. | Coder, Designer |
| A project is missing a required field, or the field is an empty string | Still render the card, with an em dash or "Not provided" placeholder for that field. Log a `console.warn` naming the project index and field. Do not drop the card silently. | Coder |
| Unknown `status` or `priority` value (for example `"paused"` or `"urgent"`) | Use the `unknown` modifier class, keep a neutral gray style, and display the raw value as the label so information isn't lost. | Coder, Designer |
| Values that differ only in case or spacing (`"High"`, `" at-risk "`) | Trim and lowercase before mapping. | Coder |
| Non-string values (numbers, null, objects) | Convert with `String()` for display only when the value is a primitive. Otherwise treat it as missing. | Coder |
| Long names, owners, or activity text, or long unbroken strings or URLs | Wrap with `overflow-wrap: anywhere`. Cards grow in height and the grid stays aligned. No horizontal overflow at 320px. | Designer |
| Many projects (20 or more) | The grid flows naturally and the page scrolls. No fixed heights. | Designer |
| HTML or script in data values | Rendered as text through `textContent`, never interpreted. | Coder |
| Port 5500 already in use | `http.server` exits with "Address already in use", so `serverReadyAction` never fires. Document the fix: stop the earlier debug session or terminal, then relaunch. Keep the port fixed at 5500 for determinism. | Coder (documented in the report) |
| Codespaces port forwarding | `openExternally` on `http://localhost:5500/index.html` is auto-forwarded by Codespaces. If the browser doesn't open, the learner can open the forwarded port 5500 from the Ports panel and add `/index.html`. | Coder (documented in the report) |
| Browser caching stale JSON during edits | Optional: `fetch('project-data.json', { cache: 'no-store' })`. | Coder |
| Reduced motion or high contrast preference | Respect `prefers-reduced-motion`. Badges stay text-labeled, so the page works without color. | Designer |

---

## Validation expectations

Run from the repo root. Every item should report pass or fail explicitly.

### Static file checks

1. All four files exist: `app/index.html`, `app/styles.css`, `app/project-data.json`, `.vscode/launch.json`.
2. `python3 -m json.tool app/project-data.json >/dev/null` exits 0.
3. `python3 -m json.tool .vscode/launch.json >/dev/null` exits 0. This proves there are no comments or trailing commas.
4. Schema check with a one-off Python snippet. Nothing is saved to the repo.
   - The top-level object has a `projects` list of length 5 or more.
   - Every item has non-empty string values for `name`, `owner`, `status`, `recentActivity`, and `priority`.
   - Every `status` is in `{on-track, at-risk, blocked, done}` and every `priority` is in `{high, medium, low}`.
   - Each enum value appears at least once.
5. Keyphrase checks that mirror `.github/workflows/3-step.yml`, all `grep -qi`:
   - `app/index.html`: `Project Pulse`, `styles.css`, `project-data.json`, `project-card`, `status`, `recentActivity`, `priority`
   - `app/styles.css`: `.dashboard`, `.project-card`, `border-radius`, `box-shadow`
   - `app/project-data.json`: `projects`, `name`, `owner`, `status`, `recentActivity`, `priority`
   - `.vscode/launch.json`: `Run Project Pulse Dashboard`, `index.html`, `python3 -m http.server 5500`, `${workspaceFolder}/app`, `http://localhost:%s/index.html`, `serverReadyAction`
6. `app/index.html` contains `<html lang="en"`, the viewport meta tag, and exactly one `<h1>`. It has no `innerHTML` assignments that use data values.
7. `app/styles.css` contains `@media` (responsive), `:focus-visible`, and `prefers-reduced-motion`.
8. `git diff --stat -- .vscode/tasks.json` shows no changes.

### Runtime checks

Run from `app/` with `python3 -m http.server 5500 &`, then stop the server afterwards.

1. `curl -s -o /dev/null -w '%{http_code}' http://localhost:5500/index.html` returns `200`.
2. `curl -s http://localhost:5500/index.html` contains `Project Pulse` and does **not** contain `Directory listing for`.
3. `curl -s http://localhost:5500/project-data.json | python3 -m json.tool` succeeds.
4. `curl -s -o /dev/null -w '%{http_code}' http://localhost:5500/styles.css` returns `200`.

### Manual checks

Done by the learner or Orchestrator in VS Code.

1. Run and Debug, then **Run Project Pulse Dashboard**, opens a browser at `…/index.html`, not a directory listing.
2. The page shows the "Project Pulse" title and one `.project-card` per project. Each card shows the name, owner, status badge, recent activity, and priority.
3. Every status and priority style is visibly distinct and labeled with text. High-priority cards stand out.
4. Resize to about 375px wide: a single column with no horizontal scroll. Resize to 1280px or wider: multiple columns.
5. Pressing Tab shows visible focus styles on any focusable elements.
6. Opening `app/index.html` directly with `file://` shows the friendly error message, not a blank page.
7. Temporarily changing a status to `"paused"` in a local, unsaved experiment shows the neutral `unknown` badge. Revert afterwards.
8. Stop the debug session; the server on port 5500 shuts down.

### Process checks

1. Each file was written only by its assigned owner.
2. No agent ran `git add`, `git commit`, or `git push`. `git status` shows the four files as untracked or modified, with nothing staged by agents.

> Note: `scripts/validate-exercise.sh` is the template maintainers' self-check. It includes a test that fails once learner answer files are committed ("Learner answer files are tracked in the template"). Don't treat its overall exit code as the pass gate for this dashboard. Use the checks above instead.

---

## Open questions

1. **Sort order:** should cards keep the JSON order (the current deterministic default), or sort by priority (high first) or by status (blocked and at-risk first)? This plan assumes JSON order, with the data file ordered by importance.
2. **Status vocabulary:** is `on-track / at-risk / blocked / done` the right set for Mona's team, or should there be values like `planning` or `paused`? Unknown values degrade gracefully either way.
3. **`recentActivity` format:** this plan uses a free-text sentence. Should it instead be a structured object with a date, so the dashboard can show relative time ("2 days ago") or sort by recency? Free text keeps the required field a simple string and keeps the workflow checks unaffected.
4. **Optional `summary` field:** the brief asks for "a short contributor-friendly summary". This plan adds it as an optional sixth field. Confirm that is acceptable, or decide whether the summary should be required.
5. **Status overview strip:** `.dashboard-summary` with counts per status is recommended but not required. Should it be included in the first iteration?
6. **Launch type:** `node-terminal` is recommended because it needs no extra extension. If the learner's environment has the Python extension and prefers `debugpy` with `"module": "http.server"`, the name, `cwd`, port, and `serverReadyAction` must stay the same. Confirm the preference.
7. **Dark mode:** should the Designer add a `prefers-color-scheme: dark` variant now, or defer it? This plan treats it as optional polish after the required items pass.
