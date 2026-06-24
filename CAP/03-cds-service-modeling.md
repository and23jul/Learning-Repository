# CDS Service Modeling — Variations Reference

The `srv/*.cds` layer: the different ways to expose, reshape, restrict, and extend what your service hands to the outside world. Companion to the schema reference — that one was *what the data is*, this one is *what you expose and how*. Summarized but complete.

The mental model: a service is a **facade**. The DB model is your private warehouse; the service is the shopfront. Every variation below is a different way to decide what's on the shelf, how it's labelled, and who's allowed to touch it.

## Table of contents

1. [Service definition & path](#service-definition--path)
2. [The exposure spectrum (the core variations)](#the-exposure-spectrum-the-core-variations)
3. [Associations in a service — redirection](#associations-in-a-service--redirection)
4. [Access shaping](#access-shaping)
5. [Draft handling (Fiori)](#draft-handling-fiori)
6. [Actions & functions](#actions--functions)
7. [Events (async / messaging)](#events-async--messaging)
8. [Authorization](#authorization)
9. [Protocols](#protocols)
10. [Service as a facade over a remote (your ECC case)](#service-as-a-facade-over-a-remote-your-ecc-case)
11. [Gotchas worth remembering](#gotchas-worth-remembering)

---

## Service definition & path

```cds
using { energy.inspections as db } from '../db/schema';

service InspectionService { ... }                       // path → /odata/v4/inspection

service AdminService @(path: '/admin') { ... }          // override the path

service PublicService @(requires: 'authenticated-user') { ... }   // service-wide auth
```

Path defaults from the service name; `@path` overrides. Annotations on the `service` apply to everything inside it.

---

## The exposure spectrum (the core variations)

This is the heart of service modeling — how you turn a DB entity into a service entity. Two families: **`projection`** (simple, single source) and **`as select from`** (full query power). Pick the lightest one that does the job.

### 1:1 passthrough

```cds
entity Inspections as projection on db.Inspections;
```

Everything, unchanged. The default. Fine until you need to hide, rename, or compute something.

### Pick specific columns

```cds
entity Inspections as projection on db.Inspections {
  ID,
  assetID,
  status,
  performedAt
};
```

Whitelist. Anything not listed isn't exposed — the safe way to keep internal columns private.

### Exclude columns

```cds
entity Inspections as projection on db.Inspections excluding { internalNotes };
```

Blacklist. Expose-all-but-these. Convenient, but riskier than a whitelist — a new sensitive column added to the DB later leaks unless you remember to add it here.

### Rename & reach through associations

```cds
entity Inspections as projection on db.Inspections {
  *,                                  // all original fields, plus…
  assetID         as equipment,       // rename
  inspector.email as inspectorEmail   // path expression through an association
};
```

`*` keeps everything and lets you add/override. Path expressions (`assoc.field`) flatten related data into the facade without the client navigating.

### Calculated columns

```cds
entity Inspections as projection on db.Inspections {
  *,
  case when status = 'submitted' then true else false end as isLocked : Boolean
};
```

Derive fields at read time. Keep heavy logic in handlers, but simple derivations belong here.

### Filtered exposure

```cds
entity OpenInspections as projection on db.Inspections where status <> 'actioned';
```

Same table, a pre-filtered view. Great for role- or status-scoped shopfronts off one entity.

### Full query — `as select from`

```cds
entity InspectionStats as select from db.Inspections {
  inspector,
  count(*)            as total      : Integer,
  sum(case when status='submitted' then 1 else 0 end) as submitted : Integer
} group by inspector;
```

Use `as select from` when you need **joins, aggregation, group by, having, unions, order by** — anything projection can't do. Joins:

```cds
entity Enriched as select from db.Inspections as i
  join db.Assets as a on a.ID = i.assetID
{ i.ID, i.status, a.location, a.criticality };
```

**projection vs select from, decided:** projection for "the same entity, trimmed/renamed/filtered." `as select from` the moment a join, an aggregate, or a union enters the picture. Don't reach for `select from` when a projection would do — it's heavier and read-only by default.

### Parameterized entities

```cds
entity InspectionsByAsset(asset : String) as select from db.Inspections
  where assetID = :asset;
```

Called as `InspectionsByAsset(asset='PUMP-001')/Set`. A view that takes an argument — useful for mandatory, indexed filters you don't want clients to forget.

---

## Associations in a service — redirection

When you project an entity, its associations still point at the **DB** entity by default, not your exposed one. Fix that so navigation stays inside the service:

```cds
service S {
  entity Inspections as projection on db.Inspections;
  entity Items as projection on db.InspectionItems {
    *,
    parent : redirected to Inspections      // point at the exposed Inspections, not db.Inspections
  };
}
```

CAP **auto-redirects** when both ends are exposed and unambiguous; you only write `redirected to` when there's ambiguity or you're reshaping. Symptom of forgetting: navigation/`$expand` returns DB-shaped data or errors.

---

## Access shaping

```cds
@readonly   entity AuditLog   as projection on db.AuditLog;    // GET only — no writes
@insertonly entity Events      as projection on db.Events;      // POST only — write, never read back
```

Element-level too:

```cds
entity Inspections as projection on db.Inspections {
  *,
  createdAt @readonly,
  ID        @Core.Immutable      // settable on create, never updatable
};
```

Coarse intent (`@readonly`/`@insertonly`) at entity level; precise field rules at element level.

---

## Draft handling (Fiori)

```cds
@odata.draft.enabled
entity Inspections as projection on db.Inspections;
```

Turns on Fiori's draft mechanism — save-as-draft, resume editing, activate. **Required** for editable Fiori Object Pages; a field inspector half-filling a form and coming back to it needs this. Skip it for read-only or API-only entities.

---

## Actions & functions

The verbs of your service — operations beyond CRUD. Two axes:

- **Function** = read-only, no side effects, GET, args in URL. **Action** = has side effects, POST.
- **Bound** = attached to an entity, operates on an instance (`/Inspections(ID)/submit`). **Unbound** = service-level, standalone.

```cds
service InspectionService {
  entity Inspections as projection on db.Inspections actions {
    action submit() returns Inspections;          // bound action on an instance
    action reopen(reason : String);               // bound, no return
    function failedCount() returns Integer;        // bound function
  };

  action bulkClose(ids : many UUID) returns { closed : Integer; };   // unbound action
  function openTotal() returns Integer;                              // unbound function
}
```

Return types: scalar, entity, structured (`returns { ok : Boolean; }`), array (`returns many X`), or nothing. Your "on submit → raise ECC notification" logic hangs off the bound `submit` action's handler — the action is the clean place for that side effect, not an `UPDATE` hook.

**action vs function, decided:** changes state or calls out (ECC write, email, status flip) → **action**. Pure calculation/lookup with no effect → **function**.

---

## Events (async / messaging)

```cds
service InspectionService {
  event InspectionFailed {
    inspectionID : UUID;
    assetID      : String;
    failedItems  : Integer;
  }
}
```

Declares an event the service can `emit` (handler: `this.emit('InspectionFailed', {...})`). This is your hook into Event Mesh / Advanced Event Mesh for async, decoupled integration — the right pattern when a downstream system should react without your app blocking on it. Pairs with the async discipline that matters under your energy-trading volume spikes.

---

## Authorization

Two tools, different grain:

```cds
// coarse — whole service or entity, role gate
service AdminService @(requires: 'InspectionAdmin') { ... }

// fine — per-operation, per-role, optionally per-instance
entity Inspections @(restrict: [
  { grant: 'READ',                   to: 'Inspector' },
  { grant: ['CREATE','UPDATE'],      to: 'Inspector' },
  { grant: '*',                      to: 'InspectionAdmin' },
  { grant: 'READ', to: 'RegionLead', where: 'region = $user.region' }   // instance-level
]) as projection on db.Inspections;
```

- `@requires` → simple "must have this role." Service- or entity-level gate.
- `@restrict` → array of grants: which `grant` (CRUD verbs or `*`), `to` which role, optionally `where` for **instance-based** filtering using `$user` / `$user.<attribute>`.
- Bound actions take their own `@requires`/`@restrict`.

`$user.region` etc. come from the token's attributes (XSUAA/IAS) — this is where instance-level authorization lives, and it's the difference between "can read inspections" and "can read inspections *in their own region*."

**@requires vs @restrict, decided:** binary role gate → `@requires`. Different rights per verb, or filtering rows by who's asking → `@restrict`.

---

## Protocols

```cds
@protocol: 'odata'                  entity A as projection on db.A;   // default, v4
@protocol: 'rest'                   entity B as projection on db.B;   // plain REST/JSON
@protocol: 'graphql'               entity C as projection on db.C;
@protocol: ['odata', 'rest']        entity D as projection on db.D;   // both
@cds.api.ignore                     entity E as projection on db.E;   // exposed in model, not as API
```

OData v4 is the default and what Fiori/UI5 expect. Drop to `rest` for a simpler JSON contract when a non-SAP consumer doesn't want OData ceremony; `graphql` is available but verify maturity for your version.

---

## Service as a facade over a remote (your ECC case)

```cds
using { ECC_PM } from './external/ECC_PM';     // imported via `cds import` from EDMX

service GatewayService {
  // reshape ECC's released API into your own clean contract
  entity Notifications as projection on ECC_PM.NotificationCollection {
    NotificationNo as id,
    Equipment      as assetID,
    ShortText      as title
  };
}
```

You can project a **remote** service exactly like a DB one — rename ECC's fields into your domain language, hide the noise, expose only what your app needs. This is the clean-core boundary expressed in the service layer: ECC's API stays as-is, your facade gives your app a stable, sane contract that survives ECC field churn.

---

## Gotchas worth remembering

- **Forgotten redirection** → associations in a projection silently point at the DB entity; navigation returns wrong-shaped data. Expose both ends, or write `redirected to`.
- **`as select from` is read-only by default** — if clients need to write, you usually want `projection` (writable) or custom handlers, not a `select`-based view.
- **`excluding` leaks on schema growth** — a new sensitive DB column appears in the service unless you update the exclude list. Whitelist (`{ … }`) is safer for anything sensitive.
- **No draft, no editable Fiori Object Page** — `@odata.draft.enabled` is mandatory for the save-and-resume edit flow; its absence is a common "why can't I edit" surprise.
- **Side effects in the wrong place** — put ECC calls / status changes behind a named **action** and its handler, not buried in a generic `UPDATE` hook; it's clearer, separately authorizable, and testable.
- **`@requires` where you needed `@restrict`** — a flat role gate can't do per-verb or per-row rules; reach for `@restrict` the moment rights differ by operation or by user attribute.
- **Over-exposing the DB** — `as projection on db.X` with no column list dumps your whole table into the API. Default to a column whitelist on anything that isn't trivially public.

---

*CAP/CDS evolves; for the authoritative grammar (projections, actions, `@restrict`, protocols) check `cap.cloud.sap` against your cds version before relying on edge behaviour.*
