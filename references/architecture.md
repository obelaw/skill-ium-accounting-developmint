# Obelawium Accounting (`ium-accounting`) Architecture Reference

This document details the architectural foundation, design patterns, state machines, domain models, and multi-currency engine of the `obelaw/ium-accounting` domain package.

---

## 1. Domain Overview & High-Level Architecture

The Accounting domain (`obelaw/ium-accounting`) is the core Financial Management System (FMS) and General Ledger engine of the Obelawium ERP ecosystem. It is strictly **headless**, containing zero HTTP routes, controllers, blade views, or UI assets.

All enterprise operations are invoked through the centralized Obelawium fluent DSL:

```php
ium()->accounting()->{service}()->{method}()
```

```
+-----------------------------------------------------------------------------------------------------------------------+
|                                              Central Obelawium Gateway                                                |
|                                                  ium()->accounting()                                                  |
+-----------------------------------------------------------+-----------------------------------------------------------+
                                                            |
                                                            v
+-----------------------------------------------------------------------------------------------------------------------+
|                                             AccountingDomain (Gateway)                                                |
+------------+----------------+--------------+---------------+---------------+-----------+--------+------------+--------+
| accounts() | journalEntries()|   ledger()   |fiscalPeriods()| costCenters() | closing() | audit()|   acl()    |currencies()
+-----+------+-------+--------+-------+------+-------+-------+-------+-------+-----+-----+----+---+-----+----+----+----+---+
      |              |                |              |               |             |          |         |          |
      v              v                v              v               v             v          v         v          v
+------------+----------------+--------------+---------------+---------------+-----------+--------+------------+--------+
|  Account   |  JournalEntry  |LedgerService | FiscalPeriod  |  CostCenter   |  Closing  | Audit  |    ACL     |Currency|
|  Service   |    Service     | (GL Engine)  |    Service    |    Service    |  Service  |Service |  Service   |Service |
+-----+------+-------+--------+-------+------+-------+-------+-------+-------+-----+-----+----+---+-----+----+----+----+---+
      |              |                |              |               |             |          |         |          |
      +--------------+----------------+--------------+---------------+-------------+----------+---------+----------+
                                                            |
                                                            v
+-----------------------------------------------------------------------------------------------------------------------+
|                                      Enterprise Double-Entry Persistence Layer                                        |
|        ium_accounting_accounts                      ium_accounting_journal_entries                                    |
|        ium_accounting_journal_entry_lines           ium_accounting_gl_postings (Immutable)                            |
|        ium_accounting_fiscal_years                  ium_accounting_fiscal_periods                                     |
|        ium_accounting_cost_centers                  ium_accounting_audit_logs                                         |
|        ium_accounting_event_account_mappings        ium_accounting_currencies                                         |
|        ium_accounting_exchange_rates                                                                                  |
+-----------------------------------------------------------------------------------------------------------------------+
```

---

## 2. Fundamental Accounting Invariants

Every AI agent developing with or integrating into `ium-accounting` must adhere to these mathematical and regulatory principles:

### A. The Zero-Sum Invariant
$$\sum \text{Debits} = \sum \text{Credits} \quad (\text{Exact integer parity down to minor unit})$$
- A journal voucher is fundamentally invalid if the sum of all debit line amounts does not equal the sum of all credit line amounts down to exact integer minor units (0 cents imbalance).
- All transactional amounts and ledger balances persist as integer minor units to eliminate floating-point rounding inaccuracies.
- Attempting to submit or post an unbalanced voucher immediately throws `UnbalancedJournalEntryException`.

### B. Double-Entry General Ledger Immutability
- Once a journal voucher transitions into `POSTED` status, it is **strictly immutable**.
- The `JournalEntry` Eloquent model registers boot lifecycle listeners that reject updates and deletions to posted entries, throwing `ImmutableJournalEntryException`.
- **Erroneous entries are NEVER deleted or overwritten in place.** Corrections must be executed via the **Storno reversal pattern** (`ium()->accounting()->journalEntries()->reverse(...)`), creating an equal-and-opposite posted reversal voucher linked bi-directionally.

### C. Leaf vs. Aggregator Account Integrity
- The Chart of Accounts (COA) is a hierarchical tree (`class` $\to$ `main_group` $\to$ `sub_group` $\to$ `posting`).
- Postings can **only** occur on leaf-level accounts (`is_leaf = true`, classification `posting`).
- Parent accounts aggregate children and reject direct postings by throwing `NonPostingAccountException`.
- Adding a child account to an existing leaf account automatically promotes the parent to non-leaf aggregator (`is_leaf = false`).

