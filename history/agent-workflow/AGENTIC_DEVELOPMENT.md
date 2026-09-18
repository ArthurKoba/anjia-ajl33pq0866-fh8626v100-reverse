# Эволюция агентной разработки

Назначение: восстановить, как проект перешёл от ручного взаимодействия в одном чате к многоагентной инженерной системе, пригодной для реверса, портирования и сопровождения камер.

## Главный вопрос

Как организовать работу так, чтобы человек не выступал диспетчером команд и анализатором логов, а выполнял только те операции, которые действительно требуют физического доступа или доверенного операторского действия: прошивку, подключение камеры, аппаратный recovery, проверку изображения/звука/движения и другие target-only тесты.

## Предварительная модель этапов

### A0 — Ручной браузерный чат
Статус после `CHAT-005` + `CHAT-001`: `CONSOLIDATED`.

Самый ранний `CHAT-005` показывает исходную модель почти без внешней инфраструктуры:
- пользователь приносит сырой bootlog/full-flash evidence и физически работает с UART, programmer и U-Boot;
- агент интерпретирует raw output, формирует следующую команду и проектирует безопасный bring-up;
- пользователь вручную переносит команды/выводы между терминалами;
- continuity практически полностью живёт в браузерном чате.

`CHAT-001` затем показывает ту же схему уже при интенсивном media bring-up.

Сильная сторона: быстро можно начать исследование неизвестного железа без предварительной инфраструктуры.
Слабая сторона: пользователь остаётся транспортом для файлов/команд/контекста, а цена длинного чата и повторных проверок быстро растёт.

### A0.5 — Artifact-assisted analysis
Статус после `CHAT-006`: `OBSERVED`.

После самого раннего manual-chat режима появляется первый заметный перенос работы с оператора на агента: полный SPI dump становится рабочим источником, из которого агент сам извлекает filesystem, modules и ABI evidence.

Сначала агент ещё по привычке просил вручную найти и передать media modules с камеры. После замечания пользователя дальнейший inventory и reverse были перенесены на уже имеющийся firmware artifact.

Это важный промежуточный этап:
- persistent shared workspace ещё нет;
- но уже работает принцип «получить богатый artifact один раз и дальше анализировать его автономно»;
- target нужен только для новых runtime/hardware фактов.

Позднее этот принцип развивается в full searchable bundles, curated workspace и внешние authority-системы.

### A1 — Локальный workspace + checkpoint/handoff
Статус после `CHAT-001` + `CHAT-008` + `CHAT-009` + `CHAT-010` + `CHAT-011`: `CONSOLIDATED`.

По мере роста количества helper binaries, dumps и reverse-наработок появился устойчивый локальный workspace и checkpoint-наборы. Перед reboot начали сохранять состояние, а к концу чата — собирать воспроизводимый handoff.

Это первый важный перенос ответственности от памяти пользователя/чата к внешнему состоянию проекта.

Но workspace ещё локален и не является общей authority: новый агент зависит от переданного Markdown и наличия локального дерева.

`CHAT-008` показывает практическую ценность локального checkpoint: после очистки runtime `/tmp` рабочие sensor/ISP/H.264 helper'ы находят в `checkpoints/openipc-20260825` и переиспользуют без повторной реконструкции. Это уже persistent project state, хотя ещё только на машине пользователя.

`CHAT-009` доводит критерий handoff до воспроизводимости: при лимите контекста мало пересказать findings — нужно перечислить canonical files/directories, reverse/disassembly/memory evidence и exact build/transfer/run recipes, чтобы следующий агент продолжил без скрытого знания.

`CHAT-010` — первый прямой acceptance-тест этого handoff: новый агент сразу продолжает с dequeue boundary, не возвращается к sensor/ISP/H.264 bring-up и использует сохранённые checkpoint paths и safety constraints как рабочее состояние.

`CHAT-011` добавляет reconciliation-поведение: когда параллельный агент уже работает по старой master-версии, ему не создают ещё один независимый handoff, а дают точный prompt на обновление существующего authoritative файла. Fresh proven state поднимается наверх, старые observations сохраняются как historical/superseded evidence, а временные dev hacks получают отдельный cleanup debt.

### A1.5 — Идея repository-backed workflow
Статус после `CHAT-007`: `OBSERVED`, но ещё не внедрено как authority.

Пользователь прямо формулирует следующую проблему ручного режима: если дать агенту доступ к репозиторию, он сможет сам разбирать дерево, готовить патчи/коммиты и уменьшить постоянный copy/paste.

В том же чате проектируется будущая структура рабочего reverse-репозитория:
- текущие hardware/boot/media/ioctl факты;
- отдельный known-good bring-up;
- experiments/history для ошибочных гипотез;
- provenance для бинарников/evidence;
- upstream OpenIPC changes отдельно от полного reverse-журнала.

Это ещё не переход на GitHub как source of truth: фактическая работа по-прежнему идёт через WSL, файлы и чат. Но архитектура будущего repo-mediated workflow уже явно сформулирована.

### A2 — Постоянное внешнее evidence-хранилище
Статус: `BOOTSTRAP`.

Google Drive в `CHAT-001` ещё не является наблюдаемым этапом.

### A3 — Git как инженерный authority
Статус: `BOOTSTRAP`.

В `CHAT-001` используются upstream Git repositories для сборки/сравнения, но собственное долговременное состояние порта ещё не организовано через Git как authority.

### A4 — Прямые operational-каналы
Статус после `CHAT-005` + `CHAT-009` + `CHAT-001`: `CONSOLIDATED`.

Первые прямые operational-каналы появляются уже в `CHAT-005`: U-Boot/TFTP заменяет перенос test payload через браузер. В `CHAT-001` к нему добавляются SSH/SCP для OpenIPC runtime и затем оптимизация SSH startup/connection reuse.

Это уменьшило зависимость от браузерной передачи бинарных артефактов, но потребовало собственного transport contract: один и тот же способ передачи нельзя механически применять к разным runtime states.

`CHAT-009` показывает, что operational channel нужно оптимизировать как инженерную подсистему: network readiness, TCP port, SSH banner, host key, entropy и PTY диагностируются как отдельные boundaries, а не как одно расплывчатое «SSH не работает».

