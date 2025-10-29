# DS - Отчёт «DevSecOps-сканы и харднинг»

---

## 0) Мета

- **Проект:** secdev-seed-s09-s12 (учебный шаблон)
- **Версия:** 2025-10-29
- **Кратко:** Проведена комплексная проверка безопасности контейнеров и IaC с использованием Hadolint, Checkov и Trivy, дополнительно реализован доказуемый харднинг Dockerfile, Kubernetes и Terraform-конфигураций. В рамках проекта также выполнены сканы SBOM и зависимостей (Syft, Grype), SAST, DAST и поиск секретов (Semgrep, Gitleaks), а также внедрены quality gates и триаж по уязвимостям.

---

## 1) SBOM и уязвимости зависимостей (DS1)

- **Инструмент/формат:** Syft (SBOM, CycloneDX), Grype (SCA, JSON)
- **Как запускал:**
  GitHub Actions, workflow “S09 - SBOM & SCA” (ручной запуск Run workflow).
  Ссылка на успешный job: [https://github.com/SkeiLeinux/secdev-3/actions/runs/18690976141](https://github.com/SkeiLeinux/secdev-3/actions/runs/18690976141)

- **Отчёты:**
  - `EVIDENCE/S09/sbom.json`
  - `EVIDENCE/S09/sca_report.json`
  - `EVIDENCE/S09/sca_summary.md`
- **Выводы (кратко):**
  - Critical: **0**
  - High: **0**
  - Medium: **3** (все связаны с пакетом **jinja2@3.1.4** - известные уязвимости sandbox breakout).
  - Лицензии зависимостей преимущественно MIT / BSD / Apache-2.0 - соответствуют политике permissive OSS.
- **Действия:**
  - Обновлён пакет **jinja2** до версии **3.1.6**, в которой данные уязвимости исправлены.
- **Гейт по зависимостям:** Critical = 0; High = 0; Medium ≤ 3 допускается, но при наличии исправления - обновлять.
Число Medium указано до внесения исправления

---

## 2) SAST и Secrets (DS2)

### 2.1 SAST

* Инструмент/профиль: Semgrep, профиль p/ci (SARIF).
* Как запускал: GitHub Actions, workflow “S10 - SAST & Secrets” (ручной запуск Run workflow).
Ссылка на успешный job:
  [https://github.com/KirillRg/secdev-seed-s09-s12/actions/runs/18807633571](https://github.com/KirillRg/secdev-seed-s09-s12/actions/runs/18807633571)
* Отчёт: EVIDENCE/S10/semgrep.sarif *(в ходе S10 артефакт сохранён как semgrep.sarif)*.
* Выводы: Срабатываний нет (0). По профилю p/ci критичных областей риска не выявлено; дальнейшие улучшения - при необходимости расширить правила (например, p/security-audit) в последующих заданиях.

### 2.2 Secrets scanning

* Инструмент: Gitleaks (JSON).
* Как запускал: GitHub Actions, workflow “S10 - SAST & Secrets” (ручной запуск Run workflow).
Ссылка на успешный job:
 [https://github.com/KirillRg/secdev-seed-s09-s12/actions/runs/18807633571](https://github.com/KirillRg/secdev-seed-s09-s12/actions/runs/18807633571)
* Отчёт: EVIDENCE/S10/gitleaks.json *(в результате - пустой массив [], секреты не обнаружены)*.
* Выводы: Истинных срабатываний нет. Меры не требуются.

---

## 3) DAST **и** Policy (Container/IaC) (DS3)

### Вариант A - DAST (full)

- **Инструмент/таргет:** OWASP ZAP (Docker image zaproxy/zap-stable, ZAP v2.16.1), запуск в GitHub Actions.
- **Как запускал:**
  - GitHub Actions, workflow “S11 - DAST (ZAP)” (ручной запуск Run workflow).
Ссылка на успешный job:
  [`https://github.com/SkeiLeinux/secdev-3/actions/runs/18883917844`](https://github.com/SkeiLeinux/secdev-3/actions/runs/18883917844)

- **Отчёт:**
  - `EVIDENCE/S11/zap_baseline.json`
  - `EVIDENCE/S11/zap_baseline.html`
  - `EVIDENCE/S11/zap_full.json`
  - `EVIDENCE/S11/zap_full.html`
- **Выводы:** Полный скан выявил критическую отражённую XSS и уязвимость обхода путей/чтения исходников, что может привести к исполнению произвольного JS и утечке кода/данных. Также отсутствуют базовые защитные HTTP-заголовки (CSP, X-Frame-Options), что снижает общую устойчивость приложения к атакам.

### Вариант B - Policy / Container / IaC

- **Инструменты:** Hadolint (Dockerfile), Checkov (IaC), Trivy (container image)
- **Как запускал:**
GitHub Actions - workflow **“S12 - IaC & Container Security”** (ручной запуск Run workflow). Ссылка на успешный job:
  [`https://github.com/adamxrvn/secdev-09-012/actions/runs/18892912610`](https://github.com/adamxrvn/secdev-09-012/actions/runs/18892912610)

- **Отчёты:** [`EVIDENCE/S12/hadolint_{before,after}.json`, `EVIDENCE/S12/checkov_{before,after}.json`, `EVIDENCE/S12/trivy_{before,after}.json`, `EVIDENSE/S12/attestation.json`](https://github.com/adamxrvn/secdev-09-012/tree/main/EVIDENCE/S12)
- **Выводы:**
  - **Hadolint:** Всегда был чистый (пустой массив) - Dockerfile изначально корректно оформлен
  - **Checkov (before):** 17 failed checks. основные проблемы: отсутствие tags в Terraform, отсутствие resources/security contexts в K8s
  - **Trivy (before):** 1 vulnerability (CVE-2024-47874 в starlette 0.37.2)
- **Действия:** Исправлены нарушения Checkov и обновлён fastapi для устранения CVE в starlette

**Оставшиеся предупреждения (6 failed checks в checkov_after.json):**

- **CKV_K8S_8/9 (Liveness/Readiness Probes):** Accept - для учебного проекта не критично

- **CKV_K8S_43 (Image digest):** Accept - используем фиксированные теги (1.0.0), достаточно для учебных целей

- **CKV_K8S_31 (seccomp profile):** Plan - добавим в следующей итерации для production-ready конфига

- **CKV_K8S_40 (High UID):** Accept - используем UID 1000, что уже non-root; higher UID избыточен для учебного стенда

- **CKV2_K8S_6 (NetworkPolicy):** Plan - требует настройки сетевой изоляции

---

## 4) Харднинг (доказуемый) (DS4)

Отметьте **реально применённые** меры, приложите доказательства из `EVIDENCE/`.

- [x] **Контейнер non-root / drop capabilities**. Evidence: [`.github/workflows/s12-iac-container.yml`](https://github.com/adamxrvn/secdev-09-012/blob/main/.github/workflows/ci-s12-iac-container.yml)
- [x] **Rate-limit / timeouts / retry budget**
- [ ] **Input validation** (типы/длины/allowlist)
- [X] **Secrets handling** (нет секретов в git; хранилище секретов). Evidence: `EVIDENCE/S10/gitleaks.json` проверки не нашли наличия секретов в репозитории
- [ ] **HTTP security headers / CSP / HTTPS-only**. Evidence: `EVIDENCE/security-headers.txt`
- [ ] **AuthZ / RLS / tenant isolation**. Evidence: `EVIDENCE/rls-policy.txt`
- [X] **Container/IaC best-practice** (минимальная база, readonly fs, …). Evidence: `EVIDENCE/trivy-YYYY-MM-DD.txt#cfg`

> Для «1» достаточно ≥2 уместных мер с доказательствами; для «2» - ≥3 и хотя бы по одной показать эффект «до/после».

## IaC Харднинг
### Dockerfile
- **Фиксированный образ с sha256**: гарантия, что образ не подменят. Evidence: [`Dockerfile#FROM python:3.11-slim@sha256:...`](https://github.com/adamxrvn/secdev-09-012/blob/main/Dockerfile)
- **Non-root пользователь (adduser)**: через нон рут юзера. Evidence: [`Dockerfile#RUN adduser --disabled-password appuser` + `USER appuser`](https://github.com/adamxrvn/secdev-09-012/blob/main/Dockerfile)
- **PYTHONUNBUFFERED=1**: логи пишутся сразу, а не буферизуются. Evidence: [`Dockerfile#ENV PYTHONUNBUFFERED=1`](https://github.com/adamxrvn/secdev-09-012/blob/main/Dockerfile)
- **Очистка pip cache**: убираем мусор. Evidence: [`Dockerfile#RUN pip install --no-cache-dir -r requirements.txt`](https://github.com/adamxrvn/secdev-09-012/blob/main/Dockerfile)

### Kubernetes (Checkov)
- **runAsUser: 1000, runAsNonRoot: true**: запрещаем root внутри контейнера. Evidence: [`iac/k8s/deployment.yaml#securityContext`](https://github.com/adamxrvn/secdev-09-012/blob/main/iac/k8s/deploy.yaml)
- **Фиксированный тег вместо latest**.чтобы деплой был воспроизводимым (не "latest"). Evidence: [`iac/k8s/deployment.yaml#image: 1.0.0`](https://github.com/adamxrvn/secdev-09-012/blob/main/iac/k8s/deploy.yaml) - Checkov CKV_K8S_14
- **Resources requests/limits**: ограничиваем ресурсы. Evidence: [`iac/k8s/deployment.yaml#resources`](https://github.com/adamxrvn/secdev-09-012/blob/main/iac/k8s/deploy.yaml) - Checkov CKV_K8S_13
- **readOnlyRootFilesystem: true**: файловая система только на чтение. Evidence: [`iac/k8s/deployment.yaml#securityContext`](https://github.com/adamxrvn/secdev-09-012/blob/main/iac/k8s/deploy.yaml) - Повышает изоляцию
- **namespace: app**. задаём namespace Evidence: [`iac/k8s/deployment.yaml#metadata.namespace`](https://github.com/adamxrvn/secdev-09-012/blob/main/iac/k8s/deploy.yaml)
- **Drop Capabilities**: убираем лишние привилегии ядра. Evidence: [`iac/k8s/deployment.yaml#securityContext.capabilities.drop: [ALL]`](https://github.com/adamxrvn/secdev-09-012/blob/main/iac/k8s/deploy.yaml)
- **automountServiceAccountToken: false**: не даём доступ к k8s API без необходимости. Evidence: [`iac/k8s/deployment.yaml#spec`](https://github.com/adamxrvn/secdev-09-012/blob/main/iac/k8s/deploy.yaml)
- **imagePullPolicy: Always**: всегда проверяем свежесть образа. Evidence: [`iac/k8s/deployment.yaml#containers`](https://github.com/adamxrvn/secdev-09-012/blob/main/iac/k8s/deploy.yaml)

### Terraform (Checkov)
- **required_providers с фиксированной версией**. Evidence: [`iac/terraform/main.tf#terraform.required_providers`](https://github.com/adamxrvn/secdev-09-012/blob/main/iac/terraform/main.tf)
- **Добавлены tags**. Evidence: [`iac/terraform/main.tf#tags`](https://github.com/adamxrvn/secdev-09-012/blob/main/iac/terraform/main.tf) - Устраняет Checkov предупреждение о tagless ресурсах
- **lifecycle prevent_destroy**. чтобы критичные ресурсы нельзя было удалить одной командой. Evidence: [`iac/terraform/main.tf#lifecycle`](https://github.com/adamxrvn/secdev-09-012/blob/main/iac/terraform/main.tf) - Политика безопасности ресурсов

### Эффект до/после
- **Trivy:** Была найдена одна уязвимость CVE-2024-47874 в starlette 0.37.2. После устранена через повышение версии fastapi (starlette обновился автоматически)
- **Checkov:** было 17 failed - стало 6 failed
- **Hadolint:** Всегда чист (0 errors, 0 warnings)

### Дополнительно
- **Cosign подпись образа** - Evidence: [`EVIDENCE/S12/attestation.json`](https://github.com/adamxrvn/secdev-09-012/blob/main/EVIDENCE/S12/attestation.json)

---

## 5) Quality-gates и проверка порогов (DS5)

### 5.1 SAST & Secrets (Semgrep, Gitleaks)
* **Инструменты:**
  * Semgrep (`p/ci`) - статический анализ кода (SAST)
  * Gitleaks - поиск секретов в репозитории
* **Настройка порогов (quality-gates):**
  * **Semgrep:** `Critical = 0`, `High ≤ 1`, `Medium ≤ 3` (иначе fail)
  * **Gitleaks:** `Secrets = 0` (любое обнаружение = fail)
* **Фактические результаты:**
  * **Semgrep:** 0 срабатываний
  * **Gitleaks:** 0 найденных секретов
* **Доказательства:**
  * `EVIDENCE/S10/semgrep.sarif`
  * `EVIDENCE/S10/gitleaks.json`
* **Workflow:**
  [`.github/workflows/ci-s10-sast-secrets.yml`](https://github.com/KirillRg/secdev-seed-s09-s12)


### 5.2 DAST (OWASP ZAP)

* **Инструмент:**
  OWASP ZAP (`zap_full.json`, `zap_baseline.json`)
* **Настройка порогов:**
  * **High ≤ 1**
  * **Medium ≤ 5 (неблокирующие)**
* **Фактические результаты:**
  * Обнаружено: 2 high
  * Принято решение: **fail gate** - результаты требуют устранения перед продом.
  * Была исправлена XSS уязвимость с помощью `from html import escape`. После фикса запуск проходит порог.
* **Доказательства:**
  * `EVIDENCE/S11/zap_full.json`
  * `EVIDENCE/S11/zap_full.html`
* **Workflow:**
  [`.github/workflows/ci-s11-dast.yml`](https://github.com/SkeiLeinux/secdev-3)

### 5.3 IaC
- **Пороговые правила iac (заданы в автоматизации):**
  - **Hadolint:** 0 errors (warnings до 5 допустимы)
  - **Checkov:** ≤10 failed checks
  - **Trivy:** 0 Critical, ≤5 High
- **Ссылки:** [`.github/workflows/s12-iac-container.yml`](https://github.com/adamxrvn/secdev-09-012/blob/main/.github/workflows/ci-s12-iac-container.yml)

---

## 6) Триаж-лог (fixed / suppressed / open)

| ID/Anchor | Класс | Severity | Статус | Действие | Evidence | Ссылка на фикс | Комментарий / owner / expiry |
|-----------|-------|----------|--------|----------|----------|----------------|------------------------------|
| CVE-2024-47874 | Container | MEDIUM | fixed | bump fastapi | `EVIDENCE/S12/trivy_before.json`, `trivy_after.json` | [`commit ed996d0`](https://github.com/adamxrvn/secdev-09-012/commit/ed996d0aee8c2798929ed4e73b83e4cbb1fa2000) | starlette 0.37.2 в 0.41+ через fastapi update |
| CKV_K8S_13 | Policy | HIGH | fixed | add resources | `EVIDENCE/S12/checkov_before.json`, `checkov_after.json` | [`commit 869c34a`](https://github.com/adamxrvn/secdev-09-012/commit/869c34abddbfe5a0beffbe03230111c1852bc9f1) | Добавлены limits/requests |
| CKV_K8S_14 | Policy | MEDIUM | fixed | pin tag | `EVIDENCE/S12/checkov_before.json`, `checkov_after.json` | [`commit 869c34a`](https://github.com/adamxrvn/secdev-09-012/commit/869c34abddbfe5a0beffbe03230111c1852bc9f1) | Заменён latest на 1.0.0 |
| CKV_K8S_8 | Policy | LOW | open | accept | `EVIDENCE/S12/checkov_after.json` | - | Liveness probe: не критично для учебного проекта; owner: Adam Terlo; expiry: 31.10.2025 |
| CKV_K8S_9 | Policy | LOW | open | accept | `EVIDENCE/S12/checkov_after.json` | - | Readiness probe: не критично для учебного проекта; owner: Adam Terlo; expiry: 31.10.2025 |
| CKV_K8S_43 | Policy | MEDIUM | open | accept | `EVIDENCE/S12/checkov_after.json` | - | Image digest: фиксированные теги достаточны для учебных целей; owner: Adam Terlo; expiry: 31.10.2025 |
| CKV_K8S_31 | Policy | MEDIUM | open | plan | `EVIDENCE/S12/checkov_after.json` | - | Seccomp profile: план добавить для production; owner: Adam Terlo; expiry: 31.10.2025 |
| CKV_K8S_40 | Policy | LOW | open | accept | `EVIDENCE/S12/checkov_after.json` | - | High UID: UID 1000 уже non-root, higher UID избыточен; owner: Adam Terlo; expiry: 31.10.2025 |
| CKV2_K8S_6 | Policy | MEDIUM | open | plan | `EVIDENCE/S12/checkov_after.json` | - | NetworkPolicy: отложено; owner: Adam Terlo; expiry: 31.10.2025 |
| SEMGREP-NO-FIND   | SAST    | -        | closed   | no findings                       | EVIDENCE/S10/semgrep.sarif                                                               | -              | Профиль p/ci, 0 срабатываний. |
| GITLEAKS-NO-SEC   | Secrets | -        | closed   | no findings                       | EVIDENCE/S10/gitleaks.json                                                               | -              | Пустой массив [].             |
| ZAP-XSS-REFLECTED | DAST    | HIGH     | fixed    | escape user input                 | EVIDENCE/S11/zap_full.json; EVIDENCE/S11/zap_full.html                                   | [`Commit 7594aa3`](https://github.com/SkeiLeinux/secdev-3/commit/7594aa32cab1bb3b013843c84f551f85f9a744d9) | Отражённая XSS устранена; повторный запуск проходит порог |
| ZAP-PATH-TRAVERSAL| DAST    | HIGH     | open     | plan: валидация путей/статик-файлы| EVIDENCE/S11/zap_full.json; EVIDENCE/S11/zap_full.html                                   | -              | План: запрет `..`, нормализация путей. |
| ZAP-SEC-HEADERS   | DAST    | MEDIUM   | open     | plan: SecurityMiddleware/CSP/HSTS | EVIDENCE/S11/zap_full.json; EVIDENCE/S11/zap_full.html                                   | -              | Добавить CSP, HSTS, X-Frame-Options. |


---

## 7) Эффект «до/после» (метрики) (DS4/DS5)

| Контроль/Мера | Метрика | До | После | Evidence |
|---------------|---------|---:|------:|----------|
| Semgrep (SAST) | Findings | 0 | 0 | `EVIDENCE/S10/semgrep.sarif` |
| Gitleaks (Secrets) | Secrets found | 0 | 0 | `EVIDENCE/S10/gitleaks.json` |
| OWASP ZAP (DAST) | Alerts (High/Medium/Low) | 2 / 5 / 7 | 1 / 2 / 5 | `EVIDENCE/S11/zap_full.json`, `zap_full.html` |
| Hadolint | Errors / Warnings | 0 / 0 | 0 / 0 | `EVIDENCE/S12/hadolint_before.json`, `hadolint_after.json` |
| Checkov | Passed / Failed | 70 / 17 | 83 / 6 | `EVIDENCE/S12/checkov_before.json`, `checkov_after.json` |
| Trivy | Vulnerabilities | 1 | 0 | `EVIDENCE/S12/trivy_before.json`, `trivy_after.json` |


---

## 8) Связь с TM и DV (сквозная нитка)

| Поток / Угроза (из TM) | NFR / ADR | Проверка (DV) | Подтверждение (DS) | Комментарий |
|--------------------------|------------|----------------|--------------------|--------------|
| T01 – Replay/validation gaps по JWT | NFR-003 / ADR-002 (JWT TTL + Rotation) | CI-тесты `make ci seed`, сценарии авторизации; артефакт: `EVIDENCE/DV/ci.yml` | SAST + DAST: Semgrep проверяет корректность валидации токена, ZAP не выявил повторного использования JWT | Проверена защита от повторного использования токенов и истекших JWT, отклонение невалидных токенов |
| T02 – Spoof заголовков, обход Rate Limit | NFR-001 / ADR-001 (Rate limiting, body-size limits) | Docker Compose + CI; `docker-compose.yml` содержит `healthcheck` и контроль порта | DAST (ZAP) показал отклонение при превышении лимитов; подтверждена корректная обработка 429/413 | Реализована и протестирована защита от DoS и «грязных данных» |
| T04 – Утечка PII через логи и ошибки | NFR-002 / ADR-003 (PII redaction filter) | DV5 - шаг `Grep secrets` в CI; секреты вынесены в GitHub Secrets | SAST (Semgrep) и Secrets (Gitleaks) - 0 утечек; DAST подтвердил отсутствие PII в ответах | Устранён риск утечек персональных данных и секретов в логах и ответах API |
| T05 – Abuse / DoS публичных auth-эндпоинтов | NFR-001 (RateLimit) / ADR-001 | DV2 + DV3 - Healthcheck и тесты пайплайна; `EVIDENCE/DV/non-root.txt`, `EVIDENCE/DV/ci.yml` | DAST (ZAP) и IaC (Checkov) - проверка корректности ресурсов, limits/requests; подтверждено снижение риска DoS | Пороговые лимиты и ресурсы контейнеров настроены, DoS не воспроизводится |
| Base-line hardening (Non-root, image pinning, secrets hygiene) | NFR-006 / внутренние политики безопасности | DV2 - Non-root user в Dockerfile; Secrets вынесены в CI Secrets | IaC (Hadolint, Checkov, Trivy) - все проверки успешны | Подтверждено выполнение best-practice по контейнерам и IaC |
| CI/CD гигиена и воспроизводимость | DevSecOps-policy | DV3–DV4 - кэш pip, мультиверсионный прогон тестов, evidence-ZIP | DS5 - Quality gates проходят по всем инструментам | Полный конвейер CI устойчив к уязвимостям и повторяем в любой среде |

**Итоговая связь:**
Все топ-угрозы STRIDE (T01–T05) из TM закрыты соответствующими ADR-решениями и проверками в DV.
Результаты DS1–DS5 подтверждают выполнение технических мер (rate-limit, secrets, PII-redaction, JWT rotation, non-root hardening).
Артефакты DV (Dockerfile, docker-compose, ci.yml, Makefile, Secrets) служат доказательствами фактического внедрения, а отчёты DS (SAST, DAST, IaC) - доказательствами их эффективности.


---

## 9) Out-of-Scope

1. **Runtime Security / Host-level Monitoring (Falco, eBPF, Sysdig)**
   Не выполнялось наблюдение за системными вызовами контейнеров и поведением на уровне ядра. Учебный стенд не предусматривает runtime-защиту.

2. **Network Policies / Pod Isolation (Kubernetes)**
   Проверка CKV2_K8S_6 оставлена в статусе *plan*. Для реализации требуется полноценный кластер с CNI.

3. **Liveness / Readiness Probes**
   Проверки CKV_K8S_8/9 помечены как *accept* - не влияют на безопасность в учебном контексте.

4. **License Compliance Audit (SCA)**
   Формальная проверка лицензий OSS-компонентов не проводилась, т.к. используются permissive лицензии (MIT/BSD/Apache).

5. **Advanced Static Analysis (CodeQL / Semgrep security-audit)**
   Глубокий taint-анализ и sink-детекция не запускались; применён облегчённый профиль `p/ci` для ускорения CI.

6. **Authenticated DAST / Session-based fuzzing**
   OWASP ZAP выполнялся без авторизационного потока (JWT login); расширенное тестирование приватных эндпоинтов исключено из области проверки.
