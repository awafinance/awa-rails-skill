# Search received invoices: MCP and GraphQL

`search_notes` searches **stored received NF-e and NFS-e** for one authorized `company_id`. It never fetches provider documents, emits, manifests, consumes a fiscal quota unit, or changes the receipt cursor. It does not include your own NFS-e emission operations; use `list_emissions` for those. Search completeness is limited to what Rails has received, not the complete government history.

## Filters and pagination

Optional filters combine with AND:

- `document_type`: `nfe` or `nfse`; omitted searches both.
- `supplier_cnpj`: exact issuer CNPJ; accepts 14 digits or normal punctuation.
- `number`: exact numeric fiscal number as a string; leading zeros are normalized. **Number is not unique**. Preserve all matches from different suppliers, series or document types. Always present type, number, supplier, series when available, and key.
- `access_key`: exact 44-digit NF-e key or 50-digit NFS-e key, as a string. This is distinct from the internal UUID `id` and the distribution NSU.
- `amount_min_cents`, `amount_max_cents`: inclusive integer-cent boundaries. For an exact amount, set both equal. The amount is the persisted document amount: NF-e total and NFS-e liquid value (`vLiq`), not a payment due amount.
- `issued_from`, `issued_to`: RFC3339 timestamps with timezone; start inclusive, end exclusive. Convert the user's local calendar dates to the intended timezone boundaries; do not silently treat dates as UTC. Notes without an issue date do not match a dated filter.
- `environment`: `production`, `homologation` or `mock`.
- `first`: page size, default 50, allowed 1–200.
- `after`: opaque cursor from the preceding page.

Response: `notes` and `page_info {has_next_page, end_cursor}`. Every note includes `id`, `document_type`, `number`, `series` (empty for NFS-e), `access_key`, supplier CNPJ/name, `amount_cents`, `issued_at`, `received_at`, `environment` and `mocked`.

To list all stored matching notes, keep the same company and filters, call again with `after=page_info.end_cursor` while `has_next_page` is true, and combine pages. Never claim a single page is the full history. Cursors are opaque, authenticated and bound to the company and normalized filters. Changing filters requires a fresh search. Pagination is ordered by receipt time with stable type/id tie-breakers; a cursor's upper time bound excludes newly received records after the search starts. It is not a database snapshot across concurrent transactions. A fresh search shows subsequent receipts.

Synthetic MCP example:

```json
{
  "company_id": "00000000-0000-4000-8000-000000000001",
  "document_type": "nfe",
  "supplier_cnpj": "90000000000265",
  "number": "42",
  "amount_min_cents": 10000,
  "amount_max_cents": 50000,
  "issued_from": "2026-01-01T00:00:00-03:00",
  "issued_to": "2026-02-01T00:00:00-03:00",
  "first": 50
}
```

Omit supplier to find every supplier with number 42. Use an exact `access_key` to select a specific fiscal document, then use the returned `id` with `get_note` or `get_received_nfse` for detail. Existing NF-e list/detail tools now expose both number and series, derived from the fiscal access key even for previously stored summaries.

There is no due-date filter. Financial maturity is not indexed by these fiscal queries. Do not treat emission date or competence as maturity.

## GraphQL

Authenticated `POST <AWA_RAILS_URL>/graphql`, same bearer API key/browser session as REST. `Content-Type: application/json` is required. This schema is query-only. Discover the live schema through authenticated introspection; do not invent mutations.

```graphql
query FindNotes($company: ID!, $filter: NoteSearchFilter, $after: String) {
  searchNotes(companyId: $company, filter: $filter, first: 50, after: $after) {
    notes {
      id documentType number series accessKey
      supplierCnpj supplierName amountCents issuedAt environment mocked
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

Synthetic variables:

```json
{
  "company": "00000000-0000-4000-8000-000000000001",
  "filter": {
    "documentType": "nfe",
    "number": "42",
    "amountMinCents": "10000",
    "amountMaxCents": "50000",
    "issuedFrom": "2026-01-01T00:00:00-03:00",
    "issuedTo": "2026-02-01T00:00:00-03:00"
  }
}
```

GraphQL field/filter names are camelCase: `supplierCnpj`, `accessKey`, `amountMinCents`, `amountMaxCents`, `issuedFrom`, `issuedTo`, `documentType`, `environment`. The `Long` scalar accepts exact integers or decimal integer strings and returns strings, preserving 64-bit monetary precision; prefer strings for input amounts. `DateTime` requires a timestamp with timezone.

HTTP200 may contain GraphQL `errors`; inspect both `data` and `errors` before reporting a successful search. The endpoint bounds page size, query depth/length, execution time and ten search resolvers per request. Authentication and company authorization apply equally to MCP and GraphQL; an access key is a filter, not permission to read another company's data.
