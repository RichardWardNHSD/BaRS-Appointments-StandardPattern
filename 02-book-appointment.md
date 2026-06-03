# Book Appointment

## Purpose

Create an initial booking by POSTing an Appointment resource to the receiving system. This allocates the selected slot to the patient.

## Endpoint

```
POST /Appointment
```

## Prerequisites

1. [Get Slots](./01-get-slots.md) – You must have searched for and selected an available slot.
2. Confirm BaRS Capabilities – `GET /metadata` to verify the receiver supports booking.

## Request

### Method & URL

```
POST https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment
```

### Headers

| Header | Value |
|---|---|
| `Authorization` | `Bearer {access_token}` |
| `X-Request-Id` | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` |
| `X-Correlation-Id` | `0598efa7-fff0-4ade-9af8-3f46b4124151` |
| `NHSD-End-User-Organisation` | Base64-encoded JSON (ODS: `RYG`) |
| `NHSD-Target-Identifier` | Base64-encoded JSON (service: `2000072489`) |
| `Content-Type` | `application/fhir+json` |
| `Accept` | `application/fhir+json` |

### Request Body

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
curl -X POST \
  'https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment' \
  -H 'Authorization: Bearer eyJhbGciOi...' \
  -H 'X-Request-Id: a1b2c3d4-e5f6-7890-abcd-ef1234567890' \
  -H 'X-Correlation-Id: 0598efa7-fff0-4ade-9af8-3f46b4124151' \
  -H 'NHSD-End-User-Organisation: eyJyZXNvdXJjZVR5cGUiOi...' \
  -H 'NHSD-Target-Identifier: eyJzeXN0ZW0iOiJodHRwczovL2ZoaXIubmhzLnVrL0lkL2Rvcy1zZXJ2aWNlLWlkIiwidmFsdWUiOiIyMDAwMDcyNDg5In0=' \
  -H 'Content-Type: application/fhir+json' \
  -H 'Accept: application/fhir+json' \
  -d '{
    "resourceType": "Appointment",
    "meta": {
      "profile": ["https://fhir.hl7.org.uk/StructureDefinition/UKCore-Appointment"]
    },
    "status": "booked",
    "slot": [{"reference": "Slot/deb4c4b3-870b-4599-84df-5e54cef7afda"}],
    "description": "Patient reported nosebleed without injury",
    "start": "2025-02-12T12:30:00+00:00",
    "end": "2025-02-12T12:40:00+00:00",
    "created": "2025-02-10T09:15:00+00:00",
    "participant": [{"actor": {"reference": "Patient/788660eb-d2c9-4773-abd4-318484673fb2", "identifier": {"system": "https://fhir.nhs.uk/Id/nhs-number", "value": "9876543210"}, "display": "John Smith"}, "status": "accepted"}]
  }'
```

## Response

### HTTP 201 – Created

The Receiver returns the created Appointment with a server-assigned `id`. **Save this ID** for all future operations on this appointment.

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

### Error Responses

| HTTP Status | Meaning |
|---|---|
| 400 | Bad request (invalid resource) |
| 404 | Target service not found |
| 409 | Conflict (slot already booked, or booking already exists) |
| 422 | Unprocessable entity (validation failure) |

## After Booking

Once the Receiver processes the booking, they must create a pointer in the central Registry (DocumentReference). See the note in the [internal onboarding guide](../internal-onboarding-standard-pattern.md) regarding the NRL integration status.

## Next Steps

- To retrieve this appointment later: [View Appointment](./03-view-appointment.md)
- To modify it: [Update Appointment](./04-update-appointment.md)
- To cancel it: [Cancel Appointment](./05-cancel-appointment.md)
