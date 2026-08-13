# Echelon Sync — Phase 1: Sign-in Gate — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add required "Sign in with Google" (restricted to the company Workspace domain) in front of Echelon, so opening the app shows a sign-in screen and only company accounts get in — with offline launch for returning users. No data syncs yet (that's Phase 2); the app stays localStorage-only behind the gate.

**Architecture:** A new `<script type="module">` "Firebase bridge" loads the modular Firebase SDK from Google's CDN, initializes Auth + Firestore (multi-tab offline cache, `browserLocalPersistence`), and exposes a tiny `window.EchelonSync` API (`signIn`, `signOut`, `onAuthChange`, `domain`). The existing classic-script IIFE stays as-is and gains a small **auth-gate** section that hides the app until a company user is signed in (or a cached identity exists, for offline launch). Firestore is initialized but unused in Phase 1.

**Tech Stack:** Vanilla single-file HTML/CSS/JS (no build step). Firebase JS SDK v10/v11 **modular** API via `https://www.gstatic.com/firebasejs/...` ESM imports. Verification is **in-browser** via the preview tools (no unit-test framework exists in this project).

## Global Constraints

- **Single file:** all code lives in `tempo-studio/index.html`. No new runtime files except the deployed `firestore.rules` (lives in the Firebase project, not the app).
- **Offline-first preserved:** the app must still open and run with no network for a previously-signed-in user. First sign-in requires network.
- **Same URL:** deploys to the existing `echelon-timer` GitHub Pages repo; the app stays at `justintechguyatwork.github.io/echelon-timer`.
- **One domain constant:** the company domain string is defined ONCE (in the bridge `CONFIG` block) and read everywhere (`window.EchelonSync.domain`); it must equal the `COMPANYDOMAIN` in the deployed rules.
- **Firebase SDK version:** use a recent stable `10.x` or `11.x` modular build; the API names used here (`initializeFirestore`, `persistentLocalCache`, `persistentMultipleTabManager`, `browserLocalPersistence`, `onAuthStateChanged`, `signInWithPopup`) are stable across both. Pin one version string in the imports.
- **No secrets:** the Firebase web config (`apiKey` etc.) is public/non-secret and is committed in `index.html`.
- **Verification = browser:** each code task ends by loading the app in the preview, checking the console and DOM, and committing. Real Google sign-in is verified by the user (Claude cannot sign into the user's Google account); gate *logic* is verified by stubbing `window.EchelonSync` in the preview.
- **Backup before edits:** keep a `.bak` copy of `index.html` before the first edit (as in prior sessions).

## Phase roadmap (this plan = Phase 1 only)

1. **Sign-in gate** (this plan) — Google login required; company-domain only; offline launch.
2. **Personal (Mine) sync** — per-item Firestore sync of the user's own routines/plans/catalog/settings; tombstones; server-time ordering; echo guard; migration with `migrationComplete` flag; account-switch reset.
3. **Team library + sharing** — Team/Mine tabs; `teamRoutines`/`teamPlans`; Share / Save-a-copy; `showBuilder` owner gate; prune gating; scope-split library.
4. **Transfer + admin + backup rework** — ownership transfer; admin overrides; Mine-only backup/restore.

Each later phase gets its own detailed plan once the prior phase is verified live.

---

### Task 1: Firebase project setup + security rules (USER does this; Claude provides exact steps + the rules)

**Files:**
- Create (in the Firebase Console, not the repo): a Firebase project, Firestore database, deployed security rules.
- Deliverable to Claude: the Firebase web config object + the confirmed company domain.

**Interfaces:**
- Produces: `firebaseConfig` object (apiKey, authDomain, projectId, etc.) and `COMPANY_DOMAIN` string (bare domain, e.g. `echelonfit.com`) — consumed by Task 2.

- [ ] **Step 1: User creates the project (give them these exact steps)**

1. Go to https://console.firebase.google.com → **Add project** → name it (e.g. "echelon-timer") → you can disable Google Analytics → Create.
2. **Authentication** → Get started → **Sign-in method** → enable **Google** only (set a support email) → Save. (Do NOT enable Email/Password — Google-only keeps `email_verified` meaningful.)
3. **Authentication → Settings → Authorized domains** → Add domain → `justintechguyatwork.github.io`. (`localhost` is there by default for testing.)
4. **Firestore Database** → Create database → **Production mode** → pick a region → Enable.
5. **Project settings (gear icon) → General → Your apps → Web (`</>`)** → register an app (nickname "echelon-web") → copy the **`firebaseConfig`** object shown. Send it to Claude.
6. Tell Claude your **company email domain** (the bare domain after the `@`, e.g. `echelonfit.com`).

- [ ] **Step 2: User deploys the security rules**

Firestore Database → **Rules** tab → replace everything with the block below → set the two placeholders → **Publish**.

Replace `COMPANYDOMAIN` with your bare domain (e.g. `echelonfit.com`) and `ADMIN_EMAILS` with your lowercase admin email(s).

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function companyUser() {
      return request.auth != null
          && request.auth.token.email_verified == true
          && request.auth.token.email is string
          && request.auth.token.email.lower().split('@')[1] == 'COMPANYDOMAIN';
    }
    function callerEmail() { return request.auth.token.email.lower(); }
    function isAdmin() { return callerEmail() in [ 'ADMIN_EMAILS' ]; }
    function validNewOwner() {
      return request.resource.data.ownerEmail is string
          && request.resource.data.ownerEmail.lower().split('@')[1] == 'COMPANYDOMAIN';
    }
    function ownsTeam() {
      return resource.data.ownerEmail is string
          && resource.data.ownerEmail.lower() == callerEmail();
    }
    function ownerIsSelf() {
      return request.resource.data.ownerEmail is string
          && request.resource.data.ownerEmail.lower() == callerEmail();
    }
    function validUpdatedAt() {
      return request.resource.data.updatedAt is number
          && request.resource.data.updatedAt <= request.time.toMillis() + 300000;
    }
    function validTeamShape() {
      return request.resource.data.scope == 'team'
          && request.resource.data.keys().hasAll(['ownerEmail', 'scope', 'updatedAt']);
    }

    match /users/{uid}/{document=**} {
      allow read, write: if companyUser() && request.auth.uid == uid;
    }
    match /teamRoutines/{id} {
      allow read:   if companyUser();
      allow create: if companyUser() && ownerIsSelf() && validNewOwner() && validTeamShape() && validUpdatedAt();
      allow update: if companyUser() && (ownsTeam() || isAdmin()) && validNewOwner() && validTeamShape() && validUpdatedAt();
      allow delete: if companyUser() && (ownsTeam() || isAdmin());
    }
    match /teamPlans/{id} {
      allow read:   if companyUser();
      allow create: if companyUser() && ownerIsSelf() && validNewOwner() && validTeamShape() && validUpdatedAt();
      allow update: if companyUser() && (ownsTeam() || isAdmin()) && validNewOwner() && validTeamShape() && validUpdatedAt();
      allow delete: if companyUser() && (ownsTeam() || isAdmin());
    }
    match /{document=**} { allow read, write: if false; }
  }
}
```

(Phase 1 doesn't read/write Firestore, so these rules aren't exercised yet — deploying now means Phase 2 is ready and the rules are reviewed once.)

- [ ] **Step 3: Verify the handoff**

Claude confirms the received `firebaseConfig` has the keys `apiKey`, `authDomain`, `projectId`, `appId` (others optional) and that the domain is a bare domain (no `@`, no scheme). No commit (no code yet).

---

### Task 2: Firebase bridge module (init Auth + Firestore, expose `window.EchelonSync`)

**Files:**
- Modify: `tempo-studio/index.html` — add a `<script type="module">` just before the existing classic `<script>` (the IIFE) near the end of `<body>`.

**Interfaces:**
- Consumes: `firebaseConfig` + `COMPANY_DOMAIN` from Task 1.
- Produces: `window.EchelonSync = { auth, db, domain, signIn(), signOut(), onAuthChange(cb) }` and a `window`-level `"echelon-sync-ready"` event. `onAuthChange(cb)` calls `cb({uid, email, displayName, emailVerified})` or `cb(null)`. Consumed by Tasks 4 & 5.

- [ ] **Step 1: Back up index.html**

```bash
cp /Users/eleonard/Documents/Cluade/tempo-studio/index.html /Users/eleonard/Documents/Cluade/tempo-studio/index.html.bak-pre-phase1
```

- [ ] **Step 2: Add the bridge module script** (immediately before the existing `<script>` that opens the IIFE)

```html
<!-- ===== Firebase bridge (ES module): auth + firestore init, exposed to the app ===== -->
<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.13.0/firebase-app.js";
  import { getAuth, GoogleAuthProvider, signInWithPopup, signOut,
           onAuthStateChanged, setPersistence, browserLocalPersistence }
    from "https://www.gstatic.com/firebasejs/10.13.0/firebase-auth.js";
  import { initializeFirestore, persistentLocalCache, persistentMultipleTabManager }
    from "https://www.gstatic.com/firebasejs/10.13.0/firebase-firestore.js";

  // ---- CONFIG (paste from Task 1) ----
  const COMPANY_DOMAIN = "COMPANYDOMAIN";              // bare domain, e.g. "echelonfit.com"
  const firebaseConfig = { /* PASTE firebaseConfig FROM TASK 1 */ };
  // ------------------------------------

  const app  = initializeApp(firebaseConfig);
  const auth = getAuth(app);
  const db   = initializeFirestore(app, {
    localCache: persistentLocalCache({ tabManager: persistentMultipleTabManager() })
  });
  try { await setPersistence(auth, browserLocalPersistence); } catch (e) { /* non-fatal */ }

  const provider = new GoogleAuthProvider();
  provider.setCustomParameters({ hd: COMPANY_DOMAIN });   // hints (does NOT enforce) the domain in the chooser

  window.EchelonSync = {
    auth, db, domain: COMPANY_DOMAIN,
    signIn:  () => signInWithPopup(auth, provider),
    signOut: () => signOut(auth),
    onAuthChange: (cb) => onAuthStateChanged(auth, (u) =>
      cb(u ? { uid: u.uid, email: u.email, displayName: u.displayName, emailVerified: u.emailVerified } : null)),
  };
  window.dispatchEvent(new Event("echelon-sync-ready"));
