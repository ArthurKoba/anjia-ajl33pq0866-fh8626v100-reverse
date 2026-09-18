# Улучшения workflow и best practices

Назначение: фиксировать изменения, которые реально уменьшали ручную работу, число ошибок, количество итераций, потерю контекста или стоимость передачи данных между пользователем и агентами.

## Подтверждённые улучшения

### I-004 — Явный transport contract для target
Статус: `CONSOLIDATED`.

Прежняя проблема: канал передачи выбирался по привычке, из-за чего появлялись SFTP/SCP ошибки или попытки использовать SSH там, где его нет.

Новый подход:
- для каждого boot/runtime state заранее знать доступный transport;
- на OpenIPC использовать подтверждённый SCP legacy mode, если target не имеет SFTP server;
- на stock использовать подтверждённый TFTP/UART workflow;
- не переключать транспорт без выгоды.

Результат: меньше бесполезных итераций и меньше ручной перекладки файлов.

Источник: `CHAT-005` формулирует стратегию и доказывает U-Boot/TFTP→RAM transport; `CHAT-001` затем аппаратно подтверждает сам OpenIPC RAM boot.

### I-006 — Архив/checkpoint как транспорт состояния, рабочие файлы — распакованными
Статус: `OBSERVED`.

В `CHAT-001` перед reboot начали собирать checkpoint с helper-бинарниками и диагностическим состоянием, передавать его одним пакетом, а затем продолжать работу с распакованным содержимым.

Полезный эффект: recovery после reboot перестал означать повторное создание каждого helper-а.

Ограничение: checkpoint должен иметь явную структуру и не становиться единственным source of truth.

`CHAT-008` показывает практическую ценность checkpoint как active recovery substrate: после очистки `/tmp` старые sensor/ISP/H.264 helper'ы были найдены в `checkpoints/openipc-20260825` и переиспользованы вместо повторной сборки с нуля.

### I-007 — Buildroot/OpenIPC собирать в Linux filesystem WSL, не на Windows mount
Статус: `OBSERVED`.

Проблема: сборка на `/mnt/c` ломалась на hardlink/rsync semantics; Windows `PATH` с пробелами ломал Buildroot dependency check.

Устойчивый рецепт:
- исходники/build output держать внутри Linux filesystem;
- перед сборкой использовать контролируемый PATH или воспроизводимый wrapper;
- готовый артефакт уже копировать в Windows/TFTP area.

Источник: `CHAT-001`.

### I-008 — Безопасный bring-up: stock kernel + внешний OpenIPC initramfs из RAM
Статус: `CONSOLIDATED`.

Прежний риск: ранняя перепрошивка NOR могла усложнить recovery до того, как понятен kernel/userspace контракт.

Новый подход: оставить stock kernel, загрузить OpenIPC userspace как внешний initramfs через U-Boot/TFTP и доказать UART/network/userspace до записи flash.

Результат: первый OpenIPC milestone был получен без изменения NOR; это резко снизило цену эксперимента.

Источник: `CHAT-005` формулирует стратегию и доказывает U-Boot/TFTP→RAM transport; `CHAT-001` затем аппаратно подтверждает сам OpenIPC RAM boot.

### I-009 — Checkpoint перед reboot / destructive experiment
Статус: `OBSERVED`.

Перед перезагрузкой стали сохранять:
- helper binaries;
- ключевое runtime-состояние;
- диагностические выводы;
- boot contract.

Результат: после reboot можно восстанавливать доказанный baseline, а не повторять reverse с нуля.

Источник: `CHAT-001`.

### I-010 — Один canonical handoff вместо множества параллельных версий
Статус: `CONSOLIDATED`.

В конце `CHAT-001` появился большой handoff, затем в него были аккумулированы уникальные замечания второго агента. Позже был выбран один master-файл, а старые версии признаны backup/obsolete.

Что полезно:
- новый агент получает проверенную точку входа;
- явно сохранены доказанные факты, открытые gaps и способы воспроизведения;
- provenance gap не маскируется выдуманным объяснением.

Что улучшить в будущем: master handoff не должен бесконечно раздуваться; краткая карта + подробные приложения лучше одной гигантской стены текста. Именно поэтому текущий аудит использует двухуровневую хронологию.

