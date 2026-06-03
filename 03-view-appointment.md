# View Appointment

## Purpose

Retrieve an existing Appointment resource from the receiving system. This is required:

- To view the current state of a booking
- **Before any modification** (update, cancel, reschedule) — the read-before-write rule

## Endpoint

```
GET /Appointment/{id}
```

## When to Use

- Checking the status of a previously made booking
- Mandatory first step before [Update](./04-update-appointment.md), [Cancel](./05-cancel-appointment.md), or [Reschedule](./06-reschedule-appointment.md)
- After a List (DocumentReference) lookup returns a pointer to an appointment

## Request

### Method & URL

```
GET https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5
```

### Path Parameters

| Parameter | Required | Description | Example |
|---|---|---|---|
| `id` | Yes | The Appointment resource ID (returned when booking was created) | `aca94bdb-2e38-4399-9ece-2ba083ce65b5` |

### Headers

| Header | Value |
|---|---|
| `Authorization` | `Bearer {access_token}` |
| `X-Request-Id` | `b2c3d4e5-f6a7-8901-bcde-f12345678901` |
| `X-Correlation-Id` | `0598efa7-fff0-4ade-9af8-3f46b4124151` |
| `NHSD-End-User-Organisation` | Base64-encoded JSON (ODS: `RYG`) |
| `NHSD-Target-Identifier` | Base64-encoded JSON (service: `2000072489`) |
| `Accept` | `application/fhir+json` |

### Example cURL

```bash
curl -X GET \
  'https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5' \
  -H 'Authorization: Bearer eyJhbGciOi...' \
  -H 'X-Request-Id: b2c3d4e5-f6a7-8901-bcde-f12345678901' \
  -H 'X-Correlation-Id: 0598efa7-fff0-4ade-9af8-3f46b4124151' \
  -H 'NHSD-End-User-Organisation: eyJyZXNvdXJjZVR5cGUiOi...' \
  -H 'NHSD-Target-Identifier: eyJzeXN0ZW0iOiJodHRwczovL2ZoaXIubmhzLnVrL0lkL2Rvcy1zZXJ2aWNlLWlkIiwidmFsdWUiOiIyMDAwMDcyNDg5In0=' \
  -H 'Accept: application/fhir+json'
```

## Response

### HTTP 200 – Success

Returns the full Appointment resource in its current state.

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

### Alternative: Search by Patient

If the Appointment `id` is not known, you can search by patient demographics:

```
GET /Appointment?patient:identifier=https://fhir.nhs.uk/Id/nhs-number|9876543210
```

This returns a Bundle of matching Appointments.

### Error Responses

| HTTP Status | Meaning |
|---|---|
| 401 | Unauthorised (invalid or missing token) |
| 404 | Appointment not found |
| 501 | Not implemented (receiver does not support this operation) |

## Next Steps

Once you have the current resource, you can proceed to:

- [Update Appointment](./04-update-appointment.md) – to change `.status` or `.reasonCode`
- [Cancel Appointment](./05-cancel-appointment.md) – to set status to `cancelled`
- [Reschedule Appointment](./06-reschedule-appointment.md) – to change the slot
