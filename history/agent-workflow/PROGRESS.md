# Прогресс аудита исторических чатов

Статус: `CHAT_026_POST_REFRESH_PENDING`

## Текущее состояние

- Рабочая ветка: `audit/agent-workflow-history`
- Обработано исторических файлов: **26**
- Последний источник: `CHAT-026`
- Период последнего источника: **2026-08-29**
- Следующее действие: выполнить final post-file refresh этой пачки
- Raw chat exports в Git **не сохраняются**
- Базовый промпт сохранён неизменным в `BASELINE_PROMPT.md`

## Пять направлений

| Направление | Файл | Состояние после CHAT-026 |
|---|---|---|
| Учёт файлов и непрерывность | `PROGRESS.md` | 26/?? sources обработано |
| Ошибки/нарушения агентов | `ERRORS.md` | 40 tracked classes/directions |
| Улучшения и best practices | `IMPROVEMENTS.md` | 96 tracked improvements/directions |
| История реверса/портирования | `CHRONOLOGY.md` + `CHRONOLOGY_DETAILS.md` | добавлена D24: partial Agent4 watchdog/peripheral corpus |
| Эволюция агентной разработки | `AGENTIC_DEVELOPMENT.md` | добавлена A4.16: MASTER_CORE + REVERSE_HEAVY two-tier storage |

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
| 10 | `CHAT-010` | 2026-08-26 | `DONE` | Успешное продолжение из reproducible handoff без повторного bring-up; PTY cold-boot closure; exact `PAE 5011` release + `4D05/4D06` query semantics; full-disassembly self-service; livecapture source ordering fix (hardware retest pending); отдельная SSH regression с отклонённым ControlMaster workaround |
| 11 | `CHAT-011` | 2026-08-24 — 2026-08-26 | `IN_PROGRESS` | Divergent branch: общий префикс с CHAT-001 примерно до L32775, далее отдельная ветка ISP/VPU/H.264/dev-loop; анализируется только уникальный хвост |
| 12 | `CHAT-012` | 2026-08-26 | `DONE` | Почти полный duplicate CHAT-004: exact common prefix 8392/8556 строк (~98.1% файла), поэтому повторный dequeue/grey/RAW evidence не пересчитывался. Уникальный хвост: stage summary после RAW-DMA breakthrough и решение начать productionization параллельно, сохранив single-owner `fh8626_daemon` boundary перед Majestic |
| 13 | `CHAT-013` | 2026-08-27 | `DONE` | Day AWB mode1 сильно восстановлен и переведён в diag/shadow; hardware identity очищена от JXF37 false lead; stock runtime подтверждает GC1054 1280×720 + downstream 1080 upscale и dual-GC1054 target1/target2 lens lifecycle; сформулирован milestone-driven autonomous reverse contract |
| 14 | `CHAT-014` | 2026-08-27 | `DONE` | Parallel Agent 3: exact AWB→CCM reverse, coherent v4.2.1 hardware validation and rollback; session-quality retrospective becomes separate artifact with operational-state/state-machine/automation proposals |
| 15 | `CHAT-015` | 2026-08-27 — 2026-08-28 | `IN_PROGRESS` | Центральная orchestration-ветка: multi-agent decomposition/integration, project-file isolation, long Task N lanes, Cross-Fullhan semantic oracle, APC/total-gain diagnosis и same-SoC external research |
| 16 | `CHAT-016` | 2026-08-27 — 2026-08-28 | `DONE` | Persistent Agent 1 lane: Task1 C949C/C9898 → large Task2 full AE loop; premature completion corrected by full audit; stock GC1054 library + runtime captures close sensor-register contract and day/night replay; Task3 statically closes much of dual-lens/light/audio orchestration; quality-pack v3 adds focused-work/liveness/tool-placement rules |
| 17 | `CHAT-017` | 2026-08-27 — 2026-08-28 | `DONE` | Persistent Agent 2: image-detail Task2 corrected after premature completion; live Apollo RW/GOT closes APC/NR3D/LTM/D1DB0 tables; final current-day detail handoff + selftest; quality retrospectives consolidated into proposed full reverse-corpus architecture |
| 18 | `CHAT-018` | 2026-08-27 — 2026-08-28 | `DONE` | Expanded snapshot overlapping CHAT-016; unique tail validates wide→tele→wide on stock, shared AE context/history and implementation-ready dual-lens contract |nded snapshot of Agent 1 branch: prefix overlaps CHAT-016; unique tail adds controlled wide→tele→wide runtime validation, implementation-ready dual-lens closure and Task3 orchestrator supplement |
| 19 | `CHAT-019` | 2026-08-28 | `DONE` | Partial export: stock RTSP auth behavior and known HTTP JPEG snapshot endpoint; no new major workflow class |
| 20 | `CHAT-020` | 2026-08-27 — 2026-08-28 | `IN_PROGRESS` | Central orchestrator: quality rules become automated workspace architecture, clean master-tree, task router/sessions/runbooks/health checks, integration of three specialist lanes and Cross-Fullhan references |
| 21 | `CHAT-021` | 2026-08-28 | `DONE` | Systematic stock evidence campaign: four canonical imaging states, bidirectional lens/day-night transitions, white-light/audio/JPEG/PTZ, targeted Apollo runtime evidence, static reverse preparation, evidence catalog/state matrix and delta package; OOM/resource and metadata-correction lessons |
| 22 | `CHAT-022` | 2026-08-28 | `DONE` | Production integration catches code up to reverse; hardware validates WIDE H264, gain/APC/NR3D/LTM/manual AE/AWB→CCM and dual-sensor after GPIO5 bootstrap; lifecycle/control-plane and RAW/Bayer become focused remaining blockers |
| 23 | `CHAT-023` | 2026-08-28 | `DONE` | Orchestrator formalizes reverse vs implementation vs evidence lanes, retracts premature v20/v21 as official distribution points, introduces frozen-master integration barrier and layered progress ontology |
| 24 | `CHAT-024` | 2026-08-28 | `DONE` | OEM/device research triangulates AJL33PQ0866/YGT software identity, CF26/SM PCB family and physical 3.6/12 dual-lens configuration; builds ranked donor/reference corpus without promoting any retail SKU to target truth |
| 25 | `CHAT-025` | 2026-08-28 — 2026-08-29 | `DONE` | Cross-platform gap hunter discovers exact-FH8626 public adapter/SDK targets, exposes stats-epoch/lifecycle/RAW architecture leads, audits current source risks and packages targeted inputs instead of repeating Apollo reverse |
| 26 | `CHAT-026` | 2026-08-29 | `DONE` | Partial export: consolidated Agent4 watchdog/peripheral status; project split into MASTER_CORE + REVERSE_HEAVY with logical heavy IDs; TFTP root reorganized into active/history staging without destructive cleanup |

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

