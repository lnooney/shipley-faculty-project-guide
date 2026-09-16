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
- **Feedback prompt structure convention:** every "big" feedback prompt (Proposal
  Review, Impact Framework Review, Timeline Feedback, Budget Feedback, Final Report
  Feedback) now asks for the same shape — short named sections with capped bullets
  (RED FLAGS / YELLOW FLAGS / NEXT STEPS, or topic-specific equivalents for Proposal
  Review and Framework Review), ending in "ACTION ITEMS FOR THE PM:" — so a PM can scan
  any of the five the same way. `formatAIResponse()` (above) only renders whatever
  structure the AI's raw text already has; getting that structure to exist in the first
  place is the prompt's job, not the renderer's. If a new feedback surface is ever
  added, give it this same shape rather than inventing a new one.
- **Generated Word documents (.docx):** the `docx` JS library (CDN, same no-build-step
  pattern as ExcelJS/PDF.js) builds real, SharePoint-compatible Word files client-side.
  `BU_LOGO_BASE64` is the official Institute logo (extracted directly from Laura's Intake
  Summary template's page header) embedded as a base64 constant and used via
  `docxLogoParagraph()` at the top of every generated document, for a consistent
  letterhead. Shared builders (`docxHeaderCell`, `docxFieldRow`, `docxFullWidthTable`,
  `textToDocxParagraphs`, etc.) live just above `showGanttModal()` — reuse them for any
  future generated document rather than hand-rolling table/paragraph construction again.

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
- **Instructional design** and **Pedagogical support** are two Project Support Needs
  fields (same yes/maybe/no pattern and proposal-import auto-fill as Media/Data/Workers),
  added specifically to give the Intake Summary's "Support Needs" checkboxes a real,
  non-AI-guessed source — see Intake Summary below.
- Both upload fields are **drag-and-drop zones** (`#import-pdf-dropzone` /
  `#import-budget-dropzone`), not plain `<input type="file">` elements — a plain file
  input just looks like a small "Choose File" button, easy to miss as a drop target.
  Dropping a file populates the same hidden `<input>` via a synthesized `DataTransfer`
  and reuses the exact same `handleProposalPdfUpload()`/`handleBudgetSheetUpload()`
  logic as clicking to browse — only *how* a file arrives changes.
- A Setup field the proposal import successfully fills gets a **green `.auto-filled`
  marker**, distinct from the existing amber `.needs-input` marker for fields that came
  up empty (`markAutoFilled()`, called at every fill site in `applyProposalImport()`) —
  previously an auto-filled field looked identical to one the PM typed by hand. Both
  markers clear the instant the PM edits that field (same delegated `input`/`change`
  listener).
- **"Import from Proposal" now chains the other three tabs' own tailor/autofill actions**
  automatically (`chainTabAutofills()`): Budget → Milestones → Impact Framework, right
  after Setup fields are filled — the PM no longer has to separately visit each tab and
  click its own button. Safe because all three already independently respect the
  never-overwrite-a-manual-correction principle. Shows step-by-step progress text (not
  one spinner) since this turns one click into up to 4 sequential AI calls; if the award
  tier can't be determined from the proposal, budget tailoring is skipped with a plain
  explanation in the final summary rather than silently doing nothing. Calling
  `renderFwAutofillButton()` right before `autofillEntireFramework()` in the chain is
  required — that button is only ever rendered into the DOM when the PM visits the
  Impact Framework tab or this function primes it, so without it the framework autofill
  step would silently no-op on a PM's very first import.

**Intake Summary (Setup tab)**
- Generates a true `.docx` matching the Institute's official Intake Summary template
  exactly (pink `F4CCCC` shaded table headers, same fields, same BU logo) — Laura wanted
  it SharePoint-uploadable and further editable in Word, which ruled out a styled-HTML/
  print approach; only a real generated `.docx` behaves like a native Word file in
  SharePoint (previews, co-authoring, version history).
- Direct fields (title, lead, team, challenge → Teaching & Learning Challenge, vision →
  Proposed Academic Innovation, audience → Target Learners, Impact Framework's Impacts,
  IRB status/notes, start/end dates) come straight from existing tool data — never
  re-derived or AI-guessed when a dedicated field already holds the answer.
- The **Support Needs checkboxes are never AI-inferred** — each one maps to a specific
  existing structured field, confirmed explicitly with Laura rather than assumed:
  Instructional design/Pedagogical support ← their own fields; Technology investigation
  ← the Technology tools list being non-empty; Other technology support ← Data &
  Analytics; Other support ← Media (Laura's explicit call — Media doesn't get its own
  checkbox).
- Only the handful of fields with no dedicated home anywhere in the tool (Budget
  Rationale, Scaling/Dissemination, Anticipated concerns/risks, Timeline flexibility,
  Timeline risk factors, Institute support needs) go through one AI call, grounded in
  the proposal text, Impact Framework, and Budget — same literal-JSON-template pattern
  as everywhere else.
- A **"Notes" section** (not part of the official template — a deliberate addition)
  pulls Timeline Feasibility, Red Flags, Challenges & Suggestions, and Concerns Raised
  by the Proposal straight out of the "Feedback on Proposal and Scope" response, via the
  existing `extractBulletSection()` helper — never re-synthesized by a fresh AI call.
  This required persisting that response's raw text to `data.proposal_review_text` in
  `aiScopeReview()` (it was previously only ever rendered to the DOM, never saved) so
  the Intake Summary generator can read it later, possibly in an entirely different
  session. If no proposal review has been run yet, each subsection shows a placeholder
  telling the PM to run "Feedback on Proposal and Scope" first, rather than a misleading
  "None identified." (which would imply a review ran and found nothing).

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
- When both a proposal and a budget spreadsheet are uploaded, a dollar figure that
  appears **only** in the proposal's own narrative can never become a confirmed
  "mapping" — it's structurally routed into "estimates" (the PM-must-confirm category)
  instead. Only the spreadsheet or staff-confirmed Setup line items can produce a
  mapping in that case. This replaced a softer "trust the spreadsheet over the proposal
  narrative" prose instruction that wasn't reliably followed — a real instance of this
  tool's own core lesson (free-text instructions in a large context block aren't
  reliable for anything that must be followed precisely; structural constraints are).
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
  rather than building each one from a blank page. The button itself sits in a
  prominent highlighted gold callout at the top of the "Your Framework" card, above the
  Problems section — not tucked below the intro paragraph in the card above it, where it
  was easy to miss.
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
- Both **Download as Word (.docx)** buttons (`downloadReportDocx('snapshot'|'final')`)
  replaced a bare print-popup with a real generated Word file — same BU Institute logo
  header as the Intake Summary, for a consistent look across all three generated
  documents. Bulleted field content ("- " lines from task-summary prefills) renders as
  real Word bullets via `textToDocxParagraphs()`, reusing the same bullet-vs-paragraph
  line-detection logic as `formatAIResponse()` (on-page AI feedback) — one detection
  approach, two output targets (HTML vs. docx), kept deliberately consistent.

