# Course Management — учебное уязвимое приложение + DevSecOps-пайплайн

[![CI](https://github.com/azaleptin41k/rbpo-practice/actions/workflows/build-pipeline.yml/badge.svg)](https://github.com/azaleptin41k/rbpo-practice/actions/workflows/build-pipeline.yml)

Практика по дисциплине «Разработка безопасного ПО» (МТУСИ). Приложение на Spring Boot **намеренно содержит уязвимости**. Задача: встроить проверки безопасности в CI/CD, разобрать находки (реальные и ложные) и оценить, что автоматические сканеры находят, а что пропускают.

> ⚠️ Не разворачивайте это приложение в доступной из интернета среде.

## Стек

| Слой | Инструмент |
|---|---|
| Приложение | Java 21, Spring Boot, Spring Data JPA, Thymeleaf, H2 |
| Сборка | Maven |
| CI/CD | GitHub Actions (в GitLab CI был аналогичный пайплайн, см. REPORT.md) |
| Secret scanning | Gitleaks (сканирует всю историю, останавливает сборку) |
| SAST | Semgrep (наборы `p/java` и `p/owasp-top-ten`, отчёт SARIF → GitHub Security) |
| SCA | OWASP Dependency-Check (HTML-отчёт в артефактах) |

## Пайплайн

```
push / pull request → main
  │
  ├─ Secret Scan (Gitleaks)          — при найденном секрете сборка останавливается
  ├─ Build (Maven)                   — сборка, JAR в артефакты
  ├─ SAST (Semgrep)                  — SARIF в GitHub Security + артефакт
  └─ SCA (OWASP Dependency-Check)    — отчёт по CVE в зависимостях (HTML/JSON)
```

Приложение уязвимо намеренно, поэтому SAST и SCA работают в режиме отчёта и сборку не блокируют. Блокирует только поиск секретов. Для реального проекта пороги описаны ниже, в разделе «Что можно улучшить».

Файл пайплайна: [`.github/workflows/build-pipeline.yml`](.github/workflows/build-pipeline.yml). Промежуточные этапы лабораторных — в PR #1–#4.

## Результаты анализа

Подробный разбор SAST: [REPORT.md](REPORT.md).

Главный вывод: **первый прогон Semgrep с набором правил `auto` нашёл 1 реальную уязвимость и дал 4 ложных срабатывания.** Большая часть проблем ниже автоматическими сканерами не находится. Их выявляют ручное ревью кода и конфигурации, SCA и DAST.

| # | Уязвимость | CWE | Где | Чем выявлено |
|---|---|---|---|---|
| 1 | SQL-инъекция: конкатенация строк в запросе | CWE-89 | `CourseService.java:42` | Semgrep |
| 2 | SSRF: сервер запрашивает любой URL из параметра | CWE-918 | `ProxyController` → `/api/proxy?targetUrl=` | Ручное ревью |
| 3 | XXE: `SAXReader` без запрета внешних сущностей + dom4j 1.6.1 (CVE-2020-10683) | CWE-611 | `XmlController`, `pom.xml` | Ревью; SCA (уязвимая версия dom4j) |
| 4 | Загрузка и выполнение кода по URL без проверки целостности | CWE-494 | `PluginLoader` (`URLClassLoader`) | Ручное ревью |
| 5 | Пароли хранятся и сравниваются открытым текстом | CWE-256 | `AuthController`, `User` | Ручное ревью |
| 6 | Пароль пишется в лог при входе | CWE-532 | `AuthController` (`log.info(... password)`) | Ручное ревью |
| 7 | Нет CSRF-защиты: Spring Security не подключён | CWE-352 | формы Thymeleaf | Semgrep (правило Django) + ревью |
| 8 | CORS разрешает любой origin и любые методы | CWE-942 | `WebConfig`, `@CrossOrigin("*")` | Ручное ревью |
| 9 | Actuator: открыты все эндпоинты (`env`, `heapdump`…) | CWE-200 | `application.yaml` | Ревью конфигурации |
| 10 | H2-консоль включена и доступна извне | CWE-306 | `application.yaml` (`web-allow-others: true`) | Ревью конфигурации |
| 11 | Stack trace в ответах об ошибках | CWE-209 | `application.yaml` (`include-stacktrace: ALWAYS`) | Ревью конфигурации |
| 12 | REST API студентов без аутентификации: чтение, изменение и удаление любой записи по id | CWE-862 | `StudentController` (`/api/students/{id}`) | Ручное ревью |

### Про CSRF

Изначально срабатывание про CSRF было помечено как ложное, потому что правило написано для Django. При пересмотре ([REPORT.md, п. 3.2](REPORT.md)) выяснилось, что проблема реальна: в `pom.xml` нет `spring-boot-starter-security`, значит CSRF-токенов в формах нет. Сканер применил неподходящее правило, но нашёл реальную проблему.

## Как исправить

Исправления ещё не внесены: приложение остаётся учебным стендом.

| # | Исправление |
|---|---|
| 1 | Параметризованные запросы (`@Query` с `:param` / `PreparedStatement`) |
| 2 | Allowlist хостов и схем, запрет приватных диапазонов и `169.254.169.254`, или убрать эндпоинт |
| 3 | dom4j → 2.1.4+, `reader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)` |
| 4 | Убрать загрузку по URL или проверять подпись/хеш артефакта перед загрузкой |
| 5 | BCrypt/Argon2 через `PasswordEncoder` |
| 6 | Не логировать секреты; маскирование в логах |
| 7 | Подключить Spring Security (CSRF включён по умолчанию) |
| 8 | Явный список доверенных origin |
| 9 | `management.endpoints.web.exposure.include: health,info` |
| 10 | Выключить H2-консоль вне dev-профиля |
| 11 | `server.error.include-stacktrace: never` |
| 12 | Spring Security + проверка владельца записи / роли на каждом эндпоинте |

## Запуск

```bash
./mvnw spring-boot:run
# http://localhost:8080
```

## Что можно улучшить в пайплайне

- Сделать Semgrep блокирующим для находок уровня ERROR
- Порог CVSS для Dependency-Check (`-DfailBuildOnCVSS=7`)
- DAST (OWASP ZAP baseline) по поднятому в CI приложению
