

# Event RSVP Manager

## Problem statement

Hosts need a straightforward way to create an event (with an optional capacity) and collect RSVP responses from invited people, while keeping attendance information accurate and up to date. Invitees need a low-friction way to respond (Yes/No/Maybe) through a unique link and enter their name, and to change their response until the event starts. The system must enforce correctness rules around time-based locking, capacity limits, and waitlist promotion so that the host can trust the dashboard and invitees receive predictable outcomes.

## Context and constraints

### Technical constraints

- The solution must fit the existing full-stack scaffold: Spring Boot (Spring MVC + JPA) + PostgreSQL + React/TypeScript.
- Event start time is represented as an exact instant (derived from the host’s chosen local time), and RSVP locking is based on that instant.
- The host dashboard requires seconds-level “real-time push” updates (not just refresh).

### Product constraints

- Invitees do not sign in; access to RSVP is via a unique invite link.
- RSVP options are limited to Yes / No / Maybe.
- RSVP changes are allowed only until the event start time; after start, all RSVPs are locked.
- Capacity is optional; when set and full:
  - "Yes" responses beyond capacity are waitlisted.
  - Waitlist applies only to "Yes".
  - "Maybe" counts toward capacity similarly to "Yes".
- Waitlist promotion is automatic when a confirmed attendee changes to "No", promoting the earliest waitlisted "Yes" by the time they answered.
- The host can close the event to further responses and can cancel the event.
- When canceled, invitees are notified via email.

### Operational constraints

- Email is required at minimum for sending invitations (unique link distribution) and cancellation notifications.
- The system must behave correctly under concurrent RSVP changes (e.g., multiple invitees attempting “Yes” near capacity), because capacity and waitlisting must remain consistent.

### Organizational constraints

- This is an educational task with an explicit “design-first” workflow: the design doc is expected to be clear about scope boundaries and correctness rules before implementation.

## Facts, assumptions, and open questions (design-impacting)

Tags: [COR] correctness, [SEC] security/privacy boundary, [CONC] concurrency, [SCALE] scale/performance

### Confirmed facts

- Stack constraint: Spring Boot (Spring MVC + JPA) + PostgreSQL + React/TypeScript.
- [SEC] Invitee access: invitees do not sign in; they respond via a unique invite link.
- RSVP model: Yes / No / Maybe; invitee enters their name when responding.
- [COR] Time rule: invitees can change RSVP until event start; after start, all RSVPs are locked.
- [COR] Time representation decision: event start time is stored as an exact instant derived from the host’s chosen local time.
- [COR][CONC] Capacity/waitlist rules (when capacity is set and full):
  - "Yes" responses beyond capacity are waitlisted.
  - Waitlist applies only to "Yes".
  - "Maybe" counts toward capacity similarly to "Yes".
  - Promotion order is by the time the invitee answered (earliest waitlisted "Yes" is promoted first).
- Host controls: host can close the event to further responses and can cancel the event.
- Notifications: invitation distribution and cancellation notifications use email.
- [SCALE] Dashboard: host dashboard is seconds-level real-time push.

### Working assumptions

- [SEC] Host authorization mechanism is still a design choice; the current working idea is a host-only secret link/token for host management access.
- [SCALE] The mechanism for seconds-level real-time push (e.g., which push technology) is not yet decided.
- Email delivery is treated as required behavior, but the specific provider/integration details are not yet decided.

### Open questions (materially affect design)

- [COR] Time conversion source: when deriving the event start instant from the host’s chosen local time, what timezone source is authoritative (explicit host timezone setting vs the creating browser’s timezone)?
  - Why it matters: determines when RSVPs lock; the wrong timezone source can lock too early/late (especially around DST) and breaks correctness.
- [COR] Capacity edge case: if "Maybe" counts toward capacity and capacity is full, what happens when an invitee submits "Maybe" (accepted, rejected, or handled in some other explicit way)?
  - Why it matters: affects invariant enforcement and user-visible outcomes at capacity; ambiguity here leads to inconsistent seat accounting and unexpected behavior.
- [SEC] Host identity enforcement: how is “the event creator is the host” enforced across sessions/devices (especially given invitees have no sign-in)?
  - Why it matters: defines the trust boundary for host-only actions (invite/close/cancel/view); if this is weak, unauthorized users could manage events.
- [SCALE] Real-time push expectation: what is acceptable behavior when push is unavailable (e.g., degrade to manual refresh vs hard requirement)?
  - Why it matters: determines whether the system needs a robust push channel with reconnection/backoff and whether a polling fallback is acceptable.

## What counts as solving it (definition of done)

- A host can create an event with title, description, date/time, location, and optional max-capacity.
- The event creator is the host for that event.
- A host can invite people by email, producing a unique link per invitee.
- An invitee can open their unique link, enter their name, and submit an RSVP: Yes / No / Maybe.
- [COR] RSVP changes are allowed up until the event start time; after the start time, all RSVPs are locked.
- [SCALE] The host can view a live (real-time push, seconds-level) dashboard showing counts and a list of invitees with their current RSVP.
- [COR][CONC] If capacity is set and full:
  - New "Yes" RSVPs are waitlisted.
  - "Maybe" counts toward capacity similarly to "Yes".
  - Waitlist applies only to "Yes" responses.
- [COR][CONC] If a confirmed attendee changes their RSVP to "No", the earliest waitlisted "Yes" (by the time they answered) is promoted to confirmed.
- The host can cancel the event; when canceled, invitees are notified via email.
- [COR] The host can close the event to further responses; when closed, invitees cannot change their RSVP.
- [COR] Event start time is stored as an exact instant derived from the host’s chosen local time.

## Invariants (must always hold)

### Business invariants

- RSVP values are limited to exactly: Yes / No / Maybe.
  - Could be violated by: accepting arbitrary strings/values from the client or storing an unknown enum value.
  - Likely controls: server-side validation + strong typing/enum at the API boundary; defensive DB representation.
- [COR] If current time is at/after the event start instant, all RSVP changes are rejected (RSVPs are locked).
  - Could be violated by: using inconsistent time sources, wrong timezone conversion, or checking against local time instead of the stored instant.
  - Likely controls: compare against the stored event start instant using a single authoritative clock source; tests around boundary conditions (incl. DST scenarios).
- [COR] If the event is closed, invitees cannot submit or change RSVPs while closed.
  - Could be violated by: missing/incorrect closed-state checks on some endpoints or stale client UI bypassing server checks.
  - Likely controls: enforce closed-state checks server-side for all RSVP writes; keep the closed flag in the source of truth.
- [COR] When capacity is set and full, new "Yes" RSVPs do not become confirmed; they are waitlisted.
  - Could be violated by: race conditions at capacity, or applying the capacity check after persisting "Yes" as confirmed.
  - Likely controls: atomic capacity evaluation + write (transactional update); re-check invariants at commit time.
- [COR] Waitlist applies only to "Yes" responses.
  - Could be violated by: accidentally waitlisting "Maybe" or treating "No" as a waitlistable state.
  - Likely controls: explicit state machine/conditional logic that only creates waitlist status for "Yes".
- [COR] "Maybe" counts toward capacity similarly to "Yes" (but the exact edge behavior at full capacity is an open question).
  - Could be violated by: inconsistent counting (some code paths count "Maybe", others don’t) or leaving the edge rule undefined.
  - Likely controls: define and document the edge behavior explicitly; centralize capacity accounting logic; add tests for "Maybe" counting.
- [COR][CONC] If a confirmed attendee changes to "No", exactly one waitlisted "Yes" is promoted next, and promotion order is by the time they answered (earliest first).
  - Could be violated by: promoting multiple waitlisted entries for one freed seat; using client time for ordering; non-deterministic tie handling.
  - Likely controls: server-assigned ordering timestamp/sequence; single atomic promotion operation inside the RSVP change transaction; deterministic tie-break rule.
