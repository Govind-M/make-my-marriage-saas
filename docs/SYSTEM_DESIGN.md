```{=html}
<!-- Source: Make_My_Marriage_System_Design_V1_AI_Updated.docx. Converted from the latest agreed Word document. Content and decisions are retained; Markdown table layout may differ from Word. -->
```
**MAKE MY MARRIAGE**

**System Design Architecture**

Version 1.2 \| 9 November 2026

*A cost-conscious, domain-based multi-tenant wedding planning platform*

  ---------------------------------------------------------------------
  Document information               Value
  ---------------------------------- ----------------------------------
  Product                            Make My Marriage

  Document                           System Design Architecture --- V1

  Status                             Consolidated from product and
                                     architecture decisions discussed
                                     and approved

  Market                             Indian weddings across India

  Initial scale                      Development and portfolio
                                     demonstration

  Design principle                   Lean initial infrastructure with
                                     clear domain boundaries and
                                     migration paths
  ---------------------------------------------------------------------

Decision-status note: Confirmed decisions are recorded as such. Details
previously discussed as recommendations but not explicitly finalised are
identified as proposed defaults or open decisions.

# Contents

-   1.  Executive summary
-   2.  Product scope and functional requirements
-   3.  Architecture principles and constraints
-   4.  High-level architecture
-   5.  Application surfaces and routing
-   6.  Domain services (including AI Planning Module)
-   7.  Multi-tenancy and workspace isolation
-   8.  Data architecture
-   9.  Authentication and authorisation
-   10. Guest token access
-   11. Media, uploads and image delivery
-   12. Email and background processing
-   13. Vendor discovery and location model
-   14. Security architecture
-   14.4 AI security and privacy

## 14.4 AI security and privacy

-   Send only the minimum information needed for the current generation
    task. Do not send passwords, JWTs, refresh tokens,
    verification/reset tokens, AWS credentials, S3 presigned URLs, guest
    contact lists or unrelated private data to the LLM.

-   LLM API keys are backend-only environment configuration during
    development. Never expose them to React, commit them to source
    control or include them in logs.

-   Use synthetic wedding data with free cloud LLM tiers during
    development. Before real-user launch, review provider retention,
    training, human-review and processing terms, then choose a suitable
    provider plan and implement appropriate notice/consent.

-   Treat model output as untrusted. Validate typed output with Pydantic
    and business rules before presenting it as a draft or passing
    approved data to domain services.

-   The LLM has no direct database credentials and no unrestricted
    tools. It cannot execute arbitrary SQL, change roles, send
    invitations, publish websites, delete records or bypass domain APIs.

-   The backend owns user identity, wedding context, tenant scope and
    authorisation. Never rely on the model to determine permissions.

-   Setup sessions are accessible only to their authenticated organiser
    owner; after wedding creation, enforce wedding membership and
    applicable module permissions for any wedding-scoped operation.

-   Persist prompt_version per setup session. Track provider/model and
    request metadata without logging secrets or unnecessary sensitive
    content.

-   On provider failure, preserve the incoming user message and last
    successful draft. Retry temporary failures up to two times with
    short delays; allow later retry/resume. Use request idempotency to
    avoid duplicate assistant messages or duplicate final creation.

-   When source answers change, mark dependent sections as requiring
    review. Never silently overwrite or delete approved content.

-   15. Observability

-   16. Deployment and infrastructure

-   17. CI/CD and infrastructure as code

-   18. Reliability and evolution

-   19. V1 exclusions

-   20. Open decisions

-   Appendix A. Suggested repository structure

-   Appendix B. Key workflows

# 1. Executive Summary

Make My Marriage is a collaborative wedding planning and management
platform for couples and families across India. Each wedding is an
independent workspace containing organisers, events, tasks, guests,
vendors, expenses, website content, media and activity history.

The platform is multi-tenant: the wedding is the tenant boundary.
Multiple weddings coexist, while V1 restricts each organiser account to
membership in one wedding only. A wedding may have multiple organisers
with Admin or Manager roles. V1 also includes an optional AI-assisted
wedding setup flow; it generates editable drafts, saves/resumes
conversations and requires organiser approval before existing domain
services create records.

The selected stack is React SPA, Python FastAPI, SQLAlchemy and Alembic.
Backend responsibilities are grouped into domain-based services.
Supabase PostgreSQL is selected for initial development, using one
database with separate schemas and service-specific PostgreSQL roles.
Media is stored in private Amazon S3. Amazon SES is the intended
production email provider. The AI Planning Module is initially part of
the Wedding Service (not a separate microservice), uses a provider
adapter for external LLM APIs, and persists setup sessions and drafts in
the wedding schema.

The initial deployment direction is one AWS EC2 instance running
separate service containers with Docker Compose and Nginx. This is a
lean development/portfolio deployment, not a highly available production
topology.

## 1.1 Architecture decisions

  ---------------------------------------------------------------------
  Area                               Decision
  ---------------------------------- ----------------------------------
  Frontend                           ReactJS SPA

  Backend                            Python FastAPI

  Architecture                       Domain-based microservices

  Initial compute                    One AWS EC2 instance; start with
                                     an account-eligible free-tier
                                     micro instance, then monitor and
                                     resize only if needed.

  Containers                         Docker + Docker Compose

  Reverse proxy                      Nginx

  Initial database                   Supabase PostgreSQL Free for
                                     development

  Database layout                    One database; separate schemas and
                                     service-specific PostgreSQL roles.
                                     AI setup tables are owned by
                                     Wedding Service in the wedding
                                     schema.

  ORM/migrations                     SQLAlchemy / Alembic

  Media                              Private Amazon S3; presigned
                                     access

  CDN                                Recommended CloudFront + OAC for
                                     stable public wedding-site images
                                     from private S3; pending approval.
                                     Private gallery images remain
                                     token-authorised and separate.

  Email                              Amazon SES behind provider
                                     abstraction

  Background jobs                    PostgreSQL-backed job table +
                                     separate worker

  Vendor discovery                   Google Places API, subject to its
                                     terms

  Tenancy                            Wedding-level tenant boundary
                                     using wedding_id

  Observability                      Structured JSON logs, Docker
                                     rotation and health endpoints

  AI capability                      Optional AI-assisted wedding setup
                                     in V1; AI Planning Module within
                                     Wedding Service.

  LLM integration                    Provider-independent backend
                                     adapter; Gemini free tier with
                                     synthetic data during development;
                                     no RAG/vector database.
  ---------------------------------------------------------------------

