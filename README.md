# ACOP Materials Database

A decision-support database of adhesives and explicitly classified photopolymers for
photonic packaging, with field-level provenance for every value.

The whole application is **one self-contained HTML file**:
`index.html`. Open it in a browser. There is no build step, server-side runtime, or
installed dependency. The Database Insights charts dynamically load Chart.js from a
CDN, and the optional AI advisor makes network requests using a user-supplied API key;
the rest of the application runs locally. Data, UI, provenance engine and test suite
all live in that file.

The database's purpose is to be *trustworthy*, not merely comprehensive. A value that
cannot be traced to a document a reader can open is treated as a defect, not as data.
Most of the machinery described below exists to enforce that.

---

## 1. File layout

One file, three regions:

| Region | What it is |
|---|---|
| Head + `<style>` blocks | All CSS. A late override block `#v23-prov-late-css` wins over earlier rules. |
| `const DATA = {...};` | The entire dataset, serialized as **a single line** (~1.2M characters, around line 3534). |
| Everything after | `window.PROV` (provenance core), the UI, and `runRegressionTests()`. |

### `DATA` keys

- `DATA.adhesives[]` — one object per material (the historical array name is retained for
  compatibility). `material_class` separates adhesives from photopolymers. Property values live in flat text fields
  (`tg_c`, `refractive_index`, …) with optional numeric mirrors (`tg_c_min`, `tg_c_max`, …).
  Citations live in `a.provenance[propertyKey]`.
- `DATA.evidence[]` — the source registry. `legacy_source_code` is the stable source id.
- `DATA.source_aliases` — `{retired_id: canonical_id}`, so a merged-away source id still resolves.
- `DATA.sources_reviewed_not_merged` — pairs that look like duplicates but are deliberately kept apart.

---

## 2. The provenance model

This is the part to understand before changing anything.

```js
PROV.resolvePropertySources(adhesive, 'tg_c')
// → { status, evidence_status, sources[], refs[], missingRefs[], confidence, ... }
```

`status` is one of:

| status | meaning |
|---|---|
| `resolved` | value shown, backed by an explicit or in-text citation that resolves |
| `partially_resolved` | only weak attribution (keyword/note level), or some refs missing |
| `unresolved` | a value is displayed but nothing behind it resolves — a defect |
| `not_reported` | no value; nothing is displayed |

A citation resolves through one of four tiers, recorded as `via`:

- `explicit` — id listed in `provenance[key].source_refs`
- `text` — id written inside the stored value text, e.g. `"6300 MPa (S01-B)"`
- `note` — id appearing in the provenance notes/method
- `supports` — the source's `key_data_supported` names this property (weakest; keyword match)

Source tiers: `independent`, `manufacturer`, `manufacturer_affiliated`, `aggregator`,
`secondary`. **Datasheet aggregators republish manufacturer data and are not independent.**

Every canonical source also carries a dated link audit. Access is classified separately
as reachable public URL, resolvable DOI, or an exact manufacturer portal document. Source
records that cannot be retrieved by a database user are not retained. Two further
fields record whether document identity and cited-property support were verified from
content or remain pending. Reachability may count toward "publicly checkable", but it is
not content verification.

---

## 3. Invariants — do not break these

These encode decisions already made, usually after finding a real defect. Breaking one
silently reintroduces a class of error the database has already been cleaned of.

1. **Never fabricate a value or a citation.** If a document does not state something,
   the answer is "not reported", not a plausible number.
2. **No confidence without evidence.** A value whose sources do not resolve must never
   display a confidence level.
3. **No orphan numerics.** When a property is withdrawn or not reported, its numeric
   mirror fields must be nulled — otherwise the value leaks back into scoring, tiles
   and comparison even though the UI hides the text.
4. **Manufacturer and aggregator sources are not independent.** Do not label a value
   peer-reviewed or third-party unless a genuinely independent source resolves for it.
