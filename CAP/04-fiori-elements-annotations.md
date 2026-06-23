# Fiori Elements Annotation Syntax Reference

The UI layer: annotations that drive SAP Fiori elements to render a **List Report** and **Object Page** from your service metadata — no UI code. Third in the series after schema and service modeling. Summarized but complete, CAP CDS flavor.

The model to hold: Fiori elements is a **generic frontend**. It reads annotated metadata at runtime and builds the pages. You're not coding a UI — you're *describing* one. Annotations are that description. Same data, different annotations = different app.

Put these in a separate `app/<app>/annotations.cds` file, `annotate Service.Entity with @(…)` — keep them out of `db/` and `srv/`.

---

## CAP shortcuts (the cheap wins first)

Before the heavy vocabulary, CAP maps several core annotations to the right UI terms for you:

```cds
entity Inspections {
  assetID   : String @title: 'Equipment';      // → field label
  status    : String @readonly;                 // → not editable
  inspector : String @mandatory;                // → required + validation
}
```

`@title`, `@readonly`, `@mandatory`, `@description` get you labels and basic field control without touching `@UI.*` / `@Common.*`. Use them; reach for the full vocabulary only when these don't reach.

---

## List Report — the four annotations

A List Report = a filter bar + a table + a default sort/filter. Four terms build it.

### LineItem — the table columns

```cds
annotate InspectionService.Inspections with @UI.LineItem: [
  { Value: assetID,     Label: 'Equipment' },
  { Value: inspector },
  { Value: status,      Criticality: statusCriticality },   // colored status
  { Value: performedAt }
];
```

Order of array = order of columns. `Criticality` colors the cell (see below).

### SelectionFields — the filter bar

```cds
annotate InspectionService.Inspections with @UI.SelectionFields: [ status, assetID, performedAt ];
```

Which fields appear as filters above the table. Everything else is still filterable via "Adapt Filters," just not shown by default.

### PresentationVariant — default sort + which visualization

```cds
annotate InspectionService.Inspections with @UI.PresentationVariant: {
  SortOrder      : [ { Property: performedAt, Descending: true } ],
  Visualizations : [ '@UI.LineItem' ]
};
```

The default ordering when the list loads, and a pointer to the table (or a chart) to render.

### SelectionVariant — saved filter tabs

```cds
annotate InspectionService.Inspections with @UI.SelectionVariant #Open: {
  Text          : 'Open',
  SelectOptions : [ {
    PropertyName : status,
    Ranges       : [ { Sign: #I, Option: #EQ, Low: 'submitted' } ]
  } ]
};
```

Pre-canned filter sets — surfaces as tabs/segmented views ("Open / All"). The `#Open` is a qualifier; you can define several.

---

## Object Page — the structure annotations

An Object Page = a header (key info) + sections (facets) holding field groups and child tables.

### HeaderInfo — the header identity

```cds
annotate InspectionService.Inspections with @UI.HeaderInfo: {
  TypeName       : 'Inspection',
  TypeNamePlural : 'Inspections',
  Title          : { Value: assetID },
  Description    : { Value: status }
};
```

What the top of the page says — object type, and the title/subtitle fields.

### FieldGroup — a labelled cluster of fields

```cds
annotate InspectionService.Inspections with @UI.FieldGroup #General: {
  Data: [
    { Value: assetID },
    { Value: inspector },
    { Value: performedAt },
    { Value: status }
  ]
};
```

A reusable block of fields. The `#General` qualifier names it so a Facet can target it.

### Facets — the sections/tabs

```cds
annotate InspectionService.Inspections with @UI.Facets: [
  {
    $Type  : 'UI.ReferenceFacet',
    Label  : 'General',
    Target : '@UI.FieldGroup#General'        // point at the field group above
  },
  {
    $Type  : 'UI.ReferenceFacet',
    Label  : 'Checklist Items',
    Target : 'items/@UI.LineItem'            // a child entity's table, inline on the page
  }
];
```

Facets are the page's spine: each one renders a referenced annotation. `Target: 'items/@UI.LineItem'` is how you show child line items (your inspection checklist) as a section on the parent page — this is why composition + redirected associations matter.

### Identification — header fields & actions

```cds
annotate InspectionService.Inspections with @UI.Identification: [
  { Value: assetID },
  { $Type: 'UI.DataFieldForAction', Action: 'InspectionService.submit', Label: 'Submit' }
];
```

