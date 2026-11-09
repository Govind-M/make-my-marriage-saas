```{=html}
<!-- Source: Make_My_Marriage_PRD_V1_AI_Updated.docx. Converted from the latest agreed Word document. Content and decisions are retained; Markdown table layout may differ from Word. -->
```
# Make My Marriage

Product Requirements Document \| Version 1.2 -- V1 Scope and Decision
Reconciliation

9 November 2026

# Document Control

Document status: Updated V1 scope -- AI-assisted setup confirmed;
contradictory scope statements reconciled

**Product type:** Web application

**Primary market:** Indian weddings across India

**Primary users:** Couples and their invited wedding organisers

**Guest access model:** Browser-based; no account required

Prepared from: Previously approved V1 PRD and subsequent
product-discovery decisions

# Contents

Product Overview

Problem Statement

Product Goals

Target Users and User Types

Core Product Rules

Primary User Journeys

Functional Requirements

Main Application Navigation

Product-Level Data Relationships

Important Business Rules and Calculations

UX Principles

Privacy, Security and Non-Functional Requirements

Error and Edge Cases

V1 Non-Goals and Exclusions

Main Release Phases

V1 Success Metrics

Future Product Opportunities

Confirmed Product Decisions

Product Definition and Guiding Principle

## Document purpose and precedence

This PRD defines the product scope and expected behaviour of Make My
Marriage V1. It is intended to provide a shared baseline before system
design, UX design, data modelling and implementation begin.

This document uses the uploaded PRD.md as its structural and content
reference. It also incorporates later decisions made during product
discovery. Where the reference document and later decisions differ, the
later decisions take precedence.

### Decisions incorporated into this version

**Market:** Support Indian weddings across India, not only Kerala.
Wedding events and traditions must be configurable rather than fixed to
one regional or religious format.

**Organiser permissions:** Admins have full access. Managers receive
module-level permissions: No access, View or Manage.

**Activity history:** Activity log is included in V1.

**Vendor discovery:** The initial vendor directory is maintained by Make
My Marriage platform administrators. Vendors do not self-register.

**Guest photo uploads:** Guest-uploaded photos require organiser
approval before appearing in the approved gallery.

**RSVP:** One household-level RSVP applies to the invitation as a whole.
V1 does not collect separate attendance responses for each invited
event. One maximum attendee count applies to the whole invitation.

**Website visibility:** A published wedding website is viewable by
anyone who has its URL; no account or invitation is required.

**Gallery access:** A unique gallery link is sufficient. No passcode or
guest account is required. The gallery-view and guest-upload links do
not expire automatically; they remain active until an Admin manually
revokes or replaces them. The short-lived S3 upload/download URLs issued
during individual operations are separate from these long-lived guest
links.

**Cost records:** Amounts represent estimated or agreed/committed costs,
not payments made. Payment status and instalments are excluded.

**Email verification:** Organiser email verification is mandatory before
creating or joining a wedding.

**Photo attribution:** Guest photo uploads capture the uploader's name
and remain pending until organiser approval.

**Vendor directory operations:** Only authorised platform administrators
maintain listings. A listing requires a vendor name, category, service
location and at least one contact method; other profile details are
optional.

**Language:** The V1 application interface and system-generated messages
are English-only. User-entered event names and content remain flexible.

AI-assisted setup: Optional guided AI wedding setup is included in V1
alongside manual setup. It creates editable draft sections that require
organiser review, independent section approval and final confirmation
before records are created. RAG, vector search, a separate AI
microservice and an ongoing AI assistant are excluded from V1.

# 1. Product Overview

## 1.1 Product Name

**Make My Marriage**

## 1.2 Product Vision

Make My Marriage is a collaborative wedding planning and management
platform that helps couples and their families plan, coordinate and
manage an Indian wedding through one central wedding workspace.

Indian weddings can involve multiple ceremonies, several venues, large
guest lists, family members sharing responsibilities, vendors, expenses,
invitations, RSVPs, photographs and livestreams. Families often
coordinate these activities through disconnected tools such as WhatsApp
groups, spreadsheets, phone calls, notes, documents and shared photo
folders.

Make My Marriage brings the core activities together in one structured
application. It should feel like:

**One shared command centre for planning, coordinating and experiencing
an Indian wedding.**

## 1.3 Pan-Indian Product Principle

The product is intended for the diversity of Indian wedding cultures and
traditions across regions, religions, languages and communities. It must
not assume that every wedding follows the same sequence or includes the
same ceremonies.

Users can create, rename, edit and delete custom wedding events.

Common event suggestions may be offered as optional shortcuts, not
mandatory steps.

Wedding details, event names, schedules and website content must be
configurable.

Examples may include engagement, Roka, Mehendi, Haldi, Sangeet, Nikah,
Anand Karaj, Muhurtham, wedding ceremony and reception, among others.
These are examples only and are not an exhaustive or prescriptive list.

The initial product should use Indian context where appropriate,
including INR currency presentation and Indian wedding workflows.

## 1.4 Product Type

Responsive web application with two experiences:

**Organiser experience:** Authenticated dashboard for the couple and
invited organisers.

**Guest experience:** Mobile-friendly browser pages accessed through
invitation, website and gallery links. Guests do not create accounts or
log in.

# 2. Problem Statement

Planning an Indian wedding is collaborative but fragmented. Information
is often distributed across family members, messaging groups,
spreadsheets, email conversations and personal devices.

Typical problems include:

No single place to view wedding events and their details.

Unclear ownership of tasks and deadlines.

Guest lists and RSVP responses being maintained separately.

Vendor details and quotations being scattered across conversations.

Difficulty understanding the total expenses recorded for the wedding.

Invitations being shared through different channels without a central
status view.

Guests needing a simple way to access wedding details without installing
an app.

Wedding photographs becoming scattered across guests' phones and
separate folders.

Multiple organisers needing access without every person having
unrestricted control.

Make My Marriage addresses these problems by providing one
wedding-centred workspace and a low-friction guest-facing experience.

# 3. Product Goals

Make My Marriage V1 should enable a couple and their family to:

Register and create a wedding workspace.

Invite the other partner and trusted organisers to collaborate.

Control organiser access through simple roles and module permissions.

Plan multiple wedding events that reflect their own traditions.

Create, assign and track wedding tasks.

Maintain a household-level guest list.

Send digital invitations and collect RSVPs without guest accounts.

Track wedding expenses without building a budgeting or payment-schedule
system.

Maintain selected vendors and discover vendors from a curated directory.

Publish a personalised wedding website using a prebuilt theme.

Collect and share wedding photographs through private links and QR
access.

Allow organisers to approve guest-uploaded photographs.

Embed a YouTube Live stream on the wedding website.

Review important changes through an activity log.

-   Use an AI-assisted guided conversation as an optional alternative to
    manual wedding setup.

-   Save and resume AI setup conversations, generate editable planning
    drafts, and require organiser approval before creating wedding
    records.

