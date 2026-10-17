# Testing & Verification Reference

The `obelaw/ium-accounting` test suite is powered by **PestPHP** and **Orchestra Testbench**, executing against an in-memory SQLite database with full migration verification and transactional rollbacks.

---

## 1. Running the Test Suite

From the package directory (`packages/accounting`):

```powershell
# Run the complete test suite
./vendor/bin/pest

# Run with filter for specific feature
./vendor/bin/pest --filter="ZeroSum"

# Run only Unit tests
./vendor/bin/pest tests/Unit

# Run only Feature tests
./vendor/bin/pest tests/Feature

# Run currency tests
./vendor/bin/pest tests/Feature/CurrencyTest.php
```

---

## 2. Test Environment Architecture

### Base Test Case (`tests/TestCase.php`)
Every Pest test extends `Obelaw\Ium\Accounting\Tests\TestCase`, which configures Orchestra Testbench:
- Sets up an in-memory SQLite database (`:memory:`).
- Registers `ObelawAccountingServiceProvider` and central `Obelaw\Ium\Providers\IumServiceProvider`.
- Runs all 11 domain migrations via `$this->loadMigrationsFrom(__DIR__ . '/../database/migrations')`.
- Seeds a default base currency (`USD`) with `exchange_rate = 1.0`.
- Configures environment variables:
  ```php
  $app['config']->set('database.default', 'testing');
  $app['config']->set('database.connections.testing', [
      'driver' => 'sqlite',
      'database' => ':memory:',
      'prefix' => '',
  ]);
  $app['config']->set('ium.accounting.maker_checker.strict', true);
  $app['config']->set('ium.accounting.currency.functional', 'USD');
  ```

---

## 3. Writing Pest Feature Tests

### A. Testing the Zero-Sum Invariant
```php
use Obelaw\Ium\Accounting\Data\CreateJournalEntryDto;
use Obelaw\Ium\Accounting\Data\JournalEntryLineDto;
use Obelaw\Ium\Accounting\Enums\JournalEntryType;
use Obelaw\Ium\Accounting\Enums\PostingSide;
use Obelaw\Ium\Accounting\Exceptions\UnbalancedJournalEntryException;

it('rejects unbalanced journal vouchers during submission', function () {
    $drAccount = ium()->accounting()->accounts()->findByCodeOrFail('1111');
    $crAccount = ium()->accounting()->accounts()->findByCodeOrFail('4100');

    $dto = new CreateJournalEntryDto(
        entryType: JournalEntryType::STANDARD,
        entryDate: '2026-02-01',
        currency: 'USD',
        lines: [
            new JournalEntryLineDto(accountId: $drAccount->id, type: PostingSide::DEBIT, amount: 10000), // 100.00 USD in cents
            new JournalEntryLineDto(accountId: $crAccount->id, type: PostingSide::CREDIT, amount: 8000),  // 80.00 USD in cents
        ],
        narration: 'Unbalanced purchase voucher',
        creatorId: 'maker-user'
    );

    $entry = ium()->accounting()->journalEntries()->createDraft($dto);

    expect(fn () => ium()->accounting()->journalEntries()->submit($entry->id, 'maker-user'))
        ->toThrow(UnbalancedJournalEntryException::class);
});
```

### B. Testing Immutability & Storno Reversals
```php
use Obelaw\Ium\Accounting\Enums\JournalEntryStatus;
use Obelaw\Ium\Accounting\Exceptions\ImmutableJournalEntryException;

it('prohibits direct mutation of posted vouchers and enforces Storno reversal', function () {
    $postedVoucher = createAndPostTestVoucher(amount: 150000); // 1,500.00 USD in cents

    // Direct mutation throws exception
    expect(fn () => $postedVoucher->update(['narration' => 'Unauthorized alteration']))
        ->toThrow(ImmutableJournalEntryException::class);

    // Storno reversal generates offsetting posted voucher
    $reversal = ium()->accounting()->journalEntries()->reverse(
        entryId: $postedVoucher->id,
        actorId: 'auditor-user',
        reason: 'Customer requested refund'
    );

    expect($postedVoucher->fresh()->status)->toBe(JournalEntryStatus::REVERSED)
        ->and($reversal->status)->toBe(JournalEntryStatus::POSTED)
        ->and($reversal->reversed_entry_id)->toBe($postedVoucher->id);
});
```

### C. Testing Maker-Checker Separation of Duties
```php
use Obelaw\Ium\Accounting\Exceptions\MakerCheckerViolationException;

it('prevents voucher creators from approving their own vouchers', function () {
    $draft = createTestDraftVoucher(creatorId: 'alice');
    $submitted = ium()->accounting()->journalEntries()->submit($draft->id, 'alice');

    // Self-approval throws MakerCheckerViolationException
    expect(fn () => ium()->accounting()->journalEntries()->approve($submitted->id, 'alice'))
        ->toThrow(MakerCheckerViolationException::class);

    // Distinct supervisor approval succeeds
    $approved = ium()->accounting()->journalEntries()->approve($submitted->id, 'bob-supervisor');
    expect($approved->approver_id)->toBe('bob-supervisor');
});
```

