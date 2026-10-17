# Obelawium Accounting: Developer Recipes & Code Cookbook

Production-ready, copy-pasteable PHP recipes for common development workflows with `obelaw/ium-accounting`.

---

## Recipe 1: Initialize Chart of Accounts (COA) Hierarchy

Set up a structured 4-tier Chart of Accounts tree: Class $\to$ Main Group $\to$ Sub Group $\to$ Posting Leaf.

```php
use Obelaw\Ium\Accounting\Data\CreateAccountDto;
use Obelaw\Ium\Accounting\Enums\AccountClassification;
use Obelaw\Ium\Accounting\Enums\AccountType;
use Obelaw\Ium\Accounting\Enums\NormalBalance;

// 1. Root Category (Non-posting aggregator)
$assetClass = ium()->accounting()->accounts()->create(new CreateAccountDto(
    code: '1000',
    name: 'Assets',
    type: AccountType::ASSET,
    classification: AccountClassification::CLASS_ROOT,
    normalBalance: NormalBalance::DEBIT,
    isLeaf: false,
));

// 2. Main Group
$currentAssets = ium()->accounting()->accounts()->create(new CreateAccountDto(
    code: '1100',
    name: 'Current Assets',
    type: AccountType::ASSET,
    classification: AccountClassification::MAIN_GROUP,
    normalBalance: NormalBalance::DEBIT,
    parentId: $assetClass->id,
    isLeaf: false,
));

// 3. Sub Group
$cashEquivalents = ium()->accounting()->accounts()->create(new CreateAccountDto(
    code: '1110',
    name: 'Cash & Cash Equivalents',
    type: AccountType::ASSET,
    classification: AccountClassification::SUB_GROUP,
    normalBalance: NormalBalance::DEBIT,
    parentId: $currentAssets->id,
    isLeaf: false,
));

// 4. Posting Leaf (Can receive journal lines)
$bankAccount = ium()->accounting()->accounts()->create(new CreateAccountDto(
    code: '1111',
    name: 'Chase Operating Account (USD)',
    type: AccountType::ASSET,
    classification: AccountClassification::POSTING,
    normalBalance: NormalBalance::DEBIT,
    parentId: $cashEquivalents->id,
    currency: 'USD',
    isLeaf: true,
    isReconciliation: true,
));
```

---

## Recipe 2: Setup Fiscal Year and Monthly Periods

```php
use Obelaw\Ium\Accounting\Data\CreateFiscalPeriodDto;
use Obelaw\Ium\Accounting\Data\CreateFiscalYearDto;
use Obelaw\Ium\Accounting\Enums\FiscalPeriodStatus;

// 1. Create Fiscal Year 2026
$year2026 = ium()->accounting()->fiscalPeriods()->createYear(new CreateFiscalYearDto(
    name: 'FY 2026',
    startDate: '2026-01-01',
    endDate: '2026-12-31',
    status: FiscalPeriodStatus::OPEN,
    actorId: 'cfo-user-id',
));

// 2. Generate 12 Monthly Periods
$months = [
    1 => ['Jan 2026', '2026-01-01', '2026-01-31'],
    2 => ['Feb 2026', '2026-02-01', '2026-02-28'],
    3 => ['Mar 2026', '2026-03-01', '2026-03-31'],
    4 => ['Apr 2026', '2026-04-01', '2026-04-30'],
    5 => ['May 2026', '2026-05-01', '2026-05-31'],
    6 => ['Jun 2026', '2026-06-01', '2026-06-30'],
    7 => ['Jul 2026', '2026-07-01', '2026-07-31'],
    8 => ['Aug 2026', '2026-08-01', '2026-08-31'],
    9 => ['Sep 2026', '2026-09-01', '2026-09-30'],
    10 => ['Oct 2026', '2026-10-01', '2026-10-31'],
    11 => ['Nov 2026', '2026-11-01', '2026-11-30'],
    12 => ['Dec 2026', '2026-12-01', '2026-12-31'],
];

foreach ($months as $num => [$name, $start, $end]) {
    ium()->accounting()->fiscalPeriods()->createPeriod(new CreateFiscalPeriodDto(
        fiscalYearId: $year2026->id,
        periodNumber: $num,
        name: $name,
        startDate: $start,
        endDate: $end,
        status: FiscalPeriodStatus::OPEN,
        actorId: 'cfo-user-id',
    ));
}
```

