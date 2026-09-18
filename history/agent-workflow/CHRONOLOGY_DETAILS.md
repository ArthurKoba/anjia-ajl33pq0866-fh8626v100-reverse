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

## D18 — Controlled wide → tele → wide runtime validation

`CHAT-018` — расширенный export той же Agent 1 conversation, чей префикс перекрывает `CHAT-016`. В аудит добавлен только уникальный tail.

Controlled runtime capture подтвердил:
- lens target sequence: `1 → 2 → 2 → 1`;
- lens switch сам по себе не меняет day/night profile;
- AE context и sensor callback table остаются общими;
- отдельного AE state на tele не создаётся;
- AE history продолжается через switch;
- tele при том же profile уходит к существенно большей integration/gain;
- после возврата wide integration/gain возвращаются близко к исходному wide state;
- статически доказанный transient `intt=64/gain=64` слишком короткий для ручного `dd`, но это не блокирует implementation contract.

После validation область оформлена как implementation-ready Task 3 supplement: runtime sequence, startup-vs-runtime distinction, GPIO control table, day/night/light/audio boundaries и safe integration order.

Новый heavy reverse не потребовался; remaining optional validation — только измерение transient/frame-gap с более точной instrumentation.

## D19 — Current-day reverse convergence и runtime integration plan

`CHAT-020` — центральная orchestration branch, где результаты parallel agents перестают существовать отдельными handoff'ами и интегрируются в одну normalized system model.

К этому моменту current-day GC1054 knowledge включает:
- full AE/brightness loop до sensor registers и day/night profile contracts;
- AWB statistics/mode1 и coherent AWB→C9F68→CCM propagation с hardware validation;
- APC/detail, active NR3D, LTM and related runtime-table identities;
- dual-lens switch implementation + hardware validation;
- late/heavy ISP writers, VPU/PAE/H.264 path;
- Cross-Fullhan semantic map как reference, не target proof.

Практический вывод центральной ветки: remaining current-day work в основном implementation/hardware parity, а не broad reverse.

Roadmap нормализован примерно в порядке:
`AE/live total gain → sensor AE → CDD6C/APC → D0FEC/NR3D → D0630+D0B2C LTM → D1DB0 → cadence`.

Night/1080p/WDR/другие later modes отделяются от current-day parity, чтобы не размывать основной integration path.

Технический state при этом не повышается автоматически до hardware parity: часть блоков exact/reverse-confirmed, но ещё отсутствует или feature-gated в current owner. `IMPLEMENTATION_REGISTRY`/module matrix явно разделяют «разобрано» и «портировано/проверено».

Эта же ветка интегрирует Cross-Fullhan oracle непосредственно в runtime planning: `ctx+0x60 >> 12` закрепляется как live total-gain semantic, а missing per-frame publication рассматривается как конкретный owner gap для gain-dependent APC/NR modules.

Handoff на этой фазе проходит automated health checks и содержит current state, source/reference/evidence/tooling, что делает следующего оркестратора способным стартовать от integration roadmap без повторного wide reverse.

## D20 — Stock evidence campaign и normalized runtime dataset

`CHAT-021` — отдельная acquisition phase после master-handoff convergence.

Canonical steady imaging states:
- WIDE_DAY;
- TELE_DAY;
- WIDE_NIGHT;
- TELE_NIGHT.

Captured transition classes:
- wide↔tele;
- automatic day↔night;
- white-light coupled day-path transition and return;
- talkback full cycle;
- siren one-shot cycle;
- manual PTZ pan/tilt.

Additional stock contracts:
- HTTP snapshot `/snapshot.jpg` → VPSS channel 1 → 640×368 JPEG, quality 37;
- audio input initialized at boot; talkback/siren use AO/speaker path and GPIO24 mute;
- white illumination is coupled with IR-off/IR-cut day optical path rather than independent light-only state;
- PTZ application path observed for both pan and tilt;
- full stock boot UART retained as primary startup evidence.

Sensor/state evidence:
- GC1054 bus/address confirmed;
- page-0 state captures across four imaging states;
- exposure registers provide clear state differences;
- lens/day-night targeted transitions include selected sensor registers and small Apollo RAM regions.