### D. Fiscal Calendar Governance & Period Hard Locks
- All financial transactions belong to a `FiscalPeriod` residing within a `FiscalYear`.
- Periods have three states: `OPEN`, `SOFT_CLOSED`, and `LOCKED`.
- Postings into a `LOCKED` period throw `FiscalPeriodClosedException`.
- Year-end closing hard-locks the fiscal year and all periods within it.

### E. Maker-Checker Segregation of Duties
- In strict governance mode (`ium.accounting.maker_checker.strict = true`), the creator of a voucher (`creator_id`) cannot approve that voucher (`approver_id`).
- Attempting self-approval throws `MakerCheckerViolationException`.

### F. Base Functional Currency Rate Immutability
- The functional currency (e.g. `'USD'`) is the statutory reporting currency.
- Its exchange rate is permanently locked at `1.000000`. Attempting to set or modify its rate throws `BaseCurrencyRateImmutableException`.

---

## 3. Core Domain Entities & Database Schema

All tables carry the prefix `ium_accounting_`.

### `Account` (Table: `ium_accounting_accounts`)
- **Primary Key**: `id`
- **Natural Key**: `code` (e.g., `'1001'`, `'1100.01'`)
- **Type**: `AccountType` (`ASSET`, `LIABILITY`, `EQUITY`, `REVENUE`, `EXPENSE`)
- **Classification**: `AccountClassification` (`CLASS_ROOT`, `MAIN_GROUP`, `SUB_GROUP`, `POSTING`)
- **Normal Balance**: `NormalBalance` (`DEBIT` or `CREDIT`)
- **Hierarchy**: Self-referencing `parent_id` foreign key.
- **Current Balance**: Running balance cached as integer minor units (`current_balance`), updated atomically upon posting.
- **Presenters & Formatters**:
  - `currentBalanceMajor(?int $precision = null): float` (e.g. `125050` $\to$ `1250.50`)
  - `formatCurrentBalance(?Currency $currency = null): string` (e.g. `"$ 1,250.50"`)

### `JournalEntry` (Table: `ium_accounting_journal_entries`)
- **Aggregate Root** for accounting vouchers.
- **Lifecycle Status**: `JournalEntryStatus` (`DRAFT`, `PENDING_APPROVAL`, `APPROVED`, `POSTED`, `REVERSED`, `REJECTED`)
- **Entry Type**: `JournalEntryType` (`STANDARD`, `ADJUSTING`, `CLOSING`, `REVERSAL`, `OPENING`)
- **Totals**: `total_debit` and `total_credit` stored as integer minor units (e.g. `50000` for $500.00).
- **Presenters & Formatters**:
  - `totalDebitMajor()`, `totalCreditMajor()`
  - `formatTotalDebit(?Currency $currency = null)`, `formatTotalCredit(?Currency $currency = null)`
- **Currency & FX**: `currency` (3-character ISO code) and `exchange_rate` (rate relative to functional currency).
- **Polymorphic Source**: `source_type` and `source_id` link vouchers back to originating business entities (e.g. `Obelaw\Ium\Billing\Models\Invoice`, `FiscalYear`).
- **Reversal Cross-References**: `reversed_entry_id` (points to the original voucher) and `reversal_entry_id` (points to the offsetting reversal voucher).

### `JournalEntryLine` (Table: `ium_accounting_journal_entry_lines`)
- Individual debit or credit line item belonging to a `JournalEntry`.
- **Type**: `PostingSide` (`DEBIT`, `CREDIT`).
- **Amounts**: Stored as integer minor units in transaction currency (`amount`) and functional/reporting currency (`amount_functional`).
- **Presenters & Formatters**:
  - `amountMajor()`, `amountFunctionalMajor()`
  - `formatAmount(?Currency $currency = null)`, `formatAmountFunctional(?Currency $currency = null)`
- **Analytical Tagging**: `cost_center_id` foreign key to `CostCenter`.
- **Subledger Reference**: `party_type` and `party_id` (e.g. Customer, Vendor, Employee).

### `GeneralLedgerPosting` (Table: `ium_accounting_gl_postings`)
- The **immutable book of final entry**.
- Created exclusively by `LedgerService::postEntryToLedger()` during voucher posting.
- Always records debits, credits, and running balance in the **functional currency as integer minor units** (`debit`, `credit`, `balance_after`).
- **Presenters & Formatters**:
  - `debitMajor()`, `creditMajor()`, `balanceAfterMajor()`
  - `formatDebit(?Currency $currency = null)`, `formatCredit(?Currency $currency = null)`, `formatBalanceAfter(?Currency $currency = null)`
- Never updated, never deleted.

