# LLM Review Prompts

A curated collection of structured prompts for AI-assisted code review, organized by reviewer role and technology stack.

---

## EN — English

### What is this?

This repository contains battle-tested prompts for LLM-powered code review. Each prompt is designed for a specific role (developer, architect, security engineer, QA, etc.) and technology stack, with clear context slots and realistic few-shot examples.

Unlike generic "review my code" prompts, these are structured to:
- Enforce project-specific conventions via `PROJECT_CONTEXT`
- Classify findings by severity (`BLOCKING` / `MINOR` / `SUGGESTION`)
- Avoid false positives (no nitpicking consistent style, no commentary without reason)

### Repository Structure

```
prompts/
├── developer/       # Language-specific developer review (dotnet, rust, python, go)
├── architect/       # Architectural review by stack
├── tester/          # QA roles: manual, e2e, autotests
├── reviewer/        # Final lead engineer review
└── security/        # AppSec review (OWASP, CVE, secrets)

docs/
├── guide.md         # How to build effective review prompts (RU)
└── context-injection.md  # How to inject project context into prompts (RU)

examples/
└── context-template.md   # Filled PROJECT_CONTEXT example (.NET Blazor project)
```

### How to Use

1. **Pick the prompt** matching the role you want to simulate (e.g., `prompts/developer/dotnet.md`)
2. **Fill in the slots**:
   - `[PROJECT_CONTEXT]` — your project's `AGENTS.md`, architecture decisions, stack, conventions
   - `[DIFF]` — the git diff or changed files you want reviewed
   - `[FOCUS_AREAS]` — optional: specific concerns (e.g., "focus on error handling")
3. **Paste into your LLM** (Claude, GPT-4, Gemini, etc.)
4. **Interpret results** using the severity classification in each prompt

### Adding Project Context

The most important slot is `[PROJECT_CONTEXT]`. Without it, the LLM reviews code blindly — it doesn't know your architecture, conventions, or rules.

**What to include:**
- Your `AGENTS.md` or `CLAUDE.md` from the project root
- Key architectural decisions (e.g., "we use CQRS, no direct DB calls from controllers")
- Tech stack and versions
- Project-specific rules (e.g., "no raw SQL", "all public APIs must be documented")

See `docs/context-injection.md` for detailed guidance and `examples/context-template.md` for a real-world example.

### Adding New Prompts

See `AGENTS.md` for contribution rules.

---

## RU — Русский

### Что это?

Репозиторий содержит структурированные промпты для code review с помощью LLM. Каждый промпт заточен под конкретную роль (разработчик, архитектор, безопасник, QA и т.д.) и стек, с чёткими слотами для контекста и реалистичными few-shot примерами.

В отличие от шаблонного «посмотри мой код», эти промпты:
- Учитывают конвенции проекта через `PROJECT_CONTEXT`
- Классифицируют находки по серьёзности (`BLOCKING` / `MINOR` / `SUGGESTION`)
- Избегают ложных срабатываний (без придирок к консистентному стилю)

### Структура репозитория

```
prompts/
├── developer/       # Ревью разработчика по языку (dotnet, rust, python, go)
├── architect/       # Архитектурное ревью по стеку
├── tester/          # QA роли: ручное, e2e, автотесты
├── reviewer/        # Финальное ревью лид-инженера
└── security/        # AppSec ревью (OWASP, CVE, секреты)

docs/
├── guide.md         # Как строить эффективные промпты
└── context-injection.md  # Как передавать контекст проекта в промпт

examples/
└── context-template.md   # Заполненный пример PROJECT_CONTEXT (.NET Blazor)
```

### Как использовать

1. **Выбери промпт** под нужную роль (например, `prompts/developer/dotnet.md`)
2. **Заполни слоты**:
   - `[PROJECT_CONTEXT]` — `AGENTS.md` проекта, архитектурные решения, стек, правила
   - `[DIFF]` — git diff или изменённые файлы для ревью
   - `[FOCUS_AREAS]` — опционально: конкретные аспекты (например, «сфокусируйся на обработке ошибок»)
3. **Вставь в LLM** (Claude, GPT-4, Gemini и т.д.)
4. **Интерпретируй результаты** по классификации серьёзности в каждом промпте

### Добавление контекста проекта

Самый важный слот — `[PROJECT_CONTEXT]`. Без него LLM делает ревью вслепую — не знает архитектуру, конвенции и правила проекта.

**Что включать:**
- `AGENTS.md` или `CLAUDE.md` из корня проекта
- Ключевые архитектурные решения (например, «используем CQRS, прямых вызовов БД из контроллеров нет»)
- Стек и версии
- Правила проекта (например, «нет raw SQL», «все публичные API должны быть задокументированы»)

Подробнее — в `docs/context-injection.md`, пример — в `examples/context-template.md`.

### Добавление новых промптов

Правила — в `AGENTS.md`.
