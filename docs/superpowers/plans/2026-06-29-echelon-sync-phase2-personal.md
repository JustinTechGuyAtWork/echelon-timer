# Echelon Sync — Phase 2: Personal (Mine) Sync — Implementation Plan

> **For agentic workers:** Implement task-by-task. Binding design decisions are in the spec's **§14 (Pressure-test hardening)** — this plan implements them; if anything here conflicts with §14, §14 wins. Verification is in-browser (no test framework): the merge logic is stub-tested via `preview_eval`; true cross-device sync is **user-verified** (Claude cannot sign into the user's Google account). Build is reviewed adversarially before deploy.

**Goal:** Each signed-in user's own routines, plans, Moves catalog, and settings sync to their private cloud space (`users/{uid}/…`) and stay in sync across all their devices. Everything is "Mine" in this phase (the Team library is Phase 3). Existing local data is migrated up once, safely and idempotently. The app stays offline-first.

**Architecture (per spec §3 + §14):** The Firebase **bridge module** (ESM) gains Firestore *data operations* (subscribe / push / delete / user-doc / bulk-upload / serverTimestamp) so the classic-script IIFE never touches Firestore directly. A new **cloud-sync section** in the IIFE owns: (a) a one-way `syncApply()` that merges cloud changes into `state` **in place** + `save()` and **never** calls `markDirty`/`autoSave` (echo guard); (b) a **dirty-id Set** filled only by genuine user edits and drained by a separate debounced push; (c) **tombstones** for deletes; (d) **server-time ordering**; (e) a **guarded one-time migration** with a cloud-primary `migrationComplete` flag. Strict init order: localStorage→render→auth→(migration if first time)→subscribe→push.

**Tech Stack:** vanilla single-file HTML/JS; Firebase modular SDK (already loaded in Phase 1). No build step.

