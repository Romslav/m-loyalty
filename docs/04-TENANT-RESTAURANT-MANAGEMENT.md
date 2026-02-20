# S-04: Tenant & Restaurant Management

> **Часть 4 из 7** | Зависимости: S-01, S-02, S-03  
> Охват: multi-tenancy, Owner Registration, Subscription Plans, TenantLimits enforcement, Restaurant CRUD, OwnerRegistrationLink

---

## Содержание

1. [Обзор архитектуры multi-tenancy](#1-обзор-архитектуры-multi-tenancy)
2. [Owner Registration Flow](#2-owner-registration-flow)
3. [Subscription Plans & Pricing](#3-subscription-plans--pricing)
4. [Tenant Service](#4-tenant-service)
5. [TenantLimits Enforcement](#5-tenantlimits-enforcement)
6. [Restaurant Management](#6-restaurant-management)
7. [Prisma Schema](#7-prisma-schema)
8. [API Endpoints](#8-api-endpoints)
9. [Guards & Middleware](#9-guards--middleware)
10. [Тесты & Чеклист](#10-тесты--чеклист)

---

## 1. Обзор архитектуры multi-tenancy

### 1.1. Стратегия изоляции

Используется **Pooled Multi-Tenancy** — единая база данных, изоляция через `tenantId` в каждой строке.

```
┌─────────────────────────────────────────────────────────────┐
│                    MAX-LOYALTY PLATFORM                     │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Tenant A   │  │  Tenant B   │  │     Tenant C        │ │
│  │ "Кофейня X" │  │ "Суши-бар Y"│  │  "Сеть ресторанов Z"│ │
│  │             │  │             │  │                     │ │
│  │ 1 ресторан  │  │ 1 ресторан  │  │ 5 ресторанов        │ │
│  │ 500 гостей  │  │ 2000 гостей │  │ 6000 гостей         │ │
│  │ STANDARD    │  │ MEDIUM      │  │ PRO                 │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│                                                             │
│  PostgreSQL: tenantId фильтр во всех запросах               │
│  Redis: ключи с префиксом tenant:{id}:*                     │
│  S3: папки tenant/{id}/                                     │
└─────────────────────────────────────────────────────────────┘
```

### 1.2. Tenant Middleware

Каждый запрос проходит через `TenantMiddleware`, который извлекает `tenantId` из JWT и прикрепляет к `request`:

```typescript
// src/common/middleware/tenant.middleware.ts
import { Injectable, NestMiddleware, ForbiddenException } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class TenantMiddleware implements NestMiddleware {
  constructor(private readonly jwtService: JwtService) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const token = this.extractToken(req);
    if (!token) return next(); // публичные маршруты

    try {
      const payload = this.jwtService.verify(token);
      req['tenantId'] = payload.tenantId;
      req['userId'] = payload.sub;
      req['userRole'] = payload.role;
    } catch {
      // невалидный токен — пропустим, JwtAuthGuard разберётся
    }

    next();
  }

  private extractToken(req: Request): string | null {
    const authHeader = req.headers.authorization;
    if (authHeader?.startsWith('Bearer ')) {
      return authHeader.substring(7);
    }
    return null;
  }
}
```

### 1.3. Prisma Middleware — Tenant Isolation

```typescript
// src/database/prisma-tenant.middleware.ts

// Модели, требующие фильтрации по tenantId
const TENANT_ISOLATED_MODELS = [
  'Restaurant', 'GuestCard', 'BallTransaction', 'GuestVisit',
  'LoyaltyRule', 'LoyaltyLevel', 'LoyaltyPromo', 'POSIntegration',
  'POSTransaction', 'Notification', 'ActivityLog', 'AnalyticsDailySnapshot',
  'AnalyticsMonthlySnapshot', 'PromoBallGranted',
];

export function createTenantMiddleware(tenantId: string) {
  return async (params: any, next: (params: any) => Promise<any>) => {
    if (!TENANT_ISOLATED_MODELS.includes(params.model)) {
      return next(params);
    }

    // Инъекция tenantId в WHERE для read-операций
    if (['findMany', 'findFirst', 'count', 'aggregate'].includes(params.action)) {
      params.args.where = {
        ...params.args.where,
        tenantId,
      };
    }

    // Инъекция tenantId при создании
    if (params.action === 'create') {
      params.args.data = {
        ...params.args.data,
        tenantId,
      };
    }

    // Защита при update/delete — всегда добавляем tenantId в where
    if (['update', 'delete', 'updateMany', 'deleteMany'].includes(params.action)) {
      params.args.where = {
        ...params.args.where,
        tenantId,
      };
    }

    return next(params);
  };
}
```

---

## 2. Owner Registration Flow

### 2.1. OwnerRegistrationLink

Owner регистрируется только по invite-ссылке, созданной суперадмином (OWNER role на платформе).

```typescript
// src/modules/tenants/entities/owner-registration-link.entity.ts
// Prisma model OwnerRegistrationLink:
// id, token (uuid unique), email?, planHint (SubscriptionPlan), 
// usedAt, expiresAt, createdBy (OWNER userId), createdAt
```

```typescript
// src/modules/tenants/services/owner-registration.service.ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../../../database/prisma.service';
import { SubscriptionService } from '../../subscriptions/subscription.service';
import { HashService } from '../../auth/hash.service';
import { v4 as uuidv4 } from 'uuid';
import { addDays } from 'date-fns';

@Injectable()
export class OwnerRegistrationService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly subscriptionService: SubscriptionService,
    private readonly hashService: HashService,
  ) {}

  // === Создание invite-ссылки (только суперадмин) ===
  async createRegistrationLink(dto: CreateOwnerLinkDto, createdByUserId: string) {
    const token = uuidv4();
    const link = await this.prisma.ownerRegistrationLink.create({
      data: {
        token,
        email: dto.email ?? null,
        planHint: dto.planHint ?? 'STANDARD',
        expiresAt: addDays(new Date(), dto.expirationDays ?? 30),
        createdById: createdByUserId,
      },
    });

    return {
      linkId: link.id,
      registrationUrl: `https://app.max-loyalty.com/register/owner?token=${token}`,
      expiresAt: link.expiresAt,
    };
  }

  // === Валидация токена ===
  async validateToken(token: string) {
    const link = await this.prisma.ownerRegistrationLink.findUnique({
      where: { token },
    });

    if (!link) throw new NotFoundException('Ссылка не найдена');
    if (link.usedAt) throw new BadRequestException('Ссылка уже использована');
    if (link.expiresAt < new Date()) throw new BadRequestException('Ссылка истекла');

    return link;
  }

  // === Полный flow регистрации Owner ===
  async registerOwner(token: string, dto: RegisterOwnerDto) {
    const link = await this.validateToken(token);

    // Транзакция: User + Tenant + Subscription + TenantLimits
    return this.prisma.$transaction(async (tx) => {
      // 1. Создаём User с ролью OWNER (глобальной)
      const passwordHash = await this.hashService.hash(dto.password);
      const user = await tx.user.create({
        data: {
          email: dto.email,
          phone: dto.phone,
          passwordHash,
          role: 'OWNER',
          emailVerified: false,
          phoneVerified: false,
        },
      });

      // 2. Создаём Tenant
      const slug = this.generateSlug(dto.companyName);
      const tenant = await tx.tenant.create({
        data: {
          name: dto.companyName,
          slug,
          contactEmail: dto.email,
          contactPhone: dto.phone,
          isActive: true,
        },
      });

      // 3. Привязываем User к Tenant как OWNER
      await tx.userTenantRole.create({
        data: {
          userId: user.id,
          tenantId: tenant.id,
          role: 'OWNER',
        },
      });

      // 4. Создаём Subscription (TRIAL 14 дней)
      const plan = link.planHint ?? 'STANDARD';
      const trialEnd = addDays(new Date(), 14);
      const subscription = await tx.subscription.create({
        data: {
          tenantId: tenant.id,
          plan,
          status: 'TRIAL',
          billingPeriod: 'MONTHLY',
          priceMonthly: PLAN_PRICES[plan].monthly,
          priceYearly: PLAN_PRICES[plan].yearly,
          currency: 'RUB',
          trialEndsAt: trialEnd,
          currentPeriodStart: new Date(),
          currentPeriodEnd: trialEnd,
          autoRenew: true,
        },
      });

      // 5. Создаём TenantLimits согласно плану
      const limits = PLAN_LIMITS[plan];
      await tx.tenantLimits.create({
        data: {
          tenantId: tenant.id,
          maxRestaurants: limits.maxRestaurants,
          maxGuests: limits.maxGuests,
          maxPosIntegrations: limits.maxPosIntegrations,
          maxAdminUsers: limits.maxAdminUsers,
          currentRestaurants: 0,
          currentGuests: 0,
          currentPosIntegrations: 0,
          currentAdminUsers: 1, // Owner сам
          warningThresholdPct: 90,
        },
      });

      // 6. Создаём BillingInfo (пустой, заполнит позже)
      await tx.billingInfo.create({
        data: { tenantId: tenant.id },
      });

      // 7. Помечаем ссылку как использованную
      await tx.ownerRegistrationLink.update({
        where: { id: link.id },
        data: { usedAt: new Date() },
      });

      // 8. ActivityLog
      await tx.activityLog.create({
        data: {
          tenantId: tenant.id,
          actorId: user.id,
          action: 'OWNER_REGISTERED',
          entityType: 'TENANT',
          entityId: tenant.id,
          newValue: { tenantName: tenant.name, plan, email: dto.email },
        },
      });

      return { userId: user.id, tenantId: tenant.id, subscriptionId: subscription.id };
    });
  }

  private generateSlug(name: string): string {
    return name
      .toLowerCase()
      .replace(/[^a-z0-9а-яё\s]/gi, '')
      .replace(/\s+/g, '-')
      .substring(0, 50) + '-' + Date.now().toString(36);
  }
}
```

---

## 3. Subscription Plans & Pricing

### 3.1. Тарифные планы

```typescript
// src/modules/subscriptions/constants/plans.constants.ts

export const PLAN_PRICES = {
  FREE: { monthly: 0, yearly: 0 },
  STANDARD: { monthly: 5000, yearly: 48000 },   // 4000/мес × 12
  MEDIUM: { monthly: 15000, yearly: 144000 },   // 12000/мес × 12
  PRO: { monthly: 35000, yearly: 336000 },      // 28000/мес × 12
  ULTIMATE: { monthly: 100000, yearly: null },  // договорная
  CUSTOM: { monthly: null, yearly: null },      // индивидуально
} as const;

export const PLAN_LIMITS = {
  FREE: {
    maxRestaurants: 1,
    maxGuests: null,      // null = безлимит
    maxPosIntegrations: 0,
    maxAdminUsers: 1,
    maxStorageMb: 100,
  },
  STANDARD: {
    maxRestaurants: 1,
    maxGuests: 500,
    maxPosIntegrations: 1,
    maxAdminUsers: 3,
    maxStorageMb: 1024,
  },
  MEDIUM: {
    maxRestaurants: 3,
    maxGuests: 2000,
    maxPosIntegrations: 3,
    maxAdminUsers: 10,
    maxStorageMb: 5120,
  },
  PRO: {
    maxRestaurants: 5,
    maxGuests: 6000,
    maxPosIntegrations: 5,
    maxAdminUsers: 25,
    maxStorageMb: 20480,
  },
  ULTIMATE: {
    maxRestaurants: null,
    maxGuests: null,
    maxPosIntegrations: null,
    maxAdminUsers: null,
    maxStorageMb: null,
  },
  CUSTOM: {
    maxRestaurants: null,
    maxGuests: null,
    maxPosIntegrations: null,
    maxAdminUsers: null,
    maxStorageMb: null,
  },
} as const;

export type PlanName = keyof typeof PLAN_PRICES;
```

### 3.2. Таблица планов

| Параметр | FREE | STANDARD | MEDIUM | PRO | ULTIMATE |
|---|---|---|---|---|---|
| **Цена/мес** | 0 ₽ | 5 000 ₽ | 15 000 ₽ | 35 000 ₽ | 100 000 ₽ |
| **Цена/год** | 0 ₽ | 48 000 ₽ | 144 000 ₽ | 336 000 ₽ | договор |
| **Скидка год** | — | −20% | −20% | −20% | — |
| **Рестораны** | 1 | 1 | 3 | 5 | ∞ |
| **Гости** | ∞ | 500 | 2 000 | 6 000 | ∞ |
| **POS-интеграции** | 0 | 1 | 3 | 5 | ∞ |
| **Admins** | 1 | 3 | 10 | 25 | ∞ |
| **Trial** | нет | 14 дней | 14 дней | 14 дней | 14 дней |

---

## 4. Tenant Service

```typescript
// src/modules/tenants/tenant.service.ts
import {
  Injectable,
  NotFoundException,
  ForbiddenException,
  BadRequestException,
} from '@nestjs/common';
import { PrismaService } from '../../database/prisma.service';
import { CacheService } from '../cache/cache.service';

@Injectable()
export class TenantService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly cache: CacheService,
  ) {}

  // === Получение tenant с кэшированием ===
  async getTenantById(tenantId: string) {
    const cacheKey = `tenant:${tenantId}:info`;
    const cached = await this.cache.get(cacheKey);
    if (cached) return cached;

    const tenant = await this.prisma.tenant.findUnique({
      where: { id: tenantId, deletedAt: null },
      include: {
        subscription: true,
        tenantLimits: true,
      },
    });

    if (!tenant) throw new NotFoundException('Tenant не найден');

    await this.cache.set(cacheKey, tenant, 300); // 5 минут
    return tenant;
  }

  // === Обновление Tenant ===
  async updateTenant(tenantId: string, dto: UpdateTenantDto, actorId: string) {
    const tenant = await this.getTenantById(tenantId);

    const updated = await this.prisma.tenant.update({
      where: { id: tenantId },
      data: {
        name: dto.name ?? tenant.name,
        description: dto.description,
        contactEmail: dto.contactEmail,
        contactPhone: dto.contactPhone,
        websiteUrl: dto.websiteUrl,
        logoUrl: dto.logoUrl,
        primaryColor: dto.primaryColor,
      },
    });

    // Инвалидация кэша
    await this.cache.del(`tenant:${tenantId}:info`);

    await this.prisma.activityLog.create({
      data: {
        tenantId,
        actorId,
        action: 'TENANT_UPDATED',
        entityType: 'TENANT',
        entityId: tenantId,
        oldValue: { name: tenant.name },
        newValue: { name: updated.name },
      },
    });

    return updated;
  }

  // === Soft-delete Tenant (только OWNER-платформы) ===
  async deactivateTenant(tenantId: string, reason: string, actorId: string) {
    await this.prisma.$transaction(async (tx) => {
      await tx.tenant.update({
        where: { id: tenantId },
        data: {
          isActive: false,
          blockedReason: reason,
          deletedAt: new Date(),
        },
      });

      // Закрываем подписку
      await tx.subscription.updateMany({
        where: { tenantId, status: { in: ['ACTIVE', 'TRIAL', 'PAST_DUE'] } },
        data: { status: 'CANCELLED', cancelledAt: new Date() },
      });

      await tx.activityLog.create({
        data: {
          tenantId,
          actorId,
          action: 'TENANT_DEACTIVATED',
          entityType: 'TENANT',
          entityId: tenantId,
          newValue: { reason },
        },
      });
    });

    // Чистим весь кэш tenant'а
    await this.cache.delPattern(`tenant:${tenantId}:*`);
  }

  // === Проверка активности tenant ===
  async assertTenantActive(tenantId: string): Promise<void> {
    const tenant = await this.getTenantById(tenantId);
    if (!tenant.isActive) {
      throw new ForbiddenException('Аккаунт заблокирован. Обратитесь в поддержку.');
    }
    if (tenant.deletedAt) {
      throw new ForbiddenException('Аккаунт удалён.');
    }
  }
}
```

---

## 5. TenantLimits Enforcement

### 5.1. LimitsGuard

```typescript
// src/modules/tenants/guards/tenant-limits.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { TenantLimitsService } from '../tenant-limits.service';

export const CHECK_LIMIT_KEY = 'checkLimit';
export const CheckLimit = (resource: LimitResource) =>
  SetMetadata(CHECK_LIMIT_KEY, resource);

export type LimitResource = 'restaurant' | 'guest' | 'posIntegration' | 'adminUser';

@Injectable()
export class TenantLimitsGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly limitsService: TenantLimitsService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const resource = this.reflector.get<LimitResource>(
      CHECK_LIMIT_KEY,
      context.getHandler(),
    );

    if (!resource) return true; // нет аннотации — пропускаем

    const request = context.switchToHttp().getRequest();
    const tenantId = request['tenantId'];

    if (!tenantId) throw new ForbiddenException('Tenant не определён');

    const canAdd = await this.limitsService.checkLimit(tenantId, resource);

    if (!canAdd) {
      const limitInfo = await this.limitsService.getLimitInfo(tenantId, resource);
      throw new ForbiddenException({
        message: `Достигнут лимит плана: ${resource}`,
        limit: limitInfo.max,
        current: limitInfo.current,
        upgradeUrl: 'https://app.max-loyalty.com/billing/upgrade',
      });
    }

    return true;
  }
}
```

### 5.2. TenantLimitsService

```typescript
// src/modules/tenants/tenant-limits.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../database/prisma.service';
import { EventEmitter2 } from '@nestjs/event-emitter';

@Injectable()
export class TenantLimitsService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async checkLimit(tenantId: string, resource: LimitResource): Promise<boolean> {
    const limits = await this.prisma.tenantLimits.findUnique({
      where: { tenantId },
    });

    if (!limits) return true; // нет записи — безлимит (FREE с пустыми null)

    const { max, current } = this.getResourceFields(limits, resource);

    if (max === null) return true; // null = безлимит

    // Предупреждение при 90%
    if (current / max >= limits.warningThresholdPct / 100) {
      this.eventEmitter.emit('tenant.limit.warning', {
        tenantId,
        resource,
        current,
        max,
        pct: Math.round((current / max) * 100),
      });
    }

    return current < max;
  }

  async getLimitInfo(tenantId: string, resource: LimitResource) {
    const limits = await this.prisma.tenantLimits.findUnique({
      where: { tenantId },
    });
    const { max, current } = this.getResourceFields(limits, resource);
    return { max, current, pct: max ? Math.round((current / max) * 100) : 0 };
  }

  // Инкремент после создания ресурса
  async increment(tenantId: string, resource: LimitResource): Promise<void> {
    const field = this.getCurrentField(resource);
    await this.prisma.tenantLimits.update({
      where: { tenantId },
      data: { [field]: { increment: 1 } },
    });
  }

  // Декремент после удаления ресурса
  async decrement(tenantId: string, resource: LimitResource): Promise<void> {
    const field = this.getCurrentField(resource);
    await this.prisma.tenantLimits.update({
      where: { tenantId },
      data: { [field]: { decrement: 1 } },
    });
  }

  // Пересчёт счётчиков (для reconciliation)
  async recalculate(tenantId: string): Promise<void> {
    const [restaurants, guests, posIntegrations] = await Promise.all([
      this.prisma.restaurant.count({ where: { tenantId, deletedAt: null } }),
      this.prisma.guestCard.count({ where: { tenantId, deletedAt: null } }),
      this.prisma.pOSIntegration.count({ where: { tenantId } }),
    ]);

    await this.prisma.tenantLimits.update({
      where: { tenantId },
      data: {
        currentRestaurants: restaurants,
        currentGuests: guests,
        currentPosIntegrations: posIntegrations,
        updatedAt: new Date(),
      },
    });
  }

  private getResourceFields(limits: any, resource: LimitResource) {
    const map: Record<LimitResource, { max: string; current: string }> = {
      restaurant: { max: 'maxRestaurants', current: 'currentRestaurants' },
      guest: { max: 'maxGuests', current: 'currentGuests' },
      posIntegration: { max: 'maxPosIntegrations', current: 'currentPosIntegrations' },
      adminUser: { max: 'maxAdminUsers', current: 'currentAdminUsers' },
    };
    const fields = map[resource];
    return { max: limits[fields.max], current: limits[fields.current] };
  }

  private getCurrentField(resource: LimitResource): string {
    const map: Record<LimitResource, string> = {
      restaurant: 'currentRestaurants',
      guest: 'currentGuests',
      posIntegration: 'currentPosIntegrations',
      adminUser: 'currentAdminUsers',
    };
    return map[resource];
  }
}
```

### 5.3. Limit Warning Listener

```typescript
// src/modules/tenants/listeners/tenant-limit-warning.listener.ts
import { Injectable } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { NotificationService } from '../../notifications/notification.service';

@Injectable()
export class TenantLimitWarningListener {
  constructor(private readonly notificationService: NotificationService) {}

  @OnEvent('tenant.limit.warning')
  async handleLimitWarning(payload: {
    tenantId: string;
    resource: string;
    current: number;
    max: number;
    pct: number;
  }) {
    // Уведомляем Owner/Admin через Telegram + Email
    await this.notificationService.sendToTenantOwners(payload.tenantId, {
      type: 'LIMIT_WARNING',
      channel: ['TELEGRAM', 'EMAIL'],
      title: `⚠️ Лимит плана достигнут на ${payload.pct}%`,
      message: `Использовано ${payload.current} из ${payload.max} (${payload.resource}). Рассмотрите апгрейд тарифа.`,
      data: {
        resource: payload.resource,
        current: payload.current,
        max: payload.max,
        upgradeUrl: 'https://app.max-loyalty.com/billing/upgrade',
      },
    });
  }
}
```

---

## 6. Restaurant Management

### 6.1. RestaurantService

```typescript
// src/modules/restaurants/restaurant.service.ts
import {
  Injectable,
  NotFoundException,
  ConflictException,
  ForbiddenException,
} from '@nestjs/common';
import { PrismaService } from '../../database/prisma.service';
import { TenantLimitsService } from '../tenants/tenant-limits.service';
import { CacheService } from '../cache/cache.service';

@Injectable()
export class RestaurantService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly limitsService: TenantLimitsService,
    private readonly cache: CacheService,
  ) {}

  // === Список ресторанов tenant'а ===
  async findAll(tenantId: string, userId: string, userRole: string) {
    // Manager видит только свои рестораны
    if (userRole === 'MANAGER') {
      return this.prisma.restaurant.findMany({
        where: {
          tenantId,
          deletedAt: null,
          userRestaurantRoles: { some: { userId } },
        },
        orderBy: { createdAt: 'asc' },
      });
    }

    // Admin/Owner видит все
    return this.prisma.restaurant.findMany({
      where: { tenantId, deletedAt: null },
      orderBy: { createdAt: 'asc' },
    });
  }

  // === Создание ресторана (проверка лимита) ===
  async create(tenantId: string, dto: CreateRestaurantDto, actorId: string) {
    // Лимит проверяется в Guard, но дублируем для надёжности
    const canCreate = await this.limitsService.checkLimit(tenantId, 'restaurant');
    if (!canCreate) {
      throw new ForbiddenException('Достигнут лимит ресторанов по вашему плану');
    }

    // Уникальность slug в рамках tenant'а
    const slugBase = this.generateSlug(dto.name);
    const slug = await this.ensureUniqueSlug(tenantId, slugBase);

    const restaurant = await this.prisma.$transaction(async (tx) => {
      const r = await tx.restaurant.create({
        data: {
          tenantId,
          name: dto.name,
          slug,
          description: dto.description,
          city: dto.city,
          address: dto.address,
          latitude: dto.latitude,
          longitude: dto.longitude,
          phone: dto.phone,
          email: dto.email,
          isActive: true,
        },
      });

      // Инкремент счётчика лимитов
      await tx.tenantLimits.update({
        where: { tenantId },
        data: { currentRestaurants: { increment: 1 } },
      });

      await tx.activityLog.create({
        data: {
          tenantId,
          actorId,
          action: 'RESTAURANT_CREATED',
          entityType: 'RESTAURANT',
          entityId: r.id,
          newValue: { name: r.name, slug: r.slug },
        },
      });

      return r;
    });

    // Инвалидация кэша списка ресторанов
    await this.cache.del(`tenant:${tenantId}:restaurants`);

    return restaurant;
  }

  // === Обновление ресторана ===
  async update(
    tenantId: string,
    restaurantId: string,
    dto: UpdateRestaurantDto,
    actorId: string,
    userRole: string,
    userId: string,
  ) {
    const restaurant = await this.findOneOrFail(tenantId, restaurantId);

    // Manager может обновлять только свои рестораны
    if (userRole === 'MANAGER') {
      await this.assertManagerAccess(tenantId, restaurantId, userId);
    }

    const updated = await this.prisma.restaurant.update({
      where: { id: restaurantId },
      data: {
        name: dto.name ?? restaurant.name,
        description: dto.description,
        city: dto.city,
        address: dto.address,
        latitude: dto.latitude,
        longitude: dto.longitude,
        phone: dto.phone,
        email: dto.email,
        isActive: dto.isActive ?? restaurant.isActive,
      },
    });

    await this.prisma.activityLog.create({
      data: {
        tenantId,
        actorId,
        action: 'RESTAURANT_UPDATED',
        entityType: 'RESTAURANT',
        entityId: restaurantId,
        oldValue: { name: restaurant.name },
        newValue: { name: updated.name },
      },
    });

    await this.cache.del(`tenant:${tenantId}:restaurants`);
    await this.cache.del(`restaurant:${restaurantId}`);

    return updated;
  }

  // === Soft-delete ресторана ===
  async remove(tenantId: string, restaurantId: string, actorId: string) {
    await this.findOneOrFail(tenantId, restaurantId);

    await this.prisma.$transaction(async (tx) => {
      await tx.restaurant.update({
        where: { id: restaurantId },
        data: { deletedAt: new Date(), isActive: false },
      });

      await tx.tenantLimits.update({
        where: { tenantId },
        data: { currentRestaurants: { decrement: 1 } },
      });

      await tx.activityLog.create({
        data: {
          tenantId,
          actorId,
          action: 'RESTAURANT_DELETED',
          entityType: 'RESTAURANT',
          entityId: restaurantId,
        },
      });
    });

    await this.cache.del(`tenant:${tenantId}:restaurants`);
    await this.cache.del(`restaurant:${restaurantId}`);
  }

  // === Назначение менеджера на ресторан ===
  async assignManager(
    tenantId: string,
    restaurantId: string,
    managerId: string,
    actorId: string,
  ) {
    await this.findOneOrFail(tenantId, restaurantId);

    // Проверяем, что managerId имеет роль MANAGER в tenant'е
    const managerRole = await this.prisma.userTenantRole.findFirst({
      where: { userId: managerId, tenantId, role: 'MANAGER' },
    });
    if (!managerRole) {
      throw new ForbiddenException('Пользователь не является менеджером этого tenant\'а');
    }

    await this.prisma.userRestaurantRole.upsert({
      where: {
        userId_restaurantId_role: {
          userId: managerId,
          restaurantId,
          role: 'MANAGER',
        },
      },
      create: { userId: managerId, restaurantId, role: 'MANAGER' },
      update: {},
    });

    await this.prisma.activityLog.create({
      data: {
        tenantId,
        actorId,
        action: 'MANAGER_ASSIGNED',
        entityType: 'RESTAURANT',
        entityId: restaurantId,
        newValue: { managerId },
      },
    });
  }

  // === Назначение кассира на ресторан ===
  async assignCashier(
    tenantId: string,
    restaurantId: string,
    cashierId: string,
    actorId: string,
  ) {
    await this.findOneOrFail(tenantId, restaurantId);

    await this.prisma.userRestaurantRole.upsert({
      where: {
        userId_restaurantId_role: {
          userId: cashierId,
          restaurantId,
          role: 'CASHIER',
        },
      },
      create: { userId: cashierId, restaurantId, role: 'CASHIER' },
      update: {},
    });

    await this.prisma.activityLog.create({
      data: {
        tenantId,
        actorId,
        action: 'CASHIER_ASSIGNED',
        entityType: 'RESTAURANT',
        entityId: restaurantId,
        newValue: { cashierId },
      },
    });
  }

  // === Helpers ===
  async findOneOrFail(tenantId: string, restaurantId: string) {
    const restaurant = await this.prisma.restaurant.findFirst({
      where: { id: restaurantId, tenantId, deletedAt: null },
    });
    if (!restaurant) throw new NotFoundException('Ресторан не найден');
    return restaurant;
  }

  private async assertManagerAccess(
    tenantId: string,
    restaurantId: string,
    userId: string,
  ) {
    const access = await this.prisma.userRestaurantRole.findFirst({
      where: { userId, restaurantId, role: { in: ['MANAGER', 'CASHIER'] } },
    });
    if (!access) throw new ForbiddenException('Нет доступа к этому ресторану');
  }

  private generateSlug(name: string): string {
    return name
      .toLowerCase()
      .replace(/[^a-z0-9]/gi, '-')
      .replace(/-+/g, '-')
      .replace(/^-|-$/g, '')
      .substring(0, 50);
  }

  private async ensureUniqueSlug(tenantId: string, slug: string): Promise<string> {
    let candidate = slug;
    let counter = 1;
    while (true) {
      const exists = await this.prisma.restaurant.findFirst({
        where: { tenantId, slug: candidate },
      });
      if (!exists) return candidate;
      candidate = `${slug}-${counter++}`;
    }
  }
}
```

---

## 7. Prisma Schema

### 7.1. Tenant & Related Models

```prisma
// ============================================
// TENANT
// ============================================
model Tenant {
  id          String   @id @default(uuid())
  name        String
  slug        String   @unique
  description String?

  // Contact
  contactEmail String?
  contactPhone String?
  websiteUrl   String?

  // Branding
  logoUrl      String?
  primaryColor String?

  // Status
  isActive      Boolean   @default(true)
  blockedReason String?

  // Timestamps
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  deletedAt DateTime?

  // Relations
  restaurants      Restaurant[]
  userRoles        UserTenantRole[]
  subscription     Subscription?
  tenantLimits     TenantLimits?
  billingInfo      BillingInfo?
  loyaltySystem    LoyaltySystem?
  loyaltyRules     LoyaltyRule[]
  loyaltyLevels    LoyaltyLevel[]
  loyaltyPromos    LoyaltyPromo[]
  guestCards       GuestCard[]
  ballTransactions BallTransaction[]
  guestVisits      GuestVisit[]
  promoBallGranted PromoBallGranted[]
  posIntegrations  POSIntegration[]
  posTransactions  POSTransaction[]
  registrationLinks RegistrationLink[]
  telegramBots     TelegramBot[]
  notifications    Notification[]
  activityLogs     ActivityLog[]
  analyticsDaily   AnalyticsDailySnapshot[]
  analyticsMonthly AnalyticsMonthlySnapshot[]

  @@index([slug])
  @@index([isActive])
  @@index([deletedAt])
}

