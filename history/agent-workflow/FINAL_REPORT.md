# Финальный отчёт аудита agent workflow FH8626V100

Статус: `NEAR_FINAL / 43 HISTORICAL SOURCES PROCESSED`.

Это итоговый срез после 43 уникальных исторических источников. Аудит можно дополнять дальше, но основные повторяющиеся классы ошибок, рабочие практики и инфраструктурные переходы уже стабилизировались.

## 1. Итоговый вердикт

Главная проблема ранних агентов была не в недостатке «умения читать ARM» и не в недостатке объёма ответа.

Самые дорогие ошибки появлялись, когда инженерная истина оставалась внутри текущего чата:
- забывался target state;
- путались stock/OpenIPC/WSL/UART;
- терялись уже полученные artifacts;
- старый handoff воспринимался как current state;
- validation status переносился на другой artifact;
- пользователь становился маршрутизатором файлов и технического ветвления.

Главный успех проекта — постепенное вынесение истины из чата во внешние authority layers:
`workspace/checkpoint → structured corpus → Drive evidence → GitHub source/state → Ghidra mutable reverse → Koba Bridge Agent/Reviewer workflow`.

К финальной стадии пользователь уже не является основным диспетчером файлов/контекста. Его оптимальная роль — owner проекта, аппаратный оператор и обладатель privileged/local validation surface.

## 2. Самые частые ошибки

Число ниже означает количество **разных исторических CHAT-источников**, в которых класс явно подтверждён, а не число отдельных эпизодов.

### 2.1 Потеря текущего состояния — E-008, 12 источников

Самый частый класс.

Агент забывал:
- stock или OpenIPC сейчас загружен;
- какой owner/version активен;
- очистился ли `/tmp`;
- какой transport доступен;
- какой IP/path/compiler уже доказан;
- какие artifacts уже существуют.

Последствие: повторный bring-up, неправильные команды, ложная диагностика и ненужные reboot/recovery cycles.

Главный вывод: current state должен жить во внешнем authority, а не в памяти диалога.

### 2.2 Неправильная гранулярность — E-002, 8 источников

Агент колебался между двумя крайностями:
- огромная цепочка команд до неизвестного результата;
- микрошаги даже для рутинного доказанного процесса.

Правильная модель появилась позднее: decision-boundary driven granularity.

### 2.3 Игнорирование уже заданного рабочего протокола — E-014, 8 источников

Особенно дорого, потому что правила уже были известны:
- не выдавать unsolicited archives/SHA;
- не делать тяжёлый build в browser agent;
- использовать правильную execution lane;
- соблюдать agreed command format.

Одного «я понял» оказалось недостаточно. Устойчивость появилась, когда правила превратились в executable checklists/bootstrap artifacts.

### 2.4 Избыточная техническая экспозиция — E-016, 8 источников

Агент часто хорошо анализировал, но выдавал пользователю внутренний technical dump вместо результата/следующего действия.

Важный вывод: большая глубина внутренней работы не требует большой длины пользовательского ответа.

### 2.5 Неверная execution surface — E-019, 8 источников

Смешивались:
- WSL;
- Windows paths;
- UART;
- SSH;
- stock/OpenIPC target;
- локальные и target utilities.

Лечение — terminal lanes + environment context.

### 2.6 Нет видимого milestone progress — E-023, 7 источников

При действительно глубоком reverse пользователь не видел:
- что уже доказано;
- что осталось;
- где граница следующего meaningful milestone.

Позднее появился milestone heartbeat вместо потока мелких сообщений или долгого молчания.

### 2.7 Диагностика ломает объект исследования — E-012, 6 источников

Самые опасные эпизоды:
- live MMIO rollback;
- unload stateful vendor modules;
- LD_PRELOAD/concurrent stateful ioctl;
- resource-heavy captures.

Вывод: diagnostic action — часть hardware state machine и требует safety/resource model.

### 2.8 Изменение без запроса / premature export — E-013, 6 источников

Агент слишком рано:
- повышал master;
- собирал handoff;
- менял artifacts;
- выпускал новую «версию».

Позднее появился frozen-base/DELTA fan-in и release discipline.

### 2.9 Shell chaining / operator UX — E-024, 6 источников

`&&`, длинные one-liners, backslash continuation и UART multiline paste многократно портили execution.

Command formatting оказался не косметикой, а reliability feature.

