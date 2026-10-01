---
title: Integration permissions matrix
description: Least-privilege permissions for administrators and services that configure and operate employee lifecycle integrations.
audience:
  - administrators
  - security reviewers
  - implementation engineers
product_area: Platform Integrations
content_type: reference
---

# Integration permissions matrix

Use least-privilege permissions for both human administrators and service credentials.

## Human roles

| Activity | Integration Admin | HR Admin | Developer | Support Viewer |
| --- | :---: | :---: | :---: | :---: |
| View webhook subscriptions | ✓ | — | ✓ | ✓ |
| Create or edit subscriptions | ✓ | — | ✓ | — |
| Rotate signing secret | ✓ | — | — | — |
| View delivery metadata | ✓ | — | ✓ | ✓ |
| Replay failed delivery | ✓ | — | ✓ | — |
| View employee records in UI | As granted | ✓ | As granted | As granted |
| Manage employee data | — | ✓ | — | — |

Access to employee data should be governed separately from access to webhook configuration.

## Service credential scopes

A read-only lifecycle integration usually needs:

```text
employees:read
webhooks:read
```

A deployment or administration service that creates subscriptions may additionally need:

```text
webhooks:write
```

Do not grant employee write access unless the integration explicitly modifies employee records.

## Separation of duties

For production:

- security or integration administrators should control secret rotation;
- application services should consume secrets from a secret manager;
- developers should not need routine access to production secret values;
- support personnel should be able to inspect delivery metadata without receiving credentials.

## Logging access

Webhook logs can contain identifiers and operational metadata.

Restrict logs according to your organization's data-handling policy and redact:

- bearer tokens;
- signing secrets;
- session cookies;
- unnecessary personal information.

## Production review checklist

Before enabling the integration:

- API scopes match documented requirements;
- human access follows least privilege;
- secrets are stored outside source control;
- secret rotation has an owner;
- logs do not expose credentials;
- replay permission is limited to authorized operators; and
- production and sandbox credentials are separate.
