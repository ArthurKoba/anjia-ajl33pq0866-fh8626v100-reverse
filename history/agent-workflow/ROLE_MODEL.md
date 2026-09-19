# Agent role model

Status: `PROPOSED`.

## Implementer

Владеет:
- анализом задачи;
- source changes;
- small logical commits;
- локальными/static checks, доступными его tool surface;
- синхронизацией state/coordination docs.

Не выдаёт собственный self-review за независимую проверку.

## Reviewer

Владеет:
- independent read-only inspection;
- diff/commit/history review;
- contract/ownership/acceptance checks;
- поиском regression, missing gates и false claims;
- APPROVE / REQUEST_CHANGES / review notes.

Во время review не исправляет source тем же identity. Если нужны изменения — возвращает findings Implementer-у либо создаёт отдельный явно новый implementation cycle.

## Orchestrator

Нужен при нескольких параллельных owners/repos.

Владеет:
- scopes;
- authority/ownership;
- dependency graph;
- conflict resolution;
- fan-in;
- release/acceptance ordering.

Не заменяет specialist reverse/implementation только ради централизации.

## Когда отдельный Reviewer обязателен

По умолчанию для серьёзного изменения, если оно:
- меняет architecture/ownership;
- затрагивает несколько repos;
- hardware/boot/kernel/storage critical;
- destructive/recovery-sensitive;
- security/permissions/infrastructure related;
- готовится к upstream contribution/release;
- заменяет known-good hardware-proven contract;
- закрывает большой milestone как COMPLETE/production-ready.

Мелкая локальная правка может иметь self-review, если project policy не требует отдельного reviewer.