Important: Separate services share one EC2 host and one Supabase
database in the initial design. They have application/schema ownership
boundaries, not independent infrastructure failure domains.

# 2. Product Scope and Functional Requirements

## 2.1 Vision and principles

Support diverse Indian regional, cultural, religious and community
wedding traditions. Ceremony names, event types, schedules and website
content are configurable; the platform must not hardcode one wedding
tradition.

## 2.2 Users and organiser accounts

-   Organisers: bride, groom, family members and invited wedding
    planners. Guests do not register or log in in V1.

-   Platform administrators maintain the platform-wide vendor directory.

-   Organiser registration, mandatory email verification, login/logout,
    password recovery/reset/change and profile management.

-   Wedding creator becomes Admin. Creator may invite the other partner
    as a second Admin and family/planners as Managers.

-   Invitees register or log in, then accept/decline. Admin can
    resend/revoke invitations, remove organisers and manage permissions.

-   Exactly two roles: Admin and Manager. Manager permissions are per
    module: No access, View or Manage.

-   One organiser account participates in one wedding only in V1;
    multiple organisers may participate in the same wedding.

## 2.3 Wedding setup and dashboard

-   Wedding setup: couple names, wedding date, photos, cover image,
    welcome message/description and primary location.

-   Dashboard: countdown, upcoming events, task progress,
    invitation/RSVP statistics, confirmed headcount, committed expense
    total, vendor overview and recent activity.

## 2.4 Events and tasks

-   Create/edit/delete configurable events with name, description, date,
    start/end time, venue/location and dress code.

-   Events may associate tasks, guests, vendors, expenses, albums and
    livestream URL; list chronologically.

-   Tasks include title, description, assignee(s), event, due date,
    priority and status. Statuses: Pending, In Progress, Completed.

-   Task filters: assignee, event, status and overdue.

## 2.5 Expenses and vendors

-   Expense tracking is not budgeting. Amount means
    estimated/agreed/committed amount, not instalment or actual payment
    amount.

-   Vendor directory is platform-admin-maintained; vendor
    self-registration is excluded.

-   Minimum vendor listing fields: name, category, service location and
    at least one contact method; other fields optional.

-   Wedding vendor management supports shortlist, event association,
    quoted/agreed amount, notes and statuses Shortlisted, Contacted,
    Confirmed, Not selected.

-   Google Places API supports discovery/place selection, subject to
    attribution, data retention and usage policies.

## 2.6 Guests, invitations and RSVP

-   A guest record represents one household contact, not each family
    member. Fields include name, email, permitted attendee count and
    invited events.

-   Each household receives a unique invitation link/token. Invitations
    by email; organisers may manually share links via WhatsApp (no
    WhatsApp API).

-   Guests do not register. One overall household RSVP is collected, not
    event-by-event RSVP. Capture accept/decline and attendee count
    within permitted count.

-   No separate maximum headcount per event and no approval workflow for
    additional guests.

-   Email invitation, resend, RSVP confirmation and reminders. Dashboard
    shows sent, accepted, declined, pending and confirmed headcount.

## 2.7 Public wedding website

-   Public route /w/:slug; anyone with URL can view; no login. Website
    must be published to be available.

-   Content: couple details/photos, wedding date, welcome message, event
    schedule, venues, map links, additional information and event
    livestream.

-   Themes: Traditional Indian, Minimalist and Royal/Luxury. Organisers
    customise text/photos/theme, preview, publish/unpublish and share
    URL.

-   English UI and system-generated communication in V1. SEO/indexing
    remains open.

## 2.8 Gallery, QR and livestream

-   Organiser creates event albums and uploads photos. Guests upload
    through unique link/QR without registration.

-   Gallery private by default; unique gallery link is sufficient, no
    passcode. Guest uploads require organiser approval before appearing;
    capture uploader name.

-   QR use cases: photo upload and gallery viewing. Photos in S3;
    metadata in PostgreSQL.

-   YouTube Live URL can be associated with an event and embedded on
    public website. No native streaming infrastructure.

## 2.9 Activity and notifications

-   Activity records wedding creation, membership/permission changes,
    event/task/vendor/expense/guest/invitation/RSVP/photo actions.
    Fields: actor, action, entity type/id, timestamp and wedding.
    Attribute guest actions to guest/invitation where possible.

-   Email only: organiser/guest invitations, RSVP
    confirmations/reminders and account emails. No WhatsApp/SMS
    automation, push, in-app notification centre or realtime updates.

## 2.10 AI-Assisted Wedding Setup

AI-assisted setup is an optional V1 onboarding path; manual setup
remains available. A guided dynamic conversation collects wedding
information, asks contextual follow-up questions, and can be resumed
after the organiser leaves.

The assistant supports exact, approximate or undecided wedding dates.
Wedding tradition/community and approximate guest count are optional. It
generates editable draft sections for events, tasks, vendor categories,
wedding website text and gallery album structure. The couple chooses the
existing visual website theme.

Each section is reviewed and approved independently. A final
confirmation is required before records are created through the existing
Wedding, Planning, Vendor & Expense and Media services. AI never writes
directly to application tables.

Development uses synthetic data and an available free-tier LLM API,
initially Gemini, behind a provider-independent adapter. RAG, vector
search, a separate AI microservice and an ongoing AI assistant are
excluded from V1.

# 3. Architecture Principles and Constraints

-   Domain ownership: group related capabilities by business domain, not
    one service per screen or CRUD feature.

-   Wedding is tenant boundary; enforce wedding_id throughout
    tenant-owned data and operations.

-   Backend enforcement for authentication, authorisation, token
    validation, rate limits and data ownership.

-   Use standard PostgreSQL, SQLAlchemy and Alembic to preserve
    portability; avoid unnecessary Supabase-specific dependencies.

-   S3 remains private; access is granted through validated,
    time-limited URLs or controlled delivery.

-   Keep operations lean: no Kubernetes, Kafka, Redis or multiple EC2
    hosts initially.

-   Services own their schemas and do not directly query another
    service's tables.

-   Preserve future paths to CloudFront, RDS, SQS or ECS if justified by
    actual needs.

  ---------------------------------------------------------------------
  Constraint                         Implication
  ---------------------------------- ----------------------------------
  Portfolio/development scale        Prioritise working end-to-end
                                     product over high availability.

  Low budget                         Supabase Free initially; defer
                                     advanced infrastructure.

  Single EC2 host                    Shared CPU/memory and common host
                                     failure domain.

  Single Supabase database           Schema isolation is logical, not
                                     independent database isolation.

  No broker initially                PostgreSQL polling/claiming for
                                     background jobs.

  No realtime features               REST and periodic refresh are
                                     sufficient for V1.
  ---------------------------------------------------------------------

