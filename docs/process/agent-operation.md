# Agent operation policy

Project-specific operating policy for agents working on ANJIA AJL33PQ0866 / FH8626V100.

Universal interaction, terminal, review and reverse methodology lives in `ArthurKoba/ai-agent-workflow`; do not duplicate it here.

## Git workflow

- Agents do not create pull requests. Repository owner creates PRs unless the owner explicitly changes this policy.
- Keep production/integration and current development lines obvious.
- Use topic branches only for substantial/risky isolation, not every micro-fix.
- Fold completed topic work back into the development line after the applicable gate and remove redundant live refs.
- Historical states belong in tags/SHAs/evidence, not as competing active branches.
- Do not force-update shared/submission branches without explicit owner authorization.
- Curate upstream contribution history from a verified base into coherent commits.

Current branch roles and related-repository refs are recorded in `STATE.md` and `docs/process/upstream-integration.md`, not duplicated here.

## Project authority

- GitHub: current camera-level source, docs, contracts, state and evidence manifest.
- External evidence store: heavy/unique primary evidence referenced by manifest.
- Ghidra through Koba MCP Bridge: canonical mutable reverse-analysis workspace.

Exact current locators belong in `STATE.md`, evidence docs and reverse-analysis docs.

## Tool surface

This project is browser/API-first.

Primary operations should use Koba MCP Bridge where a suitable capability exists:
- GitHub mutation/read/review;
- Ghidra reverse tooling;
- artifact/evidence operations;
- structured HTTP/cURL;
- other project-enabled infrastructure capabilities.

Do not switch to another GitHub connector or build a manual workaround merely because a Bridge capability/permission is missing. Record the capability gap and use an alternate path only when the owner/project explicitly allows it.

## Heavy build / hardware boundary

Do not clone/materialize full repositories or build kernel/Firmware/Buildroot/Docker/toolchains merely to inspect state.

Authoritative heavy builds and physical hardware runs belong to the owner/local build surface unless explicitly delegated.

Missing build/hardware evidence is a pending gate, not permission to simulate it.

## Reverse boundary

Use the canonical Ghidra project for mutable analysis.

Promote durable camera facts/contracts into Git documentation/source.

Do not commit Ghidra databases or bulk generated reverse exports.

Cross-Fullhan material is semantic reference only until FH8626 target evidence confirms the contract.

## Evidence

Heavy primary bytes stay outside source Git and are indexed through `evidence/MANIFEST.tsv`.

A historical SHA without a current manifest locator is provenance, not an automatically retrievable dependency.

Do not promote static/source/reverse results to hardware acceptance.

## Coordination

This repository is the camera-level coordination authority.

When a related repository changes material active SHA/topology/ownership/runtime target/composition/validation gates, synchronize `STATE.md`, `TASKS.md` and relevant docs in the same working iteration.

Do not put cross-project coordination AGENTS/audit metadata into component repositories.

## Upstream

Before OpenIPC-related implementation or contribution work:
1. read `docs/process/openipc-upstream-rules.md`;
2. reopen the live upstream links it records;
3. inspect the target repository’s own README/AGENTS/CLAUDE/contribution files/current base;
4. update the cached local summary if upstream rules changed.

Upstream live rules win over this repository’s cached summary.

## Review

Serious architecture, multi-repository, boot/kernel/storage/hardware-critical, infrastructure/permission or upstream-ready changes require an independent Reviewer pass under the universal role model.

Implementer self-check is useful but is not independent review.
