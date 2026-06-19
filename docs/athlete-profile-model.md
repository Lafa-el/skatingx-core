# Athlete Profile Model

## Goal

Define the canonical athlete profile used across Training, Journal, Lab, Analysis, and future SkatingX Platform modules.

## Profile Responsibility

The athlete profile stores stable skater information needed across products:

- Name and display identity
- Discipline and level
- Club or team context
- Coach references
- Profile visibility
- Lightweight dashboard summary
- Migration references

It should not store full activity history.

## Required Fields

Core required fields:

- `athleteId`
- `ownerUid`
- `schemaVersion`
- `displayName`
- `status`
- `createdAt`
- `updatedAt`

## Recommended Optional Fields

Optional fields should be added only when useful across apps:

- `preferredName`
- `birthYear`
- `ageGroup`
- `primaryDiscipline`
- `disciplines`
- `level`
- `homeClub`
- `coaches`
- `goals`
- `summary`
- `externalRefs`

## Discipline Values

Initial discipline values:

- `figure`
- `hockey`
- `speed`
- `synchro`
- `recreational`
- `offIce`
- `other`

These values are broad by design. Product-specific discipline detail can live in extensions until it becomes shared.

## Status Values

Initial status values:

- `active`
- `inactive`
- `archived`

Do not physically delete athlete profiles by default. Archive first to preserve references.

## Dashboard Summary

The `summary` object is optional and should stay small:

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

The summary improves dashboard performance. It is not authoritative history.

## Extension Policy

Use extension objects for app-specific data:

```json
{
  "extensions": {
    "blazeTraining": {
      "preferredTrainingFocus": ["jumps", "edges"]
    },
    "blazeJournal": {
      "journalPromptSet": "confidence-builder"
    }
  }
}
```

Extension keys should be namespaced by product or module.

## Data Minimization

For youth athletes, avoid storing sensitive personal data unless an app explicitly requires it.

Prefer:

- `birthYear`
- `ageGroup`
- `displayName`

Avoid by default:

- Full address
- School name
- Medical notes
- Government IDs

## Migration Notes

Legacy Blaze Skate apps may already store athlete-like profile data in different places. During migration:

1. Create or match one canonical athlete profile.
2. Preserve legacy IDs in `externalRefs`.
3. Link all imported records to `athleteId`.
4. Avoid duplicating mutable profile fields into child records.
