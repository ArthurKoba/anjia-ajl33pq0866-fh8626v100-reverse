# Хронология реверса и портирования FH8626V100

Назначение: дать короткую карту того, **как проект пришёл к текущему состоянию**.

Это не технический журнал и не замена `STATE.md`. Здесь остаются только опорные точки — как оглавление глав с небольшим описанием результата. Подробности и развилки находятся в `CHRONOLOGY_DETAILS.md`.

## Правила ведения

- Один этап — несколько строк, а не перечень каждой команды/адреса/ioctl.
- Фиксировать только изменения, которые реально изменили направление работы или уровень готовности.
- Для каждого этапа: проблема → главный результат → причина следующего перехода.
- Если ранняя гипотеза позже опровергнута, в основной хронологии оставить только важный переход; подробности вынести в `CHRONOLOGY_DETAILS.md`.
- Не реконструировать историю только из современного `STATE.md`; источником последовательности являются исторические чаты.
- Уровень доказательства указывать только когда он меняет смысл этапа: static/source/build/hardware.

## Карта этапов

### 1. Первичная идентификация и безопасный доступ
Источник: `CHAT-001`, 2026-08-24/25.

Из stock boot/runtime были восстановлены базовые характеристики FH8626V100-камеры, dual-lens/sensor architecture, flash layout и U-Boot access. До записи flash сначала получили полный dump и рабочий способ входа/восстановления.

**Переход:** вместо немедленного портирования media — доказать безопасный RAM boot.