</script>
```

- [ ] **Step 3: Verify it loads (browser)**

Copy to the preview dir, start/reload the preview, then check the console and the bridge object:
- `preview_console_logs` level `error` → expect **none** (with a real config). NOTE: until Task 1's real config is pasted, the SDK may log an `auth/invalid-api-key` — that's expected pre-config; the import + `window.EchelonSync` shape is what we verify here.
- `preview_eval`: `(function(){return {ready: !!window.EchelonSync, hasSignIn: typeof window.EchelonSync?.signIn, domain: window.EchelonSync?.domain};})()` → expect `ready:true, hasSignIn:"function", domain:"COMPANYDOMAIN"` (or the real domain once pasted).

- [ ] **Step 4: Commit**

```bash
cd /tmp/echelon-timer-fresh && git add index.html && git commit -m "feat(sync): add Firebase bridge module (auth + firestore init)"
```
(Commit happens in the deploy clone during the deploy task; for local iteration just save the file. Keep edits in `tempo-studio/index.html` as the source of truth.)

---

### Task 3: Sign-in screen markup + CSS gate

**Files:**
- Modify: `tempo-studio/index.html` — add a `#signin-screen` element (top of `<body>`); add gate CSS in the `<style>` block.

**Interfaces:**
- Consumes: nothing yet (wired in Task 4).
- Produces: DOM `#signin-screen` with a `#btn-signin` button; a `body.signed-out` state that hides the app + topbar and shows the sign-in screen. Consumed by Task 4.

