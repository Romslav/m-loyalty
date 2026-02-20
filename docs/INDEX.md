# Документация Max Loyalty — Главный Индекс

> Полная техническая документация платформы Max Loyalty.  
> Всего: **17 файлов**, ~7.2 MB документации, 107 сессий генерации.

---

## 📊 Прогресс документации

| # | Файл | Статус | Сессии | Объём |
|---|------|--------|--------|-------|
| 01 | [01-FULL-SYSTEM-OVERVIEW.md](./01-FULL-SYSTEM-OVERVIEW.md) | 🔄 В работе | S-01…S-07 | ~420kb |
| 02 | [blocks/02-DATABASE-SCHEMA.md](./blocks/02-DATABASE-SCHEMA.md) | 🔄 В работе | S-08…S-14 | ~430kb |
| 03 | [blocks/03-BACKEND-API.md](./blocks/03-BACKEND-API.md) | ⏳ Ожидает | S-15…S-22 | ~450kb |
| 04 | [blocks/04-POS-INTEGRATION.md](./blocks/04-POS-INTEGRATION.md) | ⏳ Ожидает | S-23…S-28 | ~400kb |
| 05 | [blocks/05-FRONTEND-ADMIN.md](./blocks/05-FRONTEND-ADMIN.md) | ⏳ Ожидает | S-29…S-35 | ~420kb |
| 06 | [blocks/06-MANAGER-DASHBOARD.md](./blocks/06-MANAGER-DASHBOARD.md) | ⏳ Ожидает | S-36…S-41 | ~400kb |
| 07 | [blocks/07-CASHIER-INTERFACE.md](./blocks/07-CASHIER-INTERFACE.md) | ⏳ Ожидает | S-42…S-47 | ~400kb |
| 08 | [blocks/08-TELEGRAM-MINI-APP.md](./blocks/08-TELEGRAM-MINI-APP.md) | ⏳ Ожидает | S-48…S-53 | ~400kb |
| 09 | [blocks/09-NOTIFICATIONS.md](./blocks/09-NOTIFICATIONS.md) | ⏳ Ожидает | S-54…S-59 | ~400kb |
| 10 | [blocks/10-ANALYTICS.md](./blocks/10-ANALYTICS.md) | ⏳ Ожидает | S-60…S-65 | ~400kb |
| 11 | [blocks/11-BILLING-SUBSCRIPTIONS.md](./blocks/11-BILLING-SUBSCRIPTIONS.md) | ⏳ Ожидает | S-66…S-71 | ~400kb |
| 12 | [blocks/12-TESTING-STRATEGY.md](./blocks/12-TESTING-STRATEGY.md) | ⏳ Ожидает | S-72…S-77 | ~400kb |
| 13 | [blocks/13-INFRASTRUCTURE-DEVOPS.md](./blocks/13-INFRASTRUCTURE-DEVOPS.md) | ⏳ Ожидает | S-78…S-84 | ~450kb |
| 14 | [blocks/14-SECURITY-COMPLIANCE.md](./blocks/14-SECURITY-COMPLIANCE.md) | ⏳ Ожидает | S-85…S-90 | ~400kb |
| 15 | [15-DEVELOPMENT-PATH.md](./15-DEVELOPMENT-PATH.md) | ⏳ Ожидает | S-91…S-96 | ~400kb |
| 16 | [16-PROJECT-STRUCTURE.md](./16-PROJECT-STRUCTURE.md) | ⏳ Ожидает | S-97…S-101 | ~400kb |
| 17 | [17-ARCHITECTURE.md](./17-ARCHITECTURE.md) | ⏳ Ожидает | S-102…S-107 | ~420kb |

**Легенда:** ✅ Готово | 🔄 В работе | ⏳ Ожидает

---

## 🗺️ Навигация по темам

### Если ты разработчик Backend
1. [17-ARCHITECTURE.md](./17-ARCHITECTURE.md) — понять общую архитектуру
2. [blocks/02-DATABASE-SCHEMA.md](./blocks/02-DATABASE-SCHEMA.md) — изучить схему БД
3. [blocks/03-BACKEND-API.md](./blocks/03-BACKEND-API.md) — все API эндпоинты
4. [blocks/04-POS-INTEGRATION.md](./blocks/04-POS-INTEGRATION.md) — интеграция с POS
5. [blocks/13-INFRASTRUCTURE-DEVOPS.md](./blocks/13-INFRASTRUCTURE-DEVOPS.md) — деплой

