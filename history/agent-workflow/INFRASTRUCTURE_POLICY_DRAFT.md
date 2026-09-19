# Infrastructure policy draft

Status: `PROPOSED`.

Этот документ должен жить в отдельном infrastructure repository, не в camera/project repo.

## Primary agent surface

Для инфраструктурных операций использовать Koba MCP Bridge там, где capability существует:
- GitHub Agent App — mutations;
- GitHub Reviewer App — independent review;
- Ghidra project tools — mutable reverse workspace;
- artifact store — immutable large/binary artifacts;
- structured cURL — HTTP/API/download/stream capture.

Если нужной capability нет — зафиксировать gap. Не обходить Bridge другим connector/tool без явного решения владельца.

## Structured cURL defaults

Koba Bridge предоставляет presets:
- `chrome-desktop` — browser-like desktop Chrome HTTP headers; использовать по умолчанию для human-facing HTML/site requests, когда нужен browser-like request;
- `json-api` — JSON API requests;
- `curl` — raw/native curl semantics;
- `chrome-mobile` — только когда реально нужен mobile HTTP profile;
- `none` — только при осознанном ручном header contract.

Важно: browser presets воспроизводят HTTP headers, **не** JavaScript engine и не Chrome TLS/HTTP2 fingerprint. Если сайту нужен реальный browser execution, curl preset не выдавать за browser.

## GitHub roles

- Source mutation: Agent App.
- Independent verification/review: Reviewer App.
- Reviewer не получает обычный source mutation workflow.
- Reserved/admin operations использовать только узкими sanctioned primitives, не arbitrary force push.
- History rewrite — только controlled operation с expected head, dry-run, tree preservation и independent review.

## Secrets/local state

Не хранить credentials/tokens/private keys в universal agent repo.
Machine/network inventory и sensitive endpoints — private infra/local context layer.
Public docs могут хранить только contracts/templates/examples.

## Infrastructure bootstrap

Infra-agent перед изменениями обязан прочитать:
1. infrastructure repository `AGENTS.md`;
2. current `STATE.md`/`TASKS.md`;
3. tool capability/policy docs;
4. relevant runbook;
5. local infra context, если REQUIRED.
