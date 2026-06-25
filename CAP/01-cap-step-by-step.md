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
- [10. Transfer Data Processing to Actual Service](#10-transfer-data-processing-to-actual-service)
- [11. Add security](#11-add-security)
- [12. Deploy to Cloud Foundry](#12-deploy-to-cloud-foundry)
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
cds init demo-app --nodejs    # always pass --nodejs (or --java)
cd demo-app
npm install
```

NOTE: In case you forgot to pass --<language> argument as above, package.json would not be generated and you can create package.json manually with the content as below (reference for nodejs variant). 
```js
{
  "name": "demo-app",
  "version": "1.0.0",
  "dependencies": {
    "@sap/cds": "^9"
  },
  "devDependencies": {
    "@cap-js/sqlite": "^2.4"
  },
  "scripts": {
    "start": "cds-serve"
  },
  "private": true
}

```

You may also want to stage package-lock.json to prevent error during build
```bash
git add package-lock.json
```



This creates the standard layout:

```
demo-app/
├── app/          # UI layer (Fiori elements, or your React app points here)
├── db/           # data model + sample data
├── srv/          # service definitions + custom logic
└── package.json  # cds config lives under the "cds" key
```

Three folders, clear separation: `db` = what the data *is*, `srv` = what you *expose* and the logic, `app` = how it *looks*.

---

## 1b. Set up your workbench (VS Code)

Step 1 was the last pure-PowerShell step for a while. From here you're creating and editing files, which is editor work. Set up the two-pane workbench once:

1. From inside the `demo-app` folder in PowerShell, run `code .` to open VS Code on the project. (If `code` isn't recognized: open VS Code manually → File → Open Folder → pick the `demo-app` folder. Or reinstall VS Code with the "Add to PATH" option ticked.)

2. Install the **SAP CDS Language Support** extension (publisher: SAP SE) from the Extensions panel (`Ctrl+Shift+X`, search "CDS"). Without it you're editing `.cds` files blind — with it you get highlighting, autocomplete, and inline errors.

3. Open VS Code's integrated terminal (`` Ctrl+` ``) and run `cds watch` **now**, before you create any files. It'll sit waiting on an empty model. As you add each file below, you'll watch it pick them up live — far better for learning than building everything then running it cold.

Now you have: editor pane for files, integrated terminal running `cds watch`, browser on `localhost:4004`. That's the loop.if you are using codespace in github, then your URL should be https://<codespace_name>-4004.app.github.dev/

Why Port 4004 specifically? It's just CAP's default port. You can override it (cds watch --port 4005, or the PORT env var), but there's no reason to — leave it at 4004 and that's the address to point the browser at.

---

## 1c. The files you'll create, in order

`cds init` already made the folders (`db/`, `srv/`, `app/`) and `package.json`. You're *adding files into* those folders. In VS Code's file explorer (left panel), right-click the folder → New File → name it exactly as shown → paste the content from the matching step → save.

| # | Create this file | In folder | Content from | Type | Remarks |
|---|---|---|---|---|---|
| 1 | `schema.cds` | `db/` | Step 2 | CDS model |---|
| 2 | `demo-service.cds` | `srv/` | Step 3 | CDS service |---|
| 3 | `<Namespace>-SearchResult.csv` | `db/data/` | Step 5 | sample data | `db/data/` to be created manually |
| 4 | `<Namespace>-DocsAttachment.csv` | `db/data/` | Step 5 | sample data | `db/data/` to be created manually |
| 5 | `demo-service.js` | `srv/` | Step 6 | JS logic |---|
| 6 | `annotations.cds` | `app/demo-app/` | Step 8 *(optional UI)* | CDS annotations | `app/demo-app/` to be created manually | 


**IMPORTANT:** The two `.js`/`.cds` in `srv/` must share the base name (`demo-service`) — that's how CAP auto-wires the logic file to the service.

---

## 2. Define the data model

Create `db/schema.cds`. This is **your** model — it does not live in ECC.

```cds
namespace compname.divname.demoapp; //Namespace referred in some places in this document (all small letter, no separator on words-to be reviewed for Governance)

using { cuid, managed } from '@sap/cds/common';
// These are reusable mixins imported from SAP's standard CDS library (@sap/cds/common). When you add them to an entity, they automatically inject pre-defined fields:
// cuid — adds a UUID primary key:
// key ID : UUID;
// managed — adds four audit fields:
// createdAt  : Timestamp;
// createdBy  : String;
// modifiedAt : Timestamp;
// modifiedBy : String;
// CAP fills these in automatically on create/update — you don't write any code for it.

entity SearchResult : cuid, managed {
  PONum     : String(10);      //Ebeln                
  PRNum     : String(10);      //Banfn
  NetAmount : Decimal(20,5);   //Netwr
  DocName   : String(100);     //Name1
  NavAttachment : Composition of many DocsAttachment on NavAttachment.parent = $self; //parent owns children (delete the parent, its childern go too)
}

entity DocsAttachment : cuid {
  parent             : Association to SearchResult; //attribte have to be named "parent" would be a reference without ownership. The composition is what makes child entity a single deep document.
  AttachmentBinary   : LargeBinary;  //getdocatta/FileAtta
}
```

For more info on CDS Schema Syntax, refer to https://github.com/and23jul/Learning-Repository/blob/main/CAP/02-cds-schema-syntax.md
---

## 3. Define the service

Create `srv/demo-service.cds`. This exposes the model as an OData v4 service.

```cds
using { compname.divname.demoapp as myNS } from '../db/schema';     //Namespace Referred here and Instantiated for Local DB

service svcDemoApp {                                           //Service Instantiated here for Local DB
  entity entSearchResult     as projection on myNS.SearchResult;    //Entity Referred here and Instantiated
  entity entDocsAttachment   as projection on myNS.DocsAttachment;  //Entity Referred here and Instantiated
}
```

`projection on` means you're exposing the entity, optionally reshaped (you can rename, hide, or compute fields here later). Keep the service as the place you decide what the outside world sees — don't just blanket-expose everything when the app grows.

---

## 4. Run it — the inner loop

```bash
cds watch
```

Open **http://localhost:4004**. You have a live OData v4 service on an in-memory SQLite DB, with no deploy and no database to install. The index page lists your entities; click `SearchResult` to hit the OData endpoint.

Leave this running. Every file save reloads automatically. This is where you'll live while building.

Try a query in the browser:
```
http://localhost:4004/odata/v4/demo-service/svcDemoApp
```
or if you use GitHub Codespace:
```
https://<codespace_name>-4004.app.github.dev/
https://<codespace_name>-4004.app.github.dev/$fiori-preview/svcDemoApp/entSearchResult
```
---

## 5. Add sample data

IMPORTANT: So you're not staring at empty sets. Create CSVs in `db/data/`, named `<namespace>-<Entity>.csv`:

`db/data/compname.divname.demoapp-SearchResult.csv`
```csv
ID;PONum;PRNum;NetAmount;DocName
0000000001;8000149067;4000153006;10000.00;Dummy Doc 1
0000000002;8000148417;4000153971;20000.00;Dummy Doc 2
```

`db/data/compname.divname.demoapp-DocsAttachment.csv`
```csv
ID;parent_ID;AttachmentBinary
1000000001;0000000001;XXX
1000000002;0000000002;YYY

```

Header names = element names. The association column is `parent_ID` (the association name + `_ID`). `cds watch` reloads and the data is queryable immediately.

---

## 6. Custom logic — event handlers

The CDS files gave you CRUD for free. Now add behaviour. Create `srv/demo-service.js` — CAP auto-wires a `.js` next to the `.cds` of the same name.

```js
const cds = require('@sap/cds')

module.exports = class clsDemoApp extends cds.ApplicationService {
  init() {
    const { entSearchResult } = this.entities

    this.before('CREATE', entSearchResult, (req) => {
      if (!req.data.PONum) req.reject(400, 'PONum is required')
    })

    return super.init()
  }
}
```

The three handler phases you'll use constantly: `before` (validate / mutate input), `on` (replace the default behaviour entirely), `after` (read-only enrich, or trigger side-effects). Most of your business logic is `before` for guards and `after` for reactions.

---

## 7. Call ECC as a remote service (the clean-core boundary) 

This is the important one. Your app reaches into ECC over a **released API via a destination** — it never modifies the core. This is the single concrete piece of "clean core."

**a. Import the ECC service definition.** Get the EDMX/metadata of the ECC OData service you're calling and import it:

```bash
cds import ZMYOPTIMA_SRV.edmx --as cds
```

This generates a local CDS definition of the remote service under `srv/external/`.

**b. Declare it as a required remote service** in `package.json` under `cds.requires`, pointing at a destination (Supposedly already automatically generated, but add the credentials):

```jsonc
"cds": {
   "requires": {
      "ZMYOPTIMA_SRV": {
        "kind": "odata-v2",
        "model": "srv/external/ZMYOPTIMA_SRV",
        "credentials": { "destination": "Gateway-x96-SSO" }   //To be added manually, referring to BTP destination
      }
    }
}
```


The destination  and its Cloud Connector route are configured in your BTP subaccount — that's the environment wiring, and the exact field names depend on the actual ECC service, so confirm those against your system. The *pattern* is what matters: ECC stays untouched; your app talks to it over a released API; your inspection workflow is entirely yours.

> Local testing tip: you don't need live ECC to develop. CAP supports mocking remote services, and `cds watch --profile hybrid` can bind real destinations when you're ready. Check current docs for the mocking setup — it shifts between cds-dk versions.

---

## 8. (Optional) Fiori elements preview

If you want a UI without writing one, annotate and let Fiori elements generate it. Add `app/demo-app/annotations.cds`:

```cds
using svcDemoApp as svc from '../../srv/demo-service'; //This svcSearchResult referred from the one in search-result.cds

annotate svc.entSearchResult with @(
  UI: {
    LineItem: [
      { Value: PONum },
      { Value: PRNum }    ]
  }
);


```

Under `srv/` add `server.js` with content as below
```js
const cds = require('@sap/cds')

cds.on('bootstrap', (app) => {
  app.use((req, res, next) => {
    if (req.path.startsWith('/$fiori-preview/') && req.path.endsWith('.js')) {
      res.type('application/javascript')
    }
    next()
  })
})

module.exports = cds.server
```

in `package.json` under `cds` add `fiori.preview` with value true

```jsonc
"cds": {
  "fiori": {
    "preview": true
  },
  "requires": {
  .....
  }
}

```

`cds watch` will surface a Fiori elements preview link. This is the "free UI" path. If Fiori's too heavy for your UX (your earlier point), skip this entirely — your **UI5-Web-Components-for-React** app just consumes the same endpoint. The backend doesn't change.

---

## 9. If You need Data Persistency Swap SQLite → Actual Database

You've built everything on in-memory SQLite. Now make persistence real for production:

```bash
cds add hana
```

This adds the HANA config and `@cap-js/hana` driver. Locally you keep using SQLite (`cds watch`); the HANA config kicks in for the deployed profile. You'll bind a HANA Cloud instance from your BTP space at deploy time.

> Cost watch: HANA Cloud **meters while running**. In non-prod, schedule it to stop nights/weekends or the credit burn creeps up on you by month 9.

Alternatively, we can use Postgre (Cheaper but not very cheap also)

```bash
cds add postgres
```

This adds the `@cap-js/postgres` driver, registers Postgres as your `db`, and wires a **PostgreSQL deployer app** into your `mta.yaml` pointing at the `postgresql-db` service. (Run `cds add mta` first — Step 11 — so the resource has somewhere to land.)

Keep the fast inner loop on SQLite and use Postgres only when deployed — scope it to the production profile so `cds watch` stays in-memory and instant:

```bash
cds env requires.db --for production
```

**How the schema actually gets there.** Unlike HANA's HDI deployer, the auto-generated Postgres deployer compiles your model into `gen/pg/db/csn.json`, copies your `db/data/*.csv` alongside, and runs `cds-deploy` once at deploy time to create tables and load data. Note the dash in `cds-deploy` — the deployer runs without `@sap/cds-dk`, so the `cds` CLI isn't on its PATH.

> **Trial-account gotcha:** `cds add postgres` defaults the service plan to `development`, which **isn't available on BTP trial**. If you're on trial, open `mta.yaml` and change the `postgresql-db` resource's `service-plan: development` → `trial`. On PAYG/CPEA there's also a free instance plan worth using for non-prod.

> **Cost watch:** Postgres meters while running, same as HANA — it's cheaper at small scale, not free. Use the free/trial plan for non-prod and don't leave a `standard`/`development` instance running over nights and weekends.

> **Local-parity warning:** SQLite ≠ Postgres, the same way SQLite ≠ HANA. Postgres folds unquoted identifiers to lowercase and is stricter on types, so something that passes on SQLite can break once deployed. Before you ship, smoke-test against a real Postgres locally via Docker (`postgres:alpine`, port 5432) and run with `DEBUG=sql` to see the actual SQL. Catch the dialect gaps on your laptop, not in CF.

---


## 10. Transfer Data Processing to Actual Service 

In `db/schema.cds`. add `@cds.persistence.skip` above entity definition

```cds
namespace compname.divname.demoapp; //Namespace referred in some places in this document (all small letter, no separator on words-to be reviewed for Governance)

....

@cds.persistence.skip
entity SearchResult : cuid, managed {
  ...
}

@cds.persistence.skip
entity DocsAttachment : cuid {
  ...
}
```


Also add these lines to the event handler `srv/demo-service.js` 

```js
this.on('READ', Inspections, async (req) => {
  const ecc = await cds.connect.to('ECC_PM')   // your required remote service
  return ecc.run(req.query)                      // forward the CQN query as-is
})
```

If the entities is just SAP backend Entity, instead of redefining in `db/schema.cds`, the definition should be done on service definition `srv/demo-service.cds` instead. here's how you define them, to be on the safe side, we safeguard the query to return only 10 records.

```cds
using { compname.divname.demoapp as myNS } from '../db/schema';   //Local DB namespace
// service svcDemoApp {                                           //Service Instantiated here (for Local DB)
//   entity entSearchResult     as projection on myNS.SearchResult;    //Entity Referred here and Instantiated
//   entity entDocsAttachment   as projection on myNS.DocsAttachment;  //Entity Referred here and Instantiated
// }

using { ZMYOPTIMA_SRV } from './external/ZMYOPTIMA_SRV';           //Imported ECC OData service

service svcDemoApp {
  // Live ECC: reshape getResearchDetailsSet's fields into our names
  @cds.query.limit: { default: 10, max: 10 }   // cap page size at 10
  entity entSearchResult as projection on ZMYOPTIMA_SRV.getResearchDetailsSet {
    key Banfn as ID,
        Banfn as PRNum,
        Ebeln as PONum
  };
  entity entDocsAttachment   as projection on myNS.DocsAttachment;   //Local DB entity
}

```

In the event handler `srv/demo-service.js` the method `init` have to be changed to `async init` and add "Live ECC" section

```js
const cds = require('@sap/cds')

module.exports = class clsDemoApp extends cds.ApplicationService {
//init() { //This is for Local Model
  async init() {
    const { entSearchResult } = this.entities

    // Live ECC: forward reads of entSearchResult to the remote OData service
    const ecc = await cds.connect.to('ZMYOPTIMA_SRV')
    this.on('READ', entSearchResult, (req) => ecc.run(req.query))

    this.before('CREATE', entSearchResult, (req) => {
      if (!req.data.PRNum) req.reject(400, 'PRNum is required')
    })

    return super.init()
  }
}

```

## 11. Add security

```bash
cds add xsuaa
```

This generates `xs-security.json` and wires XSUAA. Define roles and protect the service in CDS:

```cds
{
  "xsappname": "demo-app",
  "tenant-mode": "dedicated",
  "scopes": [],
  "attributes": [],
  "role-templates": []
}

```

Create  `/app/router/package.json` assuming that the approuter is a dedicated app router
```jsonc
{
  "name": "demo-app-approuter",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "@sap/approuter": "^22"
  },
  "scripts": {
    "start": "node node_modules/@sap/approuter/approuter.js"
  }
}
```

Create  `/app/router/xs-app.json` 
```jsonc
{
  "authenticationMethod": "route",
  "routes": [
    {
      "source": "^(.*)$",
      "target": "$1",
      "destination": "srv-api",
      "authenticationType": "xsuaa",
      "csrfProtection": false
    }
  ]
}

```



`@requires` / `@restrict` map to scopes and role collections. Locally, `cds watch` mocks users (configurable in `.cdsrc.json`) so you can test authorization without XSUAA. In BTP, role collections get assigned to users in the subaccount — an unassigned role collection is the classic silent 403.

---

## 12. Deploy to Cloud Foundry

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
