---
name: status-report
description: "Generate a weekly status report for your manager by analyzing Google Calendar events, sent emails, and Google Meet transcripts. Run with /status-report for the past week, or /status-report 2026-02-23 2026-03-01 for a custom date range."
argument-hint: "[start-date end-date, e.g. 2026-02-23 2026-03-01]"
---

# Weekly Status Report Generator

Generate a comprehensive weekly status report by pulling data from Google Calendar, Gmail, Google Meet transcripts, and GitHub activity, then enriching it with user input including Slack messages.

## Configuration

- Scripts directory: `~/.claude/skills/status-report/scripts/`
- Apps Script URL file: `~/.claude/skills/status-report/scripts/config/apps-script-url.txt`
- GitHub activity script: `~/.claude/skills/status-report/scripts/fetch-github.js`

## Phase 0: Environment Setup and User Info

Before anything else, check that the environment is ready and gather user-specific info.

### Environment checks

1. Check if `~/.claude/skills/status-report/scripts/config/apps-script-url.txt` exists and contains a URL.

### User info gathering

Ask the user for the following (if not already known from memory or previous sessions):

1. **Apps Script JSON data**: Ask the user to open their Apps Script web app URL in a browser and paste the JSON output. Provide the URL with appropriate query parameters:
   ```
   <apps-script-url>?start=YYYY-MM-DD&end=YYYY-MM-DD&user=<email>&userName=<Full%20Name>
   ```
   If no Apps Script URL is configured, ask the user to get the URL from a teammate (shared via Slack/team docs) and save it:
   ```
   echo "<url>" > ~/.claude/skills/status-report/scripts/config/apps-script-url.txt
   ```

2. **GitHub username**: Ask for their GitHub username to fetch development activity.

## Phase 1: Fetch Data

Parse `$ARGUMENTS` for optional start and end dates. If no dates provided, default to the past week (Monday-Friday).

### 1a: Google Workspace data (Calendar, Email, Transcripts)

The user must paste the JSON output from their Apps Script URL in their browser. The script cannot be called directly from the CLI due to corporate SSO restrictions.

Save the pasted JSON to `/tmp/status-report-data.json`.

### 1b: GitHub activity

Run the GitHub fetch script:
```
cd ~/.claude/skills/status-report/scripts && node fetch-github.js --user <github-username> --start YYYY-MM-DD --end YYYY-MM-DD
```

The script outputs the path to a JSON file (default: `/tmp/github-activity.json`). Read that file to get the GitHub data.

If any data source has an `error` field, inform the user but continue with available data.

## Phase 2: Analyze Data

Review all fetched data and organize it:

### Calendar Events
- Count total meetings and meeting hours
- Categorize each meeting into one of:
  - **Standup/Sync**: recurring daily or frequent short meetings
  - **1:1**: two-person meetings, often with manager or direct reports
  - **Project Meeting**: meetings with 3+ attendees on specific topics
  - **External**: meetings with attendees outside the organization
  - **Focus/Block Time**: calendar blocks for individual work
- Identify key meetings (non-recurring, project-specific, or external)

### Sent Emails
- Group by topic/thread when possible
- Identify key themes (project updates, reviews, requests, decisions)
- Note significant recipients (leadership, cross-team, external)

### Meeting Transcripts
- For each transcript with user contributions:
  - Summarize what the user discussed or presented
  - Identify decisions made or action items taken
  - Note key topics the user spoke about
- If transcript content couldn't be parsed (Gemini Notes format), use the meeting title for context

