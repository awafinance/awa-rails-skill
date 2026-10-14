---
name: awa-rails
description: Set up Awa Rails for a coding agent, emit Brazilian NFS-e, receive NF-e and NFS-e and configure signed webhook notifications through its remote MCP or REST API. Use for Awa Rails onboarding and fiscal-document workflows; NF-e issuance is not an available MCP operation.
---

# Awa Rails

Public skill: https://github.com/awafinance/awa-rails-skill
Default service: `https://rails.awafinance.com`. Use a different `AWA_RAILS_URL` only when the user chooses it, preserving any base path.
Remote MCP: `<AWA_RAILS_URL>/mcp`, Streamable HTTP.
Authentication: `Authorization: Bearer <AWA_RAILS_API_KEY>`; API keys start with `afk_`.

For an installed skill, use the local `references/` copies. The links below also work when reading the service's `/SKILL.md` online.

## Setup

For installation and client-specific configuration, read [references/install.md](https://raw.githubusercontent.com/awafinance/awa-rails-skill/main/skills/awa-rails/references/install.md). Install the complete skill directory, including its references, in the current agent's supported skill location. Preserve existing skills and MCP servers.

1. Have the user sign in at `<AWA_RAILS_URL>/app/`, finish company onboarding and wait for beta approval if required. An API key is created in **Chaves de API** and shown once. An agent cannot approve its own organization.
2. Guide key entry through the client's secret settings or a local secret store as `AWA_RAILS_API_KEY`. Do not request keys, certificate passwords or certificate contents in chat, print them, or commit them. If the host renders MCP arguments in the conversation, use the browser certificate form or a secret-backed REST upload instead of exposing credentials in `upload_certificate` arguments.
3. Configure the actual agent client and check MCP initialization and `tools/list`. Then call `get_transport_capabilities` and `list_companies`. Select the company by CNPJ; do not pick the first result when multiple companies exist.
4. Report which operations are real homologation and which are mocked. Reading capabilities, listing companies and listing existing notes/emissions are sufficient connection checks. Setup alone does not authorize emission, manifestation, certificate replacement or provider calls.

## Fiscal operations

Read [references/operations.md](https://raw.githubusercontent.com/awafinance/awa-rails-skill/main/skills/awa-rails/references/operations.md) when preparing an emission, distribution sync or manifestation. Discover the live tool schemas with `tools/list` rather than guessing arguments.

- **Emit NFS-e:** obtain the user's invoice instructions and accountant-reviewed tax profile, use `get_tax_profile`, `set_tax_profile` when needed and `confirm_tax_profile` for the exact reviewed version. Then use `emit_nfse` with a stable `idempotency_key`. If the outcome is `ambiguous`, use `reconcile_emission`; never create a new key and resend blindly.
- **Receive NFS-e:** check `nfse_receipt` and the selected company's `nfse_receipt_enabled`. The pilot is disabled by default and has an explicit company allowlist. With authorization, use `receive_nfse` with a known ADN NSU, then `list_received_nfse` or `get_received_nfse`. For continuous receipt, read `get_nfse_receipt_sync` and, with explicit authorization, use `set_nfse_receipt_sync` with `enabled=true` and `start_after_nsu` on first activation. The independent persisted cursor is preserved on pause/resume. Both modes use GET-only provider access and cannot emit or manifest. Replays do not call the provider or consume another quota unit.
- **Receive NF-e:** use `sync_notes`, `list_notes` and `get_note`. A sync calls SEFAZ and must be within the user's authorized workflow. Respect distribution cooldowns and `sefaz_throttled`. Zero documents is a valid result; it does not prove the receiving flow or a new-document webhook worked.
- **Manifestation:** `manifest_note` supports ciência (`210210`), confirmação (`210200`), desconhecimento (`210220`) and operação não realizada (`210240`, justification required). Ciência is not confirmation of the transaction. Obtain the appropriate user instruction before sending an event or enabling `set_auto_ciencia`.
- **Certificates:** use the company settings form or `upload_certificate` with an authorized A1 e-CNPJ certificate matching the company CNPJ root. The current homologation `test_certificate` validates certificate material locally and reports `verification: "certificate_material"`, `provider_contacted: false`. Inspect those fields before describing the check; it does not prove provider connectivity.

Always inspect `environment`, `mocked` and the operation's status/protocol. Capabilities are independent: real NFS-e does not imply real NF-e distribution or events. Mock results have no fiscal validity. Homologation is a test environment. Production emission and NF-e events remain blocked; the NFS-e receipt pilot may use production for an explicitly enabled company. Do not change server flags to bypass a block.

The MCP **emits NFS-e and receives NF-e**, plus one existing NFS-e by a known NSU when the receipt pilot is enabled. It has no NF-e issuance tool. Do not promise NF-e issuance, all tax regimes or production issuance. NFS-e polling requires explicit company activation.

## Notifications

Read [references/webhooks.md](https://raw.githubusercontent.com/awafinance/awa-rails-skill/main/skills/awa-rails/references/webhooks.md) to set up an HTTPS receiver and a user-selected notification destination. Store the signing secret outside chat/source, verify signatures on raw bytes, deduplicate deliveries, and test with `webhook.test` before claiming the receiver is ready.

A webhook is delivered when the Rails app ingests a document; it does not itself poll SEFAZ, and an MCP connection does not push a message into an idle agent. If ongoing receipt is requested, arrange authorized `sync_notes` scheduling that respects the cooldown, and connect the receiver to a notification service or a supported agent wake-up mechanism.

## Handling failures

Check MCP `result.isError` and its text content even when HTTP status is 200. REST errors use `{"error":{"code":"...","message":"..."}}`.

- `401 unauthorized`: resolve the key in secret settings; never reveal its value.
- `403 pending_approval`: guide the user to the pending account; do not retry indefinitely.
- `no_certificate`, `cnpj_mismatch`, `cert_expired`, `bad_pfx`: correct the selected certificate with the user.
- `tax_profile_missing`, `tax_profile_unconfirmed`, `tax_profile_unsupported`, `tax_profile_invalid`, `tax_profile_version_conflict`: review the profile/fields and version, without inventing tax choices.
- `429 sefaz_throttled`: wait until the returned retry time; repeated early retries can prolong the provider block.
- Ambiguous fiscal outcomes or transport errors: inspect saved operation state before any retry and retain the original idempotency key.

Finish with what was installed, the MCP endpoint, selected company, active capabilities, verified checks and any remaining user action. Separate connection verification, mock tests, homologation authorization and live webhook receipt.
