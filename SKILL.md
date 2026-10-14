---
name: good-morning
description: Daily work briefing that shows all tasks and priorities for the day. Use when the user says "good morning", "morning briefing", "daily standup", "what's on my plate", "what do I need to do today", or asks for a summary of their work day.
---

# Good Morning - Daily Work Briefing

Comprehensive daily briefing from Jira, GitLab, Calendar, Tasks, Gmail, Drive, and Chat.

## CLI Binaries

Detect platform and use the correct binary:

```bash
# Detect OS and architecture
OS=$(uname -s | tr '[:upper:]' '[:lower:]')  # linux, darwin
ARCH=$(uname -m)
case "$ARCH" in
  x86_64) ARCH="amd64" ;;
  aarch64|arm64) ARCH="arm64" ;;
esac

# Binary paths
JIRA="{{SKILL_ROOT}}/../jira-cli/bin/jira-cli-${OS}-${ARCH}"
GL="{{SKILL_ROOT}}/../gitlab-cli/bin/gitlab-cli-${OS}-${ARCH}"
GW="{{SKILL_ROOT}}/../workspace-cli/bin/workspace-cli-${OS}-${ARCH}"
```

| Platform | OS | ARCH | Example binary |
|----------|-----|------|----------------|
| macOS M1/M2 | darwin | arm64 | jira-cli-darwin-arm64 |
| macOS Intel | darwin | amd64 | jira-cli-darwin-amd64 |
| Linux x64 | linux | amd64 | jira-cli-linux-amd64 |
| Linux ARM | linux | arm64 | jira-cli-linux-arm64 |

Aliases below: `JIRA`, `GL`, `GW`

## Execution

Run these in parallel for speed:

### 1. Jira Tickets
```bash
JIRA mine                    # All assigned, unresolved
```

### 2. GitLab MRs + Pipeline Status
```bash
GL mr list --state=opened --mine        # Your open MRs
GL mr list --state=opened --reviewer-me # MRs you need to review
GL status                               # Current branch pipeline status
```

For each MR, check pipeline and unresolved discussions:
```bash
GL mr show <iid>  # Includes pipeline_status and discussion counts
```

### 3. Today's Meetings
```bash
GW calendar list --today
```

### 4. Google Tasks
```bash
GW tasks lists list --output json       # Get task list IDs
GW tasks list <tasklist-id> --output json
```

### 5. Unread Emails
```bash
GW gmail list --unread --limit 10 --output json
```

### 6. Google Chat — Action Items & @Mentions

Google Chat has a hidden **notification DM feed** (`spaces/EXAMPLE_NOTIFICATION_FEED`) that aggregates all action items assigned to you, @mentions in documents, document shares, and comment notifications across Google Workspace. This is the most reliable source for finding what people need from you.

#### Step 1: Scan the notification feed
```bash
GW chat messages list spaces/EXAMPLE_NOTIFICATION_FEED --limit 30 --output json
```

#### Step 2: Filter for actionable items
From the results, focus on messages matching these patterns (ignore "shared X with you" and "resolved a comment"):
- `"assigned an action item to you"` — someone assigned you a task in a Google Doc/Sheet/Slides
- `"mentioned you"` — someone @mentioned you in a document
- `"replied to a comment"` / `"commented on"` — ongoing comment threads you're part of

#### Step 3: Resolve document links
For each actionable notification, extract the document name from the message text and search Google Drive to get the direct link:
```bash
GW drive list --query "name contains '<document name keywords>'" --output raw
```
The raw output includes the document URL in the `Link:` field.

#### Step 4: Scan chat spaces for direct @mentions in threads
The notification feed covers Google Docs/Sheets, but direct @mentions in **chat threads** require scanning spaces. Key spaces to check (customize per user):

```bash
# List spaces to find IDs
GW chat spaces list --limit 30 --output json

# For each relevant space, scan recent messages
GW chat messages list <space-name> --limit 40 --output json
```

Look for messages where the user's display name appears after an `@` in the text content. The notification feed user ID for the authenticated user can be found via:
```bash
GW chat messages list spaces/EXAMPLE_NOTIFICATION_FEED --limit 1 --output json
# Check annotations[].userMention.userName for the user's ID (e.g., users/EXAMPLE_USER_ID)
```

#### Step 5: Read thread context for @mentions
When a direct @mention is found in a chat thread, read the full thread to understand what's being asked:
- Use a browser-automation tool to open the thread URL and extract the conversation
- Thread URLs follow the pattern: `https://chat.google.com/room/<spaceId>/<threadId>/<messageId>?cls=10`

#### Classification of Chat Action Items