- [ ] **Step 1: Add the sign-in screen markup** (immediately after `<body>`)

```html
<div id="signin-screen">
  <div class="signin-card">
    <img id="signin-logo" alt="Echelon" />
    <h1>Echelon</h1>
    <p>Sign in with your company Google account to continue.</p>
    <button id="btn-signin" class="gold big">Sign in with Google</button>
    <p id="signin-error" class="signin-error hidden"></p>
  </div>
</div>
```

- [ ] **Step 2: Add gate CSS** (in `<style>`, near the top-level layout rules)

```css
  /* Auth gate: app hidden until signed in (or a cached identity exists). */
  #signin-screen { display: none; }
  body.signed-out #signin-screen { display: flex; position: fixed; inset: 0; z-index: 500;
    align-items: center; justify-content: center; background: var(--bg); padding: 24px; }
  body.signed-out header.topbar, body.signed-out .app { display: none !important; }
  .signin-card { text-align: center; max-width: 360px; display: flex; flex-direction: column;
    align-items: center; gap: 14px; }
  .signin-card #signin-logo { height: 40px; }
  .signin-card h1 { font-family: var(--font-display); letter-spacing: -0.5px; }
  .signin-card p { color: var(--muted); }
  .signin-error { color: var(--danger); font-size: 0.9rem; }
```

