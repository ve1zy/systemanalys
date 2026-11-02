# Walking Skeleton — приложение по обучению английскому языку

> Версия: 0.1 (базовые пути) · 02.11.2026
> Источники: `docs/requirements/01-tz.md` (ТЗ), `docs/diagrams/` (UML v1.0 · 25.10.2026).
> Матрица эндпоинтов и JSON-контракты: `ws-contracts.md`.

## Что это

Walking Skeleton — сквозные, проверяемые вручную пути «от человека до базы данных и обратно»:
пользователь выполняет действие → по шагам описано, что происходит **под капотом**
(HTTP-метод и путь, статус, JSON-тело запроса/ответа, SQL и строки, эдж-кейсы).

Ядро скелета — **3 пути** (WS-1, WS-2, WS-3) + **1 stretch-путь** (WS-4, админка; базовую
загрузку словаря решает сид-миграция из WS-2, поэтому админка не является блокером ядра).

## Самодостаточность

Документ самодостаточен для реализации:

- нет «см. ТЗ» и «см. диаграмму» как единственной справки — контракты и логика в тексте ниже;
- технологии намеренно не зафиксированы: любой язык, классический REST, БД;
- каждый путь завершается блоком **«Как проверить вручную»**.

**Checklist самодостаточности (проходить по каждому пути до «done»):**

- [ ] Все HTTP-вызовы пути имеют метод, путь и статусные коды.
- [ ] Все JSON-тела запросов/ответов расписаны (не «как в см. выше», а контракт доступен в `ws-contracts.md`).
- [ ] Все SQL-запросы пути написаны (CREATE/INSERT/SELECT/UPDATE + транзакции).
- [ ] Все эдж-кейсы пути (ноль строк, дубли, гонки, повторные попытки) описаны.
- [ ] Проверочный сценарий пути выполнен руками и дал ожидаемые строки в БД.

Стек-REST-жаргон: `auth`-зона = каждый запрос несёт `Authorization: Bearer <token>` (или HttpOnly cookie, ADR-005).

---

## Decisions (зафиксированные решения по неоднозначностям)

| # | Вопрос | Решение | Обоснование |
|---|---|---|---|
| D-01 | Хранение токена в web | HttpOnly/Secure cookie; fallback — Bearer | безопасность; рекомендация sq-1 (ADR-005) |
| D-02 | Семантика «не запомнил» | Этап N провален → повторы через **5 мин** и **1 час** → слово продолжает с этапа **N+1**. | дословно ТЗ §2.1.2: «возвращается к следующему этапу повторения после того, на котором была нажата кнопка» |
| D-03 | Этапы 7–8 интервалов | +6 мес, +12 мес | лестница из 8 этапов: 5 мин → 1 ч → 1 день → 1 нед → 1 мес → 3 мес → 6 мес → 12 мес |
| D-04 | Как словарь попадает в систему | Сид-миграция бэкенда создаёт словарь по умолчанию | без админки в ядре свежий деплой иначе пуст (ADR-006) |
| D-05 | Что отдаёт клиент при ответе | `word_id`, `stage`, `idempotency_key` в теле `POST /reviews` | сервер не доверяет клиентскому счёту этапа; stage — для отлова 409 |
| D-06 | Разблокировка resend | 60 с: серверный rate limit (`retry_after`), не только клиентский таймер | клиентский таймер подделывается |

---

## WS-1 · Авторизация по email + OTP

**Цель**: новый сотрудник входит в приложение. Результат — сессия на 30 дней, попадание на экран обучения.

**Предусловия**: нет (публичная зона). Почта — рабочая, корпоративный домен.

### Шаги

**1. Открыть приложение → сплэш.**
Под капотом: клиент проверяет наличие сохранённого токена/cookie.

**2. Проверить сессию: `GET /auth/me`** (auth-заголовок, если токен есть).

```
Ответ 200: { "user": { "id": 1, "email": "ivanov@t.ru", "role": "user" }, "expires_at": "2026-12-02T18:00:00Z" }
Ответ 401: { "error": "unauthorized" }  →  сбросить токен, перейти к шагу 3
```

SQL (за кулисами):

```sql
SELECT u.id, u.email, u.role FROM sessions s
JOIN users u ON u.id = s.user_id
WHERE s.token_hash = <hash(token)> AND s.expires_at > now();
-- пусто → 401
```

