# CAP Event Handler Syntax Reference

The implementation layer: the `srv/*.js` where you hook into CAP's request lifecycle and add behaviour. Fourth in the series after schema, service, and UI annotations. Assumes you know JavaScript — this is only the **CAP-specific API and patterns** layered on top.

The model to hold: every request flows through a pipeline — **before → on → after**. CAP supplies a generic `on` handler for CRUD (it talks to the DB for you). Your job is to hook *into* that pipeline: validate on the way in (`before`), replace or extend the core behaviour (`on`), enrich on the way out (`after`). Think of it as middleware around each operation.

## Table of contents

1. [Two implementation styles](#two-implementation-styles)
2. [The three phases](#the-three-phases)
3. [What you can register for](#what-you-can-register-for)
4. [The req object (your toolbox)](#the-req-object-your-toolbox)
5. [Messages & errors — reject vs error](#messages--errors--reject-vs-error)
6. [Querying — cds.ql](#querying--cdsql)
7. [Calling other services](#calling-other-services)
8. [Implementing actions & functions](#implementing-actions--functions)
9. [Events & messaging (async, decoupled)](#events--messaging-async-decoupled)
10. [Transactions & timing (the ECC trap)](#transactions--timing-the-ecc-trap)
11. [Wildcard & shared registration](#wildcard--shared-registration)
12. [Lifecycle / bootstrap](#lifecycle--bootstrap)
13. [TypeScript note](#typescript-note)
14. [Gotchas](#gotchas)
---

## Two implementation styles

**Class style** (preferred for anything non-trivial — clean structure, lifecycle control):

```js
const cds = require('@sap/cds')

module.exports = class InspectionService extends cds.ApplicationService {
  async init() {
    const { Inspections, InspectionItems } = this.entities

    this.before('CREATE', Inspections, req => this._validate(req))
    this.on('submit', Inspections, req => this._submit(req))
    this.after('READ', Inspections, rows => this._enrich(rows))

    await super.init()        // ALWAYS call super.init() last
  }

  _validate(req) { /* … */ }
  async _submit(req) { /* … */ }
  _enrich(rows) { /* … */ }
}
```

**Functional style** (fine for small services):

```js
const cds = require('@sap/cds')
module.exports = function () {
  this.before('CREATE', 'Inspections', req => { /* … */ })
  this.on('submit', 'Inspections', async req => { /* … */ })
}
```

Both auto-wire to the `.cds` of the same base name. Entity references: a string (`'Inspections'`) or the typed handle from `this.entities` (better — fails fast on typos).

---

## The three phases

| Phase | Signature | Purpose | If it throws |
|---|---|---|---|
| `before` | `(req)` | validate, mutate input, authorize | request aborts |
| `on` | `(req, next)` | *do* the work; for CRUD, replaces the generic handler unless you call `next()` | request fails |
| `after` | `(data, req)` | post-process the **result** (`data` = rows/row) | request fails |

```js
this.before('CREATE', Inspections, req => {
  req.data.status ??= 'draft'                    // mutate input before it's written
})

this.on('READ', Inspections, async (req, next) => {
  const rows = await next()                      // run the default DB read…
  return rows                                    // …then optionally post-process
})

this.after('READ', Inspections, (rows) => {
  for (const r of rows) r.isLocked = r.status === 'submitted'   // enrich each row
})
```

**Key distinction — `on` for CRUD vs actions:**
- For **CRUD**, the generic `on` already exists (CAP reads/writes the DB). Registering your own `on` *overrides* it — call `next()` to keep the default behaviour and wrap it.
- For **actions/functions**, there is no default — your `on` *is* the implementation.

`before`/`after` *add* to the pipeline (multiple allowed, run in order). `on` is "the" handler — use `before`/`after` for cross-cutting logic, `on` only when you're implementing or replacing the core operation.

---

## What you can register for

```js
// CRUD
this.before('CREATE', Inspections, fn)
this.after('READ',  Inspections, fn)

// multiple events at once
this.before(['CREATE','UPDATE'], Inspections, fn)

// wildcard — every event on the service
this.before('*', req => { /* logging, tenant checks */ })

// wildcard entity
this.after('READ', '*', fn)

// bound action / function (defined in the service .cds)
this.on('submit', Inspections, fn)
this.on('failedCount', Inspections, fn)

// unbound action / function
this.on('bulkClose', fn)

// draft events (when @odata.draft.enabled)
this.before('NEW',    Inspections.drafts, fn)   // new draft created
this.before('PATCH',  Inspections.drafts, fn)   // field edited in draft
this.on('SAVE',       Inspections, fn)          // draft activated
this.on('CANCEL',     Inspections.drafts, fn)   // draft discarded

// custom events (messaging) — see Events section
this.on('InspectionFailed', msg => fn)

// lifecycle
this.on('served', () => { /* this service is ready */ })
```

Draft note: `.drafts` targets the draft (in-progress) instances; the bare entity targets active ones. Validating a half-filled inspection mid-edit means hooking the draft `PATCH`/`SAVE`, not `CREATE`.

---

## The `req` object (your toolbox)

```js
req.data            // the payload (CREATE/UPDATE body, action params)
req.params          // key(s) from the URL — req.params[0] = {ID:…} for a bound op
req.query           // the CQN query object (inspect/modify the incoming query)
req.target          // the targeted entity definition
req.event           // 'CREATE' | 'READ' | 'submit' | …
req.user            // req.user.id, req.user.is('Admin'), req.user.attr.region
req.tenant          // current tenant (multitenancy)
req.headers / req.http   // raw HTTP access
req.locale          // for i18n

req.reject(code, msg, target)   // ABORT now — throws
req.error(code, msg, target)    // COLLECT error, keep going — returned at end
req.warn(...) / req.info(...) / req.notify(...)   // non-fatal messages to client
req.reply(result)               // set the response payload explicitly
```

`req.user.attr.region` is where instance-level auth lives — the same `$user.region` you referenced in `@restrict` is readable here for programmatic checks.

---

## Messages & errors — reject vs error

```js
this.before('CREATE', Inspections, req => {
  // reject: hard stop, aborts immediately
  if (!req.data.assetID) return req.reject(400, 'assetID is required', 'assetID')

  // error: collect MANY validation problems, return them together
  if (!req.data.inspector) req.error(400, 'inspector missing', 'inspector')
  if (req.data.performedAt > Date.now()) req.error(400, 'date in future', 'performedAt')
})
```

**Decided:** one fatal condition → `reject` (stop now). Validating a form where you want *all* the field errors at once → `error` (collect, return together). `target` ties the message to a field so Fiori highlights it.

---

## Querying — `cds.ql`

The fluent query API. These run against the bound DB (or any connected service):

```js
const { Inspections, InspectionItems } = this.entities

// SELECT
const all   = await SELECT.from(Inspections)
const one   = await SELECT.one.from(Inspections).where({ ID })
const some  = await SELECT.from(Inspections)
                .columns('ID','status')
                .where({ status: 'submitted' })
                .orderBy('performedAt desc')
                .limit(50)

// existence / aggregation
const failed = await SELECT.from(InspectionItems).where({ parent_ID: ID, result: 'fail' })

// INSERT
await INSERT.into(Inspections).entries({ assetID: 'PUMP-001', status: 'draft' })

// UPDATE
await UPDATE(Inspections).set({ status: 'actioned' }).where({ ID })

// DELETE
await DELETE.from(Inspections).where({ ID })

// UPSERT
await UPSERT.into(Inspections).entries({ ID, status: 'submitted' })
```

`await` runs it. `SELECT.one` returns a single object (or null) instead of an array. The `where({…})` object form is the everyday case; a tagged-template form exists for complex predicates.

---

## Calling other services

```js
// the database
const db = await cds.connect.to('db')
await db.run(SELECT.from(Inspections))

// a REMOTE service (your ECC, imported via cds import + declared in cds.requires)
const ecc = await cds.connect.to('ECC_PM')

// typed CRUD against the remote
await ecc.create('NotificationCollection', {
  Equipment: assetID,
  ShortText: `Inspection ${ID}: failed items`
})

// or a raw request when you need control
await ecc.send({ method: 'POST', path: 'NotificationCollection', data: {…} })
const note = await ecc.get(`NotificationCollection('${notifNo}')`)
```

Same query/CRUD API works whether the target is your DB or ECC over a destination — that uniformity is CAP's whole point. The destination + Cloud Connector wiring lives in config; your code just calls `ecc.create(...)`.

---

## Implementing actions & functions

```js
// bound action: instance key comes from req.params
this.on('submit', Inspections, async req => {
  const { ID } = req.params[0]
  await UPDATE(Inspections).set({ status: 'submitted' }).where({ ID })
  await this._raiseEccIfFailed(ID, req)
  return SELECT.one.from(Inspections).where({ ID })   // return the updated entity
})

// unbound action: params in req.data
this.on('bulkClose', async req => {
  const { ids } = req.data
  await UPDATE(Inspections).set({ status: 'actioned' }).where({ ID: ids })
  return { closed: ids.length }
})
```

Calling actions programmatically from elsewhere:

```js
await this.send('submit', { ID })          // generic
await srv.submit(ID)                        // if the service exposes it as a method
```

---

## Events & messaging (async, decoupled)

```js
// PRODUCE — emit an event (declared as `event` in the service .cds)
this.after('UPDATE', Inspections, async (data, req) => {
  if (data.status === 'submitted')
    await this.emit('InspectionFailed', { inspectionID: data.ID, assetID: data.assetID })
})

// to an external broker (Event Mesh / AEM) via the messaging service
const messaging = await cds.connect.to('messaging')
await messaging.emit('inspection/failed', payload)

// CONSUME — subscribe
this.on('InspectionFailed', async msg => {
  const { inspectionID } = msg.data
  // react asynchronously, non-blocking
})
messaging.on('inspection/failed', async msg => { /* … */ })
```

`emit` is fire-and-forget (async); `send` is request/response (sync). Use events when a downstream system should react *without your request waiting on it* — the right call under trading-volume spikes where blocking on every downstream is how you fall over.

**Decided:** need a result back, now → `send`. Notify something that'll handle it on its own time → `emit`.

---

## Transactions & timing (the ECC trap)

CAP wraps each request in a transaction automatically. The trap: if you call ECC inside the handler and the request then rolls back, you've created an ECC notification for an inspection that didn't save.

```js
// WRONG-ish: ECC call inside the tx — rolls back leave ECC out of sync
this.on('submit', Inspections, async req => {
  await UPDATE(...)              // in the request tx
  await ecc.create(...)          // if a later step fails, this isn't undone
})

// RIGHT: fire external side-effects only AFTER the tx commits
this.on('submit', Inspections, async req => {
  await UPDATE(...)
  req.on('succeeded', async () => {
    await ecc.create('NotificationCollection', {…})   // runs post-commit
  })
})
```

`req.on('succeeded' | 'failed' | 'done', …)` hooks the transaction outcome. External, non-rollback-able side effects (ECC writes, emails, events) belong on `succeeded`. For genuinely detached background work:

```js
cds.spawn({ after: 0 }, async tx => { /* runs in its own tx, off the request path */ })
```

This timing point is the single most important thing in this doc for your ECC integration — get it wrong and you get phantom notifications.

---

## Wildcard & shared registration

```js
this.before('*', req => log(req.event, req.user.id))     // every operation
this.before(['CREATE','UPDATE'], '*', stampTenant)        // all writes, all entities
this.prepend(() => {                                       // register BEFORE framework/base handlers
  this.on('READ', Inspections, customReadOverride)
})
```

`prepend` puts your handler ahead of inherited/base ones — for overriding behaviour from an extended service.

---

## Lifecycle / bootstrap

```js
// srv/server.js  (optional custom bootstrap)
const cds = require('@sap/cds')
cds.on('bootstrap', app => { app.use('/health', (_, res) => res.send('ok')) })  // raw express
cds.on('served',    () => { /* all services wired — start schedulers, warm caches */ })
module.exports = cds.server
```

`bootstrap` = the Express app (add middleware, custom routes). `served` = every service is ready (good place for one-time setup). Per-service `this.on('served', …)` also works.

---

## TypeScript note

CAP supports TS. Run `cds-typer` to generate typed entity definitions from your CDS, then handlers get autocompletion and compile-time checks on `req.data` shapes. Worth it once a service grows past trivial; the handler patterns above are identical, just typed.

---

## Gotchas

- **Forgetting `next()` in a custom CRUD `on`** → you silently replaced the DB read/write and nothing persists. Either call `next()` to keep default behaviour, or fully implement it.
- **External calls inside the request tx** → phantom data on rollback. Move ECC/email/event side-effects to `req.on('succeeded')`.
- **`this` binding in class handlers** → passing `this._method` as a bare reference loses `this`. Use arrow wrappers (`req => this._method(req)`) as shown.
- **`after('READ')` data shape** → it's the array for collection reads; iterate it. Don't assume a single object.
- **`reject` vs `error` mix-up** → `reject` aborts on the first problem (user fixes one, resubmits, hits the next); `error` collects them all. For form validation, prefer `error`.
- **Draft vs active target** → validations on `@odata.draft.enabled` entities often need `.drafts` (mid-edit) handlers, not `CREATE`/`UPDATE`.
- **`super.init()` not called / not last** in class style → handlers may not register correctly. Call it, call it last.
- **Heavy work blocking the response** → use `cds.spawn` or the `succeeded` hook for anything the caller doesn't need to wait on.

---

*CAP's Node.js runtime evolves; for the authoritative handler API (events, `req`, `cds.ql`, messaging) check `cap.cloud.sap/docs/node.js` against your `@sap/cds` version — especially draft events and messaging, which have shifted across releases.*
