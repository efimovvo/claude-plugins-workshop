---
name: dmazurin-autobug-analyzer
description: Analyzes Sentry autobugs — fetches the issue and its stack trace, finds root cause in source code, then recommends and applies the best fix: weaken logging (transient errors), fix code in-place (small obvious bugs), or create a Yandex Tracker ticket with Sentry ignore for N days (complex issues). Always invoke when the user says "разбери автобаг", "проанализируй sentry issue", "что делать с автобагом", "посмотри автобаг", pastes a Sentry issue URL, or asks what to do with an error from Sentry.
compatibility:
  mcps:
    - sentry       # setup: /mcp-setup:sentry
    - yandex-tracker
    - sourcebot
---

# Autobug Analyzer

Analyzes a Sentry issue, traces the root cause in source code, and applies one of three targeted actions.

## Prerequisites

This skill requires the Sentry MCP server. If `mcp__sentry__*` tools are unavailable, tell the user:
> Для работы с автобагами нужен Sentry MCP. Запусти `/mcp-setup:sentry` и перезапусти Claude Code.

**Before any destructive action (code edit, ticket creation, Sentry ignore) — show the plan to the user first (see Step 5).**

## Step 1 — Gather inputs

If not already provided by the user, ask for:
- **Sentry issue** — URL (e.g. `https://autobugs.mindbox.ru/organizations/mindbox/issues/12345/`) or numeric ID

That's the only required input. Derive everything else from the issue itself.

## Step 2 — Fetch Sentry data

If a URL is given, use `mcp__sentry__get_sentry_resource` (pass the URL directly). If only an ID is given, use `mcp__sentry__list_issues`. Fetch in parallel:
- Issue details via `mcp__sentry__get_sentry_resource` or `mcp__sentry__list_issues`
- Latest event with stack trace: `mcp__sentry__list_issue_events`
- AI analysis if available: `mcp__sentry__analyze_issue_with_seer`

Extract and note these facts before moving on:
- Error type and message
- Culprit (service, file, function)
- First/last seen, event count, affected users
- Full stack trace with file paths and line numbers

**Checkpoint:** if the event has no stack trace and the culprit is unknown — skip Step 3 and go directly to Option C with a note that code location could not be determined.

## Step 3 — Find root cause in source code

Use the stack trace to locate the relevant code. Search with `mcp__sourcebot__search_code` using class name, method name, or fragments of the error message. Then read the file with `mcp__sourcebot__read_file` to see the full context around the error.

Focus on:
- The exact line that throws or logs the error
- How the error is caught and handled (or not)
- The surrounding logic — what state leads to this code path

If the culprit spans multiple files or services, follow the call chain enough to understand the real root cause, not just the symptom.

**Checkpoint:** if `mcp__sourcebot__search_code` returns no results after 2 attempts with different queries — proceed to Step 4 with a note "source code not found". This does not block classification: use issue data alone and prefer Option C.

## Step 4 — Classify and recommend

Based on the issue data and code, choose **one** action. Explain your reasoning — why this option fits better than the alternatives.

---

### Option A: Weaken logging

**Choose when:** The error is transient or expected infrastructure noise — network timeouts, connection resets, circuit breaker trips, retry-able HTTP errors (502/503/504), third-party service blips, or errors that self-resolve without user impact.

**Signals:**
- Message contains: `timeout`, `connection refused`, `unavailable`, `retry`, `circuit`, `deadline exceeded`
- Stack trace points to HTTP client, gRPC stub, or retry library — not business logic
- High frequency but zero or minimal affected users
- Errors cluster in time (infrastructure event) rather than being continuous

**What to do:**
1. Find the exact `LogError` / `logger.Error` / `_logger.LogError` call in source
2. Change severity to `LogWarning` or add an exception filter that suppresses this specific error type
3. Apply the change directly to the file
4. Show the diff and explain the reasoning

---

### Option B: Fix code in-place

**Choose when:** There's a clear, small bug fixable in 1–10 lines without architectural decisions — null reference missing a guard, unhandled enum/switch case, missing validation, incorrect condition, off-by-one.

