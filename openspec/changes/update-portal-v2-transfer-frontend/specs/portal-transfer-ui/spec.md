## ADDED Requirements

### Requirement: Portal Transfer List and Detail Use v2 API
The portal frontend SHALL fetch transfer list and detail data from v2 API endpoints (`/api/v2/portal/...`) instead of v1. TypeScript types SHALL match the v2 response structure with nested `beneficiary_info`, `payout_instrument`, and `BeneficiaryAddress` objects.

#### Scenario: Transfer list fetches from v2
- **WHEN** the transfer list page loads
- **THEN** the API call uses the v2 base URL for fetching transfers

#### Scenario: Transfer detail fetches from v2
- **WHEN** a user navigates to a transfer detail page
- **THEN** the API call uses the v2 base URL for fetching the single transfer

### Requirement: Transfer Detail Displays Complete Beneficiary Info
The transfer detail page SHALL display all user-submitted beneficiary fields from the v2 response: name, date of birth, email, phone number, ID document details, and full address. Fields not present in the response (null-filtered by backend) SHALL be gracefully hidden.

#### Scenario: Detail shows beneficiary email and phone
- **WHEN** a transfer has `beneficiary_info.beneficiary_email` and `beneficiary_info.beneficiary_phone_number`
- **THEN** the detail page displays these fields in the beneficiary section

#### Scenario: Detail hides absent fields
- **WHEN** a transfer's `beneficiary_info` does not include certain fields (e.g., no dob)
- **THEN** those fields are not rendered in the UI

### Requirement: OFF_RAMP Payout Instrument Rendered Dynamically
For OFF_RAMP transfers, the payout instrument section SHALL render all key-value pairs from `destination.payout_instrument` dynamically, since fields vary by country and payment method. Keys SHALL be formatted from snake_case to human-readable labels.

#### Scenario: Brazil PIX transfer shows br_cpf
- **WHEN** an OFF_RAMP transfer has `payout_instrument.br_cpf`
- **THEN** the detail page displays "Br Cpf" (or similar formatted label) with its value

#### Scenario: Standard bank transfer shows account details
- **WHEN** an OFF_RAMP transfer has `payout_instrument.account_number` and `payout_instrument.routing_number`
- **THEN** the detail page displays these fields with monospace formatting

### Requirement: Amount Flow Shows Harbor Fee
The amount flow section in transfer detail SHALL display harbor_fee as a separate line item alongside commission_fee in the fee breakdown, providing clear visibility into the fee structure.

#### Scenario: Both fees displayed
- **WHEN** a transfer has both `receipt.commission_fee` and `receipt.harbor_fee`
- **THEN** the amount flow shows both as separate line items in the fee breakdown

#### Scenario: Zero harbor fee
- **WHEN** a transfer has `receipt.harbor_fee` of "0" or "0.00"
- **THEN** the harbor fee line is still displayed (showing 0) for transparency
