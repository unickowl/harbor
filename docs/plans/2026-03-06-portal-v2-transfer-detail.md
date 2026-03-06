# Portal v2 Transfer Detail/List Endpoints

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add Portal v2 GET endpoints for transfer list and detail that return all requirements fields users filled during create transfer v2.

**Architecture:** Extend Portal v1 `TransferResource` with a new `Portal/v2/TransferResource` that restructures the `destination` section to mirror the create transfer v2 schema structure (`beneficiary_info`, `payout_instrument`, `beneficiary_address`). Reuse v1's base fields (uuid, status, source, instructions, receipt, etc.) and add the missing fields. Routes delegate to a new `Portal/v2/TransferController` with `list` and `show` methods.

**Tech Stack:** Laravel 12, PHP 8.2+, PHPUnit

---

## Sensitive Field Analysis

Before proceeding — fields that are stored but may be sensitive:

| Field | Stored Column | Decision |
|---|---|---|
| `destination_account_number` | dedicated column | **Include** — Portal is the merchant's own dashboard, they submitted this data |
| `destination_id_doc_number` | dedicated column | **Include** — same reasoning, merchant needs to review what was submitted |
| `aml_sender_travel_rule_data_json` | JSON column | **Exclude** — this is internal AML compliance data populated by the system, not user-submitted |

All other fields are user-submitted requirements data and should be included.

---

## Response Structure Comparison

### v1 destination (current Portal)
```json
{
  "destination": {
    "asset": "USD",
    "account_number": "...",
    "bank_name": "...",
    "is_self_transfer": true,
    "beneficiary_name": "John",
    "amount": "100",
    "transfer_purpose": "SALARY"
  }
}
```

### v2 destination (target — mirrors create schema)
```json
{
  "destination": {
    "asset": "USD",
    "amount": "100",
    "is_self_transfer": true,
    "transfer_purpose": "SALARY",
    "beneficiary_info": {
      "beneficiary_name": "John",
      "beneficiary_dob": "1990-01-01",
      "beneficiary_email": "john@example.com",
      "beneficiary_phone_number": "+1234567890",
      "beneficiary_id_doc_type": "ID_CARD",
      "beneficiary_id_doc_country_code": "US",
      "beneficiary_id_doc_number": "123456789",
      "beneficiary_address": {
        "street": "123 Main St",
        "city": "New York",
        "state_province": "NY",
        "postal_code": "10001",
        "country": "US"
      }
    },
    "payout_instrument": { ... },
    "beneficiary_receiving_wallet_type": "personalWallet",
    "beneficiary_institution_name": "MetaMask"
  }
}
```

The `payout_instrument` section differs per transfer type:
- **OFF_RAMP**: Flat merge of `destination_local_payout_instrument_json` (contains bank details, br_cpf, hk_fps_id, etc.) — same approach as Application V2
- **ON_RAMP / SWAP**: `{ "chain": "...", "address": "...", "address_memo": "..." }`

---

## Task List

### Task 1: Create Portal v2 TransferResource

**Files:**
- Create: `api/app/Http/Resources/Portal/v2/TransferResource.php`

**Step 1: Create the Resource file**