**Signals:**
- A specific null dereference or missing case is visible in the stack trace
- The fix is obvious from reading the code — no ambiguity about correct behavior
- The change is self-contained; no ripple effects

**What to do:**
1. Apply the fix directly to the source file
2. Show the diff
3. Briefly explain what was broken and why the fix is correct

---

### Option C: Create tracker ticket + ignore

**Choose when:** The issue is real but requires investigation, design decisions, multiple components, or significant refactoring. When in doubt between B and C — pick C. A well-described ticket beats a rushed in-place fix.

**What to do:**

1. **Ask the user** which Yandex Tracker queue to use. Suggest one based on the service name from the stack trace if you can infer it.

2. **Create the ticket** via `mcp__yandex-tracker__issue_create`:
   - Title: `[Autobug] <concise description>`
   - Description: use the template below

3. **Link back to Sentry**: add a comment to the Sentry issue via `mcp__sentry__update_issue` containing the tracker ticket URL, so anyone looking at Sentry sees the issue is tracked.

4. **Ignore the Sentry issue for 30 days** via `mcp__sentry__update_issue` with:
   ```json
   {"status": "ignored", "statusDetails": {"ignoreDuration": 30}}
   ```
   This stops the autobug from re-firing while the ticket is being worked.

#### Tracker ticket description template

```
## Проблема
<One or two sentences: what goes wrong, what the error message says, where it happens.>

## Причина
<Root cause as understood. Reference the specific file and line. Be honest if it's partially unclear.>

## Мотивация к исправлению
<Why this matters: frequency, user impact, technical debt. Use concrete numbers from Sentry.>

## Контекст
- Sentry issue: <full URL>
- Сервис / компонент: <name>
- Файл: <path:line>
- Впервые замечено: <first seen>
- Последний раз: <last seen>
- Всего событий: <count>

## Предлагаемое решение
<If there's a clear direction, describe briefly. Otherwise: "Требует исследования".>
```

---

## Step 5 — Confirm and execute

Before applying any change, show the user:
- **Диагноз:** one-sentence root cause
- **Рекомендация:** which option and why
- **Что будет сделано:** specific action

For Option A/B — show the plan and proceed (user can revert).
For Option C — confirm the tracker queue before creating the ticket.

## Step 6 — Summary

After completing:

```
✓ Проблема: <one sentence>
✓ Действие: <Option A / B / C — what was done>
[Option C only]
  Тикет: <tracker URL>
  Sentry ignore: 30 дней
```

---

## Troubleshooting

**Sourcebot не нашёл код** — proceed with classification using only Sentry data. In the diagnosis, write "source code not found in Sourcebot". Use Option C: a ticket with partial context is better than a blind code fix.

**Option C частично выполнился** (тикет создан, но `update_issue` упал) — tell the user: "Тикет создан: <URL>. Sentry ignore не выставлен — выставь вручную или повтори команду." Do not retry automatically.

**Sentry issue not found** — tell the user the ID/URL didn't resolve and ask to double-check. Do not proceed.

---

## Example

**Input:** `https://autobugs.mindbox.ru/organizations/mindbox/issues/4068049/`

**Sentry data:** `No suitable for migration format was found (formatId: 713642)` — 15 events, staging only, 0 users impacted. Logger: `FormatTemplatesAttributeToQuokkaBlockMigrator`.

**Code found:** `mailings/src/Worker/.../FormatTemplatesAttributeToQuokkaBlockMigrator.cs` — `logger.LogError(...)` fires when `TryGetSuitableForMigrationFormatAsync` returns null. Two distinct cases collapsed into one: format not in DB, or format already migrated (no attribute blocks).

**Diagnosis:** `LogError` used for an expected "nothing to do" case in a Kafka-driven migration, causing false noise.

**Chosen option:** B — small, obvious fix: split the two null-return cases, log `Warning` for missing format and `Information` for already-migrated format.

**Output:**
```
✓ Проблема: LogError срабатывает на ожидаемый исход миграции (формат уже смигрирован)
✓ Действие: Option B — изменены уровни логирования в FormatTemplatesAttributeToQuokkaBlockMigrator.cs
```
