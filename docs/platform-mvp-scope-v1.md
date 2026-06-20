# SkatingX Platform MVP Scope v1

## Goal

Define the SkatingX Platform MVP scope based on the existing Blaze Skate Training, Blaze Skate Journal, Blaze Skate Analysis apps, and the v1 platform data model.

This document is architecture and product scope documentation only. It does not define application code, Firebase SDK wrappers, Cloud Functions, Firestore security rules, UI implementation, or migration scripts.

## 1. MVP Product Goal

The MVP goal is to create one unified athlete workspace that proves the core SkatingX Platform model can support daily training, reflection, technical analysis, equipment context, competition history, and coach feedback without breaking the current Blaze Skate apps.

The MVP should answer one product question:

> Can an athlete, parent, and coach see the same skater profile and the same recent development history across Training, Journal, and Analysis data?

The MVP should prioritize:

- A stable canonical `Athlete` profile.
- Low-cost athlete dashboards.
- Import or dual-read compatibility with existing Blaze Skate apps.
- A small set of athlete-scoped records that map directly to the v1 data model.
- Clear boundaries for what is not yet part of the platform.

The MVP should not attempt to replace every Blaze Skate app workflow at once. It should consolidate the data surface first, then add deeper product workflows in later phases.

## 2. Target Users

### Athlete

Primary needs:

- View personal athlete profile.
- Track recent and historical training records.
- Write or review journal entries.
- Review coach notes and analysis summaries.
- See competition results and basic progress context.

MVP access:

- Read and create own training and journal records where permissions allow.
- Read shared analysis, competition, equipment, and coach note records.
- View a simple dashboard built from athlete summary fields and recent records.

### Parent

Primary needs:

- Manage one or more athlete profiles.
- Track training consistency, journal context, equipment status, and competition history.
- Understand coach feedback without scanning multiple apps.

MVP access:

- Own or manage athlete profiles.
- Create and edit athlete records where appropriate.
- View cross-module athlete dashboard.
- Preserve privacy boundaries for entries or notes marked `private` or `coachOnly`.

### Coach

Primary needs:

- Review athlete development across training, journal, competition, and analysis records.
- Add coach notes tied to specific records.
- See enough context to guide the next training cycle.

MVP access:

- Read athlete records for athletes with active access.
- Create and update `CoachNote` records.
- Reference training records, competition results, journal entries, and analysis sessions from coach notes.
- Avoid broad team-management workflows in MVP.

### Academy Owner

Primary needs:

- Confirm that future academy programs can connect to athlete development data.
- Understand which training, journal, analysis, and coach-note data can later support learning pathways.
- Prepare Lindii Academy integration without forcing it into the MVP runtime.

MVP access:

- No full academy delivery module in MVP.
- Optional read-only product validation against the `AcademyProgram` relationship design.
- No billing, course playback, quizzes, certification automation, or academy enrollment workflows in MVP.

## 3. MVP Modules

### Athlete Profile

Purpose:

- Establish the canonical skater identity for all MVP modules.

Included:

- Create and view athlete profile.
- Basic profile fields: display name, preferred name, age group or birth year, discipline, level, home club, status.
- Owner and access metadata.
- Small `summary` object for dashboard counts and latest timestamps.
- Legacy app references under `externalRefs`.

Excluded:

- Full organization roster management.
- Complex team membership.
- Public athlete profiles.
- Medical, school, address, or sensitive identity fields.

### Training

Purpose:

- Represent planned or completed training work from the Training app.

Included:

- Recent training list.
- Training detail view.
- Core fields: date, training type, focus areas, duration, intensity or RPE, location, status, notes.
- Links to related equipment and competition context when available.
- Dashboard summary fields such as latest training date and count.

Excluded:

- Complex recurring plan builder.
- Drill library authoring.
- Training marketplace content.
- Automated training plan generation.

### Journal

Purpose:

- Preserve athlete reflection and parent/coach observation context.

Included:

- Journal entry list and detail.
- Core fields: date, type, title, prompt, body, mood, confidence, energy, tags, visibility.
- Links to related training and competition records.
- Draft, published, and archived states.

Excluded:

- Public blogging.
- Social comments.
- AI sentiment dashboards.
- Unbounded threaded discussions.

### Analysis

Purpose:

- Bring technical review and video/sensor analysis summaries into the athlete workspace.

Included:

- Analysis session list and detail.
- Core fields: analyzed date, analysis type, skills, summary, bounded findings, recommendations, status.
- Lightweight media references only.
- Links to training records and competition results.

Excluded:

- Raw frame-by-frame pose data in Firestore documents.
- Heavy artifact rendering inside the core document.
- Full video editing.
- Automated AI analysis pipeline as a required MVP dependency.

### Equipment

Purpose:

- Track equipment context that affects training, performance, comfort, and safety.

Included:

- Equipment list and detail.
- Core fields: equipment type, brand, model, size, status, purchase date, first used date, fit notes.
- Bounded maintenance notes.
- Links from training records when relevant.

Excluded:

- Equipment marketplace.
- Shop integrations.
- Receipt storage in core documents.
- Unbounded maintenance event history in the parent equipment document.

### Competition Results

Purpose:

- Store competition, test, race, game, showcase, or assessment outcomes.

Included:

- Competition result list and detail.
- Core fields: event name, event date, competition type, discipline, level, location, placement, field size, score, notes, status.
- Links to preparation training records and related analysis sessions.

Excluded:

- Shared event catalog.
- Live scoring.
- Team tournament management.
- Judge sheet parsing as a required MVP dependency.

### Coach Notes

Purpose:

- Let coaches add structured feedback tied to athlete activity.

Included:

- Coach note list and detail.
- Core fields: coach UID, note type, body, visibility, status.
- Optional title, target skill, action items, tags.
- Links to training, journal, competition, analysis, or future academy context.

Excluded:

- Full messaging.
- Team announcement system.
- Threaded comments.
- Coach billing, scheduling, or payroll.

## 4. Existing Blaze App Contributions

### Training App

The Training app should contribute:

- Existing athlete/profile data as candidate `Athlete` records.
- Training sessions, tasks, completions, focus areas, notes, and simple metrics as `TrainingRecord` documents.
- Existing training summary values as inputs for `Athlete.summary`.
- Any equipment context, if currently stored, as candidate `EquipmentRecord` data or `extensions.blazeTraining`.
- Legacy IDs under `externalRefs.blazeTraining`.

Migration priority:

1. Preserve the current production model while inventorying fields.
2. Map the existing profile/main data to a canonical athlete profile without destructive migration.
3. Map completed or planned training items to `trainingRecords`.
4. Add compatibility reads before any cutover.

### Journal App

The Journal app should contribute:

- Reflections, prompts, notes, mood, confidence, energy, tags, and privacy state as `JournalEntry` documents.
- Parent or coach observations where they are currently journal-like records.
- Links to training or competition context when available.
- Legacy IDs under `externalRefs.blazeJournal`.

Migration priority:

1. Preserve privacy and visibility semantics first.
2. Map entries by athlete and date.
3. Keep private notes private during any dual-read period.
4. Move app-specific prompt metadata to `extensions.blazeJournal` unless it becomes a platform field.

### Analysis App

The Analysis app should contribute:

- Video, sensor, coach review, scoring, and comparison records as `AnalysisSession` documents.
- Findings, metrics, recommendations, and media references in bounded core fields.
- Heavy media and generated artifacts as storage references or future artifact documents.
- Links to training and competition context where available.
- Legacy IDs under `externalRefs.blazeAnalysis` or `externalRefs.blazeLab`.

Migration priority:

1. Inventory media storage paths and public/private URL behavior.
2. Map lightweight session metadata into `analysisSessions`.
3. Keep large payloads outside core documents.
4. Validate that analysis records can render without reading unrelated training or journal collections.

## 5. Excluded from MVP

The MVP should explicitly exclude:

- Full Lindii Academy runtime.
- Academy enrollment, progress tracking, quizzes, video lessons, and certificates.
- Reward engine and gamified leaderboards.
- Billing, subscriptions, invoicing, and payments.
- Marketplace for coaching, equipment, programs, or academy content.
- Organization and club management.
- Team roster management.
- Public social feeds, comments, likes, follows, or sharing pages.
- Notification system beyond app-local UI needs.
- Full calendar scheduling.
- Coach availability, booking, or payroll.
- AI-generated training plan automation.
- Production-grade analytics warehouse.
- Cross-tenant admin console.
- Destructive migration or forced cutover from existing Blaze apps.

These exclusions keep MVP focused on proving the unified athlete workspace and shared data model.

