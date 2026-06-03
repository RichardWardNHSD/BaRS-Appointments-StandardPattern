# Reschedule Appointment

## Purpose

Change the slot against an existing appointment using PATCH. This is the only way to change the `.slot` element — if you also need to change `.status` or `.reasonCode`, use [Update](./04-update-appointment.md) as a separate operation, or use [Rebook](./07-rebook-appointment.md) for a full replacement.

## Endpoint

```
PATCH /Appointment/{id}
```

## Prerequisites

1. **Read before write**: [GET /Appointment/{id}](./03-view-appointment.md) to retrieve the current resource.
2. **Get new slots**: [GET /Slot](./01-get-slots.md) to find an alternative available slot.

## Request

### Step 1: GET the current resource

```
GET https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5
```

### Step 2: GET available slots

```
GET https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Slot?Schedule.actor:HealthcareService=2000072489&start=ge2025-02-12T00:00:00+00:00&end=le2025-02-12T23:59:59+00:00&status=free
```

Select the new slot: `265a53d7-1d21-4fc6-a5b7-761f650e75eb` (10:00–10:10)

### Step 3: PATCH with the new slot

#### Method & URL

```
PATCH https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5
```

#### Headers

| Header | Value |
|---|---|
| `Authorization` | `Bearer {access_token}` |
| `X-Request-Id` | `e5f6a7b8-c9d0-1234-efab-456789012345` |
| `X-Correlation-Id` | `0598efa7-fff0-4ade-9af8-3f46b4124151` |
| `NHSD-End-User-Organisation` | Base64-encoded JSON (ODS: `RYG`) |
| `NHSD-Target-Identifier` | Base64-encoded JSON (service: `2000072489`) |
| `Content-Type` | `application/fhir+json` |
| `Accept` | `application/fhir+json` |

#### Request Body

The full Appointment resource with **only** the `.slot` (and corresponding `.start`/`.end`) updated to the new slot. No other elements should be changed.

```json
{
  "resourceType": "Appointment",
  "id": "aca94bdb-2e38-4399-9ece-2ba083ce65b5",
  "meta": {
    "lastUpdated": "2025-02-10T10:30:00.000+00:00",
    "profile": [
      "https://fhir.hl7.org.uk/StructureDefinition/UKCore-Appointment"
    ]
  },
  "status": "booked",
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

### Example cURL

```bash
curl -X PATCH \
  'https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5' \
  -H 'Authorization: Bearer eyJhbGciOi...' \
  -H 'X-Request-Id: e5f6a7b8-c9d0-1234-efab-456789012345' \
  -H 'X-Correlation-Id: 0598efa7-fff0-4ade-9af8-3f46b4124151' \
  -H 'NHSD-End-User-Organisation: eyJyZXNvdXJjZVR5cGUiOi...' \
  -H 'NHSD-Target-Identifier: eyJzeXN0ZW0iOiJodHRwczovL2ZoaXIubmhzLnVrL0lkL2Rvcy1zZXJ2aWNlLWlkIiwidmFsdWUiOiIyMDAwMDcyNDg5In0=' \
  -H 'Content-Type: application/fhir+json' \
  -H 'Accept: application/fhir+json' \
  -d @appointment-reschedule.json
```

## Response

### HTTP 200 – Success

Returns the rescheduled Appointment resource with the new slot.

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
  "status": "booked",
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

## After Rescheduling

The Receiver must **update (PUT)** the pointer in the central Registry to reflect the new slot time:

```json
"context": {
  "period": {
    "start": "2025-02-12T10:00:00+00:00",
    "end": "2025-02-12T10:10:00+00:00"
  }
}
```

### Error Responses

| HTTP Status | Meaning |
|---|---|
| 400 | Bad request (invalid resource) |
| 404 | Appointment not found |
| 409 | Conflict (slot no longer available) |
| 422 | Unprocessable entity (validation failure) |

## When to Use Reschedule vs Rebook

| Scenario | Use |
|---|---|
| Same service, different slot | **Reschedule** (PATCH) |
| Different service entirely | **Rebook** (Cancel + new Book) |
| Need to change status AND slot | **Rebook** (Cancel + new Book) |
| Only changing slot | **Reschedule** (PATCH) |
