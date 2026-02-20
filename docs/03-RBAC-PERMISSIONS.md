# 03 — RBAC & PERMISSIONS SYSTEM
**Max Loyalty Platform | Part 3 of 7**

> **Навигация:** [← 02 Auth System](./02-AUTH-SYSTEM.md) | [INDEX](./INDEX.md) | [04 Tenant & Restaurant →](./04-TENANT-RESTAURANT.md)

---

## Оглавление

1. [Обзор RBAC стратегии](#1-обзор-rbac-стратегии)
2. [Роли системы](#2-роли-системы)
3. [30 Permissions — полный список](#3-30-permissions--полный-список)
4. [Маппинг ролей к правам](#4-маппинг-ролей-к-правам)
5. [Динамические (кастомные) права](#5-динамические-кастомные-права)
6. [PermissionsGuard + PermissionsService](#6-permissionsguard--permissionsservice)
7. [ABAC — атрибутный контроль доступа](#7-abac--атрибутный-контроль-доступа)
8. [Cashier PIN-код](#8-cashier-pin-код)
9. [Approval Workflow (Manager → Admin)](#9-approval-workflow-manager--admin)
10. [Кэш прав Redis (5 мин TTL)](#10-кэш-прав-redis-5-мин-ttl)
11. [Prisma Schema — RBAC сущности](#11-prisma-schema--rbac-сущности)
12. [Декораторы и Guards](#12-декораторы-и-guards)
13. [API Endpoints](#13-api-endpoints)
14. [Матрица прав (сводная таблица)](#14-матрица-прав-сводная-таблица)
15. [Тесты RBAC](#15-тесты-rbac)
16. [Чеклист реализации](#16-чеклист-реализации)

---

## 1. Обзор RBAC стратегии

```typescript
// RBAC Strategy Configuration
const RBAC_STRATEGY = {
  model: 'Role-Based + Attribute-Based (hybrid)',
  enforcement: 'Guards + Decorators (NestJS)',
  granularity: 'Endpoint + Resource level',
  dynamicPermissions: true,       // Admin может добавлять/убирать права у конкретного пользователя
  permissionCaching: true,        // Redis 5 мин TTL
  tenantIsolation: true,          // Пользователь видит только свой tenant
  approvalWorkflow: true,         // Manager → Admin для критичных операций
  cashierPIN: true,               // PIN-код вместо пароля для кассиров
};
```

### Принцип работы

```
HTTP Request
    │
    ▼
JwtAuthGuard          (1) Проверяет JWT токен
    │
    ▼
TenantGuard           (2) Проверяет tenant isolation
    │
    ▼
PermissionsGuard      (3) Проверяет права из Redis/DB
    │
    ▼
ResourceOwnershipGuard (4) Ресурс принадлежит tenant?
    │
    ▼
ABACGuard             (5) Атрибутные ограничения (Manager → своя точка)
    │
    ▼
Controller            (6) Бизнес-логика
```

---

## 2. Роли системы

```typescript
// apps/backend/src/modules/auth/enums/role.enum.ts
export enum Role {
  OWNER   = 'OWNER',    // Владелец платформы (Max Loyalty team)
  ADMIN   = 'ADMIN',    // Администратор ресторана (RestaurantAdmin)
  MANAGER = 'MANAGER',  // Менеджер ресторана / точки
  CASHIER = 'CASHIER',  // Кассир на POS-терминале
  GUEST   = 'GUEST',    // Гость программы лояльности
}
```

### Описание ролей

| Роль | Кто | Создаёт | Кол-во |
|------|-----|---------|--------|
| `OWNER` | Команда Max Loyalty | Только вручную в системе | 1 (платформенный) |
| `ADMIN` | Владелец ресторана, директор | Owner через спец. ссылку | 1-5 на tenant |
| `MANAGER` | Менеджер смены, старший кассир | Admin | Не ограничено |
| `CASHIER` | Кассир | Admin / Manager | Не ограничено |
| `GUEST` | Посетитель ресторана | Кассир / самостоятельно | Миллионы |

### Иерархия доступа

```
OWNER
  └─► Видит ВСЕ tenant'ы, может имперсонировать любого
  
ADMIN (per tenant)
  └─► Видит весь свой tenant
  └─► Создаёт Manager, Cashier
  └─► Управляет лояльностью, биллингом, API-ключами
  
MANAGER (per tenant, per restaurant)
  └─► Видит свои рестораны (опционально все в tenant)
  └─► Управляет гостями, проводит транзакции
  └─► Создаёт правила (требует одобрения Admin)
  
CASHIER (per tenant, per restaurant)
  └─► Работает с одной точкой
  └─► Регистрирует гостей, начисляет/списывает баллы
  
GUEST
  └─► Видит только свою карту лояльности
```

---

## 3. 30 Permissions — полный список

```typescript
// apps/backend/src/modules/rbac/enums/permission.enum.ts
export enum Permission {
  // ──────────────────────────────────────────
  // GUESTS (5 прав)
  // ──────────────────────────────────────────
  GUESTS_VIEW   = 'guests:view',    // Просмотр гостей и карточек
  GUESTS_CREATE = 'guests:create',  // Регистрация нового гостя
  GUESTS_UPDATE = 'guests:update',  // Редактирование данных гостя
  GUESTS_DELETE = 'guests:delete',  // Удаление гостя (мягкое)
  GUESTS_EXPORT = 'guests:export',  // Экспорт базы гостей в CSV/Excel

  // ──────────────────────────────────────────
  // LOYALTY (5 прав)
  // ──────────────────────────────────────────
  LOYALTY_VIEW             = 'loyalty:view',             // Просмотр правил, уровней, промо
  LOYALTY_CONFIGURE        = 'loyalty:configure',        // Создание/редактирование правил и уровней
  LOYALTY_MANUAL_ADJUST    = 'loyalty:manual_adjust',    // Ручная корректировка баллов
  LOYALTY_TRANSACTIONS_VIEW = 'loyalty:transactions_view', // Просмотр транзакций
  LOYALTY_PROMO_ACTIVATE   = 'loyalty:promo_activate',   // Активация промо (Admin или с permission)

  // ──────────────────────────────────────────
  // ANALYTICS (3 права)
  // ──────────────────────────────────────────
  ANALYTICS_VIEW         = 'analytics:view',         // Просмотр дашборда и отчётов
  ANALYTICS_EXPORT       = 'analytics:export',       // Экспорт отчётов
  ANALYTICS_ADVANCED     = 'analytics:advanced',     // RFM, когортный анализ, LTV (Admin+)

  // ──────────────────────────────────────────
  // BILLING (2 права)
  // ──────────────────────────────────────────
  BILLING_VIEW   = 'billing:view',   // Просмотр счетов и тарифа
  BILLING_MANAGE = 'billing:manage', // Смена тарифа, оплата

  // ──────────────────────────────────────────
  // TEAM (4 права)
  // ──────────────────────────────────────────
  TEAM_VIEW             = 'team:view',             // Список сотрудников
  TEAM_INVITE           = 'team:invite',           // Приглашение новых пользователей
  TEAM_REMOVE           = 'team:remove',           // Удаление пользователей из команды
  TEAM_EDIT_PERMISSIONS = 'team:edit_permissions', // Изменение прав конкретных пользователей

  // ──────────────────────────────────────────
  // SETTINGS (2 права)
  // ──────────────────────────────────────────
  SETTINGS_VIEW = 'settings:view', // Просмотр настроек
  SETTINGS_EDIT = 'settings:edit', // Редактирование настроек

  // ──────────────────────────────────────────
  // POS (2 права)
  // ──────────────────────────────────────────
  POS_VIEW      = 'pos:view',      // Просмотр POS-интеграций и транзакций
  POS_CONFIGURE = 'pos:configure', // Настройка POS-интеграции, API-ключи

  // ──────────────────────────────────────────
  // RESTAURANTS (2 права)
  // ──────────────────────────────────────────
  RESTAURANTS_VIEW   = 'restaurants:view',   // Просмотр точек
  RESTAURANTS_MANAGE = 'restaurants:manage', // Создание/редактирование точек

  // ──────────────────────────────────────────
  // APPROVAL WORKFLOW (1 право)
  // ──────────────────────────────────────────
  APPROVALS_MANAGE = 'approvals:manage', // Одобрение/отклонение заявок от Manager

  // ──────────────────────────────────────────
  // ADMIN-ONLY (2 права)
  // ──────────────────────────────────────────
  ADMIN_IMPERSONATE      = 'admin:impersonate',       // Имперсонация пользователей (только Owner)
  ADMIN_VIEW_ALL_TENANTS = 'admin:view_all_tenants',  // Просмотр всех tenant'ов (только Owner)
}
```

**Итого: 30 прав** в 9 группах.

---

## 4. Маппинг ролей к правам

```typescript
// apps/backend/src/modules/rbac/constants/role-permissions.constant.ts
import { Role } from '../../auth/enums/role.enum';
import { Permission } from '../enums/permission.enum';

export const ROLE_PERMISSIONS: Record<Role, Permission[]> = {

  [Role.OWNER]: [
    // Все права без исключения
    ...Object.values(Permission),
  ],

  [Role.ADMIN]: [
    // GUESTS — полный доступ
    Permission.GUESTS_VIEW,
    Permission.GUESTS_CREATE,
    Permission.GUESTS_UPDATE,
    Permission.GUESTS_DELETE,
    Permission.GUESTS_EXPORT,
    // LOYALTY — полный доступ
    Permission.LOYALTY_VIEW,
    Permission.LOYALTY_CONFIGURE,
    Permission.LOYALTY_MANUAL_ADJUST,
    Permission.LOYALTY_TRANSACTIONS_VIEW,
    Permission.LOYALTY_PROMO_ACTIVATE,
    // ANALYTICS — полный доступ
    Permission.ANALYTICS_VIEW,
    Permission.ANALYTICS_EXPORT,
    Permission.ANALYTICS_ADVANCED,
    // BILLING — полный доступ
    Permission.BILLING_VIEW,
    Permission.BILLING_MANAGE,
    // TEAM — полный доступ
    Permission.TEAM_VIEW,
    Permission.TEAM_INVITE,
    Permission.TEAM_REMOVE,
    Permission.TEAM_EDIT_PERMISSIONS,
    // SETTINGS — полный доступ
    Permission.SETTINGS_VIEW,
    Permission.SETTINGS_EDIT,
    // POS — полный доступ
    Permission.POS_VIEW,
    Permission.POS_CONFIGURE,
    // RESTAURANTS — полный доступ
    Permission.RESTAURANTS_VIEW,
    Permission.RESTAURANTS_MANAGE,
    // APPROVALS — Admin одобряет заявки Manager
    Permission.APPROVALS_MANAGE,
    // NB: ADMIN_IMPERSONATE и ADMIN_VIEW_ALL_TENANTS — только Owner!
  ],

  [Role.MANAGER]: [
    // GUESTS — без удаления и экспорта (по умолчанию)
    Permission.GUESTS_VIEW,
    Permission.GUESTS_CREATE,
    Permission.GUESTS_UPDATE,
    // LOYALTY — без конфигурирования (нужно одобрение Admin)
    Permission.LOYALTY_VIEW,
    Permission.LOYALTY_MANUAL_ADJUST,
    Permission.LOYALTY_TRANSACTIONS_VIEW,
    // ANALYTICS — без расширенной аналитики
    Permission.ANALYTICS_VIEW,
    // SETTINGS — только просмотр
    Permission.SETTINGS_VIEW,
    // RESTAURANTS — только просмотр
    Permission.RESTAURANTS_VIEW,
    // TEAM — только просмотр
    Permission.TEAM_VIEW,
    // POS — только просмотр
    Permission.POS_VIEW,
  ],

  [Role.CASHIER]: [
    // Минимальный набор для работы на кассе
    Permission.GUESTS_VIEW,
    Permission.GUESTS_CREATE,
    Permission.LOYALTY_VIEW,
    Permission.LOYALTY_TRANSACTIONS_VIEW,
    Permission.RESTAURANTS_VIEW,
  ],

  [Role.GUEST]: [
    // Гость видит только свою карточку — handled at resource level
    Permission.LOYALTY_VIEW,            // Только своя карта
    Permission.LOYALTY_TRANSACTIONS_VIEW, // Только свои транзакции
  ],
};
```

---

## 5. Динамические (кастомные) права

Admin может точечно добавлять или убирать права у конкретного пользователя, не меняя его роль.

```typescript
// Пример: Manager получает право экспортировать данные
// (по умолчанию у Manager нет GUESTS_EXPORT)
// Admin добавляет через UI → UserTenantRole.customPermissions

// Пример: Cashier лишается права создавать гостей
// (добавляем GUESTS_CREATE в removedPermissions)
```

### CustomPermissions Entity

```typescript
// packages/database/prisma/schema.prisma

model UserTenantRole {
  id               String    @id @default(uuid())
  userId           String
  tenantId         String
  role             Role
  restaurantIds    String[]  // Массив restaurant ID (для ограничения по точкам)
  
  // Кастомные права
  customPermissions CustomPermission?
  
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  tenant Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  @@unique([userId, tenantId])
  @@index([userId])
  @@index([tenantId])
  @@index([role])
}

model CustomPermission {
  id               String   @id @default(uuid())
  userTenantRoleId String   @unique
  
  addedPermissions   String[] // Права СВЕРХ роли
  removedPermissions String[] // Права УБРАНЫ из роли
  
  changedBy  String   // userId того, кто менял
  reason     String?  // Причина изменения
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  userTenantRole UserTenantRole @relation(fields: [userTenantRoleId], references: [id], onDelete: Cascade)
}
```

### Применение кастомных прав

```typescript
// apps/backend/src/modules/rbac/services/permissions.service.ts

@Injectable()
export class PermissionsService {
  constructor(
    private prisma: PrismaService,
    @InjectRedis() private redis: Redis,
  ) {}

  /**
   * Получить итоговые права пользователя (роль + кастомные) с кэшированием
   */
  async getUserPermissions(userId: string, tenantId: string): Promise<Permission[]> {
    const cacheKey = `permissions:${userId}:${tenantId}`;
    
    // 1. Проверяем кэш
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached) as Permission[];
    }
    
    // 2. Берём из БД
    const userTenantRole = await this.prisma.userTenantRole.findUnique({
      where: { userId_tenantId: { userId, tenantId } },
      include: { customPermissions: true },
    });
    
    if (!userTenantRole) {
      return [];
    }
    
    // 3. Базовые права из роли
    let permissions: Permission[] = [
      ...(ROLE_PERMISSIONS[userTenantRole.role] ?? []),
    ];
    
    // 4. Применяем кастомные изменения
    if (userTenantRole.customPermissions) {
      const custom = userTenantRole.customPermissions;
      
      // Добавляем разрешённые
      permissions.push(...(custom.addedPermissions as Permission[]));
      
      // Убираем отозванные
      permissions = permissions.filter(
        (p) => !(custom.removedPermissions as string[]).includes(p),
      );
    }
    
    // 5. Дедупликация
    permissions = [...new Set(permissions)];
    
    // 6. Сохраняем в Redis на 5 минут
    await this.redis.setex(cacheKey, 300, JSON.stringify(permissions));
    
    return permissions;
  }

  /**
   * Проверить одно право
   */
  async hasPermission(
    userId: string,
    tenantId: string,
    permission: Permission,
  ): Promise<boolean> {
    const permissions = await this.getUserPermissions(userId, tenantId);
    return permissions.includes(permission);
  }

  /**
   * Инвалидировать кэш при изменении прав
   */
  async invalidateCache(userId: string, tenantId: string): Promise<void> {
    await this.redis.del(`permissions:${userId}:${tenantId}`);
  }

  /**
   * Обновить кастомные права пользователя
   */
  async updateCustomPermissions(
    userId: string,
    tenantId: string,
    addedPermissions: Permission[],
    removedPermissions: Permission[],
    changedBy: string,
    reason?: string,
  ): Promise<void> {
    const userTenantRole = await this.prisma.userTenantRole.findUnique({
      where: { userId_tenantId: { userId, tenantId } },
    });
    
    if (!userTenantRole) {
      throw new NotFoundException('UserTenantRole not found');
    }
    
    // Upsert CustomPermission
    await this.prisma.customPermission.upsert({
      where: { userTenantRoleId: userTenantRole.id },
      create: {
        userTenantRoleId: userTenantRole.id,
        addedPermissions,
        removedPermissions,
        changedBy,
        reason,
      },
      update: {
        addedPermissions,
        removedPermissions,
        changedBy,
        reason,
      },
    });
    
    // Инвалидируем кэш
    await this.invalidateCache(userId, tenantId);
    
    // Логируем изменение
    await this.activityLogService.create({
      actorId: changedBy,
      tenantId,
      action: 'PERMISSIONS_UPDATED',
      targetId: userId,
      metadata: {
        added: addedPermissions,
        removed: removedPermissions,
        reason,
      },
    });
  }
}
```

---

## 6. PermissionsGuard + PermissionsService

### Guard

```typescript
// apps/backend/src/modules/rbac/guards/permissions.guard.ts

@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private permissionsService: PermissionsService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // 1. Получаем требуемые права из декоратора
    const requiredPermissions = this.reflector.getAllAndOverride<Permission[]>(
      PERMISSIONS_KEY,
      [context.getHandler(), context.getClass()],
    );
    
    if (!requiredPermissions || requiredPermissions.length === 0) {
      return true; // Права не требуются — пропускаем
    }
    
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    if (!user) {
      throw new UnauthorizedException('Authentication required');
    }
    
    const tenantId = user.tenantId ?? request.headers['x-tenant-id'];
    
    if (!tenantId && user.role !== Role.OWNER) {
      throw new ForbiddenException('Tenant context required');
    }
    
    // 2. Owner имеет все права
    if (user.role === Role.OWNER) {
      return true;
    }
    
    // 3. Получаем права пользователя (из кэша или БД)
    const userPermissions = await this.permissionsService.getUserPermissions(
      user.id,
      tenantId,
    );
    
    // 4. Проверяем наличие ВСЕХ требуемых прав
    const hasAllPermissions = requiredPermissions.every((permission) =>
      userPermissions.includes(permission),
    );
    
    if (!hasAllPermissions) {
      const missing = requiredPermissions.filter(
        (p) => !userPermissions.includes(p),
      );
      throw new ForbiddenException({
        message: 'Insufficient permissions',
        required: requiredPermissions,
        missing,
      });
    }
    
    return true;
  }
}
```

### RolesGuard (для простых случаев)

```typescript
// apps/backend/src/modules/rbac/guards/roles.guard.ts

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    
    if (!requiredRoles || requiredRoles.length === 0) {
      return true;
    }
    
    const { user } = context.switchToHttp().getRequest();
    
    if (!user) {
      throw new UnauthorizedException('Authentication required');
    }
    
    const hasRole = requiredRoles.includes(user.role);
    
    if (!hasRole) {
      throw new ForbiddenException({
        message: 'Insufficient role',
        required: requiredRoles,
        actual: user.role,
      });
    }
    
    return true;
  }
}
```

---

## 7. ABAC — атрибутный контроль доступа

ABAC дополняет RBAC: Manager может работать только с теми ресторанами, к которым привязан.

```typescript
// apps/backend/src/modules/rbac/guards/resource-ownership.guard.ts

@Injectable()
export class ResourceOwnershipGuard implements CanActivate {
  constructor(private prisma: PrismaService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const resourceId = request.params.id;
    
    // Owner видит всё
    if (user.role === Role.OWNER) return true;
    
    // Проверяем, что ресурс принадлежит tenant пользователя
    if (resourceId) {
      const guest = await this.prisma.guestCard.findUnique({
        where: { id: resourceId },
        select: { tenantId: true },
      });
      
      if (!guest) {
        throw new NotFoundException('Resource not found');
      }
      
      if (guest.tenantId !== user.tenantId) {
        throw new ForbiddenException('Access denied: cross-tenant access attempt');
      }
    }
    
    return true;
  }
}

// apps/backend/src/modules/rbac/guards/restaurant-access.guard.ts

@Injectable()
export class RestaurantAccessGuard implements CanActivate {
  constructor(private prisma: PrismaService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    // Owner и Admin видят все рестораны tenant
    if (user.role === Role.OWNER || user.role === Role.ADMIN) {
      return true;
    }
    
    const restaurantId = 
      request.params.restaurantId ?? 
      request.body?.restaurantId ?? 
      request.query?.restaurantId;
    
    if (!restaurantId) return true; // Без restaurantId — не применяем
    
    // Manager и Cashier: проверяем привязку к ресторану
    const userTenantRole = await this.prisma.userTenantRole.findUnique({
      where: { userId_tenantId: { userId: user.id, tenantId: user.tenantId } },
    });
    
    if (!userTenantRole) {
      throw new ForbiddenException('No role in this tenant');
    }
    
    // Если restaurantIds пустой — пользователь имеет доступ ко всем ресторанам
    if (userTenantRole.restaurantIds.length === 0) {
      return true;
    }
    
    if (!userTenantRole.restaurantIds.includes(restaurantId)) {
      throw new ForbiddenException('No access to this restaurant');
    }
    
    return true;
  }
}
```

---

## 8. Cashier PIN-код

Кассиры используют короткий PIN вместо пароля для быстрого входа на POS-терминале.

```typescript
// packages/database/prisma/schema.prisma (добавление к User)

model User {
  // ... existing fields

  // Cashier PIN
  pinHash        String?   // bcrypt hash PIN-кода (4 цифры)
  pinSetAt       DateTime?
  pinFailedCount Int       @default(0)
  pinLockedUntil DateTime?
  
  // Устройство кассира
  terminalId     String?   // ID POS-терминала
  terminalName   String?   // Название терминала ("Касса 1")
}
```

```typescript
// apps/backend/src/modules/auth/services/cashier-pin.service.ts

@Injectable()
export class CashierPinService {
  private readonly PIN_MAX_ATTEMPTS = 5;
  private readonly PIN_LOCKOUT_DURATION = 15 * 60 * 1000; // 15 минут
  private readonly PIN_SESSION_DURATION = 15 * 60; // 15 минут (в секундах)

  constructor(
    private prisma: PrismaService,
    private jwtService: JwtService,
    @InjectRedis() private redis: Redis,
  ) {}

  /**
   * Установить PIN-код кассиру (Admin/Manager создаёт)
   */
  async setPin(userId: string, pin: string, setBy: string): Promise<void> {
    // Валидация: ровно 4 цифры
    if (!/^\d{4}$/.test(pin)) {
      throw new BadRequestException('PIN must be exactly 4 digits');
    }
    
    // Проверяем роль (только CASHIER)
    const user = await this.prisma.user.findUnique({ where: { id: userId } });
    if (!user || user.role !== Role.CASHIER) {
      throw new BadRequestException('PIN can only be set for Cashier role');
    }
    
    // Хэшируем PIN
    const pinHash = await bcrypt.hash(pin, 10);
    
    await this.prisma.user.update({
      where: { id: userId },
      data: {
        pinHash,
        pinSetAt: new Date(),
        pinFailedCount: 0,
        pinLockedUntil: null,
      },
    });
    
    // Лог
    await this.activityLogService.create({
      actorId: setBy,
      action: 'CASHIER_PIN_SET',
      targetId: userId,
    });
  }

  /**
   * Войти по PIN-коду
   */
  async loginByPin(
    terminalId: string,
    pin: string,
    tenantId: string,
  ): Promise<{ accessToken: string; cashier: User }> {
    // 1. Найти кассира по терминалу
    const cashier = await this.prisma.user.findFirst({
      where: {
        terminalId,
        role: Role.CASHIER,
        tenantId,
        deletedAt: null,
      },
    });
    
    if (!cashier) {
      throw new UnauthorizedException('Terminal not found or not assigned');
    }
    
    if (!cashier.pinHash) {
      throw new UnauthorizedException('PIN not set for this cashier');
    }
    
    // 2. Проверяем блокировку
    if (cashier.pinLockedUntil && cashier.pinLockedUntil > new Date()) {
      const remainingMs = cashier.pinLockedUntil.getTime() - Date.now();
      const remainingMin = Math.ceil(remainingMs / 60000);
      throw new TooManyRequestsException(
        `PIN locked. Try again in ${remainingMin} minutes`,
      );
    }
    
    // 3. Проверяем PIN
    const isPinValid = await bcrypt.compare(pin, cashier.pinHash);
    
    if (!isPinValid) {
      // Увеличиваем счётчик ошибок
      const newFailedCount = cashier.pinFailedCount + 1;
      const updateData: Partial<User> = { pinFailedCount: newFailedCount };
      
      if (newFailedCount >= this.PIN_MAX_ATTEMPTS) {
        updateData.pinLockedUntil = new Date(Date.now() + this.PIN_LOCKOUT_DURATION);
        updateData.pinFailedCount = 0;
      }
      
      await this.prisma.user.update({
        where: { id: cashier.id },
        data: updateData,
      });
      
      throw new UnauthorizedException('Invalid PIN');
    }
    
    // 4. Сбрасываем счётчик ошибок
    await this.prisma.user.update({
      where: { id: cashier.id },
      data: { pinFailedCount: 0, pinLockedUntil: null },
    });
    
    // 5. Генерируем короткий JWT (15 минут)
    const accessToken = this.jwtService.sign(
      {
        sub: cashier.id,
        role: cashier.role,
        tenantId: cashier.tenantId,
        terminalId,
        loginMethod: 'PIN',
      },
      { expiresIn: '15m' },
    );
    
    return { accessToken, cashier };
  }

  /**
   * Автоматический logout по таймауту (front-end timer)
   * Кассир должен повторно ввести PIN через 15 минут
   */
  async extendSession(cashierId: string): Promise<string> {
    const cashier = await this.prisma.user.findUnique({
      where: { id: cashierId },
    });
    
    if (!cashier || cashier.role !== Role.CASHIER) {
      throw new UnauthorizedException('Invalid cashier');
    }
    
    // Новый токен на ещё 15 минут
    return this.jwtService.sign(
      {
        sub: cashier.id,
        role: cashier.role,
        tenantId: cashier.tenantId,
        terminalId: cashier.terminalId,
        loginMethod: 'PIN',
      },
      { expiresIn: '15m' },
    );
  }
}
```

### PIN-контроллер

```typescript
// apps/backend/src/modules/auth/controllers/cashier-pin.controller.ts

@ApiTags('Cashier PIN Auth')
@Controller('auth/cashier')
export class CashierPinController {
  constructor(private cashierPinService: CashierPinService) {}

  // Вход по PIN
  @Post('pin-login')
  @ApiOperation({ summary: 'Cashier login via PIN code' })
  async pinLogin(@Body() dto: PinLoginDto) {
    return this.cashierPinService.loginByPin(
      dto.terminalId,
      dto.pin,
      dto.tenantId,
    );
  }

  // Установить PIN кассиру
  @Post('set-pin/:cashierId')
  @UseGuards(JwtAuthGuard, PermissionsGuard)
  @RequirePermissions(Permission.TEAM_EDIT_PERMISSIONS)
  @ApiOperation({ summary: 'Set PIN for cashier (Admin/Manager only)' })
  async setPin(
    @Param('cashierId') cashierId: string,
    @Body() dto: SetPinDto,
    @CurrentUser() actor: User,
  ) {
    return this.cashierPinService.setPin(cashierId, dto.pin, actor.id);
  }

  // Продлить сессию (avoid full re-login)
  @Post('extend-session')
  @UseGuards(JwtAuthGuard)
  @ApiOperation({ summary: 'Extend cashier session (15 more minutes)' })
  async extendSession(@CurrentUser() cashier: User) {
    return this.cashierPinService.extendSession(cashier.id);
  }
}
```

---

## 9. Approval Workflow (Manager → Admin)

Manager создаёт правила лояльности — они попадают в очередь на одобрение Admin.

```typescript
// packages/database/prisma/schema.prisma

model ApprovalRequest {
  id          String         @id @default(uuid())
  tenantId    String
  requestedBy String         // userId Manager
  approvedBy  String?        // userId Admin
  rejectedBy  String?
  
  type    ApprovalType   // LOYALTY_RULE, LOYALTY_PROMO, LOYALTY_LEVEL, MANUAL_ADJUST_HIGH
  status  ApprovalStatus @default(PENDING)
  
  entityType  String  // 'LoyaltyRule', 'LoyaltyPromo', etc.
  entityId    String? // ID созданной сущности (если уже создана в PENDING статусе)
  payload     Json    // Полные данные для создания/изменения
  
  reason      String? // Причина отклонения
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  reviewedAt DateTime?
  
  tenant      Tenant @relation(fields: [tenantId], references: [id])
  requester   User   @relation("ApprovalRequester", fields: [requestedBy], references: [id])
  approver    User?  @relation("ApprovalApprover", fields: [approvedBy], references: [id])
  rejector    User?  @relation("ApprovalRejector", fields: [rejectedBy], references: [id])
  
  @@index([tenantId, status])
  @@index([requestedBy])
}

enum ApprovalType {
  LOYALTY_RULE        // Создание/изменение правила лояльности
  LOYALTY_PROMO       // Создание/изменение промо-акции
  LOYALTY_LEVEL       // Создание/изменение уровня
  MANUAL_ADJUST_HIGH  // Ручная корректировка > 1000 баллов
  GUEST_DELETE        // Удаление гостя
}

enum ApprovalStatus {
  PENDING   // Ожидает одобрения
  APPROVED  // Одобрено
  REJECTED  // Отклонено
  EXPIRED   // Истёк срок (72 часа)
}
```

### ApprovalService

```typescript
// apps/backend/src/modules/rbac/services/approval.service.ts

@Injectable()
export class ApprovalService {
  private readonly APPROVAL_TTL = 72 * 60 * 60 * 1000; // 72 часа

  constructor(
    private prisma: PrismaService,
    private notificationService: NotificationService,
    private permissionsService: PermissionsService,
  ) {}

  /**
   * Manager создаёт запрос на одобрение
   */
  async createApprovalRequest(
    tenantId: string,
    requestedBy: string,
    type: ApprovalType,
    entityType: string,
    payload: Record<string, unknown>,
  ): Promise<ApprovalRequest> {
    const request = await this.prisma.approvalRequest.create({
      data: {
        tenantId,
        requestedBy,
        type,
        entityType,
        payload,
        status: ApprovalStatus.PENDING,
      },
    });
    
    // Уведомляем всех Admin'ов tenant'а
    const admins = await this.prisma.userTenantRole.findMany({
      where: { tenantId, role: Role.ADMIN },
      include: { user: true },
    });
    
    for (const adminRole of admins) {
      await this.notificationService.send({
        userId: adminRole.userId,
        type: 'APPROVAL_REQUESTED',
        title: '📋 Новый запрос на одобрение',
        body: `Manager запросил одобрение: ${type}`,
        data: { approvalId: request.id, type },
      });
    }
    
    return request;
  }

  /**
   * Admin одобряет запрос
   */
  async approveRequest(
    requestId: string,
    approvedBy: string,
    tenantId: string,
  ): Promise<void> {
    const request = await this.prisma.approvalRequest.findUnique({
      where: { id: requestId },
    });
    
    if (!request || request.tenantId !== tenantId) {
      throw new NotFoundException('Approval request not found');
    }
    
    if (request.status !== ApprovalStatus.PENDING) {
      throw new BadRequestException('Request is not in PENDING status');
    }
    
    // Проверяем, что одобряющий — Admin
    const isAdmin = await this.permissionsService.hasPermission(
      approvedBy,
      tenantId,
      Permission.APPROVALS_MANAGE,
    );
    
    if (!isAdmin) {
      throw new ForbiddenException('Only Admin can approve requests');
    }
    
    // Применяем изменение (создаём/изменяем сущность)
    await this.applyApprovedAction(request);
    
    // Обновляем статус
    await this.prisma.approvalRequest.update({
      where: { id: requestId },
      data: {
        status: ApprovalStatus.APPROVED,
        approvedBy,
        reviewedAt: new Date(),
      },
    });
    
    // Уведомляем Manager
    await this.notificationService.send({
      userId: request.requestedBy,
      type: 'APPROVAL_APPROVED',
      title: '✅ Запрос одобрен',
      body: `Ваш запрос "${request.type}" был одобрен администратором`,
      data: { approvalId: requestId },
    });
  }

  /**
   * Admin отклоняет запрос
   */
  async rejectRequest(
    requestId: string,
    rejectedBy: string,
    tenantId: string,
    reason: string,
  ): Promise<void> {
    await this.prisma.approvalRequest.update({
      where: { id: requestId },
      data: {
        status: ApprovalStatus.REJECTED,
        rejectedBy,
        reason,
        reviewedAt: new Date(),
      },
    });
    
    const request = await this.prisma.approvalRequest.findUnique({
      where: { id: requestId },
    });
    
    // Уведомляем Manager
    await this.notificationService.send({
      userId: request!.requestedBy,
      type: 'APPROVAL_REJECTED',
      title: '❌ Запрос отклонён',
      body: `Причина: ${reason}`,
      data: { approvalId: requestId, reason },
    });
  }

  /**
   * Применить одобренное действие
   */
  private async applyApprovedAction(request: ApprovalRequest): Promise<void> {
    const payload = request.payload as Record<string, unknown>;
    
    switch (request.type) {
      case ApprovalType.LOYALTY_RULE:
        await this.prisma.loyaltyRule.upsert({
          where: { id: (payload.id as string) ?? 'new' },
          create: { ...payload, tenantId: request.tenantId, status: 'ACTIVE' },
          update: { ...payload, status: 'ACTIVE' },
        });
        break;
        
      case ApprovalType.LOYALTY_PROMO:
        await this.prisma.loyaltyPromo.upsert({
          where: { id: (payload.id as string) ?? 'new' },
          create: { ...payload, tenantId: request.tenantId, status: 'ACTIVE' },
          update: { ...payload, status: 'ACTIVE' },
        });
        break;
        
      case ApprovalType.MANUAL_ADJUST_HIGH:
        // Применяем ручную корректировку
        await this.loyaltyService.applyManualAdjustment(payload);
        break;
        
      default:
        throw new Error(`Unknown approval type: ${request.type}`);
    }
  }

  /**
   * CRON: Истекать старые запросы (72 часа)
   */
  @Cron('0 */6 * * *') // Каждые 6 часов
  async expireOldRequests(): Promise<void> {
    const expireThreshold = new Date(Date.now() - this.APPROVAL_TTL);
    
    await this.prisma.approvalRequest.updateMany({
      where: {
        status: ApprovalStatus.PENDING,
        createdAt: { lt: expireThreshold },
      },
      data: { status: ApprovalStatus.EXPIRED },
    });
  }
}
```

---

## 10. Кэш прав Redis (5 мин TTL)

```typescript
// Структура ключей Redis для прав

// permissions:{userId}:{tenantId} → JSON массив Permission[]
// TTL: 300 секунд (5 минут)

// Инвалидация кэша происходит при:
// 1. Изменении роли пользователя
// 2. Изменении кастомных прав
// 3. Удалении пользователя из tenant
// 4. Ручном триггере (через API)

// apps/backend/src/modules/rbac/services/permissions.service.ts (дополнение)

/**
 * Инвалидировать кэш для всех пользователей tenant (при глобальных изменениях)
 */
async invalidateTenantCache(tenantId: string): Promise<void> {
  const pattern = `permissions:*:${tenantId}`;
  const keys = await this.redis.keys(pattern);
  
  if (keys.length > 0) {
    await this.redis.del(...keys);
  }
}

/**
 * Прогреть кэш для всех активных пользователей tenant
 */
async warmupCache(tenantId: string): Promise<void> {
  const users = await this.prisma.userTenantRole.findMany({
    where: { tenantId },
    select: { userId: true },
  });
  
  await Promise.all(
    users.map((u) => this.getUserPermissions(u.userId, tenantId)),
  );
}

// Метрика для мониторинга
async getCacheHitRate(): Promise<number> {
  const hits = await this.redis.get('metrics:permissions:cache_hits') ?? '0';
  const misses = await this.redis.get('metrics:permissions:cache_misses') ?? '0';
  const total = parseInt(hits) + parseInt(misses);
  return total === 0 ? 0 : (parseInt(hits) / total) * 100;
}
```

---

## 11. Prisma Schema — RBAC сущности

```prisma
// packages/database/prisma/schema.prisma

// Роли и права
enum Role {
  OWNER
  ADMIN
  MANAGER
  CASHIER
  GUEST
}

// Связь пользователя с tenant и ролью
model UserTenantRole {
  id            String   @id @default(uuid())
  userId        String
  tenantId      String
  role          Role
  restaurantIds String[] // Ограничение по ресторанам ([] = все)

  customPermissions CustomPermission?

  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  tenant Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@unique([userId, tenantId])
  @@index([tenantId, role])
}

// Кастомные права пользователя (добавление/удаление к базовой роли)
model CustomPermission {
  id               String   @id @default(uuid())
  userTenantRoleId String   @unique

  addedPermissions   String[] // Permission enum values
  removedPermissions String[] // Permission enum values

  changedBy String
  reason    String?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  userTenantRole UserTenantRole @relation(
    fields: [userTenantRoleId],
    references: [id],
    onDelete: Cascade
  )
}

// Запросы на одобрение (Manager → Admin)
model ApprovalRequest {
  id          String         @id @default(uuid())
  tenantId    String
  requestedBy String
  approvedBy  String?
  rejectedBy  String?

  type    ApprovalType
  status  ApprovalStatus @default(PENDING)

  entityType String
  entityId   String?
  payload    Json

  reason     String?
  reviewedAt DateTime?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  tenant    Tenant @relation(fields: [tenantId], references: [id])
  requester User   @relation("Requester", fields: [requestedBy], references: [id])
  approver  User?  @relation("Approver", fields: [approvedBy], references: [id])

  @@index([tenantId, status])
  @@index([requestedBy])
}

enum ApprovalType {
  LOYALTY_RULE
  LOYALTY_PROMO
  LOYALTY_LEVEL
  MANUAL_ADJUST_HIGH
  GUEST_DELETE
}

enum ApprovalStatus {
  PENDING
  APPROVED
  REJECTED
  EXPIRED
}
```

---

## 12. Декораторы и Guards

```typescript
// apps/backend/src/modules/rbac/decorators/require-permissions.decorator.ts

export const PERMISSIONS_KEY = 'permissions';

export const RequirePermissions = (...permissions: Permission[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);

// ─────────────────────────────────────────────────────────────────────────────

// apps/backend/src/modules/rbac/decorators/roles.decorator.ts

export const ROLES_KEY = 'roles';

export const Roles = (...roles: Role[]) =>
  SetMetadata(ROLES_KEY, roles);

// ─────────────────────────────────────────────────────────────────────────────

// apps/backend/src/modules/rbac/decorators/require-approval.decorator.ts

export const REQUIRE_APPROVAL_KEY = 'require_approval';

/**
 * Декоратор для операций, требующих одобрения Admin (если исполнитель — Manager)
 */
export const RequireApproval = (type: ApprovalType) =>
  SetMetadata(REQUIRE_APPROVAL_KEY, type);

// ─────────────────────────────────────────────────────────────────────────────

// apps/backend/src/modules/rbac/guards/approval-workflow.guard.ts

@Injectable()
export class ApprovalWorkflowGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private approvalService: ApprovalService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const approvalType = this.reflector.getAllAndOverride<ApprovalType>(
      REQUIRE_APPROVAL_KEY,
      [context.getHandler(), context.getClass()],
    );
    
    if (!approvalType) return true;
    
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    // Admin и Owner — без workflow
    if (user.role === Role.ADMIN || user.role === Role.OWNER) {
      return true;
    }
    
    // Manager — создаём approval request и блокируем выполнение
    if (user.role === Role.MANAGER) {
      await this.approvalService.createApprovalRequest(
        user.tenantId,
        user.id,
        approvalType,
        request.path,
        request.body,
      );
      
      throw new HttpException(
        {
          statusCode: 202,
          message: 'Approval request submitted. Awaiting Admin approval.',
          approvalPending: true,
        },
        202, // Accepted
      );
    }
    
    return true;
  }
}
```

### Пример использования в контроллере

```typescript
// apps/backend/src/modules/loyalty/controllers/loyalty-rules.controller.ts

@ApiTags('Loyalty Rules')
@Controller('loyalty/rules')
@UseGuards(JwtAuthGuard, PermissionsGuard, RestaurantAccessGuard)
@ApiBearerAuth()
export class LoyaltyRulesController {

  // Просмотр правил — Manager может
  @Get()
  @RequirePermissions(Permission.LOYALTY_VIEW)
  async findAll(@CurrentUser() user: User) {
    return this.loyaltyRulesService.findAll(user.tenantId);
  }

  // Создание правила — Manager создаёт, но Admin должен одобрить
  @Post()
  @RequirePermissions(Permission.LOYALTY_VIEW) // Manager имеет LOYALTY_VIEW
  @UseGuards(ApprovalWorkflowGuard)
  @RequireApproval(ApprovalType.LOYALTY_RULE)
  @ApiResponse({ status: 201, description: 'Rule created (Admin only)' })
  @ApiResponse({ status: 202, description: 'Approval request submitted (Manager)' })
  async create(@Body() dto: CreateLoyaltyRuleDto, @CurrentUser() user: User) {
    // Сюда попадёт только Admin/Owner
    return this.loyaltyRulesService.create(dto, user.tenantId, user.id);
  }

  // Ручная корректировка > 1000 баллов → требует одобрения
  @Post('manual-adjust')
  @RequirePermissions(Permission.LOYALTY_MANUAL_ADJUST)
  @UseGuards(ApprovalWorkflowGuard)
  @RequireApproval(ApprovalType.MANUAL_ADJUST_HIGH)
  async manualAdjust(
    @Body() dto: ManualAdjustDto,
    @CurrentUser() user: User,
  ) {
    return this.loyaltyService.manualAdjust(dto, user.id);
  }

  // Удаление — только Admin
  @Delete(':id')
  @RequirePermissions(Permission.LOYALTY_CONFIGURE)
  async remove(@Param('id') id: string, @CurrentUser() user: User) {
    return this.loyaltyRulesService.remove(id, user.tenantId);
  }
}
```

---

## 13. API Endpoints

### RBAC Management API

```typescript
// apps/backend/src/modules/rbac/controllers/rbac.controller.ts

@ApiTags('RBAC Management')
@Controller('admin/rbac')
@UseGuards(JwtAuthGuard, PermissionsGuard)
@ApiBearerAuth()
export class RbacController {

  // ── PERMISSIONS ───────────────────────────────────────────────────────────

  /** GET /admin/rbac/permissions-matrix
   * Полная матрица прав (роль → список прав)
   */
  @Get('permissions-matrix')
  @RequirePermissions(Permission.TEAM_VIEW)
  async getPermissionsMatrix() {
    return {
      roles: Object.keys(Role),
      permissions: Object.values(Permission),
      matrix: Object.entries(ROLE_PERMISSIONS).map(([role, perms]) => ({
        role,
        permissions: perms,
      })),
    };
  }

  /** GET /admin/rbac/users/:userId/permissions
   * Итоговые права конкретного пользователя
   */
  @Get('users/:userId/permissions')
  @RequirePermissions(Permission.TEAM_VIEW)
  async getUserPermissions(
    @Param('userId') userId: string,
    @CurrentUser() actor: User,
  ) {
    const permissions = await this.permissionsService.getUserPermissions(
      userId,
      actor.tenantId,
    );
    return { userId, permissions };
  }

  /** PATCH /admin/rbac/users/:userId/permissions
   * Изменить кастомные права пользователя
   */
  @Patch('users/:userId/permissions')
  @RequirePermissions(Permission.TEAM_EDIT_PERMISSIONS)
  async updateUserPermissions(
    @Param('userId') userId: string,
    @Body() dto: UpdatePermissionsDto,
    @CurrentUser() actor: User,
  ) {
    await this.permissionsService.updateCustomPermissions(
      userId,
      actor.tenantId,
      dto.addedPermissions,
      dto.removedPermissions,
      actor.id,
      dto.reason,
    );
    return { success: true };
  }

  /** GET /admin/rbac/users/:userId/role
   * Роль пользователя в tenant
   */
  @Get('users/:userId/role')
  @RequirePermissions(Permission.TEAM_VIEW)
  async getUserRole(
    @Param('userId') userId: string,
    @CurrentUser() actor: User,
  ) {
    return this.rbacService.getUserRole(userId, actor.tenantId);
  }

  /** PATCH /admin/rbac/users/:userId/role
   * Изменить роль пользователя (только Admin)
   */
  @Patch('users/:userId/role')
  @RequirePermissions(Permission.TEAM_EDIT_PERMISSIONS)
  async updateUserRole(
    @Param('userId') userId: string,
    @Body() dto: UpdateRoleDto,
    @CurrentUser() actor: User,
  ) {
    return this.rbacService.updateUserRole(
      userId,
      actor.tenantId,
      dto.role,
      actor.id,
    );
  }

  // ── APPROVAL WORKFLOW ─────────────────────────────────────────────────────

  /** GET /admin/rbac/approvals
   * Список ожидающих одобрения
   */
  @Get('approvals')
  @RequirePermissions(Permission.APPROVALS_MANAGE)
  async getPendingApprovals(@CurrentUser() actor: User) {
    return this.approvalService.getPendingApprovals(actor.tenantId);
  }

  /** POST /admin/rbac/approvals/:id/approve
   * Одобрить запрос
   */
  @Post('approvals/:id/approve')
  @RequirePermissions(Permission.APPROVALS_MANAGE)
  async approveRequest(
    @Param('id') requestId: string,
    @CurrentUser() actor: User,
  ) {
    await this.approvalService.approveRequest(
      requestId,
      actor.id,
      actor.tenantId,
    );
    return { success: true };
  }

  /** POST /admin/rbac/approvals/:id/reject
   * Отклонить запрос
   */
  @Post('approvals/:id/reject')
  @RequirePermissions(Permission.APPROVALS_MANAGE)
  async rejectRequest(
    @Param('id') requestId: string,
    @Body() dto: RejectApprovalDto,
    @CurrentUser() actor: User,
  ) {
    await this.approvalService.rejectRequest(
      requestId,
      actor.id,
      actor.tenantId,
      dto.reason,
    );
    return { success: true };
  }

  // ── CASHIER PIN ───────────────────────────────────────────────────────────

  /** POST /admin/rbac/cashiers/:id/set-pin
   * Установить PIN кассиру
   */
  @Post('cashiers/:id/set-pin')
  @RequirePermissions(Permission.TEAM_EDIT_PERMISSIONS)
  async setCashierPin(
    @Param('id') cashierId: string,
    @Body() dto: SetPinDto,
    @CurrentUser() actor: User,
  ) {
    return this.cashierPinService.setPin(cashierId, dto.pin, actor.id);
  }
}
```

---

## 14. Матрица прав (сводная таблица)

| Permission | Owner | Admin | Manager | Cashier | Guest |
|-----------|:-----:|:-----:|:-------:|:-------:|:-----:|
| `guests:view` | ✅ | ✅ | ✅ | ✅ | — |
| `guests:create` | ✅ | ✅ | ✅ | ✅ | — |
| `guests:update` | ✅ | ✅ | ✅ | — | — |
| `guests:delete` | ✅ | ✅ | — | — | — |
| `guests:export` | ✅ | ✅ | — | — | — |
| `loyalty:view` | ✅ | ✅ | ✅ | ✅ | ✅ (своя) |
| `loyalty:configure` | ✅ | ✅ | ⏳¹ | — | — |
| `loyalty:manual_adjust` | ✅ | ✅ | ✅² | — | — |
| `loyalty:transactions_view` | ✅ | ✅ | ✅ | ✅ | ✅ (своя) |
| `loyalty:promo_activate` | ✅ | ✅ | — | — | — |
| `analytics:view` | ✅ | ✅ | ✅ | — | — |
| `analytics:export` | ✅ | ✅ | — | — | — |
| `analytics:advanced` | ✅ | ✅ | — | — | — |
| `billing:view` | ✅ | ✅ | — | — | — |
| `billing:manage` | ✅ | ✅ | — | — | — |
| `team:view` | ✅ | ✅ | ✅ | — | — |
| `team:invite` | ✅ | ✅ | — | — | — |
| `team:remove` | ✅ | ✅ | — | — | — |
| `team:edit_permissions` | ✅ | ✅ | — | — | — |
| `settings:view` | ✅ | ✅ | ✅ | — | — |
| `settings:edit` | ✅ | ✅ | — | — | — |
| `pos:view` | ✅ | ✅ | ✅ | — | — |
| `pos:configure` | ✅ | ✅ | — | — | — |
| `restaurants:view` | ✅ | ✅ | ✅ | ✅ | — |
| `restaurants:manage` | ✅ | ✅ | — | — | — |
| `approvals:manage` | ✅ | ✅ | — | — | — |
| `admin:impersonate` | ✅ | — | — | — | — |
| `admin:view_all_tenants` | ✅ | — | — | — | — |

> ¹ ⏳ = требует одобрения Admin (ApprovalWorkflow)
> ² Manager может делать корректировку ≤ 1000 баллов; > 1000 → ApprovalWorkflow

---

## 15. Тесты RBAC

### Unit тесты

```typescript
// apps/backend/src/modules/rbac/tests/permissions.service.spec.ts

describe('PermissionsService', () => {
  describe('getUserPermissions', () => {
    it('should return cached permissions on second call', async () => {
      // Первый вызов — из БД
      const perms1 = await service.getUserPermissions(userId, tenantId);
      // Второй вызов — из Redis
      const perms2 = await service.getUserPermissions(userId, tenantId);
      expect(mockPrisma.userTenantRole.findUnique).toHaveBeenCalledTimes(1);
      expect(perms1).toEqual(perms2);
    });

    it('should apply added custom permissions', async () => {
      mockPrisma.userTenantRole.findUnique.mockResolvedValue({
        role: Role.MANAGER,
        customPermissions: {
          addedPermissions: [Permission.GUESTS_EXPORT],
          removedPermissions: [],
        },
      });
      const perms = await service.getUserPermissions(userId, tenantId);
      expect(perms).toContain(Permission.GUESTS_EXPORT);
    });

    it('should remove revoked permissions', async () => {
      mockPrisma.userTenantRole.findUnique.mockResolvedValue({
        role: Role.MANAGER,
        customPermissions: {
          addedPermissions: [],
          removedPermissions: [Permission.LOYALTY_MANUAL_ADJUST],
        },
      });
      const perms = await service.getUserPermissions(userId, tenantId);
      expect(perms).not.toContain(Permission.LOYALTY_MANUAL_ADJUST);
    });

    it('should return all permissions for OWNER', async () => {
      // Owner всегда проходит — в PermissionsGuard проверяется role
      const guard = new PermissionsGuard(reflector, permissionsService);
      mockUser.role = Role.OWNER;
      const result = await guard.canActivate(mockContext);
      expect(result).toBe(true);
      expect(permissionsService.getUserPermissions).not.toHaveBeenCalled();
    });
  });

  describe('PermissionsGuard', () => {
    it('should throw ForbiddenException for missing permissions', async () => {
      mockUser.role = Role.CASHIER;
      reflector.getAllAndOverride.mockReturnValue([Permission.BILLING_MANAGE]);
      await expect(guard.canActivate(context)).rejects.toThrow(ForbiddenException);
    });

    it('should pass if no permissions required', async () => {
      reflector.getAllAndOverride.mockReturnValue([]);
      const result = await guard.canActivate(context);
      expect(result).toBe(true);
    });
  });
});
```

### Таблица тест-кейсов

| # | Сценарий | Ожидаемый результат |
|---|----------|---------------------|
| 1 | Cashier пытается удалить гостя | 403 Forbidden |
| 2 | Manager создаёт правило лояльности | 202 Accepted (pending approval) |
| 3 | Admin одобряет правило Manager'а | 200 OK, правило активно |
| 4 | Manager с кастомным правом `guests:export` | 200 OK, экспорт доступен |
| 5 | Cashier с убранным правом `guests:create` | 403 Forbidden |
| 6 | Owner имперсонирует Admin | 200 OK |
| 7 | Manager пытается изменить биллинг | 403 Forbidden |
| 8 | Cashier входит по PIN | 200 OK, токен 15 мин |
| 9 | Неверный PIN 5 раз подряд | 429 Too Many Requests, блокировка 15 мин |
| 10 | Cross-tenant запрос (guest из другого tenant) | 403 Forbidden |
| 11 | Manager пытается одобрить свой же запрос | 403 Forbidden |
| 12 | Admin инвалидирует кэш прав пользователя | Новые права применяются мгновенно |

---

## 16. Чеклист реализации

| Задача | Приоритет | Статус |
|--------|-----------|--------|
| `Permission` enum (30 прав) | P0 | ☐ |
| `ROLE_PERMISSIONS` маппинг | P0 | ☐ |
| `PermissionsGuard` | P0 | ☐ |
| `PermissionsService` с Redis кэшем | P0 | ☐ |
| `UserTenantRole` Prisma model | P0 | ☐ |
| `CustomPermission` Prisma model | P0 | ☐ |
| `RolesGuard` | P0 | ☐ |
| `ResourceOwnershipGuard` | P0 | ☐ |
| `RestaurantAccessGuard` (ABAC) | P1 | ☐ |
| `ApprovalRequest` Prisma model | P1 | ☐ |
| `ApprovalService` | P1 | ☐ |
| `ApprovalWorkflowGuard` | P1 | ☐ |
| Cashier PIN (hash, login, lockout) | P1 | ☐ |
| RBAC API endpoints (10 эндпоинтов) | P1 | ☐ |
| Approval API endpoints (4 эндпоинта) | P1 | ☐ |
| `@RequirePermissions` декоратор | P0 | ☐ |
| `@RequireApproval` декоратор | P1 | ☐ |
| `@Roles` декоратор | P0 | ☐ |
| Redis кэш инвалидация | P1 | ☐ |
| Unit тесты (12 сценариев) | P2 | ☐ |
| E2E тесты RBAC | P2 | ☐ |

---

> **Следующая часть:** [04 — Tenant & Restaurant Management →](./04-TENANT-RESTAURANT.md)
