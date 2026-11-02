# SQ-2 · Получение следующей карточки и приоритет очереди

> Источник: `docs/sources/tbank-english-learning-uml.drawio`, страница `sq-next-card` (v1.0 · 25.10.2026).
> Сначала due-карточки в ротации — от самой ранней; новые слова — только после опустошения due-очереди.

## Диаграмма

```mermaid
sequenceDiagram
    actor U as Сотрудник
    participant W as Web App · SPA
    participant B as Backend · Learning API
    participant Q as Queue Engine
    participant DB as База данных · SQL

    Note over U,W: 01 · Открыть обучение / завершить карточку
    W->>B: 02 · GET /learning/next-card?vocabulary_id
    B->>B: 03 · Проверить JWT и доступ к словарю
    B->>Q: 04 · Запросить следующую карточку
    Q->>DB: 05 · SELECT due: next_review_at ≤ now ORDER BY next_review_at ASC LIMIT 1
    Q-->>B: 06 · due-карточка / пусто

    alt [есть due-карточка]
        Q->>Q: 07 · Взять самую раннюю, orientation = random
        Q->>DB: 08 · UPDATE last_shown_at (опционально)
        Q-->>B: 09 · Карточка: stage N, orientation, due_at
        B-->>W: 10 · 200 {type: 'due', card, stage}
    else [due-карточек нет]
        Q->>DB: 11 · SELECT new word WHERE progress IS NULL LIMIT 1
        Q-->>B: 12 · новое слово / пусто
        alt [есть новое слово]
            Q->>Q: 13 · Новая карточка: EN→RU, stage=0
            B-->>W: 14 · 200 {type: 'new', card}
        else [новых слов нет]
            Q-->>B: 15 · null
            B-->>W: 16 · 204 {state: 'completed'}
            W->>U: 17 · Empty state: все слова пройдены (выход)
        end
    end

    opt [карточка получена]
        W->>U: 18 · Показать карточку: перевод скрыт, бейдж N/8
        W->>W: 19 · Сохранить idempotency_key
        W->>U: 20 · Показать слово / ответить → SQ-3
    end
```

## Приоритет очереди

1. `in_rotation` и `next_review_at ≤ now` (due-карточки).
2. `ORDER BY next_review_at ASC` (самые ранние первыми).
3. Новые слова — только если due пуста.
4. Иначе — empty state.
   Пример: `sick` due 13:40 → `feel` due 14:50 (показываем sick первым).

## Ориентация

- Новое слово — всегда EN → RU, `stage = 0`.
- Ротация — случайно EN → RU / RU → EN.
- Перевод передаётся в ответе, но скрыт до нажатия «Показать слово».

## Контракт 200

```json
{ "type": "due" | "new", "card": { "word_id": 1, "word": "sick", "transcription": "sɪk", "translation": "больной", "word_type": "Прилагательное", "orientation": "EN_RU" }, "stage": 2, "due_at": "2026-01-14T13:40:00Z" }
```

- `204` — карточек больше нет;
- `401` — истёк токен;
- `409` — доступ к словарю не подтверждён / словарь не найден.

## Почему по одной карточке

Сервер пересчитывает очередь на каждый запрос. Клиент не хранит порядок; после ответа по карточке Web снова вызывает `next-card`.

**Инварианты**: одна карточка за запрос · due раньше новых · новый тип всегда EN → RU · перевод не раскрывается автоматически · после ответа очередь пересчитывается.
