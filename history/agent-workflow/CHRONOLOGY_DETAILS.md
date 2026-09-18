# Подробная история реверса и портирования FH8626V100

Этот файл дополняет `CHRONOLOGY.md`. Основная хронология остаётся короткой; здесь сохраняются только детали, объясняющие важную развилку, неудачный подход, метод доказательства или инженерный контракт.

Источник первого блока: `CHAT-005` → `CHAT-001` (примерно 2026-08-24 — 2026-08-26).

## D1 — Первичная разведка и recovery baseline

Самый ранний источник (`CHAT-005`) начинается с boot/UART log и решения не пытаться сразу строить полноценную камеру.

Сначала:
- идентифицированы SoC, 8 MiB NOR, RAM, GC1054 и dual-lens признаки;
- полный flash dump снят внешним способом до любой мутации;
- из dump восстановлены фактическая MTD/U-Boot environment и данные, необходимые для штатного доступа к bootloader;
- после входа в U-Boot подтверждены flash/network commands и реальная environment;
- маленьким TFTP-файлом напрямую доказан путь `PC → Ethernet → U-Boot → RAM`.

Важно, что cross-device/web hints на этом этапе использовались только как ориентир; authoritative факты затем брались из собственного dump/bootloader.

После получения root shell и полного firmware corpus CHAT-006 добавляет ещё один ранний поворот: dump используется не только как recovery image, но и как самостоятельный reverse source.

Из него без дополнительных действий на target были извлечены stock kernel/initramfs, SquashFS /app, Fullhan media modules, ARC firmware и sensor/MIPI libraries. Это дало ранний ABI baseline: vermagic, module dependencies, memory reservation/VMM contract и первые ioctl mappings.

Практический урок: первый milestone порта — immutable recovery anchor и безопасный путь исполнения из RAM, а все bytes, уже полученные в полном dump, должны анализироваться офлайн без лишних target-команд.

## D2 — OpenIPC RAM boot

Стратегия гибридного запуска была выбрана ещё в `CHAT-005`: не подменять FH8626 неподтверждённым kernel от другого Fullhan SoC, а использовать stock FH8626 kernel и внешний OpenIPC initramfs через уже доказанный TFTP/RAM transport.

В `CHAT-001` этот план дошёл до hardware result:
- stock FH8626 kernel;
- внешний OpenIPC initramfs;
- U-Boot/TFTP;
- OpenIPC shell/network/MTD без записи NOR.

Это позволило аппаратно доказать OpenIPC userspace до появления native OpenIPC kernel.

Разделение kernel/userspace оказалось полезным: неизвестность «работает ли вообще Buildroot/OpenIPC на CPU» была закрыта независимо от более сложного media/kernel port.

## D3 — Buildroot и чистый OpenIPC baseline

Первые сборки выявили не FH8626-specific, а workflow-проблемы:
- Buildroot плохо переносил Windows-mounted filesystem semantics;
- Windows PATH загрязнял WSL build;
- stock kernel содержал встроенный factory initramfs, поэтому внешний OpenIPC CPIO не удалял старые init scripts автоматически.

После переноса build tree в Linux filesystem и правильного overlay стало возможным загрузить чистый OpenIPC userspace без конфликтующего factory startup.

Важная методическая деталь: ошибки инфраструктуры нужно отделять от ошибок target port, иначе media bring-up быстро превращается в смешанный debugging всего сразу.

`CHAT-007` даёт подробную механику этого перехода: первый RAM image действительно дошёл до `Welcome to OpenIPC`, DHCP и Dropbear, но stock kernel merge-ил встроенный factory initramfs с внешним CPIO. Это породило uClibc/factory init scripts внутри musl rootfs. После переноса build с `/mnt/c` в Linux filesystem и правильного overlay factory startup был перекрыт, получен чистый OpenIPC baseline.


## D4 — Stock media stack внутри OpenIPC

Заводской `/app` стал ценнее поиска абстрактного SDK:
- были найдены Fullhan media kernel modules;
- восстановлен их порядок загрузки и VMM/ARC firmware setup;
- media device nodes появились внутри OpenIPC;
- sensor/MIPI shared libraries загрузились в musl process при правильном порядке dependencies.

Это резко сократило неизвестность. Вместо немедленного clean-room переписывания всего media SDK стало возможно использовать stock kernel ABI как мост и постепенно восстанавливать userspace contract.

