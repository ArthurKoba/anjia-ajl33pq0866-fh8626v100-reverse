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

`CHAT-022` возвращает hot-plugin идею уже как production-candidate: long-lived media owner + reloadable `libfhisp_algo.so`, runtime gates и rollback позволяют менять AE/AWB/IQ без reboot при здоровом media state.

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

`CHAT-015` делает ownership зрелым: main agent остаётся единственным интегратором, specialist lanes получают независимые state/dataflow chains, не делают hardware writes и не сливают код друг друга; return contract включает exact reverse, unresolved и integration note.

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

`CHAT-015` расширяет self-service corpus внешними именованными Fullhan binaries: после локальной подготовки `libispcore/libisp/libadvapi` дальнейший semantic matching идёт по полным disassembly/symbol/relocation artifacts без ручного range extraction.

`CHAT-016` ещё раз показывает правильный режим: если full ARM TXT уже подготовлен, агент анализирует его напрямую и не заставляет пользователя повторять objdump/архивирование.

`CHAT-017` доводит принцип до двухслойного Apollo corpus: существующий authoritative ARM_FULL отвечает за код/control flow, а отдельно снятый full runtime image — за GOT/RW/heap. Повторный disassembly не нужен.

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

`CHAT-021` показывает зрелую форму capture-once: четыре canonical steady states и transitions собираются в один normalized stock evidence corpus; дальнейший reverse работает с delta/targeted regions вместо повторных hardware sessions.

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

`CHAT-016` уточняет transport matrix: stock runtime — TFTP-only, OpenIPC — SCP/SSH; Windows-visible root и `explorer.exe` должны быть частью готового transfer recipe.

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

`CHAT-015` превращает adjacent-SoC reference из разовой подсказки в формальный semantic-oracle workflow: сначала ищется именованный homolog FH8852, затем algorithm fingerprint и структура, после чего вывод возвращается к FH8626 и доказывается target-specific ARM/MMIO evidence.

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

`CHAT-028` расширяет этот принцип до product strategy: FH8626 hardware/media backend должен быть независим от конкретного frontend; Divinus, Majestic или другой consumer подключаются поверх generic stream/control capability boundary.

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

`CHAT-016` показывает зрелую границу автономности: агент сначала исчерпывает ARM/xrefs/source/runtime snapshots, а к пользователю возвращается только с конкретным отсутствующим input (`libgc1054_mipi.so`, затем exact night-state capture).

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
Статус: `CONSOLIDATED`.

Перед тем как объявить длинную reverse/integration задачу законченной, агент должен вернуться к **исходному заданию**, а не к собственной сокращённой модели задачи.

В `CHAT-014` это изменило качество работы заметно: static AWB→CCM reverse уже дал правильный механизм, но пользователь потребовал закрыть задачу «полностью». После checklist-а стало ясно, что нужен runtime before/after evidence. Только затем были созданы coherent `v4.2.1` path, hardware validation и rollback.

Рекомендуемый completion gate:
1. перечислить original deliverables;
2. для каждого отметить `DONE / PARTIAL / BLOCKED`;
3. указать evidence class;
4. отдельно перечислить unresolved, которые **не** блокируют исходную цель;
5. слово `COMPLETE` использовать только если все обязательные пункты закрыты или пользователь явно сузил scope.

`CHAT-016` является вторым прямым доказательством: после первоначального «Task 2 завершён» полный requirement audit нашёл крупные недоделки, после чего работа продолжалась до v5 и полного day/night numerical replay.

`CHAT-027` показывает, что completion audit должен проверять не только исходный P0 checklist, но и смену цели пользователя: сначала «release-relevant reverse», затем «максимально полный stock reverse». Иначе STATIC_COMPLETE по одному scope ошибочно звучит как абсолютное DONE.

### I-047 — Machine-readable operational session state
Статус: `CONSOLIDATED`.

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

`CHAT-020` реализует proposal: `ENVIRONMENT_CURRENT.md`, `CURRENT_SESSION.json` и `CURRENT_BINARIES.md` становятся реальными central-state files и входят в handoff doctor.

### I-048 — Hardware experiment как явная state machine
Статус: `CONSOLIDATED`.

В quality-pack `CHAT-014` эксперимент формализуется как последовательность состояний, а не произвольный список команд:

`BOOTED → OWNER_READY → RUNNING → BASELINE_VALIDATED → BASELINE_CAPTURED → CHANGE_APPLIED → READBACK_VALIDATED → AFTER_CAPTURED → ROLLBACK → ROLLBACK_VALIDATED`.

Преимущества:
- нельзя случайно сделать `step` при `running=0`;
- before/after dumps имеют однозначную семантику;
- rollback является частью теста, а не необязательным хвостом;
- следующий блок команд выдаётся только после реальной decision boundary.

`CHAT-018` даёт прямое runtime-подтверждение этого подхода на lens switch: capture разделён на baseline → immediate-after → settled → restored-wide. Благодаря этому доказаны общий AE context/history, отсутствие profile reload и возврат wide exposure без смешивания состояний.

`CHAT-021` расширяет state-machine на stock acquisition: transitions фиксируются before/after_early/after_settled, а full cycles — ON→settled→OFF→settled, что позволяет анализировать обе стороны одной операцией.

`CHAT-022` показывает staged hardware parity loop на production candidate: baseline → одна gated feature → capture/readback → off/rollback → следующая feature. Gain publication, APC, NR3D, LTM и manual AE проверяются именно по одному.

### I-049 — Ретроспектива качества агента как отдельный проектный артефакт
Статус: `CONSOLIDATED`.

В конце `CHAT-014` пользователь специально просит создать отдельный материал для агента, который будет улучшать качество работы других агентов. В результате появляется quality-improvement pack с:
- каталогом повторяющихся ошибок;
- interaction protocol;
- command UX patterns;
- handoff requirements;
- tooling plan;
- automation opportunities;
- инструкцией основному оркестратору внедрить правила в основной workflow.

Это важный agentic-переход: **сам процесс разработки становится объектом версионируемого инженерного анализа**, а не только неформальной корректировки в текущем чате.

`CHAT-016` создаёт quality-improvement pack v3 с новыми правилами этой ветки: stock=TFTP-only, minimal BusyBox, focused workspace, heavy reverse off-target, liveness protocol и Windows publish helper.

`CHAT-017` объединяет несколько независимых agent-retrospective packs в один master quality/reverse-preparation пакет вместо размножения несовместимых правил.

### I-050 — Persistent specialist lanes с последовательными Task N
Статус: `CONSOLIDATED`.

После первых коротких parallel задач `CHAT-015` меняет orchestration model:
- specialist остаётся в той же вкладке и сохраняет локально набранный контекст;
- следующая крупная область оформляется как `Task 2`, затем `Task 3`;
- task охватывает целый control loop/subsystem, а не одну функцию;
- внутри task есть 5–10 обязательных подзадач, false-hypothesis audit и final acceptance;
- основной агент интегрирует только итоговый finished unit.

Это уменьшает стоимость повторного onboarding и делает параллельность полезной для глубокого reverse, а не только для мелких lookup-задач.

Важно: цель — не «заставить агента работать N минут», а дать достаточную cohesive depth, чтобы он исчерпал область до meaningful boundary.

`CHAT-016` — direct acceptance persistent lane: тот же Agent 1 после Task 1 получает большой Task 2, сохраняет локальный контекст, проходит несколько evidence/blocker циклов и доводит full AE loop до final v5 без нового onboarding.

`CHAT-017` подтверждает модель на Agent 2: после table-identity Task он получает большой image-detail Task2 и продолжает тот же lane через static blockers, runtime evidence и финальный selftest.

### I-051 — Artifact-access preflight для каждого specialist lane
Статус: `CONSOLIDATED`.

`CHAT-015` показывает, что доступ к Project conversation context не гарантирует доступ к physical attachments.

Перед началом exact reverse parallel-agent должен подтвердить:
- что видит master/handoff;
- что видит authoritative full disassembly/binary;
- что видит required local notes/reference source;
- что может материализовать/читать их bytes.

Если любого обязательного объекта нет, задача не стартует. Оркестратор должен либо прикрепить bundle непосредственно к lane, либо дать durable shared locator.

Это предотвращает reverse по пересказу и является историческим аргументом в пользу будущего Drive/GitHub/MCP shared state.

`CHAT-016` снова подтверждает preflight: exact reverse не начинается, пока handoff archive физически не загружен в specialist chat.

