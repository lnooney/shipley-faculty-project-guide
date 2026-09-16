# CLAUDE.md

Project memory for the **Shipley Academic Innovation Project Guide**. Read this at the
start of every session, before touching `index.html`. It exists because each Claude Code
session starts in a fresh container with no memory of prior sessions — this file is the
one thing that reliably survives, since it's committed straight to GitHub (see workflow
note below). Keep it updated as work continues; don't let it go stale.

## Who's building this, and how

- Owner: Laura (lnooney@bu.edu). Institute for Excellence in Teaching & Learning at BU.
  **Never** call it "IETL" — always "the Institute" or "the Institute team."
- **Process rule, strictly enforced:** log every requested change precisely, but take
  **no action** on `index.html` until Laura gives an explicit, unambiguous signal like
  "execute" or "please make those changes." Investigating a request, answering a
  clarifying question, or discussing options is never itself permission to implement.
  When several requests are pending, batch them and execute together only on her
  explicit go-ahead.
- **File workflow is split in two:**
  - `index.html` (the actual application): Claude commits locally only, **never pushes**.
    After executing a batch, deliver the updated file to Laura via `SendUserFile`; she
    uploads it to GitHub herself through the web UI. This is why the branch's git log is
    full of generic "Add files via upload" commits with no descriptive messages — that's
    expected, not a problem to fix.
  - `CLAUDE.md` (this file): Claude **pushes directly** to GitHub — Laura's explicit
    exception, since it's documentation/rationale, not application code. Keep it current
    and push it whenever it changes; no need to route it through her for upload.
- Verification workflow before delivering any change: implement → `node --check` on the
  extracted `<script>` content → Playwright test against a local `python3 -m http.server`
  instance (mock `window.fetch` for AI calls) → screenshot if visually relevant → commit
  locally → `SendUserFile`.

## What this tool is

A single-file HTML/CSS/JS web app (`index.html`, no build step, GitHub Pages–deployable)
for Institute staff who manage Shipley Academic Innovation grant projects on behalf of
faculty leads. The staff member (PM) is the primary user throughout — faculty lead the
actual project work, but the tool's voice, feedback, and action items are addressed to
the PM, never ambiguously to faculty. This distinction has been corrected many times
across nearly every feature; treat it as a hard constraint on all new copy and AI prompts.

Six tabs: **Project Setup → Impact Framework → Milestones → Budget → Ask Guide re:
Project → Project Report.**

## Architecture

- **Storage:** `localStorage`, multi-project support (save/switch/delete named projects,
  keyed separately from the single "live" working document). No backend, no accounts —
  see "Future direction" below for the discussed-but-not-yet-detailed multi-user version.
- **AI calls:** `callGuide(userMsg, opts)` posts to a Val.town relay wrapping Claude
  (`claude-sonnet-4-5`). `opts.skipCtx` skips automatic project-context injection;
  `opts.maxTokens` overrides the 1000-token default. Every call's system prompt gets
  today's real date appended — this exists because the AI cannot infer "today" on its
  own and previously produced wrong "N months into the project" claims.
- **Structured AI output pattern:** any AI call whose result needs to drive app logic
  (not just be displayed) asks for a literal JSON shape and parses it with
  `raw.match(/\{[\s\S]*\}/)` + `JSON.parse()`. Free-text/descriptive instructions to the
  AI are unreliable for anything structured; exact templates with hard caps are reliable.
  This is the single most important lesson from building this tool — apply it to any new
  feature that needs the AI to return something the app will act on.
- **"Never silently overwrite a manual correction" principle:** applies everywhere the AI
  fills in a value a PM might also enter by hand (Budget tailoring, Milestone date
  estimation). A nonzero/non-empty existing value is treated as a PM correction and is
  never overwritten by a later AI run.
- **Task ID scheme:** `MILESTONE_TASKS[milestone]` array-index-based IDs (`m1_0`, `m4_2`,
  etc.) — inserting/removing tasks mid-array shifts every ID after it. Accepted as a
  known fragility given the tool's stage; display order is separately controlled by
  `group`/`sortLast` fields, independent of raw array position, so reordering within a
  milestone is usually safe without touching IDs.
