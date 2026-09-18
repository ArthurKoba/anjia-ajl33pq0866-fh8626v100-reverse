# Прогресс аудита исторических чатов

Статус: `CHAT_010_IN_PROGRESS`

## Текущее состояние

- Рабочая ветка: `audit/agent-workflow-history`
- Обработано исторических файлов: **9**
- Последний источник: `CHAT-009`
- Период последнего источника: **2026-08-26**
- Следующее действие: завершить анализ `CHAT-010`
- Raw chat exports в Git **не сохраняются**
- Базовый промпт сохранён неизменным в `BASELINE_PROMPT.md`

## Пять направлений

| Направление | Файл | Состояние после CHAT-009 |
|---|---|---|
| Учёт файлов и непрерывность | `PROGRESS.md` | 9/?? уникальных источников обработано |
| Ошибки/нарушения агентов | `ERRORS.md` | 27 tracked classes/directions |
| Улучшения и best practices | `IMPROVEMENTS.md` | 38 tracked improvements/directions |
| История реверса/портирования | `CHRONOLOGY.md` + `CHRONOLOGY_DETAILS.md` | D5-D8 дополнены GPIO5 reset, H.264 capture, stateful lifetime и reproducible handoff |
| Эволюция агентной разработки | `AGENTIC_DEVELOPMENT.md` | manual local workflow limits зафиксированы; reproducibility/serial-script rules добавлены |

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
| 6 | `CHAT-006` | 2026-08-24 — 2026-08-25 | `DONE` | Перекрывает CHAT-005, но добавляет root-shell inventory, самостоятельное извлечение `/app` из SPI dump, media module baseline, первые ioctl ABI mappings и переход от ручного target inventory к artifact-assisted analysis |
| 7 | `CHAT-007` | 2026-08-25 — 2026-08-26 | `DONE` | Partial-overlap: префикс до clean-room fh_mpi повторяет CHAT-006 и не пересчитан. Новая часть: clean OpenIPC RAM baseline, полный stock media stack под OpenIPC, sensor/MIPI/VPU/PAE/ISP reverse, transport/context ошибки, external Fullhan references, documentation/repository design и parallel-agent checkpoint |
| 8 | `CHAT-008` | 2026-08-25 | `DONE` | Partial-overlap с CHAT-007 до ~L24104. Уникальная ветка: Apollo/ISP reverse, checkpoint recovery, adaptive granularity correction, parallel-agent merge, FH8852 semantic reference, ISP interrupt-mask breakthrough, ~25-fps sensor→ISP proof и SoC-first backend goal |
| 9 | `CHAT-009` | 2026-08-26 | `DONE` | Новый уникальный source (~13% exact-line overlap с CHAT-008 только в reused code/helper fragments): hardware H.264 capture + repeated descriptor, broken ISP unload/multi-owner lifetime, persistent daemon direction, SSH key/CRNG/PTTY dev-loop optimization, UART paste fragility и context-limit reproducibility handoff |
| 10 | `CHAT-010` | 2026-08-26 | `IN_PROGRESS` | Продолжение после handoff: PTY cold-boot closure, точный dequeue reverse (`5011`, `4D05/4D06`), full-disassembly self-service, livecapture ordering fix и отдельная SSH-regression ветка |

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

## Что CHAT-006 добавил к картине

### Новый уникальный error-class
- агент попросил пользователя вручную искать/передавать media modules, хотя полный SPI dump уже был доступен и позволял извлечь их автономно;
- после замечания пользователя работа была перенесена на уже имеющийся artifact.

### Усиленные существующие классы
- неподтверждённые process names: сначала предполагался `MainApp`, затем `noodles`, а фактический startup script показал основной binary `apollo`;
- лишние target-действия должны исчезать, если нужные bytes уже есть в dump/workspace.

### Новые/уточнённые best practices
- full firmware dump используется не только как recovery image, но и как offline development substrate;
- из одного dump можно автономно получить kernel/initramfs, `/app`, modules, sensor libraries и начать ABI reverse;
- перед новой target-командой сначала проверяется уже имеющийся corpus.

### Технический вклад
Источник добавляет ранний stock BSP baseline:
- фактические Fullhan media modules и порядок их роли;
- VMM reserved-memory contract;
- sensor/MIPI libraries;
- первые VPSS/VENC/media ioctl mappings;
- первый read-only standalone probe.

Это не отдельная поздняя глава, а детализация начальной D1-фазы до первого полноценного OpenIPC RAM pass.

### Пересечение с CHAT-005
Начальная часть `CHAT-006` почти повторяет `CHAT-005` (hardware/U-Boot/TFTP). Повторные эпизоды не продублированы; использована только новая часть после stock root access.

## 6-file live-state refresh после CHAT-006

Выполнено после шестого обработанного источника.

