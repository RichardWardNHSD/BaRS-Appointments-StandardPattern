# Review: BaRS Technical Deep Dive

**Document reviewed:** `BaRS+Technical+Deep+Dive.doc` (Confluence export, "BaRS Technical Deep Dive")
**Scope of the source doc:** Whether the current BaRS design can support patient-facing appointment access via the NHS App / Pharmacy First (PF), focusing on authentication, routing, API behaviour, scale, and security.

---

## 1. Overall Assessment

The document is thorough and well structured. It correctly separates *current implementation* from *intended future (PF) behaviour*, and consistently surfaces the same cross-cutting themes (authentication/delegation, `_include` sizing, routing/EPC availability, provider conformance, cross-team ownership). The risk and open-question registers are genuinely useful and mostly actionable.

The main weakness is in **Section 6 (Appointments API & Endpoints)**: it describes booking, updating and cancelling appointments as being done through `POST /$process-message`, and omits the direct RESTful Appointment write operations (`POST` / `PUT` / `PATCH` / `DELETE /Appointment`). This is inconsistent with the Standard Pattern defined in this very repository and, in my view, points the future design at the wrong primitive.

---

## 2. Primary Observation — Prefer direct Appointment operations over `$process-message`

**This confirms and expands the initial observation raised.** The point here is *not* that the document is inaccurate — it correctly describes what BaRS does today. The point is that the document does not separate **current** behaviour from the **proposed** target behaviour, and the two differ for appointment management.

### Current state (accurate as documented)
BaRS today genuinely manages appointments through the message workflow:

- Booking, cancellation, and appointment updates use `POST /$process-message` with a FHIR `MessageBundle` (Section 6.1, 6.2.3, 6.3.1, and the Appendix route table).
- The only direct Appointment operations exposed by the production spec are the two reads: `GET /Appointment` and `GET /Appointment/{id}`.

This is a correct description of the live service. No correction is needed to the "current" facts — only a clear label that this is the *current* model.

### Proposed state (target)
The target design is to manage appointments with **REST verbs on `/Appointment`**, matching the Standard Pattern already defined in this repository. These operations are **already defined in the BaRS OAS** (`bars api OAS.json` in `BaRS-proxy-poc` — `POST /Appointment`, and `PUT` / `PATCH` / `DELETE /Appointment/{id}`), so this is a change in how appointments are managed, not a specification that needs writing:

| Lifecycle action | Proposed operation | Source (Standard Pattern) |
|---|---|---|
| Book | `POST /Appointment` (201 Created, server-assigned id) | `02-book-appointment.md` |
| View | `GET /Appointment/{id}` | `03-view-appointment.md` |
| Update (status / reasonCode) | `PUT /Appointment/{id}` (read-before-write) | `04-update-appointment.md` |
| Cancel | `PUT /Appointment/{id}` with `status: cancelled` | `05-cancel-appointment.md` |
| Reschedule | `PATCH /Appointment/{id}` | `06-reschedule-appointment.md` |
| Rebook | cancel + new `POST /Appointment` | `07-rebook-appointment.md` |

So there are two models, and the deep dive should present them as a **current → proposed** transition rather than a single "current" fact:

| | Current (live) | Proposed (target) |
|---|---|---|
| Book / update / cancel / reschedule | `POST /$process-message` + `MessageBundle` | `POST` / `PUT` / `PATCH` / `DELETE /Appointment` |
| Read | `GET /Appointment`, `GET /Appointment/{id}` | unchanged |
| Style | Message-based (inherited from referral workflows) | RESTful resource operations |

### Why the proposed REST model is preferred (wherever practical)

1. **Idempotency and safe retries.** The doc itself flags (Sections 3.4, 3.7) that "blind retries must not duplicate booking." `PUT`/`PATCH`/`DELETE` on a known resource id are naturally idempotent; `POST /$process-message` is not. Moving to targeted resource operations directly mitigates one of the doc's own headline scalability/resilience risks.
2. **Clear semantics and status codes.** `POST /Appointment` → 201 with a resource id; `PUT` → 200; conflict → 409. `$process-message` returns a "provider-defined FHIR response" (Section 6.3.1), which is harder to contract, test, and monitor.
3. **Read-before-write fits REST, not messaging.** The Standard Pattern mandates a GET before any PUT/PATCH (README "Key Rules"). That optimistic-concurrency flow (409 on stale writes — see `04-update-appointment.md`) is a REST pattern, not a message pattern.
4. **Patient-authorisation checks are simpler.** A core PF requirement in the doc (Sections 2, 4) is binding the authorised user to the requested patient. Extracting the patient/appointment target from a typed REST request is far more tractable than parsing a `$process-message` `MessageBundle` — the doc even lists this as an open question (7.1: *"how should the NHS number be extracted from … `$process-message` bundles?"*). Moving to direct operations largely removes that question for the booking flow.
5. **`_include` alignment.** The whole `_include` uplift is a FHIR **search** concept. It only applies to the RESTful search/read model, reinforcing that appointment interactions should live in the REST model rather than the message model.

