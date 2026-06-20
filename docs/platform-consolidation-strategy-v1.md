# SkatingX Platform Consolidation Strategy v1

## Goal

Determine whether SkatingX should remain as three separate applications sharing one Core, or evolve into a single unified SkatingX application.

This document is architecture review only. It does not define application code, UI implementation, Firebase SDK wrappers, Cloud Functions, Firestore security rules, or migration scripts.

## Executive Recommendation

SkatingX should evolve into one unified application, but not through a hard rewrite or immediate shutdown of the existing Blaze Skate apps.

Recommended direction:

1. Keep Blaze Skate Training, Blaze Skate Journal, and Blaze Skate Analysis operational during the transition.
2. Standardize all three apps on SkatingX Core schemas and shared Firestore-compatible records.
3. Build the unified SkatingX application around the canonical `User` and `Athlete` model.
4. Move users by workflow and data confidence, not by app boundary.
5. Retire separate apps only after the unified app covers their core workflows and migration risk is low.

The long-term product should be unified because athlete, parent, and coach workflows naturally cross Training, Journal, Analysis, Equipment, Competition Results, and Coach Notes. Keeping separate apps permanently would preserve short-term simplicity but create product fragmentation, repeated Firebase logic, duplicated UI patterns, harder coach workflows, and higher deployment overhead.

## 1. Current State of Each App

### Blaze Skate Training

Current responsibility:

- Daily and structured training records.
- Training session intensity, duration, focus areas, notes, and completion state.
- Athlete-like profile data that has historically lived in a compact app-specific model.

Current architecture signal:

- Existing work has prioritized preserving the current production Firestore shape while adding SkatingX-compatible metadata.
- The known Training compatibility constraint is to preserve the current `profile/main` model during integration work and avoid destructive collection moves.
- Training is the strongest candidate to become the first data contributor to the unified platform because it anchors the daily activity loop.

Risk profile:

- High product importance.
- Moderate migration risk because active users may depend on existing training flows.
- Technical debt exists around legacy shape compatibility and previous app-level coupling, but it is manageable through repository boundaries and additive metadata.

### Blaze Skate Journal

Current responsibility:

- Athlete reflections.
- Parent or coach observations.
- Mood, confidence, recovery, goals, prompts, and private notes.

Current architecture signal:

- Journal data maps cleanly to `JournalEntry`.
- Privacy and visibility semantics are more sensitive than the Training app because journal entries may contain private youth-athlete context.
- Journal should not be merged through broad profile-level denormalization; each entry should remain athlete-scoped and visibility-aware.

Risk profile:

- High privacy risk.
- Moderate product complexity.
- Migration must preserve private/shared/coach-only boundaries before any unified read surface is trusted.

### Blaze Skate Analysis

Current responsibility:

- Video or sensor-based technical review.
- Findings, scores, tagged clips, metrics, and recommendations.
- Media references and analysis summaries.

Current architecture signal:

- Analysis data maps to `AnalysisSession`.
- Core Firestore records should contain lightweight metadata only.
- Heavy media, frame data, transcripts, generated artifacts, and raw sensor payloads should stay in Storage or future artifact subcollections.

Risk profile:

- Highest data-size and media-risk profile.
- Moderate to high engineering risk if unified too early.
- Should be migrated after the platform proves athlete identity, access, and lightweight record rendering.

## 2. Shared Data Already Implemented

SkatingX Core already defines the shared foundation:

- `User` as the Firebase Auth identity anchor.
- `Athlete` as the domain anchor.
- `TrainingRecord`.
- `JournalEntry`.
- `EquipmentRecord`.
- `CompetitionResult`.
- `AnalysisSession`.
- `CoachNote`.
- `AcademyProgram`.
- `Reward`.

Shared data conventions already established:

- User-scoped athlete paths under `users/{uid}/athletes/{athleteId}`.
- Athlete-scoped subcollections for growing history.
- Required `ownerUid` and `athleteId` on child records.
- `schemaVersion`, `createdAt`, `updatedAt`, and `sourceApp` on persisted entities.
- `externalRefs` for legacy IDs.
- `extensions.{sourceApp}` for source-specific fields.
- Lightweight dashboard summaries on `Athlete.summary`.
- Storage or artifact references for large media and analysis payloads.

