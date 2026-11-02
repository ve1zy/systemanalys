# WS · Контракты и матрица эндпоинтов

> Общий справочник для `walking-skeleton.md`. При изменении WS обновлять контракты здесь.

## 1. JSON-схемы

### Card (карточка в ответе next-card)

```json
{
  "word_id": 42,
  "word": "sick",
  "transcription": "sɪk",
  "translation": "больной",
  "word_type": "Прилагательное",
  "orientation": "EN_RU",
  "stage": 2,
  "stage_total": 8
}
```

- `orientation`: `EN_RU` | `RU_EN`. Новое слово — всегда `EN_RU`.
- `stage`: 0 у нового слова; 1–8 у слова в ротации.
- `translation` клиенту передаётся, но НЕ показывается до нажатия «Показать слово».

### Ответ `GET /learning/next-card`

```json
// 200
{ "type": "due", "card": { "...": "см. Card" }, "next_review_at": "2026-01-14T13:40:00Z" }
{ "type": "new", "card": { "...": "см. Card" } }
// 204 — карточек больше нет
// 401 — нет/истёк токен · 409 — нет доступа к словарю
```

### Ответ `POST /reviews`

```json
// запрос
{ "result": "remembered", "idempotency_key": "uuid:card-shown-2026-01-14T13:40", "word_id": 42, "stage": 2 }

// 200 успех
{ "status": "in_rotation", "stage": 3, "next_review_at": "2026-01-14T14:40:00Z", "learned": false, "stats_delta": { "in_progress": 0, "memorized": 0 } }
// 200 после «не запомнил» (быстрый повтор запущен, D-02)
{ "status": "in_rotation", "stage": 2, "quick_repeat_step": 1, "next_review_at": "2026-01-14T13:45:00Z", "learned": false, "stats_delta": {} }
// 200 — слово выучено
{ "status": "memorized", "stage": 8, "next_review_at": null, "learned": true, "stats_delta": { "in_progress": -1, "memorized": 1 } }
// 200 идемпотентный повтор
{ "replayed": true, "stage": 3 }
// 409 stale_card
{ "error": "stale_card", "actual_card": { "word_id": 42, "stage": 5 } }
```

- `result`: `remembered` | `not_remembered`. Остальные поля ответа одинаковой формы; семантика веток — в WS-3.

### Ответ `POST /auth/request-code` и `/auth/resend-code`

```json
// 202
{ "resend_after": 60 }
// 400 — некорпоративная почта; 429 — rate limit { "retry_after": 60 }
```

### Ответ `POST /auth/verify-code`

```json
// 200
{ "access_token": "<jwt>", "expires_at": "2026-12-02T18:00:00Z" }
// 401 — код неверный { "attempts_left": 4 }
// 410 — код истёк / лимит исчерпан { "attempts_left": 0 }
```

### JWT claims

```json
{ "sub": "<user_id>", "role": "user", "exp": 1767290400, "iat": 1764709200 }
```

## 2. Матрица эндпоинтов

| Метод и путь | Зона | Статусы |
|---|---|---|
| `GET /auth/me` | публичный* | 200 · 401 |
| `POST /auth/request-code` | публичный | 202 · 400 · 429 |
| `POST /auth/resend-code` | публичный | 202 · 400 · 429 |
| `POST /auth/verify-code` | публичный | 200 · 401 · 410 |
| `GET /vocabularies` | auth | 200 · 401 |
| `GET /vocabularies/{id}` (скачивание словаря) | auth | 200 · 401 · 404 |
| `POST /vocabularies/{id}/sync` (прогресс) | auth | 200 · 409 · 422 |
| `GET /learning/next-card?vocabulary_id=` | auth | 200 · 204 · 401 · 409 |
| `POST /reviews` | auth | 200 · 409 · 401 · 422 |
| `GET /learning/stats?vocabulary_id=` | auth | 200 · 401 |
| `GET /admin/vocabularies/{id}` | moderator+ | 200 · 401 · 403 |
| `POST /admin/import` (multipart) | moderator+ | 200 preview · 422 · 413 · 401 · 403 |
| `POST /import/commit` | moderator+ | 200 · 401 · 403 |
| `PATCH /words/{id}` (If-Match: version) | moderator+ | 200 · 409 · 422 · 401 · 403 |
| `GET /admin/vocabularies/{id}/export.xlsx` | moderator+ | 200 stream · 401 · 403 |

\* «публичный» = с валидным токеном у `/auth/me`; все остальные auth-эндпоинты требуют `Authorization: Bearer <token>` (или HttpOnly cookie, ADR-005).

## 3. 🔤 Формат токена и ошибок — единые правила

- Любой ответ ошибки: `{ "error": "<код>", "message": "<человекочитаемо>", ...доп.поля }`.
- Auth-ошибки не раскрывают валидность email: одинаковые ответы на request-code для существующего и несуществующего корпоративного email (для не-корпоративных доменов — явный `400 forbidden_domain`, т.к. правило доменов публичное).
