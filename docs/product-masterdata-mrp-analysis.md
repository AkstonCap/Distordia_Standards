# Product Masterdata vs. Industrial MRP Needs — Gap Analysis & Production-Readiness Proposal

> **Scope:** Critical review of the [Distordia Product Masterdata Standard](../standards/product-standard.json)
> (`distordia-type: "product"`, v1.0.0) measured against the requirements of a **decentralized,
> B2B, common Material Requirements Planning (MRP) system** spanning a wide range of product
> categories.
>
> **Verdict in one line:** v1 is a competent *product catalog* record, but it is **not yet an
> MRP master-data standard**. MRP needs structures the current schema simply does not have
> (bills of material, planning parameters, sourcing, multi-plant context, units-of-measure
> conversions), and its single-flat-1KB-asset shape cannot carry them. This document explains
> the gaps, walks through industry-by-industry requirements, and proposes a modular `product.v2`
> redesign that fits the Nexus blockchain constraints.

---

## 1. Executive Summary

The v1 product standard does three things well:

1. It anchors a product to an **immutable, timestamped, namespace-owned on-chain address** — a
   genuinely useful universal reference.
2. It captures the **identity and logistics basics** (GTIN, manufacturer, dimensions, weight,
   country of origin, HS code, GPC).
3. It is **disciplined about the 1KB / typed-field / no-nested-object** constraints of Nexus.

But "a universal product reference with logistics attributes" is roughly the *first 10%* of what
an MRP system consumes. MRP is fundamentally a **time-phased netting engine**:

```
   Net Requirement = Gross Requirement (demand) − On-hand − Scheduled Receipts
   ... then exploded down the Bill of Materials, offset by lead time,
       rounded to lot-sizing rules, and routed to make-or-buy.
```

Every italicised term above maps to master data that **v1 does not model**: demand linkage, BOM,
lead-time offset, lot-sizing, make-vs-buy. Without them the register is a catalog, not an MRP
backbone. The remaining gaps — multi-plant context, UOM conversions, sourcing/supplier records,
costing, compliance depth, classification breadth, and data-governance for *multi-party* editing —
are what separate "demo-grade" from "production-grade" for a **common** (shared, B2B) register.

The good news: the ecosystem already contains the architectural pattern needed to fix the
biggest structural problem. The [Article standard](../standards/article-standard.json) solves the
1KB ceiling with a **linked chain of assets**. The same idea — a small immutable *core* asset plus
address-linked *extension* assets — turns the product register from a flat record into a
normalized, composable master-data graph. Section 6 specifies that redesign.

---

## 2. What an MRP System Actually Requires From Master Data

For reference, here is the master-data surface that mature MRP/ERP systems (SAP `MARA`/`MARC`/
`MBEW`/`MAST`/`STPO`, Oracle Item Master, Dynamics 365 SCM, Infor, NetSuite) treat as
**mandatory** before a material can be planned:

| MRP concern | Master data required | In v1? |
|---|---|---|
| **What it is** | Material/part number, description, material *type* (raw, semi-finished, finished, packaging, MRO, service) | Partial — no material *type* |
| **How it's measured** | Base UOM **+ alternative UOMs with conversion factors** | ❌ single `uom` only |
| **What it's made of** | **Bill of Materials** (component, qty, scrap %, alternates, effectivity) | ❌ none |
| **Make or buy** | Procurement type (in-house / external / both) | ❌ none |
| **When to plan** | Planned delivery (purchasing) lead time, in-house production time, GR processing time | ❌ none |
| **How much to order** | MRP type, lot-sizing procedure, min/max/fixed lot, rounding value, reorder point, safety stock | ❌ none |
| **Where from** | Source list / approved vendors, supplier part numbers, MOQ, price breaks, incoterms | ❌ none (only `mfr` name) |
| **Where it lives** | Plant / site / storage-location context (the *same* part is planned differently per plant) | ❌ flat, single-context |
| **What it costs** | Standard/moving price, currency, price unit, valuation class | ❌ none |
| **Quality & batch** | Batch-managed flag, serial profile, inspection setup, certificates, shelf life | Partial — only `perish`, `shelf-days` |
| **Compliance** | Hazmat (UN number / class / packing group), RoHS/REACH, conflict minerals, certifications | ❌ only a boolean `hazard` |
| **Versioning** | Engineering revision / ECN with effectivity dates, supersession | Partial — `replaces`/`replaced-by` only |
| **Classification** | UNSPSC / eCl@ss / ETIM / GPC + flexible characteristic-value attributes | Partial — one `gpc`, one `cat`/`subcat` |
| **Demand & supply** | Links to forecasts, sales orders, on-hand stock, open POs / production orders | ❌ none (out of scope of a register, but must be referenceable) |

