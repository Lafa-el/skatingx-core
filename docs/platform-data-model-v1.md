# SkatingX Platform Data Model v1

## Goal

Define the complete v1 platform data model for SkatingX Platform and the Blaze Skate ecosystem.

This document is architecture documentation only. It does not define application code, Firebase SDK wrappers, Cloud Functions, or Firestore security rules.

## Design Direction

SkatingX Platform uses `User` as the Firebase Auth identity anchor and `Athlete` as the domain anchor. Athlete records own the growing sport history through subcollections so dashboards can stay low-cost and records can scale past 10000 users.

Core conventions:

- Every persisted entity includes stable IDs, `schemaVersion`, `createdAt`, `updatedAt`, and `sourceApp`.
- Child records include both `ownerUid` and `athleteId` even when those IDs already appear in the Firestore path.
- Product-specific fields live under `extensions.{moduleName}` until they become shared platform fields.
- Legacy IDs live under `externalRefs` for migration traceability.
- Large media, transcripts, sensor payloads, and generated artifacts stay outside core documents and are referenced by lightweight metadata.

Recommended shared status values:

- Active profile or catalog entities: `active`, `inactive`, `archived`.
- Draftable content entities: `draft`, `published`, `archived`.
- Review workflow entities: `draft`, `reviewed`, `shared`, `archived`.

Recommended visibility values:

- `private`
- `shared`
- `coachOnly`
- `team`

## Entity: User

### 1. Purpose

Represents a Firebase Auth account and platform-level identity. A user may be an athlete, parent, coach, club admin, academy learner, or platform admin. The user is not the same entity as an athlete profile.

### 2. Firestore Path

```text
users/{uid}
```

Future organization-scoped membership may use:

```text
organizations/{organizationId}/members/{uid}
```

### 3. Required Fields

| Field | Type | Notes |
| --- | --- | --- |
| `uid` | string | Firebase Auth UID and document ID. |
| `schemaVersion` | string | v1 document version, for example `1.0.0`. |
| `email` | string | Primary account email when available. |
| `roles` | string[] | Platform-level roles such as `athleteOwner`, `athlete`, `parent`, `coach`, `clubAdmin`, `platformAdmin`. |
| `status` | string | `active`, `inactive`, or `archived`. |
| `createdAt` | timestamp/string | First creation time. |
| `updatedAt` | timestamp/string | Last mutation time. |
| `sourceApp` | string | App or migration source. |

### 4. Optional Fields

| Field | Type | Notes |
| --- | --- | --- |
| `displayName` | string | User-facing account name. |
| `photoUrl` | string | Avatar URL or storage-backed image reference. |
| `phoneNumber` | string | Optional contact field. Avoid requiring it. |
| `locale` | string | Preferred locale. |
| `timezone` | string | Preferred timezone. |
| `defaultAthleteId` | string | Optional default athlete for quick app launch. |
| `preferences` | object | Small UI or notification preferences. Keep app-specific keys namespaced. |
| `onboarding` | object | Small onboarding state. Do not store large tutorials or logs. |
| `organizationMemberships` | object[] | Lightweight membership summary; canonical membership can move to organization collections. |
| `lastLoginAt` | timestamp/string | Convenience field for support and audit workflows. |
| `externalRefs` | object | Legacy or third-party identity mappings. |
| `extensions` | object | Namespaced product-specific metadata. |

### 5. Relationships

- Owns or can access many `Athlete` documents.
- May create `TrainingRecord`, `JournalEntry`, `EquipmentRecord`, `CompetitionResult`, `AnalysisSession`, and `CoachNote` records through `createdByUid`.
- May be referenced inside `Athlete.access.members`.
- May be a coach referenced by `CoachNote.coachUid`.
- May enroll in or manage `AcademyProgram` records through future enrollment documents.
- May receive `Reward` records.

### 6. Future Expansion Notes

- Keep `users/{uid}` small. Do not embed athlete histories, academy progress logs, rewards history, or notification history.
- If role logic becomes complex, move role assignments into scoped membership collections instead of expanding arrays indefinitely.
- Security rules should treat `uid` and `ownerUid` as immutable after creation.

