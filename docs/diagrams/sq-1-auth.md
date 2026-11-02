# SQ-1 · Авторизация по email и OTP

> Источник: `docs/sources/tbank-english-learning-uml.drawio`, страница `sq-auth` (v1.0 · 25.10.2026).
> Сплэш и проверка сессии: валидный токен сразу ведёт на обучение; иначе — email + OTP, resend и токен на 30 дней.

## Диаграмма

```mermaid
sequenceDiagram
    actor U as Сотрудник
    participant W as Web App · SPA
    participant B as Backend · Auth API
    participant DB as База данных · SQL
    participant M as Mail Service · SMTP

    Note over U,W: 01 · Открыть приложение (сплэш)
    W->>W: 02 · Проверить сохранённый токен
    W->>B: 03 · GET /auth/me
    B-->>W: 04 · 200 {user, expires_at}
    W->>U: 05 · Открыть экран обучения · сессия активна

    alt до: [токена нет / 401 / истёк]
        W->>U: 06 · Показать экран входа
        U->>W: 07 · Ввести рабочую почту, нажать «Войти»
        W->>B: 08 · POST /auth/request-code {email}
        B->>DB: 09 · Сохранить OTP hash, TTL, attempts
        B->>M: 10 · Отправить 6-значный код
        B-->>W: 11 · 202 {resend_after: 60}
        W->>U: 12 · Экран кода + таймер 60 с

        opt [код не пришёл / таймер истёк]
            U->>W: 13 · Запросить код повторно
            W->>B: 14 · POST /auth/resend-code
            B->>DB: 15 · Новый OTP; старый инвалидируется
            B-->>W: 16 · 202 {resend_after: 60}
        end

        U->>W: 17 · Ввести 6-значный код
        W->>B: 18 · POST /auth/verify-code {email, code}
        B->>DB: 19 · Проверить hash, TTL, attempts
        alt [код неверный / истёк / лимит попыток]
            B-->>W: 20 · 401 / 410 {attempts_left}
            W->>U: 21 · Ошибка; повторить ввод или запросить код
        else [код верный]
            B->>DB: 22 · Создать/найти user; сессия 30 дней; OTP → used
            B-->>W: 23 · 200 {access_token, expires_at}
            W->>W: 24 · Сохранить токен; Authorization: Bearer
            W->>U: 25 · Главный экран обучения
        end
    end
```

## Правила

| Блок | Правило |
|---|---|
| **СЕССИЯ** | Токен живёт 30 дней. `401` → повторный вход. Для web рекомендуется HttpOnly/Secure cookie (ADR-005). |
| **OTP** | 6 цифр, TTL 10 минут, ≤5 попыток. Письмо получает сотрудник; новый код инвалидирует старый. Хранится hash кода, не сам код. |
| **ОШИБКИ** | `202` — код отправлен · `400` — некорпоративная почта · `429` — rate limit · `401` — код неверный · `410` — код истёк/исчерпан лимит (с `attempts_left`) |

Эндпоинты и контракты: `docs/walking-skeleton/ws-contracts.md`.