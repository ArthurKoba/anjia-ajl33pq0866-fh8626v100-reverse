# Local / Project Context Contract

Статус: `PROPOSED`.

Назначение: отделить универсальные правила поведения агента от environment-specific данных, которые нельзя зашивать в общий base prompt.

## Рекомендуемая раскладка

### 1. Account / Project instructions
Хранят:
- универсальный Base Prompt v2;
- общие пользовательские предпочтения, которые относятся ко всем задачам данного Project;
- правило поиска project bootstrap и local context.

Не должны хранить:
- временный target IP;
- текущий PID/owner;
- случайный checkout path;
- быстро меняющийся branch SHA;
- одноразовый hardware state.

### 2. Repository `AGENTS.md`
Хранит:
- project/repository-specific startup order;
- authority files;
- ownership boundaries;
- contribution/tool policy;
- указание, нужен ли local context;
- ссылки на `STATE.md`, `TASKS.md`, process/runbook docs.

Не должен хранить приватные credentials и локальные секреты.

### 3. `STATE.md` / `TASKS.md`
Хранят:
- динамический current engineering state;
- active refs;
- current gates/blockers;
- доказанные hardware facts;
- actionable next work.

### 4. Local context override
Рекомендуемое имя: `LOCAL_AGENT_CONTEXT.md`.

Файл **не должен коммититься в публичный source repository**. Его можно:
- держать локально и gitignore;
- передавать через Project instructions/files;
- генерировать из локального bootstrap;
- предоставлять через доверенный connector/tool.

Коммитить можно только шаблон:
`LOCAL_AGENT_CONTEXT.example.md`.

## Что может содержать local context

Только environment-specific данные, например:

- `workspace_root: <local path>`
- `downloads_root: <local path>`
- `target_control_lane: <UART/SSH/...>`
- `target_address: <current address>`
- `file_transfer_method: <scp/tftp/...>`
- `toolchain_path: <local path>`
- `authoritative_build_surface: <owner WSL / CI / ...>`
- `artifact_storage: <local/external location>`
- `current_target_boot_mode: <state>`
- `current_owner_process: <state>`
- `local_tool_availability: <facts>`

Не хранить там пароли/tokens/private keys, если можно избежать этого.

## Startup contract

Repository/project bootstrap должен явно объявить одно из:

- `local_context: REQUIRED`
- `local_context: OPTIONAL`
- `local_context: NOT_USED`

Если local context REQUIRED и файл/источник недоступен, агент обязан до environment-specific действий сообщить:

> Local execution context не найден. Я могу продолжить repository/source analysis, но не буду угадывать локальные пути, target address, transport или build environment. Environment-dependent шаги будут менее надёжны до предоставления контекста.

После этого агент не должен повторять предупреждение в каждом сообщении.

Если local context OPTIONAL — работать дальше без предупреждения, пока реально не понадобится отсутствующий параметр.

## Precedence

При конфликте:

1. прямое текущее указание пользователя;
2. фактическое live state/tool/repository evidence;
3. repository `AGENTS.md` / project rules;
4. `STATE.md` / `TASKS.md`;
5. local context override;
6. старые handoff/chat summaries;
7. model memory/assumption.

Machine-specific значение никогда не должно побеждать свежий фактический state только потому, что было сохранено раньше.

## Template: LOCAL_AGENT_CONTEXT.example.md

```md
# Local agent context

status: CURRENT
updated: <date/time>

## Workspace
workspace_root: <path>
downloads_root: <path>
artifact_root: <path>

## Build
authoritative_build_surface: <description>
toolchain_path: <path or unavailable>

## Target
target_control_lane: <UART/SSH/...>
target_address: <address or unavailable>
file_transfer_method: <method>
boot_mode: <state>
active_owner: <process/state>

## Local tools
<tool>: <available/unavailable + notes>

## Notes
Only current environment facts. No project history here.
```

## Почему это лучше hardcoded base prompt

Base prompt остаётся переносимым между Windows/WSL/Linux, разными камерами и другими hardware projects.

Project-specific contracts живут рядом с кодом и версионируются.

Machine-specific state можно менять без загрязнения Git/history.

Если local context пропал, failure видимый и fail-closed: агент сообщает об ухудшенном контексте и перестаёт галлюцинировать пути/IP.
