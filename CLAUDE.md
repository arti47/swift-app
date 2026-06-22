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
3. **Podium** — teacher validates the top 3 by votes (can swap), points are awarded (5/3/2), names revealed, winner crowned "Model Answer"; students self-compare to the model.
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
| `index.html` | **Student app.** Phase-aware: waiting → answer form → locked → voting feed → podium/self-compare. Auto-handle identity, draft auto-save, vote/critique budgets. |
| `teacher.html` | **Teacher console** (private, on the teacher's laptop). PIN-gated. Push questions, manage classes & lessons, validate podium, star critiques, live feed, leaderboard, CSV export, housekeeping. |
| `projector.html` | **Read-only classroom display** (the big screen). Phase-aware, **anonymity-safe** (never shows names during voting), runs the podium reveal + standings, shows spotlighted answers. No PIN. |
| `review.html` | **Student revision book.** Pick a name/handle → see all past answers, feedback received, and model answers; print/save-as-PDF. |
| `firebase-config.js` | Firebase project config (shared by all pages). Contains the teacher's real keys. |
| `CLAUDE.md` | This file. |
| `.claude/launch.json` | Local preview-server config for testing. |

---

## 4. Firebase data model

All paths are world-readable/writable (no auth). **Rules must allow every
top-level path below — if you add a new top-level path you MUST tell the teacher
to add it to the Realtime Database rules, or reads/writes silently fail.**

```
session/current                     # the one live round, or null when idle
  roundId        "r<timestamp>"
  question       string
  image          data-URL string | null   # compressed JPEG, embedded (no Firebase Storage)
  phase          "answer" | "vote" | "podium"
  critLimit      number                    # critiques allowed per student this round
  endsAt         epoch ms | null           # countdown end, or null = no timer
  startedAt      server timestamp
  redoOf         roundId                   # present only on a redo round
  podium         { first|second|third: {id, name} }   # set when validated
  spotlight      postId | null             # teacher projecting one answer

posts/<roundId>/<pushId>            # one student submission
  name           string (auto-handle, e.g. "Falcon-7")
  s,w,i,f,t      string  (the 5 dimensions)
  upvotes        number
  submittedAt    server timestamp
  selfcheck      { s|w|i|f|t: "ok"|"fix" }  # student's self-comparison vs model
  critiques/<id> { by, affirm, clarify, suggest, starred? }

rounds/<roundId>                    # permanent record (survives session changes)
  question       string             # prefixed "(Redo) " for redo rounds
  hadImage       bool
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
  { name, questions: [ {q, image, mins, crit}, ... up to 10 ] }
```

**Identity model:** students no longer type a name. Each device generates a
persistent fun **auto-handle** (e.g. `Otter-318`) stored in `localStorage`
(`swift-name`). All scoring/podium/review keys off this handle. The class
**roster** feature still exists in the console but, with auto-handles, it won't
match handles — so the "not yet submitted" list and roster name-matching are
effectively dormant unless the teacher's workflow changes. (Kept in place
intentionally; don't remove without asking.)

**Per-device state in `localStorage` (student):** `swift-name` (handle),
`swift-post-<roundId>` (their submission id), `swift-votesleft-<roundId>`,
`swift-vote-<roundId>-<postId>`, `swift-critsleft-<roundId>`,
`swift-crit-<roundId>-<postId>`, `swift-draft-<roundId>` (auto-saved draft).
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

- **Model answer on the projector** — show the crowned answer full-screen during
  podium for whole-class debrief (currently only on student iPads).
- **Mark-scheme overlay for "Tally the Marks"** — optional official mark
  allocation attached to a question, revealed at podium (predict → verify).
- **Spread the critiques** — assign each student a few specific peers to
  critique so feedback covers the whole class, not just the top of the feed.
- **Per-dimension trend over time** — track which dimension the class is weakest
  on across lessons (data already in CSV; needs a teacher-facing view).
- **Question bank polish** — richer library/search beyond the 10 lesson slots.
- **Room codes / multi-class-simultaneous** — the app assumes ONE class at a
  time; concurrent classes (or a colleague sharing the URL) would collide. Real
  work; only if the need arises.
- **Per-student protection on review.html** — currently any student can view any
  handle's history (acceptable since everything is class-visible, but flagged).

### Known limitations (by design, not bugs)
- No server-side enforcement (budgets/PIN are client-side).
- Single class live at a time.
- Auto-handles mean the teacher can't tie an answer to a named student.
- `review.html` only shows model answers for rounds validated after that feature
  shipped (older rounds have no stored podium); the 30-day cleanup erases history
  the revision book relies on — advise cleaning up only after exams.

---

## 8. Setup checklist (for the teacher / a fresh deploy)

1. Firebase project exists (`swift-analysis-81527`) with Realtime Database in
   `asia-southeast1`; `firebase-config.js` filled in (incl. `databaseURL`).
2. **Realtime Database rules** allow all seven paths:
   `session, posts, rounds, scores, settings, classes, lessons` (each
   `{".read": true, ".write": true}`).
3. Deploy: drag the whole folder to **app.netlify.com/drop**.
4. Teacher opens `teacher.html` → sets a PIN on first visit.
5. Projector opens `projector.html`; students scan the QR to land on `index.html`.

---

## 9. Changelog

Keep newest first. One line per meaningful change. Dates in YYYY-MM-DD.

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
