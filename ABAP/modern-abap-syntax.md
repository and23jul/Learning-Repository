# Modern ABAP for the Returning Developer
### A practical OLD → NEW syntax tutorial (pre-7.4 → 7.4 / 7.5x / ABAP Cloud) + tooling guide

You already know the language. What changed since ~7.4 (2013) is mostly **expressions** replacing **statements**: instead of declaring a variable, then filling it, then using it, you now write one expression that does all three. The compiler infers types, and a lot of `DATA:` / `CALL METHOD` / `READ TABLE` boilerplate disappears.

Mental model: old ABAP was *imperative step-by-step*; new ABAP lets you write *declarative expressions* inline, the way you'd think the result rather than spell out each move. Everything below still compiles the old way — the new way is just shorter and is what code reviews and clean-core now expect.

> **Part I** is the language. **Part II** is the tooling (Eclipse/ADT vs SAP GUI/SE80) — because the clean-core dialect is written in Eclipse, and half the modern artifacts have no editor in SAP GUI at all.

---

# Part I — Language

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

> This is only the shallow end. Modern ABAP SQL can do JOINs, aggregates, `CASE`, arithmetic and CTEs **inside the SELECT** — see §20. If you're still pulling rows into ABAP to process them in a loop, you're leaving the biggest HANA win on the table.

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

> When a *lossy* conversion should fail loudly instead of silently truncating, use `EXACT` — see §22.

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

> Critical difference: a table expression that finds nothing **raises** `CX_SY_ITAB_LINE_NOT_FOUND` — it does NOT set `sy-subrc`.

The idiomatic "get the row or nothing" is `OPTIONAL` / `DEFAULT` — **one** table access, no exception:

```abap
" preferred: single lookup, returns an initial-value row if not found
DATA(ls) = VALUE #( lt_carr[ carrid = 'LH' ] OPTIONAL ).

" or supply your own fallback row
DATA(ls) = VALUE #( lt_carr[ carrid = 'LH' ] DEFAULT ls_fallback ).
```

Use `line_exists( )` / `line_index( )` only when you genuinely just want the boolean or the index — don't pair `line_exists( )` with a second `lt[...]` read, that scans the table **twice**:

```abap
" AVOID: two lookups for one row
IF line_exists( lt_carr[ carrid = 'LH' ] ).
  DATA(ls_bad) = lt_carr[ carrid = 'LH' ].   " second scan
ENDIF.

DATA(lv_idx) = line_index( lt_carr[ carrid = 'LH' ] ).  " 0 if not found
```

If you *want* the exception (the row must exist and its absence is a real error), wrap it in `TRY/CATCH` — see §19.

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

> Note: string templates always produce type `string`. If you need a fixed-length `c` target you'll get a conversion — usually fine, but worth knowing.

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

> **Where the data lives matters.** `REDUCE` is right for a table you *already hold in memory*. But if `lt_items` came straight from a `SELECT`, don't pull every row over the DB interface just to add them up in ABAP — aggregate in the database: `SELECT SUM( amount ) FROM items WHERE ... INTO @DATA(lv_sum).` Same rule for `COUNT`, `MAX`, `AVG`. Code-to-data beats data-to-code on HANA every time (§20).

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

> Use `xsdbool`, not `boolc`. `boolc` returns a **`string`** whose false value is a *zero-length* string; `xsdbool` returns **`char1`** ('X' / ' ') that matches `abap_bool` exactly. Compare a `boolc` result against `abap_false` and the length mismatch bites you — `xsdbool` is the type-safe one.

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

## 19. Class-based exceptions & `TRY / CATCH`

Modern ABAP is exception-*class* based, not `sy-subrc`-based. Table expressions (§7), `CAST`, `EXACT`, arithmetic overflow — all throw. If you don't catch, you dump.

```abap
" catch what a table expression throws
TRY.
    DATA(ls_carr) = lt_carr[ carrid = 'LH' ].
  CATCH cx_sy_itab_line_not_found INTO DATA(lx).
    DATA(lv_text) = lx->get_text( ).   " human-readable message
ENDTRY.
```

Raising your own — two modern forms:

```abap
" classic form
RAISE EXCEPTION TYPE zcx_my_error
  EXPORTING mv_detail = lv_x.

" 7.52+ combined NEW form (shorter)
RAISE EXCEPTION NEW zcx_my_error( mv_detail = lv_x ).

" attach a T100 message so it surfaces cleanly in logs / Fiori
RAISE EXCEPTION TYPE zcx_my_error
  MESSAGE e001(zmsg) WITH lv_a lv_b.
```

Structure:
- `TRY. ... CATCH cx_a cx_b INTO DATA(lx). ... CLEANUP. ... ENDTRY.` — `CLEANUP` runs only when an exception leaves the block (release locks, roll back).
- Catch **specific** classes before generic (`cx_root` last).
- Inherit from `cx_static_check` (must declare/catch), `cx_dynamic_check` (need not declare), or `cx_no_check` (runtime, e.g. `cx_sy_...`).