Источник: `CHAT-001`.

### I-011 — Reverse только до уровня, который разблокирует следующий эксперимент
Статус: `CONSOLIDATED`.

Проблема: был начат детальный разбор каждого callback/format-id sensor library, хотя главной целью оставался запуск video pipeline.

Коррекция: после замечания пользователя reverse остановили на достаточном уровне и перешли к загрузке vendor media modules и hardware bring-up.

Обобщаемое правило: **reverse is question-driven**. Если найденного уже достаточно для практического теста, тест имеет приоритет над дальнейшим статическим разбором.

Источник: `CHAT-001`.

CHAT-007 даёт прямое подтверждение: после вопроса пользователя о чрезмерности sensor-vtable reverse работа была остановлена на достаточном уровне и переключена на загрузку media stack.

### I-012 — Один долгоживущий owner для stateful vendor media pipeline
Статус: `CONSOLIDATED`.

Проблема: повторное open/close и concurrent instrumentation vendor ISP/PAE fd приводили к dirty state, зависаниям и power-cycle.

Новый подход: перейти к одному долгоживущему процессу, который владеет ISP/PAE/media/VMM и принимает команды на capture/status/force-I-frame без переоткрытия контекста.

Результат: резко сокращается число reboot и вероятность driver lifetime bugs.

Источник: `CHAT-001`.

### I-013 — Оптимизация dev-loop SSH, а не только самой прошивки
Статус: `OBSERVED`.

В ходе `CHAT-001` выяснилось, что долгий SSH после RAM boot был отдельным bottleneck разработки. Были устранены ненормальная задержка RNG/host-key lifecycle и PTY-проблема; после этого оставшуюся криптографическую задержку признали характеристикой слабого CPU, а не продолжили бесконечную оптимизацию. Для повторных операций предложено переиспользование SSH-соединения.

Обобщаемый урок: ускорять нужно и **контур эксперимента** — boot, network, auth, file transfer, recovery — потому что десятки итераций делают эти секунды/минуты частью стоимости reverse.

Источник: `CHAT-001`.

### I-014 — Разделять «работает» и «полностью доказано»
Статус: `OBSERVED`.

В конце чата handoff специально сохранял provenance gaps и различал:
- рабочий hardware bring-up;
- валидный H.264;
- отсутствие доказательства корректного moving image/RAW для части параметров;
- неполностью доказанные sensor/lens соответствия.

Обобщаемое правило: не повышать уровень evidence молча. «Устройство отвечает» ≠ «production contract доказан».

Источник: `CHAT-001`.

### I-015 — Формальный one-archive delivery protocol
Статус: `OBSERVED`.

В `CHAT-002` пользователь превратил разрозненные привычки передачи файлов в явный контракт:

- один осмысленный test-stage = один уникально versioned archive;
- для серьёзного owner change архив содержит полный source set, а не diff;
- после download агент даёт готовый WSL block: unpack → compile → `scp -O`;
- отдельный camera block: reload/run/test;
- агент сам выбирает compile/link flags;
- пользователь не создаёт большие `.c` через heredoc и не угадывает пути/имена;
- SHA/checksums не добавляются без отдельной необходимости;
- архив не создаётся/не отправляется просто ради локального изменения документации.

Это первый явно сформулированный **artifact delivery API** между агентом и оператором.

Исторический precursor в CHAT-007: пользователь отказывается вручную редактировать source и требует один готовый копируемый блок, который сам переписывает исходник, собирает и передаёт test binary.

### I-016 — Self-guarding critical actions
Статус: `OBSERVED`.

После повторного запуска media owner был введён атомарный start lock и process guard.

Обобщаемый принцип:
- destructive/stateful operations должны быть безопасны при случайном повторе;
- duplicate paste/retry не должен создавать второй owner или повторно применять опасную операцию;
- guard должен быть частью готового test workflow, а не инструкцией «не забудь проверить».

Это развитие safety-правила one-owner-per-boot из `CHAT-001`.

### I-017 — Hot-plugin loop вместо reboot для каждого runtime stage
Статус: `CONSOLIDATED`.