This means the strategic question is no longer whether the apps can share data. They can. The question is whether the user-facing product should remain split after the shared model exists.

## 3. User Workflows

### Cross-App Workflow Reality

Actual athlete development does not happen in separate product silos:

- A training session may produce a journal reflection.
- A competition result may trigger an analysis session.
- An analysis finding may become a coach note.
- A coach note may affect the next training focus.
- Equipment changes may explain training or competition changes.

The unified platform model represents this reality better than three disconnected app shells.

### MVP Workflow Anchor

The core MVP workflow should be:

```text
Open athlete workspace
  View profile and summary
  Review recent training, journal, analysis, equipment, competition, and coach notes
  Add or update the relevant record
  Preserve links between records through stable IDs
```

If the apps stay separate, users must mentally connect records across products. If the app becomes unified, the platform can connect them directly through `athleteId`, `ownerUid`, and related record IDs.

## 4. Mobile Usage Patterns

### Athlete Mobile Usage

Athletes are likely to use mobile in short sessions:

- Log training quickly after practice.
- Write a short reflection.
- Check coach notes.
- Review a recent analysis summary.

This favors one mobile-optimized app because switching between apps adds friction.

### Parent Mobile Usage

Parents are likely to check status and add context between other tasks:

- Confirm training consistency.
- Review journal or coach feedback.
- Track competition history.
- Track equipment status.

This favors a unified dashboard with focused module entry points.

### Coach Mobile Usage

Coaches may use mobile at the rink:

- Quickly review recent athlete context.
- Add notes after a session.
- Link feedback to a training or analysis record.

This also favors a unified athlete workspace. Coaches should not need to ask which app contains the relevant context.

### Mobile Conclusion

Mobile usage strongly favors Option B, one unified SkatingX application, as long as the interface stays modular and does not become a crowded all-in-one dashboard.

## 5. Coach Workflows

Coach workflows are inherently cross-module:

- Review recent training history.
- Read selected journal context when visibility permits.
- Review analysis findings.
- Add a coach note tied to a specific record.
- Track whether feedback changes future training.

Three apps create coach workflow problems:

- Repeated athlete lookup.
- Repeated authentication and access checks.
- Fragmented context.
- More chances for privacy inconsistencies.
- Harder cross-module feedback loops.

A unified app can make coach workflow simpler:

- One athlete access model.
- One coach-facing athlete workspace.
- Coach notes linked to training, journal, competition, and analysis records.
- Lower cognitive load when reviewing development history.

Coach workflow is one of the strongest arguments for unification.

## 6. Parent Workflows

Parent workflows also cross app boundaries:

- Manage athlete identity and profile details.
- Track training consistency.
- Understand journal/wellness context.
- Review coach feedback.
- Track competition outcomes.
- Monitor equipment needs.

Parents usually do not think in app modules. They think in terms of one skater's progress.

Three apps create parent workflow problems:

- Multiple places to check for the same athlete.
- Potential duplicate athlete/profile data.
- Harder setup for parent-managed youth athletes.
- More confusing privacy and access settings.

A unified app should present:

- One athlete switcher.
- One profile.
- One summary dashboard.
- Clear module tabs for Training, Journal, Analysis, Equipment, Competition Results, and Coach Notes.

Parent workflow favors unification, especially for families managing multiple athletes.

## 7. Technical Debt

### Debt If Three Apps Remain Permanent

Keeping three apps long-term creates recurring debt:

- Duplicate auth handling.
- Duplicate Firebase initialization patterns.
- Duplicate profile lookup and athlete access logic.
- Repeated schema adapters.
- Repeated validation and date handling.
- Divergent UI patterns.
- Harder cross-app testing.
- More deployments to coordinate.

Even if all apps share one Core package, separate app shells still need to interpret identity, access, and record relationships consistently.

### Debt If Unified Too Early

Building one app too early also creates debt:

- Large surface area in a new shell.
- Risk of rebuilding workflows before legacy behavior is understood.
- Risk of losing small app-specific interaction details.
- Risk of forcing migration before data quality is verified.
- Risk of one oversized app if module boundaries are weak.

### Debt Conclusion

Short-term debt is lower with separate apps. Long-term debt is lower with a unified app. The right strategy is phased unification after shared data contracts and migration inventories are stable.