**3. Показать экран входа** (одно поле email + кнопка «Войти»).

**4. Отправить код: `POST /auth/request-code`.**

```
Запрос:  { "email": "ivanov@t.ru" }
Ответ 202: { "resend_after": 60 }
Ответ 400: { "error": "forbidden_domain" }      → почта не корпоративная
Ответ 429: { "error": "rate_limited", "retry_after": 60 }
```

Под капотом, сервер:
1. проверяет домен против списка корпоративных (конфиг);
2. лимитирует повторные запросы на email (напр. 1 код в 60 c; не чаще N в час);
3. генерирует 6-значный код;
4. сохраняет **hash** кода (не открытый текст):

```sql
INSERT INTO otp_codes (user_email, code_hash, attempts, expires_at, used)
VALUES ('ivanov@t.ru', <bcrypt(code)>, 0, now() + interval '10 minutes', false);
```

5. отправляет письмо через SMTP-шлюз (внешняя интеграция; при недоступности — retry с backoff, ошибка не светится пользователю в log целиком);
6. **инвалидирует предыдущий код** (если был): `UPDATE otp_codes SET used = true WHERE user_email = 'ivanov@t.ru' AND used = false`.

**5. Экран кода + таймер 60 с.** (input-otp; отправка по кнопке или при вводе последней цифры.)

**6. Повторный код (опционально): `POST /auth/resend-code`** — только после 60 с (сервер отдаёт 429 раньше), старый код инвалидируется (см. шаг 4.6).

**7. Подтвердить код: `POST /auth/verify-code`.**

```
Запрос:  { "email": "ivanov@t.ru", "code": "123456" }
Ответ 200: { "access_token": "<jwt>", "expires_at": "2026-12-02T18:00:00Z" }
Ответ 401: { "error": "invalid_code", "attempts_left": 4 }
Ответ 410: { "error": "code_expired", "attempts_left": 0 }
```

Под капотом, сервер:
1. `SELECT code_hash, attempts, expires_at, used FROM otp_codes WHERE user_email = ? AND used = false ORDER BY id DESC LIMIT 1`;
2. проверки по порядку: не used → не истёк (TTL 10 мин) → attempts < 5 → hash совпадает;
3. при неверном коде: `UPDATE otp_codes SET attempts = attempts + 1 ...`; при 5-й попытке — код считается сгоревшим;
4. при успехе: создать пользователя, если нет (первый вход = регистрация по рабочей почте):

```sql
INSERT INTO users (email, role) VALUES ('ivanov@t.ru', 'user') ON CONFLICT (email) DO NOTHING;
-- сессия 30 дней:
INSERT INTO sessions (user_id, token_hash, expires_at) VALUES (<id>, <hash(token)>, now() + interval '30 days');
UPDATE otp_codes SET used = true WHERE id = <otp_id>;
```

5. отдать JWT (claims: `sub`, `role`, `exp`).

**8. Сохранить токен** (cookie HttpOnly/Secure — D-01), редирект на главный экран обучения.

### Эдж-кейсы WS-1

| Кейс | Поведение |
|---|---|
| Два вкладки / два устройства | Каждая сессия независима; 30-дневный TTL на session-строку, не на пользователя |
| Код не пришёл | Resend после 60 с; N-я попытка — подтверждение для пользователя (таймер) |
| Истёк TTL между вводом и submit | 410 + предложить resend |
| 5 неверных попыток | Код invalidated, необходим новый code request |
| Повторная отправка кода | Старый код сгорает (used=true) |

### Как проверить вручную

- curl `POST /auth/request-code` на несуществующий корпоративный email → 202 (не раскрываем существование).
- curl на некорпоративный домен → 400.
- Проверить: в `otp_codes` лежит hash, не открытый код.
- `verify-code` с неверным кодом ×5 → 401/410, дальше только через новый request-code.
- `GET /auth/me` с токеном → 200; без → 401.
- В `sessions` появилась строка с `expires_at` = +30 дней.

---

## WS-2 · Словарь: сид, выбор, скачивание, синк прогресса

**Цель**: у пользователя есть словарь по умолчанию; словарь скачан на клиент; локальный прогресс синхронизируется с бэком.