-   AI output is untrusted input: validate structured responses before
    use and require human approval before permanent business changes.

-   Keep AI provider integration behind a backend adapter; do not couple
    product logic to one model vendor.

-   Persist conversation progress and structured drafts in PostgreSQL;
    do not rely on provider conversation state as the system of record.

-   Do not introduce RAG, vector infrastructure or a separate AI service
    without a demonstrated retrieval requirement.

# 4. High-Level Architecture

Organisers / Guests / Public visitors \| HTTPS \| Nginx reverse proxy \|
React SPA + API routing \| -------------------------------- \| AWS EC2 /
Docker \| \| \| \| Identity Wedding \| \| Planning Guest \| \|
Vendor/Expense Media \| \| Notification / Worker \|
-------------------------------- \| \| \| v v v Supabase PG Amazon S3
Amazon SES (schemas) (objects) (email)

External integration: Google Places API Future delivery: CloudFront in
front of private S3 origin

React may initially be served by Nginx on EC2. Hosting the static
frontend separately on S3/CloudFront remains an open deployment choice.

## 4.1 Request flow

1.  Browser requests a public page or private dashboard route.

2.  Nginx serves the React SPA and forwards API requests to the
    appropriate FastAPI service.

3.  The service authenticates an organiser or validates a guest
    capability token.

4.  The service derives/validates the authorised wedding context.

5.  The service accesses its own schema and calls other services through
    APIs when needed.

6.  For media, the service authorises the operation and issues a
    short-lived S3 URL; browser uploads/downloads directly.

7.  For email, the service creates a persisted job; the worker sends it
    through the email provider.

AI setup requests are routed to the AI Planning Module within the
Wedding Service. The module validates the authenticated organiser and
setup-session ownership, persists the user message and current state,
calls the configured LLM provider, validates the response, persists the
assistant response/draft, and returns the result. It does not create
events, tasks, vendors, website records or albums directly.

# 5. Application Surfaces and Routing

  ---------------------------------------------------------------------------------
  Surface      Route             Access             Purpose
  ------------ ----------------- ------------------ -------------------------------
  Private      /app/\*           Organiser          Dashboard, events, tasks,
  dashboard                      authentication +   guests, expenses, vendors,
                                 wedding/module     settings
                                 authorisation      

  Public       /w/:slug          Public; wedding    Public
  website                        must be published  couple/event/venue/livestream
                                                    information

  Invitation   /invite/:token    Invitation         View invitation and submit
                                 capability token   household RSVP

  Gallery      /gallery/:token   Gallery read token View approved gallery

  Upload       /upload/:token    Upload capability  Upload to authorised
                                 token              wedding/event
  ---------------------------------------------------------------------------------

A public slug is an identifier, not a secret. Public APIs return only
publishable fields. Guest tokens are bearer credentials and must be
scoped, protected and revocable.

/api/v1/auth/\* -\> Identity /api/v1/weddings/\* -\> Wedding
/api/v1/ai-setup/\* -\> AI Planning Module (within Wedding Service)
/api/v1/planning/\* -\> Planning /api/v1/guests/\* -\> Guest
/api/v1/vendors/\* -\> Vendor & Expense /api/v1/expenses/\* -\> Vendor &
Expense /api/v1/media/\* -\> Media

Exact endpoint contracts and naming are implementation details to be
finalised during API design.

### AI setup request flow

1.  Authenticated organiser starts or resumes a setup session through
    the AI setup API.

2.  Wedding Service verifies session ownership and, when a wedding_id
    exists, validates wedding membership and applicable permissions.

3.  The module persists the incoming user message and current structured
    state before contacting the LLM.

4.  The provider adapter sends only the minimum context required for the
    current generation request; secrets, guest contact data and
    unrelated wedding data are excluded.

5.  The module validates model output against the expected Pydantic
    schema and application rules. Invalid output is rejected and may be
    retried within the configured limit.

6.  The module persists the assistant message, updated structured state
    and draft section, then returns the result to React.

7.  The organiser edits and approves sections independently. Final
    confirmation triggers idempotent calls to the relevant domain
    services; those services re-check permissions and validate all data.

# 6. Domain Services

  ---------------------------------------------------------------------
  Service                            Responsibilities
  ---------------------------------- ----------------------------------
  Identity                           Organiser registration, mandatory
                                     verification, login/logout,
                                     password recovery, access/refresh
                                     tokens and profile. Does not own
                                     wedding membership permissions.

  Wedding                            Wedding creation/details, primary
                                     location, organiser memberships,
                                     Admin/Manager roles, module
                                     permissions and wedding settings.
                                     Authoritative owner of membership.

  Planning                           Configurable events, schedules,
                                     venue details, tasks, assignments,
                                     status and filters.

  Guest                              Household guest records, invited
                                     events, invitation tokens,
                                     invitation status and one overall
                                     household RSVP.

  Vendor & Expense                   Curated vendor directory
                                     records/integration, Google Places
                                     references, wedding
                                     shortlists/selections, vendor
                                     status and expense records.

  Media                              Website
                                     content/themes/publication,
                                     albums/photos, upload
                                     authorisation, guest tokens, S3
                                     presigned URLs, approval workflow
                                     and livestream settings.

  Notification / Worker              Email job persistence/processing,
                                     provider abstraction, SES
                                     integration, retry/backoff and
                                     manually initiated reminder requests.

  AI Planning Module (within Wedding AI setup sessions/messages,
  Service)                           dynamic conversation
                                     orchestration, structured
                                     information extraction, draft
                                     generation, provider adapter,
                                     prompt versioning, usage limits,
                                     output validation and approval
                                     workflow coordination. It does not
                                     directly write to other domain
                                     schemas or perform unrestricted
                                     actions.
  ---------------------------------------------------------------------

## 6.1 Activity logging

Activity logging is cross-cutting, not a standalone microservice in V1.
Use a shared activity schema/table with a reusable Python helper.
Services record actor, wedding_id, action, entity type/id and timestamp.
Where practical, write the activity record in the same database
transaction as the business action. Service roles receive only the
required insert permissions; the dashboard reader receives read access.

## 6.2 Service communication

-   Synchronous REST/API for immediate validation and responses.

-   PostgreSQL email job table for asynchronous email work.