```php
<?php

namespace App\Http\Resources\Portal\v2;

use App\Enums\TransferSettlementStrategyEnum;
use App\Enums\TransferStatusEnum;
use App\Enums\TransferTypeEnum;
use App\Http\Resources\Application\V2\QuoteFeeResource;
use App\Http\Resources\Application\V2\TransferResource as ApplicationV2TransferResource;
use App\Models\Application;
use Illuminate\Http\Request;

class TransferResource extends ApplicationV2TransferResource
{
    public function toArray (Request $request): array
    {
        $base = parent::toArray($request);

        // Add beneficiary_info section with all requirements fields
        $base['destination']['beneficiary_info'] = $this->getBeneficiaryInfo();

        // Add residential/address fields for OFF_RAMP (v1 had these commented out)
        if ($this->type == TransferTypeEnum::OFF_RAMP) {
            $base['destination']['destination_type'] = $this->destination_type;
        }

        return $base;
    }

    protected function getBeneficiaryInfo (): array
    {
        return array_filter([
            'beneficiary_name' => $this->destination_beneficiary_name,
            'beneficiary_dob' => $this->destination_dob,
            'beneficiary_email' => $this->destination_email,
            'beneficiary_phone_number' => $this->destination_phone_number,
            'beneficiary_id_doc_type' => $this->destination_id_doc_type,
            'beneficiary_id_doc_country_code' => $this->destination_id_doc_country_code,
            'beneficiary_id_doc_number' => $this->destination_id_doc_number,
            'beneficiary_address' => array_filter([
                'street' => $this->destination_residential_address_1,
                'street_2' => $this->destination_residential_address_2,
                'city' => $this->destination_residential_city,
                'state_province' => $this->destination_residential_state,
                'postal_code' => $this->destination_residential_postal_code,
                'country' => $this->destination_residential_country_code,
            ], fn ($v) => $v !== null),
        ], fn ($v) => $v !== null && $v !== []);
    }
}
```

**Step 2: Verify file compiles**

Run: `cd api && php artisan tinker --execute="new \App\Http\Resources\Portal\v2\TransferResource(new \App\Models\Transfer());"`
Expected: No errors

**Step 3: Commit**

```bash
git add api/app/Http/Resources/Portal/v2/TransferResource.php
git commit -m "feat: add Portal v2 TransferResource with complete requirements fields"
```

---

### Task 2: Create Portal v2 TransferController

**Files:**
- Create: `api/app/Http/Controllers/Portal/v2/TransferController.php` (add methods to existing file)

Note: `Portal/v2/TransferController.php` already exists with quote/create methods. We add `list` and `show` to it.

**Step 1: Add list and show methods**

Add these methods to the existing `Portal/v2/TransferController`:

```php
// Add to existing imports at top:
use App\Http\Resources\Portal\v2\TransferResource;
use App\Http\Requests\Portal\TransferListRequest;
use App\Models\Transfer;

// Add these methods to the class:

public function list (TransferListRequest $request, Application $application)
{
    $transfers = $this->transferQuoteService
        ->getTransfers($request->all(), $application)
        ->paginate($request->get('per_page', 15));

    $transfers->load(['onBehalfOf', 'transferQuote.fees']);

    return TransferResource::collection($transfers);
}

public function show (Application $application, Transfer $transfer)
{
    return TransferResource::make(
        $transfer->load(['application', 'onBehalfOf', 'transferQuote.fees'])
    );
}
```

**Important:** The `list` method needs `TransferService` not `TransferQuoteService`. Check if `TransferService` is already injected or needs to be added. Looking at the existing constructor: `public function __construct(public TransferQuoteService $transferQuoteService) {}` — we need to add `TransferService`.

Updated constructor:

```php
public function __construct(
    public TransferQuoteService $transferQuoteService,
    private TransferService $transferService,
) {}
```

And the list method should use `$this->transferService->getTransfers(...)`.

**Step 2: Verify methods are callable**

Run: `cd api && php artisan route:list --name=portal.v2` (after routes are added in Task 3)

**Step 3: Commit**

```bash
git add api/app/Http/Controllers/Portal/v2/TransferController.php
git commit -m "feat: add list and show methods to Portal v2 TransferController"
```

---

### Task 3: Add Portal v2 Routes

**Files:**
- Modify: `api/routes/portal/v2.php`

**Step 1: Add the two new routes**

Add inside the existing middleware group, after the existing routes:

```php
Route::get('/applications/{application:uuid}/transfers', [TransferController::class, 'list'])
    ->can('view', 'application')
    ->name('applications.transfers');

Route::get('/applications/{application:uuid}/transfers/{transfer:uuid}', [TransferController::class, 'show'])
    ->can('view', 'application')
    ->can('view', ['transfer', 'application'])
    ->name('applications.transfers.show');
```

