---
description: Jira ticket → Claude router → plan → code → review → PR → Coolify preview
argument-hint: <TICKET-ID or Jira URL>  e.g. PW-123 or https://possibleworks.atlassian.net/browse/PW-123
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent, mcp__claude_ai_Atlassian__getJiraIssue, mcp__claude_ai_Atlassian__addCommentToJiraIssue, mcp__claude_ai_Atlassian__editJiraIssue, mcp__claude_ai_Atlassian__createJiraIssue
---

You are the Claude router for PossibleWorks engineering.

The input is: $ARGUMENTS

Extract the ticket ID from the input (works with bare ID like `PW-123` or full Jira URL).

---

## REPOS

| Repo | Path | GitHub | Dev Branch |
|------|------|--------|------------|
| Frontend | `/Users/sudheer7781/Documents/pw-react-client-v3` | `PossibleWorks/pw-react-client-v3` | `dev` |
| Backend | `/Users/sudheer7781/Documents/pw-server-v3` | `PossibleWorks/pw-server-v3` | `coolify-dev-v3` |
| AI Server | `/Users/sudheer7781/Documents/pw-ai-server` | `PossibleWorks/pw-ai-server` | `dev` |
| Notifications | `/Users/sudheer7781/Documents/pw-notifications` | `PossibleWorks/pw-notifications` | `notif_dev` |
| AI Cron Server | `/Users/sudheer7781/Documents/ai-cron-server` | `PossibleWorks/ai-cron-server` | `dev` |
| Cron Jobs | `/Users/sudheer7781/Documents/pw-cron-jobs` | `PossibleWorks/pw-cron-jobs` | `jobs_dev` |

---

## STEP 1 — Fetch Jira Ticket (Full Context)

Use the `atlassian` MCP to fetch the full ticket for the extracted ID from `possibleworks.atlassian.net`.

### 1a — Core Ticket Data

Call `mcp__claude_ai_Atlassian__getJiraIssue` with:
- `issueIdOrKey`: the extracted ticket ID
- `fields`: `["summary", "description", "issuetype", "status", "assignee", "reporter", "comment", "attachment", "subtasks"]`
- `responseContentFormat`: `"markdown"`

IMPORTANT: Always pass the `fields` array above so that `attachment` and `comment` are included in the response. Without this, attachments will be missing.

Extract and display:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Ticket  : <ID> — <Summary>
  Type    : <Bug | Story | Task | Sub-task>
  Status  : <status>
  Reporter: <name>
  Assignee: <name>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Description:
<full description>

Acceptance Criteria:
<AC items, one per line>

Figma Links:
<list any figma.com URLs found in description/comments — or "None">

Attachments:
<list all attachments with filename, type: screenshot | PDF | CSV | XLSX | doc | other>

Sub-tasks:
<list sub-task IDs + summaries — or "None">

Comments:
<count> comments found
```

### 1b — Fetch All Comments

Read comments from `fields.comment.comments` in the already-fetched response. No additional API call needed.

Extract:
- **UI feedback / bug reports** — broken behaviour, wrong layout, wrong colour, missing elements
- **Screenshot references** — any comment mentioning a screenshot or containing an image
- **Additional context** — clarifications from reporter, PM, or designer
- **Decision log** — "agreed to do X instead of Y" type comments

Display:

```
─── Comments ────────────────────────────────────────
  [<author> — <date>]
  <comment body — full text, do not truncate>
─────────────────────────────────────────────────────
```

### 1c — Fetch Attachments (screenshots, PDFs, docs)

From the ticket's `fields.attachment` array, collect every attachment.

**IMPORTANT — How to fetch attachment content:**

The `mcp__claude_ai_Atlassian__fetch` tool does NOT work for attachment download URLs.
Instead, use authenticated `curl` via the Bash tool. Credentials are available as environment variables:
- `$ATLASSIAN_EMAIL`
- `$ATLASSIAN_API_TOKEN`

The attachment content URL is in the `content` field of each attachment object.

**For Screenshots / images (PNG, JPG, GIF, WEBP):**

```bash
curl -s -u "$ATLASSIAN_EMAIL:$ATLASSIAN_API_TOKEN" "<content_url>" -L -o /tmp/<filename>
```
Then use the Read tool to view `/tmp/<filename>` as an image.
Describe what is shown: screen name, UI element highlighted, error state, layout issue, red annotations, arrows.
If it is a bug screenshot — note exactly what looks wrong visually.
Label it: `[Screenshot: <filename> — <what it shows>]`

**For CSVs:**

```bash
curl -s -u "$ATLASSIAN_EMAIL:$ATLASSIAN_API_TOKEN" "<content_url>" -L
```
Read and summarise all rows/columns.
Label it: `[CSV: <filename> — <summary>]`

**For XLSX files:**

```bash
curl -s -u "$ATLASSIAN_EMAIL:$ATLASSIAN_API_TOKEN" "<content_url>" -L -o /tmp/<filename>
pip3 install openpyxl -q
python3 -c "
import openpyxl
wb = openpyxl.load_workbook('/tmp/<filename>')
for sheet in wb.sheetnames:
    ws = wb[sheet]
    print(f'=== Sheet: {sheet} ===')
    for row in ws.iter_rows(values_only=True):
        if any(cell is not None for cell in row):
            print(row)
    print()
