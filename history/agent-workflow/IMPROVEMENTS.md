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
Статус: `CONSOLIDATED`.

В `CHAT-001` перед reboot начали собирать checkpoint с helper-бинарниками и диагностическим состоянием, передавать его одним пакетом, а затем продолжать работу с распакованным содержимым.

Полезный эффект: recovery после reboot перестал означать повторное создание каждого helper-а.

Ограничение: checkpoint должен иметь явную структуру и не становиться единственным source of truth.

`CHAT-008` показывает практическую ценность checkpoint как active recovery substrate: после очистки `/tmp` старые sensor/ISP/H.264 helper'ы были найдены в `checkpoints/openipc-20260825` и переиспользованы вместо повторной сборки с нуля.

`CHAT-009` подтверждает checkpoint как recovery substrate ещё раз: после reboot/очистки `/tmp` рабочие H.264/ISP helpers и их source находились через `~/FH8626/checkpoints/...` и восстанавливались без повторного reverse.

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

`CHAT-009` доводит требование handoff до воспроизводимости: при исчерпании контекста пользователь требует не только narrative state, но и полный индекс disassembly/memory/stock evidence, файлов/директорий и exact build/transfer/reverse/run recipes, чтобы следующий агент мог продолжить без скрытого знания.

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

`CHAT-009` даёт аппаратное доказательство причины: `rmmod isp` оставил dangling IRQ action и привёл к kernel Oops, а несколько ISP fd давали duplicate handlers. Поэтому persistent owner/daemon — не только ускорение, но и safety requirement.

`CHAT-011` даёт ещё одно прямое основание для single-owner architecture: после H.264 capture пользователь спрашивает, можно ли убрать обязательный power-cycle, и решение формулируется как один daemon, который держит `/dev/isp`, `/dev/pae`, `/dev/media_process` и VMM весь lifetime.

`CHAT-014` уточняет границу hot replacement: `kill -9` старого owner не гарантирует чистую повторную инициализацию vendor sensor/ISP state на том же boot. Без явного lifecycle-safe shutdown/supervisor clean reboot остаётся обязательным fallback.

### I-013 — Оптимизация dev-loop SSH, а не только самой прошивки
Статус: `CONSOLIDATED`.

В ходе `CHAT-001` выяснилось, что долгий SSH после RAM boot был отдельным bottleneck разработки. Были устранены ненормальная задержка RNG/host-key lifecycle и PTY-проблема; после этого оставшуюся криптографическую задержку признали характеристикой слабого CPU, а не продолжили бесконечную оптимизацию. Для повторных операций предложено переиспользование SSH-соединения.

Обобщаемый урок: ускорять нужно и **контур эксперимента** — boot, network, auth, file transfer, recovery — потому что десятки итераций делают эти секунды/минуты частью стоимости reverse.

Источник: `CHAT-001`.

`CHAT-009` подробно раскладывает dev-loop optimization по слоям: постоянный Dropbear host key на mtd4, статический network path без сломанного `fw_printenv`, persistent `seedrng` state для ранней CRNG readiness, корректный `devpts newinstance`/`ptmx`, а также отдельные замеры IP → TCP/22 → SSH banner. Это превращает «SSH долго» из гадания в boundary-by-boundary diagnosis.

`CHAT-011` показывает полную dev-loop reconciliation: boot-time SSH ускорился, но затем пользователь обнаружил 5-секундную regression каждого SCP; оптимизация была пересмотрена по measured boundaries, а не принята как завершённая.

### I-014 — Разделять «работает» и «полностью доказано»
Статус: `CONSOLIDATED`.

В конце чата handoff специально сохранял provenance gaps и различал:
- рабочий hardware bring-up;
- валидный H.264;
- отсутствие доказательства корректного moving image/RAW для части параметров;
- неполностью доказанные sensor/lens соответствия.

Обобщаемое правило: не повышать уровень evidence молча. «Устройство отвечает» ≠ «production contract доказан».

Источник: `CHAT-001`.