A major methodological correction came from heap analysis: independent full heaps changed ~49–52% because of live buffers/allocator/temporal activity. Those snapshots remain provenance, but semantic transition analysis uses synchronized targeted regions around known Apollo state objects. The transition deltas are far smaller and causally interpretable.

Acquisition also exposed unsafe/invalid approaches:
- storing a large physical-RAM dump in stock RAM-backed `/tmp` caused OOM and killed Apollo;
- VMM userdev mappings are not ordinary sequential files and their mmap offset must not be assumed to be physical DRAM;
- `/proc/PID/mem` works for ordinary process heap but not device-backed mappings.

Static stock preparation was expanded for selected modules/libraries, each with binary provenance, file/readelf/symbol/relocation/string/disassembly material. Existing authoritative Apollo/JXF37 representations were deliberately not regenerated.

Final evidence organization distinguishes:
- canonical captures;
- superseded/mislabeled raw captures retained only for provenance;
- corrected metadata;
- available vs unavailable trace/PCAP capabilities;
- remaining optional gaps.

The stock evidence is packaged as a delta over stable master rather than duplicating the complete handoff.

## D21 — Production catch-up и hardware parity session

`CHAT-022` — production implementation branch, стартующая от master v19 с целью сократить разрыв `reverse knowledge → working implementation`.

Source catch-up реализовал или восстановил:
- live total-gain publication;
- physical GC1054 integration/gain commits и rollback;
- C883C/C7EB0 AE state-machine pieces включая поздний D26D0/action6 tail;
- AWB→C9F68→CCM;
- APC;
- NR3D;
- dynamic LTM;
- YC/detail pieces;
- wide-default dual sensor + runtime switching;
- reloadable `libfhisp_algo.so`;
- per-feature gates, status, rollback and capture tooling.

Hardware testing по одному delta подтвердил:
- WIDE native H.264 capture;
- live gain publication;
- APC runtime execution/rollback;
- NR3D runtime execution/rollback;
- LTM runtime execution/rollback;
- manual sensor integration commit + rollback;
- manual sensor gain commit + rollback;
- AWB→CCM register/gain change + restore.

Automatic AE не повышен до PASS: runtime `API_ISP_GetAeStat=0`/provider unavailable оставил свежую statistics path неподтверждённой.

Dual-sensor integration сначала систематически ломалась на TELE: GPIO selection проходил, но SensorFmt давал sensor-register write failures. Несколько гипотез были hardware-отброшены. Надёжный discriminator — stock `sensor_probe` — показал, что TELE физически не виден до release общего sensor reset.

Root cause:
`GPIO5 LOW → media modules load → GPIO5 HIGH`.

После включения этого stock-like cold-boot sequence оба GC1054 становятся видимыми, WIDE/TELE переключение реально даёт изображение, WIDE остаётся product default. Это board bootstrap contract, а не runtime workaround.

Owner lifecycle продвинулся только частично. Experimental `shutdown` доказал возможность завершить здоровый owner и заново запустить процесс без полного reboot, но exact stock teardown не восстановлен. При broken/blocking media state control FIFO может зависнуть вместе с stream retrieval. Поэтому hot replacement остаётся production debt.

Цветовой эксперимент дал важный отрицательный результат: AWB/CCM registers и gains меняются сильно, но постоянный green cast визуально почти остаётся. Следующая focused задача формализована как:
`RAW format/bit depth → Bayer order/permutation → VI/ISP input → RAW packing → black level → demosaic/early color → затем CCM`.

В конце сессии source snapshot явно классифицирован как integration/diagnostic lineage, не готовое production tree; orchestrator должен провести source consolidation и сохранить hardware findings отдельно от временных diagnostics.

## D22 — OEM-identity AJL33PQ0866, CF26/SM и donor search

`CHAT-024` — внешний research thread, стартовавший с повреждённой наклейки/QR и визуального поиска корпуса.

Раннее visual matching было недостаточно надёжным. Исследование стало существенно сильнее после target-local evidence:
- application/internal model: `AJL33PQ0866`;
- firmware string: `YGT.AJL33PQ0866-v230920.1051`;
- reported SDK marker: `3051380`;
- physical optics: 3.6 mm wide + 12 mm tele;
- dual physical cameras / PTZ / 8 front emitters;
- PCB silkscreen уверенно содержит `CF26`, `SM`, `V1.0` и date 20210401; средняя часть строки читается неоднозначно.

