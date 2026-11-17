# sdlc-autopilot

Anthropic's AI-native SDLC playbook with Jira, Figma, GitHub, Vercel and Supabase.

Run `/sdlc-autopilot:autopilot`. After editing the plugin source, bump `version` in plugin.json and reinstall (installs are cached copies).

Plan (Jira -> intent.md) -> GATE -> Design (Figma + spec.md) -> GATE -> Build (worktree, plan.md, lint+build, PR) -> GATE (production gate) -> Ship (merge, remove worktree, Vercel prod) -> Maintain (incident.md) -> GATE -> new Jira stories in the same sprint.

Requires: Atlassian, Figma, Supabase MCP connectors; `gh` and `vercel` CLIs logged in (Vercel MCP used if connected).
Config with discovered IDs: `.claude/autopilot.json`. Work artifacts: `docs/work/<date>-<KEY>-<slug>/`.

## Install

```
/plugin marketplace add Marcdaou/sdlc-autopilot
/plugin install sdlc-autopilot@sdlc-autopilot
/reload-plugins
```

Then create `.claude/autopilot.json` in your project (Jira cloudId/projectKey, GitHub repo, Vercel team/project, Supabase ref, Figma plan key). The plugin discovers and fills missing IDs on first run.
