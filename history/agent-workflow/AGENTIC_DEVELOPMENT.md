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

### A5 — Специализированные MCP/агенты
Статус: `BOOTSTRAP`.

В `CHAT-001` ещё не наблюдается. Следующие чаты должны показать переход от ручного handoff к прямому доступу агентов к reverse/storage/repository state.

### A6 — Агентная инженерная система
Статус: `BOOTSTRAP`.

В `CHAT-001` пользователь всё ещё выполняет очень много промежуточных команд. Однако направление уже видно: агент всё больше берёт на себя reverse, подготовку probes, интерпретацию результатов и документирование, а пользователь начинает требовать сокращения ручного диспетчерства.

`CHAT-009` особенно хорошо показывает предел ручного режима: сложные UART paste-блоки повреждаются, пользователь вынужден вручную восстанавливать process/kernel state и отдельно просит прекратить микрошаги. Это сильный исторический аргумент в пользу будущего direct agent workspace/operational tooling.

В `CHAT-011` появляется ещё одна граница ручного режима: proven development image начинает расходиться с canonical source из-за временных overlay/init/network mutations. Без внешнего debt ledger пользователь вынужден сам напоминать, что перед final port эти изменения нельзя забыть вернуть или интегрировать чисто.

## Уроки CHAT-001/002/003/004/005/006/007/008/009/010/011/012/013/014/015/016/017 для будущей agentic-системы

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

## Следующие исторические переходы, которые нужно искать

- когда локальный workspace перестал быть достаточным;
- когда появился Google Drive как долговременное evidence-хранилище;
- когда GitHub стал source of truth для текущего состояния;
- когда agents получили прямой MCP-доступ вместо пользовательского handoff;
- когда пользователь перестал быть главным маршрутизатором файлов/контекста между агентами.