**Предусловие**: авторизация (WS-1).

### Шаги

**1. Сид словаря по умолчанию (деплой, один раз).**
Миграция/скрипт создаёт словарь и слова; ТЗ не фиксирует стартовый объём — пример 100 слов:

```sql
INSERT INTO vocabularies (name, words_count, version, is_default) VALUES ('Основной 100', 100, 1, true);
-- на каждое слово:
INSERT INTO words (vocabulary_id, word, translation, transcription, type) VALUES (1, 'sick', 'больной', 'sɪk', 'Прилагательное');
```

**2. Получить список словарей: `GET /vocabularies`.**

```
Ответ 200: { "vocabularies": [ { "id": 1, "name": "Основной 100", "words_count": 100, "is_default": true } ] }
```

**3. Выбор в настройках** (клиент хранит `vocabulary_id` локально; дефолт — `is_default`).

**4. Скачать словарь: `GET /vocabularies/{id}`.**

```
Ответ 200: {
  "vocabulary": { "id": 1, "name": "Основной 100", "version": 1 },
  "words": [ { "id": 1, "word": "sick", "transcription": "sɪk", "translation": "больной", "word_type": "Прилагательное" } ]
}
```

Под капотом: `SELECT ... FROM words WHERE vocabulary_id = 1` — весь словарь одним ответом (объём мал; пост-MVP — пагинация/дельты по версии). Клиент кладёт слова в локальное хранилище (IndexedDB/SQLite).

**5. Синк прогресса: `POST /vocabularies/{id}/sync`.**

```
Запрос:  { "progress": [ { "word_id": 1, "action": "new" }, { "word_id": 2, "action": "already_known" } ], "known_version": 1 }
Ответ 200: { "applied": 2, "conflicts": [], "server_version": 1 }
Ответ 409: { "error": "version_conflict", "server_version": 2 }  →  клиент перескачивает словарь (шаг 4)
Ответ 422: { "error": "validation_failed", "row": 3, "reason": "unknown word_id" }
```

Под капотом: каждая запись = строка `progress` (upsert по UNIQUE(user_id, word_id); конфликтует с существующей — отметка `already_known` не отменяется ротацией). Статус «уже знаю» из оффлайна пишется только если слово ещё `not_started`. Прогресс-точность: синк идемпотентен — повторная отправка тех же записей не дублирует (upsert).

**6. Сброс данных по словарю / всех данных (настройки).**

```
POST /vocabularies/{id}/reset          → удалить progress пользователя по словарю
POST /user-data/reset                  → удалить весь progress пользователя
Оба: 204. Локальный кэш клиента чистится следом.
```

### Эдж-кейсы WS-2

| Кейс | Поведение |
|---|---|
| Оффлайн-обучение, потом синк | Клиент шлёт батч с локальными временными метками; сервер применяет по порядку |
| Словарь обновился на бэке (новый version) | 409 → перескачивать словарь; прогресс сохраняется по `word_id` |
| Слово удалено из словаря | При синке — 422 c `unknown word_id`; клиент показывает «словарь обновлён» |
| Синк во время ротации | Новые этапы приходят в след. синке: `next_review_at`, `stage` (см. WS-3) |

### Как проверить вручную

- БД: после деплоя в `vocabularies` 1 строка, в `words` N строк.
- `GET /vocabularies/{id}` отдаёт полный словарь с `version`.
- Синк `already_known` → в `progress` статус `already_known`; повторный синк → 200 без дублей.
- Сброс → `progress` пользователя по словарю пуст, статистика обнулена.

---

## WS-3 · Цикл обучения: следующая карточка + ответ (обе ветки)

**Цель**: пользователь учит слова: очередь отдаёт due-карточки раньше новых; ответ «запомнил» двигает по лестнице; «не запомнил» — парные быстрые повторы. 8-е подтверждение = выучено.

**Предусловие**: авторизация, словарь выбран и скачан (WS-2).

### Шаги

**1. Открыть экран обучения → `GET /learning/next-card?vocabulary_id=1`.**

