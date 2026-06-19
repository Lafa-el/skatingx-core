# SkatingX Core

SkatingX Core is the shared data-model repository for the Blaze Skate and SkatingX ecosystem.

It is the single source of truth for portable schemas, Firestore collection design, authentication assumptions, and migration guidance used by:

- Blaze Skate Training
- Blaze Skate Journal
- Blaze Skate Lab
- Blaze Skate Analysis
- Future SkatingX Platform modules

This repository intentionally contains documentation, JSON Schemas, and example documents only.

It does not contain:

- UI code
- Firebase app initialization
- Cloud Functions
- Firestore security rules
- Runtime SDK wrappers

## Repository Structure

```text
docs/
  platform-architecture.md
  firestore-schema.md
  auth-model.md
  athlete-profile-model.md
  migration-plan-from-blaze-apps.md
schemas/
  athlete.schema.json
  training-record.schema.json
  journal-entry.schema.json
  analysis-session.schema.json
  equipment-record.schema.json
  competition-result.schema.json
examples/
  athlete.example.json
  training-record.example.json
  journal-entry.example.json
  analysis-session.example.json
  equipment-record.example.json
  competition-result.example.json
```

## Design Principles

1. Shared models must be stable across products.
2. Product-specific fields should live under explicit extension objects.
3. Firestore reads and writes should stay predictable at 10000+ users.
4. Authentication identity and athlete identity are separate concepts.
5. Schemas should support future platform consolidation without forcing immediate app rewrites.
6. Data should be exportable, versioned, and migration friendly.

## Model Versioning

Every schema includes:

- `schemaVersion`: semantic model version for the document shape.
- `createdAt`: ISO 8601 timestamp for first creation.
- `updatedAt`: ISO 8601 timestamp for latest mutation.
- `sourceApp`: the app that originally created or imported the document.

Breaking changes should create a new major schema version and be documented in `docs/migration-plan-from-blaze-apps.md`.

## Canonical Entity Scope

The core shared entities are:

- Athlete profile
- Training record
- Journal entry
- Analysis session
- Equipment record
- Competition result

The athlete profile is the anchor model. Other records should reference athletes through `athleteId`, not duplicated profile data.

## Firestore Direction

The preferred Firestore shape is user-scoped and athlete-indexed:

```text
users/{uid}
users/{uid}/athletes/{athleteId}
users/{uid}/athletes/{athleteId}/trainingRecords/{recordId}
users/{uid}/athletes/{athleteId}/journalEntries/{entryId}
users/{uid}/athletes/{athleteId}/analysisSessions/{sessionId}
users/{uid}/athletes/{athleteId}/equipmentRecords/{equipmentId}
users/{uid}/athletes/{athleteId}/competitionResults/{competitionId}
```

This keeps common app reads scoped to one user and one athlete, while allowing collection group queries for platform-level features later.

See [Firestore Schema](docs/firestore-schema.md) for details.

## Schema Validation

The files in `schemas/` are JSON Schema Draft 2020-12 documents. They are designed for validation in build scripts, migration tools, tests, and future SDK packages.

Examples in `examples/` are representative documents, not production IDs.

## Current Status

This is the initial repository foundation. The next logical steps are:

1. Add schema validation scripts.
2. Add TypeScript type generation from JSON Schema.
3. Add Firestore security-rule design docs.
4. Add migration mappers for existing Blaze Skate apps.
