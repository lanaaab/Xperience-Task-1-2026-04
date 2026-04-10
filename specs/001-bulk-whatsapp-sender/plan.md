# Implementation Plan: Bulk WhatsApp Sender

**Branch**: `001-bulk-whatsapp-sender` | **Date**: 2026-03-27 | **Spec**: [spec.md](spec.md)  
**Input**: Feature specification from `specs/001-bulk-whatsapp-sender/spec.md`

---

## Summary

Allow a single demo operator to upload a CSV or Excel file containing phone numbers,
preview the extracted list, type a message, and send it to all numbers via the
Wasender WhatsApp API. The backend (Spring Boot) handles all file parsing and API
calls. The frontend (React + Vite) drives the 3-step UI flow. Nothing is persisted.

**Technical approach**: Multipart file upload → Apache POI / Commons CSV parsing →
in-memory list returned to frontend → frontend POSTs list + message → backend loops
WasenderService → per-number results returned.

---

## Technical Context

**Language/Version**: Java 17 (backend) · TypeScript ~5.6 (frontend)  
**Primary Dependencies**:
- Backend: Spring Boot 4.0.x, Apache POI 5.3.0 (Excel), Apache Commons CSV 1.12.0
- Frontend: React 18, Vite 5, Tailwind CSS (utility styling)  

**Storage**: PostgreSQL provisioned but **NOT used** by this feature (constitution Principle IV)  
**Testing**: Spring Boot Test (existing) — manual curl tests per workflow rule 3  
**Target Platform**: Local demo — backend HTTP server (port 8080) + Vite dev server (port 5173)  
**Project Type**: web-service (full-stack web application)  
**Performance Goals**: No req/s target — demo scale (~100–500 rows, single operator)  
**Constraints**: No auth · No persistence · Synchronous sends · Files ≤ ~500 rows  
**Scale/Scope**: 1 operator · 1 session · 2 REST endpoints · 1 page (3 steps)

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design. ✅ All pass.*

| Principle | Check | Status |
|-----------|-------|--------|
| I. Demo-First Simplicity | No job queue, no auth, no persistence, synchronous sends, Tailwind not MUI | ✅ PASS |
| II. Full-Stack Boundary | REST only between tiers; Vite proxy `/api/**` → `localhost:8080`; no shared code | ✅ PASS |
| III. Fail Visibly | All 3 user stories define error states; `ErrorResponse` DTO for 400s; frontend handles upload/send error states | ✅ PASS |
| IV. Stateless Session | No DB writes; all state lives in `useState` in `App.tsx`; results returned in HTTP response only | ✅ PASS |
| V. Single Outbound Channel | `BulkSendService` calls `WasenderService` directly; no adapter layer | ✅ PASS |

**No complexity violations.** Complexity Tracking table is not required.

---

## Project Structure

### Documentation (this feature)

```text
specs/001-bulk-whatsapp-sender/
├── plan.md          ← this file
├── research.md      ← Phase 0 complete
├── data-model.md    ← Phase 1 complete
├── quickstart.md    ← Phase 1 complete
├── contracts/
│   └── api.md       ← Phase 1 complete
├── checklists/
│   └── requirements.md
└── tasks.md         ← created by /speckit.tasks (next step)
```

### Source Code

```text
hero-backend/
└── src/main/java/com/xperience/hero/
    ├── whatsapp/                        ← existing (unchanged)
    │   ├── WasenderService.java
    │   └── WasenderProperties.java
    └── bulk/                            ← NEW
        ├── BulkSendController.java      ← REST endpoints: POST /api/bulk/upload, POST /api/bulk/send
        ├── BulkSendService.java         ← orchestrates parse + send loop
        ├── FileParserService.java       ← Apache POI + Commons CSV phone extraction
        └── dto/
            ├── UploadResponse.java
            ├── BulkSendRequest.java
            ├── RecipientResult.java
            ├── BulkSendResponse.java
            └── ErrorResponse.java

hero-frontend/
└── src/
    ├── App.tsx                          ← REPLACE scaffold; owns state + step navigation
    ├── App.css                          ← minimal overrides
    ├── services/
    │   └── bulkSendApi.ts               ← NEW: fetch wrappers for /api/bulk/upload and /api/bulk/send
    └── components/
        ├── FileUpload.tsx               ← NEW: drag/drop or click file input + error display
        ├── RecipientTable.tsx           ← NEW: preview table of extracted phone numbers
        ├── MessageComposer.tsx          ← NEW: textarea + char count + Send button
        └── ResultsTable.tsx             ← NEW: per-number success/failure + summary row
```

**Structure Decision**: Web application layout (separate `hero-backend` / `hero-frontend`
directories already exist in the repo). No new top-level directories introduced.
