# Tachles VC — Entity Reference

## Entities

| Entity | Type | Entity ID | Reconcile URL |
|--------|------|-----------|---------------|
| Tachles VC Management, LLC | Management Company (ManCo) | `PKLo5B82` | `https://tachles.decilehub.com/firm_admin/entities/PKLo5B82/accounting/reconcile` |
| Tachles VC GP, LLC | General Partner | `6NQ9lxNr` | `https://tachles.decilehub.com/firm_admin/entities/6NQ9lxNr/accounting/reconcile` |
| Tachles VC Fund I, LP | Fund | `6N0rRo8y` | `https://tachles.decilehub.com/firm_admin/entities/6N0rRo8y/accounting/reconcile` |

**Slug:** `tachles`

## Bank Accounts

- **ManCo default:** Mercury Checking ••5474

## ManCo Reconciliation Notes

- Standard ManCo expense rules apply (see main SKILL.md `### ✅ ManCo Expenses` section)
- Transaction type for all ManCo expenses: `admin_fee`
- $75 IRS receipt cap applies — do not reconcile any expense ≥ $75 without a receipt on file
- Inbound management fee payments from Tachles VC Fund I, LP → GL 163 (1550 - Management Fee Receivable)
