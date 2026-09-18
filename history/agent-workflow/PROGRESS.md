# Прогресс аудита исторических чатов

Статус: `CHAT_005_POST_REFRESH_PENDING`

## Текущее состояние

- Рабочая ветка: `audit/agent-workflow-history`
- Обработано исторических файлов: **5**
- Последний источник: `CHAT-005`
- Период последнего источника: **2026-08-24 — 2026-08-25**
- Следующее действие: выполнить обязательный post-file refresh
- Raw chat exports в Git **не сохраняются**
- Базовый промпт сохранён неизменным в `BASELINE_PROMPT.md`

## Пять направлений

| Направление | Файл | Состояние после CHAT-005 |
|---|---|---|
| Учёт файлов и непрерывность | `PROGRESS.md` | 5/?? источников обработано |
| Ошибки/нарушения агентов | `ERRORS.md` | 21 tracked classes/directions |
| Улучшения и best practices | `IMPROVEMENTS.md` | 30 tracked improvements/directions |
| История реверса/портирования | `CHRONOLOGY.md` + `CHRONOLOGY_DETAILS.md` | backfill начальной hardware/recovery/TFTP/RAM-boot фазы |
| Эволюция агентной разработки | `AGENTIC_DEVELOPMENT.md` | A0 и A4 дополнены самым ранним manual-chat/TFTP этапом |

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
| 3 | `CHAT-003` | 2026-08-26 — 2026-08-27 | `DONE` | Исторический backfill: v3.8→v4.0.4, persistent owner/hot reload, deterministic test lessons, boot automation, AE feedback, отказ от live MMIO rollback; по source numbering пропущенный/смещённый #2 считается закрытым и отдельно не ожидается |
| 4 | `CHAT-004` | 2026-08-26 — 2026-08-27 | `DONE` | Dequeue/release semantics и движущаяся stream queue; grey-frame локализован выше encoder/upscale; full Apollo + stock runtime bundle; переход к self-service reverse. Поздняя часть частично перекрывает CHAT-003 и использована только как дополнительное evidence |
| 5 | `CHAT-005` | 2026-08-24 — 2026-08-25 | `DONE` | Самый ранний backfill: hardware/dual-lens identification, immutable full-flash dump, U-Boot access, TFTP→RAM proof и выбор hybrid stock-kernel + OpenIPC initramfs strategy |

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

## Что CHAT-003 добавил к картине

### Новые уникальные ошибки
- физический A/B test нельзя строить на ручном отсчёте секунд и неоднозначных командах;
- test artifact нельзя отдавать в hardware loop без доступного compile/preflight;
- будущие fallback-сценарии не должны вытеснять выбранный основной dev path.

### Усиленные существующие классы
- потеря уже имеющихся dumps/runtime state;
- лишняя проверочность и остановки между очевидными действиями;
- intrusive live-MMIO experimentation;
- опасность второго owner;
- избыточные SHA/служебная информация при передаче файлов.

### Новые/уточнённые best practices
- persistent owner + reloadable algorithm plugin;
- автоматизированные `openipc_boot/stock_boot`;
- deterministic per-state evidence capture;
- hardware-proven baseline + regression isolation;
- compile/self-test before hardware delivery.

### Историческая роль
Этот источник хронологически заполняет разрыв перед `CHAT-002`: именно здесь появляется стабильный experimental substrate, на котором затем стало возможно быстро проверять source-derived CB970 stages и параллелить reverse.

### Нумерация входных файлов
Пользователь указал, что исходный файл/индекс `#2` утерян либо историческая нумерация смещена. Не ждать отдельный `#2` и не создавать для него placeholder; текущая выгрузка `#3` обработана как audit ID `CHAT-003`.

## 3-file live-state refresh после CHAT-003

Выполнено после третьего обработанного источника.

Проверено:
- актуальные `STATE.md` и `TASKS.md` на `main`;
- текущие branch roles reverse/Firmware/Builder/Linux/Divinus/U-Boot;
- современная authority-модель GitHub / Google Drive / Ghidra MCP.

Современное состояние согласуется с уже записанным anchor:
- GitHub — authority для текущего source/state/contracts;
- Drive — heavy/unique primary evidence;
- Ghidra MCP — canonical mutable reverse workspace.