The right-hand column is the gap analysis in miniature. The rest of this document expands it.

---

## 3. The Current Standard at a Glance

`product-standard.json` v1 defines **24 fields** in a single JSON-format Nexus asset
(~820 bytes of usable space):

- **Identity:** `art-nr`, `gtin`, `mfr`, `brand`, `desc`
- **Classification:** `cat`, `subcat`, `gpc`, `hs`
- **Logistics:** `origin`, `uom`, `weight-kg` (grams), `length/width/height-mm`
- **Handling:** `hazard` (bool), `perish` (bool), `shelf-days`
- **Lifecycle:** `status`, `replaces`, `replaced-by`
- **Media:** `url`, `img`

Mutability is binary per field (`mutable: true/false`), and the asset is owned by a single
namespace. This is a clean **GS1-flavoured trade-item record**. It is *not* a material master.

---

## 4. Gap Analysis — Why v1 Is Not Production-Ready for B2B MRP

### 4.1 Structural / architectural gaps

**G1 — No Bill of Materials. (The single most important gap.)**
MRP *is* BOM explosion. Given "build 100 finished units," MRP must walk the parent→child component
tree, multiply quantities, apply scrap factors, and net each component against stock. v1 has no
parent-child relationship, no component list, no per-line quantity, no scrap, no alternates.
Worse, the Nexus constraint "no arrays, no nested objects" means a BOM cannot live *inside* a
product asset at all — it must be modelled as **separate linked BOM-line assets** (see §6.4).

**G2 — Flat, single-context record conflates "what the product is" with "how I plan it."**
In real MRP the *same* material number carries:
- **Org-level** data (description, weight, GTIN) — stable, manufacturer-authored.
- **Plant/site-level** data (lead time, safety stock, MRP controller, lot size) — different at
  every facility, buyer-authored.
- **Valuation-level** data (price, currency) — finance-authored.

A decentralized B2B system makes this split *more* important, not less: a manufacturer publishes
the immutable identity, while each buyer/distributor overlays their own planning parameters. v1
forces everything into one owner's flat asset, so it cannot represent "Acme's part as planned by
Distributor X in Plant Y."

**G3 — 1KB single-asset ceiling.** Even ignoring BOMs, the breadth of attributes a wide product
range needs (electronics specs, chemical safety data, food nutrition, pharma serialization) cannot
fit in ~820 bytes. v1 has no extension mechanism. The schema must become **composable**.

**G4 — No demand/supply linkage.** A register is not an MRP run, but to *feed* one it must be
referenceable from demand (forecast, sales order) and supply (stock, open PO, production order)
objects. v1 defines no stable cross-reference contract for transactional standards to point at —
beyond the raw asset address, there is no notion of "the planning view" to attach to.

### 4.2 Missing MRP planning fields

**G5 — No procurement type (make / buy / both).** Without it, MRP cannot decide whether to raise a
planned *production order* or a *purchase requisition*. This is a one-field omission with total
impact.

**G6 — No lead times.** Time-phasing is impossible. MRP offsets each requirement backward by the
planned delivery time (buy) or production time (make). Missing entirely.

**G7 — No lot-sizing or netting parameters.** No `safety-stock`, `reorder-point`, `min-lot`,
`max-lot`, `rounding-value`, `lot-size-procedure`, or `mrp-type`. These convert a raw net
requirement into an actual, orderable quantity. Their absence reduces any "MRP" built on v1 to
naive 1:1 reordering.

