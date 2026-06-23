# Modern ABAP for the Returning Developer
### A practical OLD → NEW syntax tutorial (pre-7.4 → 7.4 / 7.5x / ABAP Cloud)

You already know the language. What changed since ~7.4 (2013) is mostly **expressions** replacing **statements**: instead of declaring a variable, then filling it, then using it, you now write one expression that does all three. The compiler infers types, and a lot of `DATA:` / `CALL METHOD` / `READ TABLE` boilerplate disappears.

Mental model: old ABAP was *imperative step-by-step*; new ABAP lets you write *declarative expressions* inline, the way you'd think the result rather than spell out each move. Everything below still compiles the old way — the new way is just shorter and is what code reviews and clean-core now expect.

---

## 1. Inline declarations

The single biggest change. Declare-and-assign in one line; type is inferred from the right-hand side.

```abap
" OLD
DATA: lv_count TYPE i,
      lv_name  TYPE string.
lv_count = 5.
lv_name  = `ABAP`.
```
```abap
" NEW
DATA(lv_count) = 5.        " inferred as i
DATA(lv_name)  = `ABAP`.   " inferred as string
```

Works anywhere a target is written for the first time: `SELECT ... INTO`, `LOOP ... INTO`, `ASSIGNING`, receiving parameters, etc. Use `FINAL(...)` instead of `DATA(...)` (7.55+/Cloud) when the value should never be reassigned — the compiler enforces immutability.

> Gotcha: inline declaration is only allowed at the **write** position, and the variable's scope is the whole method, not just the block — so don't declare the same name twice.

---

## 2. ABAP SQL: inline target + `@` escaping

```abap
" OLD
DATA: lt_carr TYPE TABLE OF scarr.
SELECT * FROM scarr INTO TABLE lt_carr.
```
```abap
" NEW
SELECT carrid, carrname            " comma-separated field list
  FROM scarr
  WHERE carrid = @lv_carrid        " host variables MUST be escaped with @
  INTO TABLE @DATA(lt_carr).       " inline + @ on the target too
```

Three changes at once: field list is comma-separated, every host variable/target gets `@`, and the result table is declared inline. `SELECT *` still works but is discouraged — name your fields (and required outright in ABAP Cloud).

---

## 3. `VALUE` — build structures and tables in one expression

```abap
" OLD — structure
DATA ls_carr TYPE scarr.
ls_carr-carrid   = 'LH'.
ls_carr-carrname = 'Lufthansa'.
```
```abap
" NEW
DATA(ls_carr) = VALUE scarr( carrid = 'LH' carrname = 'Lufthansa' ).
```

```abap
" OLD — table
DATA lt_num TYPE TABLE OF i.
APPEND 1 TO lt_num.
APPEND 2 TO lt_num.
APPEND 3 TO lt_num.
```
```abap
" NEW
DATA(lt_num) = VALUE int4_table( ( 1 ) ( 2 ) ( 3 ) ).

" table of structures
TYPES tt_carr TYPE STANDARD TABLE OF scarr WITH EMPTY KEY.
DATA(lt_carr) = VALUE tt_carr(
  ( carrid = 'LH' carrname = 'Lufthansa' )
  ( carrid = 'AA' carrname = 'American'  ) ).
```

Each `( ... )` is one row. Use `VALUE #( ... )` when the type is already known from context (e.g. a method parameter), and `BASE` to append to an existing value:
```abap
lt_carr = VALUE #( BASE lt_carr ( carrid = 'UA' carrname = 'United' ) ).
```

---

## 4. `NEW` instead of `CREATE OBJECT`

```abap
" OLD
DATA lo_obj TYPE REF TO zcl_car.
CREATE OBJECT lo_obj EXPORTING iv_id = '123'.
```
```abap
" NEW
DATA(lo_obj) = NEW zcl_car( iv_id = '123' ).
```
Use `NEW #( ... )` when the reference type is already known from the target.

---

## 5. `CAST` instead of `?=` (down-casting)

```abap
" OLD
DATA lo_sub TYPE REF TO cl_sub.
lo_sub ?= lo_super.
```
```abap
" NEW
DATA(lo_sub) = CAST cl_sub( lo_super ).

" and you can cast inline mid-chain, no temp variable:
CAST cl_sub( lo_super )->do_something( ).
```

`CONV` is the sibling for value/type conversions when you need to force a type:
```abap
DATA(lv_dec) = CONV decfloat34( lv_int ).
```

---

## 6. `COND` / `SWITCH` — conditionals as expressions

```abap
" OLD
DATA lv_status TYPE string.
IF lv_score >= 50.
  lv_status = 'Pass'.
ELSE.
  lv_status = 'Fail'.
ENDIF.
```
```abap
" NEW
DATA(lv_status) = COND string( WHEN lv_score >= 50 THEN 'Pass' ELSE 'Fail' ).
```

