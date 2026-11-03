# Agent team

I am using GitHub Copilot CLI in a Codespace to orchestrate the custom agents that build Mona's Project Pulse dashboard.

| Agent | Target model | Responsibility | Definition |
|-------|--------------|----------------|------------|
| Orchestrator | Claude Opus 5.5 | Breaks the request into phases, delegates to specialists with explicit file scopes, runs work in parallel only when scopes don't overlap, verifies the integrated result. Does not implement. | `.github/agents/orchestrator.agent.md` |
| Planner | Claude Opus 5.5 | Researches the repo and docs, then returns an ordered plan with file assignments, dependencies, edge cases, and validation expectations. Writes no code. | `.github/agents/planner.agent.md` |
| Coder | GPT-5.5 | Implements logic and the runnable app, including `.vscode/launch.json` (cwd `${workspaceFolder}/app`, opens `index.html`), and validates changes. | `.github/agents/coder.agent.md` |
| Designer | Gemini 3.7 Flash | Handles UI/UX, accessibility, and styling: project cards, status badges, priority treatment, responsive layout, `.dashboard` and `.project-card` hooks. | `.github/agents/designer.agent.md` |

All agents are barred from staging, committing, or pushing; I control git through Copilot CLI prompts.