## Entity: Athlete

### 1. Purpose

Represents the canonical skater profile shared by Training, Journal, Analysis, Academy, and future SkatingX Platform modules. Athlete is the primary domain anchor for sport records.

### 2. Firestore Path

```text
users/{uid}/athletes/{athleteId}
```

Long-term platform or organization indexes may mirror references, but the user-scoped athlete path remains the v1 write model.

### 3. Required Fields

| Field | Type | Notes |
| --- | --- | --- |
| `athleteId` | string | Stable athlete document ID. |
| `ownerUid` | string | Owning Firebase Auth UID. |
| `schemaVersion` | string | v1 document version. |
| `displayName` | string | Primary display name. |
| `status` | string | `active`, `inactive`, or `archived`. |
| `createdAt` | timestamp/string | First creation time. |
| `updatedAt` | timestamp/string | Last mutation time. |
| `sourceApp` | string | App or migration source. |

### 4. Optional Fields

| Field | Type | Notes |
| --- | --- | --- |
| `preferredName` | string | Short or preferred name. |
| `birthYear` | number | Prefer over full date of birth for privacy. |
| `ageGroup` | string | Product-friendly category such as `juvenile` or `adult`. |
| `primaryDiscipline` | string | `figure`, `hockey`, `speed`, `synchro`, `recreational`, `offIce`, or `other`. |
| `disciplines` | string[] | Additional disciplines. |
| `level` | string | Skating level, test level, league, or program stage. |
| `homeClub` | string | Club or rink context. |
| `coaches` | object[] | Lightweight coach summary. Use `uid` when a coach has a SkatingX account. |
| `goals` | string[] | Small active goal list. Do not store goal history here. |
| `summary` | object | Low-cost dashboard counters and latest timestamps. |
| `access` | object | Owner and small access member list. Move to dedicated collections if it grows. |
| `visibility` | string | Product-level visibility hint. |
| `externalRefs` | object | Legacy app IDs. |
| `extensions` | object | Product-specific fields. |
| `archivedAt` | timestamp/string | Archive timestamp. |

### 5. Relationships

- Belongs to one owner `User`.
- Can grant access to many `User` accounts through `access.members`.
- Has many `TrainingRecord`, `JournalEntry`, `EquipmentRecord`, `CompetitionResult`, `AnalysisSession`, `CoachNote`, and `Reward` records.
- Can be enrolled in many `AcademyProgram` records through future enrollment/progress documents.

### 6. Future Expansion Notes

- Do not duplicate mutable athlete profile fields into child records unless needed as immutable snapshots.
- Keep the athlete document focused on identity, access, and summaries.
- If a club/team model is added, link athletes to organizations rather than moving athlete records immediately.
- Preserve compatibility with existing Blaze Skate profile data through additive fields and `externalRefs`.

## Entity: TrainingRecord

### 1. Purpose

Stores one planned or completed training activity, including on-ice, off-ice, strength, mobility, mental training, recovery, and other practice work.

### 2. Firestore Path

```text
users/{uid}/athletes/{athleteId}/trainingRecords/{recordId}
```

### 3. Required Fields

| Field | Type | Notes |
| --- | --- | --- |
| `recordId` | string | Stable training record ID. |
| `athleteId` | string | Parent athlete ID. |
| `ownerUid` | string | Owning user UID. |
| `schemaVersion` | string | v1 document version. |
| `occurredAt` | timestamp/string | Planned or actual training time. |
| `trainingType` | string | `onIce`, `offIce`, `strength`, `conditioning`, `mobility`, `mental`, `recovery`, or `other`. |
| `status` | string | `planned`, `completed`, `skipped`, or `archived`. |
| `createdAt` | timestamp/string | First creation time. |
| `updatedAt` | timestamp/string | Last mutation time. |
| `sourceApp` | string | App or migration source. |

### 4. Optional Fields