// ============================================
// OWNER REGISTRATION LINK
// ============================================
model OwnerRegistrationLink {
  id          String           @id @default(uuid())
  token       String           @unique @default(uuid())
  email       String?
  planHint    SubscriptionPlan @default(STANDARD)
  usedAt      DateTime?
  expiresAt   DateTime
  createdById String
  createdBy   User             @relation(fields: [createdById], references: [id])
  createdAt   DateTime         @default(now())

  @@index([token])
  @@index([expiresAt])
}

// ============================================
// TENANT LIMITS
// ============================================
model TenantLimits {
  id       String @id @default(uuid())
  tenantId String @unique
  tenant   Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  // Лимиты (NULL = безлимит)
  maxRestaurants     Int?
  maxGuests          Int?
  maxPosIntegrations Int?
  maxAdminUsers      Int?
  maxStorageMb       Int?

  // Текущие счётчики
  currentRestaurants     Int @default(0)
  currentGuests          Int @default(0)
  currentPosIntegrations Int @default(0)
  currentAdminUsers      Int @default(0)
  currentStorageMb       Int @default(0)

  warningThresholdPct Int      @default(90)
  updatedAt           DateTime @updatedAt
}

// ============================================
// BILLING INFO
// ============================================
model BillingInfo {
  id       String @id @default(uuid())
  tenantId String @unique
  tenant   Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  // Company
  companyName String?
  taxId       String?

  // Address
  billingAddress String?
  city           String?
  postalCode     String?
  country        String? @default("RU")

  // Payment method
  paymentProvider       PaymentProvider?
  paymentMethodId       String?
  providerCustomerId    String?

  // Contact
  billingEmail String?
  billingPhone String?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

// ============================================
// RESTAURANT
// ============================================
model Restaurant {
  id       String @id @default(uuid())
  tenantId String
  tenant   Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  name        String
  slug        String
  description String?

  // Address
  city      String?
  address   String?
  latitude  Decimal? @db.Decimal(10, 8)
  longitude Decimal? @db.Decimal(11, 8)

  // Contact
  phone String?
  email String?

  isActive  Boolean   @default(true)
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  deletedAt DateTime?

  // Relations
  userRestaurantRoles  UserRestaurantRole[]
  guestCards           GuestCard[]
  ballTransactions     BallTransaction[]
  guestVisits          GuestVisit[]
  posIntegrations      POSIntegration[]
  posTransactions      POSTransaction[]
  telegramBots         TelegramBot[]
  analyticsDaily       AnalyticsDailySnapshot[]
  analyticsMonthly     AnalyticsMonthlySnapshot[]

  @@unique([tenantId, slug])
  @@index([tenantId])
  @@index([isActive])
  @@index([deletedAt])
}
```

---

## 8. API Endpoints

### 8.1. Owner Registration Controller

```typescript
// src/modules/tenants/controllers/owner-registration.controller.ts
import { Controller, Post, Get, Body, Param, UseGuards } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiBearerAuth } from '@nestjs/swagger';
import { JwtAuthGuard } from '../../auth/guards/jwt-auth.guard';
import { RequirePermissions } from '../../auth/decorators/require-permissions.decorator';
import { Permission } from '../../auth/enums/permission.enum';