`CASE` becomes `SWITCH`:
```abap
DATA(lv_text) = SWITCH string( lv_type
  WHEN 'A' THEN 'Alpha'
  WHEN 'B' THEN 'Beta'
  ELSE 'Unknown' ).
```

---

## 7. Table expressions instead of `READ TABLE`

```abap
" OLD
DATA ls_carr TYPE scarr.
READ TABLE lt_carr INTO ls_carr WITH KEY carrid = 'LH'.
IF sy-subrc = 0.
  " use ls_carr
ENDIF.
```
```abap
" NEW
DATA(ls_carr) = lt_carr[ carrid = 'LH' ].   " by key
DATA(ls_first) = lt_carr[ 1 ].              " by index
```

> Critical difference: a table expression that finds nothing **raises** `CX_SY_ITAB_LINE_NOT_FOUND` — it does NOT set `sy-subrc`. Guard it:

```abap
IF line_exists( lt_carr[ carrid = 'LH' ] ).
  DATA(ls) = lt_carr[ carrid = 'LH' ].
ENDIF.

DATA(lv_idx) = line_index( lt_carr[ carrid = 'LH' ] ).  " 0 if not found
```

---

## 8. String templates instead of `CONCATENATE`

```abap
" OLD
DATA lv_msg TYPE string.
CONCATENATE 'Hello' lv_name 'welcome' INTO lv_msg SEPARATED BY space.
```
```abap
" NEW
DATA(lv_msg) = |Hello { lv_name } welcome|.
```

Embedded expressions and formatting live inside the `{ }`:
```abap
DATA(lv_out) = |Total: { lv_amount NUMBER = USER } as of { sy-datum DATE = USER }|.
DATA(lv_up)  = |{ lv_text CASE = UPPER }|.
```
For plain concatenation use `&&`:  `lv_a && lv_b`.

---

## 9. String operations are now functions, not statements

```abap
" OLD
REPLACE 'a' WITH 'b' INTO lv_str.
SEARCH lv_str FOR 'xyz'.
```
```abap
" NEW — pure functions, composable inline
lv_str          = replace( val = lv_str sub = 'a' with = 'b' occ = 0 ).
DATA(lv_pos)    = find( val = lv_str sub = 'xyz' ).      " offset, -1 if absent
DATA(lv_part)   = substring( val = lv_str off = 0 len = 3 ).
DATA(lv_upper)  = to_upper( lv_str ).
DATA(lv_len)    = strlen( lv_str ).
```

---

## 10. Functional & chained method calls

```abap
" OLD
DATA lv_result TYPE i.
CALL METHOD lo_calc->add
  EXPORTING iv_a = 2 iv_b = 3
  RECEIVING rv_sum = lv_result.
```
```abap
" NEW
DATA(lv_result) = lo_calc->add( iv_a = 2 iv_b = 3 ).

" single importing parameter: drop the name
DATA(lv_sq) = lo_calc->square( 4 ).

" chain calls, no intermediate variables
DATA(lv_name) = lo_factory->get_car( '123' )->get_name( ).

" a method call can be an argument directly
write_log( lo_calc->add( iv_a = 2 iv_b = 3 ) ).
```

`CALL METHOD` is effectively dead — call methods functionally.

---

## 11. Inline work areas & field-symbols in `LOOP`

```abap
" OLD
DATA ls_carr TYPE scarr.
FIELD-SYMBOLS <fs> TYPE scarr.
LOOP AT lt_carr INTO ls_carr.
ENDLOOP.
LOOP AT lt_carr ASSIGNING <fs>.
ENDLOOP.
```
```abap
" NEW
LOOP AT lt_carr INTO DATA(ls_carr).
ENDLOOP.

LOOP AT lt_carr ASSIGNING FIELD-SYMBOL(<fs>).
  <fs>-carrname = 'X'.   " modify in place, no MODIFY needed
ENDLOOP.
```

Prefer `ASSIGNING FIELD-SYMBOL(...)` when you'll change the row — it edits the table directly and avoids a copy.

---

## 12. `FOR` — table comprehensions (map / filter without LOOP)

This replaces the "loop, build a row, append" pattern entirely.

```abap
" map: pull one column into a new table
DATA(lt_names) = VALUE string_table( FOR ls IN lt_carr ( ls-carrname ) ).

" filter + map
DATA(lt_lh) = VALUE tt_carr( FOR ls IN lt_carr WHERE ( carrid = 'LH' ) ( ls ) ).

" generate a sequence
DATA(lt_seq) = VALUE int4_table( FOR i = 1 THEN i + 1 WHILE i <= 5 ( i ) ).
```

---

## 13. `REDUCE` — collapse a table to a single value

```abap
" OLD
DATA lv_sum TYPE i.
LOOP AT lt_items INTO DATA(ls_item).
  lv_sum = lv_sum + ls_item-amount.
ENDLOOP.
```
```abap
" NEW
DATA(lv_sum) = REDUCE i(
  INIT x = 0
  FOR ls IN lt_items
  NEXT x = x + ls-amount ).
```

