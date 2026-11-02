# SQ-4 · Админка: работа со словарями

> Источник: `docs/sources/tbank-english-learning-uml.drawio`, страница `sq-admin` (v1.0 · 25.10.2026).
> Импорт XLS с preview и подтверждением, ручное редактирование с optimistic locking, экспорт XLSX.
> Предусловие: модератор/администратор уже авторизован; RBAC проверяется на каждом запросе.

## Диаграмма

```mermaid
sequenceDiagram
    actor M as Модератор
    participant P as Admin Panel · SPA
    participant B as Backend · Admin API (RBAC)
    participant X as XLS Worker
    participant DB as База данных · SQL

    M->>P: 01 · Открыть словарь
    P->>B: 02 · GET /admin/vocabularies/{id}
    B->>DB: 03 · SELECT dictionary, words, version
    B-->>P: 04 · 200 {words, version}
    P->>M: 05 · Редактор словаря

    opt 1 · Импорт XLS
        M->>P: 06 · Загрузить .xlsx, режим merge/replace
        P->>B: 07 · POST /admin/import (multipart)
        B->>X: 08 · Parse + validate: columns, types, duplicates
        X-->>B: 09 · rows | errors[]
        B->>DB: 10 · Preview + diff, import_token, audit
        alt [есть ошибки]
            B-->>P: 11 · 422 {row, column, reason}
            P->>M: 12 · Отчёт, данные не изменены
        else [файл валиден]
            B-->>P: 13 · 200 preview {created, updated}, показать в UI
            M->>P: 14 · Подтвердить импорт
            P->>B: 15 · POST /import/commit {import_token}
            B->>DB: 16 · BEGIN, upsert, version++, audit, COMMIT
            B-->>P: 17 · 200 {imported_count, version}
            P->>M: 18 · Таблица обновлена
        end
    end

    opt 2 · Редактирование и сохранение
        M->>P: 19 · Изменить слово, нажать «Сохранить»
        P->>B: 20 · PATCH /words/{id}, If-Match: version
        B->>DB: 21 · Валидация, проверка version
        alt [конфликт версий]
            B-->>P: 22 · 409 + current data
            P->>M: 23 · Показать конфликт
        else [версия актуальна]
            B->>DB: 24 · UPDATE word, version++, audit
            B-->>P: 25 · 200 {word}
            P->>M: 26 · Сохранено
        end
    end

    opt 3 · Экспорт XLS
        M->>P: 27 · Экспорт XLS
        P->>B: 28 · GET /admin/vocabularies/{id}/export.xlsx
        B->>X: 29 · Сформировать XLSX
        B-->>P: 30 · 200 {file stream}
        P->>M: 31 · Скачать файл
    end
```

## Импорт

- Валидация: обязательные колонки `word`, `translation`, `transcription`, `type`; уникальность `word` в словаре.
- Поток: Preview → подтверждение → транзакция. Ошибка → 422 и полный откат, данные не меняются.
- Режимы: **merge** (дефолт) / **replace** (явный флаг подтверждения).

## Редактирование

- Optimistic locking по `version` (If-Match). `409` → показать конфликт, не перезаписывать.
- Audit log: кто / что / когда.

## Экспорт

- XLSX с колонками `word`, `translation`, `transcription`, `type`. Имя файла: `vocabulary-{id}.xlsx`. Файл собирает backend.

## Ошибки

| Код | Причина |
|---|---|
| 401 / 403 | нет сессии / нет прав |
| 413 | файл слишком большой |
| 422 | ошибки строк (row, column, reason) |
| 409 | конфликт версии |
| 500 | транзакция откатывается |

## Соседние админ-сценарии (не в этой диаграмме)

Статистика пользователей (+выгрузка XLS) и управление ролями — отдельные админ-сценарии с тем же RBAC-принципом.