### I-052 — Cross-Fullhan semantic oracle: сначала homolog, затем target proof
Статус: `CONSOLIDATED`.

Внешняя research-ветка `CHAT-015` превращает FH8852V100/V201 в систематический semantic oracle.

Workflow:
1. зафиксировать факты/unknown на FH8626 Apollo;
2. найти именованный homolog в близком Fullhan поколении;
3. сравнить algorithm fingerprint: constants, shifts, loop geometry, call shape, tables;
4. использовать reconstructed headers/structures только как semantic vocabulary;
5. вернуться к FH8626 ARM dump и доказать target offsets/state/MMIO;
6. confidence маркировать `EXACT / VERY_STRONG / STRONG / HYPOTHESIS`.

Критическое правило: соседний SoC **никогда** сам по себе не доказывает FH8626 ABI/register map.

Эта методика резко сократила blind reverse и дала semantic map большого участка `CB970`.

`CHAT-024` расширяет semantic-oracle принцип до OEM donor material: retail clone/dump полезен для symbols/config/layout comparison, но собственный target dump всегда сильнее и donor нельзя автоматически прошивать.

### I-053 — Living external-research document с provenance и reusable method
Статус: `CONSOLIDATED`.

Вместо серии одноразовых заметок `CHAT-015` создаёт один расширяемый research MD, где вместе хранятся:
- source URLs/repositories и локальные artifact paths;
- reproducible WSL reverse preparation;
- homolog map и confidence;
- superseded interpretations;
- selected reference excerpts;
- method, когда Cross-Fullhan oracle полезен/неполезен;
- instructions/addendum для следующих specialist agents.

Это отделяет **research method + provenance** от конкретного текущего handoff и позволяет следующим агентам переиспользовать внешний semantic corpus без повторного веб-поиска.

`CHAT-024` строит reusable external-research handoff: найденные форумы, официальные PDF/FCC, donor dumps и model aliases группируются по provenance и степени сходства, а не остаются списком ссылок.

### I-054 — Приоритет внешних references: named adjacent-SoC → same-SoC → дальние аналоги
Статус: `CONSOLIDATED`.

`CHAT-015` формирует эффективную лестницу внешнего исследования:
1. близкий Fullhan с именованными функциями — для semantic vocabulary;
2. vendor libraries близкого поколения — для instruction-level homologs;
3. другой firmware/rootfs **того же FH8626V100** — для function/binary/table diff;
4. официальные same-SoC SDK adapters/samples — для ABI/timestamp/orchestration hints;
5. дальние SoC — только если остаются пробелы.

Так external research отвечает на конкретный blocker и не превращается в бесконечный поиск «похожих камер».

### I-055 — Focused reverse workspace вместо recursive corpus scan
Статус: `CONSOLIDATED`.

Для глубокой функции/подсистемы полезно один раз собрать минимальный рабочий corpus:
- exact disassembly нужных функций;
- relevant full-library TXT;
- current reverse document;
- runtime/reference files только этой ветки.

Дальше искать `grep -n` / `sed -n` по конкретным адресам и файлам, а не обходить весь project tree.

В `CHAT-016` пользователь прямо остановил тяжёлые recursive file searches и потребовал точечный режим. После этого был выделен focused workspace для AE/controller files.

Преимущества:
- меньше I/O и зависаний;
- меньше false hits;
- проще provenance каждого вывода;
- быстрее повторная проверка конкретного адреса.

`CHAT-017` повторно подтверждает focused-corpus правило: пользователь останавливает broad find/re-disassembly и требует работать по уже известным точным путям/ARM_FULL/runtime dump.

`CHAT-027` уточняет эффективный режим для гигантского disassembly: известный address/symbol → короткое окно → один xref → следующий address. Не перечитывать whole ARM dump и не запускать recursive search без конкретного вопроса.

### I-056 — Размещать вычисление там, где ему место: static heavy work off-target
Статус: `CONSOLIDATED`.

`CHAT-016` формулирует устойчивое разделение:
- камера — только runtime/hardware evidence, которое нельзя восстановить офлайн;
- WSL/PC — disassembly, readelf, strings, diff, parsing, table audit, selftests;
- уже готовый TXT — анализировать напрямую, не дизассемблировать повторно;
- тяжёлую обработку не запускать на target и не заставлять оператора делать её вручную.

Если новый binary действительно требует full reverse, агент сначала говорит, **какой именно** artifact нужен и почему, затем один раз готовит его на WSL.

`CHAT-017` окончательно разделяет камеры и workstation: target снимает только live RW/GOT evidence; полный reverse и таблицы анализируются по подготовленному ARM_FULL/runtime corpus на WSL/PC.

### I-057 — Sparse liveness heartbeat без промежуточного reasoning
Статус: `OBSERVED`.

Autonomous milestone reporting и видимость работы не противоречат друг другу.

В `CHAT-016` оптимальная модель уточнилась:
- не публиковать каждую hypothesis/function finding;
- если локальная операция заметно затянулась — коротко сообщить текущий блок и отсутствие/наличие blocker;
- при tool failure/оборванном ответе быстро обозначить точку продолжения;
- после substantial milestone дать нормальный технический summary.

Это устраняет циклы «ты тут? / завис?» без возврата к шумному stream-of-consciousness.

`CHAT-027` усиливает sparse heartbeat: при длинном reverse пользователь хочет коротко знать текущий блок/последнюю подтверждённую точку; после tool/reasoning failure checkpoint должен быть сообщён сразу.

### I-058 — Stale/cache/queue state нужно отличать от active controller state
Статус: `OBSERVED`.

При runtime capture одно и то же числовое поле может быть:
- текущим active state;
- deferred queue value;
- cached previous value;
- profile default;
- dirty=0 stale value.

`CHAT-016` даёт хороший пример: `shared+0x00=894` сначала выглядел как конфликтующий active max-intt, но после проверки `C6AC8` оказался stale deferred queue value при `dirty=0`.

Правило:
- значение без lifecycle/dirty/owner semantics не повышать до active fact;
- для stateful controller фиксировать producer, dirty flag, commit point и consumer;
- runtime snapshot интерпретировать вместе с action/queue state, а не только по числу.

Это особенно важно для AE/AWB и других deferred control loops.

### I-059 — Двухслойный reverse corpus: static code + live runtime data
Статус: `OBSERVED`.

`CHAT-017` окончательно разделяет два разных вида authoritative evidence:

- `apollo_unpacked_ARM_full.txt` — executable/RX code, instructions, xrefs, call flow, MMIO access;
- `apollo_full_runtime.bin` / targeted RW dumps — initialized RW, GOT, runtime pointers, mutable tables, heap/state objects.

Они **дополняют**, а не заменяют друг друга. Повторный objdump runtime heap как ARM-кода не создаёт нового знания и может вводить в заблуждение.

Практический workflow:
1. static code дизассемблировать один раз;
2. runtime capture снимать только для missing mutable objects;
3. адреса/segments связывать через map;
4. runtime bytes интерпретировать как data, пока ARM consumer не докажет иное;
5. оба слоя держать рядом в одном corpus manifest.

Этот подход снял последние GOT/table blockers APC, LTM, NR3D и D1DB0 без повторного blind reverse.

### I-060 — Corpus-first architecture: подготовить reverse substrate один раз и запретить бессмысленное повторение
Статус: `CONSOLIDATED`.

В конце `CHAT-017` пользователь формулирует уже системную архитектуру:
- firmware binaries извлекаются один раз;
- для них заранее готовятся disassembly/symbols/relocations/strings/indexes;
- raw binaries становятся труднее случайно использовать как рабочий путь;
- агенты по умолчанию работают с prepared TXT/index/extracts;
- существует machine-readable reverse status matrix: что уже PREPARED/EXACT/PARTIAL/MISSING;
- хранится история expensive work, чтобы full objdump/reverse не запускался повторно;
- есть карта, где физически лежит artifact в Windows/WSL/camera/archive.

Это прямой precursor canonical reverse workspace: knowledge становится адресуемым corpus, а не набором вложений конкретного чата.

`CHAT-020` реализует corpus-first архитектуру физически: raw/reference/reverse/source/evidence разнесены, Apollo SQLite index и lookup helpers используются как primary path, full objdump отмечается как already-materialized expensive work.

`CHAT-021` распространяет corpus-first policy на stock static artifacts: модули/libs/binaries подготавливаются один раз с metadata/readelf/symbols/relocations/strings/disassembly; Apollo/JXF37 не переразбираются, если authoritative representations уже есть.

