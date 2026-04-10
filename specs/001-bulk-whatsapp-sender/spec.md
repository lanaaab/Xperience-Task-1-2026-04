# Feature Specification: Bulk WhatsApp Sender

**Feature Branch**: `001-bulk-whatsapp-sender`  
**Created**: 2026-03-27  
**Status**: Draft  
**Source**: Derived from `DESIGN.md` (goals G1–G5, non-goals NG1–NG10, constitution v1.0.0)

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Upload File & Preview Recipients (Priority: P1)

The demo operator opens the app, uploads a CSV or Excel file that contains phone numbers,
and sees the extracted list of numbers displayed in a table before anything is sent.
This is the entry point to the entire flow and delivers value on its own by proving
the file parsing works.

**Why this priority**: Without a working upload and preview, no other part of the
feature can function. It is the foundation of the end-to-end demo.

**Independent Test**: Can be fully tested by uploading a sample CSV/Excel file and
verifying the numbers appear in the preview table — no message needs to be sent.

**Acceptance Scenarios**:

1. **Given** a CSV file with a column of phone numbers, **When** the user uploads it,
   **Then** each phone number from that column is displayed in a preview table in the UI.

2. **Given** an Excel (.xlsx) file with a column of phone numbers, **When** the user
   uploads it, **Then** the numbers are extracted and shown in the preview table.

3. **Given** an empty file or a file with no recognisable phone number column,
   **When** the user uploads it, **Then** the UI displays a clear error message and
   no preview table is shown.

4. **Given** a file of an unsupported type (e.g., PDF, image), **When** the user tries
   to upload it, **Then** the upload is rejected immediately with a clear error message.

---

### User Story 2 — Compose Message & Send to All Numbers (Priority: P2)

After previewing the list, the demo operator types a message text and clicks Send.
The app dispatches the same message to every phone number in the preview list via
WhatsApp.

**Why this priority**: This is the core demo capability — the visible proof that
bulk WhatsApp messaging works. It depends on US1 being complete.

**Independent Test**: With a known preview list already populated (from US1), type
a message and click Send. Verify that WhatsApp messages are received on the target
phones.

**Acceptance Scenarios**:

1. **Given** a populated preview list and a non-empty message text, **When** the user
   clicks Send, **Then** the backend sends the message to each phone number in order
   using the Wasender API.

2. **Given** a populated preview list and an **empty** message text, **When** the user
   clicks Send, **Then** the send is blocked and a validation message is shown.

3. **Given** a populated preview list, **When** the user clicks Send, **Then** the UI
   shows a loading / in-progress indicator while sending is under way.

4. **Given** the Wasender API returns an error for one number, **When** the backend
   processes that row, **Then** the error is recorded and sending continues to the
   remaining numbers (skip-and-continue strategy).

---

### User Story 3 — View Send Results (Priority: P3)

After all messages have been dispatched, the demo operator sees a per-number results
table showing which numbers succeeded and which failed.

**Why this priority**: This completes goal G4 and closes the demo narrative. It depends
on US2 being complete.

**Independent Test**: After triggering a send (US2), verify that the results table
appears and accurately reflects the success/failure status of each number.

**Acceptance Scenarios**:

1. **Given** all messages were sent successfully, **When** the send completes, **Then**
   every row in the results table shows a success status.

2. **Given** one or more numbers failed (API error), **When** the send completes,
   **Then** the results table shows a failure status for those rows, alongside the
   successful ones.

3. **Given** the send has completed, **When** the user sees the results table, **Then**
   the total count of sent and failed messages is shown as a summary.

---

### Edge Cases

- File contains duplicate phone numbers — all duplicates are sent to (no deduplication
  in v1; the user is responsible for the file content)
- File has extra columns beyond phone numbers — they are parsed but silently ignored;
  only the phone number column is used
- Phone number column contains blank cells — blank rows are skipped and excluded from
  the preview count
- Network failure mid-send — remaining numbers are not attempted; results reflect what
  succeeded before the failure