Проблема: замена самого owner требовала clean reboot из-за lifetime Fullhan ISP/media state.

Улучшение: оставить один стабильный owner и проверять новые source-derived ISP writers через reloadable hot-plugin.

Цикл стал:
`reverse locally → build .so → scp -O → reload through existing owner → read register/log/capture`.

Результат: несколько последовательных CB970 stages были проверены без reboot между каждым изменением. Это существенно ускорило reverse/runtime convergence.

Исторически `CHAT-003` показывает сам переход к persistent owner + reloadable `.so`; `CHAT-002` уже использует этот механизм как стандартный test loop.

### I-018 — Workspace hygiene и один authoritative reverse artifact
Статус: `CONSOLIDATED`.

В `CHAT-002` выяснилось, что старый `apollo.unpacked` был неполным и не содержал позднюю data/GOT область, тогда как полный textual ARM dump содержал нужный материал.

После этого workflow изменён:
- повреждённый/устаревший artifact исключается из active workspace;
- archive после успешной extraction удаляется;
- дублирующие extraction directories удаляются;
- один полный artifact объявляется authoritative;
- handoff/reverse notes переписываются на него;
- newest source tree не заменяется старым baseline при cleanup.

Это важный переход от «накопить всё» к **curated workspace with explicit authority**.

`CHAT-004` добавляет вторую сторону того же принципа: если агент снова и снова просит вырезать диапазоны из одного бинарника, нужно один раз материализовать полный searchable disassembly/binary bundle, проверить его provenance и дальше искать локально без участия пользователя.

### I-019 — Параллельный reverse с непересекающимися зонами ответственности
Статус: `CONSOLIDATED`.

Вторая половина `CHAT-002` показывает более зрелый multi-agent process:
- основной агент держит canonical runtime и integration;
- parallel reverse-agent получает конкретный тяжёлый блок;
- scope явно исключает функции, которыми занимается основной агент;
- hardware tests выполняются только в интеграционном контуре;
- parallel agent передаёт finished reverse unit: contracts, tables, exact pseudo-C/reference C, unresolved points;
- основной агент интегрирует unit в canonical runtime, а не создаёт ещё одну независимую реализацию.

Так были параллельно восстановлены shared LUT/heavy writers и затем AE/AWB frontend.

Это уже не просто handoff между чатами, а ранняя форма **role-specialized agent pipeline**, хотя пользователь всё ещё вручную маршрутизирует пакеты между агентами.

CHAT-007 показывает более раннюю форму того же pipeline: findings из external-reference ветки передаются второму ISP reverse-agent как технический checkpoint с локальными путями, confirmed/gaps и запретом повторно реверсить закрытое.

`CHAT-008` усиливает процесс: параллельный ISP checkpoint прямо требует сверить findings с текущим состоянием и **не реверсить заново уже подтверждённое**; затем результат второго агента используется как semantic guide, но основной агент сохраняет canonical integration.

### I-020 — Автоматизация boot/recovery development loop
Статус: `OBSERVED`.

Проблема: каждый Linux reboot требовал снова вручную заходить в U-Boot и набирать длинную TFTP/RAM boot последовательность.

Решение в `CHAT-003`:
- проверенная последовательность вынесена в сохранённую U-Boot команду `openipc_boot`;
- штатная загрузка получила симметричную `stock_boot`;
- после проверки `bootcmd` переведён на OpenIPC/TFTP default;
- операторский reboot перестал требовать повторного ручного ввода boot sequence.

Результат: hardware recovery/owner replacement стал значительно дешевле по времени и вниманию.

Обобщаемый принцип: если один и тот же recovery/boot ritual повторяется между экспериментами, превратить его в проверенную именованную операцию, сохранив простой rollback path.

### I-021 — Deterministic state capture для физических A/B тестов
Статус: `OBSERVED`.

Проблема: timed test «открыто → через несколько секунд закрыть → снова открыть» оказался трудно воспроизводимым; marker lines в owner log могли быть затёрты из-за file-offset поведения.

Улучшение:
- каждое физическое состояние снимается отдельной командой;
- результат каждого состояния сохраняется в отдельный файл;
- сравнение выполняется после завершения всех фаз.

