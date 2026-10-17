# Obelawium Cross-Domain Accounting Integration Guide

This guide explains how external Obelawium domain modules (e.g., `billing`, `inventory`, `purchasing`, `payroll`, `crm`) safely integrate with the Accounting module (`obelaw/ium-accounting`).

---

## 1. Domain Boundary Principles

1. **Complete Database Isolation**: External modules must **NEVER** write to `ium_accounting_*` tables, execute direct SQL queries against them, or call Eloquent model methods directly (e.g. `Account::create()`, `JournalEntry::create()`).
2. **Gateway-Only Interactivity**: All interactions must flow through the central `ium()->accounting()` gateway or injected domain services (`AclIngestionService`, `JournalEntryService`, `CurrencyService`).
3. **Anti-Corruption Layer (ACL)**: Use the ACL to translate upstream domain concepts (Invoices, Purchase Orders, Stock Adjustments, Payroll Slips) into double-entry accounting records without contaminating either domain's vocabulary.

---

## 2. Integration Patterns

### Pattern A: Standard 2-Leg Ingestion via `AclIngestionService` (Recommended for Simple Events)

Used when an operational event has a clear debit and credit leg (e.g., Customer Payment Received, Cash Disbursement).

```php
use Obelaw\Ium\Accounting\Data\AclIngestEventDto;

public function handlePaymentReceived(Payment $payment): void
{
    ium()->accounting()->acl()->ingest(new AclIngestEventDto(
        eventName: 'billing.payment.received',
        documentReference: $payment->reference_number,
        transactionDate: $payment->received_at->format('Y-m-d'),
        amount: $payment->amount,
        currency: $payment->currency,
        debitAccountCode: '1010',  // Cash / Bank
        creditAccountCode: '1200', // Accounts Receivable
        sourceType: Payment::class,
        sourceId: $payment->id,
        actorId: auth()->id() ?? 'system-billing',
        autoPost: true,
        metadata: [
            'payment_method' => $payment->method,
            'customer_id' => $payment->customer_id,
        ]
    ));
}
```

---

### Pattern B: Multi-Currency Event Ingestion via ACL

When upstream modules bill or pay in foreign currencies, pass the transaction currency. The ACL automatically coordinates with `CurrencyService` to determine the exchange rate and calculate the functional ledger amounts:

```php
use Obelaw\Ium\Accounting\Data\AclIngestEventDto;

public function handleForeignInvoiceIssued(Invoice $invoice): void
{
    // Invoice issued in EUR for a USD-reporting company (5,000.00 EUR = 500,000 cents)
    ium()->accounting()->acl()->ingest(AclIngestEventDto::from([
        'event_name' => 'billing.foreign_invoice.issued',
        'document_reference' => $invoice->number,
        'transaction_date' => $invoice->issue_date->format('Y-m-d'),
        'amount' => 500000,              // 5,000.00 EUR in integer minor units (cents)
        'currency' => 'EUR',
        'debit_account_code' => '1201',  // Accounts Receivable (EUR)
        'credit_account_code' => '4100', // Sales Revenue (USD functional)
        'auto_post' => true,
        'source_type' => Invoice::class,
        'source_id' => $invoice->id,
        'actor_id' => 'billing-worker',
    ]));
}
```

---

### Pattern C: Decoupled Tenant Mapping via `EventAccountMapping`

Allows domain modules to emit events *without hardcoding account codes*. The tenant's finance team configures the mapping in advance.

#### Step 1: Register Mapping (in a module migration, seeder, or admin UI)
```php
ium()->accounting()->acl()->registerMapping(
    eventName: 'inventory.goods.received',
    debitAccountCode: '1300', // Inventory on Hand (Asset)
    creditAccountCode: '2010', // Goods Received Not Invoiced / GRNI (Liability)
    descriptionTemplate: 'Automatic posting for warehouse receipt'
);
```

#### Step 2: Operational Domain Emits Without Account Codes
```php
// Upstream Inventory Service does NOT need to know accounting codes:
ium()->accounting()->acl()->ingest(AclIngestEventDto::from([
    'event_name' => 'inventory.goods.received',
    'document_reference' => 'GRN-2026-901',
    'transaction_date' => now()->format('Y-m-d'),
    'amount' => 4500000, // 45,000.00 in minor units (cents)
    'source_type' => GoodsReceivedNote::class,
    'source_id' => $grn->id,
    'auto_post' => true,
]));
```

---

### Pattern D: Multi-Line Complex Vouchers via `JournalEntryService`