-   No Kafka/general event-streaming platform in V1.

-   No direct cross-schema reads/writes by another service; use service
    APIs/contracts.

-   Cross-service references use IDs validated through the owning
    service or an agreed integration contract.

## 6.3 AI Planning Module

For V1, AI Planning is a logical module deployed as part of the Wedding
Service container/application. It is not a separate microservice and
does not need its own database or vector store. The module owns
setup-session orchestration and the AI-specific persistence records in
the wedding schema.

Responsibilities: create/resume/delete an organiser's setup session;
maintain chronological messages and structured collected information;
call the configured LLM through a provider adapter; produce typed draft
sections; validate output with Pydantic; track prompt version and
generation usage; coordinate independent section approvals and final
confirmation.

The module may call existing domain APIs/contracts to validate and
create approved records. It must not directly insert/update Planning,
Vendor & Expense or Media business tables. Existing services remain
authoritative for validation, permissions and persistence.

Draft sections: wedding details, event suggestions, task checklist,
vendor category requirements, website written content and album
structure. The user selects an existing website theme; the model does
not create visual themes or publish the site.

Conversation and draft state are persisted before/after provider calls.
A user message is saved before the LLM request. A successful assistant
response and any structured state update are saved before being
returned. Provider failures preserve the user message and last
successful draft so the user can retry or resume.

The module uses bounded prompts and token budgets, application-level
usage counters and provider quota handling. Initial development
defaults: up to 50 user messages per setup session and up to 3
generation attempts per generated section (events, tasks, vendor
categories, website content and albums). Editing existing suggestions
does not consume a generation attempt.

Initial development provider is Gemini API free tier, subject to current
provider quotas; Groq and OpenRouter may be used for comparison.
Provider and model are configuration-driven. Use synthetic data only
during development. Production provider, privacy terms and commercial
plan are deferred until real-user launch.

# 7. Multi-Tenancy and Workspace Isolation

The application is multi-wedding and multi-tenant. Each wedding is an
independent workspace and the tenant boundary. V1 restricts each
organiser account to at most one wedding membership.

User A ---- membership ---- Wedding A User B ---- membership ----
Wedding A User C ---- membership ---- Wedding A

User D ---- membership ---- Wedding B User E ---- membership ----
Wedding B

One user -\> at most one active wedding membership (V1). One wedding -\>
multiple organiser memberships.

-   All tenant-owned records carry wedding_id: events, tasks, guests,
    invitations, RSVPs, expenses, wedding vendor selections, albums,
    photos, website configuration, activity and email jobs.

-   Platform-wide vendor catalogue is global; wedding-specific
    shortlist/notes/amounts are tenant-owned.

-   S3 object keys include wedding ID,
    e.g. weddings/{wedding_id}/gallery/{event_id}/...

1.  Authenticate organiser and resolve membership through trusted
    Wedding Service context.

2.  Derive wedding_id from server-validated membership; never trust
    request-body wedding_id as proof of access.

3.  Scope every tenant-owned query by authorised wedding_id.

4.  When accessing a record by ID, verify it belongs to the authorised
    wedding.

5.  Apply Manager module permission in addition to membership
    validation.

6.  Include and validate wedding_id in background jobs.

7.  Use tenant-aware unique constraints and indexes.

8.  Test isolation with at least two wedding fixtures.

Enforce one-wedding-per-user in backend logic and, where practical, a
database uniqueness constraint for active/accepted memberships. Define
pending, declined and revoked membership semantics explicitly.

# 8. Data Architecture

## 8.1 Supabase PostgreSQL

Supabase PostgreSQL Free is selected for initial development. Mumbai
(ap-south-1) is the proposed region. Production database provider
remains open for later review: paid Supabase or Amazon RDS may be
evaluated.

## 8.2 Schema organisation

  ---------------------------------------------------------------------
  Schema                             Representative data
  ---------------------------------- ----------------------------------
  identity                           users, email_verifications,
                                     refresh_tokens,
                                     password_reset_tokens

  wedding                            weddings, memberships,
                                     organiser_invitations,
                                     module_permissions,
                                     wedding_settings

  planning                           events, tasks, task_assignments

  guest                              guests, guest_event_invitations,
                                     invitation_tokens,
                                     rsvp_reminder_requests, rsvps

  vendor_expense                     vendor catalogue references,
                                     wedding_vendors, expenses

  media                              website settings/content, albums,
                                     photos, upload_tokens,
                                     gallery_tokens, media metadata

  notification                       email_jobs,
                                     email_delivery_records, provider
                                     events

  activity                           activity_logs

  AI setup (Wedding Service; wedding ai_setup_sessions,
  schema)                            ai_setup_messages,
                                     ai_finalisation_workflows and
                                     ai_finalisation_steps; structured
                                     draft/state stored on session
  ---------------------------------------------------------------------

These are logical schema boundaries. AI setup session tables are owned
by the Wedding Service and placed in the wedding schema for V1; no
separate ai schema is required initially. Exact table placement may be
refined. The global vendor catalogue must remain distinct from
wedding-specific vendor selections.

### 8.2.1 AI setup persistence

AI setup uses four tables owned by the Wedding Service in the wedding schema:

-   ai_setup_sessions: id, organiser_user_id, nullable wedding_id
    (linked after workspace creation), status, current_stage,
    collected_information (validated JSONB), draft_plan (validated
    JSONB), prompt_version, message_count, per-section generation
    counters, created_at, updated_at and optional
    completed_at/deleted_at.

-   ai_setup_messages: id, session_id, sender_type
    (user/assistant/system as required), content, optional
    structured_payload/metadata, request_id or idempotency reference
    where needed, created_at. Keep messages ordered by timestamp and
    stable ID.

-   ai_finalisation_workflows: one durable, idempotent coordinator record per
    setup session; records wedding, status (`PENDING`, `IN_PROGRESS`,
    `COMPLETED`, `FAILED_RETRYABLE`), lifecycle timestamps and sanitised error
    data.

-   ai_finalisation_steps: per-workflow steps for `wedding`, `events`,
    `tasks`, `vendor_categories`, `website_content` and `albums`; records
    status, attempts, timestamps, errors and owning-service resource
    references. Completed steps are skipped when a workflow resumes.

Use a foreign key from messages to sessions with an intentional deletion
policy. Enforce session ownership in application logic and database
access boundaries. A session created before a wedding exists has
wedding_id NULL; after successful workspace creation it is associated
with that wedding. Store structured draft state independently of raw
messages so a session can resume without replaying the entire
transcript.

