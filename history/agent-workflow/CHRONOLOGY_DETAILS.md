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