## 6. MVP Firestore Collections

MVP write collections:

```text
users/{uid}
users/{uid}/athletes/{athleteId}
users/{uid}/athletes/{athleteId}/trainingRecords/{recordId}
users/{uid}/athletes/{athleteId}/journalEntries/{entryId}
users/{uid}/athletes/{athleteId}/analysisSessions/{sessionId}
users/{uid}/athletes/{athleteId}/equipmentRecords/{equipmentId}
users/{uid}/athletes/{athleteId}/competitionResults/{competitionId}
users/{uid}/athletes/{athleteId}/coachNotes/{noteId}
```

MVP reserved but not required:

```text
academyPrograms/{programId}
rewardDefinitions/{rewardDefinitionId}
```

MVP excluded until Phase 2:

```text
users/{uid}/athletes/{athleteId}/academyEnrollments/{enrollmentId}
users/{uid}/athletes/{athleteId}/rewards/{rewardId}
users/{uid}/athletes/{athleteId}/analysisSessions/{sessionId}/artifacts/{artifactId}
organizations/{organizationId}
organizations/{organizationId}/members/{uid}
organizations/{organizationId}/athletes/{athleteId}
```

Cost rules:

- Athlete dashboard should read `users/{uid}`, one `Athlete`, and a bounded set of recent records.
- Use `Athlete.summary` for counts and latest timestamps.
- Use pagination for record lists.
- Avoid collection group queries in normal athlete workspace screens unless the feature explicitly needs cross-athlete data.

## 7. MVP User Flows

### Flow 1: Parent Creates or Opens an Athlete Workspace

1. Parent signs in.
2. Platform loads `users/{uid}`.
3. Platform lists `users/{uid}/athletes`.
4. Parent creates or opens one `Athlete`.
5. Platform shows profile summary and recent records.

Success condition:

- Parent can understand the athlete's recent training, journal, analysis, equipment, competition, and coach-note state from one workspace.

### Flow 2: Athlete Logs Training

1. Athlete or parent opens athlete workspace.
2. User creates a `TrainingRecord`.
3. Platform updates or queues update to `Athlete.summary`.
4. Training appears in recent activity and training list.

Success condition:

- Training data is available to Journal, Analysis, Competition Results, and Coach Notes through stable IDs.

### Flow 3: Athlete Writes Journal Entry

1. Athlete opens Journal module.
2. User creates a `JournalEntry`.
3. User selects visibility.
4. Entry optionally links to a training record or competition result.
5. Platform updates or queues update to `Athlete.summary`.

Success condition:

- Private and shared journal data remains clearly separated.

### Flow 4: Coach Reviews Athlete Context

1. Coach signs in.
2. Platform loads athletes where the coach has active access.
3. Coach opens one athlete workspace.
4. Coach reviews recent training, journal, competition, and analysis context.
5. Coach creates a `CoachNote` linked to the relevant record.

Success condition:

- Coach can add useful feedback without duplicating athlete data or copying record details manually.

### Flow 5: Analysis Session Is Added

1. User opens Analysis module.
2. User creates or imports an `AnalysisSession`.
3. Session stores summary, findings, recommendations, and lightweight media references.
4. Session optionally links to a training record or competition result.

Success condition:

- Analysis context appears in the athlete workspace without storing heavy media or raw analysis payloads in the core document.

### Flow 6: Competition Result Is Recorded

1. Parent, athlete, or coach opens Competition Results.
2. User creates a `CompetitionResult`.
3. User links preparation training records or follow-up analysis sessions where available.
4. Competition appears in athlete history.

Success condition:

- Competition context can be used by Journal, Analysis, Training, Coach Notes, and later Rewards.

### Flow 7: Equipment Is Tracked

1. Parent or athlete opens Equipment.
2. User creates an `EquipmentRecord`.
3. User marks equipment active, backup, maintenance, retired, or archived.
4. Training records can reference equipment IDs when relevant.

Success condition:

- Equipment context can explain changes in comfort, performance, maintenance, or injury-risk conversations without bloating training records.

## 8. MVP Success Criteria

Data model criteria:

- Active MVP records map cleanly to v1 platform entities.
- Every child record includes `ownerUid`, `athleteId`, `schemaVersion`, `createdAt`, `updatedAt`, and `sourceApp`.
- Legacy IDs are preserved in `externalRefs`.
- Product-specific leftovers are isolated under `extensions`.