| Field | Type | Notes |
| --- | --- | --- |
| `timezone` | string | Local timezone for display. |
| `focusAreas` | string[] | Skills or focus tags. |
| `durationMinutes` | number | Duration, capped to avoid invalid day-length values. |
| `intensity` | number | 1-10 intensity score. |
| `rpe` | number | 1-10 rate of perceived exertion. |
| `location` | string | Rink, gym, or off-ice location. |
| `coachName` | string | Lightweight text field when coach UID is not known. |
| `planned` | boolean | Convenience indicator for planned sessions. |
| `notes` | string | General notes. |
| `metrics` | object | Small structured metrics such as jump counts or heart-rate summary. |
| `relatedCompetitionId` | string | Linked `CompetitionResult`. |
| `relatedEquipmentIds` | string[] | Linked `EquipmentRecord` IDs. |
| `visibility` | string | Product-level visibility hint. |
| `createdByUid` | string | User who created the record. |
| `updatedByUid` | string | User who last updated the record. |
| `externalRefs` | object | Legacy app IDs. |
| `extensions` | object | Product-specific fields. |
| `archivedAt` | timestamp/string | Archive timestamp. |

### 5. Relationships

- Belongs to one `Athlete`.
- Owned by one `User`.
- Can link to one `CompetitionResult`.
- Can link to many `EquipmentRecord` documents.
- Can be referenced by `JournalEntry`, `AnalysisSession`, `CoachNote`, and `Reward`.

### 6. Future Expansion Notes

- Keep detailed drill libraries, recurring plans, and generated practice templates outside individual completed records.
- For recurring schedules, add a future `trainingPlans` or `trainingTemplates` collection instead of overloading `TrainingRecord`.
- Use athlete summary fields for latest training and record counts to avoid reading the full subcollection for dashboards.

## Entity: JournalEntry

### 1. Purpose

Stores athlete, parent, or coach reflections and wellness notes tied to training, competitions, goals, or general development.

### 2. Firestore Path

```text
users/{uid}/athletes/{athleteId}/journalEntries/{entryId}
```

### 3. Required Fields

| Field | Type | Notes |
| --- | --- | --- |
| `entryId` | string | Stable journal entry ID. |
| `athleteId` | string | Parent athlete ID. |
| `ownerUid` | string | Owning user UID. |
| `schemaVersion` | string | v1 document version. |
| `entryDate` | timestamp/string | Journal date/time. |
| `entryType` | string | `reflection`, `trainingNote`, `competitionReflection`, `goal`, `coachNote`, `parentNote`, `wellness`, or `other`. |
| `visibility` | string | `private`, `shared`, `coachOnly`, or `team`. |
| `createdAt` | timestamp/string | First creation time. |
| `updatedAt` | timestamp/string | Last mutation time. |
| `sourceApp` | string | App or migration source. |

### 4. Optional Fields

| Field | Type | Notes |
| --- | --- | --- |
| `timezone` | string | Local timezone for display. |
| `title` | string | Optional title. |
| `prompt` | string | Prompt text shown to the author. |
| `body` | string | Entry content. |
| `mood` | string | Lightweight mood label. |
| `confidence` | number | 1-10 confidence score. |
| `energy` | number | 1-10 energy score. |
| `tags` | string[] | Search or grouping tags. |
| `relatedTrainingRecordId` | string | Linked `TrainingRecord`. |
| `relatedCompetitionId` | string | Linked `CompetitionResult`. |
| `status` | string | `draft`, `published`, or `archived`. |
| `createdByUid` | string | User who created the entry. |
| `updatedByUid` | string | User who last updated the entry. |
| `externalRefs` | object | Legacy app IDs. |
| `extensions` | object | Product-specific fields. |
| `archivedAt` | timestamp/string | Archive timestamp. |

### 5. Relationships

- Belongs to one `Athlete`.
- Owned by one `User`.
- Can be authored by athlete, parent, or coach users.
- Can link to `TrainingRecord` and `CompetitionResult`.
- Can be used by future analytics as qualitative context for `AnalysisSession`.

### 6. Future Expansion Notes

- Sensitive wellness or mental-health details may require stricter visibility and security-rule treatment.
- Do not store large AI summaries or longitudinal sentiment analysis directly in each entry unless those are bounded fields.
- If journal comments are needed, add a bounded comments subcollection rather than unbounded arrays.

