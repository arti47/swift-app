# CLAUDE.md — S.W.I.F.T. Arena

> **⚠️ STANDING INSTRUCTION FOR ANY AI AGENT**
> This file is the single source of truth for the project. **Whenever you add a
> feature, change behaviour, alter the data model, or fix a notable bug, you
> MUST update this file in the same turn** — at minimum the relevant section
> plus the **Changelog** at the bottom. Treat "the code changed but CLAUDE.md
> didn't" as an incomplete task. Keep it accurate over polished.

---

## 1. What this project is

**S.W.I.F.T. Arena** is a lightweight, browser-based classroom app for teaching
exam answer-technique to ~12-year-old boys in Science. The teacher flashes a
question; each student analyses it on their own iPad across five fixed
dimensions; then the class anonymously votes on and critiques each other's
answers; the teacher validates a podium and awards points. It is a live,
real-time, no-login "arena" designed to keep easily-distracted students engaged.

**The five S.W.I.F.T. dimensions (do not rename without the teacher's say-so):**
- **S** — Number of Setups
- **W** — Command Word
- **I** — Idea of Science
- **F** — Facts / Data
- **T** — Tally the Marks

**The lesson loop (per question / "round"):**
1. **Answer** — teacher pushes a question; students fill the 5 fields and "lock in". No one sees peers yet.
2. **Vote** — teacher opens voting; answers appear **anonymously** ("Answer #N"); students upvote (budget) and give Affirm-Clarify-Suggest critiques (budget).
3. **Podium** — teacher validates the top 3 by votes (can swap), points are awarded (5/3/2), names revealed, winner crowned "Class Champion"; students self-compare against the champion and, if the teacher pre-authored one for that question, the **teacher's model answer** shown alongside.
A session = many rounds. The teacher can run questions ad-hoc or from a saved **lesson**, for any of several saved **classes**.

**Audience/author:** a teacher (non-developer). Keep everything no-login,
QR-code-and-go, and resilient to classroom chaos. Prioritise clarity over
cleverness.

---

## 2. Tech stack & how it runs

- **Pure static front-end.** Plain HTML/CSS/vanilla JS. No build step, no
  framework, no bundler. Each page is a single self-contained `.html` file with
  one inline `<script>`.
- **Firebase Realtime Database** (compat SDK, loaded from gstatic CDN) is the
  only backend — it's the real-time message bus between teacher, students, and
  projector. Config lives in `firebase-config.js` (the teacher's real project,
  `swift-analysis-81527`, RTDB region `asia-southeast1`).
- **No server code, no auth.** Access control is a 4-digit PIN on the teacher
  console and "security by obscure URL". This is deliberate (no student logins).
- **Hosting:** the folder is dragged to **Netlify Drop**. After any change the
  teacher must **re-drag the folder to Netlify** to deploy. (Tell them this every
  time you finish a change.)
- **Local testing:** `.claude/launch.json` defines a `python3 -m http.server`
  on port 8470 for the preview tool. Test against the live Firebase DB, then
  **clean up any test data** (delete test rounds/posts/lessons, reset the
  session) so you don't pollute the teacher's real data.

---

## 3. Files

| File | Role |
|---|---|
| `index.html` | **Student app.** Phase-aware: identity gate (pick name + PIN) → waiting → answer form → locked → voting feed → podium/self-compare. Roster+PIN identity, draft auto-save, vote/critique budgets. |
| `teacher.html` | **Teacher console** (private, on the teacher's laptop/iPad). PIN-gated. Split into two tabs: **🎬 Run Lesson** (live: sticky phase-aware control bar with step strip + one highlighted next-step button, counts & timer, push questions ad-hoc or next-in-sequence, validate podium, star critiques, spotlight, live feed, leaderboard) and **🛠 Setup** (classes, lesson builder, QR/join, CSV export, danger zone). Tab choice persists in `localStorage` (`swift-teacher-tab`). |
| `projector.html` | **Read-only classroom display** (the big screen). Phase-aware, **anonymity-safe** (never shows names during voting), runs the podium reveal + standings, shows spotlighted answers, and (teacher-triggered at podium) the full-screen model answer for debrief. No PIN. |
| `review.html` | **Student revision book.** Pick a name → enter that student's PIN (if set) → see all past answers, feedback received, and model answers; print/save-as-PDF. |
| `firebase-config.js` | Firebase project config (shared by all pages). Contains the teacher's real keys. |
| `qrcode.min.js` | Vendored MIT QR generator (davidshimjs/qrcodejs). Used by `teacher.html` + `projector.html` to render the join QR **locally** (same-origin) — no third-party image service. Must be deployed with the folder. |
| `CLAUDE.md` | This file. |
| `.claude/launch.json` | Local preview-server config for testing. |

---

## 4. Firebase data model

All paths are world-readable/writable (no auth). **Rules must allow every
top-level path below — if you add a new top-level path you MUST tell the teacher
to add it to the Realtime Database rules, or reads/writes silently fail.**

**Multi-tenant (rooms):** every path shown below is now namespaced under
`rooms/<roomCode>/…` — each teacher operates their own **room** (their classes,
lessons, scores, identities, and their own live `session`), so multiple teachers
can run live lessons simultaneously without colliding. The room code is chosen by
the teacher at login and carried to students/projector/review via `?room=CODE` in
the URL (encoded in the join QR). All four pages resolve `room` first, then use a
`ref(path)` helper = `db.ref('rooms/'+room+'/'+path)`. Two **global** paths sit
outside rooms: `rooms` (the container) and `migrated` (one-time legacy-migration
flag = the first room that absorbed the old single-tenant data). The special
`.info/connected` presence path is never namespaced.

```
rooms/<roomCode>/                   # one per teacher; all paths below are inside it
session/current                     # the one live round, or null when idle
  roundId        "r<timestamp>"
  question       string
  image          data-URL string | null   # compressed JPEG, embedded (no Firebase Storage)
  phase          "answer" | "vote" | "podium"
  voteLimit      number                    # upvotes allowed per student this round (teacher-set)
  critLimit      number                    # critiques allowed per student this round (teacher-set)
  model          { s,w,i,f,t } | null      # teacher's pre-authored model answer (shown at podium)
  showModel      bool                      # podium debrief: flip the projector to the model answer
  endsAt         epoch ms | null           # countdown end, or null = no timer
  timerMins      number                    # the round's configured minutes (default 3; used by ↺ Reset)
  paused         bool                      # timer paused (teacher live control)
  pauseLeft      ms                        # remaining time captured while paused
  stuck          { nameKey: name }         # students who tapped "I'm stuck" (answer phase; teacher-only)
  assignments    { nameKey: [postId,...] } # directed critiques: which peers each student must critique (set at Open Voting)
  startedAt      server timestamp
  redoOf         roundId                   # present only on a redo round
  podium         { first|second|third: {id, name} }   # set when validated
  spotlight      postId | null             # teacher projecting one answer

posts/<roundId>/<pushId>            # one student submission
  name           string (auto-handle, e.g. "Falcon-7")
  s,w,i,f,t      string  (the 5 dimensions)
  upvotes        number
  voters         { nameKey: true }   # who upvoted (enforces 1 vote/student across devices)
  submittedAt    server timestamp
  selfcheck      { s|w|i|f|t: "ok"|"fix" }  # student's self-comparison vs model
  critiques/<id> { by, affirm, clarify, suggest, starred? }

rounds/<roundId>                    # permanent record (survives session changes)
  question       string             # prefixed "(Redo) " for redo rounds
  hadImage       bool
  image          data-URL | null    # the question image, persisted for review.html / past-questions / model debrief
  model          { s,w,i,f,t } | null  # teacher's pre-authored model answer (for review.html)
  class          { slot, name } | null  # active class when pushed (CSV filter; absent on old rounds)
  lesson         { slot, name }         # lesson it was pushed from (lesson questions only)
  redoOf         roundId            # optional
  awarded        true               # set once podium points are given (guards double-award)
  podium         { first|second|third: {id, name} }   # permanent copy for review.html

scores/c<slot>/<nameKey>            # per-class scoreboard (slot = active class 1..6)
scores/default/<nameKey>            #   used when no active class is selected
  { name, points }

settings/
  pin            "1234"             # teacher console PIN (first visit sets it)
  roster         [names]            # mirror of the ACTIVE class roster (legacy/compat path the student app reads)
  activeClass    { slot, name }     # which class is live

classes/<1..6>                      # up to 6 saved classes
  { name, roster: [names] }

lessons/<1..10>                     # up to 10 saved lessons
  { name, questions: [ {q, image, mins, vote, crit, model}, ... up to 10 ] }
    #   model = { s,w,i,f,t } | null  (teacher's pre-authored model answer)

identities/<nameKey>                # per-student login (NEW TOP-LEVEL PATH — add to rules!)
  { name, pin }                     # nameKey = name lowercased, punctuation stripped
```

**Identity model:** students **pick their name from the active class roster**
(`settings/roster`) and unlock it with a personal **4-digit PIN** stored at
`identities/<nameKey>`. So scoring/podium/review key off the student's real name
and **follow the student across any iPad and any day**, not the device. The
chosen name is remembered in `localStorage` (`swift-name`) for one-tap rejoin on
the same device, but is always re-selectable via the "switch" link. First time a
name is claimed the student creates its PIN; after that the PIN is required
(also to open that name's Revision Book). The teacher can reset a forgotten PIN
from the Classes card. If the roster is empty the student free-types a name; if
the `identities` path is denied by rules, the app degrades to name-only (no PIN).
*(Older builds used a silent per-device auto-handle; a one-time `swift-idv2`
flag clears it so everyone re-picks a real identity on first launch.)*

**Per-device state in `localStorage` (student):** `swift-name` (chosen identity),
`swift-idv2` (migration flag). All **per-round** keys are **suffixed with the
identity's nameKey** so two students sharing one iPad in the same round keep
separate state: `swift-post-<roundId>-<nameKey>` (their submission id),
`swift-votesleft-<roundId>-<nameKey>`, `swift-vote-<roundId>-<nameKey>-<postId>`,
`swift-critsleft-<roundId>-<nameKey>`, `swift-crit-<roundId>-<nameKey>-<postId>`,
`swift-draft-<roundId>-<nameKey>` (auto-saved draft). Switching identity re-inits
the round for the new student (clears the stale `myPostId`). Also `swift-textsize`
(`big`|`normal`, accessibility toggle).
Teacher: `swift-teacher-pin`, `swift-runner-<lessonSlot>` (next-question pointer).

---

## 5. Conventions & gotchas (read before editing)

- **One inline `<script>` per page.** Keep it that way; the syntax-check tooling
  greps `<script>…</script>`.
- **Always escape user text** with the `esc()` helper before putting it in
  `innerHTML`. Students type arbitrary text.
- **Dimensions are data-driven** via the `DIMS` array + `renderDims()` in each
  page. Change labels in ONE place per file.
- **Feeds re-render on every DB event.** When rebuilding a feed via `innerHTML`,
  preserve in-progress `<input>` text and focus (see `renderFeed()` in
  `index.html`). Don't regress this — it matters with 30 kids typing at once.
- **Budgets & PIN are localStorage/client-enforced**, not server-enforced. Good
  enough for classroom mischief, not real security. Don't claim otherwise.
- **Images are embedded as compressed data-URLs** in the DB (Firebase Storage
  needs a paid plan for new projects). Keep the compress-and-cap logic; warn if
  >~900 KB.
- **New top-level DB path ⇒ Firebase rules update required.** Add graceful
  failure (show a warning) when a path is denied, like the classes/lessons
  listeners do.
- **Projector must stay anonymous** during `answer`/`vote` phases and for
  spotlight. Never leak names there before the podium.
- **After ANY change:** syntax-check, ideally test live against Firebase, clean
  up test data, remind the teacher to re-deploy to Netlify, and **update this
  file**.

---

## 6. Current feature set (as built)

- 5-field S.W.I.F.T. answer form with min-length guard and paste-blocking.
- Teacher-gated phases (answer → vote → podium); "Open Voting" / "Validate Top 3".
- **Anonymous voting** ("Answer #N"); names revealed at podium.
- **Vote budget** (3/round, one per post, never own) + **critique budget**
  (teacher-set per round, default 2, one per post).
- Affirm-Clarify-Suggest critiques; teacher **⭐ stars** good ones (+1 point).
- **Podium validation** with full-answer previews; **5/3/2 points**; staged
  reveal + confetti on projector; winner crowned **Model Answer**.
- **Self-comparison**: students rate own answer vs model per dimension (✅/🔧);
  teacher sees a class-wide gap tally.
- **Redo Round**: re-push same question; students see their first attempt +
  feedback and improve it.
- **Spotlight**: teacher projects any single answer anonymously for discussion.
- **Classes** (6 slots, each own roster + own scoreboard) and **Lessons**
  (10 slots × up to 10 questions) with prepare-ahead + run-in-sequence.
- Lesson questions: **edit in place**, **reorder** (↑/↓ and drag-and-drop),
  **select-to-push** + **push-next-in-sequence**.
- **Auto-handle** identity (no name entry); **draft auto-save** (survives
  reload/sleep); **warn-before-push** when students are still answering.
- **CSV export** (all rounds + critiques + leaderboard); **30-day cleanup**;
  per-class score reset; PIN gate; projector view; revision book.

---

## 7. Roadmap / backlog (not yet built)

Ordered roughly by teaching value. Confirm scope with the teacher before building.
The improvement map below was reviewed with the teacher on 2026-06-28; the teacher
prioritised **teaching effectiveness**, and within it the **Question Bank**, whose
full agreed spec is in §7.1. The other items remain open.

**A. Teaching effectiveness**
- **Question bank (reuse past questions)** — NEXT UP; full spec in §7.1 below.
- **Mark-scheme overlay for "Tally the Marks"** — optional official mark
  allocation attached to a question, revealed at podium (predict → verify).
- **Per-dimension trend over time** — track which dimension the class is weakest
  on across lessons (data already in CSV / `selfcheck`; needs a teacher view).
- **Exemplar library** — save any strong student/model answer as a reusable model
  answer for future questions (builds a bank of worked examples).

**B. Reliability & data safety**
- **`review.html` name-match bug** — `doLoadBook()` matches a student by
  `name.toLowerCase()` only, while every other page uses `nameKey()` (which ALSO
  strips `.#$[]/` punctuation). A name like `O'Brien` / `J.Tan` can therefore fail
  to match its own history in the revision book. Fix: use the shared `nameKey()`
  rule when comparing in `review.html`. Low-risk, high-value.
- **Backup before 30-day cleanup** — `deleteOldRounds()` destroys the `rounds`
  history that `review.html` AND the question bank depend on, with no safety net.
  Add an auto-CSV-export (or "are you sure, here's the export first") step.
- **Undo for destructive actions** — podium award, score reset, and End Round are
  guarded only by type-to-confirm; no undo once done.

**C. Security hardening** (all "by design" today, but candidates if it matters)
- DB is world-read/write; security is obscure-URL + client-side PIN only. Anyone
  with a room code can read or wipe everything. **Firebase rules with validation**
  (shape/size caps, lock `migrated` once set, cap image bytes) would harden this
  without adding student logins. NOTE: any such change is a **rules edit** the
  teacher must apply in the Firebase console.
- PINs are stored in **plaintext** at `identities/<nameKey>` and `settings/pin`.

**D. UX polish**
- Student: clearer "disconnected — don't retype" affordances around the existing
  reconnect banner.
- Teacher: keyboard shortcuts for the live phase buttons; the live feed can still
  be a wall of text in a big class.
- Projector: font-size scaling for very large rooms.
- **Per-room teacher PIN reset / room admin** — no way yet to change a room's PIN
  or delete a room from the console (edit the DB directly if needed).

**E. Code maintainability**
- `DIMS`, `esc()`, `renderDims()`, `ref()`, the room-resolution snippet, and the
  countdown timer are **copy-pasted across all 4 HTML files** and drift over time.
  A shared `swift-common.js` would cut the duplication — but it bends the
  "one self-contained file, one inline `<script>`" convention (§2, §5) and the
  syntax-check tooling, so weigh that trade-off with the teacher first.

### Known limitations (by design, not bugs)
- No server-side enforcement (budgets/PIN are client-side).
- Single class live at a time.
- `review.html` only shows model answers for rounds validated after that feature
  shipped (older rounds have no stored podium); the 30-day cleanup erases history
  the revision book relies on — advise cleaning up only after exams.

---

## 7.1 Question Bank — agreed spec (NEXT UP, not yet built)

Agreed with the teacher 2026-06-28. **Goal:** reuse past questions instead of
retyping them. **Key constraint chosen:** build it **from the `rounds` history
that is already stored** — so there is **NO new Firebase path and NO rules
change**; deploy is a plain re-drag to Netlify.

**Where it lives:** UPGRADE the existing **🗂 Past Questions** card in the Setup
tab (don't add a separate card). Keep the current read-only review behaviour
(`showHistoryRound()`); the bank *adds* search + reuse actions, it doesn't replace
the viewer. Existing scaffolding to build on: `loadHistory()` (loads `rounds` +
`posts` into `historyRounds`/`historyPosts`), `populateExportFilters()` (already
builds class/lesson option lists), `pushQuestion()`, `pushRedo()`, `addToLesson()`,
`setModelForm()` / `expandModel()`, `questionImage`, `classTag()`.

**1. Build a deduped list** (new helper, e.g. `buildQuestionBank()`), from
`historyRounds`:
- Normalize a key per round: trim, strip a leading `"(Redo) "`, lowercase — so a
  question and its redos collapse together.
- **Collapse duplicates:** keep the **most recent** round per key (max
  `startedAt`); prefer a version that HAS a `model` so the kept entry carries an
  exemplar where one exists.
- **Include image-only ad-hoc questions** (text === `"Analyze the image using
  S.W.I.F.T."`): keep them, grouped by their `image` string, labelled by date /
  thumbnail since the image *is* the question.
- Each entry retains: `question`, `image`, `model`, `class`, `lesson`,
  `startedAt`, and the source `roundId`.

**2. Search & filter UI** (above the existing dropdown), re-rendering on change:
- Text search box (case-insensitive substring on question text).
- Class `<select>` + Lesson `<select>` (reuse the `populateExportFilters()`
  pattern; tags already live on `rounds/<id>.class` / `.lesson`).
- Checkbox "only questions with a model answer" (filter where
  `model && DIMS.some(([k]) => model[k])`).

**3. Four reuse actions per result row** (all carry the stored image + model):
- **Push live now** — same payload shape as `pushQuestion()` but seeded from the
  entry; write the `rounds/<newId>` record with `classTag()`; respect
  `confirmNoClass()` / `confirmInterrupt()`.
- **Add to a lesson slot** — append `{q, image, mins, vote, crit, model}` to the
  chosen lesson's `questions` (mirror `addToLesson()`).
- **Load into composer** — set the compose textarea, image preview
  (`questionImage`), and model form (`setModelForm()`, then `expandModel()` if a
  model exists), so the teacher can tweak before saving/pushing.
- **Duplicate as redo** — like `pushRedo()` but seeded from the entry; set
  `redoOf` to the entry's source `roundId` and `"(Redo) "` prefix on the `rounds`
  record.

**Docs to update when built** (per the standing instruction): the Past Questions
description in §6, and a Changelog entry. No §4 data-model change (nothing new is
stored). **Caveat:** the bank only reaches as far back as `rounds` is retained —
the 30-day cleanup truncates it, which is why item B "backup before cleanup"
pairs naturally with this.

**Conventions to respect while building:** `esc()` all rendered question text
before `innerHTML`; keep the one-inline-`<script>` rule; remind the teacher to
re-drag to Netlify (no rules change needed for this feature).

---

## 8. Setup checklist (for the teacher / a fresh deploy)

1. Firebase project exists (`swift-analysis-81527`) with Realtime Database in
   `asia-southeast1`; `firebase-config.js` filled in (incl. `databaseURL`).
2. **Realtime Database rules.** With multi-tenant rooms, the live data lives under
   `rooms`, plus a `migrated` flag, plus the legacy top-level paths (kept readable
   so the one-time migration can copy old data in). Allow:
   `rooms, migrated, session, posts, rounds, scores, settings, classes, lessons,
   identities` (each `{".read": true, ".write": true}`). The two that MUST be added
   for rooms are **`rooms`** and **`migrated`** — without `rooms`, nothing loads.
3. Deploy: drag the whole folder to **app.netlify.com/drop**.
4. Teacher opens `teacher.html` → sets a PIN on first visit.
5. Projector opens `projector.html`; students scan the QR to land on `index.html`.

---

## 9. Changelog

Keep newest first. One line per meaningful change. Dates in YYYY-MM-DD.

- 2026-06-28 — **Docs/roadmap only (no code change).** Reorganised §7 into a
  full improvement map (A teaching effectiveness, B reliability & data safety,
  C security hardening, D UX polish, E maintainability) from a planning session
  with the teacher, and added **§7.1 — the agreed Question Bank spec** (reuse past
  questions, built from `rounds` history → no new Firebase path / no rules change;
  upgrades the 🗂 Past Questions card with search + Push-live / Add-to-lesson /
  Load-into-composer / Duplicate-as-redo). Also logged the **`review.html`
  name-match bug** (matches by `name.toLowerCase()` instead of `nameKey()`, so
  punctuated names can miss their own history) as a known fix. Nothing built yet.

- 2026-06-22 — Timer now arrives **ready-but-paused at its full duration**
  (default 3 min) when a question is pushed, instead of auto-running. The Run-bar
  button is a single **▶ Start ⇄ ⏸ Pause** toggle; **↺ Reset** returns to the full
  duration, paused. Push/redo set `paused:true, pauseLeft:timerMins*60000`.

- 2026-06-22 — Teacher console header now shows **Room: <code>** and a **🚪 Log out**
  link (clears the cached room + PIN on this device, returns to the room-login
  screen — for shared laptops / switching rooms).
- 2026-06-22 — **Multi-tenant rooms (multiple teachers, concurrent live lessons).**
  Everything is now namespaced under `rooms/<roomCode>/…`. Each teacher logs in
  with a chosen **room code** (+ per-room PIN); students/projector/review carry the
  code via `?room=CODE` in the URL (encoded in the join QR). Each room has its own
  live `session`, so teachers run lessons simultaneously without colliding. New
  global paths **`rooms`** + **`migrated`** (must be added to RTDB rules). On the
  first room's PIN creation, the old single-tenant classes/lessons/scores/roster
  are **auto-migrated** into it. New **📥 Import from another room** tool copies a
  colleague's class/lesson into your slots (by their room code). All four pages
  resolve `room` first and use a `ref()` helper for every DB path.

- 2026-06-22 — **Directed critiques (assign + restrict).** When the teacher opens
  voting, the app assigns each submitter **N specific peers** to critique (N = the
  critique budget), spread evenly cyclically so every answer gets ~N critiques.
  Students may **only** critique their assigned answers (shown first, highlighted,
  with a banner); voting stays open to all. Stored at `session.assignments`
  (`{nameKey:[postId]}`). Backward-compatible: rounds with no assignments allow
  open critiquing. No new top-level paths.
- 2026-06-22 — Effectiveness/QoL batch: **(5)** students see their **own points &
  rank** on the waiting/locked screens; teacher shows **📈 Most Improved** (biggest
  upvote gain vs first attempt) at a redo podium. **(6)** student **"🙋 I'm stuck"**
  toggle (answer phase) → teacher sees a live count + names in the Run bar
  (`session.stuck`, cleared on lock-in/next round). **(7)** a "locked-in" **chime**
  on the student device (audio unlocked by their own tap). **(8)** CSV export now
  includes the **self-check (✅/🔧)** columns per dimension. **(10)** student
  **🔠 bigger-text** accessibility toggle (`swift-textsize`). No new top-level paths.
- 2026-06-22 — Teacher Setup gains a **🗂 Past Questions** viewer: load past
  rounds, pick any one from a dropdown, and review its answers/votes/critiques/
  podium/model read-only (the live console only ever shows the active round; old
  responses were retained in `rounds`/`posts` but weren't browsable in-app).
- 2026-06-22 — **QR codes now generated locally** (vendored `qrcode.min.js`,
  same-origin) on `teacher.html` + `projector.html`, replacing the external
  `api.qrserver.com` image service that was blocked on the school network (QR
  showed blank). New file must be deployed with the folder. No data-model change.
- 2026-06-22 — Projector now keeps a compact **"scan to join" QR** on the answer
  and voting screens (not just the idle screen), so latecomers can join mid-round.
- 2026-06-22 — **Bugfix: shared-iPad identity switch.** Per-round student state
  (submission pointer, draft, vote/critique budgets) was keyed by round only, so
  logging in as a second student on the same device in the same round showed the
  first student's locked-in answer. All per-round `localStorage` keys are now
  suffixed with the identity's nameKey, and switching identity re-inits the round
  (resets `myPostId`).
- 2026-06-22 — **Filtered CSV download.** Rounds are now stamped with the active
  **class** and (for lesson-pushed questions) the **lesson** they came from; the
  Manage card gained **Class** + **Lesson (paper)** dropdowns to scope the export
  (and it now includes Class/Lesson columns + that class's leaderboard). Old
  untagged rounds appear only under "All". No new top-level paths (tags nested on
  `rounds/<id>`).
- 2026-06-22 — Polish batch: **(a)** timer now **defaults to 3 min** and the Run
  bar gained a **↺ Reset** button (resets to the round's `timerMins`, default 3).
  **(b)** Teacher can **🗑 delete a student's answer** from the live feed.
  **(c)** Student PIN box resized (was oversized). **(d)** During the **vote**
  phase the student feed **hides peers' vote counts and critiques** (revealed at
  podium) so voting/feedback stays independent. **(e)** **Un-vote**: tapping an
  upvoted answer again removes the vote and refunds the budget. **(f)** Podium
  **validation redesigned**: shows the top 5 answers by votes, each with a
  place dropdown (🥇 1st +5 / 🥈 2nd +3 / 🥉 3rd +2; top 3 pre-set), instead of
  one student-picker per slot. **(g)** Projector **model-answer screen now shows
  the question** above the answer. New `session.timerMins`; no new top-level paths.
- 2026-06-22 — User-friendliness batch (student + teacher): **(1)** connection
  banner on `index/teacher/projector` — a "🔌 Reconnecting…" bar driven by Firebase
  `.info/connected` (2s debounce) so flaky-wifi drops are obvious. **(2)** live
  answer validation on the student form — per-field character counters and the
  Lock-In button stays disabled until all five parts meet the min length (no more
  after-the-fact alert). **(3)** teacher live timer controls — **⏸ Pause/Resume**
  and **+1m / +2m** in the Run bar, no re-push needed (new `session.paused` /
  `session.pauseLeft`; student & projector clocks honour the freeze). **(4)**
  active-class clarity — the Run bar always shows the live class + roster size,
  and pushing with **no class selected** now warns (points go to a shared board &
  students would have to free-type names). No new top-level Firebase paths.
- 2026-06-22 — **Student identity overhaul: roster pick + personal PIN.** Replaces
  the per-device auto-handle. Students choose their name from the active class
  roster and unlock it with a 4-digit PIN stored at the new `identities/<nameKey>`
  path, so points/history follow the student across iPads and days. PIN also gates
  that name's Revision Book. Teacher gets a "Manage student PINs" reset tool in the
  Classes card. **NEW Firebase path `identities` — must be added to the RTDB rules**
  (degrades to name-only if absent). One-time `swift-idv2` flag clears old handles.
  Roster-based "not yet submitted" tracking now works again as a bonus. Closes the
  "per-student protection on review.html" roadmap item.
- 2026-06-22 — Teacher live feed reading-load controls: critiques are now
  **collapsed by default** behind a per-answer "💬 N critiques — show" toggle, and
  the feed shows only the **top 8 answers by votes** with a "Show all N" switch.
  Cuts the wall of text during a live lesson; voting order does the triage.
- 2026-06-22 — **Model answer on the projector** for whole-class debrief: at the
  podium the teacher console shows a **📺 Show Model on Screen** toggle
  (`session.showModel`) that flips the projector to a full-screen model answer —
  the teacher's pre-authored one if set, otherwise the crowned Class Champion's
  answer — and back to the podium standings. (Closes the matching roadmap item.)
- 2026-06-22 — **Pre-authored teacher model answer** (optional, per lesson
  question): a collapsible 5-dimension "Model answer" section in the Setup
  composer. Carried on `session.model` and stored on `rounds/<id>.model`. At the
  podium the student compare panel now shows the **🎯 teacher's model** alongside
  the **🏆 top peer** (class champion) and the student's own answer; the Revision
  Book (`review.html`) shows the teacher's model per question too. The crowned
  peer answer is now called the "Class Champion" to distinguish it from the
  teacher's model. Ad-hoc Run-tab questions have no pre-authored model. No new
  Firebase paths (nested under existing session/rounds/lessons).
- 2026-06-22 — Run tab now lists the selected lesson's questions inline:
  tap one to select, then **🚀 Push Selected**, or **▶ Push Next in Sequence**
  (no need to leave for the Setup tab). **Vote budget is now teacher-set per
  round** (`session.voteLimit`, new `vote` field on lesson questions), alongside
  the existing critique budget; the composer defaults to **2 votes / 3 critiques**.
  Student app reads `session.voteLimit` (falls back to 2). No new top-level paths.
- 2026-06-22 — Teacher console UX overhaul: split into **Run Lesson** / **Setup**
  tabs (remembered in `localStorage`). Run tab has a sticky phase-aware control
  bar (Answer→Vote→Podium step strip, live submitted/critique counts + timer, and
  a single highlighted next-step button), a collapsible "Push a Question" panel
  (push next-in-sequence from a lesson, or a one-off ad-hoc question), then the
  live feed/leaderboard. Setup tab holds classes, the lesson builder (compose +
  Add to Lesson, edit/reorder/drag), QR/join info, CSV export, and a red **Danger
  zone** (Reset Scores, Delete Old Rounds) gated by **type-to-confirm** prompts;
  "End Round" (was Clear Board) is type-to-confirm too. No data-model or Firebase
  rule changes. Behaviour note: a Redo Round now starts with no timer (the old
  build reused whatever was typed in the compose box).
- 2026-06-22 — Created CLAUDE.md.
- 2026-06-22 — Added draft auto-save (student typing survives reload/sleep,
  cleared on lock-in) and warn-before-push when students are still in the answer
  phase (count adapts to roster presence).
- 2026-06-22 — Lesson questions: select-to-push with a prominent "Push Selected"
  button (replaced per-row push), kept "Push Next in Sequence".
- 2026-06-22 — Lesson questions: drag-and-drop reordering (with ↑/↓ as touch
  fallback) and in-place editing.
- 2026-06-22 — Removed student name entry; introduced persistent per-device
  auto-handles so podium/scores/review keep working with zero name friction.
- 2026-06-22 — Added Classes (6) with per-class scoreboards and Lessons (10×10
  preloaded questions); new `classes`/`lessons` DB paths.
- 2026-06-22 — Added Redo Round, Spotlight, self-comparison vs model + class
  tally, and `review.html` revision book; podium now stored permanently.
- 2026-06-22 — Per-round critique limit (teacher-set); collapsible feed.
- 2026-06-22 — Multi-question sessions, CSV export, image-as-question support,
  Firebase realtime backend, teacher console + projector view, points/podium
  system, PIN gate.
- (earlier) — Initial single-page localStorage prototype.
```