---

## Recipe 3: Create, Submit, Approve, and Post a Journal Entry

Demonstrates full Maker-Checker segregation and zero-sum validation.

```php
use Obelaw\Ium\Accounting\Data\CreateJournalEntryDto;
use Obelaw\Ium\Accounting\Data\JournalEntryLineDto;
use Obelaw\Ium\Accounting\Enums\JournalEntryType;
use Obelaw\Ium\Accounting\Enums\PostingSide;

$cashAccount = ium()->accounting()->accounts()->findByCodeOrFail('1111');
$revenueAccount = ium()->accounting()->accounts()->findByCodeOrFail('4100');

// 1. Maker creates Draft Voucher ($5,000 cash sale = 500,000 minor units/cents)
$draftVoucher = ium()->accounting()->journalEntries()->createDraft(new CreateJournalEntryDto(
    entryType: JournalEntryType::STANDARD,
    entryDate: '2026-01-15',
    currency: 'USD',
    lines: [
        new JournalEntryLineDto(
            accountId: $cashAccount->id,
            type: PostingSide::DEBIT,
            amount: 500000, // $5,000.00 in cents
            narration: 'Cash received for software licensing',
        ),
        new JournalEntryLineDto(
            accountId: $revenueAccount->id,
            type: PostingSide::CREDIT,
            amount: 500000, // $5,000.00 in cents
            narration: 'SaaS License Revenue',
        ),
    ],
    narration: 'Direct software subscription payment',
    creatorId: 'maker-accountant-01',
));

// 2. Maker submits for approval (Enforces Zero-Sum invariant)
$submittedVoucher = ium()->accounting()->journalEntries()->submit(
    entryId: $draftVoucher->id,
    actorId: 'maker-accountant-01'
);

// 3. Checker approves (Maker-Checker strictly forbids 'maker-accountant-01' from approving)
$approvedVoucher = ium()->accounting()->journalEntries()->approve(
    entryId: $submittedVoucher->id,
    approverId: 'supervisor-auditor-02'
);

// 4. Post to the General Ledger (Commits to GL postings and updates running account balances)
$postedVoucher = ium()->accounting()->journalEntries()->post(
    entryId: $approvedVoucher->id,
    actorId: 'supervisor-auditor-02'
);
```

---

## Recipe 4: Storno Reversal of a Posted Voucher

Posted entries are **never deleted or modified**. Fix errors using an opposite reversal voucher:

```php
// Reverses all debits to credits and credits to debits
$reversalVoucher = ium()->accounting()->journalEntries()->reverse(
    entryId: $postedVoucher->id,
    actorId: 'senior-controller-id',
    reason: 'Customer cancelled transaction within 24h refund window',
    reversalDate: '2026-01-16' // Optional; defaults to original entry date
);

// $original is now status 'reversed' (with reversal_entry_id set)
// $reversalVoucher is a newly posted 'reversal' voucher netting out both account balances
```

---

## Recipe 5: Ingesting Cross-Domain Business Events via ACL

Used by external domains (e.g., Billing, Sales, CRM, Inventory) to send accounting transactions across the Anti-Corruption Layer.

```php
use Obelaw\Ium\Accounting\Data\AclIngestEventDto;

// Option A: Ingest using integer minor units (cents - e.g. $12,500.00 = 1250000 cents)
$postedEntry = ium()->accounting()->acl()->ingest(AclIngestEventDto::from([
    'event_name' => 'billing.invoice.issued',
    'document_reference' => 'INV-2026-0042',
    'transaction_date' => '2026-01-20',
    'amount' => 1250000,            // $12,500.00 in minor units (cents)
    'currency' => 'USD',
    'debit_account_code' => '1200',  // Accounts Receivable
    'credit_account_code' => '4100', // SaaS Revenue
    'cost_center_id' => 3,           // Enterprise Sales Cost Center
    'source_type' => 'Obelaw\Ium\Billing\Models\Invoice',
    'source_id' => 42,
    'actor_id' => 'billing-worker',
    'auto_post' => true,             // Immediately validate and post to GL
    'metadata' => [
        'customer_id' => 991,
        'tax_rate' => 0.0,
    ],
]));

// Option B: Ingest from decimal major units using fromMajor
$postedFromMajor = ium()->accounting()->acl()->ingest(AclIngestEventDto::fromMajor(
    eventName: 'billing.invoice.issued',
    documentReference: 'INV-2026-0043',
    amountMajor: 12500.00,
    debitAccountCode: '1200',
    creditAccountCode: '4100',
));
```

