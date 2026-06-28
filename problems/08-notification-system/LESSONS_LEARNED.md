# Lessons Learned — Notification System

---

## Mistake 1: If-Else on Channel Type (No Interface)

**What happens:**
```java
void send(Notification n) {
    if (n.getChannel() == EMAIL) sendEmail(n);
    else if (n.getChannel() == SMS) sendSMS(n);
    else if (n.getChannel() == PUSH) sendPush(n);
}
```

**Why it's wrong:** Adding WhatsApp requires editing this method — OCP violation.

**Fix:**
```java
interface NotificationChannel { DeliveryResult send(Notification n, User u); }
// New channel = new class, zero changes to NotificationService.send()
```

**Gap dimension:** Design Patterns + SOLID (Dimensions 3, 4)

---

## Mistake 2: Email Failure Stops SMS Attempt

**What happens:** All channels dispatched in a try-catch that aborts on first failure:
```java
try {
    emailChannel.send(notification, user);
    smsChannel.send(notification, user); // never reached if email throws
} catch (Exception e) { ... }
```

**Why it's wrong:** Channels are independent. If email provider is down, SMS should still go out.

**Fix:** Separate try-catch per channel:
```java
for (NotificationChannel channel : channels) {
    try {
        deliverWithRetry(notification, user, channel);
    } catch (Exception e) {
        log.error("Channel {} failed permanently", channel.getChannelType(), e);
        // continue to next channel
    }
}
```

**Gap dimension:** Entity Modeling / Requirements (Dimensions 2, 1)

---

## Mistake 3: No Retry Logic

**What happens:** Channel throws exception → notification lost. No retry.

**Why it's wrong:** Transient failures (network blip, provider rate limit) should be retried. For TRANSACTIONAL notifications (OTP, password reset), loss is critical.

**Fix:**
```java
// Distinguish retriable vs permanent failure in DeliveryResult
enum DeliveryResult { SUCCESS, FAILURE_PERMANENT, FAILURE_RETRY }
// Retry only on FAILURE_RETRY with exponential backoff
```

**Gap dimension:** Requirements (Dimension 1) — should have been caught during requirements phase

---

## Mistake 4: User Preferences Hardcoded

**What happens:** Candidate assumes all users want email, or hardcodes channels per notification type without a preference model.

**Fix:**
```java
class UserPreferenceService {
    List<NotificationChannel> getChannelsFor(String userId, NotificationType type);
}
// Users can enable/disable channels per notification type
```

**Gap dimension:** Entity Modeling (Dimension 2)

---

## Mistake 5: No DeliveryLog

**What happens:** Notifications sent but no record of what was sent, when, to which channel, with what result.

**Why it's matters:** Without a delivery log, debugging "why didn't I get my OTP?" is impossible. Interviewers ask about auditability.

**Fix:** `DeliveryLog` entity tracking: notificationId, channelType, attempt count, status, timestamps.

**Gap dimension:** Requirements (Dimension 1)

---

## Problem-Specific Gotchas

- Priority levels: ALERT should bypass rate limits; MARKETING is queued during quiet hours
- In-app channel doesn't call an external provider — it just stores in DB/memory
- Rate limiting is best modeled as a Decorator wrapping any channel — not embedded in each channel
- `NotificationType` determines which channels are eligible — TRANSACTIONAL should never go to channels user opted out of marketing for

---

## Self-Assessment Checklist

- [ ] Is `NotificationChannel` an interface (not if-else)?
- [ ] Does each channel fail independently (no shared try-catch)?
- [ ] Is retry logic present (with max attempts and backoff)?
- [ ] Is `UserPreferenceService` a separate entity?
- [ ] Is delivery logged per notification per channel?
- [ ] Can a new channel be added by adding one class only?
- [ ] Is rate limiting a decorator (not embedded in EmailChannel)?

---

## Remediation Targets

| Dimension | If you scored 0–1 | Go to |
|-----------|------------------|-------|
| Design Patterns | No channel interface | GAP_REMEDIATION.md#gap-3 — trigger: "multiple types behave differently" |
| SOLID | OCP violation, shared try-catch | GAP_REMEDIATION.md#gap-4 — violation hunt |
| Requirements | No retry, no preferences, no delivery log | GAP_REMEDIATION.md#gap-1 — requirements drill |
| Extensibility | Adding WhatsApp touches NotificationService | GAP_REMEDIATION.md#gap-5 — extension challenge |
