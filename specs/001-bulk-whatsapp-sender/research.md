# Research: Bulk WhatsApp Sender

**Branch**: `001-bulk-whatsapp-sender`  
**Date**: 2026-03-27  
**Status**: Complete — all unknowns resolved

---

## R1 — Backend: Excel/CSV Parsing Library

**Unknown**: Which library parses `.xlsx`, `.xls`, and `.csv` files on the Spring Boot backend?

**Decision**: **Apache POI** (poi-ooxml) for Excel + **Apache Commons CSV** for CSV.

**Rationale**:
- Apache POI is the Java standard for Excel files. No realistic alternative exists for `.xlsx`/`.xls` parsing.
- Apache Commons CSV is lightweight (single JAR, no transitive deps beyond commons-io), handles quoted fields, different delimiters, and BOM characters correctly. Simpler than OpenCSV.
- Both are well-maintained, have no CVEs in current versions, and are already used in the Spring Boot ecosystem.

**Alternatives considered**:
- OpenCSV — more popular but heavier API; not needed for this use case.
- EasyExcel (Alibaba) — designed for large files; overkill for demo scale.

**Maven artifacts to add** (hero-backend/pom.xml):
```xml
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.3.0</version>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-csv</artifactId>
    <version>1.12.0</version>
</dependency>
```

---

## R2 — Backend: File Upload Mechanism

**Unknown**: How does the frontend send the file to the backend?

**Decision**: **HTTP multipart/form-data** upload to a Spring MVC endpoint accepting `MultipartFile`.

**Rationale**:
- Standard browser file upload; no base64 encoding overhead.
- Spring MVC has native `@RequestParam MultipartFile file` support — zero configuration.
- Keeps the file in memory (no temp disk writes) for demo-scale files; perfectly safe under ~500 rows.

**Alternatives considered**:
- Base64-encoded JSON body — unnecessary encoding overhead; no benefit here.
- Chunked streaming upload — overkill for demo scale.

---

## R3 — Backend: CORS Configuration

**Unknown**: How does the React dev server (port 5173) call the Spring Boot backend (port 8080) without CORS errors?

**Decision**: Two-pronged approach:
1. Vite dev proxy — `vite.config.ts` proxies `/api/**` → `http://localhost:8080` during development (constitution Principle II requirement).
2. Spring `@CrossOrigin` on the controller — allows `http://localhost:5173` as an explicit origin, so the API also works when called directly (e.g., from curl or Postman during development).

**Alternatives considered**:
- Global Spring `WebMvcConfigurer` CORS config — heavier than needed for a single controller in a demo.

---

## R4 — Backend: Phone Number Column Detection

**Unknown**: Files may have varying column names. How does the parser find the phone column?

**Decision**: Header name matching with prioritised fallback.

Strategy (in order):
1. Scan headers case-insensitively for: `phone`, `phone_number`, `mobile`, `number`, `tel`
2. If no match, use column index 0 (first column)
3. Skip rows where the matched cell is blank or null

**Rationale**: Handles the most common real-world file formats without requiring the user to pre-format the file. Aligns with G1 (no scripting or external tools needed).

---

## R5 — Frontend: UI Approach

**Unknown**: No UI library is installed yet. What visual approach fits a demo?

**Decision**: **Tailwind CSS** (via CDN or npm) — utility-class styling only, no component library.

**Rationale**:
- Tailwind is a pure styling utility; it adds no JavaScript bundle weight and no component abstractions.
- A clean table + button layout is all this demo requires. No modal system, no date picker, no complex components that would justify a full library like MUI or Ant Design.
- Installs in one `npm install` and one line in `vite.config.ts` / `index.css`.
- Satisfies constitution Principle I (simplest option) while producing a professional-looking demo.

**Alternatives considered**:
- No CSS library (plain CSS) — achievable but slower to write and less consistent-looking for a pitch demo.
- MUI / Ant Design — powerful but 40–100 kB of components for a 3-screen app; violates Principle I.
- shadcn/ui — good choice but requires Radix UI and more setup; overkill for this scope.

**npm package to add** (hero-frontend):
```
npm install -D tailwindcss @tailwindcss/vite
```

---

## R6 — Frontend: State Management

**Unknown**: How is UI state managed across the three steps (upload → compose → results)?

**Decision**: **React `useState` in `App.tsx`** — no external state library.

**Rationale**:
- The app has one page with three sequential steps driven by a single `step` enum (`UPLOAD | PREVIEW | RESULTS`).
- All data (recipients list, message text, results) can be held as `useState` values at the `App` level and passed as props.
- Redux, Zustand, etc. are unjustified complexity for a linearly-sequenced 3-step flow.

---

## R7 — API Design: Upload vs Send as One or Two Endpoints

**Unknown**: Should upload+parse and send be a single endpoint or two separate calls?

**Decision**: **Two separate endpoints** — `/api/bulk/upload` and `/api/bulk/send`.

**Rationale**:
- Separating upload from send mirrors the UX flow exactly: the user reviews the parsed list *before* committing to send.
- A single endpoint would force sending immediately on upload, removing the preview step (violating G2).
- Two endpoints also means the backend parse step can fail independently without triggering any sends.

---

## Summary Table

| # | Unknown | Decision |
|---|---------|----------|
| R1 | Excel/CSV library | Apache POI + Apache Commons CSV |
| R2 | File upload mechanism | Multipart/form-data, Spring `MultipartFile` |
| R3 | CORS | Vite proxy + `@CrossOrigin` on controller |
| R4 | Phone column detection | Header name matching with first-column fallback |
| R5 | Frontend UI library | Tailwind CSS (utility only) |
| R6 | Frontend state | React `useState` in `App.tsx` |
| R7 | Endpoint count | Two: `/api/bulk/upload` + `/api/bulk/send` |
