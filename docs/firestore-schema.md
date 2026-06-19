# Firestore Schema

## Goal

Define a Firestore layout that supports current Blaze Skate apps and future SkatingX Platform consolidation while controlling read cost and avoiding oversized documents.

## Preferred Collection Layout

```text
users/{uid}
users/{uid}/athletes/{athleteId}
users/{uid}/athletes/{athleteId}/trainingRecords/{recordId}
users/{uid}/athletes/{athleteId}/journalEntries/{entryId}
users/{uid}/athletes/{athleteId}/analysisSessions/{sessionId}
users/{uid}/athletes/{athleteId}/equipmentRecords/{equipmentId}
users/{uid}/athletes/{athleteId}/competitionResults/{competitionId}
```

This structure optimizes the most common reads:

- Load current user profile.
- List the user's athletes.
- Load one athlete dashboard.
- List recent training, journal, analysis, equipment, or competition records for one athlete.

## Top-Level User Document

Path:

```text
users/{uid}
```

Recommended fields:

```json
{
  "uid": "firebase-auth-uid",
  "email": "owner@example.com",
  "displayName": "Owner Name",
  "roles": ["athleteOwner"],
  "createdAt": "2026-06-19T12:00:00Z",
  "updatedAt": "2026-06-19T12:00:00Z"
}
```

Keep this document small. Do not embed athlete profiles or activity history here.

## Athlete Documents

Path:

```text
users/{uid}/athletes/{athleteId}
```

The athlete document is the canonical profile for one skater.

Keep it focused on:

- Identity and display fields
- Birth year or age group, not necessarily full birth date
- Discipline and level
- Club and coach references
- Dashboard summary fields
- Profile visibility and ownership

Avoid embedding:

- Full training history
- Full journal history
- Large media references
- Long analysis results

## Activity Subcollections

Use subcollections for records that grow over time:

- `trainingRecords`
- `journalEntries`
- `analysisSessions`
- `equipmentRecords`
- `competitionResults`

Each child document should include `athleteId` and `ownerUid` even though the path already contains them. This supports collection group queries and migration exports.

## Dashboard Summaries

For low-cost dashboards, keep a small summary object in the athlete document:

```json
{
  "summary": {
    "lastTrainingAt": "2026-06-18T20:30:00Z",
    "lastJournalAt": "2026-06-18T22:00:00Z",
    "lastAnalysisAt": "2026-06-17T18:30:00Z",
    "trainingRecordCount": 42,
    "journalEntryCount": 15,
    "analysisSessionCount": 6
  }
}
```

This avoids reading multiple subcollections for common dashboard cards.

Summary fields may be updated by app code or future Cloud Functions. They are convenience fields, not the source of truth.

## Indexing Direction

Recommended query patterns:

- Recent records for one athlete: order by `occurredAt` or `createdAt` descending.
- Records by date range: filter by `occurredAt`.
- Records by type: filter by `type` or `category`.
- Records by visibility: filter by `visibility`.
- Cross-athlete platform views: collection group query by `ownerUid`, `athleteId`, and date fields.

Potential composite indexes:

```text
trainingRecords: ownerUid ASC, athleteId ASC, occurredAt DESC
journalEntries: ownerUid ASC, athleteId ASC, entryDate DESC
analysisSessions: ownerUid ASC, athleteId ASC, analyzedAt DESC
competitionResults: ownerUid ASC, athleteId ASC, eventDate DESC
equipmentRecords: ownerUid ASC, athleteId ASC, status ASC
```

Create indexes only when real queries require them.

## Cost Controls

1. List views should use pagination.
2. Dashboard views should use summary fields.
3. Do not read every subcollection to render an athlete home screen.
4. Do not store unbounded arrays inside athlete documents.
5. Store analysis artifacts and media in separate documents or Firebase Storage.
6. Prefer selective reads over broad collection group queries in app screens.

## Deletion and Archiving

Use soft archive fields for user-facing records:

```json
{
  "status": "archived",
  "archivedAt": "2026-06-19T12:00:00Z"
}
```

Permanent deletion should be a separate product decision because athlete data may be referenced across modules.

## Cross-App Migration

Legacy app documents should keep original IDs under `externalRefs`:

```json
{
  "externalRefs": {
    "blazeTraining": {
      "legacyRecordId": "old-training-id"
    }
  }
}
```

This makes migrations reversible and easier to audit.