## 8. Deployment Complexity

### Three Applications

Deployment advantages:

- Smaller deployments.
- Independent release timing.
- Lower blast radius per app.
- Easier rollback for one product area.

Deployment disadvantages:

- More Vercel projects or deployment targets.
- More environment variables to keep aligned.
- More Firebase configuration surfaces.
- More release coordination when shared Core changes.
- More chances for one app to lag behind schema changes.

### Unified Application

Deployment advantages:

- One primary web app deployment.
- One auth surface.
- One Firebase client integration pattern.
- One routing and role-based access strategy.
- Easier platform analytics and support.

Deployment disadvantages:

- Larger deployment blast radius.
- More need for feature flags.
- Stronger regression testing requirements.
- More careful release gating by module.

### Deployment Conclusion

For MVP, separate app deployments can remain while the unified app is introduced. Long term, one app should reduce operational overhead, provided module-level feature flags and test coverage exist.

## 9. Firebase Architecture

### Current Direction

The preferred platform Firestore layout is:

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

This structure supports either three apps or one app, but it naturally serves one athlete workspace.

### Firebase Implications of Three Apps

Pros:

- Each app can query only the collections it owns.
- Existing security and data assumptions can be preserved during transition.

Cons:

- Access logic must remain consistent across apps.
- Security rules may become harder to reason about if each app has slightly different behavior.
- Summary fields and cross-module links become harder to keep consistent.

### Firebase Implications of One App

Pros:

- One app can load the athlete profile once and then selectively load module records.
- One access model can drive all modules.
- Athlete summary fields can power one dashboard.
- Cross-record links become product features rather than migration artifacts.

Cons:

- Unified app must avoid reading every child collection on each dashboard load.
- Security rules must be designed carefully before production cutover.
- Feature flags and module-level loading are required to avoid cost spikes.

### Firebase Conclusion

The Firestore model supports both options, but its strongest long-term fit is a unified app with modular reads and athlete-scoped subcollections.

## 10. Long-Term Scalability

### Product Scalability

One unified app scales better as product scope grows:

- Academy integration.
- Rewards.
- Coach dashboards.
- Parent dashboards.
- Organization and club management.
- Shared competition/event catalogs.
- Analysis artifacts.

If the product stays split, each new cross-cutting capability must be integrated into multiple app shells.

### Engineering Scalability

One app scales better if it is internally modular:

- Shared routing and auth.
- Feature modules for Training, Journal, Analysis, Equipment, Competition, Coach Notes, Academy.
- Shared repository/data-access layer.
- Shared validation and type generation.
- Shared mobile-responsive design system.

One app scales poorly if it becomes a monolith without module boundaries. The recommendation is not "one giant app"; it is "one app shell with clear modules."

### Firestore Scalability

The platform can scale to 10000+ users if:

- Dashboards use summary fields.
- Lists are paginated.
- Records remain in athlete subcollections.
- Large media remains outside core documents.
- Collection group queries are reserved for deliberate cross-athlete workflows.
- Security rules enforce ownership and access consistently.

Long-term scalability favors one app backed by shared Core, not three apps that individually reimplement the same platform rules.

## Option A: Keep Three Applications Sharing One Core

### Description

Training, Journal, and Analysis remain separate user-facing applications. They share SkatingX Core schemas, Firestore model guidance, auth assumptions, and migration contracts.

### Pros

- Lowest immediate product disruption.
- Existing app-specific workflows can remain intact.
- Smaller per-app release surface.
- Lower short-term regression risk.
- Teams can migrate one app at a time.
- Easier to keep current users on familiar interfaces.

### Cons

- Users continue switching between apps.
- Coaches and parents lack one complete athlete workspace.
- Auth, access, Firebase setup, and profile logic remain duplicated.
- Cross-module workflows remain awkward.
- More deployment targets and release coordination.
- Higher long-term maintenance cost.
- Greater risk that apps drift away from shared Core over time.

### Migration Effort

Medium.

Required work:

- Add shared schema adapters to each app.
- Preserve current app paths while writing or reading platform-compatible records.
- Add `externalRefs`, `sourceApp`, and `schemaVersion`.
- Standardize profile/athlete identity mapping.
- Add compatibility defaults for legacy records.