---

## Recipe 6: Registering Reusable Event-to-Account Mappings

Allow external modules to emit events without needing to know specific GL account codes:

```php
use Obelaw\Ium\Accounting\Data\AclIngestEventDto;

// 1. Accounting admin registers mapping
ium()->accounting()->acl()->registerMapping(
    eventName: 'wms.inventory.scrapped',
    debitAccountCode: '5200',  // Inventory Shrinkage Expense
    creditAccountCode: '1300', // Inventory Asset
    descriptionTemplate: 'Obsolete warehouse inventory write-off'
);

// 2. WMS module simply emits the event with document ref and integer minor units (cents)
$entry = ium()->accounting()->acl()->ingest(new AclIngestEventDto(
    eventName: 'wms.inventory.scrapped',
    documentReference: 'SCRAP-8821',
    transactionDate: '2026-01-22',
    amount: 45000, // $450.00 in cents
    autoPost: true
));
```

---

## Recipe 7: Generating Financial Reports

### Trial Balance (Debit vs Credit parity check)
```php
$trialBalance = ium()->accounting()->ledger()->generateTrialBalance($fiscalPeriodId = null);

if (!$trialBalance['is_balanced']) {
    throw new RuntimeException("Ledger imbalance detected: {$trialBalance['imbalance']}");
}

// Display integer cents and decimal major units
echo "Total Debit: {$trialBalance['total_debit']} cents (${$trialBalance['total_debit_major']})\n";
echo "Total Credit: {$trialBalance['total_credit']} cents (${$trialBalance['total_credit_major']})\n";

// Access rows
foreach ($trialBalance['accounts'] as $row) {
    echo "{$row['account_code']} | {$row['account_name']}: Dr {$row['debit_balance_major']} / Cr {$row['credit_balance_major']}\n";
}
```

### Balance Sheet (Assets = Liabilities + Equity)
```php
$balanceSheet = ium()->accounting()->ledger()->generateBalanceSheet();

echo "Total Assets: $" . $balanceSheet['assets']['total_major'] . "\n";
echo "Total Liabilities & Equity: $" . $balanceSheet['total_liabilities_and_equity_major'] . "\n";
echo "Is Balanced: " . ($balanceSheet['is_balanced'] ? 'YES' : 'NO') . "\n";
```

### Income Statement / P&L (Revenue - Expenses = Net Income)
```php
$pnl = ium()->accounting()->ledger()->generateIncomeStatement();

echo "Total Revenue: $" . $pnl['revenues']['total_major'] . "\n";
echo "Total Expense: $" . $pnl['expenses']['total_major'] . "\n";
echo "Net Income: $" . $pnl['net_income_major'] . "\n";
echo "Profitable: " . ($pnl['is_profitable'] ? 'YES' : 'NO') . "\n";
```

---

## Recipe 8: Cost Centers & Analytical Dimensions

Create cost centers and tag them on journal lines:

```php
use Obelaw\Ium\Accounting\Data\CreateCostCenterDto;
use Obelaw\Ium\Accounting\Enums\DimensionType;

// Create Engineering Department cost center
$engineeringCC = ium()->accounting()->costCenters()->create(new CreateCostCenterDto(
    code: 'CC-ENG',
    name: 'Engineering & R&D',
    dimensionType: DimensionType::DEPARTMENT,
));

// Filter cost centers by type
$departments = ium()->accounting()->costCenters()->list(DimensionType::DEPARTMENT);
```

---

## Recipe 9: Fiscal Year-End Closing Workflow

Zeroes out nominal accounts (Revenue & Expense) and rolls net profit/loss into Retained Earnings:

```php
use Obelaw\Ium\Accounting\Data\FiscalYearCloseDto;

$closedYear = ium()->accounting()->closing()->closeFiscalYear(new FiscalYearCloseDto(
    fiscalYearId: $year2026->id,
    actorId: 'cfo-user-id',
    retainedEarningsAccountCode: '3100', // Retained Earnings
    incomeSummaryAccountCode: '3900',    // Income Summary Clearing
    closingDate: '2026-12-31',
    notes: 'Formal statutory year-end closing 2026'
));

// Status of $year2026 and all its 12 periods is now LOCKED
```

---

## Recipe 10: Writing Pest Tests for Accounting Modules

```php
use Obelaw\Ium\Accounting\Data\AclIngestEventDto;
use Obelaw\Ium\Accounting\Data\CreateAccountDto;
use Obelaw\Ium\Accounting\Enums\AccountClassification;
use Obelaw\Ium\Accounting\Enums\AccountType;
use Obelaw\Ium\Accounting\Enums\NormalBalance;

it('verifies double entry ledger balances after invoice payment', function () {
    // 1. Seed accounts
    $cash = ium()->accounting()->accounts()->create(new CreateAccountDto(
        code: '1010',
        name: 'Operating Cash',
        type: AccountType::ASSET,
        classification: AccountClassification::POSTING,
        normalBalance: NormalBalance::DEBIT,
    ));

    $ar = ium()->accounting()->accounts()->create(new CreateAccountDto(
        code: '1200',
        name: 'Accounts Receivable',
        type: AccountType::ASSET,
        classification: AccountClassification::POSTING,
        normalBalance: NormalBalance::DEBIT,
    ));

    // 2. Perform transaction via ACL
    ium()->accounting()->acl()->ingest(AclIngestEventDto::from([
        'event_name' => 'billing.payment.collected',
        'document_reference' => 'PAY-001',
        'transaction_date' => date('Y-m-d'),
        'amount' => 1500.00,
        'debit_account_code' => '1010',
        'credit_account_code' => '1200',
        'auto_post' => true,
    ]));

    // 3. Assert balances
    expect($cash->fresh()->current_balance)->toEqual(1500.00)
        ->and($ar->fresh()->current_balance)->toEqual(-1500.00);

    // 4. Assert trial balance parity
    $tb = ium()->accounting()->ledger()->generateTrialBalance();
    expect($tb['is_balanced'])->toBeTrue();
});
```

---

## Recipe 11: Registering ISO Currencies & Base Currency Setup

```php
use Obelaw\Ium\Accounting\Data\CreateCurrencyDto;

// 1. Register Euro with precision and formatting
$eur = ium()->accounting()->currencies()->create(new CreateCurrencyDto(
    code: 'EUR',
    name: 'Euro',
    symbol: '€',
    precision: 2,
    symbolFirst: false,
    decimalMark: ',',
    thousandsSeparator: '.',
    exchangeRate: 1.085000,
    actorId: 'treasury-manager',
));

// 2. Inspect active reporting base currency
$base = ium()->accounting()->currencies()->getBaseCurrency();
echo "Base Functional Currency: {$base->code}\n"; // USD
```

---

## Recipe 12: Managing Exchange Rates & Historical Cutoffs

```php
// Record new exchange rate for Euro
$rateRecord = ium()->accounting()->currencies()->setRate(
    code: 'EUR',
    rate: 1.092000,
    effectiveDate: '2026-03-20',
    source: 'ecb_daily_feed',
    actorId: 'exchange-feed-service'
);

// Resolves latest rate on or before 2026-03-25
$resolvedRate = ium()->accounting()->currencies()->getRate('EUR', 'USD', '2026-03-25');
echo "Effective EUR/USD Rate: {$resolvedRate}\n"; // 1.092000
```

---

## Recipe 13: High-Precision Currency Conversion

```php
// 1. Convert major decimal amounts (500.00 EUR to USD)
$usdAmount = ium()->accounting()->currencies()->convert(
    amount: 500.00,
    fromCurrency: 'EUR',
    toCurrency: 'USD',
    effectiveDate: '2026-03-20'
);
// $usdAmount = 546.00 (500 * 1.092000)

// 2. Convert integer minor units directly (50,000 cents EUR -> USD cents)
$usdCents = ium()->accounting()->currencies()->convertMinor(
    minorAmount: 50000,
    from: 'EUR',
    to: 'USD',
    date: '2026-03-20'
);
// $usdCents = 54600

// 3. Detailed conversion metadata with effective date and rate
$conversion = ium()->accounting()->currencies()->convertDetailed(
    amount: 500.00,
    fromCurrency: 'EUR',
    toCurrency: 'USD',
    effectiveDate: '2026-03-20'
);

echo "Converted: {$conversion->convertedAmount} at rate {$conversion->rate}\n";
```