```
Ответ 200 (due):  { "type": "due", "card": { "word_id": 42, "word": "sick", "transcription": "sɪk", "translation": "больной",
                    "word_type": "Прилагательное", "orientation": "EN_RU", "stage": 2, "stage_total": 8 }, "next_review_at": "..." }
Ответ 200 (new):  { "type": "new", "card": { "...", "orientation": "EN_RU", "stage": 0 } }
Ответ 204:        (карточек больше нет → empty state)
```

Под капотом, queue engine (две последовательные выборки, не одна):

```sql
-- 1) due-карточки: в ротации и время наступило, самая ранняя первая
SELECT p.word_id, w.word, w.transcription, w.translation, w.type, p.stage, p.next_review_at
FROM progress p JOIN words w ON w.id = p.word_id
WHERE p.user_id = 1 AND p.vocabulary_id = 1
  AND p.status = 'in_rotation' AND p.next_review_at <= now()
ORDER BY p.next_review_at ASC
LIMIT 1;

-- пусто → 2) новые слова: нет строки progress у пользователя
SELECT w.id, w.word, w.transcription, w.translation, w.type
FROM words w
LEFT JOIN progress p ON p.word_id = w.id AND p.user_id = 1
WHERE w.vocabulary_id = 1 AND p.id IS NULL
LIMIT 1;

-- снова пусто → 204
```

- `orientation` для due: случайно `EN_RU` / `RU_EN`; для new: всегда `EN_RU`.
- Перевод **передаётся**, клиент скрывает до «Показать слово».
- Клиент генерирует и сохраняет `idempotency_key` (UUID) на показанную карточку (нужен для шага 3; D-05 — отдаёт `word_id` + `stage`, которые получил).

**2. Показать слово (по кнопке)** — локальное действие: `translation` открывается на карточке. Серверного запроса нет.

**3а. «Я запомнил» → `POST /reviews`.**

```
Запрос:  { "result": "remembered", "word_id": 42, "stage": 2, "idempotency_key": "uuid-..." }
Ответ 200: { "status": "in_rotation", "stage": 3, "next_review_at": "2026-01-14T14:40:00Z",
             "learned": false, "stats_delta": { "in_progress": 0, "memorized": 0 } }
```

**3б. «Не запомнил» → `POST /reviews`** (запуск цикла быстрых повторов, D-02).

```
Запрос:  { "result": "not_remembered", "word_id": 42, "stage": 2, "idempotency_key": "uuid-..." }
Ответ 200: { "status": "in_rotation", "stage": 2, "quick_repeat_step": 1, "next_review_at": "<now + 5 мин>",
             "learned": false, "stats_delta": {} }
```

Под капотом (общий для 3а/3б), review engine. Механика ветки «не запомнил» (ТЗ §2.1.2, D-02):

1. Провал на регулярном этапе N (`quick_repeat_step = 0`) → быстрый повтор 1: `next_review_at = now + 5 мин`, `stage = N` без изменений, `quick_repeat_step = 1`.
2. Ответ (любой) на быстром повторе 1 → быстрый повтор 2: `next_review_at = now + 1 час`, `quick_repeat_step = 2`.
3. «Запомнил» на быстром повторе 2 → выход из цикла: `stage = N+1`, `quick_repeat_step = 0`, `next_review_at = now + interval[N+1]`. Если `N = 8` — слово сразу `memorized` (`stage = 8`, `next_review_at = NULL`), как в регулярной ветке.
4. «Не запомнил» на быстром повторе 2 → цикл заново с п. 1 (снова `now + 5 мин`).

Слово НЕ застревает в `+5 мин`/`+1 ч` навсегда: выход на следующую ступень наступает по второму быстрому повтору.