## Что CHAT-010 добавил к картине

### Новый error-class
- E-028: если пользователь сообщает, что после наших изменений ранее быстрый/рабочий путь деградировал, нельзя нормализовать это как «особенность железа» и скрывать workaround-ом; сначала нужен known-good diff и regression isolation.

### Усиленные существующие классы
- E-001: оператор явно выполняет команды строго по агенту; лишнее техническое ветвление/уточнения нельзя перекладывать на него;
- E-002: новый unknown reverse идёт пошагово, но routine не должен дробиться без decision boundary;
- E-019: camera lane — прямые UART-команды, WSL отдельно;
- E-023: stage summary по текущему blocker полезнее расплывчатой оценки остатка.

### Новые/уточнённые improvements
- I-022 `Known-good baseline + regression isolation` → `CONSOLIDATED`;
- I-024 `Full searchable evidence` → `CONSOLIDATED`: полный `enc/media_process` disassembly заменяет ручные диапазоны;
- I-038 reproducible handoff → `CONSOLIDATED`: новый агент реально продолжает с dequeue boundary;
- I-039: предыдущий агент — fallback только для отсутствующего уникального gap;
- I-040: fresh regression observation имеет приоритет над старой интерпретацией handoff.

### Технический вклад
- `PAE 0xC0045011` статически доказан как `pae_enc_stream_release(channel 0..7)`;
- `4D05` и `4D06` оба query одного encoded stream; различается wait policy;
- queue read-index двигается через `release → media_stream_release → enc_stream_get`;
- найден bug текущего livecapture: release выполнялся до чтения первого queued AU;
- source исправлен на `query → copy/CRC → release current`, но clean-boot hardware retest в этом source ещё не завершён;
- отдельная SSH-ветка заканчивается на локализации 5–7 sec regression по TCP/banner/KEX/auth boundaries, без подтверждённого root cause.

### Agentic role
`CHAT-010` — первый прямой acceptance-test handoff из предыдущего исчерпанного чата: доказанные sensor/ISP/H.264 этапы не повторяются, checkpoint paths и safety constraints используются как рабочее состояние.

Пользователь всё ещё остаётся ручным мостом к предыдущему агенту и локальным WSL artifacts; фактического перехода на GitHub/Drive/MCP authority в этом источнике ещё нет.

