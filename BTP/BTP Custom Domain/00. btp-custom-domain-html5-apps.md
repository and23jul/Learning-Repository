# Serving Standalone HTML5 Apps on a Custom Domain (SAP BTP, Cloud Foundry)

How to expose HTML5 apps that live in the **HTML5 Application Repository** — *not* published in SAP Build Work Zone — under a clean, branded URL like:

```
https://apps.<companyname>.com/aps/apps/app1
https://apps.<companyname>.com/aps/apps/app2
```

> **Grounding note.** This targets the **Cloud Foundry** environment of SAP BTP. Service names, the Custom Domain Manager UI labels, and certificate options shift between releases; exact click-paths and the DNS CNAME target are tenant/region-specific. **Confirm against current SAP Help and what the Custom Domain Manager shows you.** No live network was used to build this — treat command/label specifics as a verify-in-your-tenant skeleton, not gospel.

---

## 0. Read this first: the requirement, corrected

Your stated URL `https://<companyname>/aps/apps/app1` needs two fixes to be real, and the fixes map to two different BTP capabilities:

1. **`<companyname>` must be a fully-qualified domain name** (e.g. `apps.companyname.com`), not a bare label. A browser and a TLS certificate both need an FQDN. → **Custom Domain service.**
2. **The `/aps/apps/<app>` path is *not* something the domain controls.** A domain resolves to a host; everything after the first `/` is decided by the app the route points at — here, the **standalone approuter** and its `xs-app.json`. → **Approuter.**

So the mental model:

```
  Browser
    │  https://apps.companyname.com/aps/apps/app1/index.html
    ▼
  ┌─────────────────────────────┐
  │ Custom Domain  (the HOST)   │  TLS cert + DNS → routes to ↓
  └─────────────────────────────┘
    │
    ▼
  ┌─────────────────────────────┐
  │ Standalone Approuter        │  xs-app.json rewrites the PATH:
  │  /aps/apps/(.*) → /$1       │   /aps/apps/app1/... → /app1/...
  │  + XSUAA login              │
  └─────────────────────────────┘
    │  forwards to html5-apps-repo-rt
    ▼
  ┌─────────────────────────────┐
  │ HTML5 Application Repository│  serves app1, app2, … by name
  └─────────────────────────────┘
```

One approuter serves **all** your apps. Adding app3 later is just deploying it to the repo — no domain or approuter change. That's the payoff of doing it this way.

**Why standalone, not Work Zone:** the managed approuter behind Build Work Zone owns its own routing and URL scheme; you don't get to define `/aps/apps/<app>`. A standalone approuter is a CF app you control, so you own the `xs-app.json` and therefore the path. Your instinct to avoid Work Zone here is correct.

---

## 1. Prerequisites

- A BTP subaccount in the **Cloud Foundry** environment, with a Space and CF CLI access.
- Your HTML5 apps building to static content (UI5/Fiori or plain HTML5), deployable to the HTML5 Application Repository.
- **MTA build tooling**: `mbt` (Cloud MTA Build Tool) + the MultiApps CF plugin (`cf install-plugin multiapps`).
- **Entitlements** in the subaccount for: `html5-apps-repo` (app-host + app-runtime), `xsuaa` (application), and the **Custom Domain** service.
- **DNS control** over `companyname.com` (ability to add CNAME/TXT records).
- A **TLS server certificate** for your host (e.g. `apps.companyname.com` or wildcard `*.companyname.com`) **or** use an SAP-managed certificate if your tenant offers it (verify availability).

---

# Part A — Serve the apps under `/aps/apps/<app>` (do this first, on the default domain)

Get the path pattern working on the standard `*.cfapps.<region>.hana.ondemand.com` URL **before** touching DNS. Debugging routing and custom domain at the same time is how weekends die.

## A.1 Project layout

```
aps-html5/
├── mta.yaml
├── xs-security.json
├── approuter/
│   ├── package.json        # depends on @sap/approuter
│   └── xs-app.json
├── app1/                   # a UI5/HTML5 app project
└── app2/
```

