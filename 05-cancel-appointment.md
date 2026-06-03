# Cancel Appointment

## Purpose

Cancel an existing booking by setting its status to `cancelled` (or `entered-in-error` if the booking was made in error). This is technically the same operation as [Update](./04-update-appointment.md) — it uses PUT — but the intent is specifically to end the booking.

## Endpoint

```
PUT /Appointment/{id}
```

## Prerequisites

**Read before write**: You must first perform a [GET /Appointment/{id}](./03-view-appointment.md) to retrieve the current resource.

## Request

### Step 1: GET the current resource

```
GET https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5
```

### Step 2: PUT with status set to "cancelled"

#### Method & URL

```
PUT https://int.api.service.nhs.uk/booking-and-referral/FHIR/R4/Appointment/aca94bdb-2e38-4399-9ece-2ba083ce65b5
```

#### Headers

| Header | Value |
|---|---|
| `Authorization` | `Bearer {access_token}` |
| `X-Request-Id` | `d4e5f6a7-b8c9-0123-defa-345678901234` |
| `X-Correlation-Id` | `0598efa7-fff0-4ade-9af8-3f46b4124151` |
| `NHSD-End-User-Organisation` | Base64-encoded JSON (ODS: `RYG`) |
| `NHSD-Target-Identifier` | Base64-encoded JSON (service: `2000072489`) |
| `Content-Type` | `application/fhir+json` |
| `Accept` | `application/fhir+json` |

#### Request Body – Standard Cancellation

Set `.status` to `cancelled`. You may also update `.reasonCode` to indicate why.

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
  "status": "cancelled",
  "reasonCode": [
    {
      "coding": [
        {
          "system": "http://snomed.info/sct",
          "code": "410543007",
          "display": "Not required (qualifier value)"
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

#### Request Body – Entered in Error

If the booking was made in error, set `.status` to `entered-in-error` instead:

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
  "status": "entered-in-error",
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
  -H 'X-Request-Id: d4e5f6a7-b8c9-0123-defa-345678901234' \
  -H 'X-Correlation-Id: 0598efa7-fff0-4ade-9af8-3f46b4124151' \
  -H 'NHSD-End-User-Organisation: eyJyZXNvdXJjZVR5cGUiOi...' \
  -H 'NHSD-Target-Identifier: eyJzeXN0ZW0iOiJodHRwczovL2ZoaXIubmhzLnVrL0lkL2Rvcy1zZXJ2aWNlLWlkIiwidmFsdWUiOiIyMDAwMDcyNDg5In0=' \
  -H 'Content-Type: application/fhir+json' \
  -H 'Accept: application/fhir+json' \
  -d @appointment-cancel.json
```

## Response

### HTTP 200 – Success

Returns the cancelled Appointment resource.

```json
{
  "resourceType": "Appointment",
  "id": "aca94bdb-2e38-4399-9ece-2ba083ce65b5",
  "meta": {
    "lastUpdated": "2025-02-10T11:00:00.000+00:00",
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
          "code": "410543007",
          "display": "Not required (qualifier value)"
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

## After Cancellation

Once processed, the Receiver must **DELETE** the pointer in the central Registry (DocumentReference). See the [DocumentReference Standard Pattern – Receiver](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Core/1-4-1/DocumentReference-StandardPattern/Receiver-DocumentReference?version=1.11.1) for details.

### Error Responses

| HTTP Status | Meaning |
|---|---|
| 400 | Bad request (invalid resource or booking doesn't exist) |
| 404 | Appointment not found |
| 409 | Conflict |

## Next Steps

If the patient needs a new appointment after cancellation, proceed to [Rebook Appointment](./07-rebook-appointment.md).
