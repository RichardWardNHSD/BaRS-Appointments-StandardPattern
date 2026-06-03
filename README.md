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

- Completed [BaRS Proxy onboarding](../current-onboarding-process.md) (Sender and/or Receiver)
- A valid OAuth bearer token (application-restricted, signed JWT)
- Selected a target service via [Service Discovery](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Core/1-4-1/End-to-end-workflow/Service-Discovery?version=1.11.1)
- Confirmed capabilities via `GET /metadata`

## Key Rules

- **Read before write**: All PUT and PATCH operations must be preceded by a GET of the resource.
- **Headers are mandatory**: Every request must include `X-Request-Id`, `X-Correlation-Id`, and `NHSD-End-User-Organisation-ODS`.
- **NHSD-Target-Identifier**: Required for all requests routed via the BaRS Proxy (Base64-encoded JSON identifying the target service).
- **The Appointment profile**: All Appointment resources must conform to [UKCore-Appointment](https://simplifier.net/HL7FHIRUKCoreR4/UKCore-Appointment).