Это уменьшает роль памяти/тайминга пользователя и делает аппаратное наблюдение пригодным как evidence.

### I-022 — Known-good baseline + regression isolation
Статус: `OBSERVED`.

В `CHAT-003` серия v4.0.x регрессий была локализована сравнением с ранее доказанным v3.8, а не продолжением случайных правок. Возврат критического pre-start stage восстановил нормальный H.264 baseline.

Практика:
- сохранять последнюю hardware-proven baseline;
- при регрессии сравнивать lifecycle/order и минимальный diff относительно неё;
- новые hypotheses вводить поверх baseline по одной связной причине;
- не считать новый version number прогрессом без сохранения доказанных свойств.

Это особенно важно в reverse engineering, где несколько одновременно изменённых неизвестных быстро делают результат неинтерпретируемым.

### I-023 — Preflight перед hardware delivery
Статус: `OBSERVED`.

После build-time и ABI ошибок стало ясно, что hardware loop должен начинаться только после дешёвых проверок, доступных агенту:
- compile с реальными flags;
- проверка archive completeness;
- self-test/reference vectors для восстановленной математики;
- анализ подозрительных warnings.

Это переносит дешёвые ошибки с аппаратного контура обратно в агентный/локальный контур.

### I-024 — Full searchable evidence вместо серии ручных extract-команд
Статус: `OBSERVED`.

Проблема: reverse одного и того же `apollo.unpacked` шёл серией `objdump/grep` запросов, каждый из которых требовал очередного действия пользователя.

Улучшение в `CHAT-004`:
- один раз сформирован полный ARM disassembly + strings + ELF metadata + сам unpacked binary;
- бинарник и disassembly проверены между собой;
- после этого агент сам ищет xrefs, literal pools и call chains;
- оператор возвращается только для hardware-only действий.

Это резко уменьшает количество terminal round-trips и является ранним прообразом будущего shared reverse workspace/MCP.

В CHAT-007 эта практика дополнительно развивается до полной локальной копии stock userspace и поиска по ней вместо постоянных target-команд.

### I-025 — Capture once, analyze offline
Статус: `OBSERVED`.

Когда stock/OpenIPC target находится в ценном живом состоянии, выгоднее один раз снять достаточный read-only evidence bundle и затем анализировать его локально, чем постоянно возвращаться на камеру за очередным адресом.

В `CHAT-004` эта практика выросла от отдельных MMIO/VMM файлов до:
- stock ioctl trace;
- ISP parameter/context;
- MMIO/MIPI snapshots;
- VMM/process mappings;
- interrupts/clock;
- проверенного Apollo code/data bundle.

Практический эффект:
- меньше риска сломать stateful media pipeline;
- меньше reboot;
- меньше ручных вопросов к пользователю;
- hypotheses можно проверять офлайн и возвращаться на железо только с осмысленным A/B.

CHAT-007 усиливает принцип: valuable stock runtime сначала снимается/копируется, после чего Apollo/ioctl/startup анализ переносится в WSL и camera state не дёргается без необходимости.

### I-026 — Использовать стандартный специализированный инструмент вместо временного велосипеда
Статус: `OBSERVED`.

Пользователь явно разрешил устанавливать необходимые dev-tools в WSL и потребовал не заменять нормальный `ffprobe/ffmpeg` самописным parser-ом без причины.

Устойчивый паттерн:
- если стандартный инструмент предметной области доступен или легко устанавливается, использовать его;
- свои parser/helper писать только для отсутствующего vendor-specific контракта;
- сразу сохранять визуально проверяемые результаты в Windows filesystem, когда оператор может валидировать их через VLC/другой GUI.

### I-027 — Явные terminal lanes
Статус: `OBSERVED`.

В multi-terminal hardware workflow закрепилось удобное разделение:
- `WSL` — source/build/analysis/scp;
- `UART camera` — непосредственные target commands;
- `U-Boot` — boot/recovery environment.

Команды должны приходить уже для нужного lane. Это снимает с пользователя обязанность преобразовывать SSH one-liner в UART-команды и уменьшает ошибки вставки.

### I-028 — Полный flash dump до любой мутации
Статус: `OBSERVED`.