### 2.10 Premature COMPLETE — E-031, 6 источников

Одна из самых важных epistemic ошибок.

Агент находил главный механизм и называл задачу закрытой. Requirement audit затем обнаруживал:
- missing hardware gate;
- missing lifecycle/error path;
- missing feature surface;
- неполный reverse.

Позднее completion стал отдельным acceptance audit.

## 3. Другие устойчивые классы

По 5 источников подтверждены:
- неподтверждённые пути/имена/структура окружения;
- ненужное усложнение простых операций;
- лишние проверки после уже достаточного доказательства;
- необоснованная уверенность;
- передача непроверенного artifact;
- предположение наличия target utilities.

По 4 источникам:
- перекладывание анализа на пользователя;
- нескоупленный broad search;
- просьба пользователю сделать то, что агент уже может сделать сам;
- хрупкий UART paste.

Поздний этап добавил более зрелые архитектурные ошибки:
- competing Ghidra projects;
- branch sprawl;
- coordination metadata в implementation repos;
- preservation architecture как product target;
- cross-repo ownership mixing;
- cleanup, который удалил runtime dependency до появления replacement owner.

## 4. Что работало лучше всего

### 4.1 Full searchable evidence — I-024, 7 источников

Самая часто подтверждённая positive practice.

Один подготовленный searchable corpus лучше десятков target extraction commands.

Это постепенно выросло из text disassembly в Ghidra semantic projects.

### 4.2 Progress по engineering boundaries — I-034, 5 источников

Пользователю полезнее:
- ISP→VPU закрыт;
- AE actuator остался;
- hardware parity pending;

чем «готово 82%» без denominator.

### 4.3 Persistent single owner — I-012, 4 источника

Stateful vendor media нельзя безопасно многократно открывать/закрывать случайными helpers.

Single owner стал основой:
- media lifecycle;
- hot reload;
- rollback;
- streamer decoupling.

### 4.4 Workspace hygiene / one authority — I-018, 4 источника

Очень сильная практика: один authoritative artifact/work tree вместо «final2/new/fixed/latest».

Позднее она выросла в GitHub/Ghidra authority.

### 4.5 Parallel reverse с разделёнными scopes — I-019, 4 источника

Параллелизм полезен только при непересекающихся обязанностях и explicit fan-in.

Без этого agents мешали друг другу и повторяли работу.

### 4.6 Явные terminal lanes — I-027, 4 источника

Простой label `WSL / UART / U-Boot / target` предотвращал целый класс expensive mistakes.

### 4.7 Hardware experiment как state machine — I-048, 4 источника

Baseline → action → observation → rollback → postcondition сделал hardware evidence воспроизводимым.

## 5. Самые сильные поздние улучшения

Они встречаются в меньшем числе чатов, но качественно изменили систему.

### Structured evidence и reverse
- canonical corpus;
- static code vs mutable runtime data;
- evidence gap contracts;
- Ghidra decompile/CFG/XREF/types;
- explicit unresolved indirect flow;
- target ASM proof поверх semantic pseudocode.

### Orchestration
- persistent specialist lanes;
- orchestrator-owned fan-in;
- frozen master + DELTA;
- implementation-agent отдельно от reverse-agent;
- evidence-agent ↔ reverse-agent closure loop;
- shared contract issue между Divinus/Majestic agents.

### Storage/authority
- workspace → Drive;
- Drive/heavy evidence → GitHub current authority;
- Ghidra MCP → mutable reverse authority;
- Koba Bridge → Agent mutation + Reviewer independent verification.

### Repository architecture
- ownership by natural layer;
- source Git отдельно от generated binaries/evidence;
- preservation snapshot ≠ product architecture;
- OpenIPC conventions > factory conventions;
- Builder как thin composed device layer;
- streamer-neutral core + runtime overlays.

### Validation
- source/build/hardware/product statuses разделены;
- exact artifact identity для PASS;
- hardware-proven baseline protected;
- pre-deploy dependency closure;
- fail-closed capability/ABI guards.

## 6. Эволюция взаимодействия человек ↔ агент

### Фаза 1 — человек как диспетчер
Пользователь:
- передаёт файлы;
- выполняет почти каждую команду;
- напоминает state;
- решает, что делать по выводу.

Agent в основном отвечает и предлагает next commands.

