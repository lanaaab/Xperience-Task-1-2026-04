# API Contract: Bulk WhatsApp Sender

**Branch**: `001-bulk-whatsapp-sender`  
**Date**: 2026-03-27  
**Base URL**: `http://localhost:8080/api/bulk`  
**Frontend proxy**: Vite proxies `/api/**` → `http://localhost:8080` (constitution Principle II)

---

## POST /api/bulk/upload

Accepts a file upload, parses it, and returns the list of extracted phone numbers.
No messages are sent by this endpoint.

### Request

```
POST /api/bulk/upload
Content-Type: multipart/form-data

file: <binary file content>   (field name: "file")
```

Accepted file types: `.csv`, `.xlsx`, `.xls`  
Max recommended size: ~500 rows (demo scale)

### Response — 200 OK

```json
{
  "recipients": [
    "+972526208082",
    "+972501234567"
  ],
  "count": 2
}
```

### Response — 400 Bad Request

Returned when: unsupported file type, empty file, corrupt file, or no data rows found.

```json
{
  "error": "No phone numbers found in the uploaded file."
}
```

### Manual Test

```
curl -X POST http://localhost:8080/api/bulk/upload \
  -F "file=@contacts.csv"
```

Expected: JSON with `recipients` array and `count` matching the number of non-blank rows.

---

## POST /api/bulk/send

Sends the same message text to every phone number in the request.
Returns a per-number success/failure result for all attempts.

### Request

```
POST /api/bulk/send
Content-Type: application/json

{
  "message": "Hello, this is a demo message!",
  "recipients": [
    "+972526208082",
    "+972501234567"
  ]
}
```

| Field | Required | Constraint |
|-------|----------|------------|
| `message` | Yes | 1–1000 characters |
| `recipients` | Yes | Non-empty list |

### Response — 200 OK

Always 200, even if some sends failed (failure is per-row, not total).

```json
{
  "results": [
    { "phone": "+972526208082", "status": "SUCCESS", "error": null },
    { "phone": "+972501234567", "status": "FAILURE", "error": "Recipient not reachable" }
  ],
  "sentCount": 1,
  "failedCount": 1,
  "totalCount": 2
}
```

### Response — 400 Bad Request

Returned when: message is blank, recipients list is empty, or request body is malformed.

```json
{
  "error": "Message text must not be empty."
}
```

### Manual Test

```
curl -X POST http://localhost:8080/api/bulk/send \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Hi from the demo!",
    "recipients": ["+972526208082"]
  }'
```

Expected: JSON with `results` array, `sentCount`, `failedCount`, `totalCount`.

---

## Frontend → Backend Call Summary

| Step | Frontend action | Endpoint called |
|------|----------------|----------------|
| User uploads file | `FormData` POST | `POST /api/bulk/upload` |
| User clicks Send | JSON POST | `POST /api/bulk/send` |
