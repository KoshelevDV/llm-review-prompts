# Как передавать контекст проекта в промпт

## Проблема «ревью вслепую»

Когда ты просишь LLM сделать code review без контекста, происходит следующее:

- LLM не знает, что в вашем проекте запрещён raw SQL — и не сообщает о нарушении
- LLM не знает, что `IUserRepository` — это не просто интерфейс, а часть Domain layer в DDD
- LLM не знает, что вы используете specific версию библиотеки с известными ограничениями
- LLM не знает, что в проекте принято оборачивать все ошибки в `Result<T>`, а не бросать исключения

В результате — generic ревью: «добавь null-check», «можно вынести в метод». Полезность стремится к нулю.

**Решение:** Передавать PROJECT_CONTEXT — структурированное описание проекта — в каждый промпт.

---

## Что входит в PROJECT_CONTEXT

### 1. AGENTS.md / CLAUDE.md проекта

Это главный источник контекста. Если в репозитории уже есть `AGENTS.md` — это ваш PROJECT_CONTEXT. Он содержит:
- Стек и версии
- Структуру проекта
- Правила разработки
- Архитектурные решения
- Ловушки и нюансы

### 2. Архитектурные решения (Architecture Decision Records)

Если в проекте есть ADR-ы (Architecture Decision Records), краткое описание ключевых:
- «Мы используем CQRS с MediatR, команды и запросы строго разделены»
- «Bounded Context: Orders и Inventory — отдельные сервисы, общаются через события»
- «Все внешние API-вызовы идут через Circuit Breaker (Polly)»

### 3. Стек и версии

```
.NET 8, ASP.NET Core 8, EF Core 8 (PostgreSQL provider)
MediatR 12, FluentValidation, Polly 8
Frontend: Blazor Server
Auth: Keycloak (OIDC)
```

### 4. Правила проекта

Явные правила, которые не вычислить из кода:
- «Нет raw SQL — только EF Core LINQ или хранимые процедуры через `ExecuteSqlRaw` с параметрами»
- «Контроллеры не содержат бизнес-логику — только маппинг запрос → команда/запрос»
- «Все эндпоинты защищены `[Authorize]`, исключения — явно помечены `[AllowAnonymous]`»
- «Тесты для каждого Command и Query handler обязательны перед мержем»

---

## Способ 1: Ручной — вставить AGENTS.md в промпт

Самый простой способ. Берёшь `AGENTS.md` из репозитория, вставляешь в слот `[PROJECT_CONTEXT]`:

```
## Project Context

# AGENTS.md — MyProject

## What is this
Enterprise .NET 8 application for order management.

## Stack
- .NET 8, ASP.NET Core 8, EF Core 8 (PostgreSQL)
- MediatR 12, FluentValidation, Polly 8
- Blazor Server for admin UI
- Keycloak for authentication

## Structure
- src/Domain/ — entities, value objects, domain events
- src/Application/ — CQRS handlers, validators
- src/Infrastructure/ — EF Core, external services
- src/API/ — controllers, Blazor pages

## Development Rules
- No raw SQL. All DB access through EF Core or stored procs with parameters.
- Controllers are thin. No business logic. Only HTTP → Command/Query mapping.
- All handlers must be tested before merge.
- No .Result or .Wait() on async calls.
- Use Result<T> pattern, never throw for expected errors.

## Pitfalls
- EF Core tracking is ON by default — use .AsNoTracking() for read-only queries
- Keycloak token refresh happens automatically via middleware, don't handle it manually
```

Это уже достаточно, чтобы LLM понял контекст и не давал irrelevant советы.

---

## Способ 2: Автоматический — через GitLab API

Если у вас есть CI/CD пайплайн или бот для ревью (например, `gitlab-reviewer`), контекст можно читать автоматически.

### Как это работает

1. При открытии Merge Request бот получает список изменённых файлов через GitLab API
2. Читает `AGENTS.md` из корня репозитория (тот же API)
3. Формирует `PROJECT_CONTEXT` из `AGENTS.md` + метаданных MR
4. Собирает diff изменённых файлов
5. Подставляет в промпт и отправляет в LLM
6. Результат постит комментарием к MR