## Entity: EquipmentRecord

### 1. Purpose

Tracks athlete equipment such as boots, blades, helmets, pads, sticks, apparel, and training aids, including fit notes and maintenance history.

### 2. Firestore Path

```text
users/{uid}/athletes/{athleteId}/equipmentRecords/{equipmentId}
```

### 3. Required Fields

| Field | Type | Notes |
| --- | --- | --- |
| `equipmentId` | string | Stable equipment record ID. |
| `athleteId` | string | Parent athlete ID. |
| `ownerUid` | string | Owning user UID. |
| `schemaVersion` | string | v1 document version. |
| `equipmentType` | string | `boots`, `blades`, `helmet`, `pads`, `stick`, `apparel`, `trainingAid`, or `other`. |
| `status` | string | `active`, `backup`, `maintenance`, `retired`, or `archived`. |
| `createdAt` | timestamp/string | First creation time. |
| `updatedAt` | timestamp/string | Last mutation time. |
| `sourceApp` | string | App or migration source. |

### 4. Optional Fields

| Field | Type | Notes |
| --- | --- | --- |
| `brand` | string | Equipment brand. |
| `model` | string | Model name. |
| `size` | string | Size or fit label. |
| `serialNumber` | string | Optional serial or shop reference. |
| `purchaseDate` | date/string | Purchase date. |
| `firstUsedAt` | timestamp/string | First use time. |
| `retiredAt` | timestamp/string | Retirement time. |
| `fitNotes` | string | Fit, comfort, or sizing notes. |
| `maintenance` | object[] | Bounded maintenance entries. |
| `visibility` | string | Product-level visibility hint. |
| `createdByUid` | string | User who created the record. |
| `updatedByUid` | string | User who last updated the record. |
| `externalRefs` | object | Legacy app IDs. |
| `extensions` | object | Product-specific fields. |
| `archivedAt` | timestamp/string | Archive timestamp. |

### 5. Relationships

- Belongs to one `Athlete`.
- Owned by one `User`.
- Can be linked from many `TrainingRecord` documents.
- Can be linked from `CompetitionResult` or `AnalysisSession` when equipment context affects performance.

### 6. Future Expansion Notes

- If maintenance grows beyond a bounded array, move maintenance into `equipmentRecords/{equipmentId}/maintenanceEvents/{eventId}`.
- Equipment marketplace, fitting recommendations, or shop integrations should use explicit extension objects or separate platform collections.
- Avoid storing receipts or large images directly in the document; use storage references.

## Entity: CompetitionResult

### 1. Purpose

Stores competition, test, race, game, showcase, or assessment outcomes for one athlete.

### 2. Firestore Path

```text
users/{uid}/athletes/{athleteId}/competitionResults/{competitionId}
```

### 3. Required Fields

| Field | Type | Notes |
| --- | --- | --- |
| `competitionId` | string | Stable competition result ID. |
| `athleteId` | string | Parent athlete ID. |
| `ownerUid` | string | Owning user UID. |
| `schemaVersion` | string | v1 document version. |
| `eventName` | string | Event, meet, test, race, or game name. |
| `eventDate` | timestamp/string | Event date/time. |
| `competitionType` | string | `competition`, `test`, `race`, `game`, `showcase`, `assessment`, or `other`. |
| `createdAt` | timestamp/string | First creation time. |
| `updatedAt` | timestamp/string | Last mutation time. |
| `sourceApp` | string | App or migration source. |

### 4. Optional Fields

| Field | Type | Notes |
| --- | --- | --- |
| `timezone` | string | Local timezone for display. |
| `discipline` | string | Discipline category. |
| `level` | string | Competition level or category. |
| `location` | string | Venue or city. |
| `program` | string | Program, race, event segment, or game context. |
| `placement` | number | Final placement. |
| `fieldSize` | number | Number of competitors or teams. |
| `score` | number | Primary score. |
| `scoreBreakdown` | object | Small structured score details. |
| `outcome` | string | Text result such as `qualified`, `personalBest`, or `passed`. |
| `notes` | string | Result notes. |
| `relatedTrainingRecordIds` | string[] | Preparation or follow-up training records. |
| `relatedAnalysisSessionIds` | string[] | Linked analysis sessions. |
| `visibility` | string | Product-level visibility hint. |
| `status` | string | `scheduled`, `completed`, `cancelled`, or `archived`. |
| `createdByUid` | string | User who created the result. |
| `updatedByUid` | string | User who last updated the result. |
| `externalRefs` | object | Legacy app IDs. |
| `extensions` | object | Product-specific fields. |
| `archivedAt` | timestamp/string | Archive timestamp. |

