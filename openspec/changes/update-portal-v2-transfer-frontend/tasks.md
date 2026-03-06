## 1. Types and API Layer
- [x] 1.1 Update TypeScript types in `portal/src/types/transfer.ts`: restructure destination interfaces with nested `beneficiary_info`, `payout_instrument`, `BeneficiaryAddress`; update `BaseTransfer` (add `on_behalf_of`, `fees`, `meta_data`; remove `application_id/uuid/name`); update `OffRampSource` with `payment_instrument`
- [x] 1.2 Switch `getTransfers()` and `getTransfer()` in `portal/src/api/transfers.ts` to use v2 base URL

## 2. Transfer List
- [x] 2.1 Update `getBeneficiaryName()` in `CardView.vue` to read from `destination.beneficiary_info.beneficiary_name`

## 3. Transfer Detail - Deposit (On-Ramp)
- [x] 3.1 Update `DepositTransferDetailContent.vue`: beneficiary section reads from `beneficiary_info`, blockchain from `payout_instrument`, add email/phone fields, optimize amount flow (harbor_fee line, asset labels)

## 4. Transfer Detail - Withdraw (Off-Ramp)
- [x] 4.1 Update `WithdrawTransferDetailContent.vue`: bank details from `payout_instrument` (dynamic key-value rendering), beneficiary/address from `beneficiary_info`, add email/phone/destination_type, optimize amount flow

## 5. Verification
- [x] 5.1 Run `pnpm type-check` to confirm no TypeScript errors
- [x] 5.2 Run `pnpm build` to confirm production build succeeds
