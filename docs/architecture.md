# HealthPass MVP Architecture

> Status: Final MVP architecture for a single-clinic local pilot in India. This is not a claim of regulatory compliance or readiness for broad production deployment.

## 1. Purpose and Scope

HealthPass is a patient-controlled, clinic-operated digital health-record platform. It lets a participating clinic create and manage patient history, while patients use a responsive web application to approve access, view history, and download reports.

The first pilot is intentionally narrow:

- One clinic, deployed on clinic-controlled infrastructure.
- Patient, clinic, and supervisor web applications.
- Phone-number and OTP authentication.
- QR-assisted patient identification and visit-level consent.
- Structured clinical records: diagnosis, prescription, observation, lab report, and visit summary.
- Patient-provided documents kept visibly separate from clinic-authored records.
- Aggregate, non-clinical supervisor metrics.

The MVP excludes ABHA/ABDM integration, multi-clinic exchange, emergency override, family accounts, offline access to history, and diagnostic decision support.

## 2. Architecture Style

HealthPass starts as a **modular monolith**: one deployable FastAPI application with clear internal module boundaries and one PostgreSQL database.

This is appropriate for a single clinic because consent, visit state, doctor assignment, record creation, and audit events need consistent transactions. Independent microservices would add network, deployment, and debugging complexity without solving a current pilot problem.

```text
Patient web app ───┐
Clinic web app ────┼── HTTPS ──> Nginx ──> FastAPI modular monolith
Supervisor web app ┘                              │
                                                   ├── PostgreSQL
                                                   ├── Private object storage
                                                   └── OTP provider
```

## 3. Deployment: Local Clinic Pilot

The first pilot is hosted on a clinic-controlled server. Because patients must view their records and approve visits from their own phones, the web applications require a secure public HTTPS entry point; only Nginx is exposed to the internet. PostgreSQL and object storage remain private to the clinic server network.

```text
Clinic network
│
├── Staff browsers ────────────────────────────────┐
├── Patient phones ─────────────────────────────────┼── HTTPS
└── Supervisor browser ─────────────────────────────┘
                                                     │
                                            Nginx reverse proxy
                                                     │
                              ┌──────────────────────┴─────────────────────┐
                              │ Docker Compose on one secured clinic server │
                              ├─────────────────────────────────────────────┤
                              │ FastAPI API                                  │
                              │ PostgreSQL                                   │
                              │ S3-compatible private object storage         │
                              │ Background worker / scheduled jobs           │
                              └─────────────────────────────────────────────┘
```

Use Docker Compose for the pilot. Store reports in local S3-compatible object storage so the application uses an object-storage interface from the beginning; AWS S3 can replace it later without changing record business logic.

The clinic server must have encrypted disks, restricted administrator access, operating-system patching, TLS certificates, regular encrypted backups stored separately from the server, and a tested restore process. One local server remains a single point of failure; the pilot must have a documented downtime and recovery procedure.

## 4. Applications

### Patient web application

React, TypeScript, Vite, and Tailwind CSS. It is mobile-first and responsive.

Responsibilities:

- Phone OTP login and account activation.
- Display a short-lived HealthPass QR token.
- Receive and approve or reject visit-consent requests.
- View clinic-authored and patient-provided records separately.
- Download authorized reports.
- Upload personal external documents.
- View visit and access history.

### Clinic web application

React, TypeScript, Vite, and Tailwind CSS.

Responsibilities:

- Create pending patient accounts.
- Search and verify patients.
- Scan a patient QR code.
- Open a visit in an awaiting-consent state.
- Assign a doctor to the visit.
- Allow authorized staff to upload lab reports.
- Allow doctors to read permitted history and create clinical records.
- Let receptionists and clinic administrators close visits.

### Supervisor web application

A separate, tightly restricted responsive web application.

Responsibilities:

- Show platform-wide and per-clinic aggregate counts.
- Show registered patients, unique scanned patients, successful scans, and completed visits.
- Show no diagnoses, reports, prescriptions, individual patient identity, or raw record-access history.

## 5. Backend Modules

