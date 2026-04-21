# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Claude Code plugin that provides the `/status-skill` slash command. It generates weekly status reports by analyzing Google Calendar, Gmail, Google Meet transcripts, Google Docs/Slides activity, GitHub commits/PRs, and Slack messages.

## Architecture

The repo follows the Claude Code plugin structure:

- `.claude-plugin/` -- plugin manifest (`plugin.json`) and marketplace metadata (`marketplace.json`)
- `skills/status-skill/SKILL.md` -- the skill definition that Claude reads at invocation time
- `skills/status-skill/scripts/fetch-github.js` -- Node.js script that fetches GitHub activity via `gh` CLI
- `skills/status-skill/references/data-format.md` -- full JSON schema for all data sources (loaded on demand, not at startup)
- `evals/` -- test cases (`evals.json`) and fixture data (`fixtures/`)

Google Workspace data is fetched via an Apps Script web app (deployed externally, not in this repo). The user pastes JSON output from their browser because corporate SSO blocks direct API calls from the CLI. GitHub data is fetched automatically via the bundled `fetch-github.js` script. Slack data is fetched automatically via MCP tools when the Slack MCP server is configured (see README for setup).

## Testing

Run `/status-skill` and paste contents from `evals/fixtures/` when prompted:

- `normal-week.json` + `github-activity.json` -- typical busy week
- `light-week.json` + `github-activity-light.json` -- light week with PTO
- `partial-errors.json` -- data source failures (pair with any GitHub fixture)

## Key Conventions

- The skill name is `status-skill` (not `status-report` -- that was the old name)
- Report output must be plain text (no markdown formatting) so it pastes cleanly into Google Docs
- Reports use `documents:` and `development:` sections, not headers
- GitHub repos always use full `org/repo` names
- The `scripts/config/apps-script-url.txt` file is gitignored -- it contains a deployment-specific URL shared via Slack
