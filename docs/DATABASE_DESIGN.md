```{=html}
<!-- Source: Make_My_Marriage_Database_Design_Specification_V1.docx. Converted from the latest agreed Word document. Content and decisions are retained; Markdown table layout may differ from Word. -->
```
** MAKE MY MARRIAGE **

**Database Design Specification**

Version 1.2 \| PostgreSQL \| 37 tables across 8 domains

  Property           Value
  ------------------ ----------------------------
  Document status    V1 implementation baseline
  ORM / migrations   SQLAlchemy / Alembic
  Tenant boundary    Wedding workspace
  Date               9 November 2026

## Purpose

This specification consolidates the approved product and system-design
decisions into a relational database baseline. It defines 37 proposed
tables, their key columns, PostgreSQL types, constraints, relationships,
indexes, tenant rules and lifecycle behaviour. It guides SQLAlchemy
models, Alembic migrations and service contracts.

This is a design specification, not an executable migration script.
Details that remain unresolved are explicitly listed in Section 13.

# Document control and contents

  -----------------------------------------------------------------------
  Version                 Date                    Description
  ----------------------- ----------------------- -----------------------
  1.0                     7 Nov 2026              Initial consolidated
                                                  specification covering
                                                  eight domains.

  -----------------------------------------------------------------------

-   1.  Design overview and conventions
-   2.  Domain ownership and relationship overview
-   3.  Identity schema
-   4.  Wedding schema
-   5.  Planning schema
-   6.  Guest schema
-   7.  Vendor & Expense schema
-   8.  Media schema
-   9.  Notification schema
-   10. Activity schema
-   11. Cross-domain integrity and tenant isolation
-   12. Indexing, deletion and operational guidance
-   13. Open decisions and implementation checklist

# 1. Design overview and conventions

-   PostgreSQL database with schemas: identity, wedding, planning,
    guest, vendor_expense, media, notification and activity.

-   UUID primary keys; snake_case table/column names; TIMESTAMPTZ for
    instants and DATE for calendar dates.

-   SQLAlchemy ORM and Alembic migrations. Use separate least-privilege
    runtime DB roles per service and a separate migration credential.

-   Use NUMERIC(12,2) for money with ISO currency_code. Use JSONB only
    for validated flexible structures.

-   Wedding is the tenant root. All wedding-owned records carry
    wedding_id and every request/query is tenant-scoped.

-   Within a schema, use composite foreign keys (entity_id,wedding_id)
    wherever practical to prevent cross-wedding references.

-   Cross-service references are logical references validated through
    service APIs/contracts, not unrestricted cross-schema foreign keys.

-   Never store plaintext passwords, raw reusable tokens, presigned URLs
    or cloud credentials.

-   Soft-delete where specified; preserve historical associations and
    moderation/audit history.

## 1.1 Domain inventory

  -----------------------------------------------------------------------
  Schema / domain   Tables            Service owner     Responsibility
  ----------------- ----------------- ----------------- -----------------
  identity          4                 Identity          Organiser
                                                        accounts and
                                                        authentication
                                                        lifecycle

  wedding           9                 Wedding           Tenant,
                                                        memberships,
                                                        invitations,
                                                        permissions,
                                                        settings, AI
                                                        setup and
                                                        finalisation

  planning          4                 Planning          Events,
                                                        schedules, tasks
                                                        and assignments

  guest             5                 Guest             Households, event
                                                        invitations,
                                                        guest tokens and
                                                        RSVP

  vendor_expense    5                 Vendor & Expense  Global vendors,
                                                        wedding
                                                        selections and
                                                        expense records

  media             6                 Media             Website, assets,
                                                        albums, photos,
                                                        guest tokens and
                                                        livestream

  notification      3                 Notification /    Email jobs,
                                      worker            attempts and
                                                        provider delivery
                                                        events

  activity          1                 Cross-cutting     Business activity
                                                        history
  -----------------------------------------------------------------------

## 1.2 Tenant and security rules

-   Backend derives authorised wedding context from authenticated user
    membership; never trust client-supplied wedding_id alone.

-   One organiser account can have only one active wedding membership in
    V1; a wedding can have multiple organisers.

-   Maximum four active Admins per wedding. Enforce transactionally in
    Wedding Service, not with a simple CHECK constraint.

-   Manager permissions are module-level: NO_ACCESS, VIEW or MANAGE.
    Admin has full access by definition.

-   Guest tokens are separate from organiser authentication, high
    entropy, hashed, scoped, expiring and revocable.

-   Public website access is limited to intentionally published content.
    Gallery view and upload tokens are distinct.

-   S3 object keys include wedding_id; issue URLs only after tenant and
    token/permission validation.

# 3. Identity schema

Owner: Identity Service. Guests do not have accounts in V1.

## 3.1 organiser_users

Canonical organiser account.

  Column              PostgreSQL type   Null?   Key / default / rule
  ------------------- ----------------- ------- -------------------------
  id                  UUID              No      PK
  full_name           VARCHAR(150)      No      
  email               VARCHAR(255)      No      Case-insensitive unique
  password_hash       TEXT              No      Password hash only
  is_email_verified   BOOLEAN           No      DEFAULT FALSE
  account_status      VARCHAR(20)       No      DEFAULT ACTIVE
  created_at          TIMESTAMPTZ       No      DEFAULT now()
  updated_at          TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   Unique index on lower(email), or CITEXT unique constraint.

Indexes:

-   UNIQUE(lower(email)).

Implementation notes:

-   Normalise email consistently across registration, login and
    invitations.

## 3.2 email_verification_tokens

One-time verification token lifecycle.

  --------------------------------------------------------------------------
  Column            PostgreSQL type   Null?             Key / default / rule
  ----------------- ----------------- ----------------- --------------------
  id                UUID              No                PK

  user_id           UUID              No                Logical FK →
                                                        organiser_users.id

  token_hash        TEXT              No                Unique hash of
                                                        high-entropy token

  expires_at        TIMESTAMPTZ       No                

  used_at /         TIMESTAMPTZ       Yes               Lifecycle marker;
  revoked_at                                            see table purpose

  created_at        TIMESTAMPTZ       No                DEFAULT now()
  --------------------------------------------------------------------------

Constraints / relationships:

-   user_id references identity.organiser_users.id logically/within
    schema.

-   Token hash must be unique; validate not expired and not already
    used/revoked.

Indexes:

-   UNIQUE(token_hash); INDEX(user_id,expires_at).

Implementation notes:

-   Never store raw token. Refresh tokens rotate on use and can be
    revoked.

## 3.3 password_reset_tokens

One-time password reset token lifecycle.

  --------------------------------------------------------------------------
  Column            PostgreSQL type   Null?             Key / default / rule
  ----------------- ----------------- ----------------- --------------------
  id                UUID              No                PK

  user_id           UUID              No                Logical FK →
                                                        organiser_users.id

  token_hash        TEXT              No                Unique hash of
                                                        high-entropy token

  expires_at        TIMESTAMPTZ       No                

  used_at /         TIMESTAMPTZ       Yes               Lifecycle marker;
  revoked_at                                            see table purpose

  created_at        TIMESTAMPTZ       No                DEFAULT now()
  --------------------------------------------------------------------------

Constraints / relationships:

-   user_id references identity.organiser_users.id logically/within
    schema.

-   Token hash must be unique; validate not expired and not already
    used/revoked.

Indexes:

-   UNIQUE(token_hash); INDEX(user_id,expires_at).

Implementation notes:

-   Never store raw token. Refresh tokens rotate on use and can be
    revoked.

## 3.4 refresh_sessions

