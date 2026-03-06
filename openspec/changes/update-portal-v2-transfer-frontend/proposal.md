# Change: Migrate Portal Transfer List/Detail to v2 API and Optimize Amount Flow

## Why
The Portal frontend currently fetches transfer list and detail data from v1 API endpoints, which lack beneficiary_info (dob, email, phone, id_doc), full payout_instrument, and beneficiary_address. The v2 backend endpoints (built in `add-portal-v2-transfer-detail`) restructure the response with nested `beneficiary_info` and `payout_instrument` objects. The frontend needs to migrate to v2 and update all data access patterns, types, and UI accordingly. The amount flow section also needs visual optimization to better communicate the fee structure.

## What Changes

### API Layer
- Switch `getTransfers()` and `getTransfer()` from v1 base URL to v2 (`VITE_APP_PORTAL_API_V2`)
- No new API functions needed — same HTTP methods, just different base URL

### TypeScript Types
- Restructure `OnRampDestination` and `OffRampDestination` to match v2 response:
  - New `BeneficiaryInfo` and `BeneficiaryAddress` interfaces
  - `payout_instrument` nested object (ON_RAMP: chain/address/memo, OFF_RAMP: merged from JSON)
  - Remove flat beneficiary/bank fields from destination interfaces
  - Add `on_behalf_of`, `fees`, `meta_data` to BaseTransfer
  - Remove `application_id`, `application_uuid`, `application_name` (not in v2 response)
- Add `PayoutInstrument` type for OFF_RAMP (dynamic key-value from `destination_local_payout_instrument_json`)
- Update `OffRampSource` to include `payment_instrument` sub-object

### Transfer List (CardView)
- Update `getBeneficiaryName()` to read from `destination.beneficiary_info.beneficiary_name`

### Transfer Detail - DepositTransferDetailContent
- Beneficiary section: read from `destination.beneficiary_info.*` instead of `destination.*`
- Address: read from `destination.beneficiary_info.beneficiary_address.*` with new field names (`street` vs `residential_address_1`)
- Blockchain details: read chain/address/memo from `destination.payout_instrument.*`
- Show new fields: `beneficiary_email`, `beneficiary_phone_number`

### Transfer Detail - WithdrawTransferDetailContent
- Bank details: read from `destination.payout_instrument.*` instead of `destination.*`
- Beneficiary/address: read from `destination.beneficiary_info.beneficiary_address.*`
- Show new fields: `beneficiary_email`, `beneficiary_phone_number`, `destination_type`
- Render payout_instrument dynamically (key-value) since OFF_RAMP fields vary by country/payment method

### Amount Flow (Both Detail Components)
- Add harbor_fee as separate line in fee breakdown (currently only commission_fee shown)
- Show initial/final asset labels for clarity (e.g., "100 USD" not just "100")
- Visual polish: better spacing, clearer fee hierarchy

## Impact
- Affected specs: portal-transfer-ui (new capability)
- Affected code:
  - `portal/src/api/transfers.ts` — switch 2 endpoints to v2
  - `portal/src/types/transfer.ts` — restructure destination interfaces
  - `portal/src/views/Transfers/components/CardView.vue` — beneficiary name accessor
  - `portal/src/components/DepositTransferDetailContent.vue` — v2 data paths + amount flow
  - `portal/src/components/WithdrawTransferDetailContent.vue` — v2 data paths + amount flow
- Non-breaking to backend: v1 endpoints remain, v2 already deployed
- Portal-only change — no Admin or API modifications