### A4.1 — Self-service reverse evidence
Статус после `CHAT-004` + `CHAT-010`: `CONSOLIDATED`.

В начале этапа reverse всё ещё требовал серии ручных команд: пользователь запускал `objdump/grep`, возвращал очередной диапазон, после чего агент просил следующий.

В ходе `CHAT-004` это было признано дорогим интерфейсом. Для часто используемых источников стали передаваться полные searchable artifacts:
- full module disassembly/symbol/section bundle;
- полный Apollo disassembly + strings + unpacked image;
- stock runtime evidence bundle.

Агент получил возможность самостоятельно искать xrefs/call chains и возвращаться к пользователю уже только за hardware evidence или target-only экспериментом.

Это ранний шаг от **user-mediated reverse lookup** к будущему shared reverse workspace: источник истины ещё передаётся файлами, но пользователь перестаёт выполнять каждый поисковый запрос агента.

`CHAT-010` повторяет модель уже для `enc.ko` и `media_process.ko`: пользователь один раз генерирует full disassembly/symbols/sections, после чего агент самостоятельно закрывает ioctl mapping и queue semantics без новых ручных диапазонов.

### A4.2 — Persistent experiment substrate
Статус после `CHAT-012` → `CHAT-003`: `CONSOLIDATED`.

До этого пользователь всё ещё тратил много внимания на повторный boot, повторный media bring-up и смену тестовых бинарников.

В `CHAT-003` появляется другой контур:
- один долгоживущий media owner держит stateful vendor devices;
- runtime-алгоритм меняется через reloadable `.so`;
- проверенная U-Boot команда автоматизирует возврат в development OpenIPC;
- stock boot остаётся отдельной именованной recovery-командой;
- hardware test всё больше сводится к «доставить один artifact → reload/run/capture → вернуть результат».

Это не только техническая оптимизация. Она уменьшает роль человека как диспетчера инфраструктуры и делает цикл пригоднее для автономной агентной разработки.

`CHAT-012` показывает предшествующее архитектурное решение: когда dequeue и stream boundary уже доказаны, productionization можно начинать до идеального ISP, если unresolved RAW path остаётся внутри одного диагностического/persistent owner. В этой схеме Majestic — downstream consumer stream interface, а не новый владелец stateful vendor fd.

Главный недостаток этого этапа: test protocol ещё часто был ручным и неоднозначным, а compile/preflight не всегда выполнялся до передачи artifact.

### A4.5 — Handoff-based parallel agents
Статус после `CHAT-007` + `CHAT-002`: `CONSOLIDATED`.

`CHAT-007` показывает ранний parallel checkpoint: пользователь вручную передаёт второму агенту новый external-reference пакет, paths, confirmed/gaps и текущий priority. Позже в конце другого чата уже виден более полный многоагентный режим:
- другой агент ведёт параллельный участок;
- пользователь приносит его handoff;
- текущий агент сравнивает два состояния;
- уникальные находки объединяются;
- выбирается один master для продолжения.

В `CHAT-002` этот режим становится заметно более формальным: пользователь назначает основному и parallel reverse-агенту непересекающиеся зоны, требует finished reverse units и затем просит основной агент объединять их в один canonical runtime.

Это всё ещё не автономная агентная система: пользователь по-прежнему является маршрутизатором контекста и архивов между чатами.

В `CHAT-010` прошлый агент становится fallback, а не основной continuity channel: сначала используются handoff/checkpoint/artifacts, и только отсутствующий уникальный факт предполагается уточнять через пользователя у предыдущего агента.

Главный урок: handoff должен быть не «всё подряд», а **доказанные факты + gaps + reproduction path + next boundary**.

### A4.6 — Role-specialized agent pipeline
Статус после `CHAT-008` + `CHAT-002`: `CONSOLIDATED`.

Появляется уже не просто «второй агент», а разделение ролей:

- **основной агент** держит canonical source/runtime, интегрирует writers и отвечает за test archives;
- **parallel reverse-agent** глубоко разбирает отдельные heavy functions, tables и contracts;
- scopes явно исключают дублирование;
- hardware validation вынесена из parallel reverse;
- результат передаётся как finished unit с reproduction/self-test, а не как набор догадок;
- после передачи unit должен быть слит в основную реализацию, а временная параллельная реализация не должна жить отдельно.

Это существенный шаг к агентной разработке: человек уже меньше занимается техническим анализом, но пока вручную координирует границы работы и переносит пакеты между чатами.

В `CHAT-008` тот же паттерн виден раньше и проще: parallel ISP agent получает checkpoint с `confirmed/gaps/next priority` и прямым запретом заново реверсить подтверждённое. Основной агент после возврата handoff сверяет его со своей canonical картой и продолжает integration.

### A4.7 — Milestone-driven autonomous reverse loop
Статус после `CHAT-013`: `OBSERVED`.

К этому моменту пользователь явно перестаёт хотеть поток промежуточных reverse-находок. Он формулирует другой контракт:
- агент сам ведёт внутреннюю карту функций/адресов/структур;
- каждую гипотезу перепроверяет по коду/disassembly/data;
- при опровержении самостоятельно меняет направление;
- использует уже загруженные artifacts без нового ручного посредничества;
- возвращается только при operator-only blocker или после существенного milestone;
- user-facing update кратко разделяет proven / changed / unresolved.

В `CHAT-013` эта модель частично проявляется practically: длинный static AWB reverse продолжается без камеры, hardware tests накапливаются до безопасного checkpoint, а JXF37-гипотеза в итоге снимается stock/hardware evidence без превращения её в permanent architecture.

Ограничение этапа: continuity всё ещё file/handoff-based, а пользователь по-прежнему вручную переносит большие archives и запускает hardware commands. До MCP/repository-native autonomous loop ещё далеко.

### A4.8 — Quality engineering of the agent workflow
Статус после `CHAT-014` + `CHAT-016`: `CONSOLIDATED`.

