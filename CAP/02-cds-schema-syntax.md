# CDS Schema Syntax Reference

The data-modeling language (CDL) you use in `db/schema.cds`. Summarized but complete — keep it open while you model. Service-layer syntax (`srv/*.cds`) is a separate concern and not covered here.

## Table of contents

1. [File header](#file-header)
2. [Entities & keys](#entities--keys)
3. [Built-in types](#built-in-types)
4. [Element modifiers & constraints](#element-modifiers--constraints)
5. [Reusable aspects (the : mixin)](#reusable-aspects-the--mixin)
6. [Associations (references, no ownership)](#associations-references-no-ownership)
7. [Compositions (containment / ownership)](#compositions-containment--ownership)
8. [Enums](#enums)
9. [Custom types](#custom-types)
10. [Structured & nested elements](#structured--nested-elements)
11. [Localized text](#localized-text)
12. [Calculated & virtual elements](#calculated--virtual-elements)
13. [Validation annotations (schema-level)](#validation-annotations-schema-level)
14. [Persistence-control annotations](#persistence-control-annotations)
15. [Extend & annotate existing definitions](#extend--annotate-existing-definitions)
16. [Comments](#comments)
17. [Gotchas worth remembering](#gotchas-worth-remembering)
---

## File header

```cds
namespace energy.inspections;                              // prefixes every name in the file

using { cuid, managed, Currency } from '@sap/cds/common';  // reusable aspects & code lists
using { energy.shared.Asset } from './shared';             // import from another .cds file
```

- `namespace` is optional but standard — it makes fully-qualified names (`energy.inspections.Inspections`), which also drive CSV filenames (`energy.inspections-Inspections.csv`).
- `using` imports definitions. `@sap/cds/common` is the SAP-provided grab bag (`cuid`, `managed`, `temporal`, `Currency`, `Country`, `Language`).

---

## Entities & keys

```cds
entity Inspections {
  key ID        : UUID;          // single key
  assetID       : String(20);
  performedAt   : Timestamp;
}

entity LineItem {
  key order     : Association to Orders;   // composite key
  key pos       : Integer;
}
```

- `key` marks primary-key elements. Repeat it for composite keys.
- An entity with no `key` is legal (e.g. a view projection) but won't get a generated table key.

---

## Built-in types

| Type | Notes |
|---|---|
| `UUID` | string GUID; the default key type |
| `Boolean` | true/false |
| `Integer`, `Int16`, `Int32`, `Int64`, `UInt8` | `Integer` = Int32 |
| `Decimal(p,s)` | fixed precision/scale — use for money |
| `Double` | floating point |
| `Date`, `Time`, `DateTime`, `Timestamp` | `Timestamp` is the high-precision one |
| `String(n)` | length optional; defaults to a DB-specific max if omitted |
| `LargeString` | CLOB / long text |
| `Binary(n)`, `LargeBinary` | BLOB |
| `Vector(n)` | embeddings (newer; HANA Cloud — verify availability) |

Rule of thumb: always size your `String(n)` and use `Decimal(p,s)` for anything financial — unbounded strings and `Double` money are the two most common modeling smells.

---

## Element modifiers & constraints

```cds
entity E {
  key ID    : UUID;
  name      : String(100) not null;
  active    : Boolean default true;
  createdAt : Timestamp default $now;        // expression defaults: $now, $user
  amount    : Decimal(10,2) @mandatory;      // UI + input validation
  note      : String @readonly;
}
```

- `not null` → DB-level constraint. `@mandatory` → CAP input validation + Fiori "required". They overlap but aren't identical; use both for a hard-required field.
- `default` takes a literal or `$now` / `$user`.

---

## Reusable aspects (the `:` mixin)

```cds
entity Inspections : cuid, managed {
  assetID : String(20);
}
```

Common aspects from `@sap/cds/common`:
- `cuid` → adds `key ID : UUID;`
- `managed` → adds `createdAt`, `createdBy`, `modifiedAt`, `modifiedBy`, auto-maintained on write
- `temporal` → adds `validFrom`, `validTo` for time-slice data

Define your own:

```cds
aspect trackable {
  source : String;
  region : String;
}
entity X : cuid, trackable { ... }   // mix in as many as you like, comma-separated
```

An `aspect` is a named, reusable bundle of elements/annotations. It's the DRY mechanism — pull shared columns into an aspect instead of repeating them.

---

## Associations (references, no ownership)

```cds
// to-one, managed — auto-creates FK column author_ID
author : Association to Authors;

// to-one, explicit FK condition
author : Association to Authors on author.ID = author_ID;
author_ID : UUID;

// to-many — ALWAYS needs an `on` backlink
books : Association to many Books on books.author = $self;
```

- **Managed to-one** is the everyday case: CAP generates the foreign-key column (`<name>_<key>`) for you. Don't declare the FK yourself unless you need it explicitly.
- **To-many always requires `on`** pointing back via the other side's association and `$self`.
- Association = a pointer. Deleting the source does **not** delete the target.

---

## Compositions (containment / ownership)

```cds
entity Orders : cuid {
  items : Composition of many Items on items.parent = $self;
}
entity Items : cuid {
  parent : Association to Orders;
  descr  : String;
}
```

- Composition = parent **owns** children. Delete the parent → children go with it. Children have no independent life cycle.
- This is what makes Order+Items one deep document for read/write (deep insert, deep update).
- **Composition of aspect** (inline, no separate named entity — CAP fully manages the backlink):

```cds
entity Orders : cuid {
  items : Composition of many {
    key pos : Integer;
    descr   : String;
  };
}
```

Use a named entity when children need to be addressable on their own; use the inline aspect form when they're purely internal detail.

Association vs Composition in one line: **Association points at something that exists independently; Composition contains something that lives and dies with the parent.**

---

## Enums

```cds
status : String enum { draft; submitted; actioned; } default 'draft';

result : String enum {
  pass = 'P';        // explicit DB value differs from name
  fail = 'F';
  na   = 'N';
};

priority : Integer enum { low = 1; medium = 2; high = 3; };
```

Names are what you use in code/annotations; the `= value` is what's stored. Omit the value and the name is stored.

---

## Custom types

```cds
type Name   : String(100);                  // simple alias
type Amount {                               // structured type
  value    : Decimal(10,2);
  currency : Currency;
}

entity E {
  contact : Name;
  total   : Amount;                         // reuse anywhere
}
```

Define once, reuse across entities. Good for domain primitives (an ISIN, a meter ID, a money amount) so the rule lives in one place.

---

## Structured & nested elements

```cds
entity E {
  key ID : UUID;
  address {                 // inline structure
    street : String;
    city   : String;
    zip    : String(10);
  }
}
```

Flattens to `address_street`, `address_city`, etc. in the table.

---

## Localized text

```cds
entity Products : cuid {
  title : localized String(100);
  descr : localized String;
}
```

`localized` auto-generates a `Products.texts` entity keyed by language, plus the right joins on read. You get translatable fields without modeling the text table yourself.

---

## Calculated & virtual elements

```cds
entity E {
  price    : Decimal(10,2);
  tax      : Decimal(10,2);
  gross    : Decimal(10,2) = price + tax;   // calculated on read, not stored
  virtual uiFlag : Boolean;                 // never persisted; transient for UI/logic
}
```

- `= expression` → computed at read time, no column.
- `virtual` → exists in the service payload but has no DB column.

---

## Validation annotations (schema-level)

```cds
entity E {
  email  : String  @assert.format: '^[^@]+@[^@]+$';
  rating : Integer @assert.range : [ 1, 5 ];          // inclusive bounds
  code   : String  @assert.unique;                    // single-field uniqueness
}

@assert.unique: { byAssetAndDate: [ assetID, performedAt ] }   // multi-field, entity-level
entity Inspections { ... }
```

These run in CAP's generic handlers before persistence — free validation without writing handler code.

---

## Persistence-control annotations

```cds
@cds.persistence.skip    entity Calc { ... }   // no table at all (logic-only)
@cds.persistence.exists  entity Legacy { ... } // table already exists; don't deploy DDL
```

Reach for these when an entity is a pure projection/calculation or maps onto something the DB already has — stops CAP from generating an unwanted table.

---

## Extend & annotate existing definitions

```cds
extend entity Inspections with {
  weatherCode : String(4);              // add a field to an existing entity
}

annotate Inspections with {
  assetID @mandatory;                   // attach annotations without touching the original
}
```

The clean way to layer onto SAP-delivered or imported models without modifying the source file — same spirit as clean-core extension, one level down.

---

## Comments

```cds
// line comment
/* block comment */
/** doc comment — surfaces as @description metadata */
```

---

## Gotchas worth remembering

- **To-many without `on`** → error. Managed FKs are only auto-generated for to-**one**.
- **`String` with no length** → silently becomes a DB-specific default max; size it deliberately.
- **`Double` for money** → rounding bugs. Always `Decimal(p,s)`.
- **Composition vs Association mix-up** → wrong delete behaviour. Owned child = Composition; independent reference = Association.
- **CSV filename must match the fully-qualified name** with `-` instead of the last `.` (`energy.inspections-Inspections.csv`), and the association column is `<assoc>_<key>` (e.g. `parent_ID`).
- **Annotation vs constraint**: `@mandatory` is validation/UX, `not null` is a DB constraint — they're not the same; use both for hard-required fields.
- **Namespace drives everything downstream** — change it later and you rename CSVs, service references, and deployed artifacts. Pick it once.

---

*CAP/CDS evolves; if a construct behaves oddly check `cap.cloud.sap` (CDL reference) for your cds version.*