Used when a transaction involves three or more accounts (e.g., Sales Invoices with multiple VAT lines, Payroll runs with taxes and pensions).

```php
use Obelaw\Ium\Accounting\Data\CreateJournalEntryDto;
use Obelaw\Ium\Accounting\Data\JournalEntryLineDto;
use Obelaw\Ium\Accounting\Enums\JournalEntryType;
use Obelaw\Ium\Accounting\Enums\PostingSide;

public function postSalesInvoiceWithTax(Invoice $invoice): void
{
    $arAccount = ium()->accounting()->accounts()->findByCodeOrFail('1200');
    $revenueAccount = ium()->accounting()->accounts()->findByCodeOrFail('4100');
    $vatOutputAccount = ium()->accounting()->accounts()->findByCodeOrFail('2210');

    // Total: $1,150.00 = 115,000 cents (Net $1,000.00 = 100,000 cents + VAT $150.00 = 15,000 cents)
    $lines = [
        // Debit Accounts Receivable (Full Amount including Tax)
        new JournalEntryLineDto(
            accountId: $arAccount->id,
            type: PostingSide::DEBIT,
            amount: 115000,
            partyType: 'customer',
            partyId: $invoice->customer_id,
            narration: "Invoice #{$invoice->number} AR",
        ),
        // Credit Product Sales Revenue (Net Amount)
        new JournalEntryLineDto(
            accountId: $revenueAccount->id,
            type: PostingSide::CREDIT,
            amount: 100000,
            costCenterId: $invoice->department_cost_center_id,
            narration: "Product sales revenue",
        ),
        // Credit Output VAT Liability
        new JournalEntryLineDto(
            accountId: $vatOutputAccount->id,
            type: PostingSide::CREDIT,
            amount: 15000,
            narration: "15% Standard VAT on Sales",
        ),
    ];

    $dto = new CreateJournalEntryDto(
        entryDate: $invoice->issue_date->format('Y-m-d'),
        lines: $lines,
        entryType: JournalEntryType::STANDARD,
        narration: "Commercial Sales Invoice #{$invoice->number}",
        sourceType: Invoice::class,
        sourceId: $invoice->id,
        creatorId: 'billing-daemon',
    );

    $entry = ium()->accounting()->journalEntries()->createDraft($dto);
    ium()->accounting()->journalEntries()->post($entry->id, 'billing-daemon');
}
```

---

## 3. Subledger & Party Tracking

When posting journal lines associated with specific counterparties:
- Set `party_type` (e.g. `'customer'`, `'vendor'`, `'employee'`).
- Set `party_id` (the ID in the operational table).

This enables subledger reconciliation (e.g., verifying that the total balance of all customer subledger accounts matches the General Ledger balance of Account `1200 - Accounts Receivable`).

---

## 4. Subscribing to Accounting Domain Events

External modules can register listeners to accounting events published by `obelaw/ium-accounting`:

| Event Class | Emitted When | Payload | Use Case for Other Domains |
| :--- | :--- | :--- | :--- |
| `JournalEntryPosted` | Voucher committed to General Ledger | `$event->entry`, `$event->actorId` | Update operational document status to 'accounted'. |
| `JournalEntryReversed` | Voucher cancelled via Storno | `$event->originalEntry`, `$event->reversalEntry` | Flag originating invoice/bill as unposted. |
| `FiscalPeriodClosed` | Monthly period is locked | `$event->period`, `$event->actorId` | Prevent back-dated invoices from being created. |
| `FiscalPeriodReopened` | Locked period is reopened | `$event->period`, `$event->actorId`, `$event->reason` | Log audit alert in upstream systems. |
| `FiscalYearClosed` | Year-end closing executed | `$event->fiscalYear`, `$event->closingEntry` | Freeze all operational ledgers for that year. |
| `BaseCurrencyChanged` | Reporting base currency switched | `$event->previousBase`, `$event->newBase` | Re-index product price catalogs or refresh caches. |
| `CurrencyCreated` | New ISO currency registered | `$event->currency`, `$event->actorId` | Enable foreign currency options in UI select inputs. |
| `ExchangeRateUpdated` | Exchange rate updated | `$event->exchangeRate`, `$event->actorId` | Update live currency conversion displays. |
| `AccountBalanceThresholdBreached` | Account exceeds credit limit / min cash | `$event->account`, `$event->currentBalance`, `$event->threshold` | Trigger automated budget freeze or finance email alerts. |