@ApiTags('Owner Registration')
@Controller('owner-registration')
export class OwnerRegistrationController {
  constructor(private readonly service: OwnerRegistrationService) {}

  // POST /owner-registration/links — создать invite (только платформа OWNER)
  @Post('links')
  @UseGuards(JwtAuthGuard)
  @RequirePermissions(Permission.PLATFORM_ADMIN)
  @ApiOperation({ summary: 'Создать invite-ссылку для нового Owner' })
  async createLink(
    @Body() dto: CreateOwnerLinkDto,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.service.createRegistrationLink(dto, user.sub);
  }

  // GET /owner-registration/validate/:token — проверить токен (публично)
  @Get('validate/:token')
  @ApiOperation({ summary: 'Проверить валидность invite-токена' })
  async validateToken(@Param('token') token: string) {
    const link = await this.service.validateToken(token);
    return {
      valid: true,
      planHint: link.planHint,
      email: link.email,
      expiresAt: link.expiresAt,
    };
  }

  // POST /owner-registration/register — регистрация Owner (публично)
  @Post('register')
  @ApiOperation({ summary: 'Зарегистрировать нового Owner по invite-токену' })
  async register(@Body() dto: RegisterOwnerWithTokenDto) {
    return this.service.registerOwner(dto.token, dto);
  }
}
```

### 8.2. Tenant Controller

```typescript
// src/modules/tenants/controllers/tenant.controller.ts
@ApiTags('Tenants')
@ApiBearerAuth()
@UseGuards(JwtAuthGuard, TenantGuard)
@Controller('tenants')
export class TenantController {
  // GET /tenants/me — текущий tenant
  @Get('me')
  async getMyTenant(@TenantId() tenantId: string) {
    return this.tenantService.getTenantById(tenantId);
  }