### Фаза 2 — checkpoints/workspace
Появляются:
- распакованные рабочие каталоги;
- checkpoint/handoff;
- persistent owner;
- prepared reverse artifacts.

Человек ещё остаётся основным router.

### Фаза 3 — parallel specialists
Появляются Agent 1/2/3/4:
- AE;
- AWB;
- detail/IQ;
- evidence acquisition;
- productization.

Проблемой становится fan-in и доступ к physical artifacts.

### Фаза 4 — evidence corpus + orchestration
Master workspace получает:
- maps/indexes;
- doctor/health;
- status matrix;
- source/evidence layers;
- frozen-base DELTA fan-in.

User всё меньше объясняет проект вручную.

### Фаза 5 — Drive durability
Heavy evidence и recovery checkpoints перестают зависеть от chat sandbox/local transient state.

### Фаза 6 — GitHub/Ghidra authority
Current source/state/contracts живут в GitHub.
Heavy evidence — в отдельном storage.
Mutable reverse — в Ghidra.

Это фундаментальный переход: chat становится control/analysis surface, а не storage.

### Фаза 7 — Koba Bridge multi-agent engineering
Agent:
- изменяет repositories штатным API;
- работает в own implementation lane;
- синхронизирует authority;
- читает findings других agents.

Reviewer:
- независимо проверяет commits/refs/trees.

Пользователь:
- определяет архитектуру/приоритет;
- выполняет hardware/local build gates;
- даёт privileged permission decisions.

Это уже не «чат с помощником», а распределённая инженерная система.

## 7. Где baseline prompt был хорош

Исходный prompt очень рано правильно поймал:
- копируемые command blocks;
- no `&&` / no backslash continuation;
- explicit `cd`;
- agent-owned branching;
- не использовать Python для тривиальных задач;
- не перегружать будущими сценариями;
- archive only as transport.

Эти правила многократно подтвердились.

## 8. Где baseline prompt был недостаточен

Он почти не описывал:
- startup authority/read order;
- local environment context;
- state recovery;
- artifact identity/provenance;
- validation ontology;
- hardware safety/resource budgets;
- canonical reverse project;
- repository ownership;
- branch discipline;
- multi-agent fan-in;
- tool permission boundaries;
- completion audit;
- documentation/source synchronization.

Поэтому агенты могли формально соблюдать command formatting и всё равно делать дорогие системные ошибки.

## 9. Главная рекомендация по новому prompt

Base prompt должен быть:
- универсальным;
- коротким относительно всей project документации;
- жёстким по process invariants;
- свободным от IP/paths/board-specific values.

Проектная конкретика должна жить:
- в Project instructions;
- repository `AGENTS.md`;
- `STATE.md/TASKS.md`;
- process/runbook docs;
- untracked local context.

Base prompt не должен превращаться в энциклопедию FH8626.

## 10. Рекомендуемая runtime hierarchy

1. Current user instruction.
2. Live repository/tool/hardware facts.
3. Project/account instructions.
4. Repository `AGENTS.md`.
5. Current `STATE.md/TASKS.md`.
6. Local environment context.
7. Historical handoff/summary.
8. Model memory/assumption.

## 11. Что делать с machine-specific параметрами

IP, usernames, WSL paths, Downloads path, toolchain path, target control lane и другие local facts не должны быть частью universal prompt.

Рекомендуется:
- tracked `LOCAL_AGENT_CONTEXT.example.md`;
- actual `LOCAL_AGENT_CONTEXT.md` — untracked / Project-provided;
- repository bootstrap объявляет REQUIRED/OPTIONAL;
- если REQUIRED context отсутствует, агент один раз сообщает degraded-start condition и не угадывает значения.

## 12. Финальный verdict

Проект ускорился не тогда, когда агенты начали писать больше кода, а когда:
1. state перестал жить только в чате;
2. evidence стал воспроизводимым;
3. roles/ownership стали явными;
4. validation перестала быть бинарным «работает/не работает»;
5. пользователь перестал быть file/context router;
6. agents получили прямые Git/reverse surfaces;
7. cross-agent knowledge начал проходить через shared authority.

Следующий существенный выигрыш теперь даст не ещё больше orchestration infrastructure, а **сокращение и стабилизация bootstrap contract**: Base Prompt v2 + repository AGENTS + local context contract.

После этого процесс уже достаточно зрелый, чтобы новые ошибки считать отклонениями от системы, а не отсутствием самой системы.
