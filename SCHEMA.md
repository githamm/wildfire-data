# Wildfire data schema

This repo (`githamm/wildfire-data`) publishes the JSON files the Wildfire
Tracker app fetches client-side from `raw.githubusercontent.com`. The two
repos have no shared type system and no CI linking them, so this doc is the
contract between them: if you change a field here without checking the
consumer, the app breaks silently at runtime, not at merge time.

## Files

### `last-updated.json`
```json
{ "generatedAt": "2026-07-22T23:04:09Z" }
```
Written last, only if every fetch/archive step succeeded. A stale value on
the site is a real pipeline-health signal, not cosmetic.

### `map_data.json`
GeoJSON `FeatureCollection`, US-wide, current WFIGS incidents (`WF`/`CX`
types only — `RX` prescribed burns excluded). Raw NIFC field names
(`IncidentName`, `PercentContained`, `POOState`, `IncidentSize`, ...).

**Consumed by:** `map.js` (primary), `table.js` (cross-referenced by name +
`POOState === 'US-CO'` to flag which List-tab rows are still active).

### `table_data.json`
GeoJSON `FeatureCollection`, Colorado year-to-date, `WF` only. Raw NIFC
field names. Client-side filtered to 10+ acres (List tab) / 1,000+ acres
(Chart tab).

**Consumed by:** `table.js`, `chart.js`.

### `fire-index.json`
Object keyed by `attr_UniqueFireIdentifier`. One entry per fire ever
archived — entries for inactive fires are kept, not deleted, so past fires
stay browsable in the Perimeters tab picker.

```json
{
  "2026-COCUX-001160": {
    "name": "Aspen Acres",
    "cause": "Human",
    "city": "Aspen",
    "landownerCategory": "Private",
    "county": "Pitkin",
    "discoveryDate": "2026-07-15T18:22:00Z",
    "firstSeen": "2026-07-15",
    "lastSeen": "2026-07-22",
    "active": true,
    "acres": 101546
  }
}
```
`acres` is the **latest known value**, not a running peak — this replaced
the old `peakAcres` field (one-time migration in `archive_perimeters.py`;
a stale running-max used to drift out of sync with `map_data.json`/
`table_data.json`, which is what caused the Aspen Acres card-lag bug).

**Consumed by:** `perimeters.js` (fire picker/dropdown).

### `perimeter-history/<safeId>.json`
Array of daily snapshots for one fire, oldest to newest. `safeId` is the
fire ID with `[^A-Za-z0-9_-]` replaced by `_`.

```json
[
  {
    "date": "2026-07-21",
    "name": "Aspen Acres",
    "cause": "Human",
    "acres": 101546,
    "percentContained": 61,
    "personnel": 412,
    "complexity": "Type 2",
    "city": "Aspen",
    "landownerCategory": "Private",
    "county": "Pitkin",
    "discoveryDate": "2026-07-15T18:22:00Z",
    "geometry": { "type": "MultiPolygon", "coordinates": [ ... ] }
  },
  {
    "date": "2026-07-22",
    "name": "Aspen Acres",
    "cause": "Human",
    "acres": 101546,
    "percentContained": 63,
    "personnel": 388,
    "complexity": "Type 2",
    "city": "Aspen",
    "landownerCategory": "Private",
    "county": "Pitkin",
    "discoveryDate": "2026-07-15T18:22:00Z",
    "geometry": null,
    "geometrySameAsDate": "2026-07-21"
  }
]
```

Every field except `geometry` reflects **that specific day** — an old
entry doesn't inherit today's value if, say, `cause` is later reclassified.

**`geometry` is nullable.** When a day's mapped perimeter is identical to
the most recent prior day's (common once growth stalls but containment
work continues), it's stored as `null` with a `geometrySameAsDate`
pointing at the entry that has the real one. This is the single biggest
lever on file size — don't "fix" it by making geometry non-nullable again
without re-checking the size math.

> **Invariant:** `geometrySameAsDate` always points **directly** at an
> entry with literal geometry, never at another reference. A consumer
> only ever needs one lookup, never a chain walk. If a future pipeline
> change breaks this (e.g. reference chains), existing consumers will
> silently render nothing for the second-hop day, since they don't
> chain-resolve.

**Geometry precision:** coordinates are rounded to 5 decimal places
(~1.1m) and simplified via Douglas-Peucker at a 5m max-deviation tolerance
(measured in real meters, not raw degrees) before being written. Don't
expect sub-5m accuracy, and don't re-simplify client-side expecting
further gains — this is already near the practical floor for this map's
zoom levels.

**Consumer contract:** any code reading this file MUST resolve
`geometry: null` + `geometrySameAsDate` before use. `perimeters.js`'s
`resolveGeometryReferences()` does this once, immediately after fetch. A
new consumer that skips this will render a blank map for every deduped
day.

## Making a schema change here

Changing a field name, adding/removing a field a consumer relies on, or
changing the null/reference semantics above requires checking the app
repo by hand — grep it for whatever you're touching (`.geometry`,
`.geometrySameAsDate`, `.acres`, etc.) before merging, since nothing else
will catch a mismatch. Land the pipeline change and the consumer change
together, even across two repos/PRs, rather than one after the other.