### I-061 — Specialist role-lock вместе с artifact preflight
Статус: `OBSERVED`.

Artifact preflight недостаточен: после загрузки общего handoff specialist должен заново подтвердить локальный role scope.

Минимальный preflight:
- `ROLE = Agent N / Task M`;
- owned functions/subsystem;
- excluded functions/other-agent boundaries;
- current authoritative inputs;
- expected final artifact;
- текущий blocker.

Это предотвращает случай `CHAT-017`, когда Agent 2 после загрузки общего handoff временно начал выдавать результат Agent 1.

`CHAT-020` материализует role separation в отдельном `roles/` и distinct orchestrator/executor entrypoints, уменьшая риск scope bleed.

### I-062 — Consolidated quality pack вместо независимых retrospective-файлов
Статус: `OBSERVED`.

К `CHAT-017` несколько agents уже создали собственные quality retrospectives. Пользователь требует не складывать их рядом, а:
1. сравнить;
2. дедуплицировать;
3. сохранить unique findings;
4. собрать один master quality/reverse-preparation document;
5. дать orchestrator import prompt.

Это превращает feedback разных specialist lanes в общий evolving process contract и предотвращает расхождение правил между агентами.

### I-063 — Layered project map с отдельными role/context entrypoints
Статус: `OBSERVED`.

В `CHAT-020` локальный master-tree получает один верхнеуровневый вход `PROJECT_MAP.md` и физическое разделение:
- `00_START/` — bootstrap/navigation;
- `state/` — только current state;
- `roles/` — orchestrator vs executor/reverse agent;
- `ops/` — WSL/OpenIPC/stock/build/hardware protocols;
- `runtime/` — implemented vs missing behavior;
- `roadmap/` — priorities/gates;
- `history/` — decisions/work/expensive operations;
- `knowledge/`, `reverse/`, `reference/`, `evidence/`, `source/`, `tools/`.

Это уменьшает context mixing: новый агент не читает весь handoff подряд, а входит через role/task-specific route.

### I-064 — Automated structural invariants вместо ручной проверки handoff
Статус: `OBSERVED`.

Quality rules в `CHAT-020` превращаются в исполняемые проверки:
- `HANDOFF_DOCTOR`;
- `PATH_AUDIT`;
- `ARCHIVE_AUDIT`;
- `DUPLICATE_AUDIT`;
- `SQLITE_INDEX`;
- `PROJECT_HEALTH`;
- coverage audit;
- обязательные entrypoints/tooling checks.

Успех измеряется не «архив вроде открывается», а зелёными invariants:
`nested_archives=0`, `exact_duplicates=0`, valid paths/index/state files.

### I-065 — Task router + context packs вместо загрузки всего проекта в каждый agent
Статус: `OBSERVED`.

Создаются `TASK_ROUTER.md`, machine-readable `TASK_ROUTER.json`, context packs и module-status matrix.

Agent task сначала классифицируется как AE/AWB/detail/hardware/build/video/Cross-Fullhan/unknown, после чего агент получает только нужные entrypoints/evidence.

Это одновременно:
- сокращает контекст;
- удерживает ownership boundaries;
- уменьшает repeated reverse;
- делает specialist startup воспроизводимым.

### I-066 — Session ledger и recovery playbook для долгих agent runs
Статус: `OBSERVED`.

`CHAT-020` добавляет постоянный `sessions/` слой:
- checkpoint/current task;
- command ledger;
- expensive-work history;
- blockers;
- exact next action.

Плюс `AGENT_RECOVERY_PLAYBOOK.md`: если агент потерял files/state/tool/semantics, он знает конкретный документ или индекс, который нужно перечитать.

Это переносит continuity из памяти чата в explicit operational substrate.

`CHAT-027` демонстрирует реальный context-loss recovery: после потери активной нити агент восстанавливает current checkpoint из handoff/DELTA/saved closure artifacts и корректирует next action с более поздней точки, а не начинает заново.

### I-067 — Нормализация artifacts: active knowledge отдельно от provenance/archive
Статус: `OBSERVED`.

Clean master-tree в `CHAT-020` вводит явные правила:
- normalized current knowledge — в active layers;
- original agent results/external refs — provenance;
- historical/superseded — archive/history;
- ни одного nested archive в active handoff;
- exact duplicates удаляются;
- один canonical disassembly на объект;
- provenance movement должен быть отслеживаемым.

Итог: handoff перестаёт расти простым накоплением файлов.

`CHAT-021` применяет normalization к evidence: superseded WIDE_DAY и misclassified audio capture остаются в provenance/raw manifest, но исключаются из canonical runtime set.

### I-068 — Fixed progress ontology через module/status matrix
Статус: `OBSERVED`.

Вместо одной плавающей оценки «готово X%» создаётся `MODULE_STATUS_MATRIX` / implementation registry, где для блока раздельно фиксируются:
- semantic identity/confidence;
- reverse status;
- runtime/source implementation;
- hardware evidence;
- next action.

Процент можно считать поверх этой модели, но он больше не является primary state representation.

Это прямое исправление E-036.

`CHAT-023` уточняет ontology: один и тот же subsystem может иметь `100% contract/hardware fact`, но только `80% production integration`; progress matrix должна показывать эти оси раздельно, а не запрещать слово 100 вообще.

### I-069 — Resource-budgeted capture: read-only не значит безвредный
Статус: `OBSERVED`.

После OOM в `CHAT-021` hardware acquisition получает явное правило ресурса:
- до dump вычислить размер;
- проверить target RAM/tmpfs/storage;
- на constrained camera снимать только нужные regions;
- большие объекты передавать chunk/stream на host;
- после каждого чанка удалять target copy;
- full target RAM никогда не держать целиком в RAM-backed `/tmp`.

Это превращает resource footprint в такой же prerequisite эксперимента, как boot mode или owner state.

### I-070 — Provenance-preserving metadata correction
Статус: `OBSERVED`.

Если capture технически хороший, но label/manifest оказался неверным, переснимать его не обязательно.

Правильный workflow, показанный в `CHAT-021`:
1. raw archive остаётся неизменным;
2. correction записывается отдельно с source/reason;
3. old interpretation помечается superseded;
4. canonical catalog использует corrected semantics;
5. downstream analysis знает, что bytes исходные, а metadata было исправлено.

Так были сохранены white-light capture с неверным lens label и audio capture, оказавшийся talkback.

### I-071 — Transition-targeted RAM evidence эффективнее diff независимых full heaps
Статус: `OBSERVED`.

Independent steady heap snapshots в `CHAT-021` отличаются примерно на половину bytes из-за allocator/live-buffer/temporal noise и плохо подходят для semantic diff.

Гораздо полезнее:
- `before`;
- `after_early`;
- `after_settled`;
- небольшие заранее доказанные VA regions.

Targeted transitions показали изменения порядка ~1–2%, которые легче связать с AE/lens/day-night objects.

Правило: full heap сохраняется как provenance/baseline, а причинный reverse строится на synchronized targeted transition capture.

### I-072 — Capture state matrix + canonical/superseded evidence catalog
Статус: `OBSERVED`.

Stock acquisition в `CHAT-021` оформляется как матрица, а не список tar-файлов.

Для каждого state/transition фиксируется наличие:
- UART/system snapshot;
- RAM evidence;
- sensor regs;
- GPIO/media/audio/network;
- packet/trace capability;
- status `DONE/PARTIAL/NOT_AVAILABLE/NOT_APPLICABLE`.

Отдельный evidence catalog хранит artifact ID, state, transition, source, firmware, sensor/lens, SHA/provenance и notes.

Это делает явными пробелы и предотвращает бесконечные «а это мы уже снимали?» hardware cycles.

### I-073 — Capability map target-а нужно использовать как executable contract
Статус: `OBSERVED`.

В stock preflight уже были известны абсолютные пути доступных tools, но позже агент снова пытался вызвать `tftp` через сломанный PATH и предполагал отсутствующие utilities.

Улучшение:
- один раз снять capability/tool matrix;
- recipes используют известные absolute paths на minimal target;
- отсутствие host-like utility не проверяется заново в каждом experiment;
- WSL/host выполняет parsing/compression, если target toolset ограничен.

Capability map должен быть входом command generator, а не просто справочной заметкой.