`CHAT-005` показывает исходный recovery/provenance принцип проекта:
- до записи NOR снять полный raw dump;
- использовать dump как источник фактической MTD/env/config информации;
- только после этого переходить к экспериментам;
- первый OpenIPC milestone строить без записи flash.

Практический результат оказался шире recovery: один dump дал точную U-Boot environment, boot layout, dual-lens/config GPIO сведения и позволил убрать догадки вокруг bootloader access.

Это отличается от обычного checkpoint перед экспериментом: full flash dump — базовый immutable аппаратный anchor проекта.

### I-029 — Проверять тот канал, который реально нужен следующему шагу
Статус: `OBSERVED`.

В раннем network bring-up агент начал чинить ICMP reachability Windows-хоста, хотя следующим реальным действием был TFTP. Пользователь остановил лишнюю ветку и потребовал «без ICMP».

После прямого `tftpboot` маленького файла путь `PC → Ethernet → U-Boot → RAM` был доказан сразу.

Обобщение:
- functional-path test важнее ancillary health-check;
- если TFTP нужен для RAM boot, проверяй TFTP;
- не превращай необязательную диагностику в новый blocker.

### I-030 — Аппаратный оператор может отдавать сырые данные, агент должен делать разбор
Статус: `CONSOLIDATED`.

Уже в самом начале проекта закрепился полезный split:
- пользователь обеспечивает физический доступ, UART, programmer, bootloader и аппаратные действия;
- агент разбирает raw boot logs, dumps, environment и строит следующий инженерный шаг.

Это ранняя форма будущего принципа «не заставлять пользователя предварительно анализировать/чистить evidence». `CHAT-006` подтверждает следующий шаг: после замечания пользователя агент самостоятельно извлёк rootfs/`/app`/модули из уже переданного SPI dump и продолжил ABI reverse без новых ручных копирований. В дальнейшем эта практика развивается в archive/workspace, full searchable artifacts и MCP.

### I-031 — Сначала переиспользовать уже загруженный artifact
Статус: `OBSERVED`.

Проблема: hardware operator может быть по привычке втянут в поиск/копирование файлов, хотя нужные данные уже находятся в полном firmware dump или другом переданном artifact.

В `CHAT-006` после прямого замечания пользователя workflow изменился:
- full SPI dump стал primary extraction source;
- из него агент самостоятельно получил kernel/initramfs, `/app`, media modules и sensor libraries;
- target больше не требовался для банального файлового inventory;
- дальнейший reverse шёл уже по извлечённому corpus.

Обобщение: **target нужен для нового evidence, а не для повторного извлечения уже имеющихся bytes**.

### I-032 — Искать внешние reference implementations до продолжения дорогого blind reverse
Статус: `CONSOLIDATED`.

В `CHAT-007` после длинного локального ISP reverse пользователь отдельно просит искать существующие наработки. Поиск находит несколько источников, но особенно полезен clean-room reverse соседнего Fullhan FH8852V201.

Новый workflow:
- сначала формулировать конкретный missing contract;
- искать vendor SDK/samples и reverse соседних SoC по уникальным API names;
- использовать найденное как semantic/API architecture map;
- затем проверять каждый ABI/address/register на FH8626 собственным evidence.

Эффект: вместо абстрактного поиска причины `ISP IRQ=0` появляется конкретная гипотеза missing chain `MemInit → SensorRegCb → SensorInit → SetSensorFmt → Init → Kick`.

Критическое ограничение: cross-SoC reference не является доказательством FH8626 ABI/MMIO.

`CHAT-008` показывает зрелое применение этого правила: FH8852 reference используется для имён `SensorRegCb/SensorInit/SensorKick/KickStart` и ожидаемой архитектуры, а каждый адрес/handler затем подтверждается независимым FH8626 disassembly/runtime evidence.

### I-033 — Разделять рабочую reverse-документацию, историю экспериментов и upstream deliverable
Статус: `OBSERVED`.

В `CHAT-007` впервые явно проектируется будущий knowledge repository:
- current hardware/boot/media/ioctl facts — в структурированных docs;
- known-good bring-up — отдельным reproducible recipe;
- failed hypotheses и archaeology — в experiment/history layer;
- provenance binary/source — отдельно;
- upstream OpenIPC получает только нужный source/config/device delta и hardware evidence, а не весь reverse-дневник.

