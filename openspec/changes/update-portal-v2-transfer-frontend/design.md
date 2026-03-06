## Context
The v2 API restructures the transfer response with nested objects (`beneficiary_info`, `payout_instrument`) instead of flat fields. The frontend must update all data access patterns. The OFF_RAMP `payout_instrument` is particularly challenging because its keys vary by country/payment method (e.g., `br_cpf` for Brazil, `hk_fps_id` for Hong Kong).

## Goals / Non-Goals
- Goals: Migrate to v2 API, update types, update all detail components, improve amount flow
- Non-Goals: Adding new pages, changing transfer creation flow, adding new features beyond displaying existing v2 data

## Decisions

### Type Strategy
- Keep discriminated union pattern (`OnRampTransfer | OffRampTransfer`) — it works well
- `payout_instrument` for OFF_RAMP uses `Record<string, string>` since fields are dynamic per country
- `payout_instrument` for ON_RAMP uses a strict typed object `{ chain, address, address_memo }`
- `BeneficiaryInfo` is a shared interface used by both transfer types
- Fields omitted by backend (`array_filter` removes nulls) are all marked optional (`?`)

### OFF_RAMP Payout Instrument Rendering
- Since payout_instrument keys vary by country/payment method, render dynamically as key-value pairs
- Format keys from snake_case to Title Case for display (e.g., `account_number` -> "Account Number")
- Known sensitive fields (account_number) rendered in monospace
- This future-proofs against new country-specific fields without frontend changes

### Amount Flow Enhancement
- Add harbor_fee as a separate line below commission_fee in the fee breakdown
- Show asset labels on source and destination amounts (already shown, but ensure consistency)
- Keep the existing visual hierarchy: source (neutral) -> fees (orange) -> destination (emerald)
- Add a fee total line when both commission and harbor fees exist

### v1 BaseTransfer Fields
- `application_id`, `application_uuid`, `application_name` are NOT in v2 response
- Grep confirms no views/components access these fields — safe to remove from types
- The `application_id` in `TransferFilters` is a query parameter (not from response) — keep it

## Risks / Trade-offs
- **Risk**: OFF_RAMP payout_instrument dynamic rendering may show unexpected/ugly keys for new countries
  - Mitigation: snake_case -> Title Case formatter handles most cases gracefully
- **Risk**: v1 consumers outside our scope (if any external tools query v1)
  - Mitigation: v1 endpoints remain unchanged, only the portal frontend switches

## Open Questions
- None — all structure decisions verified against v2 backend response
