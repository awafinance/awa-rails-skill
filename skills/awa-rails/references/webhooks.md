# Receive signed notifications

Awa Rails sends HTTPS POST deliveries to a company's configured webhook URL. It does not send pushes directly to an MCP client. Help the user connect a receiver to their chosen notification destination (for example, an existing automation service, app backend or agent inbox). Sending a notification, deploying a receiver, registering a URL and scheduling syncs must stay within the user's instructions.

## Configure a receiver

1. Prefer the user's existing service. The endpoint must be publicly reachable over HTTPS, accept the raw request body, preserve the signature headers, store a secret, and deduplicate deliveries. Production delivery blocks private/loopback/link-local destinations and does not follow redirects. Do not use a public request bin for actual fiscal documents or secrets. If a no-code service cannot verify the raw-body HMAC, put a verifying receiver in front of it.
2. With authorization, call `set_webhook` with `{company_id, url}`. On the first save the response includes `webhook_secret` once. Save it directly to the receiver's secret store without printing it or placing it in source/chat. If the MCP host exposes secret outputs in chat, register using a secret-backed REST client that writes that field straight to the secret store. Never invent or ask the user to paste a signing secret in chat.
3. An existing webhook may not return the secret again. Use the existing secret if available. Do not set `rotate_secret: true` merely to retrieve a key; coordinate a rotation with the existing receiver to avoid disrupting delivery.
4. Verify the signature before parsing/processing the payload. Header:

   `Awa-Rails-Signature: t=<unix>,v1=<hex>`

   Compute `HMAC_SHA256(secret, ASCII(t) + "." + raw_body_bytes)` and compare the hex digest in constant time. Reject malformed signatures and timestamps outside a five-minute window (including future timestamps). Use the exact raw body; JSON parse/reserialize changes its bytes.
5. Deduplicate using `Awa-Rails-Delivery`, which remains stable across retries. Persist the accepted delivery/job before returning 2xx; send the user notification asynchronously. Awa retries unsuccessful deliveries up to six total attempts with backoff. Replayed deliveries must not generate duplicate notifications.

## Test and notify

There is currently no webhook-test MCP tool. Use the authenticated REST operation:

`POST <AWA_RAILS_URL>/api/v1/companies/{company_id}/webhook/test`

It queues `webhook.test`; a queued response is not proof of delivery. Check the receiver's verified event and, when authorized, its notification result. The test never emits or manifests an invoice. It proves receiver plumbing only; live fiscal receipt requires a genuine `nfe.received` or `nfse.received` event.

Envelope:

```json
{
  "event": "nfe.received",
  "company_id": "<company-id>",
  "created_at": "2026-10-01T12:00:00Z",
  "data": {
    "note_id": "<note-id>",
    "chave": "<44-digit-access-key>",
    "kind": "summary",
    "emitter_cnpj": "<issuer-cnpj>",
    "emitter_name": "<issuer-name>",
    "amount_cents": 10000
  }
}
```

Events: `nfse.received`, `nfe.received`, `nfe.manifested`, `nfe.full_xml_available`, `nfse.emitted`, `tax_profile.review_required`, `webhook.test`. Use `nfe.received` for NF-e and `nfse.received` for NFS-e new-document notifications. The NFS-e pilot receives one known NSU; it does not start continuous polling. `nfe.full_xml_available` indicates an upgrade of an existing note, not necessarily a new invoice.

A fiscal event payload does not itself certify a real fiscal operation. Use its `company_id`/`note_id` with `get_note` (or `list_emissions` for NFS-e) to inspect `environment`, `mocked` and status before claiming validity. Notify only the user-selected destination and disclose the minimal fiscal information needed there.

## Keep receiving documents

The receiver is reactive: it gets a notification after a Rails sync ingests a document. Setting the URL or enabling auto-ciência does not start polling. NFS-e receipt has a separate opt-in worker configured with `set_nfse_receipt_sync`; it runs independently of browser visits. If the user requests ongoing receipt, configure an authorized scheduler to call `sync_notes` for the selected company, respecting SEFAZ's one-hour cooldown after empty/caught-up responses and any returned retry timestamp. Coordinate with other systems using the same CNPJ. An idle agent needs its platform's inbox/wake-up integration or a separate notification service; an MCP connection alone cannot wake it.

Report receiver deployment, signature verification, test-event delivery, notification delivery, scheduler setup and live fiscal delivery as separate results. Leave a clear pending state when there is no new fiscal document.
