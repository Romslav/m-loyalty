# S-07 · Telegram Bot & Mini App

> **Версия:** 1.0  
> **Дата:** 2026-02-20  
> **Статус:** READY FOR IMPLEMENTATION  
> **Предыдущий:** [S-06 Loyalty Engine](./06-LOYALTY-ENGINE.md)

---

## Содержание

1. [Архитектура](#1-архитектура)
2. [Регистрация и авторизация гостя](#2-регистрация-и-авторизация)
3. [Telegram Bot — команды и сцены](#3-telegram-bot)
4. [Mini App — экраны и UX](#4-mini-app)
5. [initData — аутентификация Mini App](#5-initdata)
6. [Push-уведомления](#6-push-уведомления)
7. [Перевод баллов через Mini App](#7-перевод-баллов)
8. [Поддержка (Support)](#8-поддержка)
9. [Настройки уведомлений](#9-настройки-уведомлений)
10. [Multi-bot (Premium)](#10-multi-bot)
11. [i18n](#11-i18n)
12. [Rate Limiting](#12-rate-limiting)
13. [Deployment](#13-deployment)
14. [API Endpoints](#14-api-endpoints)
15. [Чеклист P0/P1/P2](#15-чеклист)

---

## 1. Архитектура

### Стек

| Компонент | Технология |
|---|---|
| **Telegram Bot** | NestJS + **Telegraf** (TypeScript, middleware, scenes, webhook) |
| **Mini App** | React + Vite + **Telegram Web App SDK** (`@twa-dev/sdk`) |
| **Shared types** | `packages/shared` — типы и утилиты |
| **Backend API** | REST `/v1/*` (тот же NestJS backend) |
| **Auth Mini App** | `initData` HMAC-SHA256 верификация |

### Монорепо структура

```
apps/
  telegram-bot/src/
    bot/           ← bot instance, scenes, handlers
    services/      ← вызовы backend API
    keyboards/     ← inline / reply keyboards
    middleware/    ← session, rate-limit, i18n
    main.ts
  telegram-miniapp/
    src/
      pages/       ← Card, History, Promos, Transfer, Settings
      hooks/       ← useBalance, usePromos, useHistory
      stores/      ← Zustand (UI state)
      api/         ← axios instance с Bearer token
    vite.config.ts
packages/shared/
  src/
    types/
    utils/
```

### Flow высокого уровня

```
Гость открывает бот
       │
       ▼
  /start [?start=restaurantId]
       │
  Зарегистрирован? ──No──► SMS верификация
       │                         │
      Yes                   User + GuestProfile
       │                    + GuestCard (TELEGRAM)
       ▼
  Главное меню Bot
  ├── 🃏 Карта  → inline-кнопка → открывает Mini App
  ├── 📜 История
  ├── 🎁 Промо
  └── 💬 Поддержка
```

---

## 2. Регистрация и авторизация

### 2.1. /start flow

```
1. Получить update с /start
2. Найти User по telegramId
3. Не найден →
   a. Запросить контакт (RequestContact button)
   b. Получить phone
   c. Отправить SMS-код (Twilio / SMSC)
   d. Принять код → верифицировать
   e. Создать User { telegramId, phone } + GuestProfile + GuestCard
      registrationSource = TELEGRAM
   f. ActivityLog: CARD_CREATED
4. Найден + telegramId уже привязан → приветствие + меню
5. Найден по телефону, telegramId отсутствует →
   a. Привязать telegramId к существующему User
   b. Уведомление об успешной привязке
```

### 2.2. UNIFIED vs SEPARATE

| Режим | Поведение |
|---|---|
| `UNIFIED` | Одна GuestCard на весь тенант, QR и баланс единый |
| `SEPARATE` | Отдельная GuestCard на каждое заведение (`?start=restaurantId`) |

При `SEPARATE`: если гость открывает через deep-link `/start restaurantId`, создаётся (или выбирается) GuestCard именно для этого ресторана.

### 2.3. Отвязка Telegram

```typescript
// Mini App → Settings → «Отвязать Telegram»
await api.delete('/v1/guests/me/telegram');
// Backend:
//   User.telegramId = null
//   User.telegramUsername = null
//   TelegramSession.deleteMany(userId)
//   ActivityLog: UNLINK_TELEGRAM
```

---

## 3. Telegram Bot

### 3.1. Команды

| Команда | Описание |
|---|---|
| `/start` | Регистрация / главное меню |
| `/card` | Показать карту (баланс, уровень, QR, 6-digit) |
| `/history [N]` | Последние N транзакций (default 10) |
| `/promos` | Список доступных промо |
| `/transfer` | Начать перевод баллов |
| `/support` | Создать тикет поддержки |
| `/settings` | Настройки уведомлений |
| `/help` | Справка |

### 3.2. Главное меню (Reply Keyboard)

```
┌──────────────────────────────────────┐
│  🃏 Моя карта    │  📜 История       │
│──────────────────┼───────────────────│
│  🎁 Промо        │  💸 Перевод       │
│──────────────────┼───────────────────│
│  💬 Поддержка    │  ⚙️ Настройки    │
└──────────────────────────────────────┘
```

### 3.3. /card — сообщение с картой

```
🃏 Ваша карта MAX Loyalty

👤 Иван Иванов
🏅 Уровень: Silver  (до Gold: 248 000 ₽)

💰 Баланс: 2 900 баллов
   Regular: 2 500  •  Promo: 400

📅 Промо сгорят: 01.03.2026 (400 баллов)

🔢 Код для кассира: 123456

[📱 Открыть Mini App]   [🔄 Обновить]
```

Knoppka «📱 Открыть Mini App» — `InlineKeyboardButton` с `web_app: { url: MINIAPP_URL }`.

### 3.4. Scenes (Telegraf Scenes)

| Сцена | Шаги |
|---|---|
| `registration` | Запрос контакта → SMS-код → создание пользователя |
| `transfer` | Ввод телефона получателя → ввод суммы → подтверждение → SMS-код → результат |
| `support` | Выбор темы → ввод сообщения → создание тикета → уведомление |

### 3.5. NestJS + Telegraf структура

```typescript
// apps/telegram-bot/src/bot/bot.module.ts
@Module({
  imports: [
    TelegrafModule.forRootAsync({
      useFactory: (config: ConfigService) => ({
        token: config.get('TELEGRAM_BOT_TOKEN'),
        middlewares: [session(), i18n.middleware()],
        launchOptions: {
          webhook: {
            domain: config.get('WEBHOOK_DOMAIN'),
            path: '/api/telegram/webhook',
          }
        }
      })
    }),
    StageModule,   // Scenes
    CardModule,    // /card handlers
    HistoryModule, // /history handlers
    PromoModule,   // /promos handlers
    TransferModule,// /transfer scene
    SupportModule, // /support scene
    NotifModule,   // push sending
  ]
})
export class BotModule {}
```

### 3.6. Идемпотентность сообщений

- При каждом `EARN` или `REDEEM` QR-код и 6-digit код **ротируются** (`BallTransactionService.regenerateCodes`)
- Push в Telegram отправляется после успешного commit транзакции
- Повторная отправка защищена Redis-ключом `telegram:push:{transactionId}` (TTL 24ч)

---

## 4. Mini App

### 4.1. Экраны

| Экран | URL | Описание |
|---|---|---|
| **Card** | `/` | Баланс, уровень, прогресс до след. уровня, QR, 6-digit |
| **History** | `/history` | Список транзакций с фильтрами (EARN/REDEEM/PROMO/EXPIRE) |
| **Promos** | `/promos` | Доступные промо + условия |
| **Transfer** | `/transfer` | Форма перевода баллов |
| **Personal Stats** | `/stats` | Аналитика: визиты, сэкономлено, любимые блюда |
| **Settings** | `/settings` | Профиль, уведомления, отвязка Telegram |

### 4.2. Card Screen (главный экран)

```
┌─────────────────────────────────────┐
│  MAX Loyalty                        │
│  ──────────────────────────────     │
│  Иван Иванов          🏅 Silver    │
│                                     │
│  💰 2 900 баллов                   │
│  Regular 2 500 • Promo 400         │
│                                     │
│  ████████████░░░░  38% → Gold      │
│  до Gold: 248 000 ₽                │
│                                     │
│  📅 Промо сгорят 01.03 (400 балл)  │
│                                     │
│  ┌─────────────┐                   │
│  │  [QR CODE]  │  Код: 1 2 3 4 5 6│
│  └─────────────┘                   │
│                                     │
│  [Показать кассиру]                 │
└─────────────────────────────────────┘
```

### 4.3. History Screen

```typescript
// GET /v1/guests/me/transactions?limit=20&offset=0&type=EARN,REDEEM
const { data } = useQuery(
  ['transactions', filters],
  () => api.get('/v1/guests/me/transactions', { params: filters })
);
```

Фильтры: тип (EARN / REDEEM / PROMO / EXPIRE / MANUAL), период, ресторан.

### 4.4. Transfer Screen (UI flow)

```
1. Ввести телефон получателя (+7...)
2. Ввести сумму (min: LoyaltySystem.minTransferAmount)
3. Превью: «Перевести 500 баллов → Мария +7 999 765-43-21»
4. Кнопка «Подтвердить» → запрос SMS/Telegram-кода
5. Ввести 6-значный код
6. POST /v1/loyalty/transfer → результат
7. Анимация ✅ + обновление баланса
```

### 4.5. Personal Stats Screen

```
GET /v1/analytics/guest/me?period=LAST_90_DAYS

- Всего визитов: 47
- Потрачено: 110 450 ₽
- Средний чек: 2 350 ₽
- Сэкономлено баллами: 8 500 ₽
- Любимый день: Пятница 18:00–20:00
- Прогресс к Gold: 38%
- Использовано промо: 8 (4 200 бонусных баллов)
```

### 4.6. React + Vite конфигурация

```typescript
// apps/telegram-miniapp/src/main.tsx
import WebApp from '@twa-dev/sdk';

WebApp.ready(); // уведомить Telegram, что Mini App загружен
WebApp.expand(); // развернуть на весь экран

// Применить тему Telegram
document.documentElement.style.setProperty(
  '--tg-theme-bg-color',
  WebApp.themeParams.bg_color ?? '#ffffff'
);
```

---

## 5. initData

### 5.1. Верификация на бэкенде

```typescript
// libs/telegram/src/telegram-auth.service.ts
@Injectable()
export class TelegramAuthService {
  verifyInitData(initData: string, botToken: string): TelegramUser {
    const params = new URLSearchParams(initData);
    const hash = params.get('hash');
    params.delete('hash');

    // 1. Собрать data_check_string
    const dataCheckString = Array.from(params.entries())
      .sort(([a], [b]) => a.localeCompare(b))
      .map(([k, v]) => `${k}=${v}`)
      .join('\n');

    // 2. Вычислить secret key
    const secretKey = crypto
      .createHmac('sha256', 'WebAppData')
      .update(botToken)
      .digest();

    // 3. Вычислить hash
    const calculatedHash = crypto
      .createHmac('sha256', secretKey)
      .update(dataCheckString)
      .digest('hex');

    if (calculatedHash !== hash) throw new UnauthorizedException('Invalid initData hash');

    // 4. Проверить auth_date (max 60 секунд)
    const authDate = parseInt(params.get('auth_date') ?? '0', 10);
    if (Date.now() / 1000 - authDate > 60) throw new UnauthorizedException('initData expired');

    return JSON.parse(params.get('user') ?? '{}') as TelegramUser;
  }
}
```

### 5.2. JWT после верификации

```
POST /v1/auth/telegram/miniapp
Body: { initData: "..." }

1. verifyInitData(initData, botToken)
2. Найти User по telegramId
3. Не найден → создать через Bot flow (ошибка 404 с redirect)
4. Вернуть { accessToken, refreshToken, user }

Далее Mini App использует Bearer token для всех запросов.
```

---

## 6. Push-уведомления

### 6.1. Типы уведомлений

| Тип | Триггер | Каналы |
|---|---|---|
| `BALLS_EARNED` | После EARN-транзакции | Telegram |
| `BALLS_REDEEMED` | После REDEEM-транзакции | Telegram |
| `BALLS_EXPIRING_SOON` | CRON 23:00 за 3/7/14 дней | Telegram, Email |
| `BALLS_EXPIRED` | CRON expire | Telegram, Email |
| `LEVEL_UPGRADED` | При апгрейде уровня | Telegram, Email |
| `PROMO_ACTIVATED` | При выдаче промо | Telegram |
| `PROMO_AVAILABLE` | CRON (birthday, inactivity) | Telegram, Email |
| `TRANSFER_RECEIVED` | После перевода | Telegram |

### 6.2. Шаблоны (Telegram)

```
BALLS_EARNED:
  «✅ +285 баллов!
  Чек: 2 850 ₽ в {{restaurant}}
  Баланс: 3 185 баллов»

LEVEL_UPGRADED:
  «🏆 Поздравляем! Вы достигли уровня Gold!
  Теперь вы получаете +10% к каждому начислению.»

BALLS_EXPIRING_SOON:
  «⏰ Через {{days}} дней сгорят {{amount}} баллов.
  Успейте потратить!»
```

### 6.3. GuestProfile.notificationSettings

```typescript
GuestProfile.notificationSettings = {
  notifyTransactions: true,   // EARN, REDEEM
  notifyLevelChange: true,
  notifyPromo: true,
  notifyExpire: true,
  notifyTransfers: true,
  globalMute: false
}
```

### 6.4. Отправка через NotificationService

```typescript
await this.notificationService.send({
  recipientId: guestCard.userId,
  recipientType: 'GUEST',
  type: NotificationType.BALLS_EARNED,
  channels: ['TELEGRAM'],
  priority: 'NORMAL',
  data: { amount: 285, checkAmount: 2850, balance: 3185, restaurant: 'Ресторан №1' }
});
```

Idempotency: Redis-ключ `telegram:push:{transactionId}:BALLS_EARNED` TTL 24ч — предотвращает дублирование при retry.

---

## 7. Перевод баллов

### 7.1. Full Flow

```
Mini App: POST /v1/loyalty/transfer/init
  { fromCardId, toPhone, amount }

1. Проверить allowTransfers = true
2. Проверить cooldown (5 мин с последней операции)
3. Проверить лимиты: minTransferAmount, maxPerOperation, maxPerDay
4. Отправить SMS/Telegram-код получателю
5. Создать BallTransfer { status: PENDING, verificationCode }

Mini App: POST /v1/loyalty/transfer/confirm
  { transferId, verificationCode }

6. Верифицировать код (TTL 5 мин)
7. Prisma $transaction:
   - BallTransaction { type: TRANSFER_SENT,     guestCardId: sender }
   - BallTransaction { type: TRANSFER_RECEIVED, guestCardId: receiver }
   - BallTransfer { status: COMPLETED, completedAt: now }
8. Push обоим участникам
9. ActivityLog: BALLS_TRANSFERRED
```

### 7.2. История переводов в Mini App

```
📤 Исходящие переводы:
  → +7 999 765-43-21  500 баллов  01.02.2026

📥 Входящие переводы:
  ← +7 999 123-45-67  200 баллов  28.01.2026
```

---

## 8. Поддержка

### 8.1. Bot: сцена /support

```
1. Выбор темы: [Начисление] [Списание] [Карта] [Другое]
2. Ввод сообщения (max 1000 символов)
3. POST /v1/support/tickets
   { type, message, guestCardId, channel: 'TELEGRAM' }
4. «✅ Тикет #12345 принят. Ответим в течение 2 часов.»
5. При ответе Admin — push в Telegram
```

### 8.2. SupportTicket model

```typescript
model SupportTicket {
  id          String   @id @default(uuid())
  tenantId    String
  guestCardId String
  channel     String   // TELEGRAM, MINIAPP
  type        String   // EARN, REDEEM, CARD, OTHER
  message     String
  status      String   @default("OPEN") // OPEN, IN_PROGRESS, RESOLVED
  assignedTo  String?  // Admin/Manager userId
  resolution  String?
  createdAt   DateTime @default(now())
  resolvedAt  DateTime?
}
```

---

## 9. Настройки уведомлений

### 9.1. Mini App: Settings Screen

```
GET  /v1/guests/me/notification-settings
PUT  /v1/guests/me/notification-settings

Настройки:
  ✅ Начисление баллов (Telegram)
  ✅ Списание баллов (Telegram)
  ✅ Сгорание баллов (Telegram + Email)
  ✅ Промо-предложения (Telegram)
  ✅ Смена уровня (Telegram + Email)
  ✅ Переводы (Telegram)
  ☐  Маркетинг (выкл. по умолчанию)
  ☐  Глобальное отключение всех уведомлений
```

### 9.2. Bot: /settings

Inline-клавиатура с toggle-кнопками. При нажатии — `PATCH /v1/guests/me/notification-settings`.

---

## 10. Multi-bot

### MVP (A): Один бот на платформу

```
Bot: @maxloyaltybot
/start → базовый flow
?start=restaurantId → SEPARATE mode
```

### Premium/Enterprise (B): Бот для каждого тенанта

```
Owner создаёт бот через BotFather → вводит botToken в Admin Panel.
Backend: TelegramBot { botToken (encrypted), tenantId, restaurantId? }
Webhook: POST /api/telegram/webhook/:botToken
Каждый бот изолирован на уровне tenantId.
```

Модель `TelegramBot` уже есть в Prisma схеме (S-06).

---

## 11. i18n

### Bot

```typescript
// telegraf-i18n
const i18n = new I18n({
  defaultLanguage: 'ru',
  directory: path.resolve(__dirname, 'locales'),
  useSession: true,
});
// Locale из User.languageCode (Telegram API)
```

### Mini App

```typescript
// react-i18next
import i18n from 'i18next';
const lang = window.Telegram.WebApp.initDataUnsafe?.user?.language_code ?? 'ru';
i18n.changeLanguage(lang);
```

Файлы локалей: `locales/ru.json`, `locales/en.json`. MVP — только RU.

---

## 12. Rate Limiting

```typescript
// Middleware (Bot)
const RATE_LIMITS = {
  general:      { max: 10,  window: 60 },  // 10 req/мин
  smsCode:      { max: 3,   window: 300 }, // 3 SMS за 5 мин
  transfer:     { max: 3,   window: 60 },  // 3 переводa/мин
};
// Redis key: telegram:ratelimit:{telegramId}:{action}, TTL = window
```

```typescript
// Mini App: Axios interceptor
axiosInstance.interceptors.response.use(
  r => r,
  err => {
    if (err.response?.status === 429) {
      const retryAfter = err.response.headers['retry-after'];
      toast.error(`Слишком много запросов. Подождите ${retryAfter} сек.`);
    }
    return Promise.reject(err);
  }
);
```

---

## 13. Deployment

### Bot

| Окружение | Метод | Описание |
|---|---|---|
| dev/staging | Long polling | `bot.launch()` |
| production | Webhook | `POST /api/telegram/webhook` |

```typescript
// Webhook setup
await bot.telegram.setWebhook(
  `${process.env.WEBHOOK_DOMAIN}/api/telegram/webhook`,
  { secret_token: process.env.TELEGRAM_WEBHOOK_SECRET }
);
```

### Mini App (CDN)

```yaml
# .github/workflows/miniapp-deploy.yml
build:
  run: npm run build  # → dist/
deploy:
  # Cloudflare Pages / Vercel / S3 + CloudFront
  url: https://miniapp.max-loyalty.com
```

URL прописывается в `InlineKeyboardButton.web_app.url`. Mini App верифицирует `initData` через `POST /v1/auth/telegram/miniapp`.

### Переменные окружения

```env
TELEGRAM_BOT_TOKEN=...
TELEGRAM_WEBHOOK_SECRET=...
WEBHOOK_DOMAIN=https://api.max-loyalty.com
MINIAPP_URL=https://miniapp.max-loyalty.com
```

---

## 14. API Endpoints

### Auth

| Метод | URL | Описание |
|---|---|---|
| `POST` | `/v1/auth/telegram/miniapp` | Авторизация по initData → JWT |
| `DELETE` | `/v1/guests/me/telegram` | Отвязка Telegram |

### Guest (Mini App)

| Метод | URL | Описание |
|---|---|---|
| `GET` | `/v1/guests/me` | Профиль гостя |
| `PUT` | `/v1/guests/me` | Обновить профиль |
| `GET` | `/v1/guests/me/card` | Баланс + уровень + QR + 6-digit |
| `GET` | `/v1/guests/me/transactions` | История (с пагинацией) |
| `GET` | `/v1/loyalty/promos/eligible` | Доступные промо |
| `GET` | `/v1/analytics/guest/me` | Персональная аналитика |

### Notifications

| Метод | URL | Описание |
|---|---|---|
| `GET` | `/v1/guests/me/notifications` | Список уведомлений |
| `PUT` | `/v1/guests/me/notifications/:id/read` | Отметить прочитанным |
| `GET` | `/v1/guests/me/notification-settings` | Настройки |
| `PUT` | `/v1/guests/me/notification-settings` | Обновить настройки |

### Support

| Метод | URL | Описание |
|---|---|---|
| `POST` | `/v1/support/tickets` | Создать тикет |
| `GET` | `/v1/support/tickets` | Список тикетов гостя |

### Transfer

| Метод | URL | Описание |
|---|---|---|
| `POST` | `/v1/loyalty/transfer/init` | Инициировать перевод |
| `POST` | `/v1/loyalty/transfer/confirm` | Подтвердить по коду |
| `GET` | `/v1/loyalty/transfer/history` | История переводов |

### Telegram Webhook

| Метод | URL | Описание |
|---|---|---|
| `POST` | `/api/telegram/webhook` | Webhook от Telegram (MVP) |
| `POST` | `/api/telegram/webhook/:botToken` | Webhook per-bot (Premium) |

---

## 15. Чеклист

### P0 — Critical

- [ ] `/start` flow: регистрация + SMS-верификация + создание GuestCard
- [ ] Главное меню (Reply Keyboard) + `/card` с балансом и 6-digit кодом
- [ ] initData HMAC-SHA256 верификация на бэкенде (TTL 60 сек)
- [ ] JWT после верификации initData → Mini App авторизован
- [ ] Card Screen: баланс, уровень, прогресс, QR, 6-digit
- [ ] Push `BALLS_EARNED` после каждой транзакции (idempotent)
- [ ] Webhook production: HTTPS, secret_token проверка
- [ ] QR/6-digit ротация при каждой транзакции

### P1 — High

- [ ] History Screen: список транзакций с фильтрами
- [ ] Promos Screen: список доступных промо + условия
- [ ] `/support` сцена: создание тикета → push при ответе
- [ ] Settings Screen: управление уведомлениями
- [ ] Transfer сцена (Bot) + Transfer Screen (Mini App)
- [ ] Push: `BALLS_EXPIRING_SOON`, `LEVEL_UPGRADED`, `PROMO_ACTIVATED`
- [ ] UNIFIED vs SEPARATE mode (deep-link `?start=restaurantId`)
- [ ] Привязка существующего аккаунта по телефону

### P2 — Normal

- [ ] Personal Stats Screen (`/v1/analytics/guest/me`)
- [ ] i18n: `en` локаль (MVP — только RU)
- [ ] Multi-bot (Premium): botToken per tenant
- [ ] Геймификация: достижения (Phase 2)
- [ ] `/settings` в боте с inline toggle-кнопками
- [ ] Экспорт истории транзакций из Mini App (PDF/CSV)
- [ ] Push: `globalMute` toggle

---

*S-07 закрывает спецификацию Telegram Bot & Mini App. Следующий: S-08 — POS Integration (iiko / R-Keeper).*