Refresh-token/session lifecycle and revocation.

  --------------------------------------------------------------------------
  Column            PostgreSQL type   Null?             Key / default / rule
  ----------------- ----------------- ----------------- --------------------
  id                UUID              No                PK

  user_id           UUID              No                Logical FK →
                                                        organiser_users.id

  token_hash        TEXT              No                Unique hash of
                                                        high-entropy token

  expires_at        TIMESTAMPTZ       No                

  used_at /         TIMESTAMPTZ       Yes               Lifecycle marker;
  revoked_at                                            see table purpose

  created_at        TIMESTAMPTZ       No                DEFAULT now()
  --------------------------------------------------------------------------

Constraints / relationships:

-   user_id references identity.organiser_users.id logically/within
    schema.

-   Token hash must be unique; validate not expired and not already
    used/revoked.

Indexes:

-   UNIQUE(token_hash); INDEX(user_id,expires_at).

Implementation notes:

-   Never store raw token. Refresh tokens rotate on use and can be
    revoked.

# 4. Wedding schema

Owner: Wedding Service. Owns the tenant root, organiser membership,
invitations, permissions, settings and AI setup state.

-   One active wedding membership per organiser account in V1.

-   A maximum of four active Admin memberships per wedding, enforced
    transactionally.

-   AI setup tables remain in this schema; AI is not a separate
    microservice.

## 4.1 weddings

Root tenant and couple/workspace information.

  Column                   PostgreSQL type   Null?   Key / default / rule
  ------------------------ ----------------- ------- -------------------------------------
  id                       UUID              No      PK
  slug                     VARCHAR(150)      No      UNIQUE; public route /w/:slug
  bride_name               VARCHAR(150)      No      
  groom_name               VARCHAR(150)      No      
  wedding_date             DATE              Yes     Exact date only
  date_status              VARCHAR(20)       No      CONFIRMED / APPROXIMATE / UNDECIDED
  approximate_start_date   DATE              Yes     
  approximate_end_date     DATE              Yes     
  welcome_message          TEXT              Yes     
  description              TEXT              Yes     
  primary_location         JSONB             Yes     Validated location
  cover_image_key          TEXT              Yes     Media/S3 logical key
  status                   VARCHAR(20)       No      ACTIVE / ARCHIVED / DELETED
  archived_at              TIMESTAMPTZ       Yes     
  deleted_at               TIMESTAMPTZ       Yes     
  deleted_by_user_id       UUID              Yes     Logical Identity reference
  created_at               TIMESTAMPTZ       No      DEFAULT now()
  updated_at               TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   UNIQUE(slug).

-   CONFIRMED requires wedding_date; approximate dates NULL.

-   APPROXIMATE requires wedding_date NULL, both bounds present and
    start \<= end.

-   UNDECIDED requires all date fields NULL.

Indexes:

-   UNIQUE(slug); INDEX(status); INDEX(wedding_date).

Implementation notes:

-   Soft-delete; disable normal access and public website. Archival is
    separate. Do not duplicate slug in Media.

## 4.2 wedding_memberships

Organiser membership and role in a wedding.

  Column       PostgreSQL type   Null?   Key / default / rule
  ------------ ----------------- ------- ----------------------
  id           UUID              No      PK
  wedding_id   UUID              No      FK → weddings.id
  user_id      UUID              No      Logical Identity ref
  role         VARCHAR(20)       No      ADMIN / MANAGER
  joined_at    TIMESTAMPTZ       No      DEFAULT now()
  is_active    BOOLEAN           No      DEFAULT TRUE

Constraints / relationships:

-   Partial unique index on user_id WHERE is_active=true enforces one
    active wedding per organiser.

-   Maximum four active ADMINs per wedding requires transaction/locking
    in Wedding Service.

Indexes:

-   Partial UNIQUE(user_id) WHERE is_active=true;
    INDEX(wedding_id,is_active); INDEX(wedding_id,role).

Implementation notes:

-   Deactivation preserves membership history and removes workspace
    access.

## 4.3 organiser_invitations

Invitation for an organiser to join a wedding.

  -----------------------------------------------------------------------------
  Column                  PostgreSQL type   Null?             Key / default /
                                                              rule
  ----------------------- ----------------- ----------------- -----------------
  id                      UUID              No                PK

  wedding_id              UUID              No                FK → weddings.id

  invited_by_user_id      UUID              No                Logical Identity
                                                              ref

  email                   VARCHAR(255)      No                Normalised

  intended_role           VARCHAR(20)       No                ADMIN / MANAGER

  invitation_token_hash   TEXT              No                UNIQUE hash

  status                  VARCHAR(20)       No                PENDING /
                                                              ACCEPTED /
                                                              DECLINED /
                                                              REVOKED / EXPIRED

  expires_at              TIMESTAMPTZ       No                

  accepted_at             TIMESTAMPTZ       Yes               

  created_at              TIMESTAMPTZ       No                DEFAULT now()
  -----------------------------------------------------------------------------

Constraints / relationships:

-   Acceptance validates token, email, expiry, status, membership
    uniqueness and Admin cap.

-   Pending invitation is not membership.

Indexes:

-   UNIQUE(invitation_token_hash); INDEX(wedding_id,status);
    INDEX(lower(email),status).

Implementation notes:

-   Revoking an invitation cancels its queued email job.

## 4.4 module_permissions

Manager's module-level access.

  Column             PostgreSQL type   Null?   Key / default / rule
  ------------------ ----------------- ------- -----------------------------
  id                 UUID              No      PK
  membership_id      UUID              No      FK → wedding_memberships.id
  module_name        VARCHAR(60)       No      Controlled module key
  permission_level   VARCHAR(20)       No      NO_ACCESS / VIEW / MANAGE
  created_at         TIMESTAMPTZ       No      DEFAULT now()
  updated_at         TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   UNIQUE(membership_id,module_name). Admin permissions are
    implicit/full.

Indexes:

-   UNIQUE(membership_id,module_name).

## 4.5 wedding_settings

Workspace preferences.

  ------------------------------------------------------------------------------
  Column                  PostgreSQL type   Null?             Key / default /
                                                              rule
  ----------------------- ----------------- ----------------- ------------------
  id                      UUID              No                PK

  wedding_id              UUID              No                FK; UNIQUE

  timezone                VARCHAR(100)      No                Wedding IANA
                                                              timezone

  currency_code           CHAR(3)           No                DEFAULT 'INR'; ISO
                                                              4217 code

  date_format             VARCHAR(20)       No                DEFAULT
                                                              'DD_MMM_YYYY';
                                                              allowed formats
                                                              documented by API

  time_format             VARCHAR(5)        No                DEFAULT '12H'; 12H
                                                              / 24H

  rsvp_submissions_open   BOOLEAN           No                DEFAULT TRUE;
                                                              controls
                                                              new/updated RSVP
                                                              submissions

  default_language        VARCHAR(10)       No                DEFAULT en; V1
                                                              UI/system messages
                                                              are English-only

  gallery_visibility      VARCHAR(20)       No                DEFAULT LINK_ONLY;
                                                              V1 gallery access
                                                              is
                                                              token/link-only,
                                                              not publicly
                                                              listed

  created_at              TIMESTAMPTZ       No                DEFAULT now()

  updated_at              TIMESTAMPTZ       No                DEFAULT now()
  ------------------------------------------------------------------------------

Constraints / relationships:

-   UNIQUE(wedding_id).

Implementation notes:

-   Do not duplicate website publication state; canonical state is
    media.wedding_websites.

## 4.6 ai_setup_sessions

Resumable optional AI onboarding state.

  Column                  PostgreSQL type   Null?   Key / default / rule
  ----------------------- ----------------- ------- -------------------------------------------
  id                      UUID              No      PK
  organiser_user_id       UUID              No      Logical Identity owner
  wedding_id              UUID              Yes     Logical Wedding ref; NULL before creation
  status                  VARCHAR(30)       No      Session lifecycle
  current_stage           VARCHAR(60)       No      Current stage
  collected_information   JSONB             No      DEFAULT {}
  draft_plan              JSONB             No      DEFAULT {}
  prompt_version          VARCHAR(50)       No      
  message_count           INTEGER           No      DEFAULT 0
  generation_counters     JSONB             No      DEFAULT {}
  created_at              TIMESTAMPTZ       No      DEFAULT now()
  updated_at              TIMESTAMPTZ       No      DEFAULT now()
  completed_at            TIMESTAMPTZ       Yes     
  deleted_at              TIMESTAMPTZ       Yes     