**G8 — No sourcing / supplier relationships.** `mfr` is a free-text *manufacturer name*, not a
*procurement source*. MRP needs: which supplier(s) can provide this, their **supplier part
number**, MOQ, price breaks, lead time, incoterms, and a preference/quota split across multiple
sources. In a B2B network this is precisely the high-value data — and it is absent.

### 4.3 Identity & interoperability gaps

**G9 — Weak identifier model.** `gtin` is an unvalidated string (no check-digit rule), and there
is no **MPN** (Manufacturer Part Number — the true key in electronics/industrial), no supplier
part number, no internal↔external cross-reference. B2B interoperability *is* identifier
cross-referencing; v1 supports a single GTIN and a single internal `art-nr`.

**G10 — No GS1 packaging hierarchy.** Real trade items have a GTIN *hierarchy*: each / inner-pack /
case / pallet, each with its own GTIN and dimensions. Logistics and MRP order/consume at different
levels. v1 models exactly one level.

**G11 — Shallow classification, no crosswalks.** One `gpc` and free-text `cat`/`subcat`. Industrial
buyers classify by **UNSPSC**, **eCl@ss**, or **ETIM** (electrical), and need a documented
crosswalk to GS1 GDSN and to ERP field names. v1 references GS1/ISO in prose but ships no mapping.

**G12 — No alignment with ISO 8000 (master-data quality).** A "standard of choice" for B2B master
data must speak **ISO 8000-110/115/116** (data quality, identifiers, provenance) and ideally
ISO 22745 (open technical dictionaries). v1 has no provenance, completeness, or accuracy metadata.

### 4.4 Product-domain breadth gaps

**G13 — Fixed 24 fields cannot describe a wide product range.** An electronic component needs
voltage/tolerance/package/RoHS; a chemical needs CAS number/concentration/SDS; food needs
allergens/nutrition/storage temperature; apparel needs size/colour/material/care. v1 has **no
extensible characteristic-value mechanism**, so every new domain either abuses `desc` or cannot be
represented.

**G14 — Compliance modelled as a single boolean.** `hazard: 0/1` is operationally useless. Shipping
or storing hazardous goods legally requires **UN number, hazard class, packing group, flash point,
ADR/IMDG/IATA data**. Likewise there is no RoHS/REACH, conflict-minerals, CE/UL marking,
allergen, or country-of-origin *certificate* support.

### 4.5 Data-governance gaps for a *common* (shared) register

**G15 — Single-owner model is wrong for a common register.** "Common" implies many parties read,
contribute, and *correct* the same material. v1 ties every field to one namespace owner. There is
no concept of **field-level authority** (manufacturer owns identity; each buyer owns their planning
overlay), no **data-steward role**, no **dispute/correction workflow**, and no **data-quality
score**. The namespace standard's tiers/reputation/slashing are the right primitives — but the
product standard does not use them.

**G16 — Naive mutable/immutable flags, no change audit.** Master data legitimately changes
(corrections, reclassification, dimension updates) — but under controlled, audited change, not
free mutation. v1 offers only `mutable: true` (silent overwrite, no history) or `mutable: false`
(frozen forever). Production MRP needs an **append-only revision** model with effectivity dates and
a change reason.

**G17 — No localization.** `desc` is one string in one language. Global B2B trade needs
multi-language descriptions and region-specific data.

### 4.6 Lifecycle / engineering-change gaps

**G18 — No engineering revision / effectivity.** Industrial parts carry a drawing revision and
change orders (ECN/ECO) with **effective-from/to dates**. v1's `replaces`/`replaced-by` capture
only full-part supersession, not in-part revisions or date-effective BOM/spec changes.

---

## 5. Industry-by-Industry Requirements (Wide Product Range)

MRP must serve very different product worlds. The table shows the **critical master-data
attributes per domain**, whether v1 can express them, and the single biggest blocker.

### 5.1 Electronics & electrical components
- **Needs:** MPN (primary key), manufacturer + AVL/AML (approved vendor/manufacturer list),
  lifecycle status (NRND/EOL/active), RoHS/REACH, MSL (moisture sensitivity level), package/case
  code, electrical params (V/I/tolerance/temp range), datasheet, ETIM class, reel/tape packaging.
