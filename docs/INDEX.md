# 📚 Max Loyalty — Индекс документации

> Полная навигация по всей технической документации проекта Max Loyalty.
> Каждый файл содержит детальное описание соответствующего блока системы.

---

## 🗺 Структура документации

```
docs/
├── INDEX.md                          ← Этот файл
├── 01-FULL-SYSTEM-OVERVIEW.md        ← Полный обзор всей системы (~420kb)
├── 15-DEVELOPMENT-PATH.md            ← Путь разработки (~400kb)
├── 16-PROJECT-STRUCTURE.md           ← Структура проекта (~400kb)
├── 17-ARCHITECTURE.md                ← Архитектура системы (~420kb)
└── blocks/
    ├── 02-DATABASE-SCHEMA.md         ← База данных и Prisma (~430kb)
    ├── 03-BACKEND-API.md             ← Backend API NestJS (~450kb)
    ├── 04-POS-INTEGRATION.md         ← POS-интеграция iiko/R-Keeper (~400kb)
    ├── 05-FRONTEND-ADMIN.md          ← Frontend Admin Panel (~420kb)
    ├── 06-MANAGER-DASHBOARD.md       ← Manager Dashboard (~400kb)
    ├── 07-CASHIER-INTERFACE.md       ← Cashier Interface (~400kb)
    ├── 08-TELEGRAM-MINI-APP.md       ← Telegram Bot + Mini App (~400kb)
    ├── 09-NOTIFICATIONS.md           ← Система уведомлений (~400kb)
    ├── 10-ANALYTICS.md               ← Аналитика и отчёты (~400kb)
    ├── 11-BILLING-SUBSCRIPTIONS.md   ← Биллинг и подписки (~400kb)
    ├── 12-TESTING-STRATEGY.md        ← Стратегия тестирования (~400kb)
    ├── 13-INFRASTRUCTURE-DEVOPS.md   ← Инфраструктура и DevOps (~450kb)
    └── 14-SECURITY-COMPLIANCE.md     ← Безопасность и compliance (~400kb)
```

---

## 📖 Документы — обзор содержания

### 01. Полный обзор системы
**Файл:** [`01-FULL-SYSTEM-OVERVIEW.md`](./01-FULL-SYSTEM-OVERVIEW.md)

Комплексное описание всей платформы Max Loyalty от бизнес-концепции до технических деталей:
- Бизнес-модель и целевая аудитория
- Глоссарий всех терминов
- Все 5 ролей RBAC с детальным описанием
- Полный user journey каждой роли
- Auth-система (JWT, MFA, refresh rotation)
- Loyalty Engine (правила, уровни, промо, расчёт)
- POS-интеграция (PUSH/PULL, iiko, R-Keeper)
- Billing (6 планов, платёжный flow)
- Analytics (Owner/Admin/Manager/Guest)
- Infrastructure (Fly.io, Neon, Upstash, R2)
- Security (OWASP, GDPR, PCI DSS)

---

### 02. База данных и Prisma Schema
**Файл:** [`blocks/02-DATABASE-SCHEMA.md`](./blocks/02-DATABASE-SCHEMA.md)

Полное описание схемы базы данных:
- Все 20+ enum'ов с объяснением
- 35+ Prisma моделей с полным кодом
- Стратегии индексирования
- Multi-tenancy Row-Level Security
- Миграции и seed-данные
- Производительность и оптимизация

---

### 03. Backend API
**Файл:** [`blocks/03-BACKEND-API.md`](./blocks/03-BACKEND-API.md)

Полная документация NestJS Backend:
- Модульная архитектура (15+ модулей)
- Все 150+ API эндпоинтов с примерами запросов/ответов
- Auth API (login, logout, refresh, 2FA, impersonation)
- Loyalty Engine API (earn, redeem, manual, transfer)
- Loyalty Builder API (rules, levels, promos)
- Background Jobs (BullMQ, 5 очередей, CRON)
- Analytics API, Export API
- Error handling, Idempotency, Testing

---

### 04. POS-интеграция
**Файл:** [`blocks/04-POS-INTEGRATION.md`](./blocks/04-POS-INTEGRATION.md)

Детальная документация интеграций с POS-системами:
- Архитектура PUSH vs PULL
- iiko: webhook flow, XML API, схема данных
- R-Keeper: WPF/C# DLL plugin, FarCard interface
- Reconciliation алгоритм
- Offline-режим и edge cases

