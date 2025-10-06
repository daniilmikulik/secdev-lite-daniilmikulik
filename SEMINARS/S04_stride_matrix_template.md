# S04 - Шаблон STRIDE per element (матрица)

## Матрица угроз STRIDE

| Element                                   | Data/Boundary            | Threat (S/T/R/I/D/E) | Description                                                 | NFR link (ID)    | Mitigation idea (ADR later)                     |
| ----------------------------------------- | ------------------------ | -------------------- | ----------------------------------------------------------- | ---------------- | ----------------------------------------------- |
| **USER_AGENT**                            | Клиент/Браузер           | S                    | Подмена легитимного пользователя через кражу учетных данных | NFR-009          | MFA + короткоживущие JWT токены                 |
| **USER_AGENT**                            | Клиент/Браузер           | R                    | Отрицание операций из-за недостаточного аудита действий     | NFR-008          | Централизованный аудит с WORM-хранилищем        |
| **Edge: USER_AGENT → Wishlist_GW**        | JWT/HTTPS/correlation_id | S                    | Подделка/перехват JWT токенов при передаче                  | NFR-009          | JWT с коротким TTL + refresh tokens             |
| **Edge: USER_AGENT → Wishlist_GW**        | JWT/HTTPS/correlation_id | T                    | Изменение параметров запроса для обхода лимитов пагинации   | NFR-003          | Строгая валидация limit/offset на сервере       |
| **Edge: USER_AGENT → Wishlist_GW**        | JWT/HTTPS/correlation_id | D                    | DDoS атака на API эндпойнты                                 | NFR-001, NFR-002 | Rate limiting + WAF на API Gateway              |
| **Wishlist_GW**                           | API Gateway              | D                    | Перегрузка большим количеством одновременных запросов       | NFR-001, NFR-002 | Автоскейлинг + circuit breaker                  |
| **Wishlist_GW**                           | API Gateway              | I                    | Утечка данных через детализированные ошибки API             | NFR-003          | RFC 7807 без stack traces                       |
| **Edge: Wishlist_GW → Wishlist_Service**  | DTO / correlation_id     | I                    | Перехват конфиденциальных данных между микросервисами       | NFR-005, NFR-007 | mTLS между сервисами                            |
| **Wishlist_Service**                      | Бизнес-логика            | E                    | Обход RBAC через уязвимости в бизнес-логике                 | NFR-005, NFR-006 | Проверка прав на каждом уровне + unit-тесты     |
| **Wishlist_Service**                      | Бизнес-логика            | T                    | Изменение логики для обхода tenant isolation                | NFR-006          | Strict RBAC policies + code review              |
| **Edge: Wishlist_Service → PostgreSQL**   | SQL/ORM                  | T                    | SQL-инъекции через непараметризованные запросы              | NFR-010          | Prepared statements + ORM validation            |
| **Edge: Wishlist_Service → PostgreSQL**   | SQL/ORM                  | I                    | Нарушение изоляции тенантов на уровне БД                    | NFR-005          | Row-level security + tenant_id в каждом запросе |
| **PostgreSQL**                            | База данных              | I                    | Прямой доступ к БД с данными всех тенантов                  | NFR-005, NFR-007 | Database encryption + strict access controls    |
| **PostgreSQL**                            | База данных              | D                    | Отказ службы из-за ресурсоемких SQL-запросов                | NFR-010          | Query timeouts + connection pooling             |
| **Edge: Wishlist_Service → Logs_Service** | JSON logs/traces         | I                    | Перехват логов с немасскированными PII                      | NFR-007          | PII masking перед отправкой в логи              |
| **Edge: Wishlist_Service → Logs_Service** | JSON logs/traces         | R                    | Потеря аудит-трейла при передаче                            | NFR-008          | Guaranteed delivery через message queue         |
| **Logs_Service**                          | Административный сервис  | I                    | Доступ к логам с конфиденциальной информацией               | NFR-007          | Role-based access to logs + encryption          |
| **Logs_Service**                          | Административный сервис  | T                    | Модификация/удаление аудит-логов                            | NFR-008          | Immutable log storage (WORM)                    |
| **Edge: Logs_Service → Logs_PostgreSQL**  | SQL/ORM                  | T                    | Изменение исторических логов в хранилище                    | NFR-008          | Append-only database configuration              |
| **Edge: Logs_Service → Logs_PostgreSQL**  | SQL/ORM                  | I                    | Утечка аудит-логов с PII из хранилища                       | NFR-007          | Column-level encryption для PII полей           |
| **Logs_PostgreSQL**                       | База данных логов        | I                    | Неавторизованный доступ к полным аудит-логам                | NFR-007, NFR-008 | Database encryption + strict access controls    |

## Ключевые наблюдения

### Наиболее уязвимые элементы:

1. **USER_AGENT → Wishlist_GW** - граница доверия с внешним миром
2. **Wishlist_Service** - центральный узел бизнес-логики и RBAC
3. **PostgreSQL** - основное хранилище конфиденциальных данных

### Критические NFR для защиты:

- **NFR-009** (AuthN) - защита от spoofing
- **NFR-005/006** (AuthZ/RBAC) - защита от elevation of privilege
- **NFR-001/002** (Performance/Scaling) - защита от DoS
- **NFR-007** (Privacy/PII) - защита от information disclosure
- **NFR-008** (Auditability) - защита от repudiation

Все митигации готовы для преобразования в ADR на следующем этапе.
