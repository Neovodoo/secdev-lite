# #ADR — “PII Masking + Redaction Filter”


**Status:** Proposed
**Owner:** DevSecOps Lead
**Date created:** 2025-10-13


### Context
```
Risk: R-05 (L=3, I=4, Score=12)
DFD: Logs (сквозной поток)
NFR: NFR-002(NFR-003/NFR-004 из EVIDENCE) (Privacy/PII)
Assumptions:
- Все сервисы логируют JSON через единый middleware
- Содержимое логов иногда включает поля DTO и stack traces
- Требования: отсутствие PII/секретов в логах, хранение ≤30 дней
```


### Decision
Реализовать централизованную **маскировку PII и секретов** в логах:
- Param/Policy: применить **redaction filter** на уровне лог-middleware (gateway, service)
- Param/Policy: шаблон denylist ключей → `["password","email","token","ip","salt","secret","key"]`
- Param/Policy: значения заменяются на `"***"` до сериализации в JSON
- Param/Policy: запрет логирования stack trace по умолчанию (error-mapper RFC7807)
- Layer: gateway, service; scope: все endpoints с DTO-логированием
- Storage: ротация логов ≤30 дней, только обезличенные события


### Alternatives
- **B – Whitelist logging:** отказались из-за высокой трудоёмкости и риска потери нужных полей.
- **C – Tokenization:** отклонено — требует сервис токенизации, задержки и сложное сопровождение.


### Consequences
**Положительные:**
- Снижение вероятности утечки PII/секретов в логах.
- Соответствие комплаенсу и политике приватности (GDPR-like).
  **Негативные/издержки:**
- Потеря части контекста для отладки.
- Требуется поддерживать denylist в актуальном состоянии.


### DoD / Acceptance
```gherkin
Given DTO содержит поля password, email, token
When сервис записывает лог уровня INFO/ERROR
Then значения этих полей заменяются на "***", отсутствуют в теле лога
```
Checks:
- **test:** e2e `log-redaction.spec` — проверка шаблонов JSON-логов.
- **log:** поиск по `"password"`/`"email"` → 0 совпадений.
- **scan/policy:** статический анализ шаблонов логов (policy “no-PII”).
- **metric/SLO:** доля логов, прошедших redaction check ≥ 99 %.


### Rollback / Fallback
- Флаг `LOG_REDACTION=false` в конфиге (временное отключение).
- Мониторинг через дашборд `pii_redacted_ratio`.
- При сбоях возвращаемся к plain-logs с ручным фильтром в pipeline.


### Trace
- DFD: S04.md → Logs (сквозной поток)
- STRIDE: строка R-05 (Information Disclosure)
- Risk scoring: S04_risk_scoring.md → R-05, Top-5
- NFR: NFR-008 (Observability/Logging)
- Issue: #SEC-LOG-MASK-2025


### Open Questions
- Требуется ли маскирование внутренних ID (например, user_id)?
- Нужна ли ретроспективная очистка старых логов при rollout?

### 0) Контекст риска

* **Risk-ID:** R-05
* **Threat:** I (Information Disclosure)
* **DFD element/edge:** Logs (сквозной поток)
* **NFR link (ID):** NFR-008 (Observability/Logging)
* **L×I (1-5):** L=3, I=4, Score=12
* **Ограничения/предпосылки:** централизованное логирование (JSON, ELK/CloudWatch), PII встречается в DTO и exception-логах; регуляторные ограничения (GDPR-like retention 30d).

---

### 1) Критерии и шкалы

**Польза (чем больше — тем лучше, ↑):**

* **Security impact (↑):** насколько альтернатива снижает риск (предотвращает/обнаруживает/сдерживает).
* **Blast radius reduction (↑):** насколько сужает возможный ущерб (только один тенант/аккаунт/фича вместо всего сервиса).

**Стоимость/сложность (чем меньше — тем лучше, ↓):**

* **Complexity (↓):** сложность внедрения (код/конфиг/политики).
* **Time-to-mitigate (↓):** сколько времени до эффекта (в днях/спринтах).
* **Dependencies (↓):** внешние/внутренние зависимости (команды, провайдеры, миграции).

> Оценка **1-5**: 1 — минимально / 5 — максимально. Для «стоимостных» критериев **меньше — лучше**.

**Итоговые метрики:**

* **Benefit = Security impact + Blast radius reduction**
* **Cost = Complexity + Time-to-mitigate + Dependencies**
* **Net = Benefit − Cost** *(чем больше, тем привлекательнее)*

---

### 2) Таблица сравнения вариантов

| Alternative | Summary                                                                          | Security impact (↑,1-5) | Blast radius reduction (↑,1-5) | Complexity (↓,1-5) | Time-to-mitigate (↓,1-5) | Dependencies (↓,1-5) | **Benefit** | **Cost** | **Net** | Notes                                                             |
|-------------|----------------------------------------------------------------------------------|------------------------:|-------------------------------:|-------------------:|-------------------------:|---------------------:|------------:|---------:|--------:|-------------------------------------------------------------------|
| **A**       | Маскирование PII + denylist ключей в логах (pattern filter на уровне middleware) |                       4 |                              4 |                  2 |                        2 |                    2 |       **8** |    **6** |  **+2** | Простая реализация, быстрый эффект, без внешних зависимостей.     |
| **B**       | Полное удаление полей PII перед сериализацией логов (whitelist logging)          |                       5 |                              5 |                  4 |                        3 |                    3 |      **10** |   **10** |   **0** | Самое надёжное, но ломает отладку и требует рефакторинга логеров. |
| **C**       | Токенизация/псевдонимизация PII перед логированием                               |                       5 |                              5 |                  5 |                        4 |                    4 |      **10** |   **13** |  **−3** | Высокая защита, но сильно повышает сложность и задержку.          |

---

### 3) Тай-брейкеры при равенстве Net

* **Compliance/Privacy:** влияет на выполнение требований приватности и GDPR.
* **Maintainability:** простота поддержки и расширения denylist.
* **Team fit:** команда уже имеет опыт с JSON-логированием и middleware.
* **Observability:** redaction не мешает метрикам и трассировке.
* **Rollback safety:** легко отключается флагом конфигурации.

---

### 4) Решение (для переноса в ADR)

* **Chosen alternative:** A — PII Masking + Denylist pattern filter.
* **Почему:** даёт достаточную защиту и соответствует принципу “privacy by default” при минимальных затратах и риске регрессий. Реализуется быстро и безопасно, без внешних зависимостей.
* **ADR candidate:** `PII Masking + Redaction Filter`.
* **Связки:** Risk-ID R-05, NFR-ID NFR-008, DFD Logs (сквозной поток).
* **Следующие шаги:**
    1. Добавить redaction middleware на gateway и service.
    2. Определить denylist ключей и шаблон фильтра.
    3. Настроить unit/e2e тесты для проверки маскировки.
    4. Включить мониторинг `pii_redacted_ratio`.
    5. Обновить документацию и политику логирования.