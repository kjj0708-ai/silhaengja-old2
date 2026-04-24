# Security Specification: The Doers (실행자들) App

## 1. Data Invariants
- **Identity & Role**: 
  - Standard users can only view notices, missions, transactions, and execute missions.
  - Only users registered in the `admins` collection can create/update/delete notices, missions, meetings, accounting, and manually update user attendance/points.
  - A user cannot modify another user's profile or points.
- **Mission Executions**:
  - A `mission_execution` must reference a valid `missionId`.
  - When a user submits a `mission_execution`, it MUST be submitted under their own `request.auth.uid`.
  - A user can only execute a mission once (enforced by ID constraints: `missionExecutions/{missionId}_{userId}`).
  - When creating a `mission_execution`, the user MUST batch-update their `users/{userId}` document to increment `totalPoints` atomically.
- **Attendance**:
  - Only `admins` can take attendance.
  - Adding attendance (`attendances`) must batch-update `totalPoints` if applicable, although Admins generally have broad access to update user points.
- **Transactions & Notices**:
  - Strictly admin-managed. Read-only for authenticated members.

## 2. Dirty Dozen Payloads
1. Unauthorized user tries to create an admin document `admins/{myUid}`. (Fails because `admins` collection cannot be written by non-admins).
2. User tries to execute a mission for another user `ownerId: targetUid`. (Fails: `incoming().ownerId == request.auth.uid`).
3. User tries to execute a mission without a valid `missionId`. (Fails: validation helper checks for valid string format, AND relation check `exists(/databases/.../missions/{payload.missionId})`).
4. User tries to modify an existing `mission_execution`'s points after submission. (Fails: `affectedKeys().hasOnly([...])` prevents updating points fields, or just no update allowed).
5. User tries to increment their `totalPoints` by 1,000,000 via a profile update. (Fails: User updates strictly controlled. If executing a mission, `existsAfter` checks if execution exists. Or simpler: only Admins can update `totalPoints`, OR users can increment `totalPoints` ONLY if `existsAfter(mission_execution)`. Wait, since users can only submit once, and the amount is fixed by the mission... This requires complex atomic checks `get(/databases/.../missions/...).data.autoPoints`. To simplify, let's have the user "submit" the mission, and a Cloud Function increments points, OR we trust the client increment *if* they create the `mission_execution` and the increment exactly matches. But actually, no backend logic is allowed other than Firestore Rules. So we must write atomic rules: `incoming().totalPoints <= existing().totalPoints + 30` and `existsAfter(/.../mission_executions/...)`.
6. Admin updates `users/{userId}` to give them an arbitrary score. (Admin overriding is explicitly allowed via `isAdmin()`).
7. User queries `users` to scrape PII. (App currently only stores `name`, `email` which might be public for the members list, but we should safely expose what is needed).
8. ID Poisoning: User sends `1MB_junk_string` as a `mission_execution` ID. (Fails: `isValidId`).
9. Shadow Update: User adds `isAdmin: true` into their user document. (Fails: `affectedKeys().hasOnly()` or schema definition rejects extra fields).
10. Spoofing: User tries to log in with an unverified email matching admin list. (Fails: `request.auth.token.email_verified == true`).
11. Denial of Wallet: Huge arrays. (Fails: `.size() <= MAX` for all strings and arrays).
12. Terminal State Lock: Attempt to re-submit or delete a submitted mission. (Fails: no `delete` access to `mission_executions` by users).

## 3. Test Runner
We will generate `firestore.rules.test.ts`.
