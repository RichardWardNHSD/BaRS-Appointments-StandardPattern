# Rebook Appointment

## Purpose

A rebook is a **composite operation**: cancel the existing appointment and then create a new one. Use this when a simple reschedule (PATCH) is insufficient — for example, when booking with a different service or when multiple fields need to change.

## Endpoint

This is not a single API call. It is a combination of:

1. [Cancel Appointment](./05-cancel-appointment.md) – `PUT /Appointment/{id}` with `status: "cancelled"`
2. [Book Appointment](./02-book-appointment.md) – `POST /Appointment` with a new slot

## When to Use Rebook

| Scenario | Operation |
|---|---|
| Patient needs a different slot at the **same** service | Use [Reschedule](./06-reschedule-appointment.md) (PATCH) |
| Patient needs a slot at a **different** service | Use Rebook (this page) |
| Existing booking needs multiple changes beyond just slot | Use Rebook (this page) |
| Booking was made in error and patient needs a fresh one | Use Rebook (this page) |

## Workflow

### Step 1: Cancel the Existing Appointment

Follow the full [Cancel Appointment](./05-cancel-appointment.md) process:

1. **GET** the current appointment:

```
GET https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5
```

2. **PUT** with status `cancelled`:

```
PUT https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5
```

```json
{
  "resourceType": "Appointment",
  "id": "aca94bdb-2e38-4399-9ece-2ba083ce65b5",
  "meta": {
    "lastUpdated": "2025-02-10T11:45:00.000+00:00",
    "profile": [
      "https://fhir.hl7.org.uk/StructureDefinition/UKCore-Appointment"
    ]
  },
  "status": "cancelled",
  "reasonCode": [
    {
      "coding": [
        {
          "system": "http://snomed.info/sct",
          "code": "182856006",
          "display": "Appointment rebooked (finding)"
        }
      ]
    }
  ],
  "slot": [
    {
      "reference": "Slot/265a53d7-1d21-4fc6-a5b7-761f650e75eb"
    }
  ],
  "description": "Patient reported nosebleed without injury",
  "start": "2025-02-12T10:00:00+00:00",
  "end": "2025-02-12T10:10:00+00:00",
  "created": "2025-02-10T09:15:00+00:00",
  "participant": [
    {
      "actor": {
        "reference": "Patient/788660eb-d2c9-4773-abd4-318484673fb2",
        "identifier": {
          "system": "https://fhir.nhs.uk/Id/nhs-number",
          "value": "9876543210"
        },
        "display": "John Smith"
      },
      "status": "accepted"
    }
  ]
}
```

The Receiver must DELETE the old DocumentReference pointer from the Registry.

---

### Step 2: Book a New Appointment

Follow the full [Book Appointment](./02-book-appointment.md) process:

1. **GET /Slot** to find available slots (may be at a different service):

```
GET https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Slot?Schedule.actor:HealthcareService=2000072489&start=ge2025-02-13T00:00:00+00:00&end=le2025-02-13T23:59:59+00:00&status=free
```

2. Select a new slot (e.g., `f1a2b3c4-d5e6-7890-abcd-ef1234567890`, 2025-02-13 14:00–14:10)

3. **POST /Appointment** with the new slot:

```
POST https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment
```

```json
{
  "resourceType": "Appointment",
  "meta": {
    "profile": [
      "https://fhir.hl7.org.uk/StructureDefinition/UKCore-Appointment"
    ]
  },
  "status": "booked",
  "slot": [
    {
      "reference": "Slot/f1a2b3c4-d5e6-7890-abcd-ef1234567890"
    }
  ],
  "description": "Patient reported nosebleed without injury - rebooked",
  "start": "2025-02-13T14:00:00+00:00",
  "end": "2025-02-13T14:10:00+00:00",
  "created": "2025-02-10T12:00:00+00:00",
  "participant": [
    {
      "actor": {
        "reference": "Patient/788660eb-d2c9-4773-abd4-318484673fb2",
        "identifier": {
          "system": "https://fhir.nhs.uk/Id/nhs-number",
          "value": "9876543210"
        },
        "display": "John Smith"
      },
      "status": "accepted"
    }
  ]
}
```

### Response – New Appointment Created

```json
{
  "resourceType": "Appointment",
  "id": "f8c3de3d-1729-4b67-8f1a-2c9d4e5f6a7b",
  "meta": {
    "lastUpdated": "2025-02-10T12:00:30.000+00:00",
    "profile": [
      "https://fhir.hl7.org.uk/StructureDefinition/UKCore-Appointment"
    ]
  },
  "status": "booked",
  "slot": [
    {
      "reference": "Slot/f1a2b3c4-d5e6-7890-abcd-ef1234567890"
    }
  ],
  "description": "Patient reported nosebleed without injury - rebooked",
  "start": "2025-02-13T14:00:00+00:00",
  "end": "2025-02-13T14:10:00+00:00",
  "created": "2025-02-10T12:00:00+00:00",
  "participant": [
    {
      "actor": {
        "reference": "Patient/788660eb-d2c9-4773-abd4-318484673fb2",
        "identifier": {
          "system": "https://fhir.nhs.uk/Id/nhs-number",
          "value": "9876543210"
        },
        "display": "John Smith"
      },
      "status": "accepted"
    }
  ]
}
```

The new Appointment ID is `f8c3de3d-1729-4b67-8f1a-2c9d4e5f6a7b`. The Receiver must POST a new DocumentReference pointer to the Registry for this booking.

---

## Summary of Operations

| Step | Method | Endpoint | Purpose |
|---|---|---|---|
| 1a | GET | `/Appointment/aca94bdb-...` | Read existing appointment |
| 1b | PUT | `/Appointment/aca94bdb-...` | Cancel existing appointment |
| 2a | GET | `/Slot?...` | Find new available slots |
| 2b | POST | `/Appointment` | Create new booking |

## Headers (All Requests)

| Header | Value |
|---|---|
| `Authorization` | `Bearer {access_token}` |
| `X-Request-Id` | Unique UUID per request |
| `X-Correlation-Id` | `0598efa7-fff0-4ade-9af8-3f46b4124151` (same across the rebook flow) |
| `NHSD-End-User-Organisation` | Base64-encoded JSON (ODS: `RYG`) |
| `NHSD-Target-Identifier` | Base64-encoded JSON (target service) |
| `Content-Type` | `application/fhir+json` (for POST/PUT) |
| `Accept` | `application/fhir+json` |

> **Note**: The `X-Correlation-Id` should remain the same across both the cancel and the new booking to allow the full rebook flow to be traced as a single logical operation.

## Error Handling

If the cancel succeeds but the new booking fails, the patient is left without an appointment. Implementers should consider:

- Retrying the new booking
- Alerting the user that manual intervention may be needed
- Logging the correlation ID for support investigation