## 3.1 V1 Success Definition

V1 is functionally successful when a couple can create a wedding
workspace either manually or through an optional AI-assisted setup,
invite family members to manage it, create events and tasks, manage
guests and invitations, collect RSVPs, record estimated or agreed costs,
maintain and discover vendors, publish a public wedding website,
configure a YouTube livestream, and collect and share approved wedding
photographs through a unique gallery link and QR code. The AI setup flow
must preserve progress, generate editable drafts, and create records
only after user approval.

# 4. Target Users and User Types

## 4.1 Primary Users

The primary users are:

Bride

Groom

Parents

Siblings

Other family members

Trusted friends helping organise the wedding

A wedding planner invited to help manage one wedding

All registered people who participate in the wedding workspace are
called **Wedding Members**.

## 4.2 Wedding Admin

The couple are expected to be the Admins of the wedding. Multiple Admins
are supported; for example, the bride and groom can both be Admins.

The user who creates the wedding becomes its first Admin. An Admin can:

Manage wedding details and settings.

Create and manage events, tasks, expenses, guests and vendors.

Manage the wedding website, gallery and livestream settings.

Invite and remove Wedding Members.

Assign or change a Manager's module permissions.

View the activity log.

Access all wedding modules.

## 4.3 Wedding Manager

Managers are family members, trusted friends or wedding planners who
help organise the wedding.

A Manager:

Must register and use their own organiser account.

Can access only the modules and actions granted by an Admin.

Can receive No access, View or Manage permissions for each applicable
module.

Cannot invite or remove Wedding Members.

Cannot change another member's role or permissions.

## 4.4 Guest

Guests do not have accounts, passwords or organiser roles. They use
secure, unique browser links to:

View their invitation and the events included in it.

Submit or update a household-level RSVP.

View the wedding website where access permits.

View approved gallery photographs where access permits.

Upload photographs through the guest upload experience.

## 4.5 Platform Operations Administrator

A separate internal Platform Operations Administrator maintains the
vendor discovery directory in the initial release. This is not a Wedding
Member role and does not give the platform operator membership in a
couple's wedding workspace.

Platform Operations Administrators can create, update and deactivate
vendor directory listings. Vendors cannot register or manage their own
profiles in V1.

# 5. Core Product Rules

**Wedding-centred workspace:** Wedding is the primary workspace.
Wedding-related records belong to a wedding.

**One wedding per user in V1:** A registered organiser account can
participate in only one wedding.

**Multiple members per wedding:** One wedding can have multiple Wedding
Members.

**Multiple Admins:** A wedding can have more than one Admin.

**Two organiser roles:** The only Wedding Member roles are Admin and
Manager.

**Manager permissions:** Admins control Manager access at module level
using No access, View or Manage.

**Member administration:** Only Admins can invite, remove or change the
role/permissions of Wedding Members.

**No guest accounts:** Guests never need to register, log in or install
an application.

**Household-level guest record:** One guest record represents the
primary invitation contact and the accompanying household. Individual
family members are not modelled separately.

**One invitation-level guest cap:** The organiser sets one maximum
attendee count for the household invitation. There is no separate
maximum count per event.

**Configurable wedding events:** Events are user-defined. Suggested
event names are optional and must not constrain regional or community
wedding practices.

**No private organiser data areas:** V1 does not include
bride-side/groom-side financial partitions or private task departments.
Access is controlled by module permissions.

**No planner portfolio accounts:** A planner cannot manage multiple
weddings under one account in V1.

**Curated vendor directory:** Vendor listings are maintained by platform
administrators. Vendor self-service is excluded.

**Guest photo moderation:** Guest-uploaded photographs remain pending
until an organiser approves them.

**Public wedding website:** Once published, the wedding website can be
viewed by anyone with its URL without login or invitation.

**Link-accessible gallery:** A unique gallery link is sufficient to view
approved photographs; no passcode or guest account is required. The
gallery is not listed in a public directory.

**One overall household RSVP:** One Attending or Not Attending response
applies to all events included in the household's invitation. No
event-by-event RSVP is collected.

**Estimated/agreed costs:** Expense records represent estimated or
agreed/committed costs, not amounts paid.

**Verified organiser accounts:** Email verification is required before
an organiser can create or join a wedding.

**Uploader attribution:** Guest photo uploads require the uploader's
name and remain pending until organiser approval.

**English-only interface:** V1 interface and system-generated messages
are in English. Free-text wedding and event names remain configurable.

**Platform-maintained vendor directory:** Only authorised platform
administrators maintain the directory; vendors do not self-register in
V1.

**AI-assisted setup in V1:** Optional guided AI wedding setup is
included in V1. RAG, vector search, a separate AI microservice and an
ongoing AI assistant are excluded.

# 6. Primary User Journeys

## 6.1 New Wedding Creator Journey

User registers with name, email and password.

User verifies the email address using the verification link sent by
email. Verification is mandatory before creating or joining a wedding.

User logs in.

User is prompted to create a wedding if they are not already a member of
one.

User enters the required wedding information.

The creator becomes the first Admin.

The Admin invites the other partner and other organisers.

The couple configures wedding events and begins planning.

## 6.2 Invited Organiser Journey

Admin enters the invitee's name and email and selects the
role/permissions.

The system sends an organiser invitation email with a unique link.

Invitee opens the link.

Invitee registers or logs in using the invited email address.

Invitee verifies the invited email address if it is not already
verified.

Invitee accepts the invitation and joins the wedding.

The invitee can access only the modules and actions granted by the
Admin.

## 6.3 Wedding Planning Journey

Organisers configure the wedding details.

Organisers create the wedding's events.

Organisers create tasks and assign responsibilities.

Organisers add vendors and, where needed, search the curated vendor
directory.

Organisers record expenses.

Organisers add household guest contacts and select the events each
contact is invited to.

Organisers send invitation emails or manually share invitation links.

Guests submit RSVPs through their unique links.

Organisers publish the wedding website and configure the gallery and
livestream.

## 6.4 Guest Invitation and RSVP Journey

Guest receives an email invitation or a link shared manually by an
organiser.

Guest opens the unique invitation URL in a browser.

Guest sees only the events included in that invitation.

Guest submits one overall RSVP for the invited household and the full
invitation.

If attending, the guest selects an attendee count no greater than the
invitation-level maximum.

The same overall response applies to all events shown on the invitation;
the guest can revisit the link to update it.

Organisers see the latest response in the guest dashboard.

## 6.5 Guest Photo Contribution Journey

Guest scans the wedding gallery QR code or opens a guest gallery link.

Guest chooses to view approved photographs or upload photographs.

Guest enters their name, selects photographs and, optionally, the
associated event album.

The system validates and receives the upload and stores the uploader
name with the submission.

Guest-uploaded photographs enter Pending Approval status.

An authorised organiser approves or rejects each submission.

Approved photographs become visible in the private gallery.

## 6.6 AI-Assisted Wedding Setup Journey

