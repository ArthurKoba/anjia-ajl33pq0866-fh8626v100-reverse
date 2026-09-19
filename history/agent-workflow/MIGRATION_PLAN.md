# Universal agent/infrastructure extraction plan

Status: `READY_FOR_REPO_CREATION`.

## Решение

Универсальные agent/workflow документы не должны оставаться частью ANJIA/FH8626 repository как действующая policy.

Нужны два отдельных repositories.

### Repository A — `ArthurKoba/ai-agent-workflow`

Назначение: vendor/project-neutral правила работы AI engineering agents.

Предлагаемая структура:

```text
README.md
AGENTS.md
prompts/
  BASE_PROMPT.md
  ACCOUNT_PROMPT.md
context/
  LOCAL_CONTEXT_CONTRACT.md
  LOCAL_AGENT_CONTEXT.example.md
roles/
  ROLE_MODEL.md
workflows/
  engineering.md
  hardware-reverse.md
templates/
  HANDOFF.md
  STATE.md
  TASKS.md
lessons/
  workflow-audit-summary.md
```

Туда переезжают:
- canonical Base Prompt;
- account-level compact prompt;
- local context contract;
- implementer/reviewer/orchestrator model;
- general workflow invariants;
- generalized hardware/reverse module;
- anonymized/generalized conclusions audit.

Не переезжают:
- FH8626 chronology;
- ANJIA GPIO/sensor facts;
- OpenIPC branch SHAs;
- camera-specific acceptance matrix.

### Repository B — `ArthurKoba/infrastructure` (предпочтительно private)

Назначение: личная agent/development infrastructure и её operational contract.

Предлагаемая структура:

```text
README.md
AGENTS.md
STATE.md
TASKS.md
docs/
  koba-mcp-bridge.md
  github-app-roles.md
  curl-http-policy.md
  ghidra.md
  artifact-storage.md
  local-runners.md
runbooks/
inventory/
  README.md
context/
  LOCAL_INFRA_CONTEXT.example.md
```

Туда переезжают:
- Koba MCP Bridge as primary surface;
- exact tool/preset conventions;
- Agent/Reviewer App roles;
- cURL preset policy;
- artifact/Ghidra infrastructure;
- local runner/build surface contracts;
- infra troubleshooting/runbooks.

Sensitive IPs/credentials/machine inventory — только private/local context.

### GitHub profile repository `ArthurKoba/ArthurKoba`

Не хранит rules.

Добавить только короткий section со ссылками:
- AI engineering workflow → `ai-agent-workflow`;
- personal infrastructure → `infrastructure`.

Profile README — discovery/index, не authority.

## Что остаётся в ANJIA repository

- конкретная история/аудит FH8626 как case study;
- camera/project `AGENTS.md`;
- STATE/TASKS;
- ownership/acceptance/hardware docs;
- ссылка на canonical universal workflow;
- ссылка на infrastructure docs, если агент использует Koba tooling.

После migration текущие universal prompt drafts в `history/agent-workflow` остаются только historical provenance либо заменяются коротким pointer.

## Account vs Project vs repository

### Account-level ChatGPT prompt
Только 10 универсальных interaction rules.
Не включать reverse methodology, Koba, OpenIPC, scp flags, IP/paths.

### ChatGPT Project instructions
Короткий project bootstrap:
- use Koba MCP Bridge;
- read repo AGENTS/STATE/TASKS;
- read canonical agent workflow;
- read infrastructure repo when touching infrastructure;
- reviewer separation for serious changes.

### Repository AGENTS.md
Repository-specific authority, ownership, contribution policy, required local context.

### Local context
Machine-specific mutable facts only.

## Legacy flags

Например `scp -O` — не account-level и не universal base rule.
Если конкретный target требует legacy SCP, это transport contract конкретного project/local context.

## Текущий blocker миграции

Koba GitHub App сейчас не даёт операции создания repository, а list-repositories endpoint в этой сессии возвращает transport error. Проверенные candidate repo names также не установлены для Agent App.

Поэтому сейчас подготовлен exact content/layout, но новые repositories не создаются обходным GitHub tool.

Следующий owner action:
1. создать/выбрать два repositories;
2. добавить их в Koba Agent + Reviewer App installations;
3. после этого агент переносит файлы штатно через Bridge;
4. затем ANJIA audit получает links/stubs, universal policy оттуда удаляется как active authority.