Constraints / relationships:

-   Owner-only access plus membership/permission checks where
    applicable.

-   Atomically enforce usage counters and idempotent finalisation.

Indexes:

-   INDEX(organiser_user_id,status); INDEX(wedding_id);
    INDEX(updated_at).

Implementation notes:

-   Initial limits: 50 user messages/session; up to 3 generation
    attempts per section. Editing suggestions does not consume another
    generation.

## 4.7 ai_setup_messages

Chronological transcript and structured assistant payload.

  Column               PostgreSQL type   Null?   Key / default / rule
  -------------------- ----------------- ------- ---------------------------
  id                   UUID              No      PK
  session_id           UUID              No      FK → ai_setup_sessions.id
  sender_type          VARCHAR(20)       No      USER / ASSISTANT / SYSTEM
  content              TEXT              No      
  structured_payload   JSONB             Yes     Validated structure
  request_id           UUID              Yes     Idempotency/correlation
  created_at           TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   FK session_id → ai_setup_sessions.id.

-   Unique (session_id,request_id) when request_id is not NULL prevents
    duplicate responses.

Indexes:

-   INDEX(session_id,created_at); partial UNIQUE(session_id,request_id).

Implementation notes:

-   Never store secrets, tokens, presigned URLs, credentials or
    unrelated guest contact lists.

## 4.8 ai_finalisation_workflows

Durable, resumable coordination record for creating approved AI setup
sections through their owning services. It is separate from the setup
conversation/session record.

| Column | PostgreSQL type | Null? | Key / default / rule |
| --- | --- | --- | --- |
| id | UUID | No | PK |
| session_id | UUID | No | FK -> ai_setup_sessions.id; UNIQUE |
| wedding_id | UUID | No | FK -> weddings.id |
| status | VARCHAR(30) | No | PENDING / IN_PROGRESS / COMPLETED / FAILED_RETRYABLE |
| idempotency_key | VARCHAR(200) | No | UNIQUE finalisation request identity |
| started_at / completed_at | TIMESTAMPTZ | Yes | Lifecycle timestamps |
| last_error_code / last_error_message | VARCHAR / TEXT | Yes | Sanitised failure detail |
| created_at / updated_at | TIMESTAMPTZ | No | DEFAULT now() / updated on change |

Constraints and implementation notes:

- Wedding Service creates at most one workflow for a session and resumes that
  workflow for a matching idempotent finalise request.
- Execution order is `wedding`, `events`, `tasks`, `vendor_categories`,
  `website_content`, then `albums`. Skipped optional sections have no step.
- A workflow is `COMPLETED` only after every intended step completes. A
  recoverable dependency failure leaves it `FAILED_RETRYABLE` for explicit
  organiser retry/resume.

Indexes: UNIQUE(session_id); UNIQUE(idempotency_key); INDEX(wedding_id,status).

## 4.9 ai_finalisation_steps

Per-step durable state for an AI finalisation workflow. Wedding Service owns
these records but calls each domain service through its internal API; it never
writes to another service schema.

| Column | PostgreSQL type | Null? | Key / default / rule |
| --- | --- | --- | --- |
| id | UUID | No | PK |
| workflow_id | UUID | No | FK -> ai_finalisation_workflows.id |
| step_key | VARCHAR(60) | No | wedding / events / tasks / vendor_categories / website_content / albums |
| status | VARCHAR(30) | No | PENDING / IN_PROGRESS / COMPLETED / FAILED_RETRYABLE |
| attempt_count | INTEGER | No | DEFAULT 0; >= 0 |
| idempotency_key | VARCHAR(200) | No | Unique per workflow/step request |
| created_resource_reference | JSONB | Yes | Validated owning-service resource IDs/references |
| started_at / completed_at | TIMESTAMPTZ | Yes | Lifecycle timestamps |
| last_error_code / last_error_message | VARCHAR / TEXT | Yes | Sanitised failure detail |
| created_at / updated_at | TIMESTAMPTZ | No | DEFAULT now() / updated on change |

Constraints and implementation notes:

- UNIQUE(workflow_id,step_key). Completed steps are skipped on retry.
- Each owning-service call carries the step idempotency key. A matching retry
  returns its original result, preventing duplicate records after partial
  success or a coordinator crash.
- Record the created-resource reference only after the owning service confirms
  success; retryable errors are retained without erasing completed-step data.

Indexes: UNIQUE(workflow_id,step_key); INDEX(workflow_id,status).

# 5. Planning schema

Owner: Planning Service. Events are configurable; soft-deleted events
retain linked tasks and schedules.

## 5.1 events

Wedding event.

  Column                   PostgreSQL type   Null?   Key / default / rule
  ------------------------ ----------------- ------- ----------------------------
  id                       UUID              No      PK
  wedding_id               UUID              No      Tenant scope
  name                     VARCHAR(150)      No      
  description              TEXT              Yes     
  start_at                 TIMESTAMPTZ       Yes     
  end_at                   TIMESTAMPTZ       Yes     Must follow start
  venue_name               VARCHAR(200)      Yes     
  formatted_address        TEXT              Yes     
  city/state/postal_code   VARCHAR           Yes     Structured location fields
  country_code             VARCHAR(2)        Yes     ISO country
  latitude/longitude       NUMERIC(9,6)      Yes     
  google_place_id          VARCHAR(255)      Yes     
  dress_code               VARCHAR(150)      Yes     
  display_order            INTEGER           No      DEFAULT 0
  created_by_user_id       UUID              No      Logical Identity ref
  created_at               TIMESTAMPTZ       No      DEFAULT now()
  updated_at               TIMESTAMPTZ       No      DEFAULT now()
  deleted_at               TIMESTAMPTZ       Yes     Soft-delete

Constraints / relationships:

-   UNIQUE(id,wedding_id) for composite references.

-   CHECK(end_at IS NULL OR start_at IS NULL OR end_at \> start_at).

Indexes:

-   INDEX(wedding_id,start_at); INDEX(wedding_id,deleted_at).

Implementation notes:

-   Deleted events remain visible as deleted references on associated
    tasks/schedules.

## 5.2 tasks

Wedding-level or event-linked task.

  --------------------------------------------------------------------------
  Column               PostgreSQL type   Null?             Key / default /
                                                           rule
  -------------------- ----------------- ----------------- -----------------
  id                   UUID              No                PK

  wedding_id           UUID              No                Tenant scope

  event_id             UUID              Yes               Composite FK to
                                                           events

  title                VARCHAR(200)      No                

  description          TEXT              Yes               

  due_date             DATE              Yes               

  priority             VARCHAR(10)       No                LOW / MEDIUM /
                                                           HIGH

  status               VARCHAR(20)       No                PENDING /
                                                           IN_PROGRESS /
                                                           COMPLETED

  completed_at         TIMESTAMPTZ       Yes               Set on
                                                           completion; clear
                                                           on reopen

  created_by_user_id   UUID              No                Logical Identity
                                                           ref

  created_at           TIMESTAMPTZ       No                DEFAULT now()

  updated_at           TIMESTAMPTZ       No                DEFAULT now()

  deleted_at           TIMESTAMPTZ       Yes               
  --------------------------------------------------------------------------

Constraints / relationships:

-   Composite FK (event_id,wedding_id) → events(id,wedding_id).

-   Completed task requires completed_at; clear it when reopened.