На этом же этапе был важный поворот workflow: детальный разбор каждого callback sensor library был остановлен, когда стало ясно, что он не отвечает на текущий blocking question.

В `CHAT-007` этот этап доказан по частям на живой системе: stock `/app` смонтирован read-only, `libmipi.so` и `libgc1054_mipi.so` успешно `dlopen()`-ятся непосредственно под musl OpenIPC, а kernel chain `vmm → xbus_rpc → media_process → isp → enc → jpeg → bgm → gpio_wave` загружается в штатном порядке. Появляются `/dev/rtxbus`, `/dev/media_process`, `/dev/isp`, `/dev/pae`, `/dev/jpeg`, `/dev/bgm` и gpio-wave nodes.


## D5 — Оживление ISP

Sensor и MIPI path удалось довести до состояния, совпадающего с ключевыми stock hardware признаками, но ISP IRQ оставался нулевым.

Дальнейшая работа объединила:
- stock runtime tracing;
- static reverse driver/userspace;
- reference source как semantic map;
- live MMIO comparison.

Ключевой найденный разрыв был не в sensor clock/MIPI, а в базовом ISP hardware state, включая interrupt enable/mask. После восстановления соответствующего состояния ISP начал стабильно генерировать interrupt cadence.

Это важный пример смены уровня диагностики: после доказанного sensor/MIPI больше не возвращались к ним при каждом симптоме downstream.

`CHAT-007` заполняет предысторию нулевого ISP IRQ: GC1054 ID/initialization, GPIO select, clocks и MIPI state были доведены до stock-like состояния; VMM/VPU/PAE/media bind уже работали, но ISP/PAE IRQ оставались нулевыми. Stock runtime tracing и внешний clean-room Fullhan reference затем сузили missing layer до полноценной ISP sensor lifecycle/registration sequence, вместо дальнейшего широкого sensor или encoder reverse.

`CHAT-008` даёт независимую hardware-localization ветку того же milestone. После восстановления API lifecycle и сравнения с `C4998` обнаружено, что на OpenIPC `ISP+0x08` оставался нулевым, тогда как stock init первой аппаратной записью включает interrupt mask. Запись маски немедленно подняла ISP IRQ. Последующая изоляция отдельных mask bits показала реальный sensor cadence около 25 fps. Это доказало границу: sensor/MIPI → ISP уже живы, а следующий blocker находится в переходе ISP → VPU/PAE.

`CHAT-009` уточняет cold-boot sensor precondition: для GC1054 недостаточно закончить с GPIO5=1. После reboot нужен reset pulse GPIO5 `0 → 1`; без него I2C controller жив, но GC1054 не отвечает. После корректного pulse ID снова читается как `0x10/0x54`, а vendor sensor init возвращается к успешному состоянию. Это делает GPIO5 частью reproducible cold-boot bring-up.


## D6 — Hardware H.264 и граница доказательства

После ISP пришлось отдельно согласовать geometry и encoder path. Когда ISP/VPU/PAE параметры стали согласованы, encoder начал работать с ожидаемой cadence.

Далее был получен H.264 файл:
- Annex-B framing;
- SPS;
- PPS;
- IDR;
- активные ISP/PAE IRQ.

Этого достаточно, чтобы считать hardware encoding path запущенным.

Но `CHAT-001` корректно оставил открытой границу acceptance: capture повторял один descriptor/первый encoded unit. Поэтому нельзя повышать результат до «полностью корректное moving video/image quality». Следующая задача — правильный dequeue/live frame progression, а не повторный базовый bring-up.

`CHAT-009` показывает ранний capture milestone подробно: единый helper поднимает ISP config/VMM, VPU, PAE, bind/start/enable и получает Annex-B H.264 с SPS/PPS/IDR. Сохранён `/tmp/capture.h264` на 50 выборок. Одновременно все выборки указывали на один `virt/len`, а декодирование давало серое/неподвижное содержимое. Поэтому milestone фиксируется как «hardware encoder path работает», но live dequeue/frame progression остаётся отдельным gate.

`CHAT-011` независимо приходит к тому же integrated milestone через другую ветку reverse: sensor init, ISP config/start, VPU/PAE и media stream сведены в один process/one `/dev/isp` lifetime. Capture сохраняет 50 выборок и начало файла содержит Annex-B SPS/PPS/IDR, при растущих ISP/PAE IRQ. Одновременно `virt/len` повторяются, поэтому источник правильно оставляет live dequeue отдельным unresolved gate, а не объявляет moving video доказанным.


