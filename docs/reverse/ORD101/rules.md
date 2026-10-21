# Business Rules — ORD101 (Order-Line Editing with VAT)

## Purpose

ORD101 lets a user view and edit the lines of a single customer order.
It is called with the order number (ORID) as a parameter.
It reads ORDER1 (to get the customer) and DETORD1 (to list and update order lines),
and calls the VAT service program (vat300.rpgle) to compute tax amounts.
It calls ART300 (GetArtDesc, GetArtVatCode) for article data and CUS300 (GetCusName)
for the customer name.
It can delete a line and update quantity, unit price, and delivered quantity on a line.
The stock trigger ORD700 fires on DETORD after each update or delete (invisible to ORD101).

**Files read:** ORDER1, DETORD1, VATDEF (via vat300), ARTICLE1 (via ART300), CUSTOME1 (via CUS300).
**Files updated:** DETORD1 (update of line fields, delete of a line).
**Service program called:** VAT300 (ClcVAT, GetVATRate, GetArtVatCode via ART300).
**Screen:** ORD101D (subfile SFL01/CTL01 for the line list; FMT02 for the detail edit screen).

---

## Data Dictionary

| Field | File | Length / Type | Meaning |
|---|---|---|---|
| ORID | ORDER, DETORD | 6P 0 | Order number (key) |
| ORYEAR | ORDER | 4P 0 | Year part of the order key |
| ORCUID | ORDER | 5P 0 | Customer id on the order |
| ODORID | DETORD | 6P 0 | Order number on the line (foreign key to ORDER) |
| ODLINE | DETORD | 5P 0 | Line number within the order |
| ODARID | DETORD | 6 char | Article id on the line |
| ODQTY | DETORD | 5S 0 | Ordered quantity |
| ODQTYLIV | DETORD | 5S 0 | Delivered quantity (French *livre*) |
| ODPRICE | DETORD | 7P 2 | Unit price |
| ODTOT | DETORD | 9P 2 | Line total without VAT (ODQTY × ODPRICE) |
| ODTOTVAT | DETORD | 9P 2 | Line total with VAT (ODTOT + VAT amount) |
| VATCODE | VATDEF | 1 char | VAT rate code (key) |
| VATRATE | VATDEF | 4S 2 | VAT percentage (e.g. 20.00 for 20%) |
| VATDESC | VATDEF | 20 char | Description of the VAT rate |
| VATDEL | VATDEF | 1 char | Delete flag; 'X' means soft-deleted |
| DSQTY | Screen | same as ODQTY | Editable copy of ordered quantity on FMT02 |
| DSQTYLIV | Screen | same as ODQTYLIV | Editable copy of delivered quantity on FMT02 |
| DSPRICE | Screen | same as ODPRICE | Editable copy of unit price on FMT02 |
| TOT | Screen | same as ODTOT | Running sum of ODTOT for all lines, shown at screen bottom |
| TOTVAT | Screen | same as ODTOTVAT | Running sum of ODTOTVAT for all lines, shown at screen bottom |

---

## Business Rules

### Startup

**R1 — Load order header.**
On entry the program chains ORDER1 by the passed order id to retrieve the customer id (ORCUID),
then calls GetCusName(ORCUID) to get the customer name for display on both screens.
Source: `ORD101.PGM.RPGLE` lines 278–282 (`*inzsr`).

---

### Subfile — Line List (Panel 1)

**R2 — Load all lines into the subfile.**
The program positions DETORD1 with SETLL on the order id, then reads forward with READE until
end-of-file. For each record it calls GetArtDesc(ODARID) and writes a subfile row showing ODLINE,
ODARID, article description, ODQTY, ODPRICE, and ODTOT.
ODQTYLIV and ODTOTVAT are in the subfile record but hidden (field type H in the display file).
Source: `ORD101.PGM.RPGLE` lines 102–117; `ORD101D.DSPF` lines 23–24.

**R3 — Accumulate order totals.**
While loading lines, the program adds each line's ODTOT to TOT and each line's ODTOTVAT to TOTVAT.
These running totals are displayed at the bottom of the subfile screen (row 22/23, column 66).
They are reset to 0 at the start of each screen refresh (step prp).
Source: `ORD101.PGM.RPGLE` lines 98–99, 108–109; `ORD101D.DSPF` lines 84–87.

**R4 — Validate the option column.**
After the user presses Enter the program reads all changed subfile rows (READC).
Any option other than blank (0), 2, 4, or 6 is rejected: indicator 34 (reverse image) is
set on the row, and the message "Invalid Option" appears in the subfile control area.
Source: `ORD101.PGM.RPGLE` lines 150–173; `ORD101D.DSPF` lines 36–38.

**R5 — Block delete when line has a delivery.**
Option 4 is rejected if ODQTYLIV > 0 (the line already has at least one unit delivered).
The row is shown in reverse image and the message "Line with delivery can not be deleted."
appears in the subfile control area.
Source: `ORD101.PGM.RPGLE` lines 161–169; `ORD101D.DSPF` lines 37–38.