Indexes:

-   INDEX(wedding_id,status); INDEX(wedding_id,due_date);
    INDEX(wedding_id,event_id).

Implementation notes:

-   Tasks may have no event. Preserve tasks when event is soft-deleted.

## 5.3 task_assignments

Many-to-many task/organiser assignment.

  Column                PostgreSQL type   Null?   Key / default / rule
  --------------------- ----------------- ------- -----------------------
  id                    UUID              No      PK
  wedding_id            UUID              No      Tenant scope
  task_id               UUID              No      Composite FK to tasks
  user_id               UUID              No      Logical Identity ref
  assigned_by_user_id   UUID              No      Logical Identity ref
  assigned_at           TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   UNIQUE(task_id,user_id).

-   Composite FK (task_id,wedding_id) → tasks(id,wedding_id).

-   Assignee must be active member of same wedding; validate via Wedding
    Service.

Indexes:

-   UNIQUE(task_id,user_id); INDEX(wedding_id,user_id).

## 5.4 event_schedules

Agenda items belonging to an event only.

  Column          PostgreSQL type   Null?   Key / default / rule
  --------------- ----------------- ------- ------------------------
  id              UUID              No      PK
  wedding_id      UUID              No      Tenant scope
  event_id        UUID              No      Composite FK to events
  title           VARCHAR(200)      No      
  description     TEXT              Yes     
  start_at        TIMESTAMPTZ       Yes     
  end_at          TIMESTAMPTZ       Yes     
  display_order   INTEGER           No      DEFAULT 0
  created_at      TIMESTAMPTZ       No      DEFAULT now()
  updated_at      TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   Composite FK (event_id,wedding_id) → events(id,wedding_id).

-   End must be after start when both supplied.

Indexes:

-   INDEX(event_id,start_at); INDEX(wedding_id,event_id).

Implementation notes:

-   No wedding-wide schedule in V1.

# 6. Guest schema

Owner: Guest Service. One guest record represents a household contact;
there are no individual family-member records or guest accounts.

## 6.1 guest_households

Primary contact representing one household.

  Column                     PostgreSQL type   Null?   Key / default / rule
  -------------------------- ----------------- ------- ----------------------
  id                         UUID              No      PK
  wedding_id                 UUID              No      Tenant scope
  contact_name               VARCHAR(150)      No      
  email                      VARCHAR(255)      Yes     
  phone_number               VARCHAR(40)       Yes     
  permitted_attendee_count   INTEGER           No      \> 0
  notes                      TEXT              Yes     
  created_by_user_id         UUID              No      Logical Identity ref
  created_at                 TIMESTAMPTZ       No      DEFAULT now()
  updated_at                 TIMESTAMPTZ       No      DEFAULT now()
  deleted_at                 TIMESTAMPTZ       Yes     Soft-delete

Constraints / relationships:

-   CHECK(permitted_attendee_count \> 0).

-   CHECK(NULLIF(BTRIM(email), '') IS NOT NULL OR
    NULLIF(BTRIM(phone_number), '') IS NOT NULL). This rejects missing,
    empty and whitespace-only contact values.

-   No unique email constraint.

Indexes:

-   INDEX(wedding_id,contact_name); INDEX(wedding_id,deleted_at).

Implementation notes:

-   Soft-delete to preserve invitation and RSVP history.

## 6.2 household_event_invitations

Events to which a household is invited.

  Column         PostgreSQL type   Null?   Key / default / rule
  -------------- ----------------- ------- ----------------------------
  id             UUID              No      PK
  wedding_id     UUID              No      Tenant scope
  household_id   UUID              No      Guest household reference
  event_id       UUID              No      Logical Planning reference
  created_at     TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   UNIQUE(household_id,event_id).

-   Validate same-wedding event through Planning Service; not a direct
    cross-service FK.

Indexes:

-   UNIQUE(household_id,event_id); INDEX(wedding_id,event_id).

Implementation notes:

-   Event invitation mapping does not create event-specific RSVP.

## 6.3 guest_invitation_tokens

Secure RSVP link token per household.

  -----------------------------------------------------------------------
  Column            PostgreSQL type   Null?             Key / default /
                                                        rule
  ----------------- ----------------- ----------------- -----------------
  id                UUID              No                PK

  wedding_id        UUID              No                Tenant scope

  household_id      UUID              No                Household
                                                        reference

  token_hash        TEXT              No                UNIQUE hash

  expires_at        TIMESTAMPTZ       No                Persisted calculated
                                                        expiry; use the API
                                                        rule at generation

  revoked_at        TIMESTAMPTZ       Yes               NULL while
                                                        active; manual
                                                        revocation
                                                        timestamp

  created_at        TIMESTAMPTZ       No                DEFAULT now()
  -----------------------------------------------------------------------

Constraints / relationships:

-   UNIQUE(token_hash).

-   Only one active token per household; enforce replacement
    transactionally/partial unique index with explicit active state.

-   Resend revokes old token and creates new one.

Indexes:

-   UNIQUE(token_hash); INDEX(household_id,expires_at); PARTIAL INDEX
    (household_id,expires_at) WHERE revoked_at IS NULL. The time comparison
    remains in the query because an index predicate cannot safely use `now()`.

Implementation notes:

-   Persist the API-calculated expiry at generation. Both invitation retrieval
    and RSVP submission reject tokens that are revoked or expired. GET does not
    consume a valid token, so email scanners may open links safely.

## 6.4 rsvp_reminder_requests

Durable Guest-Service record for an organiser's manually initiated RSVP
reminder request. It supports idempotency and a later processing summary; it
does not create automated reminder campaigns.

| Column | PostgreSQL type | Null? | Key / default / rule |
| --- | --- | --- | --- |
| id | UUID | No | PK; returned as reminder_request_id |
| wedding_id | UUID | No | Tenant scope |
| requested_by_user_id | UUID | No | Logical Identity reference |
| target_scope | VARCHAR(20) | No | SELECTED_PENDING / ALL_PENDING |
| requested_household_count | INTEGER | No | >= 0 |
| queued_household_count | INTEGER | No | >= 0 |
| idempotency_key | VARCHAR(200) | No | UNIQUE per organiser/operation |
| created_at / updated_at | TIMESTAMPTZ | No | DEFAULT now() / updated on change |

Constraints and implementation notes:

- Guest Service validates the selected or resolved all-pending households in one
  tenant-scoped transaction before creating the request.
- Notification jobs carry this request ID as an originating-request reference;
  Guest Service retrieves queued/sent/failed totals through Notification
  Service's authenticated internal API rather than its database schema.

Indexes: UNIQUE(idempotency_key); INDEX(wedding_id,created_at).

## 6.5 rsvp_responses

One overall RSVP response per household.

  -----------------------------------------------------------------------
  Column            PostgreSQL type   Null?             Key / default /
                                                        rule
  ----------------- ----------------- ----------------- -----------------
  id                UUID              No                PK

  wedding_id        UUID              No                Tenant scope

  household_id      UUID              No                Household FK

  response_status   VARCHAR(20)       No                PENDING /
                                                        ACCEPTED /
                                                        DECLINED

  attendee_count    INTEGER           Yes               Pending NULL;
                                                        accepted
                                                        1..permitted;
                                                        declined 0

  responded_at      TIMESTAMPTZ       Yes               NULL while
                                                        pending

  created_at        TIMESTAMPTZ       No                DEFAULT now()

  updated_at        TIMESTAMPTZ       No                DEFAULT now()
  -----------------------------------------------------------------------

Constraints / relationships:

-   UNIQUE(wedding_id,household_id).

-   PENDING: attendee_count and responded_at NULL.

-   ACCEPTED: attendee_count \>= 1 and \<= household permitted count
    (cross-table validation).

-   DECLINED: attendee_count = 0 and responded_at NOT NULL.