Finalisation is separate from session persistence. Wedding Service creates the
workflow and invokes each owning service with a per-step idempotency key; it
does not write another service's schema. On partial success, it records the
completed resource reference, leaves a recoverable failure as
`FAILED_RETRYABLE`, and resumes from the first incomplete step after retry.

Do not store LLM API keys, access/refresh tokens, S3 presigned URLs or
guest contact lists in session messages or draft JSON. Apply a
retention/deletion policy before real-user launch. Deleting an
unfinished setup removes its messages and unapproved draft but must not
delete any approved wedding records.

## 8.3 Roles, migrations and connections

-   Separate PostgreSQL runtime roles per service, restricted to the
    service's schema and required sequences/functions.

-   Use a separate migration/admin credential; do not put the Supabase
    postgres superuser credential in every container.

-   Each service owns its models and Alembic migration history for its
    schema.

-   Use TLS and store credentials outside source control.

-   Set conservative SQLAlchemy pool sizes across containers to avoid
    exhausting shared database connections.

-   Use standard PostgreSQL through SQLAlchemy/Psycopg. Supabase Auth,
    Supabase Storage and direct browser Data API access are not
    selected.

-   Use direct connection where runtime networking supports it;
    otherwise use Supabase session pooler for IPv4-only environments.
    Verify plan/connection details during setup.

## 8.4 Development backups

-   Supabase Free does not include automatic backups; create manual
    logical exports, especially before migrations.

-   Keep Alembic migrations and deterministic seed scripts in version
    control.

-   Store backups securely and do not commit real personal data.

## 8.5 Future database migration

Keep application data access standard PostgreSQL. Moving to RDS later
will still require schema/data transfer, roles/permissions/extensions
recreation, connection changes, validation and a controlled cutover;
portability reduces application changes but does not make migration
automatic.

# 9. Authentication and Authorisation

Custom organiser authentication is selected. Identity Service owns
credentials and authentication lifecycle. Passwords must be stored using
a modern password hashing algorithm; never store or log plaintext
passwords.

-   Email verification is mandatory before protected organiser use.

-   Use a 15-minute access token and a 7-day refresh token as the agreed
    initial policy; rotate refresh tokens on use and support revocation.

-   Support refresh-token rotation and revocation.

-   Password reset tokens are random, time-limited, single-use and
    stored safely (preferably hashed).

-   Validate JWT signature, expiry and configured
    issuer/audience/claims.

-   Keep signing secrets outside source control.

-   Authentication establishes identity; authorisation validates wedding
    membership and module permission.

-   Backend enforces Admin/Manager access. Frontend route guards are not
    security controls.

  ---------------------------------------------------------------------
  Role                               Permission
  ---------------------------------- ----------------------------------
  Admin                              Full wedding workspace control,
                                     membership/role/permission
                                     management and all modules.

  Manager                            Only module permissions assigned
                                     by an Admin: No access, View or
                                     Manage.
  ---------------------------------------------------------------------

# 10. Guest Token Access

  -----------------------------------------------------------------------
  Token                   Allowed capability      Must not allow
  ----------------------- ----------------------- -----------------------
  Invitation              View relevant           Organiser dashboard or
                          invitation and submit   unrelated wedding data
                          household RSVP          

  Gallery                 View approved gallery   Upload, approve, delete
                          within its              or manage photos
                          wedding/gallery scope   

  Upload                  Upload to authorised    View private organiser
                          wedding/event scope     data or approve/publish
                                                  photos
  -----------------------------------------------------------------------

-   Generate high-entropy tokens with a cryptographically secure
    generator.

-   Store token hashes where feasible; reveal raw token only for link
    distribution.

-   Bind every token to a wedding and capability; store only token
    hashes where feasible.

-   Organiser/guest invitation tokens retain their separately defined
    expiry rules.

-   Guest gallery-view and guest-upload tokens have no `expires_at` and
    do not expire automatically; they remain valid until manually
    revoked or replaced by an Admin.

-   Do not consume invitation token on GET; email scanners may prefetch
    links. RSVP state change requires explicit POST.

-   Do not log raw tokens, token URLs or presigned URLs.

-   Consider exchanging token for a short-lived scoped HTTP-only cookie
    and redirecting to a clean URL.

A guest token is a bearer credential: anyone possessing it can use its
scoped capability until it is revoked; invitation tokens additionally
stop working at their defined expiry. Gallery-view and guest-upload
tokens have no automatic expiry.

# 11. Media, Uploads and Image Delivery

## 11.1 Storage

Store binaries in a private S3 bucket; store metadata, object keys,
uploader, approval state, wedding_id and event/album associations in
PostgreSQL.

private-bucket/ weddings/{wedding_id}/ profile/ website/
gallery/{event_id}/

## 11.2 Upload flow

1.  Organiser/guest requests upload authorisation from Media Service.

2.  Service validates organiser membership/permission or guest upload
    token revocation state, scope and quota.

3.  Service creates a short-lived presigned S3 upload URL or constrained
    presigned POST.

4.  Browser uploads directly to S3; binary does not pass through
    FastAPI/EC2.

5.  Browser reports completion; backend verifies object and writes
    metadata.

6.  Guest uploads are marked PENDING_APPROVAL and include uploader name.

7.  Organiser approves/rejects; only approved guest photos appear in
    gallery.

## 11.3 Proposed file limits (not final approval)

  Category              Proposed max   Formats
  --------------------- -------------- ----------------------
  Guest gallery image   10 MB          JPEG, PNG, WebP
  Organiser image       10 MB          JPEG, PNG, WebP
  Normal API body       1--2 MB        JSON/small form data

-   Validate extension and actual signature/MIME; do not trust browser
    content type.

-   Safely decode images; reject malformed files.

-   Enforce file size, dimensions and pixel-count limits.

-   Generate object keys server-side; do not trust original filenames.

-   Re-encode images to approved formats and strip unnecessary EXIF
    metadata where appropriate.

-   Keep S3 private, encrypted and IAM-restricted.

## 11.4 Delivery and CDN

Recommended public website image delivery (pending product approval):
use Amazon CloudFront with Origin Access Control (OAC) in front of the
existing private S3 bucket. Keep S3 Block Public Access enabled and
grant the CloudFront distribution read access only to a dedicated
`website-public/` object prefix. Published website images use stable
CloudFront URLs (not expiring S3 presigned URLs). Only images
intentionally selected for a published public wedding website are
copied/placed under that prefix. The website itself is public to anyone
with its URL, so these published image assets are also publicly
retrievable through CloudFront; unpublishing removes them from the page
but should not be treated as a guarantee that a previously known image
URL is immediately inaccessible.