External correlation затем связывает target с широким CF26/SM400/AJ Fullhan cluster:
- CF26-54SM+400-PL / CF26-37SM400 relatives;
- `AJ-SM-FH8626V100` software identification у родственных devices;
- официальный documentation/reference с FH8626V100, dual 1054, 3.6+12, 4 IR + 4 white and CareCamPro-like stack;
- соседние internal `AJL33*` model IDs.

Найденные donor/research sources различаются по назначению:
- максимально близкие hardware relatives;
- devices с реальными SPI dumps;
- generic FH8626 cases с dump→repack→root/telnet;
- firmware families с большим количеством RTSP/userspace discussion.

Важная evidence boundary:
- own PCB/software/dump = target truth;
- external model = relative/donor/semantic source;
- сходство корпуса или SoC не доказывает binary firmware compatibility.

Итоговый search fingerprint:
`AJL33PQ0866 + CF26/SM400 + FH8626V100`,
с расширением на соседние `AJL33*`, `AJ-SM`, firmware-version and PCB aliases.

Эта линия важна не для переименования проекта в конкретную retail SKU, а для поиска более близких firmware, less-stripped binaries, configs, sensor/ISP data, updater mechanisms и SDK artifacts.

## D23 — Cross-platform gap hunter и targeted reverse leads

`CHAT-025` — persistent Agent 2 branch, теперь явно назначенная External Research / Cross-Platform Gap Hunter.

Первый audit уточняет, что физическая AE математика уже существенно закрыта; remaining problem разделяется на production implementation и live fresh-statistics bridge. Главные P0 research gaps:
- exact owner lifecycle/re-init/resource ownership;
- ISP statistics freshness/epoch/cadence;
- RAW/Bayer/MIPI/early-color contract.

External/source audit выявляет архитектурные риски текущего owner:
- algo tick связан с encoded descriptor dequeue, а не доказанной stats-ready epoch;
- HOLD не гарантирует algorithm quiesce;
- plugin reload разрушает рабочий plugin до полного candidate init;
- configuration changes не имеют systematic generation invalidation;
- diagnostic AWB write не равен production AWB→CCM transaction;
- partial VMM init может оставлять resource tail;
- full-word MMIO ownership не везде доказан;
- control/plugin surface слишком широк для production.

Второй targeted pass находит:
- официальный exact-FH8626V100 adapter/source contract с lifecycle API и ожидаемым Fullhan SDK tree;
- официальные downloadable FH8626V100 SDK/toolchain package names, честно отмеченные как DISCOVERED_NOT_ACQUIRED;
- independent FH8626 runtime evidence: sensor/VI около 25 fps, encoder около 16.66 fps — encoded dequeue не является доказанным sensor/statistics epoch;
- close Fullhan runtime с той же advanced-ISP revision, где day/night transaction включает scene/profile, fps/frame-height, AE bounds, orientation and encoder restart;
- отсутствие публичной готовой FH8626 OpenIPC target/backend в найденных current sources.

Главный proposed omission:
`frame/stat-ready → immutable snapshot → algorithms → staged transaction → frame-boundary commit → affected-frame publication`.

Другие targeted leads: distinct pause/quiesce states, rebuild AE limits after timing change, early BLC/CFA checks before CCM, generation invalidation and dependency-ordered VMM free.

Все external findings остаются semantic leads. Exact offsets/MMIO/layout/ioctl truth должен подтвердить Agent 1 на FH8626.

## D24 — Late Agent-4 watchdog/peripheral corpus

`CHAT-026` — partial export. Earlier Agent-4 messages are explicitly unavailable, so the audit records only the consolidated technical state visible in the surviving tail and does not reconstruct the missing investigation.

Visible consolidated watchdog findings:
- `/dev/watchdog` is owned by stock Apollo;
- stock `wdt_stop` closes the watchdog fd and `wdt_start` reopens it;
- built-in platform driver is identified as `fh_wdt`;
- watchdog MMIO window and IRQ were localized;
- kernel symbols include watchdog pause/resume and PMU restart paths;
- `fh8626v100_restart` reaches the PMU restart path.

