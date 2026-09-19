# ChatGPT Project instructions — OpenIPC reverse/porting draft

Назначение: короткий bootstrap для проекта. Детальные правила не дублировать здесь; читать canonical universal workflow и repository authority.

1. Для GitHub, Ghidra, HTTP/cURL, artifacts и других поддерживаемых операций использовать **Koba MCP Bridge как primary surface**. Не переключаться на другой GitHub connector или обходной workflow без явного разрешения.
2. Перед работой прочитать repository `AGENTS.md`, актуальные `STATE.md`/`TASKS.md` и указанный ими workflow/authority.
3. Перед universal engineering work прочитать canonical agent workflow repository (после его создания/подключения).
4. Если задача затрагивает Koba MCP Bridge, GitHub Apps, Ghidra hosting, artifact storage, local runners, network/tool defaults или другую инфраструктуру — дополнительно прочитать canonical infrastructure repository.
5. Camera/project-specific facts живут в project repositories; universal agent rules и infrastructure policy здесь не копируются.
6. Git mutations выполнять Agent identity. Для серьёзных архитектурных/cross-repo/hardware-critical/upstream-ready изменений нужен отдельный Reviewer pass/identity.
7. Browser/API-first. Heavy owner build/hardware validation не имитировать в browser-agent среде.
8. Если обязательный local context объявлен repository bootstrap-ом, но недоступен, сообщить один раз и не угадывать machine-specific значения.