```sql
-- идемпотентность: уже отвечали по этому ключу?
SELECT id, result, stage_at_review FROM review_history WHERE user_id = 1 AND idempotency_key = 'uuid-...';
-- есть → 200 { replayed: true, ... } и НИЧЕГО не меняем (выход)

-- блокировка строки прогресса
BEGIN;
SELECT stage, status, quick_repeat_step FROM progress WHERE user_id = 1 AND word_id = 42 FOR UPDATE;
-- пусто или stage != 2 или status != in_rotation → ROLLBACK, 409 { error: "stale_card", actual_card: {...} }

-- регулярный ответ, не быстрый повтор (quick_repeat_step = 0):
--   «запомнил»: stage 8 → status = memorized, next_review_at = NULL, stats.in_progress--, stats.memorized++;
--               stage < 8 → stage +1, next_review_at = now + interval[stage+1]  ← 5мин→1ч→1д→1н→1мес→3мес→6мес→12мес
--   «не запомнил»: stage не меняется; quick_repeat_step = 1; next_review_at = now + 5 минут   ← запуск быстрого цикла

-- ответ на быстром повторе 1 (quick_repeat_step = 1):            ← любой результат
--   quick_repeat_step = 2; next_review_at = now + 1 час
-- ответ на быстром повторе 2 (quick_repeat_step = 2):
--   «запомнил» → если N = 8: status = memorized, next_review_at = NULL, stats.memorized++
--               иначе: stage = N+1; quick_repeat_step = 0; next_review_at = now + interval[N+1]
--   «не запомнил» → quick_repeat_step = 1; next_review_at = now + 5 минут   ← цикл заново

UPDATE progress SET stage = <new>, status = <new>, next_review_at = <new>, quick_repeat_step = <new> WHERE user_id = 1 AND word_id = 42;
INSERT INTO review_history (user_id, word_id, idempotency_key, result, stage_at_review) VALUES (1, 42, 'uuid-...', 'remembered', 2);
UPDATE statistics SET in_progress = in_progress + <delta>, memorized = memorized + <delta> WHERE user_id = 1 AND vocabulary_id = 1;
COMMIT;
```

**4. Клиент обновляет локальный кэш** (`stage`, `next_review_at`), показывает результирующее состояние: «Слово выучено» при `learned: true`; иначе — следующая карточка (возврат к шагу 1).

**5. 409 stale_card** (ответ по устаревшей карточке — например, оффлайн-дубль): клиент подменяет карточку на `actual_card` и показывает её, НЕ считая ответ повторённым.

### Лестница интервалов (D-03)

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

### Логика очереди суммарно

1. Приоритет: due-карточки (`in_rotation AND next_review_at <= now`), `ORDER BY next_review_at ASC`.
2. Новые слова — только когда due пуста; выборка `progress IS NULL`.
3. Ответ «не помню» на новом слове = вход в ротацию с этапом 1, `next_review_at = now + 5 мин`.
4. Ответ «уже знаю» на новом слове = `status = already_known`, из очереди навсегда, в статистику.
5. 8-е «запомнил» = `memorized`, `next_review_at = NULL`, из очереди, в статистику «выучено».

### Эдж-кейсы WS-3

| Кейс | Поведение |
|---|---|
| Двойной тап по кнопке | Оба запроса уходят с одним `idempotency_key` → второй отвечает `replayed: true`, прогресс не задваивается |
| Ответ прилетел после другого устройства/вкладки | 409 → показать актуальную карточку |
| Слово удалено из словаря админом | next-card его не вернёт (LEFT JOIN progress — строка остаётся, но word пропал из выборки; синк избавится от строки) |
| Due-очередь пуста, новых нет | 204 → empty state «все слова пройдены» |
| 60 due-карточек за раз (после отпуска) | Порядок строго по `next_review_at ASC`; каждая по одной за запрос |
| Провалы подряд на слове | Цикл «+5 мин → +1 ч → этап N+1»; при повторном провале на быстром повторе 2 цикл стартует заново (не застревает навсегда в +5 мин/+1 ч) |

### Как проверить вручную

- Сид 3 слова; на все три ответ `not_remembered` → они в ротации stage=1, all `next_review_at = +5 мин`.
- За 1 минуту: `next-card` → 204 (ничего не due) — **не** показывать не-due карточки.
- Через 5+ минут: первая due-карточка — ровно одна, `ORDER BY` соблюдается.
- Дважды отправить один `idempotency_key` → второй ответ `replayed: true`; `review_history` — одна строка.
- Прогнать 8 этапов (руками поправляя `next_review_at` в БД) → `status = memorized`, `next_review_at = NULL`, statistics.memorized +1.
- Ответ «запомнил» с неверным `stage` → 409; БД не изменилась.

---

## WS-4 · Админка: импорт словаря из XLS (stretch)

**Цель**: модератор загружает словарь из `.xlsx` с предпросмотром и подтверждением. Роли — RBAC.

