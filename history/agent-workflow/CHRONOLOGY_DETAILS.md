# Подробная история реверса и портирования FH8626V100

Этот файл дополняет `CHRONOLOGY.md`. Основная хронология остаётся короткой; здесь сохраняются только детали, объясняющие важную развилку, неудачный подход, метод доказательства или инженерный контракт.

Источник первого блока: `CHAT-001` (примерно 2026-08-24 — 2026-08-26).

## D1 — Первичная разведка и recovery baseline

Стартовали с boot/UART логов и factory firmware. Быстро выяснилось, что камера имеет два переключаемых lens/sensor target, а stock U-Boot даёт достаточно функциональности для network RAM boot.

Вместо ранней записи flash сначала:
- сняли полный NOR dump;
- восстановили U-Boot environment и доступ к интерактивному bootloader;
- подтвердили TFTP path;
- сохранили stock layout как recovery/evidence baseline.

Практический урок: первый milestone порта — не streamer, а безопасный возврат к известному состоянию и возможность загрузить тестовый код без записи flash.

## D2 — OpenIPC RAM boot

Первый OpenIPC запуск был намеренно гибридным:
- stock FH8626 kernel;
- внешний OpenIPC initramfs;
- загрузка через U-Boot/TFTP.

Это позволило аппаратно доказать OpenIPC userspace, Ethernet, MTD и shell до появления native OpenIPC kernel.

Разделение kernel/userspace оказалось полезным: неизвестность «работает ли вообще Buildroot/OpenIPC на CPU» была закрыта независимо от более сложного media/kernel port.

## D3 — Buildroot и чистый OpenIPC baseline

Первые сборки выявили не FH8626-specific, а workflow-проблемы:
- Buildroot плохо переносил Windows-mounted filesystem semantics;
- Windows PATH загрязнял WSL build;
- stock kernel содержал встроенный factory initramfs, поэтому внешний OpenIPC CPIO не удалял старые init scripts автоматически.

После переноса build tree в Linux filesystem и правильного overlay стало возможным загрузить чистый OpenIPC userspace без конфликтующего factory startup.

Важная методическая деталь: ошибки инфраструктуры нужно отделять от ошибок target port, иначе media bring-up быстро превращается в смешанный debugging всего сразу.

## D4 — Stock media stack внутри OpenIPC

Заводской `/app` стал ценнее поиска абстрактного SDK:
- были найдены Fullhan media kernel modules;
- восстановлен их порядок загрузки и VMM/ARC firmware setup;
- media device nodes появились внутри OpenIPC;
- sensor/MIPI shared libraries загрузились в musl process при правильном порядке dependencies.

Это резко сократило неизвестность. Вместо немедленного clean-room переписывания всего media SDK стало возможно использовать stock kernel ABI как мост и постепенно восстанавливать userspace contract.

На этом же этапе был важный поворот workflow: детальный разбор каждого callback sensor library был остановлен, когда стало ясно, что он не отвечает на текущий blocking question.

## D5 — Оживление ISP

Sensor и MIPI path удалось довести до состояния, совпадающего с ключевыми stock hardware признаками, но ISP IRQ оставался нулевым.

Дальнейшая работа объединила:
- stock runtime tracing;
- static reverse driver/userspace;
- reference source как semantic map;
- live MMIO comparison.

Ключевой найденный разрыв был не в sensor clock/MIPI, а в базовом ISP hardware state, включая interrupt enable/mask. После восстановления соответствующего состояния ISP начал стабильно генерировать interrupt cadence.

Это важный пример смены уровня диагностики: после доказанного sensor/MIPI больше не возвращались к ним при каждом симптоме downstream.

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

## D9 — Source-derived ISP runtime

`CHAT-002` начинается уже после первого рабочего hardware pipeline. Главная инженерная смена — отказ от лечения визуальных дефектов одиночными snapshot-регистрами.

Вместо этого:
- восстанавливается lifecycle `C540C → CB890 → CB970`;
- writers портируются в stock order;
- значения вычисляются из runtime context/profile;
- каждый законченный subset сначала compile-checkится, затем при необходимости проверяется на железе.

Через owner `v4.1.1+` и последовательные runtime stages были подтверждены source-derived writers, H.264 оставался 1280×720@25 и 125 frames / 5 s. Две фиксированные горизонтальные полосы, которые существовали на раннем pipeline, на этом этапе визуально исчезли. Причинность одному конкретному writer не была назначена; отдельно зафиксировано, что standalone `D0238` раньше не помогал.

Зелёный оттенок сохранялся и рассматривался отдельно как незакрытая AE/AWB/Bayer/CCM часть, а не как та же проблема, что горизонтальные полосы.

## D10 — Parallel heavy reverse и authoritative artifacts

Во время дальнейшего reverse обнаружилось, что активный binary `apollo.unpacked` неполон для поздних RW/GOT областей. Workspace был нормализован вокруг полного textual ARM dump; старый неполный artifact и лишние extraction copies были исключены из active work.

После этого heavy reverse был выделен параллельному агенту. Основной агент продолжал canonical runtime/integration, а parallel agent разбирал отдельные функции без hardware tests и без создания второй runtime-ветки.

Ключевые результаты этой фазы:
- shared signed `int16_t` Q7 sine LUT восстановлен как exact 360-entry table;
- `CFEB0/D0238` восстановлен exact и прошёл self-test против stock outputs;
- `D1258` восстановлен exact;
- `D1724` и `D1DB0` доведены до finished/parameterized reverse units с честно оставленными отсутствующими GOT-backed data objects;
- затем отдельно восстановлена значимая часть AE/AWB frontend, включая Bayer permutation и gain/state path; final day-mode estimator оставался незакрыт.

Методическая граница: если data-object физически отсутствует в authoritative source, он остаётся unresolved/parameterized; значения не угадываются.