## Что CHAT-011 добавил к картине

### Source relation
Файл имеет общий префикс с ранним `CHAT-001` примерно до строки 32775, затем расходится в самостоятельную ветку. Общая часть не пересчитывалась как новое evidence.

### Новый error-class
- E-029: временный bring-up hack может потерять статус «временный» и незаметно стать частью финальной архитектуры; нужен явный cleanup/upstream debt ledger.

### Усиленные существующие классы
- E-002: прямой запрет на микрошаги кроме действительно emergency/decision boundaries;
- E-008/E-022: после power-cycle вместо нового blind reverse надо восстанавливать known-good state; в этой ветке missing delta оказался GPIO5 reset pulse;
- E-012: intrusive stream-tap снова повесил target;
- E-019: Windows/WSL execution surface снова пришлось уточнять;
- E-023: повторные вопросы пользователя о том, сколько осталось до запуска;
- E-028: SCP regression после SSH/network правок подтверждена независимо.

### Новые/уточнённые improvements
- I-033 переведён в `CONSOLIDATED`: development reverse docs и final/upstream architecture должны быть разными слоями;
- I-041: temporary workaround ledger + отдельный final reconciliation gate;
- living master handoff должен сохранять старое evidence как superseded history, но верхний authoritative state актуализировать;
- hardware-proven generated image не считается canonical source, пока proven fixes не перенесены в source tree.

### Техническая роль
Дивергентная ветка независимо проходит уже известные milestones D5-D8:
- ISP frame path и geometry;
- integrated one-process/one-ISP-fd bring-up;
- Annex-B H.264 с SPS/PPS/IDR и активными ISP/PAE IRQ;
- repeated descriptor остаётся отдельным dequeue gate;
- persistent owner/daemon как выход из обязательных power-cycle;
- SSH/RNG/network/PTTY dev-loop optimization.

Новых опорных технических глав не создавалось; источник использован как дополнительное доказательство существующих milestones.

### Agentic role
К концу ветки master handoff уже обслуживается как living authoritative artifact: параллельный агент работает по старой версии, а вместо нового handoff получает reconciliation prompt на обновление текущего master. Пользователь отдельно требует не потерять список dev-only изменений перед финальным портом.

Фактического перехода на GitHub/Drive/MCP authority в этом источнике ещё нет: координация остаётся через локальный workspace, файлы и ручной handoff между агентами.

## Что CHAT-012 добавил к картине

### Source relation
`CHAT-012` почти полностью совпадает с уже обработанным `CHAT-004`: точный общий префикс составляет **8392 из 8556 строк**. Поэтому dequeue, grey-frame localization, 1080 upscale, Apollo/SREG reverse, stock-runtime bundle и RAW-DMA breakthrough повторно не учитывались как новое evidence.

Уникальным является только финальный хвост после расхождения.

### Новых error-ID нет
Все существенные ошибки/workflow corrections внутри общего префикса уже были учтены через `CHAT-004`. Повторное повышение статусов по одному и тому же историческому эпизоду не выполнялось.

### Новый improvement
- I-042: productionization можно начинать параллельно с остаточным low-level reverse, но только за доказанной subsystem boundary.

### Техническая/архитектурная роль уникального хвоста
На момент расхождения:
- dequeue уже закрыт;
- реальный H.264 и stock-like 720→1080 VPU upscale доказаны;
- RAW DMA уже ожил после восстановления missing ISP input state;
- оставшийся blocker локализован в корректности RAW/Bayer и последующем ISP tuning.

Пользователь предлагает переходить к firmware/Majestic. В ответ впервые явно формулируется production architecture:
- один долгоживущий `fh8626_daemon` владеет stateful Fullhan fd и ISP runtime;
- Majestic не открывает `/dev/isp` вторым owner;
- Majestic получает готовый stream через downstream interface;
- stable rootfs/module/startup pieces можно productionize параллельно с последними RAW/ISP A/B tests.

Это design direction, а не Majestic hardware-pass.

### Agentic role
Это ранний переход от «probe как конечная форма эксперимента» к «stable subsystem contract → production daemon → streamer». Позднее `CHAT-003` реализует persistent experimental substrate гораздо глубже; поэтому A4.2 теперь считается `CONSOLIDATED`.

## 12-file live-state refresh после CHAT-012

Выполнено после двенадцатого уникального источника.

Проверено read-only:
- `main:STATE.md@a0022eb2...`;
- `main:TASKS.md@4f57717e...`;
- `main:AGENTS.md@58075253...`;
- live branch lists reverse/Firmware/Builder/Linux/Divinus/U-Boot.

