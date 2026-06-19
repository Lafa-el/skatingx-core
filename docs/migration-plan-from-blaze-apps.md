# Migration Plan from Blaze Skate Apps

## Goal

Move existing Blaze Skate app data toward shared SkatingX Core models without breaking current products.

This is a staged migration plan, not a requirement to rewrite all apps at once.

## Migration Principles

1. Preserve existing production data.
2. Keep legacy IDs in `externalRefs`.
3. Migrate by entity, not by UI screen.
4. Prefer additive writes before destructive rewrites.
5. Validate every migrated document against the JSON Schemas.
6. Keep rollback possible until each product has verified reads from the new shape.

## Phase 1: Inventory

For each app, document:

- Existing collections
- Document shapes
- Required fields
- Optional fields
- Current user ownership model
- Date fields and formats
- Media references
- Known duplicate or stale data

Output:

```text
docs/migrations/inventory-{app-name}.md
```

## Phase 2: Athlete Matching

Create or identify canonical athlete profiles.

Matching priority:

1. Existing stable athlete ID.
2. Explicit user selection.
3. Same owner account plus high-confidence display name match.
4. Manual review.

Do not auto-merge ambiguous athlete profiles.

## Phase 3: Entity Mapping

Map each legacy entity into a core schema:

| Legacy Area | Core Entity |
| --- | --- |
| Training session or task completion | Training record |
| Reflection, note, journal prompt | Journal entry |
| Video review or technical analysis | Analysis session |
| Skates, blades, gear, fitting notes | Equipment record |
| Meet, test, race, game, or event result | Competition result |

Legacy-only fields should go under `extensions.{sourceApp}`.

## Phase 4: Validation

For each migrated document:

1. Validate JSON syntax.
2. Validate against the matching schema.
3. Verify required identity fields.
4. Verify date format is ISO 8601.
5. Verify migrated record has `externalRefs`.
6. Verify parent athlete exists.

## Phase 5: Dual Read or Dual Write

Recommended sequence:

1. Existing app continues reading old data.
2. Migration script writes new SkatingX Core-shaped data.
3. App adds read support for new shape behind a feature flag.
4. App dual-writes old and new shape if needed.
5. App switches default reads to new shape.
6. Old shape becomes read-only.
7. Old shape is archived after verification.

## Phase 6: Cutover

Cutover checklist:

- All active athletes have canonical athlete profiles.
- Recent records validate against schemas.
- Dashboard summaries match record counts within accepted tolerance.
- Firestore indexes exist for production queries.
- Security rules are reviewed for the new paths.
- Export and rollback plan is documented.

## Risk Areas

### Duplicate Athletes

Risk:

- A parent, coach, and athlete may each have separate profile records for the same skater.

Mitigation:

- Use manual review for uncertain matches.
- Keep `externalRefs` for traceability.

### Inconsistent Dates

Risk:

- Legacy apps may use local dates, Firestore timestamps, or strings.

Mitigation:

- Store canonical timestamps as ISO 8601 strings in examples and schema.
- Migration code can convert to Firestore Timestamp at app boundary if needed.

### Privacy

Risk:

- Journal and analysis records may contain sensitive youth athlete data.

Mitigation:

- Preserve visibility fields.
- Keep private notes scoped to owner access.
- Do not make media public by default.

### Cost Growth

Risk:

- Unified platform dashboards may accidentally scan all child records.

Mitigation:

- Maintain athlete summary fields.
- Paginate record lists.
- Use collection group queries only for deliberate cross-athlete views.

## Initial App Mapping

### Blaze Skate Training

Priority:

1. Athlete profiles.
2. Training records.
3. Training summary counters.

### Blaze Skate Journal

Priority:

1. Athlete profiles.
2. Journal entries.
3. Visibility and privacy mapping.

### Blaze Skate Lab and Analysis

Priority:

1. Athlete profiles.
2. Analysis sessions.
3. Media references and technical metrics.

## Commit Message Template

Use Conventional Commits:

```text
docs(core): add migration plan for Blaze Skate apps
```
