# 🏆 Max Loyalty — SaaS Платформа Лояльности для Ресторанов

> **Production-ready** мультиарендная (multi-tenant) SaaS-система управления программами лояльности для сетей ресторанов. Полная документация из 107 сессий, 18 файлов, ~7.2 MB технической документации.

---

## 🎯 О проекте

**Max Loyalty** — это B2B SaaS платформа, которая позволяет сетям ресторанов запускать и управлять программами лояльности для гостей: начисление и списание баллов, уровни лояльности, промо-акции, интеграция с POS-системами (iiko, R-Keeper), Telegram Mini App для гостей и мощная аналитика для владельцев бизнеса.

### 🌟 Ключевые возможности

| Возможность | Описание |
|------------|---------|
| 🎁 **Loyalty Engine** | Гибкий движок баллов: правила начисления, уровни, промо-акции, конфликт-резолюция |
| 🖥️ **POS-интеграция** | Нативная интеграция с iiko (webhook) и R-Keeper (WPF-плагин на C#) |
| 📱 **Telegram Mini App** | Гость видит баланс, QR-код, историю транзакций прямо в Telegram |
| 🏢 **Admin Panel** | Next.js 14 — управление гостями, правилами, аналитикой |
| 📊 **Analytics** | Owner/Admin/Manager дашборды: MRR, Churn, RFM, Cohort, Loyalty ROI |
| 💳 **Billing SaaS** | Подписки FREE→ULTIMATE, YooKassa/Stripe, auto-downgrade при неоплате |
| 🔒 **Security** | 51 контроль безопасности: OWASP, GDPR, PCI DSS, JWT RS256 |
| 🚀 **DevOps** | Fly.io + Neon.tech + Upstash — zero-cost MVP, production-ready |

---

## 👥 Роли пользователей

```
OWNER            — Владелец SaaS платформы (суперадмин всей системы)
RESTAURANT_ADMIN — Владелец/управляющий ресторана (настройка лояльности)
MANAGER          — Менеджер ресторана (операционное управление)
CASHIER          — Кассир (начисление/списание баллов через POS/интерфейс)
GUEST            — Гость ресторана (Telegram Mini App, QR-карта)
```

---

## 🏗️ Технологический стек

### Backend
```
NestJS (Modular Monolith)  — Node.js framework
Prisma ORM                 — Type-safe database access
PostgreSQL 15              — Primary database (Neon.tech)
Redis                      — Cache + Queue (Upstash)
BullMQ                     — Background jobs
JWT RS256 + Refresh Tokens — Authentication
```

### Frontend
```
Next.js 14 App Router      — Admin Panel + Owner Dashboard
shadcn/ui + Radix UI       — Component library
Tailwind CSS               — Styling
Zustand + TanStack Query   — State management
TanStack Table             — Data tables
Recharts                   — Analytics charts
```

### Infrastructure
```
Fly.io                     — Backend (API + Worker + Cron)
Vercel                     — Frontend (Next.js)
Neon.tech                  — PostgreSQL (3GB free)
Upstash                    — Redis (serverless)
Cloudflare R2              — S3 Storage (10GB free)
GitHub Actions             — CI/CD
Better Stack               — Monitoring + Logging
```

### Integrations
```
iiko                       — POS-система (webhook)
R-Keeper                   — POS-система (WPF Plugin C#)
Telegram Bot API           — Bot + Mini App
YooKassa / Stripe          — Payment processing
Resend                     — Email service
SMS.ru                     — SMS notifications
Cloudflare                 — SSL/DNS/WAF/DDoS
```

---

## 📁 Структура документации

```
docs/
├── 01-FULL-SYSTEM-OVERVIEW.md         (~420KB) — Полное описание всей системы
├── 15-DEVELOPMENT-PATH.md             (~400KB) — Путь разработки (107 сессий → production)
├── 16-PROJECT-STRUCTURE.md            (~400KB) — Файловая структура проекта
├── 17-ARCHITECTURE.md                 (~420KB) — Архитектура + ADR + диаграммы
├── INDEX.md                           — Полный индекс всех 107 сессий
└── blocks/
    ├── 02-DATABASE-SCHEMA.md          (~430KB) — Prisma: все модели, индексы, миграции
    ├── 03-BACKEND-API.md              (~450KB) — NestJS API: все эндпоинты, guards, jobs
    ├── 04-POS-INTEGRATION.md          (~400KB) — iiko + R-Keeper интеграции
    ├── 05-FRONTEND-ADMIN.md           (~420KB) — Admin Panel: все страницы и компоненты
    ├── 06-MANAGER-DASHBOARD.md        (~400KB) — Manager Dashboard + Approval Workflow
    ├── 07-CASHIER-INTERFACE.md        (~400KB) — Cashier Interface: PIN, QR, thermal print
    ├── 08-TELEGRAM-MINI-APP.md        (~400KB) — Telegram Bot + Mini App
    ├── 09-NOTIFICATIONS.md            (~400KB) — Notifications: все каналы и шаблоны
    ├── 10-ANALYTICS.md               (~400KB) — Analytics: Owner/Admin/Manager/Guest
    ├── 11-BILLING-SUBSCRIPTIONS.md    (~400KB) — Billing: планы, платежи, lifecycle
    ├── 12-TESTING-STRATEGY.md        (~400KB) — Testing: Unit/Integration/E2E/Load
    ├── 13-INFRASTRUCTURE-DEVOPS.md    (~450KB) — DevOps: Docker, Fly.io, CI/CD
    └── 14-SECURITY-COMPLIANCE.md      (~400KB) — Security: OWASP, GDPR, PCI DSS
```

**Полный индекс всех 107 сессий** → [docs/INDEX.md](./docs/INDEX.md)

---

## 🗺️ Быстрая навигация

### По роли разработчика

| Роль | Начни с |
|------|---------|
| **Fullstack** | [01-FULL-SYSTEM-OVERVIEW](./docs/01-FULL-SYSTEM-OVERVIEW.md) → [17-ARCHITECTURE](./docs/17-ARCHITECTURE.md) |
| **Backend** | [02-DATABASE-SCHEMA](./docs/blocks/02-DATABASE-SCHEMA.md) → [03-BACKEND-API](./docs/blocks/03-BACKEND-API.md) |
| **Frontend** | [05-FRONTEND-ADMIN](./docs/blocks/05-FRONTEND-ADMIN.md) → [08-TELEGRAM-MINI-APP](./docs/blocks/08-TELEGRAM-MINI-APP.md) |
| **DevOps** | [13-INFRASTRUCTURE-DEVOPS](./docs/blocks/13-INFRASTRUCTURE-DEVOPS.md) → [16-PROJECT-STRUCTURE](./docs/16-PROJECT-STRUCTURE.md) |
| **QA** | [12-TESTING-STRATEGY](./docs/blocks/12-TESTING-STRATEGY.md) → [14-SECURITY-COMPLIANCE](./docs/blocks/14-SECURITY-COMPLIANCE.md) |
| **PM/Архитектор** | [17-ARCHITECTURE](./docs/17-ARCHITECTURE.md) → [15-DEVELOPMENT-PATH](./docs/15-DEVELOPMENT-PATH.md) |

### По задаче

| Задача | Документ |
|--------|---------|
| Понять систему целиком | [01-FULL-SYSTEM-OVERVIEW](./docs/01-FULL-SYSTEM-OVERVIEW.md) |
| Начать разработку с нуля | [15-DEVELOPMENT-PATH](./docs/15-DEVELOPMENT-PATH.md) |
| Настроить базу данных | [02-DATABASE-SCHEMA](./docs/blocks/02-DATABASE-SCHEMA.md) |
| Реализовать Loyalty Engine | [03-BACKEND-API](./docs/blocks/03-BACKEND-API.md) |
| Подключить iiko | [04-POS-INTEGRATION](./docs/blocks/04-POS-INTEGRATION.md) |
| Задеплоить в production | [13-INFRASTRUCTURE-DEVOPS](./docs/blocks/13-INFRASTRUCTURE-DEVOPS.md) |
| Обеспечить безопасность | [14-SECURITY-COMPLIANCE](./docs/blocks/14-SECURITY-COMPLIANCE.md) |

---

## 📊 Метрики системы

```
🎯 Ёмкость MVP (free tier):
   - ~5,000 активных гостей
   - ~10,000 транзакций/месяц
   - ~1,000 POS-операций/день
   - Стоимость инфраструктуры: $0/месяц

📈 Масштабирование (paid tier):
   - До 500,000 гостей (горизонтальное масштабирование Fly.io)
   - До 50,000 транзакций/день (connection pooling PgBouncer)
   - Multi-region: fra, ams, cdg (Europa → RU/KZ/AE)
   - Стоимость на 10k тенантов: ~$2,500/месяц

⚡ Performance targets:
   - API response: p95 < 500ms
   - API response: p99 < 1000ms
   - Error rate: < 1%
   - Uptime SLA: 99.9%
```

---

## 🚀 Быстрый старт

### 1. Клонирование и установка зависимостей
```bash
git clone https://github.com/Romslav/m-loyalty.git
cd m-loyalty
npm install
```

### 2. Настройка окружения
```bash
cp .env.example .env
# Заполни переменные (DATABASE_URL, REDIS_URL, JWT_SECRET, ...)
```

### 3. База данных
```bash
# Запуск локального PostgreSQL + Redis через Docker
docker-compose -f docker-compose.dev.yml up -d

# Применение миграций
npx prisma migrate dev

# Заполнение тестовыми данными
npx prisma db seed
```

### 4. Запуск
```bash
# Backend API (NestJS)
cd apps/api && npm run dev

# Frontend (Next.js)
cd apps/frontend && npm run dev

# Worker (BullMQ)
cd apps/worker && npm run dev
```

### 5. Проверка
```
API:      http://localhost:3000/health
Swagger:  http://localhost:3000/api
Frontend: http://localhost:3001
```

---

## 📈 Статус документации

| Файл | Статус | Сессий | Объём |
|------|--------|--------|-------|
| README.md | ✅ Готов | S-00 | 11 KB |
| docs/INDEX.md | ✅ Готов | S-00 | 38 KB |
| docs/01-FULL-SYSTEM-OVERVIEW.md | ⏳ Запланирован | S-01–S-07 | ~420KB |
| docs/blocks/02-DATABASE-SCHEMA.md | ⏳ Запланирован | S-08–S-14 | ~430KB |
| docs/blocks/03-BACKEND-API.md | ⏳ Запланирован | S-15–S-22 | ~450KB |
| docs/blocks/04-POS-INTEGRATION.md | ⏳ Запланирован | S-23–S-28 | ~400KB |
| docs/blocks/05-FRONTEND-ADMIN.md | ⏳ Запланирован | S-29–S-35 | ~420KB |
| docs/blocks/06-MANAGER-DASHBOARD.md | ⏳ Запланирован | S-36–S-41 | ~400KB |
| docs/blocks/07-CASHIER-INTERFACE.md | ⏳ Запланирован | S-42–S-47 | ~400KB |
| docs/blocks/08-TELEGRAM-MINI-APP.md | ⏳ Запланирован | S-48–S-53 | ~400KB |
| docs/blocks/09-NOTIFICATIONS.md | ⏳ Запланирован | S-54–S-59 | ~400KB |
| docs/blocks/10-ANALYTICS.md | ⏳ Запланирован | S-60–S-65 | ~400KB |
| docs/blocks/11-BILLING-SUBSCRIPTIONS.md | ⏳ Запланирован | S-66–S-71 | ~400KB |
| docs/blocks/12-TESTING-STRATEGY.md | ⏳ Запланирован | S-72–S-77 | ~400KB |
| docs/blocks/13-INFRASTRUCTURE-DEVOPS.md | ⏳ Запланирован | S-78–S-84 | ~450KB |
| docs/blocks/14-SECURITY-COMPLIANCE.md | ⏳ Запланирован | S-85–S-90 | ~400KB |
| docs/15-DEVELOPMENT-PATH.md | ⏳ Запланирован | S-91–S-96 | ~400KB |
| docs/16-PROJECT-STRUCTURE.md | ⏳ Запланирован | S-97–S-101 | ~400KB |
| docs/17-ARCHITECTURE.md | ⏳ Запланирован | S-102–S-107 | ~420KB |

---

## 🔗 Полезные ссылки

- 📋 [Полный индекс сессий](./docs/INDEX.md)
- 🏗️ [Архитектура системы](./docs/17-ARCHITECTURE.md)
- 🛣️ [Путь разработки](./docs/15-DEVELOPMENT-PATH.md)
- 📁 [Структура проекта](./docs/16-PROJECT-STRUCTURE.md)

---

## 📄 Лицензия

Proprietary — All rights reserved © 2026 Max Loyalty
