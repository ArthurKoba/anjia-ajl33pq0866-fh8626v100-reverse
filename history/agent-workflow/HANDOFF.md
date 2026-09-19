# Handoff: FH8626 historical case-study audit

Canonical location:
`main/history/agent-workflow/`

Source audit development was performed on:
`audit/agent-workflow-history`

The historical result is now merged into `main` as project history. The old audit branch is provenance/work history, not the current authority for these documents.

## Scope

This directory owns only:
- historical source registry;
- FH8626/ANJIA technical chronology;
- project-specific workflow case-study details.

Universal agent methodology lives in:

https://github.com/ArthurKoba/ai-agent-workflow

Before changing universal prompts, roles, audit rules, MCP strategy, terminal-interaction policy or generalized best practices, work in that repository instead.

## When to read this history

Do **not** load the full historical audit during normal implementation/startup.

Read it when:
- auditing agent behavior or workflow quality;
- onboarding into why the current project architecture exists;
- investigating an old decision/dead end;
- extending the historical chronology;
- extracting another reusable lesson for the universal workflow library.

## Historical-audit startup

1. Read this file.
2. Read `PROGRESS.md`.
3. Read `CHRONOLOGY.md`.
4. Read `CHRONOLOGY_DETAILS.md` only as needed.
5. For reusable agent-workflow analysis, read the universal workflow repository `AGENTS.md` and route through its workflow-audit skill.

## Raw sources

Raw chat exports remain input-only:
- do not commit them;
- do not copy secrets/private identifiers;
- use neutral IDs `CHAT-NNN`;
- deduplicate overlapping exports.

## Processing a new FH8626 historical source

- determine whether content is unique;
- assign the next neutral CHAT ID only for unique content;
- update the source registry;
- add only meaningful project chronology;
- send generalized workflow lessons to the universal audit registry;
- do not recreate local universal ERRORS/IMPROVEMENTS/prompt files.

Use a short-lived audit/topic branch for a substantial new historical batch if isolation is useful; merge the resulting documentation back into `main` after review.

## Live-state rule

Historical state must not be confused with current project state.

When a source refers to modern branches/tools, verify current state only if needed to understand the historical claim; do not retroactively rewrite history.
