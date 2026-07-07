NAP-NOTIFY
==========

Shell-Rendered Notifications
----------------------------

`draft`

**NAP ID:** NAP-NOTIFY
**Domain:** `notify`
**Web binding (NIP-5D):** `window.napplet.notify` · `shell.supports("notify")`

## Description

NAP-NOTIFY provides notification delivery between napplets and the shell. Napplets that need to alert users -- incoming messages, completed tasks, background events -- send notification requests to the shell. The shell renders notifications using its own UI (toasts, system notifications, badge counts) and routes user interaction back to the napplet.

Napplets do not render notifications themselves. The shell controls presentation, grouping, priority, and dismissal. This ensures consistent UX across napplets and allows the shell to enforce notification policies (rate limiting, do-not-disturb, channel muting).

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `send` | `notification` (`NotificationPayload`) | `NotificationResult` | `notify.send` / `notify.send.result` |
| `dismiss` | `notificationId` (`tstr`) | none | `notify.dismiss` |
| `badge` | `count` (`uint`) | none | `notify.badge` |
| `registerChannel` | `channel` (`NotificationChannel`) | none | `notify.registerChannel` |
| `requestPermission` | optional `channel` (`tstr`) | `PermissionResult` | `notify.requestPermission` / `notify.requestPermission.result` |
| `onAction` | handler for `notificationId` and `actionId` | `Subscription` handle | `notify.action` |
| `onClicked` | handler for `notificationId` | `Subscription` handle | `notify.clicked` |
| `onDismissed` | handler for `notificationId` and optional reason | `Subscription` handle | `notify.dismissed` |
| `onControls` | handler for list of `NotifyControl` | `Subscription` handle | `notify.controls` |

### Schemas

Enumerations:

| Name | Values |
|------|--------|
| `NotificationPriority` | `low`, `normal`, `high`, `urgent` |
| `NotifyControl` | `toasts`, `badges`, `actions`, `channels`, `system` |

`NotificationPayload`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | `tstr` | yes | Notification title. |
| `body` | `tstr` | no | Notification body text. |
| `icon` | `tstr` | no | Icon URL or shell-understood icon reference. |
| `actions` | list of `NotificationAction` | no | Action buttons. |
| `channel` | `tstr` | no | Notification channel ID. |
| `priority` | `NotificationPriority` | no | Notification urgency hint. |

`NotificationResult`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `notificationId` | `tstr` | yes | Shell-assigned notification ID. |
| `error` | `tstr` | no | Error reason when notification delivery failed. |

`NotificationAction`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | `tstr` | yes | Action identifier returned by `notify.action`. |
| `label` | `tstr` | yes | User-visible action label. |

`NotificationChannel`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channelId` | `tstr` | yes | Channel identifier. |
| `label` | `tstr` | yes | User-visible channel label. |
| `description` | `tstr` | no | User-visible channel description. |
| `defaultPriority` | `NotificationPriority` | no | Default priority for this channel. |

`PermissionResult`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `granted` | `bool` | yes | Whether notification permission was granted. |

**`send(notification)`** -- Send a notification to the shell. The shell decides how to render it (toast, system notification, etc.) based on priority, channel, and shell policy. Returns the assigned `notificationId`. The shell MAY reject the notification (e.g., permission denied, rate limited).

**`dismiss(notificationId)`** -- Request dismissal of a notification by ID. Fire-and-forget. The shell may have already dismissed it.

**`badge(count)`** -- Set the badge count for this napplet. The shell displays the badge on the napplet's tile, tab, or icon. Pass `0` to clear. Fire-and-forget.

**`registerChannel(channel)`** -- Register a notification channel. Channels let users control notification categories independently (e.g., mute "promotions" but keep "messages"). Fire-and-forget. The shell MAY ignore channels it does not support.

**`requestPermission(channel?)`** -- Request permission to send notifications. If `channel` is provided, requests permission for that specific channel. Returns whether permission was granted. The shell MAY prompt the user or apply policy silently.

**`onAction(callback)`** -- Listen for action button clicks. When a user clicks an action button on a notification, the shell sends the `notificationId` and `actionId`. Returns a Subscription with `close()`.

**`onClicked(callback)`** -- Listen for notification body clicks. Returns a Subscription with `close()`.

**`onDismissed(callback)`** -- Listen for notification dismissals. The `reason` field indicates why: `'user'` (user dismissed), `'timeout'` (auto-dismissed), `'replaced'` (replaced by another notification). Returns a Subscription with `close()`.

**`onControls(callback)`** -- Listen for the shell's notification capability list. The shell tells the napplet which notification features it supports (toasts, badges, actions, channels, system notifications). Returns a Subscription with `close()`.

## Wire Protocol

`notify.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `notify.send` | napplet -> shell | `id`, `title`, `body?`, `icon?`, `actions?`, `channel?`, `priority?` |
| `notify.send.result` | shell -> napplet | `id`, `notificationId?`, `error?` |
| `notify.dismiss` | napplet -> shell | `notificationId` |
| `notify.badge` | napplet -> shell | `count` |
| `notify.channel.register` | napplet -> shell | `channelId`, `label`, `description?`, `defaultPriority?` |
| `notify.permission.request` | napplet -> shell | `id`, `channel?` |
| `notify.permission.result` | shell -> napplet | `id`, `granted` |
| `notify.action` | shell -> napplet | `notificationId`, `actionId` |
| `notify.clicked` | shell -> napplet | `notificationId` |
| `notify.dismissed` | shell -> napplet | `notificationId`, `reason?` |
| `notify.controls` | shell -> napplet | `controls` |