"
```
Summarise all sheets and their content.
Label it: `[XLSX: <filename> — <summary>]`

**For PDFs:**

```bash
curl -s -u "$ATLASSIAN_EMAIL:$ATLASSIAN_API_TOKEN" "<content_url>" -L -o /tmp/<filename>
```
Then use the Read tool to read `/tmp/<filename>` as a PDF.
Summarise the content: requirements, flow diagrams, spec pages.
Label it: `[PDF: <filename> — <summary>]`

**For other docs (DOCX, TXT):**

Download via curl and read/summarise.
Label it: `[Doc: <filename> — <summary>]`

Display:

```
─── Attachments ─────────────────────────────────────
  [Screenshot: error-state.png]
  Shows the initiative card in error state — red border
  visible, "Failed" badge overlapping the title text.

  [CSV: finance-spec.csv]
  Lists all 6 finance modules and their doctypes.

  [XLSX: finance-spec.xlsx]
  12 sheets covering AP/AR field specs, role access, and open bugs.
─────────────────────────────────────────────────────
```

### 1d — Fetch Sub-tasks (if any)

For each sub-task listed on the parent ticket, call `mcp__claude_ai_Atlassian__getJiraIssue` with the same `fields` array as step 1a.

Extract for each sub-task:
- Summary, status, assignee
- Full description and AC
- Comments (repeat 1b for each sub-task)
- Attachments (repeat 1c for each sub-task)

Display:

```
─── Sub-task: <ID> — <Summary> ──────────────────────
  Status   : <status>
  Assignee : <name>
  Description: <full text>
  AC: <items>
  Comments: <if any>
  Attachments: <if any — describe fully>
─────────────────────────────────────────────────────
```

### 1e — Synthesise All Context

After fetching everything, produce a single consolidated understanding block:

```
─── Synthesised Understanding ───────────────────────
  Core requirement: <1-2 sentences — what needs to be built/fixed>

  UI issues to fix (from screenshots/comments):
  - <exact visual problem 1 and where it appears>
  - <exact visual problem 2>

  Additional constraints from comments:
  - <any PM/designer clarification that overrides description>

  Sub-task scope:
  - <ID>: <what it covers — include or exclude from this implementation>

  Open questions (if any):
  - <anything ambiguous — will be surfaced in plan's Out of Scope>
─────────────────────────────────────────────────────
```

> **Think Before Coding** — Before moving to the plan, explicitly list every assumption you are making about this ticket. Unresolved assumptions become Open Questions in the plan's Out of Scope. Never silently guess.

```
─── Assumption Audit ────────────────────────────────
  Assuming: <e.g. "the existing AP socket event pattern is reused for AR">
  Assuming: <e.g. "Sales Invoice outstanding amount comes from Frappe, not computed client-side">
  Ambiguous: <anything the ticket does not clearly specify — flag it>