Prepared workstation evidence includes the exact stock kernel image/decompressed body/full ARM disassembly, U-Boot/bootstrapping reverse material and watchdog runtime acquisition. These are explicitly classified as heavy reverse artifacts, not ordinary working handoff material.

The same surviving summary preserves two peripheral status decisions:
- stock human detection uses a local object-detection path with VPSS input and retained model files; OpenIPC adapter/reimplementation remains future work;
- PTZ software/backend is treated as closed for current acceptance, while physical actuator validation can be deferred and must not return to the current blocker list solely because the actuator is not connected.

Because the original detailed Agent-4 conversation prefix is missing, confidence here applies to the summarized status, not to a reconstructed command-by-command chronology.

## D25 — Evidence-directed final reverse closure

`CHAT-027` follows Agent 1 from master-v23 static closure through multiple later evidence/reverse waves.

### Static exhaustion and evidence decomposition

The first broad pass classifies most camera functionality to implementation-ready depth:
- dual GC1054 startup/runtime switch;
- physical AE path from prepared statistics to GC1054 registers;
- early RAW/VI/ISP numeric selectors;
- main/sub/analytics video roles;
- audio capture/playback boundaries;
- illumination/IR-cut;
- software PTZ boundary;
- lifecycle architecture.

Instead of continuing generic disassembly, remaining uncertainty is normalized into E1–E7:
- E1 teardown/re-init/resource release;
- E2 statistics producer/epoch/ownership;
- E3 physical RAW/CFA/packing/orientation;
- E4 DAY/NIGHT/WLIGHT transaction;
- E5 IQ publication/atomicity;
- E6 physical PTZ;
- E7 narrow lifecycle/cadence/history checks.

Agent 2 external leads refine these gaps but are not promoted to target truth.

### Agent 4 feedback and second-pass closure

New target evidence later closes or sharply narrows the boundaries:
- controlled lifecycle and same-boot reacquisition become target-observed rather than inferred;
- statistics epoch/double-buffer publication becomes a ~25 Hz live contract;
- RAW10 1280x720 and physical CFA orientations become measurable target facts;
- scene failure semantics and staged IQ publication are characterized;
- software/PTZ ioctl boundary is pinned while physical actuator semantics remain hardware-only.

Agent 1 then revisits the static corpus only at the exact affected functions. Important corrections include:
- `CED28` is gamma/LUT candidate→active copy/commit, not a generic CCM double-buffer;
- stock SIGTERM is a fatal path distinct from the normal service deinit chain;
- userspace stats provider chain is reconstructed around `C6934→C6960→C6C00→C73F8`;
- WLIGHT is a color/day-style low-light scene, not monochrome NIGHT.

### Watchdog and human detection

A separate heavy kernel/U-Boot corpus allows watchdog closure:
- DesignWare watchdog register/timeout model;
- userspace open/feed/close behavior;
- magic-close nuance;
- PMU pause/resume and restart;
- `fh8626v100_restart → fh_pmu_restart` reset chain.

A crucial correction is preserved: stock `wdt_stop` does not simply disable hardware; it stretches timeout, performs magic close and releases the fd.

Human detection is reduced to an implementation-facing contract:
- VPSS Y8 input around 640x368 / visible 640x360;
- <=5 fps path;
- OBJDETECT lifecycle;
- head/shoulder and person model families;
- bbox/confidence result layout;
- event callback routing;
- later refinement of borrowed frame ownership and `human_interval` special behavior.

Unknown vendor names remain numeric instead of being invented.

### Color/HAL and persistent green cast

After physical CFA evidence, persistent green can no longer be blamed on an unknown Bayer order alone. The strongest implementation-side findings become:
- incomplete mode1 AWB state chain before full C9F68/CCM publication;
- unsafe whole-word writes to shared ISP state;
- missing generation/invalidation discipline across lens/orientation/profile;
- BLC/GB observed as present but numerically inactive in a captured DAY state;
- YC `D1DB0` is a later luma/chroma/user-control block rather than an early CFA root cause.