`CHAT-009` даёт чистый пример evidence boundary: H.264 Annex-B с SPS/PPS/IDR и активными ISP/PAE IRQ доказал hardware encoder path, но серое изображение и один и тот же descriptor означали, что moving-image dequeue ещё не доказан.

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

`CHAT-011` подчёркивает различие между именем image и его фактической lineage: серии FINAL/FINAL2/FINAL3/FINAL4 недостаточно; master handoff отдельно предупреждает проверять, какой `rootfs.cpio` реально упакован в загруженный uInitrd.

`CHAT-013` усиливает curated-workspace contract: старый неполный `apollo.unpacked` исключается, один `reference/apollo_unpacked_ARM_full.txt` объявляется authoritative, current source checkpoint отделяется от historical checkpoint, а поздние reverse-notes не выдаются за уже интегрированный код.

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
Статус: `CONSOLIDATED`.

В `CHAT-003` серия v4.0.x регрессий была локализована сравнением с ранее доказанным v3.8, а не продолжением случайных правок. Возврат критического pre-start stage восстановил нормальный H.264 baseline.

Практика:
- сохранять последнюю hardware-proven baseline;
- при регрессии сравнивать lifecycle/order и минимальный diff относительно неё;
- новые hypotheses вводить поверх baseline по одной связной причине;
- не считать новый version number прогрессом без сохранения доказанных свойств.

Это особенно важно в reverse engineering, где несколько одновременно изменённых неизвестных быстро делают результат неинтерпретируемым.

`CHAT-010` даёт второй классический regression case: после SSH/rootfs правок новый `scp` стал стабильно медленнее, а пользователь помнил быстрый pre-change baseline. После отказа от workaround-гипотезы агент вернулся к boundary timing и поиску фактического delta.

`CHAT-011` — после cold boot прежний sensor/media path «сломался», но сравнение с known-good session быстро выявило пропущенный GPIO5 reset pulse. Это ещё один пример восстановления через known-good delta, а не новый blind reverse.

### I-023 — Preflight перед hardware delivery
Статус: `OBSERVED`.

После build-time и ABI ошибок стало ясно, что hardware loop должен начинаться только после дешёвых проверок, доступных агенту:
- compile с реальными flags;
- проверка archive completeness;
- self-test/reference vectors для восстановленной математики;
- анализ подозрительных warnings.

Это переносит дешёвые ошибки с аппаратного контура обратно в агентный/локальный контур.

`CHAT-014` показывает полезный preflight нового owner: patcher сначала проверяется локально, затем build должен явно дать ARM EABI5/musl binary до любой camera iteration.

### I-024 — Full searchable evidence вместо серии ручных extract-команд
Статус: `CONSOLIDATED`.

Проблема: reverse одного и того же `apollo.unpacked` шёл серией `objdump/grep` запросов, каждый из которых требовал очередного действия пользователя.

Улучшение в `CHAT-004`:
- один раз сформирован полный ARM disassembly + strings + ELF metadata + сам unpacked binary;
- бинарник и disassembly проверены между собой;
- после этого агент сам ищет xrefs, literal pools и call chains;
- оператор возвращается только для hardware-only действий.

Это резко уменьшает количество terminal round-trips и является ранним прообразом будущего shared reverse workspace/MCP.

В CHAT-007 эта практика дополнительно развивается до полной локальной копии stock userspace и поиска по ней вместо постоянных target-команд.

`CHAT-010` подтверждает self-service модель на `enc.ko`/`media_process.ko`: вместо дальнейшей серии ручных диапазонов пользователь передаёт full disassembly + symbols + sections, после чего агент самостоятельно закрывает `4D05/4D06`, callback mapping и read-index flow.

`CHAT-013` распространяет self-service evidence на sensor variants: JXF37/JXF37P libraries, tuning blobs, readelf/strings/full disassembly собираются одним reverse bundle, после чего agent должен анализировать их локально вместо серии target-side ranges.

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
Статус: `CONSOLIDATED`.

В multi-terminal hardware workflow закрепилось удобное разделение:
- `WSL` — source/build/analysis/scp;
- `UART camera` — непосредственные target commands;
- `U-Boot` — boot/recovery environment.

