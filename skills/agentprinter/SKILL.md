---
name: agentprinter
description: Send physical mail (a real letter or postcard) to a US address via the AgentPrinter API. Use when the user wants to mail, post, or send a physical document, letter, postcard, invoice, notice, or reminder to a street address — e.g. "mail this letter to Jane", "send a postcard to my mom", "post this invoice to a client". No account or API key is required; the user completes a one-time Stripe payment per piece.
---

# AgentPrinter — Send Physical Mail

AgentPrinter turns a document + a US mailing address into a real letter or postcard
dropped in the mail. It is account-free: you create a job with a single HTTP call,
hand the user a Stripe payment link, and the mail is printed and shipped (USPS First
Class) once they pay.

- **Base URL:** `https://api.agentprinter.app/v1`
- **No authentication** required to create a job.
- **US recipient addresses only.**

## Critical constraint — make the call directly

You **must** make the HTTP request as a direct tool/function call (e.g. Bash `curl`,
fetch in a code tool, or an HTTP request tool). Do **NOT** attempt the call from a
browser artifact, canvas, or sandboxed iframe — Content Security Policy will block it.

If you cannot make outbound HTTP requests at all (common on mobile/iOS apps or sandboxed
environments), do not guess or fake a result. Tell the user to repeat the request from a
browser-based session (claude.ai, chatgpt.com, or any web LLM that can call APIs), and
echo back the details they gave you so they can paste them in.

## Workflow

1. **Gather inputs.** You need a document, a recipient, and the sender's return address.
   Ask for the return address if you don't have it; never make one up. Confirm any missing required
   fields (see below) before sending. If the user didn't specify, default to a B&W
   `letter`.
2. **Create the job** — `POST /v1/print` (see Quick start). The API verifies the address
   and counts pages before returning.
3. **Give the user the `pay_url`.** This is a short, branded link that forwards to Stripe
   Checkout — prefer it over the raw `payment_url`. Payment is required before anything is
   printed. Also share the `status_url` so they can track delivery in a browser. The link
   expires in ~4 hours; the job moves to `expired` if unpaid.
4. **Track status** — either pass a `webhook_url` to receive push updates, or poll
   `GET /v1/jobs/:id`. Don't poll aggressively (read limit is 60/IP/hour).
5. **Cancel if needed** — `DELETE /v1/jobs/:id`, allowed only before fulfillment and
   within 30 minutes of creation.

## Quick start

Plain-text letter (simplest case — let AgentPrinter format it):

```bash
curl -sS -X POST https://api.agentprinter.app/v1/print \
  -H "Content-Type: application/json" \
  -d '{
    "document": {
      "text": "Dear Jane,\n\nThinking of you. See you in June!\n\nLove,\nSam"
    },
    "recipient": {
      "name": "Jane Smith",
      "line1": "123 Main St",
      "line2": "Apt 4B",
      "city": "Portland",
      "state": "OR",
      "zip": "97201"
    },
    "from": { "name": "Your Name", "line1": "456 Your St", "city": "Your City", "state": "CA", "zip": "94107" },
    "options": { "mail_type": "letter", "color": false }
  }'
```

Successful response (HTTP 200):

```json
{
  "job_id": "job_abc123def456",
  "status_url": "https://agentprinter.app/jobs/job_abc123def456",
  "pay_url": "https://agentprinter.app/pay/job_abc123def456",
  "payment_url": "https://checkout.stripe.com/c/pay/cs_live_...",
  "price_usd": 1.99,
  "expires_at": "2026-06-13T22:00:00Z",
  "webhook_secret": "whsec_..."   // only present if you sent a webhook_url
}
```

Then tell the user: *"Your letter is ready — pay here to send it: <pay_url>.
Track it anytime at <status_url>."*

## Request reference (`POST /v1/print`)

Four required top-level fields: `document`, `recipient`, `from`, `options`. One optional:
`webhook_url`.

### `document` — provide **exactly one** of:
| Field | Notes |
|---|---|
| `text` | Plain text, formatted into a letter using `options.font`. Max 50 KB, and at most 1,000 characters in a row with no place to break a line. Western European letters, digits and common punctuation only: write other alphabets, check marks and math signs in a PDF. |
| `pdf_url` | Publicly reachable HTTPS URL to a PDF. Max 5 MB. Up to 5 HTTPS redirects are followed; localhost/private-network URLs are rejected. For a letter, embed all fonts in the PDF. |
| `pdf_base64` | Base64-encoded PDF. Max 5 MB decoded. For a letter, embed all fonts in the PDF. |

Letters are limited to **10 pages**; each page after the first adds $0.10 (B&W) or $0.20 (color). Postcards print only the front (`text` or first PDF page).

### `recipient` (required) — US address only
`name`, `line1`, `city`, `state` (2-letter), `zip` (5-digit or ZIP+4) are required.
`company` and `line2` are optional. Lengths: `name` and `company` up to 40 characters,
`line1` and `line2` up to 64, `city` up to 200. For mail to a business, put the person or
department in `name` and the business in `company` (printed on the next line). A field
that is missing or too long returns `400 address_format_invalid`, with the field in
`details`. The address is verified server-side; an undeliverable one returns `422
address_undeliverable`.

### `from` (required) — return address
Same shape and limits as `recipient`. Ask the user for it if you don't have it. With
`options.branding: false` it must also pass the same deliverability check as `recipient`.
Only use an address the user may use; the Terms (section 3) require their authorization
for anyone else's.