**Important:** These routes must be placed BEFORE the `transfers/quotes` POST route to avoid conflicts, OR after all `transfers/...` sub-routes. Since route resolution is prefix-based, placing them after the specific sub-routes (quotes, DepositSettings, etc.) is correct — Laravel matches specific routes first. Place them at the end of the group.

**Step 2: Verify routes are registered**

Run: `cd api && php artisan route:list --name=portal.v2.applications.transfers`
Expected: Should show the two new GET routes plus existing routes.

**Step 3: Commit**

```bash
git add api/routes/portal/v2.php
git commit -m "feat: register Portal v2 transfer list and detail routes"
```

---

### Task 4: Update TransferFactory for OFF_RAMP test data

**Files:**
- Modify: `api/database/factories/TransferFactory.php`

**Step 1: Add missing fields to the `offRamp` state**

The existing `offRamp()` state is missing several fields needed for testing the new resource. Add them:

```php
public function offRamp (): static
{
    return $this->state([
        'type' => TransferTypeEnum::OFF_RAMP->value,
        'source_asset' => 'USDC',
        'source_amount' => 100,
        'source_chain' => 'stellar',
        'destination_asset' => 'USD',
        'destination_amount' => 98,
        'destination_chain' => null,
        'destination_address' => null,
        'destination_address_memo' => null,
        'destination_customer_crypto_address_id' => null,
        'destination_account_number' => '1234567890',
        'destination_routing_number' => '021000021',
        'destination_bank_name' => 'Chase Bank',
        'destination_bank_address' => '270 Park Ave, New York, NY 10017',
        'destination_account_holder_name' => 'John Smith',
        'destination_beneficiary_name' => $this->faker->name,
        'destination_residential_country_code' => 'US',
        'destination_residential_state' => 'NY',
        'destination_residential_city' => 'New York',
        'destination_residential_address_1' => '123 Main St',
        'destination_residential_address_2' => 'Apt 4B',
        'destination_residential_postal_code' => '10001',
        'destination_dob' => '1990-01-01',
        'destination_phone_number' => '+12125551234',
        'destination_email' => 'john@example.com',
        'destination_id_doc_type' => 'ID_CARD',
        'destination_id_doc_country_code' => 'US',
        'destination_id_doc_number' => '123456789',
        'destination_type' => 'individual',
        'destination_local_payout_instrument_json' => [
            'account_number' => '1234567890',
            'routing_number' => '021000021',
            'bank_name' => 'Chase Bank',
            'account_holder_name' => 'John Smith',
        ],
        'instruction_chain' => 'stellar',
        'instruction_address' => 'GA7ZO7WNKTUFPAAR2YM6FJGK7WNTY5JBFOZHHKBQYT6GUHO3HGXOTDMH',
        'instruction_memo' => 'test-memo',
    ]);
}
```

**Step 2: Commit**

```bash
git add api/database/factories/TransferFactory.php
git commit -m "feat: add complete requirements fields to TransferFactory offRamp state"
```

---

### Task 5: Write Feature Tests

**Files:**
- Create: `api/tests/Feature/Portal/V2/TransferControllerTest.php`

**Step 1: Write the test file**