- **`task_meta[id].start`/`.due`** is the single source of truth for all task dates —
  used by Milestones rendering, due/overdue flags, Excel export, and the Gantt chart.
- **AI feedback rendering:** every feedback surface (Proposal Review, Impact Framework
  Review, Budget Feedback, Timeline Feedback, Report feedback, per-field guidance, Ask
  Guide chat) shares one render path, `renderAIResponse()` → `formatAIResponse()`. That
  function HTML-escapes the AI's raw text, then converts it to real markup: markdown
  `#`/`##`/`**bold**`, ALL-CAPS lines (tolerating a trailing parenthetical), or short
  lines ending in `:` all become `<h4>` headers; `-`/`•` lines become `<ul><li>`; numbered
  lines become `<ol><li>`. Fixing formatting for the whole tool only ever requires
  touching this one function — no need to rewrite every prompt's phrasing to get
  consistent headers/bullets everywhere.

## Design decisions and why (by area)

**Proposal Import (Setup tab)**
- Proposals always arrive as PDFs, so the Import card accepts a PDF upload alongside the
  paste box — PDF.js (CDN, same pattern as ExcelJS) extracts embedded text client-side
  straight into the existing `#import-text` textarea, so nothing downstream needed to
  change. The paste box stays as a fallback/manual-edit option, not a replacement — some
  PDFs are scanned images with no text layer, and extraction that comes back too short
  tells the PM to paste manually instead of quietly feeding near-empty text into the AI
  extraction call. Extraction fills the box but does not auto-run "Import from
  proposal" — the PM still reviews the extracted text and clicks it themselves.

**Proposal review rubric (Setup tab)**
- Backend AI-scored, not a manual UI — 7 weighted criteria plus an eligibility screen
  with auto-fail conditions (12-month completion, valid letter of support, BU students
  must be among the learning-impact beneficiaries, **student-submitted proposals are
  never fundable** — only BU faculty/program directors can lead a Shipley project;
  suggested action is to point the student toward a faculty sponsor).
- Letter of support must be from within BU and from someone senior to the applicant
  *within their own department/school* — it exists to confirm backing from someone with
  real financial decision-making power over sustainability/tech costs, so a missing or
  non-qualifying letter downgrades the Sustainability criterion, not just an eligibility
  note.
- Criterion 6 (Implementation Feasibility): the Guide must independently judge
  feasibility using its own PM experience — a proposal having detailed steps does not
  by itself mean it's feasible. Significant risk caps the score at Developing (2) or
  lower, specifically so an unworkable-timeline project can't score well just because
  it's well-written.
- Criterion 7, renamed **"Award Level Alignment"** (was "Grant-Type Alignment"): checks
  three things now — dollar range fits the tier, the *nature* of the work matches the
  tier's intent (Seed = early pilot, Innovation = program-level), and the budget
  realistically covers the described costs (mirrors the Budget tab's own realism check).
- New tech with no IS&T leadership / Wendy Colby sign-off is a named red flag — that
  approval is required before piloting.
- ELIGIBILITY SCREEN is a dedicated top section of the reply, added because auto-fail
  eligibility issues were being generated as AI context but had no guaranteed spot in the
  actual reply — they could get buried in prose and never reach the PM.

**Budget tab**
- Awards are paid as a single lump sum, not by term — the whole tab was flattened from
  per-term columns to single amounts for this reason.
- Fringe rates are **fixed institutional rates hardcoded in `BUDGET_DEFS`**, corrected to
  BU's actual current rates (Laura read these off BU's own budget spreadsheets): 31.20%
  (faculty/postdoc/temp-exempt/course-release/stipend/overbase), 11.60% (grad student),
  0% (hourly), 33.20% (temp non-exempt). If these ever need correcting again, treat it
  with the same care as last time — confirm exact values and exactly which rows each
  value applies to, never assume a rate carries over to a row that wasn't named
  explicitly.