Команды должны приходить уже для нужного lane. Это снимает с пользователя обязанность преобразовывать SSH one-liner в UART-команды и уменьшает ошибки вставки.

`CHAT-009` показывает практическую цену смешения lane и сложного interactive shell: UART paste повреждал regex/quotes/addresses, PlatformIO monitor и shell prompt смешивались; после этого сложную логику логичнее переносить в script/helper, а UART оставлять плоским.

`CHAT-010` ещё раз формализует terminal lanes: после проверки PTY пользователь просит все camera-команды давать именно для serial UART, а WSL держать отдельно.

`CHAT-014` повторно закрепляет удачный UX: capture bundle сразу кладётся в Windows-visible каталог и открывается через `explorer.exe`, чтобы пользователь не искал файлы между WSL и Windows.

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
Статус: `CONSOLIDATED`.

В `CHAT-007` впервые явно проектируется будущий knowledge repository:
- current hardware/boot/media/ioctl facts — в структурированных docs;
- known-good bring-up — отдельным reproducible recipe;
- failed hypotheses и archaeology — в experiment/history layer;
- provenance binary/source — отдельно;
- upstream OpenIPC получает только нужный source/config/device delta и hardware evidence, а не весь reverse-дневник.

Это исторический предшественник современной схемы reverse repo: одна текущая authority на факт, история отдельно, тяжёлое evidence отдельно.

`CHAT-011` независимо подтверждает границу current-dev vs upstream: пользователь требует не забыть вернуть временные network/init hacks, а handoff получает отдельный reconciliation список dev-only изменений и финальных production решений.

### I-034 — Progress tracking по engineering boundaries
Статус: `CONSOLIDATED`.

Повторные вопросы пользователя «сколько ещё» в `CHAT-007` показывают необходимость не временной оценки, а карты оставшейся работы.

Устойчивый формат:
- закрытые subsystem gates;
- текущий blocker;
- следующий hardware-visible milestone;
- 1–3 оставшихся крупных reverse/integration блока.

Так длинный reverse остаётся управляемым без ложных обещаний по времени.

`CHAT-008` повторно подтверждает необходимость tracker: пользователь несколько раз спрашивал остаток работы, а прежняя оценка «2–3 узких неизвестных» оказалась слишком оптимистичной.

`CHAT-011` — многократные вопросы «сколько ещё осталось» и «скоро закончится вечный reverse?» повторно показывают, что прогресс нужно отражать subsystem gates, особенно перед длинными ISP call-chain исследованиями.

`CHAT-013` — пользователь просит общий прогресс по направлениям простым языком и repeatedly спрашивает остаток; module-level completion map оказывается полезнее потока адресов и локальных функций.

`CHAT-014` показывает, что progress percentage без requirement checklist мало помогает: лучше сообщать, какие обязательные evidence gates уже закрыты и какой конкретный gate блокирует слово COMPLETE.

### I-035 — Adaptive granularity: группировать routine, дробить только decision boundaries
Статус: `CONSOLIDATED`.

В `CHAT-008` пользователь сформулировал правило почти в готовом виде:
- если следующий технический вывод зависит от результата команды — агент даёт небольшой шаг, получает вывод и **сам** принимает решение;
- если путь уже неоднократно проходили и ветвления нет — знакомый bring-up/build/transfer нужно дать одним цельным копируемым блоком.

Это объединяет две прежние жалобы, которые по отдельности выглядят противоречиво: «ПОШАГОВО» и «зачем дробить знакомую процедуру». Они на самом деле задают одну модель — granularity определяется **engineering decision boundary**, а не фиксированным числом команд.

`CHAT-010` начинается со строгого пошагового reverse по новому unknown boundary и при этом сохраняет правило не требовать лишних действий: granularity остаётся привязанной к decision point, а не к одному фиксированному стилю.

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


### I-038 — Handoff как воспроизводимый manifest, а не только narrative
Статус: `CONSOLIDATED`.

В конце `CHAT-009` контекст диалога исчерпывается и создаётся большой handoff. Пользователь сразу задаёт более строгий acceptance criterion: новый агент должен суметь повторить работу **без автора исходного чата**.