```php
<?php

namespace Tests\Feature\Portal\V2;

use App\Enums\CustomerStatusEnum;
use App\Models\Application;
use App\Models\Customer;
use App\Models\Transfer;
use App\Models\User;
use Illuminate\Foundation\Testing\LazilyRefreshDatabase;
use Tests\TestCase;

class TransferControllerTest extends TestCase
{
    use LazilyRefreshDatabase;

    private User $user;
    private Application $application;
    private Customer $customer;

    protected function setUp (): void
    {
        parent::setUp();
        $this->user = User::factory()->create();
        $this->application = Application::factory()->create(['email' => $this->user->email]);
        $this->customer = Customer::factory()->create([
            'application_id' => $this->application->id,
            'status' => CustomerStatusEnum::VERIFIED,
        ]);
    }

    public function test_list_requires_authentication (): void
    {
        $response = $this->getJson(route('portal.v2.applications.transfers', [$this->application->uuid]));
        $response->assertStatus(401);
    }

    public function test_list_returns_transfers (): void
    {
        Transfer::factory()->count(2)->create(['application_id' => $this->application->id]);

        $response = $this->actingAs($this->user, 'portal')
            ->getJson(route('portal.v2.applications.transfers', [$this->application->uuid]));

        $response->assertStatus(200)
            ->assertJsonCount(2, 'data');
    }

    public function test_show_requires_authentication (): void
    {
        $transfer = Transfer::factory()->create(['application_id' => $this->application->id]);

        $response = $this->getJson(route('portal.v2.applications.transfers.show', [
            $this->application->uuid,
            $transfer->uuid,
        ]));
        $response->assertStatus(401);
    }

    public function test_show_on_ramp_returns_complete_structure (): void
    {
        $transfer = Transfer::factory()->create([
            'application_id' => $this->application->id,
            'destination_dob' => '1990-01-01',
            'destination_phone_number' => '+12125551234',
            'destination_email' => 'john@example.com',
            'destination_id_doc_type' => 'ID_CARD',
            'destination_id_doc_country_code' => 'US',
            'destination_id_doc_number' => '123456789',
            'destination_residential_country_code' => 'US',
            'destination_residential_state' => 'NY',
            'destination_residential_city' => 'New York',
            'destination_residential_address_1' => '123 Main St',
            'destination_residential_postal_code' => '10001',
        ]);

        $response = $this->actingAs($this->user, 'portal')
            ->getJson(route('portal.v2.applications.transfers.show', [
                $this->application->uuid,
                $transfer->uuid,
            ]));

        $response->assertStatus(200)
            ->assertJsonStructure([
                'data' => [
                    'uuid',
                    'status',
                    'type',
                    'source',
                    'destination' => [
                        'asset',
                        'amount',
                        'payout_instrument',
                        'is_self_transfer',
                        'transfer_purpose',
                        'beneficiary_info' => [
                            'beneficiary_name',
                            'beneficiary_dob',
                            'beneficiary_email',
                            'beneficiary_phone_number',
                            'beneficiary_id_doc_type',
                            'beneficiary_id_doc_country_code',
                            'beneficiary_id_doc_number',
                            'beneficiary_address' => [
                                'street',
                                'city',
                                'state_province',
                                'postal_code',
                                'country',
                            ],
                        ],
                    ],
                    'transfer_instructions',
                    'receipt',
                    'created_at',
                    'updated_at',
                ],
            ]);

        $data = $response->json('data');
        $this->assertEquals('1990-01-01', $data['destination']['beneficiary_info']['beneficiary_dob']);
        $this->assertEquals('+12125551234', $data['destination']['beneficiary_info']['beneficiary_phone_number']);
        $this->assertEquals('john@example.com', $data['destination']['beneficiary_info']['beneficiary_email']);
        $this->assertEquals('123456789', $data['destination']['beneficiary_info']['beneficiary_id_doc_number']);
        $this->assertEquals('US', $data['destination']['beneficiary_info']['beneficiary_address']['country']);
    }

    public function test_show_off_ramp_returns_payout_instrument_json (): void
    {
        $payoutInstrument = [
            'account_number' => '1234567890',
            'routing_number' => '021000021',
            'bank_name' => 'Chase Bank',
            'account_holder_name' => 'John Smith',
        ];

        $transfer = Transfer::factory()->offRamp()->create([
            'application_id' => $this->application->id,
            'destination_local_payout_instrument_json' => $payoutInstrument,
        ]);

        $response = $this->actingAs($this->user, 'portal')
            ->getJson(route('portal.v2.applications.transfers.show', [
                $this->application->uuid,
                $transfer->uuid,
            ]));

        $response->assertStatus(200);

        $data = $response->json('data');

        // payout_instrument should contain the full JSON from DB
        $this->assertEquals('1234567890', $data['destination']['payout_instrument']['account_number']);
        $this->assertEquals('021000021', $data['destination']['payout_instrument']['routing_number']);

        // beneficiary_info should be present
        $this->assertArrayHasKey('beneficiary_info', $data['destination']);
        $this->assertArrayHasKey('beneficiary_name', $data['destination']['beneficiary_info']);
        $this->assertArrayHasKey('beneficiary_address', $data['destination']['beneficiary_info']);

        // destination_type should be present for off-ramp
        $this->assertArrayHasKey('destination_type', $data['destination']);
    }

    public function test_show_omits_null_beneficiary_fields (): void
    {
        // Transfer with minimal data — no dob, no id_doc, no phone
        $transfer = Transfer::factory()->create([
            'application_id' => $this->application->id,
            'destination_dob' => null,
            'destination_phone_number' => null,
            'destination_id_doc_number' => null,
        ]);

        $response = $this->actingAs($this->user, 'portal')
            ->getJson(route('portal.v2.applications.transfers.show', [
                $this->application->uuid,
                $transfer->uuid,
            ]));

        $response->assertStatus(200);

        $beneficiaryInfo = $response->json('data.destination.beneficiary_info');
        $this->assertArrayNotHasKey('beneficiary_dob', $beneficiaryInfo);
        $this->assertArrayNotHasKey('beneficiary_phone_number', $beneficiaryInfo);
        $this->assertArrayNotHasKey('beneficiary_id_doc_number', $beneficiaryInfo);
    }
}
```