**Условие stretch**: ядро скелета WS-1..WS-3 админку не требует (словарь приходит сидом, WS-2). Этот путь включается, когда нужен полноценный путь администратора.

**Предусловие**: модератор авторизован (WS-1 с ролью `moderator`/`admin`).

### Шаги

**1. Открыть словарь: `GET /admin/vocabularies/{id}`** → `200 { words: [...], version: 1 }` (RBAC middleware: роль moderator+).

**2. Загрузить файл: `POST /admin/import`** (multipart; флаг `mode: merge|replace`).

Под капотом, XLS worker:
1. парсит колонки `word`, `translation`, `transcription`, `type`;
2. валидирует: обязательные колонки, допустимые `type`, **уникальность `word`** внутри файла и против словаря;
3. ошибки → собирает список:

```
Ответ 422: { "error": "validation_failed", "errors": [ { "row": 3, "column": "type", "reason": "invalid enum value" } ] }
// данные НЕ изменены
```

4. без ошибок — строит diff (created/updated):

```
Ответ 200: { "preview": { "created": 90, "updated": 10 }, "import_token": "<uuid>", "version": 1 }
```

**3. UI показывает preview → модератор жмёт «Подтвердить».**

**4. Коммит: `POST /import/commit`.**

```
Запрос:  { "import_token": "<uuid>" }
Ответ 200: { "imported_count": 100, "version": 2 }
```

Под капотом:

```sql
BEGIN;
-- upsert слов (merge) или полная замена (replace: DELETE words WHERE vocabulary_id, затем INSERT)
INSERT INTO words (vocabulary_id, word, translation, transcription, type) VALUES (...)
  ON CONFLICT (vocabulary_id, word) DO UPDATE SET translation = EXCLUDED.translation, ...;
UPDATE vocabularies SET version = version + 1, words_count = <итог> WHERE id = 1;
INSERT INTO audit_log (admin_id, action, entity_type, entity_id, payload) VALUES (1, 'vocabulary_import', 'vocabulary', 1, '{"created":90,"updated":10}');
COMMIT;
```

**5. Редактирование слова (inline): `PATCH /words/{id}` c `If-Match: version`.** Конфликт → `409` + актуальные данные; совпадение — `UPDATE ... version++` + audit.

**6. Экспорт: `GET /admin/vocabularies/{id}/export.xlsx`** → `200` файл `vocabulary-{id}.xlsx`, собранный backend (колонки word/translation/transcription/type).

### Эдж-кейсы WS-4

| Кейс | Поведение |
|---|---|
| Файл с дублями слов | 422, весь импорт отклоняется, ничего не записано |
| replace сбросит прогресс? | Нет: `progress` привязан к `word_id`; replace не трогает `progress`, но удалённые слова дают 422 при синке (WS-2) |
| Два модератора правят параллельно | Optimistic locking: второй получает 409, перезагружает |
| Валидный preview, потом приходит новый импорт того же словаря | `import_token` невалиден → 404/410; повторить импорт |

### Как проверить вручную

- `.xlsx` с ошибкой в 3-й строке → 422, `words` не изменился.
- Валидный файл → preview(200) → commit → `version` +1, `words_count` обновлён, в `audit_log` строка.
- `PATCH` со старым `If-Match` → 409.
- Дайджест экспорта открывается в Excel, колонки на месте.

---

## Что НЕ входит в скелет (явно)

- Все post-MVP фичи (`docs/architecture/non-mvp.md`): профиль, лидерборд, content of the day, пуши, PWA, ручное повторение, примеры употребления из внешних API.
- Drag'n'drop карточек (MVP — только кнопки).
- Настройки темы (MVP: локальный переключатель, вне путей).

## Связанные документы

- Контракты и матрица эндпоинтов: [`ws-contracts.md`](ws-contracts.md)
- Диаграммы: [sq-1-auth](../diagrams/sq-1-auth.md) · [sq-2-next-card](../diagrams/sq-2-next-card.md) · [sq-3-review](../diagrams/sq-3-review.md) · [sq-4-admin](../diagrams/sq-4-admin.md) · [uc-1-context](../diagrams/uc-1-context.md) · [uc-2-learning](../diagrams/uc-2-learning.md)
- ТЗ: [01-tz](../requirements/01-tz.md)