## D7 — Dev-loop и stateful driver lifetime

Fullhan media drivers оказались чувствительны к lifetime:
- multi-open/close;
- concurrent ioctl;
- повторному instrumentation;
- dirty state после неудачных экспериментов.

Один экспериментальный preload-thread привёл к полному зависанию target. После этого направление изменилось на один долгоживущий owner process, который должен держать media fd/VMM и принимать команды на capture/status без постоянного переоткрытия.

Параллельно была оптимизирована инфраструктура итераций:
- стабилизирован быстрый SSH startup;
- исправлен PTY path;
- повторное соединение стало переиспользуемым.

Эти изменения не являются media port сами по себе, но резко снижают стоимость каждого последующего эксперимента.

`CHAT-009` даёт прямое доказательство broken lifetime vendor driver: unload ISP/media оставил висячую IRQ registration, и последующее чтение `/proc/interrupts` закончилось kernel Oops. Несколько ISP contexts также давали duplicate handlers. После этого persistent single-owner/daemon стал safety requirement, а не просто удобством. Параллельно SSH dev-loop был разложен по слоям: постоянный Dropbear key на mtd4, persistent `seedrng`, статическая сеть без broken `fw_printenv`, корректный `devpts newinstance` и `/dev/ptmx -> pts/ptmx`.

`CHAT-010` сохраняет отдельную SSH regression после ранних boot-time fixes: каждый новый `ssh/scp` стал занимать около 5–7 секунд. Первоначальная трактовка «это нормальная стоимость KEX» и предложение скрыть задержку через ControlMaster были сняты после напоминания пользователя, что до SSH-правок обычный SCP был быстрым. К концу чата правильный следующий метод — boundary timing TCP connect → SSH banner → KEX → auth; сама причина регрессии в этом источнике ещё не закрыта.

`CHAT-011` дополнительно показывает цену ускорения dev-loop временными image hacks. В процессе SSH/RNG/network оптимизации менялись init ordering, DHCP hook, persistent keys/seed и generated CPIO. К концу ветки пользователь требует отдельной reconciliation заметки: рабочий RAM image не равен финальной OpenIPC архитектуре, все dev-only изменения должны быть либо возвращены, либо перенесены в board-scoped/canonical source перед upstream.


## D8 — Handoff и первые признаки многоагентного workflow

К концу `CHAT-001` контекст одного диалога перестал быть надёжным хранилищем проекта.

Появился handoff, который включал:
- доказанные milestones;
- открытые gaps;
- пути к рабочим артефактам;
- repeatability recipes;
- crash hazards;
- методы stock tracing/static reverse;
- границу между test image и будущей production integration.

Затем пользователь принёс handoff второго агента. Вместо хранения двух независимых истин уникальные сведения были сопоставлены и объединены в один master.

Положительный момент: неизвестный provenance одного runtime artifact был оставлен как gap, а не заменён выдуманным объяснением.

Отрицательный момент: handoff быстро вырос до очень большого размера. Это один из исторических аргументов в пользу нынешней схемы «короткая карта + детализированные приложения», а не одного бесконечного master-документа.

Ещё до финального master handoff `CHAT-007` показывает отдельную форму knowledge transfer: пользователь просит передать новые находки параллельному ISP reverse-agent. Handoff содержит только новую external-reference ветку, локальные artifact paths, confirmed contracts, unresolved names и конкретный следующий priority, чтобы второй агент не повторял уже закрытое.

`CHAT-008` усиливает этот этап двумя практиками. Во-первых, старые рабочие helper'ы поднимаются из локального checkpoint после потери `/tmp`, а не восстанавливаются по памяти. Во-вторых, parallel ISP checkpoint передаётся с явным требованием сверить с текущими findings и не повторять уже закрытый reverse; результат второго агента затем используется как reference, а не как новая независимая canonical ветка.

`CHAT-009` добавляет context-limit handoff acceptance: при приближении лимита диалога пользователь требует большой handoff, но затем уточняет, что narrative недостаточно. Новый агент должен видеть working files/directories, firmware/memory disassembly, stock captures, checkpoints и exact build/transfer/reverse/run recipes. Это ранняя формулировка handoff как reproducibility manifest.