- The host can cancel an event and cancellation notifications are sent via email (delivery success is outside the system boundary).
  - Could be violated by: persisting cancellation but never initiating emails; or initiating emails without persisting cancellation.
  - Likely controls: make cancel a durable state change; use a reliable async email dispatch pattern (e.g., queued/outbox + retries) so “send requested” is preserved.

### Data integrity invariants

- Every event has exactly one stored start instant (the value used for lock checks) [COR].
  - Could be violated by: storing multiple competing time fields and checking different ones in different places.
  - Likely controls: a single canonical field for lock comparisons; API/DB constraints to ensure it is always present.
- Capacity, when present, is a non-negative integer.
  - Could be violated by: accepting negative values or non-integers from the client, or arithmetic underflow/overflow.
  - Likely controls: server-side validation; DB constraint (non-negative); bounds checks.
- Each invitee RSVP is associated to exactly one event via a specific invite link.
  - Could be violated by: broken foreign keys, link reuse across events, or copying invite links between events.
  - Likely controls: enforce relationships in persistence (FKs); generate links scoped to a single invite record.
- Each invite link uniquely identifies a single invite (and therefore a single event) [SEC].
  - Could be violated by: collisions in link generation or allowing multiple invites to share the same link.
  - Likely controls: uniqueness guarantee in storage; sufficiently large/random identifiers.
- For each invite, there is exactly one current RSVP state at any given time.
  - Could be violated by: modeling RSVPs as append-only without a clear “current”, or concurrent writes producing duplicates.
  - Likely controls: single-row “current RSVP” per invite (or explicit current pointer); idempotent update semantics.
- [COR] The timestamp/ordering field used for waitlist promotion is persisted and comparable across invites for a given event.
  - Could be violated by: using non-monotonic client timestamps, mixing timezones, or not persisting the ordering value.
  - Likely controls: server-generated timestamps/sequences; consistent type/precision; store in DB and index for promotion queries.
- Dashboard counts and the invitee list are consistent with the stored RSVP states (no “phantom” confirmed/waitlisted entries).
  - Could be violated by: computing counts from stale caches, push events without reconciling to DB, or partial updates.
  - Likely controls: compute dashboard view from the source of truth; if using push, ensure updates reflect committed state; periodic reconciliation if needed.

### Authorization invariants

- Host-only actions (invite, view dashboard, close, cancel) require host authorization; invitee links must not grant host privileges [SEC].
  - Could be violated by: treating possession of any event-related link as host access, or missing auth checks on host endpoints.
  - Likely controls: separate credentials for host vs invitee; consistent server-side authorization checks on every host action.
- RSVP submission/change requires possession of the unique invite link; requests without a valid link are rejected without leaking whether an event/invite exists [SEC].
  - Could be violated by: endpoints that accept eventId/invitee email directly, or different error messages for “not found” vs “unauthorized”.
  - Likely controls: capability-style link requirement; uniform error handling that avoids existence leaks.
- A valid invite link grants access only to that invitee’s RSVP context for that event (not to other invites/events) [SEC].
  - Could be violated by: using the link to enumerate other invitees, or allowing mutation of other invites via guessed IDs.
  - Likely controls: always scope reads/writes to the invite identified by the link; never accept arbitrary invite IDs from the client for RSVP writes.
- The system does not rely on invitee authentication/sign-in for access control (by constraint).
  - Could be violated by: introducing implicit user accounts or session identity assumptions that block intended access.
  - Likely controls: keep RSVP access capability-based; ensure flows work in a fresh browser/session with only the invite link.

### Concurrency invariants

- [COR][CONC] When capacity is set, the number of confirmed "Yes" responses never exceeds capacity, even under concurrent submissions.
  - Could be violated by: two concurrent transactions both observing “one seat left” and confirming.
  - Likely controls: transactional enforcement around seat allocation; a DB-level mechanism that prevents over-confirmation (exact approach TBD).
- [CONC] A single RSVP submission/change results in one consistent final state (no partial application of capacity/waitlist/promotion rules).
  - Could be violated by: multi-step writes without a transaction (e.g., update RSVP then separately promote), or failures between steps.
  - Likely controls: wrap related writes in a single atomic unit; retry-safe operations.
- [COR][CONC] Waitlist promotion is deterministic (based on the stored ordering field) and does not promote the same waitlisted "Yes" twice.
  - Could be violated by: non-deterministic ordering queries, or concurrent promotions selecting the same next-in-line.
  - Likely controls: deterministic ordering (including tie-break); atomic “select + mark promoted” semantics.
- [CONC] Concurrent RSVP writes cannot produce contradictory states (e.g., an invite simultaneously marked confirmed and waitlisted).
  - Could be violated by: representing state in multiple fields/tables without consistency constraints, or concurrent updates racing.
  - Likely controls: model RSVP status as a single consistent state; DB constraints and transactional updates to prevent invalid combinations.

### Tenant isolation invariants

- No multi-tenant requirement is specified; by default the system is treated as single-tenant.
  - Could be violated by: accidentally introducing tenant identifiers/assumptions inconsistently, leading to ambiguous scoping.
  - Likely controls: explicitly document “single-tenant” as the baseline; if tenancy is added later, add explicit tenant scoping everywhere.
- [SEC] Regardless of tenancy, all capabilities (invite links, host management access) are scoped to a single event and must not allow cross-event data access.
  - Could be violated by: using a capability token to query by eventId supplied by the client, or not binding tokens to a specific event.
  - Likely controls: bind capability tokens to a specific event/invite in storage; enforce scoping server-side on every query.

## High-level architecture (no-code)

This architecture is derived from the confirmed facts + explicit assumptions above. Where a choice is not yet decided, it is called out as a design decision/open question rather than assumed.

### Components (and primary responsibilities)

- **Web UI (React/TypeScript)**
  - Responsibility: provide two views (invitee RSVP via unique link; host management + dashboard).
  - Reason to exist: the feature is inherently user-driven (host creates/manages; invitee responds) and is constrained to React/TypeScript.
- **Backend API (Spring Boot: Spring MVC + JPA)**
  - Responsibility: be the single enforcement point for correctness + authorization (lock/close rules, capacity/waitlist, deterministic promotion, host-only actions) [COR][SEC][CONC].
  - Reason to exist: browsers are not trusted for rule enforcement; concurrency safety and security boundaries must be enforced server-side.
- **Database (PostgreSQL)**
  - Responsibility: system of record for events, invites/links, current RSVP state, and ordering timestamps used for promotion [COR][CONC].
  - Reason to exist: correctness and concurrency invariants require a durable source of truth.

### External integrations (exist because requirements cross the system boundary)

- **Email delivery integration (provider TBD)**
  - Responsibility: deliver invitation emails (unique links) and cancellation notifications.
  - Reason to exist: email is a stated operational requirement; delivery is outside the system’s direct control.
- **Real-time update transport (push mechanism TBD)**
  - Responsibility: deliver seconds-level updates to the host dashboard [SCALE].
  - Reason to exist: seconds-level “real-time push” is a stated product constraint; the transport choice remains an open decision.

### Ownership & trust boundaries

- **Browser vs server boundary**
  - Browsers (host/invitee UIs) are untrusted for correctness: all rule enforcement happens in the backend API [COR][CONC].
- **Capability boundary (no invitee sign-in)**
  - Invitee access is capability-based: possession of a unique invite link grants access to that invite’s RSVP context [SEC].
  - Host access is also expected to be capability-based per the current working assumption (host-only secret link/token), but the enforcement mechanism is not yet finalized [SEC].