`CHAT-022` усиливает требование: current operational recipe должен использовать capability map и runbook — `.204`, `scp -O`, UART-first, mount `/dev/mtdblock6`, volatile `/tmp` и stock module order нельзя восстанавливать по памяти.

### I-074 — Evidence delta package поверх stable master
Статус: `OBSERVED`.

Вместо пересборки всего master-handoff после acquisition `CHAT-021` формирует отдельный `stock_evidence_delta`:
- ссылается на base master version;
- содержит только новое normalized evidence/static reverse material;
- не дублирует крупные уже-authoritative objects;
- имеет orchestrator README, gaps, provenance, state matrix и manifest.

Это хороший промежуточный transport pattern между mutable workspace и тяжёлым monolithic handoff.

`CHAT-023` закрепляет fan-in модель: specialist возвращает только DELTA относительно frozen master; orchestrator дедуплицирует и интегрирует его, а не пересылает whole master.

### I-075 — Source consolidation: canonical tree вместо цепочки diagnostic snapshots
Статус: `OBSERVED`.

`CHAT-022` показывает, что после быстрого experimental development source lineage `v4.2.0 → v5 → v7 → v8` стала опасной: в одном snapshot смешиваются productionizable code, diagnostics, temporary lifecycle patches и hardware-debug logging.

Правильный следующий этап:
- выбрать один canonical implementation tree;
- разнести board/platform, sensor, media lifecycle, ISP runtime, AE, AWB/CCM, IQ, control plane, diagnostics;
- validated deltas переносить маленькими patch/commit;
- versioned/candidate snapshots уводить в provenance;
- feature registry связывать с hardware evidence;
- не принимать последний diagnostic snapshot целиком как production source.

### I-076 — Board bootstrap — отдельный hardware contract, а не shell ritual
Статус: `OBSERVED`.

Главный dual-sensor root cause `CHAT-022` оказался не в runtime lens-switch logic, а в cold-boot board sequence:

`mount stockapp RO → product environment → GPIO5 LOW → load media modules → GPIO5 HIGH → owner`.

После этого stock `sensor_probe` видит оба GC1054 и WIDE↔TELE реально даёт изображение.

Вывод: boot-time GPIO/reset/module ordering должен жить в board/platform initialization и иметь отдельный acceptance status. Нельзя оставлять его ручной последовательностью UART-команд.

### I-077 — Control plane должен переживать потерю видеопотока
Статус: `OBSERVED`.

В diagnostic owner frame retrieval и FIFO control оказались связаны одним execution flow. При зависшем/blocking media call процесс может оставаться владельцем hardware, но перестать отвечать на `status`, `shutdown` и rollback.

Production architecture должна разделять:
- media retrieval;
- command/control plane;
- watchdog/health;
- shutdown/recovery.

Hardware owner обязан оставаться управляемым даже при полном video loss.

### I-078 — Hardware-proven mechanism отдельно от image-quality correctness
Статус: `OBSERVED`.

`CHAT-022` аппаратно доказал выполнение и rollback нескольких paths — live gain, APC, NR3D, LTM, manual AE, AWB→CCM. Но это не означает визуальную parity stock.

Особенно AWB→CCM существенно изменял gains/registers, а persistent green cast почти не менялся. Поэтому status должен различать:
- механизм выполняется;
- register contract совпадает;
- visual/algorithmic parity подтверждена.

Это предотвращает ложное повышение механического PASS до полного subsystem PASS.

### I-079 — Rejected-hypothesis ledger экономит hardware cycles
Статус: `OBSERVED`.

Tele-debug в `CHAT-022` последовательно исключил несколько правдоподобных причин: GPIO polarity, pinmux, hidden wrapper magic, один ioctl, отсутствие второго SetSensorFmt и другие.

Финальный handoff сохраняет их как `REJECTED_HYPOTHESES`, чтобы следующие агенты не повторяли те же эксперименты без нового противоречащего evidence.

Для длинного hardware reverse отрицательный результат должен быть таким же долговечным knowledge artifact, как положительный contract.

### I-080 — Frozen master + integration barrier для параллельных агентов
Статус: `OBSERVED`.

`CHAT-023` формализует проблему преждевременного version churn. Пока несколько agents работают от одного master:
- distribution base остаётся frozen;
- входящие DELTA можно анализировать и готовить к merge, но не объявлять новым canonical master автоматически;
- orchestrator ждёт согласованную integration window;
- затем выполняет conflict/dedup/source-consolidation pass;
- только после fan-in повышается official master version.

Это делает specialist results сопоставимыми и предотвращает ситуацию, когда разные agents уже работают от разных «самых новых» handoff.

### I-081 — Evidence-preparation pass перед глубоким reverse-agent
Статус: `OBSERVED`.

Оркестратор в `CHAT-023` проверил самодостаточность master и обнаружил: implementation-agent уже можно запускать, а research-agent будет эффективнее после отдельного stock evidence acquisition.

Последовательность стала:
`stable master + stock camera/dump → acquisition agent → normalized evidence delta → orchestrator merge → deep reverse agent`,
пока implementation-agent параллельно догоняет уже закрытые contracts.

Принцип: сначала один раз заполнить очевидные file/runtime-evidence gaps, чем заставлять reverse-agent многократно останавливаться ради нового hardware capture.

### I-082 — Contract-level 100% нужно показывать отдельно от product-level readiness
Статус: `OBSERVED`.

Слишком консервативный progress report `CHAT-023` создавал ложное ощущение, что в проекте вообще нет завершённых областей. После коррекции явно появились 100%-закрытые contracts: TFTP path, lens polarity, GPIO5 bootstrap requirement, physical TELE visibility, manual exposure/gain writes, отдельные IQ hardware contracts и prepared Apollo corpus.

Правило: для каждого блока отдельно показывать:
- fact/contract closure;
- source implementation;
- reproducible build;
- hardware validation;
- rollback/regression;
- product/upstream readiness.

Тогда «100% конкретного контракта» не смешивается с «100% всей подсистемы».

### I-083 — Device identity triangulation: software ID + PCB + physical fingerprint
Статус: `OBSERVED`.

В `CHAT-024` поиск retail-модели становится надёжным только после объединения независимых target-local признаков:
- internal software model `AJL33PQ0866`;
- firmware branch `YGT.AJL33PQ0866-...`;
- PCB family `CF26 / SM ... V1.0`;
- SoC FH8626V100;
- фактическая dual-lens optics 3.6 mm + 12 mm;
- physical 4+4 illumination/PTZ layout.

External listing считается сильным relative только когда совпадает несколько независимых осей. Внешний корпус сам по себе недостаточен.

### I-084 — Разделять hardware similarity и reverse utility donor-а
Статус: `OBSERVED`.

`CHAT-024` показывает, что «самая похожая камера» и «самый полезный donor» — разные рейтинги.

Например:
- один retail reference почти идеально совпадает по FH8626V100/dual-sensor/3.6+12/LED;
- другой device хуже совпадает по корпусу, но имеет downloadable SPI dump;
- третья линия имеет богатые firmware/root/RTSP discussions.

Donor catalog должен хранить минимум две независимые оценки:
1. hardware/software proximity к target;
2. ценность доступных artifacts для reverse.

### I-085 — Поисковый fingerprint строить пересечением внутренних идентификаторов
Статус: `OBSERVED`.

Вместо бесконечного поиска «dual lens Chinese camera» `CHAT-024` сводит поисковые ключи к пересечению:
`AJL33PQ0866 + CF26/SM400 + FH8626V100`.

Дальше поиск расширяется по соседним `AJL33*`, `AJ-SM-FH8626V100`, exact PCB aliases и firmware version strings.

Так retail/OEM aliases превращаются в systematic donor-firmware search, а не визуальный browsing.

### I-086 — Negative architecture audit: искать не только аналог, но и отсутствующую стадию
Статус: `OBSERVED`.

Cross-platform research в `CHAT-025` получает отдельный режим:
не только «как внешний SDK реализует X?», но и
«какой обязательный этап повторяется в нескольких vendor implementations и отсутствует в нашей architecture?».

Так сформирован сильный lead:
`ISP frame/stat-ready → immutable statistics snapshot → algorithms → staged transaction → frame-boundary apply → affected-frame publication`.

Если наш runtime перескакивает одну из повторяющихся стадий, это маркируется как `POSSIBLE_ARCHITECTURAL_OMISSION` и передаётся Agent 1 на exact FH8626 confirmation.

### I-087 — External artifact status должен различать FOUND, ACQUIRED и VERIFIED
Статус: `OBSERVED`.