### `Currency` (Table: `ium_accounting_currencies`)
- Master ISO 4217 currency registry.
- Fields: `code` (3-char), `name`, `symbol`, `precision` (0–6), `symbol_first`, `decimal_mark`, `thousands_separator`, `is_base` (boolean), `is_active`, `exchange_rate`.
- Only one record can have `is_base = true` at any time.
- **Minor Unit & Sub-Unit Helpers**:
  - `subUnitMultiplier(): int` (e.g. `10^2 = 100`)
  - `toMinorUnits(float|int $amount): int` (e.g. `12.50` $\to$ `1250`)
  - `toMajorUnits(int $minorUnits): float` (e.g. `1250` $\to$ `12.50`)
  - `formatMinor(int $minorUnits): string` (e.g. `"$ 12.50"`)
  - `subUnitName(): string` (`Cents`, `Pence`, `Fils`, `Halalas`, `Piasters`)
  - `subUnitSymbol(): string` (`¢`, `c`, `p`, `pt`)
  - `formatSubUnit(int $subUnits, bool $useSymbol = true): string` (`"50 Cents"` or `"50 ¢"`)
  - `splitUnits(float|int $amount): array` (`['major' => 12, 'minor' => 50, 'formatted' => ... ]`)
  - `formatMajorAndSubUnit(float|int $amount, bool $useSymbol = false): string` (`"$ 12 and 50 Cents"`)

### `Money` (Value Object: `src/ValueObjects/Money.php`)
- Immutable financial value object encapsulating an amount and ISO currency.
- **Factory**: `Money::of(float|int $amount, string $currency)` or `Money::fromMinor(int $minorUnits, string $currency)`.
- **Conversion**: `$money->toMinorUnits(?int $precision = null): int`.
- **Arithmetic**: `add()`, `subtract()`, `multiply()`, `equals()`, `isZero()`, `isPositive()`, `isNegative()`.

### `ExchangeRate` (Table: `ium_accounting_exchange_rates`)
- Historical daily exchange rate snapshots.
- Fields: `currency_id`, `currency_code`, `rate` (multiplier to base currency), `inverse_rate` ($1 / \text{rate}$), `effective_date`, `source` (e.g. `'ecb'`, `'manual'`, `'central_bank'`).
- Indexed by `[currency_code, effective_date]`.

### `FiscalYear` & `FiscalPeriod` (Tables: `ium_accounting_fiscal_years`, `ium_accounting_fiscal_periods`)
- Define the organization's financial accounting calendar.
- Enforce posting permissions based on transaction dates.

### `CostCenter` (Table: `ium_accounting_cost_centers`)
- Multi-dimensional analytical tracking (`COST_CENTER`, `PROJECT`, `DEPARTMENT`).
- Supports self-referential parent-child hierarchy for departmental roll-ups.

### `AccountingAuditLog` (Table: `ium_accounting_audit_logs`)
- Forensic audit trail recording every state transition, approval, reversal, and period lock.
- Captures `actor_id`, `action`, `auditable_type`, `auditable_id`, `before_state`, and `after_state`.

### `EventAccountMapping` (Table: `ium_accounting_event_account_mappings`)
- Configures how external domain events map to debit and credit account codes in the Anti-Corruption Layer.

---

## 4. Multi-Currency Engine Architecture

```
+-------------------------------------------------------------------------+
|                        CurrencyService Gateway                          |
+--------------------+-------------------------------+--------------------+
                     |                               |
                     v                               v
         +-----------------------+       +-----------------------+
         |   ISO 4217 Registry   |       | Exchange Rate History |
         |   (Currency Model)    |       | (ExchangeRate Model)  |
         +-----------+-----------+       +-----------+-----------+
                     |                               |
                     +---------------+---------------+
                                     |
                                     v
                 +---------------------------------------+
                 |    Rate Resolution & Conversion Engine|
                 |  getRate($from, $to, $effectiveDate)  |
                 +-------------------+-------------------+
                                     |
          +--------------------------+--------------------------+
          |                                                     |
          v                                                     v
+-----------------------------+               +-----------------------------+
|    Direct / Inverse Rates   |               |   Cross-Rate Triangulation  |
|  $to === BASE: amount * rate|               |   $from -> BASE -> $to      |
|  $from === BASE: amount/rate|               |   rate = fromRate / toRate  |
+-----------------------------+               +-----------------------------+
```

### Rate Resolution Algorithm
When converting from Currency $A$ to Currency $B$ on date $D$:
1. If $A = B$, rate is `1.0`.
2. Look up the most recent effective rate where $\text{effective\_date} \le D$ ordered by `effective_date DESC`.
3. If $B$ is the base functional currency:
   $$\text{Converted Amount} = \text{Amount} \times \text{Rate}_A$$