- **System vs external providers**
  - Email delivery and push delivery are outside the system’s direct control; the system can initiate sends/publishes but cannot guarantee delivery [SCALE].

### Flow boundaries (API surface at a high level)

- **Invitee-facing API surface (capability: invite link)**
  - Resolve invite link → show current state.
  - Submit RSVP / change RSVP → apply lock/close + capacity/waitlist rules → persist → publish dashboard update.
- **Host-facing API surface (capability: host management access)**
  - Create event (includes deriving/storing event start instant) [COR].
  - Invite: generate unique invite links and initiate invitation emails.
  - Close: set closed state.
  - Cancel: set canceled state + initiate cancellation emails.
  - Read dashboard state and subscribe to real-time updates.

### Key design choices (explicitly bounded by facts/assumptions)

- **Time as an instant for lock checks** (confirmed): all lock decisions are based on the stored event start instant [COR].
- **Authoritative timezone source** (open question): needed to derive the event start instant from host-entered local time [COR].
- **Host identity / authorization mechanism** (assumption + open question): current working idea is a host-only secret link/token; specifics determine the security boundary for host-only actions [SEC].
- **Seconds-level push mechanism + fallback expectation** (assumption + open question): push technology is not decided; acceptable behavior when push is unavailable is not decided [SCALE].
- **Concurrency control for capacity + promotion** (required by constraints, mechanism TBD): the backend + DB must preserve the capacity/waitlist invariants under concurrent writes [COR][CONC].
- **Correctness-first vs latency tradeoff** (implicit, but real): for hot events near capacity, preserving invariants can require contention/serialization/retries; RSVP tail latency may spike during bursts [COR][CONC][SCALE].
## Actors and workflows

### Actors

- **Host**: creates and manages an event (invite, view dashboard, close, cancel).
- **Invitee**: receives a unique link, enters name, responds Yes/No/Maybe, and may change response until lock.
- **Web UI**: user interface for host flows and invitee RSVP via unique link.
- **Backend API**: enforces business rules (time lock, capacity/waitlist), authorization, and persistence.
- **Database (PostgreSQL)**: source of truth for events, invites, RSVPs, and timestamps.
- **Email delivery integration/provider**: delivers invitations (unique links) and cancellation notifications.
- **Real-time update transport**: delivers seconds-level updates to the host dashboard.

### User-facing flows

- **Create event (Host)**
  - Trigger: host submits the create-event form.
  - Major steps: enter event details (including local date/time, optional capacity) → submit → receive a link/token/handle for host management access (mechanism is a design choice) [SEC].
  - State changes: new event exists; event start instant is stored; host management access credential/handle exists per chosen design [COR][SEC].
  - Dependencies: backend API; database; time conversion rules (timezone source is an open question) [COR]; host authorization mechanism [SEC].
- **Invite people by email (Host)**
  - Trigger: host submits a set of invitee emails for an event.
  - Major steps: validate host access [SEC] → generate a unique link per invitee → send invitation emails.
  - State changes: invite records exist (with unique links); outbound invitation emails are initiated.
  - Dependencies: host authorization mechanism [SEC]; backend API; database; email delivery system/provider.
- **RSVP via unique link (Invitee)**
  - Trigger: invitee opens their unique link and submits an RSVP.
  - Major steps: validate invite link [SEC] → capture invitee name → accept RSVP choice (Yes/No/Maybe) → enforce closed + locked rules [COR] → apply capacity/waitlist rules if capacity is set [COR][CONC].
  - State changes: invitee’s current RSVP is stored/updated; answered-at timestamp used for ordering is stored; if "Yes" beyond capacity then waitlist status is stored; if "Maybe" at capacity, behavior is an open question [COR][CONC].
  - Dependencies: invite link validation [SEC]; backend API; database; event start instant/time rule [COR]; capacity/waitlist enforcement [COR][CONC].
- **Change RSVP (Invitee)**
  - Trigger: invitee submits a new RSVP choice using their unique link.
  - Major steps: validate invite link [SEC] → check event not closed [COR] → check current time is before event start instant [COR] → apply capacity/waitlist and promotion rules as needed [COR][CONC].
  - State changes: invitee’s current RSVP is updated; if a confirmed attendee becomes "No", the earliest waitlisted "Yes" is promoted [COR][CONC].
  - Dependencies: invite link validation [SEC]; backend API; database; event start instant/time rule [COR]; capacity/waitlist enforcement + promotion ordering [COR][CONC].
- **Live dashboard (Host)**
  - Trigger: host opens the dashboard for an event.
  - Major steps: validate host access [SEC] → load current counts + invitee list → subscribe to seconds-level real-time updates via the real-time update transport [SCALE].
  - State changes: none to core event state (read-only view); an active subscription/connection exists while viewing.
  - Dependencies: host authorization mechanism [SEC]; backend API; database; real-time update transport [SCALE].
- **Close responses (Host)**
  - Trigger: host selects “close responses”.
  - Major steps: validate host access [SEC] → mark event as closed → subsequent invitee RSVP submits/changes are rejected while closed [COR].
  - State changes: event closed flag/state is set.
  - Dependencies: host authorization mechanism [SEC]; backend API; database; (optional) dashboard real-time update [SCALE].
- **Cancel event (Host)**
  - Trigger: host selects “cancel event”.
  - Major steps: validate host access [SEC] → mark event as canceled → send cancellation notification emails.
  - State changes: event canceled flag/state is set; outbound cancellation emails are initiated.
  - Dependencies: host authorization mechanism [SEC]; backend API; database; email delivery system/provider; (optional) dashboard real-time update [SCALE].

### Internal system flows (server-side / rule enforcement)

- **Derive and store event start instant**
  - Trigger: event creation (and any future event-time edits, if allowed).
  - Major steps: interpret host input local date/time → derive exact instant → persist instant and any required interpretation context.
  - State changes: event start instant is stored/updated [COR].
  - Dependencies: authoritative timezone source (open question) [COR]; backend API; database.
- **Generate and validate access tokens/links**
  - Trigger: invite creation (invite links) and host management access creation (host access).
  - Major steps: generate unique values → persist association to event/invitee → validate on each use → authorize requested action.
  - State changes: invite link identifiers exist; host management access identifier exists (exact mechanism is a design choice) [SEC].
  - Dependencies: backend API; database; chosen host authorization mechanism [SEC].
- **Record RSVP changes with ordering**
  - Trigger: invitee submits an RSVP or change.
  - Major steps: validate request context (invite link, event state) → persist current RSVP → persist answered-at timestamp used for ordering [COR][CONC].
  - State changes: RSVP state is updated; ordering timestamp is stored.
  - Dependencies: backend API; database; event state/time rule checks [COR].
- **Enforce capacity and waitlist invariants**
  - Trigger: any RSVP submit/change that could affect counts or waitlist ordering.
  - Major steps: compute current confirmed count → decide confirmed vs waitlisted for "Yes" → apply "Maybe" capacity rule → when a confirmed becomes "No", promote earliest waitlisted "Yes" by answered time [COR][CONC].
  - State changes: invitees move between confirmed/waitlisted states for "Yes"; promotions update at least two invitees’ statuses in one action [COR][CONC].
  - Dependencies: database concurrency control/transactions (mechanism TBD) [CONC]; consistent time ordering field [COR]; "Maybe at capacity" behavior (open question) [COR].
- **Publish dashboard updates**
  - Trigger: state change that affects what the host sees (RSVP updates, promotions, close, cancel).
  - Major steps: compute updated view model (counts + list) → publish an update to the real-time update transport.
  - State changes: none to persisted event state; real-time update messages/events are emitted [SCALE].
  - Dependencies: backend API; real-time update transport [SCALE].