### Recommended changes to the document
- **Split current vs proposed throughout Section 6.** Keep the accurate "current" description (`$process-message` manages appointments today), then add a "proposed" subsection stating the target is REST verbs on `/Appointment` — `POST` (book), `PUT` (update / cancel), `PATCH` (reschedule), and `DELETE` where supported — with `$process-message` retained only for asynchronous / notification-style workflows where a direct operation is not practical.
- **Appendix route table:** Mark the existing `$process-message` row as **current**, and add **proposed** rows for `POST /Appointment`, `PUT /Appointment/{id}`, `PATCH /Appointment/{id}`, and `DELETE /Appointment/{id}` (each resolved via the same S3 `appointment` routing key).
- **Record the gap accurately (the OAS is not the gap):** The write verbs already exist in the BaRS OAS — `bars api OAS.json` in `BaRS-proxy-poc` defines `POST /Appointment` (`createAppointment`) and, on `/Appointment/{id}`, `PUT` (`updateAppointment`), `PATCH` (`patchAppointment`) and `DELETE` (`deleteAppointment`) alongside the two GETs. The gap is therefore **not** the specification. The gap is behavioural/adoption: appointment management is still being *driven* through `$process-message` in practice, and receivers/senders need to actually use the REST verbs the OAS already defines. The migration item is to route appointment lifecycle actions through the existing REST operations (and have receivers advertise/support them), not to add anything new to the OAS.
- **Resilience section (3):** Note that adopting idempotent Appointment operations is itself a mitigation for the "duplicate booking on retry" risk, and is a benefit of the proposed model over the current one.

> Caveat worth stating in the doc: the move is a "wherever practical" target. Some receivers may only implement the message workflow today, so `/metadata` (CapabilityStatement) is the correct mechanism to discover which interactions a given receiver supports before choosing the path. The transition should be driven by receivers advertising the REST Appointment interactions in their CapabilityStatement.

---

## 3. Other Observations

### 3.1 Unbounded appointment search (raised well, worth elevating)
The doc repeatedly and correctly flags that `GET /Appointment` has no pagination, date filter, status filter, or maximum result size, and that `_include` compounds this. This is arguably a higher near-term risk than the doc's ordering implies: combined with multi-provider fan-out it directly threatens the performance bands mentioned in Section 1.3. Recommend deciding on at least date/status filtering **before** enabling `_include`, and stating that decision rather than leaving it purely as an open question.

### 3.2 `_include` and authorisation boundary
Good catch in Section 4 that `_include` must not let a caller pull linked resources outside the authorisation scope of the original request. This should be promoted from "consideration" to a firm requirement with a test case, since it is a data-protection control, not just a design nicety.

### 3.3 NRLF token cache key
The constant cache key `nrlfAccessToken` is flagged in both Section 2 and Section 4. This is a concrete, fixable item (include client/audience in the key). Recommend tracking it as a defect rather than an open question — **though note this becomes moot if the NRLF / DocumentReference capability is removed (see 3.7).**

### 3.4 `debug=true` in `JC-AWSSignV4.xml`
Flagged under Security risks. This is a concrete potential PII/credential-exposure issue and should be verified and remediated as a discrete action, not bundled with general "confirm logging behaviour" wording.

### 3.5 EPC / routing flat file → service migration
Correctly scoped as another team's work with a communication/documentation dependency rather than a DI-owned change. The framing is right; the doc should name the owning team and a coordination point if known.

### 3.6 Terminology consistency
- Section 2 header renders as "A uthentication & Token Exchange" (stray space) — cosmetic, from the export.
- The doc uses "PF" (Pharmacy First / patient-facing) heavily; a one-line definition near the top would help readers who join mid-document.
- `patient:identifier` vs `NHSD-Target-Identifier` vs Appointment path id is clarified well in Section 6.9 Considerations — good.

### 3.7 Proposed removal of NRLF / DocumentReference (retire the capability)
**Direction of travel:** the intent is to **remove the NRLF aspects of BaRS (`/DocumentReference`)**, as this capability never gained traction. The deep dive currently treats NRLF/DocumentReference as an established part of the design — it appears throughout as the exception to the standard routing model:

