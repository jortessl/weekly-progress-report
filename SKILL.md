# Weekly Progress Report Updater

This skill updates Confluence progress report pages by pulling data from Aha! and Jira.

## Invocation
- Trigger: `/weekly-progress-report`
- User-invocable: yes

## Prerequisites

Before running this skill, verify the following tools are available:

### 1. Aha! MCP
Check if Aha MCP tools are available by testing `mcp__aha__aha_graphql`. If not available, instruct the user:

> **Aha! MCP Setup Required**
>
> You need to set up the Aha! MCP server to use this skill.
>
> 1. Clone the MCP: `git clone https://github.com/jortessl/aha-mcp`
> 2. Follow the setup instructions in that repository
> 3. Add the MCP to your Claude Code configuration
> 4. Restart Claude Code

### 2. Confluence CLI
Check if `confluence` CLI is available by running `confluence --version`. If not available, instruct the user:

> **Confluence CLI Setup Required**
>
> You need to install and configure the Confluence CLI:
>
> 1. Install via Homebrew: `brew install confluence-cli`
> 2. Create config directory: `mkdir -p ~/.confluence-cli`
> 3. Create config file at `~/.confluence-cli/config.json`:
>    ```json
>    {
>      "domain": "your-domain.atlassian.net",
>      "email": "your-email@company.com",
>      "token": "your-api-token"
>    }
>    ```
> 4. Get your API token from: https://id.atlassian.com/manage-profile/security/api-tokens

### 3. Atlassian MCP (for Jira)
Check if Atlassian MCP tools are available by testing `mcp__atlassian__getJiraIssue`. If not available, instruct the user to set up the Atlassian MCP for Jira access.

## Input Requirements

**Required**: A Confluence page URL or page ID

If the user does not provide a page link, ask:
> Please provide the Confluence page URL or page ID for the progress report you want to update.

**Template Detection**: If the user provides this URL:
`https://cisco-sbg.atlassian.net/wiki/spaces/dev/pages/1425168911/XX-XX-XXXX+Agentic+AI+Progress+Report+Template+-+Make+a+Copy`

Respond with:
> This is the template page. You need to:
> 1. Make a copy of this page (click "..." menu → "Copy")
> 2. Rename it with the current date (e.g., "05-15-2026 Agentic AI Progress Report")
> 3. Update all the Aha! release links and Jira links to point to your actual records
> 4. Then run `/weekly-progress-report` with the URL of your new page

## Workflow

### Step 1: Read the Confluence Page
```bash
confluence read <page_id> --format html > /tmp/progress-report.html
```

### Step 2: Extract All Workstream Cells
Parse the HTML to find all table cells containing workstreams. Each workstream cell contains:
- An Aha! release link (e.g., `https://ciscosecurity.aha.io/releases/AAI-R-31`)
- A Jira link (e.g., `https://cisco-sbg.atlassian.net/browse/ZTMCP-1205`)
- A progress bar image (`<ri:attachment ri:filename="progress-bar-XX.svg"/>`)
- A status macro (`<ac:structured-macro ac:name="status">`)
- A "Child Features" expand section

### Step 3: For Each Workstream, Update Data

#### 3a. Query Aha! for Release Data
Use `mcp__aha__aha_graphql` with this query pattern:
```graphql
query {
  release(id: "RELEASE-ID") {
    referenceNum
    name
    progress
    workflowStatus { name }
    features {
      nodes {
        referenceNum
        name
        workflowStatus { name }
        originalEstimate
        percentDone
      }
    }
  }
}
```

Extract from response:
- `progress` → Update progress bar (0-100)
- `workflowStatus.name` → Update status pill
- `features.nodes` → Update child features list

#### 3b. Query Jira for Due Date
Use `mcp__atlassian__searchJiraIssuesUsingJql` or `mcp__atlassian__getJiraIssue` to get the target due date field.

Common due date fields:
- `dueDate` or `duedate`
- Custom field for "Target Due Date"