**Step 2: Run tests to verify they fail**

Run: `cd api && php artisan test --filter=Tests\\Feature\\Portal\\V2\\TransferControllerTest`
Expected: FAIL — routes not yet registered / resource not yet created (depending on task order)

**Step 3: Commit test file**

```bash
git add api/tests/Feature/Portal/V2/TransferControllerTest.php
git commit -m "test: add Portal v2 transfer detail/list feature tests"
```

---

### Task 6: Run All Tests and Verify

**Step 1: Run the new tests**

Run: `cd api && php artisan test --filter=Tests\\Feature\\Portal\\V2\\TransferControllerTest -v`
Expected: All 6 tests PASS

**Step 2: Run existing Portal transfer tests to ensure no regression**

Run: `cd api && php artisan test --filter=Tests\\Feature\\Portal\\TransferControllerTest -v`
Expected: All existing tests PASS

**Step 3: Run full test suite**

Run: `cd api && php artisan test --parallel`
Expected: No failures

**Step 4: Final commit**

```bash
git add -A
git commit -m "feat: Portal v2 transfer list/detail with complete requirements fields"
```

---

## Summary of New Endpoints

| Method | URI | Description |
|--------|-----|-------------|
| GET | `api/v2/portal/applications/{app}/transfers` | List transfers (paginated) |
| GET | `api/v2/portal/applications/{app}/transfers/{transfer}` | Transfer detail with all requirements fields |

## Key Differences from v1

| Aspect | v1 | v2 |
|--------|----|----|
| `beneficiary_info` section | Flat `beneficiary_name` only | Full nested object with dob, email, phone, id_doc, address |
| `payout_instrument` (OFF_RAMP) | Hardcoded individual columns | Full `destination_local_payout_instrument_json` merge (handles all country-specific fields) |
| `beneficiary_address` | Not returned (commented out in v1) | Full nested object (street, city, state, postal_code, country) |
| `destination_type` (OFF_RAMP) | Not returned (commented out in v1) | Included |
| `fees` | Not returned | Returned (harbor fees via `transferQuote.fees`) |
| Null handling | Returns null values | `array_filter` omits null fields for cleaner responses |