- **v1 coverage:** GTIN, `mfr`, dimensions only. **No MPN, no AVL, no electrical attributes.**
- **Biggest blocker:** no MPN + no extensible attributes (G9, G13).

### 5.2 Chemicals, paints, lubricants
- **Needs:** CAS/EC number, UN number + hazard class + packing group, SDS/MSDS link, concentration,
  GHS pictograms, flash point, storage temp, REACH registration, batch management, shelf life.
- **v1 coverage:** `hazard` boolean, `shelf-days`, `perish`. **No CAS/UN/hazard-class/SDS.**
- **Biggest blocker:** boolean hazard is non-compliant for real handling/shipping (G14).

### 5.3 Food & beverage
- **Needs:** allergen declarations, nutrition (per-100g), ingredients, storage temperature, best-
  before vs. use-by, GS1 GDSN attributes, lot/batch traceability, origin per ingredient, kosher/
  halal/organic certs, net content + drained weight.
- **v1 coverage:** `perish`, `shelf-days`, `origin`, `weight-kg`. **No allergens/nutrition/temp.**
- **Biggest blocker:** food-safety attributes + multi-level batch traceability (G13, G16).

### 5.4 Pharmaceuticals & medical devices
- **Needs:** NDC / GTIN-14 + **serialization (GS1 SGTIN, DSCSA/FMD)**, lot + expiry, UDI (medical
  devices), storage conditions (cold chain), controlled-substance schedule, dosage form/strength,
  regulatory approval (NDA/CE/FDA), aggregation hierarchy.
- **v1 coverage:** GTIN, `shelf-days`. **No serialization, no UDI, no lot/expiry contract.**
- **Biggest blocker:** serialization & regulated traceability (G10, G14, G16).

### 5.5 Apparel, footwear & textiles
- **Needs:** style/colour/size **variant matrix** (one style → many SKUs), material composition,
  care instructions, season/collection, fit, GS1 GTIN per variant, country of origin + textile
  labeling, sustainability/origin certs.
- **v1 coverage:** one flat SKU. **No variant/parent-style model, no composition.**
- **Biggest blocker:** variant configuration (parent style + characteristic axes) (G13).

### 5.6 Automotive, industrial spares & MRO
- **Needs:** OEM part number + aftermarket cross-references, fitment/application data, supersession
  chains, AML, criticality (for spares stocking), reman/core tracking, serial/lot, drawing
  revision, country of origin, AVL with lead times.
- **v1 coverage:** `replaces`/`replaced-by`, `mfr`, `origin`. **No cross-references, no fitment, no
  AVL/lead time.**
- **Biggest blocker:** cross-reference web + sourcing/lead-time data (G8, G9, G18).

### 5.7 Raw materials, metals & commodities
- **Needs:** grade/spec (e.g., AISI 304, ASTM), form (bar/sheet/coil), dimensional tolerances,
  certificate of analysis / mill cert, **catch-weight / variable UOM** (sold by weight, stocked by
  piece), heat/lot number, density for conversions, commodity code.
- **v1 coverage:** single fixed UOM, weight. **No UOM conversions, no catch-weight, no grade/spec.**
- **Biggest blocker:** UOM conversion + catch-weight + material certs (G4-UOM, i.e. no alt-UOM).

### 5.8 Construction & building materials
- **Needs:** ETIM classification, technical datasheets, DoP (Declaration of Performance / CE),
  coverage rate, pack/pallet quantities, weight per unit area, fire/thermal ratings, batch, lead
  times for made-to-order items.
- **v1 coverage:** dimensions, weight, `gpc`. **No ETIM, no DoP, no coverage/pack hierarchy.**
- **Biggest blocker:** packaging hierarchy + technical-attribute breadth (G10, G13).

### 5.9 Machinery, equipment & engineered (ETO) products
- **Needs:** multi-level BOM, configurable variants, drawing/revision + ECN effectivity, serial
  numbers, long lead-time component sourcing, in-house production routing reference, service/spare-
  parts linkage, warranty.
