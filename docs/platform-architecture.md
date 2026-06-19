# SkatingX Platform Architecture

## Goal

SkatingX Core defines the shared data contract used across Blaze Skate Training, Blaze Skate Journal, Blaze Skate Lab, Blaze Skate Analysis, and future SkatingX Platform applications.

The immediate goal is not to merge every app into one runtime. The goal is to make each app write and read compatible data so future consolidation is possible without destructive migration.

## Architectural Boundary

SkatingX Core owns:

- Shared entity definitions
- JSON Schemas
- Firestore collection guidance
- Authentication and identity assumptions
- Migration documentation
- Example documents

SkatingX Core does not own:

- React UI
- Vite app shells
- Firebase app initialization
- Firestore SDK clients
- Product-specific workflows
- Cloud Functions
- Billing, notifications, or storage implementation

## Product Context

### Blaze Skate Training

Primary responsibility:

- Daily and structured training records
- Session intensity, duration, focus areas, coach notes, and completion state

Shared model dependency:

- Athlete profile
- Training record
- Equipment references
- Competition context when training is competition-specific

### Blaze Skate Journal

Primary responsibility:

- Athlete reflections
- Coach and parent observations
- Mood, confidence, recovery, goals, and private notes

Shared model dependency:

- Athlete profile
- Journal entry
- Training references
- Competition references

### Blaze Skate Lab and Analysis

Primary responsibility:

- Video or sensor-based analysis sessions
- Technical observations
- Scores, tagged clips, metrics, and recommendations

Shared model dependency:

- Athlete profile
- Analysis session
- Training references
- Competition references

### SkatingX Platform

Future responsibility:

- Unified athlete workspace
- Cross-product dashboards
- Role-based access
- Longitudinal progress history
- Academy and learning integrations

Shared model dependency:

- All core entities
- Stable auth and role model
- Migration-safe IDs

## Core Domain Model

```text
Firebase Auth User
  owns or can access
Athlete Profile
  has many Training Records
  has many Journal Entries
  has many Analysis Sessions
  has many Equipment Records
  has many Competition Results
```

The Firebase Auth user is not the athlete. A parent, coach, club admin, or adult athlete may own or access one or more athlete profiles.

## Identity Strategy

Use these identifiers consistently:

- `uid`: Firebase Auth user ID.
- `athleteId`: stable athlete profile ID.
- `recordId`, `entryId`, `sessionId`, `equipmentId`, `competitionId`: document IDs for child entities.
- `externalRefs`: optional mapping to IDs from legacy Blaze Skate apps or third-party systems.

IDs should be stable after creation. Avoid deriving IDs from mutable fields such as name, email, club, or date.

## Multi-App Data Strategy

Each product may keep app-specific UI state, drafts, and local preferences outside the shared models. Shared records should contain only durable domain data that another app can safely understand.

When a product needs extra fields, use:

```json
{
  "extensions": {
    "blazeTraining": {},
    "blazeJournal": {},
    "blazeLab": {}
  }
}
```

Extension fields must not replace canonical fields. If an extension becomes broadly useful, promote it into the core schema through a versioned change.

## Event and Audit Direction

Initial schemas use document timestamps only:

- `createdAt`
- `updatedAt`
- `archivedAt`

Future platform services may add append-only audit/event collections. Do not place large audit logs directly inside athlete documents because that increases read cost and document size.

## File and Media Direction

Media files should be stored outside core JSON documents. Core documents may store lightweight references:

- `storagePath`
- `downloadUrl`
- `thumbnailUrl`
- `durationSeconds`
- `mimeType`

Do not store large transcripts, frame data, or raw sensor payloads inside core documents. Store them in dedicated analysis artifacts or storage objects and reference them from the analysis session.

## Scalability Rules

1. Keep frequently listed records in subcollections under `users/{uid}/athletes/{athleteId}`.
2. Avoid large arrays that grow without bound.
3. Prefer small summary fields for dashboards.
4. Use collection group queries only for cross-athlete or platform-wide views.
5. Denormalize small immutable labels when it saves repeated reads, but keep canonical profile data in the athlete document.

## Future Package Direction

This repository can later publish:

- `@skatingx/core-schemas`
- `@skatingx/core-types`
- `@skatingx/firestore-models`

Those packages should be generated from the schemas where possible to avoid drift.