5. **Retired sources still resolve, but cannot support by keyword.** A retired record
   keeps its id resolvable (so old citations and exports still work) but is excluded
   from the `supports` tier — this is what stopped a family-level chart from silently
   backing 63 product-level numbers.
6. **Transmission needs a path length.** A transmission percentage with no stated
   sample thickness is marked "Cannot normalize to dB/cm — path length not reported"
   and must not be used as a comparable number. When wavelength, percentage, and path
   length are all reported, the UI derives apparent attenuation as
   `-10 log10(T/100) / thickness_cm`, preserves the source value, and reverses bounds.
   It is not labelled bulk attenuation unless interface losses were removed.
7. **Controlled-vocabulary fields must hold canon values** (`PROV.VOCAB`). These are not
   cosmetic: the AI advisor's safety rules match these strings exactly, and the card
   styling switches on them. Prose in a categorical field silently disables both.
8. **Absence in an extraction is not absence in the document.** Never withdraw an
   existing value because a re-read failed to capture it; keep it and flag it for
   re-check. (This rule exists because violating it destroyed a correct haze value.)
9. **Never collapse different test types into one numeric range.** A lap-shear and a
   compression-shear figure must not become a single min–max for ranking.

---

## 4. Validating a change

Run in the browser console:

```js
runRegressionTests()   // 129 checks
PROV.intakeReport()    // per-record errors/warnings
PROV.audit()           // field-level integrity report
PROV.auditJSON()       // same, exportable
```

**Current: 129 pass, 0 fail, 0 partial.** Compare against that baseline; any new
non-passing check is a regression.

### Intake codes

Errors mark a record **Provisional** — it stays visible and searchable, is badged, and
is withheld from ranking.

