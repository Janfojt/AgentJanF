# Golden Master — ORD101 (Order-Line Editing with VAT)

## Overview

This directory contains `cases.json`: the golden master for the ORD101 slice.
Each case specifies concrete input records (in the layout of the physical files), user actions,
and the exact outputs the program must produce. A rebuild in any language must reproduce
these outputs byte for byte.

No IBM i is available in this environment. All expected outputs are derived from reading
the RPG source code and applying its arithmetic rules exactly.

---

## How the cases are organised

Cases fall into two groups:

### Group 1 — VAT300 unit test cases (GM-VAT-01 to GM-VAT-06)

These exercise `GetVATRate`, `GetVATDesc`, `ClcVAT`, and `ExistVATRate` in isolation.
They use the **test fixture** defined in `QTESTSRC/vat300.test.sqlrpgle` (`setUpSuite`),
which loads only codes `'1'` (rate 20.00, active) and `'3'` (rate 10.00, deleted).
Code `'2'` is intentionally absent from that fixture so code `'2'` behaves as unknown
in these cases. The seed data in `data/VATDEF.csv` is separate and is used in Group 2.

| Case | Procedure | Rule | What it tests |
|------|-----------|------|----------------|
| GM-VAT-01 | GetVATRate('1') | R17, R19 | Active code returns its rate |
| GM-VAT-02 | GetVATRate('3') | R17, R19, Q3 | Deleted code still returns its rate |
| GM-VAT-03 | GetVATRate('2') | R17, R18 | Unknown code returns 0.00 (clear before CHAIN) |
| GM-VAT-04 | ClcVAT('1', 200.00) | R17, R18, R20 | Standard calculation: 40.00 |
| GM-VAT-05 | ClcVAT('2', 200.00) | R17, R18, R20, Q2 | Unknown code: silent zero |
| GM-VAT-06 | ClcVAT('3', 200.00) | R17, R18, R20, Q3 | Deleted code: rate still applied |

### Group 2 — ORD101 program cases (GM-01 to GM-15)

These exercise ORD101 as a whole, using the seed VATDEF (codes 1–4 active/deleted as seeded).
Each case is labelled with the rule(s) from `docs/reverse/ORD101/rules.md` it exercises.

| Case | Rules | Description |
|------|-------|-------------|
| GM-01 | R1 | Startup: order header load, customer name from CUS300 |
| GM-02 | R2, R3 | Subfile load for order 60716: four rows, running totals |
| GM-03 | R4 | Invalid option (e.g. 5): reverse image + "Invalid Option" message |
| GM-04 | R5 | Option 4 blocked: ODQTYLIV > 0 on a line, delivery-block message |
| GM-05 | R6, Q7 | Option 6 accepted by validator but s01act has no branch — no action |
| GM-06 | R7 | Option 4 deletes a line (ODQTYLIV = 0): record removed, totals decremented, row blanked |
| GM-07 | R8, R9 | FMT02 load: fields from DB + VAT recalculated on stored ODTOT for display |
| GM-08 | R12, R13, R15, R16 | Normal save: ODTOT = DSQTY × DSPRICE, VAT recalculated, five fields written |
| GM-09 | R10 | ERR1001: DSQTYLIV entered > ODQTY stored — no save, indicator stays on (Q5) |
| GM-10 | R11 | ERR1002: DSQTY entered < ODQTYLIV stored — no save, note Q6 quirk |
| GM-11 | R12, R13, Q1 | Half-rounding: 0.3850 (Packed 11:4) → %dech → 0.39, not 0.38 |
| GM-12 | R20, Q2 | Unknown VAT code (ARVATCD='9'): ClcVAT = 0.00, no error shown |
| GM-13 | R20, Q3 | Deleted VAT code (ARVATCD='3'): rate 10.00% still applied |
| GM-14 | R12, R13 | Ordered quantity = 0: ODTOT = 0.00, VAT = 0.00 |
| GM-15 | R10, R11 | Delivered = ordered: boundary passes both validation checks |

---

## Rules cross-reference

| Rule | Cases |
|------|-------|
| R1 | GM-01 |
| R2 | GM-02 |
| R3 | GM-02 |
| R4 | GM-03 |
| R5 | GM-04 |
| R6 | GM-05 |
| R7 | GM-06 |
| R8 | GM-07 |
| R9 | GM-07 |
| R10 | GM-09, GM-15 |
| R11 | GM-10, GM-15 |
| R12 | GM-08, GM-11, GM-14 |
| R13 | GM-08, GM-11, GM-13, GM-14 |
| R15 | GM-08 |
| R16 | GM-08 |
| R17 | GM-VAT-01, GM-VAT-02, GM-VAT-03, GM-VAT-04, GM-VAT-05, GM-VAT-06 |
| R18 | GM-VAT-03, GM-VAT-04, GM-VAT-05, GM-VAT-06 |
| R19 | GM-VAT-01, GM-VAT-02, GM-VAT-03 |
| R20 | GM-VAT-04, GM-VAT-05, GM-VAT-06, GM-12, GM-13 |
| Q1 | GM-11 |
| Q2 | GM-VAT-05, GM-12 |
| Q3 | GM-VAT-02, GM-VAT-06, GM-13 |
| Q5 | GM-09 |
| Q6 | GM-10 |
| Q7 | GM-05 |

---

## Cases from the RPGUnit test

The following cases in Group 1 are direct translations of assertions in
`QTESTSRC/vat300.test.sqlrpgle`:

| Case | Source line | Assertion |
|------|------------|-----------|
| GM-VAT-01 | line 39 | `GetVATRate(gVatCodeValid) = 20.00` |
| GM-VAT-02 | line 41 | `GetVATRate(gVatCodeDeleted) = 10.00` |
| GM-VAT-03 | line 40 | `GetVATRate(gVatCodeInvalid) = 0.00` |
| GM-VAT-04 | line 57 | `ClcVAT(gVatCodeValid : 200.00) = 40.00` |
| GM-VAT-05 | line 58 | `ClcVAT(gVatCodeInvalid : 200.00) = 0.00` |
| GM-VAT-06 | line 59 | `ClcVAT(gVatCodeDeleted : 200.00) = 20.00` |

In the test, `gVatCodeValid='1'`, `gVatCodeInvalid='2'`, `gVatCodeDeleted='3'`.

---

## How outputs are derived

No IBM i is available here. Every expected output is derived from the source code:
- Arithmetic follows RPG packed-decimal rules: `EVAL` (no suffix) truncates; `EVAL(H)` and `%dech` half-adjust (round half-up).
- All derivations are in each case's `derivation` field, showing the intermediate arithmetic.
- On a real project, outputs would come from recorded runs of the original program. The `derivation` field would then serve as documentation of *why* the output is what it is.

## Custom records

Case GM-11 (half-rounding) uses a custom ORDER row (ORID=60720) and DETORD row
(60720/line 1, price=5.00) not present in the seed data. This was necessary because no seed
line produces a 4-decimal intermediate value ending in exactly .XX50 on a real VAT code.
All other cases use seed data from `data/`.