AWS currently provides a monthly CloudFront free allowance of 1 TB data
transfer out and 10 million HTTP/HTTPS requests. Usage beyond the
allowance is billable, so configure billing alerts and monitor usage.
OAC keeps the S3 bucket private while allowing the designated
distribution to fetch objects. This is the recommended low-complexity
AWS-native approach; final adoption remains pending approval. Private
gallery images remain separate: validate the gallery-view token first,
then issue short-lived viewing URLs. Do not expose guest gallery objects
through the public website-image prefix. CloudFront does not resize
images; thumbnails/variants are a separate processing concern.

# 12. Email and Background Processing

## 12.1 Provider

Amazon SES is the intended production email provider. The application
uses an EmailProvider abstraction; a Resend adapter may be used during
development/testing if desired. Notification code must not be tightly
coupled to SES.

## 12.2 PostgreSQL-backed job table

A lightweight database-backed job mechanism is selected. No SQS, Celery,
Redis, RabbitMQ or Kafka initially.

Canonical job lifecycle shared by database, API, worker and UI:
`PENDING`, `PROCESSING`, `SENT`, `FAILED`, `CANCELLED`. A scheduled job
remains `PENDING` until `scheduled_at` is due. A retryable failure also
returns to `PENDING` with `next_attempt_at`; `SCHEDULED` and `RETRYING`
are not separate job statuses. `SENT` means the provider accepted the
message, while delivery outcomes (DELIVERED, BOUNCED, COMPLAINED, etc.)
are recorded separately.

  ---------------------------------------------------------------------
  Field                              Purpose
  ---------------------------------- ----------------------------------
  id                                 Unique job ID

  wedding_id                         Tenant context where applicable

  recipient_email                    Recipient

  template_name                      Invitation, verification, RSVP
                                     confirmation, reminder, etc.

  payload                            Validated template data

  status                             PENDING, PROCESSING, SENT, FAILED,
                                     CANCELLED

  attempts                           Number of attempts

  scheduled_at                       Earliest processing time

  last_error                         Sanitised error

  provider_message_id                Provider reference

  timestamps                         Created/updated
  ---------------------------------------------------------------------

-   Separate worker container polls for due jobs.

-   Initial development default: one worker, poll every 5 seconds, claim
    up to 5 jobs per batch and process sequentially. Tune after
    observing resource usage and provider quotas.

-   Claim jobs transactionally using SELECT FOR UPDATE SKIP LOCKED or
    equivalent atomic claiming.

-   Use bounded concurrency and respect SES sending quotas.

-   Record provider response/message ID and update status.

-   Retry transient errors with exponential backoff and jitter; classify
    permanent failures.

-   Use idempotency keys/state transitions to reduce duplicate sends.

-   Track delivery/bounce/complaint events where configured.

-   Start with up to 3 attempts for transient email failures using
    increasing delays; classify permanent failures. Polling interval and
    batch size may be tuned after testing.

-   Cancellation uses no extra status. A `PENDING` job becomes `CANCELLED`
    immediately. A cancellation of a `PROCESSING` job persists
    `cancel_requested_at`; the worker checks it before provider submission and
    suppresses later retries. A provider-accepted message remains `SENT` and
    terminal jobs cannot be cancelled again.

FastAPI BackgroundTasks is not the durable queue: process restarts may
lose in-memory work. Bulk invitations and manually initiated reminder requests use
persisted jobs.

# 13. Vendor Discovery and Location Model

## 13.1 Google Places API

-   Use Places search/autocomplete/details for vendor discovery and
    place selection as needed.

-   Store Google Place ID as an external reference where permitted.

-   Do not assume all Places response fields may be retained
    indefinitely; follow current Google terms for caching, attribution
    and display.

-   Use minimal field masks, API restrictions, quotas and billing
    alerts.

-   Platform-admin-curated vendor records remain the source of truth;
    discovery does not enable vendor self-registration.

## 13.2 Structured location

Store the primary wedding location separately from each event venue;
events may occur in different cities.

  Field                    Purpose
  ------------------------ -------------------------------------------
  venue_name               Optional venue/business name
  formatted_address        Human-readable address
  city                     City
  state                    State/region
  postal_code              Optional PIN/postal code
  country / country_code   Country name and standard code
  latitude / longitude     Optional coordinates
  google_place_id          Optional Places reference
  timezone                 IANA timezone; recommended for scheduling

Location data should be user-confirmed/application-owned. Google-derived
content must follow Google storage and display rules. PostGIS is not
required for V1.

# 14. Security Architecture

## 14.1 Network and secrets

-   Expose only Nginx HTTP/HTTPS publicly; FastAPI ports remain on
    private Docker network.

-   Supabase is external to the AWS VPC: use TLS, strong service
    credentials and database network restrictions where available.

-   Use EC2 IAM role for AWS access, not static AWS keys on the host.

-   Prefer Systems Manager Session Manager for EC2 administration rather
    than opening SSH globally.

-   Use HTTPS on all user-facing routes.

-   Never log passwords, JWTs, guest tokens, presigned URLs or provider
    secrets.

-   Keep database/JWT/provider secrets out of Git and frontend bundles;
    use least-privilege IAM.

## 14.2 Rate limiting

Use Nginx IP-based throttling plus application-level
account/token/action limits. Initial defaults: general API 60
requests/minute/IP; login 5/minute/IP; registration 5/hour/IP; forgot
password 3/15 minutes/account or email; organiser invitations
10/minute/organiser; RSVP 5/minute/token; guest upload URL generation
10/minute/token. Tune with tests; these are starting thresholds.

  Endpoint          Suggested protection
  ----------------- ------------------------------------------------------------
  Login             Strict per-IP and per-account limits
  Registration      Per-IP abuse control
  Forgot password   Per-IP and per-email; avoid account enumeration
  Public APIs       Per-IP
  RSVP              Per-token and per-IP
  Gallery           Per-token and per-IP
  Upload            Per-token, per-IP, per-wedding and upload-count/size quota
  Vendor search     Per-user/IP and Google API quota

PostgreSQL counters can support token/account-specific quotas at initial
scale. Redis/shared rate-limit storage can be considered if multiple
hosts or higher throughput are introduced.

## 14.3 Request and file limits

-   Set a small normal API body limit (1--2 MB was proposed).

