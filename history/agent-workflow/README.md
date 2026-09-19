# Historical case study: FH8626V100 agent workflow

This directory stores the **project-specific historical case study** for the ANJIA AJL33PQ0866 / FH8626V100 reverse and porting effort.

Universal agent rules, prompt hierarchy, recurring error patterns, best practices, MCP strategy and agent-system evolution have moved to:

https://github.com/ArthurKoba/ai-agent-workflow

## Kept here

- `PROGRESS.md` — registry of processed historical chat sources.
- `CHRONOLOGY.md` — compact technical/project chronology.
- `CHRONOLOGY_DETAILS.md` — detailed project-specific history.
- `HANDOFF.md` — how to continue this case-study audit.

## Not authoritative here anymore

Do not maintain universal:
- account/project prompts;
- agent role model;
- error registry;
- best-practice registry;
- MCP/tooling strategy;
- generalized workflow evolution.

Those belong to `ArthurKoba/ai-agent-workflow`.

The original audit work was developed on `audit/agent-workflow-history`; the canonical historical documents are now merged into `main`. Git history preserves the former universal audit files for provenance, but they are no longer active authority.

## New historical source workflow

When another FH8626 chat/source arrives:

1. register/deduplicate it in `PROGRESS.md`;
2. update project chronology only if it adds real FH8626/project history;
3. if it reveals a reusable agent error/best practice/system evolution, update the universal AI workflow repository instead of recreating a local registry here;
4. never commit raw chat exports.

This keeps the camera repository focused on camera/project history while reusable agent methodology evolves independently.