Проверено:
- актуальные `STATE.md` и `TASKS.md` на `main`;
- живые branch roles reverse/Firmware/Builder/Linux/Divinus/U-Boot;
- современная authority-модель GitHub / Google Drive / Ghidra MCP.

Текущий endpoint по смыслу не изменился:
- GitHub — authority для текущих source/docs/contracts/state;
- Google Drive — heavy/unique evidence;
- Ghidra MCP через Koba MCP Bridge — canonical mutable reverse workspace.

При этом параллельная работа действительно продолжается: некоторые work-branch tips уже сдвинулись относительно предыдущей 3-file сверки. Эти новые SHA считаются только современными locators и не переписывают историческую последовательность старых чатов.

Особенно важно: ранний `CHAT-006` описывает factory dump/module/ioctl reverse как файловый ручной workflow. Наличие современной Ghidra/GitHub инфраструктуры не используется ретроспективно как доказательство, что она существовала тогда.

## Дубликаты входных источников

- Полученный после `CHAT-006` файл с тем же исходным номером `#6` проверен как точный побайтовый дубль уже обработанного источника. Новый `CHAT-NNN` не назначен, счётчик обработанных уникальных источников остаётся **6**.
- Дубликаты не создают новые evidence entries, error examples или chronology events.

## Что CHAT-007 добавил к картине

### Новые уникальные error-классы
- отсутствие progress visibility на длинном reverse;
- прямое нарушение baseline: длинные shell-цепочки через `&&` / `;`;
- cwd-dependent команды без обязательного `cd`;
- предположение наличия desktop-утилит на минимальном BusyBox target.

### Сильные подтверждения существующих ошибок
- чрезмерный reverse после уже достаточного evidence;
- ручное редактирование source вместо готового rewrite/build блока;
- Python там, где достаточно shell/binutils;
- повторное забывание `scp -O` на OpenIPC;
- попытка использовать SSH/SCP на stock, где SSH отсутствует;
- потеря текущего stock/OpenIPC state;
- слишком широкий grep с огромным выводом.

### Новые/уточнённые improvements
- искать public/vendor/adjacent-SoC reference implementations до продолжения дорогого blind reverse;
- использовать cross-SoC material только как semantic map, а не как доказательство FH8626 ABI/MMIO;
- разделять current reverse docs, failed experiments/history, provenance и upstream deliverable;
- вести progress tracker по subsystem boundaries;
- historical precursor будущего artifact-delivery protocol: один готовый source-rewrite/build/transfer блок вместо ручного редактирования.

### Agentic transition
Впервые явно формулируется желание дать агенту прямой repository access, чтобы он сам работал с деревом, патчами и commits. GitHub обсуждается как возможный канал, а структура рабочего reverse repository уже проектируется. Это **концептуальный переход**, а не ещё фактический GitHub authority.

Параллельно второй агент уже получает специализированный ISP reverse checkpoint с новыми external references, paths, confirmed/gaps и конкретным следующим priority.

### Технический вклад
`CHAT-007` заполняет середину раннего порта: clean OpenIPC RAM baseline → загрузка stock Fullhan media modules в OpenIPC → musl-compatible MIPI/GC1054 plugins → sensor/MIPI stock-like state → VMM/VPU/PAE/media ABI → локализация blocker в ISP startup/lifecycle. External FH8852 reverse затем превращает широкий поиск в конкретную гипотезу missing sensor-registration/init/kick chain.

### Partial overlap
Начальный большой префикс `CHAT-007` повторяет уже обработанный `CHAT-006`. Он не использован для повторного увеличения evidence/count; анализ начат с нового продолжения после прежнего clean-room fh_mpi endpoint.

## Что CHAT-008 добавил к картине

### Source relation
`CHAT-008` имеет общий префикс с `CHAT-007` примерно до строки 24104, после чего это отдельная историческая ветка. Общий префикс повторно не учитывался.

### Новых error-ID не потребовалось
Источник усилил существующие классы:
- E-001/E-002: пользователь сам сформулировал adaptive granularity — ветвящиеся шаги по результату, знакомый routine цельным блоком;
- E-005: `scp -O` снова был забыт;
- E-011: оценка «2–3 узких неизвестных» оказалась чрезмерно уверенной;
- E-019: путаница WSL/Windows при передаче файлов;
- E-023: пользователь многократно спрашивал остаток reverse;
- E-024: условные действия снова склеивались через `&&`.

### Новые/уточнённые improvements
- I-035: adaptive granularity по engineering decision boundaries;
- I-036: runtime `LD_PRELOAD`/ioctl interception как более дешёвая альтернатива полному reverse helper'а, если нужен только boundary ABI;
- I-037: SoC-first reusable backend вместо одноразового board port.

I-019, I-032 и I-034 переведены в `CONSOLIDATED`.