A couple may choose AI-assisted setup instead of manual setup. The
assistant conducts a guided, dynamic conversation, asks contextual
follow-up questions, saves progress throughout the session, and
generates an editable initial wedding plan. The couple reviews and
approves sections independently, then confirms the complete plan before
records are created through the existing domain services.

1.  The organiser selects AI-assisted setup and starts a setup session.

2.  The assistant collects essential wedding information and asks
    relevant follow-up questions. Optional information can be skipped.

3.  The assistant suggests wedding events based on the information
    provided. The couple can add, edit or remove suggestions.

4.  After events are approved, the assistant generates a task checklist
    and vendor category suggestions.

5.  The assistant drafts wedding website text and suggests an
    event-based album structure. The couple chooses the website theme
    from the existing themes.

6.  The couple reviews, edits and approves each section independently.

7.  After all desired sections are reviewed, the couple gives final
    confirmation. The backend validates and creates the approved records
    through the relevant services.

8.  If the organiser leaves and returns later, the session, messages and
    structured draft are restored so the conversation can resume.

# 7. Functional Requirements

## 7.1 Authentication and Account Management

### 7.1.1 Registration

Users must be able to register with:

Name

Email address

Password

Requirements:

Email addresses must be unique.

Email verification is mandatory. The user must verify the email address
before creating a wedding or accepting an organiser invitation.

Verification links must be single-use and time-limited; users can
request a new verification email.

Passwords must be stored securely; plaintext password storage is
prohibited.

The system must validate required fields and reject invalid registration
attempts.

A user account is for organisers only. Guests do not register.

### 7.1.2 Login and Logout

Users can log in with email and password.

Invalid credentials produce a clear error without disclosing whether an
account exists unnecessarily.

Users can log out.

Organiser-only pages and actions require an authenticated session.

Persistent login/session behaviour must be secure and can be finalised
during system design.

### 7.1.3 Password Management

Users can request a password reset.

Reset links must be time-limited and single-use.

Logged-in users can change their password.

### 7.1.4 Account and Membership Routing

After login, a user who belongs to a wedding is directed to that wedding
workspace.

A user without a wedding membership is directed to wedding creation or
an outstanding organiser invitation.

A user who already belongs to a wedding cannot join another wedding in
V1.

## 7.2 Wedding Workspace Creation and Setup

### Required fields

Bride name

Groom name

Primary wedding date

Wedding city/location

### Optional fields

Wedding title

Description or welcome message

Cover image

Couple photographs

Requirements:

A user without a wedding membership can create a wedding.

The creator automatically becomes the first Admin.

Wedding information can be edited by Admins and by Managers with Manage
permission for the relevant module.

The primary wedding date is used for the wedding countdown.

Wedding setup must not require selecting a specific region, religion or
fixed event sequence.

## 7.3 Wedding Member Management and Permissions

### Member administration

Admins can:

Invite an organiser by name and email.

Invite the other partner as an additional Admin.

Invite family members, trusted friends or a planner as Managers.

View current and pending members.

Resend or revoke pending invitations.

Remove an existing member.

Change a member's role, subject to the rule that a wedding must retain
at least one Admin.

Update a Manager's module permissions.

### Invitation acceptance

Organiser invitations use unique, secure links.

Invitees must use the email address to which the invitation was sent.

Invitees must register or log in and verify the invited email address
before joining.

The invitation must be accepted using the email address to which it was
sent.

An invitation cannot be used to bypass the one-wedding-per-user rule.

### Roles and permissions

**Admin:** Full access to all wedding modules and member administration.

**Manager:** Access is configured separately for applicable modules:

No access

View

Manage

The initial permission model should cover at least:

Wedding details/dashboard

Events

Tasks

Guests and invitations

Expenses

Vendors

Vendor discovery

Wedding website

Gallery

Livestream

Activity log

Managers cannot manage Wedding Members or change permissions. The
backend must enforce the permission assigned to the user; hiding a
navigation item alone is not sufficient.

## 7.4 Wedding Dashboard

The dashboard is the wedding command centre and should present an
at-a-glance view of wedding preparation.

### Summary information

Days remaining until the primary wedding date.

Number of events.

Tasks completed versus total tasks.

Upcoming events.

Pending and overdue tasks.

Total invited households/guest contacts.

RSVP summary and attendee estimate.

Total estimated/agreed costs recorded.

Number of vendors maintained for the wedding.

Recent activity.

### Dashboard behaviour

Information is aggregated from the relevant modules.

Dashboard cards link to the corresponding module or filtered view.

No advanced analytics or forecasting is required.

No realtime updates are required; users see current data when they
revisit or refresh the page.

## 7.5 Wedding Event Management

Wedding Members with appropriate permissions can create, view, edit and
delete events.

### Event fields

Event name

Date

Start time

End time (optional)

Venue name (optional)

Venue address/location (optional)

Map/directions link (optional)

Description (optional)

Dress code (optional)

Cover image (optional)

### Event capabilities

Create custom events.

Edit and delete events.

View events in chronological order.

View event details.

Associate tasks, vendors, expenses, guest invitations and photo albums
with an event where applicable.

Associate a YouTube Live URL with an event.

Suggested event names may be offered as optional shortcuts. The event
list must remain fully customisable and must support different Indian
regional, religious and community traditions. No event is mandatory.

## 7.6 Task and Responsibility Management

### Task fields

Title

Description (optional)

Assigned Wedding Member

Related event (optional)

Due date (optional)

Priority

Status

### Status values

To Do

In Progress

Completed

### Priority values

Low

Medium

High

### Task capabilities

Create, edit and delete tasks.

Assign a task to an eligible Wedding Member.

Associate a task with an event.

View all tasks.

View tasks assigned to the current user (My Tasks).

View completed tasks.

Filter by status, event, assigned member and priority.

Identify overdue tasks.

V1 does not include comments, subtasks, dependencies, task attachments
or complex project-management workflows.

## 7.7 Guest Management

A guest record represents one primary invitation contact and the
household/group covered by that invitation. V1 does not create separate
records for every family member.

### Guest fields

Primary contact name

Email address (required to send an email invitation; may be absent when
the link is shared manually)

Phone number (optional)

Maximum attendees allowed for the invitation

Events invited to

RSVP status

Number attending (as provided in the household response)

Notes (optional)

### Guest capabilities

Add, edit and remove guest contacts.

Search and filter the guest list.

Select one or more wedding events for each invitation.

View invitation and RSVP status.

View the number of confirmed household attendees.

### Household rules

One invitation represents one household/group contact.

Individual family members are not modelled separately.

The organiser sets one maximum attendee count for the invitation.

The maximum is shared across the invitation and is not configured
separately for each event.

The household submits one overall Attending or Not Attending response
for the entire invitation. The response applies to all events shown on
the invitation.

V1 does not collect separate RSVP responses or attendee counts for
individual events.

Individual family members cannot submit separate responses.

## 7.8 Invitations and RSVP

### Unique invitation links