  // PATCH /tenants/me — обновить tenant
  @Patch('me')
  @RequirePermissions(Permission.MANAGE_SETTINGS)
  async updateMyTenant(
    @TenantId() tenantId: string,
    @Body() dto: UpdateTenantDto,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.tenantService.updateTenant(tenantId, dto, user.sub);
  }

  // GET /tenants/me/limits — лимиты и текущее использование
  @Get('me/limits')
  async getMyLimits(@TenantId() tenantId: string) {
    return this.prisma.tenantLimits.findUnique({ where: { tenantId } });
  }
}
```

### 8.3. Restaurant Controller

```typescript
// src/modules/restaurants/restaurant.controller.ts
@ApiTags('Restaurants')
@ApiBearerAuth()
@UseGuards(JwtAuthGuard, TenantGuard)
@Controller('restaurants')
export class RestaurantController {
  // GET /restaurants
  @Get()
  @RequirePermissions(Permission.VIEW_RESTAURANT_SETTINGS)
  async findAll(
    @TenantId() tenantId: string,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.restaurantService.findAll(tenantId, user.sub, user.role);
  }

  // GET /restaurants/:id
  @Get(':id')
  @RequirePermissions(Permission.VIEW_RESTAURANT_SETTINGS)
  async findOne(
    @TenantId() tenantId: string,
    @Param('id') id: string,
  ) {
    return this.restaurantService.findOneOrFail(tenantId, id);
  }