- **v1 coverage:** none of the structural items. **No BOM, no revision, no routing reference.**
- **Biggest blocker:** BOM + engineering change management (G1, G18).

### 5.10 Digital goods / software / services
- **Needs:** license model, version/build, entitlement, no physical dimensions, service UOM (hour/
  seat/subscription), delivery method.
- **v1 coverage:** assumes a physical good (weight/dimensions). **No license/version/service UOM.**
- **Biggest blocker:** material *type* abstraction so non-physical items are first-class (G2-type).

### Cross-industry summary

| Capability | Elec | Chem | Food | Pharma | Apparel | Auto/MRO | Metals | Constr | Machinery | Digital |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Bill of Materials | – | – | ◐ | ◐ | ◐ | – | – | – | ✔ | – |
| UOM conversions / catch-weight | ◐ | ✔ | ✔ | ◐ | ◐ | ◐ | ✔ | ✔ | ◐ | ✔ |
| Sourcing / AVL / lead time | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ◐ |
| Variant configuration | ◐ | – | – | – | ✔ | ◐ | ✔ | ◐ | ✔ | ◐ |
| Compliance depth (UN/RoHS/UDI…) | ✔ | ✔ | ✔ | ✔ | ◐ | ◐ | ✔ | ✔ | ◐ | – |
| Serialization / batch traceability | ◐ | ✔ | ✔ | ✔ | – | ✔ | ✔ | ◐ | ✔ | – |
| Extensible characteristic attributes | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Engineering revision / effectivity | ✔ | – | – | ✔ | ◐ | ✔ | ◐ | ◐ | ✔ | ✔ |

✔ = critical · ◐ = important · – = minor. **Every column needs at least three capabilities v1
lacks.** The four capabilities that appear in nearly every column — *sourcing/lead time*,
*UOM conversions*, *compliance depth*, and *extensible attributes* — should be the top priorities.

---

## 6. Proposed Improvements — A Production-Ready `product.v2`

The fix is **not** "add 40 fields to the flat asset" (impossible under 1KB) but to **normalize the
model into a small core asset plus address-linked extension assets**, reusing the linked-chain
pattern already proven by the [Article standard](../standards/article-standard.json) and the
trust/stewardship primitives of the [Namespace standard](../standards/namespace-standard.json).

### 6.1 Design principles

1. **Separate identity from planning.** Manufacturer publishes the immutable *core*; each buyer
   attaches their own *planning overlay*. This is what makes the register genuinely *common*.
2. **Composable, not monolithic.** Optional concerns (BOM, sourcing, costing, compliance,
   classification, packaging levels) are separate asset types linked by address.
3. **Append-only change history.** New versions are new assets that `supersede` the prior address;
   the chain *is* the audit trail. Stop overloading `mutable: true`.
4. **Field-level authority via namespaces.** Each extension asset is owned/stewarded by the party
   with authority over that data, with tier/reputation from the namespace standard.
5. **Interoperability first.** Every field carries a documented crosswalk to GS1 GDSN, UNSPSC/
   eCl@ss/ETIM, ISO 8000, and common ERP field names (Appendix A).

### 6.2 Asset family

```
                       ┌───────────────────────────┐
                       │   product (core, v2)       │  ← manufacturer-owned, mostly immutable
                       │   identity + base logistics│
                       └─────────────┬──────────────┘
                                     │ address links
   ┌──────────────┬──────────────┬───┴───────┬──────────────┬───────────────┐
   ▼              ▼              ▼            ▼              ▼               ▼
product-plan   product-source product-cost product-comp product-class   product-pack
(buyer/plant)  (supplier/AVL) (valuation)  (compliance) (UNSPSC/ETIM…)  (GTIN hierarchy)
   │
   ▼
product-bom-line  (one asset per BOM component; chained — solves "no arrays")
```

### 6.3 Proposed core schema (`product.v2`)

Additions to v1 are **bold**; the asset stays well within 1KB by pushing optional data to
extensions.