Generate a unique, unguessable URL for each guest invitation.

Example format: makemymarriage.com/invite/{token}.

The link identifies the relevant wedding invitation and its invited
events.

The invitation page displays only events included in that invitation.

Guests do not log in.

The same link can be used to revisit and update an RSVP, subject to the
invitation's validity.

### Digital invitation page

The invitation page should display:

Couple names and wedding branding.

Welcome/invitation message.

Events the household is invited to.

Event dates and times.

Venue details and map links where available.

RSVP form.

### RSVP requirements

Guest can select Attending or Not Attending.

If attending, the guest selects the number of attendees.

The selectable number cannot exceed the single invitation-level maximum.

The household submits one overall Attending or Not Attending response
for the entire invitation. The response applies to all events displayed
on that invitation.

V1 does not collect separate RSVP responses or attendee counts for
individual events.

The guest can update the overall response through the same invitation
link.

The system records the latest RSVP status, attendee count and response
time.

Organisers can view Pending, Attending and Not Attending statuses.

There is no organiser approval workflow for the permitted attendee
count; the guest can select any count within the configured maximum.

## 7.9 Email Invitations, Reminders and WhatsApp Sharing

### Email invitations

Organisers can send an invitation email to a guest with an email
address.

The email contains couple/wedding identity, a short message, primary
wedding date and the unique invitation link.

The system records send status and relevant delivery errors where the
email provider supplies them.

Organisers can resend an invitation.

### RSVP reminders

Organisers can identify guests whose RSVP remains Pending.

Organisers can manually initiate a reminder request for one pending
guest, a selected set of pending guests, or all currently pending
households. The request reports how many reminders were queued and lets
the organiser review its processing summary.

Reminder emails link to the guest's existing invitation.

Scheduled automatic reminder campaigns are not required in V1.

### WhatsApp sharing

Provide a Share on WhatsApp action for organisers.

The action opens WhatsApp or WhatsApp Web with a pre-filled message
containing the invitation link.

The organiser manually sends the message.

No WhatsApp Business API integration or automated WhatsApp sending is
included.

## 7.10 Expense Tracker

The expense tracker records the estimated or agreed/committed cost of
wedding items and services. It is not a budgeting, accounting or
payment-schedule system, and it does not record amounts actually paid.

### Expense fields

Expense title

Amount (estimated or agreed/committed cost)

Date

Category

Related event (optional)

Related vendor (optional)

Notes (optional)

### Suggested categories

Venue

Catering

Photography

Videography

Decoration

Clothing

Jewellery

Entertainment

Invitations

Gifts

Travel

Makeup

Miscellaneous

Categories should be editable or extensible only if required by the
chosen implementation; the product must at minimum provide useful
defaults and a way to classify an expense.

### Expense capabilities

Add, edit and delete an expense.

View all expenses.

Filter and sort expense records.

View total estimated/agreed costs recorded.

View totals by category, vendor or event where data is associated.

### Explicit limits

No wedding budget planning.

No budget allocation or budget-vs-actual calculations.

No split expenses.

No advance, instalment or partial-payment tracking.

No due/outstanding balance calculation.

No vendor payment schedules.

No private bride-side or groom-side expense areas.

An expense record stores an estimated or agreed/committed cost and its
context. It does not represent a payment transaction. V1 does not track
whether that amount has been paid, partially paid or is still
outstanding.

## 7.11 My Vendors --- Wedding Vendor Management

Wedding Members can maintain the list of vendors selected or considered
for their wedding.

### Vendor fields

Vendor name

Category

Contact person (optional)

Phone number (optional)

Email (optional)

Address/location

Website or portfolio URL (optional)

Quoted/agreed cost (optional)

Related events (optional)

Notes (optional)

Status

### Suggested statuses

Shortlisted

Contacted

Confirmed

Not selected

### Vendor capabilities

Add, view, edit and remove a vendor from the wedding's vendor list.

Associate vendors with one or more events.

Record quoted/agreed cost for reference.

Search and filter My Vendors by category, event and status.

Link an expense record to a vendor.

V1 does not support vendor contracts, booking transactions, vendor
payments, payment schedules or vendor-managed accounts.

## 7.12 Vendor Discovery Directory

V1 includes a curated vendor directory maintained by Make My Marriage
Platform Operations Administrators.

### Platform administrator capabilities

Only authorised Platform Operations Administrators can create, edit,
publish and deactivate vendor directory listings.

A listing must include vendor name, category, service location (city or
service area) and at least one contact method (phone, email or website).

Description, full address, portfolio URL, images and additional contact
details are optional.

Platform administrators are responsible for keeping published listings
reasonably current. Vendors cannot register or edit their own listings
in V1.

### Couple/organiser capabilities

Browse vendor categories.

Search vendor listings.

Filter by category and location.

View vendor profiles and available contact/portfolio information.

Shortlist a vendor.

Add a selected vendor to My Vendors.

### V1 exclusions

Vendor self-registration.

Vendor profile management by vendors.

Reviews and user-generated ratings unless explicitly added later.

Enquiry/lead-management workflows.

Booking, contracts or payment processing.

Commission handling or marketplace transactions.

## 7.13 Wedding Website Builder

Each wedding can have a hosted, shareable wedding website. Once
published, it is public to anyone who has the website URL; visitors do
not need an account, login or invitation link.

### Website sections

Hero section with couple names, wedding date and cover photograph.

Welcome message or short description.

Event schedule with dates, times, venues and directions.

Optional gallery section when gallery access is enabled.

Optional livestream section when a YouTube Live URL is configured.

### Themes

V1 provides approximately three prebuilt themes:

Traditional Indian.

Minimalist.

Royal/Luxury.

Theme names are working labels. The themes are visual presentation
options and must not imply a specific region, religion or ceremony
sequence.

### Website management

Select a theme.

Edit supported text and images.

Preview the website.

Publish or unpublish the website.

Share the hosted website URL.

Anyone with the published website URL can view the website without
logging in. The website can be unpublished by an Admin.

Website content uses the wedding and event information maintained in the
workspace; users should not need to re-enter the same details in
multiple places.

V1 does not include drag-and-drop page building, arbitrary page layouts
or custom domains.

## 7.14 Photo Gallery, Albums, Guest Uploads and QR Code

### Gallery and albums

Each wedding has a gallery accessible through a unique,
difficult-to-guess link. Anyone who has the current gallery link can
view approved photographs; no passcode or account is required.

Organisers can create or use event-based albums.

A general "Other / Wedding Memories" album may be available.

Organisers can upload, view, download and delete photographs.

Organisers can associate photographs with an event album.

The gallery is not listed or discoverable through the public website
directory; access is link-based. An Admin can replace/revoke the current
link if it is shared too widely.

### Guest upload

Guests can upload photographs without creating accounts.

The upload page is mobile-friendly.

Guests must enter their name; the name is stored with each upload and
shown to organisers for attribution.

Guests can optionally choose the relevant event album.