| Code | Meaning |
|---|---|
| E0 | no material name |
| E1 | a displayed value with no citation that resolves |
| E2 | citation names a source id not in the registry |
| E3 | confidence claimed where nothing resolves |
| E4 | withdrawn/unreported property still carries a number |
| E5 | a cited source has no usable title |
| E6 | record has no source at material level |
| E7 | product identity unverified (grade name not found in the maker's catalogue) |

| Code | Warning |
|---|---|
| W0 | no manufacturer recorded |
| W1 | property matched to a source by keyword rather than an explicit ref |
| W2 | no publicly checkable source |
| W3 | transmission with no stated thickness |
| W4 | independence claimed but no independent source resolves |
| W5 | citation names a retired id (resolves via the alias table) |
| W6 | a field holds a value outside its controlled vocabulary |

---

## 5. Current state (2026-09-03)

| | |
|---|---|
| Materials | 121 (112 adhesives + 9 structural photopolymers) |
| Populated properties | 764 |
| Traceable properties | 764 / 764 (100%) |
| Fully resolved / partially resolved citations | 764 / 0 |
| Unresolved values | **0** |
| Publicly checkable | 752 / 764 (98.4%) |
| Source records | 169 (16 alias ids still resolving) |
| Public reachability | 150 reachable URLs / 16 resolvable DOIs / 166 total |
| Content verification | 25 document identities verified / 20 property-support scopes verified / 143 pending |
| Other source classifications | 3 manufacturer-portal / 0 local / 0 private / 0 placeholders / 0 unavailable |
| Property access | 752 public / 12 exact-portal / 0 inaccessible-only |
| Records clean / warnings only / provisional | 73 / 40 / 8 |
| Vocabulary violations | 0 |

### External polymer research source-use policy

Six discovery resources are listed in the Evidence Repository without adding material
records or property evidence: [IBM Materials Discovery projects](https://research.ibm.com/projects?tag=materials-discovery),
[IBM Materials Discovery topic](https://research.ibm.com/topics/materials-discovery),
[IBM PatCID](https://research.ibm.com/blog/patcid-tool-for-accelerating-materials-discovery),
[MIT polymer-property guide](https://libguides.mit.edu/properties-bymaterial/poly),
[NIMS PoLyInfo](https://polymer.nims.go.jp/), and
[Chicago 3PDB](https://pppdb.uchicago.edu/).

Discovery/index pages never support a property by themselves. PoLyInfo evidence requires
the exact sample, value, units, conditions and underlying publication; registration is
required and scraping or mass downloading is prohibited. 3PDB literature values require
their underlying DOI, while calculator/model output remains computational. PatCID results
are patent-search leads, not validated polymer products; cite the originating patent.
IBM generative outputs remain predicted candidates and cannot raise experimental confidence.
SPG-TED289M's one million pretraining samples are serialized polymer graphs, not one
million commercially available, experimentally characterized polymers.

Six provisional records are the Nye SmartGel `OCK-*` grades, flagged E7 deliberately:
none of those grade names exists in Nye's catalogue. They were **not** renamed to real
grades, because matching on refractive index alone would be a fabricated identification.

---

## 6. Known gaps — the live worklist

1. **37 stub records** (≤2 populated properties): Luvantix 9, DELO 8, Master Bond 6,
   Dymax 4, Henkel 3, Norland 1, plus the 6 Nye records that are correctly held back.
2. **Inaccessible source records are not retained.** Private, broken, unavailable, and
   placeholder records were removed. Values that depended solely on them were withdrawn;
   12 claims remain supported by three exact DELO TDS files whose identities and values
   were checked and whose current retrieval path requires registration and a product-code
   search in DELO's manufacturer portal.
3. **NTT-AT revision discrepancy** — the operator's own copy of the waveguide brochure
   is print code `202201B`; the public link cited is `201802A`. Stored values came from
   the newer copy, so the public link is a different revision. A companion fiber-array
   brochure (E2, 202201B) is not yet in the registry.
4. `retrieved_date` remains historical acquisition metadata; the separate `link_audit`
   object now records the 2026-09-03 check outcome and final redirect target for every
   source. Panacol's URL scheme and Henkel's `en_US` paths have both changed between builds — rot here is measured
   in months.
5. **EPO-TEK 353ND transmission (≥95%, 1100–1600 nm) has no published path length** in
   any revision; it is marked not normalized and must not enter a loss budget until
   Meridian supplies the test geometry.
6. **The polymer branch remains evidence-bounded:** the three baseline records plus
   Nanoscribe IP-n162, Nanoscribe IPX-Clear, UpNano UpOpto, UpNano UpSol, UpNano UpBrix 2,
   and micro resist technology OrmoComp are classified as structural photopolymers, not
   adhesives, and are excluded from adhesive recommendations. Their published sources do
   not establish an adhesive bond. No exact IBM materials dataset or formulation record
   was identifiable; an IBM relationship requires the feedback provider's exact database
   name or URL.

---

## 7. Editing the data safely

`DATA` is a single ~1.2M-character line. **Do not hand-edit it** and do not paste values
back from truncated tool output. Parse, mutate, re-serialize:

```python
import json, re
p = 'index.html'
html = open(p, encoding='utf-8').read()
m = re.search(r'^const DATA = (\{.*\});$', html, re.M)
data = json.loads(m.group(1))

# ... mutate data ...

blob = json.dumps(data, ensure_ascii=False, separators=(',', ':'))
assert '</script' not in blob          # must not break out of the script tag
open(p, 'w', encoding='utf-8').write(html[:m.start()] + 'const DATA = ' + blob + ';' + html[m.end():])
```

Then reload the file and re-run `runRegressionTests()` against the baseline in §4.

When you change a value, record what it was. The convention is a `verification` object
on the provenance entry (`{step, date, action, sources[], previous_value}`) plus a
sentence in `notes` explaining the evidence. Never overwrite a value silently.

Inline registry codes (`S01-B`) in stored text are rendered as named, openable source
links at display time by `PROV.linkifyCodes()`. **The stored text is intentionally left
unchanged** — the code is one of the resolution tiers and is the record of what was
actually cited. Change the presentation, not the data.