В `CHAT-025` exact-FH8626V100 SDK packages найдены по официальным источникам, но bytes не удалось получить в текущем окружении. Агент правильно фиксирует их как `DISCOVERED_NOT_ACQUIRED`, а не как имеющийся SDK/BSP.

Для внешнего corpus полезна state machine:
`DISCOVERED → ACQUIRED → INVENTORIED → PREPARED → VERIFIED_USEFUL`.

Это не позволяет URL или имя архива случайно превратить в локальный authoritative artifact.

### I-088 — Targeted cross-platform lead должен быть коротким контрактом для reverse-agent
Статус: `OBSERVED`.

Вместо огромного external dump Agent 2 формирует для Agent 1 lead с полями:
- FH8626 target function/block;
- external analogue repository/file/function;
- что показывает analogue;
- какой stage предположительно отсутствует;
- что exact проверить на FH8626;
- evidence/provenance;
- confidence.

Так external research сокращает reverse, а не создаёт ещё один corpus, который Agent 1 должен разбирать с нуля.

### I-089 — Independent-source corroboration с учётом «расстояния» SoC
Статус: `OBSERVED`.

`CHAT-025` формализует confidence external inference:
- один exact same-SoC official source может быть сильным semantic evidence;
- для close relatives желательно 2–3 независимых implementations;
- чем дальше SoC/SDK generation, тем слабее inference;
- offsets/MMIO/layouts/constants всё равно требуют target proof.

Это делает external research falsifiable и уменьшает риск переноса «красивого» чужого ABI на FH8626.

### I-090 — Typed transactions + generation invalidation как architecture pattern для stateful ISP
Статус: `OBSERVED`.

Source/code audit в `CHAT-025` обнаруживает системный риск: AE/AWB/CCM/APC/NR/LTM независимо изменяют state/registers, а lens/profile/orientation/geometry меняются без общей invalidation model.

Предложена схема:
`immutable snapshot → candidate calculations → typed transaction → validate generation → ordered commit → readback/rollback`.

Минимальные generations: sensor, lens, profile, geometry, orientation, stream, statistics, algorithm.

Это не target reverse fact само по себе, а reusable production-architecture pattern, возникший из сопоставления reverse, source audit и внешних Fullhan state machines.

### I-091 — Двухуровневое хранение: MASTER_CORE + REVERSE_HEAVY
Статус: `CONSOLIDATED`.

`CHAT-026` формализует первый явный разрыв монолитного handoff на два canonical storage-role; `CHAT-034` затем физически материализует эту модель на master/core и heavy corpus и подтверждает её как рабочую:

**MASTER_CORE**
- часто обновляемый;
- status/contracts/maps/history/runbooks/source summaries;
- достаточно самодостаточен для обычной работы agents;
- содержит только небольшие representative excerpts/evidence.

**REVERSE_HEAVY**
- большой, редко изменяемый vault;
- full firmware/MTD/kernel/U-Boot binaries;
- full disassemblies/indexes;
- RAM/VMM/MMIO/RAW/YUV bulk;
- vendor/reference reverse corpora.

Это прямой исторический предшественник современной модели «Git current knowledge / external heavy evidence».

### I-092 — Heavy artifacts должны адресоваться стабильными logical IDs
Статус: `OBSERVED`.

После выноса bulk files из master нельзя оставлять документацию привязанной к случайным filenames/paths.

`CHAT-026` вводит `REVERSE_HEAVY_INDEX` и logical references вида:
`HEAVY:KERNEL_STOCK_DECOMPRESSED_ARM_FULL`.

Index разрешает logical ID в:
- actual path;
- type/size/provenance;
- назначение;
- derived artifacts;
- canonical/superseded status;
- master summary.

Это позволяет менять physical storage/filename heavy corpus без переписывания сотен knowledge-docs.

### I-093 — Рабочий transfer staging нужно отделять от истории
Статус: `OBSERVED`.

После inventory TFTP root в `CHAT-026` вводятся:
- `tftp_active/` — только файлы текущего обмена;
- `tftp_history/` — завершённые исторические transfer artifacts, разложенные по типам;
- `TFTP_LAYOUT.md` — простой lifecycle contract.

Новые acquisition files запрещено складывать прямо в корень. После окончания этапа они должны уйти либо в project/evidence corpus, либо в history.

### I-094 — Cleanup сначала inventory/move, удаление только после canonicalization
Статус: `OBSERVED`.

При уборке старого TFTP-корня агент сознательно не делает `rm`:
1. inventory;
2. classify;
3. move по категориям;
4. проверить, что root чист;
5. позже сравнить history с canonical MASTER/HEAVY;
6. удалять только гарантированные duplicates.

Это безопасный общий паттерн для reverse-проектов, где старый «мусор» может оказаться единственной копией evidence.

### I-095 — Master хранит conclusions/excerpts, heavy — bulk evidence
Статус: `CONSOLIDATED`.

`CHAT-026` задаёт явное anti-duplication rule:
- full disassembly/raw firmware/RAM/RAW/YUV/SQLite — только heavy;
- master содержит summary, addresses, provenance, acquisition method, derived diff и exact heavy reference;
- representative RAW/YUV в master допускается только если реально нужен для понимания;
- nested handoff archives внутри canonical archives запрещены.

Это уменьшает размер everyday handoff без потери reproducibility.

### I-096 — Storage role должен быть независим от транспортного контейнера
Статус: `OBSERVED`.

Архив в `CHAT-026` определяется как способ передачи, а не внутренняя структура знания:
- старые nested tar/zip распаковываются в canonical directories;
- исключение — только если архив сам является исследуемым firmware/container artifact;
- source vault и working core могут обновляться независимо.

Этот принцип позже естественно масштабируется с локальных tar-архивов на отдельные Git/evidence storage systems.

### I-097 — Evidence-directed reverse closure loop
Статус: `CONSOLIDATED`.

`CHAT-027` оформляет зрелый цикл закрытия reverse; `CHAT-034` показывает реальный второй проход Agent 4 evidence → Agent 1 closure и переводит паттерн в устойчивый:

`static corpus exhaustion → explicit E1–E7 evidence gaps → acquisition Agent 4 → normalized live evidence → Agent 1 targeted second pass → implementation-facing contracts`.

Ключевой эффект — после первого статического прохода агент не продолжает бессистемный objdump. Всё, что нельзя доказать статикой, превращается в named capture package с acceptance criteria. После получения evidence reverse возвращается только в конкретные границы.

### I-098 — Focused pack dependency closure должен быть fail-closed
Статус: `OBSERVED`.

После ошибки missing reverse substrate в `CHAT-027` появляется reusable packaging rule:
- построить dependency graph;
- resolve dedup/redirect;
- каждый dependency = `EMBEDDED` или `EXTERNAL_VERIFIED`;
- проверять actual bytes/role, а не только filename;
- unresolved dependency блокирует выдачу pack.

Это соединяет двухуровневое MASTER/HEAVY хранение с реальной specialist orchestration: heavy можно не копировать, но его доступность должна быть проверяемой.

### I-099 — Reverse status должен различать «release-relevant exhausted» и «maximal corpus exhausted»
Статус: `OBSERVED`.

В `CHAT-027` несколько раз возникал конфликт целей:
- сначала задача была «закрыть всё, необходимое для OpenIPC»;
- затем пользователь явно повысил планку до «дореверсить вообще весь доступный stock corpus, включая optional branches».

Поэтому статусы должны быть отдельными:
- `IMPLEMENTATION_READY`;
- `RELEASE_RELEVANT_REVERSE_EXHAUSTED`;
- `MAXIMAL_STATIC_CORPUS_EXHAUSTED`;
- `HARDWARE_ONLY_RESIDUAL`.

Это предотвращает преждевременное «всё закончено», когда optional WDR/LSC/dev_ctrl/recognition archaeology всё ещё входит в текущую цель.

### I-100 — Physical evidence может supersede human-readable inference, не уничтожая numeric contract
Статус: `OBSERVED`.

E3 в `CHAT-027` проходит несколько уровней:
сначала числовая permutation table `0,3,1,2`, затем Agent 4 физически подтверждает RAW10 и CFA orientations.

Правильная модель:
- numeric machine contract сохраняется;
- human label добавляется только после physical corroboration;
- старые неуверенные названия помечаются superseded;
- implementation может использовать numeric contract независимо от человекочитаемой CFA терминологии.

