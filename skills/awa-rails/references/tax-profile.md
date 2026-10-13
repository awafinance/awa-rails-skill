# Supported tax profile

Only non-Simples B2B software licensing (`lucro_presumido` or `lucro_real`, national service code `010501`, special regime `0`) with compact federal taxes is supported. Simples, other services/regimes and detailed PIS/COFINS fields return HTTP 422 `tax_profile_unsupported` with “not tested yet”. This restriction is the application policy; the lower-level DPS validator remains in place.

The compact XML contains CST, `tpRetPisCofins=3`, `vRetIRRF`, and `vRetCSLL`. In this convention **vRetCSLL holds total retained CSRF (PIS/COFINS/CSLL)**; `csll_rate` must reflect that total, not CSLL alone. Get all tax choices from the customer/accountant. The numbers below are illustrative, not tax advice.

## Setup and confirmation

`POST /api/v1/companies` accepts an optional `tax_profile` object. Creation never confirms it. Existing companies are not assigned inferred profiles by migration 0004. A company without one receives `422 tax_profile_missing` on emission, including mock emissions.

Create/replace with `PUT /api/v1/companies/{cid}/tax-profile`, or merge fields with `PATCH` on the same path. Both return the profile, `profile_version`, confirmation fields, `created_at`, `updated_at`, `last_review_reminder_at`, and `next_review_at`. `GET` reads it. GET without a profile also returns `422 tax_profile_missing`.

```json
{
  "regime": "lucro_presumido",
  "reg_esp_trib": "0",
  "c_trib_nac": "010501",
  "c_trib_mun": "001",
  "municipality_code": "3304557",
  "municipal_registration": "",
  "iss_rate": "",
  "pis_cofins_cst": "01",
  "pis_cofins_ret_type": "3",
  "irrf_withheld": true,
  "irrf_rate": "1.50",
  "csll_withheld": true,
  "csll_rate": "4.65"
}
```

`reg_esp_trib` defaults to `0`. ISS empty or null delegates its rate to SEFIN. Rates are percentage strings; monetary amounts use two decimals. Metadata is read-only. Unknown fields are rejected.

After reviewing with the customer/accountant, send `POST /api/v1/companies/{cid}/tax-profile/confirm` with `{"profile_version":1}` (use the current version). A stale version returns `409 tax_profile_version_conflict`. Confirmation stores the timestamp and the user id or API key id. Every effective settings change increments the version and clears confirmation; an identical save preserves it. Confirmation is required when the NFS-e capability is anything other than `mock`, independently of the NF-e transport. Unconfirmed mock emissions are allowed. Confirmation does not expire automatically after 90 days; a review reminder becomes due.

MCP provides the same behavior:

- `get_tax_profile`: `{company_id}`.
- `set_tax_profile`: `{company_id, tax_profile}`; merges supplied fields like PATCH.
- `confirm_tax_profile`: `{company_id, profile_version}`.

The dashboard has an optional onboarding form and a company settings form showing confirmation and the next review date. Save changes before confirming the displayed version.

## Emission defaults and overrides

[emission-example.json](emission-example.json) is a profile-defaulted body. Its checksum-valid CNPJ is generated synthetic test data; never use this example for a real emission. Customer address, description, competence, service amount, and approximate federal/state/municipal tax amounts remain required from the caller. The server does not invent approximate tax totals.

Precedence: **`tax_overrides` > existing explicit `issuer`/`service`/`values` fields > profile**. Existing compact explicit-body callers keep their field names; they must provision a profile and confirm it for non-mock emission. The formerly documented detailed PIS/COFINS shape is outside the supported application scope and is now explicitly rejected.

Optional `tax_overrides` fields:

| Field | Overrides |
| --- | --- |
| `municipality_code` | Issuer and service municipality |
| `municipal_registration` | Issuer municipal registration |
| `c_trib_nac`, `c_trib_mun` | Service codes (supported-shape restrictions still apply) |
| `iss_rate` | ISS rate; explicit empty uses SEFIN default |
| `pis_cofins_cst`, `pis_cofins_ret_type` | Compact federal-tax fields |
| `irrf_retention_amount`, `csll_retention_amount` | Monetary withholding amounts |

Omitted withholding amounts are computed from `service_amount × rate / 100`, using exact decimals and half-up rounding to cents. For example, `100.33 × 4.65%` produces `4.67`. An explicit zero or empty IRRF amount is rejected, including in existing `values.federal_taxes` bodies. For retention type 3, both retained flags and positive rates/amounts are required. IRRF rounded to zero is also rejected before sending, to prevent SEFIN **E0700**. Invalid merged fields return `422 tax_profile_invalid`, with an `error.fields` map and field names in the message. The final merged DPS is checked before allocating a number or saving an emission. Its payload contains `tax_profile_snapshot` with the source profile, version, and confirmation.