The color path is organized as one contract:
`GC1054 RAW10 → CFA/RMF → VI/ISP format → BLC → CFA/demosaic → early gains/GB/FC → AWB → C9F68 → CCM → YC → gamma/LUT`.

### Illumination, audio, dev_ctrl and optional ISP

Illumination is closed to implementation-ready depth:
- IR/white GPIO and IR-cut lines;
- PWM allocation/on/off/level;
- modes 0–4;
- LDR hysteresis and dwell behavior;
- DAY/NIGHT/WLIGHT integration;
- stock shutdown does not guarantee a physical safe-off lamp state.

`dev_ctrl` is unpacked offline from its packed executable and classified as a board/service daemon for storage/network/GPIO/MTD/env/shell/reboot, not media/ISP/watchdog ownership.

Late source work moves into optional archaeology: LSC, WDR, GME and remaining writers. The chat ends before the final WDR/GME cleanup is complete; a previously assumed WDR mapping is corrected toward LTM and the true WDR controller is still being localized.

### Methodological outcome

The important historical result is the closure loop itself:
`static exhaustion → named evidence request → acquisition specialist → target evidence → narrow static revisit → implementation contract`.

This replaces broad reverse as the normal next step.

## D26 — OpenIPC product strategy и Divinus-first path

`CHAT-028` is a large central-orchestrator thread with substantial overlap with earlier handoff/evidence sources. The unique late contribution is a productization strategy after Agent 1–4 convergence.

The historical research in this chat separates three concerns:

1. **FH8626 hardware/media backend**
   - sensor/bootstrap;
   - ISP/VENC/audio;
   - lifecycle/ownership;
   - board controls.

2. **Generic capability/control boundary**
   - stream;
   - lens;
   - night/illumination;
   - audio capture/playback;
   - PTZ;
   - ISP controls.

3. **Frontend**
   - Divinus;
   - Majestic;
   - minimal RTSP/other consumer.

The chat's source-level survey treats Majestic as a mature but closed component whose exact FH8626 backend cannot simply be assumed. Divinus is historically evaluated as open and extensible but less feature-complete: streaming/web/audio-capture foundations are attractive, while generic talkback, PTZ, multi-lens and broad ISP controls require extension.

This leads to a staged port:
- **Stage A:** keep the known single hardware owner and expose encoded H.264 through a sidecar/external-source contract to Divinus;
- **Stage B:** only after lifecycle/ABI confidence, move ownership into a native FH8626 HAL if that remains desirable.

The same thread separates **engineering bring-up** from **upstream-clean productization**. Locally supplied vendor dependencies can help establish hardware behavior, but official upstream needs a defensible source/provenance/build path.

A related external-research lane searches for FH8626 SDK/BSP traces and downloadable OEM firmware. Its purpose is not arbitrary flashing; extracted kernel/modules/SDK strings are compared against the target heavy corpus to establish lineage and reusable provenance. Same-SoC firmware is explicitly not treated as board-compatible by default.

The chat also clarifies historical dual-sensor attribution: early `Agent 3 / Task 3` reverse and later `Agent 3 OpenIPC` are different agent waves. The actual hardware root cause GPIO5 sequencing was found during a later integration/hardware session.

## D27 — Agent 3 OpenIPC productization

`CHAT-029` is the primary Agent 3 productization branch.

Early research establishes two boundaries:
- FH8626V100 must not be modeled as a renamed FH8852 target;
- absence of a public Majestic/FH8626 backend and clean SDK provenance are real upstream constraints, but they do not have to stop engineering bring-up.

Agent 2 findings are consumed as production constraints: statistics epoch is separated from encoder dequeue, lifecycle/unwind is explicit, plugin reload should be candidate-first, and generation invalidation is required.

The implementation work then produces source-level pieces:
- `fh8626-media-runtime` lifecycle/state model with bounded frame leases and generation invalidation;
- explicit statistics-driven scheduling contract;
- fixed byte-level sidecar ABI to avoid native-struct padding/alignment assumptions;
- Annex-B parser;
- RTP single-NAL/FU-A;
- loopback RTSP E2E;
- read-only `fh8626-abi-probe`;
- read-only device evidence pipeline;
- source-only Buildroot/OpenIPC packages and staging metadata;
- owner-side publisher contract that keeps exactly one VENC lease owner;
- Divinus external encoded-source adapter/RFC;
- separate engineering-bringup and upstream-clean profiles.