  // POST /restaurants
  @Post()
  @UseGuards(TenantLimitsGuard)
  @CheckLimit('restaurant')
  @RequirePermissions(Permission.MANAGE_RESTAURANTS)
  async create(
    @TenantId() tenantId: string,
    @Body() dto: CreateRestaurantDto,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.restaurantService.create(tenantId, dto, user.sub);
  }

  // PATCH /restaurants/:id
  @Patch(':id')
  @RequirePermissions(Permission.MANAGE_RESTAURANTS)
  async update(
    @TenantId() tenantId: string,
    @Param('id') id: string,
    @Body() dto: UpdateRestaurantDto,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.restaurantService.update(tenantId, id, dto, user.sub, user.role, user.sub);
  }

  // DELETE /restaurants/:id
  @Delete(':id')
  @RequirePermissions(Permission.MANAGE_RESTAURANTS)
  async remove(
    @TenantId() tenantId: string,
    @Param('id') id: string,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.restaurantService.remove(tenantId, id, user.sub);
  }

  // POST /restaurants/:id/assign-manager
  @Post(':id/assign-manager')
  @RequirePermissions(Permission.MANAGE_TEAM)
  async assignManager(
    @TenantId() tenantId: string,
    @Param('id') restaurantId: string,
    @Body('managerId') managerId: string,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.restaurantService.assignManager(tenantId, restaurantId, managerId, user.sub);
  }