- File too large (>500 rows) — [NEEDS CLARIFICATION: accepted as-is or rejected at upload with a size warning?]

---

## Requirements *(mandatory)*

### Functional Requirements

**File Upload & Parsing**

- **FR-001**: The system MUST accept file uploads in CSV (`.csv`) and Excel (`.xlsx`, `.xls`) formats.
- **FR-002**: The backend MUST parse the uploaded file and extract a list of phone numbers from it. All parsing happens server-side (constitution Principle V).
- **FR-003**: The backend MUST identify the phone number column by looking for a header named `phone`, `phone_number`, `mobile`, or `number` (case-insensitive). If no matching header is found, the first column is used.
- **FR-004**: Rows with blank or unparseable values in the phone column MUST be silently skipped; the remaining rows MUST still be returned.
- **FR-005**: If the file is empty, corrupt, or of an unsupported type, the backend MUST return a descriptive error; the frontend MUST display it clearly (constitution Principle III).

**Preview**

- **FR-006**: After a successful upload, the frontend MUST display the extracted phone numbers in a table before any message is sent (goal G2).
- **FR-007**: The preview table MUST show the total count of extracted numbers.

**Message Composition**

- **FR-008**: The user MUST be able to type a message text in the UI before triggering the send.
- **FR-009**: The message text field MUST enforce a minimum of 1 character and a maximum of 1,000 characters. An empty message MUST block the send action.

**Bulk Send**

- **FR-010**: The frontend MUST send the message text and the list of phone numbers to the backend in a single request. The backend is solely responsible for calling the Wasender API (constitution Principle V).
- **FR-011**: The backend MUST call `WasenderService.sendTextMessage()` once per phone number, in sequence.
- **FR-012**: If the Wasender API call fails for a given number, the backend MUST record the failure and continue to the next number (skip-and-continue; non-goal NG5).
- **FR-013**: The frontend MUST show a loading indicator while the send is in progress.

**Results**

- **FR-014**: After all sends complete, the backend MUST return a per-number result list, with each entry indicating success or failure (and the error reason for failures).
- **FR-015**: The frontend MUST display the results in a table, with a clear success/failure status per number (goal G4).
- **FR-016**: The frontend MUST display a summary line showing total sent, total failed, and total attempted.

### Key Entities

- **UploadedFile** — The file submitted by the user. Has a format (CSV/Excel) and contains one or more rows. Not persisted (constitution Principle IV).
- **RecipientRow** — A single extracted row from the file. Has a phone number. May have other fields that are ignored.
- **SendRequest** — A transient payload: a list of phone numbers + a message text. Never stored.
- **SendResult** — A per-number outcome: phone number, status (success | failure), and optional error message. Returned in the HTTP response only, never persisted.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The demo operator can complete the entire flow — upload, preview, compose, send, view results — without leaving the browser or writing any code (goal G5).
- **SC-002**: Every phone number successfully parsed from the file appears in the preview table before sending begins (goal G2).
- **SC-003**: Every number in the preview list has a corresponding entry in the results table after sending completes, with an unambiguous success or failure status (goal G4).
- **SC-004**: An invalid file (wrong type, empty, no phone column) is rejected with a readable error message within 3 seconds of upload.
- **SC-005**: A message is delivered to all reachable numbers in the file without any manual intervention beyond clicking Send.

---

## Assumptions

- The uploaded file contains phone numbers in E.164 format (e.g., `+972526208082`). The app does not normalize or validate number format — the Wasender API handles that.
- The Wasender API key is configured in the backend environment at all times during the demo. The frontend never receives or handles the API key.
- File size is bounded to demo scale (up to a few hundred rows). No async job queue or rate-limit throttling is required for this scale.
- The message is the same text for every recipient. Personalization per row is explicitly excluded (non-goal NG2).
- Send results are displayed in the active browser session only and are not stored anywhere (constitution Principle IV, non-goal NG3).
- The backend and frontend run locally for the demo (backend on port 8080, frontend via Vite dev server). No deployment or hosting configuration is in scope.