─────────────────────────────────────────────────────
```

Do NOT stop here. Continue automatically to STEP 2.

---

## STEP 2 — Fetch Figma Designs (if any)

If Figma links were found in the ticket, use the `figma` MCP to fetch each one.

For each design extract:
- Screen/component names
- Layout and interaction notes
- Any visible text, labels, states, colors

Display a design summary. If no Figma links → skip and note "No designs attached".

---

## STEP 3 — Route to Repos + Deep Codebase Exploration

### 3a — Route

Read the ticket summary, description, and AC. Decide which repos are affected:

**Routing rules:**
- UI changes, new screens, components, chat cards → **Frontend**
- API endpoints, socket events, DB/Prisma, services → **Backend**
- AI prompts, LLM logic, embeddings, agent flows → **AI Server**
- Push notifications, email notifications, in-app alerts → **Notifications**
- Scheduled AI jobs, initiative/goal automation → **AI Cron Server**
- Scheduled data jobs, reports, cron triggers → **Cron Jobs**
- A feature can affect multiple repos — include all that apply

Display routing result:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Repos affected:
  ✅ Frontend        (pw-react-client-v3)
  ✅ Backend         (pw-server-v3)
  ⬜ AI Server       (not needed)
  ⬜ Notifications   (not needed)
  ⬜ AI Cron Server  (not needed)
  ⬜ Cron Jobs       (not needed)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 3b — Deep Codebase Exploration (ALL affected repos, run in PARALLEL)

**Use the Agent tool to spawn one `Explore` subagent per affected repo, all at the same time.**
Do NOT explore repos serially — launch all subagents in a single message so they run in parallel.

For each affected repo, the subagent prompt must instruct it to:

1. **Read CLAUDE.md** at root or `.claude/CLAUDE.md` — conventions, patterns, forbidden patterns
2. **Grep for keywords from the ticket** (feature name, entity name, domain terms) to find where related code lives
3. **Read the relevant existing files** — service, controller, component, socket handlers, types for the touched domain
4. **Read the data model** — Prisma schema or DB models if backend is affected
5. **Read socket event files** — `src/models/enums/event-names-enum.ts` (Frontend) + backend socket handlers
6. **Find cross-repo contracts** — API endpoints, socket event names, request/response payload shapes
7. **Find the closest reference implementation** — the most similar existing feature to use as a pattern

Each subagent prompt example:

```
Explore the codebase at /Users/.../pw-react-client-v3 for a ticket about "<ticket summary>".

Find and read:
- .claude/CLAUDE.md
- Any files related to "<domain keyword>" (grep across src/)
- src/constants/utils.ts (CARD_EVENTS, ACTION_STATUS)
- src/models/enums/event-names-enum.ts (EVENTNAMES_ENUM)
- src/components/molecules/SingleScreen/Chat/hooks/useSocketEvents.ts
- The existing component/card closest to this feature

Return: file paths found, relevant code snippets, existing socket events, and the best reference pattern to follow.
```

Wait for ALL subagents to complete, then consolidate their findings.

Display a brief exploration summary:
```
─── Codebase Exploration (parallel subagents) ───────────
  Backend  [Explore subagent]:
  - Found: src/services/InitiativeService.ts (existing logic)
  - Found: src/controllers/InitiativeController.ts
  - Prisma model: Initiative (fields: id, title, endDate, ...)
  - Existing socket events: initiative_update, initiative_create

  Frontend  [Explore subagent]:
  - Found: src/components/molecules/InitiativeCard/index.tsx
  - EVENTNAMES_ENUM already has: INITIATIVE_UPDATE
  - CARD_EVENTS already has: initiative_update
  - useSocketEvents.ts: handles initiative_update at line 142
  - Reference card: ApplyLeaveCard (closest pattern match)