-   Do not proxy large image binaries through FastAPI; use direct S3
    uploads.

-   Use S3 POST content-length-range where applicable.

-   Validate actual file type, dimensions, decodeability and quotas.

-   Generate random object keys and prevent users from choosing
    arbitrary S3 keys.

# 15. Observability

V1 uses basic operational visibility only; no external observability
platform is selected initially.

-   Structured JSON application logs from each service and worker.

-   Nginx access/error logs.

-   Docker log rotation with bounded max-size/max-file.

-   Request/correlation ID propagated where practical.

-   Basic liveness and readiness endpoints, e.g. /health/live and
    /health/ready.

-   Sanitised errors; never log secrets or sensitive guest data.

The business Activity Log is separate from operational logs and remains
a product feature.

  Possible future tool   Purpose
  ---------------------- ------------------------------------------
  CloudWatch             Centralised AWS logs, metrics and alarms
  Sentry                 Application exception tracking
  Prometheus             Metrics collection
  Grafana                Dashboards
  OpenTelemetry          Distributed tracing
  OpenSearch/ELK         Searchable centralised logs

# 16. Deployment and Infrastructure

## 16.1 Initial hosting

One EC2 instance runs Nginx, React build if co-hosted, domain FastAPI
containers and background worker. Docker Compose manages containers and
private network. Exact EC2 family/size and ARM64 vs x86_64 remain open.

  ---------------------------------------------------------------------
  Resource                           Placement/access
  ---------------------------------- ----------------------------------
  EC2                                Publicly reachable through Nginx;
                                     expose only 80/443.

  FastAPI containers                 Private Docker network; no direct
                                     public exposure.

  Supabase PostgreSQL                External managed database; TLS and
                                     restricted credentials/network
                                     access.

  Amazon S3                          Private bucket; IAM and presigned
                                     access.

  Amazon SES                         Managed email service used by
                                     worker/service.
  ---------------------------------------------------------------------

## 16.2 EC2 candidate

T4g.medium (2 vCPU, 4 GiB RAM) was discussed as a candidate, subject to
ARM64 support across base images and native Python dependencies.
T3.medium is an x86_64 alternative. Neither has been finalised. Size
using measured resource usage.

## 16.3 Network and cost posture

-   RDS is not used in the initial development phase; Supabase is
    external to VPC.

-   No NAT Gateway, ALB, ECS, EKS, Redis or multiple EC2 instances
    planned initially.

-   Nginx handles reverse proxy for the single-host deployment.

-   Keep S3 private and use least-privilege IAM.

-   CloudFront can be added later.

## 16.4 Frontend, domain and TLS