`CHAT-011` превращает handoff в reconciled living master: другой агент уже работает по старой версии, поэтому вместо создания нового документа ему передаётся prompt на обновление существующего master. Новый authoritative верх переписывается под свежие hardware facts, а старые logs/observations сохраняются как historical evidence с явными superseded notes. Отдельно добавляется checklist временных dev workarounds, которые нельзя спутать с final port.


## D9 — Dequeue, grey frame и stock runtime evidence

`CHAT-004` начинается с конкретного незакрытого дефекта раннего H.264: userspace повторно видел один stream descriptor.

Reverse `enc.ko` и `media_process.ko` установил:
- `PAE 0xC0045011` — release текущего encoded stream для encoder channel;
- `MEDIA 4D05` и `4D06` — query одного stream с разным wait policy, а не query/get pair;
- read index продвигается через release path.

После перестановки цикла на `query → copy → release` hardware test дал последовательные descriptors, растущие timestamps и разные CRC. Это закрыло queue-consume defect и впервые отделило «поток действительно движется» от «изображение корректно».

Декодированный поток при этом оказался математически постоянным серым YUV. Дополнительные проверки показали:
- encoder/H.264 framing не является причиной;
- stock-like VPU upscale 1280×720 → 1920×1080 воспроизводится, но масштабирует тот же серый source;
- значит проблема находится выше VPU/PAE.

После этого reverse сместился к ISP lifecycle. Были найдены:
- GC1054 SREG container с `day/night/wlight` profiles;
- `API_ISP_LoadIspParam`;
- `API_ISP_Run` и длинная runtime writer-chain;
- связь scene parameter object с ISP context.

Параллельно stock был загружен как reference runtime. Вместо дальнейших догадок был снят read-only evidence bundle: process/VMM mappings, ISP/MIPI state, ISP parameter data, ioctl trace, clock/interrupt evidence и code artifacts.

Ключевой методический результат этой главы: если проблема остаётся после доказанного transport/encoder path, следующий уровень должен восстанавливаться из stock lifecycle/code/runtime evidence, а не серией несвязанных register pokes.

`CHAT-010` закрывает точную dequeue semantics статически. `PAE 0xC0045011` проходит через handler `0x10370`, копирует 32-bit аргумент и вызывает `pae_enc_stream_release`; сама функция принимает encoder channel `0..7` и при active stream вызывает `media_stream_release`. `4D05` и `4D06` оказались не query/get парой: оба вызывают один `media_query_stream`, где `4D05` — nonblocking query, а `4D06` — query с timeout. Для encoded stream type 4 callback ведёт в `pae_enc_stream_query → media_stream_query → enc_stream_query`. Read index продвигается только release-path через `enc_stream_get`. Текущий livecapture был найден с ошибочным release-before-first-query порядком и исправлен на `query → copy/CRC → release current`. В этом чате исправленный helper ещё не прошёл clean-boot hardware validation из-за отдельной SSH-regression ветки; hardware proof dequeue остаётся в другой ветке/источнике.


## D10 — Persistent owner, hot reload и автоматизация dev-loop

`CHAT-003` заполняет переход между ранним H.264 bring-up и более зрелым source-derived runtime.

Сначала правильная lifecycle-фаза стала видна на `v3.8`: stock-like init, применённый до `ISP_START`, резко улучшил encoded stream и подтвердил, что проблема уже не в базовом transport. Попытка выдёргивать отдельные runtime stages и делать live MMIO rollback позже дала тяжёлые регрессии и один hard hang. После этого массовые live writes были исключены из нормального метода.

Параллельно обнаружились инженерные проблемы самого test loop:
- wrapped PAE descriptors;
- повторное открытие stateful media/ISP;
- owner replacement, требующий reboot;
- ручной U-Boot/TFTP ritual после каждого reboot.

Ответом стала архитектура одного долгоживущего media owner:
- один набор `/dev/isp`/PAE/media fd и VMM mappings на boot;
- ring-wrap handling;
- reloadable `libfhisp_algo.so`;
- дальнейшие ISP runtime изменения без замены owner;
- опасный второй owner запрещён.

Серия `v4.0 → v4.0.4` одновременно показала ценность hardware-proven baseline: несколько новых dequeue/pack/init гипотез дали регрессии, и сравнение с рабочим `v3.8` позволило локализовать обязательный pre-start state и вернуть стабильный transport.