### 5. Relationships

- Belongs to one `Athlete`.
- Owned by one `User`.
- Can be linked from many `TrainingRecord`, `JournalEntry`, `AnalysisSession`, `CoachNote`, and `Reward` records.

### 6. Future Expansion Notes

- If event catalogs become shared across athletes, add a platform-level `events/{eventId}` catalog and let `CompetitionResult` reference it.
- Keep judge sheets, PDFs, and media in storage or artifact collections.
- For team sports, future result models may need team-level context in addition to athlete-level records.

## Entity: AnalysisSession

### 1. Purpose

Stores technical analysis, coach review, video analysis, sensor review, comparison, or scoring sessions tied to an athlete.

### 2. Firestore Path

```text
users/{uid}/athletes/{athleteId}/analysisSessions/{sessionId}
```

Large generated artifacts may use:

```text
users/{uid}/athletes/{athleteId}/analysisSessions/{sessionId}/artifacts/{artifactId}
```

### 3. Required Fields

| Field | Type | Notes |
| --- | --- | --- |
| `sessionId` | string | Stable analysis session ID. |
| `athleteId` | string | Parent athlete ID. |
| `ownerUid` | string | Owning user UID. |
| `schemaVersion` | string | v1 document version. |
| `analyzedAt` | timestamp/string | Analysis date/time. |
| `analysisType` | string | `video`, `sensor`, `coachReview`, `technicalScore`, `comparison`, or `other`. |
| `status` | string | `draft`, `reviewed`, `shared`, or `archived`. |
| `createdAt` | timestamp/string | First creation time. |
| `updatedAt` | timestamp/string | Last mutation time. |
| `sourceApp` | string | App or migration source. |

### 4. Optional Fields

| Field | Type | Notes |
| --- | --- | --- |
| `timezone` | string | Local timezone for display. |
| `discipline` | string | Discipline category. |
| `skills` | string[] | Skills or elements analyzed. |
| `summary` | string | Human-readable summary. |
| `findings` | object[] | Bounded findings list with labels, descriptions, severity, and timestamps. |
| `metrics` | object | Small structured metrics. |
| `media` | object[] | Lightweight storage or URL references. |
| `recommendations` | string[] | Bounded next-step recommendations. |
| `relatedTrainingRecordId` | string | Linked training record. |
| `relatedCompetitionId` | string | Linked competition result. |
| `visibility` | string | Product-level visibility hint. |
| `createdByUid` | string | User who created the session. |
| `updatedByUid` | string | User who last updated the session. |
| `externalRefs` | object | Legacy app IDs. |
| `extensions` | object | Product-specific fields. |
| `archivedAt` | timestamp/string | Archive timestamp. |

### 5. Relationships

- Belongs to one `Athlete`.
- Owned by one `User`.
- Can link to one `TrainingRecord` and one `CompetitionResult`.
- Can be referenced by `CoachNote`, `JournalEntry`, and `Reward`.
- Can reference media in Firebase Storage or future artifact documents.

### 6. Future Expansion Notes

- Do not store raw pose data, frame-by-frame metrics, transcripts, or generated clips directly in the session document.
- Add artifact subcollections for heavy analysis outputs.
- AI-generated findings should include provenance in `extensions` or future audit fields.

## Entity: CoachNote

### 1. Purpose

Stores structured coach feedback, action items, corrections, and review notes. It is separate from `JournalEntry` so coach workflows can support assignment, review, visibility, and follow-up without overloading athlete journals.

### 2. Firestore Path

```text
users/{uid}/athletes/{athleteId}/coachNotes/{noteId}
```