## A.2 `mta.yaml` (approuter + content deployer + apps + services)

```yaml
_schema-version: "3.2"
ID: com.company.aps.html5
version: 1.0.0

modules:
  # 1) Standalone application router — the thing the custom domain will point at
  - name: aps-approuter
    type: approuter.nodejs
    path: approuter
    parameters:
      memory: 256M
      disk-quota: 512M
    requires:
      - name: aps-uaa
      - name: aps-html5-runtime
    provides:
      - name: app-api
        properties:
          url: ${default-url}

  # 2) Content deployer — uploads the built apps into the HTML5 repo (app-host)
  - name: aps-html5-content
    type: com.sap.application.content
    path: .
    requires:
      - name: aps-html5-host
        parameters:
          content-target: true
    build-parameters:
      build-result: resources
      requires:
        - name: app1
          artifacts: [app1.zip]
          target-path: resources/
        - name: app2
          artifacts: [app2.zip]
          target-path: resources/

  # 3) The HTML5 apps themselves
  - name: app1
    type: html5
    path: app1
    build-parameters:
      builder: custom
      commands: [npm ci, npm run build]
      build-result: dist
      supported-platforms: []
  - name: app2
    type: html5
    path: app2
    build-parameters:
      builder: custom
      commands: [npm ci, npm run build]
      build-result: dist
      supported-platforms: []

resources:
  - name: aps-uaa
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json
  - name: aps-html5-host        # where apps are stored
    type: org.cloudfoundry.managed-service
    parameters:
      service: html5-apps-repo
      service-plan: app-host
  - name: aps-html5-runtime     # what the approuter binds to, to serve them
    type: org.cloudfoundry.managed-service
    parameters:
      service: html5-apps-repo
      service-plan: app-runtime
```

| Piece | What it does |
|---|---|
| `approuter.nodejs` module | The standalone approuter app. The custom domain route gets mapped to **this**. |
| `com.sap.application.content` module | A one-shot deployer that pushes the built app zips into the repo's **app-host** instance, then exits. |
| `html5` modules | Your actual apps; `npm run build` produces the static `dist` that gets zipped. |
| `html5-apps-repo / app-host` | Storage for the deployed apps. |
| `html5-apps-repo / app-runtime` | Bound to the approuter so it can **serve** apps via `html5-apps-repo-rt`. |
| `xsuaa` | Authentication; issues the login the approuter enforces. |

## A.3 `approuter/xs-app.json` — where the `/aps/apps/<app>` path is created

**Primary approach — generic rewrite (low maintenance):**

```json
{
  "authenticationMethod": "route",
  "sessionTimeout": 30,
  "welcomeFile": "/aps/apps/app1/index.html",
  "routes": [
    {
      "source": "^/aps/apps/(.*)$",
      "target": "/$1",
      "service": "html5-apps-repo-rt",
      "authenticationType": "xsuaa"
    }
  ]
}
```

