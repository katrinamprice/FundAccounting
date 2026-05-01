---
name: decile-reconciliation
description: >
  Cash reconciliation on Decile Hub (*.decilehub.com). Use this skill whenever a member of fund accounting
  asks to reconcile transactions, work through unreconciled items, or train on a new fund's
  reconcile page. Covers capital call payments, management fees, bank charges, investment wires,
  management company (ManCo) expenses, and how to flag unclear transactions for human review.
  Applies to all Decile Hub entities (Spatial Capital, Capital M, Func Ventures, Prana Tech Ventures,
  TheFounderVC, Begin Ventures, Capacity Capital, Energy Mix Ventures, Jolly VC, Team Ignite,
  Ranchos Ventures, Tachles, Blockchain Builders, Riceberg, and any new funds).
  Trigger on any mention of "reconcile", "reconciliation", "unreconciled transactions", or navigating
  to a /accounting/reconcile URL on decilehub.com.
---

## Session Start — What's New

At the start of every reconciliation session, **before doing anything else**, check for rule updates:

1. **Read** the last-seen commit SHA from: `[workspace folder]/skill_last_commit.txt`
   - If the file doesn't exist, this is a first session — skip to step 4.

2. **Fetch recent commits** for this skill file:
   ```
   GET https://api.github.com/repos/katrinamprice/FundAccounting/commits?path=SKILL.md&per_page=20
   Authorization: token [GITHUB_WRITE_TOKEN if set, otherwise omit]
   ```

3. **Compare** the latest commit SHA to the stored SHA. Collect every commit that is newer
   (i.e., every commit between the stored SHA and the current HEAD, exclusive of the stored one).

4. **If there are new commits**, display a "What's New" banner before proceeding:

   > 📋 **Skill updates since your last session:**
   > - [commit message] — [date]
   >
   > *(Tip: ask "show me what changed in the [rule name] rule" for details)*

5. **Update** `skill_last_commit.txt` in the workspace folder with the current HEAD SHA.
   Use the Write tool to save it. If the file doesn't exist yet, create it.

6. If there are no new commits, continue silently — no banner needed.

### Commit message convention (for whoever updates the skill)

To make the "What's New" summary meaningful, all skill updates pushed via Claude should use
descriptive commit messages in this format:

- `Add rule: [description] — by [your name]`
- `Update rule: [description] — by [your name]`
- `Remove rule: [description] — by [your name]`
- `Fix: [description] — by [your name]`

**Examples:**
- `Add rule: IO AUTOPAY rows require GP info before reconciling — by Katrina`
- `Update rule: Spicer Jeffries $75 threshold exception — by Marcus`
- `Add rule: New fund Dash VC entity IDs — by Katrina`

This way the "What's New" banner is self-explanatory to any team member starting a session.

---


# Decile Hub Cash Reconciliation

## Overview

Every unreconciled bank transaction needs four fields filled before clicking Reconcile:

1. **Transaction Type** — the category of transaction
2. **What** — the GL account (debit/credit)
3. **Who** — the counterparty (LP, portfolio company, or organization)
4. **Why** — a short description/reason

The UI uses TomSelect dropdowns. Native select manipulation doesn't work — always use the TomSelect API and fire events (see Technical section below).

