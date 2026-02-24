# PRD  — Final Implementation Specification

**Version:** 2.0  
**Date:** February 23, 2026  
**Status:** Final — Authoritative Engineering Contract  
**Platform:** Android (Flutter)  
**Classification:** Internal — Engineering & Product  

**Conflict resolution priority:** v1.2 rules override v1.1 rules override v1.0 rules.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Product Principles](#2-product-principles)
3. [User Workflows](#3-user-workflows)
4. [Functional Requirements](#4-functional-requirements)
5. [Transaction Model](#5-transaction-model)
6. [Financial Calculation Rules](#6-financial-calculation-rules)
7. [Temporal Integrity Rules](#7-temporal-integrity-rules)
8. [Customer Identity Rules](#8-customer-identity-rules)
9. [Data Persistence & Database Rules](#9-data-persistence--database-rules)
10. [Backup & Restore](#10-backup--restore)
11. [Error Handling](#11-error-handling)
12. [Performance Requirements](#12-performance-requirements)
13. [Non-Functional Requirements](#13-non-functional-requirements)
14. [External Design Reference (design.md)](#14-external-design-reference-designmd)

---

## 1. Overview

### 1.1 Product Vision

SimpleLedger is a **financial memory replacement system** for small retail shopkeepers across India. It is **not** accounting software. It replaces three things every shopkeeper currently depends on:

1. The paper _udhar_ notebook
2. Mental arithmetic
3. The calculator

The core design promise: **a shopkeeper can record any transaction in under 3 seconds, with one hand, while a customer is waiting.**

The app requires zero accounting knowledge. It never uses words like debit, credit, ledger, journal, or reconciliation. It speaks the shopkeeper's language: **Sale, Udhar, Kharcha, Report.**

### 1.2 Target Users

**Primary Persona:** Ramesh Gupta (archetype)  
**Age:** 28–55  
**Business:** Kirana store, tea stall, stationery shop, garment shop, medical store  
**Location:** Tier 2–4 Indian cities and towns  
**Phone:** Android (entry-level to mid-range, 2–4 GB RAM)  
**Language:** Hindi primary, English secondary  
**Tech familiarity:** WhatsApp, YouTube, UPI apps — nothing more

**Environmental Context:**

| Factor | Reality |
|--------|---------|
| Workspace | Crowded counter, one hand often busy |
| Customers | 2–5 waiting at any time during peak hours |
| Connectivity | Unreliable; 2G/3G pockets, frequent outages |
| Power | Intermittent; phone may die mid-transaction |
| Current system | Paper notebook, memory, calculator |
| Pain | Forgets entries, miscalculates totals, loses notebooks |
| Trust threshold | **App must be more reliable than the paper notebook** |

**User Behavior Assumptions:**

- Will not read onboarding tutorials.
- Will not create accounts or passwords.
- Will not fill forms with more than 2–3 fields.
- Will abandon the app permanently after one data-loss incident.
- Will use the app with greasy, wet, or dusty hands.
- Phone screen may be cracked or low-brightness.

### 1.3 Offline-First Philosophy

All features of SimpleLedger operate with zero internet connectivity. This is a non-negotiable foundational constraint, not a fallback mode. No feature may depend on network access to function. No data leaves the device except via user-initiated backup export. There are no accounts, no cloud sync, and no OTP flows in v1.

---

## 2. Product Principles

These are ranked. In any design conflict, the higher-ranked principle wins.

| Rank | Principle | Rationale |
|------|-----------|-----------|
| 1 | **Never lose data** | One data loss = permanent uninstall. The database must persist writes immediately. |
| 2 | **Offline-first** | All features work with zero internet. Always. |
| 3 | **Instant save** | No confirmation dialogs, no "Are you sure?" screens. Tap = saved. |
| 4 | **≤ 3 taps to record a sale** | Enter amount → tap SALE → tap Cash/UPI. Done. |
| 5 | **One-hand operation** | All primary actions reachable by right thumb in portrait mode. |
| 6 | **Large, thumb-friendly UI** | Minimum touch target: 48×48 dp. Preferred: 56×56 dp. |
| 7 | **Faster than a calculator** | The app IS the calculator. No switching apps. |
| 8 | **Always correct totals** | Balances computed from transaction history. Never stored. Never editable. |

### 2.1 Terminology Map

The app uses shopkeeper-friendly language. The following mapping is exhaustive:

| Concept | App Term (English) | App Term (Hindi Context) | NEVER Use |
|---------|-------------------|--------------------------|-----------|
| Record income | Sale | Sale | Revenue, Income, Receipt |
| Record expense | Cash Out / Kharcha | Kharcha | Expenditure, Debit, Outflow |
| Record credit given | Credit / Udhar | Udhar | Accounts Receivable, Debit Note |
| Record payment received | Payment Received | Payment Received | Credit Note, Settlement |
| Customer balance | Pending amount | Pending amount | Outstanding, Receivable, Balance Due |
| Cash available | Cash in Hand | Cash in Hand | Cash Balance, Net Position |
| Daily summary | Today's Summary | Today's Summary | P&L, Income Statement, Report Card |
| Transaction history | History | History | Journal, Ledger, Log |
| Undo | Undo | Undo | Reverse, Void, Contra |
| Backup | Backup | Backup | Export, Archive, Dump |
| Restore | Restore | Restore | Import, Recover, Replay |

---

## 3. User Workflows

### 3.1 Information Architecture

```
┌─────────────────────────────────────────────┐
│                 HOME SCREEN                  │
│  ┌────────────────────────────────────────┐  │
│  │  Cash in Hand: ₹ 12,450               │  │
│  ├────────────────────────────────────────┤  │
│  │  Amount Display: ₹ ____               │  │
│  ├────────────────────────────────────────┤  │
│  │        NUMERIC KEYPAD                  │  │
│  │   [1]  [2]  [3]                        │  │
│  │   [4]  [5]  [6]                        │  │
│  │   [7]  [8]  [9]                        │  │
│  │   [.]  [0]  [⌫]                        │  │
│  ├────────────────────────────────────────┤  │
│  │  [  SALE  ] [CASH OUT] [ CREDIT ]      │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  Bottom Nav: [Home] [Due List] [Report] [More] │
└─────────────────────────────────────────────┘
```

**Navigation Map:**

```
Home ──────────── Sale ──── Cash / UPI → Saved
  │
  ├──────────── Cash Out ── Category + Note → Saved
  │
  ├──────────── Credit ──── Customer Form → Saved
  │
  ├── Due List ──── Customer Detail ──── Record Payment
  │
  ├── Report ──── Today Summary (Daily Closing)
  │
  └── More ──── Backup / Restore
                Edit Categories
                Settings
                About
```

### 3.2 Flow: Record Cash Sale

```
[Home Screen]
    │
    ├── User taps digits on keypad (e.g., 2, 5, 0)
    │   → Amount display shows: ₹ 250
    │
    ├── User taps [SALE]
    │   → Bottom sheet slides up with [Cash] and [UPI]
    │
    ├── User taps [Cash]
    │   → Transaction saved (type: sale_cash, amount: 25000 paisa)
    │   → Bottom sheet dismisses
    │   → Amount resets to ₹ 0
    │   → Cash in Hand cache incremented by ₹250
    │   → Undo banner: "✓ ₹250 Cash Sale saved [UNDO]"
    │   → Banner auto-dismisses after 5 seconds
    │
    └── DONE (2 functional taps)
```

### 3.3 Flow: Record UPI Sale

```
[Home Screen]
    │
    ├── User enters amount
    ├── User taps [SALE]
    ├── User taps [UPI]
    │   → Transaction saved (type: sale_upi)
    │   → Cash in Hand UNCHANGED (UPI goes to bank)
    │   → Undo banner appears
    │
    └── DONE
```

### 3.4 Flow: Record Expense

```
[Home Screen]
    │
    ├── User enters amount (e.g., 100)
    ├── User taps [CASH OUT]
    │   → Bottom sheet: category chips + note field + Save
    │
    ├── User taps "Chai/Snacks" chip
    │   → If note is empty: auto-save immediately
    │   → Transaction saved (type: expense, category: "Chai/Snacks")
    │   → Cash in Hand decreases by ₹100
    │   → Undo banner appears
    │
    └── DONE (2 functional taps — fastest path)

    ALTERNATE:
    ├── User selects category
    ├── User types note: "Biscuits for shop"
    ├── User taps [Save]
    │
    └── DONE (3 functional taps)
```

### 3.5 Flow: Record Credit (Udhar)

```
[Home Screen]
    │
    ├── User enters amount (e.g., 500)
    ├── User taps [CREDIT]
    │   → Bottom sheet: Name input + Phone input + Note input + Save
    │
    ├── User types "Raj" in name field
    │   → Autocomplete shows: "Rajesh Kumar", "Rajendra"
    │   → Each suggestion shows disambiguator: pending amount, last date, or phone
    │
    ├── User taps "Rajesh Kumar"
    │   → Phone auto-fills if saved previously
    │
    ├── User taps [Save]
    │   → Transaction saved (type: credit_given, customer_id: Rajesh's id)
    │   → Cash in Hand decreases by ₹500
    │   → Undo banner appears
    │
    └── DONE
```

### 3.6 Flow: Receive Payment from Customer

```
[Due List]
    │
    ├── User taps "Rajesh Kumar — ₹2,500"
    │   → Customer Detail Screen opens
    │
    ├── User taps [Receive Payment]
    │   → Calculator-style keypad appears
    │
    ├── User enters 1000
    ├── User taps [Save]
    │   → Transaction saved (type: credit_received, customer_id: Rajesh, amount: 100000 paisa)
    │   → Outstanding recomputes: ₹2,500 − ₹1,000 = ₹1,500
    │   → Cash in Hand increases by ₹1,000
    │   → Undo banner appears
    │
    └── DONE
```

### 3.7 Flow: Undo a Transaction

```
[Undo banner visible]
    │
    ├── User taps [UNDO]
    │   → Transaction soft-deleted (is_deleted = 1, deleted_at = now)
    │   → All balances recompute from event log (WHERE is_deleted = 0)
    │   → Cash-in-Hand cache decrementally updated (undo reversal)
    │   → Toast: "Entry removed"
    │   → Banner dismisses
    │
    └── DONE
```

### 3.8 Flow: Backup Data

```
[Home] → [More] → [Backup & Restore]
    │
    ├── User taps [Save Backup]
    │   → WAL checkpoint runs (ensure all data in main DB file)
    │   → SHA-256 of JSON payload computed
    │   → File created: SimpleLedger_Backup_YYYYMMDD_HHMMSS.slbackup
    │   → Saved to Downloads folder
    │   → Toast: "Backup saved to Downloads folder"
    │
    └── DONE
```

### 3.9 Flow: Restore Data

```
[More] → [Backup & Restore] → [Restore Backup]
    │
    ├── File picker opens (filtered to .slbackup)
    │
    ├── User selects backup file
    │   → Header parsed. Magic bytes verified. CRC-32 verified.
    │   → If encrypted: passphrase prompt shown
    │   → SHA-256 of payload verified
    │   → pre-restore comparison runs (§10.5)
    │
    ├── Warning dialog shown (standard or age-mismatch variant)
    │
    ├── User confirms
    │   → BEGIN EXCLUSIVE transaction
    │   → All current data deleted
    │   → Backup data inserted
    │   → COMMIT
    │   → Full Ledger Rebuild runs (§10.6)
    │   → All volatile state cleared (§10.7)
    │   → Navigate to Home
    │   → Toast: "Data restored successfully."
    │
    └── DONE
```

### 3.9 Flow: Daily Closing Review

```
[Home] → [Report] tab
    │
    ├── Today's Summary displayed (business date)
    │   Total Sales: ₹15,200 (Cash: ₹12,000 / UPI: ₹3,200)
    │   Kharcha: ₹1,800
    │   Udhar Given: ₹3,500
    │   Udhar Received: ₹2,000
    │   Cash in Hand: ₹12,450 (all-time, not date-filtered)
    │
    ├── User taps [◀ Previous Day]
    │   → Shows previous business day's summary
    │
    └── DONE
```

---

## 4. Functional Requirements

### 4.1 Home Screen (Calculator Entry)

**Purpose:** The home screen IS the transaction entry point. It looks and behaves like a calculator because that is the mental model the user already has.

**Layout (Top to Bottom):**

| Zone | Content | Behavior |
|------|---------|----------|
| **Status Bar** | "Cash in Hand: ₹ XX,XXX" | Always visible. Computed live from in-memory cache. |
| **Amount Display** | Large numeric display (₹ 0) | Right-aligned. Font size ≥ 32sp. Rupee symbol always visible. |
| **Keypad** | 0–9, decimal point, backspace | Calculator-style grid. Each button minimum 56×56 dp. Haptic feedback on tap. |
| **Action Buttons** | SALE · CASH OUT · CREDIT | Three wide buttons. Distinct colors. Minimum height 56 dp. |
| **Bottom Nav** | Home · Due List · Report · More | Fixed bottom navigation. 4 items max. Icon + label. |

**Interaction Rules:**

- Amount display starts at ₹ 0.
- Tapping digits appends to the current number.
- Backspace removes last digit.
- Decimal point allowed once. Maximum 2 decimal places.
- Maximum amount: ₹ 99,99,999.99 (stored as 9999999999 paisa).
- If amount is 0 or empty when an action button is tapped → show inline toast: "Please enter amount" (2 seconds, non-blocking). No bottom sheet opens.
- After a transaction is saved, amount display resets to ₹ 0.
- Leading zeros stripped except before decimal (0.5 is valid).

**Opening Balance (One-Time, First Launch Only):**

On the Start Fresh path, after the welcome screen, show a single optional screen:

```
┌─────────────────────────────────────────────┐
│  How much cash do you have right now?        │
│                                              │
│  ₹ ____  (keypad)                            │
│                                              │
│  [Set Starting Cash]   [Skip — Start at ₹0]  │
└─────────────────────────────────────────────┘
```

- If user sets a starting cash: create one transaction of type `opening_balance`. No undo banner for this transaction.
- Opening balance can only be set **once** during first launch. There is no way to set it again later. Adjustments must be recorded as a `sale_cash` (if more cash) or `expense` (if less cash).
- **Business Day Edge Case — First Install (CT-8):** If the app is first installed between midnight and the business closing time (default 04:00 AM), the `opening_balance` transaction MUST have its `business_date` set to the **current calendar date**, NOT the previous business day. This is a one-time exception to the §23.A business-day boundary rule.

### 4.2 Sale Flow

**Trigger:** User enters amount → taps **SALE** button.

**Interaction:**

1. A bottom sheet slides up with two large buttons: **Cash** (green icon) and **UPI** (blue icon).
2. User taps one.
3. Transaction saved instantly.
4. Bottom sheet dismisses.
5. Amount display resets to ₹ 0.
6. Undo banner appears for 5 seconds.

**System Actions on Save:**

- Create transaction record with type `sale_cash` or `sale_upi`.
- `sale_cash`: Cash in Hand cache increments by amount.
- `sale_upi`: Cash in Hand cache unchanged. UPI goes to bank, not cash drawer.
- Undo banner appears.

**Why Two Sub-Types Matter:**

`sale_upi` is **NEVER** included in the Cash in Hand formula. It must not appear in any CASE branch of the Cash in Hand SQL query.

### 4.3 Expense (Cash Out / Kharcha)

**Trigger:** User enters amount → taps **CASH OUT** button.

**Interaction:**

1. A bottom sheet slides up with:
   - Category selector — horizontal scrollable chips.
   - Note field — single-line text input. Optional. Placeholder: "Short note (optional)".
   - Save button — large, prominent.
2. Default category: most recently used category is pre-selected.
3. If user taps a category chip and note is empty: auto-save immediately.
4. If note is filled: user must tap Save explicitly.
5. Transaction saved. Bottom sheet dismisses. Undo banner appears.

**Category Management:**

- Default categories (pre-installed): Chai/Snacks, Transport, Rent, Supplier Payment, Staff, Other.
- User can add custom categories (More → Edit Categories). Maximum 15 categories. Category names: max 20 characters.
- User can rename or hide (not delete) categories.

### 4.4 Credit (Udhar)

**Trigger:** User enters amount → taps **CREDIT** button.

**Interaction:**

1. A bottom sheet slides up with: customer name (required, with autocomplete), phone number (optional, 10-digit Indian mobile), note (optional), Save button.
2. As user types, autocomplete shows matching existing customers. Maximum 5 suggestions.
3. **Each suggestion MUST show at least one disambiguator:** last due amount OR last transaction date OR phone number.
4. If user selects an existing customer: pre-fill phone number.
5. If user types a new name: new customer record is created on save with a new unique integer `id` and UUID.
6. Transaction saved. Undo banner appears.

**System Actions on Save:**

- Create transaction with type `credit_given`.
- If new customer name: create customer record with a unique, immutable integer primary key `id` (and a `uuid` for backup identity).
- If existing customer selected: link transaction to that customer's `id`.
- **All transactions bind to `customer_id` (integer), never to the customer name string.**
- Cash in Hand cache decrements by amount (goods left the shop without cash coming in).

### 4.5 Partial Payment Handling

When a customer repays part or all of their outstanding udhar, the app MUST **NEVER** edit or modify the original credit entry. Instead, it creates a new transaction of type `credit_received`.

**Flow:**

1. User navigates to **Due List** → taps a customer.
2. Customer Detail screen shows: name, phone, total outstanding, transaction history (reverse chronological by `id`).
3. User taps **"Receive Payment"** button.
4. Calculator-style keypad appears.
5. User enters amount → taps **Save**.
6. New `credit_received` transaction is created.
7. Outstanding balance recomputes from event log.
8. Undo banner appears.

**Edge Cases:**

| Scenario | Behavior |
|----------|----------|
| Payment amount > outstanding balance | Allow. Show warning toast: "Payment exceeds pending amount by ₹{difference}." Outstanding goes negative (advance). See §6.3. |
| Payment amount = 0 | Block. Show toast: "Please enter amount". |
| Payment exactly clears balance | Outstanding becomes ₹0. Customer disappears from Due List. Remains in customer database. |
| Customer with zero outstanding receives another credit | They reappear in Due List. |

### 4.6 Undo Last Entry

**Purpose:** Shopkeepers make mistakes. Undo must be instant and effortless.

**Behavior:**

1. After **every** successful transaction save (sale, expense, credit given, credit received), a banner appears at the top of the screen:
   ```
   ┌─────────────────────────────────────────┐
   │  ✓ ₹500 Sale saved         [UNDO]      │
   └─────────────────────────────────────────┘
   ```
2. Banner is visible for **5 seconds**, then auto-dismisses.
3. If user taps **UNDO**:
   - The transaction is **soft-deleted**: `is_deleted = 1`, `deleted_at = current UTC timestamp`.
   - All computed values recompute from the event log (`WHERE is_deleted = 0`).
   - The in-memory Cash in Hand cache is decrementally updated (reverse the original increment).
   - Confirmation toast: "Entry removed".
   - Banner dismisses.
4. If user navigates away before tapping Undo, the banner dismisses and the transaction is final.
5. Only the **most recent** transaction can be undone. If two transactions are saved in quick succession, only the second is undoable.

**[Superseded by v2.0 rule]** ~~v1.0 §4.6 stated "hard-deleted from the database" and "No soft-delete needed."~~ Data-safety rules (Principle #1) override this. Soft-delete is mandatory.

**Banner Content by Transaction Type:**

| Type | Banner Text |
|------|-------------|
| `sale_cash` | "✓ ₹{amount} Cash Sale saved" |
| `sale_upi` | "✓ ₹{amount} UPI Sale saved" |
| `expense` | "✓ ₹{amount} Kharcha saved" |
| `credit_given` | "✓ ₹{amount} Udhar for {customer_name} saved" |
| `credit_received` | "✓ ₹{amount} Payment from {customer_name} saved" |

**Undo Button Debounce:** After the first tap on UNDO, the button is immediately disabled (grayed out). A 300ms debounce window prevents accidental double-taps.

**Undo State Lifecycle:**

| Event | Undo State |
|-------|------------|
| Transaction saved | Undo state = transaction ID. Timer starts (5s). |
| Second transaction saved before timer expires | Old undo state replaced by new. Old transaction becomes permanent. |
| Timer expires | Undo state cleared. Transaction permanent. |
| User taps Undo | Transaction soft-deleted. Undo state cleared. |
| User navigates to different tab | Undo state cleared. Transaction permanent. |
| App goes to background | Timer continues. If 5s elapsed on resume, undo state cleared. |
| App killed (force stop) | Undo state lost (volatile memory). Transaction persists (already saved). Correct behavior. |
| App crash | Same as app killed. Transaction persists. Undo state lost. Correct. |
| Backup restore | Undo state cleared immediately. See §10.7. |

**The undo state is NEVER persisted to disk. It exists only in volatile memory. This is intentional and correct.**

**Undo and Customer Orphans:**

When a `credit_given` transaction is undone:

1. Check if the `customer_id` referenced has any **other** non-deleted transactions.
2. If `COUNT(*) WHERE customer_id = X AND is_deleted = 0 AND id != undone_txn_id` == 0:
   - Set customer `status = "inactive"`. Do NOT soft-delete or hard-delete the customer record.
   - This customer remains visible in autocomplete with an `(inactive)` label or dimmed styling.
3. If the customer has other non-deleted transactions: do nothing. Customer persists as active.

**Cleanup of Soft-Deleted Rows:** A background cleanup job runs on app startup (non-blocking). It hard-deletes all rows where `is_deleted = 1` AND `deleted_at` is older than **24 hours**. This prevents unbounded growth of soft-deleted rows.

### 4.7 Due List

**Purpose:** Show all customers who owe money to the shopkeeper. Replaces the udhar notebook.

**Access:** Bottom navigation → **Due List** tab.

**Layout:**

| Element | Detail |
|---------|--------|
| **Header** | "Due List" + total outstanding: "Total Pending: ₹ XX,XXX" |
| **Search bar** | Filter by customer name. Visible at top. |
| **Customer list** | Scrollable list of all customers with outstanding > ₹0. |

**Each Customer Row:**

```
┌─────────────────────────────────────────────┐
│  Rajesh Kumar                    ₹ 2,500    │
│  Since 15 days                              │
└─────────────────────────────────────────────┘
```

| Field | Source |
|-------|--------|
| Customer name | `customer.name` |
| Pending amount | Computed: Σ `credit_given` − Σ `credit_received` for this customer (`WHERE is_deleted = 0`) |
| Days pending | Days since oldest unpaid `credit_given` for this customer, using FIFO walk (see §6.4), computed using **business dates** (see §7.4) |

**Sorting:** Primary sort: Largest outstanding amount first (descending).

**Visual Indicators:**

| Condition | Style |
|-----------|-------|
| Outstanding > 0 and days pending ≤ 30 | Normal (default text color) |
| Outstanding > 0 and days pending > 30 | **Red text** for amount and days |
| Outstanding = 0 | Not shown in default Due List view |
| Outstanding < 0 (advance) | Not shown in default "Show pending" view; shown in "Show all" toggle |

**"Show All" Toggle:** A toggle switch on Due List allows the shopkeeper to see all customers including settled (outstanding = 0) and advance (outstanding < 0) customers. Advance customers show "₹ X,XXX advance" with positive styling and absolute value.

**Empty State:**

```
┌─────────────────────────────────────────────┐
│                                             │
│       🎉  No pending dues!                   │
│       All customers have paid up.           │
│                                             │
└─────────────────────────────────────────────┘
```

### 4.8 Daily Closing Screen (Report)

**Purpose:** Give the shopkeeper a clear financial snapshot. Defaults to today's business day.

**Access:** Bottom navigation → **Report** tab.

**Layout:**

```
┌─────────────────────────────────────────────┐
│  Today's Summary — 22 Feb 2026              │
│                                             │
│  Total Sales          ₹ 15,200              │
│    ├─ Cash Sales      ₹ 12,000              │
│    └─ UPI Sales       ₹  3,200              │
│                                             │
│  Total Kharcha        ₹  1,800              │
│                                             │
│  Udhar Given          ₹  3,500              │
│  Udhar Received       ₹  2,000              │
│                                             │
│  ─────────────────────────────────          │
│  Cash in Hand         ₹ 12,450              │
│  ─────────────────────────────────          │
│                                             │
│       [◀ Previous Day]  [Next Day ▶]        │
└─────────────────────────────────────────────┘
```

**Data Definitions (All Computed):**

| Field | Computation |
|-------|-------------|
| Total Sales | Σ(`sale_cash` + `sale_upi`) for the selected **business_date**, `WHERE is_deleted = 0 AND type != 'opening_balance'` |
| Cash Sales | Σ(`sale_cash`) for the selected business_date |
| UPI Sales | Σ(`sale_upi`) for the selected business_date |
| Total Kharcha | Σ(`expense`) for the selected business_date |
| Udhar Given | Σ(`credit_given`) for the selected business_date |
| Udhar Received | Σ(`credit_received`) for the selected business_date |
| Cash in Hand | **All-time, unfiltered by date:** Σ `sale_cash` − Σ `expense` − Σ `credit_given` + Σ `credit_received` + `opening_balance` (`WHERE is_deleted = 0`). See §6.1. |

**Authoritative Daily Summary SQL Query (v1.2):**

```sql
-- Uses covering index idx_txn_business_date
SELECT
  type,
  SUM(amount) AS total_paisa,
  COUNT(*) AS count
FROM transactions
WHERE business_date = :selected_date   -- e.g., '2026-02-23'
AND is_deleted = 0
AND type != 'opening_balance'          -- Opening balance EXCLUDED from all date-scoped queries
GROUP BY type;
```

**Date Navigation:**

- Default: today's business date = `getBusinessDate(now(), closingHour)`.
- "Previous Day" / "Next Day" navigate by business date.
- "Next Day" is disabled if the next business date > today's business date.
- Date header shows business date in format: "DD MMM YYYY".

### 4.9 Edit Customer

**Purpose:** Allow the shopkeeper to correct a misspelled name or missing phone number without affecting transaction data.

**Access:** Due List → tap customer → Customer Detail Screen → **Edit** icon (pencil) in top-right.

**Editable Fields:**

| Field | Rules |
|-------|-------|
| Customer name | Required. 1–50 characters. Cannot be empty. |
| Phone number | Optional. Must be exactly 10 digits if provided. |

**Non-Editable:** Transaction history (immutable). Outstanding balance (computed).

**Save Behavior:**

- Tap Save → customer record updated.
- All references to this customer across the app reflect the new name immediately (since transactions reference `customer_id`, not name string).
- **Renaming a customer NEVER merges their ledger with another customer.** Even if the new name exactly matches an existing customer's name, the two remain separate entities with distinct `id` values.
- No undo for customer edits.

**Duplicate Name Handling:** If edited name matches an existing customer: show warning "A customer with this name already exists. Save anyway?" Allow saving. Ledgers never merge.

### 4.10 First Launch Experience

**Trigger:** First app install, no existing data.

**Required functional elements:**

- App identity display.
- "Start Fresh" action → navigates to empty Home screen (with optional opening balance step).
- "I have a backup" action → goes to restore flow.
- No login, OTP, or account creation required.

---

## 5. Transaction Model

### 5.1 Design Philosophy

The system uses an **event-sourced** transaction log. Current state (balances, totals) is always derived from the complete history of active (non-deleted) transactions. No running balances are stored anywhere. This ensures:

- Auditability
- Correctness (recomputation from source of truth)
- Simplicity of undo operations (soft-delete exclusion)
- No balance corruption

**Transaction Immutability:** Existing transactions are NEVER modified after creation. Payments reduce outstanding via new `credit_received` transactions. There is no "edit transaction" feature beyond the 5-second undo window.

### 5.2 Database Schema

**Technology:**

- SQLite via `sqflite` Flutter package.
- WAL (Write-Ahead Logging) mode: `PRAGMA journal_mode = WAL`.
- Foreign keys enforced: `PRAGMA foreign_keys = ON` — executed on **every** database connection open.
- Synchronous mode: `PRAGMA synchronous = NORMAL`.
- Page size: `PRAGMA page_size = 4096`.
- Auto-vacuum: `PRAGMA auto_vacuum = INCREMENTAL`.
- All writes wrapped in SQLite transactions (BEGIN/COMMIT) for atomicity.

**Schema Version:** `PRAGMA user_version = 2` (as of v1.2 of the app).

**CREATE TABLE Statements:**

```sql
CREATE TABLE IF NOT EXISTS customers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    uuid TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL CHECK(length(trim(name)) > 0 AND length(name) <= 50),
    phone TEXT CHECK(
        phone IS NULL OR (
            length(phone) = 10
            AND phone GLOB '[6-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'
        )
    ),
    status TEXT NOT NULL DEFAULT 'active' CHECK(status IN ('active', 'inactive')),
    is_deleted INTEGER NOT NULL DEFAULT 0 CHECK(is_deleted IN (0, 1)),
    created_at TEXT NOT NULL,   -- UTC, ISO 8601
    updated_at TEXT NOT NULL    -- UTC, ISO 8601
);

CREATE TABLE IF NOT EXISTS transactions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    uuid TEXT NOT NULL UNIQUE,
    type TEXT NOT NULL CHECK(type IN (
        'sale_cash', 'sale_upi', 'expense',
        'credit_given', 'credit_received', 'opening_balance'
    )),
    amount INTEGER NOT NULL CHECK(amount > 0 AND amount <= 9999999999),
    -- amount stored in paisa. ₹1.00 = 100. Max ₹99,99,999.99 = 9999999999.
    customer_id INTEGER REFERENCES customers(id),
    category TEXT,
    note TEXT CHECK(note IS NULL OR length(note) <= 100),
    is_deleted INTEGER NOT NULL DEFAULT 0 CHECK(is_deleted IN (0, 1)),
    deleted_at TEXT,                -- NULL unless soft-deleted. UTC ISO 8601.
    created_at TEXT NOT NULL,       -- UTC, ISO 8601 (logical monotonic time per TIR-1)
    local_created_at TEXT NOT NULL, -- Device local time. ISO 8601 with timezone offset.
    business_date TEXT NOT NULL,    -- YYYY-MM-DD. Pre-computed from local_created_at and closing_hour.

    -- Referential integrity: credit transactions MUST have a customer
    CHECK(
        (type IN ('credit_given', 'credit_received') AND customer_id IS NOT NULL)
        OR
        (type NOT IN ('credit_given', 'credit_received') AND customer_id IS NULL)
    ),

    -- Expense transactions MUST have a category
    CHECK(
        (type = 'expense' AND category IS NOT NULL AND length(trim(category)) > 0)
        OR
        (type != 'expense' AND category IS NULL)
    )
);

CREATE TABLE IF NOT EXISTS expense_categories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    uuid TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL CHECK(length(trim(name)) > 0 AND length(name) <= 20),
    is_active INTEGER NOT NULL DEFAULT 1 CHECK(is_active IN (0, 1)),
    sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS app_metadata (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

-- Default metadata
INSERT OR IGNORE INTO app_metadata (key, value) VALUES ('schema_version', '2');
INSERT OR IGNORE INTO app_metadata (key, value) VALUES ('last_backup_at', '');
INSERT OR IGNORE INTO app_metadata (key, value) VALUES ('install_id', '');
INSERT OR IGNORE INTO app_metadata (key, value) VALUES ('opening_balance_set', '0');
INSERT OR IGNORE INTO app_metadata (key, value) VALUES ('business_day_closing_hour', '4');
INSERT OR IGNORE INTO app_metadata (key, value) VALUES ('integrity_log', '[]');
INSERT OR IGNORE INTO app_metadata (key, value) VALUES ('clean_shutdown', '1');

-- Default expense categories
INSERT OR IGNORE INTO expense_categories (uuid, name, is_active, sort_order)
VALUES ('cat-chai', 'Chai/Snacks', 1, 1);
INSERT OR IGNORE INTO expense_categories (uuid, name, is_active, sort_order)
VALUES ('cat-transport', 'Transport', 1, 2);
INSERT OR IGNORE INTO expense_categories (uuid, name, is_active, sort_order)
VALUES ('cat-rent', 'Rent', 1, 3);
INSERT OR IGNORE INTO expense_categories (uuid, name, is_active, sort_order)
VALUES ('cat-supplier', 'Supplier Payment', 1, 4);
INSERT OR IGNORE INTO expense_categories (uuid, name, is_active, sort_order)
VALUES ('cat-staff', 'Staff', 1, 5);
INSERT OR IGNORE INTO expense_categories (uuid, name, is_active, sort_order)
VALUES ('cat-other', 'Other', 1, 6);
```

### 5.3 Why INTEGER Primary Keys

| Factor | UUID v4 (TEXT) | INTEGER AUTOINCREMENT |
|--------|---------------|----------------------|
| Storage per row | 36 bytes | 8 bytes |
| Index size at 100k rows | ~3.6 MB per index | ~800 KB per index |
| Insert performance | Randomized B-tree (fragmentation) | Sequential append (optimal) |
| Join performance | String comparison | Integer comparison (5–10× faster) |
| SQLite `rowid` alias | No | Yes (IS the rowid) |

**Decision:** Use `INTEGER PRIMARY KEY AUTOINCREMENT` for internal use. Retain a `uuid TEXT` column (UNIQUE indexed) for backup file identification and future cloud sync compatibility.

### 5.4 Why UTC Storage + Local Time

- `created_at` (UTC) is stable across timezone changes. It is the field used for clock-regression detection (TIR-1).
- `local_created_at` preserves the user's local time at entry for display in Daily Summary and transaction history.
- `business_date` is pre-computed from `local_created_at` at insert time using the `getBusinessDate()` function (§7.3).

### 5.5 Monetary Storage Rules

**All amounts stored as INTEGER in paisa.** ₹1.00 = 100 paisa. ₹99,99,999.99 = 9999999999 paisa.

- User enters ₹10.50 → stored as `1050`.
- User enters ₹100 → stored as `10000`.
- Display: integer paisa divided by 100. Format: `₹ XX,XX,XXX.PP`.
- If paisa portion is `.00`, display as whole number: `₹ 12,450`.
- Truncation rule: if a fractional paisa value is ever computed (impossible with pure integer arithmetic, but as a safeguard), truncate down.
- **Rationale:** Eliminates IEEE 754 floating-point rounding errors permanently. Integer arithmetic is exact. SUM operations over millions of integers produce exact results.

### 5.6 Computed Values (NEVER Stored)

| Value | Computation |
|-------|-------------|
| **Cash in Hand** | See §6.1 — all-time formula |
| **Customer Outstanding** | Σ `credit_given` − Σ `credit_received` for customer (`WHERE is_deleted = 0`) |
| **Daily Sales Total** | Σ `sale_cash` + Σ `sale_upi` for a given `business_date` |
| **Daily Expense Total** | Σ `expense` for a given `business_date` |
| **Days Pending** | FIFO walk — see §6.4 |

### 5.7 Transaction Ordering Guarantee (CT-3)

**The `id` column (INTEGER PRIMARY KEY AUTOINCREMENT) is the sole authoritative ordering mechanism. No exceptions.**

```sql
ORDER BY id ASC    -- chronological (oldest first)
ORDER BY id DESC   -- reverse chronological (newest first)
```

**Do NOT use:**

```sql
ORDER BY created_at ASC              -- WRONG: ties are non-deterministic
ORDER BY local_created_at ASC        -- WRONG: same problem
ORDER BY created_at ASC, id ASC      -- UNNECESSARY: id alone is sufficient and faster
```

**This rule applies to:** Transaction history displays, FIFO walk, Daily Summary, backup export, undo determination, clock change scenarios. No query in the codebase may order transactions by timestamp.

**[Superseded by v2.0 rule]** ~~v1.0 Appendix A step 1 specified "sorted by date ascending."~~ Replaced by `ORDER BY id ASC` per CT-3.

### 5.8 Indexes

| Table | Index Name | Columns | Type | Purpose |
|-------|-----------|---------|------|---------|
| Transaction | `idx_txn_type_amount` | `(type, amount) WHERE is_deleted = 0` | Partial covering | Cash in Hand aggregation — index-only scan |
| Transaction | `idx_txn_customer_type` | `(customer_id, type, amount) WHERE is_deleted = 0` | Partial covering | Customer outstanding — index-only scan |
| Transaction | `idx_txn_business_date` | `(business_date, type, amount) WHERE is_deleted = 0` | Partial covering | Daily Summary by business_date — index-only scan |
| Transaction | `idx_txn_deleted` | `(is_deleted, deleted_at)` | Standard | Cleanup job: find soft-deleted rows older than 24h |
| Transaction | `idx_txn_uuid` | `(uuid)` | Unique | Backup file cross-reference |
| Customer | `idx_customer_name` | `(name) WHERE is_deleted = 0` | Partial | Autocomplete search |
| Customer | `idx_customer_uuid` | `(uuid)` | Unique | Backup file cross-reference |

```sql
CREATE INDEX IF NOT EXISTS idx_txn_type_amount
ON transactions(type, amount) WHERE is_deleted = 0;

CREATE INDEX IF NOT EXISTS idx_txn_customer_type
ON transactions(customer_id, type, amount) WHERE is_deleted = 0;

CREATE INDEX IF NOT EXISTS idx_txn_business_date
ON transactions(business_date, type, amount) WHERE is_deleted = 0;

CREATE INDEX IF NOT EXISTS idx_txn_deleted
ON transactions(is_deleted, deleted_at);

CREATE UNIQUE INDEX IF NOT EXISTS idx_txn_uuid ON transactions(uuid);
CREATE INDEX IF NOT EXISTS idx_customer_name ON customers(name) WHERE is_deleted = 0;
CREATE UNIQUE INDEX IF NOT EXISTS idx_customer_uuid ON customers(uuid);
```

### 5.9 Constraint Enforcement Summary

| Constraint | Enforcement Layer | Notes |
|-----------|-----------------|-------|
| Amount > 0 | SQLite CHECK + app-layer | Double enforcement |
| Amount ≤ 9999999999 paisa | SQLite CHECK + keypad prevents input | ~₹99.99 lakh |
| Customer name non-empty | SQLite CHECK + UI validation | Trim before check |
| Phone format | SQLite CHECK (GLOB) + UI validation | Indian mobile: starts with 6–9, exactly 10 digits |
| Credit → customer required | SQLite CHECK | DB-level guarantee |
| Expense → category required | SQLite CHECK | DB-level guarantee |
| Foreign key integrity | `PRAGMA foreign_keys = ON` | On every connection |
| Soft-delete values | SQLite CHECK (0 or 1) | Prevents invalid flag values |
| Transaction type | SQLite CHECK (IN list) | Prevents invalid types |

### 5.10 Schema Versioning and Migration

**Version tracking:** `PRAGMA user_version` stores the schema version integer.

| App Version | Schema Version | Changes |
|------------|---------------|---------|
| v1.0 (not released) | — | Original PRD spec (UUID keys, REAL amounts) |
| v1.1 | 1 | Production schema: INTEGER keys, paisa amounts, soft-delete, opening_balance, UTC timestamps |
| v1.2 | 2 | Added `business_date` column, `business_day_closing_hour` and `integrity_log` to app_metadata |

**Migration from schema version 1 to 2:**

```sql
ALTER TABLE transactions ADD COLUMN business_date TEXT;

-- Backfill using default closing_hour = 4
UPDATE transactions SET business_date =
  CASE
    WHEN CAST(strftime('%H', local_created_at) AS INTEGER) < 4
    THEN date(local_created_at, '-1 day')
    ELSE date(local_created_at)
  END
WHERE business_date IS NULL;

PRAGMA user_version = 2;
```

**Migration Script Rules:**

1. Every migration is a function `migrationV{N}(Database db)` that upgrades from version N-1 to N.
2. Migrations are **idempotent** — running the same migration twice does not cause errors.
3. Migrations MUST be wrapped in a SQLite transaction. If any statement fails, the entire migration rolls back.
4. Migrations MUST NOT drop columns or delete data. Only additive changes.
5. After migration, run `PRAGMA integrity_check` to verify consistency.

**Migration Framework (Dart pseudocode):**

```dart
Future<Database> openDatabase() async {
  return openDatabase(
    path: 'simpleledger.db',
    version: CURRENT_SCHEMA_VERSION,  // 2 for v1.2
    onCreate: (db, version) {
      // Run all CREATE TABLE statements
      // Run all CREATE INDEX statements
    },
    onUpgrade: (db, oldVersion, newVersion) {
      for (int v = oldVersion + 1; v <= newVersion; v++) {
        runMigration(db, v);
      }
    },
    onOpen: (db) {
      // MUST run on every open
      db.execute('PRAGMA foreign_keys = ON');
      db.execute('PRAGMA journal_mode = WAL');
      db.execute('PRAGMA synchronous = NORMAL');
    },
  );
}
```

---

## 6. Financial Calculation Rules

### 6.1 Cash in Hand Formula

**This value is NEVER stored. It is computed from the transaction log and maintained in an in-memory cache.**

**Authoritative SQL Formula:**

```sql
SELECT
  COALESCE(SUM(CASE WHEN type = 'sale_cash' AND is_deleted = 0 THEN amount ELSE 0 END), 0)
  + COALESCE(SUM(CASE WHEN type = 'credit_received' AND is_deleted = 0 THEN amount ELSE 0 END), 0)
  + COALESCE(SUM(CASE WHEN type = 'opening_balance' AND is_deleted = 0 THEN amount ELSE 0 END), 0)
  - COALESCE(SUM(CASE WHEN type = 'expense' AND is_deleted = 0 THEN amount ELSE 0 END), 0)
  - COALESCE(SUM(CASE WHEN type = 'credit_given' AND is_deleted = 0 THEN amount ELSE 0 END), 0)
AS cash_in_hand_paisa
FROM transactions;
```

**Critical rules:**

1. `sale_upi` is **NEVER** included in this formula. It must not appear in any CASE branch.
2. `opening_balance` adds to Cash in Hand.
3. `is_deleted = 0` filter ensures soft-deleted (undone) transactions are excluded.
4. `COALESCE(..., 0)` ensures NULL (no transactions of that type) doesn't propagate NULLs.
5. Result is in paisa. Display layer divides by 100.
6. This formula includes `opening_balance`. It is the **only** formula that does so.

**Cash in Hand Caching Strategy:**

1. Maintain an **in-memory balance cache** (a single integer: `_cachedCashInHandPaisa`).
2. On app launch: compute from database via full query above. Store in memory.
3. On every transaction save: incrementally update the cache:

   | Transaction Type | Cache Update | Undo Reversal |
   |-----------------|-------------|--------------|
   | `sale_cash` | `cache += amount` | `cache -= amount` |
   | `sale_upi` | No change | No change |
   | `expense` | `cache -= amount` | `cache += amount` |
   | `credit_given` | `cache -= amount` | `cache += amount` |
   | `credit_received` | `cache += amount` | `cache -= amount` |
   | `opening_balance` | `cache += amount` | (Not undoable) |

4. On app resume from background: do NOT recompute. Trust the cache.
5. On restore from backup: invalidate cache. Recompute from database.
6. The cache is **never** written to the database or included in backup files.
7. If the cache is not yet initialized, show a loading indicator — **never** show ₹0 as a placeholder.

**Cash-in-Hand Trust Rule (CT / §23.F): Mandatory Verification**

Every 100th transaction save triggers a background verification:

```
transactionCounter = 0   // volatile, in memory

function onTransactionSaved():
    transactionCounter += 1
    if transactionCounter % 100 == 0:
        scheduleVerification()

function scheduleVerification():
    cachedValue = getCachedCashInHand()
    computedValue = [run full SQL formula above]

    if cachedValue != computedValue:
        // Log the discrepancy to app_metadata key 'integrity_log'
        // Auto-correct the cache IMMEDIATELY
        setCachedCashInHand(computedValue)
        // Trigger UI refresh
        notifyHomeScreenRefresh()
        // Do NOT show error to user. This is a silent self-heal.
```

Additionally, the cache MUST be fully recomputed from the database in these events: app cold launch, backup restore, schema migration, soft-delete cleanup job, database integrity check failure.

**Negative Cash in Hand:** If Cash in Hand is negative (expenses + credit given exceed cash received), display with red/warning color and minus sign: "Cash in Hand: −₹ 2,500". No blocking. No error. This is a valid real-world state.

### 6.2 Opening Balance Exclusion (CT-1)

**Rule:** Opening balance is NOT a business transaction. It represents starting cash at app installation. It participates **only** in the all-time Cash in Hand formula.

**Opening balance MUST be excluded from:**

- Daily Summary (§4.8)
- All reports and date-range queries
- Business-day grouping
- Income statistics

**Required query filter:** All date-scoped or report-scoped aggregation queries MUST include `AND type != 'opening_balance'`.

**Only the all-time Cash in Hand formula includes `opening_balance`.**

**[Superseded by v2.0 rule]** ~~v1.0 §4.8 Data Definitions table omitted the exclusion filter.~~ The exclusion is now mandatory in all date-scoped queries.

### 6.3 Overpayment / Advance Handling (CT-6 / §23.B)

**Rule:** Negative outstanding = advance. There are no separate "advance" and "dues" buckets. A single scalar is computed from the event log.

**Outstanding Balance Formula:**

```
Outstanding(customer) = Σ credit_given(customer, active) − Σ credit_received(customer, active)

If Outstanding < 0 → customer has advance of |Outstanding|.
    Future credit_given automatically consumes advance before creating new dues.
If Outstanding > 0 → customer owes Outstanding.
If Outstanding = 0 → settled.
```

**Advance consumption is implicit via arithmetic. No special transaction types, no splitting.**

**Advance Consumption Example (all amounts in paisa):**

| Step | Transaction | Outstanding | Explanation |
|------|-------------|-------------|-------------|
| 1 | `credit_given`: 50000 | +50000 | Customer owes ₹500 |
| 2 | `credit_received`: 80000 | −30000 | Overpaid by ₹300 (advance) |
| 3 | `credit_given`: 20000 | −10000 | New credit consumed by advance. ₹100 advance remains. |
| 4 | `credit_given`: 15000 | +5000 | Advance consumed + ₹50 new dues. |

**SQL — Outstanding Query:**

```sql
SELECT
  COALESCE(SUM(CASE WHEN type = 'credit_given' THEN amount ELSE 0 END), 0)
  - COALESCE(SUM(CASE WHEN type = 'credit_received' THEN amount ELSE 0 END), 0)
AS outstanding_paisa
FROM transactions
WHERE customer_id = :customer_id
AND is_deleted = 0;
```

Result may be positive (dues), zero (settled), or negative (advance). **All three are valid states. No clamping.**

**Display Rules by Outstanding State:**

| Screen | Outstanding > 0 | Outstanding = 0 | Outstanding < 0 |
|--------|----------------|----------------|----------------|
| Due List (default "Show pending") | Shown, warning style | Hidden | Hidden |
| Due List ("Show all" toggle) | Shown, warning style | Shown, "Settled" label | Shown, "₹X advance" positive style |
| Customer Detail | "Pending: ₹X,XXX" (warning style) | "Settled" | "Advance: ₹X,XXX" (positive style, absolute value) |
| Receive Payment button | Visible | Hidden | Hidden |

### 6.4 Days Pending — FIFO Walk Algorithm

**Purpose:** "Days Pending" for a customer represents how long their oldest outstanding credit has been unpaid.

**Authoritative Algorithm (v1.2 — uses business dates and `ORDER BY id ASC`):**

```
function computeDaysPending(customerId, closingHour):
    // Step 1: Get all credit_given for this customer, oldest insertion order first
    credits = SELECT amount, local_created_at, business_date
              FROM transactions
              WHERE customer_id = :customerId
              AND type = 'credit_given'
              AND is_deleted = 0
              ORDER BY id ASC    -- authoritative ordering per CT-3

    // Step 2: Get total payments received (single scalar)
    totalReceived = SELECT COALESCE(SUM(amount), 0)
                    FROM transactions
                    WHERE customer_id = :customerId
                    AND type = 'credit_received'
                    AND is_deleted = 0

    // Step 3: FIFO walk — consume credits from oldest
    remainingPayment = totalReceived
    oldestUnpaidBusinessDate = NULL

    for each credit in credits:
        if remainingPayment >= credit.amount:
            remainingPayment -= credit.amount
            // This credit is fully paid. Continue.
        else:
            // This credit is partially or fully unpaid.
            oldestUnpaidBusinessDate = credit.business_date
            break

    // Step 4: Calculate days using business dates
    if oldestUnpaidBusinessDate is NULL:
        return 0
    else:
        todayBusinessDate = getBusinessDate(now(), closingHour)
        return daysBetween(oldestUnpaidBusinessDate, todayBusinessDate)
```

**[Superseded by v2.0 rule]** ~~v1.0 Appendix A used "sorted by date ascending" and calendar-day `daysBetween`.~~ Replaced by `ORDER BY id ASC` and business-date delta per CT-3 and §23.A.

**Edge Cases Handled:**

| Scenario | Result |
|----------|--------|
| No `credit_given` transactions | Loop doesn't execute → returns 0 |
| No `credit_received` transactions | `totalReceived` = 0 → first credit becomes oldest unpaid |
| Overpayment (received > given) | `remainingPayment` consumes all credits → returns 0 |
| Multiple partial payments | All payments aggregated into `totalReceived` scalar, then consumed FIFO |
| Undo of a `credit_received` | `is_deleted = 0` filter excludes it; `totalReceived` decreases |
| Undo of a `credit_given` | `is_deleted = 0` filter excludes it; that credit is removed from walk |

---

## 7. Temporal Integrity Rules

### 7.1 Purpose

SimpleLedger is a **financial ledger**, not a clock recorder. Financial continuity is more important than wall-clock accuracy. The rules in this section prevent temporal corruption caused by incorrect device clocks, timezone changes, or manual clock manipulation. They ensure that the transaction timeline never moves backward, regardless of what the device clock reports.

### 7.2 Core Principle

**Transactions always save. The clock NEVER blocks a transaction. The `id` column (INTEGER PRIMARY KEY AUTOINCREMENT) is the authoritative ordering truth.**

A shopkeeper cares that:
1. Every entry is saved — always, without exception.
2. Entries appear in the order they were made — never shuffled.
3. Today's summary shows today's work — not yesterday's or tomorrow's.
4. Days pending is always correct — never negative, never jumping backward.

**Design principle: When the device clock disagrees with the ledger's timeline, the ledger wins.**

### 7.3 Business Day Logic (§23.A)

Small shops do not operate by calendar day. A sale at 12:30 AM belongs to the previous business day.

**Business Day** is defined as the period from `closing_hour` on one calendar day to `closing_hour` on the next.

| Parameter | Value |
|-----------|-------|
| Default closing hour | `04:00` local time (24-hour format) |
| Configurable | Yes — stored in `app_metadata`, key: `business_day_closing_hour` |
| Valid range | `00` to `06` (integer hour, full hours only) |
| Access | More → Settings → "Day Ends At" (dropdown: 12 AM to 6 AM) |

**`getBusinessDate()` Function:**

```
function getBusinessDate(localTimestamp, closingHour):
    calendarDate = localTimestamp.date()    // e.g., 2026-02-23
    localHour    = localTimestamp.hour()    // 0–23

    if localHour < closingHour:
        return calendarDate - 1 day         // belongs to PREVIOUS calendar day's business day
    else:
        return calendarDate                 // belongs to THIS calendar day's business day
```

**Examples (closing hour = 04:00):**

| Local Time | Calendar Date | Business Date | Rationale |
|-----------|--------------|--------------|-----------|
| 2026-02-23 09:15:00 | Feb 23 | **Feb 23** | 09:00 ≥ 04:00 → same day |
| 2026-02-23 23:45:00 | Feb 23 | **Feb 23** | 23:00 ≥ 04:00 → same day |
| 2026-02-24 00:30:00 | Feb 24 | **Feb 23** | 00:00 < 04:00 → previous day |
| 2026-02-24 03:59:59 | Feb 24 | **Feb 23** | 03:00 < 04:00 → previous day |
| 2026-02-24 04:00:00 | Feb 24 | **Feb 24** | 04:00 ≥ 04:00 → new business day starts |

**`business_date` Column:** Pre-computed at insert time using `getBusinessDate(local_created_at, business_day_closing_hour)`. Stored as `YYYY-MM-DD` text. **Immutable after insertion** — even if the user changes the closing hour, existing transactions retain their original `business_date`. Only new transactions use the updated setting.

**Changing Closing Hour:** When the user changes `business_day_closing_hour`, existing transactions are NOT retroactively updated. This may cause transitional inconsistency around the change boundary, which is acceptable.

**Cash in Hand — UNAFFECTED by business_date:** Cash in Hand remains an all-time aggregation. It does not use `business_date`. No business_date filter is applied to the Cash in Hand SQL query.

**First Install Exception (CT-8):** If the app is first installed between midnight and the closing hour, the `opening_balance` transaction's `business_date` MUST be set to the **current calendar date**, NOT the previous business day. This is a one-time exception.

### 7.4 Monotonic Transaction Time (TIR-1)

**Rule:** The financial ledger MUST never move backward in time. Every new transaction's `created_at` (UTC) must be ≥ the previous transaction's `created_at`.

**Algorithm:**

```
function getLogicalTransactionTime():
    device_time = getCurrentDeviceTimestampUTC()
    last_txn_time = SELECT MAX(created_at) FROM transactions WHERE is_deleted = 0

    if last_txn_time is NULL:
        return device_time                          // First transaction — use device time

    if device_time >= last_txn_time:
        return device_time                          // Normal case

    if device_time < last_txn_time:
        // CLOCK REGRESSION DETECTED
        logical_time = last_txn_time + 1 second
        return logical_time
```

**On clock regression:**

1. Compute `logical_time = last_txn_time + 1 second`.
2. Save transaction with `created_at = logical_time` (UTC).
3. Save `local_created_at` from actual device clock (for display/debugging).
4. **Do NOT block the user.** Transaction is saved immediately.
5. Show **one-time warning toast:** "Phone time appears incorrect. Entries will still be recorded safely." (4 seconds, non-blocking). Shown once per regression event, not on every subsequent transaction while clock remains behind.

**`business_date` MUST be computed from the logical `created_at` value** (after TIR-1 adjustment), not from the raw device clock. If `business_date` were computed from the raw device clock while `created_at` uses logical time, the mismatch would corrupt Daily Summary and Days Pending.

```
logical_time = getLogicalTransactionTime()         // TIR-1
local_logical_time = convertToLocalTimezone(logical_time)
business_date = getBusinessDate(local_logical_time, closingHour)
```

### 7.5 Clock Change Handling (CT-7 / §23.E)

**Clock change detection algorithm:**

```
function onTransactionSave(newTransaction):
    lastTxnTime = SELECT created_at FROM transactions
                  WHERE is_deleted = 0
                  ORDER BY id DESC LIMIT 1

    if lastTxnTime is NOT NULL:
        timeDelta = newTransaction.created_at - lastTxnTime   // in seconds

        if timeDelta < -3600:
            // Clock moved backward by more than 1 hour
            showWarningToast(
                "Your phone's time seems incorrect. "
                "Entry saved, but it may appear under the wrong day."
            )
            logClockAnomaly(lastTxnTime, newTransaction.created_at)

    // ALWAYS save the transaction regardless of clock state
    INSERT INTO transactions (...)
```

**Scenario Matrix:**

| Scenario | Transaction Saved? | Warning Shown? | Ordering Impact |
|----------|-------------------|----------------|----------------|
| Timezone change (e.g., IST → GST) | Yes | No | None — UTC stable. `id` ordering unaffected. |
| Manual clock forward | Yes | No | Transactions appear in future business date. Corrects when real time catches up. |
| Manual clock backward ≤ 1 hour | Yes | No | Minor anomaly. `id` ordering still correct. |
| Manual clock backward > 1 hour | Yes | **Yes — one-time** | `id` ordering overrides timestamp. Daily Summary may show wrong business day. |
| DST transition | Yes | No | UTC unaffected. `id` ordering correct. |
| NTP auto-correction | Yes | No (< 1 hour) | Negligible. |

**Warning toast behavior:** Shown once per backward clock event. Suppressed for subsequent transactions while clock remains behind. Duration: 4 seconds. Non-blocking.

**Impact on Cash in Hand:** None. Cash in Hand is all-time aggregation. Does not use dates. Clock state is irrelevant.

---

## 8. Customer Identity Rules

### 8.1 Customer Identity Protection (CT-5)

**Customer name is NOT identity. The immutable `customer.id` (INTEGER PRIMARY KEY AUTOINCREMENT) is identity.**

| Rule | Detail |
|------|--------|
| Transaction binding | All `credit_given` and `credit_received` transactions store `customer_id` (integer). Transactions bind to `customer_id`, **never** to name string. |
| Renaming safety | Renaming a customer updates only `customer.name`. It does **not** merge ledgers with another customer who happens to share the same name. |
| Payment binding | Payments (`credit_received`) bind to `customer_id`. Even if two customers share a name, their ledgers remain completely separate. |
| Autocomplete disambiguation | When multiple customers have similar names, autocomplete MUST show disambiguating context: last due amount OR last transaction date OR phone number. |

### 8.2 Customer Record Permanence (FEIR-1)

**Rule:** A customer record that has ever been referenced by at least one financial transaction MUST be permanent. It can never be hard-deleted from the database.

| Condition | Behavior |
|-----------|----------|
| Customer has ≥ 1 transaction (active or soft-deleted) | Customer record is **permanent**. Cannot be hard-deleted, ever. |
| Customer has 0 transactions and was never referenced | May be hard-deleted (user created customer by accident and never saved a transaction). |

**Rationale:** Financial transactions reference `customer_id`. If the customer record is deleted, the transaction log has dangling references — violating referential integrity and making the ledger unauditable.

### 8.3 Undo Does NOT Delete Customer (FEIR-2)

**Rule:** When the user undoes a `credit_given` transaction that was the first and only transaction for a customer, the customer record MUST NOT be deleted or soft-deleted.

Instead:
1. The transaction is soft-deleted (per §4.6).
2. The customer record transitions to `status = "inactive"`.
3. The customer remains in the database with all fields intact.

**[Superseded by v2.0 rule]** ~~Any earlier description stating "soft-delete the customer record" when undoing the first transaction is overridden.~~ The customer is archived (inactive), never deleted.

### 8.4 Inactive Customer Behavior (FEIR-3)

A customer with `status = "inactive"` (no active transactions, all transactions undone or settled):

| Context | Visibility |
|---------|-----------|
| Default Due List | **Hidden** (outstanding = 0 or all transactions undone) |
| Search / autocomplete during credit entry | **Visible** — with `(inactive)` label or dimmed styling |
| Customer Detail screen | **Visible** — full transaction history retained |
| Backup export | **Included** — inactive customers are always exported |
| Reports and summaries | Not counted in active customer statistics |

**Reactivation:** If a new `credit_given` transaction is saved for an inactive customer, the customer automatically becomes `status = "active"` again.

### 8.5 Autocomplete — Prefer Existing Identity (FEIR-4)

When the user enters a name in the credit flow and the name matches an inactive customer:

1. The autocomplete dropdown MUST show the inactive customer as a suggestion.
2. The suggestion MUST be visually distinguished (e.g., dimmed text, "(inactive)" suffix, or "last active: DD MMM YYYY" label).
3. If the user selects the inactive customer, the new transaction binds to the **existing `customer_id`**.
4. If the user explicitly types a new name that does not match any existing customer, a new customer is created.

**Priority:** Existing inactive customer match > creating a new customer with the same name.

### 8.6 Historical Guarantee (FEIR-5)

Transaction history MUST remain permanently associated with the original `customer_id`.

| Action | Effect on History |
|--------|-----------------|
| Rename customer | Transaction history stays linked to same `customer_id`. Only display name changes. |
| Archive (make inactive) | Transaction history fully preserved and accessible. |
| Undo a transaction | Transaction is soft-deleted. Customer record persists. |
| Restore backup | Customer records and transaction links fully restored. |

**No action may sever the link between a transaction and its `customer_id`.** This is the foundational audit guarantee of the ledger.

### 8.7 UI Terminology — Archive, Not Delete (FEIR-6)

All user-facing actions related to removing a customer from active view MUST use the term **"Archive"**, never "Delete."

| Prohibited Term | Required Term |
|----------------|--------------|
| "Delete Customer" | **"Archive Customer"** |
| "Remove Customer" | **"Archive Customer"** |
| "Customer deleted" (toast) | **"Customer archived"** |

**Archiving behavior:** Customer moves to inactive status. Hidden from default Due List. Remains searchable. All financial history permanently retained. Reversible by recording a new transaction.

---

## 9. Data Persistence & Database Rules

### 9.1 Atomic Write Requirements

Every write operation that involves multiple statements MUST be wrapped in a single SQLite transaction:

| Operation | Statements | Atomicity Requirement |
|-----------|------------|----------------------|
| Save `credit_given` with new customer | INSERT customer + INSERT transaction | MUST be atomic. If transaction INSERT fails, customer INSERT rolls back. |
| Undo `credit_given` with orphan customer | UPDATE transaction (soft-delete) + UPDATE customer (set inactive) | MUST be atomic. |
| Restore from backup | DELETE all tables + INSERT all data | MUST be atomic. If any INSERT fails, DELETE rolls back to preserve original data. |
| Schema migration | ALTER/CREATE + data transforms | MUST be atomic per migration step. |

### 9.2 Crash Safety

**WAL mode guarantees atomicity.** Either the full SQLite transaction committed or nothing did. On next open, SQLite automatically rolls back uncommitted WAL entries.

**Clean shutdown detection:**

- On app startup: set `app_metadata['clean_shutdown'] = '0'`.
- On app graceful close (pause/stop): set `app_metadata['clean_shutdown'] = '1'`.
- On next startup: if `clean_shutdown == '0'`, the previous session was a crash/kill → run integrity check.

**WAL Checkpoint Strategy:**

- Explicit checkpoint (`PRAGMA wal_checkpoint(TRUNCATE)`) on:
  - App moving to background (if WAL size > 100 pages)
  - Before backup export (ensure all data is in main DB file)
  - After restore (clean WAL state)

### 9.3 Startup Integrity Checks

On every app startup:

1. Open database (SQLite auto-recovers from WAL if needed).
2. Run `PRAGMA quick_check` (every cold start — O(1) header check). If fails → warn user immediately.
3. Run `PRAGMA foreign_key_check` (background, non-blocking, after crash/kill detection). If violations → log them.
4. Run `PRAGMA integrity_check` (background, only if `clean_shutdown == '0'`). If fails → surface warning to user.

**On integrity check failure:** Show warning banner: "SimpleLedger has detected a data issue. Please back up your data immediately." Show "Backup Now" button. Do not block app usage.

### 9.4 No History Rewriting

**Principle #1 (Never Lose Data) forbids any operation that destroys or rewrites committed transaction history. Specifically:**

- Transactions are never edited after the 5-second undo window expires.
- The undo operation uses soft-delete (`is_deleted = 1`), not hard-delete.
- Hard-deletes are only permitted by the background cleanup job for soft-deleted rows older than 24 hours, or for customers with zero transactions ever.
- Backup restore is the only operation that replaces the entire database, and it requires explicit user confirmation.

### 9.5 Disk-Full Handling

| Scenario | Expected Behavior |
|----------|------------------|
| SQLite write fails due to `SQLITE_FULL` | Catch error. Show toast: "Phone storage is full. Please free up space." Transaction NOT saved. Amount display retains entered value (do not reset). |
| Backup export fails due to no storage | Catch error. Show toast: "Not enough space to create backup. Free up space and try again." |
| Continuous disk-full state | Show persistent (non-dismissable) banner: "⚠ Storage full — entries cannot be saved." Banner dismisses when a write succeeds. |

### 9.6 Concurrency Rules

| Situation | Handling |
|-----------|---------|
| User saves transaction while home screen is rendering | WAL mode handles this. Reader sees pre-write state. Next frame renders post-write state. |
| Background cleanup runs while user saves | Different rows. No conflict. Both succeed. |
| User taps SALE button twice rapidly before first save | Debounce action buttons. After first tap, disable all action buttons for 500ms. Re-enable after save completes. |
| Bottom sheet Save button double-tapped | Disable Save button on first tap. Re-enable if save fails. |
| Restore runs while background computation in progress | Restore acquires exclusive database lock. Background jobs must respect this. |

### 9.7 Data Validation Rules

**Amount Validation:**

| Rule | Constraint |
|------|-----------|
| Minimum value | ₹0.01 (must be > 0; stored as 1 paisa) |
| Maximum value | ₹99,99,999.99 (stored as 9999999999 paisa) |
| Decimal places | Maximum 2 |
| Type | Numeric only |
| Required | Yes |

**Customer Name Validation:**

| Rule | Constraint |
|------|-----------|
| Minimum length | 1 character (after trimming whitespace) |
| Maximum length | 50 characters |
| Allowed characters | All Unicode (support Hindi, regional scripts) |
| Uniqueness | Not enforced (warning only) |
| Trimming | Leading and trailing whitespace auto-trimmed |

**Phone Number Validation:**

| Rule | Constraint |
|------|-----------|
| Length | Exactly 10 digits (if provided) |
| Starting digit | Must start with 6, 7, 8, or 9 |
| Format | No country code, no spaces, no dashes |
| Required | No |

**Note Validation:** Maximum 100 characters. All Unicode. Optional.

**Category Name Validation:** 1–20 characters. Unique (case-insensitive). Maximum 15 categories.

### 9.8 Database Integrity Validation Function

This function runs after every backup restore, on app startup, before backup export, and as part of the unit test suite:

```
function validateDatabaseIntegrity(db):
    errors = []

    // 1. No transaction has amount <= 0
    bad = SELECT COUNT(*) FROM transactions WHERE amount <= 0 AND is_deleted = 0
    if bad > 0: errors.add("Found transactions with non-positive amounts")

    // 2. All credit transactions have customer_id
    bad = SELECT COUNT(*) FROM transactions
          WHERE type IN ('credit_given', 'credit_received')
          AND customer_id IS NULL AND is_deleted = 0
    if bad > 0: errors.add("Found credit transactions without customer")

    // 3. All expense transactions have category
    bad = SELECT COUNT(*) FROM transactions
          WHERE type = 'expense' AND (category IS NULL OR length(trim(category)) = 0)
          AND is_deleted = 0
    if bad > 0: errors.add("Found expenses without category")

    // 4. All customer_ids reference existing customers
    bad = SELECT COUNT(*) FROM transactions t
          WHERE t.customer_id IS NOT NULL
          AND t.customer_id NOT IN (SELECT id FROM customers)
          AND t.is_deleted = 0
    if bad > 0: errors.add("Found orphan customer references")

    // 5. All transaction types are valid
    bad = SELECT COUNT(*) FROM transactions
          WHERE type NOT IN ('sale_cash','sale_upi','expense',
                             'credit_given','credit_received','opening_balance')
          AND is_deleted = 0
    if bad > 0: errors.add("Found invalid transaction types")

    // 6. At most one opening_balance transaction exists
    count = SELECT COUNT(*) FROM transactions WHERE type = 'opening_balance' AND is_deleted = 0
    if count > 1: errors.add("Multiple opening balance entries")

    // 7. Non-credit transactions must NOT have customer_id
    bad = SELECT COUNT(*) FROM transactions
          WHERE type NOT IN ('credit_given', 'credit_received')
          AND customer_id IS NOT NULL AND is_deleted = 0
    if bad > 0: errors.add("Found non-credit transaction with customer_id set")

    // 8. Cash in Hand cache matches computation (if cache exists)
    cached = getCache('cash_in_hand')
    computed = computeCashInHand(db)
    if cached != computed: errors.add("Cache mismatch: cached=" + cached + " computed=" + computed)

    return errors
```

---

## 10. Backup & Restore

### 10.1 Backup Export Flow

1. User taps **"Save Backup"** in More → Backup & Restore.
2. Run `PRAGMA wal_checkpoint(TRUNCATE)` to ensure all data is in the main DB file.
3. Serialize all data to JSON (all active transactions, all customers including inactive, all categories, all app_metadata).
4. Compute SHA-256 hash of the JSON string.
5. Compress JSON payload with gzip.
6. Optionally encrypt with AES-256-GCM if passphrase is set.
7. Write `.slbackup` file with header + payload.
8. File saved to device's Downloads folder.
9. File name: `SimpleLedger_Backup_YYYYMMDD_HHMMSS.slbackup`.
10. Success toast: "Backup saved to Downloads folder."
11. Update `app_metadata['last_backup_at']`.

**Backup Reminder:** If no backup has been taken in the last 7 days, show a non-blocking reminder banner on the Home screen: "It's been a while since your last backup. [Backup Now]". Dismissable. Reappears after another 7 days.

### 10.2 Backup File Format

```
┌──────────────────────────────────────────────┐
│  HEADER (fixed size, unencrypted)            │
│  ├── Magic bytes: "SLBK" (4 bytes)           │
│  ├── File format version: uint16 (2 bytes)   │
│  ├── Schema version: uint16 (2 bytes)        │
│  ├── App version: UTF-8 string (32 bytes)    │
│  ├── Created at: ISO 8601 UTC (30 bytes)     │
│  ├── Is encrypted: uint8 (1 byte)            │
│  ├── Encryption salt: (32 bytes, zeros if    │
│  │   not encrypted)                          │
│  ├── Data checksum: SHA-256 (32 bytes)       │
│  │   (hash of unencrypted JSON payload)      │
│  └── Header checksum: CRC-32 (4 bytes)       │
├──────────────────────────────────────────────┤
│  PAYLOAD (variable size)                     │
│  ├── JSON data (compressed with gzip)        │
│  │   ├── customers: [...]                    │
│  │   ├── transactions: [...]                 │
│  │   ├── expense_categories: [...]           │
│  │   ├── app_metadata: {...}                 │
│  │   └── record_counts: {                    │
│  │       "customers": 45,                    │
│  │       "transactions": 12000,              │
│  │       "categories": 8                     │
│  │     }                                     │
│  └── (encrypted with AES-256-GCM if         │
│       passphrase set)                        │
└──────────────────────────────────────────────┘
```

**Soft-deleted rows are excluded from backups.** Backups contain only active data (`is_deleted = 0`).

### 10.3 Encryption Key Management

- If user sets a passphrase: derive key using **PBKDF2-HMAC-SHA256** with 100,000 iterations and a random 32-byte salt stored in the header.
- If no passphrase: backup is **not encrypted**. JSON is only gzip-compressed.
- **No device-specific component** in key derivation. Backup is portable across devices.
- Passphrase is never stored on device. User must remember it. No recovery mechanism.

**[Superseded by v2.0 rule]** ~~v1.0 §4.10 specified "device-derived key + user-settable passphrase."~~ Device-derived keys prevent cross-device restore. Passphrase-only derivation is mandatory.

### 10.4 Backup Schema Version Handling

Every backup file includes `schema_version` in its header. On restore:

1. If `schema_version > CURRENT_SCHEMA_VERSION`: reject with error "Backup from newer version. Please update app."
2. If `schema_version == CURRENT_SCHEMA_VERSION`: restore directly.
3. If `schema_version < CURRENT_SCHEMA_VERSION`: restore, then run migrations from the backup's version to the current version.

### 10.5 Pre-Restore Protection (§23.C)

Before presenting the final restore confirmation, compare the backup's data recency against current data.

**Algorithm:**

```
function preRestoreCheck(currentDb, backupFile):
    backupCreatedAt = backupFile.header.created_at

    currentNewest = SELECT MAX(created_at) FROM transactions WHERE is_deleted = 0
    backupNewest  = MAX(created_at) across all transactions in backup data

    currentCount = SELECT COUNT(*) FROM transactions WHERE is_deleted = 0
    backupCount  = backupFile.payload.record_counts.transactions

    if currentCount == 0:
        return WARNING_NONE

    if backupNewest is NULL or backupNewest == "":
        return WARNING_BACKUP_EMPTY

    if backupNewest < currentNewest:
        return WARNING_BACKUP_OLDER

    if backupNewest >= currentNewest:
        return WARNING_NONE
```

**UI by Warning Level:**

- `WARNING_NONE`: Standard confirmation dialog showing backup date and record count. Button: "Replace Data".
- `WARNING_BACKUP_OLDER`: Enhanced warning showing current newest date, backup newest date, and count of newer records being erased. Button: "Restore Anyway".
- `WARNING_BACKUP_EMPTY`: Warning that backup contains no transactions, restoring will erase all current data. Button: "Restore Anyway".

All warnings are **non-blocking** — the user can still proceed. Restore is never prevented.

### 10.6 Full Ledger Rebuild (CT-2)

After every restore, the system MUST recompute ALL derived values before the user can interact. **New entries are blocked during rebuild.** A progress indicator is shown.

**Ordered rebuild steps:**

1. Replace database contents with backup data (SQLite `BEGIN EXCLUSIVE` atomic transaction).
2. Clear all volatile state (undo, timer, banner, cache, counter, clock anomaly state) — per §10.7.
3. Recompute: Cash in Hand (full SQL formula, including opening_balance).
4. Recompute: every customer outstanding (Σ `credit_given` − Σ `credit_received` per `customer_id`).
5. Recompute: days pending per customer (FIFO walk using `ORDER BY id ASC` per customer with outstanding > 0).
6. Rebuild: due list ordering (sort by recomputed outstanding descending).
7. Invalidate and rebuild: all cached aggregates (daily summaries by `business_date`, in-memory balance cache, denormalized cache table).
8. Run post-restore validation (`validateDatabaseIntegrity()`).
9. Navigate to Home. Show success toast: "Data restored successfully."

**Failure → Safe Mode:** If any rebuild step fails:

- App enters **safe mode** (export-only, no transactions allowed).
- Banner: "Data issue detected. Please export your data and contact support."
- Exit safe mode: re-restore from good backup or start fresh.

**Post-Restore Validation Checks:**

1. Referential integrity: count transactions with `customer_id` not in customers table. If count > 0: log error; display customer as "Unknown Customer". Do not fail restore.
2. Record count check: compare inserted rows with `record_counts` from backup file. If mismatch: log warning but do not fail.
3. Cash in Hand sanity check: compute and log. If magnitude > 10× sum of all transaction amounts: log warning.
4. Recompute all caches.

### 10.7 Restore + Undo Interaction (§23.G)

When a backup restore completes successfully, the following state resets MUST occur atomically before the user can interact:

| State | Action | Rationale |
|-------|--------|-----------|
| **Undo state** (volatile) | Set to NULL. Clear transaction ID. | Old ID no longer exists in new database. |
| **Undo timer** (volatile) | Cancel. Set to 0. | Timer for non-existent transaction. |
| **Undo banner** (UI) | Dismiss immediately. No animation. | User should not see Undo for data that no longer exists. |
| **Cash-in-Hand cache** (volatile) | Invalidate. Set to NULL (uninitialized). | Restored database has different transactions. |
| **Cash-in-Hand recomputation** | Trigger full recomputation. | Must happen before home screen renders. |
| **Transaction counter** (volatile) | Reset to 0. | Restart every-100-transactions verification. |
| **Clock anomaly state** (volatile) | Reset. | Previous anomaly state irrelevant to restored data. |
| **Navigation state** | Navigate to Home screen. | User lands on clean home screen with fresh data. |

### 10.8 Restore Atomicity

```sql
BEGIN EXCLUSIVE;
DELETE FROM transactions;
DELETE FROM customers;
DELETE FROM expense_categories;
DELETE FROM app_metadata;
-- insert all restored data --
COMMIT;
```

If any INSERT fails, the entire restore rolls back and original data is preserved. The user sees: "Restore failed. Your current data is unchanged."

### 10.9 Backup Validation on Import

**On backup restore:**

1. Read header. Verify magic bytes = "SLBK". If not → error: "Invalid file format."
2. Verify header CRC-32. If mismatch → error: "Backup file is damaged (header corrupted)."
3. Check file format version. If > supported → error: "Please update SimpleLedger."
4. If encrypted: prompt for passphrase. Derive key. Decrypt payload. If wrong passphrase: error: "Incorrect passphrase."
5. Decompress JSON.
6. Compute SHA-256 of decompressed JSON. Compare with header checksum. If mismatch → error: "Backup file is damaged and cannot be restored."
7. Parse JSON. Verify `record_counts` match actual array lengths. If mismatch → error: "Backup file is incomplete."
8. Run pre-restore protection check (§10.5).
9. Proceed with restore.

### 10.10 Phone Change Recovery

1. User installs SimpleLedger on new phone.
2. App opens with empty state: welcome screen with "Start Fresh" and "Restore Backup" options.
3. No login, no OTP, no account needed.
4. Backup file is the sole mechanism for data transfer between devices.
5. User is responsible for transferring the `.slbackup` file (via WhatsApp, Bluetooth, cable, Google Drive, etc.).

---

## 11. Error Handling

### 11.1 Data Entry Edge Cases

| Scenario | Expected Behavior |
|----------|------------------|
| User taps SALE/CASH OUT/CREDIT with ₹0 entered | Show toast: "Please enter amount". No bottom sheet opens. No transaction created. |
| User enters amount with more than 2 decimal places | Prevent input beyond 2 decimal places. Ignore additional digit taps. |
| User enters amount > ₹99,99,999.99 | Prevent input. Ignore digit tap that would exceed maximum. |
| User taps decimal point twice | Ignore second decimal point. |
| User taps backspace on empty display (₹0) | No action. Display remains ₹0. |
| User enters "0.00" and taps SALE | Show toast: "Please enter amount". Block transaction. |
| User enters amount starting with multiple zeros (e.g., "007") | Display as "7". Leading zeros stripped except before decimal (0.5 is valid). |

### 11.2 Credit (Udhar) Edge Cases

| Scenario | Expected Behavior |
|----------|------------------|
| Customer name is empty and user taps Save | Show inline error below name field: "Customer name is required". Block save. |
| Customer name is only spaces | Trim whitespace. Treat as empty. Block save. |
| Two customers with the same name | Allowed. Show warning "A customer with this name already exists. Save anyway?" Allow saving. |
| Phone number is not exactly 10 digits (if provided) | Show inline error: "Phone number must be 10 digits". Block save. |
| Very long customer name (>50 chars) | Prevent input beyond 50 characters. |

### 11.3 Payment Edge Cases

| Scenario | Expected Behavior |
|----------|------------------|
| Payment received exceeds outstanding balance | Allow. Show warning toast: "Payment exceeds pending amount by ₹{difference}." Outstanding goes negative (advance). See §6.3. |
| Payment of ₹0 | Block. Show toast: "Please enter amount". |
| Customer balance goes negative (overpayment) | Show "Advance: ₹X,XXX" on Customer Detail. Do NOT show in Due List. Advance consumed by future `credit_given`. |

### 11.4 App Lifecycle Edge Cases

| Scenario | Expected Behavior |
|----------|------------------|
| App killed mid-transaction (before save completes) | No data loss. Transaction was never fully committed. User re-enters on next launch. |
| App killed immediately after transaction save | Data safe. SQLite with WAL mode ensures write durability. |
| Phone battery dies during backup export | Partial file may exist. On next backup, a new file is created. Partial file is harmless. |
| Phone battery dies during restore | Data may be in inconsistent state. On next launch, detect incomplete restore and show error: "Last restore may be incomplete. Please restore again." |
| App opened after months of inactivity | All data intact. No expiration. Cash in Hand recomputes accurately. |

### 11.5 Undo Edge Cases

| Scenario | Expected Behavior |
|----------|------------------|
| User saves transaction A, then immediately saves transaction B | Undo banner for A is replaced by banner for B. Only B can be undone. A is permanent. |
| User taps Undo after banner has disappeared | Not possible. Undo button is gone. Transaction is final. |
| User navigates to another tab while undo banner is showing | Banner dismisses. Transaction is final. |
| User rotates phone while undo banner is showing | Banner remains if within 5 seconds. Timer continues. |
| App goes to background while undo banner is showing | On return, if 5 seconds elapsed, banner is gone. Otherwise, remaining time continues. |
| Rapid double-tap on UNDO button | 300ms debounce prevents second tap from processing. Only one deletion occurs. |

### 11.6 Backup & Restore Edge Cases

| Scenario | Expected Behavior |
|----------|------------------|
| User tries to restore a corrupted backup file | Show error: "This backup file is damaged and cannot be restored." No data changed. |
| User tries to restore a file that is not `.slbackup` | File picker filter prevents selection. If bypassed: show error: "Invalid file format." |
| User tries to restore a backup from a newer app version | Show error: "This backup was created with a newer version. Please update SimpleLedger." |
| User tries to restore backup from older app version | Allow. Perform schema migration to current version. |
| No `.slbackup` files found on device | Show message: "No backup files found." |
| Storage permission denied | Show system permission dialog. If denied: "Storage access is needed to save/restore backups." |
| Undo banner visible when restore starts | Undo banner dismissed as part of restore (§10.7). Old transaction no longer undoable. |

### 11.7 Database Integrity Edge Cases

| Scenario | Expected Behavior |
|----------|------------------|
| `PRAGMA integrity_check` returns errors on startup | Warning banner: "SimpleLedger has detected a data issue. Please back up your data immediately and reinstall." Show "Backup Now" button. Do not block app usage. |
| Transaction references `customer_id` that doesn't exist | Should never happen with foreign keys enabled. If detected: log error. Display customer as "Unknown Customer." Do not crash. |
| Sum computation returns NULL (empty database) | `COALESCE` wrappers handle this. Cash in Hand shows ₹0. |

### 11.8 Error Recovery — Defense Layers

```
Layer 1: Input Validation    → Prevent bad data from entering (UI + app logic)
Layer 2: Schema Constraints  → Database rejects invalid data (CHECK, NOT NULL, FK)
Layer 3: Atomic Writes       → Partial writes impossible (SQLite transactions)
Layer 4: WAL + Sync          → Crash recovery guaranteed (journal_mode = WAL)
Layer 5: Integrity Checking  → Detect corruption early (PRAGMA integrity_check)
Layer 6: Backup System       → User can recover from catastrophic failures
```

**Fail-Safe States:**

1. Never silently continue with incorrect data. Always show an error.
2. Never delete or modify data to "fix" corruption. Only the user (via backup/restore) can make destructive data changes.
3. Always allow backup export even in degraded state.
4. If the database is completely unreadable: show a clear screen with instructions.

---

## 12. Performance Requirements

### 12.1 Performance Targets

| Metric | @ 10k txns | @ 50k txns | @ 100k txns |
|--------|-----------|-----------|------------|
| App cold launch time | < 1.5s | < 2.0s | < 2.5s |
| App warm launch time | < 0.3s | < 0.3s | < 0.5s |
| Transaction save latency | < 0.3s | < 0.3s | < 0.5s |
| Home screen render (cached balance) | < 0.1s | < 0.1s | < 0.1s |
| Home screen render (cold, no cache) | < 0.3s | < 0.5s | < 0.8s |
| Due List render (100 customers) | < 0.5s | < 0.8s | < 1.0s |
| Due List render (500 customers) | < 1.0s | < 1.5s | < 2.0s |
| Daily Summary computation | < 0.3s | < 0.3s | < 0.5s |
| Customer autocomplete response | < 0.1s | < 0.15s | < 0.2s |
| Backup export | < 3s | < 10s | < 20s |
| Backup restore | < 5s | < 15s | < 30s |
| Soft-delete cleanup | < 0.5s | < 0.5s | < 0.5s |
| Integrity check (background) | < 2s | < 5s | < 10s |

**Benchmark device:** Entry-level Android phone with 2 GB RAM, eMMC storage (not UFS).

**If any metric exceeds target:** Review covering index strategy. Consider adding the denormalized `customer_balance_cache` table. **Never compromise on transaction save latency.**

### 12.2 Transaction Volume Projections

| Usage Pattern | Daily Txns | 1 Year | 3 Years | 5 Years |
|--------------|-----------|--------|---------|---------|
| Low (tea stall) | 10 | 3,650 | 10,950 | 18,250 |
| Medium (kirana) | 30 | 10,950 | 32,850 | 54,750 |
| High (busy shop) | 60 | 21,900 | 65,700 | 109,500 |

**Design target:** All operations performant at 100k transactions with 500 customers.

### 12.3 Authoritative Query Patterns

**Cash in Hand (critical path — uses covering index `idx_txn_type_amount` for index-only scan):**

```sql
SELECT
  COALESCE(SUM(CASE WHEN type = 'sale_cash' AND is_deleted = 0 THEN amount ELSE 0 END), 0)
  + COALESCE(SUM(CASE WHEN type = 'credit_received' AND is_deleted = 0 THEN amount ELSE 0 END), 0)
  + COALESCE(SUM(CASE WHEN type = 'opening_balance' AND is_deleted = 0 THEN amount ELSE 0 END), 0)
  - COALESCE(SUM(CASE WHEN type = 'expense' AND is_deleted = 0 THEN amount ELSE 0 END), 0)
  - COALESCE(SUM(CASE WHEN type = 'credit_given' AND is_deleted = 0 THEN amount ELSE 0 END), 0)
AS cash_in_hand_paisa
FROM transactions;
```

**Daily Summary (uses covering index `idx_txn_business_date`):**

```sql
SELECT
  type,
  SUM(amount) AS total_paisa,
  COUNT(*) AS count
FROM transactions
WHERE business_date = :selected_date
AND is_deleted = 0
AND type != 'opening_balance'
GROUP BY type;
```

**Due List:**

```sql
SELECT
  c.id, c.name, c.phone,
  (COALESCE(SUM(CASE WHEN t.type = 'credit_given' THEN t.amount ELSE 0 END), 0)
   - COALESCE(SUM(CASE WHEN t.type = 'credit_received' THEN t.amount ELSE 0 END), 0)) AS outstanding_paisa
FROM customers c
INNER JOIN transactions t ON t.customer_id = c.id AND t.is_deleted = 0
WHERE c.is_deleted = 0
AND t.type IN ('credit_given', 'credit_received')
GROUP BY c.id
HAVING outstanding_paisa > 0
ORDER BY outstanding_paisa DESC;
```

**Customer Autocomplete:**

```sql
SELECT id, uuid, name, phone, status
FROM customers
WHERE name LIKE '%' || :query || '%'
AND is_deleted = 0
ORDER BY name
LIMIT 5;
```

### 12.4 Optional Denormalized Cache

If Due List performance degrades beyond targets, implement a `customer_balance_cache` table:

| Field | Type |
|-------|------|
| `customer_id` | INTEGER (FK) |
| `outstanding_paisa` | INTEGER |
| `oldest_unpaid_business_date` | TEXT (YYYY-MM-DD) |
| `last_updated_txn_id` | INTEGER |

Refreshed on any `credit_given` or `credit_received` save or undo for that customer, and on app startup. **The cache is advisory.** If it disagrees with live computation, live computation wins and cache is corrected.

### 12.5 Database Size Projections

| Transactions | Estimated DB Size | Backup File Size (compressed) |
|-------------|-----------------|------------------------------|
| 1,000 | ~150 KB | ~50 KB |
| 10,000 | ~1.5 MB | ~400 KB |
| 50,000 | ~7 MB | ~2 MB |
| 100,000 | ~14 MB | ~4 MB |

---

## 13. Non-Functional Requirements

### 13.1 Reliability

| Requirement | Detail |
|-------------|--------|
| Data persistence | SQLite with WAL mode. All writes durable immediately. |
| Crash recovery | Database in consistent state after any app crash or force-kill. |
| Data integrity | Foreign key constraints enforced. Amounts always positive. |
| Battery optimization | App must handle aggressive OEM battery optimizer kills gracefully. |

### 13.2 Storage

| Estimate | Value |
|----------|-------|
| App size (APK) | < 15 MB |
| Database per 1,000 transactions | ~150 KB |
| Backup file per 1,000 transactions | ~50 KB (compressed) |
| Expected storage for 1 year of heavy use (~60 txns/day) | ~3 MB |

### 13.3 Compatibility

| Parameter | Requirement |
|-----------|------------|
| Minimum Android version | Android 6.0 (API 23) |
| Screen sizes | 4.5" to 7" (phones only) |
| Orientation | Portrait only (lock orientation) |
| Languages (v1) | English UI with Hindi terminology |
| Accessibility | Content descriptions on all interactive elements. TalkBack support for navigation. |

### 13.4 Security

| Requirement | Detail |
|-------------|--------|
| Authentication | None. No login. No PIN. No biometrics (v1). |
| Data encryption at rest | Optional `sqlcipher` (if performance allows). |
| Backup encryption | AES-256-GCM with optional passphrase (PBKDF2 key derivation). |
| Network security | No network calls in v1. No data leaves the device except via user-initiated backup export. |

### 13.5 Behavioral Guarantees

| Guarantee | Specification |
|-----------|-------------|
| Max 3 taps per entry | Enter amount → tap SALE → tap Cash/UPI. Done. |
| Instant save | Transaction save latency < 0.5s at any scale. Undo banner must appear before next customer is served. |
| No confirmation screen | Exception only: restore backup (destructive operation). |
| One-hand operation | All primary actions reachable by right thumb in portrait mode. |
| No blocking dialogs | Error messages are toasts (non-blocking, auto-dismiss). |
| No loading spinners | If computation takes time, show stale data and update. Only exception: restore rebuild progress indicator. |

### 13.6 Testing Requirements

**Unit Test Requirements — Financial Calculations:**

| Test ID | Scenario | Input Transactions | Expected Cash (paisa) |
|---------|----------|-------------------|----------------------|
| T-CIH-01 | Empty database | None | 0 |
| T-CIH-02 | Single cash sale | sale_cash: 25000 | 25000 |
| T-CIH-03 | UPI sale excluded | sale_upi: 50000 | 0 |
| T-CIH-04 | Cash sale + expense | sale_cash: 100000, expense: 30000 | 70000 |
| T-CIH-05 | Credit given reduces cash | sale_cash: 100000, credit_given: 40000 | 60000 |
| T-CIH-07 | All types combined | sale_cash: 1000000, sale_upi: 500000, expense: 200000, credit_given: 300000, credit_received: 100000 | 600000 |
| T-CIH-08 | Negative cash in hand | expense: 500000 | -500000 |
| T-CIH-09 | Opening balance | opening_balance: 1000000, sale_cash: 500000 | 1500000 |
| T-CIH-10 | Soft-deleted excluded | sale_cash: 100000 (active), sale_cash: 50000 (deleted) | 100000 |
| T-CIH-12 | Decimal precision | sale_cash: 1050 × 3 | 3150 |

**Customer Outstanding Tests, Days Pending Tests, Business Day Tests, Advance Payment Tests, Clock Change Tests, Cash-in-Hand Trust Tests, and Restore+Undo Tests:** All specified in §23.A–G of the v1.2 behavioral specification (included as binding requirements in this document).

**Stress Testing Scenarios:**

| Test ID | Scenario | Pass Criteria |
|---------|----------|--------------|
| T-ST-01 | Insert 100,000 transactions programmatically | All inserted. DB < 20 MB. |
| T-ST-02 | Compute Cash in Hand at 100k transactions | < 800ms on benchmark device. |
| T-ST-03 | Render Due List with 500 customers at 100k transactions | < 2 seconds. |
| T-ST-07 | 50 rapid sequential saves | All 50 saved correctly. Cash in Hand correct. No crashes. |
| T-ST-08 | Save → kill app → restart, repeat 100 times | All transactions persisted or not started. None partial. |

### 13.7 Out of Scope (v1)

| Feature | Reason for Exclusion |
|---------|---------------------|
| Inventory management | Adds complexity. Not needed for financial memory replacement. |
| Barcode scanning | Requires camera integration and product database. |
| GST billing | Regulatory complexity. This is not a billing app. |
| Online payments (PG integration) | This app records transactions; it doesn't process payments. |
| Cloud sync | Requires accounts, internet dependency. Violates offline-first. |
| Multi-user accounts | Single shopkeeper, single device in v1. |
| Analytics graphs / charts | Simple number summaries are sufficient for v1. |
| Advertisements | Trust-destroying. Never. |
| SMS/WhatsApp reminders to customers | Requires permissions and internet. Consider for v2. |
| Multi-language support | English + Hindi terms in v1. Full localization in v2. |
| PIN / biometric lock | Adds friction to the 3-second transaction goal. Consider as optional in v2. |
| Transaction editing (after undo window) | Intentional. Immutable transaction history prevents accidental corruption. |
| Profit/loss calculation | Requires cost-of-goods data. Out of scope. |
| Bank account tracking | Only cash drawer tracking in v1. |

### 13.8 Future Extensibility (Schema Forward-Compatibility)

The v1.2 schema is designed so the following future features can be added without destructive migrations:

- **Cloud sync:** `uuid` columns on all entities + UTC timestamps + soft-delete tombstones + event-sourced log.
- **User accounts:** `app_metadata` table can store `user_id`, `auth_token` without schema change.
- **Multi-language:** All user-facing strings are externalizable. No language-specific data in database.
- **New transaction types:** `type` is TEXT; new types extend the CHECK constraint. Cash in Hand formula uses explicit CASE/WHEN — new types are excluded by default (safe opt-in).

**Schema Extension Rules for Future Developers:**

1. NEVER remove a column. Only add columns.
2. NEVER change a column type.
3. NEVER change the meaning of an existing transaction type.
4. ALWAYS add new columns as NULLABLE or with DEFAULT values.
5. ALWAYS increment `PRAGMA user_version` for any schema change.
6. ALWAYS update the backup format version if the schema changes.
7. ALWAYS write a migration function from version N to N+1.
8. ALWAYS test restore of old backup files on new schema versions.

### 13.9 Performance Monitoring

The app logs internally (not to any external server) key performance metrics to `app_metadata`:

| Metric | When Logged |
|--------|-------------|
| Cold launch time | Every cold start |
| Cash in Hand computation time | Every cold computation (not from cache) |
| Transaction save time | Every save |
| Due List render time | Every Due List open |

If any metric exceeds 2× the target threshold, a diagnostic entry is written to `app_metadata` with key `perf_warning_{metric}_{date}`. This data is included in backup files for debugging.

---

## 14. External Design Reference (design.md)

> **Policy:** All visual and aesthetic implementation is governed by the external document `design.md`. This PRD specifies behavioral and functional requirements only.
>
> If any conflict occurs between visual guidelines in `design.md` and behavioral rules in this PRD, **the behavioral rules in this PRD take precedence.**
>
> **Developers must not block functional implementation due to missing or incomplete visual specifications.** Functional behavior must be implemented regardless of `design.md` availability.

The following elements are **entirely defined in `design.md`** and must not be specified in this PRD:

- Color system (hex codes for all states including positive, warning, negative, accent colors)
- Typography scale (font sizes, weights, line heights)
- Touch target minimum dimensions
- Spacing rules and padding values
- Animation timings, easing curves, transition durations
- Haptic feedback patterns and intensity levels
- WCAG contrast compliance targets
- Screen layout details (zone positioning, element sizing)

The following elements are **behavioral requirements retained in this PRD** (mandatory regardless of design.md):

| Interaction | Behavioral Requirement |
|-------------|----------------------|
| Transaction saved | Undo banner MUST appear. Auto-dismiss after 5 seconds. |
| Error toast | MUST appear and auto-dismiss. User MUST NOT be blocked. |
| Cash in Hand | MUST always be visible on Home screen. Never scrolled off-screen. |
| No confirmation dialogs | Exception: restore backup (destructive operation) only. |
| Maximum navigation depth | 2 levels (e.g., More → Backup). |
| Customer name input | MUST be focused automatically when Credit bottom sheet opens. |

---

## Appendix A: Glossary

| Term | Definition |
|------|-----------|
| **Sale** | Money received for goods sold. Recorded as `sale_cash` or `sale_upi`. |
| **Kharcha / Cash Out** | Money paid out for business expenses. |
| **Udhar / Credit** | Goods given to a customer on trust. Customer will pay later. |
| **Payment Received** | Cash received from a customer against their pending udhar. |
| **Cash in Hand** | Computed value representing physical cash in the shop's cash drawer. Never stored. |
| **Due List** | List of all customers with pending (unpaid) udhar amounts. |
| **Today's Summary** | End-of-day snapshot of all financial activity for the current business day. |
| **Pending Amount** | Total udhar a customer still owes, computed from transaction history. |
| **Days Pending** | Number of business days since the oldest unpaid credit for a customer. |
| **Business Day** | Period from `closing_hour` on one calendar day to `closing_hour` on the next. |
| **Opening Balance** | One-time transaction type representing cash on hand at app installation. |
| **Advance** | Negative outstanding for a customer — they have overpaid. Consumed by future credits. |
| **Soft Delete** | Marking a transaction as `is_deleted = 1` rather than removing the row. |
| **Backup** | A `.slbackup` file containing all app data, exportable to device storage. |
| **Restore** | Replacing current app data with data from a backup file. |
| **Insertion ID** | The `id` column (INTEGER PRIMARY KEY AUTOINCREMENT). Authoritative transaction ordering. |
| **Paisa** | 1/100th of a rupee. The internal unit of monetary storage. |
| **FIFO Walk** | Algorithm that consumes credit_given transactions from oldest to newest to determine Days Pending. |

---

## Appendix B: Acceptance Criteria

### AC-01: Cash Sale Recording

```
GIVEN   the user is on the Home screen
AND     the user has entered a valid amount (e.g., ₹250)
WHEN    the user taps SALE and then taps Cash
THEN    a transaction of type sale_cash with amount 25000 paisa is saved to the database
AND     the amount display resets to ₹0
AND     Cash in Hand cache increments by ₹250
AND     an undo banner appears: "✓ ₹250 Cash Sale saved [UNDO]"
AND     the undo banner auto-dismisses after 5 seconds
AND     the total time from tapping Cash to banner appearing is < 0.5 seconds
```

### AC-02: UPI Sale Recording

```
GIVEN   the user is on the Home screen
AND     the user has entered a valid amount (e.g., ₹500)
WHEN    the user taps SALE and then taps UPI
THEN    a transaction of type sale_upi with amount 50000 paisa is saved
AND     Cash in Hand does NOT change
AND     the undo banner appears: "✓ ₹500 UPI Sale saved [UNDO]"
```

### AC-03: Expense Recording

```
GIVEN   the user is on the Home screen and has entered a valid amount
WHEN    the user taps CASH OUT
THEN    a bottom sheet appears with category chips and a note field
WHEN    the user selects a category
THEN    if the note field is empty, the transaction is saved immediately
AND     Cash in Hand decreases by the amount
AND     the undo banner appears
```

### AC-04: Credit Given Recording

```
GIVEN   the user is on the Home screen and has entered a valid amount
WHEN    the user taps CREDIT
THEN    a bottom sheet appears with customer name, phone, and note fields
WHEN    the user enters a customer name and taps Save
THEN    a transaction of type credit_given is saved
AND     the customer record is created if new (with a new integer id and uuid)
AND     all transactions bind to customer_id (integer), not name string
AND     Cash in Hand decreases by the amount
AND     the customer appears in the Due List
AND     the undo banner appears
```

### AC-06: Payment Received (Full Clearance)

```
GIVEN   a customer "Rajesh Kumar" has an outstanding balance of ₹2,500
WHEN    the user navigates to Due List → taps Rajesh → taps Receive Payment
AND     enters ₹2,500 and taps Save
THEN    a transaction of type credit_received with amount 250000 paisa is saved
AND     the original credit_given transaction is NOT modified in any way
AND     Rajesh Kumar's outstanding becomes ₹0 (computed, not stored)
AND     Rajesh Kumar no longer appears in the Due List
AND     Cash in Hand increases by ₹2,500
AND     the undo banner appears
```

### AC-08: Undo Transaction

```
GIVEN   a transaction has just been saved
AND     the undo banner is visible
WHEN    the user taps UNDO within 5 seconds
THEN    the transaction is soft-deleted (is_deleted = 1)
AND     Cash in Hand cache reversal applied correctly
AND     a toast "Entry removed" appears
AND     the undo banner dismisses
```

### AC-11: Due List Display

```
GIVEN   there are 3 customers with outstanding balances:
        Customer A: ₹5,000 (15 days)
        Customer B: ₹8,000 (45 days — business days)
        Customer C: ₹2,000 (10 days)
WHEN    the user opens the Due List
THEN    customers are shown in order: B (₹8,000), A (₹5,000), C (₹2,000)
AND     Customer B's amount and days are displayed in RED (> 30 days)
AND     the total pending shown is ₹15,000
```

### AC-12: Daily Summary Computation

```
GIVEN   today's business-day transactions are:
        sale_cash: ₹1,000, ₹500
        sale_upi: ₹800
        expense: ₹200, ₹100
        credit_given: ₹600
        credit_received: ₹400
WHEN    the user opens the Report tab
THEN    the summary shows:
        Total Sales: ₹2,300 (Cash: ₹1,500, UPI: ₹800)
        Total Kharcha: ₹300
        Udhar Given: ₹600
        Udhar Received: ₹400
AND     opening_balance (if any) does NOT appear in this summary
AND     Cash in Hand shows all-time computed value (unaffected by date filter)
```

### AC-13: Cash in Hand Accuracy

```
GIVEN   the following all-time active transactions:
        sale_cash: ₹10,000 (stored as 1000000 paisa)
        sale_upi: ₹5,000
        expense: ₹2,000
        credit_given: ₹3,000
        credit_received: ₹1,000
WHEN    the Home screen renders
THEN    Cash in Hand = 1000000 − 200000 − 300000 + 100000 = 600000 paisa = ₹6,000
AND     sale_upi (₹5,000) does NOT affect Cash in Hand
```

### AC-17: Offline Operation

```
GIVEN   the device has no internet connectivity (airplane mode)
WHEN    the user performs any action (sale, expense, credit, payment, backup)
THEN    the action completes successfully
AND     no error related to network connectivity is shown
AND     all features function identically to online mode
```

### AC-21: Overpayment Handling

```
GIVEN   a customer has ₹1,000 outstanding
WHEN    the user records a payment of ₹1,500
THEN    the transaction is saved with a warning toast: "Payment exceeds pending amount by ₹500"
AND     the customer's outstanding becomes −₹500 (advance)
AND     the customer does NOT appear in the Due List
AND     Cash in Hand increases by ₹1,500
WHEN    the shopkeeper later gives ₹200 credit to this customer
THEN    the outstanding becomes −₹300 (advance reduced by ₹200)
AND     no special transaction type or splitting occurs — arithmetic handles it
```

### AC-22: Transaction Immutability

```
GIVEN   a credit_given transaction exists for ₹2,000 for "Rajesh"
WHEN    a payment of ₹500 is received from "Rajesh"
THEN    the original ₹2,000 credit_given transaction is NOT modified in any way
AND     a NEW credit_received transaction of ₹500 is created
AND     outstanding is computed as ₹2,000 − ₹500 = ₹1,500
```

### AC-23: Business Day Accuracy

```
GIVEN   the business closing hour is set to 04:00
AND     a transaction is recorded at 02:30 AM on Feb 24
WHEN    the user opens the Report tab for Feb 23
THEN    the 02:30 AM transaction appears under Feb 23's summary (business date = Feb 23)
AND     it does NOT appear under Feb 24's summary
```

### AC-24: Customer Identity After Rename

```
GIVEN   a customer "Rajsh" exists with ₹1,500 outstanding
WHEN    the user renames the customer to "Rajesh"
THEN    the customer name is updated to "Rajesh" everywhere in the app
AND     all transaction history for this customer is preserved
AND     the outstanding balance is unchanged (₹1,500)
AND     the customer's id (integer) is unchanged
AND     no other customer's ledger is affected
```

---

## Appendix C: Recommended Flutter File Structure

```
lib/
├── main.dart
├── app.dart
├── models/
│   ├── transaction.dart
│   ├── customer.dart
│   └── expense_category.dart
├── database/
│   ├── database_helper.dart
│   ├── migrations/
│   │   ├── migration_v1.dart
│   │   └── migration_v2.dart
│   └── queries/
│       ├── transaction_queries.dart
│       ├── customer_queries.dart
│       └── cash_in_hand_queries.dart
├── services/
│   ├── transaction_service.dart
│   ├── customer_service.dart
│   ├── balance_service.dart
│   ├── backup_service.dart
│   └── integrity_service.dart
├── screens/
│   ├── home/
│   │   ├── home_screen.dart
│   │   ├── keypad_widget.dart
│   │   ├── amount_display.dart
│   │   ├── cash_in_hand_strip.dart
│   │   └── action_buttons.dart
│   ├── sale/
│   │   └── sale_type_sheet.dart
│   ├── expense/
│   │   └── expense_sheet.dart
│   ├── credit/
│   │   └── credit_sheet.dart
│   ├── due_list/
│   │   ├── due_list_screen.dart
│   │   └── customer_detail_screen.dart
│   ├── report/
│   │   └── daily_summary_screen.dart
│   ├── more/
│   │   ├── more_screen.dart
│   │   ├── backup_restore_screen.dart
│   │   └── edit_categories_screen.dart
│   └── welcome/
│       └── welcome_screen.dart
├── widgets/
│   ├── undo_banner.dart
│   └── bottom_nav.dart
└── utils/
    ├── formatters.dart
    ├── business_day.dart
    └── constants.dart
```

---

_End of Document_

**Document version:** 2.0 — Final Implementation Specification  
**Last updated:** February 23, 2026  
**Status:** Authoritative Engineering Contract. Ready for deterministic implementation.  
**Supersedes:** v1.0 (Feb 22, 2026), v1.1 (Feb 23, 2026), v1.2 (Feb 23, 2026)  
**Conflict resolution:** v1.2 rules override v1.1 rules override v1.0 rules.