| Type | Priority | Action |
|------|----------|--------|
| Direct @mention with a question | **High** | Reply needed |
| Action item assigned in a doc | **Medium** | Open doc, resolve item |
| @mention in a doc comment | **Medium** | Review and respond |
| Comment reply on your thread | **Low/FYI** | Read, respond if needed |
| Document shared with you | **FYI** | Acknowledge if needed |
| Comment resolved | **FYI** | No action needed |

### 7. Recent Drive Comments (on files you own)
```bash
GW drive list --query "'me' in owners" --limit 10 --output json
# For each file, check comments:
GW comments list <file-id> --output json
```

## Smart Analysis

### Meeting Time Awareness
- Calculate time until each meeting from current time
- **⚠️ Starting soon** if within 30 minutes
- **🔴 In progress** if currently happening
- ~~Strikethrough~~ if already passed

### Task Prioritization
- **🔴 Overdue** if due date < today
- **⚠️ Due today** if due date = today
- Sort by due date ascending

### Jira Prioritization (Priority Score)

Calculate a priority score for each ticket to rank them. Higher score = higher priority.

| Factor | Points | Description |
|--------|--------|-------------|
| **Priority field** | P1=40, P2=30, P3=20, P4=10 | Jira priority |
| **Age (days open)** | +1 per day, max 30 | Older tickets bubble up |
| **Days since update** | +2 per day idle, max 20 | Stale tickets need attention |
| **Status: Blocked** | +25 | Needs unblocking |
| **Status: In Review** | +15 | Close to completion |
| **Status: In Progress** | +10 | Active work |
| **Has due date approaching** | +20 if ≤3 days | Deadline pressure |
| **Overdue** | +50 | Critical |

Display tickets sorted by score descending. Show the age indicator:
- `(5d)` = opened 5 days ago
- `(2w)` = opened 2 weeks ago
- `(1mo)` = opened 1 month ago
- 🔥 = opened 14+ days ago (getting old)

### MR Prioritization (Priority Score)

Calculate priority score for MRs to rank them:

| Factor | Points | Description |
|--------|--------|-------------|
| **Pipeline failing** | +50 | Blocking - fix immediately |
| **Unresolved discussions** | +5 per discussion | Needs response |
| **Days since opened** | +2 per day, max 30 | Older MRs need attention |
| **Days since last activity** | +3 per day idle, max 30 | Stale MRs |
| **Approved but not merged** | +20 | Ready to ship |
| **Waiting on your review** | +15 | Teammate blocked |

Age indicators for MRs:
- `(1d)` = opened 1 day ago (fresh)
- `(5d)` = opened 5 days ago
- `(2w)` = opened 2 weeks ago 😴
- 🔥 = opened 7+ days with no activity (stale)

### MR Health Checks
- **❌ Pipeline failing** - needs immediate attention
- **⏳ Pipeline running** - check back later
- **💬 Unresolved discussions** - has comments needing response
- **😴 Stale (3+ days)** - no activity, may need a ping
- **✅ Ready to merge** - pipeline passing, no blockers

### Jira Grouping
After scoring, group by status for readability:
1. **Blocked/Needs Info** - action required from others
2. **In Review** - awaiting approval
3. **In Progress** - actively working
4. **Ready/To Do** - can start anytime
5. **Epics** - parent items (show separately)

Within each group, sort by priority score descending.

## Output Format

```markdown
## 🌅 Good Morning! Here's your day:

### ⏰ Coming Up
- **10:30** Team Standup (in 25 min) ⚠️
- ~~09:00~~ Daily Sync (2h ago)

### 🎫 Jira Tickets (X open)
**Needs Attention:**
- [PROJ-123] Blocked ticket (Blocked) 🔥 (2w) - score: 85
- [PROJ-456] Review pending (In Review) (5d) - score: 60

**In Progress:**
- [PROJ-789] Current work item (3d) - score: 45

**Ready to Start:**
- [PROJ-012] Next up (1d) - score: 25

### 🔀 Merge Requests

**Your MRs (X):** *(sorted by priority)*
- !5 Fix bug Y ❌ Pipeline failing (2d) - score: 58
- !3 Refactor Z 💬 3 discussions (5d) - score: 40
- !7 Add feature X ✅ Ready to merge (1d) - score: 25

**Awaiting Your Review (X):** *(sorted by wait time)*
- !4 Feature ABC by @teammate 🔥 (8d) - teammate waiting!

### ✅ Tasks
- Update documentation 🔴 Overdue (was due Feb 18)
- Review PR ⚠️ Due today

### 📧 Unread Emails (X)
- **Jane Doe** - Q1 Planning Doc (2h ago)
- **CI Bot** - Pipeline failed (30m ago)

### 💬 Chat Action Items & @Mentions (X items)

| # | From | Source | What's Needed | Link |
|---|------|--------|---------------|------|
| 1 | **Alex P.** | Chat: "Thread title" | Reply to follow-up question (2d waiting) | [Thread](https://chat.google.com/room/...) |
| 2 | **Sam R.** | Doc: "Meeting Notes" | Resolve action item assigned Feb 25 | [Doc](https://docs.google.com/...) |
| 3 | **Taylor H.** | Doc: "Article Review" | Respond to @mention | [Doc](https://docs.google.com/...) |

### 📝 Drive Comments (X unresolved)
- **Design Doc** - 2 new comments from @reviewer

---

### 🎯 Suggested Actions *(auto-ranked by priority score)*
1. 🔥 Fix failing pipeline on !5 (blocking, 2d old)
2. 🔥 Review !4 - teammate waiting 8 days!
3. 💬 Reply to @mention from Alex (waiting 2 days)
4. 🔴 Unblock PROJ-123 (stuck 2 weeks)
5. 📋 Resolve 3 doc action items (Alex, Sam, Taylor)
```

