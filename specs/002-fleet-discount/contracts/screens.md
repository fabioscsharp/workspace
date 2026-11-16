# Contract: screen text (v2.4)

This file lists only what changes. Every other screen prints exactly as in v2.3 (FR-007).
`␣` marks significant spaces where they matter. Money uses the existing `%12.2f` column.

## Header (all screens)

`  DEALERDESK v2.4 %44s` (only `2.3` → `2.4`; normalised to `vX.Y` in goldens).

## Customers menu

```
  1. List customers
  2. Add customer
  3. Find customer
  4. Set fleet status        <- new
  0. Back
```

## Add Customer

The prompts in order are Name, Phone, Email, `  Trade customer (Y/N)? `, then the new
`  Fleet customer (Y/N)? `. Y/y means fleet, and anything else means not fleet (as for
trade). The rest is unchanged ("Name is required.", "Customer N added.").

## Set Fleet Status (new, Customers menu option 4)

```
<header "Set Fleet Status">
  Customer ID : <id>
```

- Unknown id: `\n  Customer not found.\n`, then the ENTER pause.
- Known id:

```

  Customer : <name> (<type text>)
  Fleet    : Yes | No

  Fleet customer (Y/N)? <answer>
```

  then one of:
  - `Y`/`y`: `\n  Customer <id> is now a fleet customer.\n`, and `customers.dat` is saved
  - `N`/`n`: `\n  Customer <id> is no longer a fleet customer.\n`, and `customers.dat` is saved
  - anything else, including empty: `\n  Fleet status not changed.\n`, and nothing is saved

  then the ENTER pause.

## Type text (customer list, find, quote/preview customer line)

`Retail`, `Trade` (unchanged); `Retail, Fleet`, `Trade, Fleet` (fleet only, D-10).
List row: the last column is the type text (`%s`, so it is not truncated).
Quote line: `  Customer : <name> (<type text>)`.

## Quote and invoice preview

```
  List price               %12.2f
  Trade discount           %12.2f      (if trade_disc > 0, unchanged)
  Fleet discount           %12.2f      (new: if fleet_disc > 0, value -fleet_disc)
  Discount (%4.1f%%)         %12.2f      (if disc > 0, unchanged)
  Net price … TOTAL                    (unchanged)
```

## Invoice view

Non-fleet invoices (`fleet_disc == 0`) are byte-identical to v2.3, including `-0.00` when
`disc` is 0. Fleet invoices:

```
  List price               %12.2f
  Discounts                %12.2f      (-(disc - fleet_disc); 0.00 unsigned when zero)
  Fleet discount           %12.2f      (new: -fleet_disc)
  Net price … TOTAL                    (unchanged)
```

## Help

A new line directly after `  - Trade customers receive a 5%% trade discount.\n`:
`  - Fleet customers receive an automatic 4% fleet discount.` (`%%` in the format
string). The other lines are unchanged.
