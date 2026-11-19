> 🚀 [Открыть интерактивную документацию Swagger UI (Live Demo)](https://rinata03.github.io/b2b-support-api-portfolio/)

# Сервис оформления и выдачи справок и выписок (L2 Support & API)

**Косолапова Рината** • [Telegram](https://t.me/rinatakosolapova) • [Email](mailto:rinatakocneva@gmail.com)

Демонстрационный проект в финтех-домене (выписки по счетам, справки об остатках, налоговые справки). Показывает полный цикл технического сопровождения и интеграции REST API сервиса — от чтения контракта и воспроизведения запросов в Postman до диагностики инцидентов в логах, SQL и оформления эскалаций на L3.

---

## 🔄 Сквозной сценарий работы сервиса
1. Смежная система (CRM, B2B-портал) отправляет запрос на обработку и факсимильное подписание справки/выписки (`POST /documents/process`).
2. Сервис проверяет JWT, лимиты и валидность данных, защищает заявку от дублей (`X-Idempotency-Key`) и передает ее во внешний сервис подписания (`SignSvc`).
3. После завершения обработки клиенту отправляется Webhook-уведомление о готовности (`STATEMENT_READY`). При сбоях L2-инженер локализует причину по Runbook.

---

## 📂 Структура проекта

### [1. Спецификация REST API и архитектура взаимодействия](./01-api-specification/README.md)
* **Инструменты:** OpenAPI 3.0.3 (YAML), Swagger UI, Mermaid, RFC 7807 (Problem Details).
* **Что внутри:**
  * Контракт [openapi.yaml](./01-api-specification/openapi.yaml) с описанием типов справок (`ACCOUNT_STATEMENT`, `BALANCE_CERTIFICATE`, `TAX_CERTIFICATE`), заголовка идемпотентности и унифицированных ошибок (`400`, `401`, `409`, `422`, `429`, `504`).
  * Sequence-диаграмма прохождения запроса с точками отказа и разделением ответственности между клиентом и бэкендом.

### [2. Интеграционный гайд (Quick Start) и Postman-коллекция](./02-integration-guide/README.md)
* **Инструменты:** Markdown, Postman (Environments, Pre-request scripts, Tests), cURL.
* **Что внутри:**
  * Пошаговый гайд с примером `cURL`, правилами идемпотентности (`X-Cache-Lookup: HIT` / `409 Conflict`) и политикой ретраев при `429` / `504`.
  * Готовая [коллекция Postman (JSON)](./02-integration-guide/document-service.postman_collection.json) с автогенерацией `UUIDv4` и автотестами проверки ответа для быстрого воспроизведения обращений.

### [3. Runbook L2: локализация инцидентов и эскалация на L3](./03-troubleshooting-runbook/README.md)
* **Инструменты:** SQL (PostgreSQL), `kubectl` (логи), Mermaid (Decision Tree), Jira.
* **Что внутри:**
  * Пошаговый алгоритм диагностики недоставки Webhook о готовности справки: SQL-запросы к таблицам `integration_outbox` и `documents`, фильтр логов и дерево решений.
  * Политика повторных попыток (`Exponential Backoff`, `DLQ`) и готовый шаблон баг-репорта для передачи инцидента разработчикам (`504 Gateway Timeout` на `SignSvc`).

---

## 🛠️ Стек и технологии
* **API и контракты:** REST API, OpenAPI 3.0 (Swagger), RFC 7807, JSON, YAML, Base64.
* **Диагностика и БД:** Postman, cURL, Chrome DevTools, SQL (PostgreSQL: `SELECT`, `WHERE`, `ORDER BY`), анализ логов (`kubectl`).
* **Документация и процессы:** Markdown, Mermaid, Git / GitHub, Jira, Confluence.
