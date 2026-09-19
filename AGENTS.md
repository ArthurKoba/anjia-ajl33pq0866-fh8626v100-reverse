# AGENTS.md

Mandatory map for agents entering this repository.

Account-level and ChatGPT Project prompts are assumed to be injected by the harness. **Do not reread prompt source files from Git as a startup requirement.**

Universal agent workflow authority:
`ArthurKoba/ai-agent-workflow`

## Start here

1. `README.md` — repository purpose and layout.
2. `STATE.md` — current engineering state.
3. `TASKS.md` — current actionable work.
4. `ROADMAP.md` — staged direction when broader planning context is needed.
5. `docs/README.md` — subsystem/document map.
6. `docs/process/agent-operation.md` — project-specific Git/tool/build/coordination operating rules.

## Route by task

### Reverse / hardware analysis
Read:
- `docs/architecture/reverse-analysis.md`
- relevant subsystem docs
- universal `skills/hardware-reverse/README.md`

Canonical mutable reverse surface is defined in the project docs; do not invent a second reverse workspace.

### Related OpenIPC repository work
Read:
- `docs/architecture/repositories.md`
- `docs/process/upstream-integration.md`
- `docs/process/openipc-upstream-rules.md`
- live upstream rules linked there
- universal `skills/software-engineering/README.md`

### Build / flash / target operation
Read:
- `docs/process/build-and-flash.md`
- `docs/process/operator-interaction.md`
- `docs/process/stock-restore-safety.md` when recovery/destructive work is relevant
- universal `skills/terminal-operations/README.md`

### Evidence
Read:
- `evidence/README.md`
- `evidence/MANIFEST.tsv`

### Source components
Read:
- `source/fh8626v100/components/README.md`

### Independent review
Use the universal Reviewer role and `skills/code-review/README.md`.

### Historical FH8626 workflow case study
Read:
- `history/agent-workflow/README.md`

Generalized agent errors/best practices/prompts no longer live here; they are maintained in `ArthurKoba/ai-agent-workflow`.

## Where findings belong

- current camera/subsystem facts → `docs/`
- actionable current state → `STATE.md` / `TASKS.md`
- external primary evidence index → `evidence/`
- reusable camera-specific source/contracts → `source/`
- historical project chronology/provenance → `history/`
- universal agent methodology → `ArthurKoba/ai-agent-workflow`
- implementation changes → the repository that naturally owns that component

Keep this file a map. Do not grow it back into a duplicate policy manual.