> Rule of thumb: use `OPTIONAL`/`DEFAULT` (§7) when "not found" is normal; use `TRY/CATCH` when "not found" is a genuine error worth a message.

---

## 20. Modern ABAP SQL — push the work into the database

The single biggest performance shift with HANA is **code-to-data**: do the filtering, joining and aggregating in the `SELECT`, not in ABAP loops afterward. §2 is just the syntax; this is the point.

```abap
" JOIN + aggregate + alias, all in the DB
SELECT c~carrid, c~carrname,
       SUM( b~loccuram ) AS revenue,
       COUNT(*)          AS bookings
  FROM scarr AS c
  INNER JOIN sbook AS b ON b~carrid = c~carrid
  WHERE b~fldate >= @lv_from
  GROUP BY c~carrid, c~carrname
  HAVING SUM( b~loccuram ) > @lv_threshold
  ORDER BY revenue DESCENDING
  INTO TABLE @DATA(lt_rev).
```

`CASE`, arithmetic and string functions work in the SELECT list too:

```abap
SELECT carrid,
       CASE WHEN loccuram > 1000 THEN 'HIGH' ELSE 'LOW' END AS tier,
       loccuram * @lv_rate AS converted
  FROM sbook
  INTO TABLE @DATA(lt_x).
```

Common table expressions (`WITH`, note the `+` prefix on the CTE name):

```abap
WITH
  +big AS ( SELECT carrid, SUM( loccuram ) AS total
              FROM sbook GROUP BY carrid )
  SELECT c~carrname, b~total
    FROM +big AS b
    INNER JOIN scarr AS c ON c~carrid = b~carrid
    INTO TABLE @DATA(lt_out).
```

> **`FOR ALL ENTRIES` — the trap that will page you at 2am:** if the driver table is **empty**, the `WHERE` condition built on it is ignored and you select the **whole table**. Always guard: `IF lt_keys IS NOT INITIAL.` FAE also silently removes duplicates and gives no order guarantee. Where a JOIN or a subquery expresses the same thing, prefer it — it's one DB round trip instead of packet-batched many.

---

## 21. `LET` — name a subexpression once

Inside `VALUE` / `COND` / `SWITCH` / `REDUCE` / `FOR`, `LET` binds a local name so you don't recompute or repeat it.

```abap
DATA(lv_status) = COND string(
  LET score = expensive_calc( ) IN
  WHEN score >= 80 THEN 'A'
  WHEN score >= 50 THEN 'B'
  ELSE 'F' ).
```

`expensive_calc( )` runs once, not three times. Keeps constructor expressions readable when the same value appears in several branches.

---

## 22. `EXACT` — conversion that refuses to lose data

`CONV` (§5) converts silently, truncating or rounding if it must. `EXACT` does the same conversion but **raises** if the result isn't a lossless, valid representation.

```abap
DATA(lv_ok)  = EXACT decfloat34( '12.34' ).     " fine
DATA(lv_bad) = EXACT i( '12.99' ).              " raises cx_sy_conversion_error — not silent 13
```

Use it wherever a bad input should fail loudly instead of corrupting data — parsing user/file input, financial figures, external interface fields.

---

## 23. Enumerations

Replaces the old "interface full of constants" pattern with a real, type-checked enum (7.51+).

```abap
TYPES:
  BEGIN OF ENUM ty_status,
    open,
    in_process,
    closed,
  END OF ENUM ty_status.

DATA(lv_state) = open.
IF lv_state = closed. " ... ENDIF.
```

The compiler restricts the variable to the declared values — you can't accidentally assign a stray literal, which the constants pattern never enforced.

---

# Part II — Tooling: Eclipse (ADT) vs SAP GUI (SE80)

If Part I is the *language* shift, this is the *tooling* shift that comes with it — and they're linked. The clean-core dialect is designed to be written in Eclipse, and Eclipse's quick-fixes will literally rewrite your old syntax into the new expressions for you.

**Mental model:** SE80 is the all-in-one workshop you grew up in — every tool bolted to one bench, one window, one system at a time. ADT (ABAP Development Tools, running on Eclipse) is a modern IDE: project-based, multi-window, source-driven. For anything clean-core, **ADT isn't a preference — it's the only door**. CDS, RAP, AMDP and ABAP Cloud have no editor in SAP GUI.

## The one-line answer

- Touching **CDS, RAP, AMDP, ABAP Cloud, or S/4 clean-core**? → **ADT only.** SE80 can't open these objects.
- Doing **legacy ECC break-fix on screens, SAPscript, or Smart Forms**? → **SE80**, because ADT has no editor for those.
- Everything else (classes, reports, function modules, DDIC)? → **ADT**, it's just a better editor. In practice: ADT primary, SAP GUI open beside it for the GUI-only bits.

## What you can build where