If an organization/club model later owns coach notes:

```text
organizations/{organizationId}/athletes/{athleteId}/coachNotes/{noteId}
```

### 3. Required Fields

| Field | Type | Notes |
| --- | --- | --- |
| `noteId` | string | Stable coach note ID. |
| `athleteId` | string | Parent athlete ID. |
| `ownerUid` | string | Owning user UID for v1 path. |
| `coachUid` | string | Coach user UID. |
| `schemaVersion` | string | v1 document version. |
| `noteType` | string | `technical`, `training`, `competition`, `equipment`, `wellness`, `academy`, or `other`. |
| `body` | string | Main feedback text. |
| `visibility` | string | Usually `coachOnly`, `shared`, or `team`. |
| `status` | string | `draft`, `shared`, `resolved`, or `archived`. |
| `createdAt` | timestamp/string | First creation time. |
| `updatedAt` | timestamp/string | Last mutation time. |
| `sourceApp` | string | App or migration source. |

### 4. Optional Fields

| Field | Type | Notes |
| --- | --- | --- |
| `title` | string | Short summary title. |
| `coachName` | string | Denormalized display label for low-cost rendering. |
| `targetSkill` | string | Skill, element, or focus area. |
| `actionItems` | object[] | Bounded next actions with status and due date. |
| `relatedTrainingRecordId` | string | Linked training record. |
| `relatedJournalEntryId` | string | Linked journal entry. |
| `relatedCompetitionId` | string | Linked competition result. |
| `relatedAnalysisSessionId` | string | Linked analysis session. |
| `relatedAcademyProgramId` | string | Linked academy program. |
| `tags` | string[] | Search or grouping tags. |
| `createdByUid` | string | User who created the note. |
| `updatedByUid` | string | User who last updated the note. |
| `externalRefs` | object | Legacy app IDs. |
| `extensions` | object | Product-specific fields. |
| `archivedAt` | timestamp/string | Archive timestamp. |

### 5. Relationships

- Belongs to one `Athlete`.
- Authored by one coach `User`.
- May reference `TrainingRecord`, `JournalEntry`, `CompetitionResult`, `AnalysisSession`, and `AcademyProgram`.
- May lead to future `Reward` issuance when feedback milestones are achieved.

### 6. Future Expansion Notes

- If threaded coach-athlete conversations are needed, create a `coachNoteComments` subcollection instead of embedding unbounded replies.
- Coach visibility must be enforced by security rules, not only by the `visibility` field.
- For organization-managed teams, move canonical ownership to organization paths while preserving user-scoped read models if needed.

## Entity: AcademyProgram

### 1. Purpose

Represents a Lindii Academy or SkatingX learning program, course, pathway, clinic, or structured curriculum that can connect learning content to athlete development.

### 2. Firestore Path

Canonical catalog:

```text
academyPrograms/{programId}
```

Future athlete enrollment/progress:

```text
users/{uid}/athletes/{athleteId}/academyEnrollments/{enrollmentId}
```

Future organization-owned program variants:

```text
organizations/{organizationId}/academyPrograms/{programId}
```

### 3. Required Fields

| Field | Type | Notes |
| --- | --- | --- |
| `programId` | string | Stable academy program ID. |
| `schemaVersion` | string | v1 document version. |
| `title` | string | Program title. |
| `programType` | string | `course`, `pathway`, `clinic`, `challenge`, `assessment`, or `other`. |
| `status` | string | `draft`, `published`, `inactive`, or `archived`. |
| `createdAt` | timestamp/string | First creation time. |
| `updatedAt` | timestamp/string | Last mutation time. |
| `sourceApp` | string | App or migration source. |

### 4. Optional Fields