Современная authority-модель сохраняется:
- GitHub — current source/docs/contracts/state/manifests;
- Google Drive — heavy/unique primary evidence;
- Ghidra MCP через Koba MCP Bridge — canonical mutable reverse workspace.

Технический endpoint остаётся существенно дальше исторического `CHAT-012`:
- U-Boot native source/build line `fh8626v100-mainline@7ac0aa7e...` готова к hardware acceptance, stock-compatible recovery сохранён отдельно;
- Linux curated work line `work/fh8626v100@357c2d13...` и PR-facing `fullhan-fh8626v100@0dfafa64...` присутствуют;
- Firmware сохраняет streamer-neutral core и отдельные Divinus/Majestic runtime directions;
- current status по-прежнему `ACTIVE / UBOOT_OPENIPC_NATIVE_SOURCE_READY / KERNEL_GATE / FIRMWARE_CLEAN_CANDIDATE / DIVINUS_REFERENCE / MAJESTIC_TARGET`.

Одновременно выявлен **coordination drift**, который нельзя молча принимать за current truth:
- `STATE.md` указывает Firmware core `eabd1ccd...`, live branch уже `80169887...`;
- `STATE.md` указывает Firmware Divinus `0b12c87c...`, live branch `255b8c8d...`;
- `STATE.md` указывает Firmware Majestic `7ed2a17a...`, live branch `a1acd664...`;
- `STATE.md` указывает Divinus work `1e624bd5...`, live branch уже `bd878506...`;
- Builder device branch в `STATE.md` указан как `a51eec5b...`, live `work/fh8626v100-anjia` уже `dc7ddabf...`;
- main `AGENTS.md` и `STATE.md` называют Builder Majestic staging `work/fh8626v100-anjia-majestic`, но в live branch list такой ветки сейчас нет.

Это не исправлялось в рамках исторического аудита: current coordination docs/branches read-only. Для последующей инженерной работы live refs должны быть reconciled с authority-документами отдельной рабочей сессией.

Исторический вывод `CHAT-012` от этого не меняется: в августе это была только ранняя design boundary `single owner daemon → downstream Majestic`, а не современная реализация.


## Что CHAT-013 добавил к картине

### Новый error-class
- E-030: наличие vendor driver/tuning/format доказывает поддержку firmware-варианта, но не физическое наличие sensor на конкретной board revision.

### Усиленные существующие классы
- E-003/E-008: снова всплыли неверный Windows path, забытый boot/mount state и повторный запрос уже имеющегося authoritative Apollo;
- E-011: JXF37 был слишком рано повышен из software capability до hardware hypothesis;
- E-016 теперь `CONSOLIDATED`: пользователь прямо потребовал автономной работы до milestone и меньше промежуточной экспозиции;
- E-022: повторный запрос Apollo при уже существующем authoritative artifact;
- E-023: local reverse progress без module-level map снова воспринимался как слишком медленный;
- E-026/E-027: host utilities и длинные UART paste-blocks снова показали ограничения target lane.

### Новые improvements
- I-043: milestone-driven autonomous reverse loop — внутренне проверять/отбрасывать гипотезы и возвращаться к оператору только при blocker или substantial milestone;
- I-044: firmware capability и hardware identity — разные evidence levels;
- I-045: closed-loop ISP algorithm сначала проходит offline self-test → live shadow/diag → gated hardware commit.

### Технический вклад
- day `CA4F4` mode1 существенно восстановлен на 9 AWB-stat records;
- правильный live stats mapping идёт через ISP ioctl offset, а не фиксированный VMM offset;
- `C949C/C9898` day-path и integer-sqrt/state contract заметно уточнены;
- JXF37/JXF37P остаются поддерживаемыми firmware variants, но не current AJL33PQ0866 hardware path;
- stock подтверждает GC1054 1280×720 как source и 1920×1080 как downstream upscale;
- stock zoom подтверждает две GC1054/lens targets: target1 wide и target2 tele, переключаемые GPIO4/GPIO14 внутри уже работающего media lifecycle;
- night mode отделён как следующий branch того же GC1054/ISP runtime, а не отдельный sensor mode.

### Agentic role
Впервые пользователь формулирует почти готовый autonomous-reverse contract: глубокий анализ и hypothesis checking выполняются внутри агента, user-facing сообщения — только по крупным завершённым этапам или реальным hardware blockers. Это ещё file/handoff-era, но уже явный предшественник будущего agentic workflow.