Fields and **action buttons** that sit in the header area.

---

## DataField `$Type` variants (cells & buttons)

Inside `LineItem`, `FieldGroup`, `Identification`, each entry has a `$Type`. The default is a plain value; the others give you buttons and links:

| `$Type` | Renders |
|---|---|
| `UI.DataField` (default) | a plain value — can omit `$Type` |
| `UI.DataFieldForAction` | button calling a service **action** (your `submit`) |
| `UI.DataFieldForIntentBasedNavigation` | button navigating to another app |
| `UI.DataFieldWithUrl` | value shown as a hyperlink |
| `UI.DataFieldForAnnotation` | embeds another annotation (Contact card, chart, connected fields) |

```cds
{ $Type: 'UI.DataFieldForAction', Action: 'InspectionService.submit', Label: 'Submit' }
{ $Type: 'UI.DataFieldWithUrl',  Value: assetID, Url: assetSheetUrl }
```

This is where your bound actions become buttons — the service-layer `action submit()` surfaces here as a clickable thing.

---

## Common vocabulary essentials

### Label, Text, TextArrangement

```cds
annotate InspectionService.Inspections with {
  status @Common.Label: 'Inspection Status';
  status @Common.Text: { $value: statusText, ![@UI.TextArrangement]: #TextOnly };
};
```

`Text` shows a readable description in place of (or alongside) a code. `TextArrangement`: `#TextOnly`, `#TextFirst`, `#TextLast`, `#TextSeparate` — controls "Submitted" vs "S (Submitted)" vs "Submitted (S)".

### SemanticKey — what identifies a row to the user

```cds
annotate InspectionService.Inspections with @Common.SemanticKey: [ assetID ];
```

The human-facing key (distinct from the technical `ID`). Drives the draft indicator and clean row identity in the table.

### ValueList — F4 / value help

```cds
annotate InspectionService.Inspections with {
  status @Common.ValueList: {
    CollectionPath : 'StatusCodes',
    Parameters     : [
      { $Type: 'Common.ValueListParameterInOut',        LocalDataProperty: status, ValueListProperty: 'code' },
      { $Type: 'Common.ValueListParameterDisplayOnly',  ValueListProperty: 'text' }
    ]
  };
};
```

Wires a dropdown/value-help dialog to a code-list entity. `InOut` binds the picked value back; `DisplayOnly` shows extra context columns.

### SideEffects — refresh on change

```cds
annotate InspectionService.Inspections with @Common.SideEffects #onStatus: {
  SourceProperties : [ status ],
  TargetProperties : [ 'isLocked' ],
  TargetEntities   : [ items ]
};
```

Tells Fiori "when `status` changes, re-fetch these fields/children" — without it, derived fields and child tables go stale after an edit.

---

## Status colors & KPIs

### Criticality (the traffic lights)

```cds
{ Value: status, Criticality: statusCriticality }
```

`statusCriticality` is an element returning: `0` neutral/grey, `1` red/negative, `2` orange/critical, `3` green/positive, `5` new/blue. Compute it in the projection (a `case` expression) and reference it here. A failed inspection going red without writing UI code is exactly this.

### DataPoint — a KPI / rating / progress

```cds
annotate InspectionService.Inspections with @UI.DataPoint #compliance: {
  Value         : complianceScore,
  TargetValue   : 100,
  Visualization : #Progress       // or #Rating, #Number
};
```

A single highlighted metric — header KPIs, ratings, progress bars.

---

## Field control & visibility

```cds
annotate InspectionService.Inspections with {
  internalRef @UI.Hidden;                          // never shown
  reason      @UI.Hidden: (status = 'submitted');  // conditionally hidden (path/expression)
  remarks     @Common.FieldControl: #Optional;     // #Mandatory / #ReadOnly / #Optional / #Hidden
};
```

Static (`@UI.Hidden: true`) or dynamic (bind to a field/expression). `@Common.FieldControl` can be **dynamic too** — drive mandatory/read-only by data, e.g. a field becomes read-only once the inspection is submitted.

---

## Capabilities (what the service permits)

```cds
annotate InspectionService.Inspections with @Capabilities: {
  InsertRestrictions : { Insertable: true },
  DeleteRestrictions : { Deletable: false },
  FilterRestrictions : { NonFilterableProperties: [ internalRef ] },
  SortRestrictions   : { NonSortableProperties: [ remarks ] }
};
```