Минимальное содержимое такого handoff:
- current proven state и unresolved blockers;
- canonical working binaries/sources/checkpoints;
- stock firmware corpus и его provenance;
- disassembly/reverse artifacts;
- memory/MMIO/stock runtime captures;
- точные пути и роль каждой директории;
- build/toolchain commands;
- transfer commands и обязательные transport flags;
- boot/recovery/run recipes;
- known-bad/crash experiments;
- команды, которыми заново получить ключевое evidence, если artifact потерян.

Практический вывод: handoff следует проверять вопросом **«сможет ли новый агент воспроизвести milestone только по этому документу и доступным artifacts?»**. Если ответ зависит от неявной памяти старого агента, handoff неполон.

Это дополняет I-010: canonical handoff должен быть компактной картой, но иметь ссылки/manifest на воспроизводимые подробности, а не превращаться в монолитный transcript.

`CHAT-011` развивает master handoff через reconciliation: старые observations не удаляются молча, а сохраняются как historical evidence с пометкой superseded, при этом верхний authoritative state переписывается под текущий proven runtime.

`CHAT-013` снова достигает context limit; пользователь требует передать не только handoff, но и полный workspace/archive плюс prompt следующему агенту. Это подтверждает, что continuity должна переносить и canonical artifacts, и explicit next-agent contract.

### I-039 — Прошлый агент как fallback, а не основной канал continuity
Статус: `OBSERVED`.

В начале `CHAT-010` новый агент получает воспроизводимый handoff и работает по нему самостоятельно. Пользователь отдельно предлагает при необходимости спросить прошлый чат/агента, но просит не создавать лишних действий и уточнений.

Устойчивый порядок:
1. сначала использовать текущий handoff, checkpoint и загруженные artifacts;
2. самостоятельно закрывать пробелы статическим/reverse/runtime evidence;
3. к предыдущему агенту обращаться только за уникальным фактом или provenance, который отсутствует в сохранённом corpus;
4. ответ прошлого агента после получения переносить в canonical state, чтобы не зависеть от повторного ручного посредничества.

Это важный переход от «пользователь постоянно синхронизирует два чата» к handoff-first continuity.

### I-040 — Свежая regression-observation выше удобного объяснения из handoff
Статус: `OBSERVED`.

`CHAT-010` показывает, что handoff и прежние выводы нельзя превращать в догму. В SSH-ветке handoff содержал измеренный 5-секундный delay, и агент сначала нормализовал его как стоимость KEX и предложил `ControlMaster`. Пользователь указал более сильный факт: **до SSH-правок обычный SCP был быстрым**.

После этого правильный процесс стал:
- признать текущую деградацию регрессией;
- не маскировать её multiplexing/workaround-ом;
- разложить соединение на TCP connect → banner → KEX → auth;
- сравнить с known-good behavior и локализовать появившийся delta.

Обобщение: handoff — стартовое инженерное состояние, но свежая воспроизводимая observation имеет приоритет над старой интерпретацией.

### I-041 — Ledger временных workaround-ов и отдельный final-reconciliation gate
Статус: `OBSERVED`.

В bring-up неизбежны временные решения: board-неспецифичный overlay, ручная mutation generated rootfs, dev-only service ordering, compatibility runtime, hard-coded test assumptions или удаление блокирующего универсального hook.

`CHAT-011` показывает, что просто помнить о них недостаточно. К концу длинной сессии часть working image уже опережала canonical source, а пользователь отдельно потребовал сохранить обязательство «всё вернуть обратно» при финальном оформлении.

Практика:
- при каждой временной правке сразу писать **зачем она введена**, **где живёт**, **что считается clean final replacement**;
- держать единый debt/reconciliation section в handoff/state;
- различать `PROVEN_DEV_WORKAROUND` и `FINAL_INTENDED_ARCHITECTURE`;
- перед upstream/production выполнить отдельный проход: каждую запись либо revert, либо board-scope, либо перенести в canonical source нормальным способом;
- не считать generated CPIO/image source of truth, даже если именно он hardware-proven;
- generic OpenIPC behavior не менять permanently только потому, что это ускоряет одну development camera.

