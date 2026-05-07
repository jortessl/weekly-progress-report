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

**Child Features**: In the "Child Features:" expand section, replace the content with features from Aha.

Query Aha for child features using:
```graphql
query {
  release(id: "RELEASE-ID") {
    features {
      referenceNum
      name
      workflowStatus { name }
      progress
    }
  }
}
```

Replace the content between `Child Features:</ac:parameter><ac:rich-text-body>` and `</ac:rich-text-body></ac:structured-macro>` with:
```html
<ul>
  <li><p><ac:structured-macro ac:name="status" ac:schema-version="1">
    <ac:parameter ac:name="title">PROGRESS%</ac:parameter>
    <ac:parameter ac:name="colour">COLOR</ac:parameter>
  </ac:structured-macro>
  <a href="https://ciscosecurity.aha.io/features/FEATURE-ID">FEATURE-ID</a> -
  Feature Name (Status)</p></li>
  ...
</ul>
```

**Child feature percentage pill colors:**
- 0% → Blue
- 1-99% → Yellow
- 100% → Green

**IMPORTANT: Fix ALL colors across the entire page.**
After updating child features, scan the ENTIRE page for percentage status macros and fix any with incorrect colors. This catches colors from previous updates that may have been set incorrectly.

Pattern to find and fix (colour can appear before or after title):
```
<ac:parameter ac:name="colour">COLOR</ac:parameter><ac:parameter ac:name="title">XX%</ac:parameter>
<ac:parameter ac:name="title">XX%</ac:parameter><ac:parameter ac:name="colour">COLOR</ac:parameter>
```

**Sort child features by percentage descending:**
Within each cell, sort child features from highest percentage to lowest:
- 100% items at the top
- 0% items at the bottom

This makes it easy to see which features are complete vs still in progress.

If a release has no child features in Aha, display:
```html
<ul><li><p>No child features</p></li></ul>
```

**Status Update Date**: Update dates next to "Status Update:" headers.
- Extract the date from the page title (e.g., "05-08-2026 Agentic AI Progress Report" → May 8, 2026)
- Replace `[Date]` placeholders with the Confluence time macro:
```html
<time datetime="2026-05-08" />
```
- **Also replace existing date macros** from previous runs. Look for:
```html
<time datetime="YYYY-MM-DD" />
```
  and replace with the new date from the page title.
- This ensures the skill is idempotent - running it multiple times always updates to the current report date.
- The macro renders as a formatted date in Confluence (e.g., "May 8, 2026")

### Step 4: Sort Cells Within Each Section

Under each major heading (e.g., "Duo", "Secure Access", "CII"):

**Primary sort rules:**
1. **[Stretch Goal] items always go to the bottom** of their section
2. Non-stretch items are sorted by target due date (if available)

**Fallback when no target due date:**
If a workstream has no Jira link or no due date, fall back to sorting by:
1. **Progress percentage** (highest first)
2. **Aha Status** (deprioritize these statuses to the bottom):
   - "Under Consideration"
   - "Planned"
   - "Not Started"
   - "Researching"
   - "Evaluation"

**Sort priority (top to bottom):**
```
1. Non-stretch items with due dates → earliest date first
2. Non-stretch items without dates → highest % first, "active" statuses before "planned" statuses
3. [Stretch Goal] items → sorted by % descending
```

**Example sort key function:**
```python
def get_sort_key(item):
    # Stretch goals always at bottom
    stretch_order = 1 if item['stretch'] else 0

    # Deprioritize "waiting" statuses
    low_priority_statuses = ['under consideration', 'planned', 'not started', 'researching', 'evaluation']
    status_penalty = 1 if item['status'].lower() in low_priority_statuses else 0

    # 0% items go lower (but above low-priority statuses)
    zero_penalty = 0.5 if item['pct'] == 0 else 0

    # Higher percentage = higher priority (negative for descending)
    return (stretch_order, status_penalty, zero_penalty, -item['pct'])
```

### Step 5: Calculate Overall Progress

**CRITICAL: Correct Counting Logic**

Count ALL workstreams that have a progress bar, regardless of whether they have an Aha link.
Do NOT filter by presence of Aha links - some workstreams may not have Aha releases linked yet but still count toward the total.

**What to count:**
- Any table cell with a `progress-bar-XX.svg` attachment = 1 workstream
- Include workstreams even if they have no Aha link or no Jira link

**What to exclude:**
- Only exclude items with `[Stretch Goal]` or `[Stretch]` in the title
- These are excluded from BOTH the percentage calculation AND the workstream count

**Calculation steps:**
1. Find all table cells containing `progress-bar-XX.svg`
2. Extract the title from each cell's `<h3>` tag
3. If title contains `[Stretch` → exclude from calculation
4. For all non-stretch items: collect their percentages
5. Calculate: `average = sum(percentages) / count(non-stretch workstreams)`
6. Round to nearest integer
7. Update the header: `XX% Overall Progress (N workstreams)` where N = count of non-stretch workstreams

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

## Critical Best Practices

**ALWAYS re-read the page fresh before making any updates.**
Users may add, remove, or modify cells between your reads. Never rely on cached/stale page data from earlier in the conversation. Each update cycle should start with a fresh `confluence read`.

**Count workstreams by progress bars, not by Aha links.**
Some workstreams may not have Aha releases linked yet but still count toward the total.

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