Это исторический предшественник современной схемы reverse repo: одна текущая authority на факт, история отдельно, тяжёлое evidence отдельно.

### I-034 — Progress tracking по engineering boundaries
Статус: `CONSOLIDATED`.

Повторные вопросы пользователя «сколько ещё» в `CHAT-007` показывают необходимость не временной оценки, а карты оставшейся работы.

Устойчивый формат:
- закрытые subsystem gates;
- текущий blocker;
- следующий hardware-visible milestone;
- 1–3 оставшихся крупных reverse/integration блока.

Так длинный reverse остаётся управляемым без ложных обещаний по времени.

### I-035 — Adaptive granularity: группировать routine, дробить только decision boundaries
Статус: `CONSOLIDATED`.

В `CHAT-008` пользователь сформулировал правило почти в готовом виде:
- если следующий технический вывод зависит от результата команды — агент даёт небольшой шаг, получает вывод и **сам** принимает решение;
- если путь уже неоднократно проходили и ветвления нет — знакомый bring-up/build/transfer нужно дать одним цельным копируемым блоком.

Это объединяет две прежние жалобы, которые по отдельности выглядят противоречиво: «ПОШАГОВО» и «зачем дробить знакомую процедуру». Они на самом деле задают одну модель — granularity определяется **engineering decision boundary**, а не фиксированным числом команд.

### I-036 — Runtime interception вместо reverse отсутствующего helper source
Статус: `OBSERVED`.

В `CHAT-008` исходник старого H.264 helper'а не сохранился. Вместо нового глубокого reverse его бинарника агент собрал маленький `LD_PRELOAD`-interposer для `ioctl()`, запустил existing helper и получил реальные request/payload до и после вызовов.

Эффект:
- быстро восстановлены фактические VPU/PAE structures и порядок вызовов;
- закрыта ложная гипотеза о неправильных 1280×720 размерах;
- live contract получен напрямую из исполняемой программы без реконструкции всего её source.

Обобщение: если нужен **runtime ABI одного boundary**, сначала предпочесть trace/interposition/ptrace/strace-like capture полному reverse вызывающей программы, если такой capture безопасен.

### I-037 — SoC-first backend вместо одноразового board port
Статус: `OBSERVED`.

К концу `CHAT-008` цель расширилась от «получить картинку на одной AJL33PQ0866» до reusable FH8626 compatibility layer:
- sensor/MIPI;
- ISP/VPU/ENC;
- audio;
- GPIO/PTZ;
- несколько stream channels;
- board-specific настройки вынесены наружу.

Причина: тяжёлый clean-room reverse vendor SDK/ABI делается один раз на SoC. После этого следующая камера на FH8626 должна требовать в основном sensor/GPIO/resolution/fps profile, а не повторного reverse `ISP → VPU → PAE`.

Это важный reusable architecture principle для других unsupported camera SoC: разделять **SoC backend** и **board profile** как можно раньше, но не расширять scope раньше, чем доказан основной media path.


## Исходные этапы, ещё не подтверждённые

`CHAT-008` повторно подтверждает необходимость такого tracker: пользователь несколько раз спрашивает остаток работы, а прежняя оценка «2–3 узких неизвестных» оказывается слишком оптимистичной.

### I-001 — Ручные браузерные вставки → workspace и архивы
Статус: `BOOTSTRAP`.

`CHAT-001` показывает уже локальный WSL workspace/checkpoints, но сам переход из ещё более ранней схемы пока не восстановлен.

### I-002 — Workspace → Google Drive
Статус: `BOOTSTRAP`.

В `CHAT-001` не наблюдается.

### I-003 — Google Drive → GitHub authority
Статус: `BOOTSTRAP`.

В `CHAT-001` проект ещё не организован по современной Git-authority модели.

### I-005 — Env-переменные как устойчивый способ передачи сложных значений
Статус: `BOOTSTRAP`.

Есть активное использование shell variables, но недостаточно данных, чтобы считать именно заявленный позже transition доказанным.
