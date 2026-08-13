# Echelon — Google Workspace Sign-in + Firebase Sync (Design Spec)

**Date:** 2026-06-29
**App:** Echelon — Routine Builder & Timer (`tempo-studio/index.html`, single-file, deployed to GitHub Pages at justintechguyatwork.github.io/echelon-timer)
**Status:** Design approved by user; pending spec review → implementation plan.

---

## 1. Goal

Add company Google sign-in and cloud sync to Echelon so that:
- **Shared team library** — instructors share a common set of routines/plans that everyone in the company can see and run.
- **Personal cross-device sync** — each person's own routines, plans, saved Moves, and settings follow them across their devices.

Login is restricted to the company's Google Workspace email domain. The app must keep working **offline** and stay at the **same URL**. This is an **additive** layer over the existing app — no rewrite of the builder/run/export code.

## 2. Confirmed decisions

| Decision | Choice |
| --- | --- |
| Platform | **Firebase** (Google Auth + Firestore) |
| Library model | **Team + Mine** — each routine/plan is either shared (Team) or private (Mine) |
| Share mechanism | A **"Share with team"** action moves a Mine item to Team (and "Make private" reverses it) |
| Team edit rights | **Owner edits/deletes; everyone else views/runs + "Save a copy to Mine"** |
| Moves catalog | **Personal** per user (synced across that user's devices, not shared) |
| Settings | **Personal** per user (synced) |
| Sign-in | **Required** to use the app (also keeps the public link company-only) |
| Conflict handling | **Last-write-wins** per item (whole routine/plan) |
| Admin | A small **admin email list** (starts with just the owner) that may edit/delete any Team item, for cleanup |
| Ownership transfer | **In v1** — the owner (or an admin) can reassign a Team item to a coworker by their **company email** |

## 3. Architecture

**Additive sync layer over the existing localStorage model.** The app continues to keep its working data in memory (`state`) and `localStorage` (`tempo.v1`) exactly as today; all existing screens (builder, run timer, exports, plans, catalog) are unchanged. A new **sync module** bridges local data to Firebase:

- On sign-in, it subscribes to the relevant Firestore collections and merges cloud data into local `state` + `localStorage`, then re-renders.
- On local change (the existing `markDirty`/`save` path), it pushes changed items up to Firestore.
- Firestore's built-in offline persistence queues writes while offline and flushes them when back online; `localStorage` remains the immediate local store so the app renders instantly and works with no connection.

The app never reads from Firestore directly to render — it always reads `state`/`localStorage`. The sync module is the only thing that talks to Firestore. This keeps the existing ~2,900-line app untouched and the new code isolated to one module + a sign-in gate.

**Components:**
- `auth` — Google sign-in, domain check, sign-in/out UI, "who am I".
- `sync` — Firestore read subscriptions + write push, per-item merge by `id` + `updatedAt`.
- `library UI` — Team/Mine tabs + share/copy/make-private actions (small additions to the existing library + card kebab menu).
- `migration` — first-sign-in upload of existing local data to the user's Mine space.

The Firebase SDK is loaded from Google's CDN (the app already requires internet for first sign-in; offline use after that is preserved by Firestore's cache + localStorage).

## 4. Authentication

- **Provider:** Firebase Authentication with the Google provider ("Sign in with Google").
- **Gate:** when the app loads and no user is signed in, show a sign-in screen; the rest of the app is hidden until signed in. Auth state persists on the device (one-time per device).
- **Domain restriction (enforced twice):**
  1. **Client:** after sign-in, check the user's email domain; if it isn't `@COMPANYDOMAIN`, sign them out and show "Use your company email."
  2. **Server:** Firestore security rules require `request.auth.token.email` to end with `@COMPANYDOMAIN` for every read/write — so a non-company user can't reach data even if they bypass the client.
  - (The exact domain is collected from the user during setup and dropped into both the client check and the rules.)
- **Identity used:** Firebase `uid` is the key for a user's private data; `email` and `displayName` are stored as fields and shown in the UI / as the owner label on Team items.
- **Sign-out:** in Settings — clears the session; local cached data remains on the device but won't sync until sign-in again.
- **Offline:** if the user has signed in before on this device, the app opens offline from cache; sign-in itself requires being online at least once.

## 5. Data model

Each synced item carries the app's existing internal `id`, an `updatedAt` timestamp (ms), and (for routines/plans) `scope` (`"team"` | `"mine"`) and ownership fields. **`ownerEmail` is the authoritative owner** (Team edit-permission is "your email matches `ownerEmail`, or you're an admin") — this is what makes ownership transferable by email alone, with no need to look up a user's account id. `ownerName` is stored for display.