Для reboot-loop была сохранена именованная U-Boot схема:
- `openipc_boot` — проверенный RAM/TFTP OpenIPC path;
- `stock_boot` — штатный flash path;
- OpenIPC сделан development default после проверки.

На стабильном `v4.0.4` впервые стало удобно проверять runtime control через hot plugin. Read-only probes нашли полезную image statistics, а AE v1 доказал замкнутый `statistics → set_intt` feedback. При этом две фиксированные горизонтальные полосы и зелёный оттенок были разделены как разные image-processing проблемы.

Standalone `D0238` полосы не исправил. Это вместе с неудачными одиночными writers окончательно сместило стратегию к восстановлению полного stock lifecycle/writer order, которое затем развивается в `CHAT-002`.

`CHAT-012` добавляет раннюю productionization boundary до более зрелого substrate из `CHAT-003`. В уникальном хвосте пользователь предлагает уже начинать firmware/Majestic, хотя RAW/Bayer ещё не закрыт. Архитектурный ответ: proven boot/rootfs/vendor-module/VPU/PAE/dequeue слои можно productionize параллельно, но stateful Fullhan devices остаются за одним долгоживущим `fh8626_daemon`; Majestic должен получать stream через downstream interface и не становиться вторым `/dev/isp` owner. Это пока design direction, не hardware acceptance Majestic.


## D11 — Source-derived ISP runtime

`CHAT-002` начинается уже после первого рабочего hardware pipeline. Главная инженерная смена — отказ от лечения визуальных дефектов одиночными snapshot-регистрами.

Вместо этого:
- восстанавливается lifecycle `C540C → CB890 → CB970`;
- writers портируются в stock order;
- значения вычисляются из runtime context/profile;
- каждый законченный subset сначала compile-checkится, затем при необходимости проверяется на железе.

Через owner `v4.1.1+` и последовательные runtime stages были подтверждены source-derived writers, H.264 оставался 1280×720@25 и 125 frames / 5 s. Две фиксированные горизонтальные полосы, которые существовали на раннем pipeline, на этом этапе визуально исчезли. Причинность одному конкретному writer не была назначена; отдельно зафиксировано, что standalone `D0238` раньше не помогал.

Зелёный оттенок сохранялся и рассматривался отдельно как незакрытая AE/AWB/Bayer/CCM часть, а не как та же проблема, что горизонтальные полосы.

## D12 — Parallel heavy reverse и authoritative artifacts

Во время дальнейшего reverse обнаружилось, что активный binary `apollo.unpacked` неполон для поздних RW/GOT областей. Workspace был нормализован вокруг полного textual ARM dump; старый неполный artifact и лишние extraction copies были исключены из active work.

После этого heavy reverse был выделен параллельному агенту. Основной агент продолжал canonical runtime/integration, а parallel agent разбирал отдельные функции без hardware tests и без создания второй runtime-ветки.

Ключевые результаты этой фазы:
- shared signed `int16_t` Q7 sine LUT восстановлен как exact 360-entry table;
- `CFEB0/D0238` восстановлен exact и прошёл self-test против stock outputs;
- `D1258` восстановлен exact;
- `D1724` и `D1DB0` доведены до finished/parameterized reverse units с честно оставленными отсутствующими GOT-backed data objects;
- затем отдельно восстановлена значимая часть AE/AWB frontend, включая Bayer permutation и gain/state path; final day-mode estimator оставался незакрыт.

Методическая граница: если data-object физически отсутствует в authoritative source, он остаётся unresolved/parameterized; значения не угадываются.

## D13 — Day AWB mode1 и dual-GC1054 lens architecture

`CHAT-013` продолжает ISP/runtime работу после `CHAT-002`, но уже не на уровне общей архитектуры, а на конкретной current-day ветке.

По AWB подтверждено:
- `CB7B0`, `CA13C` и `CA4F4` используют 9 statistics records (3×3), а не 8;
- live statistics берутся не из фиксированного `VMM+0x48`, а через `ioctl 0x80046905 → base + offset + 0x48`;
- `CA4F4` normal path включает validity gates, Q12 channel ratios, 64-bit channel sums, fallback на current triplet, base gains, median selection `C9E20`, temporal/hysteresis mixing и final normalization;
- normal day path не требует большой неизвестной LUT;
- создан canonical checkpoint `v4.1.9-awbmode1exact`;
- hardware activation специально разделена: сначала `diag/shadow`, commit в AWB registers остаётся отдельным gate.