- **Send required emails**
  - Trigger: invite creation (invitations) and event cancellation (cancellation notifications).
  - Major steps: assemble email payloads → hand off to delivery provider/integration.
  - State changes: outbound emails are initiated.
  - Dependencies: backend API; email delivery system/provider.

### Background flows (asynchronous / continuous)

- **Email dispatch**
  - Trigger: outbound emails are initiated (invite or cancellation).
  - Major steps: provider/integration delivers email (specifics not decided).
  - State changes: none to core event state; delivery outcome may be outside the system boundary.
  - Dependencies: email delivery system/provider.
- **Real-time push connection management**
  - Trigger: host dashboard is opened and subscribes to updates.
  - Major steps: establish/maintain connection → deliver updates within seconds when updates are published [SCALE].
  - State changes: none to core event state; transient connection/subscription state exists while connected.
  - Dependencies: real-time update transport; client connectivity/network conditions [SCALE].

### Failure flows (what happens when things go wrong)

- **Email delivery problems**
  - Trigger: email provider delays, rejects, or fails delivery.
  - Major steps: system initiates send → provider does not deliver promptly/at all → recipient does not receive expected message.
  - State changes: invitation/cancellation state still exists in the system; email receipt is not guaranteed by the system.
  - Dependencies: email delivery system/provider.
- **Invalid or unusable invite link**
  - Trigger: invitee attempts to use an invalid link, or uses a valid link in a disallowed state.
  - Major steps: validate link and event state → reject request safely [SEC] → communicate that RSVP is not accepted/changed due to lock/close [COR].
  - State changes: no RSVP change is persisted for the rejected request.
  - Dependencies: invite link validation [SEC]; backend API; database; event state/time rules [COR].
- **Real-time updates unavailable / degraded**
  - Trigger: real-time update transport is down/unreachable or client loses connectivity.
  - Major steps: dashboard does not receive seconds-level updates.
  - State changes: none to persisted event state; dashboard view may become stale.
  - Dependencies: real-time update transport and network connectivity [SCALE]; fallback expectation is an open question [SCALE].
- **Concurrency at capacity**
  - Trigger: multiple RSVP submits/changes occur near-simultaneously when capacity is near/full.
  - Major steps: apply capacity rules in a concurrency-safe way so only allowed confirmations occur; deterministically assign waitlist and promotions [COR][CONC].
  - State changes: final RSVP + waitlist states reflect a consistent, non-overbooked outcome [COR][CONC].
  - Dependencies: database concurrency control/transactions (mechanism TBD) [CONC]; stable ordering timestamps [COR].
- **Time interpretation issues**
  - Trigger: incorrect timezone source or DST interpretation during conversion.
  - Major steps: store an incorrect event start instant → lock/unlock checks use the wrong instant.
  - State changes: event start instant is incorrect; RSVP acceptance/rejection behavior becomes incorrect [COR].
  - Dependencies: authoritative timezone source decision (open question) [COR].
- **Host access compromise**
  - Trigger: host management access credential is leaked or guessed.
  - Major steps: unauthorized user presents host credential → system treats them as host → host-only actions become possible.
  - State changes: unauthorized changes may be persisted (invite/close/cancel), depending on what actions are taken [SEC].
  - Dependencies: chosen host identity/authorization mechanism and its mitigations [SEC].

## Data ownership & state model

Principle: the **Database (PostgreSQL)** is the source of truth for persisted business state. The **Backend API** is the only writer of that state and the only enforcer of business rules. The **Web UI** displays state and submits intents; it is not authoritative.

For each entity/stateful concept below:
- **Source of truth**: where the canonical value lives.
- **Mutations**: who can change it (and via what kind of flow).
- **Reads**: who reads it and how.
- **Derived state**: values computed from sources of truth (not independently authoritative).

### Where truth actually lives (and where ownership is currently blurred)

- **Truth lives in the database** for the durable domain state:
  - Event fields (including event start instant, closed/canceled flags, capacity if present).
  - Invite records (including the invite link capability identifier and its event binding).
  - Current RSVP per invite, plus the ordering field used for deterministic promotion.
  - Any persisted seat allocation/waitlist status *if the design chooses to persist it*.

- **The Backend API owns mutations** of that domain state:
  - Host-only mutations: create event, create invites, close, cancel.
  - Invitee mutations: set/change RSVP (and invitee display name captured as part of that flow).

- **The Web UI never owns truth**:
  - It can cache/display snapshots and stream real-time updates, but it must not be treated as authoritative for counts, allocations, or event state.

- **Ownership is blurred / undecided (open questions or design choices)**:
  - [SEC] **Host authorization mechanism**: host-only authority exists as a requirement, but the credential format, validation, and lifecycle are not finalized.
  - [COR] **Time interpretation authority**: the event start instant is stored, but the authoritative timezone source for conversion from host-entered local time remains undecided.
  - [COR][CONC] **Seat allocation model**: whether confirmed/waitlisted is persisted as first-class state vs derived deterministically (both are viable; invariants must hold either way).
  - [COR][CONC] **“Maybe” at capacity**: allocation/counting rules are incomplete until this is decided.
  - [SCALE] **Real-time updates delivery semantics**: real-time updates are derived from committed DB state, but what the system guarantees on reconnect/fallback is an open question.
  - **Email side effects**: invitation/cancellation sends are required, but whether “email sent/delivered” becomes tracked state (and where it lives) is not a confirmed requirement.

### Event (core entity)

- **Source of truth**: Database.
- **Mutations**: Backend API on host actions (create event; close; cancel). Invitees do not mutate event-level fields.
- **Reads**:
  - Backend API reads for all rule checks (lock/close/capacity).
  - Web UI reads via Backend API for host dashboard and invitee views.
- **Derived state**:
  - [COR] **Locked** is derived from `now >= eventStartInstant` (not a stored flag).
  - **IsFull** is derived from capacity and current RSVP allocation/counting rules.
  - Host dashboard view (counts + list) is derived from event + invite + RSVP records.
- **Notes**:
  - [COR] Event start instant is stored; the authoritative timezone source used to derive it is an open question.
  - Cancellation semantics for RSVP acceptance after cancel are not specified here; only “send cancellation email” is required.

### Invite (per-invitee capability)

- **Source of truth**: Database.
- **Mutations**: Backend API on host invite actions (create invites; generate/store unique invite links).
- **Reads**:
  - Backend API validates invite link → resolves to the invite/event context.
  - Web UI accesses an invite context only via the unique link + Backend API.
- **Derived state**:
  - None required; the invite link itself is a capability identifier that maps to an invite record.
- **Notes**:
  - [SEC] Invite links must be scoped to exactly one invite (and therefore one event).

### Invitee-provided display identity (name)

- **Source of truth**: Database (stored alongside the invitee’s RSVP context).
- **Mutations**: Backend API when an invitee submits/updates an RSVP (name is captured/updated as part of that flow).
- **Reads**: Host dashboard displays the stored name via Backend API.
- **Derived state**: None.
- **Notes**: This is not an authenticated identity; it is user-provided display data.

### RSVP (current response per invite)

- **Source of truth**: Database (one “current” RSVP state per invite).
- **Mutations**: Backend API on invitee RSVP submit/change, subject to lock + closed rules [COR].
- **Reads**:
  - Backend API reads to enforce capacity/waitlist and promotion.
  - Host dashboard reads via Backend API (counts + list).
  - Invitee view reads current RSVP via Backend API.
- **Derived state**:
  - Capacity counting inputs are derived from RSVP choice (Yes/Maybe count toward capacity; No does not).
- **Notes**:
  - [COR][CONC] A server-assigned ordering timestamp/field for “time answered” is required for deterministic promotion ordering.

### Seat allocation / waitlist status (stateful concept)