Guest-uploaded photographs enter Pending Approval status.

An authorised organiser can approve or reject a pending photograph.

Only approved photographs appear in the guest-facing gallery.

Guests cannot delete photographs in V1.

### Gallery viewing

Anyone with the unique gallery link can view approved photographs; no
additional passcode or login is required.

Organisers can share the gallery link.

An Admin can regenerate/revoke the link. The same simple link-based
access model is used for the guest gallery experience in V1.

### QR code

Generate a QR code that opens the wedding's unique gallery link.

The gallery page allows visitors to view approved photos and, when guest
uploads are enabled, submit new photos with their name. New uploads
require organiser approval before becoming visible.

Organisers can view, download and print/share the QR code.

The QR destination should remain stable while the underlying access
settings can be managed.

### Upload safeguards

The system must validate allowed image formats and file sizes, restrict
upload volume and protect upload endpoints from abuse. Exact file limits
and retention settings are finalised during system design.

## 7.15 YouTube Livestream

Organisers can add, update or remove a YouTube Live URL for a wedding
event.

The livestream is embedded on the wedding website when configured.

Guests watch through the embedded YouTube player or the provider's
supported browser experience.

Make My Marriage does not build or operate its own video streaming
infrastructure.

## 7.16 Activity Log

V1 includes a wedding-level activity log. It records important actions
so organisers can understand what changed and who performed the action.

### Activities to record

Wedding created or updated.

Wedding Member invited, accepted, removed or permission changed.

Event created, updated or deleted.

Task created, assigned, updated or completed.

Expense created, updated or deleted.

Vendor added, updated or removed from My Vendors.

Guest created or updated.

Invitation sent or resent.

RSVP submitted or updated.

Wedding website published, unpublished or updated.

Photograph uploaded, approved, rejected or deleted.

Livestream URL added, updated or removed.

### Activity record information

Actor (organiser, guest/invitation or system where applicable).

Action description.

Related module/entity.

Date and time.

The activity log is a historical record, not an in-app notification
centre. Manager visibility is controlled by the Activity Log module
permission.

## 7.17 Wedding Settings

Settings should include:

### Wedding details

Bride and groom names.

Primary wedding date.

City/location.

Wedding title and description.

Cover image and couple photographs.

### Wedding Members

Current and pending members.

Roles.

Manager permissions.

### Website

Theme.

Supported content and images.

Publish status.

Website link.

### Gallery

Guest upload availability.

Gallery access links.

QR code.

Pending photo moderation.

### Livestream

YouTube Live URL and enabled/disabled state.

## 7.18 AI-Assisted Wedding Setup

### Purpose and availability

AI-assisted wedding setup is an optional V1 onboarding path. It helps an
organiser prepare an initial wedding workspace and planning draft
through a guided conversation. Manual setup remains available and must
not depend on AI availability.

### Conversation behaviour

-   The assistant uses a guided, dynamic conversation rather than a
    fixed questionnaire.

-   It may ask contextual follow-up questions when an answer is
    incomplete or clarification would materially improve the plan.

-   It must not ask for information already provided in the current
    session unless confirmation is needed.

-   Users can correct earlier answers, skip optional questions and
    return to previous stages.

-   The assistant must distinguish confirmed user-provided information
    from AI suggestions.

-   Conversation messages and structured information must be saved
    throughout the session, not only at completion.

### Information collection

Essential information includes couple names, wedding date status and
available date detail, primary wedding location, and the couple's
planned or desired events. Wedding tradition/community and approximate
guest count are optional. The exact minimum needed to create a workspace
remains subject to the normal wedding setup validation rules.

Wedding date input must support: (a) an exact date, (b) an approximate
period such as a month or part of a month, or (c) undecided. The
assistant must not convert an approximate period into an invented exact
date. When the date is undecided, it must avoid date-dependent
deadlines.

Optional personalisation information may include wedding style, a rough
budget range for tailoring suggestions only, event venues, food
preferences, photography preferences, decoration preferences, priorities
and special requirements. A rough budget range is not a V1
budget-management feature and must not create budget records,
allocations or payment tracking. The assistant must not require users to
disclose religious or community information.

### AI-generated draft sections

-   Wedding events: suggest configurable events based on the couple's
    answers and preferences. Suggestions are optional and must not imply
    that a tradition is universal or mandatory.

-   Tasks: after event suggestions are approved, generate event-specific
    and general tasks. Exact task deadlines may be suggested only when
    supported by an exact wedding date. For an approximate date, use
    clearly labelled relative planning windows where useful, not
    invented exact calendar dates. When the date is undecided, do not
    generate date-dependent deadlines.

-   Vendor requirements: suggest relevant vendor categories and, where
    useful, associate categories with events. Actual vendor listings,
    prices, ratings and availability must come from the existing vendor
    directory or verified application data, not model invention.

-   Wedding website: draft written content such as welcome message,
    couple introduction, wedding description, event descriptions, venue
    text and closing message. The couple selects one of the existing
    website themes: Traditional Indian, Minimalist or Royal/Luxury. AI
    does not generate or change the visual theme.

-   Gallery albums: suggest an initial album structure based on approved
    events. The couple can add, rename or remove album suggestions.

### Review, approval and final creation

-   Each generated section can be reviewed, edited and approved
    independently.

-   Approval of one section does not automatically approve another
    section.

-   The user may revisit and edit sections before final confirmation.

-   Changes to approved source information, such as removing an event,
    must flag dependent draft sections (for example, tasks or albums)
    for review. The system must not silently delete approved
    information.

-   Final confirmation is required before permanent wedding records are
    created.

-   Final creation must be idempotent so duplicate submissions or
    network retries do not create duplicate records.

-   Records are created through existing Wedding, Planning, Vendor,
    Media and other relevant domain services. The AI component must not
    write directly to application tables or bypass service validation
    and authorisation.

### Conversation persistence and session management

-   Organisers can leave an unfinished setup and resume it later from
    the last saved state.

-   The application stores session status, current stage, collected
    structured information, draft plan, prompt version and timestamps.

-   Conversation messages are associated with a setup session and
    ordered chronologically.

-   An unfinished setup session and its unapproved draft can be deleted
    without affecting approved wedding records.

-   Once a wedding is created, the setup session may be associated with
    the resulting wedding for traceability, subject to the application's
    retention policy.

### AI output quality and error handling

-   All model output is untrusted input and must be validated against
    defined Pydantic schemas before being used.

-   Invalid or incomplete structured output must be rejected; the system
    may retry generation within configured limits and must preserve the
    existing conversation and draft.

-   Temporary provider failures may be retried up to two times with
    short delays. The user must be able to retry later or resume the
    session.

-   The system must prevent duplicate assistant responses or duplicate
    record creation when a request is retried.

-   Unclear answers should result in a clarification question or an
    option to skip if the information is optional; the assistant must
    not invent missing facts.

-   If one section generation fails, other saved sections and
    conversation progress remain available.