---

### 05. Frontend Admin Panel
**Файл:** [`blocks/05-FRONTEND-ADMIN.md`](./blocks/05-FRONTEND-ADMIN.md)

Документация Owner + Admin Dashboard:
- Tech stack: Next.js 14, shadcn/ui, Zustand, TanStack Query
- Owner Dashboard (MRR, ARR, Churn, Tenants)
- Admin Dashboard (KPIs, Charts, SSE real-time)
- Guests Management (таблица, фильтры, Bulk Actions)
- Guest Profile (6 вкладок)
- Loyalty Builder (Rules, Levels, Promos)
- Analytics Page, Settings, Team Management

---

### 06. Manager Dashboard
**Файл:** [`blocks/06-MANAGER-DASHBOARD.md`](./blocks/06-MANAGER-DASHBOARD.md)

Специфика интерфейса Менеджера:
- Матрица permissions Manager vs Admin
- Manager-специфичные ограничения
- Approval Workflow (PENDING_APPROVAL)
- Ограниченный доступ к настройкам

---

### 07. Cashier Interface
**Файл:** [`blocks/07-CASHIER-INTERFACE.md`](./blocks/07-CASHIER-INTERFACE.md)

Отдельное standalone-приложение для кассиров:
- PIN-аутентификация (4 цифры, numpad)
- Поиск гостя (телефон/QR/6-digit код)
- Earn/Redeem с Live Preview
- Offline-режим с IndexedDB
- Thermal печать чека

---

### 08. Telegram Bot + Mini App
**Файл:** [`blocks/08-TELEGRAM-MINI-APP.md`](./blocks/08-TELEGRAM-MINI-APP.md)

Телеграм-компонент платформы:
- Telegram Bot (grammy/telegraf)
- Mini App (TWA) — React + Telegram Web App API
- Регистрация через OTP/SMS
- Баланс, QR-код, история транзакций
- Промо-акции, передача баллов
- Уведомления через бот

---

### 09. Система уведомлений
**Файл:** [`blocks/09-NOTIFICATIONS.md`](./blocks/09-NOTIFICATIONS.md)

Централизованный notification сервис:
- Каналы: Telegram, Email (Resend), SMS (SMS.RU), Push
- Шаблоны всех типов уведомлений
- BullMQ очередь, rate limiting
- Quiet hours, opt-in/opt-out
- Delivery tracking, fallback strategy

---

### 10. Аналитика и отчёты
**Файл:** [`blocks/10-ANALYTICS.md`](./blocks/10-ANALYTICS.md)

Аналитическая подсистема по всем ролям:
- Owner: MRR, ARR, Churn, Tenant Health Score
- Admin: Revenue, Guests, RFM, Cohort Retention
- Loyalty ROI: Sankey diagram, Redemption Rate
- Manager: ограниченная аналитика
- Pre-calculation, кэш, производительность

---

### 11. Биллинг и подписки
**Файл:** [`blocks/11-BILLING-SUBSCRIPTIONS.md`](./blocks/11-BILLING-SUBSCRIPTIONS.md)

Система монетизации платформы:
- 6 тарифных планов с лимитами
- Trial period, upgrade/downgrade flow
- YooKassa / Stripe / CloudPayments интеграция
- Invoicing, PDF-инвойсы
- Grace period, dunning, auto-downgrade
- Cancellation, refund, chargeback

---

### 12. Стратегия тестирования
**Файл:** [`blocks/12-TESTING-STRATEGY.md`](./blocks/12-TESTING-STRATEGY.md)

Комплексная стратегия тестирования:
- Unit тесты (Jest) — Loyalty Engine
- Integration тесты (Supertest) — API
- E2E тесты (Playwright) — критические сценарии
- Load тесты (k6) — 500 RPS
- Security тесты, Coverage targets

---

### 13. Инфраструктура и DevOps
**Файл:** [`blocks/13-INFRASTRUCTURE-DEVOPS.md`](./blocks/13-INFRASTRUCTURE-DEVOPS.md)

Полная DevOps-документация:
- Zero-cost MVP stack
- Docker (multi-stage Dockerfile, docker-compose.dev)
- Fly.io деплой (API/Worker/Cron)
- GitHub Actions (5 workflow)
- PostgreSQL + Redis конфиги
- Monitoring (Better Stack, Grafana, OpenTelemetry)
- Backup/Recovery, Scaling Roadmap