  // POST /restaurants/:id/assign-cashier
  @Post(':id/assign-cashier')
  @RequirePermissions(Permission.MANAGE_TEAM)
  async assignCashier(
    @TenantId() tenantId: string,
    @Param('id') restaurantId: string,
    @Body('cashierId') cashierId: string,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.restaurantService.assignCashier(tenantId, restaurantId, cashierId, user.sub);
  }
}
```

### 8.4. Сводная таблица эндпоинтов

| Метод | Маршрут | Роль | Описание |
|---|---|---|---|
| POST | `/owner-registration/links` | PLATFORM_OWNER | Создать invite-ссылку |
| GET | `/owner-registration/validate/:token` | Public | Проверить токен |
| POST | `/owner-registration/register` | Public | Регистрация Owner |
| GET | `/tenants/me` | Owner/Admin | Инфо о tenant'е |
| PATCH | `/tenants/me` | Owner | Обновить tenant |
| GET | `/tenants/me/limits` | Owner/Admin | Лимиты плана |
| GET | `/restaurants` | All | Список ресторанов |
| GET | `/restaurants/:id` | All | Один ресторан |
| POST | `/restaurants` | Owner/Admin | Создать ресторан |
| PATCH | `/restaurants/:id` | Owner/Admin/Manager | Обновить ресторан |
| DELETE | `/restaurants/:id` | Owner/Admin | Удалить ресторан |
| POST | `/restaurants/:id/assign-manager` | Owner/Admin | Назначить менеджера |
| POST | `/restaurants/:id/assign-cashier` | Owner/Admin/Manager | Назначить кассира |

---

## 9. Guards & Middleware

### 9.1. TenantGuard — проверка активности

```typescript
// src/modules/tenants/guards/tenant.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { TenantService } from '../tenant.service';