#### 3c. Update the Cell HTML

**Progress Bar**: Replace `<ri:attachment ri:filename="progress-bar-XX.svg"/>` with the new percentage.
- Ensure the SVG attachment exists on the page
- If missing, create and upload it:
```bash
# Create SVG file
cat > /tmp/progress-bar-XX.svg << 'EOF'
<svg width="200" height="20" xmlns="http://www.w3.org/2000/svg">
  <rect width="200" height="20" rx="5" fill="#e0e0e0"/>
  <rect width="[percentage*2]" height="20" rx="5" fill="#36B37E"/>
</svg>
EOF
# Upload to page
confluence attachment-upload <page_id> --file /tmp/progress-bar-XX.svg
```

**Status Pill**: Update the `<ac:structured-macro ac:name="status">` element:
- Map Aha status to Confluence colors:
  - "Shipped" / "Deployed To Production" / "Live" → Purple
  - "In Progress" / "In Development" → Yellow
  - "Planned" / "Not Started" → Blue
  - "On Track" → Green
  - "At Risk" → Yellow
  - "Off Track" → Red

**Target Due Date**: Update the date text after "Target Due Date:"

**Child Features**: In the "Child Features:" expand section, create a bullet list:
```html
<ul>
  <li><ac:structured-macro ac:name="status">...</ac:structured-macro>
      <a href="https://ciscosecurity.aha.io/features/FEATURE-ID">FEATURE-ID</a> -
      Feature Name (Status) | <a href="JIRA-URL">Jira</a></li>
  ...
</ul>
```

### Step 4: Sort Cells by Due Date

Under each major heading (e.g., "Duo", "Secure Access", "CII"):
1. Identify all table cells belonging to that section
2. Parse their target due dates
3. Sort by:
   - Items with dates: earliest first
   - Items marked [Stretch Goal]: at the bottom
   - Items with no date: at the very bottom (above stretch goals)
4. Reorder the table cells in the HTML

### Step 5: Calculate Overall Progress

1. Extract all progress percentages from workstream cells
2. Exclude [Stretch Goal] items from the calculation
3. Calculate average: `sum(percentages) / count`
4. Round to nearest integer
5. Update the header: `XX% Overall Progress (N workstreams)`

### Step 6: Upload Updated Page

```bash
confluence update <page_id> --file /tmp/progress-report-updated.html --format storage
```

## Status Mapping Reference

| Aha! Status | Confluence Color |
|-------------|-----------------|
| Shipped | Purple |
| Deployed To Production | Purple |
| Live | Purple |
| In Progress | Yellow |
| In Development | Yellow |
| Development Complete | Green |
| Planned | Blue |
| Not Started | Blue |
| On Hold | Grey |
| At Risk | Yellow |
| Off Track | Red |

## Progress Bar SVG Template

```svg
<svg width="200" height="20" xmlns="http://www.w3.org/2000/svg">
  <rect width="200" height="20" rx="5" fill="#e0e0e0"/>
  <rect width="[PERCENTAGE * 2]" height="20" rx="5" fill="#36B37E"/>
</svg>
```

Width of the green bar = percentage × 2 (since total width is 200px for 100%)

## Error Handling

- If Aha! release not found: Keep existing data, log warning
- If Jira issue not found: Keep existing due date, log warning
- If progress bar SVG missing: Create and upload it
- If page update fails: Report error, do not retry destructively

## Example Usage

User: `/weekly-progress-report https://cisco-sbg.atlassian.net/wiki/spaces/dev/pages/1431935243`

Claude will:
1. Verify prerequisites (Aha MCP, Confluence CLI, Atlassian MCP)
2. Read the page
3. For each workstream: fetch Aha progress/status/children, fetch Jira due date
4. Update all cells with new data
5. Sort cells by due date under each heading
6. Calculate and update overall progress
7. Upload the updated page
8. Report summary of changes made
