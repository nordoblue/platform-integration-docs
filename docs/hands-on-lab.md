---
title: Hands-on lab: Build an employee onboarding listener
description: Create and validate a small webhook listener that receives an employee.created event and retrieves the employee record.
audience:
  - developers
  - technical learners
product_area: Platform Integrations
content_type: hands-on-lab
estimated_time: 30 minutes
---

# Hands-on lab: Build an employee onboarding listener

In this lab, you will configure a sandbox webhook and validate the basic path required for employee onboarding automation.

## Learning objectives

By the end of the lab, you will be able to:

- create a webhook subscription;
- inspect lifecycle event payloads;
- verify webhook signatures;
- retrieve an employee record through the API;
- identify the event ID used for idempotency; and
- validate delivery status.

## Prerequisites

You need:

- an Orbit sandbox tenant;
- permission to manage webhooks;
- a sandbox API token with employee read access;
- an HTTPS test endpoint; and
- a tool that can display incoming request headers and bodies.

Do not use production employee data for this exercise.

## Task 1: Create the endpoint

Configure an endpoint at:

```text
https://<your-test-host>/orbit/events
```

Your handler should initially:

1. capture the raw request body;
2. capture request headers;
3. return `204 No Content`.

Confirm the endpoint is reachable before continuing.

## Task 2: Create the subscription

Subscribe to:

```text
employee.created
```

Copy the signing secret to a secure local environment variable:

```bash
export ORBIT_WEBHOOK_SECRET="<sandbox-secret>"
```

Do not commit this value.

## Task 3: Send a test event

Use **Send test event** from the subscription.

Locate:

```text
X-Orbit-Event-Id
X-Orbit-Delivery-Id
X-Orbit-Signature
```

In the JSON body, identify:

```text
type
occurred_at
data.employee_id
```

### Checkpoint

You should be able to answer:

- What uniquely identifies the logical event?
- What uniquely identifies this delivery attempt?
- Which identifier should your deduplication store use?

**Expected:** use the event ID for event-level deduplication.

## Task 4: Verify the signature

Update the handler to:

1. parse the timestamp and `v1` digest;
2. construct `<timestamp>.<raw_body>`;
3. calculate HMAC-SHA256 with the sandbox signing secret;
4. compare the digests in constant time; and
5. reject invalid signatures.

Send another test event.

### Expected result

The valid test event returns:

```text
204 No Content
```

Modify one character of a captured request body and replay it with the old signature.

### Expected result

The modified request is rejected.

## Task 5: Retrieve the employee record

Read `data.employee_id` from the verified event.

Call:

```bash
curl \
  --url "https://api.orbit.example/v1/employees/<employee-id>" \
  --header "Authorization: Bearer $ORBIT_ACCESS_TOKEN" \
  --header "Accept: application/json"
```

Confirm the response contains the employee's:

- ID;
- status;
- work email;
- department;
- manager;
- location; and
- start date, when available.

## Task 6: Add deduplication

Store the event ID before executing the simulated onboarding action.

Send the same event twice.

Your log should resemble:

```text
evt_01J... accepted
evt_01J... onboarding action simulated
evt_01J... duplicate ignored
```

Both deliveries may receive `204`, but the business action occurs only once.

## Task 7: Simulate an onboarding decision

Use this rule:

```text
IF event = employee.created
AND employee.status = active
THEN create onboarding task
```

For the lab, log the action instead of modifying another system:

```text
Would create onboarding task for emp_f3002e
```

## Task 8: Validate in delivery history

Return to the webhook delivery history.

Confirm:

- delivery state is `delivered`;
- response code is `204`;
- event type is `employee.created`; and
- the event ID matches your application log.

## Challenge: separate acknowledgment from processing

Modify your design so the HTTP request only:

1. validates the event;
2. persists it;
3. queues it; and
4. returns `204`.

Move employee retrieval and business logic to a worker.

Explain why this design is more resilient when downstream systems are slow or unavailable.

## Completion criteria

You have completed the lab when:

- valid signatures pass;
- invalid signatures fail;
- duplicate events do not duplicate business actions;
- the Employee API returns the authoritative record;
- event and delivery identifiers are visible in logs; and
- Orbit reports successful delivery.
