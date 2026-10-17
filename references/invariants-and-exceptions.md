# Obelawium Accounting: Invariants, Exceptions, & Enums Reference

This reference documents the non-negotiable domain rules, exception hierarchy, recovery actions, and enums within `obelaw/ium-accounting`.

---

## 1. Domain Invariants Checklist

When developing or executing accounting operations, verify every item in this checklist:

| Invariant | Description | Enforcement Point | Exception Thrown |
| :--- | :--- | :--- | :--- |
| **1. Zero-Sum Balance** | Total Debits must equal Total Credits down to minor units ($\| \text{Debits} - \text{Credits} \| = 0$). | `submit()`, `post()` | `UnbalancedJournalEntryException` |
| **2. Ledger Immutability** | Posted vouchers and ledger records cannot be edited or deleted. | `JournalEntry::booted()` | `ImmutableJournalEntryException` |
| **3. Storno Reversal Only** | Erroneous postings can only be fixed by generating an opposite reversal voucher. | `reverse()` | `ReversalNotAllowedException` |
| **4. Leaf-Only Posting** | Postings are rejected on root or parent accounts (`is_leaf = false`). | `createDraft()` | `NonPostingAccountException` |
| **5. Period Open Gate** | Transactions must fall into an open, unlocked fiscal period. | `createDraft()`, `post()` | `FiscalPeriodClosedException` |
| **6. Maker-Checker Segregation** | Approver cannot be the same user who created the voucher when strict mode is active. | `approve()` | `MakerCheckerViolationException` |
| **7. Base Currency Immutability**| The base functional currency rate is locked at `1.000000` and cannot be modified directly. | `CurrencyService::setRate()` | `BaseCurrencyRateImmutableException` |
| **8. ISO 4217 Currency Code** | Currencies require a valid, unique 3-character uppercase ISO code with precision 0–6. | `CreateCurrencyDto` | `InvalidArgumentException` |
| **9. Historical Rate Cutoff** | Rate lookups resolve the most recent rate on or before the transaction date. | `CurrencyService::getRate()` | `ExchangeRateNotFoundException` |
| **10. Unique Account & Period Codes**| Account codes and cost center codes must be unique across the organization. | `AccountService::create()` | `InvalidArgumentException` |

---

## 2. Exceptions & Agent Recovery Playbook

### `UnbalancedJournalEntryException`
- **Class**: `Obelaw\Ium\Accounting\Exceptions\UnbalancedJournalEntryException`
- **When Thrown**: Total Debits $\ne$ Total Credits during `assertZeroSum()` (even 1 minor unit / 1 cent difference).
- **Message Format**: `Zero-sum invariant violated for voucher "...": Total debits (10000) do not equal total credits (8000). Imbalance is 2000.`
- **Agent Recovery**:
  1. Inspect all lines in `$dto->lines`.
  2. Sum integer amounts where `type === PostingSide::DEBIT` and `type === PostingSide::CREDIT`.
  3. Ensure the difference is 0 ($0 imbalance), adjusting fractional rounding or balancing lines before calling `submit()` or `post()`. Alternatively, use `CreateJournalEntryDto::fromMajor(...)` which handles scaling into minor units.

---

### `ImmutableJournalEntryException`
- **Class**: `Obelaw\Ium\Accounting\Exceptions\ImmutableJournalEntryException`
- **When Thrown**: Eloquent model event intercepts an `update()` or `delete()` on a `JournalEntry` with status `POSTED` or `REVERSED`.
- **Message Format**: `Journal entry "..." is posted and immutable. Mutation type "update" is prohibited.`
- **Agent Recovery**:
  - **NEVER** call `$entry->update([...])` or `$entry->delete()` on a posted entry.
  - To cancel or correct a posted entry, execute a Storno reversal:
    ```php
    $reversal = ium()->accounting()->journalEntries()->reverse(
        entryId: $entry->id,
        actorId: 'auditor-user-id',
        reason: 'Correction of erroneous debit amount'
    );
    ```
  - Then create a new, corrected draft voucher.