**Follow-ups With Your Faculty Team (Setup tab)**
- Consolidates everything the PM needs to raise with faculty into one place, surfaced on
  the page they land on first: unresolved "ask faculty team" investigate questions
  (Milestones), undismissed budget flags, and action items automatically pulled from AI
  feedback across all five main feedback surfaces — Proposal Review, Timeline Feedback,
  Impact Framework Review, Budget Feedback, and (as of the most recent batch) Final
  Report Feedback too (each via `extractBulletSection()` parsing a labeled bullet
  section from that feedback's response; `FOLLOWUP_SOURCE_META`'s `report` entry drives
  its card label/tab link). Each source's items are wholly replaced on every re-run of
  that feedback (not accumulated/duplicated), and items from other sources are untouched.
- Every text source that feeds this card — Milestones investigate-questions, Budget
  "missing" flag messages, and the "ACTION ITEMS FOR THE PM" bullets on all five
  feedback surfaces — has an explicit word cap (**under ~15 words**) instead of vague
  "short"/"concise" wording. Reported as hard to scan quickly when the underlying text
  ran long even though each item was technically "one bullet"; matches the same
  exact-caps-over-vague-adjectives principle used throughout this tool.

**Tab nav / general layout**
- The tab bar is sticky (`position:sticky;top:0`) with a CSS-only scroll-shadow
  affordance (paired `background-attachment: local/scroll` gradients — no JS) so it's
  obvious there's more to scroll to, without needing a wrapper element that would risk
  breaking the sticky positioning.
- `.card-header`'s title/subtitle block needs `flex:1;min-width:0` and the header itself
  `flex-wrap:wrap`, so a header with an inline action button (currently only the "Import
  from Proposal" card's Expand button) wraps the button below the title on narrow
  screens instead of crowding it.
- The "Milestones" tab's nav label is now **"Timeline"** — a visible-label swap only
  (`id="tab-btn-milestones"`, `renderMilestones()`, `MILESTONE_TASKS`, and the tab's own
  card titles like "Project Milestones & Timeline" / "Tailor Milestones to This
  Proposal" are all unchanged). Every AI-feedback button in the tool follows a
  consistent **"Feedback on X"** naming convention (5 top-level buttons, one per tab
  with a main review action, plus 13 per-field buttons on Snapshot/Final Report) —
  replaced a mix of "Ask guide", "Get X feedback", "Ask the guide for feedback" labels.

## Open items — not yet done, don't lose these

1. **Intake Summary download error — defensive fixes shipped, root cause NOT yet
   confirmed.** Laura hit an error downloading the Intake Summary; she gave no
   detail (exact text unknown). Code review turned up no logic bug, so the most likely
   explanation is the `docx` CDN library failing to load or a real-library API mismatch
   this session's own testing can never catch (the CDN is blocked in this sandbox, so
   the whole `.docx` feature has only ever been verified against a mocked library, never
   the real one — flagged as a known risk when that batch shipped). Shipped as a
   same-batch stopgap: (a) the CDN reference moved from a pinned patch version
   (`docx@8.5.0`) to a major-version-only pin (`docx@8`), reducing 404 risk if that exact
   patch was removed; (b) both `.docx` download flows (`downloadIntakeSummary()`,
   `downloadReportDocx()`) now detect a missing `docx` library specifically
   (`docxLibraryReady()`/`docxErrorMessage()`) and show an actionable message instead of
   a generic one, and surface the real error text for any other failure —
   `downloadReportDocx()` previously had no error handling at all, a failure just
   silently reset the button. **Still needed:** ask Laura to try the download again and
   report the exact on-page message this time (it should now be much more specific) —
   only then can the true root cause be confirmed and, if it's a real API-usage bug
   rather than a load failure, fixed properly.

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
