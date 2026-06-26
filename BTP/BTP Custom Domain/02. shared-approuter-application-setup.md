# Shared Standalone Approuter + Custom Domain — UI5 Apps from HTML5 Repo

**Context:** Multiple UI5 apps served from the SAP HTML5 Application Repository, fronted by **one shared standalone approuter** (no SAP Build Work Zone), exposed under a **custom domain**.

**Scope of this doc:** the app-side files, the approuter module, and the auth/redirect wiring.
**Out of scope (already done):** Custom Domain service entitlement, TLS cert, and Custom Domain Manager configuration.

---

## Architecture in one line

The standalone approuter is the single running CF app that the custom domain maps to. It binds the HTML5 repo (`app-runtime` plan) and resolves every app by path. Apps deploy their content (incl. their own `manifest.json` and `xs-app.json`) into the repo via the `app-host` plan. Adding an app needs **no approuter change** — the repo handles fan-out by path.

> The approuter is the building's front desk wired to the directory, not to each tenant. Add a tenant → directory updates → the desk already knows how to route.

---

## Why each piece is needed

| Requirement | What it forces |
|---|---|
| Custom domain in CF | Must map to a **running CF app** → a standalone approuter (repo content alone has no route to attach a domain to). |
| One domain, many apps | **Shared** approuter (not one-per-app) → one route, one cert, path-based resolution. |
| No Work Zone | You operate + patch the approuter and run it HA yourself. |

---

## Part A — App-side files (per UI5 app)

The custom domain does **not** change these. They exist to make sure the app routes *through* the approuter at all.

### A1. `manifest.json` — relative dataSource URIs (the silent breaker)

All `dataSource` URIs must be **relative**. Absolute `cfapps...` URLs bypass the approuter entirely.

```json
"sap.app": {
  "id": "com.yourco.myapp",
  "dataSources": {
    "mainService": {
      "uri": "/srv/my-service/",
      "type": "OData",
      "settings": {
        "odataVersion": "2.0",
        "localUri": "localService/metadata.xml"
      }
    }
  }
}
```

- `id` becomes the app name in the repo and the path segment the approuter resolves (`/com.yourco.myapp-1.0.0/...`).
- Keep `id` **stable across deploys** or you break bookmarks/links.

> **OData V2 assumption (Gateway hub → SAP ERP):** URI path is `/srv/my-service/`, `odataVersion` is `"2.0"`, and the model in `sap.ui5.models` instantiates `sap.ui.model.odata.v2.ODataModel`. Gateway serves V2 **natively**, so no adapter/compatibility layer is needed — the app side is correct as-is. The real wiring effort is in the **destination + connectivity** to reach the Gateway hub (see A2 / B1 below).

### A2. `xs-app.json` (app's own, bundled into app-host content)

Carries **this app's** backend routing. The central approuter reads it per-app from the repo.

```json
{
  "welcomeFile": "/index.html",
  "authenticationMethod": "route",
  "routes": [
    {
      "source": "^/srv/(.*)$",
      "target": "/sap/opu/odata/$1",
      "destination": "gw-hub",
      "authenticationType": "xsuaa",
      "csrfProtection": true
    }
  ]
}
```

> **Gateway hub target:** the `/srv/(.*)` source rewrites to `/sap/opu/odata/$1` — the Gateway OData entry path — so the app's `uri: "/srv/my-service/"` resolves to `/sap/opu/odata/my-service/` on the hub. Adjust the prefix if you publish under a namespace (`/sap/opu/odata4/...` only if you ever expose V4, which you're not here).

> ⚠️ The `destination` (`gw-hub`) must be bound to the **central approuter** (it does the proxying), **not** to the app module.

> **Destination + connectivity (Gateway hub):** the `gw-hub` destination points at your Gateway system. If the hub is on-prem, it's `ProxyType: OnPremise` → routed via **Cloud Connector** (run an **HA pair** — see watch-outs), and the approuter also needs the **connectivity** service bound (B1). For auth, prefer **principal propagation** (`SAP Assertion SSO` / `PrincipalPropagation`) so the ERP sees the real end user, not a shared technical user — otherwise you lose per-user authorization and traceability in ERP. If the hub is reachable directly (e.g. via Web Dispatcher with a public endpoint), `ProxyType: Internet` and Cloud Connector isn't needed.

> Default to **per-app `/srv` routes** so apps stay independently deployable. Use a single shared `/srv` route in the central approuter only if every app genuinely hits one backend.

### A3. App deploy modules in `mta.yaml` (standard `app-host`, unchanged by domain)

```yaml
  - name: myapp
    type: html5
    path: app/myapp
    build-parameters:
      build-result: dist
      builder: custom
      commands: [npm install, npm run build:cf]
      supported-platforms: []
  - name: myapp-content
    type: com.sap.application.content
    path: .
    requires:
      - name: html5-repo-host
        parameters: { content-target: true }
    build-parameters:
      build-result: resources
      requires:
        - name: myapp
          artifacts: [myapp.zip]
          target-path: resources/
```

Nothing here references the domain — that's the design working correctly.

---

## Part B — The shared standalone approuter

### B1. Approuter module in `mta.yaml`

```yaml
  - name: approuter
    type: approuter.nodejs
    path: approuter
    parameters:
      memory: 256M
      instances: 2          # HA — single front door to ALL apps; non-negotiable
    requires:
      - name: uaa-myproject
      - name: html5-repo-runtime     # app-runtime plan, serves all repo apps
      - name: dest-myproject         # destinations live HERE, not on the apps
      - name: conn-myproject         # connectivity — needed to reach on-prem Gateway via Cloud Connector
```