- **Source of truth**: Database, via one of these approaches (design choice; must preserve invariants):
  - Persist explicit allocation status (e.g., for a “Yes”: confirmed vs waitlisted), or
  - Derive allocation deterministically from RSVP choices + answered-at ordering + capacity.
- **Mutations**: Backend API as part of RSVP writes and promotion logic [COR][CONC].
- **Reads**: Backend API to compute dashboard view and to decide promotion; Web UI via Backend API.
- **Derived state**:
  - Waitlist position is derived from answered-at ordering among waitlisted “Yes”.
- **Notes**:
  - [COR] “Maybe at capacity” behavior is an open question; allocation rules are incomplete until it is decided.

### Host management access credential (capability; assumption)

- **Source of truth**: Database (association between an event and whatever host credential is chosen).
- **Mutations**: Backend API at event creation (issue/store) and on validation/authorization checks.
- **Reads**: Backend API on every host-only action (invite/view dashboard/close/cancel) [SEC].
- **Derived state**: None.
- **Notes**:
  - [SEC] The exact enforcement mechanism and lifecycle are an open question; until decided, “host-only” is protected only by intent.

### Email notifications (invitations, cancellations)

- **Source of truth**: Event and Invite state in the database; email delivery itself is outside the system boundary.
- **Mutations**: Backend API initiates outbound sends on invite creation and event cancellation.
- **Reads**: Not required for core correctness; delivery outcomes may or may not be observable depending on provider.
- **Derived state**:
  - Any “email sent/delivered” tracking is optional and not part of the confirmed facts.

### Real-time dashboard updates (push)

- **Source of truth**: Database is canonical; push messages are not authoritative.
- **Mutations**: Backend API publishes updates when committed state changes.
- **Reads**: Host Web UI receives updates via the real-time update transport [SCALE].
- **Derived state**:
  - Messages/events are derived from committed DB state (counts + list deltas or snapshots).
- **Notes**:
  - [SCALE] Expected behavior when push is unavailable is an open question.

## Correctness and concurrency notes (first pass)

This section calls out the concrete ways concurrency, retries, and asynchronous side effects can violate the invariants described earlier, tied to the flows in this design.

### Stale reads (read/write timing gaps)

- **Where it happens**: host dashboard counts/list; invitee view after submitting RSVP; reconnect after real-time update transport disruption.
- **What can go wrong**:
  - Dashboard shows incorrect fullness or waitlist status because it is behind the committed DB state.
  - Invitee sees an old RSVP state and submits a change based on stale UI, causing surprise when the server rejects due to close/lock.
- **Likely controls**:
  - Treat DB as canonical; compute dashboard view from committed state.
  - After any RSVP write, the UI should re-fetch authoritative state rather than assuming local state is final.
  - Consider including a monotonically increasing server version (e.g., `updatedAt` / sequence) in dashboard snapshots so clients can detect missed updates (implementation choice).

### Duplicate requests (client retries / double submits)

- **Where it happens**: RSVP submit/change; host close/cancel; host invite-by-email.
- **What can go wrong**:
  - RSVP “double submit” creates inconsistent ordering (answered-at) or temporarily violates capacity/promotion invariants if each request is treated as a fresh action [COR][CONC].
  - Close/cancel executed twice triggers repeated side effects (duplicate cancellation emails) or conflicting UX.
  - Invite-by-email retried sends duplicate invitations and may create duplicate invite records if not constrained.
- **Likely controls**:
  - Model RSVP as a single “current” row per invite and make writes idempotent (retry overwrites deterministically rather than creating a second RSVP) [COR].
  - Use database uniqueness constraints for invite capability uniqueness.
  - For side effects (email), treat sends as at-least-once and deduplicate by a server-assigned send intent identifier (implementation choice).

### Conflicting updates (write/write races)

- **Where it happens**:
  - Multiple invitees RSVP concurrently near capacity.
  - One invitee changes to “No” while others are submitting “Yes”.
  - Host closes responses while invitees submit/change RSVP.
  - Host cancels while invitees submit/change RSVP.
- **What can go wrong**:
  - Overbooking (confirmed count exceeds capacity) if capacity checks are non-atomic [CONC].
  - Non-deterministic promotion (two waitlisted promoted into one seat; incorrect earliest-by-time ordering) [COR][CONC].
  - RSVP accepted on one endpoint but rejected on another due to inconsistent closed/canceled/locked checks [COR].
- **Likely controls**:
  - Perform RSVP write + capacity/waitlist/promotion decision in a single transaction with explicit concurrency control (mechanism TBD) [CONC].
  - Ensure a single authoritative ordering field for promotion (server/DB-assigned answered-at) and never accept client time as authoritative [COR][CONC].
  - Centralize rule checks in the Backend API so close/cancel/lock semantics are applied consistently for every RSVP write path [COR].

### Out-of-order events (asynchronous delivery)

- **Where it happens**: real-time dashboard updates delivered over the real-time update transport.
- **What can go wrong**:
  - Updates arrive out of order (or are duplicated) and the dashboard regresses to an older state if it applies updates blindly.
- **Likely controls**:
  - Publish updates derived from committed DB state only after commit.
  - Include ordering metadata (e.g., event version/updated-at) so clients can discard older updates and request a fresh snapshot when uncertain (implementation choice).

### Unclear mutation authority (who is allowed to change what)

- **Where it appears**:
  - Host-only actions (invite, close, cancel) depend on an authorization mechanism that is not finalized.
  - Derived state (dashboard counts, waitlist positions) could be mistakenly treated as something the UI can “set”.
- **What can go wrong**:
  - Host-only endpoints become callable by anyone with a guessed/leaked identifier, violating invariants [SEC].
  - Multiple writers emerge (e.g., UI tries to maintain counts), creating divergence between displayed state and canonical state.
- **Likely controls**:
  - Backend API is the **only mutator** of persisted state; UIs only submit intents.
  - Capability validation is always performed server-side and always scoped to exactly one event/invite [SEC].
  - Treat dashboard state as derived/read model; never accept client-supplied counts/allocations.

### Vulnerable areas → what can go wrong → likely control class

- **Invitee RSVP submit/change endpoint (per invite)**
  - What can go wrong: double-submit/retry creates multiple “answers” or unstable ordering; concurrent writes overwrite each other unexpectedly.
  - Likely control class:
    - **Idempotency**: treat the write as “set current RSVP for this invite”.
    - **Unique constraint**: enforce “one current RSVP row per invite” (or equivalent schema approach).

- **Capacity decision + waitlist assignment (near/full capacity)**
  - What can go wrong: overbooking if the check+write is not atomic; two invitees confirmed into the last seat.
  - Likely control class:
    - **Transaction**: compute+apply allocation within one DB transaction.
    - **Queue serialization** (alternative): serialize all RSVP writes per event (e.g., single-flight per event) if chosen over fine-grained DB locking.

- **Promotion when a confirmed attendee becomes “No”**
  - What can go wrong: two promotions for one freed seat; promotion order differs from “earliest waitlisted by answered-at”.
  - Likely control class:
    - **Transaction**: update the “No” RSVP and the promoted invitee allocation atomically.
    - **Version check** (alternative): optimistic concurrency on the event/allocation state to detect races.

- **Host close responses vs invitee RSVP writes**
  - What can go wrong: RSVP accepted “after close” on one request path but rejected on another; close races with RSVP writes.
  - Likely control class:
    - **Transaction**: ensure the close flag read and RSVP write are consistent at the point of decision.
    - **Version check**: optimistic locking on event state so concurrent close vs RSVP produces a detectable conflict (implementation choice).