Declares which CRUD/filter/sort operations are allowed — Fiori greys out the buttons accordingly. Belt-and-suspenders with your `@restrict` auth (capabilities shape the UI affordances; `@restrict` enforces).

---

## Charts (brief)

```cds
annotate InspectionService.Inspections with @UI.Chart #byStatus: {
  ChartType  : #Donut,
  Dimensions : [ status ],
  Measures   : [ total ]
};
```

For the Analytical List Page or a header chart facet. Pair with an aggregated `as select from` entity from the service layer.

---

## How the floorplan assembles (the map)

| You want | Annotation(s) |
|---|---|
| Filter bar | `@UI.SelectionFields` |
| Table columns | `@UI.LineItem` |
| Default sort / view | `@UI.PresentationVariant` |
| Filter tabs | `@UI.SelectionVariant` |
| Object Page header | `@UI.HeaderInfo` + `@UI.Identification` |
| Page sections | `@UI.Facets` → `@UI.FieldGroup` / child `@UI.LineItem` |
| Buttons | `UI.DataFieldForAction` inside LineItem/Identification |
| Dropdowns | `@Common.ValueList` |
| Status colors | `Criticality` on a DataField |
| Required/read-only by data | `@Common.FieldControl` (dynamic) |

List Report needs the first four; Object Page needs HeaderInfo + Facets + FieldGroups. Everything else is enrichment.

---

## Gotchas

- **No `@odata.draft.enabled`, no editable Object Page.** Set it at the service layer (covered in the service-modeling reference) or the page is read-only.
- **Facet `Target` must point at a real annotation** with the matching qualifier (`@UI.FieldGroup#General`). Typos here fail silently — empty section, no error.
- **Child tables on the parent page need the association exposed and redirected** in the service. If `items/@UI.LineItem` shows nothing, check service-layer redirection first.
- **`Criticality` needs a value source** — it's not magic; it reads an element (or a `CriticalityCalculation`). Compute it in the projection.
- **Frontend annotations override backend.** An XML annotation in the UI app's `manifest.json` beats your CDS term. SAP's own guidance: keep annotations in the backend (CDS) unless you have a reason not to — fewer places to chase a bug.
- **Don't remove exposed OData properties in a live service** — it's a breaking change for any UI bound to them.

---

## Authoritative sources — what each is actually for

Annotations are a huge, monthly-evolving vocabulary. Don't memorize it — bookmark these and look up the term:

1. **Feature Showcase (CAP CDS) — your primary reference.** A single List Report + Object Page app demonstrating most features, each with the exact CDS annotation and a short explanation. Because you're on CAP, this is the one to live in: `https://github.com/SAP-samples/fiori-elements-feature-showcase`. Want a feature? Find it here, copy the annotation.

2. **capire — Serving Fiori UIs.** The CAP-specific guide: where annotations go, the local Fiori preview, draft handling. `https://cap.cloud.sap/docs/advanced/fiori`. This is your "how does this work *in CAP*" source.

3. **UI5 documentation (the dictionary).** The comprehensive per-feature reference, with the **Feature Map** (every feature, V2 vs V4 differences) and the **Flexible Programming Model Explorer** (a live sandbox to see annotation effects). `https://ui5.sap.com` → Fiori elements. The exhaustive source of truth; a daily reference once you're seasoned, too much as a first read.

4. **OData Vocabularies (term definitions).** The canonical XML/markdown defining every `UI.*`, `Common.*`, `Communication.*` term and its allowed values — when you need to know exactly what a property does or accepts: `https://github.com/SAP/odata-vocabularies`.

5. **SAP Fiori tools (don't hand-write everything).** The VS Code / BAS extension — **Page Map** and **Guided Development** generate and edit these annotations visually, with validation. For complex annotations (value helps, facets), let the tooling scaffold and learn from what it writes.

Practical loop: scaffold with Fiori tools or copy from the Feature Showcase → understand the term in the UI5 docs / vocabularies → see the effect instantly via CAP's local Fiori preview (`cds watch` → the preview link).

---

*Fiori elements ships monthly; annotation behavior and new terms change. Verify against the sources above for your UI5 version — especially anything involving draft, charts, or the newer building-block / flexible-programming-model features.*
