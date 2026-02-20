# MAX LOYALTY PLATFORM — FULL SPECIFICATION

> **Полная спецификация проекта** | 7 модулей | Версия 1.0 | 2026-02-20  
> Объединённый документ на основе S-01 — S-07

---

## Навигация

| # | Модуль | Описание |
|---|---|---|
| [S-01](#s-01-full-system-overview) | Full System Overview | Архитектура, стек, Prisma Schema, API структура |
| [S-02](#s-02-auth-system) | Auth System | JWT, refresh, роли, PINs, сессии, Telegram-auth |
| [S-03](#s-03-rbac--permissions) | RBAC & Permissions | 30 прав, роли, Guards, ABAC, кэш |
| [S-04](#s-04-tenant--restaurant-management) | Tenant & Restaurant | Multi-tenancy, Owner Registration, лимиты, рестораны |
| [S-05](#s-05-billing--subscriptions) | Billing & Subscriptions | YooKassa/Stripe, апгрейд/даунгрейд, PASTDUE, Invoice |
| [S-06](#s-06-loyalty-engine) | Loyalty Engine | Баллы, правила, уровни, промо, CRON |
| [S-07](#s-07-telegram-bot--mini-app) | Telegram Bot & Mini App | Bot (Telegraf), Mini App (React), initData, push |

---

# S-01: Full System Overview

> Источник: `docs/01-FULL-SYSTEM-OVERVIEW.md`

См. полный текст в файле [01-FULL-SYSTEM-OVERVIEW.md](./01-FULL-SYSTEM-OVERVIEW.md)

### Ключевые решения архитектуры

- **Монорепо**: Turborepo + pnpm workspaces
- **Backend**: NestJS (TypeScript), PostgreSQL (Prisma ORM), Redis, BullMQ
- **Frontend**: Next.js 14 (Admin Panel), React + Vite (Mini App)
- **Auth**: JWT access (15 мин) + refresh (30 дней), роли: OWNER / RESTAURANT_ADMIN / MANAGER / CASHIER / GUEST
- **Multi-tenancy**: Pooled — единая БД, изоляция через `tenantId`
- **Инфраструктура**: Docker Compose (dev), Kubernetes (prod), S3/R2 (media), Cloudflare (CDN)

### Стек технологий

| Слой | Технология |
|---|---|
| Backend | NestJS + TypeScript |
| ORM | Prisma + PostgreSQL |
| Кэш/Очереди | Redis + BullMQ |
| Admin Panel | Next.js 14 |
| Mini App | React + Vite + @twa-dev/sdk |
| Telegram Bot | NestJS + Telegraf |
| CI/CD | GitHub Actions |
| Мониторинг | Grafana + Prometheus + Sentry |

---

# S-02: Auth System

> Источник: `docs/02-AUTH-SYSTEM.md`

См. полный текст в файле [02-AUTH-SYSTEM.md](./02-AUTH-SYSTEM.md)

### Ключевые решения

**JWT Payload:**
```typescript
{
  sub: string;         // userId
  tenantId: string;
  role: UserRole;
  permissions: string[]; // кэш из Redis
  type: 'access' | 'refresh';
  iat: number;
  exp: number;
}
```

**Token lifecycle:**
- Access token TTL: 15 минут
- Refresh token TTL: 30 дней (Single-Use Rotation)
- Refresh token хранится в HttpOnly cookie + Redis whitelist

**Стратегии входа:**
1. Email + Password (bcrypt, cost 12)
2. Phone + SMS-код (Twilio / SMSC, TTL 5 мин, max 3 попытки)
3. Telegram initData (HMAC-SHA256, TTL 60 сек)

**Cashier PIN:**
- 4–6 цифр, bcrypt cost 10
- Refresh token TTL: 8 часов (рабочая смена)
- Rate limit: 5 попыток / 15 мин → блокировка

**Ключевые Guards:**
- `JwtAuthGuard` — проверяет access token
- `RolesGuard` — проверяет роль из JWT
- `PermissionsGuard` — проверяет права из Redis-кэша (5 мин TTL)
- `TenantGuard` — проверяет активность tenant

**Sessions:**
- Модель `UserSession` в PostgreSQL
- Redis whitelist: `session:{userId}:{sessionId}` → revoked flag
- Максимум 5 активных сессий на пользователя

---

# S-03: RBAC & Permissions

> Источник: `docs/03-RBAC-PERMISSIONS.md`

См. полный текст в файле [03-RBAC-PERMISSIONS.md](./03-RBAC-PERMISSIONS.md)

### Роли системы

| Роль | Уровень | Описание |
|---|---|---|
| `PLATFORM_OWNER` | Платформа | Суперадмин платформы |
| `OWNER` | Тенант | Владелец заведения |
| `RESTAURANT_ADMIN` | Тенант | Администратор ресторана |
| `MANAGER` | Ресторан | Менеджер смены |
| `CASHIER` | Ресторан | Кассир |
| `GUEST` | Гость | Гость (Mini App) |

### 30 Permissions (сводка)

```
// Guests
guests:view | guests:create | guests:edit | guests:block | guests:export

// Balls (Loyalty)
balls:earn | balls:redeem | balls:manual:adjust | balls:view:history | balls:transfer

// Loyalty Settings
loyalty:rules:manage | loyalty:levels:manage | loyalty:promos:manage | loyalty:system:config

// Team
team:view | team:invite | team:edit | team:remove | team:pin:reset

// Restaurants
restaurants:view | restaurants:manage | restaurants:analytics:view

// Billing
billing:view | billing:manage

// Settings
settings:view | settings:manage

// Analytics
analytics:view | analytics:export

// Platform
platform:admin
```

### Кэш прав Redis (5 мин TTL)

```typescript
// Ключ: permissions:{userId}:{tenantId}
// Value: JSON строка массива Permission[]
// TTL: 300 секунд
// Инвалидация: при изменении роли / кастомных прав
```

### Маппинг роль → права

| Право | OWNER | ADMIN | MANAGER | CASHIER |
|---|---|---|---|---|
| `guests:view` | ✅ | ✅ | ✅ | ✅ |
| `guests:create` | ✅ | ✅ | ✅ | ✅ |
| `balls:earn` | ✅ | ✅ | ✅ | ✅ |
| `balls:redeem` | ✅ | ✅ | ✅ | ✅ |
| `balls:manual:adjust` | ✅ | ✅ | ✅ | ❌ |
| `loyalty:rules:manage` | ✅ | ✅ | ❌ | ❌ |
| `loyalty:system:config` | ✅ | ❌ | ❌ | ❌ |
| `team:invite` | ✅ | ✅ | ❌ | ❌ |
| `restaurants:manage` | ✅ | ✅ | ❌ | ❌ |
| `billing:manage` | ✅ | ❌ | ❌ | ❌ |
| `platform:admin` | ❌ | ❌ | ❌ | ❌ |

> `platform:admin` — только PLATFORM_OWNER

### ABAC — Attribute-Based Control

- **Manager**: видит только назначенные ему рестораны
- **Cashier**: операции только в своём ресторане
- Проверка через `UserRestaurantRole` (userId + restaurantId)

---

# S-04: Tenant & Restaurant Management

> Источник: `docs/04-TENANT-RESTAURANT-MANAGEMENT.md`

См. полный текст в файле [04-TENANT-RESTAURANT-MANAGEMENT.md](./04-TENANT-RESTAURANT-MANAGEMENT.md)

### Multi-tenancy стратегия

**Pooled Multi-Tenancy**: единая БД, изоляция через `tenantId` во всех таблицах.

**Prisma Middleware**: автоматически инъектирует `WHERE tenantId = X` для всех tenant-изолированных моделей:

```
Restaurant, GuestCard, BallTransaction, GuestVisit,
LoyaltyRule, LoyaltyLevel, LoyaltyPromo, POSIntegration,
POSTransaction, Notification, ActivityLog, AnalyticsDailySnapshot
```

### Owner Registration Flow

```
1. Суперадмин создаёт OwnerRegistrationLink (invite-токен, UUID, TTL 30 дней)
2. Owner переходит по ссылке: GET /owner-registration/validate/:token
3. Owner регистрируется: POST /owner-registration/register
4. Транзакция: User (role: OWNER) + Tenant + Subscription (TRIAL 14 дней)
                + TenantLimits + BillingInfo
5. OwnerRegistrationLink.usedAt = NOW (одноразовая)
```

### Тарифные планы

| Параметр | FREE | STANDARD | MEDIUM | PRO | ULTIMATE |
|---|---|---|---|---|---|
| Цена/мес | 0 ₽ | 5 000 ₽ | 15 000 ₽ | 35 000 ₽ | 100 000 ₽ |
| Рестораны | 1 | 1 | 3 | 5 | ∞ |
| Гости | ∞ | 500 | 2 000 | 6 000 | ∞ |
| POS-интеграции | 0 | 1 | 3 | 5 | ∞ |
| Admin-пользователи | 1 | 3 | 10 | 25 | ∞ |
| Trial | — | 14 дней | 14 дней | 14 дней | 14 дней |

### TenantLimits Enforcement

- `TenantLimitsGuard` + `@CheckLimit('restaurant')` декоратор на эндпоинтах
- При 90% → EventEmitter → уведомление Owner (Telegram + Email)
- `TenantLimitsService.increment/decrement` при создании/удалении ресурса
- CRON `recalculate()` раз в сутки для сверки счётчиков

### Restaurant CRUD

```
GET    /restaurants         — список (Manager видит только свои)
GET    /restaurants/:id     — детали
POST   /restaurants         — создать (проверка лимита)
PATCH  /restaurants/:id     — обновить
DELETE /restaurants/:id     — soft-delete + декремент счётчика
POST   /restaurants/:id/assign-manager
POST   /restaurants/:id/assign-cashier
```

---

# S-05: Billing & Subscriptions

> Источник: `docs/05-BILLING-SUBSCRIPTIONS.md`

См. полный текст в файле [05-BILLING-SUBSCRIPTIONS.md](./05-BILLING-SUBSCRIPTIONS.md)

### Архитектура биллинга

```
Owner/Admin
    │
    ▼
POST /billing/checkout ──► YooKassa/Stripe ──► Redirect URL
                                │
                    POST /webhooks/payment/{provider}
                                │
                       PaymentService
                       verifySignature → updatePayment → activateSub
                                │
                       SubscriptionService
                       updateStatus → updateLimits → sendNotification
```

### Провайдеры оплаты

| Провайдер | Регион | Методы |
|---|---|---|
| **YooKassa** | 🇷🇺 РФ | Карты, СБП, ЮMoney, Qiwi |
| **Stripe** | 🌍 Глобально | Visa/MC, SEPA, Apple/Google Pay |
| **MANUAL** | — | Счёт/перевод (ULTIMATE/CUSTOM) |

### Subscription Statuses

```
TRIAL → ACTIVE → PAST_DUE → NON_PAYMENT (→ READ-ONLY)
                          → CANCELLED
                          → CANCELLED_REFUND
                          → CANCELLED_CHARGEBACK
```

### Upgrade / Downgrade

- **Апгрейд** — немедленно. Рассчитывается prorated delta. Если delta > 0 — требует оплаты.
- **Даунгрейд** — отложен до конца периода (`pendingPlanChange`). CRON применяет в 01:00.
- **validateDowngrade()** — проверяет, что текущие ресурсы вписываются в новые лимиты.

### PASTDUE & Dunning Flow

```
День 0:  Оплата не прошла → PAST_DUE + уведомление
День +1: Повтор #1 + Email
День +3: Повтор #2 + Email "осталось 5 дней"
День +7: Повтор #3 + Email "последнее предупреждение"
День +8: NON_PAYMENT → Tenant заблокирован (READ-ONLY)
         Данные хранятся 30 дней
```

### CRON Jobs (Billing)

| CRON | Время | Задача |
|---|---|---|
| `0 1 * * *` | 01:00 | Применить запланированные даунгрейды |
| `0 2 * * *` | 02:00 | Автопродление подписок |
| `0 3 * * *` | 03:00 | Dunning flow |
| `0 4 * * *` | 04:00 | Проверка истёкших триалов |

### Invoice

- Номер: `ML-2026-0001` (ML-YYYY-NNNN)
- НДС 20% автоматически
- PDF генерация: BullMQ → puppeteer → S3/R2
- Метрики: MRR / ARR / Churn / ARPU / LTV

### PCI DSS

- Данные карты НИКОГДА не проходят через наш backend
- Сохраняем только `paymentMethodId` (токен провайдера)
- Redirect-based flow для первичной оплаты
- Автопродление через сохранённый токен (`save_payment_method: true`)

---

# S-06: Loyalty Engine

> Источник: `docs/06-LOYALTY-ENGINE.md`

См. полный текст в файле [06-LOYALTY-ENGINE.md](./06-LOYALTY-ENGINE.md)

### Архитектура

```
POS Webhook / Manual
        │
        ▼
LoyaltyCalculationService  (pre-calculation, read-only)
  earnPercent × checkAmount + levelBonus + promoBonus
        │
        ▼
LoyaltyTransactionService  (write, Prisma Serializable transaction)
  SELECT … FOR UPDATE (pessimistic lock)
  REDEEM promo → regular
  EARN → REGULAR balance
  GuestVisit + QR rotate
        │
  ┌─────┴──────┐
  ▼            ▼
BallTransaction  GuestCard
(audit trail)   (live balance)
```

### regularBalance vs promoBalance

| Параметр | Regular | Promo |
|---|---|---|
| Источник | EARN от чека, MANUAL_ADD | Промо-кампании |
| Сгорание | Inactivity (≥90 дней) | По `expiresAt` в `PromoBallGranted` |
| Списание | Второй приоритет | Первый приоритет |
| Перевод | Да | Нет |

**Invariant**: `totalBalance ≥ 0`. `balanceAfter = balanceBefore + amount`.

### EARN Flow (алгоритм)

```
1. checkAmount = checkAmountOriginal - redeemedPoints
2. Найти активные LoyaltyRule (ACTIVE + conditions match)
3. ConflictResolutionStrategy: COMBINE_ALL / MAX_ONLY / FIRST_ONLY
4. earnBonus уровня: Silver +5%, Gold +10%
5. earnAmount = floor(checkAmountOriginal × earnPercent / 100)
6. earnAmount > 0 → BallTransaction { type: EARN, REGULAR }
7. GuestCard: regularBalance++, lifetimeSpent++, lastActivityAt = now
8. checkLevelUpgrade(lifetimeSpent)
```

### REDEEM Flow (алгоритм)

```
1. checkAmount >= minCheckAmount (default 50 RUB)
2. maxRedeemByPercent = floor(checkAmount × maxRedeemPercentage / 100)
3. maxRedeemTotal = min(totalBalance, maxRedeemByPercent)
4. requestedRedeem > maxRedeemTotal → REDEEM_LIMIT_EXCEEDED
5. Расход: сначала promoBalance, затем regularBalance
6. finalCheckAmount = checkAmount - requestedRedeem
```

### Expiration

- **Inactivity** (regularBalance): CRON 02:00, условие `lastActivityAt < NOW - inactivityExpireDays` (default 90)
- **Promo** (promoBalance): CRON 02:00, из `PromoBallGranted.expiresAt < NOW`
- **Уведомления**: за 3, 7, 14 дней до сгорания

### Loyalty Levels (пример)

| Уровень | Порог lifetimeSpent | Бонус к earnPercent |
|---|---|---|
| Bronze | 0 | 0% |
| Silver | 150 000 ₽ | +5% |
| Gold | 400 000 ₽ | +10% |

- Апгрейд — автоматически и сразу
- Даунгрейд — выключен по умолчанию (`allowDowngrade = false`)
- CRON пересчёт: 03:00 UTC через BullMQ

### Promo Campaigns (триггеры)

```
BIRTHDAY_GUEST | BIRTHDAY_CHILD | FIRST_CHECK
INACTIVITY | WEEKDAY | TIME_RANGE | CHECK_AMOUNT | CUSTOM
```

### ConflictResolution

| Стратегия | Поведение |
|---|---|
| `COMBINE_ALL` | Суммируются все подходящие правила/промо |
| `MAX_ONLY` | Берётся только максимальное |
| `FIRST_ONLY` | Берётся только с наивысшим приоритетом |

### Idempotency

- `POST /v1/loyalty/transactions` — `Idempotency-Key` header
- Хранится в Redis 24ч
- Повторный запрос → кэшированный ответ HTTP 200

### CRON Jobs (Loyalty)

| Очередь | CRON | Описание |
|---|---|---|
| `loyalty-recalc` | `0 3 * * *` | Пересчёт уровней |
| `ball-expiration` | `0 2 * * *` | Сгорание inactivity + promo |
| `notifications` | `0 23 * * *` | Уведомления о скором сгорании |
| `pos-reconciliation` | `*/30 * * * *` | POS reconciliation (30 мин) |

---

# S-07: Telegram Bot & Mini App

> Источник: `docs/07-TELEGRAM-BOT-MINIAPP.md`

См. полный текст в файле [07-TELEGRAM-BOT-MINIAPP.md](./07-TELEGRAM-BOT-MINIAPP.md)

### Стек

| Компонент | Технология |
|---|---|
| Telegram Bot | NestJS + Telegraf (TypeScript, webhook) |
| Mini App | React + Vite + @twa-dev/sdk |
| Auth Mini App | initData HMAC-SHA256 верификация |
| Shared types | `packages/shared` |

### /start Registration Flow

```
1. Получить /start update
2. Найти User по telegramId
3. Не найден → RequestContact → SMS-код → верифицировать
4. Создать User { telegramId, phone } + GuestProfile + GuestCard
5. Или: найден по телефону → привязать telegramId
```

### Bot Команды

| Команда | Описание |
|---|---|
| `/start` | Регистрация / главное меню |
| `/card` | Показать карту (баланс, уровень, QR, 6-digit) |
| `/history [N]` | Последние N транзакций |
| `/promos` | Доступные промо |
| `/transfer` | Перевод баллов (сцена) |
| `/support` | Создать тикет поддержки |
| `/settings` | Настройки уведомлений |

### Mini App Экраны

| Экран | URL | Описание |
|---|---|---|
| **Card** | `/` | Баланс, уровень, прогресс, QR, 6-digit |
| **History** | `/history` | Транзакции с фильтрами |
| **Promos** | `/promos` | Доступные промо |
| **Transfer** | `/transfer` | Форма перевода баллов |
| **Personal Stats** | `/stats` | Аналитика гостя |
| **Settings** | `/settings` | Профиль, уведомления, отвязка |

### initData Верификация

```typescript
// 1. data_check_string = sorted params joined by \n
// 2. secret_key = HMAC-SHA256("WebAppData", botToken)
// 3. hash = HMAC-SHA256(secret_key, data_check_string)
// 4. Сравнить hash с params.hash
// 5. Проверить auth_date (max 60 сек)
// POST /v1/auth/telegram/miniapp → JWT
```

### Push Уведомления

| Тип | Триггер | Канал |
|---|---|---|
| `BALLS_EARNED` | После EARN-транзакции | Telegram |
| `BALLS_REDEEMED` | После REDEEM-транзакции | Telegram |
| `BALLS_EXPIRING_SOON` | CRON за 3/7/14 дней | Telegram, Email |
| `LEVEL_UPGRADED` | При апгрейде уровня | Telegram, Email |
| `PROMO_ACTIVATED` | При выдаче промо | Telegram |
| `TRANSFER_RECEIVED` | После перевода | Telegram |

**Idempotency**: Redis-ключ `telegram:push:{transactionId}` TTL 24ч

### UNIFIED vs SEPARATE Mode

| Режим | Поведение |
|---|---|
| `UNIFIED` | Одна GuestCard на весь тенант |
| `SEPARATE` | Отдельная GuestCard на ресторан (deep-link `?start=restaurantId`) |

### QR/6-digit Rotation

После каждой транзакции (EARN/REDEEM) QR-код и 6-digit код ротируются для безопасности.

### Rate Limiting (Bot)

```
General:  10 req/мин
SMS code: 3 SMS / 5 мин
Transfer: 3 операции / мин
// Redis: telegram:ratelimit:{telegramId}:{action}
```

### Multi-bot (Premium)

```
Owner создаёт бот через BotFather → вводит botToken в Admin Panel
Webhook: POST /api/telegram/webhook/:botToken
Каждый бот изолирован на уровне tenantId
```

---

# Сквозные концепции

## ActivityLog

Все значимые действия фиксируются в `ActivityLog`:

```typescript
model ActivityLog {
  id         String   @id @default(uuid())
  tenantId   String
  actorId    String?  // userId (null = system/CRON)
  action     String   // CARD_CREATED, BALLS_EARNED, SUBSCRIPTION_ACTIVATED, ...
  entityType String   // GUEST_CARD, SUBSCRIPTION, RESTAURANT, ...
  entityId   String
  oldValue   Json?
  newValue   Json?
  createdAt  DateTime @default(now())
  ipAddress  String?
  userAgent  String?
}
```

## Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Недостаточно баллов для списания",
    "details": { "requested": 500, "available": 285 }
  },
  "timestamp": "2026-02-20T14:00:00Z",
  "path": "/v1/loyalty/transactions",
  "requestId": "req-abc123"
}
```

## API Versioning

- Prefix: `/v1/*` для всех публичных API
- Webhook: `/api/telegram/webhook`
- Внутренние: `/internal/*` (без JWT, только network policy)

## Prisma Schema — Ключевые модели

```
User → UserTenantRole → Tenant → Subscription → Payment → Invoice
                     → TenantLimits
                     → Restaurant → UserRestaurantRole
                                  → GuestCard → BallTransaction
                                              → GuestVisit
                                              → PromoBallGranted
                     → LoyaltySystem → LoyaltyRule
                                     → LoyaltyLevel
                                     → LoyaltyPromo
                     → TelegramBot
                     → ActivityLog
                     → Notification
```

## Чеклист P0 — Критично (сводный)

### Auth (S-02)
- [ ] JWT access 15 мин + refresh 30 дней (SUR)
- [ ] Phone+SMS аутентификация
- [ ] Cashier PIN flow
- [ ] Session revocation (Redis whitelist)

### RBAC (S-03)
- [ ] PermissionsGuard на всех защищённых эндпоинтах
- [ ] Redis-кэш прав 5 мин TTL
- [ ] ABAC: Manager видит только свои рестораны

### Tenant (S-04)
- [ ] TenantMiddleware глобально
- [ ] Prisma middleware изолирует по tenantId
- [ ] Owner Registration транзакция (User+Tenant+Subscription+Limits)
- [ ] TenantLimitsGuard блокирует превышение

### Billing (S-05)
- [ ] Payment PENDING до редиректа
- [ ] Webhook подписи верифицированы (YooKassa IP+HMAC, Stripe constructEvent)
- [ ] Idempotency (Idempotence-Key)
- [ ] PCI DSS: только токен, не данные карты
- [ ] Dunning 0/+1/+3/+7/+8 дней

### Loyalty (S-06)
- [ ] Serializable transaction + retry (deadlock P2034)
- [ ] earnAmount = floor(...) — целые баллы
- [ ] totalBalance никогда < 0
- [ ] Idempotency по posCheckId (UNIQUE constraint)
- [ ] CRON: inactivity expire, promo expire, level recalc

### Telegram (S-07)
- [ ] /start: регистрация + SMS + создание GuestCard
- [ ] initData HMAC верификация (TTL 60 сек)
- [ ] Webhook с secret_token
- [ ] QR/6-digit ротация при каждой транзакции
- [ ] Push BALLS_EARNED идемпотентно (Redis TTL 24ч)

---

*Документ сгенерирован автоматически из S-01...S-07 | Max Loyalty Platform v1.0 | 2026-02-20*
