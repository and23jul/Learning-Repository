
# RAP for the Returning ABAP Developer
### What the RESTful Application Programming Model is, how it relates to ABAP, and when to use it

If the modern-syntax shift was "expressions replace statements," RAP is a bigger conceptual jump — but in the *opposite* direction from what people fear. **RAP is not a new language and it does not replace ABAP.** It's a framework *built in* ABAP that standardizes how you build transactional OData services. Your ABAP doesn't disappear; it moves into specific, named slots, and the framework takes over the boilerplate you used to write by hand.

Mental model: classic ABAP was you machining a car from raw metal — Dynpro screens, PAI/PBO, manual `ENQUEUE`/`DEQUEUE`, hand-rolled transaction logic, your own buffer. RAP hands you the **chassis, drivetrain, and safety systems already assembled** (locking, draft, ETag/optimistic concurrency, transactional sequencing, OData exposure) and asks you to bolt on only the business-specific parts: validations, determinations, actions, custom save. You write less ABAP, but the ABAP you write is pure business logic.

## Table of contents

1. [RAP vs classic SAP, at a glance](#1-rap-vs-classic-sap-at-a-glance)
2. [Where RAP intersects ABAP, and where it differs](#2-where-rap-intersects-abap-and-where-it-differs)
3. [The artifact stack — what a RAP app is made of](#3-the-artifact-stack--what-a-rap-app-is-made-of)
4. [Managed vs Unmanaged — the decision that defines your project](#4-managed-vs-unmanaged--the-decision-that-defines-your-project)
5. [The business-logic building blocks (where your ABAP lives)](#5-the-business-logic-building-blocks-where-your-abap-lives)
6. [EML — calling RAP business objects from ABAP](#6-eml--calling-rap-business-objects-from-abap)
7. [Use cases — when to reach for RAP (and when not)](#7-use-cases--when-to-reach-for-rap-and-when-not)
8. [Where it runs, and the clean-core tie-in](#8-where-it-runs-and-the-clean-core-tie-in)
9. [RAP vs CAP — choosing the right model](#9-rap-vs-cap--choosing-the-right-model)
10. [Minimal anatomy — the shape of the smallest managed BO](#10-minimal-anatomy--the-shape-of-the-smallest-managed-bo)
11. [TL;DR for someone coming back](#tldr-for-someone-coming-back)

---

## 1. RAP vs classic SAP, at a glance

| Classic stack | RAP equivalent |
|---|---|
| Dynpro / module pool / report UI | Fiori Elements (or custom UI5) on OData |
| Hand-coded `ENQUEUE`/`DEQUEUE` locking | Framework-managed locking (declared, not coded) |
| Manual transaction control, your own buffer | Framework transactional buffer + save sequence |
| BOPF (the predecessor framework) | RAP business objects (BOPF is superseded) |
| `CALL TRANSACTION` / BAPI for reuse | EML (`MODIFY ENTITIES` / `READ ENTITIES`) |
| Gateway service builder (SEGW), hand-mapped | CDS + Service Definition + Service Binding |
| Authorization checks scattered in code | Declarative authorization + feature control |
| Custom "is it locked / dirty" logic | Built-in draft handling |

The pattern: **what was procedural and repetitive is now declarative; what was business-specific stays as ABAP.**

---

## 2. Where RAP intersects ABAP, and where it differs

**Intersects (still ABAP, using the modern syntax from the prior doc):**
- The **behavior implementation** — global classes ("behavior pools") where you code validations, determinations, actions, and custom saves. This is normal ABAP: `LOOP`, `READ ENTITIES`, `VALUE`, `COND`, the lot.
- **EML** — Entity Manipulation Language — new ABAP statements to drive RAP business objects from any ABAP code (more in §6).
- AMDP / CDS table functions when you need DB-pushdown logic.

**Differs (declarative artifacts, not procedural ABAP):**
- The **data model** is CDS, not `SELECT` in a report.
- The **behavior** (what CRUD is allowed, locking, draft, field control, actions) is declared in **Behavior Definition Language (BDL)** — a DSL, not ABAP.
- The **exposure** (which entities, which protocol) is Service Definition + Service Binding — config, not code.
- The **runtime** (transaction sequence, buffer, concurrency) is the framework's, not yours.

So: you stop writing the *scaffolding* (screens, locks, transaction flow, OData mapping) and keep writing the *logic*. The skill that transfers is your business knowledge and your ABAP; the skill you drop is the plumbing.

---

## 3. The artifact stack — what a RAP app is made of

A RAP business object is a **stack of layers**, each a separate artifact. Bottom to top:

```
┌─────────────────────────────────────────────┐
│  Service Binding   → OData V2/V4, UI or API  │  ← how it's exposed (published endpoint)
├─────────────────────────────────────────────┤
│  Service Definition → which entities exposed │  ← scope of the service
├─────────────────────────────────────────────┤
│  Projection CDS view + Projection Behavior   │  ← the consumption-facing slice + UI annotations
├─────────────────────────────────────────────┤
│  Behavior Definition (BDL)  ←→  Behavior     │  ← rules (declarative) + implementation (ABAP)
│                                  Pool (ABAP) │
├─────────────────────────────────────────────┤
│  Interface CDS view(s)  (root + children)    │  ← the data model / business object structure
├─────────────────────────────────────────────┤
│  Database table(s)                           │  ← persistence
└─────────────────────────────────────────────┘
```

What each does:

- **Database table** — your persistence (or an existing legacy table, in the unmanaged case).
- **Interface CDS view(s)** — `define root view entity` for the header, child entities for items, joined by `composition`/`association`. This is your business object's shape.
- **Behavior Definition (BDL)** — declares, per entity: which operations exist (`create`, `update`, `delete`), the lock master, draft on/off, ETag field, field characteristics (`readonly`, `mandatory`), validations, determinations, actions, authorization.
- **Behavior Pool (ABAP class)** — implements only the things BDL declared as needing code (each validation, determination, action, and — in unmanaged — the CRUD itself).
- **Projection views + projection behavior** — a consumption-facing slice of the BO (rename/hide fields, add UI annotations) so the same core BO can feed different services.
- **Service Definition** — `define service` listing the exposed entities.
- **Service Binding** — binds the service to a protocol (OData **V2** or **V4**) and a scenario (**UI** for Fiori, **Web API** for integration), then you *Publish* to get a live endpoint.

The win: the same BO can be exposed as a Fiori app *and* a public API just by adding a second binding — no logic duplicated.

---

## 4. Managed vs Unmanaged — the decision that defines your project

This is the single most important RAP concept and the one that maps directly to your use case.

**Managed** — the framework owns persistence. You define a CDS model and a behavior definition, and RAP **auto-generates** the create/update/delete, the transactional buffer, and the save to the database. You write *zero* CRUD code — only validations, determinations, and actions.
→ Analogy: a **furnished apartment**. Plumbing, wiring, appliances all in place; you just decorate (business logic).
→ Use for: **greenfield**, new tables, new Fiori transactional apps.

**Unmanaged** — *you* own persistence. RAP gives you the OData/transactional shell, but you implement every interaction yourself: `FOR READ`, `FOR MODIFY` (create/update/delete), `FOR LOCK`, mapping to your existing tables / function modules / legacy update logic.
→ Analogy: an **old building with existing wiring and tenants** (legacy persistence and logic); RAP is the new reception desk and public API you put in front of it.
→ Use for: **wrapping existing custom logic** — you have a working BAPI or update-task module and want a modern Fiori UI / OData API without rewriting the core.

Two hybrids worth knowing exist (check current docs for exact support in your release):
- **Managed with unmanaged save** — framework runs the interaction buffer, but you take over the final save (e.g. to call a legacy update module).
- **Managed with additional save / `with full data`** — framework saves, but you hook in extra persistence on top.

> The fork that flips it: **new persistence → managed. Existing persistence/logic you must reuse → unmanaged (or managed-with-unmanaged-save).** Don't go unmanaged on a greenfield app to "have control" — you'll hand-write everything RAP would have given you free.

---

## 5. The business-logic building blocks (where your ABAP lives)

Inside the behavior pool, these are the slots — each declared in BDL, implemented in ABAP:

- **Validation** (`validation ... on save`) — check data before save; flag failures. Replaces scattered `IF ... MESSAGE` checks. Runs at the framework's controlled point.
- **Determination** (`determination ... on modify / on save`) — auto-derive/default values when data changes (e.g. set status, compute totals). Replaces the "fill this field in PAI" reflex.
- **Action** (`action ...`) — a custom operation beyond CRUD, exposed as a button/endpoint (e.g. "Approve", "Release order"). Returns the changed instance. Can be `static`, `factory`, or instance-bound.
- **Feature control** (`features : instance`) — dynamically make fields/operations enabled/readonly/mandatory per instance state (e.g. lock the amount once approved). Replaces dynamic screen-field attribute juggling.
- **Draft handling** — declare `draft;` and the framework manages draft tables and the standard draft actions (Edit/Activate/Discard/Resume/Prepare). Replaces any home-grown "work-in-progress" persistence.
- **Authorization** (`authorization master/dependent`) — declarative auth checks at defined granularity.

The pattern is always the same: **BDL says *that* it happens and *when*; the ABAP class says *what* happens.**

---

## 6. EML — calling RAP business objects from ABAP

RAP BOs aren't only reachable over OData. From any ABAP (a report, a job, another BO's action), you drive them with **Entity Manipulation Language** — new statements that are the modern replacement for `CALL FUNCTION 'BAPI_...'` + `BAPI_TRANSACTION_COMMIT`:

```abap
" read
READ ENTITIES OF zi_travel
  ENTITY travel
  FIELDS ( overall_status ) WITH VALUE #( ( travel_id = '42' ) )
  RESULT DATA(lt_travel).

" modify
MODIFY ENTITIES OF zi_travel
  ENTITY travel
  UPDATE FIELDS ( description ) WITH VALUE #( ( travel_id = '42'
                                               description = 'Updated' ) )
  FAILED DATA(ls_failed)
  REPORTED DATA(ls_reported).

" commit
COMMIT ENTITIES.
```

This is the clean-core, framework-aware way to do programmatic transactions. It respects the BO's validations, determinations, locking, and draft — unlike a raw `UPDATE` on the table, which bypasses all of it.

---

## 7. Use cases — when to reach for RAP (and when not)

**Use managed RAP when:**
- Building a **new transactional Fiori app** on S/4HANA or BTP — new tables, standard CRUD + a few actions. This is the sweet spot; you'll write almost no plumbing.
- You want **draft-enabled** editing (save-as-you-go, multi-session) without building it yourself.

**Use unmanaged RAP (or managed-with-unmanaged-save) when:**
- You're **modernizing legacy custom logic** — a working update module / BAPI / complex posting logic you can't or won't rewrite — but want a Fiori UI and/or OData API in front of it. Wrap, don't rewrite.

**Use RAP with a Web API binding (no UI) when:**
- You need to **expose a transactional OData API** for integration (Integration Suite, external consumers, side-by-side BTP apps). Same BO, just bind it as Web API instead of (or in addition to) UI.

**Don't use RAP — use something else — when:**
- **Read-only / analytical** requirements → plain CDS (analytical/consumption views) + Fiori Elements analytical, or Datasphere/SAC. RAP's transactional machinery is dead weight for pure reporting.
- **Mass/batch processing** with no UI and no per-row business-object semantics → straight ABAP SQL / AMDP is simpler and faster than driving a BO row-by-row.
- **Non-OData protocols** (SOAP, flat file, IDoc-style) → that's integration tooling, not RAP.
- **Pure event/messaging** flows → events (RAP can *raise* business events, but the messaging backbone is Event Mesh / AEM, not RAP itself).

> Rule of thumb: RAP earns its complexity when there's a **transactional business object with a UI or API and real CRUD semantics**. The flatter and more read-only the requirement, the less RAP buys you.

---

## 8. Where it runs, and the clean-core tie-in

RAP is **the** clean-core development model, and it runs in two places with the *same* programming model:

- **On-stack (ABAP Platform / S/4HANA)** — develop in ADT (Eclipse), deploy into the S/4 system. The standard path for in-app extensions and custom apps that live close to the data.
- **BTP ABAP environment (Steampunk)** — the side-by-side path: RAP apps running on BTP, reaching S/4 data via released APIs / OData. Use when you want decoupling from the digital core, independent lifecycle, or to keep the core untouched.

Either way the rules from the clean-core world apply: **CDS + RAP, released (C1) public APIs only, no core modification, ADT not SE80.** RAP isn't a "nice to have" modern option here — on S/4 cloud and BTP it's the sanctioned way to build transactional custom apps, the same way the modern syntax is the sanctioned dialect.

---

## 9. RAP vs CAP — choosing the right model

Both are SAP-blessed, clean-core frameworks. The choice is **not** about which is "better" — it's about **where your data and logic live** and **what your team codes in**. Get those two right and the decision is usually obvious.

The one-line distinction: **RAP is the ABAP-native model for transactional apps on the ABAP stack; CAP is the polyglot (Node.js/Java) model for standalone cloud apps on Cloud Foundry/Kyma.**

### Decision tree

```
1. Is the app primarily a TRANSACTIONAL extension of S/4 / ABAP data
   (CRUD + draft + locking on business objects that live in the ABAP world)?
        YES ─────────────────────────────────────────────► RAP   (stop)
        NO  → go to 2

2. Is it a STANDALONE product with its own persistence —
   especially a multitenant SaaS you'll sell to many customers?
        YES ─────────────────────────────────────────────► CAP   (stop)
        NO  → go to 3

3. What does the team actually code in?
        ABAP ────────────────────────────► RAP  (on BTP ABAP environment)
        Node.js / Java ──────────────────► CAP
        Mixed / unsure → go to 4

4. Tie-breaker — what dominates the workload?
        Native S/4 data + ABAP logic, lowest latency to the core ─► RAP
        Non-SAP data, npm/Maven ecosystem, polyglot freedom ──────► CAP
```

### Decision matrix

| Dimension | RAP | CAP |
|---|---|---|
| Language | ABAP | Node.js or Java |
| Runtime | ABAP Platform (on-stack) **or** BTP ABAP env (Steampunk) | Cloud Foundry **or** Kyma |
| Native S/4 / ABAP data access | Yes — same stack, same transaction | No — remote via OData/RFC |
| Own persistence | ABAP-managed DB | HANA Cloud, modeled in CAP CDS |
| Transactional BO (draft, lock, ETag) | First-class, purpose-built | Draft supported, but not on ABAP data natively |
| Multitenant SaaS | Limited | First-class (MTX) |
| Non-SAP / polyglot integration | ABAP-world | Strong, open ecosystem |
| Team fit | ABAP developers | JS / Java full-stack |
| Clean core | Yes (the ABAP model) | Yes (the side-by-side model) |
| Cost profile | ABAP runtime — heavier CPEA burn | Node/Java runtime — generally lighter |
| Best for | Extending S/4, ABAP teams, core-proximate logic | Standalone cloud apps, SaaS products, JS/Java teams |

> Note on the shared name: both use "CDS," but **ABAP CDS** (Dictionary views, RAP's data layer) and **CAP CDS** (CDL compiled by the `cds` toolkit) are *different implementations of the same concept* — don't assume an artifact crosses over.

### The recommendation, and what flips it

For your situation — an ABAP team at an energy company extending S/4 — **RAP is the default.** Your data and logic are in the ABAP core, your people write ABAP, and the transactional BO machinery (draft, lock, ETag) is exactly what S/4 extensions need. Reaching for CAP here would mean rebuilding plumbing RAP gives you free and adding a remote hop to data that's sitting right there.

**The flip:** go CAP when you're building a **standalone** cloud product whose data model isn't primarily S/4 — above all a **multitenant SaaS** you sell to many customers, or anything your team would rather (or can only) build in Node/Java with the open ecosystem. This is the realistic trigger for your side-build interest: a product sold per-tenant fits CAP's multitenancy and lighter runtime far better than spinning up ABAP-environment instances.

**It's not always either/or.** A common, legitimate pattern: **RAP exposes the S/4 transactional BO as an OData API, and a CAP (or any BTP) app consumes it** for a broader, customer-facing experience. "RAP for the core BO, CAP for the surrounding standalone app" is a real architecture, not a contradiction.

**Cost flag:** the BTP ABAP environment that hosts side-by-side RAP is a heavier, more CPEA-expensive runtime than CF/Kyma running Node/Java CAP. If cost-per-tenant matters — and for a product you sell, it does — that pushes toward CAP. Verify current numbers in the BTP cost estimator / Discovery Center before you commit.

---

## 10. Minimal anatomy — the shape of the smallest managed BO

Just so the artifacts feel concrete (illustrative, not complete):

```abap
" 1. Root interface CDS view
define root view entity ZI_Travel
  as select from ztravel
{
  key travel_id    as TravelID,
      description   as Description,
      overall_status as OverallStatus,
      @Semantics.systemDate.lastChangedAt: true
      last_changed_at as LastChangedAt   " ETag field
}
```

```abap
" 2. Behavior Definition (BDL) — declarative
managed implementation in class zbp_i_travel unique;

define behavior for ZI_Travel alias Travel
persistent table ztravel
lock master
etag master LastChangedAt
{
  create; update; delete;
  field ( readonly ) TravelID;

  determination setStatus on modify { create; }
  validation    validateDates on save { field ... }
  action approve result [1] $self;
}
```

```abap
" 3. Behavior pool (ABAP) — implement only the declared logic
CLASS lhc_travel DEFINITION INHERITING FROM cl_abap_behavior_handler.
  PRIVATE SECTION.
    METHODS validateDates FOR VALIDATE ON SAVE IMPORTING keys FOR Travel~validateDates.
    METHODS approve       FOR MODIFY            IMPORTING keys FOR ACTION Travel~approve.
    METHODS setStatus     FOR DETERMINE ON MODIFY IMPORTING keys FOR Travel~setStatus.
ENDCLASS.
```

Then a Service Definition exposes `ZI_Travel`, a Service Binding publishes it as OData V4 / UI, and Fiori Elements renders it. The framework supplied the locking, the buffer, the transaction, the ETag concurrency, and the OData — you wrote three behavior methods.

---

## TL;DR for someone coming back

- RAP is **a framework in ABAP**, not a new language and not a replacement for ABAP.
- It **standardizes the plumbing** (locking, draft, transactions, OData) and leaves you the **business logic** (validations, determinations, actions, custom save).
- The artifacts are a **stack**: DB table → CDS → Behavior Definition + Behavior Pool → Service Definition → Service Binding.
- **Managed = greenfield** (framework owns persistence); **Unmanaged = wrap legacy** (you own persistence).
- It supersedes **BOPF** and the **Dynpro + manual-lock + SEGW** way of building transactional apps.
- It's the **clean-core model** for both on-stack S/4 and BTP ABAP environment — not optional there.
- Reach for it when there's a **transactional BO with a UI or API**; skip it for read-only, batch, or non-OData needs.

> Version note: specific RAP features (draft, certain action/EML capabilities, managed-with-unmanaged-save details) depend on release/SP and on whether you're on the on-stack ABAP Platform vs the BTP ABAP environment. Verify exact availability against current SAP docs for your target landscape.