Key design notes:
- `notify.send` / `notify.send.result` use `id` for correlation. The shell assigns `notificationId` on success.
- `notify.dismiss`, `notify.badge`, and `notify.channel.register` are fire-and-forget (no `id` correlation).
- `notify.permission.request` / `notify.permission.result` use `id` for correlation. The shell MAY prompt the user asynchronously.
- `notify.action`, `notify.clicked`, and `notify.dismissed` are shell-initiated. The shell sends these when the user interacts with a notification.
- `notify.controls` is shell-initiated. The shell pushes its supported notification capabilities.

### Priority

Four priority levels control notification urgency:

| Priority | Behavior hint |
|----------|---------------|
| `low` | Silent, no interruption. May appear in notification center only. |
| `normal` | Default. Standard toast or notification. |
| `high` | Prominent display. May use sound or vibration. |
| `urgent` | Immediate attention required. May bypass do-not-disturb. |

The shell ultimately decides how each priority level maps to its notification UI.

### Actions

Notifications may include up to 3 action buttons:

```json
{
  "actions": [
    { "id": "reply", "label": "Reply" },
    { "id": "dismiss", "label": "Dismiss" }
  ]
}
```

When the user clicks an action, the shell sends `notify.action` with the `notificationId` and `actionId`. The shell MAY limit the number of actions displayed.

### Examples

**Send a notification:**
```
-> { "type": "notify.send", "id": "n1", "title": "New message", "body": "Alice: hey!", "priority": "normal" }
<- { "type": "notify.send.result", "id": "n1", "notificationId": "shell-42" }
```

**Send with actions:**
```
-> { "type": "notify.send", "id": "n2", "title": "Incoming call", "priority": "urgent", "actions": [{ "id": "answer", "label": "Answer" }, { "id": "decline", "label": "Decline" }] }
<- { "type": "notify.send.result", "id": "n2", "notificationId": "shell-43" }
```

**User clicks an action:**
```
<- { "type": "notify.action", "notificationId": "shell-43", "actionId": "answer" }
```

**User clicks notification body:**
```
<- { "type": "notify.clicked", "notificationId": "shell-42" }
```

**Notification auto-dismissed:**
```
<- { "type": "notify.dismissed", "notificationId": "shell-42", "reason": "timeout" }
```

**Set badge count:**
```
-> { "type": "notify.badge", "count": 3 }
```

**Clear badge:**
```
-> { "type": "notify.badge", "count": 0 }
```

**Register a channel:**
```
-> { "type": "notify.channel.register", "channelId": "messages", "label": "Messages", "description": "Direct messages and mentions", "defaultPriority": "normal" }
```

**Request permission:**
```
-> { "type": "notify.permission.request", "id": "p1", "channel": "messages" }
<- { "type": "notify.permission.result", "id": "p1", "granted": true }
```

**Dismiss a notification:**
```
-> { "type": "notify.dismiss", "notificationId": "shell-42" }
```

**Shell sends control list:**
```
<- { "type": "notify.controls", "controls": ["toasts", "badges", "actions", "channels"] }
```

**Notification rejected:**
```
<- { "type": "notify.send.result", "id": "n3", "error": "permission denied" }
```

### Error Handling

`notify.send.result` includes an `error` field (string) on failure. Common errors: `"permission denied"`, `"rate limited"`, `"invalid channel"`. When `error` is present, `notificationId` is absent and the notification was not created.

`notify.permission.result` returns `granted: false` if the shell denies permission. This is not an error -- it is a normal policy response.

Messages referencing an unknown `notificationId` (e.g., `notify.dismiss` for an already-dismissed notification) MUST be silently ignored by both napplet and shell.

## Shell Behavior

- The shell MUST accept `notify.send` messages and respond with `notify.send.result` carrying the same `id`.
- The shell MUST accept `notify.permission.request` messages and respond with `notify.permission.result` carrying the same `id`.
- The shell MUST track active notifications per napplet and remove them when the napplet iframe is removed.
- The shell SHOULD display notifications using its own UI (toasts, system notifications, notification center).
- The shell SHOULD respect the `priority` field when deciding notification urgency and display style.
- The shell MAY send `notify.action`, `notify.clicked`, and `notify.dismissed` messages when users interact with notifications.
- The shell MAY send `notify.controls` to inform the napplet which notification features the shell supports.
- The shell MAY enforce notification rate limits per napplet.
- The shell MAY enforce ACL checks on `notify` capabilities (e.g., restricting which napplets can send notifications).
- The shell MAY support notification channels and allow users to configure per-channel policies.
- The shell MAY delegate to the browser's Notification API for system-level notifications when the shell has permission.
- The shell MUST silently ignore messages referencing unknown notification IDs.
- The shell MUST clean up all notifications for a napplet when the napplet's iframe is removed.

## Security Considerations

- Notifications expose user-facing text content. The shell SHOULD sanitize notification `title` and `body` to prevent injection attacks (HTML, scripts).
- The `icon` field is a URL or identifier fetched by the shell. The shell SHOULD validate URLs and enforce a content security policy for fetched resources.
- Notification actions allow napplets to present choices to users. The shell SHOULD clearly attribute notifications to the originating napplet to prevent spoofing (e.g., a malicious napplet pretending to be a system dialog).
- The `urgent` priority level could be abused for attention-grabbing spam. The shell SHOULD rate-limit urgent notifications and MAY require explicit user consent for urgent notifications.
- Permission requests are asynchronous and may involve user prompts. The shell MUST NOT block other napplet messages while a permission prompt is pending.
- Badge counts are advisory -- the shell displays them but the napplet controls the count. A malicious napplet could set misleading badge counts. The shell MAY cap or ignore badge counts.

## Implementations

- (none yet)

## Changelog

- `6312406` - Introduced NAP-NOTIFY for shell-rendered notifications.