---

### `NonPostingAccountException`
- **Class**: `Obelaw\Ium\Accounting\Exceptions\NonPostingAccountException`
- **When Thrown**: A journal entry line targets an aggregator/parent account (`is_leaf = false` or classification != `posting`).
- **Message Format**: `Account "1000 - Assets" is an aggregator classification (class) and does not permit direct postings.`
- **Agent Recovery**:
  1. Retrieve the target account using `ium()->accounting()->accounts()->find($id)`.
  2. Select a leaf-level child account (`is_leaf === true`, `classification === AccountClassification::POSTING`).
  3. Or create a leaf child under this parent account before generating lines.

---

### `FiscalPeriodClosedException`
- **Class**: `Obelaw\Ium\Accounting\Exceptions\FiscalPeriodClosedException`
- **When Thrown**: Transaction date resolves to a period with `status === LOCKED` (or `SOFT_CLOSED` for non-privileged actors).
- **Message Format**: `Fiscal period "January 2026" (ID: 1) is locked and prohibits new general ledger postings.`
- **Agent Recovery**:
  1. Check the voucher date: `$date = $dto->entryDate;`.
  2. Verify active periods via `ium()->accounting()->fiscalPeriods()->findPeriodForDate($date)`.
  3. Post to an open period, or if authorized, reopen the period via `ium()->accounting()->fiscalPeriods()->reopenPeriod($periodId, $actorId, $reason)`.

---

### `MakerCheckerViolationException`
- **Class**: `Obelaw\Ium\Accounting\Exceptions\MakerCheckerViolationException`
- **When Thrown**: The user calling `approve()` has the same `creator_id` as the voucher when `ium.accounting.maker_checker.strict` is `true`.
- **Message Format**: `Maker-Checker violation: User "user-123" cannot approve a journal entry they created.`
- **Agent Recovery**:
  1. Ensure the approver actor ID is distinct from the creator actor ID.
  2. In test environments or seeders where dual actors are not needed, disable strict mode via `config(['ium.accounting.maker_checker.strict' => false])`.

---

### `InvalidStateTransitionException`
- **Class**: `Obelaw\Ium\Accounting\Exceptions\InvalidStateTransitionException`
- **When Thrown**: An invalid voucher lifecycle transition is attempted (e.g. approving a `DRAFT` before submitting, or submitting an already `POSTED` voucher).
- **Message Format**: `Journal entry "..." cannot transition from "draft" to "approved".`
- **Agent Recovery**:
  - Respect the linear state machine:
    $$\text{DRAFT} \xrightarrow{\text{submit()}} \text{PENDING\_APPROVAL} \xrightarrow{\text{approve()}} \text{APPROVED} \xrightarrow{\text{post()}} \text{POSTED}$$
  - If a voucher was rejected:
    $$\text{REJECTED} \xrightarrow{\text{submit()}} \text{PENDING\_APPROVAL}$$

---

### `ReversalNotAllowedException`
- **Class**: `Obelaw\Ium\Accounting\Exceptions\ReversalNotAllowedException`
- **When Thrown**: Attempting to reverse a voucher that is not `POSTED`, or one that has already been reversed.
- **Message Format**: `Journal entry "..." is already reversed by voucher "...". Multiple reversals are prohibited.`
- **Agent Recovery**:
  1. Verify `$entry->status === JournalEntryStatus::POSTED`.
  2. Verify `$entry->reversal_entry_id === null`.

---

### `BaseCurrencyRateImmutableException`
- **Class**: `Obelaw\Ium\Accounting\Exceptions\BaseCurrencyRateImmutableException`
- **When Thrown**: Attempting to set an exchange rate for the organization's base functional currency.
- **Message Format**: `The exchange rate for base functional currency "USD" is permanently fixed to 1.0 and cannot be altered.`
- **Agent Recovery**:
  - Never call `currencies()->setRate()` for the base currency.
  - To change the base currency itself, use `currencies()->setBaseCurrency($newCode, $actorId)`.