Параллельно уточнены `C9740 → C949C → C9898`: `D2774` оказался integer sqrt, current day path использует 9 values → mean → sqrt, dirty state обновляется только при изменении. `CDD6C` calibration row зафиксирован как 13×int16 с 12 consumed values. Часть таблиц/GOT-backed state ещё оставалась unresolved.

Отдельная sensor/lens ветка сначала ушла в JXF37/JXF37P из-за наличия 1080p vendor drivers и tuning. Эта гипотеза была полезно опровергнута:
- stock `sensor_probe` конкретной платы выбирает `gc1054_mipi`;
- VI source — 1280×720;
- stock 1920×1080 main stream получается через downstream VPSS/VENC upscale;
- JXF37 I²C target физически не отвечает;
- реальный stock zoom не вызывает новый `Sensor_Create/init/set_fmt`.

Stock zoom trace показывает lifecycle переключения:
- wide/default: Lens ID 0, target1, GPIO4=1/GPIO14=0;
- tele: target2, GPIO4=0/GPIO14=1;
- перед switch stock останавливает только receive path, затем меняет target/GPIO, возвращает mirror/flip и продолжает stream;
- sensor driver остаётся GC1054.

Следовательно, для этой board revision рабочая модель — две физические GC1054 с разной оптикой, а JXF37/JXF37P остаются firmware-supported вариантами других ревизий. Это также объясняет, почему ручной pre-bringup GPIO switch не был эквивалентен stock zoom: правильный switch — lifecycle operation внутри уже работающего persistent owner.

Night mode в этом источнике отделён как следующий слой, а не новый sensor mode: тот же GC1054, но другие scene/style, AE limits, saturation/IR-cut state и night-specific branches writers. Day core должен быть закрыт первым, после чего те же функции проходят по night path.

`CHAT-016` позднее уточняет switch implementation: `zj_switch_lense → service_venc_sensor_switch → D8308`, query/toggle/explicit target semantics, stop/restart VENC, mirror/flip save/restore и отдельные GPIO light/IR-cut controls. Эти детали статически подтверждают D13, но controlled wide→tele→wide runtime capture в этом источнике ещё pending.


## D14 — AWB → CCM coherent runtime и hardware validation

`CHAT-014` — отдельная parallel-agent ветка, которая получает уже доказанные AWB inputs и занимается downstream color path без повторного reverse AE/AWB frontend.

Static reverse:
- `C9F68` не читает напрямую `ISP+0x224/+0x228`;
- downstream coordinate идёт через logical state `ctx+A8/+AA`;
- восстановлены anchor selection/interpolation и state `B0/B1/B2`;
- `CE670/CE764` восстановлены до packing `ISP+0x4C0..+0x4D4`;
- color table определена как 4 anchors × 12 signed 13-bit values; первые 9 образуют Q9 3×3 transform, последние 3 — offset terms.

Offline для `512,512,512 → 544,480,544` предсказано:
- та же anchor pair `3→2`;
- weight меняется примерно `34→43`;
- меняются только отдельные matrix coefficients;
- из шести packed CCM words заметно меняется только `+4C8`.

Первый hardware capture показал важный integration gap: direct `awbmode1 step` меняет `+224/+228/+4BC`, но `+4C0..+4D4` остаются неизменными. То есть custom step обходил stock state transition `CB4F0/CAFC0 → A8/AA → C9F68 → B0/B1/B2 → CE764`.

Для проверки создан coherent diagnostic owner `v4.2.1`. Clean-boot validation дала:
- `A8=3614`, `AA=3614`;
- pair `3→2`;
- weight `43`;
- `B0/B1/B2` меняются в ожидаемом направлении;
- `+4C8: 0x1FD1028C → 0x1FF70266`;
- остальные CCM words не меняются.

Результат совпал с offline reconstruction. Это hardware validation полного AWB→color state path.

Дополнительный важный факт: `awbmode1 restore` вернул AWB gains/state, но CCM остался изменённым. Полный rollback потребовал отдельно восстановить `+4C0..+4D4`. Поэтому будущий experiment protocol должен описывать полный write-set и rollback-set, а не только входной knob.

Попытка hot-replace owner через kill/restart без reboot оказалась небезопасной для vendor sensor/device lifecycle; clean boot остался надёжной границей для смены owner binary.

