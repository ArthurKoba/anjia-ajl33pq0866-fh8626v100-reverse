# Handoff: FH8626 historical case-study audit

Repository:
`ArthurKoba/anjia-ajl33pq0866-fh8626v100-reverse`

Audit branch:
`audit/agent-workflow-history`

## Scope

This branch now owns only:
- historical source registry;
- FH8626/ANJIA technical chronology;
- project-specific case-study details.

Universal agent methodology has moved to:

`ArthurKoba/ai-agent-workflow`

Before changing universal prompts, roles, audit rules, MCP strategy, terminal interaction policy or best practices, work in that repository instead.

## Startup

1. Read this file.
2. Read `PROGRESS.md`.
3. Read `CHRONOLOGY.md`.
4. Read `CHRONOLOGY_DETAILS.md` only as needed.
5. For reusable agent-workflow analysis, read the universal repo `AGENTS.md` and route through its workflow-audit skill.

## Raw sources

Raw chat exports remain input-only:
- do not commit them;
- do not copy secrets/private identifiers;
- use neutral IDs `CHAT-NNN`;
- deduplicate overlapping exports.

## Processing a new FH8626 source

- determine whether content is unique;
- assign next neutral CHAT ID only for unique content;
- update the source registry;
- add only meaningful project chronology;
- send generalized workflow lessons to the universal audit registry;
- do not recreate local ERRORS/IMPROVEMENTS/prompt files.

## Live-state rule

Historical state must not be confused with current project state.

When a source refers to modern branches/tools, verify current state only if needed to understand the historical claim; do not retroactively rewrite history.
