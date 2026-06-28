# Solution Guide — Notification System

**Read after your attempt.**

---

## Entity Map

| Class/Interface | Type | Responsibility |
|----------------|------|---------------|
| `Notification` | Concrete | userId, message, type, priority, metadata |
| `NotificationType` | Enum | TRANSACTIONAL, MARKETING, ALERT |
| `NotificationChannel` | Interface | `send(Notification, User): DeliveryResult` |
| `EmailChannel` | Concrete | Sends via email provider |
| `SMSChannel` | Concrete | Sends via SMS provider |
| `PushChannel` | Concrete | Sends via FCM/APNs |
| `InAppChannel` | Concrete | Stores in user's in-app inbox |
| `DeliveryResult` | Enum + metadata | SUCCESS, FAILURE(reason), RETRY |
| `RetryPolicy` | Concrete | maxAttempts, backoffMs; decides whether to retry |
| `UserPreferenceService` | Concrete | Returns List<NotificationChannel> for a userId + notificationType |
| `NotificationService` | Concrete | Orchestrates: look up preferences → dispatch to each channel independently → log |
| `DeliveryLog` | Concrete | Tracks per-notification per-channel: attempts, status, timestamps |

---

## Design Pattern: Channel as Strategy

The core extensibility win:

```java
interface NotificationChannel {
    String getChannelType(); // "EMAIL", "SMS", "PUSH", "IN_APP"
    DeliveryResult send(Notification notification, User user);
}

class EmailChannel implements NotificationChannel {
    private final EmailProvider emailProvider;

    public String getChannelType() { return "EMAIL"; }

    public DeliveryResult send(Notification notification, User user) {
        try {
            emailProvider.send(user.getEmail(), notification.getMessage());
            return DeliveryResult.SUCCESS;
        } catch (EmailProviderException e) {
            return DeliveryResult.failure(e.getMessage());
        }
    }
}

class SMSChannel implements NotificationChannel {
    private final SMSProvider smsProvider;

    public DeliveryResult send(Notification notification, User user) {
        try {
            smsProvider.send(user.getPhone(), notification.getMessage());
            return DeliveryResult.SUCCESS;
        } catch (Exception e) {
            return DeliveryResult.retry(e.getMessage()); // transient failure → retry
        }
    }
}
```

Adding WhatsApp: `class WhatsAppChannel implements NotificationChannel { ... }`. Zero existing code touched.

---

## NotificationService — Independent Channel Delivery

```java
class NotificationService {
    private final UserPreferenceService preferenceService;
    private final RetryPolicy retryPolicy;
    private final DeliveryLog deliveryLog;

    public void send(Notification notification) {
        User user = userRepository.findById(notification.getUserId());
        List<NotificationChannel> channels =
            preferenceService.getChannelsFor(user.getId(), notification.getType());

        for (NotificationChannel channel : channels) {
            deliverWithRetry(notification, user, channel);
        }
    }

    private void deliverWithRetry(Notification notification, User user, NotificationChannel channel) {
        int attempts = 0;
        DeliveryResult result;

        do {
            result = channel.send(notification, user);
            deliveryLog.record(notification.getId(), channel.getChannelType(), result);
            attempts++;

            if (result == DeliveryResult.RETRY && attempts < retryPolicy.getMaxAttempts()) {
                sleep(retryPolicy.backoffMs(attempts)); // exponential backoff
            }
        } while (result == DeliveryResult.RETRY && attempts < retryPolicy.getMaxAttempts());

        if (result != DeliveryResult.SUCCESS) {
            deliveryLog.markFailed(notification.getId(), channel.getChannelType());
            // Could publish to dead-letter queue here for manual intervention
        }
    }
}
```

Key: each channel is dispatched independently. Email failing doesn't prevent SMS.

---

## User Preferences

```java
class UserPreferenceService {
    // Map: userId → (notificationType → enabled channels)
    private final Map<String, Map<NotificationType, List<String>>> preferences = new ConcurrentHashMap<>();

    public List<NotificationChannel> getChannelsFor(String userId, NotificationType type) {
        Map<NotificationType, List<String>> userPrefs = preferences.getOrDefault(userId, defaultPrefs());
        List<String> enabledChannelTypes = userPrefs.getOrDefault(type, List.of("EMAIL"));
        return channelRegistry.getChannels(enabledChannelTypes);
    }
}
```

---

## Retry Policy

```java
class RetryPolicy {
    private final int maxAttempts;
    private final long initialBackoffMs;

    public long backoffMs(int attempt) {
        return initialBackoffMs * (long) Math.pow(2, attempt - 1); // exponential backoff
        // attempt 1: 1000ms, attempt 2: 2000ms, attempt 3: 4000ms
    }

    public int getMaxAttempts() { return maxAttempts; }
}
```

---

## Rate Limiting Extension (Decorator Pattern)

```java
class RateLimitingChannel implements NotificationChannel {
    private final NotificationChannel delegate;
    private final RateLimiter rateLimiter;

    public RateLimitingChannel(NotificationChannel delegate, RateLimiter rateLimiter) {
        this.delegate = delegate;
        this.rateLimiter = rateLimiter;
    }

    public DeliveryResult send(Notification notification, User user) {
        String key = user.getId() + ":" + notification.getType().name();
        if (!rateLimiter.isAllowed(key)) {
            return DeliveryResult.failure("Rate limit exceeded");
        }
        return delegate.send(notification, user);
    }
}

// Usage:
NotificationChannel emailWithRateLimit = new RateLimitingChannel(
    new EmailChannel(emailProvider),
    new TokenBucketRateLimiter(3, Duration.ofDays(1)) // 3 marketing emails per day
);
```

---

## Extension Points

| "Now add..." | Change |
|-------------|--------|
| WhatsApp channel | New class implementing `NotificationChannel` |
| Rate limiting | Decorator wrapping any channel; zero channel changes |
| Batching/digest | `BatchingChannel` decorator; accumulate + send periodically |
| Priority queue | `PriorityBlockingQueue<Notification>` in dispatch layer |
| Async delivery | Queue notifications; worker threads consume and deliver |

---

## What Strong Candidates Do Differently

- `NotificationChannel` interface immediately — not if-else in service
- Independent per-channel delivery loop (failure in one doesn't stop others)
- Retry on `RETRY` result, not on all failures (some are permanent: invalid phone number)
- Decorator for rate limiting (zero changes to EmailChannel)
- UserPreferences as a separate service — not hardcoded in NotificationService

## What Average Candidates Miss

- One big `sendNotification()` with if-else on channel type
- Email failure stops SMS attempt
- No retry logic modeled
- User preferences hardcoded ("send everyone email")
- No DeliveryLog — no way to audit delivery status
