# Status Skill for Claude Code

A Claude Code plugin that generates weekly status reports by analyzing Google Calendar, Gmail, Google Meet transcripts, Google Docs/Slides activity, GitHub commits/PRs, and Slack messages.

## Installation

### Option 1: Install from GitHub (recommended)

Add this repo as a marketplace, then install the plugin:

```bash
claude plugin marketplace add https://github.com/ansible-tmm/status-skill
claude plugin install status-skill
```

### Option 2: Install from a local clone

If you have the repo cloned locally:

```bash
claude plugin marketplace add /path/to/status-skill
claude plugin install status-skill
```

### Verify installation

```bash
claude plugin list
```

You should see `status-skill` in the list.

## Setup

### 1. Configure the Apps Script URL

Get the Apps Script web app URL from your team (shared via Slack or team docs), then save it:

```bash
# Find where the plugin was installed
SKILL_DIR=$(find ~/.claude/plugins -path "*/status-skill/skills/status-skill" -type d 2>/dev/null | head -1)

# Save the URL
mkdir -p "$SKILL_DIR/scripts/config"
echo "<url-from-teammate>" > "$SKILL_DIR/scripts/config/apps-script-url.txt"
```

### 2. Authenticate GitHub CLI

The skill uses `gh` CLI to fetch GitHub activity:

```bash
gh auth login
```

### 3. Configure the Slack MCP Server (optional but recommended)

The Slack MCP server lets the skill automatically fetch your Slack messages, mentions, and conversations for the reporting period. Without it, you can still paste Slack messages manually when prompted.

The server uses browser session tokens (`xoxc`/`xoxd`) extracted from a logged-in Slack session. No Slack admin approval or OAuth app registration is required.

#### Prerequisites

- Python 3.8+
- Podman or Docker

#### Option A: Automated setup (recommended)

The setup script handles everything: installs Playwright, opens a browser for Slack login, extracts tokens, and registers the MCP server with Claude Code.

```bash
python3 <(curl -fsSL https://raw.githubusercontent.com/redhat-community-ai-tools/slack-mcp/main/scripts/setup-slack-mcp.py)
```

The script will:
1. Create a Python virtual environment and install Playwright with Chromium
2. Pull the `quay.io/redhat-ai-tools/slack-mcp` container image
3. Open a Chromium browser window. Log in to your Slack workspace.
4. Prompt you for a Slack channel ID to receive server logs (a self-DM or Slackbot DM works well; navigate there in `https://app.slack.com` and copy the last segment of the URL, e.g., `DXXXXXXXXX`)
5. Extract session tokens, write a wrapper script, and register the MCP server in `~/.claude.json`

After the script finishes, restart Claude Code for the Slack tools to become available.

#### Option B: Manual token extraction (if the automated script does not work)

If the setup script fails (e.g., Playwright cannot install Chromium, or your environment blocks automated browsers), you can extract the tokens manually from your browser's developer tools.

**Step 1: Open Slack in your browser**

Navigate to `https://app.slack.com` and log in to your workspace.

**Step 2: Extract the `xoxc` token**

1. Open Developer Tools (F12 or Cmd+Option+I on macOS)
2. Go to the **Console** tab
3. Paste and run:
   ```javascript
   JSON.parse(localStorage.localConfig_v2).teams[Object.keys(JSON.parse(localStorage.localConfig_v2).teams)[0]].token
   ```
4. Copy the `xoxc-...` value (without the surrounding quotes)

**Step 3: Extract the `xoxd` token (cookie)**

1. In Developer Tools, go to the **Application** tab (Chrome) or **Storage** tab (Firefox)
2. Under **Cookies**, click `https://app.slack.com`
3. Find the cookie named `d`
4. Copy its value (starts with `xoxd-...`)

Alternatively, in the Console tab:
```javascript
document.cookie.split('; ').find(c => c.startsWith('d=')).split('=').slice(1).join('=')
```

**Step 4: Find a logs channel ID**

Navigate to any channel or DM in Slack (a self-DM or Slackbot DM works well). The channel ID is the last segment of the URL:
```
https://app.slack.com/client/TXXXXXXXX/DXXXXXXXXX
                                       ^^^^^^^^^^^^ this is the channel ID
```

**Step 5: Create the token file and wrapper script**

