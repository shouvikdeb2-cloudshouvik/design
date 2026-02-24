# SimpleLedger — Product Context Document

## 1. What this application is

SimpleLedger is a mobile offline-first shopkeeper credit ledger application.

It is designed for very small businesses (kirana shops, tea stalls, repair shops, vegetable sellers, small restaurants, and service providers) who informally track money owed by customers.

This app replaces a physical notebook (“Udhar Khata”).

The core job of the app is:
Record money quickly and remember who owes how much.

This is NOT a banking app.
This is NOT a full accounting software.
This is NOT a tax or GST bookkeeping system.

It is a **daily memory replacement tool for shopkeepers**.

The shopkeeper is not doing accounting.
The shopkeeper is trying not to forget money.

---

## 2. What problem it solves

In many small shops, transactions happen rapidly:

Customer says:
“I will pay later.”

The shopkeeper must instantly remember:

* who took items
* how much amount
* whether partial payment happened

Currently they use:

* paper notebook
* memory
* WhatsApp messages
* rough slips

These methods fail because:

* entries are forgotten
* amounts are miscalculated
* totals are wrong
* disputes occur
* paper gets lost

The main risk is not calculation error.

The main risk is **loss of trust**.

If the shopkeeper cannot prove dues, money is permanently lost.

SimpleLedger acts as a permanent memory that:

* never forgets
* never miscalculates
* is faster than writing

---

## 3. Who the users are

Primary user:
Small shopkeeper working at a counter.

Typical conditions:

* standing while using phone
* customers waiting
* one-handed usage
* sunlight glare
* limited technical knowledge
* records entries in 2–3 seconds

Age range:
Approximately 25–60 years old.

The user does not think in terms of “accounts” or “ledgers”.

The user thinks:
“Write it quickly before I forget.”

---

## 4. What type of application this is

Correct classification:

Offline First Financial Record Keeper
Informal Credit Ledger
Shopkeeper Udhar Management App
Micro-business Memory System

It is closest to:
a digital khata / udhar book.

It is NOT:

* ERP
* accounting package
* bookkeeping suite
* invoicing platform
* POS billing machine

---

## 5. Core behavior philosophy

The app must prioritize:

Speed > Features
Trust > Analytics
Clarity > Customization
Recovery > Automation

The user will stop using the app if:

* it takes too long to enter data
* totals ever become wrong
* data is lost
* mistakes cannot be corrected instantly

Therefore the most important features are:

1. Fast entry (under 3 seconds)
2. Always correct totals
3. Easy undo
4. Never lose data

Reports and charts are secondary.

---

## 6. How the app is used daily

Typical daily flow:

Morning:
Shop opens, app opened once.

During day:
Customers buy items on credit.
Shopkeeper quickly enters:
customer name + amount.

Sometimes customer pays partially → recorded immediately.

Night:
Shopkeeper checks:
“Kitna aaj cash bacha?” (cash in hand)

The app is opened many times per day for a few seconds each time.

This is a habit-forming utility, not a long-session application.

---

## 7. Why offline operation is required

The app must work without internet because:

* many shops have unstable data connection
* the shopkeeper cannot wait for loading
* transaction must never fail
* financial record must always be available

Internet-based validation is optional, not required.

The ledger must function fully locally.

---

## 8. Trust requirement

This app is a trust device.

If even once:

* a total changes unexpectedly
* a record disappears
* a payment is misapplied

the user will permanently abandon the app.

Therefore the system must:

* never modify past transactions
* never silently recalculate differently
* always allow reconstruction of history

Correctness is more important than performance.

---

## 9. Key mental model

The shopkeeper is outsourcing memory to the application.

SimpleLedger is not a calculator.

SimpleLedger is:
“the place where the shopkeeper’s memory lives.”

Every design and technical decision must preserve this role.

End of document.