Этот паттерн применим к любым reverse enum/bitfield semantics.

### I-101 — Kernel/disassembly address base должен быть частью provenance
Статус: `OBSERVED`.

При watchdog reverse обнаружено, что старые targeted kernel excerpts были декодированы с неверной адресной базой `0xA...`. Canonical kernel base — `0xC0008000`; старые excerpts помечены superseded и пересозданы.

Для prepared reverse slices нужно хранить:
- source image;
- load/virtual base;
- extraction command/range;
- symbol/address mapping;
- superseded status.

Иначе точечный disassembly может выглядеть корректным, но быть семантически ложным.

### I-102 — Stable agent identity: wave + role + task вместо bare Agent N
Статус: `OBSERVED`.

После путаницы раннего и позднего `Agent 3` в `CHAT-028` durable provenance должен использовать составной ID:
`wave / role / task / branch`.

Например:
- `W1-A3-DUAL_SENSOR_REVERSE`;
- `W2-A3-OPENIPC_PRODUCTIZATION`.

Числовой номер можно оставить удобным UI-label, но он не должен быть единственным ключом в chronology, handoff или merge notes.

### I-103 — Orchestrator routing graph: integration gap → external lead → exact reverse → productization
Статус: `CONSOLIDATED`.

Центральный поток `CHAT-028` формализует взаимодействие трёх основных lane; `CHAT-034` подтверждает эту маршрутизацию на реальных parallel results:
- OpenIPC/productization обнаруживает конкретный integration blocker;
- External Research ищет SDK/source/homolog и формирует lead;
- Exact Reverse подтверждает FH8626 semantics;
- productization получает подтверждённый contract обратно.

Промежуточные материалы передаются пакетами, а orchestrator остаётся единственной точкой canonical merge. Это уменьшает прямые cross-agent conflicts и не создаёт несколько competing truths.

### I-104 — Frontend decoupling: hardware backend не должен зависеть от Majestic/Divinus
Статус: `OBSERVED`.

В `CHAT-028` появляется чёткая продуктовая архитектура:

`FH8626 platform/media/control backend → generic capability/API → Divinus / Majestic adapter / другой frontend`.

Majestic больше не является обязательным условием работоспособности камеры. Divinus можно сначала подключить через encoded-source sidecar, а native HAL развивать позже.

Так reverse и hardware ownership остаются reusable независимо от судьбы конкретного streamer-а.

### I-105 — Engineering bring-up и upstream-clean profile нужно разделять
Статус: `OBSERVED`.

OpenIPC productization discussion в `CHAT-028` отделяет две цели:
- engineering/private bring-up может временно использовать externally supplied vendor/stock dependencies;
- upstream candidate должен иметь допустимый provenance/build path и не поставлять случайно извлечённые factory blobs как canonical source.

Это позволяет не останавливать hardware development из-за supply-chain blocker, но и не выдавать engineering hack за upstream-ready решение.

### I-106 — External firmware corpus полезен для provenance и lineage, не только для reverse
Статус: `OBSERVED`.

Поздний `CHAT-028` строит практический firmware-research workflow:
`download → binwalk/extract → inventory kernel/modules → compare SDK strings/vermagic/symbols/sections/exact files`.

Цель — не прошивать чужую OEM firmware, а:
- найти независимые повторяющиеся vendor payload;
- установить SDK/kernel lineage;
- сравнить media modules;
- усилить provenance;
- обнаружить более близкий reference corpus.

Board compatibility проверяется отдельно; same SoC не считается основанием для flash.

### I-107 — Verification evidence belongs to exact bytes/commit, not filename or design intent
Статус: `OBSERVED`.

`CHAT-029` demonstrates the correct response after artifact reconstruction:
- original apply-checked patch is lost;
- reconstructed package is not allowed to inherit the old PASS;
- status is downgraded to `RFC/rebase-required`;
- next integration must re-run the applicable gate.

This generalizes to builds, patches, binaries and generated bundles: provenance continuity is part of verification.

### I-108 — Fixed byte-level sidecar ABI decouples 32/64-bit producer/consumer layouts
Статус: `OBSERVED`.

Agent 3 replaces an implicit C-struct transport with a fixed wire format for encoded frames. The goal is to eliminate padding/alignment dependence between target ARM producer and host/frontend consumer.

The broader rule: cross-process/cross-architecture boundaries should define explicit serialized fields/versioning/generation, not share native C layout by assumption.

### I-109 — Sidecar-first productization preserves the known-good hardware owner
Статус: `OBSERVED`.

Instead of immediately making Divinus/Majestic own FH8626 hardware, `CHAT-029` builds:
`single known-good owner → copied encoded frames → sidecar transport → RTSP/Divinus`.

This creates a low-risk productization milestone:
- hardware lease/release semantics stay in one owner;
- frontend development can proceed independently;
- target streaming can be validated before native HAL ownership;
- native HAL remains a later migration, not a prerequisite.

### I-110 — Buildroot/OpenIPC staging should be source-only and repo-boundary aware
Статус: `OBSERVED`.

Agent 3 prepares real source-only packages/staging for runtime and ABI probe while explicitly excluding generated ELF, stock `.ko/.so/.bin` and fake firmware images.

The staging separates:
- generic FH8626 platform/runtime;
- camera/board-specific support;
- engineering profile;
- upstream-clean profile.

This is the first source-level productization bridge from reverse artifacts to intended OpenIPC repository ownership.

### I-111 — Accumulate offline stages; deliver one cumulative validation package
Статус: `OBSERVED`.

Agent 5 in `CHAT-030` explicitly changes strategy:
- stop sending each offline audit/stage as separate `.tar.gz`;
- keep one sequential source/integration line;
- exhaust cheap offline work first;
- at the next hardware/WSL boundary produce one package/runner;
- runner applies cumulative changes, builds/checks once and emits one report.

This is the artifact analogue of command decision boundaries.

### I-112 — Use the simplest native tool for packaging/file operations
Статус: `OBSERVED`.

User correction in `CHAT-030` reinforces baseline rule 5:
- shell/bash for orchestration;
- `tar` for archives;
- `bash -n`, `git diff --check`, `checkpatch`, `make` for relevant validation;
- Python only when actual parsing/computation benefits from it.

The issue is not Python itself, but adding a slow/opaque generation layer where ordinary system tools are clearer and more reproducible.

### I-113 — Kernel bring-up should separate safe platform closure from media/VMM high-risk layer
Статус: `OBSERVED`.

The historical Agent 5 plan explicitly stages native kernel work:
1. machine/interrupt/timer/UART/SPI/MTD/GPIO/I2C baseline;
2. Ethernet/RTC and safe peripheral integration;
3. native pinctrl and PMU dependencies;
4. USB/SDIO/SPI1/DMA/AES/audio;
5. media/VMM and upper-memory policy last.

This keeps high-risk multimedia/VMM work from destabilizing already hardware-proven platform bring-up.

### I-114 — Critical command protocol должен содержать executable examples и anti-examples
Статус: `OBSERVED`.

После многократных нарушений в `CHAT-031` пользователь требует отдельный `CRITICAL_COMMAND_PROTOCOL.md`.

В нём фиксируются не абстрактные пожелания, а конкретные patterns:
- один logical step = один fenced multi-line block;
- команды внутри отдельными строками;
- no `&&` chain;
- no one-block-per-command fragmentation;
- no backslash continuation;
- explicit `cd`;
- Explorer вызывается один раз и делает `/select,"exact-report"`;
- agent анализирует report и сам выбирает следующую ветку.

Rule с positive/negative examples оказался устойчивее устной коррекции.

### I-115 — Pre-send artifact consistency check: claimed fix должен существовать в bytes
Статус: `OBSERVED`.

`CHAT-031` показывает, что source/report state надо проверять перед выдачей:
1. открыть фактический working file;
2. проверить ожидаемый diff;
3. собрать/упаковать;
4. распаковать или перечитать packaged copy;
5. убедиться, что critical behavior реально изменён;
6. только потом обновлять README/status.

Это отдельный gate от compile/test: artifact может быть syntactically valid, но содержать старую реализацию.

### I-116 — Authoritative validation surface должен быть явно закреплён
Статус: `OBSERVED`.

Для Agent 6 пользователь фиксирует:
- source reading/patch preparation у агента;
- `git apply --check`, unit/regression, ARM cross-build, ELF/ABI и runtime — только в user WSL;
- до WSL result статус только `PENDING_WSL`;
- agent не должен превращать собственный local/static check в target/build PASS.

