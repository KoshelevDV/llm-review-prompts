# Гайд по построению промптов для code review

Этот гайд объясняет паттерны, из которых собираются промпты в этом репозитории. Если хочешь добавить новый промпт или понять, почему существующие устроены именно так — читай сюда.

---

## Проблема «плохого промпта»

Типичный плохой промпт: «Посмотри мой код и скажи что не так».

Что получаем:
- Общие советы в духе «добавь комментарии»
- Замечания к именованию переменных вместо архитектурных проблем
- Нет структуры → нет воспроизводимого результата
- LLM не знает контекст проекта → ревью вслепую

Цель этого гайда — научить строить промпты, которые дают **воспроизводимый, структурированный, полезный** результат.

---

## Паттерн 1: Role + Goal

Первое, что должен знать LLM — **кто он** и **что ему нужно сделать**.

### Плохо:
```
Review this code.
```

### Хорошо:
```
## Role
You are a senior .NET developer with 10+ years of experience in enterprise C# applications, 
ASP.NET Core, and Clean Architecture. You are doing a code review for a production system.

## Goal
Review the provided diff and identify issues that could cause bugs, security vulnerabilities, 
performance problems, or violations of the project's architecture rules.
Your goal is NOT to rewrite the code — only report actionable findings.
```

**Почему это важно:** LLM ведёт себя по-разному в зависимости от роли. «Старший разработчик» даёт другие ответы, чем «ассистент». Роль задаёт экспертизу и тон.

**Goal** ограничивает скоуп — без него LLM начнёт рефакторить весь код или давать советы, которые тебя не просили.

---

## Паттерн 2: Critical Rules с объяснением почему

Critical Rules — это жёсткие ограничения, которые LLM не может игнорировать. Ключевое: **каждое правило должно объяснять своё существование**.

### Плохо:
```
## Rules
- No raw SQL
- Use repository pattern
- No magic strings
```

### Хорошо:
```
## Critical Rules

1. **No raw SQL strings in application code** — all DB access must go through repositories.
   *Why: raw SQL bypasses the ORM's SQL injection protection and couples business logic to DB schema.*

2. **Never call `.Result` or `.Wait()` on async tasks** — use `await` throughout.
   *Why: blocking on async causes deadlocks in ASP.NET Core's synchronization context.*

3. **All public APIs must return `IActionResult` or `ActionResult<T>`, never raw objects.**
   *Why: raw object return bypasses content negotiation and makes error handling inconsistent.*
```

**Почему это важно:** LLM без объяснений не понимает, насколько критично правило, и может пропустить нарушение. С объяснением — понимает контекст и применяет правило корректно даже в нестандартных ситуациях.

---

## Паттерн 3: Слоты для контекста

Промпт должен иметь явные места для подстановки контекста. Используй формат `[ALL_CAPS_IN_BRACKETS]`.

### Основные слоты:

```markdown
## Context Slots

- [PROJECT_CONTEXT] — Project's AGENTS.md, architecture decisions, tech stack, coding conventions.
  Fill this with your project's actual context before using the prompt.

- [DIFF] — The git diff or file contents to review. 
  Paste the output of `git diff main...feature-branch` or specific file contents.

- [FOCUS_AREAS] — (Optional) Specific areas to focus on.
  Example: "Focus on error handling and the new caching layer."
```

### Как слоты выглядят в теле промпта:

```markdown
---
## Project Context

[PROJECT_CONTEXT]

---
## Diff to Review

```diff
[DIFF]
```

---
## Additional Focus

[FOCUS_AREAS]

---
```

**Почему это важно:** Явные слоты делают промпт **шаблоном** — его можно переиспользовать, автоматизировать (вставлять контекст программно), версионировать. Без слотов каждый раз надо помнить, что именно вставить и куда.

---

## Паттерн 4: Few-Shot примеры

Few-shot пример — это демонстрация желаемого формата вывода. LLM видит: «ах, вот так надо отвечать».

### Структура few-shot примера:

```markdown
## Few-Shot Example

**Input diff:**
```diff
+ public User GetUser(int id)
+ {
+     var sql = $"SELECT * FROM Users WHERE Id = {id}";
+     return _db.ExecuteQuery<User>(sql).FirstOrDefault();
+ }
```

**Expected output:**
```
[BLOCKING] SQL Injection vulnerability
File: UserRepository.cs, line 3
The query uses string interpolation with user-controlled input `id`.
Fix: Use parameterized queries — `_db.ExecuteQuery<User>("SELECT * FROM Users WHERE Id = @id", new { id })`.
```
```

**Почему это важно:** Без few-shot LLM может придумать свой формат вывода. С few-shot — воспроизводимый, парсируемый результат. Особенно важно если результат будет обрабатываться программно (например, автопостинг в MR).

### Реалистичность примеров