Indexes:

-   UNIQUE(wedding_id,household_id); INDEX(wedding_id,response_status).

Implementation notes:

-   Confirmed headcount = SUM(attendee_count) for ACCEPTED households.
    RSVP closure policy remains open.

# 7. Vendor & Expense schema

Owner: Vendor & Expense Service. Global directory data is shared;
wedding-specific selection/pricing/notes remain private to the wedding.

## 7.1 vendor_categories

Platform-maintained vendor categories.

  Column          PostgreSQL type   Null?   Key / default / rule
  --------------- ----------------- ------- ----------------------
  id              UUID              No      PK
  name            VARCHAR(100)      No      
  slug            VARCHAR(120)      No      UNIQUE
  description     TEXT              Yes     
  is_active       BOOLEAN           No      DEFAULT TRUE
  display_order   INTEGER           No      DEFAULT 0
  created_at      TIMESTAMPTZ       No      DEFAULT now()
  updated_at      TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   UNIQUE(slug).

Indexes:

-   UNIQUE(slug); INDEX(is_active,display_order).

## 7.2 vendors

Global vendor directory record.

  Column             PostgreSQL type   Null?   Key / default / rule
  ------------------ ----------------- ------- ---------------------------
  id                 UUID              No      PK
  category_id        UUID              No      FK → vendor_categories.id
  name               VARCHAR(200)      No      
  description        TEXT              Yes     
  email              VARCHAR(255)      Yes     
  phone_number       VARCHAR(40)       Yes     
  website_url        TEXT              Yes     
  service_location   JSONB             No      Validated location
  google_place_id    VARCHAR(255)      Yes     
  is_active          BOOLEAN           No      DEFAULT TRUE
  created_at         TIMESTAMPTZ       No      DEFAULT now()
  updated_at         TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   At least one of email, phone_number or website_url required (CHECK).

-   Optional partial unique google_place_id.

Indexes:

-   INDEX(category_id,is_active); optional partial
    UNIQUE(google_place_id).

Implementation notes:

-   Exactly one primary category. Deactivate rather than delete. Comply
    with Google Places retention, attribution and billing rules.

## 7.3 wedding_vendors

Wedding-specific selection of a global vendor.

  ---------------------------------------------------------------------------
  Column                PostgreSQL type   Null?             Key / default /
                                                            rule
  --------------------- ----------------- ----------------- -----------------
  id                    UUID              No                PK

  wedding_id            UUID              No                Tenant scope

  vendor_id             UUID              No                Global vendor FK

  status                VARCHAR(20)       No                SHORTLISTED /
                                                            CONTACTED /
                                                            CONFIRMED /
                                                            NOT_SELECTED

  quoted_amount         NUMERIC(12,2)     Yes               \>= 0

  agreed_amount         NUMERIC(12,2)     Yes               \>= 0

  currency_code         VARCHAR(3)        No                DEFAULT INR

  notes                 TEXT              Yes               Private to
                                                            wedding

  selected_by_user_id   UUID              No                Logical Identity
                                                            ref

  created_at            TIMESTAMPTZ       No                DEFAULT now()

  updated_at            TIMESTAMPTZ       No                DEFAULT now()

  deleted_at            TIMESTAMPTZ       Yes               Optional
                                                            soft-delete
  ---------------------------------------------------------------------------

Constraints / relationships:

-   UNIQUE(wedding_id,vendor_id).

-   Amounts must be non-negative when present.

Indexes:

-   UNIQUE(wedding_id,vendor_id); INDEX(wedding_id,status).

## 7.4 wedding_vendor_events

Vendor selection associated with multiple events.

  Column              PostgreSQL type   Null?   Key / default / rule
  ------------------- ----------------- ------- ---------------------------------
  id                  UUID              No      PK
  wedding_id          UUID              No      Tenant scope
  wedding_vendor_id   UUID              No      Composite FK to wedding_vendors
  event_id            UUID              No      Logical Planning reference
  created_at          TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   UNIQUE(wedding_vendor_id,event_id).

-   Composite FK (wedding_vendor_id,wedding_id); validate event tenant
    through Planning Service.

Indexes:

-   UNIQUE(wedding_vendor_id,event_id); INDEX(wedding_id,event_id).

## 7.5 expenses

Estimated/agreed/committed expense; not instalments or actual payment
ledger.

  Column               PostgreSQL type   Null?   Key / default / rule
  -------------------- ----------------- ------- ---------------------------------
  id                   UUID              No      PK
  wedding_id           UUID              No      Tenant scope
  title                VARCHAR(200)      No      
  amount               NUMERIC(12,2)     No      \> 0
  currency_code        VARCHAR(3)        No      DEFAULT INR
  expense_date         DATE              No      Required date
  description          TEXT              Yes     
  wedding_vendor_id    UUID              Yes     Composite FK to wedding_vendors
  event_id             UUID              Yes     Logical Planning ref
  created_by_user_id   UUID              No      Logical Identity ref
  created_at           TIMESTAMPTZ       No      DEFAULT now()
  updated_at           TIMESTAMPTZ       No      DEFAULT now()
  deleted_at           TIMESTAMPTZ       Yes     Soft-delete

Constraints / relationships:

-   CHECK(amount \> 0).

-   Composite FK for wedding_vendor_id + wedding_id. Event reference is
    validated through the Planning Service contract.

Indexes:

-   INDEX(wedding_id,expense_date); INDEX(wedding_id,event_id);
    INDEX(wedding_id,wedding_vendor_id).

Implementation notes:

-   Dashboard committed total uses expense records; do not double-count
    vendor agreed_amount.

# 8. Media schema

Owner: Media Service. S3 stores binaries; PostgreSQL stores
metadata/object keys. Bucket remains private.

## 8.1 wedding_websites

Public wedding website configuration.

  -----------------------------------------------------------------------------
  Column               PostgreSQL type   Null?             Key / default / rule
  -------------------- ----------------- ----------------- --------------------
  id                   UUID              No                PK

  wedding_id           UUID              No                UNIQUE logical
                                                           Wedding ref

  draft_theme          VARCHAR(30)       No                TRADITIONAL_INDIAN /
                                                           MINIMALIST /
                                                           ROYAL_LUXURY;
                                                           organiser-editable

  draft_content        JSONB             No                DEFAULT {};
                                                           validated structured
                                                           draft content

  draft_version        INTEGER           No                DEFAULT 1;
                                                           incremented when
                                                           draft changes

  published_snapshot   JSONB             Yes               Immutable snapshot
                                                           created on publish;
                                                           includes theme,
                                                           website content,
                                                           publishable event
                                                           data and referenced
                                                           public website media
                                                           asset IDs

  published_version    INTEGER           Yes               Draft version
                                                           captured by the
                                                           current published
                                                           snapshot

  is_published         BOOLEAN           No                DEFAULT FALSE; false
                                                           after unpublish

  published_at         TIMESTAMPTZ       Yes               Timestamp of latest
                                                           successful publish

  created_at           TIMESTAMPTZ       No                DEFAULT now()

  updated_at           TIMESTAMPTZ       No                DEFAULT now()
  -----------------------------------------------------------------------------

Constraints / relationships:

-   UNIQUE(wedding_id). Slug remains canonical in wedding.weddings.

Indexes:

-   UNIQUE(wedding_id); INDEX(is_published).

Implementation notes:

-   Store media asset IDs, not permanent S3 URLs; no arbitrary
    executable HTML.
-   Organiser edits affect only `draft_theme` and `draft_content`;
    public reads must use `published_snapshot` exclusively.
-   Publishing assembles and persists an immutable snapshot, including
    the current publishable event data obtained from Planning Service.
    Draft edits and later Planning changes do not mutate the existing
    snapshot; the organiser must publish again.
-   Unpublishing sets `is_published=false` but does not need to erase
    the last snapshot. Public website endpoints must refuse to serve it
    while unpublished.