- **Budget spreadsheet upload**: the Setup tab's Import card also accepts the actual
  budget Excel file (faculty proposals' budgets always arrive as spreadsheets). Layouts
  aren't standardized across departments, so rather than a fixed-schema parse, every
  non-empty cell across every sheet is read via ExcelJS (the same library already used
  for the Excel *export*) and flattened into text, fed to "Tailor Budget to This
  Proposal" as a second source alongside the proposal text. When the spreadsheet and the
  proposal narrative disagree on an amount for the same row, the spreadsheet wins — it's
  the authoritative working budget document, prose isn't.
- Tailor Budget only ever writes a dollar amount into the budget when it's copying a
  figure actually stated somewhere (confirmed proposal line item or proposal text
  verbatim) — never an AI estimate. Anything inferred becomes a "Suggested amount" flag
  for the PM to confirm with the faculty lead instead. This exists because the AI was
  silently "correcting" a PM-confirmed $11,000 line item to $12,276 by re-estimating
  instead of copying.
- Tailor Budget must map each distinct described payment to exactly one official row —
  overlapping categories (Course Release / Faculty Stipend / Overbase) were getting the
  same payment double-counted into two rows, inflating totals.
- The tool's official budget category names are BU's own standardized central-finance
  categories. A proposal using different wording for the same cost and getting mapped
  into the matching standard category is **correct, expected behavior** — several AI
  feedback surfaces (Tailor Budget, the Budget tab chat, Award Level Alignment) now
  explicitly know not to flag that mapping as a discrepancy.
- "Graduate Student (Hourly)" was split out then merged back into "Graduate Student" —
  the fringe toggle already handles salaried-vs-hourly distinction for every other row,
  so a separate hourly row for grad students specifically was redundant.
- The amber "Does this amount include fringe benefits?" reminder used to duplicate the
  toggle buttons directly above it (two ways to answer the same question); it's now
  plain reminder text pointing at those buttons.

**Milestones tab**
- Tailor Milestones hides irrelevant default tasks (`task_relevance`), asks the PM
  directly for genuinely ambiguous ones, and estimates start/due dates within the award
  window — grounded in effort required (not just sequence position), a predecessor
  dependency work-back rule, semester-availability constraints for student-facing tasks,
  and academic term reference dates (Fall/Spring/Summer). The funding-confirmed date
  (not the nominal project start date) is the true floor for any task's start.
- **`getAllProjectTasks()` (feeds both the Gantt chart and Excel export) must mirror
  `renderMilestones()`'s hiding logic** (`task_relevance` not_relevant/pending_faculty,
  unresolved investigate questions) — it didn't for a while, so tailored-out tasks kept
  bleeding through into the Gantt chart and export as dateless rows. If a similar bug
  ever resurfaces, check this function first.
- Gantt chart markers: a full bar needs both start and due; a start-only task gets a
  right-pointing arrow, a due-only (or invalid-range) task gets a diamond — added because
  single-date tasks were being silently dropped from the chart entirely.
- The Impact Evaluation Framework kickoff ("Meet with the project team to review and
  complete the Impact Evaluation Framework") is a distinct early General-group task,
  constrained to the funding window's first two weeks — deliberately separate from the
  existing Learning-Analytics-group "Complete Impact Evaluation Framework" task, which
  stays where it is.

**Impact Framework tab**
- Each section (Problems, Objectives, Impacts, Measures & Targets, Data Collection, Data
  Analysis) uses backward design — each is drafted with awareness of what came before.
- Problems/Objectives/Impacts can be **inferred from the proposal** via an "Infer from
  proposal ↗" button, which fills the *guided sub-question answers* (not the final
  section text) with a note to confirm with the faculty team — deliberately not
  auto-writing the final section, so the human-review checkpoint stays intact.
- **"Autofill entire framework from proposal ↗"** drafts all six sections in one AI call
  (answers + final section content together, in one JSON response) instead of the
  per-section 3-click flow above — but only ever fills sections that are still empty;
  a section with existing content (manual or otherwise) is left untouched, same
  never-overwrite principle as Budget tailoring. Intro copy was reframed around
  reviewing/refining a first draft, now that the PM can see how sections interact,
  rather than building each one from a blank page.