Результат: hardware-proven экспериментальные решения не теряются, но и не превращаются незаметно в архитектурный долг финального порта.

Источник: `CHAT-011`.

### I-042 — Параллельная productionization только за стабильной subsystem boundary
Статус: `OBSERVED`.

В уникальном хвосте `CHAT-012` пользователь спрашивает, не пора ли уже переходить от probe-ов к firmware/Majestic, хотя RAW/Bayer path ещё не закрыт.

Полезное решение оказалось не бинарным «сначала весь reverse» / «сразу Majestic», а разделением слоёв:
- уже доказанные boot/rootfs/vendor-module/GC1054/VPU/PAE/dequeue части можно начинать productionize;
- unresolved RAW/ISP path продолжает жить в минимальном диагностическом owner;
- один долгоживущий `fh8626_daemon` остаётся владельцем stateful Fullhan fd и ISP runtime;
- Majestic подключается **после** этой границы как consumer stream interface, а не открывает `/dev/isp` параллельно;
- новые hardware эксперименты не должны требовать изменения streamer/RTSP слоя.

Практический критерий начала productionization:
1. subsystem boundary уже доказана и имеет стабильный contract;
2. оставшийся blocker локализован по другую сторону этой boundary;
3. integration не усложняет диагностику unresolved hardware path.

Это позволяет параллельно готовить rootfs/startup/packages/configuration, не превращая Majestic в дополнительную переменную низкоуровневого reverse.

Источник: уникальный хвост `CHAT-012` после общего префикса с `CHAT-004`.

### I-043 — Автономный reverse loop с milestone-based reporting
Статус: `OBSERVED`.

В `CHAT-013` пользователь почти дословно формулирует желаемый режим длинной reverse-задачи:
- продолжать до полного или существенного частичного результата;
- после каждого внутреннего вывода перепроверять его по коду/disassembly/data;
- при опровержении самостоятельно менять направление;
- вести внутреннюю карту адресов, функций, структур и зависимостей;
- не публиковать каждую промежуточную находку;
- возвращаться к оператору только при реальном blocker, отсутствующем hardware evidence или завершённом milestone;
- финальный user-facing update держать кратким: сделано / подтверждено / неизвестно.

Это не означает background execution между сообщениями. Речь о поведении **внутри текущего рабочего прохода**: максимально использовать доступный artifact/tooling до следующего operator-only boundary.

Практический эффект:
- меньше «продолжай»/«ну что там?» циклов;
- reverse не дробится по каждой функции;
- промежуточные ложные гипотезы успевают быть опровергнуты до user-facing отчёта;
- аппаратный оператор получает уже сформулированный тест, а не поток reasoning.

Источник: `CHAT-013`.

### I-044 — Разделять firmware capability и hardware identity
Статус: `OBSERVED`.

Vendor image часто содержит drivers/configs для нескольких board revisions. Поэтому наличие sensor library, tuning blob или format table доказывает только **поддержку варианта прошивкой**.

`CHAT-013` даёт чистый пример:
- JXF37/JXF37P drivers и 1080p contracts действительно существуют;
- их ABI удалось реверсить и probe частично запускается;
- но stock runtime конкретной AJL33PQ0866 выбирает GC1054;
- физический JXF37 I²C target не отвечает;
- stock wide Full-HD получается из GC1054 1280×720 через scaler;
- реальный zoom переключает target/GPIO без смены sensor driver.

Устойчивое evidence ladder для hardware identity:
1. firmware support — `SUPPORTED_VARIANT`;
2. board config/hw_info — candidate;
3. stock runtime selection + bus response — strong evidence;
4. hardware behavior/visual path — acceptance.

Опровергнутую sensor-ветку не удалять: её reverse может быть полезен для другой аппаратной ревизии, но P0 текущей платы должен быть снят.

### I-045 — Shadow/diagnostic mode перед включением восстановленного closed-loop control
Статус: `OBSERVED`.