Это подтверждает **конечное текущее состояние на 2026-09-18**, но не датирует исторические переходы к Drive/GitHub/MCP. Эти переходы по-прежнему должны быть найдены в следующих исторических чатах.

Живые ветки, релевантные текущему проекту, также существуют в ожидаемой topology: reverse `main/work/fh8626v100`, Firmware shared/runtime lines, Builder `work/fh8626v100-anjia`, Linux `fullhan-fh8626v100/work/fh8626v100`, Divinus `work/fh8626v100`, U-Boot native + stock-compatible lines.

## Что CHAT-004 добавил к картине

### Новые уникальные ошибки
- команды должны быть даны для фактического terminal lane: WSL / UART / U-Boot;
- recursive search должен быть заранее ограничен, иначе binary/web noise засоряет вывод и контекст;
- baseline-запрет на физический перенос shell-команд через `\` реально нарушался в этой выгрузке.

### Сильные подтверждения существующих классов
- adaptive granularity: опасное/ветвящееся — пошагово, routine build→scp→run — одним этапом;
- не использовать Python вместо простых стандартных инструментов без необходимости;
- не просить заново artifact, уже переданный в текущую историю;
- handoff/artifact mutation только в согласованном scope;
- execution-first: после достаточного evidence переходить к реализации, а не продолжать мелкие A/B.

### Новые улучшения
- полный searchable disassembly/binary bundle вместо серии ручных `objdump/grep`;
- один comprehensive read-only stock evidence capture с последующим offline analysis;
- стандартные domain tools (`ffmpeg/ffprobe`) вместо временных parser-ов;
- явное разделение WSL / UART / U-Boot команд.

### Технический переход
`CHAT-004` закрывает gap раннего H.264: queue consume исправлен через точную semantics `PAE release + MEDIA query`, после чего серый кадр локализован выше encoder/VPU. Далее найдены GC1054 scene profiles и stock `LoadIspParam → Run` lifecycle, что подготовило более зрелый ISP runtime reverse в `CHAT-003/002`.

### Пересечение источников
Поздняя часть `CHAT-004` содержит материал, уже наблюдавшийся в `CHAT-003` (v3.x/v4.0.x, persistent owner, boot automation, AE probes). Эти эпизоды не добавлены второй раз в хронологию и используются только как повторное подтверждение соответствующих E/I-классов.

### Исторические переходы, которых всё ещё нет
- Google Drive как evidence store;
- GitHub как project authority;
- Ghidra/MCP как shared reverse workspace.

Их нельзя датировать по первым четырём источникам.

## Что CHAT-005 добавил к картине

### Новых error-ID почти не потребовалось
Источник ранний и в основном подтверждает уже найденные позднее проблемы:
- лишняя проверочность после достаточного доказательства;
- ненужная ancillary-диагностика вместо functional-path test;
- склонность заранее разворачивать слишком много будущих веток.

Самый чистый эпизод: после успешного TFTP агент предложил ещё memory display/checksum, а пользователь остановил это и потребовал сразу переходить к OpenIPC build.

### Новые/уточнённые best practices
- полный flash dump до любой мутации как immutable recovery/provenance anchor;
- прямой functional test нужного transport вместо починки необязательной проверки;
- пользователь может отдавать raw bootlog/dump, а анализ и структурирование должны оставаться на агенте;
- hybrid stock-kernel + OpenIPC initramfs был выбран как сознательная стратегия ещё до первого OpenIPC hardware pass.

### Историческая роль
Это самый ранний обработанный источник на данный момент. Он предшествует `CHAT-001` и показывает исходный A0 workflow: браузерный чат, UART/programmer/U-Boot у пользователя и практически вся инженерная continuity внутри одного диалога.

### Что по-прежнему отсутствует
В `CHAT-005` ещё нет:
- устойчивого workspace/handoff как общей системы;
- Google Drive evidence store;
- GitHub authority для собственного FH8626 проекта;
- MCP/Ghidra shared reverse workspace.

## Следующее действие

Выполнить обязательный post-file refresh. После него ожидать следующий исторический источник. Расширенная live-state сверка будет после шестого обработанного файла либо раньше при противоречии.
