# Standard Pattern – Appointments API Guide

This folder contains detailed guides for each API interaction in the BaRS Standard Pattern for Appointment Management.

All examples use **consistent, complementary data** so you can follow the full lifecycle of an appointment from booking through to cancellation or rebook.

## Common Data

The following identifiers are used consistently across all examples:

| Item | Value |
|---|---|
| **Patient ID** | `788660eb-d2c9-4773-abd4-318484673fb2` |
| **Patient NHS Number** | `9876543210` |
| **Patient Name** | John Smith |
| **Appointment ID** | `aca94bdb-2e38-4399-9ece-2ba083ce65b5` |
| **Original Slot ID** | `deb4c4b3-870b-4599-84df-5e54cef7afda` |
| **Rescheduled Slot ID** | `265a53d7-1d21-4fc6-a5b7-761f650e75eb` |
| **Rebooked Appointment ID** | `f8c3de3d-1729-4b67-8f1a-2c9d4e5f6a7b` |
| **HealthcareService ID** | `b5e0e3c2-7f1a-4d3b-9c8e-6a2d4f5e7b1c` |
| **Service Identifier** | `2000072489` |
| **Sender ODS Code** | `RYG` |
| **Receiver ODS Code** | `RXF` |
| **Original Slot Time** | 2025-02-12 12:30–12:40 |
| **Rescheduled Slot Time** | 2025-02-12 10:00–10:10 |
| **Base URL (INT)** | `https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4` |

## Pages

1. [Get Slots](./01-get-slots.md) – Search for available booking slots
2. [Book Appointment](./02-book-appointment.md) – Create an initial booking
3. [View Appointment](./03-view-appointment.md) – Retrieve an existing appointment
4. [Update Appointment](./04-update-appointment.md) – Modify status or reason
5. [Cancel Appointment](./05-cancel-appointment.md) – Cancel an existing booking
6. [Reschedule Appointment](./06-reschedule-appointment.md) – Change the slot (PATCH)
7. [Rebook Appointment](./07-rebook-appointment.md) – Cancel and create a new booking

## Prerequisites

Before using these APIs, ensure you have:

- **If using the BaRS Proxy**: Completed [BaRS Proxy onboarding](../current-onboarding-process.md) (Sender and/or Receiver). *The Proxy is optional — teams may implement the Standard Pattern directly between systems without it. See [Internal Onboarding Guide, Option B](../internal-onboarding-standard-pattern.md#option-b-standard-only-no-proxy) for details.*
- A valid OAuth bearer token (application-restricted, signed JWT) — or equivalent authentication if not using the Proxy
- Selected a target service via [Service Discovery](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Core/1-4-1/End-to-end-workflow/Service-Discovery?version=1.11.1) (or your own service resolution mechanism if not using the Proxy)
- Called `GET /metadata` on the target service to confirm its capabilities (see below)

### GET /metadata – Capability Check

Before your **first API interaction** with a target service, you must call `GET /metadata` to retrieve its [CapabilityStatement](https://www.hl7.org/fhir/capabilitystatement.html). This tells you what the receiver supports — which interactions, which resource types, and which message definitions are available.

You only need to do this **once per target service** (not before every individual call). Cache the result and use it to confirm the receiver supports the operations you intend to perform.

```
GET https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/metadata
```

**Headers:**

| Header | Value |
|---|---|
| `Authorization` | `Bearer {access_token}` |
| `X-Request-Id` | `{uuid}` |
| `X-Correlation-Id` | `{uuid}` |
| `NHSD-End-User-Organisation` | Base64-encoded JSON (ODS: `RYG`) |
| `NHSD-Target-Identifier` | Base64-encoded JSON (service: `2000072489`) |
| `Accept` | `application/fhir+json` |

The response is a FHIR CapabilityStatement describing the target receiver's supported resources, interactions, and message definitions. If using the BaRS Proxy and the receiver does not support BaRS functionality, the Proxy will return an error — in which case, you should pursue an alternative workflow.

> **Key point**: Do not proceed with any of the operations below until you have confirmed the target supports them via `/metadata`.

## Key Rules

- **Read before write**: All PUT and PATCH operations must be preceded by a GET of the resource.
- **Headers are mandatory**: Every request must include `X-Request-Id`, `X-Correlation-Id`, and `NHSD-End-User-Organisation-ODS`.
- **NHSD-Target-Identifier**: Required when requests are routed via the BaRS Proxy (Base64-encoded JSON identifying the target service). Not required if communicating directly with a receiver without the Proxy.
- **The Appointment profile**: All Appointment resources must conform to [UKCore-Appointment](https://simplifier.net/HL7FHIRUKCoreR4/UKCore-Appointment).

## Internal Integrations (NHS-to-NHS, No Proxy)

For internal integrations where NHS systems communicate directly without the BaRS Proxy, the `NHSD-End-User-Organisation` and `NHSD-Target-Identifier` headers are **still included** in requests to maintain conformance with the standard, but they carry **dummy/placeholder values** since they are not used for routing or authorisation in this context.

This ensures:
- The API contract remains consistent regardless of whether the Proxy is in the path.
- Systems can transition to Proxy-routed traffic in the future without code changes to the request structure.
- Receivers can be built once and work in both Proxy and direct integration scenarios.

### Dummy Header Values for Internal Integrations

#### NHSD-End-User-Organisation (dummy)

Use a valid structure with your own ODS code (or a placeholder if ODS codes are not relevant to your integration):

```json
{
  "resourceType": "Organization",
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "X26"
    }
  ],
  "name": "NHS England"
}
```

Base64-encode this and include it in the header as normal.

#### NHSD-Target-Identifier (dummy)

Use a valid structure with a locally-agreed service identifier (this does not need to exist in the Endpoint Catalogue since routing is handled directly):

```json
{
  "system": "https://fhir.nhs.uk/Id/dos-service-id",
  "value": "INTERNAL-001"
}
```

Base64-encode this and include it in the header as normal.

### What the Receiver Should Do

When operating in a direct (non-Proxy) integration:

- The Receiver **should accept** these headers without validation against external systems (no Endpoint Catalogue lookup, no ODS ownership check).
- The Receiver **may log** the header values for audit/tracing purposes.
- The Receiver **must not reject** requests solely because the header values don't resolve to real Proxy-registered entities.

> **Tip**: Agree the dummy values between Sender and Receiver teams upfront and document them in your integration specification. The examples in this guide use `RYG` and `2000072489` — for internal integrations, substitute with your agreed placeholders.