### GitHub Activity
- Summarize commits by repository (group related commits, describe outcomes not git mechanics)
- ALWAYS use full repository names including the org/owner prefix (e.g., `myorg/my-project`, NOT just `my-project`)
- List PRs opened, merged, and reviewed
- List issues opened and closed
- Note any new repositories created
- Highlight cross-team collaboration (reviews on other teams' repos, upstream contributions)

## Phase 3: Present Summary and Ask Clarifying Questions

Present a brief summary of what was found across all data sources, then ask 3-5 targeted questions:

1. "I found N meetings this week including [key meetings]. What were the most important outcomes or decisions from these?"
2. "You sent N emails, with themes around [topics]. Any notable progress or completions worth highlighting?"
3. "From your meeting transcripts, I see you discussed [topics]. Anything to add or correct about these?"
4. "On GitHub, you had N commits across [repos], [N PRs], [N issues]. Anything to highlight about this development work?"
5. "What were your top 2-3 accomplishments this week that might not show up in calendar/email/GitHub?"
6. "Any blockers, risks, or items needing escalation?"
7. "What are your top priorities for next week?"

Wait for the user to respond to all questions before proceeding.

## Phase 3.5: Slack Message Enrichment

Since there is no Slack API access, ask the user to paste relevant Slack messages to add detail to the report.

For each key project or topic identified from the calendar, email, and GitHub analysis:
- Ask: "Do you have any Slack messages related to **[topic/project]** you'd like to include? Paste them here, or type 'skip' to move on."

For meetings that likely had Slack follow-up:
- Ask: "Any Slack follow-ups from **[meeting name]** worth including?"

When the user pastes Slack messages:
- Extract the relevant details (decisions, updates, action items)
- Note the channel/context if visible
- Incorporate into the appropriate report section

When the user types "skip" or "done", move to the next topic or proceed to report generation.

Do NOT ask about every single meeting or topic -- focus on the 3-5 most significant ones to avoid overwhelming the user.

## Phase 4: Generate Report

Generate a markdown status report and save it to `~/Desktop/status-report-YYYY-MM-DD.md` (using the end date of the reporting period).

### Report Template

```markdown
# [User's Full Name] - Week of [Month Day]

[Activity bullet] - include context on who, why, and link to artifacts where available
[Activity bullet] - note WIP items inline (e.g., "WIP - deck in progress")
[Activity bullet] - name customers, partners, and teammates involved
development:
[org/repo-name]: [Outcome-focused description of what was done and why]
[org/repo-name]: [Another outcome]
```

### Example (good status report)

```markdown
# Jane Smith - Week of Feb 23

Customer sync [Project Alpha] - helping out teammate on migration blockers
Enablement sync with Pat Lee to review AIOps training delivery plan
Helping Dana Kim with Acme Corp (as a customer) - networking architecture questions, looping in the database team
Webinar planning sync - Q1 AIOps webinar tracker
Internal AI training coordination with Alex Chen - product pitch prep - 2026/02/25 14:26 EST - Notes by Gemini
development:
org/api-gateway: Resolved two critical issues with authentication handling and CI build failures, updated entrypoint script and project docs.
org/workshops: Updated workshop exercises to reflect platform version migration from 6.15 to 6.18.
org/product-demos: Cross-team code review for updates to the demo bootstrap branch.
org/ssl-certs, org/mcp-tools, org/service-config: Applied minor configuration changes and documentation updates across these projects.
```

### Writing Guidelines
- **Direct and concise** -- write bullets as quick notes, not formal sentences
- **Include who and why** -- "Met with Acme Corp team including sales and consulting" not just "Customer meeting"
- **Name names** -- customers, partners, teammates involved
- **Link to artifacts** -- YouTube videos, Slack threads, docs, decks, Gemini notes
- **Mark WIP inline** -- "WIP - Network Refresh 2026" not a separate section
- **Separate development work** -- list under "development:" with org/repo prefix
- **Describe outcomes not git mechanics** -- "Resolved critical CI pipeline issues" not "pushed 5 commits"
- **Always use full org/repo names** -- `myorg/my-project` not `my-project`
- **Group related small items** -- "org/ssl-certs, org/mcp-tools: Applied minor config changes across these projects"
- **Skip routine meetings** -- don't list daily standups or recurring syncs unless something notable happened
- **No tables or formal headers** -- keep it flat and scannable

## Phase 5: Review

After generating the report:
1. Show the user the complete report
2. Ask: "Would you like to adjust anything in this report?"
3. Make any requested changes
4. Confirm the final file location

## Error Handling

- If the Apps Script URL is not configured, ask the user to get it from a teammate and save it to `scripts/config/apps-script-url.txt`
- If the user can't paste JSON (Apps Script issues), proceed with GitHub data and user input only
- If `gh` CLI is not authenticated, skip GitHub activity and note the gap
- If individual data sources fail, proceed with available data and note the gap
- If no transcripts are found, that's normal -- not all meetings have transcripts enabled
