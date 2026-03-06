# AGENTS.md — llm-review-prompts

## What is this

A collection of structured LLM prompts for code review, organized by reviewer role and technology stack. Each prompt is a reusable template with context slots and severity classification.

## Stack

- Pure markdown — no build system, no dependencies
- Git for versioning
- Prompts target: Claude, GPT-4, Gemini (any capable LLM)

## Structure

```
prompts/
├── developer/     # Code quality review: dotnet.md, rust.md, python.md, go.md
├── architect/     # Architecture review: dotnet.md, rust.md, python.md, go.md
├── tester/        # QA review: manual.md, e2e.md, autotests.md
├── reviewer/      # Final review: general.md
└── security/      # Security review: general.md

docs/
├── guide.md             # How to build prompts (RU)
└── context-injection.md # How to inject project context (RU)

examples/
└── context-template.md  # Filled PROJECT_CONTEXT example
```

## Rules for Adding New Prompts

1. **Language**: Prompts are written in **English only**. Docs and guides in Russian.
2. **File naming**: `<technology>.md` in the appropriate role folder. Use lowercase, no spaces.
3. **Required sections** (in this order):
   - `## Role` — one sentence defining who the LLM is
   - `## Goal` — what the review must achieve
   - `## Critical Rules` — non-negotiable constraints with brief rationale
   - `## Context Slots` — description of each `[PLACEHOLDER]`
   - `## Instructions` — step-by-step review process
   - `## Output Format` — exact format for findings
   - `## Few-Shot Example` — one realistic example (input → output)
   - `## Prohibited` — what the reviewer must NOT do
4. **Severity levels** must use exactly: `BLOCKING` / `MINOR` / `SUGGESTION`
5. **Slots** must use `[ALL_CAPS_IN_BRACKETS]` format
6. **Few-shot examples** must be realistic — real-looking code, real-looking findings. No toy examples.
7. **Critical Rules** must explain *why*, not just *what*.

## Slot Conventions

| Slot | Used in | Description |
|------|---------|-------------|
| `[PROJECT_CONTEXT]` | all prompts | AGENTS.md + arch decisions + stack + rules |
| `[DIFF]` | all prompts | git diff or file contents being reviewed |
| `[FOCUS_AREAS]` | developer, architect | optional additional focus |
| `[ARCH_DECISIONS]` | architect | ADR records or architecture notes |
| `[PREVIOUS_REVIEWS]` | reviewer/general | outputs from other role reviews |
| `[SECURITY_BASELINE]` | security/general | known risks accepted by the project |

## Status

- [x] Initial prompt collection (developer, architect, tester, reviewer, security)
- [x] Docs: guide.md, context-injection.md
- [x] Examples: context-template.md
- [ ] CI lint for prompt structure
- [ ] Additional stacks: Java, TypeScript, Kotlin

## How to Run Locally

Just clone and open in any markdown viewer. No build steps.

```bash
git clone <repo-url>
cd llm-review-prompts
# Open prompts/ in your editor or cat any prompt
```

## Pitfalls

- Do NOT add opinionated style rules to prompts unless they are backed by a specific project rule. Generic style bikeshedding defeats the purpose.
- `[PROJECT_CONTEXT]` is the most impactful slot. Prompts without it produce generic, low-value reviews.
- Few-shot examples are not decoration — they calibrate the LLM's output format and tone. Keep them realistic.
- Avoid duplicating rules across prompts. If a rule applies to all, consider extracting it to `docs/guide.md`.