**R6 — Option 6 (Deliver) is accepted but does nothing.**
The screen legend lists "6=Deliver" and the option validator accepts 6 as a valid value,
so no "Invalid Option" message appears. However, the action step (s01act) has no branch for
option 6 — it falls through the select without performing any operation.
Source: `ORD101D.DSPF` line 76; `ORD101.PGM.RPGLE` lines 152, 176–196.

**R7 — Delete a line (option 4).**
When option 4 passes validation, the program subtracts the line's ODTOT from TOT and its
ODTOTVAT from TOTVAT (updating the screen totals), deletes the DETORD1 record by key
(id, ODLINE), then blanks the subfile row visually (ODARID blank, ODQTY/ODPRICE/ODTOT = 0,
ARTDESC = '\*\*\*\* Delete \*\*\*').
Source: `ORD101.PGM.RPGLE` lines 185–195.

---

### Detail Edit Screen (Panel 2 / FMT02)

**R8 — Load the detail screen from the database.**
The program chains DETORD1 by the full key (ORID, ODLINE) and copies the stored values to
the screen variables: DSQTY = ODQTY, DSQTYLIV = ODQTYLIV, DSPRICE = ODPRICE.
The article description is re-fetched via GetArtDesc(ODARID).
Source: `ORD101.PGM.RPGLE` lines 215–224.

**R9 — Display a recalculated VAT on the detail screen.**
Before showing FMT02 the program calls ClcVAT(GetArtVatCode(ODARID), ODTOT) to produce the
VAT amount, adds it to ODTOT to produce ODTOTVAT, and calls GetVatRate(GetArtVatCode(ODARID))
to show the current rate. These values are for display only; they are not written to the database
at this point.
Source: `ORD101.PGM.RPGLE` lines 221–223.

**R10 — Delivered quantity may not exceed stored ordered quantity.**
Before saving, the program checks DSQTYLIV (user-entered value) against the ODQTY that was
read from the database at load time (not against the newly entered DSQTY). If DSQTYLIV > ODQTY,
indicator 37 (ErrQtyLiv) is set on and error message ERR1001 appears on the DSQTY field:
"Delivered quantity must be lower or equal to ordered quantity."
The screen is redisplayed and no save occurs.
Source: `ORD101.PGM.RPGLE` lines 247–250; `ORD101D.DSPF` line 135; `SAMMSGF.MSGF` ERR1001.

**R11 — Ordered quantity may not be reduced below already-delivered quantity.**
If DSQTY (user-entered) < ODQTYLIV stored on the database record at load time, indicator 38
(ErrQty) is set on and error message ERR1002 appears on the DSQTY field:
"Ordered quantity can not be lower than the quantity already delivered."
The screen is redisplayed and no save occurs.
Source: `ORD101.PGM.RPGLE` lines 251–254; `ORD101D.DSPF` line 117; `SAMMSGF.MSGF` ERR1002.

**R12 — Net line total is DSQTY × DSPRICE, stored in ODTOT.**
After validation passes, ODTOT = DSQTY × DSPRICE. ODTOT is Packed(9:2); if the product
exceeds 9 digits the high-order digits are silently truncated (no overflow error).
Source: `ORD101.PGM.RPGLE` line 255; `common/SAMREF.PF` line 47 (TOTPRICE 9P 2).

**R13 — VAT amount and total with VAT are recalculated before saving.**
ClcVAT(GetArtVatCode(ODARID), ODTOT) computes the VAT amount (see R21–R22 below).
ODTOTVAT = ODTOT + VAT amount.
Source: `ORD101.PGM.RPGLE` lines 256–257.

**R14 — No save if the user changes the screen without pressing Enter normally.**
FMT02 carries the CHANGE keyword on indicator 27. If indicator 27 is on when s02act runs,
the program returns to the display step without writing any changes.
Source: `ORD101D.DSPF` line 92; `ORD101.PGM.RPGLE` lines 261–272.

**R15 — Save writes ODQTY, ODQTYLIV, ODPRICE, ODTOT, and ODTOTVAT back to DETORD1.**
The program assigns ODQTY = DSQTY, ODQTYLIV = DSQTYLIV, ODPRICE = DSPRICE, then executes
`update fdeto`. The record format FDETO includes ODTOT and ODTOTVAT (computed in R12–R13),
so all five fields are written together.
Source: `ORD101.PGM.RPGLE` lines 264–267; `QDDSSRC/DETORD.PF` format FDETO.

**R16 — After saving, the subfile row is refreshed and control returns to the list.**
opt01 is cleared to 0, the subfile row is rewritten (`update sfl01`), and the program sets
step01 = dsp and panel = 1, returning to the line list.
Source: `ORD101.PGM.RPGLE` lines 268–271.

---

### VAT Service Program (vat300.rpgle)

**R17 — VATDEF is opened on first use and cached per code.**
The internal procedure chainVATDEF opens VATDEF lazily (`usropn`, line 65–67). It keeps the
last looked-up code in the module-level variable VATCODE. If the requested code equals VATCODE,
the CHAIN is skipped and the in-memory fields are reused.
Source: `vat300.rpgle` lines 61–73.