The intended early architecture is:
`FH8626 owner → copy encoded frame → sidecar → open frontend/RTSP`.

This intentionally postpones native streamer ownership of ISP/VENC until target lifecycle and ABI are mature enough.

The source also contains an important packaging/provenance incident. A repacked V2 archive contains only four service documents while the README references many missing implementation directories. After the user catches this, a full package is reconstructed. Because the original apply-checked firmware patch cannot be reproduced byte-for-byte, its verification status is correctly downgraded to `RFC/rebase-required`.

A later lens-switch package isolates the early Agent 3 implementation from a different Agent 7 branch after the user rejects cross-branch contamination. The retained transaction follows the already known stop-VENC → GPIO switch → settle → restore orientation → restart VENC → 64/64 reset contract, while startup preparation remains separate.

## D28 — Native Linux platform bring-up

`CHAT-030` is a partial export from the Agent 5 kernel lane; the missing earlier prefix is not reconstructed.

Visible current state:
- hardware-proven: machine, interrupt controller, timer, UART0, SPI0/NOR/MTD, reboot, watchdog, GPIO0/1, I2C0/1/2;
- Ethernet source corrected for the exercised PHY contract, hardware retest pending;
- RTC registered but still carrying a real target defect;
- offline reverse/source work prepared for pinctrl, SADC, EFUSE, UART1/2, PWM0, USB/DWC2, SDIO/MMC, SPI1, DMA/AES, audio, PMU helpers and native defconfig;
- media/VMM and the upper-memory policy are deliberately deferred as the final high-risk layer.

Two machine-level findings matter for the future source architecture:
- stock board initialization is not a single unconditional static device array: bootargs influence pinctrl and SD platform-data, then 22 or 23 devices are registered and one SPI board-info is added separately;
- stock early init executes `fh_pmu_init()` followed by `fh_pinctrl_init(0xFE090080)`. Earlier bring-up could skip native pinctrl temporarily, but final kernel architecture cannot.

The source also records a workflow correction. Because the agent lacked direct write access to the user's WSL Linux tree, it generated many independent numbered archives. This was abandoned in favor of one cumulative offline line and one later WSL runner/build/report boundary.

The user additionally rejects Python as an unnecessary packaging/generation layer for ordinary shell/tar work. The resulting tool discipline prefers native shell tools unless Python adds real analytical value.

## D29 — Divinus V11 hardening

`CHAT-031` continues Agent 6 after context rollover. The current owner is intentionally not integrated because the handoff may contain a stale/missing owner version; that dependency is deferred until the owner lane stabilizes.

Independent Divinus-side work identifies and prepares fixes for:
- raw H.264 behavior incorrectly coupled to `mp4_enable`;
- false `running:true` after FH86 source initialization failure;
- WebUI browser-preview availability when fMP4 is unavailable;
- sidecar partial-frame/discontinuity handling;
- generation wrap/session reset;
- Unix socket cleanup and deterministic init failure;
- transport torture cases such as oversized payload, partial header/payload and generation reset;
- runtime API attempts to re-enable hardware-owned features after startup config had rejected them;
- target temperature/status calls without a provider.

The candidate keeps hardware ownership fail-closed. Frontend-side MP4 muxing remains allowed because it consumes encoded H.264 rather than touching ISP/sensor hardware.

MJPEG is clarified as a provider problem:
Divinus already has server-side JPEG/MJPEG surface, but FH86 external-source mode only has H.264. The intended future path is:
`single FH8626 owner → hardware JPEG producer → Divinus JPEG provider → snapshot/MJPEG`.
Software H.264 decode→JPEG re-encode is intentionally not chosen as the primary solution.

A WSL runner is prepared to check repo identity/worktree, apply patches, run stream/WebUI/publisher regressions, host build, ARM OpenIPC-musl build, and only commit after complete PASS. In this historical source none of those authoritative WSL results have occurred yet; status remains `PENDING_WSL`.

