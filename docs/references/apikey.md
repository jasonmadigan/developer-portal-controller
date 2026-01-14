# The APIKey Custom Resource Definition (CRD)

## Overview

The APIKey CRD is part of the Developer Portal extension for Kuadrant. It represents a request for API access credentials by a developer for a specific APIProduct and plan tier. When approved, the APIKey creates a Kubernetes Secret containing the actual API key that can be used to authenticate requests. The APIKey resource manages the entire lifecycle of API access requests, from initial submission through approval/rejection to credential generation.

## APIKey

| **Field** | **Type**                          | **Required** | **Description**                           |
|-----------|-----------------------------------|:------------:|-------------------------------------------|
| `spec`    | [APIKeySpec](#apikeyspec)         | Yes          | The specification for APIKey custom resource |
| `status`  | [APIKeyStatus](#apikeystatus)     | No           | The status for the custom resource        |

## APIKeySpec

| **Field**       | **Type**                                  | **Required** | **Description**                                                          |
|-----------------|-------------------------------------------|:------------:|--------------------------------------------------------------------------|
| `apiProductRef` | [APIProductReference](#apiproductreference) | Yes       | Reference to the APIProduct this API key provides access to             |
| `planTier`      | String                                    | Yes          | Tier of the plan (e.g., "premium", "basic", "enterprise")                |
| `requestedBy`   | [RequestedBy](#requestedby)               | Yes          | Information about who requested the API key                              |
| `useCase`       | String                                    | Yes          | Description of how the API key will be used                              |

### APIProductReference

| **Field** | **Type** | **Required** | **Description**                              |
|-----------|----------|:------------:|----------------------------------------------|
| `name`    | String   | Yes          | Name of the APIProduct in the same namespace |

### RequestedBy

| **Field** | **Type** | **Required** | **Description**                                                |
|-----------|----------|:------------:|----------------------------------------------------------------|
| `userId`  | String   | Yes          | Identifier of the user requesting the API key                  |
| `email`   | String   | Yes          | Email address of the user (must be valid email format)         |

## APIKeyStatus

| **Field**       | **Type**                          | **Description**                                                                   |
|-----------------|-----------------------------------|-----------------------------------------------------------------------------------|
| `phase`         | String                            | Current phase of the APIKey. Valid values: `Pending`, `Approved`, `Rejected`     |
| `conditions`    | [][ConditionSpec](#conditionspec) | Represents the observations of the APIKey's current state                         |
| `secretRef`     | [SecretReference](#secretreference) | Reference to the created Secret containing the API key (only when Approved)     |
| `limits`        | [Limits](#limits)                 | Rate limits for the plan                                                          |
| `apiHostname`   | String                            | Hostname from the HTTPRoute that the APIProduct references                        |
| `reviewedBy`    | String                            | Who approved or rejected the request                                              |
| `reviewedAt`    | Timestamp                         | When the request was reviewed                                                     |
| `canReadSecret` | Boolean                           | Permission to read the APIKey's secret. Default: `true`                           |

### ConditionSpec

Standard Kubernetes condition type with the following fields:

| **Field**            | **Type**  | **Description**                                                                   |
|----------------------|-----------|-----------------------------------------------------------------------------------|
| `type`               | String    | Condition type (e.g., `Ready`)                                                    |
| `status`             | String    | Status of the condition: `True`, `False`, or `Unknown`                            |
| `reason`             | String    | Unique, one-word, CamelCase reason for the condition's last transition            |
| `message`            | String    | Human-readable message indicating details about the transition                    |
| `lastTransitionTime` | Timestamp | Last time the condition transitioned from one status to another                   |
| `observedGeneration` | Integer   | The .metadata.generation that the condition was set based upon                    |

### SecretReference

| **Field** | **Type** | **Required** | **Description**                              |
|-----------|----------|:------------:|----------------------------------------------|
| `name`    | String   | Yes          | Name of the secret in the Authorino's namespace |
| `key`     | String   | Yes          | The key of the secret to select from         |

### Limits

| **Field**  | **Type**      | **Required** | **Description**                                                    |
|------------|---------------|:------------:|--------------------------------------------------------------------|
| `daily`    | Integer       | No           | Daily limit of requests for this plan                              |
| `weekly`   | Integer       | No           | Weekly limit of requests for this plan                             |
| `monthly`  | Integer       | No           | Monthly limit of requests for this plan                            |
| `yearly`   | Integer       | No           | Yearly limit of requests for this plan                             |
| `custom`   | [][Rate](#rate) | No         | Additional limits defined in terms of a RateLimitPolicy Rate       |

### Rate

| **Field**  | **Type** | **Required** | **Description**                                                    |
|------------|----------|:------------:|--------------------------------------------------------------------|
| `limit`    | Integer  | Yes          | Maximum value allowed for a given period of time                   |
| `window`   | String   | Yes          | Time period for which the limit applies (pattern: `^([0-9]{1,5}(h\|m\|s\|ms)){1,4}$`) |

## High level example

```yaml
apiVersion: devportal.kuadrant.io/v1alpha1
kind: APIKey
metadata:
  name: developer-john-premium
  namespace: payment-services
spec:
  apiProductRef:
    name: payment-api
  planTier: premium
  requestedBy:
    userId: john-doe-123
    email: john.doe@example.com
  useCase: Building a mobile payment application for retail customers
```

## Relationship to APIProduct and AuthPolicy

### APIProduct

APIKey **must** reference an existing APIProduct via `apiProductRef`. The APIProduct defines the API being accessed.

### AuthPolicy

AuthPolicy is applied to the HTTPRoute that the APIProduct references. When an APIKey is approved, a Kubernetes Secret is created with an annotation `secret.kuadrant.io/plan-id` value. The AuthPolicy validates incoming API requests by checking the API key against secrets that match specific label selectors.

### PlanPolicy

PlanPolicy defines the available tiers and their corresponding rate limits. When an APIKey specifies a `planTier`, the controller validates that this tier exists in the PlanPolicy attached to the HTTPRoute. If the tier is valid and the APIKey is approved, the Secret is annotated with `secret.kuadrant.io/plan-id: <planTier>`, allowing PlanPolicy's CEL predicates to match the request to the appropriate rate limits.