**R18 — Before every CHAIN the FVAT data structure is cleared.**
`clear *all FVAT` runs before the CHAIN (line 70). If the code is not found, VATRATE, VATDESC,
and all other FVAT fields are 0 / blank after the call.
Source: `vat300.rpgle` line 70.

**R19 — GetVATRate returns VATRATE for the code, or 0.00 if not found.**
It calls chainVATDEF and returns the VATRATE field (4S 2). An unknown code leaves VATRATE = 0.00.
Source: `vat300.rpgle` lines 23–29.

**R20 — ClcVAT computes VAT as (net × VATRATE) / 100, rounded half-up to 2 decimals.**
Steps:
1. chainVATDEF reads the rate (or leaves VATRATE = 0 for an unknown code).
2. `tot = (net × vatrate) / 100` — stored in a Packed(11:4) intermediate variable; extra
   decimals beyond 4 are truncated, not rounded.
3. `return %dech(tot : 9 : 2)` — half-adjusts to 2 decimal places (rounds 0.005 up to 0.01).
Result type: Packed(9:2).
Source: `vat300.rpgle` lines 40–50.

**R21 — GetVATDesc returns the VAT description, or blank if not found.**
Source: `vat300.rpgle` lines 31–37.

**R22 — ExistVATRate returns true only for a found, non-deleted code.**
It calls chainVATDEF and returns `%found(VATDEF) and VATDEL <> 'X'`.
A soft-deleted code (VATDEL = 'X') returns false even though the rate is still in the table.
Source: `vat300.rpgle` lines 53–59.

---

## Quirks

**Q1 — ClcVAT rounding differs from half-even (banker's rounding).**
The intermediate Packed(11:4) `tot` is truncated to 4 decimal places, then `%dech` rounds half-up
to 2 decimals. For positive amounts this is plain half-up: a 4-decimal value of 0.0650 becomes 0.07.
Java's `BigDecimal.HALF_EVEN` would give 0.06. Any port must match this behaviour.
Reference: `vat300.rpgle` lines 46–49.

**Q2 — Unknown VAT code silently returns rate = 0.**
ORD101 never calls ExistVATRate before calling ClcVAT or GetVATRate. If an article's ARVATCD
is missing from VATDEF, ClcVAT returns 0 and ODTOTVAT equals ODTOT (no tax). No error is shown.
Reference: `vat300.rpgle` lines 70–71; `ORD101.PGM.RPGLE` lines 221, 256.

**Q3 — Deleted VAT code still yields its stored rate.**
VATDEL = 'X' is only checked by ExistVATRate. GetVATRate, ClcVAT, and GetVATDesc all go through
chainVATDEF, which ignores VATDEL. A soft-deleted rate continues to be applied to orders.
Confirmed by the unit tests (lines 41, 59, 67 of `vat300.test.sqlrpgle`).
Reference: `vat300.rpgle` lines 53–73.

**Q4 — chainVATDEF caches the last code for the life of the activation group.**
If the same VATCODE is requested twice in a row, the second call skips the CHAIN. A change to
VATDEF between the two calls is invisible until a different code is requested.
Reference: `vat300.rpgle` lines 68–72.

**Q5 — Error indicators 37 and 38 are set on but never set off.**
ErrQtyLiv (*IN37) and ErrQty (*IN38) are set `*on` in s02chk (lines 248, 252) but have no
matching `= *off` anywhere in ORD101. Once triggered, both error messages remain displayed on
FMT02 for every subsequent visit to that screen in the same run, even after the user corrects
the input.
Reference: `ORD101.PGM.RPGLE` lines 248, 252.

**Q6 — Delivered-quantity check uses the old ordered quantity, not the new one.**
In s02chk, `if dsqtyliv > odqty` compares the user-entered DSQTYLIV against the ODQTY that was
read from the database at load time (S02prp), not against the newly entered DSQTY. A user who
reduces DSQTY and increases DSQTYLIV in the same screen pass may pass validation even if the
new delivered quantity exceeds the new ordered quantity.
Reference: `ORD101.PGM.RPGLE` lines 247–250.

**Q7 — Option 6 = Deliver is a dead branch.**
"6=Deliver" appears in the screen legend (ORD101D.DSPF line 76) and is accepted as valid by the
option checker (s01chk line 152), but s01act contains no `when opt01 = 6` branch. Typing 6 on a
line produces no error and no action.
Reference: `ORD101D.DSPF` line 76; `ORD101.PGM.RPGLE` lines 176–196.

**Q8 — Display-time VAT recalculation is not the same as save-time recalculation.**
S02prp calls ClcVAT on the *stored* ODTOT before displaying FMT02. If VATDEF was updated
between the last save and now, the displayed VAT and ODTOTVAT will reflect the new rate,
while the value in DETORD1 still reflects the old rate. The save (S02act) always recomputes
from scratch, so the inconsistency is resolved on the next save.
Reference: `ORD101.PGM.RPGLE` lines 221–222 vs 255–257.
