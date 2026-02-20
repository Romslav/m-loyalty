# 02 — Auth System: Authentication, Security & Session Management

> **Версия:** 1.0  
> **Дата:** 2026-02-20  
> **Статус:** Approved  
> **Источник:** Beseda-1, Beseda-3, Beseda-13 (Security & Compliance)

---

## Содержание

1. [Обзор системы аутентификации](#1-обзор-системы-аутентификации)
2. [JWT-токены и конфигурация](#2-jwt-токены-и-конфигурация)
3. [Политика паролей](#3-политика-паролей)
4. [Многофакторная аутентификация (MFA/TOTP)](#4-многофакторная-аутентификация-mfatotp)
5. [Управление сессиями](#5-управление-сессиями)
6. [Сброс пароля](#6-сброс-пароля)
7. [Impersonation (режим владельца)](#7-impersonation-режим-владельца)
8. [Rate Limiting](#8-rate-limiting)
9. [Guards, Decorators, Interceptors](#9-guards-decorators-interceptors)
10. [Flows: Registration, Login, Logout](#10-flows-registration-login-logout)
11. [Database entities (Auth)](#11-database-entities-auth)
12. [Security Testing](#12-security-testing)
13. [Чеклист реализации](#13-чеклист-реализации)

---

## 1. Обзор системы аутентификации

### 1.1 Стек и принципы

| Компонент | Технология | Назначение |
|---|---|---|
| JWT Access Token | RS256, 5 мин | Авторизация запросов |
| JWT Refresh Token | RS256, 14 дней | Выдача нового access token |
| Refresh Token Rotation | Redis + Prisma | Защита от кражи токена |
| Хеширование паролей | argon2id | Защита от GPU/ASIC атак |
| MFA | TOTP (speakeasy) + SMS | 2FA для Owner/Admin |
| Session Storage | PostgreSQL (UserSession) | Хранение активных сессий |
| Rate Limiting | rate-limiter-flexible + Redis | Защита от brute-force |
| Blacklist | Redis | Инвалидация токенов при logout |

### 1.2 Принципы безопасности

- **RS256 вместо HS256** — приватный ключ только на backend, публичный ключ можно безопасно распределять
- **Short-lived access tokens** — 5 минут минимизируют окно компрометации
- **Refresh Token Rotation** — каждое использование refresh token выдаёт новый; повторное использование = признак кражи
- **Family tracking** — цепочка refresh токенов; при обнаружении reuse — отзыв всей цепочки
- **HttpOnly cookies** для refresh token — защита от XSS
- **argon2id** — победитель Password Hashing Competition, рекомендован OWASP

### 1.3 Архитектура Auth-модуля

```
apps/backend/src/
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts          # POST /auth/*
│   ├── auth.service.ts             # Register, Login, Refresh, Reset, Logout
│   ├── services/
│   │   ├── jwt.service.ts          # Генерация/валидация JWT
│   │   ├── password.service.ts     # argon2id хеширование
│   │   ├── rate-limit.service.ts   # Redis rate limiting
│   │   ├── session.service.ts      # Управление сессиями, max 5
│   │   ├── token-blacklist.service.ts  # Redis blacklist
│   │   ├── email-verification.service.ts
│   │   ├── mfa.service.ts          # TOTP + backup codes
│   │   └── activity-log.service.ts # Аудит
│   ├── guards/
│   │   ├── jwt-auth.guard.ts       # Проверка JWT + blacklist
│   │   ├── tenant.guard.ts         # Tenant access control
│   │   ├── role.guard.ts           # RBAC
│   │   ├── mfa.guard.ts            # Проверка MFA verified
│   │   ├── step-up-auth.guard.ts   # Step-Up для критических операций
│   │   └── rate-limit.guard.ts     # Rate limit по endpoint
│   ├── decorators/
│   │   ├── current-user.decorator.ts
│   │   ├── roles.decorator.ts
│   │   ├── require-permissions.decorator.ts
│   │   └── public.decorator.ts
│   ├── interceptors/
│   │   ├── impersonation-audit.interceptor.ts
│   │   └── structured-logging.interceptor.ts
│   └── dto/
│       ├── register.dto.ts
│       ├── login.dto.ts            # email + phone
│       ├── refresh-token.dto.ts
│       └── password-reset.dto.ts
```

---

## 2. JWT-токены и конфигурация

### 2.1 Конфигурация

```typescript
// JWT Configuration
const JWT_CONFIG = {
  accessToken: {
    algorithm: 'RS256',
    expiresIn: '5m',          // 5 минут
    issuer: 'max-loyalty.com',
    audience: 'max-loyalty-api',
  },
  refreshToken: {
    expiresIn: '14d',         // 14 дней
    rotation: true,
    reuseDetection: true,
    familyTracking: true,     // цепочка refresh — при reuse: revoke chain
  },
};
```

### 2.2 Payload Access Token

```typescript
interface JwtAccessPayload {
  sub: string;          // userId (UUID)
  email: string;
  role: Role;           // OWNER | ADMIN | MANAGER | CASHIER | GUEST
  tenantId: string;     // текущий tenant
  tenantIds: string[];  // все доступные tenants (для Owner)
  permissions: Permission[];
  // Impersonation fields (если активна)
  impersonatorId?: string;
  sessionId?: string;
  iat: number;
  exp: number;
  iss: string;          // 'max-loyalty.com'
  aud: string;          // 'max-loyalty-api'
}
```

### 2.3 Refresh Token Rotation Logic

```typescript
// auth/services/jwt.service.ts
async refreshAccessToken(oldRefreshToken: string) {
  // 1. Validate old refresh token
  const session = await this.validateRefreshToken(oldRefreshToken);

  // 2. Check reuse — признак кражи токена
  if (session.refreshTokenReused) {
    // Отозвать всю цепочку (family revocation)
    await this.revokeAllUserSessions(session.userId);
    throw new UnauthorizedException('Token reuse detected');
  }

  // 3. Mark old token as used
  await this.prisma.userSession.update({
    where: { refreshToken: oldRefreshToken },
    data: { refreshTokenReused: true },
  });

  // 4. Issue new pair (access + refresh)
  const newAccessToken = this.generateAccessToken(session.userId);
  const newRefreshToken = this.generateRefreshToken();

  // 5. Create new session linked to old one (family chain)
  await this.prisma.userSession.create({
    data: {
      userId: session.userId,
      refreshToken: newRefreshToken,
      parentSessionId: session.id,  // Chain
      expiresAt: addDays(new Date(), 14),
    },
  });

  return { newAccessToken, newRefreshToken };
}
```

### 2.4 Token Blacklisting (Logout)

При logout access token добавляется в Redis blacklist со временем жизни = оставшееся время до exp токена.

```typescript
// JwtAuthGuard проверяет blacklist при каждом запросе
async canActivate(context: ExecutionContext): Promise<boolean> {
  const token = this.extractToken(context);
  
  // Проверка blacklist
  const isBlacklisted = await this.redis.get(`blacklist:${token}`);
  if (isBlacklisted) throw new UnauthorizedException('Token blacklisted');
  
  // Валидация подписи + claims
  const payload = this.jwtService.verify(token, { algorithms: ['RS256'] });
  // ...
}
```

---

## 3. Политика паролей

### 3.1 Конфигурация

```typescript
const PASSWORD_POLICY = {
  minLength: 10,
  maxLength: 128,
  requireUppercase: true,
  requireLowercase: true,
  requireDigit: true,
  requireSpecial: true,
  minEntropy: 50,           // bits (zxcvbn)
  historyCheck: 5,          // нельзя переиспользовать последние 5 паролей
  maxAgeDays: 90,           // принудительная смена через 90 дней
  lockoutThreshold: 5,      // блокировка после 5 неверных попыток
  lockoutDurationMinutes: 15,
};
```

### 3.2 Хеширование — argon2id

```typescript
import * as argon2 from 'argon2';

async hashPassword(password: string): Promise<string> {
  // Validate policy first
  this.validatePasswordPolicy(password);
  
  // Hash with argon2id — OWASP recommended
  return argon2.hash(password, {
    type: argon2.argon2id,   // Resistant to GPU/ASIC attacks
    memoryCost: 65536,        // 64 MB
    timeCost: 3,              // Iterations
    parallelism: 4,           // Threads
  });
}

// Password history check
async canReusePassword(userId: string, newPassword: string): Promise<boolean> {
  const history = await this.prisma.passwordHistory.findMany({
    where: { userId },
    orderBy: { createdAt: 'desc' },
    take: 5,
  });

  for (const record of history) {
    if (await argon2.verify(record.passwordHash, newPassword)) {
      return false; // Password was used recently
    }
  }
  return true;
}
```

> **Почему argon2id?**  
> argon2id — победитель Password Hashing Competition (2015). Устойчив к GPU и ASIC атакам благодаря высокому потреблению памяти. PCI DSS compliance.

---

## 4. Многофакторная аутентификация (MFA/TOTP)

### 4.1 MFA Policy по ролям

```typescript
const MFA_POLICY = {
  owner: {
    required: true,
    methods: ['TOTP', 'SMS'],   // TOTP — Authenticator app, SMS — backup
    gracePeriodDays: 7,          // 7 дней после регистрации
  },
  admin: {
    required: true,
    methods: ['TOTP', 'SMS'],
    gracePeriodDays: 7,
  },
  manager: {
    required: false,
    methods: ['TOTP', 'SMS'],
    recommended: true,           // Рекомендуем в UI
  },
  cashier: {
    required: false,
    methods: ['SMS'],
  },
};
```

### 4.2 Step-Up Authentication

Критические операции требуют повторной верификации MFA даже при активной сессии:

```typescript
const STEP_UP_REQUIRED_OPERATIONS = [
  'impersonateUser',
  'changeSubscriptionPlan',
  'manualBallAdjustmentOverThreshold', // > 10,000 баллов
  'deleteTenant',
  'exportAllGuestsData',
  'changePosCredentials',
];

@UseGuards(StepUpAuthGuard)
@Post('admin/guests/:id/manual-adjustment')
async manualAdjustment(@Body() dto: ManualAdjustmentDto) {
  // StepUpAuthGuard:
  // 1. Была ли MFA-верификация в последние 10 минут?
  // 2. Если нет — 403 с подсказкой для frontend: prompt MFA
}
```

### 4.3 TOTP Implementation

```typescript
import * as speakeasy from 'speakeasy';
import * as QRCode from 'qrcode';

// Setup TOTP
async setupTOTP(userId: string) {
  const secret = speakeasy.generateSecret({
    name: 'Max Loyalty',
    issuer: 'Max Loyalty',
    length: 32,
  });

  // Save secret encrypted (AES-256-GCM)
  await this.prisma.user.update({
    where: { id: userId },
    data: {
      mfaSecret: this.encrypt(secret.base32),
      mfaEnabled: false, // true — только после верификации
    },
  });

  // Generate QR code for Authenticator app
  const qrCodeUrl = await QRCode.toDataURL(secret.otpauth_url);

  return {
    secret: secret.base32,
    qrCode: qrCodeUrl,         // Scan in Google Authenticator / Authy
    backupCodes: this.generateBackupCodes(10),
  };
}

// Verify TOTP
async verifyTOTP(userId: string, token: string): Promise<boolean> {
  const user = await this.prisma.user.findUnique({ where: { id: userId } });
  const secret = this.decrypt(user.mfaSecret);

  return speakeasy.totp.verify({
    secret,
    encoding: 'base32',
    token,
    window: 1, // ±30 секунд clock skew
  });
}
```

---

## 5. Управление сессиями

### 5.1 Session Policy

```typescript
const SESSION_POLICY = {
  maxConcurrentSessions: 5,       // максимум 5 активных сессий
  idleTimeoutMinutes: 30,          // logout при 30 мин неактивности
  absoluteTimeoutHours: 12,        // re-login через 12 часов
  trackDeviceFingerprint: true,
  trackIp: true,
  suspiciousActivityCheck: true,
};
```

### 5.2 Session Creation с лимитом

```typescript
async createSession(userId: string, deviceInfo: DeviceInfo) {
  // 1. Count active sessions
  const activeSessions = await this.prisma.userSession.count({
    where: { userId, isBlacklisted: false, expiresAt: { gt: new Date() } },
  });

  // 2. If limit exceeded — revoke oldest
  if (activeSessions >= SESSION_POLICY.maxConcurrentSessions) {
    await this.prisma.userSession.updateMany({
      where: { userId, isBlacklisted: false },
      orderBy: { createdAt: 'asc' },
      take: 1,
      data: { isBlacklisted: true },
    });
  }

  // 3. Check suspicious activity
  const isSuspicious = await this.detectSuspiciousActivity(userId, deviceInfo);
  if (isSuspicious) {
    // Require MFA even if already logged in
    await this.sendSecurityAlert(userId, 'New device login detected');
  }

  // 4. Create session
  return this.prisma.userSession.create({
    data: {
      userId,
      refreshToken: this.generateRefreshToken(),
      ipAddress: deviceInfo.ip,
      userAgent: deviceInfo.userAgent,
      deviceFingerprint: this.generateFingerprint(deviceInfo),
      expiresAt: addHours(new Date(), 12),
      lastActivityAt: new Date(),
    },
  });
}
```

### 5.3 Suspicious Activity Detection

```typescript
async detectSuspiciousActivity(
  userId: string,
  deviceInfo: DeviceInfo,
): Promise<boolean> {
  const recentSessions = await this.prisma.userSession.findMany({
    where: { userId },
    orderBy: { createdAt: 'desc' },
    take: 10,
  });

  // Check 1: New country
  const currentCountry = await this.geolocate(deviceInfo.ip);
  const knownCountries = [...new Set(recentSessions.map(s => s.country))];
  if (!knownCountries.includes(currentCountry)) return true;

  // Check 2: New device fingerprint
  const currentFingerprint = this.generateFingerprint(deviceInfo);
  const knownFingerprints = recentSessions.map(s => s.deviceFingerprint);
  if (!knownFingerprints.includes(currentFingerprint)) return true;

  // Check 3: Impossible travel (логин из Москвы, потом через 10 мин из США)
  const lastSession = recentSessions[0];
  if (lastSession) {
    const timeDiff = Date.now() - lastSession.lastActivityAt.getTime();
    const distance = this.calculateDistance(lastSession.country, currentCountry);
    const maxSpeed = (distance / timeDiff) * 3600000; // km/h
    if (maxSpeed > 1000) return true; // Faster than airplane
  }

  return false;
}
```

### 5.4 Session Management API

```typescript
// GET /profile/sessions — список активных сессий
@Get('profile/sessions')
async listActiveSessions(@CurrentUser() user: User) {
  return this.prisma.userSession.findMany({
    where: { userId: user.id, isBlacklisted: false, expiresAt: { gt: new Date() } },
    select: {
      id: true, ipAddress: true, userAgent: true,
      country: true, city: true,
      createdAt: true, lastActivityAt: true,
      isCurrent: true,
    },
  });
}

// DELETE /profile/sessions/:id — отозвать конкретную сессию
@Delete('profile/sessions/:id')
async revokeSession(
  @CurrentUser() user: User,
  @Param('id') sessionId: string,
) {
  await this.prisma.userSession.update({
    where: { id: sessionId, userId: user.id },
    data: { isBlacklisted: true },
  });
}
```

---

## 6. Сброс пароля

### 6.1 Конфигурация

```typescript
const RESET_TOKEN_CONFIG = {
  expiresInMinutes: 15,         // TTL токена
  maxActiveTokensPerUser: 1,    // только 1 активный токен
  rateLimit: {
    maxRequests: 3,
    windowMinutes: 60,           // 3 запроса в час с одного IP
  },
};
```

### 6.2 Flow: запрос сброса

```typescript
// POST /auth/password-reset/request
@Post('auth/password-reset/request')
@UseGuards(RateLimitGuard) // 3 requests per hour per IP
async requestPasswordReset(@Body() { email }: { email: string }) {
  const user = await this.prisma.user.findUnique({ where: { email } });

  // SECURITY: всегда одинаковый ответ (не раскрываем существование email)
  if (!user) return { message: 'If email exists, reset link was sent' };

  // Инвалидировать старые токены
  await this.prisma.passwordResetToken.updateMany({
    where: { userId: user.id, usedAt: null },
    data: { usedAt: new Date() },
  });

  // Генерация криптографически безопасного токена
  const token = crypto.randomBytes(32).toString('hex');
  const hashedToken = crypto.createHash('sha256').update(token).digest('hex');

  // Сохраняем хеш, не plain token
  await this.prisma.passwordResetToken.create({
    data: {
      userId: user.id,
      token: hashedToken,
      expiresAt: addMinutes(new Date(), 15),
      ipAddress: req.ip,
    },
  });

  // Отправляем email с plain token
  await this.emailService.sendPasswordResetEmail(user.email, token);

  return { message: 'If email exists, reset link was sent' };
}
```

### 6.3 Flow: подтверждение сброса

```typescript
// POST /auth/password-reset/confirm
@Post('auth/password-reset/confirm')
async resetPassword(
  @Body('token') token: string,
  @Body('newPassword') newPassword: string,
) {
  const hashedToken = crypto.createHash('sha256').update(token).digest('hex');

  const resetToken = await this.prisma.passwordResetToken.findFirst({
    where: { token: hashedToken, usedAt: null, expiresAt: { gt: new Date() } },
    include: { user: true },
  });

  if (!resetToken) throw new BadRequestException('Invalid or expired token');

  // Проверка истории паролей
  const canReuse = await this.canReusePassword(resetToken.userId, newPassword);
  if (!canReuse) throw new BadRequestException('Cannot reuse recent passwords');

  // Обновляем пароль
  await this.prisma.user.update({
    where: { id: resetToken.userId },
    data: {
      passwordHash: await this.hashPassword(newPassword),
      passwordChangedAt: new Date(),
    },
  });

  // Mark token as used
  await this.prisma.passwordResetToken.update({
    where: { id: resetToken.id },
    data: { usedAt: new Date() },
  });

  // Отозвать все сессии — force re-login
  await this.revokeAllUserSessions(resetToken.userId);

  // Email-уведомление о смене пароля
  await this.emailService.sendPasswordChangedNotification(resetToken.user.email);

  return { message: 'Password reset successful' };
}
```

> **Ключевые принципы безопасности при сбросе:**
> - Хранить только SHA-256 хеш токена (не plain)
> - TTL 15 минут
> - Максимум 1 активный токен на пользователя
> - Rate limit: 3 запроса/час/IP
> - Одинаковый ответ (не раскрывать наличие email)
> - Отзывать все сессии после успешного сброса

---

## 7. Impersonation (режим владельца)

### 7.1 Impersonation Policy

```typescript
const IMPERSONATION_POLICY = {
  requireMfa: true,           // Обязательна MFA-верификация
  requireReason: true,        // Обязательно указать причину (min 10 символов)
  maxDurationHours: 2,        // Максимум 2 часа
  auditLogRetentionYears: 7,  // Хранение логов 7 лет (GDPR, SOC 2)
  notificationToTarget: true, // Уведомить пользователя
};
```

### 7.2 Start Impersonation

```typescript
// POST /owner/impersonate
@Post('owner/impersonate')
@Roles(Role.OWNER)
@UseGuards(MFAGuard) // Require MFA verification
async startImpersonation(
  @CurrentUser() owner: User,
  @Body() dto: ImpersonationDto,
) {
  // Validate MFA was verified in last 5 minutes
  const mfaValid = await this.validateRecentMFA(owner.id, 5);
  if (!mfaValid) throw new ForbiddenException('MFA verification required');

  // Validate reason
  if (!dto.reason || dto.reason.length < 10) {
    throw new BadRequestException('Reason required (min 10 characters)');
  }

  const targetUser = await this.prisma.user.findUnique({
    where: { id: dto.targetUserId },
  });

  // Create impersonation session
  const session = await this.prisma.impersonationSession.create({
    data: {
      impersonatorUserId: owner.id,
      impersonatedUserId: targetUser.id,
      reason: dto.reason,
      startedAt: new Date(),
      expiresAt: addHours(new Date(), 2),
      ipAddress: req.ip,
      userAgent: req.headers['user-agent'],
    },
  });

  // Log to Activity Log (immutable audit trail)
  await this.activityLog.create({
    actorId: owner.id,
    action: 'IMPERSONATION_START',
    targetId: targetUser.id,
    metadata: { reason: dto.reason, sessionId: session.id },
    severity: 'CRITICAL',
  });

  // Notify target user (Email + In-app)
  await this.notificationService.send({
    recipientId: targetUser.id,
    type: 'SECURITY_ALERT',
    priority: 'HIGH',
    title: 'Owner accessed your account',
    message: `Owner ${owner.email} accessed your account for: ${dto.reason}`,
    metadata: {
      impersonator: owner.email,
      startedAt: session.startedAt,
      expiresAt: session.expiresAt,
    },
  });

  // Generate impersonation token
  const impersonationToken = this.jwtService.sign({
    sub: targetUser.id,
    impersonatorId: owner.id,
    sessionId: session.id,
    exp: Math.floor(session.expiresAt.getTime() / 1000),
  });

  return { impersonationToken };
}
```

### 7.3 End Impersonation

```typescript
// POST /owner/impersonate/end
@Post('owner/impersonate/end')
async endImpersonation(@CurrentUser() owner: User) {
  const session = await this.prisma.impersonationSession.findFirst({
    where: { impersonatorUserId: owner.id, endedAt: null },
  });

  if (!session) throw new NotFoundException('No active impersonation session');

  await this.prisma.impersonationSession.update({
    where: { id: session.id },
    data: { endedAt: new Date() },
  });

  await this.activityLog.create({
    actorId: owner.id,
    action: 'IMPERSONATION_END',
    targetId: session.impersonatedUserId,
    metadata: {
      sessionId: session.id,
      durationMinutes: differenceInMinutes(new Date(), session.startedAt),
    },
    severity: 'CRITICAL',
  });

  return { message: 'Impersonation ended' };
}
```

### 7.4 Impersonation Audit Interceptor

```typescript
@Injectable()
export class ImpersonationAuditInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest();
    const user = request.user;

    // Если активна impersonation — логировать каждое действие
    if (user?.impersonatorId) {
      this.activityLog.create({
        actorId: user.impersonatorId,
        action: 'IMPERSONATED_ACTION',
        targetId: user.id,
        metadata: {
          endpoint: request.url,
          method: request.method,
          body: request.body, // Log full request
        },
        severity: 'HIGH',
      });
    }

    return next.handle();
  }
}
```

---

## 8. Rate Limiting

### 8.1 Конфигурация по endpoint

```typescript
const RATE_LIMITS = {
  // Auth endpoints
  'POST /auth/login':                  { maxRequests: 5,    windowMinutes: 15,  key: 'ip' },
  'POST /auth/password-reset/request': { maxRequests: 3,    windowMinutes: 60,  key: 'ip' },
  'POST /auth/register':               { maxRequests: 3,    windowMinutes: 60,  key: 'ip' },

  // API endpoints (authenticated)
  'GET /api/*':                        { maxRequests: 100,  windowMinutes: 1,   key: 'userId' },
  'POST /api/*':                       { maxRequests: 60,   windowMinutes: 1,   key: 'userId' },

  // Critical operations
  'POST /api/admin/guests/:id/manual-adjustment': { maxRequests: 10, windowMinutes: 60, key: 'userId' },
  'POST /api/owner/impersonate':                  { maxRequests: 5,  windowMinutes: 60, key: 'userId' },

  // POS webhooks (high limit for peak hours)
  'POST /webhooks/pos/:tenantId':      { maxRequests: 1000, windowMinutes: 1,   key: 'tenantId' },

  // Export operations
  'POST /api/admin/export/guests':     { maxRequests: 5,    windowMinutes: 60,  key: 'userId' },
};
```

### 8.2 Implementation с Redis

```typescript
import { RateLimiterRedis } from 'rate-limiter-flexible';

@Injectable()
export class RateLimitGuard implements CanActivate {
  private limiters = new Map<string, RateLimiterRedis>();

  constructor(private redis: Redis) {
    // Initialize limiters for each endpoint
    Object.entries(RATE_LIMITS).forEach(([endpoint, config]) => {
      this.limiters.set(endpoint, new RateLimiterRedis({
        storeClient: redis,
        keyPrefix: `rate-limit:${endpoint}`,
        points: config.maxRequests,
        duration: config.windowMinutes * 60,
      }));
    });
  }

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const route = `${request.method} ${request.route?.path}`;
    const config = RATE_LIMITS[route];

    if (!config) return true; // No limit configured

    const limiter = this.limiters.get(route);
    const key = this.getKey(request, config.key);

    try {
      await limiter.consume(key, 1);
      return true;
    } catch (rateLimitExceeded) {
      // Add rate limit headers
      const res = context.switchToHttp().getResponse();
      res.header('X-RateLimit-Limit', config.maxRequests);
      res.header('X-RateLimit-Remaining', 0);
      res.header('X-RateLimit-Reset', rateLimitExceeded.msBeforeNext / 1000);

      throw new HttpException(
        { statusCode: 429, message: 'Too many requests', retryAfter: rateLimitExceeded.msBeforeNext / 1000 },
        429,
      );
    }
  }

  private getKey(request: any, keyType: string): string {
    switch (keyType) {
      case 'ip':       return request.ip;
      case 'userId':   return request.user?.id || request.ip;
      case 'tenantId': return request.params.tenantId;
      default:         return request.ip;
    }
  }
}
```

---

## 9. Guards, Decorators, Interceptors

### 9.1 Guards

| Guard | Назначение |
|---|---|
| `JwtAuthGuard` | Валидация JWT, проверка blacklist |
| `TenantGuard` | Tenant-изоляция, проверка tenantId |
| `RoleGuard` | RBAC проверка роли |
| `PermissionsGuard` | Fine-grained permissions check |
| `MFAGuard` | Проверка MFA-верификации |
| `StepUpAuthGuard` | Step-Up Auth для критических операций |
| `RateLimitGuard` | Rate limiting по endpoint |

### 9.2 Декораторы

```typescript
// @CurrentUser() — получить текущего пользователя из JWT
@Get('profile')
async getProfile(@CurrentUser() user: User) { ... }

// @Roles() — требовать роль
@Roles(Role.ADMIN, Role.OWNER)
@UseGuards(JwtAuthGuard, RoleGuard)
@Get('admin/guests')
async listGuests() { ... }

// @RequirePermissions() — fine-grained
@RequirePermissions(Permission.GUESTS_READ)
@UseGuards(JwtAuthGuard, PermissionsGuard)
@Get('admin/guests')
async listGuests() { ... }

// @Public() — публичный endpoint (bypass JwtAuthGuard)
@Public()
@Post('auth/login')
async login() { ... }
```

### 9.3 Interceptors

```typescript
// StructuredLoggingInterceptor — логирует все запросы
@Injectable()
export class StructuredLoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const startTime = Date.now();
    const requestId = crypto.randomUUID();

    // Add request ID to context
    request.id = requestId;

    this.securityLogger.logSecurityEvent({
      type: SecurityEvent.API_REQUEST,
      userId: request.user?.id,
      tenantId: request.user?.tenantId,
      ipAddress: request.ip,
      action: `${request.method} ${request.url}`,
      metadata: { requestId, body: this.sanitizeBody(request.body) },
    });

    return next.handle().pipe(
      tap({
        next: (response) => {
          const duration = Date.now() - startTime;
          this.logger.log({ requestId, method: request.method, url: request.url, status: 200, durationMs: duration });
        },
        error: (error) => {
          const duration = Date.now() - startTime;
          this.securityLogger.logSecurityEvent({
            type: SecurityEvent.API_ERROR,
            severity: error.status >= 500 ? 'HIGH' : 'LOW',
            metadata: { requestId, statusCode: error.status, errorMessage: error.message },
          });
        },
      }),
    );
  }

  // Sanitize body — убрать чувствительные поля из логов
  private sanitizeBody(body: any): any {
    if (!body) return null;
    const clone = { ...body };
    const sensitiveFields = ['password', 'passwordConfirmation', 'cardNumber', 'cvv', 'apiKey'];
    for (const field of sensitiveFields) {
      if (clone[field]) clone[field] = 'REDACTED';
    }
    return clone;
  }
}
```

---

## 10. Flows: Registration, Login, Logout

### 10.1 Registration Flow

```
1. POST /auth/register { email, phone, password }
2. Validate password policy (10+ chars, entropy, etc.)
3. Check email + phone uniqueness
4. Hash password (argon2id)
5. Create User + UserSession
6. Send email verification link (24h TTL)
7. Return { accessToken, refreshToken (HttpOnly cookie) }
```

### 10.2 Login Flow (детально)

```
1. POST /auth/login { email/phone, password }
2. Rate limiting (5 попыток / 15 мин per IP)
3. Find user by email or phone
4. Check account not locked
5. Verify password (argon2.verify)
6. If MFA enabled (Owner/Admin):
   a. POST /auth/login → { mfaRequired: true, mfaToken: <temp> }
   b. POST /auth/mfa/verify { mfaToken, totpCode }
7. Create session (max 5 concurrent, suspicious activity check)
8. Generate JWT access token (RS256, 5min)
9. Generate refresh token (14d, HttpOnly Secure Cookie)
10. Log to ActivityLog
11. Return { accessToken, user: { id, email, role, tenantId } }
```

### 10.3 Logout Flow

```
1. POST /auth/logout (with Authorization: Bearer <accessToken>)
2. Add accessToken to Redis blacklist (TTL = remaining exp time)
3. Mark UserSession.isBlacklisted = true
4. Clear HttpOnly cookie (refreshToken)
5. Return { message: 'Logged out' }
```

### 10.4 Logout All Sessions

```
1. POST /auth/logout-all
2. Mark ALL UserSessions.isBlacklisted = true
3. Add all active refresh tokens to Redis blacklist
4. Clear current HttpOnly cookie
5. Return { message: 'All sessions terminated' }
```

### 10.5 Controller: все endpoints

```typescript
@Controller('auth')
export class AuthController {
  @Public() @Post('register')           // Регистрация
  @Public() @Post('login')              // Логин
  @Public() @Post('refresh')            // Обновление токена
  @Public() @Post('password-reset/request')  // Запрос сброса
  @Public() @Post('password-reset/confirm')  // Подтверждение сброса
  @Public() @Post('verify-email')       // Подтверждение email
  @Public() @Post('mfa/verify')         // Верификация MFA
  
  @UseGuards(JwtAuthGuard)
  @Post('logout')                        // Logout (одна сессия)
  @Post('logout-all')                    // Logout всех сессий
  @Post('mfa/setup')                     // Настройка TOTP
  @Delete('mfa/disable')                 // Отключение MFA
  @Get('me')                             // Текущий пользователь
}
```

---

## 11. Database entities (Auth)

### 11.1 User (auth-поля)

```prisma
model User {
  id                String    @id @default(uuid())
  email             String    @unique
  phone             String?   @unique
  passwordHash      String
  passwordChangedAt DateTime?
  emailVerified     Boolean   @default(false)
  phoneVerified     Boolean   @default(false)
  
  // MFA
  mfaEnabled        Boolean   @default(false)
  mfaSecret         String?   // Encrypted (AES-256-GCM)
  
  // Account status
  isLocked          Boolean   @default(false)
  lockedUntil       DateTime?
  failedLoginCount  Int       @default(0)
  lastLoginAt       DateTime?
  lastLoginIp       String?
  
  // Relations
  sessions          UserSession[]
  passwordResetTokens PasswordResetToken[]
  passwordHistory   PasswordHistory[]
  activityLogs      UserActivityLog[]
  
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
}
```

### 11.2 UserSession

```prisma
model UserSession {
  id                  String    @id @default(uuid())
  userId              String
  refreshToken        String    @unique
  parentSessionId     String?   // For refresh token family tracking
  
  // Device info
  ipAddress           String?
  userAgent           String?
  deviceFingerprint   String?
  country             String?
  city                String?
  
  // Status
  isBlacklisted       Boolean   @default(false)
  refreshTokenReused  Boolean   @default(false)
  lastActivityAt      DateTime  @default(now())
  expiresAt           DateTime
  
  user                User      @relation(fields: [userId], references: [id])
  
  createdAt           DateTime  @default(now())
}
```

### 11.3 PasswordResetToken

```prisma
model PasswordResetToken {
  id         String    @id @default(uuid())
  userId     String
  token      String    @unique // SHA-256 hash (never plain)
  usedAt     DateTime?
  expiresAt  DateTime
  ipAddress  String?
  
  user       User      @relation(fields: [userId], references: [id])
  createdAt  DateTime  @default(now())
}
```

### 11.4 OwnerRegistrationLink

```prisma
model OwnerRegistrationLink {
  id          String    @id @default(uuid())
  token       String    @unique
  email       String
  usedAt      DateTime?
  expiresAt   DateTime
  createdById String    // Owner who created the link
  createdAt   DateTime  @default(now())
}
```

### 11.5 ImpersonationSession

```prisma
model ImpersonationSession {
  id                   String    @id @default(uuid())
  impersonatorUserId   String    // Owner
  impersonatedUserId   String    // Target user
  reason               String    // Required min 10 chars
  startedAt            DateTime  @default(now())
  endedAt              DateTime?
  expiresAt            DateTime
  ipAddress            String?
  userAgent            String?
  
  impersonator         User      @relation("Impersonator", fields: [impersonatorUserId], references: [id])
  impersonated         User      @relation("Impersonated", fields: [impersonatedUserId], references: [id])
}
```

### 11.6 UserActivityLog

```prisma
model UserActivityLog {
  id         String    @id @default(uuid())
  actorId    String    // Кто совершил действие
  action     String    // IMPERSONATION_START, LOGIN_FAILED, etc.
  targetId   String?   // На кого направлено действие
  metadata   Json?     // Дополнительные данные
  severity   String    @default("INFO") // INFO | LOW | HIGH | CRITICAL
  ipAddress  String?
  createdAt  DateTime  @default(now())
  
  actor      User      @relation(fields: [actorId], references: [id])
}
```

---

## 12. Security Testing

### 12.1 JWT Security Tests

Файл: `apps/backend/test/security/jwt-security.security.spec.ts`

| Тест | Сценарий | Ожидаемый результат |
|---|---|---|
| Test 1 | Валидный токен | 200 |
| Test 2 | Истёкший токен | 401 "expired" |
| Test 3 | Тампер payload (GUEST → ADMIN) | 401 "invalid" |
| Test 4 | Неверный secret | 401 "invalid" |
| Test 5 | `alg: none` атака | 401 "invalid" |
| Test 6 | Refresh token rotation | Новый токен, старый недействителен |
| Test 7 | Reuse detection | 401, все сессии отозваны |
| Test 8 | Missing claims | 401 |
| Test 9 | Issuer/Audience validation | 401 |
| Test 10 | Blacklist при logout | 401 "blacklisted" |
| Test 11 | Access 5min, Refresh 14d TTL | Verify TTL |
| Test 12 | Algorithm confusion (RS256 public key) | 401 |

### 12.2 Brute Force Tests

Файл: `apps/backend/test/security/brute-force.security.spec.ts`

| Тест | Сценарий | Ожидаемый результат |
|---|---|---|
| Test 1 | 5 попыток | 401 (not rate limited) |
| Test 2 | 11-я попытка | 429 + Retry-After header |
| Test 3 | Account lockout | 423 Locked + lockedUntil |
| Test 4 | Exponential backoff | Задержки растут |
| Test 5 | IP-based rate limit | IP1 limited, IP2 ok |
| Test 6 | Reset on success | После успешного логина счётчик сброшен |
| Test 7 | CAPTCHA после 5 попыток | 403 + captchaRequired: true |
| Test 8 | Distributed brute force | Обнаружение + блокировка |
| Test 9 | Security alert | NotificationService вызван |
| Test 10 | Автоматическая симуляция | blockedCount > 0 |

### 12.3 CSRF Tests

Файл: `apps/backend/test/security/csrf.security.spec.ts`

- GET запросы — без CSRF токена (допускаются)
- POST/PATCH/PUT/DELETE — CSRF токен обязателен
- Double-submit cookie pattern
- Инвалидация при logout
- SameSite=Strict cookie attribute
- API токены (machine-to-machine) освобождены от CSRF

### 12.4 Authorization Tests

Файл: `apps/backend/test/security/authorization.security.spec.ts`

- GUEST → admin endpoints: 403
- MANAGER → Owner-only endpoints: 403
- Horizontal privilege escalation (Guest A → Guest B): 403
- IDOR (прямой доступ к чужому ресурсу): 403
- Tenant isolation: 403
- Parameter tampering (role=ADMIN в body): role не меняется
- Mass assignment: 400
- Role hierarchy enforcement

---

## 13. Чеклист реализации

### Week 1–2 (Foundation)

- [ ] Настроить JWT (RS256), генерация ключей `jwt-keygen.sh`
- [ ] Реализовать Password Policy (argon2id, 10 символов)
- [ ] Настроить RBAC guards (30 permissions)
- [ ] Включить HTTPS/TLS 1.3 на всех endpoints

### Week 3–4 (MFA & Sessions)

- [ ] Реализовать MFA (TOTP + SMS backup)
- [ ] Session Management (limit 5, device fingerprinting)
- [ ] Refresh Token Rotation + Family Tracking
- [ ] Redis Blacklist для logout

### Week 5–6 (Input Validation)

- [ ] Input validation (class-validator)
- [ ] XSS prevention (DOMPurify, CSP headers)
- [ ] CSRF protection (tokens, SameSite)
- [ ] Configure Rate Limiting (Redis)

### Week 7–8 (Monitoring)

- [ ] SIEM integration (ELK Stack / Datadog)
- [ ] Security event logging (40 событий)
- [ ] Correlation rules (5 автоматических детекций)
- [ ] Prometheus metrics + Grafana dashboards

### Контрольная таблица статусов

| Контроль | Реализация | Статус |
|---|---|---|
| 1.1 JWT Token Security | RS256, 5min access, 14d refresh with rotation | Approved |
| 1.2 Password Policy | 10 chars, argon2id, entropy check, history 5 | Approved |
| 1.3 MFA | Owner/Admin required, TOTP + SMS backup | Approved |
| 1.4 Session Management | Limit 5, idle 30min, absolute 12h, device fingerprinting | Approved |
| 1.5 Password Reset | 15min TTL, SHA-256 hash, rate limit 3/hour | Approved |
| 1.6 Impersonation | MFA required, 2h max, full audit, notification | Approved |
| 1.7 Rate Limiting | Per endpoint 5-1000 req/min, Redis-based | Approved |
| 1.8 API Key Management | HMAC-SHA256, 365d expiration, 5 keys/tenant | Approved |

---

*Документ S-02 создан на основе данных из Beseda-1, Beseda-3, Beseda-13 (Security & Compliance — 41 вопрос × 100% coverage).*