-   When an earlier answer changes, dependent sections must be flagged
    for review rather than silently overwritten.

-   The assistant must present cultural and regional customs as optional
    suggestions, acknowledge variation between families and communities,
    and avoid unsupported claims.

### Development usage controls

During development, the initial provider is Google Gemini API using its
available free tier, with synthetic wedding data only. Groq and
OpenRouter may be used as alternative providers for evaluation.
Provider/model choice must be configurable through a
provider-independent backend adapter.

Initial application-level usage defaults are: up to 50 user messages per
setup session; up to 3 generation attempts for each of events, tasks,
vendor categories, website content and album structure. These are
development defaults and may be tuned after testing. Editing existing
suggestions should not consume another generation attempt.

Each model request should use bounded input and output token budgets.
Usage should be tracked per setup session. The application should
provide a clear message when an application or provider limit is
reached. Provider free-tier quotas are external limits and may change.

### Security and privacy

-   Only the minimum information necessary for the current generation
    task is sent to the LLM provider.

-   Passwords, JWTs, refresh tokens, verification/reset tokens, AWS
    credentials, S3 presigned URLs, guest contact lists and unrelated
    private data must never be sent to the LLM.

-   LLM API keys remain on the backend and are supplied through
    environment configuration during development. They must never be
    exposed to React, committed to source control or written to logs.

-   AI setup sessions are accessible only to the authenticated organiser
    who owns the session, subject to wedding membership and permission
    checks where applicable.

-   The backend derives and validates the authorised user and wedding
    context. The LLM does not determine access rights or tenant scope.

-   The AI component cannot execute arbitrary SQL, change organiser
    roles, send invitations, publish websites, delete records or perform
    unrestricted application actions.

-   Development uses synthetic data. Before real-user launch, the
    selected provider's data retention, training, review and processing
    terms must be assessed, and appropriate user disclosure/consent and
    data-minimisation controls must be in place.

-   Logs must not contain API keys, access tokens, refresh tokens,
    presigned URLs or unnecessary sensitive conversation content.

### Prompt and provider management

-   The backend uses a provider adapter so the application is not
    tightly coupled to one LLM provider.

-   Provider and model are configurable by environment settings.

-   Each setup session records a prompt version to support debugging and
    controlled prompt updates.

### V1 AI exclusions

-   No permanent or always-available AI wedding assistant after initial
    setup.

-   No RAG pipeline, vector database or embedding-based knowledge base.

-   No separate AI microservice or AI agent with unrestricted tools.

-   No automatic event/task/vendor/album creation before user approval.

-   No AI-generated vendor identities, prices, ratings or availability.

-   No AI-generated visual website themes or automatic website
    publishing.

# 8. Main Application Navigation

Suggested organiser navigation:

Dashboard

Events

Tasks

Guests

Guest List

Invitations

RSVP

Expenses

Vendors

My Vendors

Discover Vendors

Wedding Website

Photos

Gallery

Pending Uploads

QR Code

Livestream

Activity Log

Settings

Wedding Details

Wedding Members (Admin only)

Navigation items and page actions must respect the signed-in member's
permissions.

## 8.1 Guest Navigation

Guest pages are simple, mobile-first pages reached by links or QR codes:

Invitation and RSVP page.

Wedding website.

Gallery view/upload page.

Embedded livestream section on the website.

No guest account or login flow is present.

# 9. Product-Level Data Relationships

The following relationships describe the product domain. They are not a
final database schema.

User Account \| +-- Wedding Membership (one wedding per user in V1) \|
+-- Wedding Workspace \| +-- Wedding Members and Permissions +-- Wedding
Events \| +-- Tasks \| +-- Event-related Vendors \| +-- Event-related
Expenses \| +-- Guest event invitations \| +-- Photo Albums \| +--
Livestream configuration +-- Tasks +-- Guests / Household Invitation
Contacts \| +-- Unique Invitation Link \| +-- RSVP response +-- My
Vendors +-- Expenses +-- Wedding Website and Theme +-- Gallery and
Photos \| +-- Guest Upload Moderation Status \| +-- Gallery View/Upload
Access +-- Activity Log

Platform Operations Administration +-- Curated Vendor Directory

A vendor directory listing and a vendor selected for a particular
wedding are conceptually different records: the former belongs to the
platform directory; the latter belongs to the wedding workspace.

# 10. Important Business Rules and Calculations

**Wedding countdown:** Calculate the number of days remaining until the
primary wedding date.

**Task completion:** Completed task count divided by total task count.
If there are no tasks, display an empty or zero state rather than a
division error.

**Total recorded costs:** Sum all estimated or agreed/committed cost
amounts recorded for the wedding. Optional breakdowns may group by
category, vendor or event.

**Guest count:** Keep the number of invitation contacts/households
distinct from the number of people expected to attend.

**RSVP headcount:** Count the attendee number submitted by each
household. Do not count every household member as a separate guest
record.

**Invitation cap:** A guest cannot submit a household attendee count
above the one maximum set on that invitation.

**Overall RSVP:** One household response applies to the entire
invitation and all events shown on it. V1 does not collect
event-by-event attendance responses.

**Invitation events:** The guest invitation page must show only the
events selected for that guest.

**Photo visibility:** Guest-uploaded photos are not visible in the
approved gallery until an organiser approves them.

**Uploader attribution:** The guest must provide a name when uploading;
the name is stored with the upload and is visible to organisers.

**Website access:** A published wedding website is accessible to anyone
who has its URL, without login or invitation.

**Gallery access:** Anyone with the current unique gallery link can view
approved photographs. No passcode or account is required.

**Manager access:** Every protected action must validate the relevant
module permission on the server.

**Wedding isolation:** A member of one wedding cannot access another
wedding's private workspace data.

**Admin continuity:** A wedding must always have at least one Admin.

**Email verification:** An organiser must verify their email address
before creating or joining a wedding.

**No payment status:** Cost records do not imply paid, unpaid, partially
paid or outstanding status.

# 11. UX Principles

## 11.1 Wedding First

The organiser experience should consistently identify the current
wedding through the couple names or wedding identity. All modules should
feel connected to the same workspace.

## 11.2 Simple for Families

The interface should be understandable to parents and family members who
are not highly technical. Common actions such as adding a guest,
completing a task, recording an expense and viewing an RSVP should be
straightforward.

## 11.3 Pan-Indian and Customisable

The product must not assume one regional or religious wedding format.
Event names, event sequence, dates, locations and website content are
configurable. Optional examples must remain non-prescriptive.

## 11.4 Mobile Friendly

The organiser application should be responsive on mobile and desktop.
Guest flows should be designed mobile-first, particularly:

Invitation viewing.

RSVP.

WhatsApp sharing.

QR scanning.

Photo upload.

Gallery browsing.

Livestream viewing.

## 11.5 Minimal Guest Friction

Invitation link should lead directly to invitation and RSVP. QR scan
should lead directly to the gallery experience. Guests should not be
asked to create accounts, set passwords or install an application.