─────────────────────────────────────────────────────────
```

---

## STEP 4 — Generate Implementation Plan

Using everything learned in STEP 3, generate a concrete plan with exact file paths (confirmed by exploration, not guessed):

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  IMPLEMENTATION PLAN — <TICKET-ID>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Branches:
  Frontend        → feat/<ticket-id>-<slug>  (from dev)
  Backend         → feat/<ticket-id>-<slug>  (from coolify-dev-v3)
  Notifications   → feat/<ticket-id>-<slug>  (from notif_dev)
  AI Cron Server  → feat/<ticket-id>-<slug>  (from dev)
  Cron Jobs       → feat/<ticket-id>-<slug>  (from jobs_dev)

─── Cross-Repo Contract ──────────────────────────────────
  Socket event: <event_name>
  Payload shape (Backend → Frontend):
    { field1: type, field2: type, ... }
  API endpoint (if REST): <METHOD /path>
  Request body: { ... }
  Response shape: { ... }

─── Backend Changes (pw-server-v3) ──────────────────────
1. <confirmed file path from exploration>
   - <exactly what to add/change and why>
2. prisma/schema.prisma  (if DB changes needed)
   - <field/model change>
...

─── Frontend Changes (pw-react-client-v3) ───────────────
1. <confirmed file path from exploration>
   - <exactly what to add/change and why>
2. src/models/enums/event-names-enum.ts
   - Add: <NEW_EVENT = 'new_event'>  (only if new event needed)
3. src/constants/utils.ts
   - Add: <NEW_EVENT: 'New Event Label'>  (only if new event needed)
4. src/components/molecules/SingleScreen/Chat/hooks/useSocketEvents.ts
   - Register handler for <event> (only if new socket handler needed)
...

─── Notifications Changes (pw-notifications) ────────────
1. <confirmed file path>
   - <what and why>
...

─── AI Cron Server Changes (ai-cron-server) ─────────────
1. <confirmed file path>
   - <what and why>
...

─── Cron Jobs Changes (pw-cron-jobs) ────────────────────
1. <confirmed file path>
   - <what and why>
...

─── Acceptance Criteria → Testable Outcomes ─────────────
  AC 1: <text>
    → Verified when: <concrete observable outcome — what you see/get when it works>
  AC 2: <text>
    → Verified when: <concrete observable outcome>

─── Tradeoffs ────────────────────────────────────────────
  <file or area>: chose <approach A> over <approach B> because <reason>

─── Out of Scope ─────────────────────────────────────────
- <anything NOT being done and why>
- <unresolved assumptions from Assumption Audit that need PM/designer input>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Then ask:
> **Does this plan look correct?**
> Reply with changes/additions — or say **"looks good"** to start coding.

⛔ STOP HERE. Do not write any code until the user approves.

---

## STEP 5 — Revise Plan (if needed)

If the user requests changes, update the plan and show it again.
Repeat until they say **"looks good"** / **"approved"** / **"yes"** / **"proceed"**.

---

## STEP 6 — Create Feature Branches

For each affected repo, checkout its dev branch, pull latest, then create the feature branch:

```bash
# Frontend
cd /Users/sudheer7781/Documents/pw-react-client-v3
git checkout dev && git pull origin dev
git checkout -b feat/<ticket-id>-<slug>

# Backend
cd /Users/sudheer7781/Documents/pw-server-v3
git checkout coolify-dev-v3 && git pull origin coolify-dev-v3
git checkout -b feat/<ticket-id>-<slug>

# AI Server (if needed)
cd /Users/sudheer7781/Documents/pw-ai-server
git checkout dev && git pull origin dev
git checkout -b feat/<ticket-id>-<slug>

# Notifications (if needed)
cd /Users/sudheer7781/Documents/pw-notifications
git checkout notif_dev && git pull origin notif_dev
git checkout -b feat/<ticket-id>-<slug>

# AI Cron Server (if needed)
cd /Users/sudheer7781/Documents/ai-cron-server
git checkout dev && git pull origin dev
git checkout -b feat/<ticket-id>-<slug>

# Cron Jobs (if needed)
cd /Users/sudheer7781/Documents/pw-cron-jobs
git checkout jobs_dev && git pull origin jobs_dev
git checkout -b feat/<ticket-id>-<slug>
```

---

## STEP 7 — Write the Code

For each affected repo, implement exactly what the approved plan says:

- Read every file before editing — never edit blindly
- Follow the repo's CLAUDE.md conventions strictly
- Never use raw MUI — use PWButton, PWTypography, PWIcon (Frontend)
- Always register socket.off() for every socket.on() (Frontend/Backend)
- No comments, docstrings, or extra logging unless the ticket requires it
- No refactoring beyond what the plan says
- Cross-repo payloads must match exactly — if Backend sends `{ endDate: string }`, Frontend must read `endDate`

> **Simplicity First** — The best solution is the smallest one that satisfies the ACs.
> - Prefer editing an existing file over creating a new one
> - Do not create helpers, hooks, or components used in only one place
> - Do not add configuration, feature flags, or fallbacks for scenarios that don't exist yet
> - If you find yourself building something for hypothetical future use — delete it

> **Surgical Changes** — Touch only what the plan lists. Nothing else.
> - Do not clean up surrounding code, even if it looks messy
> - Do not rename, reorder, or reformat anything outside your diff
> - If you spot a pre-existing bug — add it to the PR description as "Observed but out of scope", do not fix it

After all changes in a repo, type-check:
- **Frontend**: `npx tsc -b` in `pw-react-client-v3` — fix ALL errors before proceeding
- **Backend**: check its CLAUDE.md for the type-check command — fix ALL errors
- **Others**: verify no syntax errors

---

## STEP 7.5 — Code Review (Mandatory — do not skip)

After writing all code and passing type-check, perform a thorough self-review of every changed file before committing.

For each repo with changes, run:
```bash
cd <repo-path>
git diff HEAD
```

Review each diff against these criteria:

**Goal-Driven Execution — AC verification (do this first):**
- [ ] Go through every AC from the Testable Outcomes in the plan — mark each ✅ or ❌
- [ ] For each ❌ — fix the gap before continuing. Do not commit with a failing AC
- [ ] Every AC must be independently verifiable, not assumed correct because code was written

**Correctness:**
- [ ] Does the code fully implement every AC from the ticket?
- [ ] Are all edge cases handled (null/undefined, empty arrays, loading states, error states)?
- [ ] Are socket payloads shaped correctly and matching what the other repo sends/expects?
- [ ] Are async operations awaited correctly? No unhandled promise rejections?

**Surgical Changes — scope check:**
- [ ] Does every changed file appear in the approved plan? If not — revert the extra change
- [ ] Is any surrounding code refactored, cleaned up, or reformatted that wasn't in the plan? If yes — revert it
- [ ] Are there any new files, helpers, or abstractions not required by the plan? If yes — delete them

**Conventions (per CLAUDE.md):**
- [ ] Frontend: No raw MUI — only PWButton, PWTypography, PWIcon atoms used?
- [ ] Frontend: Every socket.on() has a matching socket.off() in cleanup?
- [ ] Frontend: No duplicate refreshChatUsers() calls?
- [ ] Backend: Follows existing service/controller pattern exactly?
- [ ] No comments or docstrings added unless logic is non-obvious?
- [ ] No console.log left in code?

**Simplicity check:**
- [ ] No new file created where an existing file could have been edited?
- [ ] No helper/hook/component built for a single call-site?
- [ ] No unused variables, dead code, or speculative logic?

**If any issue is found** — fix it immediately before committing. Do not note it and move on.

Display a review summary:
```
─── Code Review ─────────────────────────────────────────
  Backend:
  ✅ All ACs covered
  ✅ Null checks present for optional fields
  ✅ Follows service pattern
  🔧 Fixed: missing await on prisma call in InitiativeService.ts

  Frontend:
  ✅ PWButton/PWTypography used throughout
  ✅ socket.off() cleanup present
  ✅ No raw MUI
  🔧 Fixed: useEffect dependency array was missing `initiativeId`
  ✅ Type-check passes