- [ ] **Step 3: Set the logo + start hidden-until-resolved** — set `#signin-logo` src to `ECHELON_LOGO` at init (Task 4) and add `signed-out` to `<body>` by default in markup so nothing flashes:

Change the opening `<body>` tag to `<body class="signed-out">`.

- [ ] **Step 4: Verify (browser)**

Reload the preview. With `body.signed-out` present, expect the sign-in screen visible and the library hidden:
- `preview_eval`: `(function(){return {signinVisible: getComputedStyle(document.getElementById('signin-screen')).display!=='none', appHidden: getComputedStyle(document.querySelector('.app')).display==='none', btn: !!document.getElementById('btn-signin')};})()` → `signinVisible:true, appHidden:true, btn:true`.
- `preview_screenshot` → confirm the sign-in card renders.

- [ ] **Step 5: Commit** (save source; commit in deploy task)

---

### Task 4: Auth-gate logic (cached identity, domain check, show/hide)

**Files:**
- Modify: `tempo-studio/index.html` — add an auth-gate section inside the IIFE (near Init), and wire `#btn-signin`.

**Interfaces:**
- Consumes: `window.EchelonSync.{onAuthChange, signIn, domain}` (Task 2); `#signin-screen`/`body.signed-out` (Task 3).
- Produces: `signIn UI state machine`; localStorage cached identity under `tempo.auth.v1` = `{uid,email,displayName}`; functions `showApp()`, `showSignIn(msg)`. Consumed by Task 5 (sign-out clears the cached identity).

- [ ] **Step 1: Add the auth-gate code** (inside the IIFE, e.g. just above the Init section)

```javascript
  // ===========================================================================
  // Auth gate — require company Google sign-in; offline-launch from cached identity.
  //   The app shell starts hidden (body.signed-out). On load we show the app
  //   immediately if a cached identity exists (offline-first), then let
  //   onAuthChange confirm/deny in the background. Domain is enforced server-side
  //   by Firestore rules too; this client check is UX.
  // ===========================================================================
  var AUTH_KEY = "tempo.auth.v1";
  function cachedIdentity() { try { return JSON.parse(localStorage.getItem(AUTH_KEY)); } catch (e) { return null; } }
  function setCachedIdentity(id) {
    try { id ? localStorage.setItem(AUTH_KEY, JSON.stringify(id)) : localStorage.removeItem(AUTH_KEY); } catch (e) {}
  }
  function emailDomain(email) { return (email || "").toLowerCase().split("@")[1] || ""; }
  function showApp() { document.body.classList.remove("signed-out"); }
  function showSignIn(msg) {
    document.body.classList.add("signed-out");
    var e = $("signin-error");
    if (e) { e.textContent = msg || ""; e.classList.toggle("hidden", !msg); }
  }

  function startAuthGate() {
    var sync = window.EchelonSync;
    var logo = $("signin-logo"); if (logo) logo.src = ECHELON_LOGO;
    $("btn-signin").onclick = function () {
      showSignIn("");
      sync.signIn().catch(function (err) {
        showSignIn(err && /popup|cancel/i.test(err.code || "") ? "" : "Sign-in failed — please try again.");
      });
    };
    // Offline-first: a previously-signed-in device shows the app right away.
    if (cachedIdentity()) showApp(); else showSignIn("");

    sync.onAuthChange(function (user) {
      if (user && user.emailVerified && emailDomain(user.email) === sync.domain) {
        setCachedIdentity({ uid: user.uid, email: user.email, displayName: user.displayName || user.email });
        if (typeof updateAccountUI === "function") updateAccountUI(user);
        showApp();
      } else if (user) {                       // signed in but wrong domain / unverified
        sync.signOut();
        setCachedIdentity(null);
        showSignIn("Please sign in with your company (@" + sync.domain + ") email.");
      } else {                                 // signed out (or never signed in)
        setCachedIdentity(null);
        showSignIn("");
      }
    });
  }

  // Run the gate once the bridge is ready (it may load before or after this script).
  if (window.EchelonSync) startAuthGate();
  else window.addEventListener("echelon-sync-ready", startAuthGate);
```