### Пример: чтение AGENTS.md через GitLab API

```python
import httpx

GITLAB_URL = "https://gitlab.company.com"
TOKEN = "glpat-..."

def get_project_context(project_id: int, ref: str = "main") -> str:
    """Fetch AGENTS.md from repository root."""
    url = f"{GITLAB_URL}/api/v4/projects/{project_id}/repository/files/AGENTS.md/raw"
    response = httpx.get(url, params={"ref": ref}, headers={"PRIVATE-TOKEN": TOKEN})
    
    if response.status_code == 404:
        return "No AGENTS.md found. Review without project-specific context."
    
    response.raise_for_status()
    return response.text

def get_mr_diff(project_id: int, mr_iid: int) -> str:
    """Fetch diff for a merge request."""
    url = f"{GITLAB_URL}/api/v4/projects/{project_id}/merge_requests/{mr_iid}/diffs"
    response = httpx.get(url, headers={"PRIVATE-TOKEN": TOKEN})
    response.raise_for_status()
    
    diffs = response.json()
    return "\n".join(
        f"--- {d['old_path']}\n+++ {d['new_path']}\n{d['diff']}"
        for d in diffs
        if not d.get("too_large", False)
    )

def build_prompt(template: str, project_id: int, mr_iid: int) -> str:
    context = get_project_context(project_id)
    diff = get_mr_diff(project_id, mr_iid)
    
    return (
        template
        .replace("[PROJECT_CONTEXT]", context)
        .replace("[DIFF]", diff)
        .replace("[FOCUS_AREAS]", "")  # or extract from MR description
    )
```

---

## Шаблон контекстного блока

Если `AGENTS.md` в репозитории нет — используй этот шаблон для формирования `PROJECT_CONTEXT` вручную:

```markdown
## PROJECT_CONTEXT

### Project Overview
<One paragraph: what the system does, who uses it, scale.>

### Tech Stack
<Language/framework versions, key libraries, infrastructure.>

### Architecture
<2-3 sentences: architectural style (Clean Arch, hexagonal, monolith, microservices), key patterns.>
<Include: how layers interact, what's forbidden across boundaries.>

### Key Rules
<Bullet list of non-obvious rules that LLM must know:>
- <Rule 1 with brief rationale>
- <Rule 2 with brief rationale>
- ...

### Known Constraints / Pitfalls
<Non-obvious things that would cause false positives/negatives in review:>
- <Constraint 1>
- <Constraint 2>

### Out of Scope
<What the reviewer should NOT check (handled elsewhere, accepted tech debt, etc.):>
- <Item 1>
```

Готовый заполненный пример — в `examples/context-template.md`.

---

## Рекомендации по размеру контекста

### Проблема

Большой контекст = больше токенов = дороже и медленнее. Но маленький контекст = ревью вслепую.

### Ориентиры

| Размер PROJECT_CONTEXT | Токены (примерно) | Когда использовать |
|------------------------|------------------|--------------------|
| Краткий (100-300 слов) | ~400-600 | Небольшой проект, простые правила |
| Средний (300-800 слов) | ~600-1500 | Стандарт для большинства проектов |
| Полный AGENTS.md (800-2000 слов) | ~1500-4000 | Сложные проекты с нетривиальной архитектурой |
| Выше 2000 слов | >4000 | Только если LLM имеет большое окно контекста и правила реально сложные |

### Что резать при необходимости

1. **Оставить:** Правила, нарушение которых ловится при ревью (no raw SQL, no business logic in controllers)
2. **Оставить:** Архитектурные ограничения (границы bounded contexts, запрещённые зависимости)
3. **Можно убрать:** Инструкции по деплою (LLM не проверяет деплой при ревью кода)
4. **Можно убрать:** История решений (ADR) — только итог, не обоснование
5. **Можно убрать:** Примеры кода из AGENTS.md — они занимают много токенов, но LLM видит реальный код в `[DIFF]`

### Приоритет при сокращении

```
Rules > Architecture > Stack versions > Structure > History
```

Если приходится выбирать — оставляй правила, режь структуру и историю.