@Injectable()
export class TenantGuard implements CanActivate {
  constructor(private readonly tenantService: TenantService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const tenantId = request['tenantId'];

    if (!tenantId) throw new ForbiddenException('Tenant не определён в токене');

    // Проверяем активность tenant'а (с кэшем)
    await this.tenantService.assertTenantActive(tenantId);

    return true;
  }
}
```

### 9.2. RestaurantAccessGuard — ABAC для Manager

```typescript
// src/modules/restaurants/guards/restaurant-access.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { PrismaService } from '../../../database/prisma.service';

@Injectable()
export class RestaurantAccessGuard implements CanActivate {
  constructor(private readonly prisma: PrismaService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const { userId, userRole, tenantId } = request;
    const restaurantId = request.params.id;

    // Owner и Admin видят все рестораны tenant'а
    if (['OWNER', 'RESTAURANT_ADMIN'].includes(userRole)) return true;

    // Manager видит только свои
    if (userRole === 'MANAGER' && restaurantId) {
      const access = await this.prisma.userRestaurantRole.findFirst({
        where: { userId, restaurantId, role: 'MANAGER' },
      });
      if (!access) throw new ForbiddenException('Нет доступа к этому ресторану');
    }

    return true;
  }
}
```

### 9.3. @TenantId() Декоратор

```typescript
// src/common/decorators/tenant-id.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const TenantId = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request['tenantId'];
  },
);
```

---

## 10. Тесты & Чеклист

### 10.1. Сценарии тестирования

| # | Сценарий | Ожидаемый результат |
|---|---|---|
| T1 | Owner регистрируется по валидной ссылке | Tenant + Subscription TRIAL + TenantLimits созданы |
| T2 | Owner использует просроченную ссылку | HTTP 400 "Ссылка истекла" |
| T3 | Owner повторно использует ссылку | HTTP 400 "Ссылка уже использована" |
| T4 | Admin создаёт ресторан в рамках лимита | HTTP 201, currentRestaurants++ |
| T5 | Admin создаёт ресторан при достижении лимита | HTTP 403 с upgradeUrl |
| T6 | Manager видит только назначенные рестораны | Список без чужих ресторанов |
| T7 | Manager пытается обновить чужой ресторан | HTTP 403 |
| T8 | Лимит достигает 90% | Уведомление Owner через Telegram/Email |
| T9 | Tenant деактивирован, пользователь делает запрос | HTTP 403 "Аккаунт заблокирован" |
| T10 | Создание ресторана с дублирующимся slug | Slug автоматически уникализируется |
| T11 | Пересчёт лимитов через recalculate() | Счётчики совпадают с реальным count() |
| T12 | Назначение менеджера без роли MANAGER | HTTP 403 "Не является менеджером" |

### 10.2. Чеклист реализации

**P0 — Критично:**
- [ ] TenantMiddleware регистрирован глобально в AppModule
- [ ] TenantGuard применён ко всем защищённым маршрутам
- [ ] Prisma middleware изолирует данные по tenantId
- [ ] Owner Registration flow: User + Tenant + Subscription + TenantLimits в транзакции
- [ ] TenantLimitsGuard блокирует превышение лимитов
- [ ] Restaurant CRUD полностью работает с лимитами

**P1 — Важно:**
- [ ] Manager видит только свои рестораны (ABAC)
- [ ] Кэш инвалидируется при update/delete tenant/restaurant
- [ ] Limit Warning событие уведомляет Owner при 90%
- [ ] Soft-delete ресторана с декрементом счётчика

**P2 — Желательно:**
- [ ] Slug автогенерация с уникализацией
- [ ] recalculate() CRON-задача раз в сутки для reconciliation
- [ ] Geocoding через Yandex Maps API для координат ресторана

---

*S-04 завершён. Следующая часть: **S-05** — Billing & Subscriptions (тарифы, апгрейд/даунгрейд, оплата, счета, PASTDUE flow)*
