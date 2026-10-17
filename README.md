# Obelawium Accounting Skill (`ium-accounting-development`)

[![Skill Version](https://img.shields.io/badge/skill-v1.2.0-blue.svg)](./onsynk.json)
[![Domain Package](https://img.shields.io/badge/domain-obelaw%2Fium--accounting-emerald.svg)](file:///E:/projects/obelaw/ium/packages/accounting)
[![Pest Tests](https://img.shields.io/badge/tests-40%20passed-brightgreen.svg)](./references/testing-and-verification.md)
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg)](./onsynk.json)

An AI agent and developer skill for building, integrating, maintaining, and architecting the **Obelawium Accounting (FMS & General Ledger)** domain module (`obelaw/ium-accounting`).

---

## Key Features

- **Fluent Gateway DSL**: Unified API access via `ium()->accounting()->[service]()->[method]()`.
- **9 Specialized Domain Services**: `accounts()`, `journalEntries()`, `ledger()`, `fiscalPeriods()`, `costCenters()`, `closing()`, `audit()`, `acl()`, `currencies()`.
- **Zero-Sum Ledger**: Strict debit/credit integer minor unit parity ($\sum \text{Debits} = \sum \text{Credits}$, $0 imbalance down to single cent).
- **Minor Unit Financial Precision**: Database tables and GL postings store amounts as integer minor units (cents) to eliminate float rounding errors, complete with `fromMajor()` DTO factories and model formatters.
- **Automated Demo Data Seeder**: Turnkey console command `php artisan ium:accounting:demo-seed` and `AccountingDemoSeeder` establishing balanced multi-currency test topologies.
- **Immutability & Storno**: Posted vouchers are immutable; corrections strictly use Storno reversals (`reverse()`).
- **Hierarchical COA**: Parent aggregator nodes with leaf-only direct postings.
- **Maker-Checker Governance**: Segregation of duties preventing self-approval in strict mode.
- **Multi-Currency Engine**: ISO 4217 registry, daily exchange rate snapshots, direct/inverse quotation, cross-rate triangulation, sub-unit converters/formatters, and functional currency General Ledger denomination.
- **Anti-Corruption Layer (ACL)**: Clean cross-domain ingestion (Billing, Inventory, CRM) without direct database coupling.

---

## Quick Example

```php
use Obelaw\Ium\Accounting\Data\CreateJournalEntryDto;
use Obelaw\Ium\Accounting\Data\JournalEntryLineDto;
use Obelaw\Ium\Accounting\Enums\JournalEntryType;
use Obelaw\Ium\Accounting\Enums\PostingSide;

// 1. Create balanced draft voucher ($1,200.00 = 120,000 cents)
$draft = ium()->accounting()->journalEntries()->createDraft(new CreateJournalEntryDto(
    entryType: JournalEntryType::STANDARD,
    entryDate: '2026-03-15',
    currency: 'USD',
    creatorId: 'user-alice',
    lines: [
        new JournalEntryLineDto(accountId: $equipmentAccount->id, type: PostingSide::DEBIT, amount: 120000),
        new JournalEntryLineDto(accountId: $bankAccount->id, type: PostingSide::CREDIT, amount: 120000),
    ],
    narration: 'Office equipment purchase',
));

// 2. Submit, approve (Maker-Checker), and post to GL
$submitted = ium()->accounting()->journalEntries()->submit($draft->id, 'user-alice');
$approved  = ium()->accounting()->journalEntries()->approve($submitted->id, 'user-bob-supervisor');
$posted    = ium()->accounting()->journalEntries()->post($approved->id, 'user-bob-supervisor');
```

---

## Testing & Verification

Execute tests across the package (`packages/accounting`):

```powershell
# Run full Pest test suite
./vendor/bin/pest

# Run with a filter
./vendor/bin/pest --filter="Currency"
```

---

## References & Documentation

| Document | Purpose |
| :--- | :--- |
| [**SKILL.md**](./SKILL.md) | Agent decision matrix, 10-step operational workflow, and diagnostic table. |
| [**Architecture Reference**](./references/architecture.md) | Domain models, 11 `ium_accounting_*` tables, state machines, and multi-currency engine. |
| [**Recipes Cookbook**](./references/recipes.md) | 18 copy-pasteable PHP recipes (COA, vouchers, reversals, reports, currencies, demo seeder, FX). |
| [**Invariants & Exceptions**](./references/invariants-and-exceptions.md) | 10 domain invariants, recovery playbooks, and backed enum catalogs. |
| [**Cross-Domain Integration**](./references/cross-domain-integration.md) | ACL ingestion patterns for external modules (Billing, Inventory, Purchasing). |
| [**Testing & Verification**](./references/testing-and-verification.md) | PestPHP test architecture, TestCase setup, and pre-commit checklist. |

---

## License

Licensed under the [MIT License](./onsynk.json).