─────────────────────────────────────────────────────────
```

---

## STEP 8 — Commit and Push

For each repo with changes:

```bash
# Stage only the changed files explicitly — never git add . or git add -A
git add <specific files>

git commit -m "feat(<ticket-id>): <concise description>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"

git push -u origin feat/<ticket-id>-<slug>
```

---

## STEP 9 — Raise PRs

For each repo use `gh pr create` targeting the correct dev branch:

| Repo | PR target branch |
|------|-----------------|
| Frontend | `dev` |
| Backend | `coolify-dev-v3` |
| AI Server | `dev` |
| Notifications | `notif_dev` |
| AI Cron Server | `dev` |
| Cron Jobs | `jobs_dev` |

**Title:** `feat(<ticket-id>): <Jira summary>`

**Body:**
```markdown
## Jira Ticket
[<TICKET-ID>](https://possibleworks.atlassian.net/browse/<TICKET-ID>) — <Summary>

## What Changed
- <bullet per file/area changed>

## Figma Design
<link or N/A>

## Acceptance Criteria
- [ ] <AC 1>
- [ ] <AC 2>

## Test Plan
- [ ] <manual step 1>
- [ ] <manual step 2>

🤖 Generated with [Claude Code](https://claude.ai/claude-code)
```

---

## STEP 10 — Final Summary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅  <TICKET-ID> — Done
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Branches created:
  Frontend        → feat/<ticket-id>-<slug>
  Backend         → feat/<ticket-id>-<slug>
  Notifications   → feat/<ticket-id>-<slug>  (if applicable)
  AI Cron Server  → feat/<ticket-id>-<slug>  (if applicable)
  Cron Jobs       → feat/<ticket-id>-<slug>  (if applicable)

  Pull Requests:
  Frontend        → <PR URL>
  Backend         → <PR URL>
  Notifications   → <PR URL>  (if applicable)

  Code Review: ✅ passed (all issues fixed before commit)

  Coolify will auto-deploy coolify-dev-v3 on merge.
  Preview URL will appear in Coolify dashboard after merge.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## HARD RULES

- Never write code before plan is approved
- Plan must be based on actual file reads — never guess file paths
- Never use `git add .` or `git add -A` — always stage specific files
- Never force push
- Never skip type-check before committing
- Never skip the code review (STEP 7.5) — it is mandatory
- Never commit .env or secret files
- If push is rejected — investigate, do not force
- Cross-repo payloads (socket events, API responses) must be verified to match on both ends before committing