Впервые процесс улучшения работы агента сам становится отдельным deliverable проекта. После длинной hardware/reverse-сессии пользователь просит не просто «учесть замечания», а собрать отдельный пакет для агента, который будет улучшать agentic workflow.

В результате появляется quality-improvement pack с отдельными файлами для:
- retrospective ошибок/удачных паттернов;
- interaction protocol;
- command UX;
- handoff requirements;
- tooling/automation opportunities;
- proposed operational state files;
- инструкции главному оркестратору внедрить изменения в основной процесс.

Особенно важные предложения этого этапа:
- вынести текущий IP/path/version/compiler/checkpoint из памяти чата в `ENVIRONMENT_CURRENT.md` и machine-readable `CURRENT_SESSION.json`;
- формализовать hardware experiment как state machine;
- автоматизировать механический цикл build → stage → capture bundle → Windows-visible folder;
- минимизировать повторные SCP/password/ручные transfers;
- считать пользователя hardware-оператором, а не аналитиком.

Это ещё не repository-native automation и не MCP, но уже явный **meta-engineering layer**: система начинает проектировать собственный workflow так же, как технический runtime.

В `CHAT-016` quality layer становится итеративным: создаётся pack v3, который не переписывает старую ретроспективу, а добавляет новые lessons — stock=TFTP-only, minimal BusyBox, focused reverse corpus, heavy processing off-target, liveness heartbeat и stale-vs-active state discipline.

### A4.9 — Central orchestrator + persistent specialist lanes
Статус после `CHAT-015` + `CHAT-016`: `CONSOLIDATED`.

Параллельность становится не просто «второй агент помогает», а явной orchestration model.

Главный агент:
- держит canonical system map;
- делит работу по независимым state/dataflow boundaries, а не произвольным адресным диапазонам;
- формулирует universal prompt + specialist scope;
- запрещает specialist'ам hardware writes и самостоятельное взаимное merge;
- принимает finished handoff units, сверяет confidence и только затем интегрирует;
- сохраняет закрытые области как no-reverse zones для следующих задач.

После первых слишком коротких задач модель усложняется: специалист остаётся в той же вкладке и получает `Task 2`, `Task 3` с более крупной cohesive subsystem областью. Это сохраняет локально набранный контекст и уменьшает onboarding overhead.

Одновременно выявляется инфраструктурный предел эпохи: общий Project context не гарантирует общий файловый sandbox. Поэтому оркестратор ещё вручную прикрепляет authoritative handoff каждому specialist lane.

`CHAT-016` подтверждает A4.9 practically: тот же Agent 1 сохраняет контекст после Task 1, получает большой Task 2, проходит несколько самостоятельных static/runtime evidence циклов, закрывает full AE loop до GC1054 registers и затем переходит к Task 3 по lens/peripheral orchestration. То есть persistent specialist lane реально amortize'ит onboarding и переносит knowledge между последовательными задачами.

Это уже близко к настоящей multi-agent engineering system, но пользователь всё ещё остаётся router-ом physical artifacts между чатами.

### A4.10 — Reverse corpus architecture: handoff превращается в workspace map
Статус после `CHAT-017`: `OBSERVED`.

`CHAT-017` делает следующий шаг после quality packs и persistent specialist lanes: пользователь требует перестать воспринимать reverse inputs как набор случайных вложений.

Целевая file-based архитектура уже включает:
- заранее извлечённую stock firmware/filesystem и основные binaries/modules/libs;
- prepared disassembly, symbols, relocations, strings и task-specific extracts;
- отдельный runtime-memory layer: maps, initialized RW/GOT/heap, VMM/MMIO snapshots;
- `CORPUS_MANIFEST` и `REVERSE_STATUS_MATRIX`, чтобы было видно `PREPARED / PARTIAL / MISSING / EXACT`;
- карту Windows ↔ WSL ↔ camera ↔ archive;
- toolkit повторяемых операций;
- историю expensive work, запрещающую повторный full disassembly без доказанного gap;
- отдельные role/orchestrator instructions.

В той же сессии несколько agent-specific quality retrospectives дедуплицируются в один master quality/reverse-preparation pack.

Это ещё локальный file/handoff workspace: пользователь всё ещё вручную переносит artifacts. Но по структуре это уже прямой предшественник repository/MCP-native проекта — знания становятся адресуемой картой, а не памятью конкретного чата.

### A4.11 — Local master workspace с automated governance
Статус после `CHAT-020`: `OBSERVED`.

То, что в `CHAT-017` было proposed reverse-corpus architecture, в центральной orchestration-ветке становится реальной файловой системой проекта.

Появляются:
- единый `PROJECT_MAP.md` и role-specific bootstrap;
- current operational state в human + machine-readable форме;
- task router/context packs;
- module/implementation status matrices;
- persistent `sessions/` layer;
- runbooks для WSL/OpenIPC/stock/build/reverse;
- disassembly-first policy;
- SQLite index и lookup helpers;
- recovery playbook;
- decision/change/work/expensive-task history;
- automated handoff/path/archive/duplicate/index/coverage health checks.

Главное изменение ответственности: новому агенту больше не нужно «понять архив» целиком. Он должен войти через role/task router, прочитать current state, использовать prepared corpus и автоматически проверить workspace health.

Central orchestrator теперь интегрирует specialist results в normalized knowledge, а raw agent outputs сохраняет как provenance, не как второй параллельный source of truth.

Ограничение этапа сохраняется: workspace всё ещё переносится handoff-архивами и physical attachments; общий GitHub/Drive/Ghidra MCP authority исторически ещё впереди.

### A4.12 — Matrix-driven hardware evidence acquisition
Статус после `CHAT-021`: `OBSERVED`.

После того как local master workspace получил task routing и governance, тот же принцип переносится на stock hardware evidence.

Операторская сессия больше не строится как «сними ещё один dump». Появляются:
- canonical state matrix;
- transition matrix;
- capture manifests;
- evidence catalog;
- corrected/superseded metadata;
- targeted before/early/settled regions;
- explicit tool/capability limitations;
- static reverse-preparation status;
- remaining acquisition gaps;
- delta package относительно stable master.

