# Mock Interview Script — Notification System

**For the interviewer.**

---

## Opening

> "Design a notification system. Any service in our platform can send notifications to users via email, SMS, and push notifications. Users have preferences for which channels they want. Ask any clarifying questions."

| If candidate asks | Answer |
|------------------|--------|
| Channels? | Email, SMS, push, in-app — all four |
| User preferences per channel? | Yes — user can enable/disable each channel per notification type |
| Retry on failure? | Yes — retry failed deliveries up to 3 times with exponential backoff |
| Does one channel failure block others? | No — each channel is independent |
| Rate limiting? | Secondary — mention it if time allows |
| Priority? | TRANSACTIONAL (OTP, order confirm) is highest; MARKETING is lowest |
| Sync or async delivery? | For MVP, synchronous per channel is fine; async is bonus |

---

## Phase 2: Core Design — What to Watch For

**Key signal 1:** Observer / Strategy for channels
- Good: `NotificationChannel` interface with `EmailChannel`, `SMSChannel` implementing it
- Bad: Giant `NotificationService.send()` with if-else on channel type

**Key signal 2:** Independent channel delivery
- Good: Each channel attempt independent; failure in email doesn't skip SMS
- Bad: Single try-catch wrapping all channels → one failure stops rest

**Key signal 3:** User preferences lookup
- Good: `UserPreferenceService.getChannelsFor(userId, notificationType)`
- Bad: Hardcode which channels everyone gets

If no channel interface extracted:
> "If we want to add WhatsApp as a new channel, how many places in your code would you need to change?"

If retry not discussed:
> "The SMS provider is temporarily down. What happens to that notification?"

---

## Extension Probe (~35 min)

> "Add rate limiting — users should receive at most 3 marketing emails per day."

Strong: `RateLimiter` or `RateLimitingChannel` decorator wrapping `EmailChannel`. Checks count before delegating. Or `UserPreferenceService` tracks daily send count. Strategy for the limit threshold.

---

## Concurrency Probe (~40 min)

> "1 million users get a notification simultaneously — say, a major product announcement. What breaks?"

Strong: Each user's notification is independent (parallel processing is fine). Per-channel providers have their own rate limits (Twilio, SendGrid). Mentions async queue / message bus as the production approach. In-memory synchronous works for interview scope.

---

## Scoring Checklist

| Dimension | Evidence | Score (0–3) |
|-----------|----------|-------------|
| Requirements | Channel types, user preferences, retry, independent channels, notification types | |
| Entity Modeling | NotificationChannel interface, UserPreferences as entity, DeliveryLog, RetryPolicy | |
| Design Patterns | Observer (or Strategy) for channels, Decorator for rate limiting | |
| SOLID | OCP — new channel = new class; SRP — each channel class does one thing | |
| Extensibility | WhatsApp = new class, zero changes to NotificationService | |
| Concurrency | Parallel channel delivery, thread-safe preference lookup | |
| Communication | Explained WHY interface over if-else, WHY independent channel delivery | |
| **TOTAL** | | **/21** |
