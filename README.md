# Project Coverage Redesign

This is a research and analysis repository for the Coverage Redesign project at Included Health. It contains data extraction scripts, traffic control simulation planning, architecture analysis, and client profiling tools.

## Overview

This repository contains:

1. **Data extraction scripts** (Node.js/Python) for analyzing Apollo GraphQL operations and traffic patterns
2. **Traffic control simulation planning** for the Real-Time Eligibility (RTE) service
3. **Architecture analysis** of the RTE service request chains and dependencies
4. **Client profiling tools** for understanding API usage patterns

This is NOT a production codebase - it's a collection of analysis tools, data files, and design documents.

## Quick Start

See [CLAUDE.md](CLAUDE.md) for detailed project information and usage instructions.

## Key Documents

- [CLAUDE.md](CLAUDE.md) - Project overview and development guide
- [AGENTS.md](AGENTS.md) - Workspace configuration for AI agents
- [PROJECT.md](PROJECT.md) - Auto-generated project context (created by `ih-project sync`)

## Repository Structure

- `*.js` - Node.js scripts for Apollo Studio data extraction
- `*.py` - Python scripts for metrics analysis and client profiling
- `*.json` - Extracted data and analysis results
- `*.yaml` - Configuration and definition files
- `docs/` - Additional documentation

## Related Repositories

This repository is part of a multi-repository workspace managed through `project.yaml`. See [AGENTS.md](AGENTS.md) for workspace management details.

