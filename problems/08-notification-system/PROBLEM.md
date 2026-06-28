# Notification System

**Difficulty:** Intermediate
**Category:** Observer + Multi-Channel Delivery
**Time Box:** 45 min (discussion) / 90 min (machine coding)
**Key Patterns:** Observer (subscribers), Strategy (delivery channels), Builder (notification construction)
**Asked at:** Amazon, Meta, LinkedIn, Uber, Grab, Zomato

---

## Problem Statement

Design a notification system that can send notifications to users across multiple channels — email, SMS, push notification, and in-app notification. Different users have different channel preferences. The system should handle delivery failures with retry logic, and support different notification types (marketing, transactional, alert).

---

## Actors

| Actor | Description |
|-------|-------------|
| Producer | Any service that triggers a notification (order service, auth service, etc.) |
| Notification System | Routes notifications to appropriate channels per user preference |
| User | Receives notifications; has channel preferences |
| Channel Providers | Email (SendGrid), SMS (Twilio), Push (FCM/APNs) — external |

---

## Functional Requirements

**Core (must have for MVP):**
- FR1: Producer sends a notification event (user id, message, notification type)
- FR2: System looks up user's channel preferences
- FR3: System delivers notification via preferred channels (could be multiple)
- FR4: Each channel delivery is attempted independently — one channel failing doesn't block others
- FR5: Failed deliveries are retried (up to N attempts with backoff)
- FR6: System tracks delivery status per notification per channel

**Secondary (implement if time allows):**
- FR7: User can subscribe/unsubscribe from specific notification types per channel
- FR8: Rate limiting — don't spam user with same notification type
- FR9: Priority levels — alerts bypass rate limits, marketing is low priority
- FR10: Batching — group low-priority notifications and send as digest

---

## Non-Functional Requirements

- **Reliability:** At-least-once delivery for critical notifications
- **Decoupling:** Producer doesn't know which channels exist
- **Extensibility:** Adding a new channel (WhatsApp) requires zero changes to producer or router
- **Concurrency:** Multiple notifications can be processed in parallel

---

## Constraints and Assumptions

- External channel providers are mocked (assume they have a `send(to, message)` method)
- User preferences stored in-memory for interview
- Retry is synchronous (async retry queue is a bonus discussion)
- Notification types: TRANSACTIONAL, MARKETING, ALERT
- Out of scope: delivery receipts from end devices, unsubscribe links, GDPR opt-out

---

## Good Clarifying Questions to Ask

1. What channels are needed — email, SMS, push, in-app?
2. User can have multiple preferred channels simultaneously?
3. How does retry work — immediately, with backoff, or dead-letter queue?
4. Should one channel failure block other channels for same notification?
5. Rate limiting needed? (e.g., max 5 marketing emails per day)
6. Priority levels?
7. Does the system need to track delivery status per notification?

---

## Key Entities (hints — try identifying yourself first)

<details>
<summary>Reveal entities</summary>

- **Notification** — userId, message, type (TRANSACTIONAL/MARKETING/ALERT), priority, metadata
- **NotificationService** — entry point; looks up user preferences; dispatches to channels
- **NotificationChannel** — interface: `send(Notification, User): DeliveryResult`
- **EmailChannel** — sends via email provider
- **SMSChannel** — sends via SMS provider
- **PushChannel** — sends via FCM/APNs
- **InAppChannel** — stores in notification inbox
- **UserPreferences** — which channels a user wants per notification type
- **RetryPolicy** — max attempts, backoff strategy
- **DeliveryResult** — SUCCESS, FAILURE, RETRY
- **DeliveryLog** — tracks attempt count, status, timestamp per channel

</details>