### D. Testing Multi-Currency Conversions and Invariants
```php
use Obelaw\Ium\Accounting\Data\CreateCurrencyDto;
use Obelaw\Ium\Accounting\Exceptions\BaseCurrencyRateImmutableException;

it('verifies base currency rate immutability and conversion precision', function () {
    $base = ium()->accounting()->currencies()->getBaseCurrency();

    // Setting rate on base currency is rejected
    expect(fn () => ium()->accounting()->currencies()->setRate($base->code, 1.25))
        ->toThrow(BaseCurrencyRateImmutableException::class);

    // Create EUR and convert
    ium()->accounting()->currencies()->create(new CreateCurrencyDto(
        code: 'EUR',
        name: 'Euro',
        symbol: '€',
        exchangeRate: 1.085000,
    ));

    $converted = ium()->accounting()->currencies()->convert(100.00, 'EUR', 'USD');
    expect($converted)->toBe(108.50);

    // Minor unit direct conversion
    $convertedMinor = ium()->accounting()->currencies()->convertMinor(10000, 'EUR', 'USD');
    expect($convertedMinor)->toBe(10850);
});
```

### E. Testing Minor Unit Amounts and Odd Cent Distribution
```php
use Obelaw\Ium\Accounting\Data\CreateJournalEntryDto;
use Obelaw\Ium\Accounting\Data\JournalEntryLineDto;
use Obelaw\Ium\Accounting\Enums\PostingSide;
use Obelaw\Ium\Accounting\Exceptions\UnbalancedJournalEntryException;

it('enforces exact zero-sum down to single minor unit (1 cent)', function () {
    $entry = ium()->accounting()->journalEntries()->createDraft(CreateJournalEntryDto::from([
        'entry_date' => '2026-03-01',
        'lines' => [
            ['account_id' => 1, 'type' => PostingSide::DEBIT, 'amount' => 10000],
            ['account_id' => 2, 'type' => PostingSide::CREDIT, 'amount' => 9999], // 1 cent off
        ],
    ]));

    expect(fn() => ium()->accounting()->journalEntries()->submit($entry->id, 'user-1'))
        ->toThrow(UnbalancedJournalEntryException::class);
});

it('handles multi-leg split distributions with odd cents perfectly', function () {
    // 10000 cents debited; credits split 3333 + 3333 + 3334 = 10000
    $entry = ium()->accounting()->journalEntries()->createDraft(CreateJournalEntryDto::from([
        'entry_date' => '2026-03-01',
        'lines' => [
            ['account_id' => 1, 'type' => PostingSide::DEBIT, 'amount' => 10000],
            ['account_id' => 2, 'type' => PostingSide::CREDIT, 'amount' => 3333],
            ['account_id' => 2, 'type' => PostingSide::CREDIT, 'amount' => 3333],
            ['account_id' => 3, 'type' => PostingSide::CREDIT, 'amount' => 3334],
        ],
    ]));

    $posted = ium()->accounting()->journalEntries()->post($entry->id, 'user-1');
    expect($posted->isPosted())->toBeTrue();
});
```

### F. Testing the Demo Seeder Artisan Command
```php
use Illuminate\Support\Facades\Artisan;
use Obelaw\Ium\Accounting\Models\JournalEntry;
use Obelaw\Ium\Accounting\Services\LedgerService;

it('executes ium:accounting:demo-seed artisan command and establishes balanced ledger', function () {
    $exitCode = Artisan::call('ium:accounting:demo-seed', ['--force' => true]);
    expect($exitCode)->toBe(0);

    $tb = app(LedgerService::class)->generateTrialBalance();
    expect($tb['is_balanced'])->toBeTrue()
        ->and($tb['imbalance'])->toBe(0);

    $bs = app(LedgerService::class)->generateBalanceSheet();
    expect($bs['is_balanced'])->toBeTrue();
});
```

---

## 4. Verification Standards Checklist

Before submitting PRs or delivering code changes for `ium-accounting`:

1. **Test Suite Green**: Run `./vendor/bin/pest`. All 40 feature and unit tests must pass (0 failures, 0 errors, 233 assertions).
2. **Invariants Preserved**:
   - Zero-Sum $\sum \text{Debits} = \sum \text{Credits}$ holds with exact 0 integer minor unit imbalance.
   - General Ledger postings write strictly to leaf accounts.
   - Posted vouchers are strictly immutable.
   - Base functional currency rate is locked at `1.0`.
   - Money VO and models accurately support minor unit conversions.
3. **Audit Trail Verified**: Every state transition emits an audit entry in `ium_accounting_audit_logs`.
4. **Domain Events Dispatched**: Events like `JournalEntryPosted`, `JournalEntryReversed`, `ExchangeRateUpdated`, `BaseCurrencyChanged` are verified via `Event::assertDispatched()`.
5. **No Cross-Domain Leaks**: No direct foreign keys or table queries to external domain tables.