React SPA is selected. Frontend placement remains open: serve via Nginx
on EC2 initially (proposed default), or separate static assets to
S3/CloudFront. HTTPS is required. Nginx with automated certificate
renewal (e.g. Let's Encrypt/Certbot) is a proposed approach;
domain/DNS/certificate automation is not finalised.

# 17. CI/CD and Infrastructure as Code

Proposed, not yet explicitly confirmed: GitHub Actions builds/tests
images, pushes to Amazon ECR, and deploys to EC2 through Systems
Manager. Terraform is proposed for AWS infrastructure.

Git push / pull request \| v GitHub Actions: lint, tests, build images
\| v Amazon ECR \| v Systems Manager command to EC2 \| v EC2 pulls image
and updates affected Compose service \| v Health check and deployment
verification

-   Use GitHub Actions OIDC to assume AWS role instead of long-lived
    keys.

-   Tag images immutably, e.g. commit SHA.

-   Deploy affected services independently where possible.

-   Run Alembic migrations as a controlled release step.

-   Keep rollback image/version and deployment logs.

-   Terraform may manage EC2, IAM, security groups, S3 and other AWS
    resources.

Repository layout, pipeline triggers, rollback automation and migration
release procedure remain to be specified.

# 18. Reliability, Scaling and Evolution

-   One EC2 host is a single point of failure; accepted for
    development/portfolio phase.

-   Supabase Free may pause after inactivity and has no automatic
    backups; suitable only with manual exports during development.

-   Persisted email jobs survive worker restarts; idempotency/retry
    logic is required.

-   Container restart policies and health checks do not replace host
    availability or backups.

-   Verify S3 object upload completion before marking media available.

1.  Development: local Docker Compose + Supabase Free; S3/SES test
    configuration.

2.  Portfolio deployment: one EC2, Nginx, separate containers, Supabase
    PostgreSQL, private S3 and SES when configured for sending.

3.  Early production: assess paid Supabase vs RDS, backups,
    CloudWatch/Sentry, CloudFront, deployment rollback and security
    review.

4.  Growth: consider ECS/Fargate, multiple tasks/hosts, load balancing
    and shared rate limiting.

5.  Higher scale: consider SQS, independent database deployments,
    caching and advanced observability based on measured need.

# 19. V1 Exclusions

-   Guest registration/login; multi-wedding membership per organiser;
    planners managing multiple weddings.

-   Accommodation/hotels/rooms; travel/transport booking; seating
    arrangements.

-   Budget planning, instalments, advances and outstanding payment
    tracking.

-   Automated WhatsApp/SMS; native livestream; realtime/WebSockets;
    in-app notifications/push.

-   Vendor self-registration, bookings, commissions and lead management.

-   Ongoing AI wedding assistant; RAG/vector database; separate AI
    microservice; unrestricted AI agent actions; Kafka; Kubernetes/EKS;
    ECS/Fargate; Redis; complex event streaming in initial deployment.

# 20. Open Decisions and Proposed Defaults

  ---------------------------------------------------------------------
  Area                               Status / next step
  ---------------------------------- ----------------------------------
  AWS region                         Confirmed: Mumbai ap-south-1.

  EC2 instance                       Initial preference: eligible
                                     free-tier micro instance; verify
                                     account eligibility, architecture
                                     compatibility and measured memory.
                                     Upgrade only if required.

  Frontend hosting                   Confirmed initial approach: React
                                     build served by Nginx on EC2.

  Supabase region/connectivity       Confirmed: Mumbai; use Supabase
                                     shared pooler in session mode
                                     initially (IPv4-compatible).
                                     Direct connection can be
                                     reconsidered if IPv6 is available
                                     and useful.

  Production database                Deferred; use Supabase Free during
                                     development and revisit before
                                     production.

  JWT policy                         Confirmed initial defaults:
                                     15-minute access token, 7-day
                                     refresh token, rotation and
                                     revocation; access token in
                                     frontend memory, refresh token in
                                     Secure/HttpOnly/SameSite cookie
                                     with appropriate CSRF protection.

  Activity integration               Confirmed: shared activity
                                     schema/table with reusable Python
                                     helper; record in the same
                                     transaction as the business action
                                     where practical.

  Email worker tuning                Initial default: one worker,
                                     5-second polling, batch up to 5,
                                     sequential processing, up to 3
                                     retries with increasing delays;
                                     tune after testing.

  Public website image delivery      Recommended: stable CloudFront
                                     URLs backed by a private S3 bucket
                                     using OAC and a dedicated
                                     website-public/ prefix; free
                                     within the published monthly
                                     allowance, with billing alerts.
                                     Pending approval. Private gallery
                                     images continue to use short-lived
                                     URLs after token validation.

  CloudFront                         Recommended for stable public
                                     wedding website images from
                                     private S3 using OAC; pending
                                     approval. Keep gallery assets
                                     outside the public website prefix.

  Rate limits                        Initial defaults: general API
                                     60/min/IP; login 5/min/IP;
                                     registration 5/hour/IP; forgot
                                     password 3/15 min/account or
                                     email; invitations
                                     10/min/organiser; RSVP
                                     5/min/token; upload URL
                                     10/min/token.

  Guest token lifecycle              Invitation tokens expire under the
                                     guest-invitation rules.
                                     Gallery-view and guest-upload
                                     links do not expire automatically
                                     and are manually revoked/replaced.
                                     Store token hashes where feasible.

  Google Places                      Use restricted API keys, minimal
                                     field masks, quotas and billing
                                     alerts; follow attribution and
                                     data-retention rules.

  SEO                                Decide whether public wedding
                                     websites must be indexed; may
                                     affect prerendering/SSR.

  CI/CD and IaC                      GitHub Actions + ECR + SSM and
                                     Terraform are proposed, pending
                                     confirmation.

  TLS/domain                         Confirm DNS/domain and certificate
                                     renewal approach.

  AI provider                        Development: Gemini free tier with
                                     synthetic data; Groq/OpenRouter
                                     may be used for comparison.
                                     Provider adapter and model are
                                     configurable. Production
                                     provider/privacy terms deferred.

  AI architecture                    AI Planning Module within Wedding
                                     Service; not a separate
                                     microservice. No RAG/vector
                                     database in V1.

  AI usage                           50 user messages/session; up to 3
                                     generation attempts per generated
                                     section; bounded tokens and
                                     per-session usage tracking.

  AI persistence                     ai_setup_sessions and
                                     ai_setup_messages in wedding
                                     schema; save/resume; prompt
                                     version recorded.

  AI approval                        Independent section approval plus
                                     final confirmation; idempotent
                                     creation through existing domain
                                     services only.
  ---------------------------------------------------------------------

This list intentionally distinguishes agreed direction from unresolved
implementation choices. It should be reviewed before treating every
operational detail as final.

# Appendix A. Suggested Repository Layout

make-my-marriage/ frontend/ services/ identity/ wedding/ ai_setup/ \# AI
Planning Module; part of Wedding Service models/ api/ planning/ guest/
vendor_expense/ media/ notification/ shared/ contracts/ security/
observability/ ai_providers/ \# provider adapter/interfaces deploy/
nginx/ compose/ infra/ terraform/ scripts/ seed/ backup/ docs/
.env.example README.md

Suggested only. Keep shared code small to avoid hidden coupling between
services.

# Appendix B. Key Workflows

## B1. Create wedding and invite organisers

1.  User registers and verifies email.

2.  User creates wedding; Wedding Service creates workspace and assigns
    creator Admin membership.

3.  Creator invites second Admin/Managers.

4.  Invitee registers/logs in and accepts invitation.

5.  Wedding Service enforces one-wedding-per-user V1 restriction and
    records membership/permissions.

## B2. Household invitations and RSVP

1.  Organiser creates household guest records and selects invited
    events.

2.  Guest Service creates invitation records and scoped tokens.

3.  Notification functionality inserts email jobs into PostgreSQL.

4.  Worker claims jobs and sends via SES; status/provider reference is
    recorded.

5.  Guest opens invitation link; GET does not consume token.

6.  Guest submits overall household RSVP; service validates token,
    wedding/guest scope and attendee count.

7.  RSVP confirmation is queued.

## B3. Guest photo upload and approval

1.  Guest opens upload link/QR.

2.  Media Service validates upload token revocation state, wedding/album
    scope and quota; the guest media token itself has no expiry.

3.  Service issues short-lived constrained S3 URL.

4.  Guest uploads directly to private S3.

5.  Media Service records uploader name and PENDING_APPROVAL status.

6.  Organiser approves/rejects; approved images become visible through
    gallery access flow.

## B4. Public wedding website

1.  Organiser edits content/theme and previews.

2.  Media Service requests the idempotent internal Wedding-Service slug
    operation. Wedding Service returns an existing reserved slug or generates
    and persists a unique `wedding.weddings.slug`; Media Service never writes
    this Wedding-owned field.

3.  Organiser publishes website.

4.  Visitor opens /w/:slug.

5.  Backend resolves slug and returns only published public fields.

6.  Public website media uses controlled stable delivery; private
    gallery media is not exposed.

## B5. AI-Assisted Wedding Setup

1.  Organiser chooses AI-assisted setup; Wedding Service creates an
    owned setup session.

2.  The organiser and assistant exchange messages. Each user message is
    persisted before the LLM request; successful assistant messages and
    structured state are persisted before response.

3.  The assistant collects couple details, date status/detail, primary
    location and desired events; tradition and guest count are optional.
    The date may be exact, approximate or undecided.

4.  The AI Planning Module suggests event drafts. The organiser edits
    and approves the events section.

5.  The module generates task and vendor-category drafts from approved
    events and collected preferences; the organiser reviews and approves
    each section.

6.  The module generates website written content and album structure.
    The organiser selects an existing visual theme and reviews/approves
    these sections.

7.  If the organiser leaves, the saved session, messages and structured
    draft are restored on resume.

8.  Final confirmation triggers idempotent requests to Wedding,
    Planning, Vendor & Expense and Media services. Each service
    validates permissions and payloads before writing its own records.

9.  On LLM failure, preserve progress and offer retry/resume. On invalid
    output, reject it and retry within limits; do not create business
    records.

# Document End

This document consolidates the Make My Marriage V1 decisions and
proposed implementation direction. Open decisions in Section 20 should
be resolved before treating every deployment detail as final.
