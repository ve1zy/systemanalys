# T · English Learning Platform — документация

Приложение по обучению английскому языку для сотрудников Т (повышение квалификации, выход на международный рынок).

Трехкомпонентная система: **Web App** (обучение) + **Backend** (REST API, данные) + **Admin Panel** (модерация контента). Авторизация — рабочая почта + OTP-код; сессия 30 дней.

> Статус: документация (MVP). Репозиторий пока содержит только наработки: ТЗ и UML-модель (в `docs/sources/`). Кода нет.

## Навигация

| Раздел | Файл | Что внутри |
|---|---|---|
| **Индекс** (этот файл) | `README.md` | обзор, статус, порядок чтения |
| **Walking Skeleton** | [docs/walking-skeleton/walking-skeleton.md](docs/walking-skeleton/walking-skeleton.md) | 4 сквозных пути по шагам: HTTP, SQL, JSON, эдж-кейсы |
| Контракты WS | [docs/walking-skeleton/ws-contracts.md](docs/walking-skeleton/ws-contracts.md) | JSON-схемы и матрица эндпоинтов |
| **ТЗ** | [docs/requirements/01-tz.md](docs/requirements/01-tz.md) | полное техническое задание (зеркало docx) |
| UML: контекст | [docs/diagrams/uc-1-context.md](docs/diagrams/uc-1-context.md) | акторы, компоненты, интеграции, use cases |
| UML: обучение | [docs/diagrams/uc-2-learning.md](docs/diagrams/uc-2-learning.md) | карточки, ротация, правила очереди |
| UML: авторизация | [docs/diagrams/sq-1-auth.md](docs/diagrams/sq-1-auth.md) | email + OTP, токен 30 дней |
| UML: очередь | [docs/diagrams/sq-2-next-card.md](docs/diagrams/sq-2-next-card.md) | приоритет due → new → 204 |
| UML: повторение | [docs/diagrams/sq-3-review.md](docs/diagrams/sq-3-review.md) | идемпотентность, лестница этапов |
| UML: админка | [docs/diagrams/sq-4-admin.md](docs/diagrams/sq-4-admin.md) | XLS импорт/экспорт, optimistic locking |
| Глоссарий | [docs/architecture/glossary.md](docs/architecture/glossary.md) | термины ротации/очереди |
| Non-MVP | [docs/architecture/non-mvp.md](docs/architecture/non-mvp.md) | функции за пределами MVP |
| Форматные решения | [docs/ADR.md](docs/ADR.md) | почему Markdown+Mermaid, Decisions, cookie-токен |
| Исходники | [docs/sources/README.md](docs/sources/README.md) | docx + drawio (источник истины) |

## Порядок чтения

1. [ТЗ](docs/requirements/01-tz.md) — что строим (или исходник в `docs/sources/`).
2. [Walking Skeleton](docs/walking-skeleton/walking-skeleton.md) — как это работает под капотом на сквозных путях.
3. Диаграммы (UC → SQ) — детализация потоков.
4. [Контракты](docs/walking-skeleton/ws-contracts.md) — рабочий справочник при разработке.

## Происхождение документации

- ТЗ: `docs/sources/Проект_приложение_по_обучению_английскому_языку.docx`
- UML v1.0 · 25.10.2026: `docs/sources/tbank-english-learning-uml.drawio` (6 страниц)
- .md — зеркала; при расхождении верен исходник (см. [docs/sources/README.md](docs/sources/README.md)).