## Что CHAT-014 добавил к картине

### Новый error-class
- E-031: нельзя объявлять длинную задачу полностью закрытой без отдельного аудита исходных acceptance criteria.

### Новые improvements
- I-046: completion audit по original checklist;
- I-047: machine-readable operational session state;
- I-048: hardware experiment state machine;
- I-049: quality retrospective агента как отдельный project artifact.

### Технический вклад
Parallel Agent 3 восстановил и hardware-валидировал coherent AWB→CCM path: direct AWB MMIO step оказался неполным, а полный logical-state transition через A8/AA→C9F68→B0/B1/B2→CE764 дал именно предсказанный CCM diff. Rollback показал, что AWB и CCM требуют раздельного restore-set.

### Agentic role
В конце сессии ошибки взаимодействия собраны в отдельный quality-improvement pack, предназначенный уже не для reverse конкретной функции, а для изменения поведения будущих агентов/оркестратора.

## Что CHAT-015 добавил к картине

### Новые error-классы
- E-032: Project semantic context не гарантирует physical file access у parallel agent;
- E-033: слишком мелкая specialist task не окупает onboarding/handoff overhead.

### Новые improvements
- I-050: persistent specialist lanes с последовательными Task N;
- I-051: artifact-access preflight перед exact reverse;
- I-052: formal Cross-Fullhan semantic-oracle workflow;
- I-053: living external-research MD с provenance/method;
- I-054: приоритет внешних references от именованного adjacent-SoC к same-SoC.

### Усиленные существующие правила
- не экспортировать handoff/архив и не повышать внешнюю версию после каждой интеграции без запроса;
- не показывать SHA-256 пользователю без необходимости;
- центральный агент остаётся интегратором; specialists не делают hardware writes и не merge'ят друг друга;
- scope делится по state/dataflow chains, а не по случайным адресам.

### Технический вклад
Cross-Fullhan matching дал semantic map большой части CB970 и позволил target-specific диагностике найти два системных gaps текущего runtime: отсутствующий APC/detail controller `CDD6C` и отсутствующую per-frame публикацию live `total_gain` в `ctx+0x60`. Это напрямую связало оставшееся «мыло» с конкретными missing control paths.

Same-SoC external research также выделил native encoder timestamp как правильный cadence source и поддержал модель GC1054 720p source → downstream 1080p/frame-control path.

### Agentic role
Основной агент явно становится orchestrator/integrator нескольких persistent specialist threads. Но пользователь всё ещё вручную прикрепляет handoff bytes каждому агенту: shared repository/Drive/MCP authority ещё исторически не появилась.

## 15-file live-state refresh после CHAT-015

Выполнено после пятнадцатого уникального источника.

Read-only проверены `main:STATE.md`, `main:TASKS.md`, `main:AGENTS.md` и live branch lists reverse/Firmware/Builder/Linux/Divinus/U-Boot.

Современная authority-модель не изменилась:
- GitHub — current source/docs/contracts/state/manifests;
- Google Drive — heavy/unique evidence;
- Ghidra MCP через Koba MCP Bridge — canonical mutable reverse workspace.

Current technical endpoint по-прежнему существенно дальше исторического `CHAT-015`: native U-Boot source/build готов к hardware acceptance; Linux curated series существует отдельно от hardware-proven PR-facing line; Firmware/Divinus/Majestic directions продолжают развиваться.

При этом coordination drift остаётся:
- reverse `work/fh8626v100` live tip уже `4a8c59ca...`, тогда как main coordination docs не везде отражают этот locator;
- Firmware work branches живут на более новых tips, чем ряд SHA в `STATE.md`;
- Divinus `work/fh8626v100` live tip уже `809561bd...`;
- Builder live branch list содержит `work/fh8626v100-anjia@dc7ddabf...`, но не содержит `work/fh8626v100-anjia-majestic`;
- при этом текущий `main:AGENTS.md@3f880878...` снова описывает отдельную Majestic staging branch. Это противоречит live branch topology.

В рамках исторического аудита ничего из этого не исправлялось. Current coordination docs/branch roles должны быть reconciled отдельной рабочей сессией.

Исторический вывод `CHAT-015` не меняется: в конце августа multi-agent orchestration ещё работал через manual project chats/handoff attachments и не имел нынешней GitHub/Drive/Ghidra authority.

## Что CHAT-016 добавил к картине

### Новый error-class
- E-034: длительная локальная работа без краткого liveness-сигнала выглядит как зависание; milestone reporting нужно дополнить редким heartbeat.

