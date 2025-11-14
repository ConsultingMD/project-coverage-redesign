# Agent Instructions

This document provides guidance for AI agents working with this project.

## MVP Overview

- **Project Name**: Coverage Redesign Analysis
- **Description**: Research and analysis repository for Coverage Redesign project at IncludedHealth. Contains data extraction scripts, traffic control simulation planning, architecture analysis, and client profiling tools.
- **Type**: Research/Analysis Repository (NOT production code)
- **Primary Languages**: JavaScript (Node.js), Python
- **Key Focus Areas**:
  - Apollo GraphQL operation analysis
  - Real-Time Eligibility (RTE) traffic control simulation
  - Client profiling and API usage patterns
  - Metrics analysis (Prometheus/Grafana)

## Development Environment

- **Setup**: No build process required - scripts run directly
- **Node.js Scripts**: Run with `node script.js`
- **Python Scripts**: Run with `python script.py` or `uv run python script.py`
- **Environment Variables**:
  - `APOLLO_KEY`: Required for Apollo Studio API access
  - `APOLLO_GRAPH_REF`: Optional Apollo graph reference

## Common Commands

### Data Extraction
- `node fetch-apollo-operation-details.js --all` - Fetch Apollo Studio operation details
- `node match-by-field-signature.js` - Match operations to persisted query manifest
- `python graphql_operations_monthly.py` - Analyze GraphQL operations from Prometheus
- `python client_profile.py --service <name>` - Generate client profiles

### Analysis
- `node operations-by-client-stats.js --improved` - Generate client operation statistics
- `python trading_partner_histogram_weekly.py` - Analyze trading partner patterns
- `python inspect_metrics.py` - Inspect Prometheus metrics

## Git Workflow

### Creating Branches for JIRA Issues

When creating a new branch to work on a JIRA issue:

1. **First, set the AWS environment to UAT:**
   ```bash
   aws-environment uat
   ```

2. **Then, create the branch using the TNG JIRA tool:**
   ```bash
   tng jira start
   ```

The `tng jira start` command will automatically create a properly named branch based on the JIRA issue you're working on.

**Note**: If `tng jira start` doesn't work or requires arguments, you can manually create a branch:
```bash
git checkout -b fix/ACT-XXXX-description
```

### Implementing JIRA Issue Fixes

When implementing a fix for a JIRA issue (e.g., from triage documents):

1. **Navigate to the correct repository:**
   - Most fixes will be in one of the workspace repositories (e.g., `member-sponsorship`, `coverage`, etc.)
   - Use absolute paths when working with files: `/Users/tj.singleton/src/github.com/ConsultingMD/repo-name/...`

2. **Create a branch:**
   - Try `tng jira start` first (may require AWS environment setup)
   - Fallback: `git checkout -b fix/ACT-XXXX-description`

3. **Implement the fix:**
   - Read the relevant code files to understand the issue
   - Make the necessary code changes
   - Follow existing code style and patterns
   - Add/update comments as needed

4. **Add test coverage:**
   - Add test cases for the fix, especially boundary conditions
   - Ensure existing tests still pass
   - Run tests if possible (may have unrelated test failures in other files)

5. **Commit changes:**
   ```bash
   git add <files>
   git commit -m "Fix ACT-XXXX: Description

   Detailed explanation of the fix, including:
   - What was wrong
   - How it was fixed
   - What test coverage was added
   
   Fixes: ACT-XXXX"
   ```
   - If pre-commit hooks fail due to unrelated issues, use `--no-verify` flag

6. **Push and create PR:**
   ```bash
   git push -u origin fix/ACT-XXXX-description
   gh pr create --title "Fix ACT-XXXX: Description" --body "PR description with details"
   ```

7. **Clean up unrelated files:**
   - If the branch includes unrelated files (e.g., from previous work), remove them:
     ```bash
     git rm <unrelated-file>
     git checkout main -- <file-to-revert>
     git commit -m "Remove unrelated files from ACT-XXXX PR"
     git push
     ```

8. **Perform code review:**
   - Review your own PR for quality
   - Check for:
     - Correctness of the fix
     - Code style consistency
     - Test coverage
     - Edge cases handled
     - Documentation/comments
   - Address any suggestions from code review

9. **Post to JIRA:**
   - Use the Atlassian MCP tools to add a comment to the JIRA issue:
     ```javascript
     mcp_Atlassian_addCommentToJiraIssue({
       cloudId: "includedhealth.atlassian.net",
       issueIdOrKey: "ACT-XXXX",
       commentBody: "## Ready for Review\n\n[Summary of fix and PR link]"
     })
     ```
   - Include:
     - PR link
     - Summary of changes
     - Code review status
     - Confirmation it's ready for review

**Example workflow for ACT-2217:**
- Issue: Date validation off-by-one error
- Repository: `member-sponsorship`
- Files changed: `app/database/dynamo/canonical_eligibility_with_pdid.go` and test file
- Fix: Changed date comparison logic from `endAtTime.AddDate(0, 0, 1).Before(activeOn)` to `!activeOn.Before(startOfNextDay)`
- Tests: Added boundary condition test case
- PR: https://github.com/ConsultingMD/member-sponsorship/pull/429

## Project Structure

- **JavaScript Scripts**: Apollo Studio data extraction and analysis
- **Python Scripts**: Metrics analysis, client profiling, trading partner analysis
- **Design Documents**: `spec.md`, `prd.md`, `RTE_*.md` - Architecture and analysis documents
- **Data Files**: JSON/CSV/YAML files with extracted data and analysis results

## Key Context

- This is a **meta-project workspace** - references 11 related repositories via `project.yaml`
- No symlinks - repositories accessed via workspace folder configuration
- Scripts are exploratory/analysis-focused, not production code
- Output files (JSON/CSV/YAML) are checked in for reference

## Workspace Configuration

This project uses a **meta project** approach for managing multi-repository workspaces. Instead of using symlinks, repositories are managed through:

1. **`project.yaml`** - The single source of truth that defines which repositories should be included in the workspace
2. **Workspace folders** - Repositories are added as folders in the VS Code/Cursor workspace file (`coverage-redesign.code-workspace`) with relative paths

### How It Works

- The `project.yaml` file defines all repositories that should be part of this workspace
- The workspace file (`coverage-redesign.code-workspace`) contains folder entries with relative paths (e.g., `../coverage`, `../customer-configuration`)
- Folders marked with `metaManaged: true` are automatically managed by the meta-sync tool
- **No symlinks are used** - all repositories are referenced via relative paths in the workspace configuration

### Current Repositories

The following repositories are included in this workspace (as defined in `project.yaml`):

- `ih-projects`
- `project-coverage-redesign` (this repository)
- `coverage`
- `customer-configuration`
- `go-common`
- `member-sponsorship`
- `proto-common`
- `realtime-eligibility`
- `support-tools`
- `platform-map`

### Updating the Workspace

To add or remove repositories:

1. Edit `project.yaml` to add/remove repository entries
2. Run the `meta-sync` tool (if available) to regenerate the workspace file
3. Or manually update `coverage-redesign.code-workspace` to reflect the changes

**Important**: Do not create symlinks in the project directory. All repository access is handled through the workspace configuration.