## 11.6 Clear Permissions

Users should understand which modules they can view or manage.
Restricted controls should not appear as available actions, and
unauthorised direct requests must be rejected.

# 12. Privacy, Security and Non-Functional Requirements

## 12.1 Authentication and Access Control

Organiser-only pages require authentication.

Organiser email addresses must be verified before a user can create or
join a wedding.

Wedding workspace requests must verify membership in the target wedding.

Module-level permissions must be enforced for Managers.

Admin-only member-management operations must be protected.

Guest access must be limited to the data and action authorised by the
guest's unique link.

## 12.2 Token and Link Security

Invitation, gallery-view and photo-upload links must use unpredictable
tokens.

A published wedding website is public to anyone with its URL. A gallery
link is sufficient for viewing approved photos; V1 does not require a
gallery passcode or guest account.

Tokens must not expose sequential database identifiers as the only
protection.

Access links should be revocable or replaceable by an authorised Admin
where appropriate.

Invalid or expired links should show a clear, safe error page.

## 12.3 Data Privacy

Wedding workspace and organiser data must not be publicly searchable. A
published wedding website is public to anyone with its URL.

Gallery access is link-based: anyone with the unique gallery link can
view approved photos. The gallery itself is not listed in a public
directory.

Gallery images must not be exposed through guessable public object
paths.

Guest-uploaded images require validation and moderation.

Only authorised organisers can approve or reject guest photos.

Personal contact information should only be available to authorised
Wedding Members.

## 12.4 Upload Safety

Validate image file type and size.

Limit upload frequency/volume to reduce abuse.

Handle failed or interrupted uploads safely.

Do not allow guests to delete or replace other people's submissions.

## 12.5 Responsive Design and Usability

Organiser workflows must work on common desktop and mobile browsers.

Guest workflows must be mobile-first.

Forms should provide clear validation messages and confirmation for
destructive actions.

Empty states should guide users toward the next useful action.

## 12.6 Performance and Reliability

Common pages and user actions should remain responsive under expected V1
wedding usage, including several hundred invitees and multiple events.

The application should handle failures from email delivery, image
uploads and embedded third-party livestreams gracefully.

The user should be able to distinguish a saved change from a failed
operation.

Detailed throughput, latency, availability and file-size targets will be
set during system design and testing.

## 12.7 Indian Context

Default currency display is INR (₹).

Date and time presentation should be suitable for Indian users.

The V1 application interface, navigation, default website labels and
system-generated emails are English-only.

User-entered wedding names, event names and descriptions remain free
text so families can use the terms appropriate to their own traditions.

Wedding location should support cities and addresses across India.

# 13. Error and Edge Cases

## 13.1 Authentication and Membership

Email address is already registered.

Invalid login details.

Password reset link is expired or already used.

Organiser invitation is invalid, expired or revoked.

Invited email does not match the signed-in account.

Invitee already belongs to a different wedding.

Admin attempts to remove the final Admin.

Admin tries to change another Admin's role without leaving the wedding
with an Admin.

Manager attempts to access member administration.

## 13.2 Events and Tasks

Event has no venue.

Event date differs from or occurs after the primary wedding date.

Event is edited after invitations have been sent.

Event is deleted while guest invitations or tasks reference it.

A task is assigned to a member who no longer belongs to the wedding.

A user attempts to manage a task without Manage permission.

The UI should warn organisers before destructive actions that affect
related data.

## 13.3 Guests and RSVPs

Invitation link is invalid, expired or revoked.

Guest submits an RSVP more than once.

Guest updates an existing RSVP.

Guest attempts to select more attendees than permitted.

Guest has no email address and the link must be shared manually.

Guest is invited to only a subset of wedding events.

Email invitation or reminder fails to send.

## 13.4 Vendors and Expenses

Vendor exists without an expense.

Expense exists without a vendor.

Directory listing is deactivated after being shortlisted.

Vendor contact information is incomplete.

Expense amount is invalid or negative.

Both a vendor without an expense and an expense without a vendor are
valid.

## 13.5 Website and Livestream

Wedding website is unpublished.

Website theme is changed after publication.

Event details are changed after publication.

YouTube Live URL is invalid or the stream is unavailable.

Gallery or livestream section is disabled or not configured.

## 13.6 Gallery and QR Code

Guest opens an invalid or revoked gallery link.

Guest uploads an unsupported file type.

File exceeds the configured size limit.

Upload fails midway.

Guest submits duplicate photographs.

Organiser rejects a photo.

QR code is scanned after upload access has been disabled.

Guest attempts to access a pending or rejected photograph through a
direct URL.

# 14. V1 Non-Goals and Exclusions

The following features are explicitly outside the V1 scope:

### Wedding and account model

Multiple weddings per organiser account.

Professional planner business accounts managing multiple weddings.

Separate user roles for parents, siblings, planners or other family
categories.

Guest accounts or a dedicated guest mobile application.

### Advanced planning and logistics

Accommodation and hotel management.

Room allocation and check-in/check-out.

Flight or train tracking.

Airport pickups, vehicle and transport management.

Seating arrangements.

Advanced schedule optimisation.

### Financial management

Wedding budget planning.

Budget allocation and budget limits.

Split expenses.

Advance, instalment and partial-payment tracking.

Vendor payment schedules.

Outstanding payment calculations.

Bride-side/groom-side private financial areas.

### Vendor marketplace

Vendor self-registration and vendor accounts.

Vendor-managed profiles.

Booking, contracts and payments through the platform.

Commissions and marketplace transactions.

Full lead-management workflow.

### Communication and collaboration

WhatsApp Business API integration.

Automated SMS.

Push notifications.

In-app notification centre.

Realtime collaborative updates or WebSocket-based live feeds.

Automated scheduled RSVP reminder campaigns.

### Website, media and AI

Drag-and-drop website builder.

Custom domains.

Native livestreaming infrastructure.

Advanced gallery permission matrices.

RAG knowledge base and vector search.

Ongoing AI wedding assistant.

# 15. Main Release Phases

All features below are in the V1 product scope, but they should be
implemented incrementally.

## Phase 1: Foundation and Workspace

Organiser registration, login, logout and password reset.

Wedding creation and details.

Wedding Member invitations.

Admin and Manager roles.

Manager module permissions.

Dashboard shell.

## Phase 2: Planning and Coordination

Event management.

Task management.

Activity log foundation and event/task activity capture.

## Phase 3: Guest Management

Guest list.

Event-specific invitation selection.

Unique invitation links.

Email invitations and manual resend.

RSVP and attendee count.

Pending RSVP view and manual email reminders.

Manual WhatsApp sharing.

## Phase 4: Vendors and Expenses

My Vendors.

Platform-managed vendor directory.

Vendor search, filters and shortlisting.

Expense tracker and summaries.

## Phase 5: Wedding Website and Livestream

Wedding website content.

Three prebuilt themes.

Preview and publish controls.

YouTube Live URL configuration and embedding.