## D15 — Cross-Fullhan semantic oracle и systemic image-quality gaps

`CHAT-015` — центральная orchestration/research ветка после первых parallel AWB/AE задач.

Внешний FH8852V201 reverse и FH8852V100 vendor libraries дали именованный semantic vocabulary. На instruction/algorithm level были найдены сильные homologs:
- `D2774 ↔ isqrt/ae_isqrt`;
- `D27C4 ↔ easylog2`;
- `D282C ↔ bubbleSorting`;
- `C9DB0 ↔ Awb_WpConvert/wpConvert`;
- `C9F30 ↔ Awb_GetDist`;
- `C9F68 ↔ Awb_GetPos/getCTPos`;
- `CE670 ↔ ccm_ColorCorrection`;
- `CE764 ↔ ccm_ctrl_run`.

Дальнейшее сопоставление controller order и enable-bit lineage дало semantic map позднего runtime:
- `CE430` — BLC;
- `D0E5C` — NR2D;
- `CFD70` — GB;
- `D1DB0` — YC;
- `CFBC4` — Gamma;
- `D2074` — YNR;
- `CEACC` — DPC;
- `CDD6C` — APC/detail/sharpening;
- `CE7D8` — CNR;
- `D0DF4` — LTM;
- `D1258` — Purple;
- `D0238` — LC;
- `CECF0` — FC;
- `D1724` — RGBA-like branch.

Эти имена сами по себе не считались FH8626 proof. Их роль — сократить поиск. Реальные target gaps затем подтверждались по Apollo/current runtime.

Практически важные target findings:
1. `CDD6C/APC` в custom runtime отсутствовал, хотя current day profile должен активно менять detail/edge region `ISP+0x528..+0x574`.
2. `ctx+0x60 >> 12` идентифицирован как live `total_gain`, используемый gain-indexed ISP modules.
3. `C5AE8` соответствует common gain getter, а producer `ctx+0x60` находится в `C73F8`.
4. Custom runtime обновлял лишь часть C949C state и не публиковал stock-like total gain every frame; APC/NR2D/YNR/CNR поэтому могли работать на стартовом/stale gain.

Это дало конкретное объяснение «мыльной» картинки: missing APC плюс stale gain-dependent tuning вместо абстрактной нехватки «ещё каких-то регистров».

External same-SoC research также дал useful architecture hints:
- official FH8626 adapter использует native encoder stream timestamps, что делает их правильным источником cadence measurement;
- отдельный FH8626V100+GC1054 runtime свидетельствует о sensor 720p25 и downstream 1080p path с frame control, то есть sensor cadence и encoder/output cadence могут различаться.

Последнее является внешним corroborating evidence, а не заменой target hardware proof.

Важнейший методический результат: дальнейший reverse неизвестной Apollo-функции должен сначала проверять наличие именованного Fullhan homolog, но вся target-specific arithmetic/state/MMIO всё равно доказывается на FH8626.

## D16 — Полный AE loop и day/night parity

`CHAT-016` — persistent Agent 1 branch. Task 1 закрывает точную структуру `C9740/C949C/C9898`, после чего Task 2 расширяется до полного stock AE/brightness control loop.

Итоговая функциональная цепочка:
`statistics → C757C → target/error → C6D04 history → C90D4 limits → C7C3C gate → C883C/C8134 → C7058/C6E64/C72A0 → immediate/deferred actuator → C6AC8 → GC1054 callbacks → sensor registers → next frame`.

Ключевые контракты:
- `C6D04` — 60-frame rolling error history;
- `C7C3C` — hysteresis/state gate;
- current GC1054 day выбирает controller branch `C7EB0 → C883C`;
- `C6AC8` — deferred actuator queue с dirty semantics;
- `C72A0` идентифицирован как anti-flicker-related sensor control;
- `C9898` после уточнений считается publication/status tail, а не exposure controller;
- `D0DF4/D0528/D0630` отделены как самостоятельный adaptive ISP/LTM-like statistics block.

Для sensor boundary был отдельно извлечён и дизассемблирован stock `libgc1054_mipi.so`. Восстановлено:
- `set_intt` до GC1054 integration registers `0x03/0x04`;
- `set_gain` через gain ranges и registers `0xB6/0xB1/0xB2`, включая page-4 `0x40`;
- VTS/frame-length callback;
- callback table identity на live stock process.