```json
[
  {"name":"distordia-type","type":"string","value":"product","mutable":false,"maxlength":16},
  {"name":"schema-ver","type":"string","value":"2.0.0","mutable":false,"maxlength":8},
  {"name":"status","type":"string","value":"valid","mutable":true,"maxlength":8},
  {"name":"art-nr","type":"string","value":"","mutable":false,"maxlength":32},
  {"name":"mpn","type":"string","value":"","mutable":false,"maxlength":40},
  {"name":"gtin","type":"string","value":"","mutable":false,"maxlength":14},
  {"name":"mat-type","type":"string","value":"finished","mutable":false,"maxlength":12},
  {"name":"proc-type","type":"string","value":"buy","mutable":true,"maxlength":4},
  {"name":"desc","type":"string","value":"","mutable":true,"maxlength":200},
  {"name":"mfr","type":"string","value":"","mutable":false,"maxlength":64},
  {"name":"brand","type":"string","value":"","mutable":false,"maxlength":32},
  {"name":"base-uom","type":"string","value":"EA","mutable":false,"maxlength":4},
  {"name":"gpc","type":"string","value":"","mutable":false,"maxlength":8},
  {"name":"hs","type":"string","value":"","mutable":false,"maxlength":10},
  {"name":"origin","type":"string","value":"","mutable":false,"maxlength":2},
  {"name":"weight-g","type":"uint32","value":0,"mutable":false},
  {"name":"length-mm","type":"uint32","value":0,"mutable":false},
  {"name":"width-mm","type":"uint32","value":0,"mutable":false},
  {"name":"height-mm","type":"uint32","value":0,"mutable":false},
  {"name":"lifecycle","type":"string","value":"active","mutable":true,"maxlength":8},
  {"name":"rev","type":"string","value":"A","mutable":false,"maxlength":8},
  {"name":"steward","type":"string","value":"","mutable":false,"maxlength":32},
  {"name":"dq-score","type":"uint16","value":0,"mutable":true},
  {"name":"supersedes","type":"string","value":"","mutable":false,"maxlength":64},
  {"name":"ext","type":"string","value":"","mutable":true,"maxlength":256}
]
```

Key changes vs v1:
- **`mpn`, `mat-type`, `proc-type`** — the three single-field omissions that block MRP (G5, G9,
  and the make/buy/type distinction of G2).
- **`base-uom`** replaces `uom`; conversions live in the `product-pack`/UOM extension (G-UOM).
- **`rev` + `supersedes`** — engineering revision and append-only chaining (G16, G18).
- **`steward`, `dq-score`** — data-governance hooks tied to namespace tiers (G15, G12).
- **`ext`** — pipe-separated list of addresses for attached extension assets (the composability
  spine). E.g. `ext: "addr-plan|addr-source|addr-comp|addr-class"`.
- Dropped from core (moved to extensions): `cat/subcat` → classification; `url/img` → media/class;
  `hazard/perish/shelf-days` → compliance; `replaces/replaced-by` → superseded by `rev`/`supersedes`.

### 6.4 Extension asset types

**`product-plan` (plant/buyer planning overlay — the heart of MRP).** Owned by the *buyer*, not the
manufacturer.
```json
[
  {"name":"distordia-type","type":"string","value":"product-plan","mutable":false},
  {"name":"product","type":"string","value":"<core-address>","mutable":false},
  {"name":"plant","type":"string","value":"","mutable":false,"maxlength":16},
  {"name":"mrp-type","type":"string","value":"PD","mutable":true,"maxlength":4},
  {"name":"lot-proc","type":"string","value":"EX","mutable":true,"maxlength":4},
  {"name":"lead-buy-d","type":"uint16","value":0,"mutable":true},
  {"name":"lead-make-d","type":"uint16","value":0,"mutable":true},
  {"name":"gr-proc-d","type":"uint16","value":0,"mutable":true},
  {"name":"safety-stock","type":"uint32","value":0,"mutable":true},
  {"name":"reorder-pt","type":"uint32","value":0,"mutable":true},
  {"name":"min-lot","type":"uint32","value":0,"mutable":true},
  {"name":"max-lot","type":"uint32","value":0,"mutable":true},
  {"name":"round-val","type":"uint32","value":0,"mutable":true},
  {"name":"mrp-ctrl","type":"string","value":"","mutable":true,"maxlength":8}
]
```
*This single extension is what upgrades the register from "catalog" to "plannable" (closes G5–G7).*