---

### 14. Безопасность и Compliance
**Файл:** [`blocks/14-SECURITY-COMPLIANCE.md`](./blocks/14-SECURITY-COMPLIANCE.md)

51 security control:
- OWASP Top 10 coverage
- Auth & Session security
- Data Encryption (AES-256-GCM, TLS 1.3)
- GDPR compliance
- PCI DSS SAQ A
- Monitoring, Incidents, Pentest

---

### 15. Путь разработки
**Файл:** [`15-DEVELOPMENT-PATH.md`](./15-DEVELOPMENT-PATH.md)

Роадмап разработки от нуля до Production:
- Фаза 1: Database + Backend core (нед. 1–4)
- Фаза 2: POS + Frontend Admin (нед. 5–10)
- Фаза 3: Telegram + Billing + Notifications (нед. 11–16)
- Фаза 4: Analytics + Testing + Security (нед. 17–22)
- Фаза 5: DevOps + Production (нед. 23–26)

---

### 16. Структура проекта
**Файл:** [`16-PROJECT-STRUCTURE.md`](./16-PROJECT-STRUCTURE.md)

Полная файловая структура monorepo:
- apps/ (api, worker, cron, frontend)
- libs/ (shared, database, telegram)
- packages/ (pos-plugin C#)
- Config файлы, ENV переменные
- Все зависимости с версиями

---

### 17. Архитектура
**Файл:** [`17-ARCHITECTURE.md`](./17-ARCHITECTURE.md)

Архитектурная документация:
- ADR (Architectural Decision Records)
- System Context Diagram (Mermaid)
- Component Diagram
- Data Flow Diagrams
- Database ER Diagram (Mermaid)
- Deployment Architecture
- Sequence Diagrams

---

## 📊 Статус документации

| Файл | Статус | Сессий | Размер |
|------|--------|--------|--------|
| README.md | ✅ Готов | S-00 | ~8kb |
| INDEX.md | ✅ Готов | S-00 | ~6kb |
| 01-FULL-SYSTEM-OVERVIEW.md | 🔄 В процессе | S-01…S-07 | ~420kb |
| blocks/02-DATABASE-SCHEMA.md | ⏳ Ожидает | S-08…S-14 | ~430kb |
| blocks/03-BACKEND-API.md | ⏳ Ожидает | S-15…S-22 | ~450kb |
| blocks/04-POS-INTEGRATION.md | ⏳ Ожидает | S-23…S-28 | ~400kb |
| blocks/05-FRONTEND-ADMIN.md | ⏳ Ожидает | S-29…S-35 | ~420kb |
| blocks/06-MANAGER-DASHBOARD.md | ⏳ Ожидает | S-36…S-41 | ~400kb |
| blocks/07-CASHIER-INTERFACE.md | ⏳ Ожидает | S-42…S-47 | ~400kb |
| blocks/08-TELEGRAM-MINI-APP.md | ⏳ Ожидает | S-48…S-53 | ~400kb |
| blocks/09-NOTIFICATIONS.md | ⏳ Ожидает | S-54…S-59 | ~400kb |
| blocks/10-ANALYTICS.md | ⏳ Ожидает | S-60…S-65 | ~400kb |
| blocks/11-BILLING-SUBSCRIPTIONS.md | ⏳ Ожидает | S-66…S-71 | ~400kb |
| blocks/12-TESTING-STRATEGY.md | ⏳ Ожидает | S-72…S-77 | ~400kb |
| blocks/13-INFRASTRUCTURE-DEVOPS.md | ⏳ Ожидает | S-78…S-84 | ~450kb |
| blocks/14-SECURITY-COMPLIANCE.md | ⏳ Ожидает | S-85…S-90 | ~400kb |
| 15-DEVELOPMENT-PATH.md | ⏳ Ожидает | S-91…S-96 | ~400kb |
| 16-PROJECT-STRUCTURE.md | ⏳ Ожидает | S-97…S-101 | ~400kb |
| 17-ARCHITECTURE.md | ⏳ Ожидает | S-102…S-107 | ~420kb |

---

## 🔄 Легенда статусов

| Статус | Значение |
|--------|-----------|
| ✅ Готов | Файл полностью создан |
| 🔄 В процессе | Файл создаётся (текущая сессия) |
| ⏳ Ожидает | Запланировано на следующие сессии |

---

*Последнее обновление: S-00 (Setup)*
