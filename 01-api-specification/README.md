> 🚀 [Интерактивная спецификация в Swagger UI (Live Demo)](https://rinata03.github.io/b2b-support-api-portfolio/)

# Этап 1. Контракт REST API и архитектура сервиса «Справки и выписки»

## 📌 Бизнес-контекст
Сервис оформления, валидации и факсимильного подписания электронных справок и выписок банковской платформы (выписки по счетам, справки об остатках, налоговые справки).

Смежным системам (CRM, B2B-портал, мобильное приложение) требуется единый отказоустойчивый API для отправки документов в очередь на подписание с последующим получением результата по Webhook.

## 🎯 Зачем это инженеру L2-поддержки
Четкий API-контракт и единая схема ошибок позволяют саппорту быстро разделять зону ответственности при инцидентах:
* **Ошибка на стороне клиента/интегратора (`400`, `401`, `422`, `429`):** неверный формат Base64, превышен лимит файла, неизвестный `docType` или просрочен JWT — решается на уровне L2 с опорой на поле `invalidParams` без эскалации в разработку.
* **Повторная отправка при сетевом сбое (`201 HIT`, `409`):** механизм `X-Idempotency-Key` исключает создание дубликатов справок.
* **Сбой смежного сервиса (`504`):** таймаут внешнего сервиса подписания `SignSvc` — понятная точка отказа для эскалации или безопасного ретрая.

## 🛠 Что реализовано в спецификации
* Контракт `openapi.yaml` по стандарту **OpenAPI 3.0.3** с описанием схем, ограничений и примеров.
* Единый формат ошибок по стандарту **RFC 7807 (Problem Details)** (`400`, `401`, `409`, `422`, `429`, `504`).
* Механизм идемпотентности через заголовок `X-Idempotency-Key` (UUID v4, TTL 24 часа).
* Sequence-диаграмма прохождения запроса с точками отказа и обработкой таймаутов.

## 🔄 Архитектура вызова и точки отказа (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    participant Client as Клиент / CRM
    participant GW as API Gateway
    participant DocSvc as Document Service
    participant DB as PostgreSQL / Cache
    participant SignSvc as Сервис подписания (SignSvc)

    Client->>GW: POST /documents/process (X-Idempotency-Key)
    GW->>GW: Проверка JWT-токена и Rate Limit (60 req/min)

    alt Невалидный токен или превышен лимит
        GW-->>Client: 401 Unauthorized / 429 Too Many Requests (RFC 7807)
    else Авторизация успешна
        GW->>DocSvc: Проксирование запроса
        DocSvc->>DB: Поиск X-Idempotency-Key (TTL 24ч)

        alt Ключ найден (повторный запрос)
            DB-->>DocSvc: Данные созданной заявки
            DocSvc-->>GW: 201 Created (X-Cache-Lookup: HIT)
            GW-->>Client: 201 Created (X-Cache-Lookup: HIT)
        else Ключ в статусе IN_PROGRESS (Race condition)
            DB-->>DocSvc: Запрос еще выполняется
            DocSvc-->>GW: 409 Conflict (RFC 7807)
            GW-->>Client: 409 Conflict (повторить позже)
        else Новый запрос
            DocSvc->>DocSvc: Валидация схемы и payload (PDF Base64 ≤ 15 МБ)
            alt Ошибка валидации схемы / docType / размера
                DocSvc-->>GW: 400 / 422 Unprocessable Entity (RFC 7807)
                GW-->>Client: 400 / 422 (массив invalidParams)
            else Валидация успешна
                DocSvc->>DB: Запись заявки в БД (Статус: PROCESSING)
                DocSvc->>SignSvc: POST /v1/signatures (постановка на подпись)

                alt Таймаут или сбой внешнего сервиса (5 сек)
                    SignSvc--xDocSvc: Timeout / Connection Drop
                    DocSvc->>DB: Обновление статуса (FAILED)
                    DocSvc-->>GW: 504 Gateway Timeout (RFC 7807)
                    GW-->>Client: 504 Gateway Timeout (Retryable)
                else Успешная постановка в очередь
                    SignSvc-->>DocSvc: 202 Accepted (Sign ID)
                    DocSvc->>DB: Фиксация Sign ID
                    DocSvc-->>GW: 201 Created (documentId, Location)
                    GW-->>Client: 201 Created (documentId, Location)
                end
            end
        end
    end