Completion audit был важен: первоначальный Task2 v1 оказался лишь 70–80% исходного scope. Работа продолжилась через v2/v3/v5, пока static material не был исчерпан и оставшиеся gaps не были переведены в конкретные runtime requests.

Day path получил полный numerical replay. Night path потребовал отдельного read-only stock capture:
- `ctx+0x38 = 745`;
- `ctx+0x3B = 32`, что даёт effective minimum integration 2;
- `ctx+0x30 = 95` → target 1520;
- center-only statistics branch `ctx+0x3D=1`;
- live measured value 1547 → error +27;
- controller gate не делает нового actuator write, current intt/gain остаются 31/64.

Sensor callback capture доказал, что второй zoom в этом состоянии использует тот же `libgc1054_mipi.so`. Чтобы не спутать zoom profile с night profile, был использован исходный `sensor_gc1054_mipi.bin`; SREG audit показал `day/night/wlight`, а live stage2 совпал именно с night payload. Так историческая пара `745/2` получила static + runtime provenance.

После AE Task2 тот же specialist перешёл к Task3 по lens/peripheral orchestration. Static reverse уточнил `zj_switch_lense → service_venc_sensor_switch → D8308`:
- target 1 = wide, target 2 = tele;
- `-1` query, `0` toggle, `1/2` explicit target;
- wide GPIO4=1/GPIO14=0, tele GPIO4=0/GPIO14=1;
- активные VENC channels останавливаются, mirror/flip сохраняются, GPIO target меняется, затем state восстанавливается и VENC запускается снова;
- sensor create/init/set_fmt при runtime switch не повторяются;
- day/night/light остаются отдельной state machine;
- IR LED GPIO25, white LED GPIO23, IR-cut GPIO18/60, mute1 GPIO24.

Эта lens-switch часть в `CHAT-016` остаётся static-complete до controlled hardware capture; её не следует повышать до нового hardware-pass.

## D17 — Image-detail APC/NR3D/LTM и runtime RW/GOT closure

`CHAT-017` — persistent Agent 2 lane. Ранний scope был `CDD6C/D0B2C/GOT table identity`, затем в той же specialist thread задача расширилась до полного image-detail/sharpness/NR/demosaic Task 2.

Первоначальный результат был объявлен завершённым слишком рано. Requirement audit показал только ~55–65% большого Task 2. После продолжения были доказаны semantic chains:
- `imgset_sharpness → DF084 → APC config → ctx+0x314/+0x320 → CDD6C → ISP+0x528..+0x574`;
- `D0E5C = NR2D`;
- `D0FEC = NR3D`;
- `D2074 = YNR`;
- `CE7D8 = CNR`;
- `D1258 = Purplefri / anti-purple-fringe`;
- `D0DF4/D0630/D0B2C = LTM/local tone mapping`;
- `CFD70/GB` отделён от demosaic.

Статический RX-only Apollo не содержал initialized RW segment/GOT, поэтому exact identity APC/LTM/NR3D/D1DB0 tables была физически недоказуема из одного ARM dump. Вместо повторного blind reverse был снят read-only runtime RW region, а затем sparse `apollo_full_runtime.bin`, сохраняющий реальные mapped VA и unmapped holes.

В связке `apollo_unpacked_ARM_full.txt + apollo_full_runtime.bin` закрыты:
- APC/CDD6C current day: live ctx, four GOT pointers, current selectors/row1 tables, 13-halfword consumer correction и mapping ISP registers;
- LTM/D0B2C: GOT `+BE4` и `+948`, current rows 23/0;
- NR3D/D0FEC: GOT `+98C`, preset rows и live dispatch condition (`ctx+0x11AC=1`);
- D1DB0: GOT `+D14` и runtime coefficient vector;
- user sharpness baseline arrays, совпадающие с live ctx;
- corrected D0630 polynomial including quadratic term.

Final Agent 2 Task2 для current GC1054 day был завершён с reference/selftest и owner-parity audit. Integration priority: APC/CDD6C → active NR3D/D0FEC → dynamic LTM D0630/D0B2C.

Отдельный методический результат: static executable disassembly и runtime mutable state — разные evidence layers. Runtime heap не следует массово дизассемблировать как ARM code; mutable pointers/tables нужно связывать с доказанными ARM consumers.
