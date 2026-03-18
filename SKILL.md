---
name: agentify
description: Configure any project for agentic workflow. Detects stack, asks preferences, generates agents + dispatcher + board setup.
allowed-tools: Bash(*), Read, Write, Edit, Glob, Grep, Agent, AskUserQuestion
---

# /agentify — Agentic Workflow Generator

Interactive skill that sets up a complete agentic workflow for any project. Detects the stack, asks preferences, generates tailored agents + dispatcher + board config.

*Output*: ⁠ .claude/agents/*.md ⁠, ⁠ .claude/skills/{dispatcher}/SKILL.md ⁠, CLAUDE.md workflow section, board labels.

---

## Phase 0: Introduction

Before asking any configuration questions, present this introduction to the user:

---

**Welcome to /agentify!**

An agentic workflow lets Claude Code act as a team of specialized agents — an architect who plans, a developer who codes, a reviewer who checks quality, and a QA tester who validates — all coordinated automatically through your task board.

Here's what that looks like end-to-end:

```
  ┌─────────────┐
  │ You create   │
  │ a GitHub     │
  │ issue        │
  └──────┬───────┘
         ▼
  ┌──────────────┐     ┌──────────────┐
  │  🏗 Architect │────▶│  💻 Developer │
  │  Plans the   │     │  Writes code │
  │  approach    │     │  + tests     │
  └──────────────┘     └──────┬───────┘
                              ▼
                       ┌──────────────┐
                       │  🔍 Reviewer  │──── Changes? ──▶ Back to Dev
                       │  Checks code │
                       └──────┬───────┘
                              ▼
                       ┌──────────────┐
                       │  ✅ QA        │──── Fails? ────▶ Back to Dev
                       │  Runs tests  │
                       └──────┬───────┘
                              ▼
                       ┌──────────────┐
                       │  🚀 Merged!   │
                       │  Done.       │
                       └──────────────┘
```

**Why use an agentic workflow?**
- You create issues → agents do the work autonomously
- Each agent is specialized (like a real team)
- Quality gates (review + QA) catch problems before merge
- Works with your existing tools (GitHub Issues, Linear, Jira)

**Before vs After:**
- **Before**: You create issue → you plan → you code → you test → you PR → you merge
- **After**: You create issue → `/agile` → architect plans → dev codes → reviewer checks → QA tests → you approve the merge

---

Let's set this up for your project. I'll ask a few questions to tailor the workflow to your stack and preferences. It takes about 2 minutes.

---

## Phase 1: Collect Information

Work through each step sequentially. Use AskUserQuestion for all user input.

### Step 1: Task Board

1.⁠ ⁠Auto-detect git remote:
⁠ bash
REMOTE_URL=$(git remote get-url origin 2>/dev/null || echo "none")
 ⁠

2.⁠ ⁠Infer platform from remote URL:
   - ⁠ github.com ⁠ → GitHub
   - ⁠ gitlab.com ⁠ → GitLab
   - ⁠ bitbucket.org ⁠ → Bitbucket

3.⁠ ⁠Ask the user which task board they use:
   - *GitHub Issues* (default if GitHub detected)
   - *Linear*
   - *Jira*

4.⁠ ⁠If Linear: ask for team key (e.g., ⁠ ENG ⁠).
   If Jira: ask for project key (e.g., ⁠ PROJ ⁠) and base URL (e.g., ⁠ https://company.atlassian.net ⁠).

5.⁠ ⁠Check CLI availability based on selection:

| Board | CLI | Check command |
|-------|-----|---------------|
| GitHub Issues | ⁠ gh ⁠ | ⁠ gh --version ⁠ |
| Linear | ⁠ linear ⁠ | ⁠ linear --version ⁠ |
| Jira | ⁠ jira ⁠ | ⁠ jira --version ⁠ |

If the CLI is not installed, show install guidance:
•⁠  ⁠⁠ gh ⁠: ⁠ brew install gh ⁠ / ⁠ https://cli.github.com/ ⁠
•⁠  ⁠⁠ linear ⁠: ⁠ npm install -g @linear/cli ⁠
•⁠  ⁠⁠ jira ⁠: ⁠ brew install jira-cli ⁠ / ⁠ https://github.com/ankitpokhrel/jira-cli ⁠

Ask if they want to install now or continue anyway (commands will be generated but may fail until CLI is installed).

Store results:

BOARD_TYPE = "github" | "linear" | "jira"
BOARD_KEY = team/project key (Linear/Jira only)
BOARD_URL = base URL (Jira only)
PR_PLATFORM = "github" | "gitlab" | "bitbucket" (from git remote)


### How It Works — Key Concepts

Before choosing roles, here's a quick glossary:

- **Agents** = specialized Claude instances, each with a focused role (e.g., one plans, another codes)
- **Dispatcher** = the skill (e.g., `/agile`) that checks your board and picks the right agent for the next piece of work
- **Labels** = status tags on issues/PRs that track where work is in the pipeline (e.g., `ready`, `review`, `qa`)
- **Board** = your task tracker (GitHub Issues, Linear, or Jira) where issues live

### Step 2: Agent Roles

Present role presets and ask user to select:


Which agent roles do you want?

1. **Full team** (recommended) — architect, dev, reviewer, qa + e2e/unittest subagents
   - **Architect**: Reads your issue, explores the codebase, writes an implementation plan
   - **Developer**: Follows the plan, writes code + tests, creates a PR
   - **Reviewer**: Reviews the PR for quality, simplicity, and correctness
   - **QA**: Runs all tests, checks coverage, validates the build
2. **Solo dev** — dev + reviewer
3. **Minimal** — dev only
4. **Custom** — pick from: architect, dev, reviewer, qa, e2e tester, unittest tester


Rules:
•⁠  ⁠⁠ dev ⁠ is always included (cannot be deselected in custom)
•⁠  ⁠⁠ e2e ⁠ and ⁠ unittest ⁠ are only available if ⁠ qa ⁠ is selected (they become QA subagents)
•⁠  ⁠If user picks custom and selects e2e/unittest without qa, auto-include qa

Store results:

ROLES = ["architect", "dev", "reviewer", "qa"]  # example for full team
HAS_SUBAGENTS = true  # if qa selected with e2e/unittest
SUBAGENTS = ["e2e", "unittest"]  # qa subagents


### Step 3: Project Detection

Scan the working directory for project markers and extract info:

⁠ bash
# Check for project files
ls package.json pyproject.toml Cargo.toml go.mod pom.xml build.gradle Makefile 2>/dev/null
 ⁠

Detection matrix:

| File | Language | Framework detection |
|------|----------|-------------------|
| ⁠ package.json ⁠ | JavaScript/TypeScript | Check ⁠ dependencies ⁠ for react, next, vue, angular, express, etc. |
| ⁠ pyproject.toml ⁠ | Python | Check for django, flask, fastapi, etc. |
| ⁠ Cargo.toml ⁠ | Rust | Check for actix, rocket, tokio, etc. |
| ⁠ go.mod ⁠ | Go | Check for gin, echo, fiber, etc. |
| ⁠ pom.xml ⁠ | Java | Check for spring, etc. |
| ⁠ build.gradle ⁠ | Java/Kotlin | Check for spring, android, etc. |
| ⁠ Makefile ⁠ | Various | Fallback — scan for language clues |

Extract commands from project files:

| Info | JavaScript | Python | Rust | Go | Java (Maven) |
|------|-----------|--------|------|----|---------------|
| Package manager | npm/yarn/pnpm (check lockfiles) | pip/poetry/uv | cargo | go | mvn |
| Build | ⁠ npm run build ⁠ | ⁠ python -m build ⁠ | ⁠ cargo build ⁠ | ⁠ go build ./... ⁠ | ⁠ mvn package ⁠ |
| Lint | ⁠ npm run lint ⁠ | ⁠ ruff check . ⁠ | ⁠ cargo clippy ⁠ | ⁠ golangci-lint run ⁠ | ⁠ mvn checkstyle:check ⁠ |
| Test | ⁠ npm test ⁠ | ⁠ pytest ⁠ | ⁠ cargo test ⁠ | ⁠ go test ./... ⁠ | ⁠ mvn test ⁠ |
| E2E | Check for playwright/cypress in deps | Check for selenium/playwright | — | — | — |

Present findings to user:

Detected project:
- Language: TypeScript
- Framework: React (Vite)
- Package manager: pnpm
- Build: pnpm build
- Lint: pnpm lint
- Test: pnpm test
- E2E: pnpm test:e2e (Playwright detected)

Is this correct? (yes / correct any values)


Let user confirm or override any values.

Store results:

LANG = "typescript"
FRAMEWORK = "react"
PKG_MANAGER = "pnpm"
CMD_BUILD = "pnpm build"
CMD_LINT = "pnpm lint"
CMD_TEST = "pnpm test"
CMD_E2E = "pnpm test:e2e"  # empty if none detected


### Step 4: Workflow Config

Ask the user about workflow preferences:

1.⁠ ⁠*Dispatcher skill name*: What should the dispatcher skill be called? (default: ⁠ agile ⁠)

2.⁠ ⁠*Extra helper skills*: Do you want any helper skills?
   - ⁠ /review ⁠ — manual owner review (checkout PR, run dev server, open browser)
   - None
   - Custom (describe what you want)

3.⁠ ⁠*Branch naming convention*:
   - ⁠ issue-{number} ⁠ (default)
   - ⁠ {type}/{number}-{slug} ⁠ (e.g., ⁠ feat/42-add-login ⁠)
   - Custom pattern

4.⁠ ⁠*PR merge strategy*:
   - Squash (default)
   - Merge commit
   - Rebase

5.⁠ ⁠*Coverage threshold*: Minimum test coverage % for new code (default: 80%)

6.⁠ ⁠*Design-first workflow?*: Does this project have design/spec files that should be updated before implementation?
   - If yes: ask for the design files directory (e.g., ⁠ design/ ⁠, ⁠ docs/architecture/ ⁠)

7.⁠ ⁠*Final approver*: GitHub username for the ⁠ owner ⁠ step (optional — leave blank to skip owner review)

Store results:

DISPATCHER_NAME = "agile"
HELPER_SKILLS = ["review"]
BRANCH_PATTERN = "issue-{number}"
MERGE_STRATEGY = "squash"
COVERAGE_THRESHOLD = 80
DESIGN_FIRST = true
DESIGN_DIR = "design/"
APPROVER = "username"  # or empty


### Step 5: Compute & Confirm State Machine

Derive the label flow based on selected roles:

*Full team* (architect + dev + reviewer + qa):

Issue Created → [Architect] planning → ready
→ [Dev] wip → PR + review
→ [Reviewer] qa | changes → [Dev fixes] → review
→ [QA] owner | changes → [Dev fixes] → review
→ [Owner] merge

Labels: ⁠ planning ⁠, ⁠ ready ⁠, ⁠ wip ⁠, ⁠ review ⁠, ⁠ changes ⁠, ⁠ qa ⁠, ⁠ owner ⁠

*No architect* (dev + reviewer + qa):

Issue Created (ready) → [Dev] wip → PR + review
→ [Reviewer] qa | changes → [Dev fixes] → review
→ [QA] owner | changes → [Dev fixes] → review
→ [Owner] merge

Labels: ⁠ ready ⁠, ⁠ wip ⁠, ⁠ review ⁠, ⁠ changes ⁠, ⁠ qa ⁠, ⁠ owner ⁠

*Dev + reviewer*:

Issue Created (ready) → [Dev] wip → PR + review
→ [Reviewer] approved | changes → [Dev fixes] → review

Labels: ⁠ ready ⁠, ⁠ wip ⁠, ⁠ review ⁠, ⁠ changes ⁠, ⁠ approved ⁠

*Dev only*:

Issue Created (todo) → [Dev] in-progress → done

Labels: ⁠ todo ⁠, ⁠ in-progress ⁠, ⁠ done ⁠

If an approver is configured, always include the ⁠ owner ⁠ step at the end (even for dev+reviewer — replace ⁠ approved ⁠ with ⁠ owner ⁠).

Display the computed flow as an ASCII diagram and ask user to confirm:


Here's your workflow:

{ASCII diagram}

Labels: {list}

Does this look right? (yes / adjust)


### Step 6: Preview & Confirm

After all questions are answered, show the user a preview of what will be generated:

1. **Files that will be created**:
```
Files to generate:
  .claude/agents/architect.md    (if architect role selected)
  .claude/agents/dev.md
  .claude/agents/reviewer.md     (if reviewer role selected)
  .claude/agents/qa.md           (if qa role selected)
  .claude/agents/e2e.md          (if e2e subagent selected)
  .claude/agents/unittest.md     (if unittest subagent selected)
  .claude/skills/{DISPATCHER_NAME}/SKILL.md
  .claude/skills/review/SKILL.md (if /review helper selected)
  CLAUDE.md                      (updated with workflow section)
  + Board labels created on {BOARD_TYPE}
```

2. **The workflow diagram** — re-display the ASCII state machine from Step 5, now with their specific choices filled in (board type, role names, branch pattern, etc.)

3. **Prompt for confirmation**:

```
Ready to generate? (yes / adjust anything)
```

If the user wants to adjust, return to the relevant step. Otherwise, proceed to Phase 2.

---

## Phase 2: Generate Files

### Pre-flight Check

Before generating, check for existing setup:
⁠ bash
ls .claude/agents/ 2>/dev/null
ls .claude/skills/ 2>/dev/null
 ⁠

If ⁠ .claude/agents/ ⁠ already exists, ask: *Overwrite existing agents, skip generation, or abort?*

### Board Command Reference

Use this mapping when generating agent files. Always use the correct commands for the selected board:

| Operation | GitHub | Linear | Jira |
|-----------|--------|--------|------|
| List items | ⁠ gh issue list ⁠ | ⁠ linear issue list --team {BOARD_KEY} ⁠ | ⁠ jira issue list --project {BOARD_KEY} ⁠ |
| View item | ⁠ gh issue view {n} ⁠ | ⁠ linear issue view {id} ⁠ | ⁠ jira issue view {BOARD_KEY}-{n} ⁠ |
| Add label | ⁠ gh issue edit {n} --add-label {label} ⁠ | ⁠ linear issue update {id} --state {state} ⁠ | ⁠ jira issue transition {BOARD_KEY}-{n} --state {state} ⁠ |
| Remove label | ⁠ gh issue edit {n} --remove-label {label} ⁠ | (state replaces previous) | (transition replaces previous) |
| Assign | ⁠ gh issue edit {n} --add-assignee @me ⁠ | ⁠ linear issue update {id} --assignee @me ⁠ | ⁠ jira issue assign {BOARD_KEY}-{n} @me ⁠ |
| Create PR | ⁠ gh pr create ⁠ | ⁠ gh pr create ⁠ | ⁠ gh pr create ⁠ |
| List PRs | ⁠ gh pr list --label {label} ⁠ | ⁠ gh pr list --label {label} ⁠ | ⁠ gh pr list --label {label} ⁠ |
| View PR | ⁠ gh pr view {n} ⁠ | ⁠ gh pr view {n} ⁠ | ⁠ gh pr view {n} ⁠ |
| PR diff | ⁠ gh pr diff {n} ⁠ | ⁠ gh pr diff {n} ⁠ | ⁠ gh pr diff {n} ⁠ |
| PR review | ⁠ gh pr review {n} ⁠ | ⁠ gh pr review {n} ⁠ | ⁠ gh pr review {n} ⁠ |
| PR labels | ⁠ gh pr edit {n} --add-label / --remove-label ⁠ | ⁠ gh pr edit {n} --add-label / --remove-label ⁠ | ⁠ gh pr edit {n} --add-label / --remove-label ⁠ |
| PR checkout | ⁠ gh pr checkout {n} ⁠ | ⁠ gh pr checkout {n} ⁠ | ⁠ gh pr checkout {n} ⁠ |
| PR merge | ⁠ gh pr merge {n} --{MERGE_STRATEGY} --delete-branch ⁠ | ⁠ gh pr merge {n} --{MERGE_STRATEGY} --delete-branch ⁠ | ⁠ gh pr merge {n} --{MERGE_STRATEGY} --delete-branch ⁠ |

Key insight: PRs always live on GitHub/GitLab/Bitbucket regardless of task board. Board abstraction only affects issue/ticket commands.

For GitLab remotes, replace ⁠ gh ⁠ with ⁠ glab ⁠ for PR commands. For Bitbucket, note limitations and provide manual guidance.

### Generate Agents

For each role in ROLES, create ⁠ .claude/agents/{role}.md ⁠. Follow this structure for every agent:

⁠ markdown
# {Role} Agent

You are the {Role} for this project.

## Role
{What this agent does — one paragraph}

## Principles
{3-4 role-specific values, each with a brief explanation}

## Input
{How it receives work — board-specific commands to sync, claim, and read work items}

## Process
{Step-by-step numbered list of what the agent does}

## Commands
{Board CLI + build/test/lint commands relevant to this role}

## Guidelines
{Project-specific rules: design-first, coverage threshold, branch naming, patterns}

## Exit
When finished, report what was done, then exit Claude Code:
\ ⁠bash
kill $PPID
\```
```

Customize each agent dynamically based on all collected config. Below are the role-specific instructions for generating each agent:

#### Architect Agent
•⁠  ⁠*Role*: Evaluate issues and create implementation plans with acceptance criteria
•⁠  ⁠*Principles*: Simplicity first, maintainability, test coverage planning
•⁠  ⁠*Input*: Claims an issue (add ⁠ planning ⁠ label), reads it
•⁠  ⁠*Process*: Understand → Explore codebase → Simplify → Plan → Define tests → Publish (comment plan on issue, label ⁠ ready ⁠)
•⁠  ⁠*Commands*: Board commands for claiming/viewing/labeling issues. Codebase exploration commands.
•⁠  ⁠*Guidelines*: If DESIGN_FIRST, instruct to plan design file changes. Include COVERAGE_THRESHOLD. Reference project's existing patterns.
•⁠  ⁠*Output format*: Include a template for the architect's comment (design changes, implementation steps, required tests, acceptance criteria, complexity check)

#### Dev Agent
•⁠  ⁠*Role*: Implement issues per architect's plan, or fix PR change requests
•⁠  ⁠*Principles*: Simplicity, code quality, test coverage, follow existing patterns
•⁠  ⁠*Input*: Two modes — issue with ⁠ ready ⁠ label (new work) or PR with ⁠ changes ⁠ label (fixes). Include claim/checkout commands.
•⁠  ⁠*Process (issue)*: Claim → Branch (⁠ BRANCH_PATTERN ⁠) → Read plan → Design first (if enabled) → Test first → Implement → Validate → PR with ⁠ review ⁠ label
•⁠  ⁠*Process (PR fixes)*: Checkout → Read comments → Fix → Validate → Push → Label ⁠ review ⁠
•⁠  ⁠*Commands*: Branch creation using BRANCH_PATTERN, all build/lint/test commands (CMD_BUILD, CMD_LINT, CMD_TEST), PR creation, label management. Include coverage check against COVERAGE_THRESHOLD.
•⁠  ⁠*Guidelines*: If DESIGN_FIRST, update design files before implementation. Include self-review checklist.

#### Reviewer Agent
•⁠  ⁠*Role*: Review PRs for quality, simplicity, and compliance with plan
•⁠  ⁠*Principles*: Guardian of simplicity, quality bar, consistency over improvement
•⁠  ⁠*Input*: PR with ⁠ review ⁠ label. Sync, view PR, read diff, get linked issue for plan.
•⁠  ⁠*Process*: Review against checklist → Approve (label ⁠ qa ⁠ or ⁠ approved ⁠ or ⁠ owner ⁠ depending on flow) or Request changes (label ⁠ changes ⁠)
•⁠  ⁠*Commands*: PR view/diff/review commands, label management
•⁠  ⁠*Guidelines*: Include review checklist (plan compliance, simplicity, code quality, test coverage, maintainability). Include project-specific checks.

#### QA Agent
•⁠  ⁠*Role*: Final quality gate. Run all tests, validate coverage, decide pass/fail.
•⁠  ⁠*Principles*: Coverage is mandatory, automation is truth, clean merges
•⁠  ⁠*Input*: PR with ⁠ qa ⁠ label. Sync, checkout, install dependencies.
•⁠  ⁠*Process*: If HAS_SUBAGENTS, spawn e2e and unittest subagents in parallel using Agent tool. Otherwise, run tests directly. Collect results. Pass → label ⁠ owner ⁠ (or merge if no approver). Fail → label ⁠ changes ⁠ with details.
•⁠  ⁠*Commands*: All test commands, PR comment/label commands. If APPROVER set, request review from APPROVER.

#### E2E Subagent (if selected)
•⁠  ⁠*Role*: Run end-to-end tests
•⁠  ⁠*Process*: Run CMD_E2E, report pass/fail with details
•⁠  ⁠*Commands*: CMD_E2E and any setup commands (e.g., starting dev server)

#### Unittest Subagent (if selected)
•⁠  ⁠*Role*: Run unit tests, check coverage, validate build and lint
•⁠  ⁠*Process*: Run CMD_BUILD, CMD_LINT, CMD_TEST with coverage. Check coverage against COVERAGE_THRESHOLD. Report pass/fail.
•⁠  ⁠*Commands*: CMD_BUILD, CMD_LINT, CMD_TEST with coverage flags

### Generate Dispatcher

Create ⁠ .claude/skills/{DISPATCHER_NAME}/SKILL.md ⁠ with this structure:

⁠ yaml
---
name: {DISPATCHER_NAME}
description: Automated agile workflow dispatcher. Checks {BOARD_TYPE} for work items and dispatches to specialized agents.
allowed-tools: {computed from board — always include Bash(*), Read, Agent, Task}
---
 ⁠

Body structure — follow the pattern of the existing ⁠ /agile ⁠ skill:

1.⁠ ⁠*Execution note*: Process exactly ONE work item, then cleanup and EXIT. Do not loop unless ⁠ --loop ⁠ is specified.

2.⁠ ⁠*Step 1: Check Work Queues* — Priority-ordered checks, one per role:
   - For each role in ROLES, generate the appropriate board query command
   - Priority order: architect (new issues) → reviewer (PRs with review) → qa (PRs with qa) → dev PR fixes (PRs with changes) → dev new work (issues with ready/todo)
   - Only include queues for selected roles
   - Use the board command reference for the correct CLI commands

3.⁠ ⁠*Step 2: Dispatch to Agent* — Table mapping queue → agent file → command

4.⁠ ⁠*Step 3: Execute* — Read agent file, follow its process

5.⁠ ⁠*Step 4: Cleanup and Exit*:
   ⁠ bash
   # Return to main branch if on a feature branch
   CURRENT_BRANCH=$(git branch --show-current)
   if [ "$CURRENT_BRANCH" != "main" ]; then
     git checkout main
   fi
   # Delete local branches that have been merged
   git fetch --prune
   git branch -vv | grep ': gone]' | awk '{print $1}' | xargs -r git branch -D
   # Pull latest
   git pull origin main
    ⁠
   Report completion and ⁠ kill $PPID ⁠.

6.⁠ ⁠*Agents table* — list all agent files and their responsibilities

7.⁠ ⁠*Idle State* — report queue counts and exit with ⁠ kill $PPID ⁠

8.⁠ ⁠*Loop Mode* — document ⁠ --loop ⁠ for continuous processing

### Generate Helper Skills

If user selected ⁠ /review ⁠ helper skill, create ⁠ .claude/skills/review/SKILL.md ⁠:
•⁠  ⁠Find PR with ⁠ owner ⁠ label
•⁠  ⁠Checkout the branch
•⁠  ⁠Open PR in browser
•⁠  ⁠Run dev server and open localhost
•⁠  ⁠Report status with approve/reject commands

For any custom helper skills the user described, generate an appropriate SKILL.md following the same frontmatter + body pattern.

### Update CLAUDE.md

Check if CLAUDE.md exists. If it has an existing workflow section (look for ⁠ ## Agile Workflow ⁠ or ⁠ ## Workflow ⁠), ask whether to *replace* or *append*.

Append (or replace) a workflow section:

⁠ markdown
## Agile Workflow

Run `/{DISPATCHER_NAME}` to enter the automated workflow. It checks {board} and wears the appropriate hat:

| Priority | Role | Trigger | Action |
|----------|------|---------|--------|
{one row per role with priority, trigger condition, and action}

### Labels

{description of each label and what it means}

### Flow

\ ⁠
{ASCII state machine diagram}
\```
```

Also ensure CLAUDE.md includes:
•⁠  ⁠The build/test/lint commands if not already present
•⁠  ⁠The ⁠ **Agents**: When an agent finishes its task, report completion then run \`kill $PPID\ ⁠ to exit Claude Code.` instruction

### Create Board Labels

Generate and run CLI commands to create labels with colors and descriptions.

*GitHub*:
⁠ bash
# For each label in the state machine
gh label create "{label}" --color "{color}" --description "{description}" --force
 ⁠

Color scheme:
| Label | Color | Description |
|-------|-------|-------------|
| ⁠ planning ⁠ | ⁠ c5def5 ⁠ (light blue) | Architect actively working on spec |
| ⁠ ready ⁠ | ⁠ 0e8a16 ⁠ (green) | Spec complete, ready for dev |
| ⁠ wip ⁠ | ⁠ fbca04 ⁠ (yellow) | Dev actively implementing |
| ⁠ review ⁠ | ⁠ 1d76db ⁠ (blue) | PR waiting for code review |
| ⁠ changes ⁠ | ⁠ e11d48 ⁠ (red) | Changes requested, back to dev |
| ⁠ qa ⁠ | ⁠ d876e3 ⁠ (purple) | Approved, waiting for QA validation |
| ⁠ owner ⁠ | ⁠ f9d0c4 ⁠ (peach) | QA passed, waiting for owner merge |
| ⁠ approved ⁠ | ⁠ 0e8a16 ⁠ (green) | Review approved (used in dev+reviewer flow) |
| ⁠ todo ⁠ | ⁠ c5def5 ⁠ (light blue) | Ready for work (used in dev-only flow) |
| ⁠ in-progress ⁠ | ⁠ fbca04 ⁠ (yellow) | Dev working (used in dev-only flow) |
| ⁠ done ⁠ | ⁠ 0e8a16 ⁠ (green) | Completed (used in dev-only flow) |

Only create labels that are part of the computed state machine.

*Linear*: Map labels to native workflow states where possible. Provide manual setup guidance for custom states:

Linear uses native workflow states instead of labels.
Please ensure your team "{BOARD_KEY}" has these states: {list states}
Linear default states (Backlog, Todo, In Progress, Done, Canceled) may cover some of these.
Custom states can be added in Settings > Teams > {team} > Workflow.


*Jira*: Similar — map to native transitions and provide guidance:

Jira uses workflow transitions instead of labels.
Please ensure your project "{BOARD_KEY}" workflow includes these statuses: {list statuses}
Custom statuses can be configured in Project Settings > Workflows.


---

## Phase 3: Verification

After generating everything:

1.⁠ ⁠*List all created files*:
⁠ bash
echo "=== Created files ==="
find .claude/agents -name "*.md" -type f 2>/dev/null | sort
find .claude/skills -name "SKILL.md" -type f 2>/dev/null | sort
echo "CLAUDE.md (updated)"
 ⁠

2.⁠ ⁠*Show the state machine diagram* — display the ASCII flow one more time

3.⁠ ⁠*Suggest next steps*:

Setup complete! Here's what was created:
{file list}

Next steps:
1. Run /{DISPATCHER_NAME} to test the workflow
2. Create a GitHub issue to see the full cycle
3. For continuous mode: claude --permission-mode="bypassPermissions" "/{DISPATCHER_NAME} --loop"


4.⁠ ⁠*Confirm board labels* — if GitHub labels were created, verify:
⁠ bash
gh label list --json name,color,description --jq '.[] | select(.name == "planning" or .name == "ready" or ...)'
 ⁠

For Linear/Jira, remind the user to verify native states were configured.

---

## Edge Cases

•⁠  ⁠*CLI not installed*: Detect early in Step 1, offer install guidance, let user choose to continue
•⁠  ⁠*Existing CLAUDE.md workflow section*: Ask replace or append (never silently overwrite)
•⁠  ⁠*Re-running /agentify*: Detect existing ⁠ .claude/agents/ ⁠, ask overwrite/skip/abort
•⁠  ⁠*Non-GitHub PR platform*: Detect from git remote, adapt PR commands (⁠ glab ⁠ for GitLab, manual guidance for Bitbucket)
•⁠  ⁠*No git remote*: Warn user, default to GitHub, let them override
•⁠  ⁠*Monorepo*: If multiple package.json/go.mod etc found, ask which is the primary project root
•⁠  ⁠*Linear/Jira native states*: Map to built-in states where possible, provide manual setup instructions for custom states
•⁠  ⁠*No test command detected*: Ask user for test commands, or mark as "no tests configured" and skip coverage checks in agents