## Google Sheet Export — "Daily Brief"

Export the briefing to a Google Sheet named **"Daily Brief"**. Each day gets its own **sheet tab** named with the date (e.g., `2026-03-02`), so the spreadsheet accumulates a history of daily briefs. Checkboxes in column A auto-strikethrough the row when checked.

**Known sheet ID**: `YOUR_SHEET_ID`

### Step 1: Find the existing sheet (or create one)
```bash
# Search for existing sheet (avoid creating duplicates)
GW drive list --query "name = 'Daily Brief' and mimeType = 'application/vnd.google-apps.spreadsheet'" --output raw
# If not found: GW sheets create --title "Daily Brief" --output json --yes
```

### Step 2: Add a new sheet tab for today's date
Use the Sheets API `addSheet` request via curl (TOKEN from `~/gw.json`):
```bash
TOKEN=$(python3 -c "import json; d=json.load(open('$HOME/gw.json')); print(d['owner@example.com']['token']['access_token'])")

cat > /tmp/add-sheet-tab.json << 'JSONEOF'
{"requests": [{"addSheet": {"properties": {"title": "<YYYY-MM-DD>"}}}]}
JSONEOF

curl -s -X POST \
  "https://sheets.googleapis.com/v4/spreadsheets/<sheet-id>:batchUpdate" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @/tmp/add-sheet-tab.json
```
Capture the new `sheetId` (numeric) from the response for formatting requests.

If a tab for today's date already exists, clear it with `GW sheets write` or skip creation.

### Step 3: Write header + data rows
```bash
GW sheets write <sheet-id> "'<YYYY-MM-DD>'!A1:F<N+1>" --values '[
  ["✓", "#", "From", "Date", "What'\''s Needed", "Link"],
  [false, 1, "Name", "Date", "Action description", "Link text"],
  ...
]' --yes
```
- Column A data rows use `false` (becomes a checkbox after validation is applied)
- Column F uses plain text initially; hyperlinks are applied in Step 5

