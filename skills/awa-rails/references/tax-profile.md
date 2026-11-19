# Supported tax profile

Supported B2B services are software licensing (`010501`), IT consulting (`010601`) and general consulting (`170101`), with special regime `0`. The regimes are `lucro_presumido`, `lucro_real` (compact retained federal taxes) and `simples_nacional` (ME/EPP, all taxes inside SN, no withholdings). MEI, Simples with taxes outside SN, other services/regimes and detailed PIS/COFINS remain unsupported and return HTTP 422. Production emission remains disabled.

The compact XML contains CST, `tpRetPisCofins=3`, `vRetIRRF`, and `vRetCSLL`. In this convention **vRetCSLL holds total retained CSRF (PIS/COFINS/CSLL)**; `csll_rate` must reflect that total, not CSLL alone. Get all tax choices from the customer/accountant. The numbers below are illustrative, not tax advice.

## Simples Nacional (ME/EPP)

Use `regime=simples_nacional`, `simple_national_assessment_regime=1` (federal taxes and ISS inside SN), and an explicit `simple_national_total_tax_rate` from 0 to 99.99. The latter is the nominal rate for the approximate tax disclosure, **not** the ISS rate or a calculated monthly tax bill. Review it with the customer/accountant; the API does not infer it from a CNPJ, service or amount.

Synthetic test profile (the rate is illustrative):

```json
{
  "regime": "simples_nacional",
  "reg_esp_trib": "0",
  "c_trib_nac": "010601",
  "c_trib_mun": "",
  "municipality_code": "3304557",
  "municipal_registration": "",
  "simple_national_assessment_regime": "1",
  "simple_national_total_tax_rate": "6.00",
  "iss_rate": "",
  "pis_cofins_cst": "",
  "pis_cofins_ret_type": "",
  "irrf_withheld": false,
  "irrf_rate": "",
  "csll_withheld": false,
  "csll_rate": ""
}
```

For emission, supply the same customer/address, competence and service fields as [emission-example.json](emission-example.json), but `values` needs only `service_amount` when using the profile's approximate rate. Omit `federal_taxes` and all three `approximate_*_tax_amount` fields. Do not send zero-valued withholding objects: the API rejects incompatible fields rather than silently discarding them. The resulting DPS has `opSimpNac=3`, `regApTribSN=1`, `tpRetISSQN=1` and `pTotTribSN`; it has neither `tribFed` nor `pAliq` nor monetary tax estimates.

The nominal disclosure rate can be overridden with `values.simple_national_total_tax_rate` or `tax_overrides.simple_national_total_tax_rate` (highest priority). Empty/invalid overrides are rejected. Assessment regimes 2/3, ISS retention and federal-tax overrides are not supported for this initial Simples shape. Non-Simples profiles must leave both Simples fields empty. Changing regimes via PATCH requires explicitly clearing incompatible fields; PUT replaces the profile. Either effective change resets confirmation as usual.

`c_trib_mun` is optional for every supported regime. Leave it empty if the municipality has no specialization configured for the national code; the XML then omits `cTribMun`. A supplied value must have exactly three digits. The national code always remains required.

Sources: [official national service codes](https://www.gov.br/nfse/pt-br/mei-e-demais-empresas/codigos-de-tributacao-nacional-nbs); [national NFS-e FAQ, 6 November 2026, sections 20.2, 20.5 and 21.6](https://www.gov.br/nfse/pt-br/perguntas-frequentes/perguntas-frequentes-nfs-e-v1-3-20261106.pdf/@@download/file). Homologation validation does not enable production emission or establish a company's actual tax treatment.

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

`reg_esp_trib` defaults to `0`. For non-Simples, ISS empty or null delegates its rate to SEFIN; for the supported Simples shape it must remain empty. Rates are percentage strings; monetary amounts use two decimals. Metadata is read-only. Unknown fields are rejected.

After reviewing with the customer/accountant, send `POST /api/v1/companies/{cid}/tax-profile/confirm` with `{"profile_version":1}` (use the current version). A stale version returns `409 tax_profile_version_conflict`. Confirmation stores the timestamp and the user id or API key id. Every effective settings change increments the version and clears confirmation; an identical save preserves it. Confirmation is required when the NFS-e capability is anything other than `mock`, independently of the NF-e transport. Unconfirmed mock emissions are allowed. Confirmation does not expire automatically after 90 days; a review reminder becomes due.

MCP provides the same behavior:

- `get_tax_profile`: `{company_id}`.
- `set_tax_profile`: `{company_id, tax_profile}`; merges supplied fields like PATCH.
- `confirm_tax_profile`: `{company_id, profile_version}`.

The dashboard has an optional onboarding form and a company settings form showing confirmation and the next review date. Save changes before confirming the displayed version.

## Emission defaults and overrides

[emission-example.json](emission-example.json) is a profile-defaulted body. Its checksum-valid CNPJ is generated synthetic test data; never use this example for a real emission. Customer address, description, competence, service amount, remain required from the caller. Non-Simples emissions additionally require approximate federal/state/municipal tax amounts; Simples emissions use the explicit nominal rate instead. The server does not invent approximate tax totals.

Precedence: **`tax_overrides` > existing explicit `issuer`/`service`/`values` fields > profile**. Existing compact explicit-body callers keep their field names; they must provision a profile and confirm it for non-mock emission. The formerly documented detailed PIS/COFINS shape is outside the supported application scope and is now explicitly rejected.

Optional `tax_overrides` fields:

| Field | Overrides |
| --- | --- |
| `municipality_code` | Issuer and service municipality |
| `municipal_registration` | Issuer municipal registration |
| `c_trib_nac`, `c_trib_mun` | Service codes (supported-shape restrictions still apply) |
| `iss_rate` | ISS rate (non-Simples); explicit empty uses SEFIN default |
| `simple_national_total_tax_rate` | Approximate nominal Simples rate, validated from 0 to 99.99 |
| `pis_cofins_cst`, `pis_cofins_ret_type` | Compact federal-tax fields |
| `irrf_retention_amount`, `csll_retention_amount` | Monetary withholding amounts |

For non-Simples, omitted withholding amounts are computed from `service_amount × rate / 100`, using exact decimals and half-up rounding to cents. For example, `100.33 × 4.65%` produces `4.67`. An explicit zero or empty IRRF amount is rejected, including in existing `values.federal_taxes` bodies. For retention type 3, both retained flags and positive rates/amounts are required. IRRF rounded to zero is also rejected before sending, to prevent SEFIN **E0700**. Invalid merged fields return `422 tax_profile_invalid`, with an `error.fields` map and field names in the message. The final merged DPS is checked before allocating a number or saving an emission. Its payload contains `tax_profile_snapshot` with the source profile, version, and confirmation.

