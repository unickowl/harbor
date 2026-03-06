## 1. Implementation
- [x] 1.1 Create `Portal/v2/TransferResource` extending Application V2 TransferResource with `getBeneficiaryInfo()` method
- [x] 1.2 Add `list` and `show` methods to existing `Portal/v2/TransferController`
- [x] 1.3 Register GET routes in `routes/portal/v2.php` for transfer list and detail
- [x] 1.4 Update `TransferFactory::offRamp()` with complete requirements fields (dob, phone, email, id_doc, address, local_payout_instrument_json)
- [x] 1.5 Write feature tests for list/show endpoints covering auth, structure, off-ramp payout_instrument, null field omission
- [x] 1.6 Run full test suite and verify no regressions
