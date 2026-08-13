# HealthPass MVP Database Schema

This is the starting schema for the single-clinic MVP. It focuses on the patient, clinic visit, consent, and record workflow. It is intentionally small and can change after the clinic pilot.

## General Rules

- Use UUIDs for primary keys.
- Store all timestamps in UTC.
- Keep status fields explicit instead of using many Boolean columns.
- Keep medical files outside PostgreSQL; store only metadata and object-storage references in the database.
- Preserve authored clinical records. Corrections create amendments instead of replacing the original record.

## Core Tables

### `users`

Stores the login identity for a patient or clinic staff member.

- `id`
- `phone_number` — unique
- `status` — `pending`, `active`, `suspended`
- `phone_verified_at`
- `created_at`, `updated_at`

### `patients`

Stores the patient profile linked to a user account.

- `id`
- `user_id` — unique foreign key to `users`
- `full_name`
- `date_of_birth`
- `gender`
- `created_at`, `updated_at`

A clinic can create a pending patient account, but only the patient activates it by verifying their phone number.

### `clinics`

Stores the participating clinic.

- `id`
- `name`
- `address`
- `status` — `active`, `suspended`
- `created_at`, `updated_at`

### `staff_memberships`

Connects a user to a clinic and gives them a clinic role.

- `id`
- `clinic_id` — foreign key to `clinics`
- `user_id` — foreign key to `users`
- `role` — `receptionist`, `doctor`, `administrator`
- `status` — `active`, `suspended`, `removed`
- `created_at`, `updated_at`

A user can belong to more than one clinic in the future. Doctor permissions must use this membership, not only the user ID.

## Visit and Consent Tables

### `visits`

Represents one patient interaction with a clinic.

- `id`
- `patient_id`
- `clinic_id`
- `status` — `awaiting_patient_consent`, `active`, `closed`, `expired`
- `opened_by_membership_id`
- `opened_at`, `closed_at`

### `visit_assignments`

Assigns a doctor to an active visit.

- `id`
- `visit_id`
- `doctor_membership_id`
- `assigned_by_membership_id`
- `status` — `active`, `removed`
- `assigned_at`

Only a receptionist assigns doctors in the MVP. Only an assigned doctor may read history or create clinic records for that visit.

### `consent_requests`

Records the patient decision to share records for one visit.

- `id`
- `patient_id`
- `clinic_id`
- `visit_id`
- `status` — `pending`, `approved`, `rejected`, `expired`
- `requested_at`, `approved_at`, `expires_at`

Consent is valid until the visit is closed or until midnight in the clinic’s local time, whichever happens first.

## Medical Record Tables

### `clinical_records`

Stores clinic-authored structured records. One table is used for the MVP so every record has the same authorship, visit, amendment, and audit behavior.

- `id`
- `patient_id`
- `clinic_id`
- `visit_id`
- `author_membership_id`
- `record_type` — `diagnosis`, `prescription`, `observation`, `lab_report`, `visit_summary`
- `content` — validated JSON for fields specific to the record type
- `file_object_id` — optional foreign key to an attached report file
- `status` — `draft`, `published`, `amended`
- `amends_record_id` — optional self-reference to the original record
- `created_at`, `published_at`

Only the original authoring doctor can create an amendment. The original record remains available as part of the history.

### `patient_documents`

Stores patient-provided external documents separately from clinic records.

- `id`
- `patient_id`
- `file_object_id`
- `title`
- `description`
- `created_at`

Patient-provided documents must always be visually identified as patient-provided.

### `file_objects`

Stores metadata for files held in private object storage.

- `id`
- `storage_key` — unique opaque object-storage key
- `original_filename`
- `content_type`
- `size_bytes`
- `scan_status` — `pending`, `clean`, `rejected`
- `uploaded_by_user_id`
- `created_at`

Files become available only after a malware scan marks them `clean`.

## QR, Audit, and Analytics

### `qr_tokens`

Stores short-lived, opaque QR tokens.

- `id`
- `patient_id`
- `token_hash` — unique; do not store a readable token
- `expires_at`
- `used_at`
- `created_at`

The QR token identifies a consent request; it never grants record access or contains health information.

### `audit_events`

Append-only log of privacy and security events.

- `id`
- `actor_user_id`
- `action`
- `target_type`, `target_id`
- `outcome`
- `clinic_id`
- `created_at`

Examples include QR scans, consent decisions, record views, downloads, changes, and authorization denials.

### `metric_events`

Minimal event data for aggregate supervisor metrics.

- `id`
- `event_type`
- `clinic_id`
- `occurred_at`

This table must not contain medical content. It supports registered-patient, unique-scan, successful-scan, and completed-visit counts.

## Main Relationships

```text
User → Patient
User → StaffMembership → Clinic

Patient → Visit → ConsentRequest
Visit → VisitAssignment → Doctor StaffMembership
Visit → ClinicalRecord

Patient → PatientDocument → FileObject
ClinicalRecord → FileObject
```

## Important Indexes and Constraints

- Unique index on `users.phone_number`.
- Unique index on `patients.user_id`.
- Index visits by `patient_id`, `clinic_id`, and `status`.
- Index consent requests by `visit_id` and `status`.
- Index clinical records by `patient_id`, `visit_id`, and `created_at`.
- Only one active assignment per doctor per visit.
- Only one active consent request per visit.
- Database constraints and backend checks must prevent staff from accessing a patient record outside an active, consented, assigned visit.
