# Echelon — Routine Builder & Timer

A web version of **TempoFlow** (a class routine builder + live timer), styled for **Echelon**. It opens already branded — Echelon logo, deep teal-navy theme, teal accent. Each **block** of a class picks a **class format** (Lagree/Pilates, Cycle/Spin, Treadmill/Run, or Strength), and that format decides the fields each exercise shows — e.g. Cadence & Resistance for cycle, Speed & Incline for treadmill. Because format is per block, one routine can **mix formats** — perfect for crossover classes (start on the spin bike, finish on the floor with weights).

It's a **single file** (`index.html`). No install, no internet needed. Routines are saved in the browser on the computer you use.

> Built on the TempoFlow clone base. Your earlier app, **ClassFlow** (in `../class-flow`), is untouched. The Echelon logo is embedded in the file, so it stays self-contained.

---

## How to open it
Double-click **`index.html`** — it opens in any browser (Chrome, Safari, Edge). Bookmark it for one-click access.

For the best building experience, use a laptop or a tablet in landscape. The live timer looks great on any screen.

---

## The structure
A **Routine** is one class. Each routine is made of **Blocks** (sections, e.g. "Lower Body" or "On the bike"), and each block holds **Exercises**.

Each **block** picks its own **Class format** (in its header), which sets the columns its exercises show:

| Format | Exercise fields |
| --- | --- |
| **Lagree / Pilates** | Spring · Side · Reps |
| **Cycle / Spin** | Cadence · Resistance |
| **Treadmill / Run** | Speed · Incline |
| **Strength** | Weight · Reps |

Because format lives on the block, a single routine can **mix formats** — e.g. a Cycle block followed by a Strength block for a crossover class. A new block copies the previous block's format, so a normal single-format class is still just one pick.

Every exercise also always has a **Name**, a **Time** (work duration, minutes : seconds), and an optional **Rest** *after* it (seconds) — rest shows as its own step when you run. All the metric fields are free text, so you can write "2 reds", "level 4", "6.5", "15 lb", etc.

---

## What you can do

### Build a routine
- Click **+ New Routine**, name it, and (optionally) add a description and **tags**.
- Each block has a **Class format** picker in its header — set it and the block's exercise columns follow. Use **+ Add exercise** and **+ Add block** (new blocks copy the previous block's format).
- Add an optional **coaching note** to any exercise — click the **✎** button on its row and type a cue (e.g. "drive through the heel"). It shows on the run screen.
- Reorder with **↑ ↓**, duplicate with **⧉**, delete with **✕** (blocks and exercises both).
- Per-block and total times update live. Everything **autosaves** as you type.

### Run it live (tap-to-run)
- **Tap a routine card** (or its **▶ Start**) to launch the immersive timer.
- It shows the current exercise, its metrics for the format (e.g. cadence & resistance, highlighted so you can call them out) as a depleting **ring timer**, its coaching note, the block name, what's next, and overall elapsed/remaining.
- **On a big screen (laptop/TV in landscape)** it becomes a **class dashboard**: the timer on the left and the full **running order** on the right — every block and exercise (and, in a plan, every routine), with the current step highlighted so the room can see what's now and what's coming. Phones keep the full-screen timer.
- Rest intervals run as their own (amber) steps.
- Controls: **Start/Pause**, **Back**, **Skip**, **Restart**, **Exit**. Keys: **Space**, **←**, **→**, **Esc**.
- Turn on **Auto-start next exercise** in Settings for a continuous, hands-free run.

### Exercise catalog (reusable moves)
Click **☰ Moves** in the top bar to keep a library of moves you use often. Moves are kept **per class format** — pick a format at the top of the screen to see and add moves for it.
- Add moves with their usual metrics, time, and rest — or click **Import from routines** to pull in every exercise already used across your routines.
- Then, while building a routine, start typing an exercise name: saved moves appear as suggestions, and picking one **auto-fills** its metrics/time/rest *using the block's format* (so a "Climb" in a Cycle block fills cycle metrics). It only fills a fresh exercise, so it never overwrites something you already set.

### Class plans (run routines back-to-back)
Click **◫ Plans** in the top bar to chain several routines into one continuous class.
- **New Plan**, name it, then **+ Add routine** and pick routines from the dropdowns (reorder with ↑ ↓, remove with ✕).
- The total time adds up all the routines. **Start** runs them end-to-end — the timer shows which routine and block you're in as it flows through.

### Light / dark theme
Tap the **◐** button (top bar) to switch between the dark studio theme and a light theme that matches echelonfit.com. Your choice is remembered. (Tip: the **logo** is also a Home button — click it to return to your routine library from anywhere.)

### Class vs Personal mode
Toggle **Class / Personal** in the top bar (matches TempoFlow). It relabels the library ("Class Library" vs "My Workouts") and the run screen. We can make the two modes behave differently as we iterate.

### Export
**Export** (on a card or in the builder) gives you:
- **PDF run sheet** — print → "Save as PDF"; one table per block, each with that block's format columns, start times, work/rest.
- **Spreadsheet (.csv)** — for Excel / Google Sheets. A mixed-format routine gets one table covering every column, plus a "Format" column so each row is clear.
- **Copy as text** / **Download .txt**.

### Settings (⚙)
- **Quick tips** — a short how-to for building, running, plans, moves, export, and backing up.
- **Beeps** on/off · **Auto-start** on/off
- **Backup / Restore** all routines (with **Undo last restore** if you pick the wrong file)

(Branding is fixed to Echelon, so the app looks the same for everyone it's shared with.)

---

## What's faithful vs. not
- **Faithful:** Routines → Blocks → Exercises, per-exercise load/metrics, rest intervals, Class/Personal modes, tap-to-run immersive timer, tags, the dark + gold look, PDF export — plus multiple class formats beyond Lagree.
- **Can't replicate in a web app:** Apple Watch control and cloud sync between devices. Use **Backup/Restore** to move routines between computers.

## Good to know
- Routines are stored per-browser on each computer (not synced). Back up before switching machines.
- To put it on another computer, copy `index.html` there and open it.

---

## Ideas to build from here
More **class formats** (Rowing, Barre, …) — they're a one-line config addition, dropdown presets for common metric values instead of free text, per-side auto-repeat, and a shared/synced library (needs a backend).
