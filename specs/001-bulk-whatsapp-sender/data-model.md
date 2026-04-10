# Data Model: Bulk WhatsApp Sender

**Branch**: `001-bulk-whatsapp-sender`  
**Date**: 2026-03-27  
**Note**: All entities below are transient (in-memory only). Nothing is persisted to the database (constitution Principle IV).

---

## Backend DTOs

### `UploadResponse`
Returned by `POST /api/bulk/upload` after successful file parsing.

| Field | Type | Description |
|-------|------|-------------|
| `recipients` | `List<String>` | Extracted phone numbers (non-blank rows only) |
| `count` | `int` | Total number of valid phone numbers extracted |

---

### `BulkSendRequest`
Sent by the frontend to `POST /api/bulk/send`.

| Field | Type | Validation | Description |
|-------|------|------------|-------------|
| `message` | `String` | Required, 1–1000 chars | Message text to send to all recipients |
| `recipients` | `List<String>` | Required, non-empty | Phone numbers from the previous upload step |

---

### `RecipientResult`
One entry per phone number in the send response.

| Field | Type | Description |
|-------|------|-------------|
| `phone` | `String` | The phone number this result belongs to |
| `status` | `String` (enum: `SUCCESS`, `FAILURE`) | Outcome of the Wasender API call |
| `error` | `String` (nullable) | Error message if status is `FAILURE`; null otherwise |

---

### `BulkSendResponse`
Returned by `POST /api/bulk/send` after all sends complete.

| Field | Type | Description |
|-------|------|-------------|
| `results` | `List<RecipientResult>` | Per-number outcome, in the same order as the request |
| `sentCount` | `int` | Number of numbers that succeeded |
| `failedCount` | `int` | Number of numbers that failed |
| `totalCount` | `int` | Total numbers attempted (`sentCount + failedCount`) |

---

### `ErrorResponse`
Returned on HTTP 4xx errors (bad file, validation failure).

| Field | Type | Description |
|-------|------|-------------|
| `error` | `String` | Human-readable error description |

---

## Frontend State Shape

Held as `useState` in `App.tsx`. No localStorage, no external store.

```
AppState
├── step: "UPLOAD" | "PREVIEW" | "SENDING" | "RESULTS"
├── recipients: string[]           ← populated after successful upload
├── message: string                ← controlled input value
├── results: RecipientResult[]     ← populated after send completes
├── uploadError: string | null     ← shown in upload step on failure
└── sendError: string | null       ← shown in results step on total failure
```

---

## File Parsing Rules (no persistence involved)

| Input | Handling |
|-------|----------|
| `.csv` | Parse with Apache Commons CSV; detect header using R4 strategy |
| `.xlsx` | Parse with Apache POI (XSSF); use first sheet |
| `.xls` | Parse with Apache POI (HSSF); use first sheet |
| Blank cell in phone column | Skip row silently |
| Unsupported file type | Return `ErrorResponse` with HTTP 400 |
| No phone-like header found | Fall back to first column |
| Empty file | Return `ErrorResponse` with HTTP 400 |
