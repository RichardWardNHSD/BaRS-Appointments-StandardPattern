# Patient-Facing BaRS Appointments — Leadership Summary

## Executive Summary

We have an opportunity to enable patients to book, view, and cancel their own GP appointments directly through the NHS App using the existing BaRS (Booking and Referral Standard) infrastructure. The technical changes required are modest — the core appointment management API already exists and works. What's missing is support for **patient authentication** (proving the patient is who they say they are) so the system can securely handle citizen-initiated requests rather than only organisation-to-organisation ones.

This is a small, well-scoped piece of work that unlocks a significant set of patient-facing capabilities across multiple NHS programmes.

---

## What Already Exists

**The Appointment Management API is built and in production.** BaRS already supports:

- Searching for available slots
- Booking appointments
- Viewing appointments
- Updating/cancelling appointments
- Rebooking/rescheduling appointments

These operations work today in a business-to-business (B2B) context — healthcare systems calling each other on behalf of clinicians and operators. The API surface, the FHIR data model, and the Receiver integrations are all proven and live.

---

## What Does Not Exist

**Support for patient authentication.**

The current system authenticates *applications* (system-to-system credentials). There is no mechanism for a patient to authenticate as themselves and act on their own record. Specifically:

- No support for NHS login (NHS identity) token flows through the BaRS proxy
- No token exchange capability (converting a patient's identity token into a Receiver-specific access token)
- No patient-scoped access controls (ensuring a patient can only see/modify their own appointments)

---

## What Is the Impact

The changes required are contained and well-understood:

| Change                   | Scope                                                                                                                                                                                             |
| --------------------------| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **API change**           | Small modification to the existing BaRS API to accept user-restricted (patient) authentication tokens alongside the existing application-restricted tokens                                        |
| **Proxy change**         | Add token exchange capability to the BaRS proxy — this reuses the same pattern developed by the GP Connect Patient Facing Services team                                                           |
| **BaRS standard uplift** | Update the BaRS specification to document patient-facing interaction patterns, scoping rules, and Receiver responsibilities                                                                       |
| **Medicus onboarding**   | Onboard Medicus as the first Receiver for patient-facing appointments — Medicus is already being onboarded for Application 5 (A2A), so this could be an extension to that existing assurance work |

---

## Who Should Do the Work

**Preferred:** The Direct Integration team — this is architecturally closest to their existing remit.

**Alternative:** Jamie's team - they would need to provide dev resource and manage the delivery.

**Supporting:**
- Richard W — technical design and architecture guidance
- Existing BaRS team — standard uplift and Receiver onboarding/assurance

---

## Additional Benefits

Supporting patient authentication through BaRS is not a one-use investment. It is a **platform enabler** that unlocks multiple future capabilities:

| Capability | Description |
|------------|-------------|
| **National Diagnostic Service** | Patient auth is a stated requirement for patient-facing diagnostic booking |
| **All Appointments in the App** | Viewing and managing all appointments (not just GP) requires patient-level API access |
| **Self-Referral via the App** | Patients referring themselves to services (e.g. physio, MSK, mental health) requires authenticated patient identity |
| **Screening** | Patient-initiated screening appointment booking |
| **Vaccination** | Patient-led vaccination booking |
| **Any patient-led booking or referral** | Once the auth pattern exists, any service that wants to offer patient-facing booking through the NHS App can reuse it |

In short: this is a small piece of foundational work that removes a blocker for a wide range of patient-facing digital services.

---
## Coordination Risk — Proxy Codebase

The BaRS proxy requires modification for both this work (adding token exchange / patient auth) **and** the Endpoint Catalogue (EPC) migration (replacing static `targets.json` routing with dynamic endpoint resolution). Both workstreams touch the same codebase and the same request-routing logic.

This needs to be actively coordinated to avoid:
- Merge conflicts and rework if both streams develop in parallel without alignment
- Regression risk if one set of changes destabilises the other
- Duplicated effort (e.g. both teams building new routing logic independently)

Ideally these are treated as a single proxy evolution with a shared backlog and technical lead, rather than two independent changes landing on the same component.

---

## Technical Detail

A work-in-progress technical paper covering the authentication pattern, proxy architecture, endpoint catalogue integration, and Receiver responsibilities is available here:

[Patient-Facing BaRS — Technical Paper (WIP)](https://github.com/RichardWardNHSD/BaRS-Appointments-StandardPattern/blob/main/08-patient-facing-nhs-identity.md)

---

## Key Point

The hard work (the API, the standard, the Receiver integrations) already exists. What's needed is a well-understood authentication layer that has already been proven in the GP Connect Patient Facing programme. This is not new invention — it is applying an existing pattern to an existing API.