- **Host cancel vs invitee RSVP writes**
  - What can go wrong: RSVP accepted while cancel is in flight; cancellation side effects sent without the cancel state being committed.
  - Likely control class:
    - **Transaction**: commit cancel state before triggering side effects.
    - **Another explicit control**: outbox-style “commit state + record send-intent in DB” so email sends are driven from committed intents (implementation choice).

- **Invite-by-email flow (host provides emails, system generates invite capabilities)**
  - What can go wrong: retry creates duplicate invites for the same email; multiple emails sent with different links.
  - Likely control class:
    - **Idempotency**: treat “invite this list” as a set operation per event.
    - **Unique constraint** (optional, if desired): prevent multiple invite records per (event, invitee email).

- **Email sending (invitations and cancellations)**
  - What can go wrong: worker retry sends duplicates; partial failure causes “sent twice” or “sent without corresponding state”.
  - Likely control class:
    - **Idempotency**: provider request dedup keyed by a send-intent id.
    - **Unique constraint**: one send-intent per (event, invite, email type) if tracked.
    - **Queue serialization** (optional): serialize sends per event to bound duplication and ordering.

- **Real-time dashboard updates (out-of-order / duplicates)**
  - What can go wrong: dashboard applies updates out of order and regresses; duplicates cause flicker or miscounts.
  - Likely control class:
    - **Version check**: include monotonic version/updated-at so clients ignore stale updates.
    - **Another explicit control**: periodic snapshot refresh from API (treat push as advisory).

## Trust boundaries & security notes (first pass)

This section ties security notes directly to the flows in this design and avoids assuming mechanisms that are not confirmed.

### Trust Entry Points

- **Host calls into Backend API** for: create event (local date/time, capacity), invite-by-email (invitee emails), close, cancel, and dashboard reads.
- **Invitee calls into Backend API** for: resolve invite link + submit/change RSVP (invite capability + name + RSVP choice).
- **Backend API calls out to Email provider** when invites are created and when events are canceled.
- **Host subscribes to real-time update transport** for seconds-level dashboard updates (subscription/connect/reconnect behavior is a trust entry point).

### Authorization Enforcement

- **Invitee flow (RSVP via unique link / Change RSVP)**: Backend API must validate the invite capability and scope it to exactly one invite/event for every read/write [SEC].
- **Host flows (invite/dashboard/close/cancel)**: Backend API must validate host management access for the event on every host-only action [SEC].
- **Real-time dashboard updates**: subscription to event updates must be gated by the same host authorization as dashboard reads (do not rely on “already on the page”).
- **Rule enforcement must never live in the UI**: close/lock/capacity checks must be enforced server-side on every RSVP write [COR][CONC].
- **Do not accept caller-supplied scope**: if an endpoint includes a capability token, the server must derive event scope from the token→(invite,event) binding, not from client-supplied ids [SEC].

### Tenant Isolation

- **Isolation unit is the event**: invite and host capabilities must never allow cross-event reads/writes.
- **Scope binding lives in the database**: capabilities must resolve to exactly one invite/event and all queries must be constrained by that derived scope [SEC].
- **High-risk overreach points**: host dashboard (full roster) and host invite-by-email flow (invitee email list), plus any endpoint that mixes capability tokens with client-provided identifiers.

### Sensitive Data / Privileged Operations

- **Sensitive data in these flows**:
  - Invite link capability token (bearer access to one invite’s RSVP context).
  - Host management access credential/handle (assumption; bearer access to host-only operations).
  - Invitee email addresses (host-provided in invite-by-email).
  - Invitee name + RSVP choice (invitee-provided; visible to host via dashboard).
- **Privileged operations (host-only)**: create invites/send invitation emails; view dashboard; close responses; cancel event + send cancellation emails.
- **Flow-specific leakage/over-trust risks**:
  - Invite-by-email distributes bearer capabilities: forwarding an invitation forwards RSVP authority for that invite.
  - Dashboard reveals the event roster: any host-auth failure exposes names/RSVPs.
  - Capability tokens as URLs (invite links): can leak via logs/analytics that record full URLs or via referrer propagation if pages load third-party resources (token hygiene is an open question).
  - If background workers are introduced (email senders/broadcasters/promoters), they must not bypass the same scoping + rule enforcement as the request path.

## Scalability and multi-tenancy notes (first pass)

This section identifies likely growth axes and bottlenecks for *this architecture and these flows* (create event, invite-by-email, RSVP/change, host dashboard + real-time updates), and what is sufficient now vs what likely requires architectural change later.

### Likely growth axes

- **Events over time**: total number of events stored (and their historical invites/RSVPs).
- **Invites per event**: how many invitees a host adds for a single event (drives DB row counts and email send volume).
- **RSVP write rate**: bursts of invitee submissions/changes, especially near capacity where allocation/promotion logic contends.
- **Host dashboard fan-out**: number of simultaneous dashboard viewers per event and per system (drives read load + real-time publish load).
- **Email volume**: invitation sends and cancellation sends; provider throughput/limits can dominate perceived performance.

### Likely first bottlenecks (and why)

- **Database contention on capacity/promotion**
  - Why: the correctness model requires atomic updates when near capacity; this concentrates concurrency on a small set of rows per event [CONC].
  - Symptom: RSVP latency spikes during bursts; increased transaction retries/timeouts.

- **Dashboard read amplification**
  - Why: dashboard view is counts + list derived from DB; frequent refreshes or reconnects can multiply reads.
  - Symptom: elevated DB read load and slower dashboard refresh under many concurrent hosts.

- **Real-time update transport fan-out**
  - Why: every RSVP/close/cancel can produce an update; transport must deliver within seconds to all subscribed dashboards [SCALE].
  - Symptom: lagging dashboards or dropped updates under load; server memory/CPU pressure from many concurrent subscriptions.

- **Email provider throughput / retry behavior**
  - Why: invites and cancellations are explicitly required; retries are common under transient failures.
  - Symptom: delayed invitations/cancellations and duplicate sends without careful dedup.

### Noisy-neighbor risks (tenant/event hot spots)

- **One “large/busy” event can dominate shared resources**
  - Where it shows up: per-event capacity/promotion transactions; dashboard reads; real-time updates.
  - What can go wrong: elevated lock contention or transaction retries for that event; contention spills into shared DB connection pools and slows unrelated events.

- **One host can trigger large outbound bursts**
  - Where it shows up: invite-by-email for large lists; cancel event for large invite lists.
  - What can go wrong: email provider throttling/backoff causes cascading retries and backlog; request latency increases if sends are done inline.

- **One event with many dashboards/subscribers**
  - Where it shows up: real-time update transport and update publish loop.
  - What can go wrong: update fan-out becomes CPU/memory hot spot; lag increases and updates may be dropped.

### What is likely sufficient initially (fits the current scope)

- **Stateless Backend API + single PostgreSQL** as the system of record, with correctness enforced by DB transactions/concurrency control.
- **Compute dashboard view from DB state** (canonical), and treat real-time updates as derived notifications.
- **At-least-once email sending** with dedup at the “send intent” level (implementation choice) rather than requiring exactly-once delivery.
- **Event-scoped isolation**: capabilities bound to a single event match the current design (no accounts/orgs are confirmed).

### What would likely require later architectural change

- **High write contention per event** (very large events or heavy bursts)
  - Likely change: explicit per-event serialization (queue) or redesigned allocation model to reduce hot-spot contention.

- **High dashboard concurrency / heavy read load**
  - Likely change: introduce a read-optimized projection (materialized/denormalized dashboard model) updated from writes, while keeping DB as source of truth.

- **Large-scale real-time fan-out**
  - Likely change: dedicated real-time infrastructure (pub/sub or brokered delivery designed for high fan-out) and explicit client versioning/reconciliation.

