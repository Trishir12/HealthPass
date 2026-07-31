# HealthPass MVP Architecture

> Status: Provisional design based on current research. This document records confirmed product decisions and the logical architecture only. Deployment details, database schema, and API contracts are not final.

## MVP Direction

HealthPass will initially be piloted with a clinic in India. It will provide two responsive web applications: one for patients and another for clinic staff and doctors. HealthPass will centrally store the records created during the MVP, while keeping a future integration boundary for ABHA and ABDM.

## Confirmed Product Decisions

- HealthPass stores the patient records used by the MVP.
- Patients may self-register, while clinic staff may create pending accounts for patients who need assistance.
- A pending account becomes active only after the patient verifies their phone number using an OTP.
- Patient-provided records and clinic-authored records remain separate and visibly identified.
- Patients cannot directly edit clinic-authored medical records.
- Clinic administrators manage staff and patient administration but do not routinely access clinical content.
- Patients grant time-limited consent to the clinic for one complete visit.
- Only doctors assigned to that patient visit may access the patient's records.
- Consent ends when the visit is completed, with a maximum-expiry fallback still to be defined.
- The patient shares a temporary QR code through the patient web application.
- Scanning the QR identifies the patient, begins the visit workflow, and creates an access request.
- A patient approves the visit once; individual access actions during the visit remain audited.
- ABHA and ABDM integration is deferred until after the initial MVP, but the architecture will preserve an integration boundary.
- The supervisor sees aggregate operational numbers rather than medical information.

## Record Model

Patient history has two distinct sources:

```text
Patient medical history
├── Clinic-authored records
│   ├── Diagnoses
│   ├── Prescriptions
│   ├── Observations
│   ├── Lab reports
│   └── Visit summaries
│
└── Patient-provided records
    ├── Uploaded external reports
    └── Self-reported information
```

Structured clinical entries are preferred. File uploads remain available for reports and for workflows where structured entry is impractical. Every record must retain its author, source, clinic, patient, visit, and creation time.

## Registration Flow

```text
Clinic creates pending patient account
              or
Patient begins self-registration
               ↓
Patient verifies phone number using OTP
               ↓
Patient activates the account
               ↓
Patient can access the HealthPass web application
```

Clinic staff must not select the patient's credentials, receive the patient's OTP, or approve consent on the patient's behalf.

## Visit and Consent Flow

```text
Patient displays temporary QR
               ↓
Clinic scans and resolves the QR
               ↓
System begins an OPD visit and requests access
               ↓
Patient approves clinic access for the visit
               ↓
Clinic assigns a doctor
               ↓
Assigned doctor views history and creates records
               ↓
Visit is completed
               ↓
Consent expires and access ends
```

QR possession is not authorization. The QR initiates identification and consent; it does not directly reveal medical records.

## Logical Architecture

```text
Patient Web Application ───────┐
                               │
Clinic Web Application ────────┼──> HealthPass API
                               │          │
Supervisor Dashboard ──────────┘          ├── Identity and OTP
                                          ├── Patients
                                          ├── Clinics and staff
                                          ├── Visits and assignments
                                          ├── Medical records
                                          ├── Files and reports
                                          ├── Consent and authorization
                                          ├── QR identification
                                          ├── Audit
                                          └── Aggregate analytics
                                                    │
                                  ┌─────────────────┼───────────────┐
                                  │                 │               │
                             PostgreSQL       Private file     Future ABDM
                                                 storage          adapter
```

The API is currently expected to begin as a modular monolith. Module boundaries will keep identity, clinical data, authorization, audit, and analytics concerns separate without adding distributed-system complexity to the first clinic pilot.

## Authorization Rule

Access to patient records requires all of the following:

```text
Authenticated user
+ active clinic membership
+ permitted staff role
+ active patient visit
+ active clinic consent
+ doctor assigned to the visit
= authorized clinical access
```

The server must evaluate these conditions for every protected operation. Frontend visibility is not an authorization control.

## Supervisor Metrics

The initial aggregate dashboard is expected to include:

- Registered patients
- Unique scanned patients
- Successful scans
- Completed visits

Medical content must not be used in the supervisor dashboard. Whether metrics are shown per clinic as well as platform-wide remains undecided.

## Data Safety

Development and automated testing must use synthetic patient data. Real patient information may only be introduced into a secured pilot environment after the clinic workflow, consent language, access controls, retention rules, operational responsibilities, and applicable Indian legal and security requirements have been reviewed.

Production data must not be copied into local development or ordinary test environments.

## Decisions Still Open

- Exact first set of clinical record types
- Who assigns doctors and completes visits
- Maximum consent duration if a visit is not closed
- Who may upload lab reports
- Correction and amendment workflow
- Behavior when the patient has no connectivity or phone access
- Duplicate patient detection and resolution
- Support for shared family phone numbers
- Whether the supervisor is a separate application or protected portal section
- Per-clinic versus platform-wide aggregate metrics
- Detailed AWS deployment and operational model
- Database schema and API contracts
