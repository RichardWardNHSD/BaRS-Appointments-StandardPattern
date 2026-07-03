# Patient-Facing BaRS (NHS Identity)

## Overview

This page details what needs to change for BaRS Appointment interactions to be **patient-facing** — i.e., initiated directly by a citizen authenticated via [NHS Identity](https://digital.nhs.uk/services/nhs-identity) (formerly NHS Login). This shifts the trust model from organisation-to-organisation (B2B) to citizen-to-service (B2C), with significant implications for authentication, authorisation, data access, and API design.

## What Is NHS Identity?

NHS Identity is the national single sign-on and identity verification service for patients and citizens accessing NHS digital services. It provides:

- **Identity verification levels (proofing):** Low (P0), Medium (P5), High (P9)
- **Authentication:** Federated OpenID Connect (OIDC) identity tokens
- **NHS Number linkage:** Verified NHS Number bound to the authenticated identity
- **Scope-based consent:** Patient can grant granular access to their data

For patient-facing BaRS, the minimum identity verification level is **P9 (High)** — the patient's identity has been verified to a high level of confidence and their NHS Number is confirmed.

## Key Differences from B2B BaRS

| Aspect | B2B (Current) | Patient-Facing (B2C) |
|---|---|---|
| **Authentication** | Application-restricted (signed JWT, client credentials) | User-restricted (NHS Identity OIDC token + application token) |
| **Who initiates** | A healthcare system on behalf of a clinician/operator | The patient themselves via an app or website |
| **Patient context** | Passed in request body (NHS Number in FHIR resource) | Derived from the authenticated identity token — the patient can only act on their own record |
| **Authorisation model** | Organisation-level (ODS code trusted) | User-level (patient can only book/view/cancel their own appointments) |
| **Consent** | Implied (organisation has legitimate relationship) | Explicit (OAuth scopes granted by the patient) |
| **Target header** | `NHSD-End-User-Organisation` carries the sending org | Replaced or supplemented by patient identity context |

## What Needs to Change

### 1. Authentication – User-Restricted Access

**Current state:** BaRS uses application-restricted access. The calling system authenticates itself using a signed JWT (client credentials flow). There is no individual user identity in the token.

**Required change:** Patient-facing access requires **user-restricted authentication** via the NHS Identity / NHS Login OIDC flow:

1. The patient authenticates via NHS Identity (P9 level).
2. The patient's app receives an **ID token** (containing verified NHS Number and identity claims) and an **access token** (scoped to permitted actions).
3. The app exchanges these for an **NHS API Platform access token** using the token exchange or authorisation code flow.
4. API calls include this user-restricted access token in the `Authorization` header.

**Impact on the NHS API Platform and receivers:**
- The **NHS API Platform Authorisation Server** issues a user-restricted access token containing claims that identify the patient (`sub` = patient's NHS Number) and the actor (`act` = the logged-in user, relevant for delegated/proxy access).
- The API Platform does **not** extract the NHS Number and pass it downstream via a separate header. The claims are embedded within the access token itself.
- The **Receiver** (API producer) must validate the access token and extract the `sub` claim to determine the patient's NHS Number. The receiver is then responsible for enforcing that the request only accesses resources belonging to that patient.

> **Reference:** See [NHS API Producer Guidance (PXY-1)](https://nhsdigital.github.io/national-proxy-service-integration-docs/patient-facing-journeys/api-producer-guidance): "If a patient identifier is provided in the request route or query string, it must be verified against the `sub` claim in the access token."

### 2. Scoping and Consent

The OAuth scopes for patient-facing BaRS need to be defined. Suggested scope model:

| Scope | Permits |
|---|---|
| `urn:nhsd:apim:user-nhs-id:aal3:booking-and-referral/patient-access` | Base access to patient-facing BaRS |
| `patient/Slot.read` | Search for available slots |
| `patient/Appointment.read` | View own appointments |
| `patient/Appointment.write` | Book, update, or cancel own appointments |

The patient must explicitly consent to these scopes during the NHS Identity login flow. The scopes constrain what operations the access token permits.

### 3. Patient Context Enforcement

**Current state:** The patient's NHS Number is included in the request body (e.g., in the `participant.actor.identifier` of a booking request). The system is trusted to supply the correct patient identity.

**Required change:** In patient-facing mode, the patient identity is **derived from the access token claims**, not trusted from the request body:

- The access token contains a `sub` claim holding the patient's NHS Number (and an `act` claim identifying the proxy/actor, if delegated access is in use).
- The **Receiver** must validate the access token, extract the `sub` claim, and verify that any patient identifier in the request (URL, query string, or body) matches the `sub` claim. If there is a mismatch, the receiver **must reject** the request (HTTP 401 per NHS API Producer Guidance).
- This prevents a patient from booking on behalf of another person (unless a delegated access model is in use, in which case the `act` claim identifies the proxy and `sub` identifies the patient they are acting for).

**Validation rule:**
```
token.sub == request.body.participant[*].actor.identifier.value (NHS Number)
```

### 4. New / Modified Headers

| Header | Change |
|---|---|
| `Authorization` | Now carries a **user-restricted** bearer token (not application-restricted) |
| `NHSD-End-User-Organisation` | **Not applicable** for patient-facing. Could be omitted or carry a placeholder indicating "patient direct access" |
| `NHSD-End-User-Identity` | **New** — carries claims or a reference to the authenticated patient identity (NHS Number, verification level) |
| `NHSD-Target-Identifier` | Unchanged — still identifies the target service |
| `X-Request-Id` / `X-Correlation-Id` | Unchanged |

### 5. Permitted Operations (Patient Scope)

Not all BaRS operations are appropriate for patients. The patient-facing subset should be:

| Operation | Patient Can Do? | Notes |
|---|---|---|
| `GET /Slot` | ✅ Yes | Search available slots at a selected service |
| `POST /Appointment` | ✅ Yes | Book an appointment for themselves |
| `GET /Appointment/{id}` | ✅ Yes | View their own appointment only |
| `PATCH /Appointment/{id}` | ⚠️ Limited | Cancel own appointment (status → `cancelled`). Reschedule may be permitted. |
| `PUT /Appointment/{id}` | ❌ No | Full resource replacement not appropriate for patients |

### 6. Service Discovery for Patients

**Current state:** Service discovery uses clinical/operational criteria (DoS search by specialty, geography, etc.) and is performed by a healthcare system.

**Required change:** For patient-facing booking, the service discovery model needs to be simplified:

- **Option A – Pre-curated services:** The patient-facing app presents a curated list of bookable services (e.g., "GP surgery", "local pharmacy", "UTC near me") without exposing the full DoS query interface.
- **Option B – Constrained search:** The patient provides location/service type and the app performs a DoS or Endpoint Catalogue query on their behalf, filtering to services that accept patient-initiated bookings.
- **Option C – Deep link:** The patient is directed to a specific service (e.g., via NHS.uk or NHS App) and the service identifier is pre-populated.

In all cases, the target service must have a flag or capability indicating it **accepts patient-initiated bookings**. This is a new capability that needs to be expressed in the service's `CapabilityStatement` or directory entry.

### 7. CapabilityStatement Extension

The receiver's `/metadata` response should indicate support for patient-facing interactions. Suggested approach:

```json
{
  "resourceType": "CapabilityStatement",
  "rest": [{
    "security": {
      "service": [{
        "coding": [{
          "system": "http://terminology.hl7.org/CodeSystem/restful-security-service",
          "code": "SMART-on-FHIR"
        }]
      }],
      "extension": [{
        "url": "https://fhir.nhs.uk/StructureDefinition/BaRS-PatientFacing-Access",
        "valueBoolean": true
      }]
    },
    "resource": [{
      "type": "Appointment",
      "interaction": [
        {"code": "create"},
        {"code": "read"},
        {"code": "patch"}
      ],
      "extension": [{
        "url": "https://fhir.nhs.uk/StructureDefinition/BaRS-PatientFacing-Supported",
        "valueBoolean": true
      }]
    }]
  }]
}
```

### 8. Audit and Logging

**Current state:** Audit trails record the sending organisation (ODS code) and correlation IDs.

**Required change:** Patient-facing interactions must additionally record:

- The authenticated patient's NHS Number (from the token)
- The identity verification level (P9)
- The patient-facing application identifier (client_id of the app)
- Consent scope granted

This is critical for Data Protection (UK GDPR) compliance and for investigating complaints or disputes.

### 9. Rate Limiting and Abuse Prevention

Patient-facing APIs are exposed to a much larger and less trusted user base than B2B APIs. Additional protections are needed:

- **Per-user rate limits:** Limit how many bookings/searches a single NHS Number can perform in a time window.
- **Per-app rate limits:** Limit the total throughput of a patient-facing application.
- **Slot hoarding prevention:** Prevent a patient from booking multiple overlapping slots. Consider a "hold" pattern with a short TTL rather than immediate hard booking.
- **CAPTCHA / bot prevention:** If exposed via web, consider anti-automation measures.

### 10. Cancellation and No-Show Policies

When a patient cancels their own appointment:

- The `reasonCode` should use a patient-facing value set (e.g., "Patient requested cancellation").
- Services may define a minimum cancellation notice period — the API should return an appropriate error if the cancellation is too late.
- Repeated no-shows or late cancellations may trigger business rules (outside the scope of the API, but the Receiver should be able to enforce them).

## Receiver Responsibilities

Systems that wish to accept patient-initiated bookings must:

1. Declare patient-facing support in their `CapabilityStatement`.
2. Validate the user-restricted access token (signature, expiry, scopes).
3. Extract the `sub` claim (patient NHS Number) and `act` claim (proxy identity, if applicable) from the token.
4. Verify that any patient identifier in the request (URL, query string, or body) matches the `sub` claim. Reject mismatches with HTTP 401.
5. Enforce that the patient can only interact with their own resources (own-record-only access).
6. Apply appropriate business rules (cancellation windows, slot limits, etc.).
7. Return patient-appropriate error messages (no internal system details in OperationOutcome).
8. Audit both the patient (`sub`) and the actor (`act`) correctly — see [NHS API Producer Guidance PXY-4](https://nhsdigital.github.io/national-proxy-service-integration-docs/patient-facing-journeys/api-producer-guidance).

## Open Questions and Future Considerations

| Topic | Notes |
|---|---|
| **Proxy access** | Will patient-facing access be offered at the NHS App level (patient hits the proxy) or does each patient-facing app talk directly to receivers? |
| **Delegated access** | Can a parent/guardian book on behalf of a child? Can a carer book on behalf of an adult? This requires a delegation model (not yet defined). |
| **Notifications** | Should the patient receive push/email/SMS notifications when appointment status changes? What channel and what standard? |
| **Waitlists** | If no slot is available, can the patient join a waitlist? This extends beyond current BaRS scope. |
| **Multi-provider journeys** | Patient books at a service that then refers onwards — how does the patient retain visibility? |
| **PDS linkage** | Should the API perform a PDS lookup to enrich the Appointment resource, or must the patient-facing app provide full demographics? |
| **Accessibility** | Patient-facing apps must meet WCAG 2.2 AA. Error messages and flows must be understandable by non-clinical users. |

## Summary of Changes Required

```
┌─────────────────────────────────────────────────────────────┐
│                  PATIENT-FACING BaRS                         │
├─────────────────────────────────────────────────────────────┤
│  NHS Identity (P9)                                          │
│       │                                                     │
│       ▼                                                     │
│  Composite Identity Token (sub=patient, act=proxy)          │
│       │                                                     │
│       ▼                                                     │
│  Token Exchange (NHS API Platform /oauth2/token)            │
│       │                                                     │
│       ▼                                                     │
│  User-Restricted Access Token (claims inside token)         │
│       │                                                     │
│       ▼                                                     │
│  Patient-Facing App ──► Receiver (BaRS API)                 │
│                                │                            │
│                                ▼                            │
│                       Receiver validates token              │
│                       Extracts sub claim (NHS Number)       │
│                       Matches sub against request body      │
│                       Enforces own-record-only access       │
│                       Audits sub + act                      │
│                                │                            │
│                                ▼                            │
│                       Appointment created/viewed/cancelled  │
│                       (patient's own record only)           │
│                                                             │
│  Audit: sub (patient) + act (proxy) + app ID + scopes      │
└─────────────────────────────────────────────────────────────┘
```

## Next Steps

- Define the OAuth scope model with the NHS API Platform team.
- Agree the CapabilityStatement extension for patient-facing support.
- Define receiver-side token validation and `sub` claim matching requirements for BaRS.
- Identify pilot services willing to accept patient-initiated bookings.
- Produce patient-facing error message guidance (plain English OperationOutcome display text).