---

### `CurrencyNotFoundException`
- **Class**: `Obelaw\Ium\Accounting\Exceptions\CurrencyNotFoundException`
- **When Thrown**: A requested currency code does not exist in `ium_accounting_currencies`.
- **Message Format**: `Currency with ISO code "XYZ" not found in the currency registry.`
- **Agent Recovery**:
  1. Verify the currency code is correct (3-letter uppercase).
  2. Register the missing currency using `ium()->accounting()->currencies()->create($dto)`.
  3. Or run `ium()->accounting()->currencies()->seedDefaultCurrencies()` in testing/setup.

---

### `ExchangeRateNotFoundException`
- **Class**: `Obelaw\Ium\Accounting\Exceptions\ExchangeRateNotFoundException`
- **When Thrown**: No exchange rate record exists on or before the requested effective date.
- **Message Format**: `No exchange rate found for currency "EUR" on or before 2026-03-15.`
- **Agent Recovery**:
  1. Record an effective exchange rate snapshot for that date cutoff:
     ```php
     ium()->accounting()->currencies()->setRate(
         code: 'EUR',
         rate: 1.085000,
         effectiveDate: '2026-03-15',
         source: 'central_bank'
     );
     ```
  2. Or configure an external rate driver implementing `ExchangeRateProviderInterface`.

---

### `AccountNotFoundException`
- **Class**: `Obelaw\Ium\Accounting\Exceptions\AccountNotFoundException`
- **When Thrown**: An account code or ID does not exist in `ium_accounting_accounts`.
- **Message Format**: `Account with code "9999" not found.` or `Account with ID 999 not found.`
- **Agent Recovery**:
  1. Query `ium()->accounting()->accounts()->findByCode($code)`.
  2. If missing, seed or create the account before posting.

---

## 3. Enumerations Reference

### `AccountType` (`src/Enums/AccountType.php`)
```php
namespace Obelaw\Ium\Accounting\Enums;

enum AccountType: string {
    case ASSET     = 'asset';     // Normal Balance: DEBIT, Real (Balance Sheet)
    case LIABILITY = 'liability'; // Normal Balance: CREDIT, Real (Balance Sheet)
    case EQUITY    = 'equity';    // Normal Balance: CREDIT, Real (Balance Sheet)
    case REVENUE   = 'revenue';   // Normal Balance: CREDIT, Nominal (P&L)
    case EXPENSE   = 'expense';   // Normal Balance: DEBIT, Nominal (P&L)
}
```

### `AccountClassification` (`src/Enums/AccountClassification.php`)
```php
namespace Obelaw\Ium\Accounting\Enums;

enum AccountClassification: string {
    case CLASS_ROOT = 'class';      // e.g. "1 - Assets" (Non-posting aggregator)
    case MAIN_GROUP = 'main_group'; // e.g. "11 - Current Assets" (Non-posting aggregator)
    case SUB_GROUP  = 'sub_group';  // e.g. "111 - Cash and Equivalents" (Non-posting aggregator)
    case POSTING    = 'posting';    // e.g. "1111 - Operating Bank Account" (POSTING ALLOWED)
}
```

### `PostingSide` (`src/Enums/PostingSide.php`)
```php
namespace Obelaw\Ium\Accounting\Enums;

enum PostingSide: string {
    case DEBIT  = 'debit';
    case CREDIT = 'credit';

    public function opposite(): self; // Returns DEBIT for CREDIT, and CREDIT for DEBIT
}
```

### `JournalEntryStatus` (`src/Enums/JournalEntryStatus.php`)
```php
namespace Obelaw\Ium\Accounting\Enums;

enum JournalEntryStatus: string {
    case DRAFT            = 'draft';
    case PENDING_APPROVAL = 'pending_approval';
    case APPROVED         = 'approved';
    case POSTED           = 'posted';
    case REVERSED         = 'reversed';
    case REJECTED         = 'rejected';
}
```