Оператор по-прежнему вручную выполняет hardware-only actions, но всё больше может делать это без анализа: agent заранее выбирает state, recipe, expected file sizes и stop condition, затем сам интерпретирует результат.

Это важный промежуточный уровень автономности: **agent owns experimental design and evidence normalization; human owns physical actuation/command execution**.

При этом выявляется новый safety dimension — resource budget. Read-only capture может разрушить stock runtime через OOM, поэтому размер/tmpfs/RAM становятся частью experiment prerequisites.

### A4.13 — Parallel implementation lane + orchestrated hardware convergence
Статус после `CHAT-022`: `OBSERVED`.

После evidence/reverse lanes появляется отдельный production-integration executor. Его роль уже принципиально другая:
- не делать широкий reverse;
- читать existing contracts/reference code;
- догонять owner;
- отдавать пользователю testable source;
- включать новые features по одной;
- анализировать hardware feedback;
- отправлять orchestrator delta + lifecycle/source-consolidation debt.

Человек остаётся аппаратным исполнителем: WSL build, transfer, UART target commands, визуальная оценка. Agent владеет кодом, test design, разбором результата и следующей гипотезой.

Сессия показывает настоящий multi-agent pipeline:
`reverse specialists → normalized master → implementation agent → hardware operator → orchestrator`.

Появляется и новый bottleneck: уже не скорость reverse, а качество source integration, lifecycle и аппаратных regression cycles.

### A4.14 — Frozen-base fan-out / DELTA fan-in orchestration
Статус после `CHAT-023`: `OBSERVED`.

Multi-agent работа перестаёт быть просто «несколько вкладок параллельно». Оркестратор формализует release barrier:

1. все активные agents стартуют от одной frozen master version;
2. acquisition/reverse/implementation работают независимо в своих scopes;
3. каждый возвращает только DELTA, не новый master;
4. orchestrator накапливает результаты без version bump;
5. после завершения wave выполняется conflict/dedup/source-consolidation pass;
6. только затем выпускается следующая canonical master.

Параллельно разделяется sequencing:
- implementation может сразу использовать уже доказанные contracts;
- evidence-agent заранее закрывает очевидные corpus gaps;
- deep reverse стартует после evidence normalization;
- orchestrator остаётся единственной точкой canonical merge.

Пользователь отдельно замечает, что parallel-agent режим появился только примерно 27 августа; за последующие ~сутки orchestration стала похожа на небольшую embedded/reverse team, а bottleneck сместился от raw analysis к hardware validation и integration quality.

### A4.15 — Cross-platform research intelligence lane
Статус после `CHAT-025`: `OBSERVED`.

External specialist получает стабильную самостоятельную роль между broad web research и exact target reverse.

Он не обязан сам закрывать FH8626 instruction-level contract. Вместо этого:
- берёт current UNKNOWN/PARTIAL;
- ищет exact/close SDK, BSP, OEM firmware, debug sources, kernel drivers и sample apps;
- сравнивает несколько независимых implementations;
- ищет архитектурные omissions;
- формирует compact targeted leads для Agent 1;
- отдельно готовит acquisition requests для недоступных artifacts;
- OpenIPC lane получает только research-support material.

Это разгружает exact reverse-agent от поиска vendor vocabulary/source analogues и одновременно не позволяет external reference стать target truth.

### A4.16 — Two-tier local storage: working core + heavy reverse vault
Статус после `CHAT-026`: `OBSERVED`.

Local governed workspace достигает предела удобства: full dumps, disassemblies, RAM/VMM/MMIO, RAW/YUV и исторические captures уже слишком велики для everyday handoff.

Появляется явное разделение:
- `MASTER_CORE` — часто обновляемая textual/current authority для обычных agents;
- `REVERSE_HEAVY` — редко обновляемый source/reverse vault с bulk/immutable evidence.

MASTER_CORE не копирует bulk; он содержит stable logical IDs и index, указывающий, какой heavy artifact нужен для конкретного deep reverse.

Одновременно transport storage получает lifecycle:
`tftp_active → canonical store/tftp_history`,
а корень TFTP перестаёт быть долговременным архивом.

Это ещё не Google Drive/GitHub architecture, но это прямой функциональный предшественник современной разделённой authority model: **оперативное знание и тяжёлое первичное evidence получают разные storage roles**.

### A4.17 — Closure loop между reverse-agent и evidence-agent
Статус после `CHAT-027`: `OBSERVED`.

Parallel-agent система становится циклической, а не только fan-out/fan-in.

Agent 1 сначала доходит до static boundary и возвращает не абстрактное «нужны логи», а E1–E7 с:
- named states;
- exact functions/addresses;
- required observations;
- acceptance criteria.

Agent 4 получает именно этот evidence contract, проводит target acquisition и возвращает raw/live DELTA. Agent 1 затем повторно подключается к тому же вопросу, но уже не делает broad reverse: он связывает новые target facts с имеющимся static corpus и выдаёт implementation-facing closure.

Схема:
`reverse → named evidence gap → acquisition → target evidence → targeted reverse → implementation contract`.

Оркестратор остаётся точкой merge/provenance, а пользователь — аппаратным исполнителем и маршрутизатором архивов. Это уже близко к полноценной research pipeline, но shared durable storage между agents ещё не устраняет ручную передачу файлов.

### A4.18 — Role pipeline и orchestrator-owned dependency graph
Статус после `CHAT-028`: `OBSERVED`.

Центральный orchestrator уже управляет не просто параллельными tasks, а зависимостями между типами работы:

`External Research → Exact FH8626 Reverse → OpenIPC Productization`

и отдельным hardware evidence loop из A4.17.

Productization-agent может открыть integration gap, но не реверсит Apollo сам. Orchestrator маршрутизирует gap во external/reference lane и exact reverse lane, затем возвращает подтверждённый contract обратно implementation/productization.

Это важный сдвиг: специализация определяется не адресом или файлом, а **типом доказательства и ownership результата**.

CHAT-028 одновременно показывает недостаток этой схемы: bare labels `Agent 1/2/3` уже повторяются между orchestration waves. Без stable role/wave IDs provenance начинает путаться даже у центрального агента.