```text
backend/app/
├── auth/             # OTP login, sessions, account activation
├── users/            # User accounts and platform roles
├── patients/         # Patient profiles and duplicate resolution
├── clinics/          # Clinic, staff membership, and staff roles
├── visits/           # Visit lifecycle and doctor assignments
├── consent/          # Visit-scoped consent requests and expiry
├── records/          # Structured clinical and patient-provided records
├── files/            # Private upload, download, and object storage
├── qr/               # Short-lived opaque QR tokens and scan events
├── authorization/    # Central permission policies
├── audit/            # Immutable security-relevant events
├── analytics/        # Aggregate metrics derived from events
├── notifications/    # OTP and consent notifications
├── integrations/     # Future ABDM/ABHA adapters; no MVP dependency
└── common/           # Configuration, database, errors, shared utilities
```

Routes must remain thin. Each module owns its business rules, database access, Pydantic request/response models, and tests. No module may bypass the central authorization and audit services for protected operations.

## 6. Core Data Model

All primary keys are UUIDs. Timestamps use UTC. Clinical records preserve their original author and source.

| Entity | Purpose | Key relationships |
|---|---|---|
| `User` | Login identity authenticated by phone OTP | May be a patient or clinic staff member |
| `Patient` | HealthPass patient profile | Has one or more records and visits |
| `Clinic` | Participating healthcare provider | Has staff and visits |
| `StaffMembership` | A user’s active role at a clinic | Belongs to clinic; roles: receptionist, doctor, administrator |
| `Visit` | Time-bounded clinic interaction | Links patient, clinic, consent state, assignment, and records |
| `VisitAssignment` | Assigned doctor for a visit | Only an active assigned doctor may read history or author clinical records |
| `ConsentRequest` | Patient approval for a specific visit | Active only from approval until visit closure or midnight local clinic time |
| `ClinicalRecord` | Clinic-authored structured entry | Type: diagnosis, prescription, observation, lab report, visit summary |
| `PatientDocument` | Patient-uploaded external document | Separate from clinic-authored records |
| `FileObject` | Private object-storage metadata | Referenced by report or document records |
| `QrToken` | Short-lived opaque patient token | Never contains health data or a patient ID in readable form |
| `AuditEvent` | Append-only security and privacy event | Created for sensitive actions |
| `MetricEvent` | Minimal operational event | Used only to calculate aggregate supervisor metrics |

### Important constraints

- One phone number maps to one patient in the MVP; shared-family numbers are out of scope.
- A clinic can create a pending patient account but cannot activate it.
- Only a receptionist may assign doctors in the MVP.
- A receptionist or clinic administrator may close a visit in the MVP.
- Only the original authoring doctor may amend their clinical record.
- An amendment creates a new revision; it does not overwrite the original content.
- Clinic administrators manage staff and patient administration but cannot read clinical content by default.
- A patient cannot edit clinic-authored records; they can submit a correction request in a future phase.

## 7. Visit, Consent, and QR Flow

```text
1. Patient opens HealthPass and displays a short-lived QR token.
2. Receptionist scans the QR in the clinic application.
3. API validates the opaque token and creates a scan event.
4. Receptionist opens a Visit in awaiting_patient_consent state.
5. Patient receives a consent request and approves or rejects it.
6. On approval, consent becomes active and receptionist assigns a doctor.
7. Assigned doctor reads the complete permitted history and creates records.
8. Receptionist or clinic administrator closes the visit.
9. Consent immediately expires; any unclosed visit also expires at local midnight.
```

If a patient has no phone or connectivity, staff may create an `awaiting_patient_consent` visit. Existing medical history remains inaccessible. The clinic may draft today’s records, but they remain unpublished until the patient approves consent. If consent is never granted, drafts must be discarded or retained only according to a documented clinic policy; this policy must be agreed before the pilot starts.

QR rules:

- Token is random, opaque, short-lived, and single-use.
- Token resolution returns only the minimum data needed to begin consent.
- QR scanning never grants record access.
- Repeated invalid scans are rate-limited and audited.

## 8. Authorization Model

Authentication establishes identity. RBAC establishes role. Visit consent and assignment establish record-level permission.

```text
Authenticated user
+ active clinic staff membership
+ allowed staff role
+ active visit for the requested patient
+ active patient consent for that visit
+ active doctor assignment (for clinical access)
= permit operation
```

