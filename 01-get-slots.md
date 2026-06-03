# Get Slots

## Purpose

Search for available booking slots from a specified receiving system. This is the first step in making a booking — you need to know what slots are available before you can book one.

## Endpoint

```
GET /Slot
```

## When to Use

- Before making an initial booking (see [Book Appointment](./02-book-appointment.md))
- Before rescheduling (see [Reschedule Appointment](./06-reschedule-appointment.md))
- Before rebooking (see [Rebook Appointment](./07-rebook-appointment.md))

## Request

### Method & URL

```
GET https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Slot?Schedule.actor:HealthcareService=2000072489&start=ge2025-02-12T00:00:00+00:00&end=le2025-02-12T23:59:59+00:00&status=free
```

### Query Parameters

| Parameter | Required | Description | Example |
|---|---|---|---|
| `Schedule.actor:HealthcareService` | Yes | The service identifier for the target healthcare service | `2000072489` |
| `start` | Yes | Start of the search window (must use `ge` prefix) | `ge2025-02-12T00:00:00+00:00` |
| `end` | Yes | End of the search window (must use `le` prefix) | `le2025-02-12T23:59:59+00:00` |
| `status` | No | Filter by slot status (`free`, `busy`, or both) | `free` |

### Headers

| Header | Value |
|---|---|
| `Authorization` | `Bearer {access_token}` |
| `X-Request-Id` | `74c2b045-9b7d-4b78-aeee-642f6332e3c9` |
| `X-Correlation-Id` | `0598efa7-fff0-4ade-9af8-3f46b4124151` |
| `NHSD-End-User-Organisation` | Base64-encoded JSON (see below) |
| `NHSD-Target-Identifier` | Base64-encoded JSON (see below) |
| `Accept` | `application/fhir+json` |

#### NHSD-End-User-Organisation (decoded)

```json
{
  "resourceType": "Organization",
  "identifier": [
    {
      "system": "https://fhir.nhs.uk/Id/ods-organization-code",
      "value": "RYG"
    }
  ],
  "name": "Sender Organization"
}
```

#### NHSD-Target-Identifier (decoded)

```json
{
  "system": "https://fhir.nhs.uk/Id/dos-service-id",
  "value": "2000072489"
}
```

### Example cURL

```bash
curl -X GET \
  'https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Slot?Schedule.actor%3AHealthcareService=2000072489&start=ge2025-02-12T00%3A00%3A00%2B00%3A00&end=le2025-02-12T23%3A59%3A59%2B00%3A00&status=free' \
  -H 'Authorization: Bearer eyJhbGciOi...' \
  -H 'X-Request-Id: 74c2b045-9b7d-4b78-aeee-642f6332e3c9' \
  -H 'X-Correlation-Id: 0598efa7-fff0-4ade-9af8-3f46b4124151' \
  -H 'NHSD-End-User-Organisation: eyJyZXNvdXJjZVR5cGUiOi...' \
  -H 'NHSD-Target-Identifier: eyJzeXN0ZW0iOiJodHRwczovL2ZoaXIubmhzLnVrL0lkL2Rvcy1zZXJ2aWNlLWlkIiwidmFsdWUiOiIyMDAwMDcyNDg5In0=' \
  -H 'Accept: application/fhir+json'
```

## Response

### HTTP 200 – Success

Returns a FHIR Bundle containing Slot resources (and included Schedule, HealthcareService, etc.).

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 2,
  "entry": [
    {
      "fullUrl": "https://receiver.nhs.uk/FHIR/R4/Slot/deb4c4b3-870b-4599-84df-5e54cef7afda",
      "resource": {
        "resourceType": "Slot",
        "id": "deb4c4b3-870b-4599-84df-5e54cef7afda",
        "status": "free",
        "start": "2025-02-12T12:30:00+00:00",
        "end": "2025-02-12T12:40:00+00:00",
        "schedule": {
          "reference": "Schedule/a]c1234-5678-90ab-cdef-1234567890ab"
        }
      }
    },
    {
      "fullUrl": "https://receiver.nhs.uk/FHIR/R4/Slot/265a53d7-1d21-4fc6-a5b7-761f650e75eb",
      "resource": {
        "resourceType": "Slot",
        "id": "265a53d7-1d21-4fc6-a5b7-761f650e75eb",
        "status": "free",
        "start": "2025-02-12T10:00:00+00:00",
        "end": "2025-02-12T10:10:00+00:00",
        "schedule": {
          "reference": "Schedule/ac1234-5678-90ab-cdef-1234567890ab"
        }
      }
    }
  ]
}
```

### Error Responses

| HTTP Status | Meaning |
|---|---|
| 400 | Bad request (malformed parameters) |
| 401 | Unauthorised (invalid or missing token) |
| 404 | Target service not found |
| 408 | Request timeout (receiver did not respond in time) |

## Next Steps

Once you have selected a slot (e.g., `deb4c4b3-870b-4599-84df-5e54cef7afda`), proceed to [Book Appointment](./02-book-appointment.md).
