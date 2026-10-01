---
title: Troubleshoot webhook delivery
description: Diagnose webhook authentication, delivery, retry, and processing failures.
audience:
  - developers
  - support engineers
  - implementation engineers
product_area: Platform Integrations
content_type: troubleshooting
---

# Troubleshoot webhook delivery

Use this guide when employee lifecycle events are missing, delayed, rejected, or processed more than once.

Start by identifying whether the failure is in **delivery** or **processing**.

```text
Orbit generated event
        |
        v
Did endpoint receive it?
   |             |
  No            Yes
   |             |
Delivery issue   v
            Did signature pass?
               |       |
              No      Yes
               |       |
          Auth issue   v
                 Did processing succeed?
                     |       |
                    No      Yes
                     |       |
              App/downstream Done
```

## Events are not arriving

### Symptoms

- no request appears in endpoint logs;
- delivery remains `retrying` or becomes `failed`;
- test events do not arrive.

### Check

1. Confirm the subscription is `active`.
2. Verify that the correct event types are selected.
3. Confirm the endpoint URL is correct.
4. Verify the endpoint is publicly reachable over HTTPS.
5. Check firewall, WAF, proxy, and load-balancer logs.
6. Inspect the latest Orbit delivery record.

### Common causes

| Cause | Evidence | Resolution |
| --- | --- | --- |
| DNS error | Connection failure before HTTP response | Correct DNS and wait for propagation |
| TLS problem | Handshake failure | Replace invalid or expired certificate |
| Firewall block | No application request but edge rejection appears | Permit webhook traffic according to your network policy |
| Wrong path | `404` response | Correct the subscription URL |
| Endpoint disabled | `503` or connection refusal | Restore the service |

## Signature verification fails

### Symptoms

Your endpoint returns `401` even though the request came from Orbit.

### Check

Verify that you are:

- using the signing secret for the same subscription;
- hashing the **raw** body;
- using the timestamp from `X-Orbit-Signature`;
- concatenating `timestamp + "." + raw_body`;
- using HMAC-SHA256; and
- comparing hexadecimal digests consistently.

### Common mistake: parsed JSON

Incorrect:

```text
receive body
parse JSON
serialize JSON
calculate signature
```

Correct:

```text
receive raw body
calculate signature
then parse JSON
```

JSON reserialization can change whitespace or key representation.

## Valid events are rejected as stale

### Symptoms

Signature computation matches but timestamp validation fails.

### Check

Compare:

- webhook timestamp;
- application server time; and
- configured tolerance.

Synchronize host clocks using your environment's standard time service.

Do not solve clock problems by removing timestamp validation.

## The same employee action happens twice

### Cause

Webhook systems can deliver an event more than once. A retry can occur even after your application performed the downstream action if Orbit did not receive the 2xx acknowledgment.

### Resolution

Make processing idempotent.

Store the event ID before performing non-reversible side effects.

For downstream create operations, also use a deterministic idempotency key when supported.

## Delivery is successful but no downstream action occurs

A `delivered` webhook means Orbit received a 2xx response. It does not prove your application completed the business workflow.

Trace the event using:

- event ID;
- delivery ID;
- queue message ID;
- employee ID; and
- downstream request ID.

Check whether the event was:

1. persisted;
2. queued;
3. dequeued;
4. matched by a business rule;
5. authorized to call the downstream system; and
6. acknowledged by the downstream system.

## `employee.updated` creates too many actions

Do not treat every employee update as relevant.

Inspect `changed_fields` when available:

```json
{
  "changed_fields": [
    "job.manager_id",
    "job.department"
  ]
}
```

If your workflow only responds to location changes, ignore this event.

Also consider comparing the current employee record with the last synchronized state.

## Employee API returns `401`

Confirm:

- the `Authorization` header uses `Bearer`;
- the token has not expired or been revoked;
- the application is using the correct environment credential; and
- no whitespace or quotation marks were included accidentally.

Never print the complete bearer token in logs.

## Employee API returns `403`

Authentication succeeded, but the credential lacks permission.

Compare the credential's scopes with the operation.

For a read-only lifecycle integration, request only the employee read permissions required by the fields you consume.

## Delivery remains `retrying`

Inspect the latest response code.

### `429`

Your endpoint or gateway is rate limiting deliveries.

Increase safe capacity or adjust your ingress policy.

### `500`, `502`, `503`, `504`

Your application or an upstream dependency is unavailable.

Webhook acknowledgment should depend only on whether the event can be safely accepted, not whether every downstream action has completed.

Queue first, acknowledge, then process.

## A delivery is permanently `failed`

After resolving the cause:

1. open the subscription delivery history;
2. locate the failed event;
3. verify that replay will not create duplicate side effects;
4. replay the delivery; and
5. confirm the new delivery reaches `delivered`.

If your application previously completed the event but returned an error afterward, idempotency protection should prevent duplicate work.

## Escalation information

Collect the following before escalating a platform issue:

```text
subscription ID
event ID
delivery ID
event type
delivery timestamp
HTTP response code
Orbit request ID, if available
your correlation ID
```

Do not include:

- bearer tokens;
- webhook signing secrets;
- full employee records unless specifically required and approved; or
- unrelated personal data.