**`product-source` (sourcing / AVL — one per supplier).** Closes G8.
```json
[
  {"name":"distordia-type","value":"product-source"},
  {"name":"product","value":"<core-address>"},
  {"name":"supplier","value":"<namespace>"},
  {"name":"supplier-pn","value":"","maxlength":40},
  {"name":"moq","type":"uint32","value":0},
  {"name":"lead-d","type":"uint16","value":0},
  {"name":"price","type":"uint64","value":0},
  {"name":"price-uom","value":"EA"},
  {"name":"currency","value":"USD","maxlength":3},
  {"name":"incoterm","value":"","maxlength":3},
  {"name":"preference","type":"uint8","value":0},
  {"name":"quota-pct","type":"uint8","value":0}
]
```

**`product-bom-line` (one asset per component — solves "no arrays").** Closes G1. BOM lines chain
via `next`, exactly like article chunks.
```json
[
  {"name":"distordia-type","value":"product-bom-line"},
  {"name":"parent","value":"<parent-core-address>"},
  {"name":"component","value":"<component-core-address>"},
  {"name":"qty-milli","type":"uint64","value":0},
  {"name":"comp-uom","value":"EA"},
  {"name":"scrap-bps","type":"uint16","value":0},
  {"name":"alt-group","type":"uint8","value":0},
  {"name":"eff-from","type":"uint64","value":0},
  {"name":"eff-to","type":"uint64","value":0},
  {"name":"next","value":""}
]
```
(`qty-milli` = quantity ×1000 to avoid floats; `scrap-bps` = basis points.)

**`product-comp` (compliance).** Real hazmat + regulatory, closing G14: `un-number`,
`hazard-class`, `packing-group`, `sds-url`, `sds-hash`, `rohs` (0/1), `reach` (0/1),
`cas`, `allergens`, `udi`, `cert-list` (pipe-separated certificate addresses).

**`product-class` (classification).** Multiple coded schemes, closing G11: `unspsc`, `eclass`,
`etim`, `cat`, `subcat`, plus `attrs` (a `key=value;key=value` characteristic string for the
extensible attributes of G13). For deep attribute sets, point to a `raw`-format asset (like the
NexGo rating standard) that can hold nested JSON.

**`product-pack` (GTIN/packaging hierarchy + UOM conversions).** One per packaging level, closing
G10 and the UOM-conversion gap: `level` (each/inner/case/pallet), `gtin`, `qty-of-base`,
`uom`, `to-base-factor`, dimensions, `weight-g`, `catch-weight` (0/1).

### 6.5 Governance & trust model (making it a *common* register)

- **Field-level authority:** core identity must be created by an **L2+ organization** namespace
  (manufacturer/brand owner); `product-plan`/`product-source` overlays may be created by any
  verified buyer namespace and reference the core by address. Readers resolve "the planning view
  for *my* namespace" by filtering extensions on owner.
- **Data-quality score (`dq-score`, 0–1000):** computed from completeness (mandatory fields
  present), validity (GTIN check digit, ISO code membership, HS format), and stewardship tier of
  the author — mirroring **ISO 8000-61** data-quality dimensions.
- **Corrections & disputes:** reuse namespace **reputation/slashing**. A correction is a new
  superseding asset; a disputed record can be flagged via a lightweight `product-flag` raw asset,
  with stake at risk (same mechanic as swarm mission disputes).
- **Provenance:** `steward`, `rev`, `supersedes`, and the create timestamp give ISO 8000-115
  provenance out of the box.

### 6.6 Validation, conformance & reference data (the "production-ready" checklist)

To be *the standard of choice*, v2 should ship more than a schema:

1. **Validation rules** — GTIN/EAN check-digit, ISO 3166 country, ISO 4217 currency, HS-code
   format, GPC/UNSPSC membership, UOM against a published code list.