### `options` (required)
| Field | Values | Default |
|---|---|---|
| `mail_type` | `letter`, `postcard` | `letter` |
| `color` | `true`, `false` | `false` (B&W) |
| `branding` | `true`, `false` | `true` — `false` leaves AgentPrinter's branding off the piece for $0.25 |
| `font` | `sans-serif` (Helvetica), `serif` (Times), `mono` (Courier) | `serif` for postcards, `sans-serif` for letters — text only, ignored for PDFs |
| `layout` | `classic`, `bold`, `minimal`, `note`, `typewriter`, `plain` | `classic` — **postcard text only**; ignored for letters/PDFs |
| `font_size` | `auto`, `fill`, or a whole number 9–48 | `auto` — **postcard text only**. `fill` sets the largest size that fits; the preview's `X-Font-Size` header says what was used |

Postcard layouts (plain-text fronts only):

| Layout | Look |
|---|---|
| `classic` | Framed note card, auto-sized type, vertically centered when short. Soft paper wash with `color: true`. |
| `bold` | Large statement with a side accent bar — best for short punchy copy. |
| `minimal` | Clean modern card with a top rule and generous whitespace. |
| `note` | Corner brackets + hairline rule — personal letter feel. |
| `typewriter` | Dashed frame, always Courier — telegram/postcard kitsch. |
| `plain` | Old unstyled left-aligned text (escape hatch). |

Prefer `color: true` for postcard text so the paper wash and ink accents print.

### `webhook_url` (optional)
HTTPS URL that receives `POST` status updates. Store the returned `webhook_secret` to
verify the `X-AgentPrinter-Signature` (HMAC-SHA256) header. Events: `job.paid`,
`job.submitted`, then delivery progress (`job.in_transit`, `job.in_local_area`,
`job.out_for_delivery`, `job.re_routed`), `job.mailed` (delivered), `job.returned`,
`job.failed`, `job.cancelled`, `job.expired`. Omit this to poll instead.

## Pricing (charged via Stripe at payment)
| Piece | B&W | Color |
|---|---|---|
| Letter, 1 page | $1.99 | $2.49 |
| Each extra letter page | $0.10 | $0.20 |
| Postcard | $1.49 | $1.49 |
| No AgentPrinter branding (`branding: false`) | +$0.25 | +$0.25 |

## Checking status (`GET /v1/jobs/:id`)

```bash
curl -sS https://api.agentprinter.app/v1/jobs/job_abc123def456
```

`status` is one of: `pending_payment`, `queued`, `in_production`, `mailed`, `failed`,
`expired`, `cancelled`. `mailed` means delivered. While it's in the mail, `status` stays
`in_production` and `delivery_stage` says where it is: `in_transit`, `in_local_area`,
`processed_for_delivery` (out for delivery), then `delivered`; `delivery_events` lists
each with its time. Once submitted you may also get `expected_delivery`, and
`tracking_url`, which despite its name links to the rendered PDF of the piece, not tracking.

## Passing on feedback (`POST /v1/jobs/:id/feedback`)

If the user wants to tell AgentPrinter something about a paid job (or the service), send it:

```bash
curl -sS -X POST https://api.agentprinter.app/v1/jobs/job_abc123def456/feedback \
  -H 'Content-Type: application/json' \
  -d '{"message": "Arrived in two days and looked great.", "rating": 5}'
```

`message` is required (up to 2000 characters); `rating` (1–5) is optional. Paid jobs only:
before payment it answers 403 `feedback_unavailable`. Operators read it but don't reply
here. Limits: 5 per IP per hour and 5 per job per day, separate from the write limit. Only
send what the user asks you to.

## Errors to handle
- **400 `address_format_invalid`** — a field is missing, malformed, or too long; `details`
  names each one. Fix it (a long business name goes in `company`) or ask the user.
- **422 `address_undeliverable`** — verification says the address can't be delivered; ask
  the user to check the street, city, state and ZIP.
- **400 `unsupported_characters`** — the text has characters plain text can't print, such
  as a letter of another alphabet (`details.characters` lists them). Rewrite without
  them, or send the document as a PDF. Arrows and a few other characters are replaced
  with a plain form and emoji are left out without an error: the response then has
  `text_replacements`, so tell the user what changed and show the preview.
- **400 `word_too_long`** — the text has over 1,000 characters in a row with no place to
  break a line (one very long word, or base64 with no line breaks). Add line breaks
  and resend; `details.actual_chars` says how long the run is.
- **422 `document_too_long`** — letter exceeds 10 pages (`details.max_pages`); ask the
  user to shorten it or split it. With `details.type: "postcard"`, the text doesn't fit
  the card: retry with `options.font_size: "fill"`, shorten it, or send it as a letter.
  Postcard text is never cut off to fit.
- **422 `unembedded_fonts`** — a letter's PDF uses fonts that are not embedded in the file
  (`details.fonts` names them), which the printer refuses. Make the PDF again with all
  fonts embedded (printing it to a new PDF does this) and resend, or ask the user to.
- **429** — rate limited. Back off for the seconds in `retry_after` in the body (the
  `Retry-After` header has the same number); `error` is `rate_limited`. A request refused with 400 or 422 counts toward the limit too.
- **Timeouts and retries** — send an `Idempotency-Key` header set to a fresh random UUID
  on `POST /v1/print` (keys are shared across all clients). Retrying with the same key and
  body returns the same job, not a second one. 409 `idempotency_key_in_progress` means the
  first try is still running; wait `Retry-After` seconds and retry. Up to 60 such retries
  an hour don't count toward the rate limit.

## More detail
See `reference/examples.md` for PDF-by-URL, postcard, base64, webhook, and cancel
examples. Authoritative contract: `https://agentprinter.app/v1/openapi.json`.