**Local (`state` / `localStorage`) — unchanged shape, two new fields per routine/plan:**
- Routine/plan gains `scope` and `owner*` fields. `normalizeState` backfills existing items as `scope:"mine"`, owner = current user (on first sign-in).
- `state.routines` and `state.sequences` hold **both** Team and Mine items, merged; the library filters by `scope` for the two tabs. Builder/run/export treat a routine as a routine regardless of scope.

**Firestore structure:**
```
teamRoutines/{id}            # shared routines  (payload + owner* + updatedAt)
teamPlans/{id}               # shared plans
users/{uid}                  # doc: { email, displayName, settings:{…}, updatedAt }
users/{uid}/routines/{id}    # this user's private routines
users/{uid}/plans/{id}       # this user's private plans
users/{uid}/catalog/{id}     # this user's personal Moves
admins/{email}               # (optional) admin allow-list, or hardcoded in rules
```
- **Sharing** moves a routine from `users/{uid}/routines/{id}` → `teamRoutines/{id}` (scope flips to `team`); **Make private** reverses it. The item keeps its `id`.
- The sync module subscribes to: the user's own subcollections, the user doc (settings), and the two team collections.

## 6. Library UI

- The Class Library gains a **Team / Mine** filter — two tabs, with **Team shown by default** (the shared library is the common case). No "All" tab in v1.
- Each card shows a small **scope badge** (Team/Mine) and, for Team items, the **owner's name**.
- Card actions (in the existing kebab menu) become scope-aware:
  - **Mine item:** Edit · Export · Duplicate · Delete · **Share with team**.
  - **Team item you own:** Edit · Export · Duplicate · Delete · **Make private** · **Transfer ownership…**.
  - **Team item you don't own:** Run/View · Export · **Save a copy to Mine** (no Edit/Delete; the Edit entry is hidden). An **admin** additionally sees Edit/Delete/**Transfer ownership…** on any Team item.
- **Transfer ownership:** choosing it prompts for a coworker's **company email** (validated against the company domain). It sets the item's `ownerEmail` to that address — the item stays in the shared Team library; only who can edit it changes. The new owner gets edit rights the moment their email matches (i.e. on their next sync). Only the current owner or an admin can transfer, so nobody can grab ownership of someone else's item.
- A non-owner who tries to edit a Team routine is prompted to **"Save a copy to Mine to edit"** — there is no read-only builder; you always edit a copy you own.
- Plans follow the same Team/Mine model. When sharing a plan whose routines are still private, **prompt: "This plan uses N private routines — share those too?"** (Yes shares them; No shares the plan anyway and others simply see fewer steps). The existing dangling-id handling already drops unresolved routines safely.

## 7. Sync & offline behavior