The same chat is also a major workflow-quality event. Repeated command-format, SHA and Explorer mistakes are eventually captured in a dedicated critical protocol and retrospective rather than being left as conversational corrections.



## D30 — Agent 7 release-regression и forensic rollback

Источник: `CHAT-032`, 2026-08-30.

Поздняя Agent 7 ветка попыталась сразу свести release-runtime: single-owner lock, stream/stats workers, AE scheduler/provider boundary, AWB mode1→CCM, IQ manager, lens transaction, illumination/audio/watchdog hooks. Host selftests и full host linkcheck проходили, затем WSL дал корректный ARM/EABI5/musl binary.

Target acceptance показал другую картину. Bootstrap до ISP/VPU/PAE/VENC проходил, но startup/lens/image behavior расходился с physical ground truth. Наиболее важная ошибка — перенос authority с физического selector/FOV на software `current_target`: лог мог сообщать успешное переключение без изменения оптики. После этого поверх regression появились wide-only, force-write и pinmux hypotheses.

Даже возвращение части Agent 3 sequence не восстановило target behavior. Пользователь остановил release line, потребовал clean-boot retest и затем аварийный handoff. Итоговый branch сам классифицировал R13–R18 как `quarantine/do-not-merge` и разделил:
- hardware/ASM findings, которые ещё полезны;
- candidates, требующие повторной проверки;
- опровергнутые hypotheses;
- process failures, которые привели к regression.

Это важная historical boundary: успешный host/ARM pipeline больше не трактуется как близость к product parity; release-код обязан сохранить exact hardware-proven invariants и проходить независимый physical acceptance.


## D31 — Codec/ghosting reverse и integration contracts

Источник: `CHAT-035`, 2026-09-05—06.

Большой статический проход отдельно восстанавливает encoded/media область. H.264 RC разделяется на публичные режимы приложения и внутренние wire modes; corrected mapping и field semantics снимают старые противоречия `mode1 AVBR/CBR`. Realtime RC ioctl, force-I и cold-init перестают смешиваться. Для MJPEG/JPEG reverse закрывает query/release ownership, shared queue semantics, quant tables и shutdown hazards.

Параллельно разбирается BGM как motion-detection path, NR3D как отдельный temporal filter/ghosting factor и их interaction boundaries. Важная корректировка процесса: первоначальное «ghosting 5/5 complete» после host tests оказалось преждевременным. Full-owner integration audit нашёл дополнительные ABI/lifecycle defects и заставил перейти к шестиступенчатому closure cycle.

Источник также добавляет geometry/upscale implementation candidate и full Stage1 review: analytical agent уже не только описывает reverse, а пишет модульные C/H candidates, tests и exact integration notes. Пользователь при этом жёстко отделяет эту работу от target integration: подключение к актуальному owner/firmware и hardware acceptance выполняет другой агент.

Технически это переход от reverse documents к reusable source contracts, но без ложного повышения до hardware-proven production.


## D32 — Post-V2 standalone closure и canonical workspace recovery

Источник: `CHAT-036`, 2026-09-08—11.

После глубокого codec/ghosting reverse standalone слой последовательно закрывает BGM proc/capability grammar, exact bind/unbind, отдельный VMM fd и allocation lifecycle, partial-ownership recovery, NR3D cold/restart semantics, realtime H.264 RC и общую `HOT_OK / RESTART_REQUIRED / RECOVERY_REQUIRED` transition policy. Host strict/sanitizer/analyzer suites проходят; target integration по-прежнему вынесена за границу этапа.

Одновременно source вскрывает process debt, который уже влияет на техническую достоверность: поздние docs описывают код, которого нет в текущем C/H; parallel ACTIVE trees содержат разные наборы; restore из transport archives создаёт новые слои. Поэтому значительная часть работы превращается в evidence/source reconciliation.

К концу периода одна canonical directory получает текущий standalone, reference/deep reverse corpus и двухуровневый index. Historical task documents остаются provenance, но active top-level сокращается до минимального current state. Важная граница: scripts проверяют integrity, но не заменяют чтение документов агентом.

Это software/knowledge milestone, не hardware acceptance.