---

## Recipe 14: Drafting & Posting Multi-Currency Journal Vouchers

```php
use Obelaw\Ium\Accounting\Data\CreateJournalEntryDto;
use Obelaw\Ium\Accounting\Data\JournalEntryLineDto;
use Obelaw\Ium\Accounting\Enums\JournalEntryType;
use Obelaw\Ium\Accounting\Enums\PostingSide;

$eurBank = ium()->accounting()->accounts()->findByCodeOrFail('1112'); // EUR bank leaf
$arEur   = ium()->accounting()->accounts()->findByCodeOrFail('1201'); // EUR AR leaf

// Voucher entered in EUR at 1.085000 exchange rate (1,000.00 EUR = 100,000 cents)
$draft = ium()->accounting()->journalEntries()->createDraft(new CreateJournalEntryDto(
    entryType: JournalEntryType::STANDARD,
    entryDate: '2026-03-15',
    currency: 'EUR',
    exchangeRate: 1.085000,
    creatorId: 'finance-clerk-1',
    lines: [
        new JournalEntryLineDto(
            accountId: $eurBank->id,
            type: PostingSide::DEBIT,
            amount: 100000, // 1,000 EUR in cents -> 108,500 functional USD cents
        ),
        new JournalEntryLineDto(
            accountId: $arEur->id,
            type: PostingSide::CREDIT,
            amount: 100000, // 1,000 EUR in cents -> 108,500 functional USD cents
        ),
    ],
    narration: 'Settlement of European customer invoice in EUR'
));

$submitted = ium()->accounting()->journalEntries()->submit($draft->id, 'finance-clerk-1');
$approved  = ium()->accounting()->journalEntries()->approve($submitted->id, 'finance-supervisor-2');
$posted    = ium()->accounting()->journalEntries()->post($approved->id, 'finance-supervisor-2');

// General ledger stores 108500 ($1,085.00) in debit and credit (amount_functional)
```

---

## Recipe 15: Implementing a Custom Exchange Rate Provider

Implement `ExchangeRateProviderInterface` to integrate external FX providers:

```php
namespace App\Services;

use Obelaw\Ium\Accounting\Contracts\ExchangeRateProviderInterface;
use Illuminate\Support\Facades\Http;

class OpenExchangeRatesProvider implements ExchangeRateProviderInterface
{
    public function getRate(string $currencyCode, string $baseCurrencyCode, ?string $date = null): ?float
    {
        $rates = $this->getRates($baseCurrencyCode, $date);

        return $rates[$currencyCode] ?? null;
    }

    public function getRates(string $baseCurrencyCode, ?string $date = null): array
    {
        $endpoint = $date ? "historical/{$date}.json" : 'latest.json';
        $response = Http::get("https://openexchangerates.org/api/{$endpoint}", [
            'app_id' => config('services.oxr.app_id'),
            'base' => $baseCurrencyCode,
        ]);

        return $response->json('rates') ?? [];
    }
}

// Register with CurrencyService:
ium()->accounting()->currencies()->setRateProvider(new OpenExchangeRatesProvider());
```

---

## Recipe 16: Foreign Exchange (FX) Realized Gain/Loss Adjustments

When an invoice is issued at rate $R_1$ and settled at rate $R_2$, recognize the FX gain/loss:

```php
// Invoice issued at 1 EUR = 1.05 USD ($1,050 USD = 105,000 cents)
// Payment collected at 1 EUR = 1.10 USD ($1,100 USD = 110,000 cents) -> $50 Realized FX Gain (5,000 cents)

$bankAccount = ium()->accounting()->accounts()->findByCodeOrFail('1111'); // Cash USD
$arAccount   = ium()->accounting()->accounts()->findByCodeOrFail('1200'); // AR USD
$gainAccount = ium()->accounting()->accounts()->findByCodeOrFail('4900'); // Realized FX Gain

$fxVoucher = ium()->accounting()->journalEntries()->createDraft(new CreateJournalEntryDto(
    entryType: JournalEntryType::STANDARD,
    entryDate: '2026-03-25',
    currency: 'USD',
    creatorId: 'finance-clerk-1',
    lines: [
        new JournalEntryLineDto(accountId: $bankAccount->id, type: PostingSide::DEBIT, amount: 110000), // $1,100.00
        new JournalEntryLineDto(accountId: $arAccount->id, type: PostingSide::CREDIT, amount: 105000), // $1,050.00
        new JournalEntryLineDto(accountId: $gainAccount->id, type: PostingSide::CREDIT, amount: 5000),   // $50.00 FX Gain
    ],
    narration: 'Settlement with realized FX gain'
));
```

