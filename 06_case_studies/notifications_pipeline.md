# Case Study: Notifications Pipeline

**Audience:** Senior / Staff mobile engineers
**Goal:** Design a production-grade notifications system that balances deliverability, user respect, and mobile platform constraints — without spamming users into disabling notifications entirely.

---

## System Overview

A notifications pipeline is deceptively complex. What looks like "send a message to a phone" involves:

| Challenge | Complexity |
|-----------|------------|
| Multi-platform delivery | APNs, FCM, web push, SMS, email |
| Token lifecycle | Tokens expire, change, become invalid |
| User preferences | Per-channel, per-category, quiet hours |
| Delivery guarantees | At-least-once with deduplication |
| Rate limiting | Platform limits + user respect |
| Multi-device | Same user, multiple phones/tablets |
| Analytics | Delivered? Opened? Acted upon? |

A poorly designed notification system either fails silently or annoys users into permission revocation.

---

## Domain Resources

### Core Entities

```
User               → Account with notification preferences
Device             → Registered device with push token
Channel            → Delivery mechanism (push, email, SMS)
NotificationCategory → Type of notification (marketing, transactional, alerts)
Template           → Reusable notification content
Notification       → Specific message to deliver
DeliveryAttempt    → Record of send attempt and outcome
UserPreference     → Per-category, per-channel settings
```

### Resource Relationships

```
User
 ├─ Devices[] (multiple phones/tablets)
 │   └─ PushToken (per device, per environment)
 ├─ Preferences[]
 │   └─ CategoryPreference (per category × channel)
 └─ NotificationHistory[]

Notification
 ├─ Template
 ├─ TargetUser
 ├─ DeliveryAttempts[]
 └─ Analytics (delivered, opened, dismissed)
```

---

## Push Token Lifecycle

### The Token Problem

Push tokens are **ephemeral** and **platform-specific**:

| Platform | Token Behavior |
|----------|----------------|
| iOS (APNs) | Changes on app reinstall, OS update, restore from backup |
| Android (FCM) | Changes on app data clear, token refresh callback |
| Both | Become invalid if app uninstalled |

### Token Registration

```http
POST /devices
Headers:
  Authorization: Bearer {access_token}
  Idempotency-Key: dev-reg-...
Body:
{
  "platform": "IOS",
  "push_token": "abc123...",
  "token_type": "APNS_SANDBOX",
  "app_version": "2.5.1",
  "os_version": "17.2",
  "device_id": "device_uuid",
  "locale": "en-US",
  "timezone": "America/Los_Angeles"
}
```

Response:

```json
{
  "device_id": "dev_8a7b",
  "registered_at": "2025-03-14T18:30:00Z",
  "token_status": "ACTIVE",
  "capabilities": ["RICH_PUSH", "SILENT_PUSH", "ACTIONS"]
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `token_type` distinguishes sandbox/production | APNs rejects wrong environment |
| `device_id` as stable identifier | Token changes; device identity shouldn't |
| `capabilities` | Server knows what features device supports |
| `locale` + `timezone` | Personalization and quiet hours |

### Token Refresh Strategy

```swift
// iOS: Register on every app launch
func application(_ application: UIApplication,
                 didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    let token = deviceToken.hexString

    // Always send to server - let server dedupe
    api.registerDevice(token: token, platform: .ios)
}
```

**Gotcha:** Don't diff tokens client-side. Always send the current token; let the server handle deduplication and invalidation of old tokens.

### Token Invalidation

```http
DELETE /devices/{device_id}
```

Called when:
- User logs out
- User revokes notification permission
- Token explicitly rejected by APNs/FCM

Server-side detection:

| Signal | Action |
|--------|--------|
| APNs returns `BadDeviceToken` | Mark token invalid |
| APNs returns `Unregistered` | App was uninstalled |
| FCM returns `NotRegistered` | Token expired or uninstalled |
| No delivery for 30 days | Probe with silent push |

---

## User Preferences

### Preference Model

```http
GET /users/me/notification-preferences
```

Response:

```json
{
  "global_enabled": true,
  "quiet_hours": {
    "enabled": true,
    "start": "22:00",
    "end": "08:00",
    "timezone": "America/Los_Angeles"
  },
  "categories": [
    {
      "category_id": "TRIP_UPDATES",
      "name": "Trip updates",
      "description": "Driver arriving, trip completed",
      "channels": {
        "push": { "enabled": true, "required": true },
        "email": { "enabled": false, "required": false },
        "sms": { "enabled": true, "required": false }
      },
      "priority": "HIGH",
      "can_disable": false
    },
    {
      "category_id": "PROMOTIONS",
      "name": "Promotions & offers",
      "description": "Discounts and special offers",
      "channels": {
        "push": { "enabled": true, "required": false },
        "email": { "enabled": true, "required": false }
      },
      "priority": "LOW",
      "can_disable": true
    }
  ]
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `required` flag | Some notifications are non-optional (security alerts) |
| `can_disable` | Regulatory/UX: users must control marketing |
| Per-channel per-category | Granular control without overwhelming |
| `quiet_hours` with timezone | Respect user's local time |
| `priority` hint | Client can show importance indicator |

### Preference Update

```http
PATCH /users/me/notification-preferences
Headers:
  Idempotency-Key: pref-...
Body:
{
  "categories": [
    {
      "category_id": "PROMOTIONS",
      "channels": {
        "push": { "enabled": false }
      }
    }
  ]
}
```

**Gotcha:** Preference changes must propagate immediately. A user who disables promotions and then receives one will revoke all permissions.

---

## Notification Delivery

### Send API (Internal/Backend)

```http
POST /notifications/send
Body:
{
  "template_id": "trip_driver_arrived",
  "user_id": "usr_123",
  "channels": ["PUSH", "SMS"],
  "priority": "HIGH",
  "category_id": "TRIP_UPDATES",
  "data": {
    "driver_name": "Alex",
    "vehicle": "Silver Toyota Camry",
    "trip_id": "tr_456"
  },
  "options": {
    "ttl_seconds": 300,
    "collapse_key": "trip_tr_456_status",
    "respect_quiet_hours": false,
    "dedupe_key": "driver_arrived_tr_456"
  }
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `template_id` | Centralized copy management, localization |
| `channels` array | Multi-channel delivery for critical messages |
| `ttl_seconds` | Notification expires if not delivered in time |
| `collapse_key` | Later notification replaces earlier (status updates) |
| `respect_quiet_hours` | Critical alerts can override |
| `dedupe_key` | Prevent duplicate sends from retries |

### Template System

```json
{
  "template_id": "trip_driver_arrived",
  "category_id": "TRIP_UPDATES",
  "channels": {
    "push": {
      "title": "Your driver is here",
      "body": "{{driver_name}} has arrived in a {{vehicle}}",
      "sound": "arrival.wav",
      "badge_increment": 1,
      "actions": [
        { "id": "CONTACT_DRIVER", "title": "Contact" },
        { "id": "CANCEL_TRIP", "title": "Cancel", "destructive": true }
      ],
      "deep_link": "app://trips/{{trip_id}}"
    },
    "sms": {
      "body": "Your driver {{driver_name}} has arrived. Track at {{short_link}}"
    }
  },
  "locales": {
    "es": {
      "push": {
        "title": "Tu conductor ha llegado",
        "body": "{{driver_name}} ha llegado en un {{vehicle}}"
      }
    }
  }
}
```

---

## Platform-Specific Payloads

### APNs (iOS)

```json
{
  "aps": {
    "alert": {
      "title": "Your driver is here",
      "body": "Alex has arrived in a Silver Toyota Camry"
    },
    "sound": "arrival.wav",
    "badge": 1,
    "category": "TRIP_ARRIVED",
    "mutable-content": 1,
    "thread-id": "trip_tr_456"
  },
  "trip_id": "tr_456",
  "deep_link": "app://trips/tr_456",
  "image_url": "https://cdn.example.com/driver_photo.jpg"
}
```

| Field | Purpose |
|-------|---------|
| `mutable-content` | Enables Notification Service Extension (download images, modify) |
| `thread-id` | Groups notifications in Notification Center |
| `category` | Maps to registered UNNotificationCategory for actions |
| Custom keys (`trip_id`, etc.) | App-specific data for handling |

### FCM (Android)

```json
{
  "message": {
    "token": "fcm_token...",
    "notification": {
      "title": "Your driver is here",
      "body": "Alex has arrived in a Silver Toyota Camry",
      "image": "https://cdn.example.com/driver_photo.jpg",
      "channel_id": "trip_updates"
    },
    "data": {
      "trip_id": "tr_456",
      "deep_link": "app://trips/tr_456",
      "notification_id": "notif_789"
    },
    "android": {
      "priority": "high",
      "ttl": "300s",
      "collapse_key": "trip_tr_456",
      "notification": {
        "click_action": "TRIP_DETAIL_ACTIVITY"
      }
    }
  }
}
```

| Field | Purpose |
|-------|---------|
| `channel_id` | Android notification channel (user can disable per-channel) |
| `priority: high` | Wakes device, shows heads-up notification |
| `collapse_key` | Replaces existing notification with same key |
| `data` payload | Always delivered, even if notification suppressed |

---

## Silent Push Notifications

### Use Cases

| Use Case | Payload |
|----------|---------|
| Background data sync | Trigger fetch without user-visible notification |
| Token validation | Probe if app still installed |
| State invalidation | Tell app to refresh cached data |
| Logout propagation | Force re-auth on all devices |

### iOS Silent Push

```json
{
  "aps": {
    "content-available": 1
  },
  "sync_type": "TRIP_UPDATE",
  "trip_id": "tr_456"
}
```

**Gotchas:**

| Issue | Mitigation |
|-------|------------|
| iOS throttles silent push | Budget ~3/hour in practice |
| App may not wake | Background App Refresh must be enabled |
| 30-second execution limit | Fetch critical data only |
| No guarantee of delivery | Don't rely on for critical state |

### Android Data-Only Message

```json
{
  "message": {
    "token": "...",
    "data": {
      "sync_type": "TRIP_UPDATE",
      "trip_id": "tr_456"
    }
  }
}
```

---

## Multi-Device Handling

### The Fan-Out Problem

A user with 3 devices receives notification on all 3 — but should only see it on 1 active device.

### Strategies

| Strategy | Behavior | Tradeoff |
|----------|----------|----------|
| Send to all | All devices get push | Annoying if multiple active |
| Primary device only | Only "main" device | Misses if that device is off |
| Activity-based | Most recently active | May be stale |
| Smart routing | Consider app state, recency | Complex |

### Recommended: Activity-Based with Collapse

```json
{
  "collapse_key": "trip_tr_456_arrived",
  "devices": ["dev_1", "dev_2", "dev_3"],
  "routing": {
    "strategy": "MOST_RECENT_ACTIVE",
    "fallback": "ALL_DEVICES",
    "activity_threshold_minutes": 5
  }
}
```

**Logic:**
1. If device was active in last 5 minutes → send there only
2. Otherwise → send to all devices
3. `collapse_key` ensures user sees only latest if multiple arrive

---

## Delivery Pipeline Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Trigger   │────►│   Router    │────►│   Queue     │
│  (Event)    │     │ (Preferences│     │  (Priority) │
└─────────────┘     │  + Channels)│     └──────┬──────┘
                    └─────────────┘            │
                                               ▼
                    ┌─────────────────────────────────────┐
                    │           Delivery Workers          │
                    │  ┌───────┐ ┌───────┐ ┌───────┐     │
                    │  │ APNs  │ │  FCM  │ │  SMS  │     │
                    │  │Worker │ │Worker │ │Worker │     │
                    │  └───┬───┘ └───┬───┘ └───┬───┘     │
                    └──────┼────────┼─────────┼──────────┘
                           │        │         │
                           ▼        ▼         ▼
                    ┌─────────────────────────────────────┐
                    │         Platform Gateways           │
                    │    (APNs, FCM, Twilio, etc.)        │
                    └─────────────────────────────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────────────┐
                    │      Delivery Tracking Store        │
                    │   (attempts, outcomes, analytics)   │
                    └─────────────────────────────────────┘
```

### Priority Queues

| Priority | Use Case | Latency Target |
|----------|----------|----------------|
| CRITICAL | Security alerts, 2FA codes | < 5 seconds |
| HIGH | Transactional (trip updates) | < 30 seconds |
| MEDIUM | Social (messages, reactions) | < 2 minutes |
| LOW | Marketing, digests | Best effort |

---

## Rate Limiting

### Platform Limits

| Platform | Limit | Consequence |
|----------|-------|-------------|
| APNs | No hard limit, but throttles | Delayed delivery |
| FCM | 1000/second per project | Queue backpressure |
| SMS (Twilio) | Account-specific | 429 errors |

### User-Respect Limits

| Category | Limit | Rationale |
|----------|-------|-----------|
| Marketing | 1/day | Prevent spam fatigue |
| Social | 10/hour | Batch if exceeds |
| Transactional | No limit | But collapse duplicates |
| System alerts | No limit | Security critical |

### Batching and Coalescing

```json
// Instead of 10 separate "new message" notifications:
{
  "title": "5 new messages",
  "body": "From Alex, Jamie, and 3 others",
  "collapse_key": "new_messages",
  "badge": 5
}
```

---

## Analytics and Tracking

### Delivery States

```
QUEUED → SENT → DELIVERED → OPENED → ACTED
           │         │          │
           ▼         ▼          ▼
        FAILED   DISMISSED   EXPIRED
```

### Tracking Events

| Event | Source | Data |
|-------|--------|------|
| `sent` | Server | Timestamp, platform response |
| `delivered` | Platform callback (FCM) | Delivery timestamp |
| `opened` | Client app | Time to open, app state |
| `acted` | Client app | Which action tapped |
| `dismissed` | Client app (iOS) | User swiped away |

### Client Reporting

```http
POST /notifications/{notification_id}/events
Body:
{
  "event_type": "OPENED",
  "device_id": "dev_123",
  "timestamp": "2025-03-14T18:32:00Z",
  "context": {
    "app_state": "BACKGROUND",
    "time_to_open_seconds": 45,
    "action_id": null
  }
}
```

### Key Metrics

| Metric | Formula | Target |
|--------|---------|--------|
| Delivery rate | delivered / sent | > 95% |
| Open rate | opened / delivered | 5-15% (varies by category) |
| Action rate | acted / opened | 10-30% |
| Opt-out rate | disabled / total users | < 2% monthly |
| Permission grant rate | granted / prompted | > 60% |

---

## Deep Linking

### URL Structure

```
app://trips/tr_456?source=push&notification_id=notif_789
```

### Handling Matrix

| App State | Behavior |
|-----------|----------|
| Foreground | In-app alert or silent update |
| Background | Navigate on tap |
| Terminated | Launch → navigate |
| Not installed | App Store / web fallback |

### Universal Links (iOS) / App Links (Android)

```json
{
  "deep_link": "https://example.com/trips/tr_456",
  "fallback_url": "https://example.com/app-download"
}
```

**Gotcha:** Always include web fallback. Users share screenshots; deep links should work even without the app.

---

## Permission Handling

### iOS Permission Flow

```swift
func requestNotificationPermission() async -> Bool {
    let center = UNUserNotificationCenter.current()
    let settings = await center.notificationSettings()

    switch settings.authorizationStatus {
    case .notDetermined:
        // First time - show value prop first, then request
        return await center.requestAuthorization(options: [.alert, .sound, .badge])
    case .denied:
        // User previously denied - guide to Settings
        return false
    case .authorized, .provisional, .ephemeral:
        return true
    }
}
```

### Pre-Permission Pattern

Never request permission on first launch. Instead:

1. Show value proposition ("Get notified when your driver arrives")
2. User taps "Enable notifications"
3. Then trigger system prompt
4. If denied, show "Enable in Settings" path

### Provisional Authorization (iOS 12+)

```swift
let options: UNAuthorizationOptions = [.alert, .sound, .badge, .provisional]
```

Delivers notifications quietly to Notification Center without explicit permission. User can then choose to promote or disable.

---

## Error Handling

### Platform Errors

| Error | Meaning | Action |
|-------|---------|--------|
| APNs `BadDeviceToken` | Token invalid | Remove from database |
| APNs `Unregistered` | App uninstalled | Mark device inactive |
| APNs `TooManyRequests` | Rate limited | Exponential backoff |
| FCM `NotRegistered` | Token invalid | Remove from database |
| FCM `MismatchSenderId` | Wrong project | Configuration error |
| FCM `MessageTooBig` | Payload > 4KB | Truncate or split |

### Retry Strategy

| Error Type | Retry? | Strategy |
|------------|--------|----------|
| Token invalid | No | Remove token |
| Rate limited | Yes | Backoff with jitter |
| Temporary server error | Yes | 3 attempts, exponential backoff |
| Payload error | No | Log and alert |

---

## Testing Strategy

### Test Scenarios

| Scenario | Validation |
|----------|------------|
| Token refresh | Old token invalidated, new token works |
| Quiet hours | Notification queued, delivered after |
| Collapse key | Only latest notification shown |
| Multi-device | Correct routing based on activity |
| Permission denied | Graceful degradation to email/SMS |
| Silent push throttled | App handles missed syncs |

### Staging Environment

```json
{
  "environment": "SANDBOX",
  "apns_endpoint": "api.sandbox.push.apple.com",
  "fcm_project": "my-app-staging",
  "test_devices": ["dev_test_1", "dev_test_2"]
}
```

### Chaos Testing

| Chaos | Expected Behavior |
|-------|-------------------|
| APNs unavailable | Queue backpressure, retry on recovery |
| FCM rate limited | Backoff, no message loss |
| Preference service down | Use cached preferences or default to send |
| Database slow | Async write to analytics, don't block delivery |

---

## Common Gotchas Summary

| Gotcha | Mitigation |
|--------|------------|
| Token becomes invalid silently | Always send current token; server dedupes |
| User disables then re-enables | Handle `didRegisterForRemoteNotifications` every launch |
| Notification appears while app open | Handle foreground delivery gracefully |
| Badge count drift | Server-authoritative badge count |
| Timezone handling for quiet hours | Store user timezone; evaluate server-side |
| iOS silent push throttled | Don't rely on it for critical sync |
| Android notification channels | Must create before sending; can't change importance |
| Deep link fails for logged-out user | Queue deep link, execute after auth |
| Duplicate notifications from retries | `dedupe_key` with server-side deduplication |
| Marketing during onboarding | Delay promotional notifications until Day 3+ |

---

## Key Concepts

> "A notification is an interruption you're asking permission to make. Every push is a withdrawal from a trust account — overdraw and users revoke permission entirely. The pipeline must respect preferences instantly, route intelligently across devices, collapse redundant updates, and degrade gracefully when platforms throttle. Tokens are ephemeral; always re-register. Deep links must work whether the app is running, backgrounded, or not installed. Measure not just delivery but opened rates and opt-out velocity — a 'delivered' notification that trains users to ignore you is worse than no notification at all."