### Consolidated errors
- E-020: broad recursive search;
- E-026: assumptions about target utilities;
- E-027: fragile UART paste;
- E-031: premature task completion;
- E-032: project context ≠ artifact bytes.

### Новые improvements
- I-055: focused reverse workspace;
- I-056: heavy static analysis off-target, camera only for runtime evidence;
- I-057: sparse liveness heartbeat without intermediate reasoning;
- I-058: distinguish stale/cache/deferred queue state from active controller state.

I-046/I-049/I-050/I-051 reinforced; persistent specialist lane and quality-engineering stages are now consolidated.

### Технический вклад
- full stock AE loop восстановлен до GC1054 I²C registers;
- day/night/wlight profile relationship восстановлена;
- day и night имеют independent numerical runtime replay;
- historical night limits 745/2 подтверждены static profile + live context;
- C9898 окончательно отделён как publication/status tail;
- D0630 отделён как отдельный adaptive ISP block;
- lens-switch static contract уточнён до D8308, GPIO4/14, VENC stop/start, mirror/flip, IR/white LED/IR-cut/audio controls; hardware capture этого Task3 в source ещё pending.

### Agentic role
CHAT-016 является прямым proof persistent specialist model: один Agent 1 последовательно выполняет Task 1, большой Task 2 и начинает Task 3, не проходя повторный onboarding. Completion audit и concrete blocker escalation позволяют довести область существенно глубже, чем первоначальное premature DONE.

Quality pack v3 показывает, что meta-workflow улучшения уже стали итеративной практикой, а не единичной ретроспективой.

## Что CHAT-017 добавил к картине

### Новый error-class
- E-035: cross-agent specialist scope contamination / role identity loss.

### Reinforced/consolidated
- E-022 promoted to CONSOLIDATED: do not re-request/re-disassemble artifacts already present;
- E-007/E-020/E-024/E-031/E-032 reinforced.

### Новые improvements
- I-059: static code + live runtime data as two complementary reverse layers;
- I-060: corpus-first pre-materialized reverse substrate with manifest/status map;
- I-061: specialist role-lock + artifact preflight;
- I-062: consolidate multiple agent retrospectives into one master process contract.

### Технический вклад
Agent 2 finally closes current-day image-detail reverse: APC/CDD6C, active NR3D/D0FEC, dynamic LTM D0630/D0B2C and D1DB0 runtime tables. Missing APC remains the strongest firmware-side softness cause, followed by active NR3D and LTM parity gaps.

### Agentic role
The handoff starts evolving from a transport archive into a mapped local workspace/corpus: prepared reverse representations, runtime evidence, environment map, reusable tools, expensive-work history and machine-readable status.

## Что CHAT-018 добавил к картине

- Источник является expanded snapshot: префикс перекрывает CHAT-016 и повторно не учтён.
- Unique tail переводит dual-lens switch из static-only в controlled runtime validation.
- I-048 experiment state-machine повышен до CONSOLIDATED: baseline/immediate/settled/restore capture реально помог отделить transient от устойчивого state.
- Новых уникальных error-классов нет.

## 18-file live-state refresh после CHAT-018

Read-only проверены current `main:STATE.md`, `main:TASKS.md`, `main:AGENTS.md` и live branches reverse/Firmware/Builder/Linux/Divinus/U-Boot.

Современная authority-модель прежняя: GitHub current source/state/contracts, Google Drive heavy evidence, Ghidra MCP mutable reverse workspace.

Live work tips снова ушли вперёд относительно части coordination locators:
- reverse `work/fh8626v100@d6e842dd...`;
- Firmware Divinus `bc09d12c...`, Majestic `aabf18a6...`, shared core `80169887...`;
- Builder ANJIA `7db8cc1f...`;
- Divinus `work/fh8626v100@50e3e300...`;
- Linux work остаётся `357c2d13...`;
- U-Boot lines остаются `7ac0aa7e...` / `49fe46e9...`.

Current `STATE.md@a0022eb...` не отражает все эти newer work tips. В audit branch ничего не исправлялось; это отдельный coordination-reconciliation debt.

Историческая граница сохранена: эти modern refs не используются как доказательство состояния августа 2026.

## Что CHAT-019 добавил к картине

Источник экспортирован неполностью; недоступный ранний префикс не реконструировался.

Технически подтверждено в доступной части:
- stock RTSP profile0/profile1 на 8554 требуют auth;
- изменение /home/dis_onvif_auth не применяется через один rtsp_stop/rtsp_start, потому что auth callback инициализируется раньше в lifecycle Apollo;
- stock JPEG snapshot доступен по HTTP endpoint :6688/snapshot.jpg.

