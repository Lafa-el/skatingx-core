# Auth Model

## Goal

Define the identity and access assumptions for shared SkatingX data models without adding Firebase app code or security rules in this repository.

## Core Principle

Firebase Auth users and athlete profiles are separate entities.

A Firebase Auth user may be:

- An athlete using their own account
- A parent managing one or more athletes
- A coach viewing athletes they support
- A club admin managing a group
- A future platform admin

An athlete profile represents a skater. It should not be treated as the authentication account.

## Identity Fields

Use these fields consistently:

- `uid`: Firebase Auth user ID.
- `ownerUid`: user that owns the record or migrated data.
- `athleteId`: athlete profile ID.
- `createdByUid`: user that created a document.
- `updatedByUid`: user that last updated a document.

## Role Direction

Initial role values:

- `athleteOwner`
- `athlete`
- `parent`
- `coach`
- `clubAdmin`
- `platformAdmin`

Roles may appear at different scopes:

- User-level roles for global capabilities.
- Athlete-level access roles for a specific skater.
- Organization-level roles in future platform modules.

## Athlete Access Model

The athlete profile may include an `access` object:

```json
{
  "access": {
    "ownerUid": "uid-owner",
    "members": [
      {
        "uid": "uid-coach",
        "role": "coach",
        "permissions": ["read", "comment"],
        "status": "active"
      }
    ]
  }
}
```

For Firestore cost control, do not place large team or club membership lists inside every athlete document. If membership grows, move access relationships into a dedicated collection.

## Permission Concepts

Recommended permission values:

- `read`
- `write`
- `comment`
- `manageAccess`
- `archive`

Product apps should map UI capabilities to these permission concepts. Firestore rules can later enforce them.

## Visibility

Shared records include `visibility`:

- `private`
- `shared`
- `coachOnly`
- `team`

Visibility is a product-level data signal. It is not a replacement for Firestore security rules.

## Security Rule Direction

Future Firestore rules should enforce:

1. A user can read and write their own user document.
2. A user can read athlete documents they own or have active access to.
3. Child records inherit athlete-level access.
4. Writes must preserve immutable identity fields such as `ownerUid` and `athleteId`.
5. Platform admin access must be explicit and auditable.

This repository does not define final rules yet because rule implementation depends on actual app deployment topology.

## Privacy Direction

Athlete data may involve minors. Shared schemas should minimize sensitive fields.

Recommendations:

- Prefer `birthYear` and `ageGroup` over full date of birth unless required.
- Avoid storing home addresses in core athlete profiles.
- Keep medical, injury, or wellness notes in restricted product-specific records.
- Avoid public URLs for private media.

## Future Organization Model

Future SkatingX Platform may add:

```text
organizations/{organizationId}
organizations/{organizationId}/members/{uid}
organizations/{organizationId}/athletes/{athleteId}
```

Do not block this future model by assuming one athlete can only belong to one user forever.