Это сохраняет один authoritative build environment и устраняет противоречащие результаты разных sandbox/toolchain.

### I-117 — Fail-closed runtime API guards защищают single-owner boundary
Статус: `OBSERVED`.

Divinus hardening в `CHAT-031` обнаруживает, что startup config запрещал hardware-owned features, но generic HTTP API мог попытаться включить их позднее.

Candidate добавляет fail-closed guards для hardware-owned audio/ISP/JPEG/MJPEG/night controls в FH86 external-source mode, оставляя purely frontend-side muxing доступным.

Принцип: capability restriction должна применяться и на config parse, и на runtime mutation surface.

### I-118 — Provider boundary отделяет feature API от hardware producer
Статус: `OBSERVED`.

MJPEG в `CHAT-031` переклассифицируется из «невозможно» в «provider отсутствует».

Divinus-side `/image.jpg` / `/mjpeg` можно подготовить независимо, но capability остаётся `unavailable`, пока single hardware owner не предоставляет JPEG frames.

Так feature surface можно развивать заранее без создания второго hardware owner или software decode→reencode workaround.

### I-119 — Hardware-proven contract является protected baseline
Статус: `OBSERVED`.

`CHAT-032` показывает, что compile/host/selftest и красивые runtime counters не могут понизить значение уже полученного physical proof. Если новый candidate расходится с FOV/image/target behavior, first-class действие — вернуться к exact known-good contract и минимально изолировать delta.

Практическое правило: supersede аппаратно подтверждённого механизма допускается только новым physical evidence с не меньшей строгостью.

### I-120 — Failed release branch должен превращаться в negative-evidence handoff
Статус: `OBSERVED`.

После провала Agent 7 полезным результатом оказался не очередной «исправленный release», а аварийный handoff:
- R13–R18 явно `quarantine/do-not-merge`;
- опровергнутые hypotheses перечислены;
- рабочие findings отделены от неработавшего code;
- причины regression и нарушения process-contract сохранены;
- downstream orchestrator получает cherry-pick candidates, а не ложный canonical snapshot.

Провальная ветка поэтому сохраняет инженерную ценность, не заражая production lineage.

### I-121 — Ghidra semantic reverse DB вместо flat-objdump workflow
Статус: `OBSERVED`.

`CHAT-033` формулирует новый основной static workflow:
`binary → Ghidra analysis → decompiled C + CFG + XREF + callers/callees + globals/types/strings + unresolved edges → agent interpretation → ASM confirmation`.

Для каждой функции важен не гигантский TXT, а адресуемый semantic record с confidence/provenance. Знания о prototypes/structures/names должны улучшать последующие decompilations, а не жить только в заметках чата.

### I-122 — Cross-binary knowledge layer связывает userspace ↔ ioctl ↔ kernel/modules ↔ hardware
Статус: `OBSERVED`.

Ghidra хорошо распространяет имена внутри одной программы, но Apollo, kernel и .ko остаются разными binaries. Поэтому поверх per-program analysis нужен общий index:
- symbols/types;
- ioctls/device nodes;
- cross-binary links;
- claims/conflicts;
- unresolved relations.

Это позволяет одному агенту назвать функцию/contract, а другим использовать знание без повторного reverse. Git/snapshot служат долговременной фиксацией, а не единственным realtime coordination mechanism.

### I-123 — WSL-only project root + README entrypoint + three storage classes
Статус: `CONSOLIDATED`.

В `CHAT-033` постепенно нормализуется local architecture: WSL как единственная рабочая среда, `README.md` как обязательная точка входа, компактный working layer, heavy evidence и disposable `tmp`. `CHAT-037` подтверждает это уже практической очисткой project boundary и отделением тяжёлых/воспроизводимых данных.

OpenIPC repos отделяются от reverse evidence; TFTP остаётся transport/staging; transport archives не становятся внутренним storage format.

### I-124 — Complex subsystem: analysis → implementation → independent integration audit
Статус: `OBSERVED`.

`CHAT-035` показывает правильную последовательность для сложного stateful блока:
1. reverse/contract analysis;
2. отдельная source implementation;
3. unit/negative/sanitizer checks;
4. full owner preflight по init/run/error/stop/restart/mode-switch;
5. independent повторный аудит;
6. только затем `DONE` и handoff.

Именно пропуск шага 4 сначала дал ложный `5/5 COMPLETE`, хотя component code выглядел корректно.

### I-125 — Analyst готовит code candidate, integrator владеет подключением и target acceptance
Статус: `OBSERVED`.

В `CHAT-035` пользователь явно разделяет роли:
- reverse/analysis agent восстанавливает ABI/algorithms, пишет C/H, host-checks, diff и integration notes;
- integration-agent подключает это к актуальному owner/OpenIPC tree, собирает ARM и валидирует target;
- analyst не обязан тащить на себя deployment и не должен считать host PASS hardware PASS.

Так тяжёлый reverse можно выполнять глубоко и автономно, а production line не блокируется экспериментальным кодом.

### I-126 — One canonical workspace; competing ACTIVE trees запрещены
Статус: `OBSERVED`.

`CHAT-036` сводит несколько `ACTIVE/ACTIVE_NEW/OLD_INCOMPLETE` деревьев в одну рабочую директорию. Historical/raw provenance остаётся внутри специально маркированной history/evidence структуры, но не конкурирует с current source.

Это устраняет ambiguity «какой C/H настоящий» и делает archive recovery исключением, а не обычным способом навигации.

### I-127 — FILE_INDEX как машинная карта с semantic metadata и deep-index layer
Статус: `OBSERVED`.

Индекс становится не просто checksum manifest. Для normal layer фиксируются:
`path / size / mtime / mode / hash / kind / description / use_when / required / duplicate group`.

Для огромного reverse corpus используется отдельный deep index, на который ссылается верхний индекс. Обычный `index` сохраняет ручные semantic поля; `verify-index` может force-rehash. Индекс обновляется после пакета файловых изменений, а не в конце длинной сессии.

### I-128 — Recovery checkpoint и handoff/continuation — разные artifacts
Статус: `CONSOLIDATED`.

После correction в `CHAT-036` checkpoint определяется как recovery snapshot текущего workspace после значимого этапа/длительной сессии, хранится вне canonical tree и должен реально сохраняться внешне. Handoff/continuation нужен для смены чата/агента. `CHAT-037` подтверждает модель реальным Drive-backed checkpoint layer.

Resume сначала сверяет filesystem с последним known index и не должен перезаписывать повреждённое состояние.

### I-129 — Semantic startup: automation помогает, но агент обязан перечитать контекст сам
Статус: `OBSERVED`.

Старт новой итерации делится на два слоя:
1. integrity preflight — index/files/permissions/links;
2. manual semantic read — README → STATE → PROGRESS → referenced evidence/docs → финальный reread текущей задачи.

Это одновременно context restoration и documentation audit. Скрипт не должен симулировать понимание проекта.

### I-130 — Installed tooling и generated build state живут вне transferable project
Статус: `OBSERVED`.

`CHAT-037` переносит Ghidra installation в system-style location и Ghidra project DB в отдельный local store; project сохраняет только reverse exports/logs/scripts/knowledge. Аналогично Buildroot `output/dl/build` рассматриваются как воспроизводимый generated state.

Это резко уменьшает handoff и делает project boundary семантической.

### I-131 — Exact-content reconciliation сохраняет уникальный corpus и убирает transport duplicates
Статус: `OBSERVED`.

При очистке старых handoff/archive деревьев пользователь запрещает удалять их «по имени» или целыми каталогами. Метод:
1. inventory archive objects;
2. exact-content grouping;
3. unpack unique containers один раз в staging;
4. hash normal files;
5. сравнить с active corpus;
6. materialize только реально unique contents;
7. сохранить provenance/path map;
8. только затем удалить redundant transport copies.

Content hashes здесь внутренний machine mechanism для дедупликации; это не отменяет правило не засорять user-facing transfer SHA без запроса.

### I-132 — Upload limit решается semantic active/history split, а не byte-split
Статус: `OBSERVED`.

Когда полный пакет превышает лимит, `CHAT-037` предлагает не `split` произвольных byte parts, а разные storage roles:
- ACTIVE — current source/docs/reverse для обычной работы;
- HISTORY/HEAVY — legacy unique corpus, редко нужный;
- tmp/build/cache — не транспортируются.

