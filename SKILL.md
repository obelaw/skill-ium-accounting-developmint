---
name: ium-accounting-development
description: >-
  Comprehensive guide and workflow for AI agents developing with, integrating against, or extending
  the Obelawium Accounting (ium-accounting / FMS) domain module. Covers the ium()->accounting() fluent DSL,
  Chart of Accounts, Double-Entry General Ledger, Zero-Sum balance invariant, Storno reversals, Maker-Checker
  governance, Fiscal Calendar locks, Analytical Cost Centers, Multi-Currency exchange & conversion,
  Anti-Corruption Layer (ACL) cross-domain ingestion, and Year-End closing.
tags: [obelaw, ium, accounting, fms, general-ledger, double-entry, chart-of-accounts, storno, multi-currency, maker-checker, fiscal-periods, audit-trail, pest]
---

# Obelawium Accounting (`ium-accounting`) Development Guide

This skill guides an AI agent through developing with, integrating against, and extending the **Obelawium Accounting (FMS & General Ledger)** domain module (`obelaw/ium-accounting`).

---

## 1. Domain Philosophy & Fluent Architecture

The Accounting package is the central Financial Management System and General Ledger engine in the Obelawium ERP ecosystem. It is strictly **headless and decoupled from presentation** (no HTTP controllers, blade views, or frontend assets).

All external domain interactions must flow through the central Obelawium fluent DSL:

```php
ium()->accounting()->[service]()->[method]()
```

### Complete Service Catalog (9 Core Services)

