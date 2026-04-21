# Data Format Reference

This document describes the JSON data structures that the status-skill processes.
There are three data sources: Google Workspace data (from Apps Script), GitHub activity (from `fetch-github.js`), and Slack activity (from MCP tools).

## Table of Contents

- [Apps Script JSON Output](#apps-script-json-output)
  - [metadata](#metadata)
  - [calendar](#calendar)
  - [email](#email)
  - [transcripts](#transcripts)
  - [documents](#documents)
- [GitHub Activity JSON](#github-activity-json)
- [Slack Activity](#slack-activity)
- [Error Handling in Data](#error-handling-in-data)
- [Testing with Sample Data](#testing-with-sample-data)

---

## Apps Script JSON Output

The Apps Script web app returns a single JSON object when the user opens the URL in their browser. The Apps Script code itself is deployed inside Google Apps Script (not stored in this repo). The URL format is:

```
<apps-script-url>?start=YYYY-MM-DD&end=YYYY-MM-DD&user=<email>&userName=<Full%20Name>
```

### Top-level structure

```json
{
  "metadata": { ... },
  "calendar": { ... },
  "email": { ... },
  "transcripts": { ... },
  "documents": { ... }
}
```

### metadata

Basic info about the request. Not used in the report directly but useful for validation.

```json
{
  "user": "jsmith@company.com",
  "userName": "Jane Smith",
  "date_range": { "start": "2026-03-02", "end": "2026-03-06" },
  "generated_at": "2026-03-07T14:30:00.000Z"
}
```

### calendar

| Field | Type | Description |
|-------|------|-------------|
| `total` | number | Total number of events |
| `totalMeetingMinutes` | number | Sum of non-all-day event durations |
| `events` | array | Event objects (see below) |
| `error` | string? | Present only if calendar fetch failed |

**Event object:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Calendar event ID |
| `summary` | string | Event title (e.g., "Weekly Team Sync") |
| `description` | string | First 200 chars of event description |
| `start` | string | ISO 8601 datetime |
| `end` | string | ISO 8601 datetime |
| `isAllDay` | boolean | True for all-day events |
| `durationMinutes` | number | Duration in minutes (0 for all-day) |
| `location` | string | Location or empty string |
| `organizer` | object? | `{ email, displayName, self }` |
| `attendees` | array | `[{ email, displayName, responseStatus, self }]` |
| `conferenceLink` | string | Video call URL or empty string |
| `isRecurring` | boolean | True if part of a recurring series |
| `calendarName` | string | Name of the calendar it came from |

**responseStatus values:** `"yes"`, `"no"`, `"maybe"`, `"invited"` (Apps Script's GuestStatus enum, lowercased)

### email

| Field | Type | Description |
|-------|------|-------------|
| `sent` | array | Sent email objects (see below) |
| `received_summary` | object | `{ total, top_senders }` |
| `error` | string? | Present only if email fetch failed |

**Sent email object:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Gmail message ID |
| `threadId` | string | Gmail thread ID |
| `subject` | string | Email subject line |
| `to` | string | Recipient(s) |
| `date` | string | ISO 8601 datetime |
| `snippet` | string | First 200 chars of plain text body |

**received_summary:**

```json
{
  "total": 47,
  "top_senders": [
    { "email": "boss@company.com", "count": 8 },
    { "email": "teammate@company.com", "count": 5 }
  ]
}
```

### transcripts

| Field | Type | Description |
|-------|------|-------------|
| `total` | number | Number of transcripts found |
| `transcripts` | array | Transcript objects (see below) |
| `error` | string? | Present only if transcript search failed |

**Transcript object:**

| Field | Type | Description |
|-------|------|-------------|
| `meeting_title` | string | Cleaned meeting name (e.g., "Weekly Standup") |
| `date` | string | ISO 8601 datetime of last modification |
| `doc_id` | string | Google Docs ID |
| `doc_url` | string | Link to the transcript document |
| `participants` | array | List of speaker names found in the transcript |
| `user_contributions` | array | `[{ timestamp, text }]` -- what the user said |
| `total_lines` | number | Total lines in the transcript |
| `error` | string? | Present if this specific transcript failed to parse |

**How transcript parsing works:** The Apps Script exports Google Meet transcripts as plain text. The format is:

```
Speaker Name
HH:MM:SS
What they said goes here and can span
multiple lines until the next speaker.

Next Speaker
HH:MM:SS
Their text...
```

The parser matches the user's name (including variants like first name only, "Last, First", "(You)") against speaker names to extract `user_contributions`.

### documents

| Field | Type | Description |
|-------|------|-------------|
| `total` | number | Number of documents found |
| `documents` | array | Document objects (see below) |
| `error` | string? | Present only if document search failed |

**Document object:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Google Drive file ID |
| `name` | string | Document or presentation name |
| `type` | string | `"doc"` for Google Docs, `"slides"` for Google Slides |
| `modified` | string | ISO 8601 datetime of last modification |
| `created` | string | ISO 8601 datetime of creation |
| `url` | string | Link to the document |
| `isOwner` | boolean | True if the user owns this document |
| `wasCreatedThisWeek` | boolean | True if created within the queried date range |

---

## GitHub Activity JSON

Generated by `scripts/fetch-github.js` using the `gh` CLI. Default output path: `/tmp/github-activity.json`.

### Top-level structure

```json
{
  "meta": { ... },
  "commits": { ... },
  "pull_requests_opened": { ... },
  "pull_requests_merged": { ... },
  "pull_requests_reviewed": { ... },
  "issues_opened": { ... },
  "issues_closed": { ... },
  "repositories_created": { ... }
}
```

### meta

```json
{
  "username": "jsmith",
  "start_date": "2026-03-02",
  "end_date": "2026-03-06",
  "retrieved_at": "2026-03-07T14:30:00.000Z"
}
```

### commits

| Field | Type | Description |
|-------|------|-------------|
| `total_count` | number | Total commits in date range |
| `repos` | array | Sorted list of unique `org/repo` names |
| `details` | array | Commit objects (see below) |

**Commit object:** `{ repo, message, date, sha, url }`

### pull_requests_opened / merged / reviewed

| Field | Type | Description |
|-------|------|-------------|
| `total_count` | number | Total PRs in this category |
| `details` | array | PR objects (see below) |

**PR object:** `{ repo, title, url, state?, created_at?, merged_at? }`

### issues_opened / issues_closed

| Field | Type | Description |
|-------|------|-------------|
| `total_count` | number | Total issues in this category |
| `details` | array | Issue objects (see below) |

**Issue object:** `{ repo, title, url, created_at?, closed_at? }`

### repositories_created

| Field | Type | Description |
|-------|------|-------------|
| `total_count` | number | Number of new repos created |
| `details` | array | Repo objects: `{ full_name, description, url, created_at }` |

---

## Slack Activity

Slack data is fetched via MCP tools at runtime (not from a file). The skill calls `mcp__slack__search_messages` and processes the results directly.

### Data flow

1. `mcp__slack__whoami` returns the authenticated Slack username
2. `mcp__slack__search_messages` with `from:me after:YYYY-MM-DD before:YYYY-MM-DD` returns messages sent by the user
3. `mcp__slack__search_messages` with `to:me after:YYYY-MM-DD before:YYYY-MM-DD` returns DMs and direct messages sent to the user
4. `mcp__slack__search_messages` with `@username after:YYYY-MM-DD before:YYYY-MM-DD` returns channel messages mentioning the user

### Message format (compact mode, default)

Each message is returned as a formatted string:

```
[1714000000.000100] @username: message text here [channel:C01234567|channel-name] [thread:1713999000.000050]
```

| Field | Description |
|-------|-------------|
| Timestamp | Slack message timestamp (Unix format with microseconds) |
| `@username` | Sender's display name (resolved from user ID) |
| Message text | The message body with user mentions resolved to `@handle` format |
| `channel:ID\|name` | Channel where the message was posted (present in search results) |
| `thread:ts` | Parent thread timestamp if the message is a reply (omitted for top-level messages) |

### What to extract from Slack data

- **Themes and topics**: Group messages by channel or conversation thread to identify work areas
- **Decisions**: Look for messages that state decisions, approvals, or conclusions
- **Action items**: Messages with commitments ("I'll do X", "will follow up on Y")
- **Collaborations**: Cross-team or external conversations that indicate partnership work
- **Context for other data**: Slack follow-ups to calendar events, PR discussions, or email threads

### When Slack is not available

If the Slack MCP server is not configured or tokens are expired, the skill falls back to asking the user to paste relevant Slack messages manually (Phase 3.5 in SKILL.md).

---

## Error Handling in Data

Any section in the Apps Script output can have an `error` field alongside empty data arrays. This happens when:

- Calendar API is not accessible (permissions, quota)
- Gmail search fails
- Drive API cannot list files
- Individual transcript export fails (appears per-transcript, not top-level)

The skill should note which data sources had errors and proceed with whatever data is available.

---

## Testing with Sample Data

To test the skill without real Google Workspace data, you can simulate the Apps Script output by pasting sample JSON. Sample fixtures are in `evals/fixtures/`:

- `normal-week.json` -- typical week with meetings, emails, transcripts, docs
- `light-week.json` -- few meetings, no transcripts, minimal GitHub
- `partial-errors.json` -- week with some data source failures
- `github-activity.json` -- sample GitHub fetch output (busy week)
- `github-activity-light.json` -- sample GitHub fetch output (light week)

These fixtures follow the exact same schema the Apps Script produces, so the skill processes them identically to real data.