### A4.19 — Productization specialist becomes executable source lane
Статус после `CHAT-029`: `OBSERVED`.

Agent 3 no longer returns only architecture/research notes. The lane produces source-level reusable units, tests, Buildroot staging and upstream RFC material while preserving explicit hardware-evidence boundaries.

Its ownership rule is narrow:
- it may build userspace/runtime abstractions around confirmed contracts;
- it must not invent missing FH8626 HAL structures;
- it does not change canonical reverse truth;
- unverified target work remains source/host-tested, not hardware PASS;
- missing SDK/Majestic support becomes a blocker ledger, not a reason to stop all productization.

This is the first historical lane that looks like a conventional software-development workstream fed by reverse contracts.

### A4.20 — Cumulative kernel specialist lane
Статус после `CHAT-030`: `OBSERVED`.

Kernel work becomes another persistent specialist lane, but this chat exposes a coordination anti-pattern: lack of direct access to the user's WSL source tree caused the agent to serialize every intermediate stage into a separate archive.

The corrected model is:
`offline analysis/source preparation → one cumulative working line → one operator WSL validation runner → report → next delta`.

Intermediate audits remain internal provenance instead of becoming dozens of user-facing handoffs.

This is an important step toward later repository-native development: the desired unit of continuity is the source line/commit history, not numbered transport archives.

### A4.21 — Interaction failures become executable quality artifacts
Статус после `CHAT-031`: `OBSERVED`.

User corrections stop being ephemeral chat feedback and are explicitly turned into durable branch artifacts:
- `CRITICAL_COMMAND_PROTOCOL.md`;
- repeated-failure history;
- orchestrator required actions;
- startup/pre-send/artifact-consistency expectations;
- positive and negative command examples.

The important shift is from:
`agent makes mistake → user corrects → agent apologizes`

to:
`repeated mistake → classify root cause → encode invariant + example + checklist → inject into next agent bootstrap`.

CHAT-031 also shows why documentation alone is insufficient: the rule already existed in prior files, yet the agent still violated it. The next maturity step therefore has to be **pre-send enforcement/linting**, not merely more prose.

### A4.22 — Failed release → negative-evidence handoff
Статус после `CHAT-032`: `OBSERVED`.

Agent 7 впервые даёт особенно чистый пример того, что failure branch тоже должен быть нормальным engineering artifact. После серии неудачных R13–R18 пользователь требует не «ещё один фикс», а оформить весь провал для оркестратора.

Получившаяся модель:
`release attempt → physical regression → stop → quarantine → enumerate violated contracts / rejected hypotheses / useful findings → orchestrator decides cherry-picks`.

Важное изменение — failed branch больше не пытается сохранить лицо повышением версии. Он обязан явно сказать, какие части мусорны/неподтверждены и что не должно попасть в canonical line. Это защищает multi-agent fan-in от ложной уверенности и превращает неудачную hardware-сессию в reusable negative evidence.


### A4.23 — Semantic Ghidra reverse workspace
Статус после `CHAT-033`: `OBSERVED`.

После накопления ошибок flat-disassembly workflow проект меняет сам интерфейс между binary и агентом. Цель больше не «дать LLM больше ARM TXT», а построить machine-generated semantic substrate:
- decompiled function bodies;
- CFG/call graph/XREF;
- shared types/symbols;
- unresolved indirect flow;
- cross-binary links;
- confidence/provenance;
- snapshot/delta exchange между чатами.

Ghidra предлагается как локальный WSL analyzer/headless backend, а обычные файлы/knowledge index — как общий язык browser-agents. Это важный шаг к будущему Ghidra MCP: reverse knowledge начинает жить во внешней mutable model, а не в памяти одного чата.


### A4.24 — Analyst → integrator source handoff
Статус после `CHAT-035`: `OBSERVED`.

Роли становятся асимметричными и более эффективными. Deep-analysis agent получает право:
- долго разбирать ASM/Ghidra;
- писать C/H candidates;
- делать host/negative/sanitizer checks;
- прикладывать minimal diff/evidence/Ghidra pointers.

Но он не обязан и не должен автоматически становиться target integrator. Второй agent получает уже переваренный contract и отвечает за actual branch integration, ARM build, firmware/deploy и hardware acceptance.

Это уменьшает повторный reverse у integration-agent и одновременно не даёт browser-analysis ветке объявлять host-tested code production-ready.


### A4.25 — Canonical indexed workspace + external recovery checkpoints
Статус после `CHAT-036`: `OBSERVED`.

Handoff-файл перестаёт быть единственным носителем continuity. Проект пытается хранить engineering state как:
`one canonical directory + machine index + current docs + deep provenance + external recovery checkpoints`.

Появляется явное различие:
- working state;
- checkpoint для восстановления;
- continuation/handoff для смены агента;
- historical evidence.

Ещё важнее correction пользователя: никакой startup script не заменяет actual context read. Automation отвечает за integrity, агент — за понимание. Это становится базовой предпосылкой будущего repository/Drive-based workflow.


### A4.26 — Drive-backed workspace + local corpus reconciliation
Статус после `CHAT-037`: `OBSERVED`.

Локальный workspace впервые получает реальное внешнее persistent mirror: Google Drive хранит browseable canonical docs/index и полный checkpoint. Это закрывает failure mode, где browser-session local files исчезали между сессиями.

Параллельно local WSL corpus очищается не «rm старое», а полноценной archaeology/reconciliation:
- installed tools/build products выносятся за project boundary;
- transport archives inventory/dedup;
- unique historical contents materialize into semantic reverse corpus;
- active project и heavy history получают разные transfer roles;
- следующий agent должен начинать с persistent workspace context, а не краткого chat summary.

Это первый фактически доказанный шаг bootstrap-перехода `workspace → Drive`. GitHub authority и Ghidra MCP как следующий уровень ещё исторически не наступили в этом источнике.


### A5 — Специализированные MCP/агенты
Статус: `BOOTSTRAP`.

В `CHAT-001` ещё не наблюдается. Следующие чаты должны показать переход от ручного handoff к прямому доступу агентов к reverse/storage/repository state.

