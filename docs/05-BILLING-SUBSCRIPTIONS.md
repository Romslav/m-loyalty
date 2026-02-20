# S-05: Billing & Subscriptions

> **Часть 5 из 7** | Зависимости: S-01, S-02, S-03, S-04  
> Охват: SubscriptionService, PaymentService (YooKassa/Stripe), апгрейд/даунгрейд, PASTDUE flow, Invoice, метрики MRR/ARR/Churn, PCI DSS, Prisma Schema

---

## Содержание

1. [Обзор биллинга](#1-обзор-биллинга)
2. [Subscription Service](#2-subscription-service)
3. [Payment Service — YooKassa & Stripe](#3-payment-service--yookassa--stripe)
4. [Upgrade / Downgrade Flow](#4-upgrade--downgrade-flow)
5. [PASTDUE & Dunning Flow](#5-pastdue--dunning-flow)
6. [Invoice Service](#6-invoice-service)
7. [Billing Metrics (MRR/ARR/Churn)](#7-billing-metrics-mrrarrchurn)
8. [Manual Payments (ULTIMATE/CUSTOM)](#8-manual-payments-ultimatecustom)
9. [Refund & Chargeback](#9-refund--chargeback)
10. [Prisma Schema](#10-prisma-schema)
11. [API Endpoints](#11-api-endpoints)
12. [CRON Jobs](#12-cron-jobs)
13. [Тесты & Чеклист](#13-тесты--чеклист)

---

## 1. Обзор биллинга

### 1.1. Архитектура

```
┌─────────────────────────────────────────────────────────────────┐
│                      BILLING FLOW                               │
│                                                                 │
│  Owner/Admin                                                    │
│      │                                                          │
│      ▼                                                          │
│  POST /billing/checkout ──► YooKassa/Stripe ──► Redirect URL   │
│                                    │                            │
│                                    ▼                            │
│                          POST /webhooks/payment/{provider}      │
│                                    │                            │
│                          ┌─────────▼─────────┐                 │
│                          │  PaymentService   │                 │
│                          │  verifySignature  │                 │
│                          │  updatePayment    │                 │
│                          │  activateSub      │                 │
│                          └─────────┬─────────┘                 │
│                                    │                            │
│                          ┌─────────▼─────────┐                 │
│                          │SubscriptionService│                 │
│                          │ updateStatus      │                 │
│                          │ updateLimits      │                 │
│                          │ sendNotification  │                 │
│                          └───────────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. Провайдеры оплаты

| Провайдер | Регион | Методы | Использование |
|---|---|---|---|
| **YooKassa** | 🇷🇺 РФ | Карты, СБП, ЮMoney, Qiwi | Основной для RU |
| **Stripe** | 🌍 Глобально | Visa/MC, SEPA, Apple/Google Pay | Международные |
| **CloudPayments** | 🇷🇺 РФ | Карты | Альтернатива YooKassa |
| **CryptoCloud** | 🌍 Глобально | Крипто | Опционально |
| **MANUAL** | — | Счёт/перевод | ULTIMATE/CUSTOM |

---

## 2. Subscription Service

```typescript
// src/modules/subscriptions/subscription.service.ts
import {
  Injectable,
  NotFoundException,
  BadRequestException,
  ForbiddenException,
} from '@nestjs/common';
import { PrismaService } from '../../database/prisma.service';
import { TenantLimitsService } from '../tenants/tenant-limits.service';
import { NotificationService } from '../notifications/notification.service';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { addMonths, addYears } from 'date-fns';
import { PLAN_LIMITS, PLAN_PRICES } from './constants/plans.constants';

@Injectable()
export class SubscriptionService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly limitsService: TenantLimitsService,
    private readonly notificationService: NotificationService,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  // === Получение подписки tenant'а ===
  async getSubscription(tenantId: string) {
    const sub = await this.prisma.subscription.findUnique({
      where: { tenantId },
      include: {
        payments: {
          orderBy: { createdAt: 'desc' },
          take: 10,
        },
      },
    });
    if (!sub) throw new NotFoundException('Подписка не найдена');
    return sub;
  }

  // === Активация подписки после оплаты ===
  async activate(subscriptionId: string, paymentId: string): Promise<void> {
    const sub = await this.prisma.subscription.findUnique({
      where: { id: subscriptionId },
    });
    if (!sub) throw new NotFoundException('Подписка не найдена');

    const now = new Date();
    const periodEnd = sub.billingPeriod === 'YEARLY'
      ? addYears(now, 1)
      : addMonths(now, 1);

    await this.prisma.$transaction(async (tx) => {
      await tx.subscription.update({
        where: { id: subscriptionId },
        data: {
          status: 'ACTIVE',
          currentPeriodStart: now,
          currentPeriodEnd: periodEnd,
          trialEndsAt: null, // trial закончился
        },
      });

      // Обновляем TenantLimits согласно плану
      const limits = PLAN_LIMITS[sub.plan];
      await tx.tenantLimits.update({
        where: { tenantId: sub.tenantId },
        data: {
          maxRestaurants: limits.maxRestaurants,
          maxGuests: limits.maxGuests,
          maxPosIntegrations: limits.maxPosIntegrations,
          maxAdminUsers: limits.maxAdminUsers,
          maxStorageMb: limits.maxStorageMb,
        },
      });

      // SubscriptionHistory
      await tx.subscriptionHistory.create({
        data: {
          tenantId: sub.tenantId,
          subscriptionId,
          event: 'ACTIVATED',
          planFrom: sub.plan,
          planTo: sub.plan,
          paymentId,
        },
      });

      await tx.activityLog.create({
        data: {
          tenantId: sub.tenantId,
          action: 'SUBSCRIPTION_ACTIVATED',
          entityType: 'SUBSCRIPTION',
          entityId: subscriptionId,
          newValue: { plan: sub.plan, periodEnd },
        },
      });
    });

    await this.notificationService.sendToTenantOwners(sub.tenantId, {
      type: 'SUBSCRIPTION_ACTIVATED',
      channel: ['TELEGRAM', 'EMAIL'],
      title: '✅ Подписка активирована',
      message: `Тариф ${sub.plan} активен до ${periodEnd.toLocaleDateString('ru-RU')}`,
    });
  }

  // === Обновление плана (апгрейд/даунгрейд) ===
  async changePlan(
    tenantId: string,
    newPlan: string,
    billingPeriod: 'MONTHLY' | 'YEARLY',
    reason: string,
    actorId: string,
  ) {
    const sub = await this.prisma.subscription.findUnique({
      where: { tenantId },
    });
    if (!sub) throw new NotFoundException('Подписка не найдена');

    const oldPlan = sub.plan;
    const isUpgrade = this.isPlanHigher(newPlan, oldPlan);

    if (isUpgrade) {
      return this.processUpgrade(sub, newPlan, billingPeriod, reason, actorId);
    } else {
      return this.scheduleDowngrade(sub, newPlan, billingPeriod, reason, actorId);
    }
  }

  // Апгрейд — немедленно, с пересчётом суммы
  private async processUpgrade(
    sub: any,
    newPlan: string,
    billingPeriod: 'MONTHLY' | 'YEARLY',
    reason: string,
    actorId: string,
  ) {
    const now = new Date();

    // Рассчитываем пропорциональную сумму (prorated)
    const daysRemaining = Math.ceil(
      (sub.currentPeriodEnd.getTime() - now.getTime()) / (1000 * 60 * 60 * 24),
    );
    const totalDays = Math.ceil(
      (sub.currentPeriodEnd.getTime() - sub.currentPeriodStart.getTime()) / (1000 * 60 * 60 * 24),
    );
    const newPrice = billingPeriod === 'YEARLY'
      ? PLAN_PRICES[newPlan].yearly
      : PLAN_PRICES[newPlan].monthly;
    const oldPrice = billingPeriod === 'YEARLY'
      ? PLAN_PRICES[sub.plan].yearly
      : PLAN_PRICES[sub.plan].monthly;

    // delta = сколько доплатить
    const unusedValue = oldPrice * (daysRemaining / totalDays);
    const newValue = newPrice * (daysRemaining / totalDays);
    const delta = Math.ceil(newValue - unusedValue);

    return {
      type: 'UPGRADE',
      fromPlan: sub.plan,
      toPlan: newPlan,
      delta,
      message: delta > 0
        ? `К оплате сейчас: ${delta} ₽ (пропорциональная разница)`
        : 'Бесплатный апгрейд (уже оплачено)',
      requiresPayment: delta > 0,
      subscriptionId: sub.id,
    };
  }

  // Даунгрейд — откладываем до конца периода
  private async scheduleDowngrade(
    sub: any,
    newPlan: string,
    billingPeriod: 'MONTHLY' | 'YEARLY',
    reason: string,
    actorId: string,
  ) {
    await this.prisma.subscription.update({
      where: { id: sub.id },
      data: {
        pendingPlanChange: newPlan,
        pendingBillingPeriod: billingPeriod,
        pendingPlanChangeAt: sub.currentPeriodEnd,
      },
    });

    await this.prisma.subscriptionHistory.create({
      data: {
        tenantId: sub.tenantId,
        subscriptionId: sub.id,
        event: 'DOWNGRADE_SCHEDULED',
        planFrom: sub.plan,
        planTo: newPlan,
        effectiveDate: sub.currentPeriodEnd,
        reason,
        actorId,
      },
    });

    await this.notificationService.sendToTenantOwners(sub.tenantId, {
      type: 'PLAN_DOWNGRADE_SCHEDULED',
      channel: ['EMAIL'],
      title: '📋 Даунгрейд запланирован',
      message: `Тариф изменится с ${sub.plan} на ${newPlan} с ${sub.currentPeriodEnd.toLocaleDateString('ru-RU')}`,
    });

    return {
      type: 'DOWNGRADE_SCHEDULED',
      fromPlan: sub.plan,
      toPlan: newPlan,
      effectiveDate: sub.currentPeriodEnd,
      message: 'Смена тарифа запланирована на конец периода',
    };
  }

  // === Применение запланированного даунгрейда (CRON) ===
  async applyPendingPlanChanges(): Promise<void> {
    const pending = await this.prisma.subscription.findMany({
      where: {
        pendingPlanChange: { not: null },
        pendingPlanChangeAt: { lte: new Date() },
      },
    });

    for (const sub of pending) {
      await this.prisma.$transaction(async (tx) => {
        const newPlan = sub.pendingPlanChange!;
        const newPeriod = (sub.pendingBillingPeriod ?? 'MONTHLY') as 'MONTHLY' | 'YEARLY';
        const newPrice = newPeriod === 'YEARLY'
          ? PLAN_PRICES[newPlan].yearly
          : PLAN_PRICES[newPlan].monthly;

        await tx.subscription.update({
          where: { id: sub.id },
          data: {
            plan: newPlan,
            billingPeriod: newPeriod,
            priceMonthly: PLAN_PRICES[newPlan].monthly,
            priceYearly: PLAN_PRICES[newPlan].yearly,
            pendingPlanChange: null,
            pendingBillingPeriod: null,
            pendingPlanChangeAt: null,
          },
        });

        // Обновляем лимиты
        const limits = PLAN_LIMITS[newPlan];
        await tx.tenantLimits.update({
          where: { tenantId: sub.tenantId },
          data: {
            maxRestaurants: limits.maxRestaurants,
            maxGuests: limits.maxGuests,
            maxPosIntegrations: limits.maxPosIntegrations,
            maxAdminUsers: limits.maxAdminUsers,
          },
        });

        await tx.subscriptionHistory.create({
          data: {
            tenantId: sub.tenantId,
            subscriptionId: sub.id,
            event: 'DOWNGRADED',
            planFrom: sub.plan,
            planTo: newPlan,
          },
        });
      });
    }
  }

  // === Отмена подписки ===
  async cancel(tenantId: string, reason: string, actorId: string) {
    const sub = await this.prisma.subscription.findUnique({
      where: { tenantId },
    });
    if (!sub) throw new NotFoundException('Подписка не найдена');

    await this.prisma.$transaction(async (tx) => {
      await tx.subscription.update({
        where: { id: sub.id },
        data: {
          status: 'CANCELLED',
          cancelledAt: new Date(),
          autoRenew: false,
        },
      });

      await tx.subscriptionHistory.create({
        data: {
          tenantId,
          subscriptionId: sub.id,
          event: 'CANCELLED',
          planFrom: sub.plan,
          planTo: sub.plan,
          reason,
          actorId,
        },
      });

      await tx.activityLog.create({
        data: {
          tenantId,
          actorId,
          action: 'SUBSCRIPTION_CANCELLED',
          entityType: 'SUBSCRIPTION',
          entityId: sub.id,
          newValue: { reason },
        },
      });
    });

    this.eventEmitter.emit('subscription.cancelled', { tenantId, plan: sub.plan, reason });
  }

  private isPlanHigher(newPlan: string, currentPlan: string): boolean {
    const order = ['FREE', 'STANDARD', 'MEDIUM', 'PRO', 'ULTIMATE', 'CUSTOM'];
    return order.indexOf(newPlan) > order.indexOf(currentPlan);
  }
}
```

---

## 3. Payment Service — YooKassa & Stripe

### 3.1. Создание платежа (checkout)

```typescript
// src/modules/billing/payment.service.ts
import { Injectable, BadRequestException, Logger } from '@nestjs/common';
import { PrismaService } from '../../database/prisma.service';
import { SubscriptionService } from '../subscriptions/subscription.service';
import { InvoiceService } from './invoice.service';
import * as crypto from 'crypto';

@Injectable()
export class PaymentService {
  private readonly logger = new Logger(PaymentService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly subscriptionService: SubscriptionService,
    private readonly invoiceService: InvoiceService,
  ) {}

  // === Создание платёжной сессии ===
  async createCheckout(tenantId: string, dto: CreateCheckoutDto) {
    const sub = await this.prisma.subscription.findUnique({
      where: { tenantId },
      include: { billingInfo: true },
    });
    if (!sub) throw new BadRequestException('Подписка не найдена');

    const amount = dto.billingPeriod === 'YEARLY'
      ? PLAN_PRICES[dto.plan].yearly
      : PLAN_PRICES[dto.plan].monthly;

    // Сохраняем Payment с PENDING статусом
    const payment = await this.prisma.payment.create({
      data: {
        tenantId,
        subscriptionId: sub.id,
        amount,
        currency: 'RUB',
        status: 'PENDING',
        paymentProvider: dto.provider,
        description: `Подписка Max Loyalty — ${dto.plan} (${dto.billingPeriod})`,
        metadata: {
          plan: dto.plan,
          billingPeriod: dto.billingPeriod,
          tenantId,
        },
      },
    });

    const provider = dto.provider ?? sub.billingInfo?.paymentProvider ?? 'YOOKASSA';

    if (provider === 'YOOKASSA') {
      return this.createYooKassaPayment(payment, sub, amount);
    } else if (provider === 'STRIPE') {
      return this.createStripePayment(payment, sub, amount);
    }

    throw new BadRequestException('Провайдер не поддерживается');
  }

  // === YooKassa: создание платежа ===
  private async createYooKassaPayment(payment: any, sub: any, amount: number) {
    // PCI DSS: НЕ передаём данные карты — только redirect!
    const body = {
      amount: { value: (amount / 100).toFixed(2), currency: 'RUB' },
      capture: true,
      confirmation: {
        type: 'redirect',
        return_url: `https://app.max-loyalty.com/billing/result?paymentId=${payment.id}`,
      },
      description: payment.description,
      metadata: { paymentId: payment.id, tenantId: sub.tenantId },
      save_payment_method: true, // сохраняем токен для автопродления
    };

    const response = await fetch('https://api.yookassa.ru/v3/payments', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Idempotence-Key': payment.id, // идемпотентность
        Authorization: `Basic ${Buffer.from(
          `${process.env.YOOKASSA_SHOP_ID}:${process.env.YOOKASSA_SECRET_KEY}`,
        ).toString('base64')}`,
      },
      body: JSON.stringify(body),
    });

    const data = await response.json();

    // Сохраняем externalPaymentId
    await this.prisma.payment.update({
      where: { id: payment.id },
      data: { externalPaymentId: data.id },
    });

    return {
      paymentId: payment.id,
      confirmationUrl: data.confirmation.confirmation_url,
      expiresAt: data.expires_at,
    };
  }

  // === YooKassa Webhook ===
  async handleYooKassaWebhook(payload: any, signature: string): Promise<void> {
    // Проверка подписи
    this.verifyYooKassaSignature(payload, signature);

    const event = payload.event;
    const yooPayment = payload.object;

    if (event === 'payment.succeeded') {
      await this.handlePaymentSucceeded(yooPayment, 'YOOKASSA');
    } else if (event === 'payment.canceled') {
      await this.handlePaymentFailed(yooPayment, 'YOOKASSA', 'Платёж отменён');
    } else if (event === 'refund.succeeded') {
      await this.handleRefundSucceeded(yooPayment, 'YOOKASSA');
    }
  }

  // === Stripe Webhook ===
  async handleStripeWebhook(payload: Buffer, signature: string): Promise<void> {
    let event: any;
    try {
      // Stripe верифицирует через stripe.webhooks.constructEvent
      const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
      event = stripe.webhooks.constructEvent(
        payload,
        signature,
        process.env.STRIPE_WEBHOOK_SECRET,
      );
    } catch (err) {
      throw new BadRequestException(`Stripe webhook signature error: ${err.message}`);
    }

    switch (event.type) {
      case 'payment_intent.succeeded':
        await this.handlePaymentSucceeded(event.data.object, 'STRIPE');
        break;
      case 'payment_intent.payment_failed':
        await this.handlePaymentFailed(event.data.object, 'STRIPE', event.data.object.last_payment_error?.message);
        break;
      case 'charge.refunded':
        await this.handleRefundSucceeded(event.data.object, 'STRIPE');
        break;
    }
  }

  // === Успешный платёж (общий handler) ===
  private async handlePaymentSucceeded(providerPayment: any, provider: string): Promise<void> {
    const externalId = providerPayment.id;

    const payment = await this.prisma.payment.findFirst({
      where: { externalPaymentId: externalId },
    });
    if (!payment) {
      this.logger.warn(`Payment not found for externalId: ${externalId}`);
      return;
    }

    await this.prisma.$transaction(async (tx) => {
      await tx.payment.update({
        where: { id: payment.id },
        data: {
          status: 'SUCCEEDED',
          paidAt: new Date(),
        },
      });

      // Сохраняем токен для автопродления (PCI DSS: только токен!)
      if (providerPayment.payment_method) {
        const pm = providerPayment.payment_method;
        await tx.billingInfo.update({
          where: { tenantId: payment.tenantId },
          data: {
            paymentMethodId: pm.id ?? pm.saved_payment_method_id,
            paymentProvider: provider as any,
          },
        });
      }

      // Создаём Invoice
      await this.invoiceService.createFromPaymentTx(tx, payment);
    });

    // Активируем подписку
    await this.subscriptionService.activate(payment.subscriptionId, payment.id);

    this.logger.log(`Payment ${payment.id} succeeded for tenant ${payment.tenantId}`);
  }

  // === Неуспешный платёж ===
  private async handlePaymentFailed(
    providerPayment: any,
    provider: string,
    errorMessage?: string,
  ): Promise<void> {
    const payment = await this.prisma.payment.findFirst({
      where: { externalPaymentId: providerPayment.id },
    });
    if (!payment) return;

    await this.prisma.payment.update({
      where: { id: payment.id },
      data: { status: 'FAILED', errorMessage: errorMessage ?? 'Платёж не прошёл' },
    });

    // Запускаем dunning flow
    this.eventEmitter.emit('payment.failed', { tenantId: payment.tenantId, paymentId: payment.id });
  }

  // === Автопродление ===
  async renewSubscription(tenantId: string): Promise<void> {
    const sub = await this.prisma.subscription.findUnique({
      where: { tenantId },
      include: { billingInfo: true },
    });

    if (!sub || !sub.autoRenew) return;
    if (sub.status !== 'ACTIVE') return;
    if (!sub.billingInfo?.paymentMethodId) {
      // Нет сохранённого метода — переходим в PAST_DUE
      await this.setPastDue(tenantId, 'Нет сохранённого метода оплаты');
      return;
    }

    const amount = sub.billingPeriod === 'YEARLY'
      ? PLAN_PRICES[sub.plan].yearly
      : PLAN_PRICES[sub.plan].monthly;

    const payment = await this.prisma.payment.create({
      data: {
        tenantId,
        subscriptionId: sub.id,
        amount,
        currency: 'RUB',
        status: 'PENDING',
        paymentProvider: sub.billingInfo.paymentProvider!,
        description: `Автопродление — ${sub.plan}`,
      },
    });

    try {
      if (sub.billingInfo.paymentProvider === 'YOOKASSA') {
        await this.chargeYooKassa(payment, sub.billingInfo.paymentMethodId, amount);
      } else if (sub.billingInfo.paymentProvider === 'STRIPE') {
        await this.chargeStripe(payment, sub.billingInfo.paymentMethodId, amount);
      }
    } catch (err) {
      this.logger.error(`Auto-renew failed for tenant ${tenantId}: ${err.message}`);
      await this.setPastDue(tenantId, err.message);
    }
  }

  // === Повторное списание сохранённым методом ===
  private async chargeYooKassa(
    payment: any,
    paymentMethodId: string,
    amount: number,
  ): Promise<void> {
    const body = {
      amount: { value: (amount / 100).toFixed(2), currency: 'RUB' },
      capture: true,
      payment_method_id: paymentMethodId, // сохранённый токен
      description: payment.description,
      metadata: { paymentId: payment.id },
    };

    const response = await fetch('https://api.yookassa.ru/v3/payments', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Idempotence-Key': payment.id,
        Authorization: `Basic ${Buffer.from(
          `${process.env.YOOKASSA_SHOP_ID}:${process.env.YOOKASSA_SECRET_KEY}`,
        ).toString('base64')}`,
      },
      body: JSON.stringify(body),
    });

    if (!response.ok) {
      throw new Error(`YooKassa charge failed: ${response.status}`);
    }
  }

  private async chargeStripe(
    payment: any,
    paymentMethodId: string,
    amount: number,
  ): Promise<void> {
    const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
    await stripe.paymentIntents.create({
      amount, // в копейках
      currency: 'rub',
      payment_method: paymentMethodId,
      confirm: true,
      metadata: { paymentId: payment.id },
    });
  }

  // === Проверка подписи YooKassa ===
  private verifyYooKassaSignature(payload: any, signature: string): void {
    // YooKassa использует IP-whitelist + опциональный HMAC
    // В продакшне проверяем IP: 185.71.76.0/27, 185.71.77.0/27, 77.75.153.0/25
    // Дополнительно можно включить webhookSecret в настройках YooKassa
    const expectedSig = crypto
      .createHmac('sha256', process.env.YOOKASSA_WEBHOOK_SECRET ?? '')
      .update(JSON.stringify(payload))
      .digest('hex');

    if (process.env.YOOKASSA_WEBHOOK_SECRET && signature !== expectedSig) {
      throw new BadRequestException('Невалидная подпись YooKassa webhook');
    }
  }

  private async setPastDue(tenantId: string, reason: string): Promise<void> {
    await this.prisma.subscription.update({
      where: { tenantId },
      data: { status: 'PAST_DUE' },
    });
    this.eventEmitter.emit('subscription.pastDue', { tenantId, reason });
  }

  private async handleRefundSucceeded(providerRefund: any, provider: string): Promise<void> {
    const payment = await this.prisma.payment.findFirst({
      where: { externalPaymentId: providerRefund.payment_id ?? providerRefund.charge },
    });
    if (!payment) return;

    await this.prisma.payment.update({
      where: { id: payment.id },
      data: {
        status: 'REFUNDED',
        refundedAt: new Date(),
        refundedAmount: providerRefund.amount?.value
          ? Math.round(parseFloat(providerRefund.amount.value) * 100)
          : payment.amount,
      },
    });

    this.eventEmitter.emit('payment.refunded', { tenantId: payment.tenantId, paymentId: payment.id });
  }
}
```

---

## 4. Upgrade / Downgrade Flow

### 4.1. Схема состояний

```
UPGRADE (немедленно):
  STANDARD → PRO
  ├── Рассчитываем delta (proratedAmount)
  ├── delta > 0 → создаём новый Payment → redirect к провайдеру
  ├── delta = 0 → бесплатный апгрейд (уже переплачено)
  └── После оплаты: Subscription.plan = PRO, TenantLimits обновляются

DOWNGRADE (отложенно):
  PRO → STANDARD
  ├── Subscription.pendingPlanChange = 'STANDARD'
  ├── Subscription.pendingPlanChangeAt = currentPeriodEnd
  ├── Owner уведомлён по email
  └── CRON в 00:00: применяем, обновляем план + лимиты
```

### 4.2. Защита при даунгрейде (превышение текущих лимитов)

```typescript
// Перед подтверждением даунгрейда проверяем, что текущие ресурсы вписываются
async validateDowngrade(tenantId: string, newPlan: string): Promise<DowngradeValidation> {
  const limits = await this.prisma.tenantLimits.findUnique({ where: { tenantId } });
  const newLimits = PLAN_LIMITS[newPlan];
  const warnings: string[] = [];

  if (newLimits.maxRestaurants !== null && limits.currentRestaurants > newLimits.maxRestaurants) {
    warnings.push(
      `Ресторанов: ${limits.currentRestaurants} > лимит ${newLimits.maxRestaurants}. Необходимо удалить ${limits.currentRestaurants - newLimits.maxRestaurants}.`,
    );
  }

  if (newLimits.maxGuests !== null && limits.currentGuests > newLimits.maxGuests) {
    warnings.push(
      `Гостей: ${limits.currentGuests} > лимит ${newLimits.maxGuests}. Новые гости не будут добавляться.`,
    );
  }

  return { canDowngrade: warnings.length === 0, warnings };
}
```

---

## 5. PASTDUE & Dunning Flow

### 5.1. Последовательность уведомлений

```
День 0: оплата не прошла → статус PAST_DUE
        Email/Telegram: "⚠️ Оплата не прошла"

День +1: повтор списания #1
         Email: "Повторная попытка оплаты"

День +3: повтор списания #2
         Email: "Ваш аккаунт будет заблокирован через 4 дня"

День +7: повтор списания #3
         Email: "Последнее предупреждение — 1 день"

День +8 (итого): статус CANCELLED_NON_PAYMENT
         Tenant переходит в READ-ONLY режим
         Email: "Подписка отменена. Данные хранятся 30 дней."
```

```typescript
// src/modules/billing/dunning.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { PrismaService } from '../../database/prisma.service';
import { PaymentService } from './payment.service';
import { NotificationService } from '../notifications/notification.service';
import { addDays } from 'date-fns';

@Injectable()
export class DunningService {
  private readonly logger = new Logger(DunningService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly paymentService: PaymentService,
    private readonly notificationService: NotificationService,
  ) {}

  // Запускаем каждую ночь в 03:00
  @Cron('0 3 * * *')
  async runDunning(): Promise<void> {
    this.logger.log('Running dunning CRON...');

    const pastDueSubs = await this.prisma.subscription.findMany({
      where: { status: 'PAST_DUE' },
      include: { tenant: true },
    });

    for (const sub of pastDueSubs) {
      await this.processDunning(sub);
    }
  }

  private async processDunning(sub: any): Promise<void> {
    const lastFailedPayment = await this.prisma.payment.findFirst({
      where: { subscriptionId: sub.id, status: 'FAILED' },
      orderBy: { createdAt: 'desc' },
    });

    if (!lastFailedPayment) return;

    const daysSinceFail = Math.floor(
      (Date.now() - lastFailedPayment.createdAt.getTime()) / (1000 * 60 * 60 * 24),
    );

    // Определяем действие по дням
    if (daysSinceFail === 1) {
      await this.retryPayment(sub, 'Повторная попытка (день 1)');
      await this.notifyPastDue(sub, 1);
    } else if (daysSinceFail === 3) {
      await this.retryPayment(sub, 'Повторная попытка (день 3)');
      await this.notifyPastDue(sub, 3);
    } else if (daysSinceFail === 7) {
      await this.retryPayment(sub, 'Повторная попытка (день 7)');
      await this.notifyPastDue(sub, 7);
    } else if (daysSinceFail >= 8) {
      await this.cancelNonPayment(sub);
    }
  }

  private async retryPayment(sub: any, description: string): Promise<void> {
    try {
      await this.paymentService.renewSubscription(sub.tenantId);
    } catch (err) {
      this.logger.error(`Dunning retry failed for ${sub.tenantId}: ${err.message}`);
    }
  }

  private async notifyPastDue(sub: any, day: number): Promise<void> {
    const messages: Record<number, string> = {
      1: '⚠️ Оплата не прошла. Обновите платёжный метод.',
      3: '🔴 Аккаунт будет заблокирован через 5 дней.',
      7: '🚨 Последнее предупреждение! Завтра подписка будет отменена.',
    };

    await this.notificationService.sendToTenantOwners(sub.tenantId, {
      type: 'PAYMENT_FAILED',
      channel: ['TELEGRAM', 'EMAIL'],
      title: `Проблема с оплатой (день ${day})`,
      message: messages[day],
      data: { updatePaymentUrl: 'https://app.max-loyalty.com/billing/payment-method' },
    });
  }

  private async cancelNonPayment(sub: any): Promise<void> {
    await this.prisma.$transaction(async (tx) => {
      await tx.subscription.update({
        where: { id: sub.id },
        data: {
          status: 'NON_PAYMENT',
          cancelledAt: new Date(),
          autoRenew: false,
        },
      });

      // Переводим tenant в READ-ONLY
      await tx.tenant.update({
        where: { id: sub.tenantId },
        data: {
          isActive: false,
          blockedReason: 'Подписка отменена из-за неоплаты',
        },
      });

      await tx.subscriptionHistory.create({
        data: {
          tenantId: sub.tenantId,
          subscriptionId: sub.id,
          event: 'CANCELLED_NON_PAYMENT',
          planFrom: sub.plan,
          planTo: sub.plan,
        },
      });
    });

    await this.notificationService.sendToTenantOwners(sub.tenantId, {
      type: 'SUBSCRIPTION_CANCELLED_NON_PAYMENT',
      channel: ['TELEGRAM', 'EMAIL'],
      title: '❌ Подписка отменена',
      message: 'Подписка отменена из-за неоплаты. Данные хранятся 30 дней. Оплатите для восстановления.',
      data: { restoreUrl: 'https://app.max-loyalty.com/billing/restore' },
    });
  }
}
```

---

## 6. Invoice Service

```typescript
// src/modules/billing/invoice.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../database/prisma.service';

const VAT_RATE = 0.20; // 20% НДС

@Injectable()
export class InvoiceService {
  constructor(private readonly prisma: PrismaService) {}

  // Создание инвойса внутри транзакции
  async createFromPaymentTx(tx: any, payment: any) {
    const amountWithoutVat = Math.round(payment.amount / (1 + VAT_RATE));
    const vatAmount = payment.amount - amountWithoutVat;

    const invoiceNumber = await this.generateInvoiceNumber(tx, payment.tenantId);

    return tx.invoice.create({
      data: {
        tenantId: payment.tenantId,
        subscriptionId: payment.subscriptionId,
        paymentId: payment.id,
        invoiceNumber,
        amountWithoutVat,
        vatRate: 20,
        vatAmount,
        amountWithVat: payment.amount,
        currency: payment.currency,
        status: 'PAID',
        dueDate: new Date(),
        paidAt: new Date(),
      },
    });
  }

  // Создание инвойса для ручной оплаты (ULTIMATE/CUSTOM)
  async createManualInvoice(tenantId: string, dto: CreateManualInvoiceDto) {
    const amountWithoutVat = Math.round(dto.amount / 1.2);
    const vatAmount = dto.amount - amountWithoutVat;
    const invoiceNumber = await this.generateInvoiceNumber(this.prisma, tenantId);

    const invoice = await this.prisma.invoice.create({
      data: {
        tenantId,
        subscriptionId: dto.subscriptionId,
        invoiceNumber,
        amountWithoutVat,
        vatRate: 20,
        vatAmount,
        amountWithVat: dto.amount,
        currency: 'RUB',
        status: 'PENDING',
        dueDate: dto.dueDate,
        description: dto.description,
      },
    });

    // Генерируем PDF асинхронно
    await this.queuePdfGeneration(invoice.id);

    return invoice;
  }

  // Подтверждение ручной оплаты (только Owner платформы)
  async confirmManualPayment(invoiceId: string, actorId: string) {
    const invoice = await this.prisma.invoice.findUnique({ where: { id: invoiceId } });
    if (!invoice) throw new Error('Инвойс не найден');

    await this.prisma.$transaction(async (tx) => {
      await tx.invoice.update({
        where: { id: invoiceId },
        data: { status: 'PAID', paidAt: new Date() },
      });

      // Создаём Payment с типом MANUAL
      const payment = await tx.payment.create({
        data: {
          tenantId: invoice.tenantId,
          subscriptionId: invoice.subscriptionId!,
          amount: invoice.amountWithVat,
          currency: invoice.currency,
          status: 'SUCCEEDED',
          paymentProvider: 'MANUAL',
          description: `Ручная оплата инвойса ${invoice.invoiceNumber}`,
          paidAt: new Date(),
        },
      });

      await tx.activityLog.create({
        data: {
          tenantId: invoice.tenantId,
          actorId,
          action: 'INVOICE_PAID_MANUALLY',
          entityType: 'INVOICE',
          entityId: invoiceId,
        },
      });
    });
  }

  private async generateInvoiceNumber(prisma: any, tenantId: string): Promise<string> {
    const year = new Date().getFullYear();
    const count = await prisma.invoice.count({
      where: { tenantId, createdAt: { gte: new Date(`${year}-01-01`) } },
    });
    return `ML-${year}-${String(count + 1).padStart(4, '0')}`;
  }

  private async queuePdfGeneration(invoiceId: string): Promise<void> {
    // В реализации: BullMQ job → puppeteer/html-pdf → upload S3/R2 → update invoice.pdfUrl
    // Здесь заглушка
  }
}
```

---

## 7. Billing Metrics (MRR/ARR/Churn)

```typescript
// src/modules/billing/billing-metrics.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../database/prisma.service';

@Injectable()
export class BillingMetricsService {
  constructor(private readonly prisma: PrismaService) {}

  // === MRR — Monthly Recurring Revenue ===
  async getMRR(): Promise<number> {
    const activeSubs = await this.prisma.subscription.findMany({
      where: { status: { in: ['ACTIVE', 'TRIAL'] } },
    });

    return activeSubs.reduce((sum, sub) => {
      const monthly = sub.billingPeriod === 'YEARLY'
        ? (sub.priceYearly?.toNumber() ?? 0) / 12
        : sub.priceMonthly.toNumber();
      return sum + monthly;
    }, 0);
  }

  // === ARR — Annual Recurring Revenue ===
  async getARR(): Promise<number> {
    const mrr = await this.getMRR();
    return mrr * 12;
  }

  // === Churn Rate (за месяц) ===
  async getChurnRate(periodDays: number = 30): Promise<number> {
    const since = new Date(Date.now() - periodDays * 24 * 60 * 60 * 1000);

    const [cancelled, totalStart] = await Promise.all([
      this.prisma.subscription.count({
        where: {
          status: { in: ['CANCELLED', 'NON_PAYMENT'] },
          cancelledAt: { gte: since },
        },
      }),
      this.prisma.subscription.count({
        where: {
          status: { in: ['ACTIVE', 'TRIAL', 'CANCELLED', 'NON_PAYMENT', 'PAST_DUE'] },
          createdAt: { lt: since },
        },
      }),
    ]);

    if (totalStart === 0) return 0;
    return Math.round((cancelled / totalStart) * 100 * 100) / 100; // 2 знака
  }

  // === ARPU — Average Revenue Per User ===
  async getARPU(): Promise<number> {
    const [mrr, activeCount] = await Promise.all([
      this.getMRR(),
      this.prisma.subscription.count({
        where: { status: { in: ['ACTIVE', 'TRIAL'] } },
      }),
    ]);
    return activeCount > 0 ? Math.round(mrr / activeCount) : 0;
  }

  // === LTV — Lifetime Value ===
  async getLTV(): Promise<number> {
    const churnRate = await this.getChurnRate();
    const arpu = await this.getARPU();
    if (churnRate === 0) return arpu * 36; // 3 года максимум
    return Math.round(arpu / (churnRate / 100));
  }

  // === Revenue by Plan (для диаграммы) ===
  async getRevenueByPlan(): Promise<Record<string, number>> {
    const subs = await this.prisma.subscription.findMany({
      where: { status: { in: ['ACTIVE', 'TRIAL'] } },
    });

    return subs.reduce((acc, sub) => {
      const monthly = sub.billingPeriod === 'YEARLY'
        ? (sub.priceYearly?.toNumber() ?? 0) / 12
        : sub.priceMonthly.toNumber();
      acc[sub.plan] = (acc[sub.plan] ?? 0) + monthly;
      return acc;
    }, {} as Record<string, number>);
  }

  // === Полная сводка для Owner Dashboard ===
  async getSummary() {
    const [mrr, arr, churn, arpu, ltv, byPlan] = await Promise.all([
      this.getMRR(),
      this.getARR(),
      this.getChurnRate(),
      this.getARPU(),
      this.getLTV(),
      this.getRevenueByPlan(),
    ]);

    return { mrr, arr, churnRate: churn, arpu, ltv, revenueByPlan: byPlan };
  }

  // === Failed/Refunded/Chargeback за период ===
  async getPaymentStats(since: Date) {
    const [succeeded, failed, refunded, chargedBack] = await Promise.all([
      this.prisma.payment.aggregate({
        where: { status: 'SUCCEEDED', paidAt: { gte: since } },
        _sum: { amount: true },
        _count: true,
      }),
      this.prisma.payment.count({ where: { status: 'FAILED', createdAt: { gte: since } } }),
      this.prisma.payment.aggregate({
        where: { status: 'REFUNDED', refundedAt: { gte: since } },
        _sum: { refundedAmount: true },
        _count: true,
      }),
      this.prisma.payment.count({ where: { status: 'CHARGEBACK', createdAt: { gte: since } } }),
    ]);

    return { succeeded, failed, refunded, chargedBack };
  }
}
```

---

## 8. Manual Payments (ULTIMATE/CUSTOM)

```typescript
// Для ULTIMATE и CUSTOM — выставляем инвойс вручную:

// 1. Owner платформы создаёт инвойс
// POST /billing/invoices/manual  {tenantId, amount, dueDate, description}

// 2. Инвойс отправляется на email: status = PENDING_MANUAL_PAYMENT

// 3. RestaurantAdmin оплачивает банковским переводом

// 4. Owner подтверждает: PATCH /billing/invoices/:id/confirm-payment
//    → Invoice.status = PAID, Payment создаётся с provider = MANUAL, Subscription активируется

export const MANUAL_PAYMENT_PLANS = ['ULTIMATE', 'CUSTOM'] as const;
```

---

## 9. Refund & Chargeback

```typescript
// src/modules/billing/refund.service.ts
import { Injectable, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../../database/prisma.service';
import { EventEmitter2 } from '@nestjs/event-emitter';

@Injectable()
export class RefundService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  // === Возврат средств ===
  async processRefund(
    paymentId: string,
    amount: number | null, // null = полный возврат
    reason: string,
    actorId: string,
  ) {
    const payment = await this.prisma.payment.findUnique({ where: { id: paymentId } });
    if (!payment) throw new BadRequestException('Платёж не найден');
    if (payment.status !== 'SUCCEEDED') {
      throw new BadRequestException('Возврат возможен только для успешных платежей');
    }

    const refundAmount = amount ?? payment.amount;
    const isPartial = refundAmount < payment.amount;

    // Делаем возврат через провайдера
    if (payment.paymentProvider === 'YOOKASSA') {
      await this.refundYooKassa(payment.externalPaymentId!, refundAmount);
    } else if (payment.paymentProvider === 'STRIPE') {
      await this.refundStripe(payment.externalPaymentId!, refundAmount);
    }

    await this.prisma.$transaction(async (tx) => {
      await tx.payment.update({
        where: { id: paymentId },
        data: {
          status: isPartial ? 'PARTIALLY_REFUNDED' : 'REFUNDED',
          refundedAt: new Date(),
          refundedAmount: refundAmount,
        },
      });

      // При полном возврате — отменяем подписку
      if (!isPartial) {
        await tx.subscription.update({
          where: { id: payment.subscriptionId },
          data: { status: 'CANCELLED_REFUND', cancelledAt: new Date() },
        });
      }

      await tx.activityLog.create({
        data: {
          tenantId: payment.tenantId,
          actorId,
          action: 'PAYMENT_REFUNDED',
          entityType: 'PAYMENT',
          entityId: paymentId,
          newValue: { refundAmount, reason, partial: isPartial },
        },
      });
    });

    this.eventEmitter.emit('payment.refunded', {
      tenantId: payment.tenantId,
      paymentId,
      amount: refundAmount,
    });
  }

  // === Chargeback (банк принудительно возвращает средства) ===
  async handleChargeback(paymentId: string, chargebackAmount: number) {
    const payment = await this.prisma.payment.findUnique({ where: { id: paymentId } });
    if (!payment) return;

    await this.prisma.$transaction(async (tx) => {
      await tx.payment.update({
        where: { id: paymentId },
        data: {
          status: 'CHARGEBACK',
          chargebackAmount,
        },
      });

      // Блокируем tenant как предосторожность
      await tx.tenant.update({
        where: { id: payment.tenantId },
        data: { isActive: false, blockedReason: 'Chargeback получен' },
      });

      await tx.subscription.update({
        where: { id: payment.subscriptionId },
        data: { status: 'CANCELLED_CHARGEBACK' },
      });
    });

    // Уведомляем Owner платформы — влияет на MRR/LTV
    this.eventEmitter.emit('payment.chargeback', {
      tenantId: payment.tenantId,
      paymentId,
      amount: chargebackAmount,
    });
  }

  private async refundYooKassa(externalPaymentId: string, amount: number): Promise<void> {
    await fetch('https://api.yookassa.ru/v3/refunds', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Idempotence-Key': `refund-${externalPaymentId}-${Date.now()}`,
        Authorization: `Basic ${Buffer.from(
          `${process.env.YOOKASSA_SHOP_ID}:${process.env.YOOKASSA_SECRET_KEY}`,
        ).toString('base64')}`,
      },
      body: JSON.stringify({
        payment_id: externalPaymentId,
        amount: { value: (amount / 100).toFixed(2), currency: 'RUB' },
      }),
    });
  }

  private async refundStripe(externalPaymentId: string, amount: number): Promise<void> {
    const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
    await stripe.refunds.create({ payment_intent: externalPaymentId, amount });
  }
}
```

---

## 10. Prisma Schema

```prisma
// ============================================
// SUBSCRIPTION
// ============================================
model Subscription {
  id       String @id @default(uuid())
  tenantId String @unique
  tenant   Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  plan          SubscriptionPlan
  status        SubscriptionStatus @default(TRIAL)
  billingPeriod BillingPeriod      @default(MONTHLY)

  priceMonthly Decimal  @db.Decimal(10, 2)
  priceYearly  Decimal? @db.Decimal(10, 2)
  currency     String   @default("RUB")

  // Даты
  trialEndsAt          DateTime?
  currentPeriodStart   DateTime
  currentPeriodEnd     DateTime
  cancelledAt          DateTime?
  autoRenew            Boolean  @default(true)

  // Запланированная смена плана
  pendingPlanChange    String?
  pendingBillingPeriod String?
  pendingPlanChangeAt  DateTime?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  payments            Payment[]
  invoices            Invoice[]
  subscriptionHistory SubscriptionHistory[]

  @@index([status])
  @@index([currentPeriodEnd])
  @@index([pendingPlanChangeAt])
}

// ============================================
// SUBSCRIPTION HISTORY (audit trail)
// ============================================
model SubscriptionHistory {
  id             String       @id @default(uuid())
  tenantId       String
  subscriptionId String
  subscription   Subscription @relation(fields: [subscriptionId], references: [id], onDelete: Cascade)

  event         String   // ACTIVATED, UPGRADED, DOWNGRADED, CANCELLED, etc.
  planFrom      String
  planTo        String
  effectiveDate DateTime?
  reason        String?
  actorId       String?
  paymentId     String?

  createdAt DateTime @default(now())

  @@index([tenantId])
  @@index([subscriptionId])
  @@index([createdAt])
}

// ============================================
// PAYMENT
// ============================================
model Payment {
  id             String       @id @default(uuid())
  tenantId       String
  tenant         Tenant       @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  subscriptionId String
  subscription   Subscription @relation(fields: [subscriptionId], references: [id], onDelete: Cascade)

  amount   Decimal         @db.Decimal(10, 2)
  currency String          @default("RUB")
  status   PaymentStatus   @default(PENDING)

  paymentProvider  PaymentProvider
  externalPaymentId String?        @unique
  paymentMethodId   String?       // сохранённый токен

  description     String?
  errorMessage    String?
  metadata        Json?

  // Refund
  refundedAmount   Int?
  chargebackAmount Int?

  paidAt      DateTime?
  refundedAt  DateTime?
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  invoice Invoice?

  @@index([subscriptionId])
  @@index([tenantId])
  @@index([status])
  @@index([createdAt])
  @@index([externalPaymentId])
}

// ============================================
// INVOICE
// ============================================
model Invoice {
  id             String        @id @default(uuid())
  tenantId       String
  tenant         Tenant        @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  subscriptionId String?
  subscription   Subscription? @relation(fields: [subscriptionId], references: [id])
  paymentId      String?       @unique
  payment        Payment?      @relation(fields: [paymentId], references: [id])

  invoiceNumber  String        @unique // ML-2026-0001
  description    String?

  amountWithoutVat Int
  vatRate          Int     @default(20)
  vatAmount        Int
  amountWithVat    Int
  currency         String  @default("RUB")

  status  InvoiceStatus @default(PENDING)
  dueDate DateTime
  paidAt  DateTime?
  pdfUrl  String?       // S3/R2 URL

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([tenantId])
  @@index([status])
  @@index([createdAt])
}

enum InvoiceStatus {
  PENDING
  PENDING_MANUAL_PAYMENT
  PAID
  CANCELLED
  OVERDUE
}

enum SubscriptionStatus {
  ACTIVE
  TRIAL
  PAST_DUE
  NON_PAYMENT
  CANCELLED
  CANCELLED_REFUND
  CANCELLED_CHARGEBACK
}
```

---

## 11. API Endpoints

### 11.1. Billing Controller

```typescript
// src/modules/billing/billing.controller.ts
@ApiTags('Billing')
@ApiBearerAuth()
@UseGuards(JwtAuthGuard, TenantGuard)
@Controller('billing')
export class BillingController {
  // GET /billing/subscription — текущая подписка
  @Get('subscription')
  async getSubscription(@TenantId() tenantId: string) {
    return this.subscriptionService.getSubscription(tenantId);
  }

  // POST /billing/checkout — создать платёжную сессию
  @Post('checkout')
  @RequirePermissions(Permission.MANAGE_BILLING)
  async createCheckout(
    @TenantId() tenantId: string,
    @Body() dto: CreateCheckoutDto,
  ) {
    return this.paymentService.createCheckout(tenantId, dto);
  }

  // POST /billing/change-plan — апгрейд/даунгрейд
  @Post('change-plan')
  @RequirePermissions(Permission.MANAGE_BILLING)
  async changePlan(
    @TenantId() tenantId: string,
    @Body() dto: ChangePlanDto,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.subscriptionService.changePlan(
      tenantId, dto.plan, dto.billingPeriod, dto.reason, user.sub,
    );
  }

  // GET /billing/change-plan/validate — проверка возможности даунгрейда
  @Get('change-plan/validate')
  async validateChangePlan(
    @TenantId() tenantId: string,
    @Query('plan') plan: string,
  ) {
    return this.subscriptionService.validateDowngrade(tenantId, plan);
  }

  // POST /billing/cancel — отмена
  @Post('cancel')
  @RequirePermissions(Permission.MANAGE_BILLING)
  async cancel(
    @TenantId() tenantId: string,
    @Body('reason') reason: string,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.subscriptionService.cancel(tenantId, reason, user.sub);
  }

  // GET /billing/invoices — список инвойсов
  @Get('invoices')
  async getInvoices(
    @TenantId() tenantId: string,
    @Query() paginationDto: PaginationDto,
  ) {
    return this.prisma.invoice.findMany({
      where: { tenantId },
      orderBy: { createdAt: 'desc' },
      take: paginationDto.take ?? 20,
      skip: paginationDto.skip ?? 0,
    });
  }

  // GET /billing/metrics — MRR/ARR/Churn (только Owner)
  @Get('metrics')
  @RequirePermissions(Permission.PLATFORM_ADMIN)
  async getMetrics() {
    return this.metricsService.getSummary();
  }
}

// Webhooks Controller (без JWT!)
@Controller('webhooks/payment')
export class PaymentWebhookController {
  // POST /webhooks/payment/yookassa
  @Post('yookassa')
  async yookassaWebhook(
    @Body() payload: any,
    @Headers('X-YooKassa-Signature') signature: string,
  ) {
    await this.paymentService.handleYooKassaWebhook(payload, signature);
    return { received: true };
  }

  // POST /webhooks/payment/stripe
  @Post('stripe')
  async stripeWebhook(
    @Body() rawBody: Buffer,
    @Headers('stripe-signature') signature: string,
  ) {
    await this.paymentService.handleStripeWebhook(rawBody, signature);
    return { received: true };
  }

  // POST /billing/invoices/manual (только Owner платформы)
  @Post('invoices/manual')
  @RequirePermissions(Permission.PLATFORM_ADMIN)
  async createManualInvoice(@Body() dto: CreateManualInvoiceDto) {
    return this.invoiceService.createManualInvoice(dto.tenantId, dto);
  }

  // PATCH /billing/invoices/:id/confirm-payment
  @Patch('invoices/:id/confirm-payment')
  @RequirePermissions(Permission.PLATFORM_ADMIN)
  async confirmPayment(
    @Param('id') invoiceId: string,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.invoiceService.confirmManualPayment(invoiceId, user.sub);
  }
}
```

### 11.2. Сводная таблица эндпоинтов

| Метод | Маршрут | Роль | Описание |
|---|---|---|---|
| GET | `/billing/subscription` | All | Текущая подписка |
| POST | `/billing/checkout` | Owner/Admin | Создать сессию оплаты |
| POST | `/billing/change-plan` | Owner | Апгрейд/даунгрейд |
| GET | `/billing/change-plan/validate` | Owner | Проверка даунгрейда |
| POST | `/billing/cancel` | Owner | Отменить подписку |
| GET | `/billing/invoices` | Owner/Admin | Список инвойсов |
| GET | `/billing/metrics` | PLATFORM_OWNER | MRR/ARR/Churn |
| POST | `/webhooks/payment/yookassa` | Public | YooKassa webhook |
| POST | `/webhooks/payment/stripe` | Public | Stripe webhook |
| POST | `/billing/invoices/manual` | PLATFORM_OWNER | Ручной инвойс |
| PATCH | `/billing/invoices/:id/confirm-payment` | PLATFORM_OWNER | Подтвердить оплату |
| POST | `/billing/refund/:paymentId` | PLATFORM_OWNER | Оформить возврат |

---

## 12. CRON Jobs

```typescript
// src/modules/billing/billing-cron.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class BillingCronService {
  private readonly logger = new Logger(BillingCronService.name);

  // Каждую ночь в 02:00 — автопродление подписок
  @Cron('0 2 * * *')
  async renewSubscriptions(): Promise<void> {
    this.logger.log('[CRON] Renewal check...');
    const expiring = await this.prisma.subscription.findMany({
      where: {
        status: 'ACTIVE',
        autoRenew: true,
        currentPeriodEnd: { lte: new Date() },
      },
    });

    for (const sub of expiring) {
      await this.paymentService.renewSubscription(sub.tenantId);
    }
    this.logger.log(`[CRON] Renewed ${expiring.length} subscriptions`);
  }

  // Каждую ночь в 03:00 — dunning flow
  @Cron('0 3 * * *')
  async runDunning(): Promise<void> {
    await this.dunningService.runDunning();
  }

  // Каждую ночь в 01:00 — применяем запланированные даунгрейды
  @Cron('0 1 * * *')
  async applyPendingPlanChanges(): Promise<void> {
    await this.subscriptionService.applyPendingPlanChanges();
  }

  // Каждую ночь в 04:00 — проверяем истёкшие триалы
  @Cron('0 4 * * *')
  async checkTrials(): Promise<void> {
    const expiredTrials = await this.prisma.subscription.findMany({
      where: {
        status: 'TRIAL',
        trialEndsAt: { lte: new Date() },
      },
    });

    for (const sub of expiredTrials) {
      await this.prisma.subscription.update({
        where: { id: sub.id },
        data: { status: 'PAST_DUE' },
      });

      await this.notificationService.sendToTenantOwners(sub.tenantId, {
        type: 'TRIAL_EXPIRED',
        channel: ['TELEGRAM', 'EMAIL'],
        title: '⏰ Триал завершён',
        message: 'Бесплатный период завершён. Оплатите подписку для продолжения работы.',
        data: { payUrl: 'https://app.max-loyalty.com/billing/checkout' },
      });
    }

    this.logger.log(`[CRON] Trial expired: ${expiredTrials.length}`);
  }
}
```

---

## 13. Тесты & Чеклист

### 13.1. Сценарии тестирования

| # | Сценарий | Ожидаемый результат |
|---|---|---|
| B1 | Owner оплачивает подписку через YooKassa | Redirect → webhook → ACTIVE, TenantLimits обновлены |
| B2 | Owner оплачивает подписку через Stripe | Аналогично B1 |
| B3 | Апгрейд STANDARD → PRO с delta 850 ₽ | Создаётся Payment с amount=850, redirect к провайдеру |
| B4 | Даунгрейд PRO → STANDARD | pendingPlanChange записан, уведомление отправлено |
| B5 | CRON применяет даунгрейд по истечении периода | Subscription.plan = STANDARD, TenantLimits снижены |
| B6 | Автопродление с сохранённым методом | Повторное списание прошло → ACTIVE |
| B7 | Оплата не прошла (Day 0) | status = PAST_DUE, уведомление Owner |
| B8 | Dunning Day +8 без оплаты | status = NON_PAYMENT, tenant заблокирован |
| B9 | Истёк триал — нет оплаты | TRIAL → PAST_DUE, уведомление |
| B10 | Полный возврат через YooKassa | Payment.status = REFUNDED, Subscription = CANCELLED_REFUND |
| B11 | Chargeback от банка | Payment.status = CHARGEBACK, tenant заблокирован |
| B12 | Ручной инвойс ULTIMATE → подтверждение Owner | Invoice.status = PAID, Subscription активирована |
| B13 | Webhook с дублирующим externalPaymentId | Идемпотентность — дубликат отклонён |
| B14 | Невалидная подпись YooKassa webhook | HTTP 400 |
| B15 | Превышение лимита ресторанов при апгрейде | Предупреждение в validateDowngrade |

### 13.2. Чеклист реализации

**P0 — Критично:**
- [ ] PaymentService создаёт Payment в PENDING до редиректа
- [ ] Webhook подписи верифицируются (YooKassa IP + HMAC, Stripe constructEvent)
- [ ] Идемпотентность: `Idempotence-Key` для YooKassa, `paymentId` для Stripe
- [ ] PCI DSS: сохраняем только токен/paymentMethodId, никогда — номер карты
- [ ] SubscriptionService.activate() обновляет TenantLimits атомарно
- [ ] DunningService: день 0, +1, +3, +7, +8 — правильная последовательность
- [ ] CRON: renewal в 02:00, dunning в 03:00, trial-check в 04:00

**P1 — Важно:**
- [ ] validateDowngrade() предупреждает о превышении лимитов
- [ ] InvoiceService генерирует уникальный номер ML-YYYY-NNNN
- [ ] BillingMetricsService: MRR/ARR/Churn/ARPU/LTV корректно считаются
- [ ] Refund уменьшает refundedAmount атомарно
- [ ] Chargeback блокирует tenant и создаёт запись в SubscriptionHistory

**P2 — Желательно:**
- [ ] PDF генерация инвойсов через BullMQ + puppeteer → S3/R2
- [ ] Scheduled Reports для Owner (ежедневно/еженедельно CSV/XLSX)
- [ ] Stripe: настройка через Stripe Customer + Payment Method API
- [ ] Уведомление за 7 дней до конца триала
- [ ] Восстановление аккаунта после NON_PAYMENT через `/billing/restore`

---

*S-05 завершён. Следующая часть: **S-06** — Loyalty Engine (правила начисления, уровни, промо-акции, CRON истечения баллов)*
