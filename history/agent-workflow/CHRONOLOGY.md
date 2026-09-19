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
Источник: `CHAT-005` → `CHAT-001`, 2026-08-24/25.

Начальный bootlog дал SoC/flash/RAM/sensor/network ориентиры и dual-lens признаки. Затем до любой записи NOR был снят полный 8 MiB dump, из него восстановлены U-Boot environment/разметка и доступ к bootloader. В `CHAT-006` тот же dump уже стал рабочим BSP-корпусом: из него извлекли `/app`, media modules/sensor libraries и начали восстанавливать ioctl ABI. После этого напрямую проверили TFTP→RAM путь.

**Переход:** вместо немедленного портирования media — доказать безопасный RAM boot.

Подробнее: [D1](CHRONOLOGY_DETAILS.md#d1--первичная-разведка-и-recovery-baseline).

### 2. Первый OpenIPC в RAM без замены kernel
Источник: `CHAT-005` → `CHAT-001`.

`CHAT-005` формулирует hybrid-RAM стратегию и доказывает U-Boot/TFTP transport; в `CHAT-001` внешний OpenIPC initramfs уже аппаратно загружен поверх stock FH8626 Linux. Подтверждены shell, Ethernet, MTD и SSH/userspace без изменения NOR.

**Milestone:** вопрос «может ли OpenIPC userspace работать на FH8626V100» был закрыт аппаратным запуском.

Подробнее: [D2](CHRONOLOGY_DETAILS.md#d2--openipc-ram-boot).

### 3. Стабилизация build/rootfs цикла
Источник: `CHAT-007` → `CHAT-001`.

Bring-up выявил проблемы сборки на Windows-mounted filesystem, загрязнение PATH и наложение stock initramfs. Сборка была перенесена в Linux filesystem WSL, а OpenIPC rootfs очищен от конфликтующих stock init scripts.

**Переход:** после чистого системного baseline можно было заниматься media, а не инфраструктурой загрузки.

Подробнее: [D3](CHRONOLOGY_DETAILS.md#d3--buildroot-и-чистый-openipc-baseline).

### 4. Переиспользование stock Fullhan media stack
Источник: `CHAT-007` → `CHAT-001`.

Stock media modules и sensor libraries были смонтированы из заводского `/app` и проверены внутри OpenIPC. Kernel modules поднялись в штатной зависимости, а MIPI/GC1054 plugins оказались загружаемыми из musl userspace.

**Переход:** основной риск сместился с ABI «запустится ли вообще» к правильной последовательности ISP/VPU/PAE initialization.

Подробнее: [D4](CHRONOLOGY_DETAILS.md#d4--stock-media-stack-внутри-openipc).

### 5. Sensor/MIPI → ISP
Источник: `CHAT-007` + `CHAT-008` + `CHAT-009` + `CHAT-011` → `CHAT-001`.

После сочетания stock scripts, dynamic probes, static reverse и аппаратных проверок был восстановлен достаточный sensor/MIPI/ISP bring-up. `CHAT-009` дополнительно закрепил обязательный cold-boot GPIO5 reset pulse. Ключевой перелом — обнаружение пропущенного ISP interrupt-enable state; после его восстановления ISP IRQ стал стабильно идти.

**Переход:** дальнейший блокер оказался уже между ISP frames и encoder path.

Подробнее: [D5](CHRONOLOGY_DETAILS.md#d5--оживление-isp).

### 6. ISP → VPU → PAE → H.264
Источник: `CHAT-009` + `CHAT-011` → `CHAT-001`.

После исправления ISP geometry и согласования VPU/PAE конфигурации encoder начал работать на hardware cadence. Затем был получен H.264 Annex-B output с SPS/PPS/IDR и сохранён capture-файл.

**Milestone:** hardware video pipeline FH8626 был доказан рабочим на уровне `sensor → ISP → encoder → H.264 file`.

Открытый вопрос на конец чата: dequeue ещё повторял один encoded descriptor, поэтому moving image и production-quality image path не считались закрытыми.

Подробнее: [D6](CHRONOLOGY_DETAILS.md#d6--hardware-h264-и-граница-доказательства).

### 7. Ускорение инженерного цикла
Источник: `CHAT-009` + `CHAT-011` → `CHAT-001`.

Повторные power-cycle/SSH delays стали отдельным bottleneck. Появилась идея одного долгоживущего media owner/daemon, а boot/SSH path был ускорен и стабилизирован; повторные подключения начали рассматриваться как часть dev-loop, а не посторонняя проблема.

Подробнее: [D7](CHRONOLOGY_DETAILS.md#d7--dev-loop-и-stateful-driver-lifetime).

### 8. Переход к handoff и параллельной агентной работе
Источник: `CHAT-007` + `CHAT-008` + `CHAT-009` + `CHAT-011` → `CHAT-001` → `CHAT-002`, 2026-08-25/27.

`CHAT-007` показывает ранний parallel checkpoint: внешние Fullhan references, локальные пути, confirmed/gaps и текущий ISP blocker передаются второму агенту без повторного reverse. Позже, когда контекст одного чата стал исчерпываться, результаты были собраны в master handoff, а в `CHAT-002` parallel roles стали уже явно специализированными.

**Переход:** проект начал отделять долговременное инженерное состояние от памяти одного диалога.

Подробнее: [D8](CHRONOLOGY_DETAILS.md#d8--handoff-и-первые-признаки-многоагентного-workflow).

### 9. Правильный dequeue и локализация серого кадра
Источник: `CHAT-010` + `CHAT-004`, 2026-08-26.

`CHAT-010` статически доказал точный contract: `PAE 0xC0045011` вызывает `pae_enc_stream_release(channel 0..7)`, а `4D05/4D06` — оба query одного stream с разной wait-policy; consume/read-index advance происходит через release → `enc_stream_get`. `CHAT-004` затем/в параллельной ветке даёт hardware evidence правильного dequeue: меняются descriptor/timestamp/CRC и получается последовательный H.264.

Следующий A/B показал, что 720p и воспроизведённый stock-like 1080p upscale оба кодируют одинаковый серый источник. Проблема была локализована выше encoder/scaler — в ISP input/runtime processing. После этого были найдены GC1054 scene profiles и stock `API_ISP_LoadIspParam → API_ISP_Run` lifecycle, а также снят полноценный stock runtime evidence bundle.

**Переход:** вместо исправления encoder/dequeue и угадывания отдельных MMIO работа перешла к восстановлению ISP lifecycle на основе полного code/runtime evidence.

Подробнее: [D9](CHRONOLOGY_DETAILS.md#d9--dequeue-grey-frame-и-stock-runtime-evidence).

### 10. Стабильный experimental substrate: persistent owner, hot reload и автоматический boot
Источник: `CHAT-012` → `CHAT-003`, 2026-08-26/27.

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

### 13. Day AWB mode1 и подтверждение dual-GC1054 lens architecture
Источник: `CHAT-013`, 2026-08-27.

После parallel heavy reverse текущий day-path был доведён существенно дальше: `CA4F4` восстановлен на реальных 9 AWB-stat records, найден правильный statistics mapping через ISP ioctl, а вычисления получили read-only/shadow режим перед hardware commit. Параллельная JXF37-гипотеза была снята stock evidence: конкретная плата использует GC1054 1280×720, а Full-HD получается downstream upscale.

Stock zoom trace показал, что wide/tele переключение не меняет sensor driver: target1/target2 выбираются GPIO4/GPIO14 при работающем media pipeline. Это закрепило модель двух GC1054 с разной оптикой и перенесло задачу из sensor-family reverse в lifecycle-integrated lens switching.

**Переход:** дальнейшая работа разделяется на day/night ISP parity и интеграцию доказанных lens/upscale contracts в persistent owner/runtime.

Подробнее: [D13](CHRONOLOGY_DETAILS.md#d13--day-awb-mode1-и-dual-gc1054-lens-architecture).

### 14. AWB → CCM coherent runtime и hardware validation
Источник: `CHAT-014`, 2026-08-27.

Параллельный Agent 3 восстановил цепь `CB4F0/CAFC0 → C9F68 → CE670/CE764` и доказал, что прямой AWB MMIO step обновляет gains, но не downstream color state. Новый coherent path обновил `A8/AA`, anchor/interpolation state и CCM; hardware execution совпал с offline prediction.

Контролируемый тест `512,512,512 → 544,480,544` дал pair `3→2`, weight `43`, изменение `B0/B1/B2` и единственное изменение CCM word `+4C8`. Отдельно доказано, что AWB restore не восстанавливает CCM автоматически, поэтому rollback должен учитывать оба state layer.

**Переход:** color pipeline перестал быть чисто статическим reverse и стал hardware-validated coherent state transition.

Подробнее: [D14](CHRONOLOGY_DETAILS.md#d14--awb--ccm-coherent-runtime-и-hardware-validation).

### 15. Cross-Fullhan semantic oracle и системные image-quality gaps
Источник: `CHAT-015`, 2026-08-27/28.

Центральная research-ветка сравнила FH8626 Apollo с именованными FH8852V100/V201 implementations и построила semantic map значительной части ISP runtime. Homologs использовались только как ориентир; target-specific выводы перепроверялись по FH8626 ARM/dataflow.

Это позволило переосмыслить несколько оставшихся проблем изображения: `CDD6C` идентифицирован как APC/detail/sharpening controller, а `ctx+0x60` — как live total-gain publication, от которой зависят APC/NR2D/YNR/CNR. В custom runtime оба пути были неполны: APC не выполнялся, total gain оставался stale.

Параллельный same-SoC research подтвердил полезность native encoder timestamps и различие sensor/output cadence для 1080p path.

**Переход:** blind reverse сменяется semantic matching + target proof, а оставшиеся image-quality defects формулируются как конкретные missing control/dataflow paths.

Подробнее: [D15](CHRONOLOGY_DETAILS.md#d15--cross-fullhan-semantic-oracle-и-systemic-image-quality-gaps).

### 16. Полный AE loop до GC1054 registers и day/night numerical parity
Источник: `CHAT-016`, 2026-08-27/28.

Persistent Agent 1 продолжил ранний C949C/C9898 reverse как большой `Task 2` и после requirement audit довёл AE/brightness control loop до sensor actuator. Были восстановлены statistics→target/error→history/hysteresis→controller→integration/gain redistribution→deferred commit→GC1054 callbacks и конкретные sensor registers.

Static reverse был дополнен stock sensor-library reverse и targeted runtime captures. Day и night получили независимые numerical replay; night limits `745/2` были подтверждены одновременно live context и SREG profile. `C9898` окончательно отделён как publication/status tail, а `D0630` — как отдельный statistics-driven ISP block, не sensor AE actuator.

**Переход:** AE перестал быть одним из крупных неизвестных current-day runtime; дальнейшая работа сместилась к image-detail modules и точной dual-lens/peripheral orchestration.

Подробнее: [D16](CHRONOLOGY_DETAILS.md#d16--полный-ae-loop-и-daynight-parity).

### 17. Image-detail pipeline: APC/NR3D/LTM closure через live RW/GOT evidence
Источник: `CHAT-017`, 2026-08-27/28.

Persistent Agent 2 довёл image-detail ветку от предварительной классификации до current-day exact contracts. Runtime RW/GOT capture снял blockers, недоступные в RX-only Apollo dump: восстановлены APC/CDD6C codebooks, активный NR3D/D0FEC preset/dispatch, LTM D0630/D0B2C tables и D1DB0 runtime coefficients.

Главный stock-vs-owner gap по softness сформулирован конкретно: сначала отсутствующий APC/detail path, затем активный NR3D, затем динамический LTM. YNR/CNR/NR2D/Purplefri и GB были отделены и перестали ошибочно считаться основными missing sharpness modules.

**Переход:** reverse mutable data objects оформляется как отдельный runtime-evidence layer рядом с уже готовым ARM code corpus; повторный full disassembly перестаёт быть нормальным способом решать GOT/table gaps.

Подробнее: [D17](CHRONOLOGY_DETAILS.md#d17--image-detail-apcnr3dltm-и-runtime-rwgot-closure).

### 18. Dual-lens switch получает controlled hardware validation
Источник: `CHAT-018`, supplemental snapshot той же Agent 1 ветки, 2026-08-28.

Уникальный хвост расширенного экспорта подтвердил на stock runtime статически восстановленный `D8308` lens-switch contract: target `1→2→2→1`, общий AE context/history, отсутствие day/night profile switch от одного lens change и возврат wide exposure после tele.

Tele в том же профиле требует существенно больше exposure/gain; краткий reset `64/64` остаётся static-exact, но вручную capture его не успевает поймать.

**Переход:** dual-GC1054 switching из static implementation contract становится implementation-ready runtime contract.

Подробнее: [D18](CHRONOLOGY_DETAILS.md#d18--controlled-wide--tele--wide-runtime-validation).

### 19. Current-day reverse convergence и переход к runtime integration
Источник: `CHAT-020`, 2026-08-28.

Центральный оркестратор свёл результаты трёх specialist lanes: full AE, hardware-validated AWB→CCM и detail/APC/NR3D/LTM. Широкий reverse current-day пути перестал быть главным режимом; roadmap переключился на последовательную интеграцию уже доказанных loops с отдельными hardware gates.

Основной runtime order после convergence: live total gain / sensor AE → APC/CDD6C → active NR3D/D0FEC → dynamic LTM D0630/D0B2C → оставшийся D1DB0/cadence.

**Переход:** задача меняется с «найти архитектуру stock ISP» на «портировать доказанные contracts и измерять parity».

Подробнее: [D19](CHRONOLOGY_DETAILS.md#d19--current-day-reverse-convergence-и-runtime-integration-plan).

### 20. Stock evidence campaign: state/transition corpus вместо разовых логов
Источник: `CHAT-021`, 2026-08-28.

После convergence reverse проект систематически снимает stock ground truth: четыре steady imaging state, bidirectional lens/day-night transitions, white-light coupling, talkback, siren, JPEG snapshot, PTZ, boot UART, selected sensor registers и targeted Apollo runtime regions. Одновременно готовится static reverse material для ранее не подготовленных modules/libs.

Full independent heap diffs признаны слишком шумными; причинный анализ переводится на synchronized before/early/settled captures малых доказанных областей. Evidence оформляется state matrix/catalog с canonical, superseded и unavailable статусами.

**Переход:** stock firmware становится воспроизводимым evidence dataset для будущей интеграции, а не только системой, к которой приходится возвращаться за каждым новым вопросом.

Подробнее: [D20](CHRONOLOGY_DETAILS.md#d20--stock-evidence-campaign-и-normalized-runtime-dataset).

### 21. Production catch-up: reverse contracts превращаются в hardware-tested runtime
Источник: `CHAT-022`, 2026-08-28.

Production integration-agent перенёс накопленный reverse в единый candidate runtime: live gain, physical AE commits, AWB→CCM, APC, NR3D, LTM, dual-sensor switching, reloadable algorithm module, capture/gates/rollback. Hardware loop подтвердил WIDE H.264 и несколько независимых механизмов.

Ключевой dual-sensor blocker локализован до cold-boot board sequence: `GPIO5 LOW → media modules → GPIO5 HIGH`. После правильного bootstrap stock probe видит оба GC1054, а WIDE↔TELE даёт реальное изображение.

Параллельно отделены два remaining production blockers: exact owner teardown/re-init/control-plane architecture и persistent green cast, который после механически успешного AWB→CCM переносится в отдельную RAW/Bayer/early-color parity задачу.

**Переход:** проект фактически выходит из режима «reverse-first» в staged implementation/hardware-validation loop.

Подробнее: [D21](CHRONOLOGY_DETAILS.md#d21--production-catch-up-и-hardware-parity-session).

### 22. OEM/platform identity: AJL33PQ0866 + CF26/SM family и donor corpus
Источник: `CHAT-024`, 2026-08-28.

External/device research связывает target не только с Fullhan SoC, но и с конкретной OEM-линейкой. На самой камере фиксируются internal model `AJL33PQ0866`, firmware branch `YGT.AJL33PQ0866...`, PCB family `CF26/SM`, dual 3.6/12 mm optics и 4+4 illumination. Это резко сужает поиск родственных firmware/dumps.

Сильные references включают CF26/SM400/AJ-SM relatives, близкий documented retail clone на FH8626V100/dual 1054/3.6+12 и несколько публичных FH8626 donor dumps. Exact retail SKU при этом не объявляется установленной.

**Переход:** external research перестаёт искать «похожие FH8626 камеры» и начинает строить donor corpus вокруг пересечения target-specific software/PCB/SoC identifiers.

Подробнее: [D22](CHRONOLOGY_DETAILS.md#d22--oemidentity-ajl33pq0866-cf26sm-и-donor-search).

### 23. Cross-platform gap hunting: lifecycle, statistics epoch и RAW architecture leads
Источник: `CHAT-025`, 2026-08-28/29.

Specialist external-research lane перестаёт делать широкий homolog search и целится в remaining production/reverse gaps. Exact-FH8626 public adapters и соседние Fullhan SDK/runtime материалы дают имена lifecycle/API, toolchain/SDK targets и независимые архитектурные ориентиры.

Критический вывод: sensor/VI работает около 25 fps, тогда как encoder может работать на другой cadence; поэтому encoded dequeue нельзя использовать как автоматический proxy свежего ISP statistics frame. External lifecycle patterns также усиливают необходимость exact reverse teardown/re-init и early RAW/Bayer initialization.

**Переход:** внешний research становится intelligence layer, который выдаёт Agent 1 адресные hypotheses/omissions вместо повторного общего reverse.

Подробнее: [D23](CHRONOLOGY_DETAILS.md#d23--cross-platform-gap-hunter-и-targeted-reverse-leads).

### 24. Late Agent-4 corpus: watchdog evidence и peripheral backlog
Источник: `CHAT-026`, partial export, 2026-08-29.

Доступная часть экспорта не содержит сам ранний reverse-процесс Agent 4, но сохраняет consolidated technical state: stock watchdog ownership/runtime API и kernel driver path считаются разобранными; подготовлены exact stock kernel/U-Boot reverse artifacts. Human-detection path выделен в отдельный будущий OpenIPC adapter backlog, PTZ software/backend считается closed при отложенной physical actuator validation.

**Переход:** эти тяжёлые kernel/runtime artifacts становятся первым явным кандидатом на отдельный REVERSE_HEAVY vault вместо everyday master handoff.

Подробнее: [D24](CHRONOLOGY_DETAILS.md#d24--late-agent-4-watchdogperipheral-corpus).

### 25. Evidence-directed final reverse closure: E1–E7 → watchdog/human/HAL
Источник: `CHAT-027`, 2026-08-28—30.

Agent 1 сначала исчерпывает master v23 статически и превращает остаток в семь конкретных evidence packages E1–E7. После Agent 4 live captures тот же reverse возвращается уже адресно: lifecycle/re-init, 25 Hz statistics epoch/double buffering, physical RAW10/CFA, scene/IQ publication semantics и software PTZ граница существенно закрываются.

Дальше reverse расширяется за release-minimum: watchdog доведён до driver/PMU/U-Boot reset contract; human detection — до Y8/OBJDETECT/model/result ABI; color/HAL — до physical CFA, BLC/GB/AWB→CCM/YC/generation ownership; illumination — до PWM modes/LDR/IR-cut; `dev_ctrl` распакован и классифицирован как board/service daemon. Источник заканчивается на последних optional ISP ветках WDR/GME, где ещё идёт коррекция прежней WDR↔LTM типизации.

**Переход:** broad reverse перестаёт быть основным методом. Оставшаяся работа строится как closure matrix: static proof, named target evidence, implementation contract или bounded archaeology.

Подробнее: [D25](CHRONOLOGY_DETAILS.md#d25--evidence-directed-final-reverse-closure).

### 26. OpenIPC product strategy: frontend decoupling и Divinus-first path
Источник: `CHAT-028`, 2026-08-29/30.

После сведения reverse/evidence/productization веток оркестратор отдельно исследует путь к usable OpenIPC camera. Исторический вывод этой сессии: отсутствие подтверждённого Majestic backend для FH8626 не должно блокировать порт. Hardware/media ownership отделяется от frontend, а Divinus рассматривается как первый открытый consumer готового encoded stream.

Предлагается двухэтапный путь: сначала существующий FH8626 owner/sidecar → Divinus RTSP/Web/API, затем при необходимости native FH8626 HAL. Одновременно engineering bring-up отделяется от upstream-clean supply chain: отсутствие публичного SDK ограничивает чистый upstream, но не обязано блокировать локальную проверку hardware/userspace architecture.

**Переход:** проект начинает проектировать не «одну прошивку вокруг одного streamer-а», а reusable FH8626 backend с заменяемыми frontend-ами и отдельными upstream/provenance gates.

Подробнее: [D26](CHRONOLOGY_DETAILS.md#d26--openipc-product-strategy-и-divinus-first-path).

### 27. Agent 3 productization: media-runtime, sidecar RTSP и OpenIPC staging
Источник: `CHAT-029`, 2026-08-28/29.

OpenIPC/productization lane превращает reverse contracts в source-level architecture без попытки сразу подменить весь vendor stack. Создаются host-tested media lifecycle/state-machine pieces, fixed wire sidecar ABI, Annex-B H.264/RTP/RTSP path, ABI probe, read-only evidence tooling и source-only Buildroot/OpenIPC staging. Divinus получает external encoded-source boundary; Majestic исключается из критического пути.

Отсутствие официального FH8626 SDK переклассифицируется из «порт невозможен» в upstream/provenance blocker: engineering bring-up может двигаться поверх существующего hardware owner и временных externally supplied dependencies, а upstream-clean profile требует отдельного допустимого source path.

**Переход:** reverse впервые материализуется в переносимые userspace/runtime компоненты и реальные target-repository staging paths, сохраняя single-owner hardware boundary.

Подробнее: [D27](CHRONOLOGY_DETAILS.md#d27--agent-3-openipc-productization).

### 28. Native Linux platform bring-up: от базового boot к cumulative SoC integration
Источник: `CHAT-030`, partial export, 2026-08-30.

Agent 5 kernel lane показывает позднюю native-platform стадию. Hardware PASS уже есть для machine/INTC/timer/UART0, SPI0/NOR/MTD, reboot/watchdog, GPIO0/1 и I²C0/1/2. Ethernet исправлен до exact PHY contract и ждёт отдельной hardware приёмки; RTC зарегистрирован, но требует диагностики.

Офлайн подготовлены native pinctrl, SADC/EFUSE, UART1/2, PWM0, USB/DWC2, SDIO/MMC, SPI1, DMA/AES, audio, PMU gap map и defconfig. Stock machine reverse дополнительно показывает условную регистрацию 22/23 platform devices и раннюю последовательность `fh_pmu_init() → fh_pinctrl_init(0xFE090080)`. Media/VMM и расширение memory policy сознательно оставлены последним высокорисковым слоем.

**Переход:** kernel work перестаёт быть набором probes и превращается в cumulative native SoC source candidate с отдельным hardware acceptance gate.

Подробнее: [D28](CHRONOLOGY_DETAILS.md#d28--native-linux-platform-bring-up).

### 29. Divinus V11 hardening: external-source boundary и fail-closed API
Источник: `CHAT-031`, 2026-08-30.

Agent 6 продолжает Divinus независимо от ещё меняющегося hardware owner. V11 candidate исправляет transport/API/WebUI слой: raw H.264 отвязывается от fMP4 enable, source-init failure не должен оставлять ложный running state, sidecar получает более строгие generation/discontinuity/socket semantics, а runtime API закрывается от hardware-owned функций в external-source режиме.

MJPEG переосмысляется как отсутствующий JPEG provider, а не как неподдерживаемая возможность Divinus: HTTP/MJPEG frontend можно подготовить заранее, но capability остаётся unavailable до появления JPEG producer от единственного hardware owner.

Все изменения остаются `PENDING_WSL`: authoritative apply/tests/ARM build/runtime должны выполняться в пользовательском WSL.

**Переход:** Divinus lane отделяет frontend hardening от owner integration и формализует capability/provider границы, позволяя развивать userspace без второго владельца железа.

Подробнее: [D29](CHRONOLOGY_DETAILS.md#d29--divinus-v11-hardening).

### 30. Agent 7 release-regression: host-success не выдерживает physical acceptance
Источник: `CHAT-032`, 2026-08-30.

Agent 7 собрал новый production runtime с lifecycle/single-owner, stats/stream workers, AE/AWB/CCM, IQ manager, lens transaction и host gates; ARM build также дошёл до корректного FH8626 ELF. На target media/ISP/PAE/VPU bootstrap проходил, но дальнейший hardware loop выявил регрессии изображения и lens behavior.

Критическая граница стала очевидной на R13–R18: software target-state и diagnostic GPIO/logs показывали успешные transitions, тогда как физический FOV не менялся. Попытки wide-only/pinmux/force-write продолжили отклонение от ранее hardware-proven Agent 3 contract. Ветка была остановлена и передана как forensic/quarantine, а не как release.

**Переход:** compile/host gates окончательно отделяются от hardware parity; release engineering получает обязательный protected-known-good baseline и negative-evidence provenance.

Подробнее: [D30](CHRONOLOGY_DETAILS.md#d30--agent-7-release-regression-и-forensic-rollback).


### 31. Codec/ghosting reverse: H.264 RC, MJPEG/BGM/NR3D и integration contracts
Источник: `CHAT-035`, 2026-09-05/06.

Отдельный deep-audit слой переносит reverse с общего ISP на encoded/media behavior. Для H.264 восстанавливаются public/app→wire RC mapping, realtime `0xC01C5055`, field semantics и различия cold/realtime update; для MJPEG/JPEG — buffer/query/release ownership, quantization и shutdown lifecycle; отдельно разбираются audio/container delivery и geometry/upscale.

Ghosting локализуется не к одной «магической настройке»: NR3D, BGM motion path, encoder RC/IDR и queue/lifecycle анализируются раздельно. По результату создаются C/H candidates, state machines и integration contracts, но target acceptance сознательно передаётся отдельному интегратору.

**Переход:** reverse начинает выпускать implementation-ready modules с explicit lifecycle/ABI evidence, а не только описания функций.

Подробнее: [D31](CHRONOLOGY_DETAILS.md#d31--codecghosting-reverse-и-integration-contracts).


### 32. Post-V2 standalone closure и восстановление canonical workspace
Источник: `CHAT-036`, 2026-09-08—11.

Codec/ghosting contracts из предыдущего этапа переводятся в более полный standalone SDK layer: BGM capability/bind lifecycle, dedicated VMM ownership/free/recovery, NR3D cold/restart state machine, realtime H.264 RC и transition policy получают host implementation и расширенные tests. Production owner/Divinus сознательно не меняются.

Одновременно независимый integrity audit обнаруживает, что workspace continuity ненадёжен: C/H, документы и несколько ACTIVE trees расходятся, status местами опережает source. Проект восстанавливает единое canonical дерево и отделяет current source от deep historical/reverse corpus.

**Переход:** software contracts становятся host-verified standalone substrate, а сама engineering state впервые оформляется как индексируемый/recoverable workspace, а не как последовательность handoff-архивов.

Подробнее: [D32](CHRONOLOGY_DETAILS.md#d32--post-v2-standalone-closure-и-canonical-workspace-recovery).


### 33. Divinus regression audit и смена product-path на Majestic family
Источник: `CHAT-037`, 2026-09-12—14.

Сравнение current native Divinus HAL с exact owner/reference вскрывает несколько конкретных orchestration regressions: AWB/E2 stats читаются не из selected runtime bank, frontend barrier перенесён в неверный address domain, control cadence получил неподтверждённый blocking frame-wait, а RTSP нарушил descriptor/parser ownership.

Одновременно FH8852-family Majestic после минимальной адаптации способен хотя бы стартовать на FH8626. Это меняет product strategy: native Divinus больше не должен поглощать всё время только из-за уже вложенных усилий. Его сохраняют как reference/diagnostic implementation, а Majestic становится целевым practical path при условии прохождения реальных media gates.

**Переход:** reverse assets сохраняются, но продуктовая реализация отделяется от исследовательской и выбирает существующий совместимый frontend/backend там, где это сокращает неподтверждённую hardware orchestration.

Подробнее: [D33](CHRONOLOGY_DETAILS.md#d33--divinus-regression-audit-и-majestic-target-path).


### 34. OpenIPC-native U-Boot: от preservation layout к нормальной product architecture
Источник: CHAT-038, 2026-09-17/18.

Рабочий modern U-Boot port FH8626 проходит contribution-oriented audit. Первоначальная stock-compatible схема сохраняется как hardware-proven recovery baseline, но перестаёт определять production layout. Новая mainline переходит на стандарт OpenIPC 8 MiB: 256K boot / 64K env / 2M kernel / 5M rootfs / rootfs_data.

Fullhan-specific boot container остаётся внутренней board/SoC деталью стандартного boot; descriptor/JAMCRC генерируются из фактического payload, factory environment и compatibility helpers уходят в migration/recovery target. Generated binaries исключаются из source Git.

**Переход:** low-level platform port впервые оформляется не как восстановленная китайская прошивка, а как самостоятельный OpenIPC-native component с clean source/build boundary и contribution path.

Подробнее: [D34](CHRONOLOGY_DETAILS.md#d34--openipc-native-u-boot-curation).


### 35. Curated Linux series: migration dump превращается в reviewable platform support
Источник: CHAT-039, 2026-09-18.

Два исторических FH8626 kernel commits разбираются по содержимому и пересобираются от Fullhan FH8852 base в 13 логических commits. В чистую серию входят platform/SoC data, machine support, clock/pinctrl fixes, PWM wiring/hardening, AXI DMA registration, JL1101/RMII, MAC propagation, checksum cleanup, RTC error handling и DWC2 VBUS semantics.

OpenIPC MTD layout закладывается сразу, retail-specific MMC symbol заменяется физическим SD0_1BIT, а unrelated SADC cleanup исключается. Аппаратно проверенный static RMII path не переписывается ради «красоты».

**Переход:** kernel из preservation/migration snapshot превращается в reviewable upstream-oriented source series; каждый новый behavior delta отделяется от уже hardware-proven платформы.

Подробнее: [D35](CHRONOLOGY_DETAILS.md#d35--curated-fh8626-linux-series).


### 36. Cross-repo ownership sanitation: Firmware становится shared core, streamer/device work расходится по владельцам
Источник: CHAT-040, 2026-09-18.

Старый Firmware preservation WIP разбирается как инвентарь, а не как основа production. Чистый Firmware core начинает брать curated Linux source напрямую, использует стандартную OpenIPC layout и не хранит AJL-specific board code, Divinus implementation или factory media blobs как canonical source.

Параллельно создаются Divinus и Majestic runtime directions, а Builder получает только device assembly/profile responsibility. Все binary dependencies получают ownership/provenance/disposition.

**Переход:** проект окончательно переходит от одного монолитного «порт камеры» дерева к нескольким normal OpenIPC repositories с явными owner boundaries.

Подробнее: [D36](CHRONOLOGY_DETAILS.md#d36--cross-repo-ownership-sanitation).


## Современный anchor

Трёхфайловая live-state сверка подтверждает, что на 2026-09-18 текущая архитектура уже использует GitHub как engineering authority, Drive для heavy evidence и Ghidra MCP как mutable reverse workspace. Это современный anchor; следующие исторические файлы должны восстановить сам переход от handoff/checkpoint подхода к этой системе.
