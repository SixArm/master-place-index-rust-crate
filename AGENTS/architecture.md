# Architecture

## System Layers

```
+------------------------------------------------------------------+
|                        Library Crate                              |
+------------------------------+-----------------------------------+
|                               |                                   |
|  +---------------+  +--------v-------+  +-----------------------+ |
|  |   Domain      |  |   Matching     |  |   Validation          | |
|  |   Models      |  |   Algorithms   |  |   & Normalization     | |
|  |               |  |                |  |                       | |
|  | Place         |  | Name (JW)      |  | Field validation      | |
|  | PostalAddress |  | Address (wt)   |  | Address normalization | |
|  | GeoCoords     |  | Geo (Haversine)|  | Title-case, uppercase | |
|  | PlaceType     |  | Identifier     |  |                       | |
|  | Identifier    |  | Phonetic (Sdx) |  +-----------------------+ |
|  | Amenity       |  | Scoring (wt)   |                            |
|  | OpeningHours  |  +----------------+  +-----------------------+ |
|  | Consent       |                      |   Privacy              | |
|  +---------------+                      |   & GDPR               | |
|                                         |                       | |
|                                         | Data masking          | |
|                                         | GDPR export           | |
|                                         +-----------------------+ |
+------------------------------------------------------------------+
```

## Module Dependencies

```
models (no deps)
  ↑
matching (depends on models, strsim)
validation (depends on models)
privacy (depends on models, serde_json)
```

## Data Flow

### Place Matching Flow

1. Two Place structs provided as input
2. Check for GLN deterministic match (short-circuit to 1.0)
3. Compute component scores: name, geo, address, place_type, identifier
4. Apply phonetic (Soundex) bonus if names sound alike
5. Compute weighted average (only for available components)
6. Return MatchResult with score, confidence, and breakdown

### Validation Flow

1. Place provided as input
2. Check required fields (name)
3. Validate field formats (coordinates, GLN, URL, telephone)
4. Validate address completeness
5. Return list of ValidationErrors (empty = valid)

### Normalization Flow

1. Place provided as mutable input
2. Trim name whitespace
3. Title-case locality
4. Uppercase region and country

### Privacy Flow

1. Place provided as input
2. Clone and mask telephone/fax (last 4 chars -> ****)
3. Round geo coordinates to 2 decimal places (~1km precision)
4. Return masked copy (original unchanged)

## Design Decisions

- **Pure library crate**: No external service dependencies for core logic (no DB, no network)
- **schema.org/Place based**: Industry-standard domain model
- **Component-based matching**: Each dimension scored independently, then aggregated
- **Deterministic override**: GLN match bypasses probabilistic scoring
- **Adaptive weights**: Only available components count toward weighted average
- **Phonetic bonus**: +5% for Soundex matches below 0.95 threshold
