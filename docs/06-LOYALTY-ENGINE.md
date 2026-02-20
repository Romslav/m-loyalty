# S-06 · Loyalty Engine — Balls, Rules, Levels, Promos

> **Версия:** 1.0  
> **Дата:** 2026-02-20  
> **Статус:** READY FOR IMPLEMENTATION  
> **Предыдущий:** [S-05 Billing & Subscriptions](./05-BILLING-SUBSCRIPTIONS.md)

---

## Содержание

1. [Архитектура Loyalty Engine](#1-архитектура)
2. [Balls — Баланс и транзакции](#2-balls)
3. [Earn Flow](#3-earn-flow)
4. [Redeem Flow](#4-redeem-flow)
5. [Expiration Flow](#5-expiration-flow)
6. [BallTransfer](#6-balltransfer)
7. [Loyalty Rules](#7-loyalty-rules)
8. [Loyalty Levels](#8-loyalty-levels)
9. [Promo Campaigns](#9-promo-campaigns)
10. [ConflictResolution](#10-conflictresolution)
11. [CRON Jobs](#11-cron-jobs)
12. [API Endpoints](#12-api-endpoints)
13. [Error Handling](#13-error-handling)
14. [Tests](#14-tests)
15. [Чеклист P0/P1/P2](#15-чеклист)

---

## 1. Архитектура

### Обзор

Loyalty Engine — ядро системы. Отвечает за начисление (EARN), списание (REDEEM), сгорание (EXPIRE) баллов, управление уровнями (Bronze/Silver/Gold/…) и промо-кампаниями.

```
POS Webhook / Manual
        │
        ▼
┌─────────────────────────────────┐
│    LoyaltyCalculationService    │  (pre-calculation, read-only)
│  earnPercent × checkAmount      │
│  + levelBonus + promoBonus      │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│   LoyaltyTransactionService     │  (write, Prisma transaction)
│   SELECT … FOR UPDATE           │
│   REDEEM promo → regular        │
│   EARN → REGULAR balance        │
│   GuestVisit + QR rotate        │
└────────────┬────────────────────┘
             │
        ┌────┴──────┐
        ▼           ▼
  BallTransaction  GuestCard
   (audit trail)  (live balance)
```

### Модели данных (ключевые)

| Модель | Назначение |
|---|---|
| `GuestCard` | `regularBalance`, `promoBalance`, `totalBalance`, `lifetimeSpent`, `loyaltyLevelId` |
| `BallTransaction` | Каждое движение баллов: тип, сумма, `balanceBefore/After`, snapshot правила |
| `LoyaltySystem` | Настройки тенанта: `defaultEarnPercentage`, `minCheckAmount`, `maxRedeemPercentage`, `inactivityExpireDays`, `conflictResolutionStrategy` |
| `LoyaltyRule` | Правило начисления с `conditions` (JSONB): `minCheckAmount`, `daysOfWeek`, `timeRange`, `categories` |
| `LoyaltyLevel` | Порог `threshold` (lifetimeSpent), бонус `earnBonus` % |
| `LoyaltyPromo` | Промо: тип `PromoTriggerType`, `bonusPoints`, `expiresInDays`, `conflictStrategy` |
| `PromoBallGranted` | Выданные промо-баллы: `amountInitial`, `amountRemaining`, `expiresAt` |

### BallTransactionType enum

```
EARN | REDEEM | MANUAL_ADD | MANUAL_SUBTRACT | PROMO
EXPIRE | TRANSFER_SENT | TRANSFER_RECEIVED | REFUND | CANCEL
```

---

## 2. Balls

### regularBalance vs promoBalance

| Параметр | Regular | Promo |
|---|---|---|
| Источник | EARN от чека, MANUAL_ADD | Промо-кампании (Birthday, FirstCheck, …) |
| Сгорание | Через `inactivityExpireDays` (≥90 дней) | Через `expiresAt` записи `PromoBallGranted` |
| Списание | Второй приоритет | Первый приоритет (расходуется раньше) |
| Перевод | Да (`allowTransfers = true`) | Нет |

**totalBalance** — вычисляемое: `regularBalance + promoBalance`.

Invariant: `totalBalance ≥ 0`. При любой записи `BallTransaction`:
```
balanceAfter = balanceBefore + amount
```
Если `balanceAfter < 0` — `throw InsufficientBalanceException`.

---

## 3. Earn Flow

### 3.1. Алгоритм начисления

```
1. checkAmount = checkAmountOriginal - redeemedPoints (чистая сумма)
2. Найти активные LoyaltyRule (ACTIVE, соответствуют conditions)
3. Применить ConflictResolutionStrategy (COMBINE_ALL / MAX_ONLY / FIRST_ONLY)
4. Применить earnBonus уровня (Silver +5%, Gold +10%)
5. earnAmount = floor(checkAmountOriginal × earnPercent / 100)
6. earnAmount = 0 → не создавать транзакцию
7. BallTransaction { type: EARN, balanceType: REGULAR }
8. GuestCard: regularBalance += earnAmount, lifetimeSpent += checkAmount,
             totalVisits++, lastVisitAt = now, lastActivityAt = now
9. checkLevelUpgrade(lifetimeSpent) → if new level → notify
```

> **floor** — всегда округление вниз. Баллы целые.

### 3.2. LoyaltyRule — Conditions (JSONB)

```json
{
  "minCheckAmount": 500,
  "daysOfWeek": [1, 2, 3, 4, 5],
  "timeRange": { "from": "10:00", "to": "22:00" },
  "categories": ["pizza", "pasta"],
  "restaurants": ["rest-uuid-1"]
}
```

Все условия — AND-логика. Если условие не указано — не проверяется.

### 3.3. Rule Versioning

При каждом `PUT /loyalty/rules/:id` создаётся новая `LoyaltyRuleVersion`:
- `effectiveFrom = NOW`
- Предыдущая версия: `effectiveUntil = NOW`
- В `BallTransaction` сохраняется `ruleSnapshot` (JSON-снепшот версии)

### 3.4. Manual Add (MANUAL_ADD)

```typescript
// Доступно: ADMIN, MANAGER (permission: loyalty:manual:adjust)
ManualAddBallsDto {
  guestCardId: string;
  type: 'ADD' | 'SUBTRACT';
  amount: number;         // max 10 000 за операцию
  balanceType: 'REGULAR' | 'PROMO';
  reason: string;         // min 5 символов
  expiresAt?: Date;       // только для PROMO
  notifyGuest?: boolean;
}
```

ActivityLog: `BALLS_MANUAL_ADD` / `BALLS_MANUAL_SUBTRACT` с полем `reason`.

---

## 4. Redeem Flow

### 4.1. Алгоритм списания

```
1. Проверить checkAmount >= LoyaltySystem.minCheckAmount (default 50 RUB)
   → если нет: throw CHECK_AMOUNT_TOO_LOW

2. maxRedeemByPercent = floor(checkAmount × maxRedeemPercentage / 100)
   (maxRedeemPercentage: 10–100, default 20)

3. maxRedeemTotal = min(totalBalance, maxRedeemByPercent)

4. requestedRedeem > maxRedeemTotal
   → throw REDEEM_LIMIT_EXCEEDED ("Cannot redeem more than X balls")

5. Расход: сначала promoBalance, затем regularBalance
   promoUsed = min(requestedRedeem, promoBalance)
   regularUsed = requestedRedeem - promoUsed

6. BallTransaction × 2 (если promoUsed > 0 И regularUsed > 0):
   - { type: REDEEM, amount: -promoUsed, balanceType: PROMO }
   - { type: REDEEM, amount: -regularUsed, balanceType: REGULAR }

7. finalCheckAmount = checkAmount - requestedRedeem
```

### 4.2. Ограничения

| Параметр | Значение |
|---|---|
| `minCheckAmount` | 50 RUB (настраивается Admin) |
| `maxRedeemPercentage` | 10–100%, default 20% |
| `minBallsToRedeem` | 1 |
| `allowFullZero` | false (баланс не может стать отрицательным) |

---

## 5. Expiration Flow

### 5.1. Inactivity Expire (regularBalance)

```
Trigger: CRON 02:00 UTC
Condition:
  status = ACTIVE
  AND regularBalance > 0
  AND lastActivityAt < NOW - inactivityExpireDays
  (default: 90 дней, min 90)

Action:
  BallTransaction { type: EXPIRE, amount: -regularBalance, balanceType: REGULAR }
  GuestCard: regularBalance = 0, ballsExpireAt = null
  Notify: если enabled → Telegram/Email за 3, 7, 14 дней до
```

### 5.2. Promo Expire (promoBalance)

```
Trigger: CRON 02:00 UTC
Condition:
  PromoBallGranted.expiresAt < NOW
  AND PromoBallGranted.amountRemaining > 0

Action (per grant):
  BallTransaction { type: EXPIRE, amount: -amountRemaining, balanceType: PROMO, promoId }
  PromoBallGranted: amountRemaining = 0
  GuestCard: promoBalance -= amountRemaining, totalBalance -= amountRemaining
```

### 5.3. Expiration Notification Schedule

```typescript
LoyaltySystem.expirationNotifications = {
  enabled: true,
  channels: ['TELEGRAM', 'EMAIL'],
  daysBeforeExpiration: [3, 7, 14],
  notifyForRegular: true,
  notifyForPromo: true
}
```

---

## 6. BallTransfer

### 6.1. Условия

```typescript
LoyaltySystem.transferSettings = {
  allowTransfers: false,          // по умолчанию выключено
  minTransferAmount: 1,
  maxTransferAmountPerOperation: 5000,
  maxTransfersPerDay: 10000,
  transferCooldown: 300           // 5 минут между переводами
}
```

- Переводы только из `regularBalance`
- Внутри тенанта (cross-tenant — запрещено)
- Верификация: SMS / Telegram код (TTL 5 мин)
- Rate limiting: 3 операции / 60 сек
- Запрет self-transfer

### 6.2. Flow

```
POST /v1/loyalty/transfer
  fromCardId, toPhone, amount, verificationCode

1. Проверить allowTransfers = true
2. Проверить cooldown (5 мин с последней операции)
3. Проверить лимиты (per operation, per day)
4. Верифицировать код
5. Prisma transaction:
   BallTransaction { type: TRANSFER_SENT, amount: -amount, guestCardId: sender }
   BallTransaction { type: TRANSFER_RECEIVED, amount: +amount, guestCardId: receiver }
   BallTransfer { status: COMPLETED }
6. ActivityLog + уведомления обоим
```

---

## 7. Loyalty Rules

### 7.1. CRUD API

```
GET    /v1/loyalty/rules          — список (фильтр: status, tenantId)
POST   /v1/loyalty/rules          — создать (ADMIN+)
GET    /v1/loyalty/rules/:id      — детали
PUT    /v1/loyalty/rules/:id      — обновить (создаёт новую версию)
DELETE /v1/loyalty/rules/:id      — ARCHIVED (soft delete, effectiveUntil = NOW)
```

### 7.2. CreateLoyaltyRuleDto

```typescript
export class CreateLoyaltyRuleDto {
  @IsString() @MinLength(3) @MaxLength(100)
  name: string;

  @IsNumber() @Min(1) @Max(100)
  earnPercent: number;

  @IsEnum(['ACTIVE', 'DRAFT', 'ARCHIVED'])
  status: string;

  @IsOptional() @ValidateNested() @Type(() => RuleConditionsDto)
  conditions?: RuleConditionsDto;

  @IsNumber() @Min(1) @Max(1000)
  priority: number;
}
```

### 7.3. ConflictResolution (правила между собой)

| Стратегия | Поведение |
|---|---|
| `COMBINE_ALL` | Суммируются все подходящие правила |
| `MAX_ONLY` | Берётся только правило с максимальным earnAmount |
| `FIRST_ONLY` | Берётся только правило с наивысшим приоритетом (min priority number) |

---

## 8. Loyalty Levels

### 8.1. CRUD API

```
GET    /v1/loyalty/levels
POST   /v1/loyalty/levels
PUT    /v1/loyalty/levels/:id
DELETE /v1/loyalty/levels/:id?moveToLevel=:targetId
POST   /v1/loyalty/levels/recalculate   — ручной пересчёт (Owner)
```

### 8.2. One Way Up Logic

- Апгрейд — автоматически и сразу при достижении `threshold` по `lifetimeSpent`
- **Даунгрейд** — отключён по умолчанию (`allowDowngrade = false`)
- При `allowDowngrade = true`: проверяется активность за `downgradeAfterDays` (default 180)
- Удаление уровня, у которого есть гости: `→ 400 LEVEL_HAS_GUESTS` + список гостей

### 8.3. CRON пересчёт уровней

```
CRON: 03:00 UTC (per tenant, через BullMQ queue)

Для каждой активной GuestCard:
  newLevel = findLevelBySpent(lifetimeSpent, loyaltySystem.levels)
  if newLevel.id !== card.loyaltyLevelId:
    - обновить loyaltyLevelId
    - создать LevelHistory запись
    - UPGRADE → уведомить гостя
    - DOWNGRADE (если allowDowngrade) → уведомить гостя
```

### 8.4. Уровни по умолчанию (пример)

| Уровень | Порог lifetimeSpent | Бонус к earnPercent |
|---|---|---|
| Bronze | 0 | 0% |
| Silver | 150 000 ₽ | +5% |
| Gold | 400 000 ₽ | +10% |

---

## 9. Promo Campaigns

### 9.1. PromoTriggerType

| Тип | Описание |
|---|---|
| `BIRTHDAY_GUEST` | День рождения гостя |
| `BIRTHDAY_CHILD` | День рождения ребёнка гостя |
| `FIRST_CHECK` | Первый чек в заведении |
| `INACTIVITY` | Нет посещений N дней |
| `WEEKDAY` | Определённый день недели |
| `TIME_RANGE` | Определённое время |
| `CHECK_AMOUNT` | При сумме чека ≥ X |
| `CUSTOM` | Произвольные условия Admin |

### 9.2. CRUD API

```
GET    /v1/loyalty/promos
POST   /v1/loyalty/promos
GET    /v1/loyalty/promos/:id
PUT    /v1/loyalty/promos/:id
DELETE /v1/loyalty/promos/:id
GET    /v1/loyalty/promos/eligible?guestCardId=xxx   — доступные промо для гостя
```

### 9.3. CreateLoyaltyPromoDto

```typescript
{
  name: string;
  type: PromoTriggerType;
  bonusPoints: number;
  expiresInDays?: number;     // null = не сгорает
  conditions?: {
    minCheckAmount?: number;
    validRestaurants?: string[];
    daysOfWeek?: number[];
    timeRange?: { from: string; to: string };
  };
  conflictStrategy?: 'COMBINE_WITH_BASE' | 'COMBINE_ALL' | 'MAX_ONLY';
  notificationSettings?: {
    sendOnBirthday?: boolean;
    sendDaysBefore?: number;
    channels?: NotificationChannel[];
  };
  maxUsesTotal?: number;
  maxUsesPerGuest?: number;
}
```

### 9.4. PromoBallGranted lifecycle

```
1. Условие промо выполнено → create PromoBallGranted {
     amountInitial: bonusPoints,
     amountRemaining: bonusPoints,
     expiresAt: NOW + expiresInDays
   }
2. BallTransaction { type: PROMO, amount: +bonusPoints, balanceType: PROMO }
3. GuestCard: promoBalance += bonusPoints
4. При списании: уменьшается amountRemaining в PromoBallGranted
5. При сгорании (CRON): amountRemaining → 0, BallTransaction type: EXPIRE
```

---

## 10. ConflictResolution

### 10.1. Промо между собой

```typescript
LoyaltySystem.promoConflictStrategy:
  COMBINE_ALL  // сумма всех промо-бонусов
  MAX_ONLY     // берётся только промо с max bonusPoints
  FIRST_ONLY   // берётся промо с наивысшим priority (Admin 0–1000)

// Per-pair override:
LoyaltySystem.promoOverrides = [
  { promo1: 'birthday-promo', promo2: 'wednesday-promo', strategy: 'COMBINE_ALL' },
  { promo1: 'first-check-promo', promo2: 'birthday-promo', strategy: 'MAX_ONLY' }
]
```

### 10.2. Пример COMBINE_ALL

```
Birthday promo:   +1000 balls
Wednesday promo:  +500 balls
First-check promo: +1000 balls
→ итого: 2500 promo balls
```

### 10.3. Запись в PromoBallGranted

```json
{
  "activePromos": ["birthday", "wednesday", "first-check"],
  "conflictCount": 3,
  "resolutionStrategy": "COMBINE_ALL",
  "amountsPerPromo": { "birthday": 1000, "wednesday": 500, "firstCheck": 1000 },
  "finalAmount": 2500
}
```

---

## 11. CRON Jobs

| Очередь | CRON | Описание |
|---|---|---|
| `loyalty-recalc` | `0 3 * * *` | Пересчёт уровней per tenant |
| `ball-expiration` | `0 2 * * *` | Сгорание inactivity + promo |
| `pos-reconciliation` | `*/30 * * * *` | POS reconciliation (30 мин) |
| `notifications` | `0 23 * * *` | Уведомления о скором сгорании |
| `analytics` | `0 2 * * *` | Daily snapshot |

### BallExpirationProcessor

```typescript
@Processor(QueueName.BALL_EXPIRATION)
export class BallExpirationProcessor {
  @Process('check-expiration')
  async processExpiration(job: Job) {
    // 1. Inactivity expire
    const inactiveCards = await this.prisma.guestCard.findMany({
      where: {
        status: 'ACTIVE',
        lastActivityAt: { lt: new Date(Date.now() - tenant.inactivityExpireDays * 86400000) },
        totalBalance: { gt: 0 }
      }
    });
    for (const card of inactiveCards) {
      await this.prisma.$transaction(async (tx) => {
        await tx.ballTransaction.create({ data: { type: 'EXPIRE', amount: -card.regularBalance, balanceType: 'REGULAR', reason: `Inactivity ${tenant.inactivityExpireDays}d` } });
        await tx.guestCard.update({ where: { id: card.id }, data: { regularBalance: 0, promoBalance: 0, totalBalance: 0 } });
      });
      if (tenant.notifyBeforeExpiration) await this.notificationService.send(card.id, 'BALLS_EXPIRED');
    }

    // 2. Promo expire
    const expiredPromos = await this.prisma.promoBallGranted.findMany({
      where: { expiresAt: { lt: new Date() }, amountRemaining: { gt: 0 } }
    });
    for (const pg of expiredPromos) {
      await this.prisma.$transaction(async (tx) => {
        await tx.ballTransaction.create({ data: { type: 'EXPIRE', amount: -pg.amountRemaining, balanceType: 'PROMO', promoId: pg.promoId } });
        await tx.promoBallGranted.update({ where: { id: pg.id }, data: { amountRemaining: 0 } });
        await tx.guestCard.update({ where: { id: pg.guestCardId }, data: { promoBalance: { decrement: pg.amountRemaining }, totalBalance: { decrement: pg.amountRemaining } } });
      });
    }
  }
}
```

---

## 12. API Endpoints

### 12.1. POS / Cashier API

| Метод | URL | Описание |
|---|---|---|
| `POST` | `/v1/loyalty/calculate` | Pre-calculation (read-only) |
| `POST` | `/v1/loyalty/transactions` | Commit EARN/REDEEM (idempotent) |
| `POST` | `/v1/loyalty/manual-adjustment` | MANUAL_ADD / MANUAL_SUBTRACT |
| `POST` | `/v1/loyalty/transfer` | Перевод баллов между картами |

### 12.2. Admin / Builder API

| Метод | URL | Описание |
|---|---|---|
| `GET/POST` | `/v1/loyalty/rules` | CRUD правил |
| `GET/PUT/DELETE` | `/v1/loyalty/rules/:id` | Версионирование |
| `GET/POST` | `/v1/loyalty/levels` | CRUD уровней |
| `GET/PUT/DELETE` | `/v1/loyalty/levels/:id` | One-way-up |
| `POST` | `/v1/loyalty/levels/recalculate` | Ручной пересчёт |
| `GET/POST` | `/v1/loyalty/promos` | CRUD промо |
| `GET/PUT/DELETE` | `/v1/loyalty/promos/:id` | Версионирование промо |
| `GET` | `/v1/loyalty/promos/eligible` | Доступные промо для гостя |

### 12.3. Pre-calculation Response

```json
{
  "guestCard": { "id": "...", "regularBalance": 2500, "promoBalance": 300, "totalBalance": 2800, "level": "SILVER", "lifetimeSpent": 150000 },
  "calculation": {
    "earnPoints": 200,
    "appliedRules": [{ "ruleId": "...", "ruleName": "10%", "earnPercent": 10, "earnAmount": 200 }],
    "appliedPromos": [{ "promoId": "...", "promoName": "День рождения", "bonusPoints": 500, "expiresAt": "2026-02-28T23:59:59Z" }],
    "redeemLimit": { "maxPercent": 20, "maxAmount": 400, "minCheckAmount": 50, "requestedRedeem": 100, "allowedRedeem": 100, "promoBalanceUsed": 100, "regularBalanceUsed": 0 },
    "finalCheckAmount": 1900,
    "balanceAfterTransaction": 2700,
    "newBalanceAfterEarn": 2900
  },
  "warnings": ["Скоро сгорят промо-баллы", "Вы близко к уровню Gold"]
}
```

### 12.4. Commit Transaction — Headers

```
POST /v1/loyalty/transactions
Headers:
  Idempotency-Key: CHK-123456-IIKO-REST01
  Authorization: Bearer <token>
```

Idempotency: хранится в Redis 24ч. Повторный запрос с тем же ключом — возвращает кешированный ответ (HTTP 200, не 201).

---

## 13. Error Handling

### Error Codes

```typescript
export enum LoyaltyErrorCode {
  INSUFFICIENT_BALANCE     = 'INSUFFICIENT_BALANCE',
  GUEST_CARD_NOT_FOUND     = 'GUEST_CARD_NOT_FOUND',
  RULE_NOT_FOUND           = 'RULE_NOT_FOUND',
  LEVEL_NOT_FOUND          = 'LEVEL_NOT_FOUND',
  LEVEL_HAS_GUESTS         = 'LEVEL_HAS_GUESTS',
  PROMO_NOT_FOUND          = 'PROMO_NOT_FOUND',
  PROMO_ALREADY_USED       = 'PROMO_ALREADY_USED',
  PROMO_EXPIRED            = 'PROMO_EXPIRED',
  CHECK_AMOUNT_TOO_LOW     = 'CHECK_AMOUNT_TOO_LOW',
  REDEEM_LIMIT_EXCEEDED    = 'REDEEM_LIMIT_EXCEEDED',
  TRANSFER_COOLDOWN_ACTIVE = 'TRANSFER_COOLDOWN_ACTIVE',
  TRANSFER_TO_SELF         = 'TRANSFER_TO_SELF',
  TRANSFER_DISABLED        = 'TRANSFER_DISABLED',
  POS_CHECK_ALREADY_PROCESSED = 'POS_CHECK_ALREADY_PROCESSED',
}
```

### Error Response Format

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

### Pessimistic Lock + Retry

```typescript
// LoyaltyTransactionService
async processTransaction(dto: ProcessTransactionDto) {
  return this.executeWithRetry(() =>
    this.prisma.$transaction(async (tx) => {
      const guestCard = await tx.guestCard.findUnique({
        where: { id: dto.guestCardId }
        // PostgreSQL: SELECT … FOR UPDATE через raw query при необходимости
      });
      // … логика …
    }, { maxWait: 5000, timeout: 10000, isolationLevel: Prisma.TransactionIsolationLevel.Serializable })
  );
}

private async executeWithRetry<T>(operation: () => Promise<T>, maxRetries = 3): Promise<T> {
  let attempt = 0;
  while (attempt < maxRetries) {
    try { return await operation(); }
    catch (error) {
      if (error.code === 'P2034') { // deadlock
        attempt++;
        await new Promise(r => setTimeout(r, 100 * Math.pow(2, attempt)));
      } else throw error;
    }
  }
}
```

---

## 14. Tests

### 14.1. Unit Tests (coverage ≥ 80%)

| Suite | Сценарии | Target |
|---|---|---|
| `LoyaltyCalculationService` | EARN с уровнем, COMBINE_ALL, MAX_ONLY, round floor | 90% |
| `LoyaltyRulesMatchingService` | приоритет, конфликт одного priority, fallback, disabled | 90% |
| `RedeemLimitService` | 20%, баланс < лимита, checkAmount < 50 | 85% |
| `PromoValidationService` | required fields, overlap, dry-run, timeRange | 85% |
| `BallExpirationService` | boundary 365d, timezone UTC, race condition (идемпотентность) | 85% |

### 14.2. Integration Tests

```typescript
describe('POST /v1/loyalty/calculate', () => {
  it('200 — корректный расчёт');
  it('400 CHECK_AMOUNT_TOO_LOW — checkAmount < 50');
  it('401 — без токена');
});

describe('POST /v1/loyalty/transactions', () => {
  it('201 — EARN+REDEEM, баланс обновлён в БД');
  it('200 — повторный запрос с тем же Idempotency-Key (кеш)');
  it('409 — дублирующийся posCheckId');
});

describe('GET /v1/loyalty/rules', () => {
  it('200 ADMIN — список правил');
  it('403 GUEST — нет доступа');
});
```

### 14.3. Load Tests (k6)

```javascript
export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    'http_req_duration': ['p(95)<500', 'p(99)<1000'],
    'errors': ['rate<0.05'],
  }
};
// Target: 500 req/sec, p95 < 500ms, error rate < 1%
```

### 14.4. Property-Based Tests (fast-check)

```typescript
it('никогда не возвращает отрицательные баллы', () => {
  fc.assert(fc.property(
    fc.integer({ min: 1, max: 999999 }),
    fc.integer({ min: 0, max: 100 }),
    (amount, percentage) => {
      const result = service.calculateEarnedBalls({ checkAmount: amount, level: { earnPercentage: percentage }, rules: [] });
      return result >= 0;
    }
  ), { numRuns: 1000 });
});
```

---

## 15. Чеклист

### P0 — Critical (перед первым деплоем)

- [ ] `LoyaltyTransactionService.processTransaction` — Serializable transaction + retry
- [ ] `earnAmount = floor(checkAmount × earnPercent / 100)`
- [ ] Списание: сначала promoBalance, затем regularBalance
- [ ] `totalBalance` никогда не уходит в минус
- [ ] Idempotency по `posCheckId + posSystem + restaurantId` (UNIQUE constraint)
- [ ] `BallTransaction.balanceAfter = balanceBefore + amount` — инвариант
- [ ] CRON inactivity expire только для `regularBalance`
- [ ] CRON promo expire из `PromoBallGranted` уменьшает `promoBalance`
- [ ] Level upgrade — уведомление гостю

### P1 — High

- [ ] ConflictResolutionStrategy (COMBINE_ALL / MAX_ONLY / FIRST_ONLY)
- [ ] LoyaltyRule versioning (снепшот в `BallTransaction.ruleSnapshot`)
- [ ] CRUD LoyaltyLevel с one-way-up и защитой от удаления (LEVEL_HAS_GUESTS)
- [ ] CRUD LoyaltyPromo с dry-run endpoint
- [ ] Перевод баллов (BallTransfer) при `allowTransfers = true`
- [ ] ActivityLog для MANUAL_ADD, EXPIRE, TRANSFER, CANCEL

### P2 — Normal

- [ ] `GET /v1/loyalty/promos/eligible` — список доступных промо для гостя
- [ ] `POST /v1/loyalty/levels/recalculate` — ручной пересчёт Owner
- [ ] `promoOverrides` — per-pair ConflictResolution
- [ ] Load test: 500 req/sec, p95 < 500ms
- [ ] Property-based tests (fast-check, 1000 runs)
- [ ] BalanceAnomaly reconciliation job

---

*S-06 закрывает полную спецификацию Loyalty Engine. Следующий: S-07 — Telegram Bot & Mini App.*
