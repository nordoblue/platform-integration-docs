---
title: Webhook API reference
description: Reference for creating subscriptions, receiving employee lifecycle events, validating signatures, and replaying failed deliveries.
audience:
  - developers
  - implementation engineers
product_area: Platform Integrations
content_type: api-reference
---

# Webhook API reference

Use the Webhooks API to create and manage event subscriptions for Orbit Workforce Platform.

## Base URL

```text
https://api.orbit.example
```

## Authentication

Management endpoints use bearer authentication:

```http
Authorization: Bearer <access-token>
```

Webhook deliveries do not use bearer authentication. Verify deliveries with the `X-Orbit-Signature` header and the subscription signing secret.

## Create a webhook subscription

```http
POST /v1/webhooks
```

Creates a webhook subscription for one or more event types.

### Request body

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `url` | string | Yes | Public HTTPS endpoint that receives event deliveries |
| `events` | array of strings | Yes | Event types included in the subscription |
| `environment` | string | Yes | `sandbox` or `production` |
| `description` | string | No | Human-readable purpose of the subscription |

Example:

```json
{
  "url": "https://integrations.example.com/orbit/events",
  "events": [
    "employee.created",
    "employee.updated",
    "employee.terminated"
  ],
  "environment": "sandbox",
  "description": "Employee lifecycle automation"
}
```

### Response

`201 Created`

```json
{
  "id": "whsub_01J8P0B2",
  "url": "https://integrations.example.com/orbit/events",
  "status": "active",
  "events": [
    "employee.created",
    "employee.updated",
    "employee.terminated"
  ],
  "environment": "sandbox",
  "signing_secret": "whsec_example_only",
  "created_at": "2026-09-18T16:04:22Z"
}
```

The signing secret is returned only when the subscription is created or explicitly rotated.

## List webhook subscriptions

```http
GET /v1/webhooks
```

Optional query parameters:

| Parameter | Type | Description |
| --- | --- | --- |
| `environment` | string | Filter by `sandbox` or `production` |
| `status` | string | Filter by subscription state |
| `limit` | integer | Number of results to return, 1-100 |
| `cursor` | string | Cursor returned by the previous page |

## Retrieve a subscription

```http
GET /v1/webhooks/{subscription_id}
```

Example:

```bash
curl \
  --url https://api.orbit.example/v1/webhooks/whsub_01J8P0B2 \
  --header "Authorization: Bearer $ORBIT_ACCESS_TOKEN"
```

## Update a subscription

```http
PATCH /v1/webhooks/{subscription_id}
```

You can update:

- endpoint URL;
- subscribed events;
- description; and
- active/disabled state.

Example:

```json
{
  "events": [
    "employee.created",
    "employee.updated",
    "employee.termination_scheduled",
    "employee.terminated"
  ]
}
```

## Rotate a signing secret

```http
POST /v1/webhooks/{subscription_id}/rotate-secret
```

A rotation response contains the new secret:

```json
{
  "signing_secret": "whsec_new_example_only",
  "effective_at": "2026-09-19T10:30:00Z"
}
```

Plan secret rotation so your endpoint can briefly accept both the previous and new secrets during deployment.

## Delete a subscription

```http
DELETE /v1/webhooks/{subscription_id}
```

Successful deletion returns:

```text
204 No Content
```

Deletion stops future deliveries. It does not delete historical delivery records.

# Event delivery

Orbit sends events as HTTPS `POST` requests.

Example headers:

```http
Content-Type: application/json
X-Orbit-Signature: t=1790253214,v1=18f20d2b2869f1d8...
X-Orbit-Event-Id: evt_01J8NQ9R4Y7
X-Orbit-Delivery-Id: del_01J8NQA2QPC
User-Agent: Orbit-Webhooks/1.0
```

Example body:

```json
{
  "id": "evt_01J8NQ9R4Y7",
  "type": "employee.updated",
  "occurred_at": "2026-09-18T14:22:31Z",
  "data": {
    "employee_id": "emp_8d31fa",
    "changed_fields": [
      "job.manager_id",
      "job.department"
    ]
  }
}
```

## Response behavior

Return a 2xx status after the request is authenticated and safely accepted.

Recommended:

```text
204 No Content
```

Do not return 2xx if the event cannot be safely persisted or queued.

# Signature verification

The signature format is:

```text
t=<unix_timestamp>,v1=<hex_digest>
```

Construct the signed payload from:

```text
<timestamp>.<raw_request_body>
```

Calculate:

```text
HMAC-SHA256(signing_secret, signed_payload)
```

Compare the result with the `v1` digest using constant-time comparison.

Reject requests when:

- the signature is missing;
- the signature does not match;
- the timestamp is too old; or
- the body cannot be read exactly as received.

# Delivery retries

Orbit treats any non-2xx response or network failure as unsuccessful.

Example retry sequence:

```text
Initial attempt
+ 1 minute
+ 5 minutes
+ 30 minutes
+ 2 hours
+ 6 hours
+ 12 hours
```

Retry timing may vary and should not be used as a scheduling mechanism.

Your endpoint must be idempotent because the same event can be delivered more than once.

# Delivery records

## List deliveries

```http
GET /v1/webhooks/{subscription_id}/deliveries
```

Example response:

```json
{
  "data": [
    {
      "id": "del_01J8NQA2QPC",
      "event_id": "evt_01J8NQ9R4Y7",
      "event_type": "employee.updated",
      "status": "delivered",
      "attempt_count": 1,
      "last_response_code": 204,
      "created_at": "2026-09-18T14:22:31Z",
      "delivered_at": "2026-09-18T14:22:32Z"
    }
  ],
  "next_cursor": null
}
```

## Replay a delivery

```http
POST /v1/webhooks/{subscription_id}/deliveries/{delivery_id}/replay
```

Replay creates a new delivery attempt for the same event.

Example response:

```json
{
  "delivery_id": "del_01J8R43AW61",
  "event_id": "evt_01J8NQ9R4Y7",
  "status": "pending"
}
```

Replaying an event does not change its event ID. Your integration should therefore decide whether a previously processed event should be ignored or intentionally reprocessed.

# Employee events

## `employee.created`

Sent when an employee record is created.

```json
{
  "id": "evt_01J8S12X",
  "type": "employee.created",
  "occurred_at": "2026-09-21T11:05:00Z",
  "data": {
    "employee_id": "emp_f3002e"
  }
}
```

## `employee.updated`

Sent when supported employee fields change.

```json
{
  "id": "evt_01J8S19D",
  "type": "employee.updated",
  "occurred_at": "2026-09-21T11:16:42Z",
  "data": {
    "employee_id": "emp_f3002e",
    "changed_fields": [
      "job.department",
      "job.manager_id"
    ]
  }
}
```

## `employee.termination_scheduled`

Sent when a future termination is recorded.

Use this event for preparatory actions that must happen before the termination date.

## `employee.terminated`

Sent when the termination becomes effective.

Use this event for actions that require the employee to be terminated before execution.

# Error responses

Management endpoints use standard HTTP status codes.

| Status | Meaning | Typical cause |
| --- | --- | --- |
| `400` | Bad Request | Invalid URL, unsupported event, malformed JSON |
| `401` | Unauthorized | Missing or invalid API credential |
| `403` | Forbidden | Credential does not have required scope |
| `404` | Not Found | Subscription or delivery ID does not exist |
| `409` | Conflict | Conflicting subscription state |
| `422` | Unprocessable Entity | Valid JSON with invalid field values |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Unexpected platform error |

Error example:

```json
{
  "error": {
    "code": "invalid_event_type",
    "message": "The event type employee.deleted is not supported.",
    "request_id": "req_01J8T19M"
  }
}
```

Log `request_id` when escalating API errors. Do not log bearer tokens or signing secrets.
