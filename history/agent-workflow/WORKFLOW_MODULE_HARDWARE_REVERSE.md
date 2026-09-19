# Workflow module — hardware reverse / embedded bring-up

Этот модуль подключается только для embedded/reverse проектов. Он **не входит** в account-level prompt.

1. Не начинай reverse заново, если existing contract/corpus отвечает на вопрос.
2. Для сложного binary используй semantic tooling: decompiler, CFG, callers/callees, XREF, types, globals, strings, unresolved indirect flow.
3. Псевдо-C — для понимания; ASM/instruction evidence — для критического proof.
4. Неразрешённый indirect edge остаётся explicit `UNRESOLVED`.
5. Перед mutation reverse DB проверь canonical project/program; не создавай competing project для тех же binaries.
6. External/related SoC используй как semantic oracle, но target ABI/MMIO/layout подтверждай на target.
7. Когда static evidence исчерпан, сформулируй точный runtime evidence contract.
8. Capture once, analyze offline.
9. Hardware experiment: baseline → action → observation → rollback → postcondition.
10. Physical/visual evidence при конфликте выше software status/log.
11. Stateful media/hardware stack должен иметь явного owner-а; не размножай competing processes/opens.
12. Feature считается implemented только если имеет proven target contract, доказанно совместимый retained provider либо explicit unsupported.
