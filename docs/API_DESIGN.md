```{=html}
<!-- Source: Make_My_Marriage_API_Design_Specification_V1_Final.docx. Converted from the latest agreed Word document. Content and decisions are retained; Markdown table layout may differ from Word. -->
```
**MAKE MY MARRIAGE**

# API Design Specification

**Version 1.1 --- Final Consolidated Baseline**

9 November 2026 REST API • FastAPI • PostgreSQL • Multi-tenant wedding
workspaces

  Document property   Value
  ------------------- ---------------------------------------------------
  Status              Approved design baseline; implementation contract
  API base path       /api/v1
  Backend             Python FastAPI; seven domain services
  Database            PostgreSQL; SQLAlchemy and Alembic
  Tenant root         Wedding workspace
  Audience            Product, frontend, backend, QA and DevOps teams

Purpose: consolidate the approved endpoint inventory, global API
conventions, service ownership, security rules, 16 consistency decisions
and 10 approved validation-correction groups.

Scope note: this document records endpoint paths and functional
contracts captured in the approved design discussions. Where exact
field-level schemas or provider-specific behaviour were not explicitly
finalised, they are identified as implementation details or open
decisions rather than invented.

# Document control and architecture

  -----------------------------------------------------------------------
  Service / capability    Ownership               Schema
  ----------------------- ----------------------- -----------------------
  Identity Service        Organiser accounts,     identity
                          verification, password  
                          recovery and refresh    
                          sessions                

  Wedding Service         Wedding workspace,      wedding
                          membership,             
                          invitations,            
                          permissions, settings,  
                          AI setup and            
                          finalisation            
                          orchestration           

  Planning Service        Events, tasks,          planning
                          assignments and event   
                          schedules               

  Guest Service           Households,             guest
                          invited-event           
                          associations, guest     
                          tokens and RSVP         

  Vendor & Expense        Global vendors,         vendor_expense
  Service                 wedding-specific vendor 
                          selections and expenses 

  Media Service           Website, snapshots,     media
                          assets, albums, photos, 
                          guest media tokens and  
                          livestream              

  Notification Service    Email jobs, attempts,   notification
                          delivery events and     
                          worker                  

  Activity                Append-only business    activity
  (cross-cutting)         activity; originating   
                          service writes via      
                          shared helper           
  -----------------------------------------------------------------------

Activity is not a separate microservice. The architecture has seven
services and eight PostgreSQL schemas. Notification has a separate
worker process.

# Contents

-   1.  Global API conventions
-   2.  Authentication, permissions and tenant isolation
-   3.  Identity Service
-   4.  Wedding Service
-   5.  Planning Service
-   6.  Guest Service
-   7.  Vendor & Expense Service
-   8.  Media Service
-   9.  Notification Service
-   10. Cross-service consistency and workflows
-   11. Database, file and operational security
-   12. Validation corrections and open decisions

# 1. Global API conventions

## 1.1 URL, methods and response envelopes

-   REST API with URL versioning: /api/v1/. Use GET for retrieval, POST
    for creation/actions, PATCH for partial updates, PUT for complete
    replacement, and DELETE for deletion/revocation as specified.

-   Success envelope: success, message, data, and optional meta.

-   Error envelope: success:false, message, error:{code,details},
    request_id.

-   HTTP codes: 400 malformed/invalid operation; 401 unauthenticated;
    403 insufficient permission; 404 missing or inaccessible; 409
    business conflict/idempotency mismatch; 422 validation; 429 rate
    limit; 503 required dependency unavailable.

