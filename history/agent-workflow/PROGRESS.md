# Прогресс аудита исторических чатов

Статус: `READY_FOR_CHAT_003`

## Текущее состояние

- Рабочая ветка: `audit/agent-workflow-history`
- Обработано исторических файлов: **2**
- Последний источник: `CHAT-002`
- Период последнего источника: **2026-08-27**
- Следующее действие: получить `CHAT-003`
- Raw chat exports в Git **не сохраняются**
- Базовый промпт сохранён неизменным в `BASELINE_PROMPT.md`

## Пять направлений

| Направление | Файл | Состояние после CHAT-002 |
|---|---|---|
| Учёт файлов и непрерывность | `PROGRESS.md` | 2/?? источников обработано |
| Ошибки/нарушения агентов | `ERRORS.md` | 15 подтверждённых классов/статусов |
| Улучшения и best practices | `IMPROVEMENTS.md` | 15 подтверждённых улучшений |
| История реверса/портирования | `CHRONOLOGY.md` + `CHRONOLOGY_DETAILS.md` | добавлена фаза source-derived runtime + parallel heavy reverse |
| Эволюция агентной разработки | `AGENTIC_DEVELOPMENT.md` | A0/A1/A4 наблюдаются; A4.5 consolidated; A4.6 observed |

## Обязательный цикл для каждого следующего файла

### 1. Refresh
Перед новым источником перечитать:
- `HANDOFF.md`
- `PROGRESS.md`
- `ERRORS.md`
- `IMPROVEMENTS.md`
- `CHRONOLOGY.md`
- `CHRONOLOGY_DETAILS.md`
- `AGENTIC_DEVELOPMENT.md`

После каждых 3 файлов или при противоречии дополнительно освежить актуальные `STATE.md`, `TASKS.md` и relevant branches проекта.

### 2. Register
Назначить нейтральный ID `CHAT-NNN` и поставить `IN_PROGRESS`. Не коммитить исходный файл.

### 3. Analyze
Искать замечания пользователя, ошибки агента, улучшения workflow, технические этапы и изменения распределения ответственности человек ↔ агент.

### 4. Consolidate
Повторные эпизоды не превращать в новые ID без новой корневой причины.

### 5. Update
Обновить только затронутые журналы. Основную `CHRONOLOGY.md` держать короткой; подробности — в companion-файле.

### 6. Handoff
Состояние после каждого источника должно быть продолжабельным без устного контекста.

## Реестр источников

| № | ID | Период | Статус | Основной вклад |
|---:|---|---|---|---|
| 1 | `CHAT-001` | 2026-08-24 — 2026-08-26 | `DONE` | Первая FH8626 bring-up фаза: safe RAM boot, OpenIPC userspace, vendor media stack, ISP/PAE/H.264, dev-loop SSH, checkpoints/handoff; выявлен баланс пошаговости, transport/state/context ошибки |
| 2 | `CHAT-002` | 2026-08-27 | `DONE` | Source-derived ISP runtime, формальный one-archive delivery protocol, self-guarded owner launch, hot-plugin loop, workspace authority cleanup, role-specialized parallel reverse |

## Что CHAT-001 изменил в исходных гипотезах

### Подтверждено
- избыточная пошаговость и лишние проверки действительно были проблемой;
- при этом пошаговость нужна там, где вывод меняет решение;
- SCP требует target-specific режима, а transport зависит от stock/OpenIPC state;
- WSL/build environment нужно делать воспроизводимым;
- потеря текущего runtime context — один из самых дорогих классов ошибок;
- handoff/checkpoints реально стали ранним механизмом continuity;
- deep reverse нужно останавливать, когда практический эксперимент уже разблокирован.

### Пока не подтверждено
- переход workspace → Google Drive;
- переход Drive → GitHub authority;
- MCP как рабочая инфраструктура;
- специальный env-паттерн для значений, которые теряются/искажаются при вставке.

Эти темы нельзя записывать в историю как состоявшиеся до появления соответствующих чатов.

## Контроль дедупликации после CHAT-001

Проверено:
- SCP/SFTP и «stock без SSH» объединены в один transport class, а не разбиты на разные ошибки;
- потеря stock/OpenIPC state выделена отдельно от конкретной ошибки SCP, потому что корневая причина шире;
- «пошагово» и «без пошаговости» не записаны как противоречие — сформулировано адаптивное правило;
- лишний sensor disassembly и повторная проверка TFTP объединены как один класс low-value validation;
- camera hang от intrusive tracer выделен отдельно, потому что это уже experimental safety, а не просто лишняя проверка;
- изменение handoff без разрешения выделено отдельно как mutation-scope ошибка.

## Что CHAT-002 добавил к картине

### Новые подтверждённые ошибки
- явный workflow-протокол может быть нарушен даже сразу после его формулировки — его нужно применять как hard contract;
- критические команды должны быть идемпотентными/self-guarded, потому что console/paste может повторить запуск;
- длинная техническая экспозиция сама по себе стала проблемой: пользователь предпочитает execution-first и короткий progress report.

### Новые подтверждённые улучшения
- one-test-stage → one-versioned-archive + готовые WSL/camera blocks;
- hot-plugin validation поверх одного долгоживущего owner резко уменьшает reboot-cost;
- workspace должен иметь один authoritative reverse artifact, а повреждённые/дублирующие substrate удаляться;
- parallel agents эффективны при непересекающемся scope и finished integration units.

### Исторический переход
`CHAT-002` — первая явно оформленная **роль-специализированная параллельная разработка**: один агент держит canonical runtime/integration, другой делает deep reverse тяжёлых функций, пользователь пока вручную передаёт handoff-пакеты между ними.

### Пока всё ещё не подтверждено
- Google Drive как постоянное evidence-хранилище;
- GitHub как authority именно для текущего FH8626 engineering state;
- MCP/Ghidra как прямой shared workspace между агентами;
- отдельный устойчивый env-паттерн для значений, которые «съедает» терминал.

## Следующее действие

Получить `CHAT-003` и искать:
- дальнейшее подтверждение новых delivery/idempotency правил;
- момент появления постоянного внешнего хранилища;
- переход к GitHub authority и затем MCP/shared-agent infrastructure.
