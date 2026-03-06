# Change: Add Portal v2 Transfer List and Detail Endpoints

## Why
Portal v1 transfer endpoints (`GET /api/v1/portal/.../transfers`) return only basic destination fields (beneficiary_name, bank details). They omit most requirements fields that users fill during create transfer v2 — beneficiary_info (dob, email, phone, id_doc), full payout_instrument JSON, and beneficiary_address. Users cannot review the complete data they submitted.

## What Changes
- New `Portal/v2/TransferResource` extending Application V2 TransferResource, adding `beneficiary_info` nested section with all requirements fields
- New `list` and `show` methods on existing `Portal/v2/TransferController`
- New GET routes in `routes/portal/v2.php` for transfer list and detail
- OFF_RAMP transfers merge `destination_local_payout_instrument_json` for complete payout_instrument (country-specific fields like br_cpf, hk_fps_id, etc.)
- `array_filter` omits null fields for cleaner responses
- `aml_sender_travel_rule_data_json` excluded (internal AML compliance data, not user-submitted)
- Updated `TransferFactory::offRamp()` state with complete requirements fields for testing

## Impact
- Affected specs: portal-transfer-api (new capability)
- Affected code:
  - Create: `api/app/Http/Resources/Portal/v2/TransferResource.php`
  - Modify: `api/app/Http/Controllers/Portal/v2/TransferController.php`
  - Modify: `api/routes/portal/v2.php`
  - Modify: `api/database/factories/TransferFactory.php`
  - Create: `api/tests/Feature/Portal/V2/TransferControllerTest.php`
- Non-breaking: v1 endpoints remain unchanged
- Portal frontend update deferred to separate change