Архив сохраняет provenance/mtime, но возраст файла не становится единственным критерием authority.

### I-133 — Не переписывать существующий platform backend без product-level необходимости
Статус: `OBSERVED`.

После сравнения native Divinus HAL и запуска Majestic FH8852-family стратегия меняется: собственный Fullhan HAL не выбрасывается, а замораживается как reverse/reference/fallback. Product path проверяет более дешёвый compatible backend по реальным gates `VI → VENC → stable RTSP → ISP/control`.

Это anti-sunk-cost pattern: глубокий reverse сохраняется как знание, но product engineering выбирает минимальный путь, если existing family implementation обеспечивает необходимые контракты.


### I-134 — PR-facing branch отделяется от working/development line
Статус: OBSERVED.

Если branch является head уже открытого upstream PR либо intended contribution line, промежуточная работа не должна насыпаться туда по одному экспериментальному commit. Рабочие изменения идут в current work/topic branch, затем после complete review/build/hardware gate PR-facing history обновляется один раз чистой серией.

Правило не превращается в догму «ровно две ветки»: крупная изолированная задача может иметь topic branch, но после завершения она не остаётся вечным competing head.

### I-135 — Один authority repo хранит ownership matrix и live upstream rules
Статус: OBSERVED.

CHAT-038 создаёт durable openipc-upstream-rules.md: ownership Linux/Firmware/Builder/Divinus/Majestic/U-Boot/ipctool, direct links на live upstream rules, дату последней проверки и обязанность обновить local rules при upstream drift.

### I-136 — Generated binaries не живут в source Git
Статус: OBSERVED.

U-Boot pass закрепляет: .bin/.img/.elf, kernel images и другие generated artifacts остаются ignored build output, release artifacts либо external artifact/evidence storage. Source Git хранит source/config/tooling/docs.

### I-137 — Target conventions имеют приоритет над factory migration reference
Статус: OBSERVED.

OpenIPC-native U-Boot прячет Fullhan-specific 64 KiB Boot-ROM/DDR container + 192 KiB U-Boot внутри стандартного 256 KiB OpenIPC boot partition, переносит env на 0x40000, kernel на 0x50000 и rootfs на 0x250000.

Главный принцип: адаптировать SoC/board к target ecosystem, а не ecosystem к factory layout. Special case допускается только если standard contract объективно невозможен.

### I-138 — Browser/API-first agent не имитирует недоступные local build capabilities
Статус: OBSERVED.

В kernel/U-Boot работе пользователь повторно пресекает попытку разворачивать тяжёлые локальные build/checkpatch операции в browser-agent среде. Agent доводит source/audit через API, а authoritative owner build остаётся в пользовательском WSL/CI surface.

Если Bridge permission отсутствует, это фиксируется как permission gap; workaround через другой connector/Actions не подменяет штатный workflow.

### I-139 — Agent и Reviewer — разные identities и разные полномочия
Статус: OBSERVED.

Koba Bridge реально использует отдельные GitHub Apps: Agent mutates branches/files/history; Reviewer независимо читает/сверяет refs/commits/trees и не выполняет source mutations.

### I-140 — History rewrite должен быть controlled semantic operation, а не unrestricted force-push
Статус: OBSERVED.

Первый реальный Bridge use-case приводит к high-level capabilities: rewrite branch identity с expected_head_sha, dry-run, old→new mapping и tree preservation; reserved-branch admin repoint только при проверенном same-content contract; explicit capability/policy introspection.

История переписывается только при сохранении messages/dates/trees и независимой Reviewer verification.


### I-141 — Upstream series реконструируется от чистого base, а не полируется поверх migration-WIP
Статус: OBSERVED.

CHAT-039 показывает рабочий contribution pattern для уже функционирующего kernel port: старые два больших migration commits не правятся бесконечными fixup'ами. Вместо этого итоговое дерево перечитывается, изменения классифицируются, затем функционально эквивалентная clean series собирается заново от parent platform branch.

Hardware-proven semantics сохраняются; unrelated cleanup и ошибочные промежуточные гипотезы в финальную серию не попадают. Новые behavior-changing deltas получают отдельный статус и retest gate.

### I-142 — Kconfig/board naming описывает физическую capability, а не retail SKU
Статус: OBSERVED.

Kernel option FH8626V100_AJL33PQ0866_MMC заменяется на hardware-neutral FH8626V100_SD0_1BIT. Retail camera выбирает этот symbol выше по стеку, но Linux source описывает SoC/board electrical capability.

Так generic kernel не захватывает ownership конкретной камеры, а Builder/Firmware остаются местом product selection.

### I-143 — Одинаковый failure в stock и native ограничивает blame, но не доказывает отсутствие hardware
Статус: OBSERVED.

RTC/TSENSOR на AJL одинаково timeout'ится в stock и native 4.9.129; stock config также использует hw_rtc=no. Это сильное доказательство, что проблема не внесена OpenIPC port'ом.

Но из этого нельзя выводить, что RTC/TSENSOR IP физически отсутствует или неисправен. Правильный next step — отдельная bounded research task по PMU/clock/reset/analog init. Capability не публикуется до реальных изменяющихся physical samples.


### I-144 — Repository ownership определяется архитектурным слоем, а не историческим местом файла
Статус: OBSERVED.

CHAT-040 превращает informal rules OpenIPC в реальную migration matrix: kernel source/patches уходят в Linux, camera-specific profile — Builder, streamer behavior — его repository, Firmware оставляет shared SoC-family integration.

Файл не остаётся в Firmware только потому, что когда-то был туда скопирован для bring-up.

### I-145 — Preservation snapshot сначала инвентаризируется, потом разбирается
Статус: OBSERVED.

Большой Firmware WIP сохраняется как evidence/checkpoint, но не продолжается как target branch. Перед очисткой состав классифицируется по ownership/provenance и только затем переносится в соответствующие repos.

Это предотвращает потерю единственной копии work-in-progress при архитектурном cleanup.

### I-146 — Streamer-neutral core + runtime-specific overlays
Статус: OBSERVED.

Firmware получает общий FH8626 core, а Divinus и Majestic существуют как отдельные directions поверх него. Shared platform fixes должны сначала попадать в core; runtime-specific compatibility/packages не текут обратно в core автоматически.

Эта же модель позже переносится в Builder composed variants.

### I-147 — Factory-extracted binaries допустимы как evidence, но не как final product dependency
Статус: OBSERVED.

Для каждого .ko/.so/.bin фиксируется intended disposition:
1. найти полноценный SDK/source/build input;
2. если source найден — собирать воспроизводимо;
3. если source отсутствует — reverse/reconstruct replacement;
4. identical ready-made blob из SDK без source не закрывает source-replacement goal;
5. factory-extracted object остаётся reference/evidence.

Runtime data/tuning blobs рассматриваются отдельно от executable code, но также требуют provenance.


## Исходные этапы, ещё не подтверждённые

`CHAT-010` является прямым acceptance-тестом этого принципа: новый агент по handoff сразу продолжает с dequeue boundary, не повторяет sensor/ISP/H.264 bring-up и использует указанные checkpoint paths/constraints. Handoff реально переносит инженерное состояние между чатами.

### I-001 — Ручные браузерные вставки → workspace и архивы
Статус: `BOOTSTRAP`.

`CHAT-001` показывает уже локальный WSL workspace/checkpoints, но сам переход из ещё более ранней схемы пока не восстановлен.

### I-002 — Workspace → Google Drive
Статус: `CONSOLIDATED`.

`CHAT-036` сначала формализует Drive как обязательное persistent mirror/recovery layer, а `CHAT-037` показывает фактическое создание `reverse_FH8626V100_WORKSPACE` с browseable canonical docs, index и full checkpoint. Локальный workspace больше не считается достаточной долговременной authority сам по себе.

### I-003 — Google Drive → GitHub authority
Статус: `CONSOLIDATED`.

CHAT-038 показывает уже фактическую repository-native модель: reverse repo хранит current state/rules/contracts; связанные OpenIPC repos содержат implementation; Koba MCP Bridge выполняет branch/file/history mutations; Reviewer App независимо проверяет результат. Google Drive остаётся heavy evidence/recovery layer, но current engineering truth переходит в Git.

### I-005 — Env-переменные как устойчивый способ передачи сложных значений
Статус: `BOOTSTRAP`.

Есть активное использование shell variables, но недостаточно данных, чтобы считать именно заявленный позже transition доказанным.