4. If $A$ is the base functional currency:
   $$\text{Converted Amount} = \text{Amount} / \text{Rate}_B = \text{Amount} \times \text{InverseRate}_B$$
5. If neither is base (Cross-Currency Triangulation):
   $$\text{Cross Rate} = \frac{\text{Rate}_A}{\text{Rate}_B}$$
   $$\text{Converted Amount} = \text{Amount} \times \text{Cross Rate}$$
6. Converted amount is rounded to target currency's precision (`$toCurrency->precision`).

---

## 5. Voucher Lifecycle & State Machine

```
                   [ Create Draft ]
                          |
                          v
                      +-------+
                      | DRAFT | <----------------+
                      +-------+                  |
                          |                      |
                   submit() (Zero-Sum)           | reject(reason)
                          |                      |
                          v                      |
                +------------------+             |
                | PENDING_APPROVAL | ------------+
                +------------------+
                          |
                   approve() (Maker-Checker)
                          |
                          v
                     +----------+
                     | APPROVED |
                     +----------+
                          |
                    post() (GL Commit & Balance Update)
                          |
                          v
                      +--------+
                      | POSTED | (IMMUTABLE)
                      +--------+
                          |
                    reverse() (Storno)
                          |
                          v
                     +----------+
                     | REVERSED | (Points to new REV voucher)
                     +----------+
```

---

## 6. Normal Balance & Balance Update Arithmetic

When a journal entry is posted to the General Ledger, the running balance for an account is adjusted according to its `NormalBalance`:

| Account Type | Normal Balance | Balance Formula on Posting |
| :--- | :--- | :--- |
| **Asset** | `DEBIT` | $\text{New Balance} = \text{Old Balance} + (\text{Debit} - \text{Credit})$ |
| **Expense** | `DEBIT` | $\text{New Balance} = \text{Old Balance} + (\text{Debit} - \text{Credit})$ |
| **Liability** | `CREDIT` | $\text{New Balance} = \text{Old Balance} + (\text{Credit} - \text{Debit})$ |
| **Equity** | `CREDIT` | $\text{New Balance} = \text{Old Balance} + (\text{Credit} - \text{Debit})$ |
| **Revenue** | `CREDIT` | $\text{New Balance} = \text{Old Balance} + (\text{Credit} - \text{Debit})$ |

---

## 7. Financial Reporting Equations

### Trial Balance
$$\sum \text{Debit Balances} = \sum \text{Credit Balances}$$

### Balance Sheet
$$\text{Total Assets} = \text{Total Liabilities} + \text{Total Equity}$$

### Income Statement (Profit & Loss)
$$\text{Net Income} = \text{Total Revenue} - \text{Total Expense}$$

In fiscal year-end closing, if $\text{Net Income} > 0$ (profit), Retained Earnings is **credited**. If $\text{Net Income} < 0$ (loss), Retained Earnings is **debited**, and all nominal accounts (Revenue & Expense) are zeroed out.

---

## 8. Console Commands & Demo Data Seeder Architecture

The accounting domain exposes artisan tooling to provision and seed realistic accounting environments:

### `SeedAccountingDemoCommand` (`ium:accounting:demo-seed`)
- **Signature**: `ium:accounting:demo-seed {--fresh : Wipe existing accounting domain tables} {--force : Bypass confirmation}`
- **Execution Class**: [`SeedAccountingDemoCommand`](file:///E:/projects/obelaw/ium/packages/accounting/src/Console/Commands/SeedAccountingDemoCommand.php)
- **Seeder Engine**: [`AccountingDemoSeeder`](file:///E:/projects/obelaw/ium/packages/accounting/src/Demo/AccountingDemoSeeder.php)
- **Pipeline Architecture**:
  1. `seedCurrencies()`: Seeds standard ISO 4217 currencies and historical FX rate snapshots.
  2. `seedFiscalCalendar()`: Seeds Fiscal Year 2026 and 12 monthly open fiscal periods.
  3. `seedCostCenters()`: Seeds 6 analytical cost centers across key enterprise functions.
  4. `seedChartOfAccounts()`: Seeds 4-tier hierarchical COA tree (Assets, Liabilities, Equity, Revenue, Expense).
  5. `seedEventMappings()`: Seeds ACL event mapping routes for cross-domain modules.
  6. `seedTransactions()`: Drafts, validates, approves, and posts 8 realistic multi-currency transactions.
  7. **Parity Verification**: Validates Trial Balance parity and Balance Sheet equation before completing.