### A6 — Агентная инженерная система
Статус: `BOOTSTRAP`.

В `CHAT-001` пользователь всё ещё выполняет очень много промежуточных команд. Однако направление уже видно: агент всё больше берёт на себя reverse, подготовку probes, интерпретацию результатов и документирование, а пользователь начинает требовать сокращения ручного диспетчерства.

`CHAT-009` особенно хорошо показывает предел ручного режима: сложные UART paste-блоки повреждаются, пользователь вынужден вручную восстанавливать process/kernel state и отдельно просит прекратить микрошаги. Это сильный исторический аргумент в пользу будущего direct agent workspace/operational tooling.

В `CHAT-011` появляется ещё одна граница ручного режима: proven development image начинает расходиться с canonical source из-за временных overlay/init/network mutations. Без внешнего debt ledger пользователь вынужден сам напоминать, что перед final port эти изменения нельзя забыть вернуть или интегрировать чисто.

## Уроки CHAT-001…CHAT-031 для будущей agentic-системы

1. **Текущее runtime state должно быть внешним фактом, а не памятью диалога.** Потери «stock или OpenIPC?» породили дорогие ошибки.
2. **Agent handoff — необходим, но не должен становиться гигантской свалкой.** Нужны краткая карта и подробные приложения.
3. **Оператор не должен анализировать ветвление.** Его задача — выполнить hardware-only действие и вернуть факт.
4. **Granularity команд адаптивна.** Новые опасные развилки — пошагово; доказанный routine — одним блоком.
5. **Infrastructure loop тоже часть разработки.** Boot/SSH/SCP/recovery нужно оптимизировать так же, как код.
6. **Unknown должен оставаться unknown.** В handoff лучше явный provenance gap, чем уверенное восстановление по памяти.
7. **Mutation scope обязателен.** Анализ/вопрос не означает разрешение менять canonical artifact.
8. **Critical actions должны переживать duplicate execution.** Для stateful hardware одного предупреждения недостаточно — нужен lock/guard.
9. **Artifact delivery — отдельный интерфейс.** Один test-stage должен иметь один понятный archive + готовые build/transfer/test blocks.
10. **Parallelism полезен только при явном ownership.** Второй агент ускоряет работу, если scope не пересекается и его output имеет форму интегрируемого finished unit.
11. **Аппаратный тест должен быть детерминированным.** Не заставлять оператора вручную отсчитывать время и потом восстанавливать границы состояний по общему логу.
12. **Cheap preflight должен происходить до hardware loop.** Compile/self-test/archive check — работа агента, а не аппаратного оператора.
13. **Повторяющийся boot/recovery ritual нужно автоматизировать.** Именованные проверенные операции уменьшают операторскую нагрузку и стоимость каждой итерации.
14. **Повторяющийся reverse lookup нужно превращать в self-service evidence.** Если агент много раз просит ranges одного source, передать/подключить источник целиком.
15. **Живой hardware state нужно эксплуатировать экономно.** Лучше один read-only evidence bundle и офлайн-анализ, чем десятки мелких target probes.
16. **Execution lane должен быть явным.** WSL, UART и U-Boot — разные интерфейсы; оператор не должен переводить команды между ними.
17. **Стандартный dev-tool предпочтительнее временного parser-а.** Установка ffmpeg/ffprobe и подобных инструментов дешевле поддержания самописного обхода, если задача стандартная.
18. **Сначала immutable recovery anchor, потом мутация.** Full-flash dump и RAM execution path резко уменьшают цену ошибок в hardware reverse.
19. **Проверять нужно функциональный путь следующего шага.** TFTP success важнее необязательного ICMP health-check, если именно TFTP нужен для RAM boot.
20. **Сырые аппаратные данные не должны требовать предварительного анализа пользователя.** Оператор может отдавать bootlog/dump как есть; структурирование и технический вывод — работа агента.
21. **Перед новой target-командой проверять уже имеющийся corpus.** Если файл или модуль уже содержится в dump/archive/workspace, извлечение должен делать агент.
22. **Полный firmware dump — не только backup, но и offline development substrate.** Чем больше анализа переносится с живой камеры на immutable artifact, тем дешевле и безопаснее итерации.
23. **Длинный reverse требует карты прогресса по engineering boundaries.** Пользователь не должен угадывать, сколько подсистем ещё осталось до следующего hardware milestone.
24. **External reverse/reference нужно искать до продолжения дорогого blind reverse.** Соседний SoC может дать API architecture и правильные названия, но не заменяет доказательство ABI/registers на target.
25. **Рабочая документация и upstream contribution — разные продукты.** Current facts, failed experiments, provenance и reproducible bring-up должны жить раздельно.
26. **Repository access — следующий естественный шаг после file handoff.** CHAT-007 уже формулирует эту потребность, хотя фактический Git authority появляется позже.
27. **Granularity определяется decision boundary, а не количеством команд.** Routine без ветвления группируется; неизвестный шаг останавливается только там, где агенту нужен результат для следующего вывода.
28. **При потере source предпочитать runtime contract capture полному re-reverse.** Interposition/trace может восстановить нужный ioctl ABI быстрее и точнее.
29. **Unsupported SoC выгодно портировать как reusable backend, а не одноразовую плату.** Тяжёлый reverse должен превращаться в общий слой, чтобы следующие board-порты стали sensor/GPIO/profile задачами.
30. **Сложный control flow нельзя делать UART-интерфейсом.** Multi-line `if`, regex, quoting и critical MMIO/process logic должны жить в переданном script/helper; serial shell — для короткого запуска и чтения результата.
31. **Handoff должен проходить reproducibility test.** Новый агент должен суметь найти canonical artifacts и повторить build/transfer/reverse/run без неявной памяти автора.
32. **Infrastructure failure нужно локализовать по boundaries.** Network up, TCP listen, protocol banner, auth и PTY — разные gates; измерять их отдельно дешевле, чем менять несколько подсистем сразу.
33. **Handoff должен позволять начать работу с текущего blocker без рекапитуляции проекта.** CHAT-010 показывает, что это проверяемое свойство, а не просто качество текста.
34. **Прошлый агент — fallback для уникального gap, не штатная база данных.** Сначала использовать текущие artifacts и handoff, потом при необходимости вытаскивать отсутствующий provenance/context.
35. **Свежая regression observation важнее старого удобного объяснения.** Если пользователь знает, что до наших изменений путь был быстрым/рабочим, сначала изолировать delta, а не накрывать проблему workaround-ом.
36. **Development workaround должен иметь явный exit plan.** Временная правка обязана сразу получить метку dev-only, место в source/image и действие перед final/upstream: revert, board-scope или clean integration.
37. **Hardware-proven image и canonical source — разные состояния.** Если generated rootfs/cpio содержит ручные proven fixes, это нужно считать reconciliation debt, а не молча объявлять source tree актуальным.
38. **Living handoff обновляется reconciliation-ом, а не размножением master-файлов.** Authoritative верх меняется под fresh facts, historical evidence остаётся с superseded-метками.
39. **Productionization можно начинать до полного закрытия reverse, но только за доказанной boundary.** Стабильные boot/module/stream слои можно оформлять параллельно; unresolved RAW/ISP должен оставаться изолированным за одним owner и не смешиваться со streamer/RTSP.
40. **Streamer не должен владеть stateful hardware только потому, что он конечный продукт.** Если lifecycle требует одного долгоживущего owner, media daemon должен держать vendor fd, а Majestic/другой streamer получать уже готовый stream contract.
41. **Autonomous reverse — это milestone loop, а не поток сообщений.** Агент должен внутренне проверять и отбрасывать гипотезы и репортить только operator blocker или существенный завершённый этап.
42. **Firmware support не доказывает hardware population.** Driver/blob/format — capability evidence; конкретный BOM подтверждается stock runtime и физическим target evidence.
43. **Recovered control loop сначала работает в shadow mode.** Для AWB/AE и других динамических алгоритмов безопаснее offline tests → live shadow compute → gated commit, чем сразу писать в hardware state.
44. **Completion требует отдельного requirement audit.** Главный механизм может быть найден, но задача не должна называться DONE до сверки со всеми исходными deliverables/evidence gates.
45. **Operational session state должен жить вне памяти чата.** IP, active owner, compiler, checkpoint, Windows/WSL roots и unsafe workflows нужно хранить в одном current-state artifact.
46. **Эксперимент лучше моделировать state machine.** Это уменьшает invalid captures и делает baseline/change/readback/rollback воспроизводимыми.
47. **Quality retrospective — такой же инженерный artifact, как reverse handoff.** Повторяющиеся ошибки нужно собирать, версионировать и превращать в правила/automation, а не исправлять устно заново.
48. **Parallel-agent scope должен быть достаточно крупным, чтобы окупать onboarding.** Лучше persistent lane с Task N по целому subsystem, чем новая вкладка на каждую функцию.
49. **Project context и artifact bytes — разные ресурсы.** Specialist не начинает exact reverse, пока не подтвердил доступ к authoritative inputs.
50. **Главный агент интегрирует, specialists исследуют.** Они возвращают finished units с confidence/unresolved/integration notes, но не сливают друг друга и не меняют hardware runtime самостоятельно.
51. **Semantic oracle ускоряет reverse, но не заменяет target proof.** Именованный соседний Fullhan сначала даёт смысл/структуру, затем Apollo/FH8626 подтверждает ABI/state/MMIO.
52. **External research должен иметь provenance и reusable method.** Один living research document полезнее серии забытых веб-находок.
53. **Длинная автономная работа требует редкого liveness heartbeat.** Findings репортятся по milestone, но затянувшийся tool/reverse pass не должен выглядеть как зависание.
54. **Heavy static work выполняется off-target.** Камера нужна для runtime evidence; disassembly/diff/parsing/selftests — WSL/PC, желательно по уже подготовленным TXT.
55. **Focused corpus лучше recursive project scan.** Для subsystem reverse собрать минимальный набор файлов и искать адресно, а не обходить всё дерево.
56. **Runtime number без lifecycle semantics не является current state.** Dirty/cache/deferred queue нужно отличать от активного controller state.
57. **Persistent specialist lane доказан как рабочая единица orchestration.** Один thread может последовательно закрывать Task 1/2/3, сохраняя локальный контекст и возвращая интегрируемые units.
58. **Static code и live mutable data — два разных слоя evidence.** ARM_FULL отвечает за code/xrefs, runtime image — за GOT/RW/heap/tables; не надо повторно дизассемблировать mutable data.
59. **Specialist role нужно фиксировать так же жёстко, как artifact access.** Общий handoff не должен стирать ownership конкретного Agent/Task.
60. **Reverse corpus должен быть подготовлен до следующего агента.** Manifest/status map/toolkit уменьшают просьбы к оператору и повтор expensive work.
61. **Retrospectives нескольких агентов нужно консолидировать.** Общий process contract полезнее набора несовместимых quality packs.
62. **Workspace invariants должны исполняться, а не только описываться.** Doctor/path/archive/duplicate/index checks превращают quality rules в автоматические gates.
63. **Role/task routing должен определять контекст до чтения corpus.** Agent получает релевантный context pack вместо всего master-tree.
64. **Continuity долгой задачи живёт в session ledger.** Checkpoint, blocker, expensive work и next action должны переживать chat/context loss.
65. **Normalized knowledge и provenance — разные слои.** Raw agent result сохраняется для проверки, но active source of truth остаётся один.
66. **Progress должен иметь ontology.** Module/status matrix надёжнее одной плавающей цифры процентов.
67. **Read-only capture тоже имеет resource budget.** Размер tmpfs/RAM/I/O должен проверяться до hardware acquisition.
68. **Manifest должен различать intended и observed state.** Metadata correction сохраняется отдельно, raw evidence не переписывается молча.
69. **Transition evidence обычно ценнее независимых full-state heap diffs.** Synchronized targeted captures уменьшают temporal noise.
70. **Hardware acquisition тоже нуждается в coverage matrix.** Canonical/superseded/unavailable states должны быть видимы до следующего эксперимента.
71. **Evidence лучше передавать delta-пакетом поверх stable master.** Новые captures/static material не требуют пересборки всего корпуса знаний.
72. **Implementation-agent — отдельная роль от reverse-agent.** Его задача догнать code до доказанного contract, а unknown возвращать как узкий research request.
73. **Hardware operator и agent должны делить ответственность явно.** Agent готовит source/test/analysis, человек выполняет authoritative build/target run.
74. **После быстрого experimental роста нужен source-consolidation gate.** Diagnostic snapshots нельзя автоматически повышать до production tree.
75. **Рабочий механизм не равен готовой архитектуре.** Experimental shutdown или register path может быть hardware-proven и одновременно оставаться production debt.
76. **Parallel wave должна иметь frozen base.** Иначе каждый ранний DELTA создаёт новую несовместимую master-version.
77. **Fan-in принадлежит оркестратору.** Specialist возвращает delta; canonical state меняется только после единого reconcile pass.
78. **Evidence acquisition можно вынести впереди reverse.** Один structured capture pass дешевле серии случайных hardware blockers у research-agent.
79. **100% — допустимый статус для узкого доказанного contract.** Нельзя смешивать его с product/upstream readiness всей подсистемы.
80. **External research может быть отдельной intelligence-lane.** Ее output — targeted leads/acquisition requests, не второй independent target reverse.
81. **Искать нужно и архитектурные omissions.** Повторяющийся stage в нескольких vendor implementations — повод для точечной проверки target.
82. **Найденный URL не равен имеющемуся artifact.** External evidence проходит статусы discovered/acquired/prepared/verified.
83. **Cross-platform confidence зависит от дистанции.** Same-SoC source сильнее дальнего homolog, но target proof всё равно обязателен.
84. **Working core и heavy evidence требуют разных storage roles.** Everyday agent context не должен таскать full dumps/disassemblies.
85. **Heavy artifacts адресуются logical IDs.** Physical path/storage может меняться, knowledge references остаются стабильными.
86. **Transport staging не является архивом.** Active transfer files должны иметь lifecycle и уходить из root после завершения шага.
87. **Удалять reverse history можно только после canonicalization.** Сначала inventory/move/dedup proof, затем destructive cleanup.
88. **Static exhaustion должен порождать evidence contract, а не ещё один broad pass.** Named capture + acceptance criteria позволяют другому агенту закрыть физическую границу.
89. **Evidence-agent и reverse-agent могут образовывать замкнутый цикл.** Первый собирает ровно недостающие target facts, второй возвращается только к затронутым функциям.
90. **Focused handoff обязан замыкать dependencies.** Heavy artifact можно не дублировать, но он должен быть embedded или иметь verified durable locator.
91. **Максимальный reverse и release-ready reverse — разные цели.** Optional archaeology нельзя случайно смешивать с критическим портинговым backlog.
92. **При больших disassembly работать нужно address-window/xref методом.** Это быстрее, устойчивее к слабой среде и меньше засоряет контекст.
93. **Bare Agent N не является durable identity.** Нужны wave/role/task IDs, иначе разные orchestration waves становятся неразличимы.
94. **Integration gaps должны маршрутизироваться по типу доказательства.** Productization формулирует blocker, external lane даёт semantic lead, exact reverse подтверждает target contract.
95. **Hardware backend должен переживать смену frontend-а.** Majestic/Divinus — consumers, а camera-level contract остаётся отдельным.
96. **Engineering bring-up и upstream supply chain — разные gates.** Можно двигать target работу без ложного заявления upstream readiness.
97. **Verification принадлежит exact artifact identity.** Repacked/reconstructed patch не наследует PASS старой версии без byte/commit continuity.
98. **Productization specialist может работать до native HAL.** Sidecar/runtime/source packages дают реальный progress поверх подтверждённого owner contract.
99. **Cross-architecture sidecar должен иметь explicit wire ABI.** Native C layouts не являются protocol.
100. **Source-only staging лучше фиктивной firmware integration.** Не добавлять generated binaries или opaque stock payload, чтобы создать видимость готового OpenIPC target.
101. **Transport archive не должен быть unit of development.** Offline work накапливается в одной source line и выдаётся на validation boundary.
102. **Отсутствие прямого workspace access нельзя компенсировать artifact spam.** Лучше один cumulative runner/report, чем десятки numbered stages.
103. **Простой shell/tar предпочтительнее генератора-обёртки.** Tool complexity должна соответствовать задаче.
104. **Kernel platform и media/VMM имеют разные risk gates.** Сначала закрывается безопасный SoC baseline, затем high-risk multimedia layer.
105. **Повторяющийся UX-дефект должен стать executable quality rule.** Одного «я запомнил» недостаточно — нужны примеры, checklist и bootstrap.
106. **Claimed fix проверяется по working bytes до ответа.** Narrative state не является source of truth.
107. **Authoritative validation surface должна быть одна.** Если build/test принадлежит WSL владельца, локальный agent status остаётся PENDING.
108. **Capability API и provider availability — разные вещи.** Frontend можно подготовить заранее, не создавая фиктивный hardware backend.
109. **Критические interaction invariants нуждаются в pre-send enforcement.** Иначе даже прочитанный runbook не гарантирует соблюдение.

## Следующие исторические переходы, которые нужно искать

- когда локальный workspace перестал быть достаточным;
- когда появился Google Drive как долговременное evidence-хранилище;
- когда GitHub стал source of truth для текущего состояния;
- когда agents получили прямой MCP-доступ вместо пользовательского handoff;
- когда пользователь перестал быть главным маршрутизатором файлов/контекста между агентами.


## Подтверждение после CHAT-034

Расширенный orchestrator-thread не требует нового A-этапа: он делает устойчивыми уже описанные A4.14/A4.16/A4.17/A4.18. Parallel workers стартуют от frozen master, evidence-agent закрывает named physical gaps, reverse-agent получает normalized delta, productization работает независимо, а orchestrator единолично делает fan-in. Двухуровневый MASTER_CORE/REVERSE_HEAVY перестаёт быть только storage proposal и становится физической рабочей схемой.