Workflow: новых классов нет; E-016 получил ещё один небольшой пример лишнего обходного исследования вместо прямого известного действия.

## Что CHAT-020 добавил к картине

### Новый error-class
- E-036: нестабильные проценты готовности без fixed progress ontology.

### Reinforced
- E-013/E-014: unsolicited handoff/version export продолжался даже после явного запрета; E-014 теперь CONSOLIDATED.

### Новые improvements
- I-063: layered PROJECT_MAP + role/context entrypoints;
- I-064: automated structural invariants / handoff doctor;
- I-065: task router + context packs;
- I-066: sessions/ledger + recovery playbook;
- I-067: active knowledge separated from provenance/archive;
- I-068: module/status matrix as fixed progress ontology.

I-047 and I-060 promoted to CONSOLIDATED because operational state and corpus-first architecture are now implemented, not merely proposed.

### Technical/agentic transition
Three specialist lanes converge into one normalized current-day model. Broad reverse is no longer the default; the main path becomes feature-gated runtime integration and hardware validation. The local handoff becomes a governed workspace with executable quality checks.

## Что CHAT-021 добавил к картине

### Новые error-классы
- E-037: read-only diagnostic capture без resource budget может разрушить constrained target; full RAM dump в /tmp вызвал OOM и убил Apollo.
- E-038: capture manifest может описывать intended, а не observed state; metadata correction должна сохранять provenance.

### Новые improvements
- I-069: resource-budgeted capture;
- I-070: provenance-preserving metadata correction;
- I-071: synchronized targeted transition RAM evidence вместо noisy independent full-heap diff;
- I-072: capture state matrix + canonical/superseded evidence catalog;
- I-073: target capability map как executable command contract;
- I-074: evidence delta package поверх stable master.

### Технический вклад
Stock становится воспроизводимым ground-truth dataset: WIDE/TELE × DAY/NIGHT, lens/day-night/light/audio/PTZ transitions, JPEG path, boot/audio state, sensor registers, targeted Apollo RAM и prepared static module/library reverse material. Independent full heaps признаны слишком шумными для causality; targeted synchronized regions становятся preferred evidence.

### Agentic role
Проект переносит governance с reverse corpus на hardware evidence: agent проектирует experiment/state matrix, normalization и evidence catalog; пользователь в основном выполняет физические переключения и stock-команды.

## 21-file live-state refresh после CHAT-021

Read-only проверены current `main:STATE.md`, `main:TASKS.md`, `main:AGENTS.md` и live branches reverse/Firmware/Builder/Linux/Divinus/U-Boot.

Современная authority-модель без изменений:
- GitHub — current source/state/contracts/manifests;
- Google Drive — heavy/unique primary evidence;
- Ghidra MCP — canonical mutable reverse workspace.

Live branch locators на момент refresh:
- reverse work `d6e842dd...`;
- Firmware core `80169887...`, Divinus `bc09d12c...`, Majestic `aabf18a6...`;
- Builder ANJIA `7db8cc1f...`;
- Linux work `357c2d13...`, hardware-facing `0dfafa64...`;
- Divinus work `50e3e300...`;
- U-Boot work/recovery `7ac0aa7e...` / `49fe46e9...`.

Current `STATE.md@a0022eb...` по-прежнему не отражает все newer work tips. Кроме того, `main:AGENTS.md@3f880878...` описывает Builder Majestic staging branch `work/fh8626v100-anjia-majestic`, которой live branch list не содержит. Это current coordination-reconciliation debt; audit branch его не исправляет.

Исторически `CHAT-021` всё ещё file/handoff/TFTP/WSL-based. Современные GitHub/Drive/Ghidra authority роли не backdate'ятся.

## Что CHAT-022 добавил к картине

### Новый error-class
- E-039: snapshot-style patching может тихо потерять уже интегрированные features.

### Reinforced
- E-008/E-014/E-018/E-019/E-025/E-026; E-025 теперь CONSOLIDATED.

### Новые improvements
- I-075: source consolidation в один canonical tree;
- I-076: board cold-boot bootstrap как hardware contract;
- I-077: control plane должен оставаться responsive при video loss;
- I-078: hardware mechanism PASS отдельно от visual/algorithmic parity;
- I-079: rejected-hypothesis ledger.

### Technical transition
Reverse contracts впервые массово догоняются кодом и hardware validation. GPIO5 reset sequence превращает dual sensor в реальный product contract; green cast переводится из CCM tuning в отдельный RAW/Bayer early-color blocker; experimental shutdown показывает возможность healthy restart, но не exact production lifecycle.