- **Multi-tenancy beyond “event”** (accounts/organizations/tenant-level isolation)
  - Likely change: introduce a tenant identity concept and enforce tenant scoping in every query and capability mapping; add tenant-aware limits/quotas to reduce noisy-neighbor impact; revisit data model and authorization boundaries.

- **Operational requirements around auditing/abuse** (not currently confirmed)
  - Likely change: add rate limiting, audit logs, and monitoring pipelines if/when such requirements are introduced.

## Rollout / migration notes (first pass)

This system has a few rollout-sensitive properties:
- **Capability links are long-lived identifiers** (invite links; assumed host management link/token), so changes must not strand already-issued links.
- **Correctness depends on DB-enforced invariants under concurrency**, so partial rollouts must not create split-brain rule enforcement.
- **Email + real-time updates are side effects** that can duplicate under retries; rollout needs explicit controls to prevent accidental blasts.

### Staged enablement (concrete rollout for this feature)

Rollout should be staged around the parts that create irreversible external impact: **issuing capability links**, **sending email**, and **publishing real-time updates**.

- **Stage 1 — Enable core API writes, keep side effects OFF**
  - Purpose: validate the correctness-critical paths (RSVP write, capacity/waitlist, promotion ordering, close/cancel, and time-based lock) against the real DB concurrency behavior.
  - Concrete gating: allow creating events/invites and storing RSVP state, but keep **invitation sends**, **cancellation emails**, and **dashboard push publishes** disabled.
  - What must be true before moving on:
    - An issued invite link, when opened, can submit RSVP and change RSVP until the stored **event start instant**.
    - Near-capacity concurrent “Yes” responses do not overbook; promotion order follows the stored **answered-at** ordering.
    - Closing and canceling an event block further RSVP writes consistently.

- **Stage 2 — Turn ON invitation email sends in allowlist/dry-run mode**
  - Purpose: validate the external dependency (email provider) and confirm that *real emails* contain correct links and do not leak secrets via logs/headers.
  - Concrete gating: only send invitations to an allowlisted set of recipient addresses/domains (or record send intents without delivery) until link formatting and provider behavior are confirmed.
  - What must be true before moving on:
    - The email body links resolve to the correct public base URL and the token is accepted by the backend.
    - Retries (manual resend / deploy restart) do not create uncontrolled repeated sends (either deduped by send intent or limited by allowlist during this stage).

- **Stage 3 — Turn ON cancellation emails and real-time dashboard publishes**
  - Purpose: validate the “blast radius” side effects (cancellation fan-out; dashboard push fan-out) and ensure the system stays correct even if the transport is flaky.
  - Concrete gating: keep kill switches so that cancellation sends and/or real-time publishes can be disabled independently without disabling RSVP correctness.
  - What must be true:
    - Canceling triggers email sends once per invitee (at-least-once is acceptable, but duplicates must be bounded/observable).
    - If real-time transport is down, the dashboard remains correct on refresh/poll (push is a convenience, DB state is authoritative).

### Backward compatibility constraints

- **Invite link stability**: if invite tokens/IDs are ever rotated or their encoding changes, the backend must support validating both old and new formats for the lifetime of already-sent emails.
- **Public base URL stability**: invite/host links embed an origin (domain) implicitly via where the email points. Changing the public base URL later needs either redirects or dual-host support; otherwise already-issued links will break.
- **Host access stability (open question)**: once a host management credential is issued, switching auth mechanisms later needs a dual-validation period or a deliberate invalidation story; otherwise hosts can lose access to in-flight events.
- **API/UI shape changes**: keep rollout tolerant of mixed versions so that older UIs can still load the dashboard and submit RSVP during staged enablement (even if email/push is disabled).

### Database migration needs (rollout-sensitive for correctness and latency)

- **Ordering and uniqueness constraints that underpin invariants**: rollout should assume there are DB-level guarantees that prevent duplicate invite tokens and support stable promotion ordering (by stored answered-at). Missing these is a rollout blocker because it turns “rare concurrency bug” into “eventually happens in prod”.
- **High-impact index builds**: indexes that support “earliest waitlisted Yes by answered-at” and dashboard list queries can cause write latency while building; plan for that impact.
- **Time storage correctness**: event start instant storage and any timezone-related fields are correctness-critical; a migration that changes representation must include a compatibility read path (old + new) until backfill completes.

### Feature flags / operational switches

- **Email sending kill switch**: allow disabling invitation/cancellation sends without disabling core RSVP writes.
- **Email allowlist/dry-run mode**: needed specifically because invitation/cancellation emails contain capability links and are the widest external blast radius.
- **Real-time publish kill switch**: allow disabling dashboard push while keeping refresh/poll working.
- **Promotion behavior flag (if needed)**: if “Maybe at capacity” or promotion rules are finalized late, gate the rule change so it can be turned on per environment/event without rewriting history.

### Rollback and incident concerns

- **Rollback reality**: DB schema rollbacks are usually not safe once data is written. Plan rollback as “deploy previous backend that still understands the new schema” (hence additive migrations).
- **Duplicate side effects on rollback/redeploy**: if the backend is redeployed or a worker restarts, email sends and push publishes may repeat; operationally, rollout should assume at-least-once behavior and include deduplication keyed to a stable send intent.
- **Capability leakage in logs during rollout**: ensure request logging/redaction is correct before exposing real users, because invite/host tokens in URLs are high-impact if they end up in logs/analytics.

### Operationally sensitive during rollout

- **Email volume step-function**: enabling “invite people by email” changes traffic shape (bursty outbound sends). Validate provider throttling behavior and set conservative initial limits (per event/per minute) to avoid account-level suppression.
- **Hot-event contention**: initial rollout should include load observation for a single large event (DB contention, lock wait times, retry rates), since that is the dominant failure mode for correctness-first capacity enforcement.
- **Detecting silent drift**: during rollout, prefer checks that directly reflect invariants in this domain (confirmed count never exceeds capacity; promotions follow answered-at order; closed/canceled events reject RSVP writes).

## Risks and failure notes (first pass)

This section lists production risks that arise specifically from this architecture (capability links, DB-enforced capacity/promotion, email sending, real-time dashboard updates) and from the explicit open questions/assumptions.

### Correctness risks

- **Overbooking / inconsistent waitlist promotion under concurrency**
  - How it fails: concurrent RSVP writes near capacity produce non-atomic “count then write” behavior.
  - Impact: confirmed count exceeds capacity; promotion order violates “earliest waitlisted by answered-at” [COR][CONC].

- **Inconsistent enforcement of closed/locked/canceled rules across endpoints**
  - How it fails: one RSVP path checks closed/locked/canceled differently than another.
  - Impact: invitees see “accepted” in one case and “rejected” in another; invariants become endpoint-dependent [COR].

- **Time interpretation errors (timezone/DST) lead to incorrect locking**
  - How it fails: wrong authoritative timezone source or incorrect conversion when storing event start instant.
  - Impact: RSVP acceptance/rejection occurs at the wrong times [COR].

- **Out-of-order / dropped real-time updates cause dashboard regression**
  - How it fails: dashboard applies an older update after a newer one, or misses updates after reconnect.
  - Impact: host sees incorrect counts/list until a refresh; “seconds-level” expectation fails [SCALE].

### Dependency risks (external systems)

- **Email provider delays/rejections/throttling**
  - How it fails: invitation/cancellation sends are required, but the provider may delay or reject deliveries.
  - Impact: invitees do not receive links promptly; cancellations may not reach recipients; retries can amplify duplicate-send risk.

- **Real-time update transport outage/degradation**
  - How it fails: transport is down/unreachable or client connectivity is poor.
  - Impact: host dashboard becomes stale; acceptable fallback behavior is an open question [SCALE].

### Operational risks (running the system)