- [ ] **Step 2: Verify gate logic with a STUBBED bridge (browser, no real Firebase needed)**

In the preview, stub `window.EchelonSync` before the gate runs to drive each state. Use `preview_eval`:

```javascript
(function(){
  var calls = {};
  window.EchelonSync = {
    domain: "company.com",
    signIn: function(){ calls.signIn = true; return Promise.resolve(); },
    signOut: function(){ calls.signOut = true; },
    _cb: null,
    onAuthChange: function(cb){ this._cb = cb; }
  };
  localStorage.removeItem("tempo.auth.v1");
  window.dispatchEvent(new Event("echelon-sync-ready"));
  var out = {};
  out.noIdentity_showsSignIn = document.body.classList.contains("signed-out");
  // company user → app shows + identity cached
  window.EchelonSync._cb({ uid:"u1", email:"a@company.com", displayName:"A", emailVerified:true });
  out.companyUser_showsApp = !document.body.classList.contains("signed-out");
  out.identityCached = !!localStorage.getItem("tempo.auth.v1");
  // wrong domain → signed out + error + signOut called
  window.EchelonSync._cb({ uid:"u2", email:"x@gmail.com", displayName:"X", emailVerified:true });
  out.wrongDomain_signedOut = document.body.classList.contains("signed-out");
  out.wrongDomain_calledSignOut = !!calls.signOut;
  out.wrongDomain_errorShown = !$("signin-error").classList.contains("hidden");
  return out;
})()
```
Expected: `noIdentity_showsSignIn:true, companyUser_showsApp:true, identityCached:true, wrongDomain_signedOut:true, wrongDomain_calledSignOut:true, wrongDomain_errorShown:true`.

- [ ] **Step 3: Verify offline-launch (browser, stub)**

```javascript
(function(){
  localStorage.setItem("tempo.auth.v1", JSON.stringify({uid:"u1",email:"a@company.com",displayName:"A"}));
  document.body.classList.add("signed-out");
  // simulate: cached identity present, bridge not yet resolved
  if (window.EchelonSync && window.EchelonSync.onAuthChange) { /* gate already wired */ }
  // re-run the offline-first branch:
  document.body.classList.toggle("signed-out", !JSON.parse(localStorage.getItem("tempo.auth.v1")));
  return { offlineLaunch_showsApp: !document.body.classList.contains("signed-out") };
})()
```
Expected: `offlineLaunch_showsApp:true`. (Confirms a cached identity opens the app before auth resolves.)

- [ ] **Step 4: Console check** — `preview_console_logs` level `error` → no gate-related errors.

- [ ] **Step 5: Commit** (save source; commit in deploy task)

---

### Task 5: Sign-out + account display in Settings

**Files:**
- Modify: `tempo-studio/index.html` — add an account row to the Settings modal markup + an `updateAccountUI()` + wire sign-out.

**Interfaces:**
- Consumes: `window.EchelonSync.signOut` (Task 2); `cachedIdentity`/`setCachedIdentity`/`showSignIn` (Task 4).
- Produces: `updateAccountUI(user)` (referenced optimistically in Task 4); a `#btn-signout` handler. No later task depends on these.

- [ ] **Step 1: Add the account row to the Settings modal** (near the top of the settings modal body, before the tips list)

```html
    <div class="account-row">
      <span class="muted">Signed in as <b id="account-email">—</b></span>
      <button id="btn-signout" class="ghost small" data-ic="arrowLeft">Sign out</button>
    </div>
```

- [ ] **Step 2: Add CSS** (in `<style>`)

```css
  .account-row { display: flex; align-items: center; justify-content: space-between; gap: 10px;
    padding-bottom: 12px; margin-bottom: 12px; border-bottom: 1px solid var(--line); flex-wrap: wrap; }
```