| Actor | Permitted actions |
|---|---|
| Patient | View own records, upload personal documents, display QR, approve/reject consent, download authorized reports |
| Receptionist | Create pending accounts, scan QR, create visits, assign doctors, close visits, upload lab reports; cannot read history |
| Doctor | Read history only for assigned active visits; create and amend only their own clinical records |
| Clinic administrator | Manage staff memberships and patient administration, and close visits; no clinical-content access by default |
| Supervisor | Read aggregate operational metrics only |

Every protected backend endpoint performs object-level authorization. The frontend never decides whether a user may access a record.

## 9. Records and File Storage

Clinical records use structured fields, with optional private file attachments:

- Diagnosis: code/text, date, author, visit.
- Prescription: medicine details, dosage, instructions, author, visit.
- Observation: structured or free-text clinical observation, author, visit.
- Lab report: metadata plus uploaded report file.
- Visit summary: authored summary and follow-up information.

Files are uploaded using short-lived, server-authorized upload URLs. Object names are generated UUIDs, not patient names. Files are malware-scanned before being marked available. Downloads require a fresh authorization check and are served using short-lived signed URLs.

## 10. Audit and Analytics

`AuditEvent` is append-only and records security-relevant events, including:

- OTP login success and failure.
- Account activation.
- QR creation, resolution, and rejection.
- Consent request, approval, rejection, and expiry.
- Visit creation, assignment, and closure.
- Record list view, record view, record creation, amendment, upload, and download.
- Authorization denial.
- Staff role and membership changes.

Audit data is restricted to authorized privacy and system administrators. It is not a supervisor dashboard data source.

`MetricEvent` contains only the minimum information to derive aggregate counts. A scheduled job generates platform-wide and per-clinic daily summaries for the supervisor application.

## 11. Security Baseline

- HTTPS for all browser and API traffic.
- Phone OTPs are short-lived, rate-limited, hashed where stored, and never logged.
- Session tokens are short-lived; refresh tokens are rotated and revocable.
- Secrets are stored outside source control and rotated on compromise.
- Database, backups, and object storage are encrypted at rest.
- Application logs redact phone numbers, OTPs, diagnosis text, prescription details, and file URLs.
- Database access is private to application containers; no public database port.
- Least-privilege accounts for application, database, object storage, and administrators.
- Daily encrypted backups and regular restore tests.
- Dependency updates, input validation, file-type/size validation, malware scanning, rate limiting, and security-event monitoring.

Use synthetic data for development, automated tests, demonstrations, and screenshots. Real patient data may be used only in the secured clinic pilot after the clinic has approved consent wording, retention, access responsibilities, backup handling, and incident response.

## 12. Future Integration Boundary

The `integrations/` module isolates external healthcare exchange protocols. Later ABDM/ABHA work can add adapters for identity, consent, record discovery, and record sharing without allowing ABDM-specific types to spread through the core clinical modules.

The MVP must not claim ABDM compliance or use ABHA as an identifier until integration is implemented, tested, and reviewed.

## 13. Build Order

1. Foundation: Docker Compose, configuration, PostgreSQL, migrations, logging, health check, and synthetic seed data.
2. Identity and staff: phone OTP, sessions, patient activation, clinics, memberships, and roles.
3. Patient and visit workflow: patient profiles, pending accounts, QR tokens, scan events, consent requests, visits, and doctor assignment.
4. Authorization and audit: central policy checks and append-only audit events before exposing record APIs.
5. Clinical records and files: structured record types, revisions, uploads, malware scanning, and protected downloads.
6. Patient and clinic applications: role-specific screens for the complete visit workflow.
7. Supervisor analytics: aggregate metric pipeline and separate dashboard.
8. Pilot hardening: backups, restore rehearsal, threat review, clinic workflow walkthrough, and incident runbook.

## 14. Known Pilot Risks

- A single local server has limited availability and disaster recovery.
- Phone-only identity excludes patients without a unique phone number.
- Consent approval depends on the patient phone being available.
- Clinic workflow must be validated with real reception and doctor staff before relying on it.
- Real health data requires legal, privacy, and security review before a pilot.
- Centralized MVP storage differs from ABDM’s federated direction; future interoperability will require intentional integration work.