The effort is manageable because each app can be updated independently.

### Engineering Risk

Low to medium short term.

Main risks:

- Data model drift between apps.
- Inconsistent privacy and access enforcement.
- Repeated fixes across app repositories.
- Shared Core changes requiring coordinated releases.

### User Impact

Low short-term disruption.

Negative long-term impact:

- Users still need multiple app entry points.
- Coaches and parents must assemble context manually.
- Platform value is less obvious because the experience remains fragmented.

## Option B: Build One Unified SkatingX Application

### Description

Build a single SkatingX app shell with module boundaries for Athlete Profile, Training, Journal, Analysis, Equipment, Competition Results, Coach Notes, and later Academy and Rewards.

### Pros

- Best fit for the unified athlete workspace.
- One auth and access model.
- One athlete switcher and profile surface.
- Better coach and parent workflows.
- Easier cross-module links and dashboards.
- Lower long-term deployment and maintenance overhead.
- Better foundation for Academy, Rewards, organizations, and platform analytics.
- Easier to enforce consistent Firestore read patterns.

### Cons

- Higher initial engineering effort.
- Larger regression surface.
- Requires careful feature flags and module boundaries.
- Risk of rebuilding legacy workflows incorrectly.
- Requires mature migration plan before users are moved.
- Could become a large monolith if architecture discipline is weak.

### Migration Effort

High.

Required work:

- Build a new unified app shell.
- Implement shared auth and athlete access.
- Implement module routing.
- Read platform-shaped Training, Journal, Analysis, Equipment, Competition, and Coach Notes data.
- Migrate or dual-read representative records from each existing app.
- Validate privacy and visibility rules.
- Add per-module cutover controls.
- Provide a rollback path while legacy apps remain operational.

### Engineering Risk

Medium to high during transition.

Main risks:

- Underestimating legacy app behavior.
- Incomplete data mapping.
- Privacy regressions in Journal and Analysis.
- Dashboard cost spikes from naive reads.
- Larger release blast radius.

Risk can be reduced by building the unified app after the Core model is stable and by cutting over module-by-module.

### User Impact

Medium short-term impact.

Positive long-term impact:

- One place to manage athlete development.
- Better mobile experience.
- Clearer coach and parent workflows.
- Less duplicate profile setup.
- Better cross-module progress history.

Negative short-term impact:

- Users may need onboarding.
- Some legacy workflows may temporarily remain in old apps.
- Power users may notice missing advanced app-specific features until later phases.

## Direct Comparison

| Area | Option A: Three Apps + Core | Option B: Unified SkatingX App |
| --- | --- | --- |
| Short-term speed | Better | Slower |
| Short-term risk | Lower | Higher |
| Long-term maintainability | Weaker | Stronger |
| Coach workflow | Fragmented | Strong |
| Parent workflow | Fragmented | Strong |
| Athlete mobile workflow | Multiple entry points | One entry point |
| Firebase consistency | Harder to enforce | Easier to enforce |
| Deployment simplicity | More targets | One primary target |
| Cross-module features | Harder | Natural |
| Academy/Rewards future | More integration work | Better foundation |
| Best use case | Transition period | Long-term platform |

## Recommendation

Choose Option B as the long-term strategy: build one unified SkatingX application.

Use Option A only as the transition strategy: keep the three existing apps alive while they write, read, or export data compatible with SkatingX Core.

The platform should not stay permanently split. The data model, MVP scope, coach workflows, parent workflows, mobile usage, and long-term Firebase architecture all point toward one unified app shell with modular internal boundaries.

The key architectural requirement is that the unified app must remain modular:

- `AthleteProfile` module.
- `Training` module.
- `Journal` module.
- `Analysis` module.
- `Equipment` module.
- `CompetitionResults` module.
- `CoachNotes` module.
- Later `Academy` and `Rewards` modules.

This keeps the product unified for users without turning the codebase into one undifferentiated monolith.

## Phased Roadmap

### Phase 0: Stabilize Existing Apps

Goal:

- Keep current users safe while preparing for platform migration.

Actions:

- Preserve current production Firestore paths.
- Inventory each app's current collections and document shapes.
- Fix blocking runtime, lint, and build issues before migration work.
- Add or preserve compatibility metadata such as `ownerUid`, `sourceApp`, `schemaVersion`, `createdAt`, and `updatedAt`.
- Avoid destructive data movement.

Exit criteria:

- Each app has an inventory document.
- Current app behavior is stable.
- Known legacy fields are mapped to platform entity candidates.

### Phase 1: Standardize on SkatingX Core

Goal:

- Make each app capable of producing or reading platform-compatible records.

Actions:

- Map Training data to `TrainingRecord`.
- Map Journal data to `JournalEntry`.
- Map Analysis data to `AnalysisSession`.
- Create or match canonical `Athlete` profiles.
- Preserve legacy IDs under `externalRefs`.
- Move app-specific leftovers under `extensions.{sourceApp}`.
- Validate representative records against the schema contracts.

Exit criteria:

- Representative active athletes can be expressed in the platform data model.
- Recent Training, Journal, and Analysis records validate.
- No existing app requires destructive migration to continue operating.

### Phase 2: Build Unified App Shell

Goal:

- Create the single SkatingX application foundation without replacing every legacy workflow immediately.

Actions:

- Implement one auth entry point.
- Implement athlete switcher and athlete workspace shell.
- Implement module routing for Profile, Training, Journal, Analysis, Equipment, Competition Results, and Coach Notes.
- Implement low-cost dashboard reads using `Athlete.summary`.
- Use feature flags per module.
- Keep legacy apps available as fallback.

Exit criteria:

- A user can open one athlete workspace and see platform-shaped records.
- Dashboard does not read all child collections.
- Access and visibility behavior is documented and testable.

### Phase 3: Module-by-Module Cutover

Goal:

- Move core workflows from old apps into the unified app safely.

Recommended order:

1. Athlete Profile.
2. Training.
3. Journal.
4. Analysis.
5. Equipment.
6. Competition Results.
7. Coach Notes.

Rationale:

- Profile and Training anchor the daily user loop.
- Journal follows after visibility behavior is validated.
- Analysis follows after media and artifact boundaries are clear.
- Equipment, Competition, and Coach Notes deepen the unified context.

Exit criteria:

- Each module can handle its MVP workflow in the unified app.
- Existing app users have a migration or fallback path.
- Data written by the unified app remains compatible with SkatingX Core.

### Phase 4: Retire Separate Apps Gradually

Goal:

- Reduce maintenance overhead after user workflows are stable in the unified app.

Actions:

- Freeze legacy apps to critical fixes.
- Stop adding new product features to old apps.
- Move default user entry points to SkatingX.
- Keep old apps read-only for a defined support window if needed.
- Archive old app-specific write paths after validation.

Exit criteria:

- Unified app covers core workflows.
- Support burden is lower in the unified app.
- Legacy apps no longer hold unique active workflows.

### Phase 5: Expand Platform Capabilities

Goal:

- Add features that only make sense after consolidation.

Candidates:

- Academy module.
- Rewards and certificates.
- Coach team dashboards.
- Organization and club management.
- Shared event catalog.
- Analysis artifact subcollections.
- Notifications and calendar reminders.
- Parent/coach messaging.
- Advanced reporting and exports.

Exit criteria:

- New Phase 2 features build on the unified app architecture instead of reintroducing app silos.

## Decision Guardrails

Use these rules when making future consolidation decisions:

- Do not merge app code before the data model is stable.
- Do not cut over Journal until privacy and visibility behavior is verified.
- Do not cut over Analysis until media and artifact storage boundaries are verified.
- Do not load every child collection for an athlete dashboard.
- Do not make Academy, Rewards, billing, or organizations part of the consolidation MVP.
- Do not retire an existing app until its core workflow exists in the unified app.
- Do not duplicate profile identity across modules; use `Athlete` as the domain anchor.

## Final Position

SkatingX should not remain permanently as three separate applications.

The correct end state is one unified SkatingX application backed by SkatingX Core, with strong internal module boundaries and a phased migration path from Blaze Skate Training, Blaze Skate Journal, and Blaze Skate Analysis.

The transition strategy should be conservative:

- Keep current apps stable.
- Standardize their data.
- Build the unified app around the athlete workspace.
- Cut over one module at a time.
- Retire old apps only when user workflows and data integrity are proven.
