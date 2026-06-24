# Building a CAP App, Step by Step

A hands-on walkthrough using the **asset inspection app** as the example. Everything through Step 9 runs locally on your laptop — no BTP account, no credit burn. You only touch BTP at deploy time.

The CAP loop you're learning: define a CDS model → `cds watch` gives you a live OData service on in-memory SQLite → build all logic against that → swap in HANA Cloud + security only at the end. Infra last, model and logic first.

**Two tools, two jobs.** Commands that start with `cds`, `npm`, `cf`, `mbt` are **terminal** (PowerShell on Windows is fine). Everything that's CDS or JS *content* is a **file you create in an editor** (VS Code) — you never type those into PowerShell. The setup is two panes side by side: editor for the files, terminal running `cds watch` next to it.

## Table of contents

- [0. Prerequisites (one-time)](#0-prerequisites-one-time)
- [1. Scaffold the project](#1-scaffold-the-project)
- [1b. Set up your workbench (VS Code)](#1b-set-up-your-workbench-vs-code)
- [1c. The files you'll create, in order](#1c-the-files-youll-create-in-order)
- [2. Define the data model](#2-define-the-data-model)
- [3. Define the service](#3-define-the-service)
- [4. Run it — the inner loop](#4-run-it--the-inner-loop)
- [5. Add sample data](#5-add-sample-data)
- [6. Custom logic — event handlers](#6-custom-logic--event-handlers)
- [7. Call ECC as a remote service (the clean-core boundary)](#7-call-ecc-as-a-remote-service-the-clean-core-boundary)
- [8. (Optional) Fiori elements preview](#8-optional-fiori-elements-preview)
- [9. Swap SQLite → HANA Cloud](#9-swap-sqlite--hana-cloud)
- [10. Add security](#10-add-security)
- [11. Deploy to Cloud Foundry](#11-deploy-to-cloud-foundry)
- [What to actually do tonight](#what-to-actually-do-tonight)
---

## 0. Prerequisites (one-time)

You need Node.js (LTS — 20 or 22) and the CAP development kit.

```bash
node --version          # expect v20.x or v22.x
npm install -g @sap/cds-dk
cds --version           # confirms the CLI is on your PATH
```

`@sap/cds-dk` gives you the `cds` command. If `cds` isn't found after install, your npm global bin isn't on PATH — fix that before continuing.

> Version note: cds-dk moves fast. If a command below behaves differently, check `cds help <command>` and current SAP CAP docs (cap.cloud.sap) rather than fighting it.

---

## 1. Scaffold the project

```bash
cds init inspection-app
cd inspection-app
npm install
```

This creates the standard layout:

```
inspection-app/
├── app/          # UI layer (Fiori elements, or your React app points here)
├── db/           # data model + sample data
├── srv/          # service definitions + custom logic
└── package.json  # cds config lives under the "cds" key
```

Three folders, clear separation: `db` = what the data *is*, `srv` = what you *expose* and the logic, `app` = how it *looks*.

---

## 1b. Set up your workbench (VS Code)

Step 1 was the last pure-PowerShell step for a while. From here you're creating and editing files, which is editor work. Set up the two-pane workbench once:

1. From inside the `inspection-app` folder in PowerShell, run `code .` to open VS Code on the project. (If `code` isn't recognized: open VS Code manually → File → Open Folder → pick the `inspection-app` folder. Or reinstall VS Code with the "Add to PATH" option ticked.)

2. Install the **SAP CDS Language Support** extension (publisher: SAP SE) from the Extensions panel (`Ctrl+Shift+X`, search "CDS"). Without it you're editing `.cds` files blind — with it you get highlighting, autocomplete, and inline errors.

3. Open VS Code's integrated terminal (`` Ctrl+` ``) and run `cds watch` **now**, before you create any files. It'll sit waiting on an empty model. As you add each file below, you'll watch it pick them up live — far better for learning than building everything then running it cold.

Now you have: editor pane for files, integrated terminal running `cds watch`, browser on `localhost:4004`. That's the loop.if you are using codespace in github, then your URL should be https://<codespace_name>-4004.app.github.dev/

Why Port 4004 specifically? It's just CAP's default port. You can override it (cds watch --port 4005, or the PORT env var), but there's no reason to — leave it at 4004 and that's the address to point the browser at.

---

## 1c. The files you'll create, in order

`cds init` already made the folders (`db/`, `srv/`, `app/`) and `package.json`. You're *adding files into* those folders. In VS Code's file explorer (left panel), right-click the folder → New File → name it exactly as shown → paste the content from the matching step → save.

| # | Create this file | In folder | Content from | Type |
|---|---|---|---|---|
| 1 | `schema.cds` | `db/` | Step 2 | CDS model |
| 2 | `inspection-service.cds` | `srv/` | Step 3 | CDS service |
| 3 | `energy.inspections-Inspections.csv` | `db/data/` | Step 5 | sample data |
| 4 | `energy.inspections-InspectionItems.csv` | `db/data/` | Step 5 | sample data |
| 5 | `inspection-service.js` | `srv/` | Step 6 | JS logic |
| 6 | `annotations.cds` | `app/inspections/` | Step 8 *(optional UI)* | CDS annotations |

Notes on the folders:
- `db/data/` does **not** exist yet — create the `data` subfolder under `db` first (right-click `db` → New Folder → `data`), then add the two CSVs inside it.
- `app/inspections/` also doesn't exist — only create it if you do the optional Fiori preview in Step 8 (right-click `app` → New Folder → `inspections`).
- **IMPORTANT:** The two `.js`/`.cds` in `srv/` must share the base name (`inspection-service`) — that's how CAP auto-wires the logic file to the service.

Do them top to bottom. After file 1 and file 2, `cds watch` already serves a working OData service — check `localhost:4004` before going further. Files 3–4 give it data, file 5 gives it behaviour, file 6 is the optional free UI.

---

## 2. Define the data model

Create `db/schema.cds`. This is **your** model — it does not live in ECC.

```cds
namespace energy.inspections;

using { cuid, managed } from '@sap/cds/common';

entity Inspections : cuid, managed {
  assetID     : String(20);                  // references ECC equipment, not an FK into ECC
  inspector   : String(100);
  performedAt : Timestamp;
  status      : String(20) default 'draft';  // draft / submitted / actioned
  items       : Composition of many InspectionItems on items.parent = $self;
}

entity InspectionItems : cuid {
  parent    : Association to Inspections;
  checkType : String(50);
  result    : String(10);                    // pass / fail / NA
  note      : String(500);
}
```

What's doing the work here:
- `cuid` adds a UUID `ID` key automatically.
- `managed` adds `createdAt / createdBy / modifiedAt / modifiedBy` — free audit fields.
- `Composition` = parent owns children (delete the inspection, its items go too). `Association` would be a reference without ownership. The composition is what makes Inspection+Items a single deep document.

For more info on CDS Schema Syntax, refer to https://github.com/and23jul/Learning-Repository/blob/main/CAP/02-cds-schema-syntax.md 
---

## 3. Define the service

Create `srv/inspection-service.cds`. This exposes the model as an OData v4 service.

```cds
using { energy.inspections as my } from '../db/schema';

service InspectionService {
  entity Inspections     as projection on my.Inspections;
  entity InspectionItems as projection on my.InspectionItems;
}
```

`projection on` means you're exposing the entity, optionally reshaped (you can rename, hide, or compute fields here later). Keep the service as the place you decide what the outside world sees — don't just blanket-expose everything when the app grows.

---

## 4. Run it — the inner loop

```bash
cds watch
```

Open **http://localhost:4004**. You have a live OData v4 service on an in-memory SQLite DB, with no deploy and no database to install. The index page lists your entities; click `Inspections` to hit the OData endpoint.

Leave this running. Every file save reloads automatically. This is where you'll live while building.

Try a query in the browser:
```
http://localhost:4004/odata/v4/inspection-service/Inspections
```
or if you use GitHub Codespace:
```
https://<codespace_name>-4004.app.github.dev/
```
---

## 5. Add sample data

So you're not staring at empty sets. Create CSVs in `db/data/`, named `<namespace>-<Entity>.csv`:

`db/data/energy.inspections-Inspections.csv`
```csv
ID;assetID;inspector;performedAt;status
11111111-1111-1111-1111-111111111111;PUMP-001;andre;2026-06-22T08:00:00Z;submitted
22222222-2222-2222-2222-222222222222;XFMR-014;laura;2026-06-21T14:30:00Z;draft
```

`db/data/energy.inspections-InspectionItems.csv`
```csv
ID;parent_ID;checkType;result;note
aaaa1111-0000-0000-0000-000000000001;11111111-1111-1111-1111-111111111111;seal;fail;weeping at flange
aaaa1111-0000-0000-0000-000000000002;11111111-1111-1111-1111-111111111111;vibration;pass;
```

Header names = element names. The association column is `parent_ID` (the association name + `_ID`). `cds watch` reloads and the data is queryable immediately.

---

## 6. Custom logic — event handlers

The CDS files gave you CRUD for free. Now add behaviour. Create `srv/inspection-service.js` — CAP auto-wires a `.js` next to the `.cds` of the same name.

```js
const cds = require('@sap/cds')

module.exports = class InspectionService extends cds.ApplicationService {
  init() {
    const { Inspections } = this.entities

    // validation — runs before the row is written
    this.before('CREATE', Inspections, (req) => {
      if (!req.data.assetID) req.reject(400, 'assetID is required')
    })

    // react to a status change to "submitted"
    this.after('UPDATE', Inspections, async (data, req) => {
      if (data.status === 'submitted') {
        await this._raiseNotificationIfFailed(data.ID, req)
      }
    })

    return super.init()
  }

  async _raiseNotificationIfFailed(inspectionID, req) {
    const { InspectionItems } = this.entities
    const failed = await SELECT.from(InspectionItems)
      .where({ parent_ID: inspectionID, result: 'fail' })
    if (failed.length === 0) return
    // → Step 7: push a PM notification to ECC
    req.info(`${failed.length} failed item(s) — raising ECC notification`)
  }
}
```

The three handler phases you'll use constantly: `before` (validate / mutate input), `on` (replace the default behaviour entirely), `after` (read-only enrich, or trigger side-effects). Most of your business logic is `before` for guards and `after` for reactions.

---

## 7. Call ECC as a remote service (the clean-core boundary) - CURRENTLY STUCK HERE, MAY HAVE TO BACKTRACK AND ADJUST

This is the important one. Your app reaches into ECC over a **released API via a destination** — it never modifies the core. This is the single concrete piece of "clean core."

**a. Import the ECC service definition.** Get the EDMX/metadata of the ECC OData service you're calling (e.g. a PM notification service) and import it:

```bash
cds import ecc-pm-notification.edmx --as cds
```

This generates a local CDS definition of the remote service under `srv/external/`.

**b. Declare it as a required remote service** in `package.json` under `cds.requires`, pointing at a destination:

```jsonc
"cds": {
  "requires": {
    "ECC_PM": {
      "kind": "odata-v2",                  // ECC services are usually v2
      "model": "srv/external/ECC_PM",
      "credentials": { "destination": "ECC_PM_DEST" }   // BTP destination name
    }
  }
}
```

**c. Call it from your handler:**

```js
const ecc = await cds.connect.to('ECC_PM')
await ecc.create('Notifications', {
  equipment: data.assetID,
  type: 'M2',
  shortText: `Inspection ${inspectionID}: failed items`
})
```

The destination (`ECC_PM_DEST`) and its Cloud Connector route are configured in your BTP subaccount — that's the environment wiring, and the exact field names depend on the actual ECC service, so confirm those against your system. The *pattern* is what matters: ECC PM stays untouched; your app talks to it over a released API; your inspection workflow is entirely yours.

> Local testing tip: you don't need live ECC to develop. CAP supports mocking remote services, and `cds watch --profile hybrid` can bind real destinations when you're ready. Check current docs for the mocking setup — it shifts between cds-dk versions.

---

## 8. (Optional) Fiori elements preview

If you want a UI without writing one, annotate and let Fiori elements generate it. Add `app/inspections/annotations.cds`:

```cds
using InspectionService as svc from '../../srv/inspection-service';

annotate svc.Inspections with @(
  UI: {
    LineItem: [
      { Value: assetID },
      { Value: inspector },
      { Value: status },
      { Value: performedAt }
    ]
  }
);
```

`cds watch` will surface a Fiori elements preview link. This is the "free UI" path. If Fiori's too heavy for your UX (your earlier point), skip this entirely — your **UI5-Web-Components-for-React** app just consumes the same `http://localhost:4004/odata/v4/inspection-service/` endpoint. The backend doesn't change.

---

## 9. Swap SQLite → HANA Cloud

You've built everything on in-memory SQLite. Now make persistence real for production:

```bash
cds add hana
```

This adds the HANA config and `@cap-js/hana` driver. Locally you keep using SQLite (`cds watch`); the HANA config kicks in for the deployed profile. You'll bind a HANA Cloud instance from your BTP space at deploy time.

> Cost watch: HANA Cloud **meters while running**. In non-prod, schedule it to stop nights/weekends or the credit burn creeps up on you by month 9.

---

## 10. Add security

```bash
cds add xsuaa
```

This generates `xs-security.json` and wires XSUAA. Define roles and protect the service in CDS:

```cds
service InspectionService @(requires: 'authenticated-user') {

  entity Inspections @(restrict: [
    { grant: 'READ',  to: 'Inspector' },
    { grant: ['CREATE','UPDATE'], to: 'Inspector' },
    { grant: '*', to: 'InspectionAdmin' }
  ]) as projection on my.Inspections;

}
```

`@requires` / `@restrict` map to scopes and role collections. Locally, `cds watch` mocks users (configurable in `.cdsrc.json`) so you can test authorization without XSUAA. In BTP, role collections get assigned to users in the subaccount — an unassigned role collection is the classic silent 403.

---

## 11. Deploy to Cloud Foundry

```bash
cds add mta          # generates mta.yaml describing app + HANA + XSUAA
npm install -g mbt   # multi-target build tool, one-time
mbt build            # produces a .mtar archive
cf login             # target your BTP CF org/space
cf deploy mta_archives/inspection-app_1.0.0.mtar
```

The MTA descriptor bundles your CAP service, the HANA deployer, and the XSUAA instance, and binds them together. `cf deploy` provisions and wires the lot.

> Kyma path instead of CF: `cds add helm` / `cds up` is the route. Only go there if you already run k8s — otherwise CF is the simpler home for this.

---

## What to actually do tonight

Steps 1–6 take maybe 30 minutes and need nothing but Node. You'll have a running, queryable OData service with sample data and one piece of custom logic. That alone teaches you 80% of the CAP mental model.

Then the two steps worth real attention for your situation:
- **Step 7** — the ECC remote call. This is the clean-core boundary made concrete, and it's the part that's specific to running CAP beside ECC.
- **Step 10** — authorization model. Get the role design right early; retrofitting auth is miserable.

Skip 9 and 11 until you've got the app working locally and actually want it on BTP.
