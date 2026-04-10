# Quickstart: Bulk WhatsApp Sender

**Branch**: `001-bulk-whatsapp-sender`  
**Date**: 2026-03-27

---

## Prerequisites

- Java 17 (in PATH)
- Node.js 18+ and npm (in PATH)
- PostgreSQL running locally on port 5432 with database `hero` and schema `hero`
- Wasender API key (set as env var `WASENDERAPI_KEY` or hardcoded in `application.yml`)

---

## 1. Start the Backend

```powershell
cd hero-backend
.\mvnw.cmd spring-boot:run
```

Starts on **http://localhost:8080**

Verify:
```powershell
curl http://localhost:8080/api/bulk/upload
# Expected: 405 Method Not Allowed (GET not accepted — endpoint exists)
```

---

## 2. Start the Frontend

```powershell
cd hero-frontend
npm install        # first time only
npm run dev
```

Opens on **http://localhost:5173**

The Vite dev server proxies all `/api/**` requests to `http://localhost:8080`.

---

## 3. Run the Demo

1. Open **http://localhost:5173** in a browser
2. Upload a CSV or Excel file containing a column of phone numbers
3. Review the extracted numbers in the preview table
4. Type your message in the text area
5. Click **Send** — watch the loading indicator
6. Review the per-number results table

---

## 4. Sample CSV File

Create a file `contacts.csv`:

```
phone
+972526208082
+972501234567
+972509876543
```

---

## 5. Environment Variables

| Variable | Default in application.yml | Description |
|----------|---------------------------|-------------|
| `WASENDERAPI_KEY` | `c6dbb...` (dev key) | Wasender API authentication key |

To override:
```powershell
$env:WASENDERAPI_KEY = "your-key-here"
.\mvnw.cmd spring-boot:run
```