| Field | Type | Notes |
| --- | --- | --- |
| `slug` | string | Stable URL or catalog slug. |
| `description` | string | Program summary. |
| `discipline` | string | Discipline category. |
| `level` | string | Target level. |
| `ageGroup` | string | Target age group. |
| `durationDays` | number | Suggested duration. |
| `moduleCount` | number | Lightweight module count. |
| `modules` | object[] | Bounded public module summaries only. Full content can use subcollections. |
| `prerequisites` | string[] | Recommended prerequisites. |
| `outcomes` | string[] | Expected learning outcomes. |
| `coachUid` | string | Lead coach or instructor UID. |
| `organizationId` | string | Owning organization if applicable. |
| `rewardIds` | string[] | Rewards available from the program. |
| `visibility` | string | Catalog visibility. |
| `externalRefs` | object | Legacy LMS or content IDs. |
| `extensions` | object | Lindii or SkatingX-specific content metadata. |
| `archivedAt` | timestamp/string | Archive timestamp. |

### 5. Relationships

- Can be created or managed by coach/admin `User` accounts.
- Can enroll many `Athlete` records through enrollment/progress documents.
- Can issue many `Reward` records.
- Can be referenced by `CoachNote`, `TrainingRecord`, and `JournalEntry` when learning content drives practice or reflection.

### 6. Future Expansion Notes

- Keep the catalog document small; store lessons, quizzes, videos, assignments, and progress in dedicated subcollections.
- Lindii Academy can remain independently operated while sharing program and enrollment contracts with SkatingX Platform.
- If academy billing or licensing is added, keep billing data outside this core program document.

## Entity: Reward

### 1. Purpose

Represents badges, achievements, streaks, milestones, certificates, or program completion rewards. Rewards can motivate athletes and connect training, journal, analysis, competition, and academy activity.

### 2. Firestore Path

Athlete-awarded reward:

```text
users/{uid}/athletes/{athleteId}/rewards/{rewardId}
```

Optional platform reward definition catalog:

```text
rewardDefinitions/{rewardDefinitionId}
```

### 3. Required Fields

| Field | Type | Notes |
| --- | --- | --- |
| `rewardId` | string | Stable awarded reward ID. |
| `athleteId` | string | Parent athlete ID. |
| `ownerUid` | string | Owning user UID. |
| `schemaVersion` | string | v1 document version. |
| `rewardType` | string | `badge`, `streak`, `milestone`, `certificate`, `academyCompletion`, `competition`, or `other`. |
| `title` | string | Display title. |
| `awardedAt` | timestamp/string | Award timestamp. |
| `status` | string | `active`, `revoked`, or `archived`. |
| `createdAt` | timestamp/string | First creation time. |
| `updatedAt` | timestamp/string | Last mutation time. |
| `sourceApp` | string | App or migration source. |

### 4. Optional Fields

| Field | Type | Notes |
| --- | --- | --- |
| `rewardDefinitionId` | string | Link to catalog definition. |
| `description` | string | Display description. |
| `icon` | object | Small icon metadata or storage reference. |
| `criteria` | object | Snapshot of criteria used when awarded. |
| `points` | number | Optional gamification points. |
| `level` | string | Bronze/silver/gold or program level. |
| `relatedTrainingRecordIds` | string[] | Linked training records. |
| `relatedJournalEntryIds` | string[] | Linked journal entries. |
| `relatedCompetitionId` | string | Linked competition result. |
| `relatedAnalysisSessionId` | string | Linked analysis session. |
| `relatedAcademyProgramId` | string | Linked academy program. |
| `issuedByUid` | string | Coach/admin/system user that issued it. |
| `visibility` | string | Product-level visibility hint. |
| `externalRefs` | object | Legacy reward or LMS IDs. |
| `extensions` | object | Product-specific reward metadata. |
| `archivedAt` | timestamp/string | Archive timestamp. |

### 5. Relationships

- Belongs to one `Athlete`.
- Owned by one `User`.
- May be issued by a coach/admin `User` or system process.
- May reference `TrainingRecord`, `JournalEntry`, `CompetitionResult`, `AnalysisSession`, and `AcademyProgram`.
- May be based on a shared `rewardDefinitions/{rewardDefinitionId}` catalog item.

### 6. Future Expansion Notes

- Keep awarded rewards immutable where possible. If criteria change, create a new reward definition version rather than rewriting historical awards.
- Avoid recalculating every reward on dashboard load. Use event-driven or batch awarding later.
- If leaderboards are added, use aggregate documents and pagination rather than scanning all rewards.