Подробнее: [D1](CHRONOLOGY_DETAILS.md#d1--первичная-разведка-и-recovery-baseline).

### 2. Первый OpenIPC в RAM без замены kernel
Источник: `CHAT-001`.

Через U-Boot/TFTP был загружен внешний OpenIPC initramfs поверх stock FH8626 Linux. Подтверждены shell, Ethernet, MTD и SSH/userspace без изменения NOR.

**Milestone:** вопрос «может ли OpenIPC userspace работать на FH8626V100» был закрыт аппаратным запуском.

Подробнее: [D2](CHRONOLOGY_DETAILS.md#d2--openipc-ram-boot).

### 3. Стабилизация build/rootfs цикла
Источник: `CHAT-001`.

Bring-up выявил проблемы сборки на Windows-mounted filesystem, загрязнение PATH и наложение stock initramfs. Сборка была перенесена в Linux filesystem WSL, а OpenIPC rootfs очищен от конфликтующих stock init scripts.

**Переход:** после чистого системного baseline можно было заниматься media, а не инфраструктурой загрузки.

Подробнее: [D3](CHRONOLOGY_DETAILS.md#d3--buildroot-и-чистый-openipc-baseline).

### 4. Переиспользование stock Fullhan media stack
Источник: `CHAT-001`.

Stock media modules и sensor libraries были смонтированы из заводского `/app` и проверены внутри OpenIPC. Kernel modules поднялись в штатной зависимости, а MIPI/GC1054 plugins оказались загружаемыми из musl userspace.

**Переход:** основной риск сместился с ABI «запустится ли вообще» к правильной последовательности ISP/VPU/PAE initialization.

Подробнее: [D4](CHRONOLOGY_DETAILS.md#d4--stock-media-stack-внутри-openipc).

### 5. Sensor/MIPI → ISP
Источник: `CHAT-001`.

После сочетания stock scripts, dynamic probes, static reverse и аппаратных проверок был восстановлен достаточный sensor/MIPI/ISP bring-up. Ключевой перелом — обнаружение пропущенного ISP interrupt-enable state; после его восстановления ISP IRQ стал стабильно идти.

**Переход:** дальнейший блокер оказался уже между ISP frames и encoder path.

Подробнее: [D5](CHRONOLOGY_DETAILS.md#d5--оживление-isp).

### 6. ISP → VPU → PAE → H.264
Источник: `CHAT-001`.

После исправления ISP geometry и согласования VPU/PAE конфигурации encoder начал работать на hardware cadence. Затем был получен H.264 Annex-B output с SPS/PPS/IDR и сохранён capture-файл.

**Milestone:** hardware video pipeline FH8626 был доказан рабочим на уровне `sensor → ISP → encoder → H.264 file`.

Открытый вопрос на конец чата: dequeue ещё повторял один encoded descriptor, поэтому moving image и production-quality image path не считались закрытыми.

Подробнее: [D6](CHRONOLOGY_DETAILS.md#d6--hardware-h264-и-граница-доказательства).

### 7. Ускорение инженерного цикла
Источник: `CHAT-001`.

Повторные power-cycle/SSH delays стали отдельным bottleneck. Появилась идея одного долгоживущего media owner/daemon, а boot/SSH path был ускорен и стабилизирован; повторные подключения начали рассматриваться как часть dev-loop, а не посторонняя проблема.

Подробнее: [D7](CHRONOLOGY_DETAILS.md#d7--dev-loop-и-stateful-driver-lifetime).

### 8. Переход к handoff и параллельной агентной работе
Источник: `CHAT-001`, 2026-08-26.

Когда контекст чата стал исчерпываться, результаты, артефактные пути, методы reverse и открытые gaps были собраны в handoff. Материал второго агента был сопоставлен с первым и сведён в один master, чтобы следующий агент не начинал reverse заново.

**Переход:** проект начал отделять долговременное инженерное состояние от памяти одного диалога.

Подробнее: [D8](CHRONOLOGY_DETAILS.md#d8--handoff-и-первые-признаки-многоагентного-workflow).

### 9. Правильный dequeue и локализация серого кадра
Источник: `CHAT-004`, 2026-08-26.

Статический reverse связал `PAE 0xC0045011` с release/consume encoded stream, а `4D05/4D06` — с query одного и того же media stream. После исправления порядка `query → copy → release` очередь начала реально двигаться: менялись descriptor/timestamp/CRC и был получен последовательный H.264.

Следующий A/B показал, что 720p и воспроизведённый stock-like 1080p upscale оба кодируют одинаковый серый источник. Проблема была локализована выше encoder/scaler — в ISP input/runtime processing. После этого были найдены GC1054 scene profiles и stock `API_ISP_LoadIspParam → API_ISP_Run` lifecycle, а также снят полноценный stock runtime evidence bundle.

**Переход:** вместо исправления encoder/dequeue и угадывания отдельных MMIO работа перешла к восстановлению ISP lifecycle на основе полного code/runtime evidence.

Подробнее: [D9](CHRONOLOGY_DETAILS.md#d9--dequeue-grey-frame-и-stock-runtime-evidence).

### 10. Стабильный experimental substrate: persistent owner, hot reload и автоматический boot
Источник: `CHAT-003`, 2026-08-26/27.

После первых рабочих H.264 запусков основным bottleneck стал сам цикл экспериментов: reboot/U-Boot, повторный media bring-up и риск второго ISP owner. Серия v3.8→v4.0.4 привела к стабильному single-owner baseline, reloadable ISP plugin и сохранённым `openipc_boot/stock_boot`, а live MMIO rollback был признан опасным.

**Переход:** аппаратный контур превратился из «перезагрузить и заново поднять всё» в устойчивую платформу для коротких runtime-итераций; это подготовило следующий этап source-derived восстановления `CB970`.

Подробнее: [D10](CHRONOLOGY_DETAILS.md#d10--persistent-owner-hot-reload-и-автоматизация-dev-loop).

### 11. Source-derived ISP runtime вместо register poking
Источник: `CHAT-003` → `CHAT-002`, 2026-08-27.

После базового H.264 bring-up работа сместилась от одиночных MMIO-экспериментов к восстановлению stock lifecycle и `CB970` writer-chain из disassembly/context. Последовательно проверялись source-derived stages через owner/hot-plugin, при этом H.264 оставался стабильным, а ранее наблюдавшиеся фиксированные горизонтальные линии больше не появлялись.

**Переход:** целью стало не «починить картинку одним регистром», а построить связный replacement stock runtime, пригодный для замены `apollo`.

Подробнее: [D11](CHRONOLOGY_DETAILS.md#d11--source-derived-isp-runtime).

### 12. Параллельный heavy reverse и нормализация evidence
Источник: `CHAT-002`.

Workspace был очищен от неполного Apollo artifact и переведён на один полный authoritative ARM text dump. После этого тяжёлые функции начали разбираться отдельным parallel reverse-agent с непересекающимся scope. Были восстановлены общий Q7 sine LUT, exact `CFEB0/D0238`, `D1258`, большая часть `D1724/D1DB0`, а затем отдельный AE/AWB frontend.

**Переход:** reverse стал разбиваться на законченные integration units, которые основной агент должен сливать в один canonical runtime.

Подробнее: [D12](CHRONOLOGY_DETAILS.md#d12--parallel-heavy-reverse-и-authoritative-artifacts).

## Современный anchor

Трёхфайловая live-state сверка подтверждает, что на 2026-09-18 текущая архитектура уже использует GitHub как engineering authority, Drive для heavy evidence и Ghidra MCP как mutable reverse workspace. Это современный anchor; следующие исторические файлы должны восстановить сам переход от handoff/checkpoint подхода к этой системе.
