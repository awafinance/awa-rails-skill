# Fiscal operations through MCP

Always discover current schemas with `tools/list`. These tool names are the current contract:

| Purpose | Tools |
| --- | --- |
| Check active environments | `get_transport_capabilities` |
| Company and certificate | `list_companies`, `upload_certificate`, `test_certificate` |
| Tax profile | `get_tax_profile`, `set_tax_profile`, `confirm_tax_profile` |
| Received NFS-e pilot | `receive_nfse`, `list_received_nfse`, `get_received_nfse`, `get_nfse_receipt_sync`, `set_nfse_receipt_sync` |
| Received NF-e | `sync_notes`, `list_notes`, `get_note` |
| Recipient events | `manifest_note`, `set_auto_ciencia` |
| NFS-e emission | `emit_nfse`, `reconcile_emission`, `list_emissions` |
| Signed notifications | `set_webhook` |

`tools/call` arguments use `company_id`; select it from `list_companies` by the user's CNPJ. The JSON result is in MCP text content; `isError: true` means failure even with HTTP 200.

## NFS-e emission

1. Check `nfse_emission`. `mock` is simulation, `homologation` is a real test-provider call, and production is blocked. Do not infer one capability from another.
2. Check the certificate and read the tax profile. If needed, use `set_tax_profile` with `{company_id, tax_profile}`. Changes reset confirmation. Have the user/accountant review the actual choices before `confirm_tax_profile` with `{company_id, profile_version}`. Never silently confirm on their behalf.
3. Only B2B software licensing (`c_trib_nac=010501`) for `lucro_presumido` or `lucro_real`, special regime `0`, with compact federal withholding is supported. Other shapes are not tested and are rejected. See [tax-profile.md](tax-profile.md).
4. Gather customer CNPJ/name/address, competence, service description/amount and approximate federal/state/municipal tax totals from the user. [emission-example.json](emission-example.json) illustrates the shape with synthetic data; never submit it as a customer's invoice.
5. With the user's emission instruction, call `emit_nfse` with `{company_id, emission}`. The emission contains `idempotency_key`, `competence`, `customer`, `service`, `values`. The service allocates DPS numbering. Use a stable key for that invoice and preserve it across retries.
6. Inspect status, `mocked`, `environment`, DPS id and any authorization evidence. If ambiguous, call `reconcile_emission` with `{company_id, emission_id}`; it checks the saved DPS without resubmitting. Do not change the key to force a second emission. Use `list_emissions` to inspect saved outcomes.

## NF-e receipt

1. Check `nfe_distribution` and select the recipient company's A1 e-CNPJ. The certificate CNPJ root must match. A certificate upload/replacement requires the user's instruction and a secure secret route.
2. With authorization to sync, call `sync_notes` with `{company_id}`. Then use `list_notes` with `{company_id, kind, limit}` (`kind`: `summary`, `full`, or empty; limit at most 200), and `get_note` with `{company_id, note_id}`.
3. The app saves and advances the SEFAZ cursor. Do not manually reset NSU. After an empty/end-of-stream distribution response, wait at least one hour; obey any `sefaz_throttled` retry time. Avoid competing schedulers for the same CNPJ, including other apps. Zero received notes is not proof that an invoice authorized by a state service has reached national distribution.
4. For ongoing receipt, arrange an authorized scheduled caller for `sync_notes`. The Rails service currently has no automatic distribution polling worker. A webhook forwards ingested documents; it cannot make SEFAZ distribute a document.

## Recipient manifestation

Check `nfe_events` separately from distribution. Only send the event instructed by the user for the selected note:

- `210210`: ciência da emissão; awareness, not confirmation of the transaction.
- `210200`: confirmação da operação.
- `210220`: desconhecimento da operação.
- `210240`: operação não realizada; supply `justification` of 15–255 characters.

Call `manifest_note` with `{company_id, note_id, event_type, justification?}`. Inspect the resulting status/protocol/environment; a repeated registered request can return `already_registered`. A linked event can make the full XML available, but `get_note` currently returns metadata and manifestation history, not an XML download. Do not invent a download tool or route.

`set_auto_ciencia` with `{company_id, enabled}` applies to new summary NF-e ingested by a sync. It neither starts a background sync nor sends automatic confirmation. Enable it only within the user's authorized workflow.

## REST equivalents

Base path: `<AWA_RAILS_URL>/api/v1`. Same bearer key. Company paths begin `/companies/{company_id}`.

- `GET /transport/capabilities`, `GET /companies`.
- `PUT /companies/{id}/certificate` (multipart `pfx` and `password`, or JSON `pfx_base64` and `password`); `POST .../certificate/test`.
- `GET/PUT/PATCH .../tax-profile`; `POST .../tax-profile/confirm` with `{profile_version}`.
- `POST .../sync`; `GET .../notes`; `GET .../notes/{note_id}`; `POST .../notes/{note_id}/manifestations` with `{event_type, justification?}`.
- `POST .../emissions` with the emission body; `GET .../emissions`; `POST .../emissions/{emission_id}/reconcile`.
- `PATCH /companies/{id}` with `auto_ciencia`, `webhook_url`, or `rotate_webhook_secret` as needed; `POST .../webhook/test`.

REST errors have an HTTP status and `error.code`/`error.message`; validation can add `error.fields`. For `trial_exhausted`, direct the user to the account's actual quota/billing state rather than retrying or creating another organization.

## Receive one existing NFS-e

Check `nfse_receipt` and the company's `nfse_receipt_enabled`. This pilot requires an explicit company allowlist and a known NSU from the contributor's existing ADN distribution. With authorization to read that document, call `receive_nfse` with `{company_id, nsu}`. Then call `list_received_nfse` with `{company_id}` or `get_received_nfse` with `{company_id, note_id}` for XML.

The response contains `note.environment`, `mocked=false`, `provider_contacted` and `idempotent_replay`. A stored replay makes no new provider call, quota reservation or webhook. A new receipt queues `nfse.received` when a receiver is configured. NF-e manifestation tools do not apply to NFS-e. This is a point lookup, not continuous distribution or a cursor takeover. Emission remains under its separate gate.

REST: `POST /companies/{cid}/received-nfse` with `{"nsu":"42"}`; `GET /companies/{cid}/received-nfse`; `GET /companies/{cid}/received-nfse/{note_id}` under `/api/v1`.

## Continuous NFS-e receipt

Read `get_nfse_receipt_sync` for the selected company. First activation requires explicit authorization and `{company_id, enabled: true, start_after_nsu: "57"}` via `set_nfse_receipt_sync`. The number is an example: choose the user's intended starting cursor. `0` requests history; do not backfill without authorization. Subsequent `{company_id, enabled: false/true}` pauses/resumes without resetting the cursor or bypassing backoff.

The worker runs without browser traffic, at 15-minute intervals after advancing a page and one hour after empty results or failures. Only complete, validated NFS-e addressed to the company are ingested. Issued/intermediary notes are skipped after validation; unknown document formats stop processing and preserve the cursor for diagnosis. Each page's notes, quota, webhooks and cursor commit atomically. Failed pages are retried; duplicates consume no extra quota. Quota exhaustion is visible in the sync status. A webhook destination is optional, and no notification is delivered without one. This worker has its own per-company/environment cursor and does not take over another system's cursor.

REST: `GET/PATCH /api/v1/companies/{cid}/nfse-receipt-sync`. `last_success_at` proves the last completed poll, not receipt of a new note or webhook delivery. No NF-e polling or fiscal events are enabled by this setting.