- **Pull:** real-time Firestore listeners on the subscribed collections. On any change, merge into local by `id`: take the cloud version if local has no such `id` or `cloud.updatedAt > local.updatedAt`; deletions remove the local item. Re-render the affected view.
- **Push:** the existing debounced autosave also enqueues changed items to Firestore (Team items → team collection if the user is owner/admin; Mine items → the user's space). Firestore's offline queue handles writes made while offline.
- **Conflict:** last-write-wins at the item level. Owner-only editing of Team items makes real conflicts rare; for Mine items only the one user (across their own devices) edits.
- **Offline:** `localStorage` keeps the app fully functional offline; Firestore syncs on reconnect.

## 8. Migration of existing data

- On a user's **first sign-in on a device that has local routines**, upload all existing routines/plans as **Mine** (owner = this user), and upload the catalog + settings to the user's space. They then sync to the user's other devices.
- **Edge case:** the same routine built separately on two devices has different `id`s, so both upload and both are kept (possible duplicate) — the user can delete the extra. No data is lost. Documented, not auto-deduped in v1.

## 9. Permissions & security rules (sketch)

Firestore rules (pseudocode; `@COMPANYDOMAIN` and admin emails filled in at setup):
```
// NOTE: the matches('.*@DOMAIN$') sketch below was REPLACED after the security
// pressure-test — a regex domain check is unanchored with unescaped dots and lets
// lookalike domains through. Use EXACT domain comparison + email_verified. The full
// hardened, paste-ready rules live in the implementation plan / firestore.rules.
function companyUser()   = request.auth != null
                           && request.auth.token.email_verified == true
                           && request.auth.token.email.lower().split('@')[1] == 'COMPANYDOMAIN';
function isAdmin()        = request.auth.token.email.lower() in [ 'owner@COMPANYDOMAIN' ];
function ownsTeam()      = resource.data.ownerEmail.lower() == request.auth.token.email.lower();
function validNewOwner() = request.resource.data.ownerEmail.lower().split('@')[1] == 'COMPANYDOMAIN';

match /users/{uid}/{document=**} {
  allow read, write: if companyUser() && request.auth.uid == uid;          // your own private space
}
match /teamRoutines/{id} {
  allow read:   if companyUser();                                          // everyone in the company sees Team
  allow create: if companyUser()
                   && request.resource.data.ownerEmail == request.auth.token.email;
  // covers both editing AND transfer; the (new) owner must always be a company email
  allow update: if companyUser() && (ownsTeam() || isAdmin()) && validNewOwner();
  allow delete: if companyUser() && (ownsTeam() || isAdmin());
}
match /teamPlans/{id} { /* same as teamRoutines */ }
```
- The Firebase **config object** (apiKey, projectId, etc.) is embedded in the public `index.html`. This is standard and **not a secret** — security is enforced entirely by Auth + these rules.

## 10. One-time admin setup (done once, by the owner; ~15 min)

I provide a numbered checklist; the user does:
1. Create a free Firebase project.
2. Enable the **Google** sign-in provider (and, in the OAuth config, the company Workspace domain).
3. Create a **Firestore** database (production mode).
4. Add `justintechguyatwork.github.io` to Firebase **Authorized domains**.
5. Paste in the **security rules** I supply (with the domain + admin email filled in).
6. Copy the **Firebase config snippet** and send it to me (or paste where indicated).

## 11. Non-goals (v1, YAGNI)

Explicitly out of scope for the first version: real-time presence/"who's online", comments, multiple roles beyond owner+admin, an in-app admin-management UI, per-field merge / conflict resolution UI, audit logs, sharing to specific people/groups (Team is all-or-nothing), and a shared Moves catalog (catalog stays personal per the decision).

## 12. Risks & notes

- **Bigger change than prior edits:** adds login + a database to an offline file. Mitigated by the additive-layer architecture (existing code untouched).
- **First sign-in requires internet;** offline-first is preserved afterward.
- **Duplicate-on-migration** edge case (section 8) — accepted, documented.
- **Admin list is hardcoded** in rules/app for v1 — changing admins means a small edit + redeploy; a UI can come later.
- **Free tier is ample:** Firebase free plan supports up to 3,000 daily sign-ins and tens of thousands of DB ops/day — far beyond a studio team; the app stays free.

## 13. What stays the same

Same URL, same offline-first feel; builder, run timer, exports, plans, personal catalog, Cycle default, theming — all unchanged. New code is isolated to the auth gate, the sync module, and small scope-aware additions to the library.

## 14. Pressure-test hardening (2026-06-29)

An adversarial multi-agent review of this design (security rules, sync correctness, offline/lifecycle, migration, app integration), with findings verified against Firebase docs, produced the following **binding corrections**. These supersede any conflicting detail above and are the source of truth for the implementation plan.

**Security / auth**
- **Domain check by EXACT match, not regex.** `request.auth.token.email_verified == true && request.auth.token.email.lower().split('@')[1] == 'COMPANYDOMAIN'` (or `.lower().endsWith('@COMPANYDOMAIN')`). The `matches('.*@DOMAIN$')` form is unanchored with unescaped dots → lookalike-domain bypass. Same fix in `validNewOwner` and lowercase both sides of `ownsTeam`/`isAdmin`.
- **Validate writes:** on Team create/update require `scope=='team'`, `keys().hasAll(['ownerEmail','scope','updatedAt'])`, `ownerEmail` is a company email, and bound `updatedAt <= request.time + 5min` (stops last-write-wins poisoning by a future timestamp). Add an explicit terminal `match /{document=**} { allow read,write: if false }`.
- **Make-private is delete-from-`teamRoutines` + create-under-`users/{uid}`**, never an in-place scope flip (the doc physically changes collection). Same for Share (reverse) — done as a single `writeBatch`.
- **Auth persistence = `browserLocalPersistence`** (offline launch). Require the Google provider be the only enabled provider so `email_verified` is meaningful.
- **Email recycling** (a recycled `@company` address inherits Team ownership) is an **accepted v1 risk**: admins reassign a departing person's Team items before their address is recycled; v2 adds an immutable `ownerUid` secondary key. **Admin list stays hardcoded** in the rules for v1. The Firebase web config is public/non-secret (confirmed) — safe to embed in `index.html`.
- **Account switch:** if a *different* uid signs in on a device, reset local Mine/catalog/settings before subscribing as the new user (Team items are shared, keep them). Namespace the cached-identity + per-uid state.

**Sync engine**
- **Tombstones for deletes** (`tempo.tombstones.v1` = {id: deletedAtMs}). Local delete records a tombstone *before* removing the item (durable across the unload-flush) and pushes the cloud delete. The merge **consumes `snapshot.docChanges()`**, treats `type:'removed'` as delete+tombstone, and **skips any cloud doc whose id is tombstoned unless the cloud doc is newer**. GC tombstones after ~30 days. Without this, deletes silently resurrect — the single most important fix.
- **Server-time ordering.** Every push also writes `serverUpdatedAt: serverTimestamp()`; the merge winner is decided by server time (track a local `_syncedAt` per item), never by the client `Date.now()` (kept only for the "recently edited" sort). Client `updatedAt` is bounded by rules.
- **Echo guard (structural).** Cloud→local goes through a dedicated `syncApply()` that mutates state **in place** + `save()` and **never** calls `markDirty`/`autoSave`. Listeners use `includeMetadataChanges:true` and **skip changes with `metadata.hasPendingWrites === true`** (the app's own un-acked writes).
- **Atomic moves.** Share / Make-private use a single `writeBatch` (set in one collection + delete in the other). Optimistic local flip; error toast only if commit rejects.
- **Decoupled push.** A dirty-id `Set`, populated only by genuine user edits (never by `syncApply`), drained by a separate push debounce — never push whole state. Gate Team pushes on `ownerEmail===me || isAdmin`.

**Offline / lifecycle / init order**
- **Multi-tab cache:** `initializeFirestore(app, { localCache: persistentLocalCache({ tabManager: persistentMultipleTabManager() }) })` — never `enableIndexedDbPersistence()`.
- **Strict init order:** load localStorage → render → resolve auth via `onAuthStateChanged` → run **guarded one-time migration BEFORE attaching listeners** → attach listeners + start push. App shell hidden by default (CSS) until auth resolves to avoid a flash; offline launch shows the app from a cached-identity flag, not `auth.currentUser`.
- **Defer merging the open item:** if a cloud change arrives for `currentEditId` while the builder is open, queue it and apply on close.

**Migration / data integrity**
- **Cloud-primary `migrationComplete` flag** on `users/{uid}` (written in the same batch as the bulk upload). Once set, never bulk-upload again — only pull+merge. A localStorage per-uid flag is a latency shortcut only; the cloud flag is authoritative (prevents a second device re-uploading/duplicating).
- **Batched, idempotent migration** keyed by existing item ids (re-run overwrites the same docs); chunk at the 500-op limit; set the flag as the last op; only set the local skip-flag after `commit()` resolves (server ack).
- **`updatedAt` on catalog Moves and settings** (they currently have none) — backfill + stamp on every mutation; whole-doc LWW for settings.
- **Restore-from-file becomes Mine-only:** re-stamp `updatedAt`, force `scope:'mine'` + `ownerEmail = me`, strip any foreign scope/owner, push through the normal Mine path. `backup()` exports only Mine + settings + personal catalog (not merged Team items).
- **`normalizeState` stays shape-only:** backfill `scope:'mine'` + empty `ownerEmail`/`ownerName` sentinels when absent, **never overwrite existing** scope/owner. Owner-stamping of unclaimed Mine items happens once in the post-auth migration step (covers ClassFlow imports and pre-sign-in items).

**App integration**
- **One ownership-resetting copy helper** for both Duplicate and "Save a copy to Mine": new id, `scope:'mine'`, `ownerEmail=me`, `updatedAt=now`. (Fixes `duplicateRoutine`/`duplicatePlan` deep-copying scope+owner.)
- **`showBuilder` is the edit boundary:** at its top, if a Team item isn't owned by me and I'm not admin → run "Save a copy to Mine" and open the builder on the copy (covers every entry path, not just the kebab).
- **Prune gating:** the auto-discard-empty-draft call sites prune only if `scope!=='team' && ownerEmail===me`.
- **Scope-split library:** `renderLibrary` filters by the active Team/Mine tab first, then tag chips/sort/empty-state derive from that scoped list (Team empty-state has no "create" button — sharing populates Team).
- Read/render path (buildSteps, run engine, exports, plan resolution, routineDuration) needs **no change** — it ignores the new fields cleanly.

**Phased rollout (chosen):** (1) Sign-in gate → (2) Personal (Mine) sync → (3) Team library + sharing → (4) Transfer + admin + backup rework. Each phase is independently shippable and testable; each gets its own implementation plan.
