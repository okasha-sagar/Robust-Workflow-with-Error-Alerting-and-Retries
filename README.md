## Video Walkthrough
Loom video link: https://www.loom.com/share/a1ec479a19a34a4bb5f5e8c21a6eac82

## Written Walkthrough summary

# Robust Order Workflow — Design Walkthrough & Resubmission Notes

## 1. What changed since the last submission

The mentor flagged two things:

1. **Fallback response gap** — unexpected mid-flow exceptions were only caught by the
   separate *Global Error Handler* workflow (triggered via n8n's Error Trigger). That
   workflow logs to Google Sheets and sends a Gmail alert, but it runs as a **new,
   unrelated execution**, so it has no access to the original webhook connection and
   can never call `Respond to Webhook` for the caller. Any node failure inside the main
   flow (a Sheets API hiccup, a Code node throwing, Gmail auth failing, etc.) meant the
   caller got no HTTP response at all — the request would simply hang until timeout.
2. **No verifiable video walkthrough.**

This document plus the updated workflow file (`Robust_Order_Workflow_v2_FallbackFix.json`)
addresses both.

## 2. The fallback-response fix

Rather than depending solely on the external Error Trigger workflow, the main workflow
now catches its own errors **in the same execution**, guaranteeing a response every time.

**Node-level error routing (`onError` setting) added to every node that touches an
external service or can throw:**

| Node | New `onError` behavior | Why |
|---|---|---|
| Generate Idempotency Key1 | `continueErrorOutput` → generic catch branch | Throws on missing `orderId`/`customerId`/`amount` |
| Check Existing Key1 | `continueErrorOutput` → generic catch branch | Previously `continueRegularOutput`, which silently passed the input through on a real API failure and made the idempotency check falsely look like "no match" or "match" depending on timing — see note below |
| Insert Pending Log Row1 | `continueErrorOutput` → generic catch branch | Sheets append failures were previously unhandled and hard-stopped the run |
| Build Retry Attempts1 | `continueErrorOutput` → generic catch branch | Defensive |
| Build Cached Response1 | `continueErrorOutput` → generic catch branch | Defensive |
| Update Log - Success1 / Payment Failed1 / Inventory Failed1 | `continueRegularOutput` | These sit on a path where we **already know** the correct business outcome (200/409/500). If the *logging* itself fails, we still want the caller to get the correct, specific response — not a generic 500 — so we continue down the same path with best-effort logging rather than diverting. |
| Alert - Payment Failed / Alert - Inventory Failed | `continueRegularOutput` | Same reasoning — a Gmail send failure must never block the response the caller is waiting on. |

**New generic catch-all branch added** for the nodes upstream of any outcome decision
(where we don't yet know whether the right response is 200/409/500):

```
Handle Unexpected Error1 (Code)
    -> Log Unhandled Exception (Inline)1 (Sheets append, onError: continueRegularOutput)
    -> Alert - Unhandled Exception (Inline)1 (Gmail, onError: continueRegularOutput)
    -> Build Unhandled Error Response1 (Code, builds {status:'failed', message:...}, 500)
    -> Respond - Unhandled Error1 (Respond to Webhook)
```

`Handle Unexpected Error1` pulls whatever order context it can (`orderId`,
`idempotencyKey`) from the failing node's input, or falls back to the earlier
`Generate Idempotency Key1` node's output if the failure happened before that data was
attached to the current item. This means **every path through the workflow now ends in
a `Respond to Webhook` call**, whether the outcome is success, a known business failure
(retry-exhausted / inventory conflict), or a genuinely unexpected exception.

The separate **Global Error Handler** workflow (Error Trigger → log → Gmail alert) is
kept in place as a secondary safety net for the rare case a failure happens outside the
main execution path entirely (e.g. the workflow crashing before any node runs), but it
is no longer the *only* thing standing between an exception and a hung request.

### Bonus fix: ambiguous idempotency replay responses

`Build Cached Response1` previously defaulted any row without an explicit `"failed"`
status to a **200 "already processed successfully"** response — including the case
where the status lookup itself came back malformed or incomplete. It now explicitly
handles `success`, `failed`, and `pending` (a duplicate request arriving while the first
is still mid-flight now correctly gets a `202` "still processing" response instead of a
silent false-positive success), and falls back to a `500` rather than assuming success
if the logged status is missing or unrecognized.

## 3. Error simulation (how each path was tested)

Using the Postman mock server saved examples (`x-mock-response-name` header), each
scenario was triggered on demand via the `paymentTestMode` / `inventoryTestMode` fields
in the webhook payload:

- **Happy path** — `paymentTestMode: "normal"` walks the mock through `429 → 500 → 200`
  across the 3 retry attempts, then Inventory returns `200`. Log row ends as `success`,
  caller gets `200`.
- **Retries exhausted** — `paymentTestMode: "always_fail"` forces `500` on every
  attempt. After 3 attempts the workflow logs `status: failed`, `errorType:
  retry_exhausted`, sends the Gmail alert, and responds `500` to the caller.
- **Non-retryable failure** — `inventoryTestMode: "conflict"` returns `409` from the
  Inventory mock after Payment succeeds. This is treated as terminal (no retry), logged
  as `non_retryable_status`, alerted, and the caller gets `409`.
- **Unhandled exception** — simulated by temporarily revoking the Google Sheets OAuth
  credential (or omitting a required field like `orderId`) to force a genuine node
  failure. Before the fix, this hung with no response. After the fix, the caller now
  reliably receives a `500` JSON response, a Sheets row is logged, and an alert email is
  sent — all within the same execution.

## 4. Idempotency demonstration

Sending the same payload (`orderId`, `customerId`, `amount` identical) twice:

1. **First request** — no matching row in the `Order Log` sheet → `Insert Pending Log
   Row1` writes a `pending` row → external calls proceed → row is updated to its final
   status (`success` / `failed`).
2. **Second request (duplicate)** — `Check Existing Key1` finds the row by
   `idempotencyKey` (SHA-256 of `orderId|customerId|amount`) → `Key Exists?1` is true →
   `Build Cached Response1` returns the **stored outcome** immediately, without calling
   either external API again → confirms no double-charge / duplicate inventory
   reservation is possible.

If the duplicate arrives while the first request is still `pending`, the caller now
gets a `202` telling them to retry shortly, rather than a false `200`.

## 5. Deliverables checklist for resubmission

- [x] Updated workflow JSON: `Robust_Order_Workflow_v2_FallbackFix.json`
- [x] This written walkthrough (in place of / alongside a Loom recording)
- [x] The video link is correct (already provided)
- [x] Screenshot of Google Sheet log (already provided)
- [x] Screenshot of Gmail alerts (already provided)
