# Contract: data files (v2.4)

`|`-separated, one record per line, LF, whole-file rewrite on every save (unchanged).

## customers.dat

```
id|name|phone|email|type|fleet
2008|Greenway Taxis|01632 960702|fleet@greenwaytaxis.example|R|1
```

| # | Field | Write | Read |
|---|-------|-------|------|
| 1-4 | id, name, phone, email | as today | **empty fields are kept** (no shifting) |
| 5 | type | `%c` | the first char; missing or empty → `R` |
| 6 | fleet (new) | always `0` or `1` | `1` only if exactly `1`; otherwise 0 |

## invoices.dat

```
no|date|veh|cust|list|disc|net|vat|fees|tradein|total|fleet_disc
```

| # | Field | Write | Read |
|---|-------|-------|------|
| 1-11 | as today (`disc` = all discounts) | as today | empty fields kept; missing → 0 |
| 12 | fleet_disc (new) | always `%.2f` | missing or empty → 0 |

`stock.dat` and `next_ids.dat` are unchanged (E-03). The stock loader keeps `strtok`.

## Compatibility matrix

| Reader ↓ / file → | v2.3 file (5/11 fields) | v2.4 file (6/12 fields) |
|---|---|---|
| v2.4 binary | loads; every customer non-fleet, fleet_disc 0 (FR-008) | loads fully |
| v2.3 binary (`dealerdesk.old`) | as today | loads, ignores the trailing fields, totals correct. **Its next save drops field 6/12** (E-05): documented in RELEASE.txt/NOTES.md, checked by `make test-compat` (reading only) |

Rows containing `||`: v2.3 shifts the later fields (C-16), v2.4 does not (E-04, scenario 28).
