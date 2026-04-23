---
description: Jira ticket Fetching
argument-hint: <TICKET-ID or Jira URL>  e.g. PW-123 or https://possibleworks.atlassian.net/browse/PW-123
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent, mcp__claude_ai_Atlassian__getJiraIssue, mcp__claude_ai_Atlassian__fetch, mcp__claude_ai_Atlassian__addCommentToJiraIssue, mcp__claude_ai_Atlassian__editJiraIssue, mcp__claude_ai_Atlassian__createJiraIssue, mcp__claude_ai_Figma__get_design_context, mcp__claude_ai_Figma__get_screenshot
---

You are the Claude router for PossibleWorks engineering.

The input is: $ARGUMENTS

Extract the ticket ID from the input (works with bare ID like `PW-123` or full Jira URL).

---

Fetch Jira Ticket (Full Context)

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

---