| Element | What it does |
|---|---|
| `authenticationMethod: route` | Auth is decided per-route (so each route's `authenticationType` applies). |
| `welcomeFile` | Where `/` redirects — point it at your landing app. |
| `source` (regex) | Matches the incoming path; `(.*)` captures everything after `/aps/apps/`. |
| `target` | Rewrites it — strips the `/aps/apps/` prefix so the repo sees `/app1/...`. |
| `service: html5-apps-repo-rt` | Forwards the (rewritten) path to the HTML5 App Repository runtime. |
| `authenticationType: xsuaa` | Forces XSUAA login before serving. |

With this single rule, **every** app in the repo is reachable at `/aps/apps/<appName>/...`. `<appName>` is the app's registered name in the repo (derived from its `sap.app.id` in `manifest.json`).

**Variant — vanity aliases (full control, decouples URL from manifest id):**

If your manifest ids are namespaced (e.g. `com.company.app1`) but you want a clean `/aps/apps/app1`, map each explicitly:

```json
{
  "authenticationMethod": "route",
  "routes": [
    {
      "source": "^/aps/apps/app1/(.*)$",
      "target": "/com.company.app1/$1",
      "service": "html5-apps-repo-rt",
      "authenticationType": "xsuaa"
    },
    {
      "source": "^/aps/apps/app2/(.*)$",
      "target": "/com.company.app2/$1",
      "service": "html5-apps-repo-rt",
      "authenticationType": "xsuaa"
    }
  ]
}
```

Trade-off: clean, fully-controlled URLs, but you edit `xs-app.json` and redeploy the approuter for every new app. The generic rule doesn't. Pick based on how often the app list changes and whether you can live with the manifest id as the path segment.

## A.4 `xs-security.json` — whitelist redirect URIs (critical for later)

```json
{
  "xsappname": "aps-html5",
  "tenant-mode": "dedicated",
  "oauth2-configuration": {
    "redirect-uris": [
      "https://*.cfapps.<region>.hana.ondemand.com/**",
      "https://apps.companyname.com/**"
    ]
  }
}
```

| Element | What it does |
|---|---|
| `redirect-uris` | The **only** hosts XSUAA will redirect back to after login. **The custom-domain entry must be here** or, once you switch domains, login will loop or fail with a redirect-mismatch error. Add it now so Part B "just works." |

## A.5 Build, deploy, test on the default domain

```bash
mbt build                      # produces mta_archives/*.mtar
cf deploy mta_archives/*.mtar
cf apps                        # note the aps-approuter route
```

Test before going further:

```
https://<subaccount>-aps-approuter.cfapps.<region>.hana.ondemand.com/aps/apps/app1/
```

If that loads app1 after login, **Part A is done.** If resources 404 (see Gotcha G1), fix it here — it's far easier to diagnose without the custom domain in the mix.

---

# Part B — Put it on `apps.companyname.com`

Now the host. The approuter and path are already working; you're only changing what's in front of it.

## B.1 Entitle and open the Custom Domain Manager

1. In the BTP cockpit, add the **Custom Domain** service entitlement to the subaccount.
2. Subscribe to the **Custom Domain Manager** application (Service Marketplace / Subscriptions) and open it.

> The Manager is the supported way to bring your own domain + TLS to CF apps. Exact labels move between versions — follow the wizard; the steps below are the shape, not the literal buttons.

## B.2 Provide the TLS certificate

Two options (availability is tenant-dependent — check what the Manager offers):

- **Bring your own:** generate a CSR (in the Manager or your own tooling), get it signed by your CA for `apps.companyname.com` (or wildcard `*.companyname.com`), then upload the **server certificate + full intermediate chain + private key** (PEM).
- **SAP-managed certificate:** if your tenant supports automatic certificates, let the Manager provision and renew it — less to operate. Verify this is available before relying on it.

> Wildcard (`*.companyname.com`) is worth it if you'll add more branded hosts later; a single-host cert is fine for just `apps.`.

## B.3 Register the domain in the Manager

In the Custom Domain Manager: add the custom domain (e.g. `apps.companyname.com`), associate the certificate from B.2, and **activate/serve** it. The Manager will surface:

- a **domain-verification value** (usually a TXT record), and
- the **CNAME target** your DNS must point to.

## B.4 DNS records

At your DNS provider, for `companyname.com`:

| Record | Name | Value | Why |
|---|---|---|---|
| **TXT** | (as shown by Manager) | (verification token from Manager) | Proves you own the domain |
| **CNAME** | `apps` | (the CNAME target shown by the Manager) | Routes `apps.companyname.com` traffic into BTP |

> Do **not** invent the CNAME target — use exactly what the Manager displays for your region. DNS propagation can take minutes to hours; the Manager won't go green until it resolves.

## B.5 Map the CF route to the approuter

Once the domain is active in CF, point it at the approuter app:

```bash
# Custom domain registered as companyname.com, host "apps":
cf map-route aps-approuter companyname.com --hostname apps
#   → binds https://apps.companyname.com to aps-approuter
```

(If the Manager registered the full `apps.companyname.com` as the domain with an empty host, map accordingly. The Manager/CF will tell you which form you have.)

## B.6 Confirm the XSUAA redirect URI (and IAS, if used)

- You already added `https://apps.companyname.com/**` to `redirect-uris` in A.4 — confirm it's live (`cf service aps-uaa` / redeploy if you changed it).
- **If your IdP is SAP Cloud Identity (IAS)** rather than default XSUAA: in the IAS application, add the custom-domain URL to the application's **redirect/home URLs**, and make sure the **trust** between the subaccount and IAS covers it. Custom-domain SSO loops are almost always a missing redirect entry on the IAS side.

## B.7 Test

```
https://apps.companyname.com/aps/apps/app1
https://apps.companyname.com/aps/apps/app2
```

Login, app loads, network tab shows no cross-host redirects or 404s on resources. Done.

---

## 2. Gotchas that actually bite

- **G1 — Absolute resource paths break under the prefix.** When the browser loads `…/aps/apps/app1/index.html`, a well-built UI5 app requests resources **relative** to that URL, so they resolve back through the `/aps/apps/` rewrite and work. But any **absolute** reference in the app (`/app1/something.js`, hardcoded paths) bypasses the prefix and 404s. Fix: keep app resource references relative; avoid hardcoded absolute roots. This is the #1 reason a rewritten path "half works" (page loads, assets missing).
- **G2 — Redirect-URI mismatch after switching domains.** If login loops or throws a redirect error on the custom domain, the custom-domain URL isn't in XSUAA `redirect-uris` (or IAS redirect URLs). A.4 / B.6.
- **G3 — The path is the approuter's job, not the domain's.** You cannot get `/aps/apps/<app>` from the Custom Domain service alone. If someone "just adds a custom domain" and expects the path, it won't appear. Approuter `xs-app.json` owns it.
- **G4 — App name in the URL = repo registration name.** With the generic rewrite, the `<app>` segment is the app's `sap.app.id`. Namespaced ids show up in the URL. Use vanity aliases (A.3 variant) if you need the segment decoupled from the id.
- **G5 — DNS CNAME target is region-specific.** Don't copy a target from a blog or another subaccount; use the value the Manager shows you.
- **G6 — Certificate chain incomplete.** Uploading the server cert without the intermediate chain causes TLS trust failures on some clients. Include the full chain.
- **G7 — Cookies / session on a new host.** A switch of host means new session cookies; clear old ones when testing, and watch `SameSite`/secure-cookie behaviour if you embed apps in iframes elsewhere.
- **G8 — Test on the default domain first.** Bundling Part A and Part B into one deploy multiplies the failure surface. Prove the path on `cfapps.…` first.

---

## 3. End-state checklist

- [ ] HTML5 apps deployed to the repo (app-host), served by a **standalone approuter** bound to app-runtime + xsuaa.
- [ ] `xs-app.json` rewrites `/aps/apps/(.*) → /$1` (or per-app vanity aliases).
- [ ] Apps reachable at `…cfapps…/aps/apps/<app>` **before** custom domain.
- [ ] Custom Domain service entitled; Custom Domain Manager configured with a valid cert (+ full chain) for `apps.companyname.com`.
- [ ] DNS: TXT (verification) + CNAME (`apps` → Manager's target) resolving.
- [ ] CF route `apps.companyname.com` mapped to the approuter app.
- [ ] `redirect-uris` (XSUAA) and IAS redirect/home URLs include the custom-domain URL.
- [ ] `https://apps.companyname.com/aps/apps/app1` loads after login, no 404s.

---

*Compiled for SAP BTP, Cloud Foundry environment. Custom Domain Manager UI, certificate options, CNAME targets, and service plan names are version/region-specific — verify against current SAP Help and the Manager's own output before relying on any literal command or label here.*