| Artifact / task | SE80 (SAP GUI) | ADT (Eclipse) |
| --- | --- | --- |
| Classic report / executable program | Yes | Yes |
| Global class / interface | Yes | Yes (better editor) |
| Function module / group | Yes | Yes |
| DDIC: tables, structures, data elements, domains | Yes (SE11) | Yes (most; a few still push to SE11) |
| Dynpro / screen painter (SE51) | Yes | **No** |
| Menu painter (SE41) | Yes | **No** |
| Module pool | Yes | Logic yes, **screens no** |
| SAPscript (SE71) | Yes | **No** |
| Smart Forms | Yes | **No** |
| CDS view / DDL source | **No** | **ADT only** |
| CDS access control (DCL) | **No** | **ADT only** |
| AMDP (ABAP Managed DB Procedure) | **No** | **ADT only** |
| RAP behavior definition + implementation | **No** | **ADT only** |
| Service definition / binding (OData/RAP) | **No** | **ADT only** |
| ABAP Cloud / Steampunk (BTP ABAP env) | **No GUI at all** | **ADT only** |
| abapGit | via report (`ZABAPGIT`) | native plugin (smoother) |

The pattern: everything invented **after** the HANA/clean-core pivot lives in ADT exclusively; everything screen- or print-form-based from the classic era stays in SE80.

## Day-to-day editor differences

| Aspect | SE80 | ADT (Eclipse) |
| --- | --- | --- |
| Layout | Single window, object tree | Multi-tab, side-by-side editors, project-based |
| Code completion | Basic | Rich `Ctrl+Space`, live templates |
| Refactor / quick fix | Minimal | **`Ctrl+1`** quick assists: rename, extract method, create missing method, **convert classic → modern syntax** |
| Navigation | SE80 where-used | `F3` hyperlink-navigate into any source, including standard SAP |
| Unit tests | Run separately | Inline ABAP Unit runner + coverage |
| Static checks (ATC) | Separate transaction | Inline, as-you-type and on activation |
| Debugging | New debugger in GUI | ADT debugger (same backend, nicer UX) |
| Multiple systems | One per session | Many projects/systems in one workbench |
| Diff / merge | Limited | Eclipse compare, Git-friendly |

The two shortcuts that change your life: **`Ctrl+Space`** (completion) and **`Ctrl+1`** (quick fix). `Ctrl+1` on old syntax will often offer to rewrite it into the modern expression form from Part I — the fastest way to *learn* the new syntax is to watch it convert your own code.

## Gotchas when you switch to ADT

- **Backend version matters.** ADT + CDS need ~7.40 SP05+; full modern comfort wants a higher SP; RAP needs ~7.54 / S/4HANA 1909+ (or ABAP Cloud). An old ECC 6.0 box caps what ADT gives you.
- **You still need SAP GUI installed.** ADT hands off to the embedded GUI for the GUI-only tools (screens, SAPscript, some transactions).
- **Eclipse concepts are new vocabulary:** workspace, projects, perspectives. Budget an afternoon for the remap — it's not ABAP that's new, it's Eclipse.
- **Activation model feels different** — inactive versions, mass-activate — but it's the same underlying workbench.

## Bottom line

For the clean-core / S/4 / BTP direction Part I is aimed at, ADT is mandatory and SE80 is a dead end — SAP has stopped adding new features to SE80. Learn ADT now on your legacy work so the muscle memory is there when the CDS/RAP work lands. Keep SE80 in your back pocket purely for the classic screen-and-form objects it still owns.

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
| Convert, fail if lossy | manual check | `EXACT type( ... )` |
| Conditional value | `IF/ELSE` block | `COND type( WHEN ... THEN ... )` |
| Case value | `CASE` block | `SWITCH type( x WHEN ... THEN ... )` |
| Read one row (or nothing) | `READ TABLE`+`sy-subrc` | `VALUE #( lt[ key = val ] OPTIONAL )` |
| Read one row (must exist) | `READ TABLE`+`sy-subrc` | `lt[ key = val ]` in `TRY/CATCH` |
| Concatenate | `CONCATENATE ... INTO` | `\| text { var } \|` |
| Replace/find | `REPLACE` / `SEARCH` | `replace( )` / `find( )` |
| Call method | `CALL METHOD ... RECEIVING` | `lo->m( a = 1 )` |
| Loop work area | declare WA, `LOOP INTO ws` | `LOOP INTO DATA(ws)` |
| Map/filter table | LOOP + APPEND | `VALUE tt( FOR x IN lt ... )` |
| Sum/aggregate (in memory) | LOOP + accumulate | `REDUCE type( INIT ... FOR ... NEXT ... )` |
| Sum/aggregate (from DB) | LOOP + accumulate | `SELECT SUM( ) ... GROUP BY` |
| Copy fields | `MOVE-CORRESPONDING` | `CORRESPONDING ty( src )` |
| Boolean | `IF/ELSE` set flag | `xsdbool( cond )` |
| Group rows | `AT NEW` / `AT END OF` | `LOOP ... GROUP BY ... / LOOP AT GROUP` |
| Raise error | `MESSAGE` / `RAISE` | `RAISE EXCEPTION NEW zcx_...( )` |
| Constant set | interface of constants | `BEGIN OF ENUM ... END OF ENUM` |
| Name a subexpression | temp variable | `LET x = ... IN` |
| Where to write it | SE80 | ADT (Eclipse) — mandatory for CDS/RAP/Cloud |