## Phase 6: Gallery and QR Sharing

Private gallery.

Event albums.

Organiser uploads.

Guest uploads.

Pending photo approval and rejection.

Guest viewing/upload links.

QR code generation and download.

## Phase 7: AI-Assisted Wedding Setup

-   AI setup conversation and resumable session persistence.

-   Structured information extraction and Pydantic validation.

-   Event, task, vendor category, website content and album draft
    generation.

-   Independent section review and approval, followed by idempotent
    final confirmation.

-   Provider adapter, usage limits, error recovery, prompt versioning
    and privacy controls.

## Phase 8: Production Readiness

End-to-end testing.

Permission and cross-wedding isolation testing.

Guest-link and upload security testing.

Responsive design validation.

Error handling and performance checks.

Deployment, monitoring and backup readiness.

# 16. V1 Success Metrics

The following metrics can be used to evaluate product usage. They do not
need to be exposed as dashboard analytics to wedding members.

## Activation

Percentage of registered organisers who create or join a wedding.

Percentage of created weddings with at least one event.

Percentage of created weddings with at least one task.

## Collaboration

Average number of Wedding Members per wedding.

Percentage of weddings with more than one active organiser.

## Guest Engagement

Number of invitations created and sent.

Invitation email delivery rate where available.

RSVP response rate.

Percentage of invited households with a completed RSVP.

## Planning Usage

Events created per wedding.

Tasks created and completed.

Expenses recorded per wedding.

Vendors added to My Vendors.

Vendor directory searches and shortlist actions.

## Wedding Experience

Wedding websites published.

Gallery visits.

Organiser and guest photo uploads.

Percentage of guest photos reviewed.

Livestream links configured.

## AI-Assisted Setup Metrics

-   Percentage of weddings created through AI-assisted setup versus
    manual setup.

-   AI setup completion rate and session-resume rate.

-   Percentage of generated sections approved with or without edits.

-   Average generation attempts per section and provider failure rate.

-   Percentage of AI-generated records successfully created after final
    confirmation.

# 17. Future Product Opportunities

The following ideas may be considered after V1 validation. They are not
current commitments.

## Communication

WhatsApp API integration.

Automated SMS.

Automated RSVP reminder campaigns.

## Guest Logistics

Household and individual family member modelling.

Dietary preferences.

Seating arrangements.

Accommodation and transport management.

## Financial Management

Wedding budgets.

Advances and payment schedules.

Partial payment tracking.

Expense splitting.

## Vendor Marketplace

Vendor onboarding and self-service profiles.

Reviews and ratings.

Enquiries, leads, bookings and payments.

Marketplace commissions.

## Wedding Website and Gallery

Additional themes.

Custom domains.

Advanced page builder.

Advanced photo discovery and AI-based photo tagging.

## AI-Assisted Wedding Setup

A future assistant may gather requirements through conversation and
prepare an editable draft workspace. It could use a curated knowledge
base (RAG) and structured actions to suggest wedding details, events and
tasks. Any generated changes should be reviewed and approved before they
are saved.

A future ongoing assistant could answer questions about existing wedding
data or perform approved actions. The initial guided AI-assisted wedding
setup is included in V1; an ongoing assistant remains a future
opportunity.

# 18. Confirmed Product Decisions

The following decisions have been confirmed and are the baseline for V1
product design and implementation:

**Wedding website visibility:** A published wedding website is viewable
by anyone who has its URL. No login or invitation is required. An Admin
can unpublish the website.

**Gallery access:** A unique, difficult-to-guess gallery link is
sufficient to view approved photos and access the guest gallery page. No
passcode or guest account is required. The gallery is not listed in a
public directory. Gallery-view and guest-upload links have no automatic
expiry and remain active until an Admin manually revokes or replaces
them.

**Household RSVP:** A household submits one overall Attending or Not
Attending response for the entire invitation, covering all events shown
on that invitation. V1 does not collect separate RSVP responses per
event. One maximum attendee count applies to the entire invitation.

**Expense amount:** Each expense record stores an estimated or
agreed/committed cost. It does not represent money paid. V1 does not
track payment status, instalments or outstanding balances.

**Email verification:** Email verification is mandatory before an
organiser can create a wedding or accept an invitation to join one.

**Guest photo attribution:** A guest must provide their name when
uploading photographs. The uploader name is stored with the submission
and is visible to organisers. The photograph remains pending until an
organiser approves it.

**Vendor directory operations:** Only authorised Make My Marriage
Platform Operations Administrators create, edit, publish and deactivate
vendor listings. Required listing details are vendor name, category,
service location and at least one contact method. Description, full
address, portfolio and images are optional. Vendors do not self-register
or manage profiles in V1.

**Language:** The V1 interface, navigation, default website labels and
system-generated emails are English-only. User-entered wedding and event
names remain flexible.

These decisions close the product questions raised during discovery and
should be treated as confirmed requirements for the V1 PRD.

## AI-Assisted Setup Decisions

The following AI-specific decisions are confirmed for V1:

-   AI-assisted wedding setup is an optional V1 path; manual setup
    remains available.

-   The assistant uses a dynamic guided conversation with contextual
    follow-up questions.

-   The wedding date can be exact, approximate or undecided.

-   Tradition/community and approximate guest count are optional
    information.

-   AI suggestions remain drafts until reviewed and approved by the
    organiser.

-   Events, tasks, vendor categories, website text and album structure
    can be generated as editable sections.

-   The couple selects the existing website visual theme; AI generates
    written content only.

-   Sections are approved independently, followed by one final
    confirmation.

-   Conversation and draft progress are saved throughout and can be
    resumed later.

-   AI output is validated; AI cannot directly write to application
    tables or perform unrestricted actions.

-   Development uses a configurable provider adapter and synthetic data
    with free-tier LLM access where available.

-   Initial development limits are 50 user messages per session and up
    to 3 generation attempts per generated section.

-   No RAG, vector database, separate AI microservice or ongoing AI
    assistant is included in V1.

# 19. Product Definition and Guiding Principle

Make My Marriage V1 is a web application for couples and families
planning Indian weddings across India. It provides a wedding-centred
workspace for organiser collaboration, configurable events, tasks,
household guest invitations and RSVPs, expense tracking, vendor
management and discovery, a themed wedding website, link-accessible
photo sharing with organiser moderation, YouTube livestream embedding
and activity history. Couples may create their workspace manually or use
an optional AI-assisted setup that generates editable planning drafts
and requires review and approval before records are created. Organisers
authenticate with their own accounts and access the wedding according to
their Admin or Manager permissions. Guests participate through secure
browser links and do not need accounts.

Use the following question when evaluating a proposed V1 feature:

**Does this directly help a couple and their family plan, coordinate,
celebrate or preserve their wedding across the diversity of Indian
wedding traditions?**

If it supports that core experience without introducing disproportionate
complexity, it may belong in V1. If it creates a separate product or
significantly expands the operating model, defer it until the core
platform has been validated.