| Service Accessor | Service Class | Primary Responsibilities |
| :--- | :--- | :--- |
| `ium()->accounting()->accounts()` | [`AccountService`](file:///E:/projects/obelaw/ium/packages/accounting/src/Services/AccountService.php) | Hierarchical Chart of Accounts (COA) tree, classification, leaf vs aggregator validation. |
| `ium()->accounting()->journalEntries()` | [`JournalEntryService`](file:///E:/projects/obelaw/ium/packages/accounting/src/Services/JournalEntryService.php) | Journal voucher drafting, zero-sum validation, Maker-Checker workflow, posting, Storno reversals. |
| `ium()->accounting()->ledger()` | [`LedgerService`](file:///E:/projects/obelaw/ium/packages/accounting/src/Services/LedgerService.php) | General Ledger book of entry, account ledgers, Trial Balance, Balance Sheet, Income Statement (P&L). |
| `ium()->accounting()->fiscalPeriods()` | [`FiscalPeriodService`](file:///E:/projects/obelaw/ium/packages/accounting/src/Services/FiscalPeriodService.php) | Fiscal years, monthly/quarterly periods, soft/hard locks, period resolution by transaction date. |
| `ium()->accounting()->costCenters()` | [`CostCenterService`](file:///E:/projects/obelaw/ium/packages/accounting/src/Services/CostCenterService.php) | Analytical accounting dimensions (Cost Centers, Projects, Departments, Business Units). |
| `ium()->accounting()->closing()` | [`ClosingService`](file:///E:/projects/obelaw/ium/packages/accounting/src/Services/ClosingService.php) | Fiscal year-end nominal account zeroing and Retained Earnings rollover transfer. |
| `ium()->accounting()->audit()` | [`AuditService`](file:///E:/projects/obelaw/ium/packages/accounting/src/Services/AuditService.php) | Append-only forensic accounting audit trail for statutory compliance and regulatory reporting. |
| `ium()->accounting()->acl()` | [`AclIngestionService`](file:///E:/projects/obelaw/ium/packages/accounting/src/Services/AclIngestionService.php) | Anti-Corruption Layer translating upstream domain events (Billing, Inventory, CRM) into balanced vouchers. |
| `ium()->accounting()->currencies()` | [`CurrencyService`](file:///E:/projects/obelaw/ium/packages/accounting/src/Services/CurrencyService.php) | Multi-Currency ISO 4217 registry, daily exchange rate snapshots, conversion, and FX rate providers. |

---

## 2. Non-Negotiable Domain Invariants ("The Law")

When developing or executing financial actions, you **MUST NEVER violate** these 8 fundamental rules:

1. **Zero-Sum Balance Invariant**:
   $$\sum \text{Debits} = \sum \text{Credits} \quad (\text{Exact integer parity down to minor unit})$$
   Every voucher must balance perfectly in integer minor units ($0 imbalance). All database amounts, balances, lines, and GL postings persist as integer minor units (e.g. cents). Unbalanced submissions immediately throw [`UnbalancedJournalEntryException`](file:///E:/projects/obelaw/ium/packages/accounting/src/Exceptions/UnbalancedJournalEntryException.php).

2. **General Ledger Immutability & Storno Reversal**:
   Posted entries (`POSTED`, `REVERSED`) are **strictly immutable**. Eloquent hooks forbid any in-place `update()` or `delete()`, throwing [`ImmutableJournalEntryException`](file:///E:/projects/obelaw/ium/packages/accounting/src/Exceptions/ImmutableJournalEntryException.php). Corrections must strictly use the **Storno reversal pattern** (`reverse()`), generating an equal-and-opposite posted voucher.

3. **Leaf-Only Account Postings**:
   Postings are **only permitted on leaf accounts** (`is_leaf = true`, classification `posting`). Aggregator parent nodes throw [`NonPostingAccountException`](file:///E:/projects/obelaw/ium/packages/accounting/src/Exceptions/NonPostingAccountException.php). Adding a child to a leaf account automatically converts the parent to a non-leaf aggregator.

4. **Fiscal Period Hard Locks**:
   Transactions must resolve to an open period (`status === OPEN`). Postings into locked periods throw [`FiscalPeriodClosedException`](file:///E:/projects/obelaw/ium/packages/accounting/src/Exceptions/FiscalPeriodClosedException.php).

5. **Maker-Checker Segregation of Duties**:
   When strict governance is active (`ium.accounting.maker_checker.strict = true`), a voucher's creator (`creator_id`) **cannot approve** that voucher (`approver_id`). Violations throw [`MakerCheckerViolationException`](file:///E:/projects/obelaw/ium/packages/accounting/src/Exceptions/MakerCheckerViolationException.php).

6. **Base Functional Currency Rate Immutability**:
   The organization's functional currency is the primary ledger denominator. Its exchange rate is locked to `1.000000` and cannot be modified directly. Attempting to set an exchange rate on the base currency throws [`BaseCurrencyRateImmutableException`](file:///E:/projects/obelaw/ium/packages/accounting/src/Exceptions/BaseCurrencyRateImmutableException.php).

7. **Functional Currency General Ledger Denomination**:
   While vouchers can be drafted in any active ISO currency (`amount` in integer minor units), the General Ledger postings (`GeneralLedgerPosting`) are always recorded in the functional reporting currency using `amount_functional = (int) round($amount * $exchangeRate)`.

8. **Anti-Corruption Layer (ACL) Isolation**:
   External modules (Billing, Sales, Inventory, CRM) must **never write to accounting database tables directly**. They must communicate exclusively via the ACL (`ium()->accounting()->acl()->ingest(...)`) or registered domain event mappings.

For complete invariants and error diagnosis, consult the [Invariants & Exceptions Playbook](./references/invariants-and-exceptions.md).

---

## 3. Agent Decision Matrix

| What You Need to Do | Method to Invoke | Input DTO / Parameters | Expected Output |
| :--- | :--- | :--- | :--- |
| **Create an Account in COA** | `ium()->accounting()->accounts()->create($dto)` | [`CreateAccountDto`](file:///E:/projects/obelaw/ium/packages/accounting/src/Data/CreateAccountDto.php) | [`Account`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/Account.php) |
| **Get Hierarchical Account Tree** | `ium()->accounting()->accounts()->tree()` | None | `Collection<Account>` |
| **Setup Fiscal Year & Periods** | `fiscalPeriods()->createYear($dto)` / `createPeriod($dto)` | [`CreateFiscalYearDto`](file:///E:/projects/obelaw/ium/packages/accounting/src/Data/CreateFiscalYearDto.php) | [`FiscalYear`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/FiscalYear.php) |
| **Draft a Journal Voucher** | `journalEntries()->createDraft($dto)` | [`CreateJournalEntryDto`](file:///E:/projects/obelaw/ium/packages/accounting/src/Data/CreateJournalEntryDto.php) | [`JournalEntry`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/JournalEntry.php) (`draft`) |
| **Draft Voucher from Major Units** | `CreateJournalEntryDto::fromMajor($data)` | Array with decimal amounts (e.g. `12.50`) | [`CreateJournalEntryDto`](file:///E:/projects/obelaw/ium/packages/accounting/src/Data/CreateJournalEntryDto.php) |
| **Submit Voucher for Approval** | `journalEntries()->submit($id, $actorId)` | Entry ID, Actor ID | [`JournalEntry`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/JournalEntry.php) (`pending_approval`) |
| **Approve Voucher (Maker-Checker)** | `journalEntries()->approve($id, $approverId)` | Entry ID, Distinct Approver ID | [`JournalEntry`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/JournalEntry.php) (`approved`) |
| **Post Voucher to GL** | `journalEntries()->post($id, $actorId)` | Entry ID, Actor ID | [`JournalEntry`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/JournalEntry.php) (`posted`) |
| **Reverse Erroneous Voucher** | `journalEntries()->reverse($id, $actorId, $reason)` | Entry ID, Actor ID, Reason String | New Reversal [`JournalEntry`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/JournalEntry.php) |
| **Ingest Upstream Domain Event** | `acl()->ingest($dto)` | [`AclIngestEventDto`](file:///E:/projects/obelaw/ium/packages/accounting/src/Data/AclIngestEventDto.php) | Balanced & Posted [`JournalEntry`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/JournalEntry.php) |
| **Register Domain Event Mapping** | `acl()->registerMapping($name, $dr, $cr)` | Event Name, DR Code, CR Code | [`EventAccountMapping`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/EventAccountMapping.php) |
| **Run Trial Balance** | `ledger()->generateTrialBalance($periodId)` | Optional Period ID | Array with `is_balanced`, integer sums, major sums |
| **Run Balance Sheet** | `ledger()->generateBalanceSheet()` | None | Assets, Liabilities, Equity, Discrepancy |
| **Run Income Statement (P&L)**| `ledger()->generateIncomeStatement($periodId)` | Optional Period ID | Revenues, Expenses, Net Income |
| **Execute Year-End Close** | `closing()->closeFiscalYear($dto)` | [`FiscalYearCloseDto`](file:///E:/projects/obelaw/ium/packages/accounting/src/Data/FiscalYearCloseDto.php) | Sealed [`FiscalYear`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/FiscalYear.php) |
| **Register ISO Currency** | `currencies()->create($dto)` | [`CreateCurrencyDto`](file:///E:/projects/obelaw/ium/packages/accounting/src/Data/CreateCurrencyDto.php) | [`Currency`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/Currency.php) |
| **Record Exchange Rate** | `currencies()->setRate($code, $rate, $date)` | ISO Code, Rate, Date, Source, isDirect | [`ExchangeRate`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/ExchangeRate.php) |
| **Convert Amount to Currency** | `currencies()->convert($amount, $from, $to, $date)` | Amount, From Code, To Code, Date | `float` (rounded to target precision) |
| **Convert Minor Units** | `currencies()->convertMinor($minor, $from, $to, $date)` | Minor units, From, To, Date | `int` (minor units in target currency) |
| **Convert Currency Detailed** | `currencies()->convertDetailed($amount, $from, $to, $date)` | Amount, From Code, To Code, Date | [`CurrencyConversionResultDto`](file:///E:/projects/obelaw/ium/packages/accounting/src/Data/CurrencyConversionResultDto.php) |
| **Switch Base Currency** | `currencies()->setBaseCurrency($code, $actorId)` | ISO Code, Actor ID | [`Currency`](file:///E:/projects/obelaw/ium/packages/accounting/src/Models/Currency.php) |
| **Seed Default Currencies** | `currencies()->seedDefaultCurrencies()` | None | `Collection<Currency>` (USD, EUR, GBP, SAR, etc.) |
| **Convert to Sub-Units / Cents** | `currencies()->toSubUnit($amount, $code)` | Amount, Optional Currency Code | `int` (e.g. 12.50 USD $\to$ 1250) |
| **Convert from Sub-Units** | `currencies()->fromSubUnit($subUnits, $code)` | Minor Units, Optional Currency Code | `float` (e.g. 1250 $\to$ 12.50) |
| **Format Minor Units String** | `currencies()->formatMinor($subUnits, $code)` | Minor Units, Optional Code | `string` (e.g. `"$ 12.50"`) |
| **Format Sub-Unit String** | `currencies()->formatSubUnit($subUnits, $code, $useSymbol)` | Minor Units, Optional Code, Use Symbol | `string` (e.g. `"50 Cents"` or `"50 ¢"`) |
| **Format Major & Minor Units** | `currencies()->formatMajorAndSubUnit($amount, $code, $useSymbol)` | Amount, Optional Code, Use Symbol | `string` (e.g. `"$ 12 and 50 Cents"`) |
| **Split Major & Minor Units** | `currencies()->splitUnits($amount, $code)` | Amount, Optional Currency Code | `array` (`major`, `minor`, formatted) |
| **Seed Financial Demo Data** | `php artisan ium:accounting:demo-seed` | `--fresh`, `--force` flags | Turnkey balanced multi-currency demo ledger |

---

## 4. Step-by-Step AI Agent Operational Procedure

Follow this 10-step sequence when developing, refactoring, or integrating with accounting features:

### Step 1: Verify Environment & Database Tables
All accounting tables carry the `ium_accounting_` prefix:
- `ium_accounting_accounts`
- `ium_accounting_journal_entries`
- `ium_accounting_journal_entry_lines`
- `ium_accounting_gl_postings`
- `ium_accounting_fiscal_years` & `ium_accounting_fiscal_periods`
- `ium_accounting_cost_centers`
- `ium_accounting_audit_logs`
- `ium_accounting_event_account_mappings`
- `ium_accounting_currencies`
- `ium_accounting_exchange_rates`

All persistence models must extend [`Obelaw\Ium\Accounting\Base\ModelBase`](file:///E:/projects/obelaw/ium/packages/accounting/src/Base/ModelBase.php).

### Step 2: Establish Base Currency & Exchange Rates
Ensure the functional reporting currency is registered, and any transaction currencies have effective exchange rates:
```php
$baseCurrency = ium()->accounting()->currencies()->getBaseCurrency();
$eurRate = ium()->accounting()->currencies()->getRate('EUR', $baseCurrency->code, '2026-03-15');
```

### Step 3: Ensure Fiscal Period Is Open
Before drafting vouchers for a given date, verify an open fiscal period exists:
```php
$period = ium()->accounting()->fiscalPeriods()->findPeriodForDate($date);
if (!$period || !$period->status->allowsPosting()) {
    throw new RuntimeException("No open fiscal period found for transaction date {$date}.");
}
```

### Step 4: Validate Accounts in Chart of Accounts
Ensure target accounts exist and permit direct postings:
```php
$account = ium()->accounting()->accounts()->findByCodeOrFail($code);
if (!$account->allowsPostings()) {
    throw new RuntimeException("Account {$code} is an aggregator; select a leaf posting account.");
}
```

### Step 5: Construct Balanced Line Items & Multi-Currency Amounts
Build line items with matching total debits and credits in **integer minor units (cents)** or use `fromMajor()`:
```php
// Option A: Integer minor units (Recommended for zero rounding loss - e.g. 50000 = $500.00)
$lines = [
    new JournalEntryLineDto(
        accountId: $drAccount->id,
        type: PostingSide::DEBIT,
        amount: 50000,
        narration: 'Office equipment purchase'
    ),
    new JournalEntryLineDto(
        accountId: $crAccount->id,
        type: PostingSide::CREDIT,
        amount: 50000,
        narration: 'Cash disbursement'
    ),
];

// Option B: Major units with automatic conversion
$lines = [
    JournalEntryLineDto::fromMajor($drAccount->id, PostingSide::DEBIT, 500.00),
    JournalEntryLineDto::fromMajor($crAccount->id, PostingSide::CREDIT, 500.00),
];
```

### Step 6: Draft and Submit Through Governance
```php
$draft = ium()->accounting()->journalEntries()->createDraft(new CreateJournalEntryDto(
    entryType: JournalEntryType::STANDARD,
    entryDate: '2026-03-15',
    currency: 'USD',
    creatorId: 'user-alice',
    lines: $lines,
));

$submitted = ium()->accounting()->journalEntries()->submit($draft->id, 'user-alice');
$approved  = ium()->accounting()->journalEntries()->approve($submitted->id, 'user-bob-supervisor');
$posted    = ium()->accounting()->journalEntries()->post($approved->id, 'user-bob-supervisor');
```

### Step 7: Verify Zero-Sum Parity in General Ledger
Check the General Ledger Trial Balance:
```php
$trialBalance = ium()->accounting()->ledger()->generateTrialBalance();
assert($trialBalance['is_balanced'] === true);
assert($trialBalance['imbalance'] === 0);
```

### Step 8: Handle Reversals via Storno Pattern
If a transaction must be cancelled, never call `$entry->delete()`. Call:
```php
$reversal = ium()->accounting()->journalEntries()->reverse(
    entryId: $postedEntry->id,
    actorId: 'auditor-user-id',
    reason: 'Correction of erroneous invoice posting'
);
```

### Step 9: Integrate External Domains via ACL
When building modules like Billing, Inventory, or Sales:
```php
ium()->accounting()->acl()->ingest(AclIngestEventDto::from([
    'event_name' => 'sales.order.invoiced',
    'document_reference' => $order->reference,
    'transaction_date' => now()->format('Y-m-d'),
    'amount' => $order->total_cents, // Integer minor units (cents)
    'currency' => $order->currency ?? 'USD',
    'debit_account_code' => '1200',  // Accounts Receivable
    'credit_account_code' => '4100', // Sales Revenue
    'auto_post' => true,
]));
```

### Step 10: Run Pest Test Suite
Run tests inside `packages/accounting`:
```powershell
./vendor/bin/pest
```
All feature and unit tests must pass with 0 failures before delivering work (40 tests, 233 assertions).

---

## 5. Diagnostic & Error Recovery Guide

| Exception | Root Cause | Agent Remediation |
| :--- | :--- | :--- |
| `UnbalancedJournalEntryException` | Sum of Debits $\ne$ Sum of Credits ($\| \text{Diff} \| > 0$). | Verify all line amounts; ensure total debits equal total credits perfectly in minor units before submitting. |
| `ImmutableJournalEntryException` | An `update()` or `delete()` was attempted on a posted voucher. | Call `reverse()` instead of mutating or deleting the posted voucher. |
| `NonPostingAccountException` | Line targets a parent account (`is_leaf = false`). | Target a leaf child account (`classification === AccountClassification::POSTING`). |
| `FiscalPeriodClosedException` | Transaction falls into a locked period. | Change entry date to an open period or call `reopenPeriod()`. |
| `MakerCheckerViolationException` | Approver ID equals Creator ID in strict mode. | Provide a distinct approver ID or set `ium.accounting.maker_checker.strict = false` in tests. |
| `ReversalNotAllowedException` | Voucher is not posted or is already reversed. | Only reverse vouchers with status `POSTED` and `reversal_entry_id === null`. |
| `BaseCurrencyRateImmutableException` | Attempted to set rate on base currency. | The base functional currency exchange rate is immutable and permanently `1.0`. |
| `CurrencyNotFoundException` | Currency code not registered in ISO catalog. | Register currency via `currencies()->create($dto)` or seed via `seedDefaultCurrencies()`. |
| `ExchangeRateNotFoundException` | No effective rate found for date cutoff. | Record rate via `currencies()->setRate()` for the given date or check currency provider. |
| `InvalidArgumentException` | Duplicate currency code, invalid ISO length, or invalid precision. | Use 3-letter ISO code, precision 0–6, and unique currency code. |

---

## 6. Deep Dive References

For complete code examples, architectural diagrams, and cross-domain patterns:
- [Architecture & Domain Model Deep Dive](./references/architecture.md)
- [Developer Recipes & Code Cookbook](./references/recipes.md)
- [Invariants, Exceptions, & Enums Playbook](./references/invariants-and-exceptions.md)
- [Cross-Domain Integration Guide (ACL & Events)](./references/cross-domain-integration.md)
- [Testing & Verification Guide](./references/testing-and-verification.md)
