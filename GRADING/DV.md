# DV - Мини-проект «DevOps-конвейер»

## DV1

Для запуска требуется наличие файлов requirements.txt (см. EVIDENCE/DV) и .env(пример в EVIDENCE/DV)

### Зависимости

```
fa
stapi==0.115.0
uvicorn==0.30.1
jinja2==3.1.4
pydantic==2.9.2
pytest==8.3.2
httpx==0.27.2
pytest-cov==7.0.0
```

### Команды для запуска (EVIDENCE/DV/Makefile):

- **Build & run**

```bash
make run seed
```

- **Tests**

```bash
make test seed
```

- **Build + Tests**

```bash
make ci seed
```

`---`

## DV2

Добавлен и использован docker-compose (см EVIDENCE/DV)

- В docker-compose добавлен healthcheck

docker-compose:
```dockerfile
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/"]
  interval: 10s
  timeout: 3s
  retries: 5
```

Dockerfile:
```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
```

- В Dockerfile добавлен хардеринг S07-3 - Non-root user (см. EVIDENCE/DV/non-root.txt)

```dockerfile
RUN useradd -m -u 10001 appuser && chown -R appuser:appuser /app
USER appuser
```

### Запуск

Для запуска необходимы (см. EVIDENCE/DV):
+ .env
+ requirements.txt
+ Dockerfile
+ docker-compose.yml

```bash
docker compose up --build -d
```

## DV3

### Логи успешного CI

**Где найти:** EVIDENCE/DV/evidence-s08-3.10.zip; EVIDENCE/DV/evidence-s08-3.11.zip;

**Файл CI.yml:** EVIDENCE/DV/ci.yml

**Что сделано в CI:**
+ Кэш зависимостей
+ Прогон тестов на 2-х версиях Python (3.10 и 3.11)

```yml
- name: Cache pip
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

```yml
strategy:
  fail-fast: false
  matrix:
    python-version: ['3.10', '3.11']
```

## DV4

**Артефакты конвейера(для каждой версии Python свои):**
+ ZIP-архив
  + test-report.xml — Результаты тестирования (JUnit)
  + coverage.xml – Покрытие тестами
  + ci-log.txt – Метаданные CI (время, коммит, версия Python)

**Где найти:** EVIDENCE/DV/evidence-s08-3.10.zip; EVIDENCE/DV/evidence-s08-3.11.zip;

## DV5

Для обеспечения гигиены секретов было сделано:
+ Размещение секретов в Github Secrets
+ Использование секретов в ci.yml
+ Добавлен шаг в CI с проверкой git grep

![Снимок экрана 2025-10-28 в 03.55.13.png](../EVIDENCE/DV/%D0%A1%D0%BD%D0%B8%D0%BC%D0%BE%D0%BA%20%D1%8D%D0%BA%D1%80%D0%B0%D0%BD%D0%B0%202025-10-28%20%D0%B2%2003.55.13.png)

```yaml
env:
  APP_NAME: ${{ secrets.APP_NAME }}
  PORT: ${{ secrets.PORT }}
```

```yaml
- name: Grep secrets
  run: git grep -n "SECRET" > EVIDENCE/grep-secrets.txt || true
```

См. EVIDENCE/DV/ci.yml;