---

## Recipe 17: Sub-Unit & Minor Unit Formatting (Cents, Pence, Fils, Halalas) & Presenters

Useful for payment gateway integrations (e.g. Stripe integer minor units), POS cash registers, invoices, and cheque formatting:

```php
use Obelaw\Ium\Accounting\ValueObjects\Money;

// 1. Convert major units to integer minor units (e.g. for Stripe integer cents)
$cents = ium()->accounting()->currencies()->toSubUnit(49.99, 'USD');  // 4999
$fils  = ium()->accounting()->currencies()->toSubUnit(10.500, 'KWD'); // 10500

// 2. Convert from minor units back to major decimal currency
$amount = ium()->accounting()->currencies()->fromSubUnit(4999, 'USD'); // 49.99

// 3. Format sub-units with noun pluralization
echo ium()->accounting()->currencies()->formatSubUnit(1, 'USD'); // "1 Cents"
echo ium()->accounting()->currencies()->formatSubUnit(50, 'USD'); // "50 Cents"
echo ium()->accounting()->currencies()->formatSubUnit(50, 'USD', useSymbol: true); // "50 ¢"
echo ium()->accounting()->currencies()->formatSubUnit(50, 'GBP'); // "50 Pence"
echo ium()->accounting()->currencies()->formatSubUnit(250, 'KWD'); // "250 Fils"

// 4. Combined major and minor unit formatting
echo ium()->accounting()->currencies()->formatMajorAndSubUnit(12.50, 'USD'); // "$ 12 and 50 Cents"
echo ium()->accounting()->currencies()->formatMajorAndSubUnit(12.50, 'USD', useSymbol: true); // "$ 12 and 50 ¢"

// 5. Split units into associative components
$split = ium()->accounting()->currencies()->splitUnits(12.50, 'USD');
// ['major' => 12, 'minor' => 50, 'formatted' => '$ 12.50']

// 6. Using Money Value Object
$money = Money::fromMinor(1250, 'USD'); // $12.50
echo $money->amount; // 12.50
echo $money->toMinorUnits(); // 1250

// 7. Using Model Presenters & Formatters
$entry = ium()->accounting()->journalEntries()->find($voucherId);
echo $entry->totalDebitMajor(); // 500.00
echo $entry->formatTotalDebit(); // "$ 500.00"

$posting = $entry->postings->first();
echo $posting->debitMajor(); // 500.00
echo $posting->formatDebit(); // "$ 500.00"
```

---

## Recipe 18: Seeding Financial Demo Data via Artisan Command & Seeder

Rapidly establish an authentic, fully balanced multi-currency general ledger for manual exploration, demo environments, or automated end-to-end testing:

### Option A: Via Artisan Console Command
```bash
# Standard demo seeding
php artisan ium:accounting:demo-seed

# Clean wipe and re-seed without confirmation prompts
php artisan ium:accounting:demo-seed --fresh --force
```

### Option B: Programmatically in Database Seeders or Tests
```php
use Obelaw\Ium\Accounting\Demo\AccountingDemoSeeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // Execute comprehensive accounting demo seeder
        $this->call(AccountingDemoSeeder::class);
    }
}
```

### Option C: Inspecting Seeded Parity in Code
```php
// Verify zero-sum Trial Balance parity after seeding
$tb = ium()->accounting()->ledger()->generateTrialBalance();
assert($tb['is_balanced'] === true);
assert($tb['imbalance'] === 0);

// Verify Balance Sheet equation: Assets = Liabilities + Equity
$bs = ium()->accounting()->ledger()->generateBalanceSheet();
assert($bs['is_balanced'] === true);
```