## Что CHAT-023 добавил к картине

- E-013/E-014 снова подтверждены: orchestrator начал готовить/мутировать состояние раньше точной задачи и преждевременно повысил master v20/v21.
- E-036 уточнён: progress должен различать contract-level 100% и subsystem/product readiness.
- I-080: frozen master + integration barrier.
- I-081: evidence-preparation pass перед глубоким reverse.
- I-082: 100% конкретного contract отдельно от 100% подсистемы.
- A4.14: frozen-base fan-out / DELTA fan-in orchestration.
- Нового самостоятельного hardware milestone нет; CHAT-023 главным образом нормализует распределение уже идущей работы.

## Что CHAT-024 добавил к картине

- E-011/E-030 reinforced: visual or same-SoC similarity is not enough to declare exact model/hardware compatibility.
- E-016 reinforced: final orchestrator handoff had to be compressed after an overlong first version.
- I-083: target identity triangulation from software ID + PCB + physical fingerprint.
- I-084: hardware similarity and reverse usefulness are separate donor dimensions.
- I-085: search fingerprint built from internal identifiers rather than retail names.
- D22: historical establishment of AJL33PQ0866 + CF26/SM OEM-family search and donor corpus.
- Exact retail SKU remains unresolved by design; own target dump/PCB/software remains authority.

## 24-file live-state refresh после CHAT-024

Read-only проверены current `main:STATE.md`, `main:TASKS.md`, `main:AGENTS.md` и live branches reverse/Firmware/Builder/Linux/Divinus/U-Boot.

Современная authority-модель прежняя: GitHub current source/state/contracts, Google Drive heavy evidence, Ghidra MCP canonical mutable reverse workspace.

Live work tips на момент refresh остались:
- reverse work `d6e842dd...`;
- Firmware core `80169887...`, Divinus `bc09d12c...`, Majestic `aabf18a6...`;
- Builder ANJIA `7db8cc1f...`;
- Linux work `357c2d13...`, hardware-facing `0dfafa64...`;
- Divinus work `50e3e300...`;
- U-Boot `7ac0aa7e...` / `49fe46e9...`.

`STATE.md@a0022eb...` и `TASKS.md@4f57717e...` не изменились с предыдущей сверки; modern coordination drift остаётся отдельной задачей. Frozen audit baseline подтверждён SHA `1358cffd...`.

Исторически `CHAT-024` — external web/OEM research через browser/chat; modern GitHub/Drive/Ghidra roles не backdate'ятся.

## Что CHAT-025 добавил к картине

- I-052/I-053/I-054 promoted to CONSOLIDATED: Cross-Fullhan/external-research methodology is now a repeated specialist workflow.
- I-086: negative architecture audit.
- I-087: discovered vs acquired vs prepared external artifact status.
- I-088: compact targeted cross-platform lead format for Agent 1.
- I-089: independent-source corroboration weighted by SoC distance.
- I-090: typed transactions + generation invalidation as reusable stateful-ISP architecture.
- E-009/E-014 reinforced by unnecessary SHA/user-facing hash packaging.
- D23: exact-FH8626 adapter/SDK leads plus cadence/statistics/lifecycle/RAW omissions.
- A4.15: dedicated cross-platform research intelligence lane.

## Что CHAT-026 добавил к картине

Источник экспортирован частично; ранний Agent-4 reverse process отсутствует и не реконструировался.

### Новый error-class
- E-040: transport/TFTP root не должен превращаться в долговременный artifact warehouse.

### Новые improvements
- I-091: MASTER_CORE + REVERSE_HEAVY two-tier storage;
- I-092: stable logical IDs для heavy artifacts;
- I-093: tftp_active / tftp_history lifecycle;
- I-094: cleanup через inventory/move, delete только после canonicalization;
- I-095: master содержит summaries/excerpts, heavy — bulk evidence;
- I-096: storage role независимо от transport archive.

### Technical/agentic transition
Доступный хвост сохраняет consolidated watchdog/kernel corpus и peripheral backlog, но главный исторический переход — отделение everyday working knowledge от тяжёлых immutable reverse/evidence artifacts. A4.16 — прямой предшественник современной split authority, но фактический Google Drive/GitHub/MCP transition этим чатом ещё НЕ подтверждён.

## Следующее действие

Выполнить final post-file refresh. Затем ожидать следующий уникальный исторический источник; следующая expanded live-state сверка — после `CHAT-027` либо раньше при фактическом переходе к Drive/Git/MCP.
