# Update Appointment

## Purpose

Modify an existing Appointment resource. Currently only `.status` and `.reasonCode` can be updated. If you need to change the slot, use [Reschedule](./06-reschedule-appointment.md) instead.

## Endpoint

```
PUT /Appointment/{id}
```

## Prerequisites

**Read before write**: You must first perform a [GET /Appointment/{id}](./03-view-appointment.md) to retrieve the current resource. The update is then applied to the resource you received.

## Request

### Step 1: GET the current resource

```
GET https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5
```

(See [View Appointment](./03-view-appointment.md) for full details)

### Step 2: PUT the modified resource

#### Method & URL

```
PUT https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5
```

#### Headers

| Header | Value |
|---|---|
| `Authorization` | `Bearer {access_token}` |
| `X-Request-Id` | `c3d4e5f6-a7b8-9012-cdef-234567890123` |
| `X-Correlation-Id` | `0598efa7-fff0-4ade-9af8-3f46b4124151` |
| `NHSD-End-User-Organisation` | Base64-encoded JSON (ODS: `RYG`) |
| `NHSD-Target-Identifier` | Base64-encoded JSON (service: `2000072489`) |
| `Content-Type` | `application/fhir+json` |
| `Accept` | `application/fhir+json` |

#### Request Body

In this example, we add a `.reasonCode` to the appointment. Note the full resource is sent — not just the changed fields.

```json
{
  "resourceType": "Appointment",
  "id": "aca94bdb-2e38-4399-9ece-2ba083ce65b5",
  "meta": {
    "lastUpdated": "2025-02-10T09:15:30.818+00:00",
    "profile": [
      "https://fhir.hl7.org.uk/StructureDefinition/UKCore-Appointment"
    ]
  },
  "status": "booked",
  "reasonCode": [
    {
      "coding": [
        {
          "system": "http://snomed.info/sct",
          "code": "249366005",
          "display": "Bleeding from nose (finding)"
        }
      ]
    }
  ],
  "slot": [
    {
      "reference": "Slot/deb4c4b3-870b-4599-84df-5e54cef7afda"
    }
  ],
  "description": "Patient reported nosebleed without injury",
  "start": "2025-02-12T12:30:00+00:00",
  "end": "2025-02-12T12:40:00+00:00",
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
curl -X PUT \
  'https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5' \
  -H 'Authorization: Bearer eyJhbGciOi...' \
  -H 'X-Request-Id: c3d4e5f6-a7b8-9012-cdef-234567890123' \
  -H 'X-Correlation-Id: 0598efa7-fff0-4ade-9af8-3f46b4124151' \
  -H 'NHSD-End-User-Organisation: eyJyZXNvdXJjZVR5cGUiOi...' \
  -H 'NHSD-Target-Identifier: eyJzeXN0ZW0iOiJodHRwczovL2ZoaXIubmhzLnVrL0lkL2Rvcy1zZXJ2aWNlLWlkIiwidmFsdWUiOiIyMDAwMDcyNDg5In0=' \
  -H 'Content-Type: application/fhir+json' \
  -H 'Accept: application/fhir+json' \
  -d @appointment-update.json
```

## Response

### HTTP 200 – Success

Returns the updated Appointment resource with a new `meta.lastUpdated` timestamp.

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
  "reasonCode": [
    {
      "coding": [
        {
          "system": "http://snomed.info/sct",
          "code": "249366005",
          "display": "Bleeding from nose (finding)"
        }
      ]
    }
  ],
  "slot": [
    {
      "reference": "Slot/deb4c4b3-870b-4599-84df-5e54cef7afda"
    }
  ],
  "description": "Patient reported nosebleed without injury",
  "start": "2025-02-12T12:30:00+00:00",
  "end": "2025-02-12T12:40:00+00:00",
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

### What Can Be Updated

| Field | Updatable | Notes |
|---|---|---|
| `.status` | ✅ | e.g., change from `booked` to `arrived` |
| `.reasonCode` | ✅ | Add or change the clinical reason |
| `.slot` | ❌ | Use [Reschedule](./06-reschedule-appointment.md) instead |
| `.participant` | ❌ | Not supported in this pattern |

### Error Responses

| HTTP Status | Meaning |
|---|---|
| 400 | Bad request (invalid resource) |
| 404 | Appointment not found |
| 409 | Conflict (resource has been modified since your GET) |
| 422 | Unprocessable entity (validation failure) |