## Global Constraints
- All code in `tempo-studio/index.html`. Back up to `.bak-pre-phase2` before editing.
- **Offline-first preserved.** Firestore offline cache + localStorage both stay; localStorage is the render source.
- **Phase 2 = Mine only.** No Team collections, no sharing UI, no `scope` switching. Every synced item is the user's private data under `users/{uid}/…`. `scope` is backfilled to `"mine"` and carried, but unused until Phase 3.
- **Prereq:** the user must re-publish the Firestore rules with `echelonfit.com` filled in (the `users/{uid}` rule's `companyUser()` must pass, or every write is denied). Provided separately.
- Verification: stub-test merge logic in browser; deploy; user tests on two devices.

## Data shapes (cloud)
```
users/{uid}                    # doc: { email, displayName, settings:{…+updatedAt}, migrationComplete:true, updatedAt }
users/{uid}/routines/{id}      # the routine object + updatedAt + serverUpdatedAt + scope:"mine" + ownerEmail
users/{uid}/plans/{id}         # plan object + updatedAt + serverUpdatedAt
users/{uid}/catalog/{id}       # move object + updatedAt + serverUpdatedAt
```
Local adds: `tempo.tombstones.v1` = `{ id: deletedAtMs }`; per-item hidden `_syncedAt` (last server time seen); cached identity `tempo.auth.v1` (Phase 1).

---

### Task 1: Data-model prep — `updatedAt` everywhere, scope/owner backfill, tombstone helpers

**Files:** Modify `tempo-studio/index.html` (normalizeState, catalog mutators, settings writers; add tombstone helpers near Storage).

**Interfaces produced:** `markCatalogDirty(m)`; `tombstones()`, `addTombstone(id)`, `isTombstoned(id, cloudServerMs)`, `gcTombstones()`; routines/plans/catalog items always carry numeric `updatedAt`; `state.settings.updatedAt`.

- [ ] **Step 1: Backfill + stamp `updatedAt` on catalog moves and settings.**
  - In `normalizeState`: in the catalog `.map`, set `m.updatedAt = (typeof m.updatedAt==='number'? m.updatedAt : (m.createdAt||Date.now()))`. Add `if (typeof parsed.settings.updatedAt!=='number') parsed.settings.updatedAt = Date.now()`.
  - Add `function markCatalogDirty(m){ m.updatedAt = Date.now(); scheduleSave(); }` and route the catalog row edit callbacks (`buildMoveRow` name.oninput + the metric/dur/rest onChange) and `addMove`/`importMovesFromRoutines`/`applyMoveTemplate` through it (stamp `updatedAt`).
  - In `saveSettings` and `toggleTheme` (and any `state.settings` writer): set `state.settings.updatedAt = Date.now()` before save.
- [ ] **Step 2: scope/owner backfill in `normalizeState` (shape-only, idempotent).** For each routine/plan: `if (!r.scope) r.scope='mine'; if (typeof r.ownerEmail!=='string') r.ownerEmail=''; if (typeof r.ownerName!=='string') r.ownerName='';`. **Never overwrite an existing scope/ownerEmail** (so re-running over merged data is safe). Owner-stamping of empty `ownerEmail` happens in migration (Task 4), not here.
- [ ] **Step 3: Tombstone helpers** near Storage:
```javascript
  var TOMB_KEY = "tempo.tombstones.v1";
  function tombstones(){ try { return JSON.parse(localStorage.getItem(TOMB_KEY)) || {}; } catch(e){ return {}; } }
  function saveTombstones(t){ try { localStorage.setItem(TOMB_KEY, JSON.stringify(t)); } catch(e){} }
  function addTombstone(id){ var t = tombstones(); t[id] = Date.now(); saveTombstones(t); }
  function isTombstoned(id, cloudServerMs){          // true => suppress this cloud doc
    var t = tombstones(); if (!(id in t)) return false;
    return !(cloudServerMs && cloudServerMs > t[id]);  // a newer cloud write un-suppresses
  }
  function gcTombstones(){ var t=tombstones(), cut=Date.now()-30*24*3600*1000, ch=false;
    for (var k in t){ if (t[k] < cut){ delete t[k]; ch=true; } } if (ch) saveTombstones(t); }
```
- [ ] **Step 4: Verify (browser, no Firebase needed).** Reload; `preview_eval`: confirm a fresh catalog move and `state.settings` have numeric `updatedAt`; `addTombstone('x'); isTombstoned('x', 0)===true; isTombstoned('x', Date.now()+1)===false`. No console errors.

---

### Task 2: Bridge — expose Firestore data operations

**Files:** Modify the `<script type="module">` bridge.

**Interfaces produced (on `window.EchelonSync`):**
- `subscribeUser(uid, onChanges)` — attaches `onSnapshot(..., {includeMetadataChanges:true})` to `users/{uid}/routines`, `/plans`, `/catalog`. For each snapshot, calls `onChanges(collectionName, changes)` where `changes = snapshot.docChanges().map(c => ({ type:c.type, id:c.doc.id, data:c.doc.data(), hasPendingWrites:c.doc.metadata.hasPendingWrites, serverMs: (c.doc.data().serverUpdatedAt && c.doc.data().serverUpdatedAt.toMillis && c.doc.data().serverUpdatedAt.toMillis()) || null }))`. Returns an unsubscribe function.
- `subscribeUserDoc(uid, onDoc)` — `onSnapshot(doc(users/{uid}))` → `onDoc(data|null, hasPendingWrites)` (for settings + migrationComplete).
- `pushItem(uid, collection, id, data)` — `setDoc(doc(db, 'users', uid, collection, id), {...data, serverUpdatedAt: serverTimestamp()})`. Returns the (offline-tolerant) promise.
- `deleteItem(uid, collection, id)` — `deleteDoc(doc(db,'users',uid,collection,id))`.
- `setUserDoc(uid, data, {merge:true})` — `setDoc(doc(db,'users',uid), {...data}, {merge:true})`.
- `getUserDocOnce(uid)` — `getDoc(...)` → data|null (used to read migrationComplete authoritatively at first sign-in).
- `bulkUpload(uid, perCollectionItems, userDocPatch)` — chunked `writeBatch` (≤450 ops/chunk): set each `users/{uid}/{coll}/{id}` with `serverUpdatedAt`, and set `users/{uid}` with `userDocPatch` (incl. `migrationComplete:true`) in the **final** chunk; `await` each `commit()`.

- [ ] **Step 1: Add the imports** to the bridge: `collection, doc, getDoc, setDoc, deleteDoc, onSnapshot, writeBatch, serverTimestamp` from firebase-firestore.js.
- [ ] **Step 2: Implement the functions above** and attach them to `window.EchelonSync`.
- [ ] **Step 3: Verify (browser).** With a signed-in session is NOT required to confirm the SHAPE: `preview_eval` → `typeof window.EchelonSync.pushItem==='function'` etc. for all six. No console errors on load.

---

### Task 3: Cloud-sync module — merge (`syncApply`), dirty-set push, init order

**Files:** Modify IIFE — add a "Cloud sync" section; adjust Init.

**Interfaces produced:** `var me` (current `{uid,email,displayName}` or null); `dirty` Set + `enqueueSync(id, collection)`; `flushSync()` (debounced); `syncApply(collection, changes)`; `mergeItem(arr, item, serverMs)`; `startSync(user)`; `stopSync()`.

- [ ] **Step 1: The merge core** (`syncApply`) — the heart of correctness (spec §14 sync engine):
```javascript
  var me = null, unsub = [], dirty = new Set(), syncTimer = null;
  // Apply cloud changes into state IN PLACE; never mark dirty (echo guard).
  function syncApply(collection, changes) {
    var arr = collection==='routines' ? state.routines
            : collection==='plans'    ? state.sequences
            : collection==='catalog'  ? state.catalog : null;
    if (!arr) return;
    var touched = false;
    changes.forEach(function (c) {
      if (c.hasPendingWrites) return;                 // skip our own un-acked writes (echo)
      var idx = arr.findIndex(function (x){ return x.id === c.id; });
      if (c.type === 'removed') {
        if (idx >= 0) { arr.splice(idx,1); touched = true; }
        addTombstone(c.id);
        return;
      }
      // added | modified
      if (isTombstoned(c.id, c.serverMs)) return;      // locally deleted & cloud not newer
      var incoming = c.data; incoming._syncedAt = c.serverMs || 0;
      if (idx < 0) { arr.push(incoming); touched = true; }
      else {
        var local = arr[idx];
        var localServer = local._syncedAt || 0;
        // server-time ordering; fall back to client updatedAt only if no server time yet
        var newer = c.serverMs ? (c.serverMs > localServer)
                               : ((incoming.updatedAt||0) > (local.updatedAt||0));
        if (newer && currentEditId !== c.id) { arr[idx] = incoming; touched = true; }
        else if (newer) { /* defer: item open in builder — apply on close (Task 3 Step 4) */ deferMerge(collection, incoming); }
      }
    });
    if (touched) { save(); rerenderAfterSync(collection); }   // save: keep localStorage in sync; never markDirty
  }
  function rerenderAfterSync(collection){
    if (!$("view-library").classList.contains("hidden") && (collection==='routines'||collection==='plans')) renderLibrary();
    if (!$("view-catalog").classList.contains("hidden") && collection==='catalog') renderCatalog();
  }
```
- [ ] **Step 2: Dirty-set push** (decoupled from autosave; gated; never from syncApply):
```javascript
  function enqueueSync(id, collection){ if (me) { dirty.add(collection+"/"+id); scheduleFlush(); } }
  function scheduleFlush(){ clearTimeout(syncTimer); syncTimer = setTimeout(flushSync, 800); }
  function flushSync(){
    if (!me || !window.EchelonSync) return;
    dirty.forEach(function (key){
      var slash = key.indexOf("/"), coll = key.slice(0,slash), id = key.slice(slash+1);
      var item = findSyncItem(coll, id);
      if (item) window.EchelonSync.pushItem(me.uid, coll, id, stripMeta(item))
                  .catch(function(){ /* offline queues; rules-deny logged */ });
    });
    dirty.clear();
  }
  // stripMeta: drop hidden _syncedAt before pushing; ensure updatedAt present.
```
  Settings push: a `pushSettings()` → `setUserDoc(me.uid, { settings: state.settings, email:me.email, displayName:me.displayName })`.
- [ ] **Step 3: Wire local user-edits to `enqueueSync` + tombstones** (genuine edits only):
  - `markDirty()` (routine builder): after its debounce, also `enqueueSync(currentEditId,'routines')`.
  - `markPlanDirty()` → `enqueueSync(currentPlanId,'plans')`.
  - `markCatalogDirty(m)` / catalog add → `enqueueSync(m.id,'catalog')`.
  - `deleteRoutine(id)`: `addTombstone(id)` + `window.EchelonSync && me && EchelonSync.deleteItem(me.uid,'routines',id)` BEFORE the local filter. Same for `deletePlan`, `deleteMove`.
  - `saveSettings`/`toggleTheme`: `pushSettings()`.
- [ ] **Step 4: Defer-merge for the open item:** `var deferred = [];` `function deferMerge(coll,item){ deferred.push({coll,item}); }`. On leaving the builder/plan editor (btn-back / leaveCurrentEditor / closeRun n/a), drain `deferred` through the same merge path and re-render.
- [ ] **Step 5: Verify merge logic (browser, STUBBED — no Firebase).** `preview_eval`: call `syncApply('routines', [...])` with crafted changes and assert: added item appears; modified-newer replaces; modified-older ignored; removed deletes + tombstones; `hasPendingWrites:true` skipped; tombstoned id not re-added unless `serverMs` newer; open-item (`currentEditId`) deferred not applied. (Functions are in the IIFE — expose a tiny `window.__syncTest = { syncApply, state, addTombstone, tombstones }` guarded by a build flag, OR drive via the real subscribe in Task 6's user test. Prefer a temporary `window.__syncTest` hook removed before deploy.)

---

### Task 4: Migration — guarded one-time bulk upload + `migrationComplete`

**Files:** IIFE — `migrateToCloud(user)` called once in the auth success path.

- [ ] **Step 1:** `migrateToCloud(user)`:
  1. `var localFlag = localStorage.getItem("tempo.migrated."+user.uid)`. If set → skip (latency shortcut).
  2. `var cloud = await EchelonSync.getUserDocOnce(user.uid)`. If `cloud && cloud.migrationComplete` → set local flag, skip bulk upload (cloud authoritative — prevents 2nd-device re-upload/dup), proceed to subscribe.
  3. Else (first ever): owner-stamp every Mine item with empty `ownerEmail` → `user.email`/`displayName`; collect routines, plans, catalog (keyed by their existing `id`); call `EchelonSync.bulkUpload(user.uid, {routines, plans, catalog}, { email:user.email, displayName:user.displayName, settings:state.settings, migrationComplete:true })`.
  4. **Only after every `commit()` resolves**, set the local flag. (Offline at first sign-in: leave the flag unset; migration retries next online launch — bulkUpload is idempotent on item ids.)
- [ ] **Step 2: Verify (browser/stub):** simulate `migrateToCloud` twice → second run is a no-op (flag set); stub `getUserDocOnce` returning `{migrationComplete:true}` → skips bulk upload. Assert no duplicate state.

---

### Task 5: Lifecycle — startSync / stopSync / account-switch, wired into the Phase-1 gate

**Files:** IIFE — extend `startAuthGate`'s onAuthChange.

- [ ] **Step 1:** On a confirmed company user in onAuthChange (Phase 1):
```javascript
   var prev = cachedIdentity();
   if (prev && prev.uid !== user.uid) resetLocalForNewUser();   // account switch: clear Mine/catalog/settings (NOT Team — none yet)
   me = { uid:user.uid, email:user.email, displayName:user.displayName||user.email };
   gcTombstones();
   await migrateToCloud(user);            // guarded; first-time uploads, else skips
   startSync(user);                       // attach listeners + enable push
```
  `resetLocalForNewUser()`: `state.routines=[]; state.sequences=[]; state.catalog=[]; state.settings = defaultSettings(); save();` then it will repopulate from the new user's cloud via the initial subscribe snapshot.
- [ ] **Step 2:** `startSync(user)`: `stopSync()` first; `unsub.push(EchelonSync.subscribeUser(user.uid, syncApply)); unsub.push(EchelonSync.subscribeUserDoc(user.uid, applySettingsDoc));`. `stopSync()`: call all `unsub`, clear array, `me=null`, clear `dirty`.
- [ ] **Step 3:** On sign-out (Phase-1 handler): `stopSync()` before clearing the cached identity (leave localStorage data intact).
- [ ] **Step 4: Init order check:** confirm sequence is localStorage→render (existing)→auth resolves→`migrateToCloud`→`startSync`. Migration must finish before listeners attach (await it).

---

### Task 6: Review, stub-verify, deploy, cross-device test

- [ ] **Step 1: Adversarial review** of the built sync code (a workflow): merge correctness (deletion-resurrection actually prevented; echo guard; server-time vs client fallback; defer-merge), migration idempotency/race, account-switch, the dirty-set never fed by syncApply, offline behavior, and that the existing builder/run/export still work with the new fields. Fix findings.
- [ ] **Step 2: Stub-verify** all merge/migration assertions (Task 3 Step 5, Task 4 Step 2) pass; remove any `window.__syncTest` hook.
- [ ] **Step 3: Deploy** to echelon-timer (same flow as Phase 1).
- [ ] **Step 4: USER cross-device test:** sign in on Device A and Device B (both @echelonfit.com). On A: create a routine → it appears on B within seconds. Edit on B → updates on A. Delete on A → gone on B (and stays gone after reload). Change a setting / add a Move → syncs. Go offline on A, edit, come back online → syncs. Confirm existing routines uploaded on first sign-in and weren't duplicated.
- [ ] **Step 5: Confirm + record** in project memory; note Phase 3 (Team) is next.

## Verification approach (summary)
- **Algorithm correctness** (merge, tombstones, ordering, echo, migration idempotency) → browser stub-tests via a temporary `window.__syncTest` hook, removed before deploy.
- **End-to-end sync** → user, on two real devices (Claude can't sign in).
- **Safety net** → `git revert` + `.bak-pre-phase2`; Phase 2 is additive (gate + app unchanged if sync is disabled).
