# AgentPrinter — Additional Examples

Authoritative contract: https://agentprinter.app/v1/openapi.json
Base URL: `https://api.agentprinter.app/v1`

---

## 1. Letter from a PDF URL, with custom return address

```bash
curl -sS -X POST https://api.agentprinter.app/v1/print \
  -H "Content-Type: application/json" \
  -d '{
    "document": { "pdf_url": "https://pdfobject.com/pdf/sample.pdf" },
    "recipient": {
      "name": "Robert Thompson",
      "line1": "456 Oak Ave",
      "line2": "Suite 200",
      "city": "San Francisco",
      "state": "CA",
      "zip": "94102"
    },
    "from": {
      "name": "Dana Ortiz",
      "company": "Riverside Legal",
      "line1": "789 Elm St",
      "city": "Austin",
      "state": "TX",
      "zip": "78701"
    },
    "options": { "mail_type": "letter", "color": true }
  }'
```

Note: `company` prints on its own line below `name`. Each can be up to 40 characters.

## 2. Postcard from plain text (color)

Postcards are 4x6 and print only the front. Plain text defaults to the `classic`
framed layout (serif, auto-sized). Pass `options.layout` to pick another look.

```bash
curl -sS -X POST https://api.agentprinter.app/v1/print \
  -H "Content-Type: application/json" \
  -d '{
    "document": { "text": "Greetings from Portland! Wish you were here." },
    "recipient": {
      "name": "Jane Smith",
      "line1": "123 Main St",
      "city": "Portland",
      "state": "OR",
      "zip": "97201"
    },
    "from": { "name": "Your Name", "line1": "456 Your St", "city": "Your City", "state": "CA", "zip": "94107" },
    "options": { "mail_type": "postcard", "color": true, "layout": "classic" }
  }'
```

Short punchy line with the bold layout:

```bash
curl -sS -X POST https://api.agentprinter.app/v1/print \
  -H "Content-Type: application/json" \
  -d '{
    "document": { "text": "Miss you already." },
    "recipient": {
      "name": "Jane Smith",
      "line1": "123 Main St",
      "city": "Portland",
      "state": "OR",
      "zip": "97201"
    },
    "from": { "name": "Your Name", "line1": "456 Your St", "city": "Your City", "state": "CA", "zip": "94107" },
    "options": { "mail_type": "postcard", "color": true, "layout": "bold" }
  }'
```

## 3. Letter from a local PDF (base64)

Encode a local file and inject it into the request body:

```bash
B64=$(base64 < ./letter.pdf | tr -d '\n')
curl -sS -X POST https://api.agentprinter.app/v1/print \
  -H "Content-Type: application/json" \
  -d "$(cat <<JSON
{
  "document": { "pdf_base64": "$B64" },
  "recipient": {
    "name": "Jane Smith",
    "line1": "123 Main St",
    "city": "Portland",
    "state": "OR",
    "zip": "97201"
  },
  "from": { "name": "Your Name", "line1": "456 Your St", "city": "Your City", "state": "CA", "zip": "94107" },
  "options": { "mail_type": "letter", "color": false }
}
JSON
)"
```

## 4. With a status webhook

```bash
curl -sS -X POST https://api.agentprinter.app/v1/print \
  -H "Content-Type: application/json" \
  -d '{
    "document": { "text": "Your appointment is confirmed for March 22 at 10:00 AM." },
    "recipient": {
      "name": "Robert Thompson",
      "line1": "456 Oak Ave",
      "city": "San Francisco",
      "state": "CA",
      "zip": "94102"
    },
    "from": { "name": "Your Name", "line1": "456 Your St", "city": "Your City", "state": "CA", "zip": "94107" },
    "options": { "mail_type": "letter" },
    "webhook_url": "https://your-app.com/webhooks/mail"
  }'
```

The response includes a `webhook_secret`. Store it; use it to verify the
`X-AgentPrinter-Signature` HMAC-SHA256 header on each incoming webhook. Webhook body
shape:

```json
{
  "event": "job.mailed",
  "timestamp": "2026-03-19T15:22:47Z",
  "data": {
    "job_id": "job_abc123def456",
    "status": "mailed",
    "delivery_stage": "delivered",
    "lob_event": "letter.delivered",
    "expected_delivery": "2026-03-19",
    "tracking_number": null
  }
}
```

## 5. Poll status

```bash
curl -sS https://api.agentprinter.app/v1/jobs/job_abc123def456
```

## 6. Cancel a job

Allowed only before fulfillment and within 30 minutes of creation. Returns `409` if the
job is already in production or the window has passed.

```bash
curl -sS -X DELETE https://api.agentprinter.app/v1/jobs/job_abc123def456
```

## Error response shapes

```json
{ "error": "address_format_invalid", "message": "...", "stage": "format", "field": "recipient", "details": [{ "field": "name", "message": "Name can be up to 40 characters; this one is 57" }] }
{ "error": "address_undeliverable", "message": "...", "stage": "lob_verification", "deliverability": "undeliverable" }
{ "error": "document_too_long", "message": "...", "details": { "max_pages": 10, "actual_pages": 12, "type": "letter" } }
{ "error": "document_too_long", "message": "...", "details": { "type": "postcard", "layout": "classic", "font_size": "auto", "min_font_size": 11 } }
{ "error": "unembedded_fonts", "message": "...", "details": { "fonts": ["Calibri-Light", "SegoeUI"], "total": 2 } }
{ "error": "rate_limited", "message": "Rate limit exceeded. Try again later.", "retry_after": 1742 }
{ "error": "job_not_found", "message": "Job not found" }
```

Every error has a code in `error` and a sentence in `message`: branch on the code. The 429's wait in seconds is `retry_after` (also in the `Retry-After` header).
