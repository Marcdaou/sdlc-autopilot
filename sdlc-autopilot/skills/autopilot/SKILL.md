---
name: autopilot
description: Runs the full AI-native SDLC loop end to end — picks the highest-priority To Do Jira story, writes intent.md, designs it in Figma with spec.md, builds it in a git worktree with plan.md, opens a GitHub PR, and after the user approves merges it, deploys to Vercel production, checks Vercel and Supabase for incidents and turns verified incidents into new Jira stories. Use whenever the user says /autopilot, "run the loop", "pick up the next story", "ship the next Jira ticket", "run the SDLC", or wants to take a Jira story all the way to production.
---

# sdlc-autopilot — orchestrator

You run in the main conversation and own the **human gates**. Stage agents do the
work in their own context and return a stage report; they cannot talk to the
user, so every approval happens here, via AskUserQuestion, between agent calls.

Read `references/conventions.md` once now — it defines the config file, work
folder, worktree layout and Jira rules the agents rely on. Its absolute path is
`<Base directory for this skill>/references/conventions.md` (the base directory
is printed when this skill loads).

Resolve the repo root once with `git rev-parse --show-toplevel`. In **every**
agent prompt pass absolute paths: repo root, config file
(`<root>/.claude/autopilot.json`), conventions file, and — once they exist — work
folder, worktree path, branch, story key. Agents run with an unknown cwd.

Always launch agents by their full name (`sdlc-autopilot:plan-agent`, etc.).
Other SDLC plugins with similar agent names may be installed; the prefix avoids
picking the wrong one.

Keep a running state in your head (and in the reports): story key, work folder,
worktree path, branch, PR URL, deployment URL.

## The flow

```
0 Setup ─► 1 Plan ─[GATE intent]─► 2 Design ─[GATE spec]─► 3 Build+PR ─[GATE PR]─► 4 Ship ─► 5 Maintain ─[GATE incident]─► 6 Stories
```

### 0. Setup (first run only, or when config is incomplete)
1. Read `.claude/autopilot.json` in the repo root. If it has `jira.cloudId` and
   `jira.projectKey`, skip to step 1.
2. Otherwise call the Atlassian tools yourself: `getAccessibleAtlassianResources`
   to get the cloudId, then `getVisibleJiraProjects`. Ask the user which project
   to use with AskUserQuestion (list up to 4 projects; "Other" covers the rest).
   Tell the user about the deprecated v1 endpoint notice if it appears.
3. Seed the rest of the config from `.vercel/project.json` and `git remote -v`.
   Write the file. It holds IDs, not secrets; leave it uncommitted unless the
   user asks to commit it.

### 1. Plan
Launch `sdlc-autopilot:plan-agent`. It finds the highest-priority To Do story in
the active sprint and writes `intent.md` in the main checkout (uncommitted).

If it reports no To Do stories, tell the user and stop.

**GATE — intent.** Show the story key/title and the full intent.md, then ask:
`Approve` / `Request changes` / `Skip this story` / `Stop`.
- Request changes → re-launch plan-agent with the user's notes and the same story key.
- Skip → re-launch plan-agent telling it to exclude that key.
- Approve → continue.

### 2. Worktree + Design
1. Launch `sdlc-autopilot:build-agent` in **setup** mode: create branch +
   worktree, move the work folder into it, commit intent.md, move the Jira
   story to In Progress.
2. Launch `sdlc-autopilot:design-agent` with the worktree path and work folder.
   It creates a new Figma file, draws the screens, writes spec.md and commits it.

**GATE — spec.** Show the Figma link and spec.md (summarise long sections, but
show every requirement). Ask: `Approve` / `Request changes` / `Stop`.
Request changes → re-launch design-agent with the notes (it updates the same
Figma file and spec.md).

### 3. Build + PR (no gate before the PR — the user asked for this to be automatic)
Launch `sdlc-autopilot:build-agent` in **implement** mode. It writes and commits
plan.md, implements it, runs `npm run lint` and `npm run build` until both pass,
pushes the branch, opens the PR and moves Jira to In Review.

If it reports `blocked` (e.g. build failures it could not fix), show the
details and ask the user how to proceed. Do not open the PR yourself.

**GATE — PR.** Show the PR URL, the diff stats and a short summary of what
changed. Ask: `Approve — merge and deploy to production` / `Request changes` / `Stop (leave PR open)`.
This is the **production gate**: there is no branch protection on this repo, so
the user's explicit "Approve" here is the only thing standing between the code
and production. Never treat anything else (silence, "looks good so far", a
question) as approval.
- Request changes → re-launch build-agent in **revise** mode with the notes; it
  pushes new commits to the same PR. Then ask again.

### 4. Ship
Launch `sdlc-autopilot:ship-agent` with the PR number, worktree path and branch.
It merges, removes the worktree and branch, deploys to Vercel production, and
moves Jira to Done. Show the production URL.

If the deploy fails, ship-agent reports it — go straight to Maintain, the
failure is the first incident.

### 5. Maintain
Launch `sdlc-autopilot:maintain-agent` in **check** mode with the deployment
URL/ID and the work folder. It reads Vercel build + runtime logs and Supabase
advisors + logs, then writes `incident.md`.

- If it reports **no incidents**: show the summary and finish — no gate, no stories.
- Otherwise **GATE — incident.** Show incident.md. Ask:
  `Verified — create the stories` / `Edit first` / `Discard`.
  - Edit first → take the user's edits (they may name incidents to drop or
    re-grade), re-launch maintain-agent in **check** mode with those notes, ask again.
  - Discard → finish.

### 6. Stories
Launch `sdlc-autopilot:maintain-agent` in **stories** mode. It creates one Jira
story per verified incident in the same sprint, To Do, priority from severity,
skipping titles that already exist. Show the created keys with links.

### Finish
Print a short recap: story shipped, PR, production URL, incidents, new stories.
Offer to run the loop again (the new stories may now be the top priority).

## When things go wrong
- An MCP server is not connected (Jira, Figma, Vercel): stop that stage, tell the
  user which connector is missing and how to connect it (`/mcp`), and offer to
  continue once it is. Vercel has a CLI fallback inside ship/maintain agents.
- An agent returns `blocked`: show its Open questions and ask the user.
- The user says stop at any time: leave everything as is (worktree, branch, PR)
  and tell them exactly what state things are in so they can resume.