-   Public website image delivery is recommended via CloudFront OAC and
    a dedicated public website asset prefix in the private S3 bucket;
    this is pending approval. Guest gallery assets remain private.

## 8.2 media_assets

Metadata for private S3 objects.

  ---------------------------------------------------------------------------
  Column                PostgreSQL type   Null?             Key / default /
                                                            rule
  --------------------- ----------------- ----------------- -----------------
  id                    UUID              No                PK

  wedding_id            UUID              No                Tenant scope

  asset_type            VARCHAR(30)       No                WEBSITE_IMAGE /
                                                            GALLERY_PHOTO

  object_key            TEXT              No                UNIQUE S3 object
                                                            key

  original_filename     VARCHAR(255)      Yes               Sanitised

  mime_type             VARCHAR(100)      No                Validated

  file_size_bytes       BIGINT            No                \> 0

  checksum              VARCHAR(128)      Yes               

  width                 INTEGER           Yes               

  height                INTEGER           Yes               

  uploaded_by_type      VARCHAR(20)       No                ORGANISER / GUEST
                                                            / SYSTEM

  uploaded_by_user_id   UUID              Yes               Logical Identity
                                                            ref

  guest_display_name    VARCHAR(150)      Yes               

  upload_status         VARCHAR(20)       No                PENDING_UPLOAD /
                                                            UPLOADED / FAILED
                                                            / DELETED

  created_at            TIMESTAMPTZ       No                DEFAULT now()

  updated_at            TIMESTAMPTZ       No                DEFAULT now()

  deleted_at            TIMESTAMPTZ       Yes               
  ---------------------------------------------------------------------------

Constraints / relationships:

-   UNIQUE(object_key); CHECK(file_size_bytes \> 0).

-   S3 key must include wedding_id and be server-generated.

Indexes:

-   UNIQUE(object_key); INDEX(wedding_id,asset_type);
    INDEX(wedding_id,upload_status).

Implementation notes:

-   Images 10 MB. JPEG/PNG/WebP. Validate MIME/signature, decode,
    dimensions/pixel count; re-encode/strip EXIF where appropriate.
    Never store presigned URLs.

## 8.3 albums

Wedding-wide or event-linked gallery album.

  Column          PostgreSQL type   Null?   Key / default / rule
  --------------- ----------------- ------- ----------------------
  id              UUID              No      PK
  wedding_id      UUID              No      Tenant scope
  event_id        UUID              Yes     Logical Planning ref
  name            VARCHAR(150)      No      
  description     TEXT              Yes     
  album_type      VARCHAR(20)       No      EVENT / GENERAL
  visibility      VARCHAR(20)       No      DEFAULT PRIVATE
  display_order   INTEGER           No      DEFAULT 0
  created_at      TIMESTAMPTZ       No      DEFAULT now()
  updated_at      TIMESTAMPTZ       No      DEFAULT now()
  deleted_at      TIMESTAMPTZ       Yes     

Constraints / relationships:

-   Validate event_id through Planning Service when present.

Indexes:

-   INDEX(wedding_id,deleted_at); INDEX(wedding_id,event_id).

Implementation notes:

-   Soft-delete album; preserve photos and moderation history.

## 8.4 album_photos

Album-to-asset mapping with moderation state.

  ----------------------------------------------------------------------------
  Column                PostgreSQL type   Null?             Key / default /
                                                            rule
  --------------------- ----------------- ----------------- ------------------
  id                    UUID              No                PK

  wedding_id            UUID              No                Tenant scope

  album_id              UUID              No                Composite FK to
                                                            albums

  media_asset_id        UUID              No                Composite FK to
                                                            media_assets

  moderation_status     VARCHAR(20)       No                PENDING_APPROVAL /
                                                            APPROVED /
                                                            REJECTED

  approved_by_user_id   UUID              Yes               Logical Identity
                                                            ref

  approved_at           TIMESTAMPTZ       Yes               

  display_order         INTEGER           No                DEFAULT 0

  created_at            TIMESTAMPTZ       No                DEFAULT now()

  updated_at            TIMESTAMPTZ       No                DEFAULT now()

  deleted_at            TIMESTAMPTZ       Yes               
  ----------------------------------------------------------------------------

Constraints / relationships:

-   UNIQUE(album_id,media_asset_id).

-   Composite same-wedding FKs to albums and media_assets.

Indexes:

-   UNIQUE(album_id,media_asset_id);
    INDEX(wedding_id,moderation_status).

Implementation notes:

-   Guest uploads default PENDING_APPROVAL; only APPROVED non-deleted
    photos appear in guest gallery. Organiser uploads may be
    auto-approved.

## 8.5 guest_media_tokens

Separate scoped guest upload and gallery-view tokens. These are
long-lived capability links with manual revocation/replacement only;
unlike organiser/guest invitation tokens, they do not automatically
expire.

  --------------------------------------------------------------------------
  Column               PostgreSQL type   Null?             Key / default /
                                                           rule
  -------------------- ----------------- ----------------- -----------------
  id                   UUID              No                PK

  wedding_id           UUID              No                Tenant scope

  album_id             UUID              Yes               Optional
                                                           same-wedding
                                                           album

  event_id             UUID              Yes               Optional logical
                                                           Planning ref

  token_hash           TEXT              No                UNIQUE hash; raw
                                                           token is never
                                                           stored

  token_type           VARCHAR(10)       No                UPLOAD / VIEW

  revoked_at           TIMESTAMPTZ       Yes               NULL while
                                                           active; manual
                                                           revocation
                                                           timestamp

  created_by_user_id   UUID              No                Logical Identity
                                                           ref

  created_at           TIMESTAMPTZ       No                DEFAULT now()
  --------------------------------------------------------------------------

Constraints / relationships:

-   UNIQUE(token_hash). Album must belong to wedding; validate event
    through Planning.
-   UPLOAD and VIEW are distinct permissions.
-   Exactly one active token per wedding and token_type, enforced by a
    partial unique index on
    `(wedding_id, token_type) WHERE revoked_at IS NULL`.
-   There is intentionally no `expires_at` column for these token types.
    Replacing a link revokes the previous token and creates a new one.

Indexes:

-   UNIQUE(token_hash); INDEX(wedding_id,token_type); PARTIAL UNIQUE
    INDEX(wedding_id,token_type) WHERE revoked_at IS NULL.

Implementation notes:

-   One wedding-wide upload link may allow optional event/album
    selection; it does not grant access to all albums. Token-only
    gallery view; no passcode.
-   Revocation immediately prevents new token-authorised operations. Any
    already-issued short-lived S3 presigned URL remains usable until its
    own expiry.

## 8.6 event_livestreams

YouTube livestream URL for an event.

  Column           PostgreSQL type   Null?   Key / default / rule
  ---------------- ----------------- ------- ----------------------
  id               UUID              No      PK
  wedding_id       UUID              No      Tenant scope
  event_id         UUID              No      Logical Planning ref
  provider         VARCHAR(20)       No      YOUTUBE
  livestream_url   TEXT              No      Validated URL
  is_active        BOOLEAN           No      DEFAULT TRUE
  created_at       TIMESTAMPTZ       No      DEFAULT now()
  updated_at       TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   Partial unique index on event_id WHERE is_active=true; preserve
    inactive history.

-   Validate event tenant and YouTube URL.

Indexes:

-   UNIQUE(event_id) WHERE is_active=true; INDEX(wedding_id,event_id).

Implementation notes:

-   No native streaming infrastructure.

# 9. Notification schema

Owner: Notification Service and background worker. Email templates live
in version-controlled code, not database.

## 9.1 notification_jobs