- **Noisy-neighbor “hot event” behavior**
  - How it fails: one large/busy event drives DB contention (capacity/promotion) and/or real-time fan-out.
  - Impact: RSVP latency spikes; retries increase; unrelated events slow due to shared DB/connection pool pressure.

- **Retry storms + duplicate side effects**
  - How it fails: transient errors cause clients/workers to retry; email sends and update publishes are at-least-once.
  - Impact: duplicate invitation/cancellation emails; excessive load during incidents.

- **Silent correctness drift (hard-to-detect production failures)**
  - How it fails: rare concurrency interleavings (capacity/promotion) or out-of-order real-time updates produce wrong allocations or a stale/regressed dashboard state that still “looks plausible”.
  - Impact: host makes decisions using incorrect counts/list; invitees see confusing outcomes; issues persist without an explicit reconciliation signal.

### Assumption failures (design relies on these being true)

- **Host authorization mechanism is not finalized (primary [SEC] risk)**
  - How it fails: host credential issuance/validation is weak, leaks, or is guessable.
  - Impact: attacker can view rosters, invite people, close responses, or cancel events.

- **Capability tokens leak via the invite-link distribution model**
  - How it fails: invitation emails are forwarded; URLs captured by logs/analytics/referrers.
  - Impact: unauthorized RSVP changes within an invite’s scope; if host tokens leak, full event compromise.

- **“Maybe at capacity” behavior remains undefined**
  - How it fails: implementation picks an ad-hoc rule not aligned with stakeholder intent.
  - Impact: user-visible “unfair” or inconsistent capacity behavior; promotions/counts don’t match expectations [COR].

## Alternative design directions (first pass) + comparison

Baseline for comparison: the current design in this document uses a **Spring Boot Backend API** as the single rule enforcer, **PostgreSQL** as the source of truth, **capability links** (invite link; assumed host management link/token), an **email provider** for invites/cancellations, and a **real-time update transport** for seconds-level dashboard updates.

The options below are not requirements; they are realistic directions that senior engineers commonly consider for RSVP/capacity systems with real-time dashboards. Each option is described in terms of what would concretely change in *this* system.

### Direction A — Per-event command serialization (queue/lock per event)

- **Idea**: all state-changing operations for an event (RSVP/change, close, cancel, invite creation that affects event state) are processed in a single, serialized stream per event.
- **Realistic variants in this stack**:
  - DB-backed serialization: per-event DB advisory lock or `SELECT ... FOR UPDATE` on an event row to ensure single-writer semantics.
  - External serialization: per-event queue partition/consumer (more moving parts, but explicit backlog handling).
- **Correctness**: strong, because capacity/promotion and close/cancel races are removed by construction (no concurrent writes within one event).
- **Complexity**: medium (introduces a per-event serialization mechanism and backlog handling).
- **Operational burden**: medium (need to run and monitor queue/worker, handle poison messages/retries, and ensure per-event ordering).
- **Likely bottleneck**: hot events create per-event backlog; worst-case RSVP latency becomes “time in queue” for that event.
- **Future changeability**: good for adding more per-event rules (they run in one place), but harder if you later need cross-event transactions.
- **When to choose**: when correctness under bursts is the top priority and you can accept per-event queuing latency for “hot events”.

### Direction B — Append-only audit log + deterministic projections (event-sourcing-lite)

- **Idea**: persist RSVP/close/cancel actions as an append-only history (audit) stream in Postgres, and derive:
  - “current RSVP per invite”,
  - allocation/waitlist state, and
  - dashboard view
  as projections from that history.
- **Correctness**: can be strong for determinism and auditability (you can answer “why was someone promoted?” by referencing an ordered history), but depends on projection correctness and projection freshness.
- **Complexity**: high (projection correctness, backfills, projection versioning, handling projection lag).
- **Operational burden**: high (monitoring projection lag, replay/backfill tooling, storage growth management).
- **Likely bottlenecks**:
  - projection lag causing stale dashboards if the read model falls behind,
  - hot events still concentrate write throughput on a single stream.
- **Future changeability**: strong for adding new read views/audits without changing write paths, but costly to implement and evolve.
- **When to choose**: when explainability/audit trails and “replay to debug correctness issues” are first-class needs (not currently confirmed), and the team can operate projections.

### Direction C — Transactional write model + explicit read projection + outbox for updates

- **Idea**: keep the baseline transactional write model, but add:
  - a **read-optimized dashboard projection** (counts + roster + allocation state) updated as part of the same commit as RSVP/close/cancel, and
  - a **transactional outbox** record that drives email sends and real-time dashboard publishes after commit.
- **Correctness**: improved safety around stale reads/out-of-order updates because “what to show” is a committed projection, and updates are derived from a committed outbox.
- **Complexity**: medium (more tables/state, but still within a conventional DB-backed design).
- **Operational burden**: medium (outbox worker, replay/retry behavior).
- **Likely bottlenecks**:
  - projection update cost on every RSVP write,
  - outbox backlog during provider/transport incidents.
- **Future changeability**: good path to add caching and fan-out without changing core invariants; keeps the source of truth in Postgres.
- **When to choose**: when you want to keep the baseline model but make delivery (email/push) and dashboard correctness more robust under retries, reconnects, and dependency outages.

### Comparison summary (relative to baseline)

- **Correctness under concurrency**:
  - Direction A: strongest (removes same-event write races).
  - Direction B: strong (deterministic ordering by history), but depends on projection correctness/freshness.
  - Direction C: strong (baseline invariants still need transactions; projection/outbox reduces stale/out-of-order issues).

- **Implementation complexity**:
  - Direction A: medium.
  - Direction B: high.
  - Direction C: medium.

- **Operational burden**:
  - Direction A: queue/worker monitoring and backlog management.
  - Direction B: projection monitoring/replay/backfill and projection versioning.
  - Direction C: outbox worker and projection consistency monitoring.

- **Scalability pressure points (concrete)**:
  - Direction A: per-event backlog for hot events.
  - Direction B: projection lag + storage growth + hot-event stream throughput.
  - Direction C: DB write amplification from projection updates + outbox backlog on dependency outages.

- **Future changeability**:
  - Direction A: easiest to add more per-event correctness rules.
  - Direction B: easiest to add new derived views/auditing, hardest to build.
  - Direction C: easiest incremental evolution from baseline toward higher fan-out and better delivery semantics.

## Nice-to-haves

- RSVP change history visibility for the host (per invitee: previous answers + when they changed).
- More complex "closed" behavior, including:
  - Allowing "Yes"/"Maybe" → "No" while closed.
  - Allowing "No"/"Maybe" → "Yes" while closed only with host approval (host notified via dashboard + email).
  - Disallowing "Yes"/"No" → "Maybe" while closed.
  - Explicit confirmation when canceling a "Yes" while closed.

## Future ideas (explicitly out of scope for now)

- Authentication and access expansion (accounts/OAuth/roles; RSVP lookup without the unique link).
- Public/marketplace features (event discovery, public pages, SEO, social feeds).
- Payments and on-site attendance tooling (ticketing, seat selection, check-in scanning).
- Advanced scheduling and event complexity (recurring events, multi-day agendas; complex post-invite edits with rebalancing).
- Advanced communications and email operations (reminders/digests/multi-channel notifications; templates/bounces/unsubscribes/analytics).
- Integrations and org-scale features (calendar sync; multi-tenant org/team ownership/delegation).
- Rich profiles and analytics beyond the dashboard (identity verification/profiles; reporting/analytics beyond counts + list).

## Implementation preferences / decisions (not goals)

- [COR] Event start time is stored as an exact instant derived from the host’s chosen local time.
- Cancellation notifications are sent via email.
- [SEC] Working assumption (not finalized): host management access is via a host-only secret link/token created when the event is created.