### B2. Central approuter `xs-app.json` (shared routes only)

This file owns the **common** concerns — logout, an optional bare-domain landing, and the html5-repo catch-all. Per-app backend routes stay in each app's own `xs-app.json` (A2).

**Design decision: each app has its own landing page.** Apps are reached by their own deep URL (`apps.yourco.com/com.yourco.app1/...`), so there is **no single home** for the approuter to resolve. Do **not** set a shared `welcomeFile` on the central approuter — it would just force one app as a confusing default for anyone hitting the bare domain.

For the bare-domain case (`apps.yourco.com/`), ship a tiny static `index.html` in the approuter's `resources/` that links to each app. Costs one HTML file and kills the "I typed the domain and got an error" support tickets.

```json
{
  "authenticationMethod": "route",
  "logout": {
    "logoutEndpoint": "/do/logout",
    "logoutPage": "/logout-page.html"
  },
  "routes": [
    {
      "source": "^/logout-page.html$",
      "localDir": "resources",
      "authenticationType": "none"
    },
    {
      "source": "^/$",
      "localDir": "resources",
      "target": "index.html",
      "authenticationType": "xsuaa"
    },
    {
      "source": "^/(.*)$",
      "target": "$1",
      "service": "html5-apps-repo-rt",
      "authenticationType": "xsuaa"
    }
  ]
}
```

- **No `welcomeFile`** at the approuter level — per-app landing is handled by each app's own `xs-app.json` (see note below).
- The `^/$` route serves a small static directory page from `resources/index.html`. Drop this route entirely if you'd rather let the bare domain 404 (fine when the root is never advertised).
- The `html5-apps-repo-rt` route is the catch-all that resolves any `/<appname-version>/...` request to the right repo app. Keep it **last**.

> **Per-app landing — the line that actually matters (app side, not router):** each app's own `xs-app.json` must set `"welcomeFile": "/index.html"` (already in A2). That's what makes `apps.yourco.com/com.yourco.app1/` land on app1's page instead of a directory listing. The router resolves *to* the app; the app's own `xs-app.json` resolves *within* it. Confirm this line is present in **every** app.

### B3. Map the custom domain (post-deploy, not in mta)

Do this with `cf map-route` after the approuter is up — keeps a domain typo from failing the whole MTA deploy.

```bash
cf map-route approuter apps.yourco.com
```

---

## Part C — Auth / redirect wiring (the #1 cutover failure)

### C1. `xs-security.json` — redirect URIs

The moment users hit the custom domain, OAuth login breaks unless its callback origin is trusted.

```json
{
  "xsappname": "myproject",
  "tenant-mode": "dedicated",
  "oauth2-configuration": {
    "redirect-uris": [
      "https://apps.yourco.com/**",
      "https://*.cfapps.eu10.hana.ondemand.com/**"
    ]
  }
}
```

- Add the custom-domain URI **before** cutover.
- Keep the old `cfapps` wildcard during transition; remove later if desired.
- Apply by updating/redeploying the XSUAA instance (`cf update-service` or MTA redeploy) **before** pointing DNS.

### C2. IAS / corporate IdP (if used)

If you front with IAS (or corporate IdP via IAS), add the same custom-domain origin to the **IAS trusted-origins / redirect** config. Same failure mode, different screen.

---

## Cutover checklist (in order)

1. [ ] `manifest.json` → all dataSource URIs **relative** (every app)
2. [ ] App `xs-app.json` → per-app `/srv` route **and** `"welcomeFile": "/index.html"` present (every app)
3. [ ] Destination `gw-hub` (→ Gateway, `/sap/opu/odata`) bound to the **approuter**, not the apps; **connectivity** service bound too if on-prem
4. [ ] Cloud Connector reaching the Gateway hub (**HA pair**); **principal propagation** trust configured end-to-end (subaccount → CC → ERP)
5. [ ] Central approuter `xs-app.json` → **no shared `welcomeFile`**; html5-repo catch-all last; logout set; optional `^/$` static landing in `resources/`
6. [ ] Approuter at **`instances: 2`** (HA)
7. [ ] `xs-security.json` → custom-domain **redirect-uri added and applied**
8. [ ] IAS trusted-origins updated (if IAS in the chain)
9. [ ] Deploy → `cf map-route approuter apps.yourco.com`
10. [ ] Smoke-test login + one `/srv` OData call (V2) through to ERP on the custom domain **before** switching DNS

---

## Watch-outs

- **Single point of failure:** the shared approuter fronts *every* app. Run ≥2 instances; one crash = everything dark. (Work Zone handled this for you; standalone, it's on you.)
- **Shared origin = shared session/auth:** one XSUAA instance and one scope set span all apps. Plan the role-collection model so app A's scopes don't become app B's attack surface.
- **Cost is "cheaper," not "free":** you skip the Work Zone subscription, but the Custom Domain service is a paid plan and the approuter is an always-on app (×2 for HA) burning credits 24/7. Forecast steady-state monthly burn.
- **Shared dependency:** any change to the central approuter (xs-app.json, version bump, new scope) affects all apps behind it. Decide ownership and change/rollout process before app #4 arrives.

---

## Verify against current docs (drift-prone)

- `@sap/approuter` version-specific `xs-app.json` options and the html5-repo plan names.
- The `html5-apps-repo` `app-runtime` vs `app-host` plan binding details.
- XSUAA `redirect-uris` wildcard behavior in your XSUAA version.