### `JournalEntryType` (`src/Enums/JournalEntryType.php`)
```php
namespace Obelaw\Ium\Accounting\Enums;

enum JournalEntryType: string {
    case STANDARD  = 'standard';
    case ADJUSTING = 'adjusting';
    case CLOSING   = 'closing';
    case REVERSAL  = 'reversal';
    case OPENING   = 'opening';
}
```

### `FiscalPeriodStatus` (`src/Enums/FiscalPeriodStatus.php`)
```php
namespace Obelaw\Ium\Accounting\Enums;

enum FiscalPeriodStatus: string {
    case OPEN        = 'open';
    case SOFT_CLOSED = 'soft_closed';
    case LOCKED      = 'locked';
}
```

### `DimensionType` (`src/Enums/DimensionType.php`)
```php
namespace Obelaw\Ium\Accounting\Enums;

enum DimensionType: string {
    case COST_CENTER = 'cost_center';
    case PROJECT     = 'project';
    case DEPARTMENT  = 'department';
}
```

### `AuditAction` (`src/Enums/AuditAction.php`)
```php
namespace Obelaw\Ium\Accounting\Enums;

enum AuditAction: string {
    case CREATED          = 'created';
    case UPDATED          = 'updated';
    case SUBMITTED        = 'submitted';
    case APPROVED         = 'approved';
    case REJECTED         = 'rejected';
    case POSTED           = 'posted';
    case REVERSED         = 'reversed';
    case PERIOD_LOCKED    = 'period_locked';
    case PERIOD_REOPENED  = 'period_reopened';
    case YEAR_END_CLOSED  = 'year_end_closed';
}
```

---

## 4. Domain Events Catalog (13 Core Domain Events)

The accounting domain dispatches 13 domain events across the voucher lifecycle, fiscal periods, and currency management:

| Event Class | Namespace | Trigger Description |
| :--- | :--- | :--- |
| `JournalEntryCreated` | `Obelaw\Ium\Accounting\Events` | Dispatched when a draft journal voucher is created. |
| `JournalEntrySubmitted` | `Obelaw\Ium\Accounting\Events` | Dispatched after zero-sum validation passes and voucher enters pending approval. |
| `JournalEntryApproved` | `Obelaw\Ium\Accounting\Events` | Dispatched when a distinct approver signs off on a voucher. |
| `JournalEntryRejected` | `Obelaw\Ium\Accounting\Events` | Dispatched when an approver rejects a voucher with explanation. |
| `JournalEntryPosted` | `Obelaw\Ium\Accounting\Events` | Dispatched when a voucher is permanently committed to the General Ledger. |
| `JournalEntryReversed` | `Obelaw\Ium\Accounting\Events` | Dispatched when an erroneous voucher is reversed via Storno correction. |
| `FiscalPeriodClosed` | `Obelaw\Ium\Accounting\Events` | Dispatched when an accounting period is soft-closed or locked. |
| `FiscalPeriodReopened` | `Obelaw\Ium\Accounting\Events` | Dispatched when a closed period is reopened with administrative reason. |
| `FiscalYearClosed` | `Obelaw\Ium\Accounting\Events` | Dispatched when a fiscal year is closed and nominal accounts zeroed out. |
| `AccountBalanceThresholdBreached` | `Obelaw\Ium\Accounting\Events` | Dispatched when an account balance exceeds configured threshold limits. |
| `CurrencyCreated` | `Obelaw\Ium\Accounting\Events` | Dispatched when a new ISO 4217 currency is registered. |
| `ExchangeRateUpdated` | `Obelaw\Ium\Accounting\Events` | Dispatched when an exchange rate snapshot is recorded or refreshed. |
| `BaseCurrencyChanged` | `Obelaw\Ium\Accounting\Events` | Dispatched when the functional reporting base currency is reassigned. |