- Section 2.1 / 2.4: the separate NRLF client-assertion JWT flow, token exchange, and the `nrlfAccessToken` cache.
- Section 4.1 / 4.2: NRLF as a distinct trust boundary with its own service token and KVM-stored private keys.
- Section 5.1: DocumentReference bypasses the S3 routing file and uses fixed NRLF consumer/producer routes.
- Section 6.x and the Appendix route table: `GET /DocumentReference` and `POST/PUT/DELETE /DocumentReference` mapped to fixed NRLF targets.

Because this is a **proposed removal**, the document should record it the same way as the Appointment change — as a **current → proposed** transition, not a silent deletion:

| | Current (live) | Proposed (target) |
|---|---|---|
| DocumentReference / NRLF | `GET` and `POST/PUT/DELETE /DocumentReference` routed to fixed NRLF consumer/producer targets, using a separate NRLF token-exchange flow | **Capability retired** — `/DocumentReference` removed from the BaRS surface; no NRLF token exchange, cache, or private keys |

**Rationale to capture in the doc:** the capability never gained adoption, so retiring it reduces the attack surface, removes a whole trust boundary, and eliminates several of the doc's own open risk items.

**Benefits / simplifications this unlocks (worth stating explicitly):**
- **Security:** removes the separate NRLF trust boundary (Section 4.2), the client-assertion JWT flow, and the NRLF private keys held in Apigee KVMs — shrinking the credential and key-rotation footprint.
- **Defects made moot:** the `nrlfAccessToken` cache-key isolation concern (3.3) and part of the token-forwarding complexity (Section 2.5) disappear entirely rather than needing a fix.
- **Routing:** DocumentReference is the one documented exception to the S3 routing model (Section 5.1); removing it makes the routing story uniform and simpler to reason about, and simplifies the future EPC migration (3.5).
- **Scope clarity:** removes NRLF-specific testing, logging-redaction, and ownership questions from the PF work.

**Actions to record in the document:**
- Add a short "Proposed: retire NRLF / DocumentReference" note to Sections 2, 4, 5 and 6 stating the capability is being removed and why.
- Mark the DocumentReference rows in the Appendix route table as **current, to be removed**.
- Confirm no live consumers depend on `/DocumentReference` before removal (adoption check), and define a deprecation/removal path (announce → deprecate in OAS → remove) rather than an abrupt cut.
- Note the OAS/BaRS Standard uplift required to withdraw the `/DocumentReference` endpoints and the associated NRLF security schemes.

> This pairs naturally with the Appointment change: both are moving BaRS toward a simpler, uniform RESTful surface — adding the Appointment write verbs while removing the NRLF/DocumentReference exception.

---

## 4. Strengths Worth Keeping

- Clean current-vs-future split in every section.
- The consolidated **Questions** (Section 7) and **Risks** (Section 8) registers are strong and traceable back to the body.
- Security trust-model description (Section 4.2) is clear: OAuth authenticates the client, mTLS authenticates BaRS to the provider, and neither by itself proves end-user patient authorisation — this correctly motivates the PF work.
- The scalability section is honest about what is and isn't defined in the repository (timeouts, retries, circuit breaking).

---

## 5. Suggested Priority Actions

| Priority | Action | Section |
|---|---|---|
| **High** | Split the Appointment API section into **current** (`$process-message` manages appointments today) and **proposed** (REST verbs on `/Appointment`). Note the write verbs already exist in the OAS — the change is behavioural/adoption, not a spec uplift | 6, Appendix |
| **High** | Decide response-size controls (date/status filter, max bundle) before enabling `_include` | 3, 6 |
| **High** | Make "`_include` stays within the original request's authorisation scope" a firm, tested requirement | 4 |
| **High** | Record the **proposed removal of NRLF / DocumentReference** as a current → proposed transition; run an adoption check and define a deprecation/removal path | 2, 4, 5, 6, Appendix |
| **Medium** | Treat `debug=true` as a concrete defect with an owner (NRLF cache-key concern falls away if `/DocumentReference` is removed) | 4 |
| **Medium** | Tie operation choice (REST vs message) to receiver CapabilityStatement (`/metadata`) | 6 |
| **Low** | Define "PF" early; fix cosmetic export artefacts | 1, 2 |

---

*Prepared from the exported document text and cross-referenced against the Standard Pattern guides in this repository (`README.md`, `02`–`07` appointment guides).*