```bash
# Create the install directory
mkdir -p ~/.local/share/slack-mcp

# Save the tokens (replace with your actual values)
cat > ~/.local/share/slack-mcp/tokens.env << 'EOF'
SLACK_MCP_XOXC_TOKEN=xoxc-your-token-here
SLACK_MCP_XOXD_TOKEN=xoxd-your-token-here
EOF
chmod 600 ~/.local/share/slack-mcp/tokens.env

# Create the wrapper script (replace DXXXXXXXXX with your logs channel ID)
cat > ~/.local/share/slack-mcp/run-slack-mcp.sh << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

TOKENS="$HOME/.local/share/slack-mcp/tokens.env"

if [[ ! -f "$TOKENS" ]]; then
  echo "Error: $TOKENS not found." >&2
  exit 1
fi

source "$TOKENS"

exec podman run -i --rm \
  -e SLACK_XOXC_TOKEN="${SLACK_MCP_XOXC_TOKEN}" \
  -e SLACK_XOXD_TOKEN="${SLACK_MCP_XOXD_TOKEN}" \
  -e MCP_TRANSPORT=stdio \
  -e LOGS_CHANNEL_ID="DXXXXXXXXX" \
  quay.io/redhat-ai-tools/slack-mcp
SCRIPT
chmod 755 ~/.local/share/slack-mcp/run-slack-mcp.sh

# Pull the container image
podman pull quay.io/redhat-ai-tools/slack-mcp
```

**Step 6: Register the MCP server with Claude Code**

Add the `slack` entry to the `mcpServers` section of `~/.claude.json`:

```json
{
  "mcpServers": {
    "slack": {
      "type": "stdio",
      "command": "/Users/YOUR_USERNAME/.local/share/slack-mcp/run-slack-mcp.sh",
      "args": [],
      "env": {}
    }
  }
}
```

Replace `YOUR_USERNAME` with your actual username. Restart Claude Code.

#### Refreshing expired tokens

Slack session tokens expire when the browser session ends (typically weeks to months). When Slack API calls start failing:

**If you used the automated setup:**
```bash
python3 ~/.local/share/slack-mcp/.venv/../scripts/setup-slack-mcp.py --refresh-tokens
```

Or re-run the original setup command.

**If you set up manually:**

Repeat Steps 2-3 above to get fresh `xoxc` and `xoxd` values, then update `~/.local/share/slack-mcp/tokens.env`. No other changes needed.

#### Verifying Slack is working

After restarting Claude Code, test by asking: "What is my username in Slack?" If the Slack MCP server is configured correctly, Claude will call the `whoami` tool and return your Slack username.

## Usage

In Claude Code, run:

```
/status-skill
```

This generates a status report for the past week (Monday-Friday). You can also specify dates:

```
/status-skill 2026-03-02 2026-03-06
```

## How It Works

1. **Google Workspace data**: The skill asks you to open the Apps Script URL in your browser and paste the JSON output. This workaround is needed because corporate SSO prevents direct API calls from the CLI.

2. **GitHub activity**: Fetched automatically via `gh` CLI.

3. **Slack messages**: Fetched automatically via the Slack MCP server (if configured). The skill searches for messages you sent, messages sent to you (DMs), and messages where you were mentioned during the reporting period. If the MCP server is not configured, the skill falls back to asking you to paste relevant Slack messages.

4. **Analysis**: The skill analyzes all data sources, asks you clarifying questions about outcomes and context, and synthesizes everything into a report.

5. **Report**: Generates a plain-text status report in a code block (with copy button) and an HTML version styled with Red Hat fonts saved to `~/Desktop/`. Open the HTML file in a browser, select all, copy, and paste into Google Docs for formatted output.

## Data Sources

| Source | How it's fetched | What it captures |
|--------|-----------------|------------------|
| Google Calendar | Apps Script (paste JSON) | Meetings, attendees, duration |
| Gmail | Apps Script (paste JSON) | Sent emails, subjects, recipients |
| Google Meet | Apps Script (paste JSON) | Transcript text, speaker contributions |
| Google Docs/Slides | Apps Script (paste JSON) | Documents created/modified |
| GitHub | `gh` CLI (automatic) | Commits, PRs, issues, reviews |
| Slack | MCP server (automatic) | Messages sent, mentions, conversations |

## Data Format

See `skills/status-skill/references/data-format.md` for the full JSON schema of all data sources.

## Testing with Sample Data

Sample fixtures are provided in `evals/fixtures/` to test the skill without real data:

- `normal-week.json` + `github-activity.json` -- typical busy week
- `light-week.json` + `github-activity-light.json` -- light week with PTO
- `partial-errors.json` -- week with some data source failures

To test, run `/status-skill` and paste the contents of a fixture file when prompted for the Apps Script JSON. Slack data will be fetched live from the MCP server if configured, or skipped if not.

## File Structure

```
status-skill/
  .claude-plugin/
    plugin.json                   # Plugin manifest for Claude Code
    marketplace.json              # Marketplace metadata for discovery
  skills/
    status-skill/
      SKILL.md                    # Skill definition
      scripts/
        fetch-github.js           # GitHub activity fetcher (uses gh CLI)
        config/
          apps-script-url.txt.example
      templates/
        status-report.html        # HTML template for Google Docs output
      references/
        data-format.md            # Full JSON schema documentation
  evals/
    evals.json                    # Test cases
    fixtures/                     # Sample data for testing
  README.md                       # This file
```