**⛔ HARD RULE — $75 IRS receipt threshold (applies to ALL entities — fund AND ManCo):** The IRS requires a receipt for any business expense of $75 or more ([IRS $75 receipt rule](https://www.fylehq.com/blog/irs-75-dollar-receipt-rule)). Do **not** reconcile any outgoing expense transaction at or above $75 unless a receipt is already on file. This applies to every entity type — fund LPs, management companies, GP entities, and any other Hub entity. Leave transactions at or above $75 unreconciled and note them for a member of fund accounting to attach the receipt first. Transactions strictly under $75 may be reconciled without a receipt.

**Exceptions — vendors with invoices already on file (reconcile at any amount, no receipt needed):**
- **VC Lab / Decile Group / Decile Capital** — Decile is our company and invoices are always on file. Reconcile regardless of amount.
- **Spicer Jeffries / Cherry Bekaert** — tax prep invoices are always on file. Reconcile regardless of amount.
- **Bank fees / wire fees / foreign exchange fees** — these are documented on the bank statement itself. The IRS $75 rule applies to meals, travel, and entertainment; bank-initiated charges do not require a separate receipt. Reconcile at any amount.

*Note: The $75 threshold primarily affects ManCo operating expenses (SaaS, meals, travel) where receipts aren't automatically retained. Capital calls (inbound), management fees, and investment wires always have supporting documentation and are reconciled normally regardless of amount.*

---

## Final Step: Mark as Reconciled

When the queue is empty **and there are no remaining transactions that need manual review by a member of fund accounting**, click the "Set as Reconciled" button to close out the reconciliation period. This is a two-step process — the first click opens a confirmation modal, the second confirms it:

```javascript
// Step 1: Click the page-level button to open the confirmation modal
const btn = Array.from(document.querySelectorAll('button')).find(b => b.textContent.trim().includes('Set as Reconciled'));
btn?.click();

// Step 2: Confirm in the modal (run after modal appears)
const confirmBtn = Array.from(document.querySelectorAll('button')).find(b => b.textContent.trim() === 'Set as Reconciled');
confirmBtn?.click();
```

A "Reconciliation status updated" toast confirms success.

**Only do this if ONE of the following is true:**
- The queue is fully empty (no rows visible), OR
- Every remaining row has a bottom-right flag badge — Hub is already tracking those, so the period is effectively closed from a manual reconciliation standpoint

**Bottom-right badges that count as "flagged" for this purpose:**
- `INVESTMENT DOCS OPEN`
- `MISSING DOCS` (any variant)
- `ENTITY MISMATCH OPEN`
- Any other badge appearing in the bottom-right corner of a transaction card

**Detection snippet — use before deciding whether to press Set as Reconciled:**
```javascript
const rows = Array.from(document.querySelectorAll('[class*="mb-3"][class*="grid-cols-16"]'));
const unflagged = rows.filter(r => {
  const t = r.innerText;
  return !t.includes('INVESTMENT DOCS') && !t.includes('MISSING DOCS') && !t.includes('MISMATCH') && !t.includes('FLAGGED');
});
`Unflagged rows remaining: ${unflagged.length}`;
// If 0 → safe to press Set as Reconciled
```

If any rows are unreconciled and have NO badge, leave the period open and note what's outstanding.

---

## Step 1: Survey the Queue

Before reconciling anything, scroll to the bottom of the page to force lazy-load all rows, then survey:

```javascript
// First: scroll to load all rows
window.scrollTo(0, document.body.scrollHeight);
```

Then after a moment:

```javascript
const rows = Array.from(document.querySelectorAll('[class*="mb-3"][class*="grid-cols-16"]'));
const all = [];
rows.forEach((row, i) => {
  const lines = row.innerText.split('\n').map(l => l.trim()).filter(l => l.length > 0);
  const t = row.innerText;
  const flagged = t.includes('FLAGGED') ? ' ⛳ FLAGGED' : '';
  const mismatch = t.includes('MISMATCH') ? ' 🟡 MISMATCH' : '';
  const bt = lines.some(l => l.includes('Bank type:')) ? ' [BT]' : '';
  const investDocs = t.includes('INVESTMENT DOCS') ? ' 📄 INVEST-DOCS' : '';
  const missingDocs = t.includes('MISSING DOCS') ? ' 📄 MISSING-DOCS' : '';
  all.push(`[${i}] ${lines.slice(0, 4).join(' | ')}${flagged}${mismatch}${bt}${investDocs}${missingDocs}`);
});
`Total rows: ${rows.length}\n` + all.join('\n');
```

The survey output will append badges to each row. **Never reconcile any row marked FLAGGED, MISMATCH, INVEST-DOCS, or MISSING-DOCS** — these are all Hub-tracked and should be left untouched.

**Hub flag badges — skip and exclude from exception sheet:**
- `FLAGGED: MISSING TRUST DOCS` / `FLAGGED: ENTITY` / any red FLAGGED badge → skip entirely, leave untouched
- `ENTITY MISMATCH OPEN` (yellow badge at bottom of transaction card) → skip entirely; Hub is already tracking it, do not add to exception sheet
- `INVESTMENT DOCS OPEN` (bottom-right badge) → skip entirely; Hub is tracking it
- `MISSING DOCS` (bottom-right badge) → skip entirely; Hub is tracking it

**Check row[0] before assuming it's a header.** Some funds (e.g., Spatial Capital) have a Bank Balances header at row[0] that should be skipped. Others (e.g., TheFounderVC) start transactions at row[0] with no header. Read the output — if row[0] contains a date and dollar amount, it's a real transaction. If it contains "Bank Balance" or similar, skip it.

**⛔ HARD RULE — Flagged transactions:** If a transaction has ANY flag badge (e.g., `FLAGGED: MISSING TRUST DOCS`, `FLAGGED: ENTITY`, `INVESTMENT DOCS OPEN`, or any other flag), do NOT reconcile it. Leave it completely untouched. This applies regardless of how straightforward the match appears. Always scan for flag badges before reconciling any row.

**Note:** The new UI renders rows with compound class strings. Use `[class*="mb-3"][class*="grid-cols-16"]` as the row selector. If that returns nothing, fall back to `.mb-3` (older UI layout). Take a screenshot to assess the layout if needed.

**Important:** Row indices shift every time a transaction is reconciled. Always re-query fresh before each operation rather than caching the list.

---

## Step 2: Match LPs to Capital Accounts

Before reconciling capital calls, pull the full Who dropdown to match LP names from bank descriptions to their `ca_xxx` capital account IDs:

```javascript
const rows = Array.from(document.querySelectorAll('[class*="mb-3"][class*="grid-cols-16"]'));
const row = rows[0]; // any transaction row will do (adjust index if row[0] is a header)
const whoTs = row.querySelectorAll('select')[2].tomselect;
const opts = Object.values(whoTs.options);
// Filter to capital accounts only
const caAccounts = opts.filter(o => o.value.startsWith('ca_'));
JSON.stringify(caAccounts.map(o => ({ value: o.value, text: o.text })));
```

Bank descriptions are often abbreviated (e.g., "THAI TA" → ca_494 Thai Ta, "YDNTRUST" → ca_492 YDNUB Discretionary Trust). Search the options list for partial matches.

**Partial name matches:** A partial match is acceptable when the match is clearly the same person/entity with no ambiguity. For example, "Andre Moura" in the dropdown is close enough to reconcile a wire from "Andre Chambel Croft Moura". When in doubt, ask a member of fund accounting.

---

## Bank Type (BT) Rows

Mercury auto-categorizes some transactions and creates a duplicate "Bank type:" labeled row alongside the clean transaction row.

**How to identify a BT row:** Survey output will show `[BT]` suffix, or the row's text contains `Bank type:`.

**Rule:**
- **If only the BT row exists** (no clean version of the same transaction) → reconcile the BT row normally.
- **If both a BT row and a clean row exist for the same transaction** → reconcile the clean row only. Skip the BT row entirely to avoid double-counting.

To check: look for two rows with the same merchant, amount, and date — one labeled "Bank type:" and one without. If they're the same transaction, the BT is the duplicate.

**After the clean row is reconciled,** the BT duplicate will remain in the queue but should be left unreconciled. It is not a new transaction — it's Mercury's auto-categorization of the same charge.

---

## Transaction Types

### ✅ Capital Call Payment (FULLY TRAINED)

**When:** Inbound wire from an LP. Amount is positive. Description contains LP name or reference.

```javascript
function setAndFire(sel, value) {
  sel.tomselect.setValue(value);
  sel.dispatchEvent(new Event('change', { bubbles: true }));
  sel.dispatchEvent(new Event('input', { bubbles: true }));
}

function reconcileCapitalCall(rowIndex, caId) {
  const rows = Array.from(document.querySelectorAll('[class*="mb-3"][class*="grid-cols-16"]'));
  const row = rows[rowIndex];
  if (!row) return `Row ${rowIndex} not found`;
  const sel = row.querySelectorAll('select');
  setAndFire(sel[0], 'capital_call_payment');
  setAndFire(sel[1], 123);       // 1200 - Capital Call Receivable - LP
  setAndFire(sel[2], caId);      // e.g. 'ca_494' for Thai Ta
  setAndFire(sel[3], 'Capital call payment');
  const btn = Array.from(row.querySelectorAll('button')).find(b => b.textContent.includes('Reconcile') && !b.textContent.includes('Snooze'));
  if (!btn?.disabled) btn?.click();
  return `Row ${rowIndex} (${caId}) | btn disabled: ${btn?.disabled}`;
}
```

**Process for a batch of capital calls:** Work from the highest row index to the lowest so that reconciling one row doesn't shift the indices of the others you haven't reached yet. Re-query after each one to confirm the next row's index.

**Capital account ID format:** `ca_XXXX` — get these from the Who dropdown (see Step 2 above). Never guess an ID.

**⛔ HARD RULE — Money Due must match wire amount:** Before reconciling any capital call, check the **Money Due** field in the left panel of the transaction. If Money Due is blank ("-") or shows a different dollar amount than the wire, do **not** reconcile. A blank or $0 Money Due means the LP has already paid in full (or the capital account is overfunded) — reconciling the wire would double-count the payment. Leave unreconciled and flag for a member of fund accounting.

*Example: Wire is $10,000 but Money Due shows "-" (i.e., $0) → do not reconcile.*

**⛔ HARD RULE — Trust vs. individual wire mismatch:** If the capital account is a trust (e.g., "Edward H Wright Trust") but the wire description shows an individual name (e.g., "EDWARD HENNEN WRIGHT" with no "Trust"), do NOT reconcile. The wire must come from the trust account itself, not an individual account. This is a wire compliance requirement. Leave unreconciled and let a member of fund accounting review.

**⛔ HARD RULE — Partial name matches:** If a wire description partially overlaps a capital account name but key identifiers differ (e.g., wire says "DONNA LOUISE MCELRATH THOMAS CHRIST" and the only McElrath account is "The Ken McElrath 2018 Trust" — different first name), do NOT reconcile. Leave unreconciled and flag for a member of fund accounting. A match must be clear and unambiguous.

---

### ✅ Management Fee Payment (WELL TRAINED)

**When:** Outbound wire/e-Transfer to the fund's management company. Amount is negative. Description typically says "e-Transfer sent — [Management Company Name]" or "FUNDS TRANSFER TO DEP ... FROM MANAGEMENT FEES".

```javascript
function reconcileMgmtFee(rowIndex, whoId) {
  const rows = Array.from(document.querySelectorAll('[class*="mb-3"][class*="grid-cols-16"]'));
  const row = rows[rowIndex];
  if (!row) return `Row ${rowIndex} not found`;
  const sel = row.querySelectorAll('select');
  setAndFire(sel[0], 'management_fee');
  setAndFire(sel[1], 219);       // 2205 - Management Fee Payable
  setAndFire(sel[2], whoId);     // org_xxx for the management company
  setAndFire(sel[3], 'Fee payment');
  const btn = Array.from(row.querySelectorAll('button')).find(b => b.textContent.includes('Reconcile') && !b.textContent.includes('Snooze'));
  if (!btn?.disabled) btn?.click();
  return `Row ${rowIndex} (${whoId}) | btn disabled: ${btn?.disabled}`;
}
```

**Who ID:** Use `org_xxx` (organization), not `ca_xxx`. Find the management company in the Who dropdown by searching for the fund's GP/management entity name.

**Known management companies:**
- Capital M Fund I → org_158884 (Capital M Ventures, Inc.)
- Func Ventures Fund I → org_159120 (Func Ventures, LLC)

**Counterparty Type modal:** The first time you reconcile against a new `org_xxx` Who, a modal "Select Counterparty Type" appears. Fill in the entity name and select "Organization", then submit. This only happens once per org.

---

### ✅ Fund Administration Fee — Vc Lab / Decile Group (TRAINED)

**When:** Outbound payment to "Vc Lab", "VC LAB INC", "Decile Group", or "Decile Capital". All are the same entity.

- **Transaction Type:** `admin_fee`
- **What:** 369 (6120 - Fund Administration Fees) — use this almost always
- **Who:** Search the Who dropdown for "decile" or "vc lab" and use whichever match appears. The org ID varies by fund — `org_177845` is common but not universal. Always verify the ID is accepted before clicking Reconcile.
- **Why:** `Fee payment`

**Exception — organizational costs:** If the payment is large ($30,000–$50,000) AND the fund appears to be new, it may be organizational/formation costs rather than ongoing admin fees. Use a different GL account in that case.

**⛔ HARD RULE:** If any single Vc Lab / Decile Group payment is **over $10,000**, stop and check in with a member of fund accounting before reconciling.

**$75 receipt rule does NOT apply** to VC Lab / Decile Group payments — invoices are always on file. Reconcile at any amount.

---

### ✅ Bank Fee / Charge (TRAINED)

**When:** Small outbound charge from the bank itself. Description typically says "Bank Service Fee", "Monthly Fee", "Intl. Wire Fee", "Foreign Exchange Fee", or similar. Amount is small and negative.

```javascript
setAndFire(sel[0], 'admin_fee');
setAndFire(sel[1], 394);         // 7000 - Bank Charges
setAndFire(sel[2], 'org_2436'); // RBC — confirm org ID per fund
setAndFire(sel[3], 'Other');
// After firing events, fill in the reason text field that appears:
const otherInput = row.querySelector('#other_reason_input');
otherInput.value = 'Bank fees';
otherInput.dispatchEvent(new Event('input', { bubbles: true }));
```

Also applies to: international wire fees, foreign exchange fees.

**Mercury bank org IDs:** vary by fund — look up in the Who dropdown by searching "mercury". Common: `org_159392`.

**$75 receipt rule does NOT apply to bank fees** — see Exceptions above. Reconcile at any amount.

---

### ✅ Tax Prep / Spicer Jeffries (TRAINED)

**When:** Outbound payment to a tax/accounting firm (e.g., Spicer Jeffries / SJ Spicer Jeffries LLP / Cherry Bekaert).

**Spicer Jeffries and Cherry Bekaert are the same entity** — treat all payments to either name identically.

**Before reconciling — check the GL first:**
1. Navigate to `https://[slug].decilehub.com/firm_admin/entities/[ENTITY_ID]/accounting/general_ledger`
2. Look for a matching accrual in account 2120 - Tax Accrual (What ID 206) with the same amount as the payment.

**Then apply the following rules:**
- **If a matching Tax Accrual exists** → use **206 (2120 - Tax Accrual)**
- **If no matching accrual AND amount ≥ $5,000** → use **6100 (Audit Fees)**
- **If no matching accrual AND amount < $5,000** → use the Tax Prep Fees account

- **Transaction Type:** `admin_fee`
- **Who:** `org_376714` (Spicer Jeffries / Cherry Bekaert) or look up the org in the Who dropdown
- **Why:** `Fee payment`
- **$75 receipt rule does NOT apply** — tax prep invoices are always on file. Reconcile at any amount.
- **1099 Reportable Transaction:** ✅ **Always check this box** for all Spicer Jeffries and Cherry Bekaert payments — ⚠️ **requires account admin access**. The reconciliation will save but checking the 1099 box will return "Only account admins are allowed to do that." a member of fund accounting must check this manually after reconciling.

---

### ✅ ManCo Expenses (TRAINED — management company entities only)

Management company (ManCo) entities receive SaaS subscriptions, software tools, meals, travel, and other operating expenses. Apply these rules when reconciling a ManCo reconcile page.

**⛔ HARD RULE — $75 cap (IRS receipt rule):** Do **not** reconcile any ManCo expense over $75. The IRS requires a receipt for any business expense of $75 or more — see the [IRS $75 receipt rule](https://www.fylehq.com/blog/irs-75-dollar-receipt-rule). Transactions at or above $75 must have a receipt on file before reconciling. Leave them unreconciled and note them for a member of fund accounting to attach the receipt first.

**⛔ HARD RULE — Credit card autopay rows (IO AUTOPAY / Mercury Credit):** Do **not** reconcile rows that appear to be credit card autopay payments (descriptions like "IO AUTOPAY", "Mercury Credit", or similar). These require information from the GP to categorize correctly. Leave them unreconciled and flag for a member of fund accounting with a note that GP info is needed.

**Categorization rules by merchant type:**

| Merchant category | Examples | GL What ID |
|---|---|---|
| SaaS / Software | Notion, Linear, Webflow, Calendly, Xero, GoDaddy, LinkedIn (subscription), Perplexity, MacPaw/CleanMyMac, OpenAI, Adobe, Anthropic, Canva, Freepik, RocketReach, Apollo.io, ElevenLabs, Attio, Replit | 399 |
| Travel | Flights, hotels, Airbnb, Lyft, Uber, Clipper | 500 |
| Meals / Food | Restaurants, cafes, food delivery, CVS, Walgreens, grocery/convenience stores (snacks while traveling) | 418 |
| Bank charges / wire fees | Bank Service Fee, Intl. Wire Fee, Foreign Exchange Fee | 394 |

**Software vendor refund pattern (e.g., Attio):** Occasionally a vendor charges AND refunds in the same period, producing a matching positive and negative transaction. Reconcile both to GL 399:
- **Outbound payment:** `admin_fee`, GL 399, vendor org, Why = `Fee payment`
- **Inbound refund:** `other`, GL 399, vendor org, Why = `Correction`

**Transaction Type for all ManCo expenses:** `admin_fee` (exception: Riceberg uses `other` — see `funds/riceberg.md`)
**Why field:** `Fee payment` (exception: Riceberg uses `Other` with merchant name written in)

**Batch set pattern for ManCo expenses:**

```javascript
function setAndFire(sel, value) {
  sel.tomselect.setValue(value);
  sel.dispatchEvent(new Event('change', { bubbles: true }));
  sel.dispatchEvent(new Event('input', { bubbles: true }));
}
function findWho(whoSel, terms) {
  const opts = Object.entries(whoSel.tomselect.options);
  for (const t of terms) {
    const m = opts.find(([k,v]) => v.text && v.text.toLowerCase().includes(t.toLowerCase()));
    if (m) return m[0];
  }
  return null;
}

window.scrollTo(0, document.body.scrollHeight); // lazy-load all rows first
// (wait a moment, then run:)
const rows = Array.from(document.querySelectorAll('[class*="mb-3"][class*="grid-cols-16"]'));
const results = [];
for (const row of rows) {
  const text = row.innerText.split('\n').filter(l => l.trim());
  const amtRaw = text.find(l => l.match(/\$[\d,]+\.\d{2}/));
  const amt = amtRaw ? parseFloat(amtRaw.replace(/[$,()]/g,'')) * (amtRaw.includes('(') ? -1 : 1) : null;
  // Skip: BT rows (when clean version exists), positive amounts, amounts >= $75 (IRS receipt threshold), unfilled rows
  if (text.some(l => l.includes('Bank type:')) || amt === null || amt >= 0 || Math.abs(amt) >= 75) continue;

  const desc = text.join(' ').toLowerCase();
  const sel = row.querySelectorAll('select');

  // Categorize by keyword
  let glId, searchTerms;
  if (desc.includes('bank service') || desc.includes('wire fee') || desc.includes('foreign exchange') || desc.includes('monthly fee')) {
    glId = 394; searchTerms = ['mercury', 'rbc', 'bank'];
  } else if (desc.includes('flight') || desc.includes('hotel') || desc.includes('airbnb') || desc.includes('lyft') || desc.includes('uber')) {
    glId = 500; searchTerms = [];
  } else if (desc.includes('restaurant') || desc.includes('tea') || desc.includes('food') || desc.includes('coffee')) {
    glId = 418; searchTerms = [];
  } else {
    glId = 399; // default: software
    searchTerms = text.slice(0,2).join(' ').toLowerCase().split(/\s+/).filter(w => w.length > 3);
  }

  // MacPaw is sold via Paddle payment processor — search 'paddle' not 'macpaw'
  if (desc.includes('macpaw') || desc.includes('cleanmymac')) searchTerms = ['paddle'];

  const whoId = findWho(sel[2], searchTerms);
  if (!whoId) { results.push(`SKIP (no who match): ${text.slice(0,2).join(' ')}`); continue; }

  setAndFire(sel[0], 'admin_fee');
  setAndFire(sel[1], glId);
  setAndFire(sel[2], whoId);
  setAndFire(sel[3], 'Fee payment');
  results.push(`SET: ${text.slice(0,2).join(' ')} → GL ${glId}, who ${whoId}`);
}
results.join('\n');
```

**If vendor not in Who dropdown:** Skip the row (leave unreconciled). Do not force a match. Note the vendor name for a member of fund accounting.

**Special case — MacPaw / CleanMyMac:** MacPaw products are sold through Paddle as payment processor. When you see MacPaw or CleanMyMac in the description, search for "paddle" (not "macpaw") in the Who dropdown → org_1457025 (Paddle.net).

---

### ⚠️ Investment Wire (PARTIALLY TRAINED — confirm with a member of fund accounting)

**When:** Outbound wire to a portfolio company. Amount is large and negative. Description references a company name.

- **Transaction Type:** `investment_wire`
- **What:** 111 (1100 - Investment)
- **Who:** `pc_xxx` (portfolio company) — must already exist in the system
- After clicking Reconcile, a modal appears: **"Associate funding round with investment"** — select the matching funding round

**Before reconciling:**
1. Verify the portfolio company exists in the Who dropdown (search by name)
2. If it doesn't exist: go to Investments → Schedule of Investments → "Add a Company" → find or create the org → Continue. If the org doesn't exist either, stop and ask a member of fund accounting to set it up (requires creating org + portfolio company + funding round).
3. Verify a signed investment agreement exists in the fund's data room with a matching amount

**Adding a portfolio company (when org exists):**
- Navigate to `/firm_admin/entities/[ENTITY_ID]/portfolio_companies`
- Click "Add a Company" — a modal opens (if it renders off-screen, resize browser to 1920px wide)
- Search for the existing org in the Organization dropdown and select it
- Click Continue — it's now available as `pc_xxx` in the Who dropdown

---

### ✅ Bank Interest / Cashback (TRAINED)

**When:** Small inbound credit from the bank. Description typically says "Mercury IO Cashback", "Interest Earned", or similar. Amount is small and positive.

```javascript
function reconcileCashback(rowIndex) {
  const rows = Array.from(document.querySelectorAll('[class*="mb-3"][class*="grid-cols-16"]'));
  const row = rows[rowIndex];
  if (!row) return `Row ${rowIndex} not found`;
  const sel = row.querySelectorAll('select');
  setAndFire(sel[0], 'other');
  setAndFire(sel[1], 316);           // 4001 - Bank Interest Income
  setAndFire(sel[2], 'org_159392'); // Mercury Bank (confirm org ID per fund)
  setAndFire(sel[3], 'Other');
  const otherInput = row.querySelector('#other_reason_input');
  if (otherInput) {
    otherInput.value = 'Mercury IO Cashback'; // or 'Bank interest income'
    otherInput.dispatchEvent(new Event('input', { bubbles: true }));
  }
  const btn = Array.from(row.querySelectorAll('button')).find(b => b.textContent.includes('Reconcile') && !b.textContent.includes('Snooze'));
  if (!btn?.disabled) btn?.click();
  return `Row ${rowIndex} | btn disabled: ${btn?.disabled}`;
}
```

- **Transaction Type:** `other`
- **What:** 316 (4001 - Bank Interest Income)
- **Who:** The bank org (e.g., `org_159392` for Mercury Bank — confirm per fund)
- **Why:** Other → fill in reason field with "Mercury IO Cashback" or "Bank interest income"

---

### ✅ Mercury Treasury Sweep (TRAINED)

**When:** Outbound transfer described simply as "Treasury" — Mercury moves funds from the active checking account into a Mercury Treasury yield/reserve account. Amount is negative.

**How to identify:** The bank description is literally just "Treasury" with no other detail. To confirm which GL account maps to "Treasury" for a given fund, check the GL Descriptors page:
`https://[slug].decilehub.com/firm_admin/entities/[ENTITY_ID]/accounting_account_organization_details`
Search the page for "Treasury" — it will show something like `1002 - Bank - Additional Reserve Account  Mercury Treasury`.

- **Transaction Type:** `other`
- **What:** 562 (1002 - Bank - Additional Reserve Account)
- **Who:** Search Who dropdown for "mercury" → use **"Mercury (Mercury)"** org — NOT "Mercury Partners" which is an LP. Confirm with: `Object.values(sel[2].tomselect.options).filter(o => o.text && o.text.toLowerCase().includes('mercury'))`
- **Why:** Other → fill in reason field: `Mercury Treasury sweep`

```javascript
function setAndFire(sel, value) {
  sel.tomselect.setValue(value);
  sel.dispatchEvent(new Event('change', { bubbles: true }));
  sel.dispatchEvent(new Event('input', { bubbles: true }));
}

const rows = Array.from(document.querySelectorAll('[class*="mb-3"][class*="grid-cols-16"]'));
const row = rows[0]; // adjust index as needed
const sel = row.querySelectorAll('select');
setAndFire(sel[0], 'other');
setAndFire(sel[1], 562);
setAndFire(sel[2], 'org_2275350'); // Mercury (Mercury) — confirm per fund
setAndFire(sel[3], 'Other');
setTimeout(() => {
  const otherInput = row.querySelector('#other_reason_input');
  if (otherInput) {
    otherInput.value = 'Mercury Treasury sweep';
    otherInput.dispatchEvent(new Event('input', { bubbles: true }));
  }
}, 300);
```

**Known Mercury org IDs for Treasury sweeps:**
- Dash VC (dashvc, oND3qPKR) → org_2275350 (Mercury (Mercury))

**$75 receipt rule does NOT apply** — this is a bank-level account transfer, not a business expense.

---

## 1099-NEC Reportable Transactions

### When to Check the "1099 Reportable Transaction" Box

Check the box on **every qualifying transaction regardless of individual transaction amount.**

The $2,000 figure (tax year 2026 forward; $600 for prior years) is the **cumulative annual threshold** for IRS filing — not a per-transaction filter. Hub uses the checkbox to accumulate all qualifying payments to a recipient across the year so the total can be determined. Skipping a transaction because it's under $2,000 would break that tracking and could cause a missed 1099 filing.

Check the box when **both** conditions are true:
1. Payment is **outgoing** (fund paying out for services)
2. Recipient is **not a C-corp or S-corp** — unless they are an attorney/law firm (attorneys always require 1099 even if incorporated)

### By Transaction Type

| Payment Type | Check 1099? | Notes |
|---|---|---|
| **Management fees** | ✅ Yes — unless management company is a C-corp or S-corp | Most VC management LLCs/partnerships qualify — check on every payment, any amount |
| **Spicer Jeffries / Cherry Bekaert / tax prep** | ✅ Always | Every payment, any amount — do not skip small payments |
| **Legal fees** | ✅ Always | IRS requires 1099-NEC for attorneys even if incorporated — every payment, any amount |
| **Decile Group / Vc Lab** | ❌ No | Decile is incorporated (C-corp or S-corp) |
| **Capital calls (inbound)** | ❌ No | Not a service payment |
| **Bank charges / interest** | ❌ No | Not a service payment |
| **Investment wires** | ❌ No | Not a service payment |

### How to Check the Box (UI Note)

The 1099 checkbox is a hidden element. Use the label to click it:

```javascript
const label = row.querySelector('label[for="firm_admin_accounting_transaction_is_1099_reportable"]');
label?.click();
```

⚠️ **Account admin access required.** The reconciliation saves successfully, but checking the 1099 box may return "Only account admins are allowed to do that." In that case, leave a note for a member of fund accounting to check it manually — she has admin access.

---

### 🚩 Flag for Human Review

When a transaction type is unclear, involves large inter-entity transfers, or lacks supporting documentation:

```javascript
const flagBtn = Array.from(row.querySelectorAll('button')).find(b => b.textContent.includes('Flag'));
flagBtn?.click();
```

**Always flag:**
- BR TO BR / inter-fund transfers (unclear without GP confirmation)
- Investment wires without a signed doc in the data room
- Transactions the GP has specifically put on hold (e.g., "MISSING TRUST DOCS")
- Any transaction where the counterparty can't be identified

---

## GL Account Quick Reference

| What ID | Account | Use for |
|---------|---------|---------|
| 108 | 1000 - Bank - Active Account | — |
| 111 | 1100 - Investment | Investment wires to portfolio companies |
| 123 | 1200 - Capital Call Receivable - LP | Capital call payments from LPs |
| 206 | 2120 - Tax Accrual | Tax prep payments when matching GL accrual exists |
| 6100 | Audit Fees | Spicer Jeffries / Cherry Bekaert payments ≥ $5,000 with no matching accrual |
| 219 | 2205 - Management Fee Payable | Management fee payments |
| 316 | 4001 - Bank Interest Income | Cashback, interest credits from bank |
| 369 | 6120 - Fund Administration Fees | Vc Lab / Decile Group admin fee payments |
| 394 | 7000 - Bank Charges | Bank service fees, wire fees, foreign exchange fees |
| 399 | Software | ManCo SaaS/software subscriptions (<$75) |
| 418 | Business Meals | ManCo meal/food expenses (<$75); also CVS/Walgreens snacks |
| 500 | Travel (7110) | ManCo travel expenses (<$75); also Clipper card |
| 562 | 1002 - Bank - Additional Reserve Account | ICS DDA / shadow reserve accounts / Mercury Treasury sweeps |
| 163 | 1550 - Management Fee Receivable | ManCo: inbound management fee payments from funds |
| 228 | 2245 - Due to Related Party | ManCo: GP personal transfers into ManCo bank account |
| 289 | 3153 - Cash Distribution - Owner's Draw - GP | ManCo: GP withdrawals / owner's draw distributions |
| 379 | 6190 - Employee Benefits | ManCo: employee benefits payments (e.g., Matrix Trust Co) |

*Note: What IDs are numeric values passed to `tomselect.setValue()` — they are not the account numbers themselves.*

---

## Technical Foundation

### Helper Functions (use in every session)

```javascript
function setAndFire(sel, value) {
  sel.tomselect.setValue(value);
  sel.dispatchEvent(new Event('change', { bubbles: true }));
  sel.dispatchEvent(new Event('input', { bubbles: true }));
}

function findWho(whoSel, terms) {
  const opts = Object.entries(whoSel.tomselect.options);
  for (const t of terms) {
    const m = opts.find(([k,v]) => v.text && v.text.toLowerCase().includes(t.toLowerCase()));
    if (m) return m[0];
  }
  return null;
}
```


### Adding a New Organization to the Who Dropdown

When a vendor doesn't exist in the Who dropdown, you can create it inline from the reconcile page.

**Flow:**
1. Click into the Who TomSelect for the row, type the vendor name, then click **"+ Add new…"** in the dropdown.
2. A modal appears: "Select Counterparty Type" with a **Name** field and a **Counterparty Type** selector.
3. Fill in the name using the **native input setter** (plain `.value =` assignment may not register with the form):
```javascript
const nameInp = document.querySelector('.modal-content input[type="text"]');
const nativeSetter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
nativeSetter.call(nameInp, 'Vendor Name');
nameInp.dispatchEvent(new Event('input', {bubbles: true}));
```
4. Set the Counterparty Type to "Organization":
```javascript
const typeSel = document.querySelector('.modal-content select');
const orgOpt = Array.from(typeSel.options).find(o => o.text.trim() === 'Organization');
typeSel.value = orgOpt.value;
typeSel.dispatchEvent(new Event('change', {bubbles: true}));
```
5. Submit via `requestSubmit()` on the form — JS `.click()` on the Create button is unreliable:
```javascript
const form = nameInp.closest('form');
form.requestSubmit();
```
6. Confirm the **"Entity created"** toast appears. Then close the modal.
7. **⚠️ Page reload required:** Reload the page (`location.reload()`) before reconciling any row against the new org. After reload, the org is available in every row's Who dropdown with the `org_XXXXX` format.

**Counterparty type:** Always set to **"Organization"** for vendors, banks, and service providers. Never leave it as the default "Portfolio Company" unless the counterparty is actually an investee.

### TomSelect API

All four fields use TomSelect dropdowns. Native `select.value = x` doesn't work.

```javascript
select.tomselect?.setValue('value_string_or_number');

// CRITICAL: After setting all four fields, fire events on each select:
for (const s of row.querySelectorAll('select')) {
  s.dispatchEvent(new Event('change', { bubbles: true }));
  s.dispatchEvent(new Event('input', { bubbles: true }));
}
// Without this, the Reconcile button stays disabled (btn.disabled === true)
```

**Select field order in each row:** sels[0]=type, sels[1]=GL account, sels[2]=who (counterparty), sels[3]=why, sels[4]=bank account (pre-filled, do not change).

### Row Selector (New UI)

The new UI uses compound class strings. Use:

```javascript
const rows = Array.from(document.querySelectorAll('[class*="mb-3"][class*="grid-cols-16"]'));
```

Fall back to `.mb-3` if that returns nothing (older UI).

**Always scroll first** to force lazy-load all rows into the DOM:

```javascript
window.scrollTo(0, document.body.scrollHeight);
```

### Reconcile Button — Click-and-Cancel Pattern

Clicking the row "Reconcile" button saves the row AND may trigger a fund-level "Mark this fund as fully reconciled?" modal. This modal is separate from the row save — cancel it (the row is already saved). The "Transaction saved" toast confirms the row save succeeded.

**⛔ Critical:** The modal appears asynchronously. You cannot cancel it in the same JS call that clicks Reconcile. You must use **separate JS tool calls** for each click-and-cancel cycle:

```javascript
// Run this call repeatedly until "0 remain":
const c = Array.from(document.querySelectorAll('button')).find(b => b.textContent.trim() === 'Cancel');
if (c) c.click();
const rows = Array.from(document.querySelectorAll('[class*="mb-3"][class*="grid-cols-16"]'));
// Filter to rows that are fully configured AND have an enabled Reconcile button
const ready = rows.filter(r => {
  const typeVal = r.querySelectorAll('select')[0]?.value;
  const btn = Array.from(r.querySelectorAll('button')).find(b => b.textContent.includes('Reconcile') && !b.textContent.includes('Snooze'));
  return typeVal && btn && !btn.disabled;
});
const btn = Array.from(ready[0]?.querySelectorAll('button')||[]).find(b => b.textContent.includes('Reconcile') && !b.textContent.includes('Snooze'));
if (btn) btn.click();
`${ready.length} remain`;
```

**The "Only account admins" red toast** refers to the fund-level timestamp action — not the row reconcile itself. The row still saves successfully (rows disappear from the queue after cancel). This is expected behavior.

### Counterparty ID Types

| Prefix | Type | Used for |
|--------|------|---------|
| `ca_xxxx` | Capital Account | LP investors |
| `org_xxxx` | Organization | Management companies, banks, external entities |
| `pc_xxxx` | Portfolio Company | Investee companies |

### Row Index Stability

Rows shift after every reconciliation. Always re-query fresh, and process batches from highest index to lowest.

### Reconcile Button Check

```javascript
const btn = Array.from(row.querySelectorAll('button')).find(b => b.textContent.includes('Reconcile') && !b.textContent.includes('Snooze'));
// btn.disabled === false → ready to click
// btn.disabled === true → a field is wrong or an event wasn't fired
```

### Counterparty Type Modal

The first time you reconcile against a new `org_xxx` Who, a modal appears asking for entity Name and Type (Portfolio Company / Organization). Fill in and submit — only happens once per org.

### Slideover Panel (Add a Company / Investments)

On some funds, the Add a Company slideover renders off-screen. Resize browser to 1920px wide using the resize_window tool.

---

## Fund Reference

**Full fund list:** `fund-manifest.csv` (bundled with this skill — 273 active funds). Columns: `order, name, slug, fund_db_id, ptf_legal_name, owner_name, owner_email, va, base_ids, in_partners_track, mgmt_fee_url, capital_accounts_url`.

**Reconcile URL pattern:** Try the short form first: `https://[slug].decilehub.com/accounting/reconcile`. This redirects automatically for most funds. If it returns a **404**, use the entity ID lookup below to find the full path.

### Fund-Specific Files

Some funds have a dedicated file in the `funds/` subdirectory with their entity IDs, org IDs, and custom transaction patterns. **When reconciling one of these funds, use the Read tool to load the file before proceeding:**

| Fund | File to load |
|------|-------------|
| Team Ignite (any entity) | `funds/teamignite.md` |
| Riceberg (any entity) | `funds/riceberg.md` |
| Evio Venture Capital (any entity) | `funds/evio.md` |
| Tachles VC (any entity) | `funds/tachles.md` |

The file path is relative to this skill's directory. To load, e.g.: `Read("<skill_dir>/funds/teamignite.md")`.

As new funds accumulate custom transaction patterns or org IDs, add a new file to `funds/` and add it to this table.

### Entity ID Lookup (when short URL 404s)

`hub_slugs.json` (bundled with this skill) maps every entity ID to its slug — 7,789 entries covering all Hub funds. Use it to find the entity IDs for a given slug, then try each one until you land on the right reconcile page:

```python
import json
skill_dir = "<base directory shown at skill load time>"
with open(f"{skill_dir}/hub_slugs.json") as f:
    data = json.load(f)
slug = "energymix"  # replace with target slug
ids = [eid for eid, s in data.items() if s == slug]
print(ids)
# e.g. ['kKkzY68j', '3nzwYG8m', '9nG9mWKm']
```

Then navigate to `https://[slug].decilehub.com/firm_admin/entities/[ID]/accounting/reconcile` for each ID until you reach the correct reconcile page. As an alternative to Python, you can also extract the entity ID from any page link on the fund's dashboard (look for `/firm_admin/entities/[ID]/` in the href attributes).

Each slug typically has 2–3 entity IDs (fund LP, management company, GP entity). The reconcile page for the fund LP is usually the right one.

### Funds with Known Entity IDs

| Fund | Slug | Entity ID |
|------|------|-----------|
| Spatial Capital Fund I LP | spatial | GNMbmon1 |
| Capital M Fund I, LP | capitalm | WL8Zbb8O |
| Func Ventures Fund I, LP | func | A1NAqxNX |
| Prana Tech Ventures Fund 1, LP | prana | p8BGQDKk |
| TheFounderVC Fund I, LP | thefoundervc | g8b3D4Kx |
| Swizzle Ventures Fund I, LP | swizzlevc | g8gXQ2KZ |
| Not Yet Ventures Fund I, LP | notyet | BKVLLen3 | ⚠️ Prana is a completely separate fund (p8BGQDKk) — do not confuse |
| Airborne Maverick Fund I | airborne | RnePpeNq |
| Begin Ventures | begin | GNXJx98g |
| Capacity Capital Fund II, LP | capacity | 6NQ9agNr |
| Energy Mix Ventures Fund I, LP | energymix | kKkzY68j |
| Riceberg (ManCo) | riceberg | GNMQeEN1 |
| Homegrown (ManCo) | homegrown | WN6zLVN3 |
| TheFounderVC ManCo | thvvc | WnW5v7KE |
| Team Ignite Fund I ManCo | teamignite | Wnwyk2K4 |
| Team Ignite Fund II ManCo | teamignite | 6N0GqGKy |
| Jolly VC Management LLC (ManCo) | jollyvc | 3N9JWVnJ |
| Jolly VC Fund I, LP | jollyvc | l8y4j1nZ |
| Ranchos Ventures Mgmt | ranchos | 3nzwxJ8m |
| Tachles VC Management LLC | tachles | PKLo5B82 |
| Blockchain Builders LLC | blockchainbuilders | RGNMx0K1 |
| Capacity Capital ManCo | capacity | y8m7EzNW |
| Riceberg Ventures Premier Fund I, LP | riceberg | yKv4o4nx |
| Incisive Ventures Fund I, LP | incisive | 39KJqL8Z |
| Qubits Ventures Fund II, LP | qubits | 9KJMkz8Z |
| Insurtech Fund II, LP | insurtech | gNrX2m81 |
| Blockstreet Capital Fund I, LP | blockstreet | GNXzex8g |
| Lobster Capital Fund II, LP | lobstercap | 1NAW5G8X | ⚠️ Slug is "lobstercap" NOT "lobster" — lobster is the ManCo |
| Lobster Capital LLC (ManCo) | lobster | Wnw4qM84 |
| Evio Venture Capital Fund I, LP | evio | 3K5Rz0NQ |
| Evio VC GP (Evio Venture Capital GP, LLC) | evio | 2N3GdXNy |
| Evio Venture Capital, LLC (ManCo) | evio | 2nER4X8Z |
| Dash VC (fund LP) | dashvc | oND3qPKR |

**Fund-specific bank account overrides** (default is whatever is pre-selected):
- Swizzle Ventures → always use **Stifel - Account Ending 4961** (select value `109`) for every transaction

---

## Credit Card Cash Advance Pattern (Mercury Credit / Mercury Checking)

Some funds pay VC Lab / Decile Group fees via a Mercury credit card cash advance. This produces three matching transactions in the reconcile queue:

1. **Mercury Checking +$X** — cash advance deposited into checking
2. **Mercury Credit -$X** — corresponding charge on the credit card
3. *(The actual outbound VC Lab payment may be captured separately or is implicit)*

Because the Checking and Credit transactions net to zero on the fund's books, **book all three (or both) to VC Lab** as fund administration fees:

- **Transaction Type:** `admin_fee`
- **What:** 369 (6120 - Fund Administration Fees)
- **Who:** `org_177845` (Decile Group)
- **Why:** `Fee payment`

The positive Mercury Checking inflow and negative Mercury Credit charge offset each other — reconciling both to 6120 is correct. No 1099-NEC (Decile is incorporated).

---

## Adding Transactions via CSV Import

When a fund's bank account is not synced and transactions need to be manually added to Hub:

1. Download the CSV from the bank (e.g., Mercury export filtered to the relevant date range)
2. Reformat to three columns: `Date`, `Description`, `Amount` (single column, positive for inbound, negative for outbound)
3. Navigate to the import page: `https://[slug].decilehub.com/data_imports/edit/fund:bank-transaction:[ENTITY_ID]`
4. Leave Transaction Type / What / Who / Why **blank** — transactions will land in the reconcile queue for individual categorization
5. Upload the CSV and proceed through the column mapping (step 2) and preview (step 3)

**⚠️ File upload limitation:** The browser extension currently cannot upload files to financial sites directly. a member of fund accounting must upload the CSV manually. Prepare the reformatted file and save it to the workspace folder, then ask a member of fund accounting to upload it from there.

The Mercury CSV export columns to use:
- `Date (UTC)` → Date
- `Description` → Description  
- `Amount` → Amount (already signed correctly — positive = inbound, negative = outbound)

---

## Post-Reconciliation Sheet (Bulk Runs Only)

After completing a **bulk reconciliation run** (3 or more funds in a single session), generate a `Cash_Reconciliation_[YYYY-MM-DD].xlsx` and save it to the workspace outputs folder. **Skip this step for single-fund or single-ManCo runs.**

### When to generate

| Scenario | Generate sheet? |
|----------|----------------|
| Bulk run — multiple funds from a Mercury CSV batch | ✅ Yes |
| Single fund or ManCo reconciled on its own | ❌ No |
| Catch-up run on one fund with many old transactions | ❌ No |

### Sheet structure: two tabs

**Tab 1 — "Exceptions"**

Lists every transaction that could *not* be reconciled, with the reason and recommended next action. Rules for inclusion:

- **Include:** transactions still in an unreconciled queue after the session where the reason needs a member of fund accounting's attention (ambiguous LP, missing receipt ≥$75, unknown vendor, investment wire needing docs, etc.)
- **Exclude:** Any transaction with a Hub flag badge (`FLAGGED:`, `INVESTMENT DOCS OPEN`, `MISSING DOCS`, or `ENTITY MISMATCH OPEN`) — Hub is tracking those independently
- **Exclude:** Any transaction whose fund's queue is now empty (reconciled since the prior session)
- Before writing this tab, do the pre-flight scan below on each exception fund

Columns: `Date | Fund Name | Entity ID | Bank Account | Description | Amount ($) | Exception Type | Action Required`

Color coding (openpyxl PatternFill):
- 🟡 Yellow `FFF2CC` — same-day or recent transactions
- 🟠 Orange `FCE4D6` — receipt required (IRS ≥$75 rule)
- 🔵 Blue `DEEAF1` — ambiguous LP or counterparty match
- 🔴 Red `FFE0E0` — investment wire or do-not-reconcile
- ⬜ Gray `F2F2F2` — carry-forward from a prior period

**Tab 2 — "Reconciliation Log"**

One row per transaction reconciled during the session. For capital calls, always populate the Capital Account ID so a member of fund accounting can verify the correct LP was matched.

Columns: `Fund Name | Entity ID | Bank Acct | Txn Date | Description / Payee | Amount ($) | Txn Type | GL Account | Counterparty (Who) | Capital Account ID | Notes / Capital Call Detail`

Use common names (e.g. "ELM Partners", "Uber") everywhere — not Hub internal IDs — except Capital Account ID which should stay as `ca_XXXX` for a member of fund accounting's reference.

### File naming and location

```
Cash_Reconciliation_YYYY-MM-DD.xlsx
```

Save to the workspace outputs folder (the folder the user can open). The user moves it to their preferred path (e.g. `Claude > Projects > Cash Reconciliation > Daily Bank Transactions > [date]`).

### Building the sheet

Read the `xlsx` skill's SKILL.md first, then build with openpyxl. After saving, always recalculate formulas and verify zero errors:

```bash
python3 [xlsx_skill_dir]/scripts/recalc.py Cash_Reconciliation_YYYY-MM-DD.xlsx 30
```

Confirm `"status": "success"` and `"total_errors": 0` before sharing.

### Pre-flight scan before building

Before writing the Exceptions tab, navigate to each fund with carry-forward items and confirm which are still active:

```javascript
window.scrollTo(0, document.body.scrollHeight);
const rows = Array.from(document.querySelectorAll('[class*="mb-3"][class*="grid-cols-16"]'));
rows.map((r,i) => {
  const lines = r.innerText.split('\n').map(l=>l.trim()).filter(l=>l.length>2);
  const t = r.innerText;
  const flag = t.includes('MISMATCH') ? ' 🟡MISMATCH' : t.includes('FLAGGED') ? ' ⛳FLAGGED' : t.includes('INVESTMENT DOCS') ? ' 📄INVEST-DOCS' : t.includes('MISSING DOCS') ? ' 📄MISSING-DOCS' : '';
  return `[${i}] ${lines.slice(0,3).join(' | ')}${flag}`;
}).join('\n');
```

**Drop from Exceptions tab if:**
- Fund queue is empty → item was reconciled, remove it
- Row shows any badge (MISMATCH, FLAGGED, INVESTMENT DOCS, MISSING DOCS) → Hub is tracking it, remove it

## What's Still Being Trained

- Inter-entity / inter-fund transfers (BR TO BR, Online Banking transfers between accounts)
- Test transactions
- Distribution payments
- Loan repayments
- Investment wires end-to-end (funding round association modal)
- Large unusual payments (e.g., Ranchos Ventures CHISOS LLC $100K, Team Ignite SPACEX/SYDECAR)
- Riceberg: Transfer In rows (no amount, GL 108) — management fee inbounds? Categorization TBD

When encountering these, present to a member of fund accounting with a description and ask before reconciling.