- Each section's guiding sub-questions (previously hidden behind the "Answer questions
  to draft this section" toggle) now render as always-visible subheaders under the
  section header, generated from the same `FW_SUBQUESTIONS` data — the toggle/textarea
  flow still exists underneath for manually redrafting a single section.
- `reviewFramework()`'s feedback prompt was narrowed to three named things — cross-section
  misalignment, measurement realism (e.g. too many objectives/measures for the likely
  scope), and measurement relevance (measuring only what matters) — each capped at 0–3
  bullets, after feedback was reported as too long and running past the token limit
  mid-response.
- A standalone "Download Framework (.xlsx)" export exists on this tab (via the same
  ExcelJS already used for the project-plan export) — a self-contained artifact for the
  PM to hand off, separate from the full project-plan download.
- Data Collection frequency options include "Automatic collection," for trace data or
  AI chat logs that don't fit the other fixed cadences.

**Ask Guide re: Project tab** (renamed from "Proposal and Project Queries")
- Seeded question chips are reworded to be unambiguously the PM's own voice, and no
  longer include a journal/venue-suggestion chip — venue selection is the faculty team's
  own expertise to retain, not something to outsource to the Guide. Same principle
  removed venue-naming language from the Milestone 3 dissemination tip, the Report tab's
  static hint, and the SYSTEM prompt's own knowledge list.
- Clicking a chip now visibly marks it selected and scrolls the question box into view —
  previously there was no feedback that a click had registered.

**Reports (Progress Snapshot & Final Report)**
- Task-derived fields (Overview, Assessment, Dissemination, Timeline, Milestones,
  Impact) show a short completion-percentage line plus bulleted Complete / In Progress /
  Due-within-30-days groups — **not-started tasks with no near-term due date are omitted
  entirely**, not just left uncounted. The earlier version dumped every task's full
  status regardless of relevance and was "overwhelming and illegible."
- All prefilled fields use a `fill()` helper that only writes to an empty field — a PM's
  manual edit is never overwritten by re-running a prefill, and it's still tested for
  every time this area changes.

**Follow-ups With Your Faculty Team (Setup tab)**
- Consolidates everything the PM needs to raise with faculty into one place, surfaced on
  the page they land on first: unresolved "ask faculty team" investigate questions
  (Milestones), undismissed budget flags, and now — as of the most recent batch — action
  items automatically pulled from AI feedback across Proposal Review, Timeline Feedback,
  Impact Framework Review, and Budget Feedback (each via `extractBulletSection()` parsing
  a labeled bullet section from that feedback's response). Each source's items are wholly
  replaced on every re-run of that feedback (not accumulated/duplicated), and items from
  other sources are untouched.

**Tab nav / general layout**
- The tab bar is sticky (`position:sticky;top:0`) with a CSS-only scroll-shadow
  affordance (paired `background-attachment: local/scroll` gradients — no JS) so it's
  obvious there's more to scroll to, without needing a wrapper element that would risk
  breaking the sticky positioning.
- `.card-header`'s title/subtitle block needs `flex:1;min-width:0` and the header itself
  `flex-wrap:wrap`, so a header with an inline action button (currently only the "Import
  from Proposal" card's Expand button) wraps the button below the title on narrow
  screens instead of crowding it.

## Open items — not yet done, don't lose these

None currently. See "Design decisions and why," below, for what was most recently
completed (fringe rate correction, PDF/Excel uploads, AI feedback formatting).

## Future direction (discussed before, details not preserved)

Laura recalls a prior discussion about eventually giving this tool a real backend, so
multiple users could access multiple uploaded projects (i.e., moving off the current
single-user, `localStorage`-only, static-file architecture toward shared/hosted
multi-project, multi-user access). **The specifics of that discussion — tech stack,
auth approach, hosting, timeline, how "multiple users" should share or separate
projects — are not present in this file or in current session memory; they were lost
along with the rest of the pre-this-file conversation history.** Before scoping this,
ask Laura what she remembers being decided or leaning toward, rather than assuming.
This would be a major architectural shift (real backend + database + auth), not an
incremental change to `index.html` — treat it as a separate project/plan, not a batch
item.
