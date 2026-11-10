# Landing page – FE/BE API contract

Two public, unauthenticated endpoints are used by `index_new.html`:

| Purpose | Method & path |
|---|---|
| Statistics block ("Ein Einblick aus dem Schulalltag") | `GET /api/v1/public/library-stats` |
| Contact modal ("Blattflip an Ihrer Schule") | `POST /api/v1/public/contact-requests` |

The paths are relative to `API_BASE` (constant at the top of the page script, empty = same origin).
If the API runs on another origin, it must send CORS headers allowing `https://blattflip.de`
(`Access-Control-Allow-Origin`, and for the POST also `Access-Control-Allow-Headers: Content-Type`).

All payloads are JSON, UTF-8. Dates are ISO 8601, timestamps in UTC.

---

## 1. `GET /api/v1/public/library-stats`

Returns aggregated, anonymous statistics of the showcase school library. No personal data.

### Request

```
GET /api/v1/public/library-stats
Accept: application/json
```

No parameters.

### Response `200 OK`

Example: [`library-stats.example.json`](library-stats.example.json)

```json
{
  "schemaVersion": 1,
  "generatedAt": "2026-11-01T03:00:00Z",
  "isSample": false,
  "library": { "city": "Düsseldorf" },
  "metrics": {
    "booksInStock": { "value": 691, "approximate": false },
    "students":     { "value": 450, "approximate": true },
    "volunteers":   { "value": 30,  "approximate": true }
  },
  "loansPerMonth": [
    { "month": "2025-11", "count": 280 },
    { "month": "2025-12", "count": 190 },
    "… 12 entries in total …",
    { "month": "2026-10", "count": 320 }
  ]
}
```

### Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `schemaVersion` | integer | yes | Always `1` for this contract. The FE ignores responses with another version. |
| `generatedAt` | string (date-time) | yes | When the aggregate was computed. |
| `isSample` | boolean | yes | `true` if numbers are illustrative. The FE then appends "· Beispieldaten" to the chart period. |
| `library.city` | string | yes | Shown in the headline: "Eine Schulbibliothek in *{city}*." |
| `metrics.booksInStock` | Metric | yes | "Bücher im Bestand" |
| `metrics.students` | Metric | yes | "Kinder an der Schule" |
| `metrics.volunteers` | Metric | yes | "ehrenamtliche Helfer" |
| `loansPerMonth` | LoanMonth[] | yes | Chart data, **oldest first**, consecutive months. |

**Metric**

| Field | Type | Required | Description |
|---|---|---|---|
| `value` | integer ≥ 0 | yes | Raw number; FE formats it (`de-DE`, e.g. `1.234`). |
| `approximate` | boolean | yes | `true` → FE renders `~450`. |

**LoanMonth**

| Field | Type | Required | Description |
|---|---|---|---|
| `month` | string `YYYY-MM` | yes | Calendar month. |
| `count` | integer ≥ 0 | yes | Number of loans recorded in that month. |

### Rules for the backend

- Include only **completed** months (the current month is excluded). The page says "Abgeschlossene Monate".
- Return the **last 12** completed months. Months without loans (e.g. holidays) must be present with `count: 0`
  rather than omitted, so the chart has no gaps.
- Labels, month names, axis scale and number formatting are done by the FE; the BE sends raw values only.
- Caching: the data changes at most daily. Send `Cache-Control: public, max-age=3600` (or longer).

### Errors

Any non-`2xx` response, a timeout (5 s), invalid JSON or a schema mismatch → the FE silently keeps its
built-in fallback data (`FALLBACK_STATS` in the page script, marked `isSample: true`).
No error body is required.

---

## 2. `POST /api/v1/public/contact-requests`

Submitted by the contact modal.

### Request

```
POST /api/v1/public/contact-requests
Content-Type: application/json
Accept: application/json
```

```json
{
  "name": "Anna Beispiel",
  "email": "anna@grundschule-beispiel.de",
  "school": "Grundschule am Beispielweg, Köln",
  "message": "Wir haben ca. 300 Bücher …",
  "website": "",
  "source": "landing",
  "utm": { "utm_source": "newsletter" }
}
```

| Field | Type | Required | Constraints | Form label |
|---|---|---|---|---|
| `name` | string \| null | no | ≤ 200 chars | Ihr Name |
| `email` | string | **yes** | valid e-mail, ≤ 254 chars | E-Mail-Adresse |
| `school` | string | **yes** | 1–200 chars | Schule |
| `message` | string \| null | no | ≤ 2000 chars | Ihre Nachricht |
| `website` | string | yes | Honeypot (hidden field). If non-empty → treat as spam: respond `201` but do not store/notify. | – |
| `source` | string | yes | Always `"landing"` for now. | – |
| `utm` | object<string,string> | yes | `utm_*` query params of the landing page URL, may be `{}`. | – |

Strings are trimmed by the FE; empty optional fields are sent as `null`.

### Responses

| Status | When | Body |
|---|---|---|
| `201 Created` | Request accepted (also for honeypot hits). | `{ "id": "…" }` (optional, FE ignores it) |
| `400 Bad Request` | Validation failed. | `{ "error": "validation_failed", "fields": { "email": "invalid" } }` |
| `429 Too Many Requests` | Rate limit (suggested: 5 requests / IP / hour). | `{ "error": "rate_limited" }` |
| `5xx` | Server error. | any |

FE behaviour: any `2xx` shows the thank-you message
("Vielen Dank für Ihr Interesse! …"). Any other status or network error shows a generic error with the
fallback address `hello@blattflip.de`; the form content is kept so the user can retry.

### Backend responsibilities

- Validate server-side with the same constraints (do not trust the FE).
- Notify the team (e.g. e-mail to `hello@blattflip.de`) with all fields.
- Store contact requests only as long as needed to handle the request; this must be reflected in `datenschutz.html`.