Toy examples («функция складывает два числа») бесполезны. Примеры должны быть близки к реальному коду:
- Реальные имена методов и классов
- Реальный anti-pattern (не придуманный)
- Реальный fix (конкретный, не «улучши это»)

---

## Паттерн 5: Явные запреты

LLM без ограничений часто:
- Комментирует стиль («переименуй `x` в `index`»)
- Предлагает рефакторинг, который не просили
- Выдаёт общие советы без привязки к коду
- Хвалит хороший код (тратит токены впустую)

Запреты решают это:

```markdown
## Prohibited

- DO NOT comment on naming conventions unless the name is actively misleading or causes a bug.
- DO NOT suggest refactoring if the existing structure works and doesn't violate project rules.
- DO NOT praise good code — only report issues.
- DO NOT report issues you're uncertain about — if in doubt, omit.
- DO NOT suggest adding comments or documentation unless it's a project requirement.
- DO NOT repeat the same finding multiple times for similar code — report once, note if it's a pattern.
```

**Почему это важно:** Запреты уменьшают шум. Code review с 20 замечаниями, из которых 15 — про стиль, хуже, чем review с 5 важными находками.

---

## Паттерн 6: Инструкции (пошаговый процесс)

Если промпт сложный (несколько аспектов для проверки), дай LLM явный порядок действий:

```markdown
## Instructions

1. Read [PROJECT_CONTEXT] carefully. Identify key constraints and architecture rules.
2. Parse the [DIFF] section by section.
3. For each changed file, check:
   a. Does it violate any Critical Rule?
   b. Are there security issues (injection, auth bypass, secret exposure)?
   c. Are there correctness issues (null refs, off-by-one, race conditions)?
   d. Are there performance issues (N+1 queries, unbounded loops)?
4. Check if [FOCUS_AREAS] are addressed.
5. Output findings sorted by severity: BLOCKING first, then MINOR, then SUGGESTION.
6. If no issues found, output: "No issues found."
```

**Почему это важно:** Chain-of-thought в инструкциях заставляет LLM «думать» последовательно, а не выдавать первое, что пришло в голову. Меньше пропущенных проблем.

---

## Паттерн 7: Формат вывода

Задай формат явно. Без этого каждый ответ будет отличаться.

```markdown
## Output Format

For each issue, output:
```
[SEVERITY] Short title
File: <filename>, line <N> (if applicable)
<One to three sentences describing the issue and its impact.>
Fix: <Concrete fix suggestion.>
```

Severity levels:
- BLOCKING — must be fixed before merge (bug, security issue, architecture violation)
- MINOR — should be fixed but won't block merge (code smell, suboptimal approach)
- SUGGESTION — optional improvement (performance, readability — only if significant)

Separate findings with a blank line.
At the end, output a one-line summary: "X blocking, Y minor, Z suggestions."
```

---

## Структура секций промпта (рекомендуемый порядок)

```
1. ## Role          — кто ты
2. ## Goal          — что делаешь и чего не делаешь
3. ## Critical Rules — жёсткие ограничения с объяснением почему
4. ## Context Slots — описание слотов
5. ## Instructions  — пошаговый процесс
6. ## Output Format — формат ответа с severity levels
7. ## Few-Shot Example — пример входа и выхода
8. ## Prohibited    — явные запреты

--- (разделитель)

## Project Context
[PROJECT_CONTEXT]

## Diff to Review
[DIFF]

## Focus Areas
[FOCUS_AREAS]
```

Порядок важен: сначала LLM должен знать роль и правила, потом — контекст, потом — задачу.

---

## Частые ошибки

| Ошибка | Почему плохо | Как исправить |
|--------|-------------|---------------|
| Нет роли | LLM ведёт себя как обобщённый ассистент | Добавь `## Role` |
| Нет `[PROJECT_CONTEXT]` | Ревью вслепую, общие советы | Всегда вставляй AGENTS.md проекта |
| Нет few-shot | Непредсказуемый формат вывода | Добавь хотя бы один реалистичный пример |
| Правила без объяснения «почему» | LLM не применяет в нестандартных случаях | Объясни мотивацию каждого правила |
| Нет запретов | Шум, стилевые придирки | Добавь `## Prohibited` |
| Слишком длинный промпт без слотов | Сложно переиспользовать | Выдели слоты, сделай шаблон |

---

## Итоговый чеклист нового промпта

- [ ] `## Role` — чёткая роль с экспертизой
- [ ] `## Goal` — что делать и что НЕ делать
- [ ] `## Critical Rules` — с объяснением «почему» для каждого
- [ ] Слоты `[PROJECT_CONTEXT]`, `[DIFF]`, + специфичные для роли
- [ ] `## Instructions` — пошаговый процесс
- [ ] `## Output Format` — severity levels, пример блока
- [ ] `## Few-Shot Example` — реалистичный код, реалистичная находка
- [ ] `## Prohibited` — минимум 5 явных запретов