`INIT` seeds the accumulator, `NEXT` updates it each iteration.

---

## 14. `CORRESPONDING` instead of `MOVE-CORRESPONDING`

```abap
" OLD
MOVE-CORRESPONDING ls_source TO ls_target.
```
```abap
" NEW
DATA(ls_target) = CORRESPONDING ty_target( ls_source ).

" with field remapping and exclusions
DATA(ls_t) = CORRESPONDING ty_target( ls_source
  MAPPING target_field = source_field
  EXCEPT  field_to_skip ).
```

Works on internal tables too, and `CORRESPONDING ... DEEP` handles nested structures the old statement couldn't.

---

## 15. `FILTER` — extract matching rows fast

```abap
DATA(lt_active) = FILTER #( lt_all WHERE status = 'A' ).
```
Requires the filtered component to be covered by a sorted/hashed key (or specify `USING KEY`). Faster than a LOOP for large tables.

---

## 16. `xsdbool` — boolean in one line

```abap
" OLD
DATA lv_flag TYPE abap_bool.
IF lv_x > 0.
  lv_flag = abap_true.
ELSE.
  lv_flag = abap_false.
ENDIF.
```
```abap
" NEW
DATA(lv_flag) = xsdbool( lv_x > 0 ).
```
(Use `xsdbool`, not `boolc` — `boolc` returns type `c` and bites you in comparisons.)

---

## 17. `LOOP ... GROUP BY` — grouping without manual control breaks

Replaces `AT NEW` / `AT END OF` control-level processing.

```abap
LOOP AT lt_sales INTO DATA(ls) GROUP BY ( region = ls-region )
     INTO DATA(lv_group).
  " one iteration per region
  LOOP AT GROUP lv_group INTO DATA(ls_member).
    " members of this region
  ENDLOOP.
ENDLOOP.
```

---

## 18. ABAP Cloud / clean-core: what's now *forbidden*

If your target is S/4HANA clean core, the BTP ABAP environment (Steampunk), or the "ABAP for Cloud Development" language version, the modern syntax above isn't optional — and a lot of the classic ABAP you knew won't even compile:

- **No** `WRITE` to list, classic Dynpro, or selection screens — UI is RAP + Fiori only.
- **No** header lines, `OCCURS`, `TABLES`, or short-form `MOVE`.
- **No** `SELECT *` without an explicit need; name your fields.
- **No** direct access to non-released DB tables, function modules, or classes — only **released (C1) public APIs**. Check release state before you depend on anything.
- **No** `CALL FUNCTION` to unreleased FMs; **no** direct `CALL TRANSACTION`.
- Strict syntax check is always on (the old "Unicode checks" era, but stricter).

Practical takeaway: the modern expression syntax *is* the clean-core dialect. Learning it isn't cosmetic — it's the entry ticket to anything running on S/4 cloud or BTP.

---

## Quick reference

| Task | Old | New |
|---|---|---|
| Declare + assign | `DATA x TYPE i.` then `x = 5.` | `DATA(x) = 5.` |
| Read into table | `SELECT * INTO TABLE lt.` | `SELECT f1, f2 INTO TABLE @DATA(lt).` |
| Build structure | field-by-field assignment | `VALUE ty( a = 1 b = 2 )` |
| Build table | repeated `APPEND` | `VALUE tt( ( ) ( ) )` |
| Instantiate | `CREATE OBJECT lo.` | `NEW cls( )` |
| Down-cast | `lo ?= ...` | `CAST cls( ... )` |
| Convert type | implicit / temp var | `CONV type( ... )` |
| Conditional value | `IF/ELSE` block | `COND type( WHEN ... THEN ... )` |
| Case value | `CASE` block | `SWITCH type( x WHEN ... THEN ... )` |
| Read one row | `READ TABLE ... WITH KEY` | `lt[ key = val ]` + `line_exists( )` |
| Concatenate | `CONCATENATE ... INTO` | `\| text { var } \|` |
| Replace/find | `REPLACE` / `SEARCH` | `replace( )` / `find( )` |
| Call method | `CALL METHOD ... RECEIVING` | `lo->m( a = 1 )` |
| Loop work area | declare WA, `LOOP INTO ws` | `LOOP INTO DATA(ws)` |
| Map/filter table | LOOP + APPEND | `VALUE tt( FOR x IN lt ... )` |
| Sum/aggregate | LOOP + accumulate | `REDUCE type( INIT ... FOR ... NEXT ... )` |
| Copy fields | `MOVE-CORRESPONDING` | `CORRESPONDING ty( src )` |
| Boolean | `IF/ELSE` set flag | `xsdbool( cond )` |
| Group rows | `AT NEW` / `AT END OF` | `LOOP ... GROUP BY ... / LOOP AT GROUP` |
