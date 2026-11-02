# SQ-3 · Успешное повторение в ротации

> Источник: `docs/sources/tbank-english-learning-uml.drawio`, страница `sq-review` (v1.0 · 25.10.2026).
> Ответ «Я запомнил»: идемпотентность, блокировка прогресса, следующий этап и завершение после 8-го успеха.

## Диаграмма

```mermaid
sequenceDiagram
    actor U as Сотрудник
    participant W as Web App · SPA
    participant B as Backend · Learning API
    participant R as Review Engine
    participant DB as База данных · SQL

    U->>W: 01 · Нажать «Я запомнил» на карточке этапа N
    W->>B: 02 · POST /reviews {result: 'remembered', idempotency_key}
    B->>B: 03 · Проверить JWT, ownership, payload
    B->>DB: 04 · SELECT review by idempotency_key

    opt [ключ уже обработан]
        B-->>DB: 05 · existing result
        B-->>W: 06 · 200 {replayed: true}, повтор не создаётся
        W->>W: 07 · Идемпотентный ответ (выход)
    end

    B->>R: 08 · Обработать успешное повторение
    R->>DB: 09 · BEGIN, SELECT progress FOR UPDATE
    DB-->>R: 10 · progress {stage: N, status: 'in_rotation'}

    alt [stage устарел / уже обработан]
        R-->>B: 11 · 409 stale_card {actual_stage}
        B-->>W: 12 · 409 + актуальная карточка
        W->>U: 13 · Показать актуальную карточку
    else [stage актуален]
        R->>R: 14 · if N = 8 → memorized, иначе stage = N+1
        R->>R: 15 · next_at = now + interval[N+1]
        R->>DB: 16 · UPDATE progress, INSERT history, COMMIT
        R-->>B: 17 · OK {status, next_at, learned}
        R-->>B: 18 · result
        B-->>W: 19 · 200 {next_at, learned, stats_delta}
        W->>W: 20 · Обновить локальный прогресс
        W->>U: 21 · Следующая карточка / «Слово выучено»
    end
```

## Лестница повторений

| Этап | Интервал до показа |
|---|---|
| 1 | 5 минут |
| 2 | 1 час |
| 3 | 1 день |
| 4 | 1 неделя |
| 5 | 1 месяц |
| 6 | 3 месяца |
| 7 | 6 месяцев |
| 8 | 12 месяцев |

## Завершение

8-е успешное подтверждение → `status = memorized`, `next_review_at = NULL`. Слово исчезает из очереди и попадает в статистику.

## Ветка «Не запомнил»

`POST /reviews` с `result: 'not_remembered'`: слово повторяется через 5 мин и 1 ч, затем возвращается к следующему этапу. Отдельная ветка — см. WS-3, Decision D-02 (в диаграмме sq-3 не детализируется).

## Гарантии

- `Idempotency-Key` — повторная отправка не создаёт дублей (`200 {replayed: true}`).
- `SELECT ... FOR UPDATE` — блокировка строки прогресса на время транзакции.
- `409 stale_card` — ответ по устаревшему этапу не применяется.
- Аудит: INSERT history на каждое повторение.

**Лестница**: 8 успешных повторений. Интервалы 5 мин → 1 ч → 1 день → 1 нед → 1 мес → 3 мес → 6 мес → 12 мес (D-03).