Email queue and lifecycle record.

  ---------------------------------------------------------------------------
  Column                PostgreSQL type   Null?             Key / default /
                                                            rule
  --------------------- ----------------- ----------------- -----------------
  id                    UUID              No                PK

  wedding_id            UUID              Yes               NULL for
                                                            account-level
                                                            email

  recipient_email       VARCHAR(255)      No                

  template_key          VARCHAR(100)      No                Code-owned
                                                            template
                                                            identifier

  template_version      VARCHAR(50)       No                

  payload               JSONB             No                DEFAULT {};
                                                            protect sensitive
                                                            link fields

  status                VARCHAR(20)       No                PENDING /
                                                            PROCESSING / SENT
                                                            / FAILED /
                                                            CANCELLED

  idempotency_key       VARCHAR(200)      No                UNIQUE

  originating_request_id UUID             Yes                Logical Guest
                                                            reminder-request
                                                            reference where
                                                            applicable

  attempt_count         INTEGER           No                DEFAULT 0

  max_attempts          INTEGER           No                DEFAULT 3

  scheduled_at          TIMESTAMPTZ       No                

  next_attempt_at       TIMESTAMPTZ       Yes               

  locked_at             TIMESTAMPTZ       Yes               Worker lease

  locked_by             VARCHAR(100)      Yes               

  cancel_requested_at   TIMESTAMPTZ       Yes                Set only when a
                                                            PROCESSING job is
                                                            asked to cancel

  provider_message_id   VARCHAR(255)      Yes               

  last_error_code       VARCHAR(100)      Yes               Sanitised

  last_error_message    TEXT              Yes               Sanitised

  sent_at               TIMESTAMPTZ       Yes               Provider
                                                            acceptance

  created_at            TIMESTAMPTZ       No                DEFAULT now()

  updated_at            TIMESTAMPTZ       No                DEFAULT now()
  ---------------------------------------------------------------------------

Constraints / relationships:

-   UNIQUE(idempotency_key); attempt_count \>= 0; max_attempts \> 0.

-   Claim rows transactionally using FOR UPDATE SKIP LOCKED or
    equivalent.

-   `PENDING` covers both not-yet-due scheduled jobs and retry-waiting
    jobs. A job is eligible when
    `COALESCE(next_attempt_at, scheduled_at) <= now()`.

-   On retryable failure with attempts remaining, set status back to
    `PENDING` and set `next_attempt_at`; do not introduce a `RETRYING`
    status. Once attempts are exhausted or failure is permanent, set
    `FAILED`.

-   `SCHEDULED` is not a status; future delivery time is represented by
    `scheduled_at` while status remains `PENDING`.

-   Cancelling a `PENDING` job changes it immediately to `CANCELLED`.
    Cancelling a `PROCESSING` job sets `cancel_requested_at`; the worker checks
    it before provider submission and suppresses later retries. If the provider
    already accepted the message, the job remains `SENT`. Terminal jobs cannot
    be cancelled again.

Indexes:

-   INDEX(status,scheduled_at); INDEX(status,next_attempt_at);
    INDEX(wedding_id,created_at).

Implementation notes:

-   Encrypt/restrict sensitive token-bearing payload values; never log
    raw tokens/URLs. Revoked invitation jobs are cancelled. SENT means
    provider accepted, not inbox delivery.

-   `originating_request_id` is a logical Guest-Service reference used only
    for manual RSVP reminder summaries; Notification Service exposes aggregate
    job results through its authenticated internal API rather than granting
    Guest Service database access.

## 9.2 notification_attempts

History of individual provider send attempts.

  Column                PostgreSQL type   Null?   Key / default / rule
  --------------------- ----------------- ------- ---------------------------
  id                    UUID              No      PK
  notification_job_id   UUID              No      FK → notification_jobs.id
  attempt_number        INTEGER           No      
  provider              VARCHAR(40)       No      
  status                VARCHAR(20)       No      
  provider_message_id   VARCHAR(255)      Yes     
  error_code            VARCHAR(100)      Yes     Sanitised
  error_message         TEXT              Yes     Sanitised
  started_at            TIMESTAMPTZ       No      
  completed_at          TIMESTAMPTZ       Yes     

Constraints / relationships:

-   UNIQUE(notification_job_id,attempt_number).

Indexes:

-   UNIQUE(notification_job_id,attempt_number);
    INDEX(notification_job_id,started_at).

Implementation notes:

-   Attempt status records the individual provider attempt (for example
    STARTED, SUCCEEDED, FAILED or ABANDONED); it does not replace the
    canonical parent job status.

## 9.3 email_delivery_events

Provider delivery/bounce/complaint event records.

  ---------------------------------------------------------------------------
  Column                PostgreSQL type   Null?             Key / default /
                                                            rule
  --------------------- ----------------- ----------------- -----------------
  id                    UUID              No                PK

  notification_job_id   UUID              Yes               Nullable if
                                                            unmapped

  provider              VARCHAR(40)       No                

  provider_event_id     VARCHAR(255)      Yes               Unique when
                                                            provided

  provider_message_id   VARCHAR(255)      Yes               

  event_type            VARCHAR(20)       No                DELIVERED /
                                                            BOUNCED /
                                                            COMPLAINED /
                                                            REJECTED /
                                                            DELAYED

  event_at              TIMESTAMPTZ       No                Provider event
                                                            time

  event_data            JSONB             No                Sanitised

  received_at           TIMESTAMPTZ       No                DEFAULT now()
  ---------------------------------------------------------------------------

Constraints / relationships:

-   Partial unique provider_event_id where non-NULL; verify webhook
    authenticity.

Indexes:

-   Partial UNIQUE(provider_event_id);
    INDEX(notification_job_id,event_at); INDEX(provider_message_id).

Implementation notes:

-   Notification history retention: 12 months.

## 9.4 Worker baseline

-   One worker initially; poll every 5 seconds; claim up to 5 jobs;
    process sequentially.

-   Up to 3 attempts with increasing retry delays; recover expired
    worker leases.

-   Exactly-once delivery cannot be guaranteed after provider acceptance
    followed by worker crash; use provider idempotency where supported.

-   Amazon SES production adapter; Resend may be used for development.

# 10. Activity schema

Cross-cutting capability, not a separate microservice. Business activity
is distinct from operational application logs.

## 10.1 activity_logs

Append-only concise summary of significant wedding business actions.

  Column               PostgreSQL type   Null?   Key / default / rule
  -------------------- ----------------- ------- ----------------------------
  id                   UUID              No      PK
  wedding_id           UUID              No      Tenant scope
  actor_type           VARCHAR(20)       No      ORGANISER / GUEST / SYSTEM
  actor_user_id        UUID              Yes     Logical Identity ref
  guest_household_id   UUID              Yes     Logical Guest ref
  action               VARCHAR(100)      No      Action key
  entity_type          VARCHAR(50)       No      Entity category
  entity_id            UUID              No      Entity identifier
  description          TEXT              No      Human-readable summary
  metadata             JSONB             No      DEFAULT {}; sanitised
  request_id           UUID              Yes     Correlation ID
  created_at           TIMESTAMPTZ       No      DEFAULT now()

Constraints / relationships:

-   Actor references are logical; actor consistency is validated by
    service.

-   Append-only in normal app operations; no normal update/delete
    endpoint.

Indexes:

-   INDEX(wedding_id,created_at DESC); optional indexes by action and
    entity.

Implementation notes:

-   Retention lifetime of wedding workspace. Visible to organisers with
    Activity permission. Guest actions are summary only (RSVP/photo
    upload). No exhaustive field-level diffs or secrets.

# 11. Cross-domain integrity and tenant isolation