Product criteria:

- Athlete, parent, and coach can use one athlete workspace as the shared context.
- MVP covers the core loop: profile, train, reflect, analyze, compete, review.
- Coach notes can reference at least training, journal, competition, and analysis records.
- Equipment can be tracked independently and referenced by training records.

Migration criteria:

- Existing Blaze Skate apps can continue operating during migration.
- No destructive migration is required for MVP validation.
- At least one representative athlete can be mapped from each current Blaze app into the platform shape.
- Migrated records can be validated against the corresponding schemas.

Firestore cost criteria:

- Athlete dashboard does not require reading all child collections.
- Record lists are paginated.
- Large media and analysis artifacts are not embedded in core Firestore documents.

Operational criteria:

- MVP scope is small enough to implement without building academy, billing, organization, marketplace, reward, or social systems.
- Security-rule design can be completed from the documented ownership and access model before production launch.

## 9. Migration Strategy from Current Three Blaze Apps

### Stage 1: Inventory

Create inventory documents for:

- Blaze Skate Training.
- Blaze Skate Journal.
- Blaze Skate Analysis.

Each inventory should capture:

- Current Firestore paths.
- Document shapes.
- Required and optional fields.
- Current identity and ownership assumptions.
- Date formats.
- Media references.
- Privacy or visibility rules.
- Known duplicate, stale, or derived fields.

### Stage 2: Canonical Athlete Mapping

Create or identify one canonical `Athlete` for each active skater.

Matching priority:

1. Existing stable athlete ID if available.
2. Explicit user-selected match.
3. Same owner UID plus high-confidence display-name match.
4. Manual review.

Do not auto-merge ambiguous athletes.

### Stage 3: Entity Mapping

Map existing app data to MVP entities:

| Source | MVP Entity |
| --- | --- |
| Training profile or skater profile data | `Athlete` |
| Training sessions, tasks, completions | `TrainingRecord` |
| Journal reflections, prompts, notes | `JournalEntry` |
| Analysis reviews, videos, findings | `AnalysisSession` |
| Equipment or gear fields | `EquipmentRecord` |
| Competition/test/race/game records | `CompetitionResult` |
| Coach feedback fields or notes | `CoachNote` |

Mapping rules:

- Preserve source IDs in `externalRefs`.
- Preserve source-specific fields under `extensions.{sourceApp}`.
- Do not duplicate full athlete profile data into child records.
- Do not place large media or generated artifacts directly in Firestore records.

### Stage 4: Validation

For each migrated representative document:

1. Validate required identity fields.
2. Validate dates.
3. Validate parent athlete exists.
4. Validate schema compatibility.
5. Validate privacy and visibility semantics.
6. Validate dashboard summary counts against child records within an accepted tolerance.

### Stage 5: Dual Read Before Cutover

Recommended sequence:

1. Existing app continues reading its current production data.
2. Migration process writes platform-shaped documents for selected athletes.
3. Platform MVP reads platform-shaped documents.
4. Existing apps may add read support behind a feature flag if needed.
5. Once verified, apps can dual-write or move default reads by module.
6. Legacy data becomes read-only only after product verification.

### Stage 6: MVP Cutover Readiness

Cutover is ready only when:

- Active athletes have canonical athlete profiles.
- Recent Training, Journal, and Analysis records validate.
- Dashboard summaries render without broad scans.
- Privacy and visibility are preserved.
- Rollback path is documented.
- Existing apps are not broken by the platform migration.

## 10. Phase 2 Features

Phase 2 candidates:

- Full Academy module.
- Academy enrollment and athlete progress.
- Reward engine and badge/certificate issuance.
- Reward definitions and versioning.
- Organization, club, and team management.
- Coach team dashboards.
- Shared event catalog for competitions and tests.
- Analysis artifact subcollections.
- AI-assisted technical review and recommendation workflows.
- Training plan templates and recurring schedules.
- Calendar and reminders.
- Notifications.
- Parent/coach messaging.
- Comments or discussions on coach notes.
- Advanced privacy and consent management.
- Billing and subscriptions.
- Public portfolio or recruiting profile.
- Aggregated analytics and reporting.
- Export tools for families, coaches, and clubs.

Phase 2 should be planned only after the MVP proves the shared athlete workspace, migration path, and Firestore cost model.