### Техническая роль
Уникальная ветка независимо локализовала важный ISP gap:
- восстановлен lifecycle через Apollo + Fullhan reference;
- в stock hardware init найден interrupt enable/mask `ISP+0x08`;
- включение маски подняло ISP IRQ;
- изоляция interrupt bits показала sensor cadence около 25 fps;
- PAE всё ещё не получал frame path, поэтому blocker был локализован в `ISP → VPU/PAE`, а не sensor/MIPI.

### Agentic role
Локальный checkpoint уже используется как persistent engineering substrate: после потери runtime `/tmp` рабочие helpers восстанавливаются из checkpoint, а не по памяти. Parallel ISP agent получает узкий technical checkpoint с `confirmed/gaps/next priority` и запретом повторно реверсить закрытые части; основной агент после возврата сверяет и интегрирует findings.

Это всё ещё pre-Git-authority стадия: пользователь вручную маршрутизирует файлы/handoff между агентами.

## Что CHAT-009 добавил к картине

### Новый error-class
- E-027: сложные interactive paste-блоки на UART сами становятся источником ошибок; control flow/regex/quotes/critical MMIO нужно переносить в script/helper.

### Усиленные существующие ошибки
- E-002: пользователь прямо запретил микрошаги для routine и оставил пошаговость только для экстренных/ветвящихся мест;
- E-011: SSH root cause объявлялся закрытым преждевременно до локализации CRNG;
- E-012/E-015: `rmmod isp` и multi-owner показали реальную аппаратную цену dirty lifetime — dangling IRQ и kernel Oops;
- E-018: сгенерированный source с лишним `\&ch` не прошёл compile; preflight должен быть до передачи пользователю;
- E-019/E-024: WSL/UART/PlatformIO и сложные shell blocks продолжали смешиваться.

### Новые/уточнённые improvements
- I-006, I-013, I-014 и I-027 переведены в `CONSOLIDATED`;
- I-038: handoff должен быть reproducibility manifest со всеми canonical artifacts, paths и build/transfer/reverse/run recipes;
- persistent owner/daemon получает прямое safety-обоснование из broken ISP unload lifecycle;
- SSH dev-loop диагностируется по boundaries: network → TCP/22 → banner/entropy → auth → PTY.

### Технический вклад
- cold boot требует реального GPIO5 reset pulse, а не только финального уровня;
- единый ISP/VPU/PAE helper получил Annex-B H.264 с SPS/PPS/IDR и сохранил capture;
- одинаковый descriptor/серое содержимое оставили moving dequeue отдельным gate;
- unload vendor ISP module доказан небезопасным; single lifetime owner становится обязательным;
- persistent Dropbear key, persistent entropy seed и devpts/PTMX fixes переводят SSH из многоминутного bottleneck в управляемую часть dev-loop.

### Agentic role
Контекст одного диалога уже явно недостаточен. Создаётся большой handoff, а пользователь вводит более строгий acceptance: следующий агент должен воспроизводить работу по файлам/директориям/evidence/командам без скрытого знания предыдущего агента.

При этом authority всё ещё локальная: пользователь остаётся маршрутизатором WSL files, UART state и handoff между чатами. Перехода к Drive/GitHub/MCP в этом источнике нет.

## 9-file live-state refresh после CHAT-009

Выполнено после девятого уникального источника.

Проверено:
- актуальные `STATE.md`, `TASKS.md`, `AGENTS.md` на `main`;
- branch topology reverse/Firmware/Builder/Linux/Divinus/U-Boot;
- современная authority-модель GitHub / Google Drive / Ghidra MCP.

Authority endpoint не изменился:
- GitHub — current source/docs/contracts/state/manifests;
- Google Drive — heavy/unique primary evidence;
- Ghidra MCP через Koba MCP Bridge — canonical mutable reverse workspace.

Современный engineering state заметно продвинулся относительно ранних refresh:
- U-Boot `fh8626v100-mainline` теперь описан как OpenIPC-native source/build accepted, но ещё не hardware-pass;
- stock-compatible U-Boot сохранён отдельной recovery/evidence веткой;
- Linux `work/fh8626v100` содержит curated source series, а `fullhan-fh8626v100` остаётся hardware-tested/read-only integration reference;
- Firmware сохраняет shared core + Divinus/Majestic runtime lines;
- Divinus work branch и reverse work branch продолжают двигаться параллельно.

Эти refs используются только как **современные locators**. Они не подменяют исторический факт, что `CHAT-009` ещё работал локальными WSL/checkpoint/handoff средствами и не имел GitHub/Drive/Ghidra authority workflow.

Особенно хорошо видно эволюционное расстояние: в `CHAT-009` context limit требовал гигантского ручного handoff; в текущем проекте handoff опирается на repository authority, manifests, branch roles и external reverse workspace.

## Следующее действие

Получить следующий уникальный исторический источник. Следующая плановая расширенная live-state сверка — после двенадцатого уникального файла либо раньше при крупном противоречии/инфраструктурном переходе.