## 11.1 Relationship strategy

  -----------------------------------------------------------------------------------------------------
  Source                                          Target                        Strategy
  ----------------------------------------------- ----------------------------- -----------------------
  wedding_memberships.user_id                     identity.organiser_users.id   Logical Identity
                                                                                reference

  planning.tasks.event_id                         planning.events.id            Composite FK including
                                                                                wedding_id

  planning.task_assignments.task_id               planning.tasks.id             Composite FK including
                                                                                wedding_id

  guest.household_event_invitations.event_id      planning.events.id            Validate through
                                                                                Planning API

  vendor_expense.wedding_vendor_events.event_id   planning.events.id            Validate through
                                                                                Planning API

  vendor_expense.expenses.event_id                planning.events.id            Logical Planning
                                                                                reference

  media.albums.event_id                           planning.events.id            Logical Planning
                                                                                reference

  media.event_livestreams.event_id                planning.events.id            Logical Planning
                                                                                reference

  activity.actor_user_id                          identity.organiser_users.id   Logical Identity
                                                                                reference

  activity.guest_household_id                     guest.guest_households.id     Logical Guest reference
  -----------------------------------------------------------------------------------------------------

## 11.2 Authorisation layers

-   Authentication identifies organiser; it does not grant access to a
    wedding.

-   Wedding membership and role establish workspace access.

-   Manager module permissions control read/write rights; Admin has full
    access.

-   Guest tokens are scoped and separate from organiser authentication.

-   Published website exposes only approved public content; guest
    gallery requires valid VIEW token; uploads require UPLOAD token.

## 11.3 Optional PostgreSQL Row-Level Security

RLS may be added as defence in depth, not as a replacement for
service-layer authorisation. If used, set tenant context
transaction-locally for every request and ensure pooled connections
cannot leak tenant context between requests.

# 12. Indexing, deletion and operational guidance

## 12.1 Indexing principles

-   Lead common workspace indexes with wedding_id.

-   Use partial unique indexes for active-only rules (active membership,
    active livestream, active tokens where an explicit active flag
    exists).

-   Use lower(email) unique index or CITEXT for case-insensitive email
    uniqueness.

-   Use keyset pagination for large activity, notification and photo
    feeds where useful.

-   Measure query plans before adding speculative indexes.

## 12.2 Deletion and retention matrix

  -----------------------------------------------------------------------
  Entity                  Lifecycle               Preservation / effect
  ----------------------- ----------------------- -----------------------
  Wedding                 Soft delete; archive    Disable normal access
                          separately              and public site;
                                                  controlled permanent
                                                  deletion

  Membership              Deactivate              Preserve history;
                                                  revoke access

  Events                  Soft delete             Keep tasks/schedules;
                                                  show deleted-event
                                                  state

  Guest households        Soft delete             Preserve
                                                  invitation/RSVP history

  Guest tokens            Revoke/expire           Resend creates
                                                  replacement token

  Global vendors          Deactivate              Preserve wedding
                                                  references

  Expenses                Soft delete             Preserve history

  Albums                  Soft delete             Preserve
                                                  photos/moderation

  Media assets            Soft delete/status      Coordinate S3 lifecycle
                          transition              and references

  Notification history    12 months               Scheduled cleanup

  Activity logs           Lifetime of wedding     Append-only

  AI drafts               User may delete         Approved wedding
                          unapproved session      records unaffected
  -----------------------------------------------------------------------

## 12.3 Transactional operations

-   Enforce Admin cap with transaction/locking around role assignment
    and invitation acceptance.

-   Enforce one active wedding membership with partial unique index.

-   Create wedding, initial Admin membership and default settings
    atomically where practical.

-   Invitation acceptance validates token/email/status/expiry,
    membership and Admin cap in one transaction.

-   RSVP update validates permitted attendee count and closure policy
    atomically.

-   Write activity record in same transaction as business change where
    possible; otherwise use reliable outbox.

-   Use idempotency for AI finalisation, email job creation and
    retry-sensitive operations.

## 12.4 Database operations

-   Separate runtime DB roles by service; separate migration/admin
    credential.

-   No PostgreSQL superuser in application containers.

-   Services own their schemas and do not directly query other service
    tables.

-   Version-control Alembic migrations and test
    migration/backfill/rollback strategy.

-   Development baseline: Supabase PostgreSQL; production choice (paid
    Supabase or RDS) remains open.

# 13. Open decisions and implementation checklist

  -----------------------------------------------------------------------
  Open item                           Decision required before final
                                      implementation
  ----------------------------------- -----------------------------------
  RSVP closure                        Choose wedding-wide boolean,
                                      deadline timestamp or both; define
                                      Admin override and guest behaviour.

  RSVP invitation token expiry        Persist the API-defined calculated
                                      expiry for each guest invitation;
                                      gallery-view and upload tokens have no
                                      automatic expiry.

  Active-token uniqueness             Use explicit active/revoked state
                                      or transactionally replace tokens;
                                      a time-relative partial index
                                      cannot reliably use expires_at \>
                                      now().

  Enum strategy                       Choose PostgreSQL ENUM versus
                                      VARCHAR + CHECK and define
                                      evolution policy.

  Cross-service integrity             Define API contracts and failure
                                      handling for event, media,
                                      organiser and household references.

  Permanent wedding deletion          Define retention, legal/operational
                                      hold and S3 object deletion
                                      workflow.

  Media variants                      Choose separate asset rows, variant
                                      metadata or derived object-key
                                      convention for thumbnails.

  Google Places                       Confirm current storage/retention
                                      and attribution obligations.

  Website SEO                         Define indexing, sitemap, robots
                                      and metadata policy.

  AI session retention                Define retention for completed
                                      transcripts and wedding-linked
                                      setup sessions.

  Currency                            Confirm INR-only or multi-currency
                                      V1; schema currently defaults to
                                      INR and stores currency_code.

  Activity metadata                   Define allowed keys per action and
                                      sanitisation rules.

  DB deployment                       Choose production provider, backups
                                      and point-in-time recovery plan.
  -----------------------------------------------------------------------

## 13.1 Implementation checklist

-   Confirm PostgreSQL provider/version, connection and backup strategy.

-   Create schemas and least-privilege service roles.

-   Implement SQLAlchemy models and Alembic migrations for all 34
    tables.

-   Add all primary/foreign/unique/check constraints and partial
    indexes.

-   Implement same-tenant composite FKs and cross-service validation
    contracts.

-   Implement tenant-scoped repository/query patterns and authorisation
    tests.

-   Test Admin cap, one-wedding membership, RSVP bounds, token
    replacement and soft deletion.

-   Implement worker lease recovery, retry and notification retention
    jobs.

-   Test S3 key scoping and guest upload/view token isolation.

-   Resolve open decisions before production release.

# Appendix A --- Derived dashboard rules

-   Confirmed headcount = SUM(attendee_count) for ACCEPTED RSVP
    responses.

-   Committed expense total = SUM(non-deleted expenses.amount), grouped
    by currency; do not add agreed vendor amount again.

-   Upcoming events are non-deleted events with start_at in the future.

-   Task progress denominator should explicitly exclude soft-deleted
    tasks.

-   Website is publicly available only when wedding is ACTIVE and
    wedding_websites.is_published = TRUE.

-   Guest gallery shows only non-deleted APPROVED photos within valid
    VIEW token scope.

# Appendix B --- Principal ERD cardinalities

-   Wedding 1---many memberships, organiser invitations, events,
    households, assets, albums, expenses and activity logs.

-   Organiser 1---many verification/reset/refresh records and AI setup
    sessions.

-   Membership 1---many module permissions; AI setup session 1---many
    messages.

-   Event 1---many tasks and schedules; task 1---many task assignments.

-   Household many---many events through household_event_invitations;
    household 1---0/1 RSVP response.

-   Vendor category 1---many vendors; wedding many---many vendors
    through wedding_vendors; vendor selection many---many events through
    wedding_vendor_events.

-   Wedding 1---1 website; album many---many assets through
    album_photos; event has at most one active livestream.

-   Notification job 1---many attempts and delivery events.

End of specification.
