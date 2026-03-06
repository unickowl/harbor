## ADDED Requirements

### Requirement: Portal v2 Transfer List Endpoint
The system SHALL provide a paginated GET endpoint at `api/v2/portal/applications/{application:uuid}/transfers` that returns all transfers for the application using the v2 response structure. The endpoint MUST require portal authentication and application view authorization.

#### Scenario: Authenticated user lists transfers
- **WHEN** an authenticated portal user sends GET to `/api/v2/portal/applications/{app}/transfers`
- **THEN** the system returns a paginated list of transfers with v2 response structure (including beneficiary_info, payout_instrument, fees)

#### Scenario: Unauthenticated request is rejected
- **WHEN** an unauthenticated request is sent to the transfer list endpoint
- **THEN** the system returns 401 Unauthorized

### Requirement: Portal v2 Transfer Detail Endpoint
The system SHALL provide a GET endpoint at `api/v2/portal/applications/{application:uuid}/transfers/{transfer:uuid}` that returns a single transfer with all requirements fields the user filled during create transfer v2. The endpoint MUST require portal authentication, application view authorization, and transfer view authorization.

#### Scenario: Authenticated user views transfer detail
- **WHEN** an authenticated portal user sends GET to `/api/v2/portal/applications/{app}/transfers/{transfer}`
- **THEN** the system returns the transfer with complete v2 response structure

#### Scenario: Unauthenticated request is rejected
- **WHEN** an unauthenticated request is sent to the transfer detail endpoint
- **THEN** the system returns 401 Unauthorized

### Requirement: v2 Transfer Response Includes Beneficiary Info
The v2 transfer response SHALL include a `destination.beneficiary_info` nested object containing all user-submitted beneficiary requirements fields: `beneficiary_name`, `beneficiary_dob`, `beneficiary_email`, `beneficiary_phone_number`, `beneficiary_id_doc_type`, `beneficiary_id_doc_country_code`, `beneficiary_id_doc_number`, and a nested `beneficiary_address` object (`street`, `street_2`, `city`, `state_province`, `postal_code`, `country`). Fields with null values SHALL be omitted from the response.

#### Scenario: Transfer with complete beneficiary info
- **WHEN** a transfer has all beneficiary fields populated
- **THEN** the response includes all fields under `destination.beneficiary_info`

#### Scenario: Transfer with partial beneficiary info
- **WHEN** a transfer has some beneficiary fields as null (e.g., no dob, no phone)
- **THEN** the response omits those null fields from `destination.beneficiary_info`

### Requirement: v2 OFF_RAMP Payout Instrument from JSON Column
For OFF_RAMP transfers, the v2 response `destination.payout_instrument` SHALL be populated by merging `destination_local_payout_instrument_json` (which contains country-specific fields like `br_cpf`, `hk_fps_id`, bank details, etc.). This follows the same approach as Application V2 TransferResource.

#### Scenario: OFF_RAMP transfer returns full payout instrument
- **WHEN** an OFF_RAMP transfer has `destination_local_payout_instrument_json` with country-specific fields
- **THEN** the response `destination.payout_instrument` contains all those fields

### Requirement: AML Travel Rule Data Excluded
The v2 transfer response SHALL NOT include `aml_sender_travel_rule_data_json`. This is internal AML compliance data populated by the system, not user-submitted requirements data.

#### Scenario: AML data not exposed
- **WHEN** a transfer has `aml_sender_travel_rule_data_json` populated
- **THEN** the field is not present in the v2 response