Success example: {"success":true,"message":"Guests
retrieved.","data":\[\],"meta":{"page":1,"limit":10,"total_items":0,"total_pages":0}}

Error example: {"success":false,"message":"Resource not
found.","error":{"code":"RESOURCE_NOT_FOUND","details":null},"request_id":"request-uuid"}

## 1.2 Pagination, sorting and dates

-   Pagination query: page and limit; defaults 1 and 10; maximum
    limit 100. meta contains page, limit, total_items and total_pages.

-   Resource-specific filters. Sorting uses sort_by and
    sort_order=asc\|desc. Each endpoint defines allowed sort fields and
    default sort; reject unsupported fields with 422.

-   Empty list results return an empty array, not 404.

-   Date-only: YYYY-MM-DD and PostgreSQL DATE; never timezone-shift a
    date-only value.

-   Date-time: ISO 8601 with timezone information; convert to UTC and
    store TIMESTAMPTZ.

-   Wedding timezone is an IANA zone, default Asia/Kolkata. Interpret
    organiser-entered event/schedule times in wedding timezone and
    display them back in that timezone.

-   Do not invent exact dates or deadlines for approximate or undecided
    wedding dates.

# 2. Authentication, permissions and tenant isolation

  ---------------------------------------------------------------------
  Control                            Approved contract
  ---------------------------------- ----------------------------------
  Access JWT                         15-minute lifetime; Authorization:
                                     Bearer; frontend memory only.

  Refresh token                      7-day lifetime;
                                     Secure/HttpOnly/SameSite cookie;
                                     rotate on use; CSRF protection for
                                     refresh.

  Verification                       Mandatory before protected wedding
                                     operations; verification token
                                     expires in 24 hours.

  Password                           8--128 characters; no mandatory
                                     character-class complexity; secure
                                     password hashing.

  Password reset/change              Revoke all refresh sessions;
                                     organiser must log in again.

  Logout                             Revoke refresh session and clear
                                     cookie; already-issued access JWT
                                     may remain valid until its
                                     15-minute expiry in V1.

  Refresh reuse                      Reuse of rotated token is possible
                                     compromise; revoke affected token
                                     family/session chain and require
                                     login.
  ---------------------------------------------------------------------

## 2.1 Roles and nine permission modules

  Module            Manager values
  ----------------- ---------------------------
  Wedding Details   NO_ACCESS / VIEW / MANAGE
  Events            NO_ACCESS / VIEW / MANAGE
  Tasks             NO_ACCESS / VIEW / MANAGE
  Guests            NO_ACCESS / VIEW / MANAGE
  Vendors           NO_ACCESS / VIEW / MANAGE
  Expenses          NO_ACCESS / VIEW / MANAGE
  Gallery           NO_ACCESS / VIEW / MANAGE
  Website           NO_ACCESS / VIEW / MANAGE
  Activity          NO_ACCESS / VIEW / MANAGE

-   Only ADMIN and MANAGER roles exist. Admin has full access to all
    nine modules; stored Manager permissions do not restrict Admin.

-   Manager MANAGE includes read/write; VIEW is read-only; NO_ACCESS
    denies access.

-   Only Admins create/resend/revoke organiser invitations, change
    another member's role, remove another member or manage Manager
    permissions.

-   Activity is append-only. Activity MANAGE does not allow
    editing/deleting history.

-   Maximum four Admins including pending Admin invitations; enforce
    transactionally and safely under concurrency. At least one active
    Admin must remain.

## 2.2 Tenant isolation

-   Derive organiser user_id from verified JWT; never trust client
    identity.

-   For every wedding-scoped request, validate active membership,
    wedding_id, role and module permission.

-   Validate referenced resources belong to same wedding. Conceal
    cross-wedding/inaccessible resources with 404 where appropriate.

-   Each service owns and validates its own data. Cross-service
    references use authenticated internal APIs, never direct
    cross-schema queries.

-   Internal service credentials are unique per service and never
    exposed to frontend or source control.

# 3. Identity Service

Owns organiser account and authentication lifecycle. Guests do not have
accounts.

## 3.1 Authentication and profile

All paths are under /api/v1. Refresh token is cookie-only and is not
returned in JSON.

  ------------------------------------------------------------------------
  Method               Endpoint                    Purpose / key contract
  -------------------- --------------------------- -----------------------
  POST                 /auth/register              Create organiser;
                                                   full_name, email,
                                                   password; queue
                                                   verification email.

  POST                 /auth/verify-email          Validate one-time
                                                   verification token;
                                                   24-hour expiry.

  POST                 /auth/resend-verification   Issue replacement
                                                   verification email
                                                   under rate limits.

  POST                 /auth/login                 Authenticate verified
                                                   organiser; return
                                                   access token/profile
                                                   and set refresh cookie.

  POST                 /auth/refresh               Rotate refresh cookie;
                                                   CSRF required; detect
                                                   token reuse.

  POST                 /auth/logout                Revoke current refresh
                                                   session and clear
                                                   cookie.

  POST                 /auth/logout-all            Revoke all refresh
                                                   sessions.

  POST                 /auth/forgot-password       Request reset email;
                                                   avoid account
                                                   enumeration.

  POST                 /auth/reset-password        Validate reset token,
                                                   set password and revoke
                                                   refresh sessions.

  POST                 /auth/change-password       Change password and
                                                   revoke all refresh
                                                   sessions.

  GET                  /auth/me                    Return organiser
                                                   profile only; no
                                                   membership/permission
                                                   payload.

  PATCH                /auth/me                    Update allowed profile
                                                   fields; email is
                                                   excluded.

  POST                 /auth/email-change          Initiate email change;
                                                   send verification to
                                                   new address; old
                                                   address remains active
                                                   until verified.

  POST                 /auth/verify-email-change   Verify new address and
                                                   complete change; reject
                                                   address already
                                                   registered.
  ------------------------------------------------------------------------

-   Email is normalised and unique case-insensitively.

-   Verification, reset and refresh tokens are high entropy; store
    hashes, not reusable plaintext.

-   Email change is separate from profile PATCH.

-   Logout revokes refresh session; access JWT can remain valid for up
    to 15 minutes as an accepted V1 trade-off.

# 4. Wedding Service

Owns wedding workspace, memberships, organiser invitations, permissions,
settings, AI setup sessions and AI finalisation coordination.

## 4.1 Workspace and settings

  -----------------------------------------------------------------------------
  Method             Endpoint                          Purpose / key contract
  ------------------ --------------------------------- ------------------------
  POST               /weddings                         Create workspace;
                                                       creator becomes Admin;
                                                       enforce one active
                                                       membership and
                                                       four-Admin cap.

  GET                /weddings/{wedding_id}            Retrieve core wedding
                                                       details.

  PATCH              /weddings/{wedding_id}            Update couple names,
                                                       date/date status,
                                                       location,
                                                       welcome/description and
                                                       media references.

  GET                /weddings/{wedding_id}/settings   Retrieve timezone,
                                                       currency_code,
                                                       date_format,
                                                       time_format,
                                                       rsvp_submissions_open,
                                                       default_language and
                                                       gallery_visibility.

  PATCH              /weddings/{wedding_id}/settings   Update timezone,
                                                       currency_code,
                                                       date_format, time_format
                                                       and
                                                       rsvp_submissions_open.
                                                       default_language and
                                                       gallery_visibility are
                                                       read-only V1 settings.
  -----------------------------------------------------------------------------

**Settings contract**

-   `currency_code` is an ISO 4217 code and defaults to `INR`.
-   `date_format` supports `DD_MMM_YYYY` and `DD_MM_YYYY`; default
    `DD_MMM_YYYY`.
-   `time_format` supports `12H` and `24H`; default `12H`.
-   `rsvp_submissions_open` defaults to `true`.
-   `default_language` is `en` and `gallery_visibility` is `LINK_ONLY`
    in V1; both are returned but are not editable.

The `GET /me/wedding` endpoint resolves the current organiser's active
workspace, role and permissions; it returns `data: null` if there is no
active membership.

**Internal website-slug contract:** `POST /internal/weddings/{wedding_id}/website-slug`
is callable only by Media Service using its dedicated service credential. Wedding
Service returns the already reserved slug, or transactionally generates,
uniqueness-checks and persists one in `wedding.weddings.slug`. The operation is
idempotent and is the only slug-reservation operation; it does not create a
Media Service website record.

## 4.2 Membership

  ----------------------------------------------------------------------------
  Method          Endpoint                                     Purpose / key
                                                               contract
  --------------- -------------------------------------------- ---------------
  GET             /weddings/{wedding_id}/members               List members.

  GET             /weddings/{wedding_id}/members/{member_id}   Retrieve
                                                               member.

  PATCH           /weddings/{wedding_id}/members/{member_id}   Admin changes
                                                               another
                                                               member's role;
                                                               cannot change
                                                               own role.

  DELETE          /weddings/{wedding_id}/members/{member_id}   Admin removes
                                                               another member;
                                                               cannot
                                                               self-remove
                                                               through this
                                                               route.

  POST            /weddings/{wedding_id}/leave                 Self-leave;
                                                               Admin may leave
                                                               only if another
                                                               active Admin
                                                               remains.
  ----------------------------------------------------------------------------

## 4.3 Organiser invitations

  ----------------------------------------------------------------------------------------
  Method      Endpoint                                                    Purpose / key
                                                                          contract
  ----------- ----------------------------------------------------------- ----------------
  POST        /weddings/{wedding_id}/invitations                          Admin invites as
                                                                          ADMIN or
                                                                          MANAGER; token
                                                                          delivered by
                                                                          email, not
                                                                          returned in
                                                                          admin response.

  GET         /weddings/{wedding_id}/invitations                          List
                                                                          invitations;
                                                                          Manager read
                                                                          depends on
                                                                          invitation
                                                                          visibility
                                                                          policy.

  POST        /weddings/{wedding_id}/invitations/{invitation_id}/resend   Admin only;
                                                                          revoke old token
                                                                          and issue fresh
                                                                          7-day
                                                                          invitation.

  DELETE      /weddings/{wedding_id}/invitations/{invitation_id}          Admin only;
                                                                          revoke pending
                                                                          invite and
                                                                          cancel eligible
                                                                          queued email.

  POST        /organiser-invitations/{token}/accept                       Authenticated,
                                                                          email-verified
                                                                          invitee accepts;
                                                                          email must
                                                                          match; check
                                                                          membership and
                                                                          Admin capacity.

  POST        /organiser-invitations/{token}/decline                      Authenticated,
                                                                          email-verified
                                                                          invitee
                                                                          declines; email
                                                                          must match.
  ----------------------------------------------------------------------------------------

-   Invitation statuses: PENDING, ACCEPTED, DECLINED, EXPIRED, REVOKED.
    Validity: seven days; resend creates a fresh seven-day token.

-   Invitation is not membership. Membership exists only after explicit
    acceptance.

-   Pending Admin invitations count against four-Admin capacity.
    Acceptance and membership creation are transactional and
    concurrency-safe.

-   One active wedding membership per organiser. At least one active
    Admin remains. Admin-to-Manager makes stored Manager permissions
    effective; Manager-to-Admin grants full access.

## 4.4 Module permissions

  ---------------------------------------------------------------------------------------
  Method       Endpoint                                                 Purpose / key
                                                                        contract
  ------------ -------------------------------------------------------- -----------------
  GET          /weddings/{wedding_id}/members/{member_id}/permissions   Retrieve full
                                                                        nine-module
                                                                        permission map.

  PUT          /weddings/{wedding_id}/members/{member_id}/permissions   Admin replaces
                                                                        full permission
                                                                        map; all nine
                                                                        modules required;
                                                                        missing/invalid
                                                                        values return
                                                                        422; Manager
                                                                        cannot change
                                                                        own.
  ---------------------------------------------------------------------------------------

## 4.5 AI-assisted wedding setup

  ---------------------------------------------------------------------------------------------------
  Method    Endpoint                                                          Purpose / key contract
  --------- ----------------------------------------------------------------- -----------------------
  POST      /wedding-setup/sessions                                           Start empty session;
                                                                              does not create
                                                                              wedding.

  GET       /wedding-setup/sessions/{session_id}                              Retrieve session,
                                                                              progress, structured
                                                                              data, draft sections
                                                                              and messages.

  POST      /wedding-setup/sessions/{session_id}/messages                     Submit user message;
                                                                              persist conversation
                                                                              and structured
                                                                              progress; max 50 user
                                                                              messages.

  PATCH     /wedding-setup/sessions/{session_id}                              Update collected source
                                                                              information; flag
                                                                              dependent sections
                                                                              NEEDS_REVIEW.

  POST      /wedding-setup/sessions/{session_id}/generate/{section}           Generate events, tasks,
                                                                              vendor-categories,
                                                                              website-content or
                                                                              albums; max three
                                                                              successful
                                                                              generations/section.

  PATCH     /wedding-setup/sessions/{session_id}/sections/{section}           Edit section; edits do
                                                                              not consume generation
                                                                              attempts; flag
                                                                              dependent sections
                                                                              where needed.

  POST      /wedding-setup/sessions/{session_id}/sections/{section}/approve   Approve specified
                                                                              section only.

  POST      /wedding-setup/sessions/{session_id}/sections/{section}/skip      Mark optional section
                                                                              SKIPPED.

  POST      /wedding-setup/sessions/{session_id}/finalise                     Create approved records
                                                                              via shared wedding
                                                                              creation logic and
                                                                              owning services;
                                                                              idempotent/resumable.

  DELETE    /wedding-setup/sessions/{session_id}                              Delete unfinished
                                                                              session only; completed
                                                                              session cannot be
                                                                              deleted.
  ---------------------------------------------------------------------------------------------------

## 4.6 AI setup contract rules

-   Section statuses: NOT_GENERATED, DRAFT, NEEDS_REVIEW, APPROVED,
    SKIPPED. Approval is independent per section.

-   Finalise requires core information and approval of each section
    intended for creation. Skipped optional sections do not block.

-   AI output is untrusted and Pydantic-validated. AI cannot execute
    arbitrary SQL, change roles, send invitations, publish websites or
    bypass authorisation.

-   No invented exact date/deadline for approximate/undecided wedding
    dates. Traditions/customs are optional and presented as variable.

-   Wedding Service coordinates persisted workflow states PENDING,
    IN_PROGRESS, COMPLETED, FAILED_RETRYABLE and records step
    completion.

-   Steps: wedding, events, tasks, vendor-category suggestions, website
    content and albums. Owning services create their own records.
    Completed steps are not repeated; failed steps retry safely;
    organiser can resume incomplete workflow.

-   Initial provider: Gemini development adapter; Groq/OpenRouter may be
    evaluated. Review provider privacy/retention terms and disclose
    processing before real-user launch.

# 5. Planning Service

Owns events, tasks, assignments and event-specific schedules.

## 5.1 Events

  ----------------------------------------------------------------------------------
  Method          Endpoint                                   Purpose / key contract
  --------------- ------------------------------------------ -----------------------
  POST            /weddings/{wedding_id}/events              Create event; name
                                                             required; description,
                                                             start/end,
                                                             venue/location and
                                                             dress code optional.

  GET             /weddings/{wedding_id}/events              List/filter: page,
                                                             limit, search,
                                                             date_from/to,
                                                             include_deleted,
                                                             sort_by/order.

  GET             /weddings/{wedding_id}/events/{event_id}   Get active event;
                                                             deleted event requires
                                                             include_deleted=true.

  PATCH           /weddings/{wedding_id}/events/{event_id}   Partial update active
                                                             event; end requires
                                                             start and must be
                                                             later.

  DELETE          /weddings/{wedding_id}/events/{event_id}   Soft-delete event;
                                                             preserve
                                                             tasks/schedule; no
                                                             restore.
  ----------------------------------------------------------------------------------

-   Event names are configurable, not hardcoded. Start/end optional; end
    cannot exist without start and must be later if both exist.

-   Default list sort: start datetime ascending, undated events last.
    include_deleted=true includes deleted events.

-   Deleted event GET requires include_deleted=true; update only active
    event; repeated delete 404.

-   Tasks/schedules linked to deleted events remain visible with
    deleted-event indication; no new associations to deleted events.

-   Writes require Events MANAGE; reads VIEW or MANAGE.

## 5.2 Tasks and assignments

  -------------------------------------------------------------------------------------------
  Method       Endpoint                                             Purpose / key contract
  ------------ ---------------------------------------------------- -------------------------
  POST         /weddings/{wedding_id}/tasks                         Create task; title
                                                                    required; optional
                                                                    description, event_id,
                                                                    due_date, priority,
                                                                    status,
                                                                    assigned_organiser_ids.

  GET          /weddings/{wedding_id}/tasks                         List/filter: page, limit,
                                                                    search, status, priority,
                                                                    event_id, assigned_to,
                                                                    due-date range, overdue,
                                                                    sorting, include_deleted.

  GET          /weddings/{wedding_id}/tasks/{task_id}               Retrieve task, event
                                                                    context, assignments and
                                                                    lifecycle timestamps.

  PATCH        /weddings/{wedding_id}/tasks/{task_id}               Partial task update;
                                                                    assignments use separate
                                                                    endpoint.

  DELETE       /weddings/{wedding_id}/tasks/{task_id}               Soft-delete task;
                                                                    preserve
                                                                    assignment/history.

  PUT          /weddings/{wedding_id}/tasks/{task_id}/assignments   Replace full assignment
                                                                    list; empty list removes
                                                                    all.
  -------------------------------------------------------------------------------------------

-   Priority LOW/MEDIUM/HIGH, default MEDIUM. Status
    PENDING/IN_PROGRESS/COMPLETED, default PENDING.

-   Event must be active and same wedding. Assigned organisers must be
    active same-wedding members; self-assignment allowed.

-   Task plus initial assignments is one transaction. PUT assignments
    validates all IDs before commit and replaces full list.

-   Writes require Tasks MANAGE; reads VIEW or MANAGE.

## 5.3 Event schedules

  -------------------------------------------------------------------------------------------------------------------------
  Method   Endpoint                                                          Purpose / key contract
  -------- ----------------------------------------------------------------- ----------------------------------------------
  POST     /weddings/{wedding_id}/events/{event_id}/schedule                 Create item; title/start_datetime required;
                                                                             description/end_datetime/location/sort_order
                                                                             optional.

  GET      /weddings/{wedding_id}/events/{event_id}/schedule                 List chronologically; deleted parent requires
                                                                             include_deleted=true and is read-only.

  PATCH    /weddings/{wedding_id}/events/{event_id}/schedule/{schedule_id}   Update active schedule item under active
                                                                             parent event.

  DELETE   /weddings/{wedding_id}/events/{event_id}/schedule/{schedule_id}   Soft-delete schedule item.

  PUT      /weddings/{wedding_id}/events/{event_id}/schedule                 Bulk replace: create/update supplied items,
                                                                             soft-delete omitted active items.
  -------------------------------------------------------------------------------------------------------------------------

-   If end_datetime supplied, it must be later than start_datetime.

-   Bulk PUT validates every schedule ID belongs to the specified event
    and wedding, validates entire payload, then commits one transaction.

-   Deleted schedule items remain deleted. Deleted parent event is
    read-only. Writes require Events MANAGE; reads VIEW or MANAGE.

# 6. Guest Service

Owns household contacts, household-event associations, guest invitation
tokens and RSVP. Guests have no account/login.

## 6.1 Household management

  --------------------------------------------------------------------------------
  Method          Endpoint                                   Purpose / key
                                                             contract
  --------------- ------------------------------------------ ---------------------
  POST            /weddings/{wedding_id}/guests              Create household;
                                                             contact_name,
                                                             permitted_attendees
                                                             and email or phone
                                                             required;
                                                             invited_event_ids
                                                             optional.

  GET             /weddings/{wedding_id}/guests              List/filter: page,
                                                             limit, search
                                                             name/email/phone,
                                                             event_id,
                                                             rsvp_status, sorting,
                                                             include_deleted.

  GET             /weddings/{wedding_id}/guests/{guest_id}   Retrieve household,
                                                             invited events, RSVP
                                                             and invitation
                                                             status/expiry; never
                                                             raw token.

  PATCH           /weddings/{wedding_id}/guests/{guest_id}   Partial update; at
                                                             least one contact
                                                             method remains; event
                                                             association is
                                                             separate.

  DELETE          /weddings/{wedding_id}/guests/{guest_id}   Soft-delete; revoke
                                                             active token,
                                                             preserve
                                                             RSVP/history, cancel
                                                             eligible pending
                                                             email jobs.
  --------------------------------------------------------------------------------

-   Household is one primary contact, not individual family-member
    records. At least one email/phone; permitted_attendees \>=1.

-   Event IDs must be active same-wedding events. Household plus initial
    associations is one transaction.

-   Cannot reduce permitted count below accepted RSVP count; return 409
    and protect against concurrent RSVP.

-   Guest writes require Guests MANAGE; reads VIEW or MANAGE.

## 6.2 Household event associations

  --------------------------------------------------------------------------------
  Method         Endpoint                                          Purpose / key
                                                                   contract
  -------------- ------------------------------------------------- ---------------
  PUT            /weddings/{wedding_id}/guests/{guest_id}/events   Replace
                                                                   complete
                                                                   invited-event
                                                                   list; empty
                                                                   removes all;
                                                                   active
                                                                   same-wedding
                                                                   events only;
                                                                   does not
                                                                   auto-send
                                                                   email.

  --------------------------------------------------------------------------------

## 6.3 Invitation token administration

  ---------------------------------------------------------------------------------------
  Method      Endpoint                                                     Purpose / key
                                                                           contract
  ----------- ------------------------------------------------------------ --------------
  POST        /weddings/{wedding_id}/guests/{guest_id}/invitation          Generate
                                                                           token/link;
                                                                           queue email if
                                                                           email exists;
                                                                           otherwise
                                                                           return
                                                                           shareable link
                                                                           for manual
                                                                           sharing.

  POST        /weddings/{wedding_id}/guests/{guest_id}/invitation/resend   Revoke old
                                                                           token, cancel
                                                                           eligible
                                                                           pending
                                                                           old-token
                                                                           email jobs,
                                                                           issue new
                                                                           token/expiry
                                                                           and queue
                                                                           email.

  DELETE      /weddings/{wedding_id}/guests/{guest_id}/invitation          Revoke active
                                                                           invitation
                                                                           without
                                                                           deleting
                                                                           household or
                                                                           RSVP.
  ---------------------------------------------------------------------------------------

## 6.4 Manual RSVP reminders

| Method | Endpoint | Purpose / key contract |
| --- | --- | --- |
| POST | /weddings/{wedding_id}/guests/rsvp-reminders | Manually queue reminders for `selected_pending_household_ids` or `all_pending=true`; Guests MANAGE and Idempotency-Key required. |
| GET | /weddings/{wedding_id}/guests/rsvp-reminders/{reminder_request_id} | Retrieve the request's selected/eligible/queued counts and Notification-Service processing summary. |

- This is organiser-initiated only; no automated scheduled RSVP reminder
  campaigns exist in V1.
- For selected requests, every supplied household must be active, in the same
  wedding and still RSVP-pending; otherwise reject the request without queuing
  any reminder. `all_pending=true` resolves the eligible set atomically.
- Guest Service creates a durable request correlation ID and calls Notification
  Service through its authenticated internal API with idempotent per-recipient
  jobs. The response returns `reminder_request_id` and the number queued;
  Guest Service never writes Notification Service tables directly.
- The status endpoint obtains queued/sent/failed processing totals by the
  correlation ID. Reusing an Idempotency-Key with the same request returns the
  original request; a different payload returns 409.

## 6.5 Guest invitation expiry

-   No invited events or undecided event dates: 30 days from generation.

-   Otherwise G=generation and L=latest invited event start. Candidate
    expiry=min(G+30 days,L-5 days) if at least 24 hours of validity;
    otherwise expiry=latest event start.

-   Calculate in wedding timezone and store UTC. Changing invited events
    does not silently change existing expiry; reissue to recalculate.

-   Token is high entropy, hashed, scoped and revocable. GET does not
    consume it. Retrieval and RSVP submission both reject revoked or expired
    tokens using the persisted `expires_at` calculated by these rules.

## 6.6 Public invitation and RSVP

  -------------------------------------------------------------------------
  Method              Endpoint                          Purpose / key
                                                        contract
  ------------------- --------------------------------- -------------------
  GET                 /guest-invitations/{token}        Return intended
                                                        wedding/household
                                                        info, invited
                                                        events, permitted
                                                        count, existing
                                                        RSVP and
                                                        expiry/validity; no
                                                        raw token.

  POST                /guest-invitations/{token}/rsvp   Create/update one
                                                        household RSVP
                                                        while token valid
                                                        and RSVP
                                                        submissions open.
  -------------------------------------------------------------------------

## 6.7 Guest statistics

  -------------------------------------------------------------------------------------
  Method        Endpoint                                   Purpose / key contract
  ------------- ------------------------------------------ ----------------------------
  GET           /weddings/{wedding_id}/guests/statistics   Return active household
                                                           total, sent/delivered
                                                           invitations,
                                                           accepted/declined/pending,
                                                           confirmed headcount and
                                                           RSVP-open flag.

  -------------------------------------------------------------------------------------

-   One overall household RSVP, not event-specific. ACCEPTED
    attendee_count 1..permitted; DECLINED count 0.

-   Guest may update RSVP repeatedly while token valid and wedding-level
    rsvp_submissions_open=true. Closing blocks new/updated responses but
    preserves existing.

-   RSVP closure is owned by Wedding Service setting; Guests MANAGE
    controls setting; Guest Service checks it.

-   Pending means active household with no RSVP response. Expired
    invitation does not change RSVP status.

-   invitations_sent means provider accepted; invitations_delivered
    means provider confirmed delivery. Manual WhatsApp shares are not
    email sends.

-   RSVP, activity and confirmation-email outbox record commit in one
    local transaction. Email failure does not roll back RSVP.

# 7. Vendor & Expense Service

Owns platform-curated vendor directory, wedding-specific vendor
selections and expense records. No vendor self-registration, bookings,
commissions or marketplace transactions.

## 7.1 Global vendor directory and discovery

  ----------------------------------------------------------------------------------
  Method          Endpoint               Purpose / key contract
  --------------- ---------------------- -------------------------------------------
  GET             /vendors               List active global vendors;
                                         page/limit/search/category/location/sort.

  GET             /vendors/{vendor_id}   Retrieve active global vendor;
                                         inactive/missing returns 404.

  GET             /vendors/discover      Google Places discovery by
                                         query/location/category; results are
                                         discovery data, not permanent vendor
                                         records.
  ----------------------------------------------------------------------------------

## 7.2 Wedding-specific vendors

  -------------------------------------------------------------------------------------------
  Method      Endpoint                                                    Purpose / key
                                                                          contract
  ----------- ----------------------------------------------------------- -------------------
  POST        /weddings/{wedding_id}/vendors                              Add
                                                                          GLOBAL_DIRECTORY
                                                                          with vendor_id OR
                                                                          GOOGLE_PLACES with
                                                                          place_id; optional
                                                                          status,
                                                                          quoted_amount,
                                                                          notes and
                                                                          event_ids.

  GET         /weddings/{wedding_id}/vendors                              List/filter: page,
                                                                          limit, search,
                                                                          category, status,
                                                                          event, source,
                                                                          sorting and
                                                                          include_deleted.

  GET         /weddings/{wedding_id}/vendors/{wedding_vendor_id}          Retrieve
                                                                          global/Places
                                                                          reference and
                                                                          wedding-private
                                                                          status, amount,
                                                                          notes, events and
                                                                          lifecycle.

  PATCH       /weddings/{wedding_id}/vendors/{wedding_vendor_id}          Update status,
                                                                          quoted_amount or
                                                                          notes; event
                                                                          associations use
                                                                          separate endpoint.

  DELETE      /weddings/{wedding_id}/vendors/{wedding_vendor_id}          Soft-delete
                                                                          wedding-specific
                                                                          association;
                                                                          preserve
                                                                          expenses/history;
                                                                          does not deactivate
                                                                          global vendor.

  PUT         /weddings/{wedding_id}/vendors/{wedding_vendor_id}/events   Replace complete
                                                                          event association
                                                                          list; active
                                                                          same-wedding events
                                                                          only.
  -------------------------------------------------------------------------------------------

-   Source GLOBAL_DIRECTORY requires vendor_id; GOOGLE_PLACES requires
    place_id. Reject conflicting identifiers.

-   Prevent duplicate global vendor association within one wedding via
    database uniqueness constraint.

-   One primary category per vendor. Statuses SHORTLISTED, CONTACTED,
    CONFIRMED, NOT_SELECTED; flexible transitions allowed.

-   quoted_amount is optional and non-negative or null. Wedding-specific
    price/status/notes remain private.

-   Google Places attribution, retention, field masks, quotas and
    billing rules must be observed.

## 7.3 Expenses

  --------------------------------------------------------------------------------------------
  Method       Endpoint                                       Purpose / key contract
  ------------ ---------------------------------------------- --------------------------------
  POST         /weddings/{wedding_id}/expenses                Create; title, amount\>0,
                                                              required_date required;
                                                              description/vendor_id/event_id
                                                              optional.

  GET          /weddings/{wedding_id}/expenses                List/filter: page, limit,
                                                              search, vendor/event, amount
                                                              range, required-date range,
                                                              sorting, include_deleted.

  GET          /weddings/{wedding_id}/expenses/{expense_id}   Retrieve expense and
                                                              vendor/event context.

  PATCH        /weddings/{wedding_id}/expenses/{expense_id}   Partial update; amount remains
                                                              \>0; required_date non-null;
                                                              vendor/event may be cleared.

  DELETE       /weddings/{wedding_id}/expenses/{expense_id}   Soft-delete; exclude from normal
                                                              list and totals.

  GET          /weddings/{wedding_id}/expenses/summary        Active expense count and total
                                                              committed amount; filters
                                                              vendor/event/date.
  --------------------------------------------------------------------------------------------

-   Amounts are estimated/agreed/committed costs, not
    instalments/payments.

-   Use PostgreSQL NUMERIC(12,2), never floating point. Include currency
    code in expense/summary responses; no currency conversion V1.

-   Default currency INR remains a proposed candidate, not an explicitly
    confirmed decision. Confirm INR-only versus multi-currency before
    production.

-   Associated vendor/event must be active and same wedding. Deleted
    expenses excluded from totals.

-   Vendor/Expense writes require corresponding module MANAGE; reads
    VIEW or MANAGE.

# 8. Media Service

Owns website configuration and snapshots, media metadata, albums/photos,
guest media tokens and livestreams. Binaries are in private S3;
PostgreSQL stores metadata and keys.

## 8.1 Website

  ---------------------------------------------------------------------------------------
  Method        Endpoint                                   Purpose / key contract
  ------------- ------------------------------------------ ------------------------------
  GET           /weddings/{wedding_id}/website             Retrieve config; 404 if not
                                                           initialised; Website VIEW or
                                                           MANAGE.

  POST          /weddings/{wedding_id}/website             Explicitly initialise one
                                                           DRAFT config after obtaining
                                                           the Wedding-Service-owned slug;
                                                           selected/default theme;
                                                           duplicate 409.

  PATCH         /weddings/{wedding_id}/website             Update draft theme/content
                                                           only; core wedding details
                                                           remain Wedding Service-owned.

  GET           /weddings/{wedding_id}/website/preview     Preview latest draft/current
                                                           selected event data; VIEW or
                                                           MANAGE; no publish.

  POST          /weddings/{wedding_id}/website/publish     Validate essential fields;
                                                           snapshot draft plus selected
                                                           event data; publish.

  POST          /weddings/{wedding_id}/website/unpublish   Unpublish public page;
                                                           preserve draft and published
                                                           snapshot.

  GET           /public/weddings/{slug}                    Unauthenticated published
                                                           snapshot only; 404
                                                           invalid/unpublished/inactive
                                                           wedding.
  ---------------------------------------------------------------------------------------

-   Themes TRADITIONAL_INDIAN, MINIMALIST, ROYAL_LUXURY; English-only
    V1.

-   Website is not auto-created. Initial status DRAFT; unique slug;
    default theme may be TRADITIONAL_INDIAN.

-   Wedding Service owns `wedding.weddings.slug`. Before Media Service creates
    the website record, it calls the authenticated internal Wedding-Service
    slug operation. That operation returns an existing reserved slug or
    transactionally generates and reserves a unique one; repeated calls are
    idempotent. Media Service never creates or updates the slug directly.

-   Published edits change draft only. Public website serves old
    published snapshot until explicit republish.

-   Publish creates an immutable published snapshot containing the
    selected theme, website draft content, publishable event data from
    Planning Service and references to the website image assets selected
    for publication.

-   Public endpoint serves the persisted published snapshot only; it
    does not fetch live events or read the current draft. Draft edits
    and later event changes remain private until an explicit republish.

-   Unpublishing makes the public route return 404 while preserving the
    draft and last snapshot for a later republish.

-   Unpublish returns public route 404 but preserves draft/snapshot.
    Public response never leaks internal IDs, organiser data or private
    notes.

-   Recommended public-site image delivery is CloudFront with OAC and a
    dedicated `website-public/` prefix in the private S3 bucket; this is
    pending approval. Public website images use stable CloudFront URLs.
    Private gallery images remain outside this prefix and require
    gallery token validation followed by short-lived viewing URLs.

## 8.2 Albums

  ---------------------------------------------------------------------------------
  Method          Endpoint                                   Purpose / key contract
  --------------- ------------------------------------------ ----------------------
  POST            /weddings/{wedding_id}/albums              Create empty album;
                                                             name required;
                                                             description/event_id
                                                             optional; event active
                                                             same wedding.

  GET             /weddings/{wedding_id}/albums              List with pagination,
                                                             search, event/general
                                                             filter, sorting,
                                                             include_deleted.

  GET             /weddings/{wedding_id}/albums/{album_id}   Retrieve album, event
                                                             and photo counts;
                                                             include_deleted
                                                             required for deleted.

  PATCH           /weddings/{wedding_id}/albums/{album_id}   Update name,
                                                             description or event
                                                             association; event
                                                             active same wedding.

  DELETE          /weddings/{wedding_id}/albums/{album_id}   Soft-delete; photos
                                                             retained but hidden
                                                             publicly.
  ---------------------------------------------------------------------------------

## 8.3 Organiser photo upload and management

  -------------------------------------------------------------------------------------------------------
  Method    Endpoint                                                                Purpose / key
                                                                                    contract
  --------- ----------------------------------------------------------------------- ---------------------
  POST      /weddings/{wedding_id}/albums/{album_id}/photos/upload-url              Authorise
                                                                                    organiser/album and
                                                                                    issue short-lived S3
                                                                                    upload URL; Gallery
                                                                                    MANAGE.

  POST      /weddings/{wedding_id}/albums/{album_id}/photos/complete                Verify S3 object and
                                                                                    create metadata;
                                                                                    organiser upload
                                                                                    auto-approved.

  GET       /weddings/{wedding_id}/albums/{album_id}/photos                         Paginated list/status
                                                                                    filter; VIEW or
                                                                                    MANAGE.

  GET       /weddings/{wedding_id}/albums/{album_id}/photos/{photo_id}              Retrieve photo
                                                                                    metadata/moderation
                                                                                    status.

  PATCH     /weddings/{wedding_id}/albums/{album_id}/photos/{photo_id}/moderation   MANAGE;
                                                                                    approve/reject
                                                                                    permitted moderation
                                                                                    transitions.

  DELETE    /weddings/{wedding_id}/albums/{album_id}/photos/{photo_id}              Soft-delete and hide
                                                                                    immediately; preserve
                                                                                    metadata/S3 object.
  -------------------------------------------------------------------------------------------------------

-   JPEG, PNG, WebP; proposed max 10 MB. Validate signature/MIME, size,
    decodeability, dimensions/pixel count; re-encode and strip EXIF
    where appropriate.

-   Direct browser-to-private-S3 upload; FastAPI does not proxy binary.
    Proposed presigned URL expiry five minutes. Verify object before
    metadata creation.

-   Organiser uploads APPROVED; guest uploads PENDING_APPROVAL.
    Transitions: PENDING_APPROVAL→APPROVED, PENDING_APPROVAL→REJECTED,
    APPROVED→REJECTED; rejected cannot be reapproved through this
    endpoint.

-   Guest upload completion revalidates token revocation state, wedding
    scope, album active/ownership and quota. The guest upload token
    itself has no automatic expiry; the individual S3 presigned upload
    URL remains short-lived. Invalid completion creates no accessible
    metadata; S3 object eligible for cleanup.

-   One guest upload token allows 20 successfully completed photos; URL
    generation does not consume count; enforce atomically.

## 8.4 Guest upload link and public upload

  -------------------------------------------------------------------------------------
  Method        Endpoint                                           Purpose / key
                                                                   contract
  ------------- -------------------------------------------------- --------------------
  POST          /public/guest-media/{token}/upload-url             Guest requests
                                                                   upload URL with
                                                                   active token.

  POST          /public/guest-media/{token}/complete               Revalidate
                                                                   token/album/event,
                                                                   verify object and
                                                                   create
                                                                   PENDING_APPROVAL
                                                                   photo.

  POST          /weddings/{wedding_id}/gallery/guest-upload-link   Generate/replace one
                                                                   wedding-wide upload
                                                                   link; revoke
                                                                   previous; Gallery
                                                                   MANAGE.

  GET           /weddings/{wedding_id}/gallery/guest-upload-link   Return active
                                                                   status, share URL/QR
                                                                   info, created time,
                                                                   completed count and
                                                                   limit; VIEW/MANAGE.

  DELETE        /weddings/{wedding_id}/gallery/guest-upload-link   Revoke immediately;
                                                                   preserve uploaded
                                                                   photos and pending
                                                                   moderation.
  -------------------------------------------------------------------------------------

## 8.5 Guest gallery viewing

  ----------------------------------------------------------------------------------
  Method         Endpoint                                         Purpose / key
                                                                  contract
  -------------- ------------------------------------------------ ------------------
  POST           /weddings/{wedding_id}/gallery/guest-view-link   Generate/replace
                                                                  one wedding-wide
                                                                  viewing link;
                                                                  revoke prior.

  GET            /weddings/{wedding_id}/gallery/guest-view-link   Retrieve active
                                                                  status/share
                                                                  URL/creation
                                                                  information; VIEW
                                                                  or MANAGE.

  DELETE         /weddings/{wedding_id}/gallery/guest-view-link   Revoke viewing
                                                                  token; preserve
                                                                  photos.

  GET            /public/guest-gallery/{token}                    Public flat
                                                                  paginated photo
                                                                  collection with
                                                                  album metadata and
                                                                  short-lived
                                                                  viewing URLs.
  ----------------------------------------------------------------------------------

-   Guest upload and viewing tokens are separate; one active token of
    each type per wedding; neither has automatic expiry; manual
    revoke/replace only. Store token hashes, not raw tokens. The
    short-lived S3 presigned URLs issued for individual uploads or image
    viewing have their own expiry and are not the guest token.

-   Gallery returns only approved, non-deleted photos in active albums;
    exclude pending/rejected/deleted and photos in deleted albums.

-   Flat photo pagination; each photo includes album metadata; frontend
    groups current page by album.

-   Revocation prevents new viewing URL generation immediately.
    Previously issued five-minute URLs may remain usable until expiry.

-   General-album/event association must be explicitly validated. If
    both event and event-linked album are selected, they must match.

## 8.6 Event livestream

  ------------------------------------------------------------------------------------------
  Method       Endpoint                                              Purpose / key contract
  ------------ ----------------------------------------------------- -----------------------
  POST         /weddings/{wedding_id}/events/{event_id}/livestream   Create one active
                                                                     YouTube livestream for
                                                                     active same-wedding
                                                                     event.

  GET          /weddings/{wedding_id}/events/{event_id}/livestream   Retrieve active
                                                                     livestream; deleted
                                                                     event requires
                                                                     include_deleted=true.

  PATCH        /weddings/{wedding_id}/events/{event_id}/livestream   Update URL on same
                                                                     record; active event
                                                                     only.

  DELETE       /weddings/{wedding_id}/events/{event_id}/livestream   Soft-delete; hide from
                                                                     public website; later
                                                                     replacement allowed.
  ------------------------------------------------------------------------------------------

-   Accept youtube.com/watch?v=VIDEO_ID, youtube.com/live/VIDEO_ID and
    youtu.be/VIDEO_ID only; validate host/path and extract ID.

-   Gallery MANAGE for writes; VIEW or MANAGE for reads. No native
    streaming.

# 9. Notification Service

Owns email jobs, attempts, delivery events and worker. Internal
endpoints require dedicated service credentials; frontend cannot call
them.

## 9.1 Internal endpoints

  --------------------------------------------------------------------------------------------------------
  Method   Endpoint                                                      Purpose / key contract
  -------- ------------------------------------------------------------- ---------------------------------
  POST     /internal/notifications/email-jobs                            Create job: recipient_email,
                                                                         predefined template_key,
                                                                         validated template_data, optional
                                                                         wedding_id; return
                                                                         job_id/PENDING.

  GET      /internal/notifications/email-jobs/{job_id}                   Retrieve own job; originating
                                                                         service only; redact sensitive
                                                                         template data.

  GET      /internal/notifications/email-jobs                            List own jobs;
                                                                         page/limit/status/template/date
                                                                         filters/sort and optional
                                                                         originating_request_id.

  GET      /internal/notifications/email-jobs/{job_id}/delivery-events   Get own job's delivery history.

  POST     /internal/notifications/email-jobs/{job_id}/cancel            Cancel PENDING or request
                                                                         cancellation for PROCESSING.

  POST     /webhooks/email-delivery/{provider}                           Verify provider authenticity;
                                                                         process events idempotently.

  GET      /health                                                       Unauthenticated basic health;
                                                                         DB/worker check; 200 healthy or
                                                                         503 unavailable.
  --------------------------------------------------------------------------------------------------------

## 9.2 Worker, templates and delivery

-   Templates are code-managed/version-controlled; no arbitrary
    HTML/subject or template CRUD. Initial keys: EMAIL_VERIFICATION,
    PASSWORD_RESET, ORGANISER_INVITATION, GUEST_INVITATION,
    RSVP_CONFIRMATION, RSVP_REMINDER.

-   Job states PENDING, PROCESSING, SENT, FAILED, CANCELLED. SENT means
    provider accepted, not inbox delivery.

-   Processing and delivery statuses are separate;
    e.g. processing_status=SENT, delivery_status=BOUNCED.

-   Worker defaults: one worker, poll every five seconds, claim up to
    five per batch, sequential processing, up to three retries after
    initial attempt with increasing delays. Tune after testing.

-   Claim with SELECT FOR UPDATE SKIP LOCKED or equivalent; recover
    expired worker leases. Permanent failures are not retried.

-   Provider events: DELIVERY, BOUNCE, COMPLAINT, REJECT,
    RENDERING_FAILURE. Verify webhook, match provider message ID, store
    history and deduplicate.

-   Amazon SES production behind EmailProvider abstraction; Resend may
    be development adapter. Jobs, attempts and delivery events retained
    12 months.

## 9.3 Cancellation contract

  ---------------------------------------------------------------------
  Job status                         Behaviour
  ---------------------------------- ----------------------------------
  PENDING                            Cancel immediately; worker skips.

  PROCESSING                         Set cancel_requested_at; return
                                     202 Accepted; current attempt may
                                     finish.

  SENT                               Cannot cancel; provider-accepted
                                     email cannot be recalled.

  FAILED                             Cannot cancel; terminal failure.

  CANCELLED                          Already cancelled; stable
                                     already-cancelled result or
                                     consistent conflict.
  ---------------------------------------------------------------------

If PROCESSING attempt fails after cancellation request, suppress further
retries. If provider accepts, email may still be delivered. The persisted
`cancel_requested_at` field lets the worker cancel a still-unsent PROCESSING
job before provider submission; no additional cancellation-request status is
introduced.

## 9.4 Internal authentication

-   Unique dedicated service token per caller; Authorization: Bearer
    `<SERVICE_TOKEN>`{=html}. Validate identity and per-service
    permissions.

-   Store in environment/secrets management; rotate/revoke; never expose
    to frontend or commit to Git.

-   Invalid credentials 401; authenticated service without permission

    403. 

# 10. Cross-service consistency and workflows

## 10.1 Local transactions and recoverable workflows

-   Operations belonging to one business operation within the same
    service/database execute in one transaction; critical failure rolls
    back.

-   Do not attempt a distributed database transaction across services.

-   Cross-service side effects use persisted, recoverable workflows with
    bounded retries and idempotent execution.

-   AI finalisation coordinator in Wedding Service tracks steps and
    statuses; owning services write their own data.

## 10.2 Transactional outbox

-   Where a local business action requires cross-service side effect,
    save business record and outbox event in same local transaction.

-   Activity record participates in same transaction where
    required/practical.

-   Each service owns its outbox records; background dispatcher retries
    transient failures.

-   Notification job creation is idempotent; retries must not create
    duplicate email jobs.

-   No Kafka, RabbitMQ, Redis or general event bus in V1.

## 10.3 Activity and tracing

-   Originating service writes activity through shared helper. Fields:
    activity_id, wedding_id, actor_id/type, action, entity_type/id,
    concise summary, timestamp.

-   Guest actions attributed to household/invitation context, not fake
    organiser identity.

-   Activity append-only and retained for lifetime of wedding. Service
    DB roles receive only required activity-write permission.

-   If required audit record cannot be committed with business action,
    do not silently report success.

-   Generate request_id at edge/first app service; propagate through
    internal calls/logs; include in errors; new ID per independent
    request.

## 10.4 Internal dependency failures

-   Owning service validates its own references. Never directly query
    another service schema.

-   Required validation failure prevents operation. Required service
    unavailable returns controlled 503.

-   Use bounded timeouts and limited retries for transient errors. No
    indefinite waiting or silent deferred creation after failed
    validation.

# 11. Database, file and operational security

## 11.1 Database baseline

  Schema           Tables   Owner
  ---------------- -------- ---------------------
  identity         4        Identity
  wedding          9        Wedding
  planning         4        Planning
  guest            5        Guest
  vendor_expense   5        Vendor & Expense
  media            6        Media
  notification     3        Notification/worker
  activity         1        Cross-cutting

37 tables across eight PostgreSQL schemas. UUID primary keys;
snake_case; TIMESTAMPTZ for instants; DATE for calendar dates;
NUMERIC(12,2) for money; JSONB only for validated flexible structures.
SQLAlchemy ORM and Alembic migrations.

## 11.2 Database constraints

-   Same-wedding composite FKs within schema wherever practical;
    cross-service references validated through service APIs/contracts.

-   Partial unique index: one active membership per organiser.
    Four-Admin cap enforced transactionally with locking/serialisation,
    not a simple CHECK constraint.

-   Enforce one active guest invitation token per household; one website
    per wedding; one active guest upload link and view link per wedding;
    one active livestream per event.

-   Prevent duplicate global vendor association per wedding. One RSVP
    per household; accepted attendee count 1..permitted protected
    transactionally.

-   Soft-deleted records excluded from normal queries;
    include_deleted=true only where endpoint supports it.

## 11.3 S3 and upload security

-   Private S3; server-generated object keys scoped under
    weddings/{wedding_id}/; no permanent public S3 URLs.

-   Direct browser-to-S3 upload; short-lived pre-signed URLs (proposed
    five minutes); backend verifies object before metadata creation.

-   Validate actual file signature/MIME, size, decoding, dimensions and
    pixel count; re-encode/strip EXIF where appropriate.

-   Abandoned/unreferenced uploads eligible for cleanup. Photo metadata
    soft-delete hides immediately; underlying S3 object retained in V1.

-   Do not log passwords, JWTs, refresh/guest tokens, cookies,
    pre-signed URLs or credentials.

## 11.4 Rate limits

  Operation               Initial limit
  ----------------------- -------------------------------
  General API             60/minute/IP
  Login                   5/minute/IP
  Registration            5/hour/IP
  Forgot password         3/15 minutes/account or email
  Organiser invitations   10/minute/organiser
  RSVP                    5/minute/token
  Upload URL generation   10/minute/token

Initial thresholds are tuneable. Apply Nginx IP throttling plus
application-level account/token/action limits. Return 429 and
Retry-After where applicable.

## 11.5 CORS, docs and network

-   CORS explicit allowlist; development http://localhost:5173;
    production origin configured.

-   Methods GET/POST/PUT/PATCH/DELETE/OPTIONS. Headers Authorization,
    Content-Type, Idempotency-Key. Credentials only where refresh cookie
    requires; CSRF protection.

-   OpenAPI 3.x; FastAPI Swagger UI/ReDoc restricted or disabled in
    production.

-   Expose only Nginx 80/443 publicly; FastAPI private Docker network;
    HTTPS; EC2 IAM role for AWS; secrets outside Git/frontend; separate
    service DB roles and migration credentials.

# 12. Validation corrections and open decisions

  ---------------------------------------------------------------------
  Correction group                   Approved content incorporated
  ---------------------------------- ----------------------------------
  1 Identity lifecycle               Email-change verification
                                     endpoints; refresh-token reuse
                                     detection; logout access-token
                                     behaviour documented.

  2 Permissions/membership           Nine modules; Admin full access;
                                     invitation management Admin-only;
                                     Activity append-only; Admin cap
                                     concurrency-safe.

  3 AI finalisation                  Persisted workflow states/steps;
                                     idempotent retry/resume; owning
                                     services create own records.

  4 Transactions/outbox              Local transaction boundaries;
                                     transactional outbox; dispatcher
                                     retry; idempotent notification
                                     jobs.

  5 Guest/RSVP                       Sent vs delivered metrics; pending
                                     definition; RSVP+activity+outbox
                                     atomicity.

  6 Vendor/expense                   Source-specific vendor request;
                                     duplicate protection; NUMERIC
                                     money; currency in responses.

  7 Media                            Published asset delivery; upload
                                     completion revalidation;
                                     album/event validation; flat
                                     pagination; revocation semantics.

  8 Notification                     Processing cancellation contract;
                                     independent delivery status.

  9 Shared contracts                 Date/time format; sort allowlists;
                                     common errors; internal
                                     validation; log security.

  10 Idempotency/constraints         Sensitive POST matrix; database
                                     constraints; soft-delete and
                                     operational safeguards.
  ---------------------------------------------------------------------

## 12.1 Idempotency matrix

  Operation                            Service
  ------------------------------------ ------------------
  Wedding creation                     Wedding
  Organiser invitation create/resend   Wedding
  AI wedding finalisation              Wedding
  RSVP submit/update                   Guest
  Guest invitation generate/resend     Guest
  Guest upload completion              Media
  Organiser photo upload completion    Media
  Website publish                      Media
  Guest upload/view link generation    Media
  Vendor creation                      Vendor & Expense
  Expense creation                     Vendor & Expense
  Notification job creation            Notification

-   Idempotency-Key scoped to authenticated organiser or internal
    service plus operation. Same key+same fingerprint returns original
    response; same key+different payload returns 409; retention 24
    hours.

-   GET does not need idempotency key. PUT/DELETE should be naturally
    idempotent where semantics allow.

## 12.2 Explicit open decisions

  ---------------------------------------------------------------------
  Area                               Status
  ---------------------------------- ----------------------------------
  Default wedding currency           INR is the V1 default.
                                     `currency_code` is persisted per
                                     wedding and returned by expense
                                     APIs; no currency conversion is
                                     performed. Confirm whether the V1
                                     UI allows non-INR selection before
                                     production.

  Public website image delivery      Recommended: CloudFront + OAC with
                                     a dedicated website-public/ prefix
                                     in the private S3 bucket. Stable
                                     public URLs; free within monthly
                                     allowance, billable beyond it.
                                     Pending approval.

  General album event association    Define whether guest may
                                     independently associate event to
                                     general album; enforce exact
                                     chosen rule.

  Field-level request/response       Endpoint paths and functional
  schemas                            contracts are consolidated;
                                     generate final Pydantic/OpenAPI
                                     schemas and verify during
                                     implementation.

  Google Places retention            Check current field-specific
                                     storage, attribution, quota and
                                     billing terms at implementation.

  AI provider privacy                Review provider
                                     retention/training/processing
                                     terms, disclosure and consent
                                     before real-user launch.

  AI session retention               Define retention for completed
                                     transcripts and wedding-linked
                                     setup sessions.

  Activity metadata                  Define allowed metadata keys per
                                     action and sanitisation.

  Production DB/backups              Choose provider, backup cadence
                                     and point-in-time recovery before
                                     production.

  Idempotency persistence            Implement durable per-service
                                     records/indexes with 24-hour
                                     retention; exact schema placement
                                     is implementation detail.
  ---------------------------------------------------------------------

# Appendix A --- V1 exclusions

-   Guest accounts/login; multiple active weddings per organiser;
    planner business accounts managing many weddings.

-   Accommodation, hotels/rooms, travel booking and seating
    arrangements.

-   Budget planning, instalments, advances and payment schedules.

-   Automated WhatsApp/SMS, push, in-app notification centre,
    real-time/WebSocket updates.

-   Vendor self-registration, bookings, commissions, leads, marketplace
    transactions and payments.

-   Native livestream infrastructure; ongoing AI assistant, RAG/vector
    search/vector database, separate AI microservice and unrestricted AI
    actions.

-   Kafka, Kubernetes/EKS, ECS/Fargate, Redis and complex event
    streaming in initial deployment.

# Appendix B --- Release readiness checklist

-   Create eight schemas and least-privilege service DB roles; separate
    migration credential.

-   Implement 34-table SQLAlchemy/Alembic baseline and all
    unique/check/partial indexes.

-   Implement common envelopes, request_id propagation, pagination and
    sort allowlists.

-   Implement shared authorisation helper for all nine modules and
    tenant isolation tests.

-   Implement internal service credentials, bounded timeouts/retries and
    controlled 503 responses.

-   Implement transactional outbox, dispatcher, idempotency records and
    concurrency tests.

-   Implement invitation-token expiry, guest media token hashing/manual
    revocation, and RSVP/Admin-cap concurrency tests.

-   Implement S3 upload verification, public asset delivery,
    abandoned-object cleanup and gallery visibility tests.

-   Implement notification worker leases, retries, cancellation, webhook
    verification and 12-month retention.

-   Implement AI finalisation step tracking, retry/resume and
    duplicate-prevention tests.

-   Generate and review OpenAPI schemas; validate request/response
    examples against Pydantic models.

-   Run integration tests for permissions, cross-wedding isolation, soft
    deletes, failure recovery and rate limits.

Final status: 16 cross-service consistency decisions approved; all 10
grouped validation corrections approved. Architecture and design
decisions are frozen as the V1.0 baseline. This document is a
consolidated design specification, not proof of code-level
implementation. Close the explicit open decisions and complete
integration/security testing before production release.

END OF SPECIFICATION