При восстановлении AWB/AE и других динамических ISP control loops опасно сразу писать рассчитанные значения в живой pipeline. В `CHAT-013` для `CA4F4` сначала создаётся read-only `awbmode1 diag`, затем `shadow on|off|status`: алгоритм читает настоящие 9 statistics records, вычисляет stock-like gains и temporal/hysteresis state, но не меняет image registers.

Только после:
- правильного mapping статистики;
- deterministic/reference tests;
- сравнения вычисленных значений на live hardware;
- стабильности нескольких последовательных итераций

можно переходить к отдельному controlled commit в AWB registers.

Обобщение: для recovered closed-loop algorithm безопасная последовательность —
**offline self-test → live shadow compute → compare/log → gated commit**.

Это снижает риск испортить stateful ISP и отделяет correctness вычислений от side effects.

### I-046 — Completion audit по исходному acceptance checklist
Статус: `OBSERVED`.

Перед тем как объявить длинную reverse/integration задачу законченной, агент должен вернуться к **исходному заданию**, а не к собственной сокращённой модели задачи.

В `CHAT-014` это изменило качество работы заметно: static AWB→CCM reverse уже дал правильный механизм, но пользователь потребовал закрыть задачу «полностью». После checklist-а стало ясно, что нужен runtime before/after evidence. Только затем были созданы coherent `v4.2.1` path, hardware validation и rollback.

Рекомендуемый completion gate:
1. перечислить original deliverables;
2. для каждого отметить `DONE / PARTIAL / BLOCKED`;
3. указать evidence class;
4. отдельно перечислить unresolved, которые **не** блокируют исходную цель;
5. слово `COMPLETE` использовать только если все обязательные пункты закрыты или пользователь явно сузил scope.

### I-047 — Machine-readable operational session state
Статус: `OBSERVED`.

Quality-retrospective в конце `CHAT-014` впервые явно предлагает вынести постоянно теряемые operational facts из памяти диалога в единый state artifact, желательно одновременно human-readable и machine-readable.

Минимум:
- camera IP и active boot mode;
- WSL project/checkpoint root;
- Windows capture root;
- current owner source/binary/version;
- compiler absolute path;
- stockapp device/mount;
- transport conventions;
- current hardware/reverse stage;
- last hardware-validated artifact;
- unsafe workflows и required reboot conditions.

Предложенная форма: `ENVIRONMENT_CURRENT.md` + `CURRENT_SESSION.json`.

Это прямой предшественник современной project-authority модели: не заставлять нового агента восстанавливать operational state по переписке.

### I-048 — Hardware experiment как явная state machine
Статус: `OBSERVED`.

В quality-pack `CHAT-014` эксперимент формализуется как последовательность состояний, а не произвольный список команд:

`BOOTED → OWNER_READY → RUNNING → BASELINE_VALIDATED → BASELINE_CAPTURED → CHANGE_APPLIED → READBACK_VALIDATED → AFTER_CAPTURED → ROLLBACK → ROLLBACK_VALIDATED`.

Преимущества:
- нельзя случайно сделать `step` при `running=0`;
- before/after dumps имеют однозначную семантику;
- rollback является частью теста, а не необязательным хвостом;
- следующий блок команд выдаётся только после реальной decision boundary.

### I-049 — Ретроспектива качества агента как отдельный проектный артефакт
Статус: `OBSERVED`.

В конце `CHAT-014` пользователь специально просит создать отдельный материал для агента, который будет улучшать качество работы других агентов. В результате появляется quality-improvement pack с:
- каталогом повторяющихся ошибок;
- interaction protocol;
- command UX patterns;
- handoff requirements;
- tooling plan;
- automation opportunities;
- инструкцией основному оркестратору внедрить правила в основной workflow.

Это важный agentic-переход: **сам процесс разработки становится объектом версионируемого инженерного анализа**, а не только неформальной корректировки в текущем чате.

## Исходные этапы, ещё не подтверждённые

`CHAT-010` является прямым acceptance-тестом этого принципа: новый агент по handoff сразу продолжает с dequeue boundary, не повторяет sensor/ISP/H.264 bring-up и использует указанные checkpoint paths/constraints. Handoff реально переносит инженерное состояние между чатами.

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