### Step 4: Apply formatting via Sheets API batchUpdate
All formatting in a single batch request:
```bash
cat > /tmp/sheet-format.json << JSONEOF
{"requests": [
  // 1. Checkbox data validation on A2:A<N+1>
  {"setDataValidation": {
    "range": {"sheetId": <tab-sheet-id>, "startRowIndex": 1, "endRowIndex": <N+1>, "startColumnIndex": 0, "endColumnIndex": 1},
    "rule": {"condition": {"type": "BOOLEAN"}, "strict": true, "showCustomUi": true}
  }},
  // 2. Conditional format: strikethrough + gray when checked
  {"addConditionalFormatRule": {
    "rule": {
      "ranges": [{"sheetId": <tab-sheet-id>, "startRowIndex": 1, "endRowIndex": <N+1>, "startColumnIndex": 0, "endColumnIndex": 6}],
      "booleanRule": {
        "condition": {"type": "CUSTOM_FORMULA", "values": [{"userEnteredValue": "=$A2=TRUE"}]},
        "format": {
          "textFormat": {"strikethrough": true, "foregroundColorStyle": {"rgbColor": {"red": 0.6, "green": 0.6, "blue": 0.6}}},
          "backgroundColor": {"red": 0.95, "green": 0.95, "blue": 0.95}
        }
      }
    },
    "index": 0
  }},
  // 3. Bold white header on blue background
  {"repeatCell": {
    "range": {"sheetId": <tab-sheet-id>, "startRowIndex": 0, "endRowIndex": 1, "startColumnIndex": 0, "endColumnIndex": 6},
    "cell": {"userEnteredFormat": {
      "textFormat": {"bold": true, "foregroundColorStyle": {"rgbColor": {"red": 1, "green": 1, "blue": 1}}},
      "backgroundColor": {"red": 0.2, "green": 0.4, "blue": 0.65},
      "horizontalAlignment": "CENTER", "verticalAlignment": "MIDDLE"
    }},
    "fields": "userEnteredFormat(textFormat,backgroundColor,horizontalAlignment,verticalAlignment)"
  }},
  // 4. Column widths: A=40, B=30, C=150, D=90, E=450, F=120
  {"updateDimensionProperties": {"range": {"sheetId": <tab-sheet-id>, "dimension": "COLUMNS", "startIndex": 0, "endIndex": 1}, "properties": {"pixelSize": 40}, "fields": "pixelSize"}},
  {"updateDimensionProperties": {"range": {"sheetId": <tab-sheet-id>, "dimension": "COLUMNS", "startIndex": 1, "endIndex": 2}, "properties": {"pixelSize": 30}, "fields": "pixelSize"}},
  {"updateDimensionProperties": {"range": {"sheetId": <tab-sheet-id>, "dimension": "COLUMNS", "startIndex": 2, "endIndex": 3}, "properties": {"pixelSize": 150}, "fields": "pixelSize"}},
  {"updateDimensionProperties": {"range": {"sheetId": <tab-sheet-id>, "dimension": "COLUMNS", "startIndex": 3, "endIndex": 4}, "properties": {"pixelSize": 90}, "fields": "pixelSize"}},
  {"updateDimensionProperties": {"range": {"sheetId": <tab-sheet-id>, "dimension": "COLUMNS", "startIndex": 4, "endIndex": 5}, "properties": {"pixelSize": 450}, "fields": "pixelSize"}},
  {"updateDimensionProperties": {"range": {"sheetId": <tab-sheet-id>, "dimension": "COLUMNS", "startIndex": 5, "endIndex": 6}, "properties": {"pixelSize": 120}, "fields": "pixelSize"}},
  // 5. Freeze header row
  {"updateSheetProperties": {"properties": {"sheetId": <tab-sheet-id>, "gridProperties": {"frozenRowCount": 1}}, "fields": "gridProperties.frozenRowCount"}}
]}
JSONEOF

curl -s -X POST \
  "https://sheets.googleapis.com/v4/spreadsheets/<sheet-id>:batchUpdate" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @/tmp/sheet-format.json
```

### Step 5: Add clickable HYPERLINK formulas
Replace plain text in column F with `HYPERLINK` formulas via `updateCells`:
```bash
cat > /tmp/hyperlinks.json << JSONEOF
{"requests": [{
  "updateCells": {
    "range": {"sheetId": <tab-sheet-id>, "startRowIndex": 1, "endRowIndex": <N+1>, "startColumnIndex": 5, "endColumnIndex": 6},
    "rows": [
      {"values": [{"userEnteredValue": {"formulaValue": "=HYPERLINK(\"<url>\",\"<display-text>\")"}}]},
      ...
    ],
    "fields": "userEnteredValue"
  }
}]}
JSONEOF

curl -s -X POST \
  "https://sheets.googleapis.com/v4/spreadsheets/<sheet-id>:batchUpdate" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @/tmp/hyperlinks.json
```

### Link column text conventions

| Source Type | Display Text |
|-------------|-------------|
| Google Chat thread | `Chat Thread` |
| Google Doc | `Google Doc` |
| Google Sheet | `Google Sheet` |
| Google Slides | `Google Slides` |
| Jira issue | `Jira Ticket` |
| GitLab MR | `GitLab MR` |

## Tips

- Run all queries in parallel to minimize wait time
- For Chat, focus on team spaces most relevant to user (your priority spaces)
- Summarize chat messages rather than listing raw text
- Keep briefing scannable - tables for meetings, bullets for tasks
- End with 3-5 actionable suggestions based on priority
- **Chat notification feed is the single best source** — `spaces/EXAMPLE_NOTIFICATION_FEED` aggregates action items from Docs, Sheets, Slides, and Chat in one place
- Always resolve document names to clickable Google Drive links so the user can jump directly to action items
- Present the Chat Action Items table with links — this is the most actionable part of the briefing
- When scanning chat spaces for @mentions, the default message limit (20-40) may miss older threads; increase `--limit` if needed
- For chat threads with @mentions, reading the full thread context is critical to understanding what's actually being asked
- Reuse the same "Daily Brief" sheet (ID: `YOUR_SHEET_ID`) — add a new **sheet tab** per day
- Tab names use the date format `YYYY-MM-DD` (e.g., `2026-03-02`) for easy scanning
- Use `=HYPERLINK("url","text")` formulas for clickable links in Sheets (unlike Docs, no post-hoc styling needed)
- Checkboxes + conditional formatting provide auto-strikethrough when the user checks an item — no manual formatting needed
- Sheets API `batchUpdate` requires the numeric `sheetId` (from `addSheet` response or sheet metadata), not the tab title
- The `TOKEN` for Sheets API calls comes from `~/gw.json` → `owner@example.com` → `token.access_token`
