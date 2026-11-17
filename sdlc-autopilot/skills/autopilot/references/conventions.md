# sdlc-autopilot — shared conventions

Every stage agent reads this file first. It exists so that five agents running in
separate contexts still produce one coherent, traceable chain of work.

## 1. The config file: `<repo-root>/.claude/autopilot.json` (main checkout)

Holds every ID that was discovered once and should never be re-discovered.
Read it at the start of your stage; if you discover a missing value, add it
(merge, never overwrite other keys). Shape:

```json
{
  "jira":     { "cloudId": "", "siteUrl": "", "projectKey": "", "sprintFieldId": "customfield_10020",
                "priorityMap": { "critical": "Highest", "high": "High", "medium": "Medium", "low": "Low" } },
  "github":   { "repo": "owner/name", "baseBranch": "main" },
  "vercel":   { "teamId": "", "projectId": "", "projectName": "", "productionUrl": "" },
  "supabase": { "projectRef": "" },
  "figma":    { "planKey": "" }
}
```

Seed values can come from `.vercel/project.json` (projectId/orgId), `git remote -v`,
and `supabase/config.toml` / env files (project ref only — never copy secrets).
This file holds IDs, not secrets. Never write tokens or keys into it.

## 2. The work folder

One folder per story: `docs/work/<YYYY-MM-DD>-<JIRAKEY>-<slug>/`
(e.g. `docs/work/2026-11-17-AB-12-wishlist-sharing/`).

| File | Written by | Purpose |
|---|---|---|
| `intent.md` | plan-agent | why + what, measurable success criteria |
| `spec.md` | design-agent | requirements, Figma link, design, edge cases |
| `plan.md` | build-agent | files, ordered steps, checks, rollback |
| `incident.md` | maintain-agent | post-deploy findings |

**Where it lives over time:**
- Before the worktree exists (plan stage), plan-agent writes `intent.md` in the
  main checkout's `docs/work/...` folder, **uncommitted**.
- Right after the intent is approved, the orchestrator has build-agent create the
  worktree and **move** the folder into it. From then on, every artifact lives in
  the worktree and is committed on the feature branch, so the whole
  intent → spec → plan → code chain ships in one PR. That is the traceability the
  playbook asks for.
- `incident.md` is written after the merge, into the work folder in the main
  checkout (it exists on main once `git pull` has run). Leave it uncommitted;
  the user decides whether to commit it.

## 3. Git and worktrees

- Branch: `feat/<JIRAKEY>-<slug>` (lowercase slug, max ~40 chars).
- Worktree path: `<parent-of-repo-root>/<repo-name>-wt/<JIRAKEY>` (sibling of the
  repo, outside it, and space-free — the repo folder name contains a space, so
  never derive the worktree name from it). Always quote paths in shell commands.
- Paths: the orchestrator passes you absolute paths (repo root, config file,
  conventions file, work folder, worktree). Use them; never rely on the cwd.
- `env/.env` (Vite `envDir: 'env'`) is gitignored, so a new worktree lacks it.
  Symlink it in: `mkdir -p "<wt>/env" && ln -s "<repo-root>/env/.env" "<wt>/env/.env"`.
  Never edit code to work around a missing env file.
- Create with `git fetch origin && git worktree add -b <branch> <path> origin/main`.
- Run `npm ci` in a new worktree before building (node_modules is not shared).
- **Never** `git add -A` or `git add .` in the main checkout — the user has
  untracked work there (e.g. `apps/`). In the worktree, stage explicit paths.
- Commits: `<JIRAKEY>: <imperative summary>`.
- Never push to `main` directly. The only way code reaches main is the PR merge
  the orchestrator performs **after** the user approves in chat.

## 4. The production gate

Agents do everything up to the gate and nothing past it. The gates are owned by
the orchestrator skill in the main conversation — **stage agents never wait for
approval and never ask the user questions**. If you hit a decision only a human
can make, stop and put it under `Open questions` in your report.

Production deploy happens only in ship-agent, which is only launched after the
user has explicitly approved the PR.

## 5. Jira rules

- The Atlassian connector may be on a deprecated v1 endpoint. After the first
  Atlassian call in the session, include in your report the line:
  `NOTE: Atlassian connector is on the deprecated v1 endpoint (removed 2027-03-01); re-add it from Claude's connector directory or use https://mcp.atlassian.com/v2/mcp.`
- **Never hardcode status names for transitions.** Call
  `getTransitionsForJiraIssue`, pick the transition whose target name best
  matches the intent (case-insensitive: "In Progress"; "In Review"/"Review"/
  "Code Review"; "Done"/"Closed"/"Resolved"). If none fits, skip and report it.
- Priority names come from the project (read them once and store the mapping
  in `jira.priorityMap`).
- Comment on the Jira issue at each milestone with the relevant link (PR URL,
  production URL), so Jira is a readable history too.

## 6. Every stage report

End with exactly this block, which the orchestrator parses:

```
## Stage report: <stage>
- Status: done | blocked
- Story: <KEY> — <title>
- Files: <paths written>
- Links: <Jira / Figma / PR / deployment URLs>
- Jira: <transitions made, comments added>
- Notes: <anything the user should know, incl. connector notices>
- Open questions: <none | list>
```