- [ ] **Step 3: Add `updateAccountUI` + wire sign-out** (inside the IIFE, near the auth gate)

```javascript
  function updateAccountUI(user) {
    var el = $("account-email");
    var id = user || cachedIdentity();
    if (el) el.textContent = (id && (id.email)) || "—";
  }
  $("btn-signout").onclick = function () {
    if (!confirm("Sign out of Echelon on this device? Your routines stay on this device.")) return;
    setCachedIdentity(null);
    if (window.EchelonSync) window.EchelonSync.signOut();
    closeSettings();
    showSignIn("");
  };
```

Also call `updateAccountUI()` once at the end of the Init sequence so the Settings row is populated from the cached identity on load.

- [ ] **Step 4: Verify (browser, stub)**

```javascript
(function(){
  localStorage.setItem("tempo.auth.v1", JSON.stringify({uid:"u1",email:"coach@company.com",displayName:"Coach"}));
  updateAccountUI();
  var shown = document.getElementById("account-email").textContent;
  // sign-out clears identity + returns to sign-in
  window.confirm = function(){ return true; };
  document.getElementById("btn-signout").click();
  return { accountEmailShown: shown, afterSignout_signedOut: document.body.classList.contains("signed-out"),
           afterSignout_identityCleared: !localStorage.getItem("tempo.auth.v1") };
})()
```
Expected: `accountEmailShown:"coach@company.com", afterSignout_signedOut:true, afterSignout_identityCleared:true`.

- [ ] **Step 5: Commit** (save source; commit in deploy task)

---

### Task 6: Deploy Phase 1 + real sign-in verification

**Files:**
- Deploy: `tempo-studio/index.html` → `echelon-timer` repo `main`.

**Interfaces:**
- Consumes: a real `firebaseConfig` + `COMPANY_DOMAIN` pasted into Task 2's CONFIG block; deployed rules + authorized domain (Task 1).

- [ ] **Step 1: Confirm the real config is in place** — Task 2's `<script type="module">` CONFIG block has the real `firebaseConfig` and `COMPANY_DOMAIN` (not placeholders), and the same domain is in the deployed rules.

- [ ] **Step 2: Deploy** (same flow as prior deploys)

```bash
gh auth switch --user JustinTechGuyAtWork
git -C /tmp/echelon-timer-fresh fetch origin && git -C /tmp/echelon-timer-fresh reset --hard origin/main
cp /Users/eleonard/Documents/Cluade/tempo-studio/index.html /tmp/echelon-timer-fresh/index.html
cd /tmp/echelon-timer-fresh && git add index.html && git commit -m "feat(sync): Phase 1 — require company Google sign-in (offline-first gate)"
git push origin HEAD
gh auth switch --user justingonz96-creator
```
(Run network git/gh with the sandbox disabled, as in prior deploys.)

- [ ] **Step 3: Wait for the Pages build** — poll the live URL until it serves the new code (grep for `btn-signin`), as in prior deploys.

- [ ] **Step 4: USER real sign-in verification** (Claude cannot sign into the user's Google account)

Ask the user to, on the live URL:
1. Confirm the **Sign in with Google** screen appears.
2. Sign in with their **company** account → the app should appear and Settings should show "Signed in as <their email>".
3. (If they have one) try a **personal Gmail** → expect to be bounced with the "use your company email" message.
4. Reload while signed in → app opens directly (no sign-in screen).
5. Sign out (Settings) → back to the sign-in screen.

- [ ] **Step 5: Confirm + record** — on success, Phase 1 is live. Note the Firebase project id + domain in project memory for Phase 2.

---

## Self-review notes
- **No-test-framework adaptation:** "tests" are browser-eval verifications; gate logic is verified by stubbing `window.EchelonSync` so it needs no live Firebase, while real OAuth is user-verified (Task 6 Step 4). This is intentional and called out in Global Constraints.
- **Phase boundary:** Phase 1 deliberately initializes Firestore but does NOT read/write it — no sync, no data model changes, no library changes. Those are Phase 2+. This keeps Phase 1 small and shippable.
- **Domain single-source:** `COMPANY_DOMAIN` in the bridge is the one client copy; it must match the rules' `COMPANYDOMAIN` (Global Constraints + Task 1/Task 2).
