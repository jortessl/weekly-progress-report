# Weekly Progress Report Skill for Claude Code

This Claude Code skill automates updating Confluence progress report pages by pulling data from Aha! and Jira.

## What It Does

When you run `/weekly-progress-report`, Claude will:

1. **Update Progress Bars** - Pulls current progress percentages from linked Aha! releases
2. **Update Status Pills** - Syncs status from Aha! (Planned, In Progress, Shipped, etc.)
3. **Update Due Dates** - Fetches target due dates from linked Jira issues
4. **Update Child Features** - Populates the "Child Features" section with features from each Aha! release
5. **Sort by Due Date** - Reorders cells under each heading by target due date (soonest first, stretch goals last)
6. **Calculate Overall Progress** - Computes and updates the overall program progress at the top of the page

Make sure to provide the URL to the page you want Claude to run this skill on - if starting from the template page provided, be sure to make a copy of that page and save it to the folder you would like to have it within in Confluence. 

## Prerequisites

### 1. Aha! MCP

Set up the Aha! MCP server:

```bash
git clone https://github.com/jortessl/aha-mcp
cd aha-mcp
# Follow setup instructions in that repo
```

Add to your Claude Code MCP configuration.

### 2. Confluence CLI

Install and configure the Confluence CLI:

```bash
# Install via Homebrew
brew install confluence-cli

# Create config
mkdir -p ~/.confluence-cli
cat > ~/.confluence-cli/config.json << 'EOF'
{
  "domain": "your-domain.atlassian.net",
  "email": "your-email@company.com",
  "token": "your-api-token"
}
EOF
```

Get your API token from: https://id.atlassian.com/manage-profile/security/api-tokens

### 3. Atlassian MCP (for Jira)

Set up the Atlassian MCP for Jira access. This is used to fetch due dates from Jira issues.

## Installation

Copy the skill to your Claude skills directory:

```bash
mkdir -p ~/.claude/skills/weekly-progress-report
cp SKILL.md ~/.claude/skills/weekly-progress-report/
```

## Usage

```
/weekly-progress-report <confluence-page-url>
```

### Example

```
/weekly-progress-report https://your-domain.atlassian.net/wiki/spaces/dev/pages/12345/My-Progress-Report
```

## Creating a New Progress Report Page

1. Copy the template page: `XX-XX-XXXX Agentic AI Progress Report Template - Make a Copy`. Link: https://cisco-sbg.atlassian.net/wiki/spaces/dev/pages/1425168911/XX-XX-XXXX+Agentic+AI+Progress+Report+Template+-+Make+a+Copy 
2. Rename it with the current date or the title you want it to be - it's recommended that this be a standard format for the cadence in which you plan to create these reports. The template is meant to be duplicated each week as a new report. (e.g., `05-15-2026 Agentic AI Progress Report`)
3. Update all Aha! release links to point to your actual releases
4. Update all Jira links to point to your actual issues
5. Run `/weekly-progress-report` with your new page URL

## Page Structure Requirements

For the skill to work correctly, your page should have:

- **Workstream cells** in tables, each containing:
  - An Aha! release link (e.g., `https://ciscosecurity.aha.io/releases/AAI-R-31`)
  - A Jira link (e.g., `https://cisco-sbg.atlassian.net/browse/ZTMCP-1205`)
  - A progress bar image in each cell - the template already has a progress bar in each cell. When you run the skill, Claude will update this image for each cell. 
  - A status macro - again, the template already has this and will be updated when running this skill
  - An expandable "Child Features" section - again, the template already has this and will be updated when running this skill

- **Section headings** (e.g., "Duo", "Secure Access", "CII") to group workstreams

- **Overall progress header** showing `XX% Overall Progress (N workstreams)`

## Troubleshooting

**"Aha! MCP not found"** - Make sure the Aha MCP is configured in your Claude Code settings and restart Claude Code.

**"Confluence CLI not found"** - Install with `brew install confluence-cli` and configure `~/.confluence-cli/config.json`.

**"Progress bar not showing"** - The skill will automatically create and upload missing progress bar SVGs.

**"Status not updating"** - Check that the Aha! release link in the cell is valid and accessible.

## License

MIT
