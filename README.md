# Max Loyalty — SaaS Платформа Программы Лояльности

> Многотенантная B2B SaaS-платформа для ресторанного бизнеса с системой накопления и списания баллов, интеграцией с POS-системами, Telegram Mini App и полным аналитическим модулем.

[![Documentation](https://img.shields.io/badge/docs-comprehensive-blue)](./docs/INDEX.md)
[![Status](https://img.shields.io/badge/status-in%20development-orange)](./docs/15-DEVELOPMENT-PATH.md)
[![Architecture](https://img.shields.io/badge/architecture-modular%20monolith-green)](./docs/17-ARCHITECTURE.md)

---

## 🎯 Что это такое

**Max Loyalty** — это многотенантная платформа программы лояльности, разработанная специально для ресторанного бизнеса. Платформа позволяет сети ресторанов или отдельным заведениям:

- Создавать и настраивать программы лояльности (правила начисления/списания баллов)
- Управлять гостями и их картами лояльности
- Интегрироваться с POS-системами (iiko, R-Keeper, Poster)
- Предоставлять гостям Telegram Mini App для просмотра баланса и промо-акций
- Получать детальную аналитику по бизнесу и программе лояльности
- Управлять командой (Owner → Admin → Manager → Cashier)

---

## 📋 Быстрая навигация

| Раздел | Файл | Описание |
|--------|------|----------|
| 📖 Полный обзор системы | [01-FULL-SYSTEM-OVERVIEW.md](./docs/01-FULL-SYSTEM-OVERVIEW.md) | Всё о системе от начала до конца |
| 🗄️ База данных | [02-DATABASE-SCHEMA.md](./docs/blocks/02-DATABASE-SCHEMA.md) | Prisma-схема, все модели и индексы |
| 🔧 Backend API | [03-BACKEND-API.md](./docs/blocks/03-BACKEND-API.md) | NestJS API, все эндпоинты |
| 🏪 POS-интеграция | [04-POS-INTEGRATION.md](./docs/blocks/04-POS-INTEGRATION.md) | iiko, R-Keeper, webhooks |
| 🖥️ Frontend Admin | [05-FRONTEND-ADMIN.md](./docs/blocks/05-FRONTEND-ADMIN.md) | Next.js 14, Admin Panel |
| 👨‍💼 Manager Dashboard | [06-MANAGER-DASHBOARD.md](./docs/blocks/06-MANAGER-DASHBOARD.md) | Роль менеджера, права доступа |
| 💳 Касса | [07-CASHIER-INTERFACE.md](./docs/blocks/07-CASHIER-INTERFACE.md) | Интерфейс кассира, PIN-auth |
| 📱 Telegram Mini App | [08-TELEGRAM-MINI-APP.md](./docs/blocks/08-TELEGRAM-MINI-APP.md) | Bot + Mini App для гостей |
| 🔔 Уведомления | [09-NOTIFICATIONS.md](./docs/blocks/09-NOTIFICATIONS.md) | Email, SMS, Telegram, Push |
| 📊 Аналитика | [10-ANALYTICS.md](./docs/blocks/10-ANALYTICS.md) | Дашборды, отчёты, экспорт |
| 💰 Биллинг | [11-BILLING-SUBSCRIPTIONS.md](./docs/blocks/11-BILLING-SUBSCRIPTIONS.md) | Подписки, YooKassa/Stripe |
| 🧪 Тестирование | [12-TESTING-STRATEGY.md](./docs/blocks/12-TESTING-STRATEGY.md) | Jest, Supertest, Playwright, k6 |
| 🚀 DevOps | [13-INFRASTRUCTURE-DEVOPS.md](./docs/blocks/13-INFRASTRUCTURE-DEVOPS.md) | Fly.io, Docker, CI/CD |
| 🔐 Безопасность | [14-SECURITY-COMPLIANCE.md](./docs/blocks/14-SECURITY-COMPLIANCE.md) | GDPR, PCI DSS, OWASP |
| 🛣️ Путь разработки | [15-DEVELOPMENT-PATH.md](./docs/15-DEVELOPMENT-PATH.md) | Фазы, недели, приоритеты |
| 📁 Структура проекта | [16-PROJECT-STRUCTURE.md](./docs/16-PROJECT-STRUCTURE.md) | Файловая структура monorepo |
| 🏗️ Архитектура | [17-ARCHITECTURE.md](./docs/17-ARCHITECTURE.md) | ADR, диаграммы, паттерны |

---

## 🚀 Технологический стек

### Backend
- **Runtime:** Node.js 20 LTS
- **Framework:** NestJS 10 (Modular Monolith)
- **ORM:** Prisma 5 + PostgreSQL 15
- **Queue:** BullMQ + Redis
- **Auth:** JWT (access 15min + refresh 30d)

### Frontend
- **Framework:** Next.js 14 (App Router)
- **UI:** shadcn/ui + Radix UI + Tailwind CSS
- **State:** Zustand (client) + TanStack Query (server)
- **Tables:** TanStack Table v8
- **Charts:** Recharts

### Infrastructure
- **API Hosting:** Fly.io (3 VM: api + worker + cron)
- **Frontend:** Vercel
- **Database:** Neon.tech (PostgreSQL, 3 GB free)
- **Cache:** Upstash (Redis, serverless)
- **Storage:** Cloudflare R2 (S3-compatible)
- **CI/CD:** GitHub Actions

### Интеграции
- **POS:** iiko Cloud/Server, R-Keeper, Poster
- **Payments:** YooKassa, Stripe
- **Messenger:** Telegram Bot API + Mini App
- **Email:** Resend
- **SMS:** SMS.RU

---

## 👥 Роли и доступ

```
Owner (Владелец платформы)
  └── RestaurantAdmin (Администратор сети ресторанов)
        └── Manager (Менеджер ресторана)
              └── Cashier (Кассир)
                    └── Guest (Гость / держатель карты)
```

---

## 📦 Структура репозитория

```
m-loyalty/
├── apps/
│   ├── api/          # NestJS Backend
│   ├── worker/       # BullMQ Worker
│   ├── cron/         # Cron Jobs
│   └── frontend/     # Next.js Admin Panel
├── libs/
│   ├── common/       # Shared types, utils, constants
│   ├── prisma/       # Prisma schema + migrations
│   └── sdk/          # Auto-generated TypeScript SDK
├── docs/             # Вся документация (этот раздел)
├── docker/           # Dockerfiles
└── .github/          # CI/CD workflows
```

---

## 📈 Масштабируемость

| Фаза | Гостей | Инфраструктура | Стоимость/мес |
|------|--------|----------------|---------------|
| MVP | 0–500 | Free Tier | $0 |
| First Paying | 500–2k | Fly.io upgrade | ~$30 |
| PMF | 2k–10k | Neon Scale | ~$90 |
| Scale | 10k–50k | Dedicated DB + K8s | ~$300 |

---

## 📝 Лицензия

Проприетарная. Все права защищены © 2026 Max Loyalty.

---

*Документация генерируется последовательно по 107-сессионному плану. Следите за [docs/INDEX.md](./docs/INDEX.md) для актуального прогресса.*