### Если ты разработчик Frontend
1. [blocks/05-FRONTEND-ADMIN.md](./blocks/05-FRONTEND-ADMIN.md) — Admin Panel
2. [blocks/06-MANAGER-DASHBOARD.md](./blocks/06-MANAGER-DASHBOARD.md) — Manager UI
3. [blocks/07-CASHIER-INTERFACE.md](./blocks/07-CASHIER-INTERFACE.md) — Cashier UI
4. [blocks/08-TELEGRAM-MINI-APP.md](./blocks/08-TELEGRAM-MINI-APP.md) — Telegram App
5. [16-PROJECT-STRUCTURE.md](./16-PROJECT-STRUCTURE.md) — структура файлов

### Если ты DevOps
1. [blocks/13-INFRASTRUCTURE-DEVOPS.md](./blocks/13-INFRASTRUCTURE-DEVOPS.md) — вся инфраструктура
2. [blocks/14-SECURITY-COMPLIANCE.md](./blocks/14-SECURITY-COMPLIANCE.md) — безопасность
3. [blocks/12-TESTING-STRATEGY.md](./blocks/12-TESTING-STRATEGY.md) — CI/CD тесты
4. [17-ARCHITECTURE.md](./17-ARCHITECTURE.md) — архитектурные решения

### Если ты бизнес-аналитик / PM
1. [01-FULL-SYSTEM-OVERVIEW.md](./01-FULL-SYSTEM-OVERVIEW.md) — полный обзор
2. [15-DEVELOPMENT-PATH.md](./15-DEVELOPMENT-PATH.md) — план разработки
3. [blocks/10-ANALYTICS.md](./blocks/10-ANALYTICS.md) — аналитика и отчёты
4. [blocks/11-BILLING-SUBSCRIPTIONS.md](./blocks/11-BILLING-SUBSCRIPTIONS.md) — монетизация

---

## 📂 Файловая структура документации

```
docs/
├── INDEX.md                           ← Вы здесь
├── 01-FULL-SYSTEM-OVERVIEW.md         # Полный обзор системы (~420kb)
├── 15-DEVELOPMENT-PATH.md             # Путь разработки (~400kb)
├── 16-PROJECT-STRUCTURE.md            # Структура проекта (~400kb)
├── 17-ARCHITECTURE.md                 # Архитектура (~420kb)
└── blocks/
    ├── 02-DATABASE-SCHEMA.md          # БД схема (~430kb)
    ├── 03-BACKEND-API.md              # Backend API (~450kb)
    ├── 04-POS-INTEGRATION.md          # POS интеграция (~400kb)
    ├── 05-FRONTEND-ADMIN.md           # Frontend Admin (~420kb)
    ├── 06-MANAGER-DASHBOARD.md        # Manager Dashboard (~400kb)
    ├── 07-CASHIER-INTERFACE.md        # Cashier Interface (~400kb)
    ├── 08-TELEGRAM-MINI-APP.md        # Telegram Mini App (~400kb)
    ├── 09-NOTIFICATIONS.md            # Уведомления (~400kb)
    ├── 10-ANALYTICS.md               # Аналитика (~400kb)
    ├── 11-BILLING-SUBSCRIPTIONS.md    # Биллинг (~400kb)
    ├── 12-TESTING-STRATEGY.md        # Тестирование (~400kb)
    ├── 13-INFRASTRUCTURE-DEVOPS.md    # Инфраструктура (~450kb)
    └── 14-SECURITY-COMPLIANCE.md     # Безопасность (~400kb)
```

---

## 🔑 Ключевые концепции и термины

| Термин | Описание |
|--------|----------|
| **Tenant** | Арендатор — компания (сеть ресторанов), использующая платформу |
| **GuestCard** | Карта лояльности гостя в конкретном ресторане/сети |
| **BallTransaction** | Транзакция начисления или списания баллов |
| **LoyaltyRule** | Правило начисления баллов (% от чека, категория блюда, время и т.д.) |
| **LoyaltyLevel** | Уровень гостя (Бронза, Серебро, Золото, Платина) |
| **LoyaltyPromo** | Промо-акция (день рождения, первый чек, неактивность) |
| **POSTransaction** | Транзакция из кассовой системы |
| **Impersonation** | Вход под другим пользователем (Owner → Admin) для поддержки |
| **RegularBalance** | Обычные баллы, начисленные за покупки |
| **PromoBalance** | Промо-баллы с ограниченным сроком действия |
| **Reconciliation** | Сверка баланса баллов между нашей системой и POS |

---

## 🛠️ Как работаем над документацией

Документация создаётся по **107-сессионному плану**:
- Каждая сессия = один чёткий фрагмент файла
- Минимальный объём каждого файла: **400 kb**
- Формат: строгий Markdown с кодом, таблицами, схемами (Mermaid)
- Язык: русский (технические термины и код — на английском)

---

*Последнее обновление: 2026-02-20. Сессия: S-00 (Setup).*
