# Agent Instructions

This document provides guidance for AI agents working with this project.

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