2. **Reference-data registries** — on-chain (or canonically published) lists for UOM, currency,
   incoterms, hazard classes, classification schemes — so values are *codes*, not free text.
3. **Conformance test suite** — golden example assets per industry (the ten domains in §5) plus
   negative tests, so any implementer can self-certify.
4. **Crosswalk tables** — GS1 GDSN, UNSPSC/eCl@ss/ETIM, ISO 8000, SAP/Oracle field maps
   (Appendix A) for migration and EDI/PRICAT interchange.
5. **Schema governance** — semantic versioning, a deprecation policy, and backward-compatible
   field additions (the `schema-ver` field enables this).

### 6.7 Maturity roadmap (incremental, non-breaking)

| Level | Adds | Unlocks |
|---|---|---|
| **M0 (today, v1)** | flat catalog record | universal product reference |
| **M1** | core v2 + `mpn`, `mat-type`, `proc-type`, `base-uom`, `rev` | correct identity & make/buy |
| **M2** | `product-plan` extension | **first real MRP** (netting + time-phasing + lot-sizing) |
| **M3** | `product-source` + `product-pack`/UOM | multi-source procurement, packaging hierarchy |
| **M4** | `product-bom-line` chains | BOM explosion → manufacturing MRP |
| **M5** | `product-comp` + `product-class` + governance/DQ | regulated industries + *common* trust |

M1–M2 alone move the standard from "catalog" to "minimum viable MRP," and both are additive (the
v1 asset remains valid as the core's ancestor).

---

## 7. Conclusion

The Distordia product standard is a **solid foundation and a poor MRP standard** — not because it
is badly designed, but because it was scoped as a *trade-item catalog* and is being asked to be a
*material master*. The decisive missing pieces are structural: **bills of material, planning
parameters, sourcing, multi-plant context, and UOM conversions**, none of which fit a flat 1KB
asset and none of which v1 attempts.

The path to production is clear and, crucially, **already idiomatic to this ecosystem**: keep the
small immutable core, and use the **linked-extension-asset** pattern (proven by Articles) plus the
**namespace tier/reputation** governance (proven by the identity layer) to compose planning,
sourcing, costing, compliance, classification, packaging, and BOM as separate, individually-owned,
append-only assets. Doing so closes every gap in §4, satisfies every industry column in §5, and —
by separating "what the product is" (manufacturer-owned) from "how I plan it" (buyer-owned) — turns
the register into something a B2B network can genuinely *share*: a **common MRP backbone** rather
than a single owner's catalog.

---

## Appendix A — Interoperability Crosswalk (illustrative)

| Distordia v2 | GS1 GDSN | UNSPSC/eCl@ss/ETIM | ISO | SAP | Oracle |
|---|---|---|---|---|---|
| `gtin` | `gtin` | — | GS1 | `EAN11` | Item Cross Ref |
| `mpn` | `manufacturerPartNumber` | — | — | `MFRPN` | Mfg Part Number |
| `mat-type` | `tradeItemUnitDescriptor` | — | — | `MTART` | Item Type |
| `proc-type` | — | — | — | `BESKZ` | Make/Buy |
| `base-uom` | `baseUnitOfMeasure` | — | ISO 80000 / UN/ECE Rec 20 | `MEINS` | Primary UOM |
| `product-plan.lead-buy-d` | — | — | — | `PLIFZ` | Lead Time |
| `product-plan.safety-stock` | — | — | — | `EISBE` | Safety Stock |
| `product-source.moq` | — | — | — | `BSTMI` | Min Order Qty |
| `product-class.unspsc` | `gpcCategoryCode` (GPC) | UNSPSC / eCl@ss / ETIM | ISO 22745 | `PRDHA` | Category |
| `product-comp.un-number` | `dangerousGoodsUNNumber` | — | UN ADR | — | Hazard Class |
| `dq-score` | — | — | **ISO 8000-61** | — | — |

*(Crosswalk is illustrative; a production release should publish authoritative, versioned mapping
tables per Appendix A scheme.)*
