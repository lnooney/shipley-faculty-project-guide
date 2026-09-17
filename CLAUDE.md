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
- **`inlineFormatAI()` auto-bolds a bullet's lead-in label**, not just literal `**bold**`
  markdown from the AI — the short span before the first colon or em dash on a line
  (e.g. "Budget alignment: the budget looks incomplete" → "**Budget alignment**: ...")
  gets wrapped in `<strong>` automatically, since this function is the one shared choke
  point every feedback surface already renders through. Skipped when the line already
  has explicit `**bold**` (avoids double-processing), and the leading span is capped
  short so a colon or dash appearing later in a long sentence never bolds half of it —
  a real risk this ships mitigated for, not a hypothetical.
- **Nested bullets (a sub-list under one bullet) were explicitly NOT added.** Laura
  asked whether it was possible; the honest answer is it would require the AI to
  reliably indent sub-bullets in its raw text output, which is exactly the class of
  free-text formatting instruction this tool's own experience has shown the AI doesn't
  follow consistently — it would render correctly only sometimes. The real example she
  gave (a Timeline Feasibility bullet crammed with a duration, an attribution, and a
  question) turned out to need something different: splitting one run-on bullet into
  multiple flat bullets, not nesting — see Proposal Review rubric below.
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
- **The embedded logo follows BU's actual brand guidelines** (checked against the real
  guidelines doc Laura provided, not assumed). Two things confirmed there:
  - **Which variant is correct**: the current logo (red "BU" brick + "Institute for
    Excellence in Teaching & Learning" wordmark, in one combined image) is BU's
    "Signature with department" — explicitly the variant for internal-to-BU use where a
    department needs to be identified. Confirmed with Laura that all three generated
    documents stay within BU (just outside the Institute team), which is exactly the
    internal-audience case this variant is for — BU's guidelines separately require a
    different, "Boston University"-spelled-out variant for external audiences, which
    would NOT apply here. Don't second-guess this without checking with Laura first if
    document distribution ever changes to include non-BU recipients.
  - **Required clear space**: BU's rule is that clear space around the mark must equal
    the height of the red brick, which (confirmed by measuring the actual embedded PNG)
    spans the logo's full image height — 28px at the 380px render width used everywhere.
    Applied as the minimum space above/left of the logo and to its right before any
    adjacent text, in both the Intake Summary (`.intake-header` flex gap,
    `#intake-modal-box`/`#intake-print-clone` padding) and the two `.docx` documents
    (`docxLogoParagraph()`'s spacing, computed from the same logo-height measurement
    rather than hardcoded, so it stays correct if the logo's render size ever changes).
    This is a minimum, not an exact value — the Intake Summary's actual rendered space
    above the logo ends up larger than 28px because the logo sits in a flex row next to
    a taller two-line title block and gets vertically centered within that row, which is
    still fully compliant (more than the minimum is fine; only *less* would violate it).
- **Generated document filenames follow one convention everywhere:**
  `{Faculty PI last name}_{Doc type}_{MMDDYY}`, built by `buildDocFilename(docType)`
  (`getFacultyLastName()` + `getFileDateStamp()`, both just above `renderAIResponse()`).
  Applies to all 7 AI feedback "Save as PDF" surfaces (Proposal Review, Impact Framework
  Review, Timeline, Budget, Final Report, all 13 per-field guidance buttons, Ask Guide
  chat — `renderAIResponse()`'s third `filenameBase` argument, threaded through to
  `saveAIResponseAsPDF()`), the Intake Summary print form (`printIntakeSummary()` swaps
  `document.title` in just for the print, restored via `afterprint`), and the Milestone
  Snapshot/Final Report `.docx` downloads. Laura's explicit request, replacing what was
  previously either a generic shared name (every AI feedback PDF used to say
  "{Project Title} — Guide Feedback" regardless of surface, so saving two different
  kinds of feedback for the same project could silently collide) or an undated one
  (Milestone Snapshot in particular gets downloaded repeatedly over a project's life for
  periodic check-ins, so needs a date to avoid every download sharing one filename).
  `getFacultyLastName()` is a lightweight heuristic on the free-text "Lead(s)" field —
  strips a leading title (Dr./Prof./etc.), takes the first-listed name if multiple leads
  are given, strips a trailing suffix (Jr./Sr./PhD/etc.), takes the last remaining word.
  Falls back to "Project" if the field is empty. Not bulletproof for every possible name
  format, but handles the common cases correctly (verified against "Dr. Jane Smith",
  "Smith, Jane", "Jane Smith, PhD", "Prof. John A. Doe Jr.", "Jane Smith and Bob Jones",
  hyphenated last names). The downloaded Project Plan Excel (`downloadProjectExcel()`)
  now follows this convention too (`buildDocFilename('Project_Plan')` — was
  `{Project Title}_Project_Plan.xlsx`, predating the convention entirely); the standalone
  Impact Framework Excel export still uses its own pre-convention `{Project Title}_...`
  name and wasn't touched by this pass — only the file explicitly flagged.
- **Every AI feedback "Save as PDF" document's printed subhead now names its own
  feedback type**, not just the filename: `Shipley Project Guide {Type} Feedback —
  {date}` (e.g. "Shipley Project Guide Budget Feedback — September 17, 2026"), replacing
  a single generic "Shipley Project Guide feedback — {date}" used identically across all
  7 surfaces. `feedbackTypeLabel(docType)` (just above `renderAIResponse()`) derives the
  `{Type}` by stripping a docType's trailing `_Feedback` (and, for Ask Guide chat's
  `Guide_Chat`, its leading `Guide_`) and turning remaining underscores into spaces —
  reusing the exact same `docType` string each call site already passes to
  `buildDocFilename()` for the filename, so the two now come from one source instead of
  needing separately-maintained text. `renderAIResponse()` takes a 4th `docType`
  argument threaded through to `saveAIResponseAsPDF()`'s 3rd argument for this purpose;
  all 7 call sites (Proposal Review, Impact Framework Review, Timeline, Budget, Final
  Report, all 13 per-field guidance buttons, Ask Guide chat) pass the same docType
  string they already pass to `buildDocFilename()`.

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
- **Both markers are persisted, not just applied to the live DOM.** They used to vanish
  on any page reload or saved-project switch, since they were plain CSS classes never
  written to `localStorage` and never reapplied by `loadSetup()` — confirmed by Laura
  reporting the green/amber distinction had "been lost," and reproduced directly (a real
  browser reload wiped every marker while field values themselves persisted correctly).
  Fixed by persisting `auto_filled_fields` (which field IDs the Guide filled) and
  `import_completed` (whether an import has ever run for this project) in `data.setup`,
  tracked in memory via `autoFilledFieldIds`/`setupImportCompleted` and kept in sync by
  the same clear-on-edit listener. `applyFieldMarkers()` (renamed from
  `highlightUnfilledFields()`) now runs both right after an import and from `loadSetup()`
  on every load/switch, gated by `import_completed` so a pristine project that's never
  had an import run doesn't show amber with no import having explained why.
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

**Intake Summary (Project Reports tab)**
- Lives on the Project Reports tab (moved from Setup, Laura's explicit request) — it's
  the very first card on the tab, above the Milestone Snapshot/Final Report header card
  itself (also Laura's explicit request, moved up from originally sitting just above the
  mode toggle), since it's a standalone document independent of which report mode is
  selected and she wanted it to be the first thing a PM sees on the tab.
- **No longer generates a `.docx` — replaced with an editable print/PDF form** after the
  `.docx` download failed twice with the identical unresolved error even with a CDN
  fallback in place (see the old entry below, kept for the postmortem). This sandbox can
  never test the real `docx` library (its CDN is blocked here), so continuing to guess
  at fixes for a feature that could never actually be verified stopped being a
  responsible path. `openIntakeSummaryForm()` (replaces `downloadIntakeSummary()`) does
  the same data-gathering and AI narrative synthesis as before, but renders it into a
  modal (`#intake-modal`) as plain HTML with every field `contenteditable="true"` (and
  real `<input type="checkbox">`s for Support Needs) instead of building `docx.Document`
  objects — a PM edits directly in place, then "Print / Save as PDF"
  (`printIntakeSummary()`, plain `window.print()`) to finish. Zero external dependency,
  and fully testable in this sandbox for the first time.
- **Printing a modal reliably across multiple pages took real care.** A
  `position:fixed` element nested inside the modal's own `overflow:auto` box can't be
  paginated correctly by the browser's print engine — an early version of this rendered
  as an essentially blank PDF (confirmed empirically: a `page.pdf()` byte-count sanity
  check, not just a screenshot, since `fullPage` screenshots don't represent real print
  pagination either for `position:fixed` content). Fixed by giving the page a dedicated
  `#intake-print-clone` element that's a direct child of `<body>`, deliberately outside
  the modal's fixed/overflow ancestor chain; `printIntakeSummary()` clones the modal's
  live (PM-edited) `#intake-print-area` HTML into it right before calling
  `window.print()`, and the print stylesheet hides everything else on the page via
  `display:none` (not `visibility:hidden` — that still reserves layout space and would
  push real content down the page) so the clone prints in plain, normally-paginating
  document flow. Verified by generating a PDF from deliberately long content and reading
  the PDF's own `/Count` (page count) back out — confirmed 3 real pages, not 1 clipped
  page — since a screenshot alone can't be trusted to catch this class of bug.
- **Confirmed bug, now fixed: the modal never appeared at all for Laura, on the very
  first real-world use.** `#intake-modal` had been placed nested inside the Timeline
  tab's `.panel` div (copy-pasted next to the Gantt chart modal, which lives there
  safely since its own trigger button is also on that tab). Every `.panel` that isn't
  the active one has `display:none`, which hides its entire subtree regardless of a
  nested element's own inline `display:block` — but the "Edit & Print Intake Summary"
  button lives on the Project Reports tab, a different panel, so the modal was reliably
  invisible any time a PM opened it from a tab other than Timeline (which is to say,
  essentially always). Symptom matched exactly: the AI narrative call completed and the
  button's loading state cleared normally, but nothing visibly appeared — Laura herself
  correctly guessed the actual cause ("didn't specify a space for the intake summary to
  appear") before it was confirmed. Fixed the same way as the print-clone element above:
  moved to be a direct child of `<body>`. **Testing-methodology gap this exposed:** every
  verification of this feature before shipping checked
  `document.getElementById('intake-modal').style.display === 'block'`, which only
  confirms the element's own inline style, not whether it's actually rendered once
  ancestor CSS is considered — completely missing this class of bug. Re-verified after
  the fix with Playwright's `locator.isVisible()` (which does account for ancestor
  `display:none`) and a real `boundingBox()` check, specifically repro'ing the real path
  (open the app on one tab, navigate to Project Reports, click the button there) rather
  than the shortcut of calling the function directly without changing tabs first. Any
  future modal should be checked this same way, not just via its own `style.display`.
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
- **Postmortem on the abandoned `.docx` approach:** Laura's exact error text confirmed
  `window.docx` was undefined when the button was clicked. A CDN fallback (jsdelivr →
  unpkg) didn't change the outcome — she got the identical generic error again, which
  (since that fallback only triggers on an actual load failure) was the signal that the
  real problem likely wasn't a blocked/failed request at all, and that further CDN-level
  patches weren't going to reliably fix something this sandbox could never test in the
  first place. That's what motivated dropping `.docx` for this document entirely rather
  than attempting a third blind fix — see above. `docxLibraryReady()`/`docxErrorMessage()`
  still exist and are used by `downloadReportDocx()` (Milestone Snapshot/Final Report
  still generate real `.docx` files, unaffected by this change — no reported issues
  there) — untouched, only the Intake Summary's own usage of them was removed.

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
- **TIMELINE FEASIBILITY's per-component bullet template was tightened** after a real
  example bullet came back as a run-on wall of text — a duration estimate, a phase
  breakdown with two sub-date-ranges, an attribution of who does what, and a
  confirm-with-faculty question, all crammed into one bullet for "Technology
  development/customization/integration." The template's `[duration estimate] — [note
  or question]` shape had no cap on what could go into the second slot. Now each bullet
  must hold exactly one distinct point (duration as a single figure, never a phase
  breakdown; the second slot one short clause, never both a note and a question), with
  an explicit escape hatch to split a genuinely multi-phase component into a second
  bullet instead of one long one — capped at 2 bullets per component, never combining
  two different components into the same bullet. Same principle as the tool's other
  exact-caps-over-vague-adjectives fixes (Follow-ups word caps, etc.) — a template shape
  with an unbounded slot will get filled with whatever fits, so the slot itself needs
  the cap, not just a general request to "be concise."
**Budget tab**
- **"Tailor budget to this proposal" relabels to "Re-tailor budget to this proposal"
  once tailoring has run at least once** — same treatment and reasoning as the
  Milestones button above, keyed off `Array.isArray(data.budget_flags)` (that field is
  always set, even to an empty array, on every successful run — deliberately not the
  same `.length` check the Reset button uses, since that one specifically means "there
  are visible flags left to clear," a different question).
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
- **The Setup tab's "Proposed Budget Line Items" card has been removed entirely** — its
  own extraction/manual-entry staging area (`budgetLineEntries`, `data.setup.budget_lines`,
  `extractBudgetLinesFromProposal()`, etc.) was a *third* "confirmed" source Tailor Budget
  drew from alongside the proposal narrative and the spreadsheet. Two earlier attempts to
  fix a reported budget-overage discrepancy — spreadsheet-priority-over-narrative, then
  sourcing this card's extraction from the spreadsheet too — narrowed the redundancy but
  didn't eliminate it, and Laura reported the issue was still occurring. Removing the
  card removes the redundancy at its root instead of resolving it downstream in the
  tailoring prompt: budget line items now exist in exactly one place, the Budget tab
  itself. `tailorBudgetToProposal()`/`updateBudgetTailorGuard()` now read only
  `data.proposalText` and `data.budgetSheetText` — no third source, no way for the same
  payment to be represented two different ways going into that AI call. Two copy
  references elsewhere (the Technology tools note, the worker hire-date hint) that used
  to point PMs at "Proposed Budget Line Items above" were updated to point at the
  Budget tab instead.
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
- **"Tailor milestones to this proposal" relabels to "Re-tailor milestones to this
  proposal" once tailoring has run at least once** (`updateMilestoneTailorGuard()`,
  keyed off the same `data.task_relevance`/`data.investigate_questions` presence check
  already used to show/hide the Reset button). This button is usually already redundant
  the moment a PM finishes an import, since `chainTabAutofills()` runs the identical
  function automatically — deliberately relabeled rather than hidden, since a PM may
  legitimately want to re-run it after a revised proposal, and the tool already treats
  re-tailoring as a normal, expected action via the existing Reset-button precedent.
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
- **Milestones 2–5's tasks now carry `group` values** in `MILESTONE_TASKS`, same as
  Milestone 1 always has (Implementation/Data Collection for M2; Analysis/Dissemination
  for M3; Reflection/Iteration/Scaling & Sustainability for M4; Reporting/Dissemination/
  Closeout for M5) — previously only Milestone 1 tasks had a `group`, so the "Group"
  column in the downloaded Project Plan Excel export was blank for four of the five
  milestones. This only feeds the Excel export and the Tailor Milestones prompt's task
  listing (both already read `t.group`) — it deliberately does **not** turn on the
  grouped-with-subheaders rendering the Milestones tab itself uses for M1
  (`renderMilestones()`'s `if(m==='m1')` branch is unchanged, so M2–M5 keep rendering as
  a flat task list on-screen); adding real values to an already-read field, not adding a
  new UI behavior.
- **Downloaded Project Plan Excel (`downloadProjectExcel()`), "Tasks & Timeline" sheet,
  reformatted for readability** (Laura's explicit request — she liked the existing
  milestone color coding but found the sheet "a bit hard to read"):
  - Each milestone phase now gets a **full-width, color-filled banner row** (merged
    across all 8 columns, using the same `mColors` palette as before) inserted right
    before that phase's task rows, replacing the old per-cell fill on just the
    "Milestone" column (which was easy to miss scanning down a long sheet). The
    "Milestone" column itself is still populated on every task row (useful if a PM sorts
    or filters), just no longer separately colored/bolded now that the banner carries
    that signal — the old vertical cell-merge logic for that column was removed since it
    would visually fight with the new banner rows.
  - Font sizes: 16pt bold white for both sheets' top column-header row, 14pt bold white
    for the new milestone banner rows, 12pt for all body/task rows — all per Laura's
    explicit point sizes.
  - Added while already reformatting, as concrete answers to Laura's open "anything else
    that would make it easier to use and read?": the header row is frozen
    (`views:[{state:'frozen',ySplit:1}]`) on both sheets so column labels stay visible
    scrolling a long plan; light alternating row banding within each phase (between
    banner rows, so it doesn't fight the phase color-coding); `wrapText` on the Task and
    Notes columns so long text no longer gets clipped.
  - **Not yet done, flagged to Laura rather than silently changed**: this file's
    downloaded name (`{Project Title}_Project_Plan.xlsx`) still predates and doesn't
    follow the `buildDocFilename()` `{Faculty Last Name}_{Doc Type}_{MMDDYY}` convention
    every other generated document now uses — worth aligning if she wants it, but out of
    scope for a formatting-only request.
  - Verified via a functional in-memory mock of the `ExcelJS` API (real CDN load is
    blocked in this sandbox, same limitation as the old `.docx` library — see Intake
    Summary postmortem below) that asserts on the actual object structure `downloadProjectExcel()`
    builds: 5 banner rows in the right order/color/font size, 16/14/12pt sizes applied
    where expected, "Group" populated for every M2–M5 row, and both sheets' frozen views
    — not just that the function runs without throwing.

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
- **Downloaded Framework Excel (`downloadFrameworkExcel()`) redesigned around showing
  the connections between sections, not just listing each section's items separately**
  (Laura's explicit request, with a worked example: "Problem A is addressed by Objective
  1, which will be recognized as successful by Impact M and measured by Measure R via
  Target 1 which will be collected by Source and frequency C and analyzed by Data
  Analysis Z"). Previously the sheet was six stacked sections, each just its own bullet
  list or small table, in one narrow two-column layout — there was no way to see which
  Objective addressed which Problem, etc.
  - **The underlying data gap this ran into**: Problems/Objectives/Impacts/Data Analysis
    are each stored as one free-text blob (a PM types bullet lines into one textarea per
    section) with no ID linking a specific line in one section to a specific line in
    another; Measures & Data Collection are the only two sections already structured as
    row tables, and even those aren't linked to a specific Impact. There was no existing
    data in the app that says "this Objective addresses that Problem."
  - **Two ways to close that gap were discussed with Laura**: (1) restructure the
    Framework tab itself so each item is entered with an explicit "which prior item does
    this relate to" link as the PM builds it — most accurate, but a real change to how
    PMs fill out the tab, turning free-text bullets into individually-linked items; or
    (2) leave the tab exactly as it is and have the Guide infer the connections at
    export time from the existing free text. Laura chose (2) specifically because the PM
    already reviews the whole framework with the faculty lead before treating anything
    as final, so she'd rather keep data entry as free-text bullets and have the PM/
    faculty team catch and correct any wrong AI-inferred connections during that
    existing review — explicitly asked that the export "direct PMs to review and revise
    the connection logic."
  - Implementation: `downloadFrameworkExcel()` sends all six sections' current content
    to the Guide (skipped entirely if fewer than 2 sections have anything in them — one
    section alone has nothing to connect) with a literal-JSON-template prompt (this
    tool's established reliable pattern for AI output the app needs to act on) asking
    for one row per distinct Problem→Objective→Impact→Measure→Target→Source→Frequency→
    Data Analysis chain it can actually trace from the text — explicitly told to leave a
    field blank rather than force a weak match, to never collapse a real one-to-many
    branch (one Problem serving multiple Objectives, etc.) into a single row, and to
    copy every field verbatim rather than paraphrasing. The same prompt also asks for
    each section's items that didn't fit into any chain (`unmatched_problems`,
    `unmatched_objectives`, etc.) — every single bullet/row from every section must land
    in a connection row, an unmatched list, or both; the export never silently drops an
    item a PM entered. If the AI call is skipped, fails, or returns nothing usable, every
    item still falls through into "unmatched" rather than vanishing from the file.
  - The sheet itself: a bold amber disclaimer banner directly above the table — "These
    connections were drafted by the Guide from your framework entries — review and
    revise them with your faculty lead before relying on them" — using the same
    amber alert styling (`--amber-bg`/`--amber-text`) already used for cautionary
    messages elsewhere in the app, satisfying Laura's explicit ask to direct PMs to
    review/revise rather than just labeling it quietly. Below that, one 8-column table
    (Problem/Objective/Impact/Measure/Target/Data Source/Frequency/Data Analysis), one
    row per connection. Each column is color-coded by its originating section (Measure/
    Target share one color, Source/Frequency share another, matching the 6 original
    sections) — the "color-code sections" part of Laura's broader ask, now expressed as
    column color since the layout moved from stacked sections to columns. Font sizes
    match the same 16/14/12pt tiers established for the Project Plan export (title/
    column headers/body). Consecutive rows that share the same Problem (and, one level
    deeper, the same Problem+Objective) get their leftmost matching cells vertically
    merged, so a one-to-many branch reads as a visible tree instead of repeating the
    same Problem text on every row — merging is guarded to require every column to its
    left to also match, so two rows are never merged just because they coincidentally
    share the same Objective text under two different Problems. A frozen header row and
    wrapped text on every column round out the readability pass (also part of Laura's
    ask).
  - **Unmatched items stay in their own originating column, not a separate generic
    list** (Laura's explicit correction to the first version, which had pulled every
    leftover item into a plain two-column "Section | Item" list below the table). Below
    the same full-width amber "Not yet connected" banner (the "color filled row label
    across all columns" she asked for), each column independently stacks its own
    leftover items top-down — a leftover Problem sits in the Problem column, a leftover
    Data Analysis method sits in the Data Analysis column, etc. — using the exact same
    column positions and colors as the connections table above, so nothing changes
    columns between the two sections. A row here carries no cross-column meaning (row 3's
    leftover Problem and row 3's leftover Impact are NOT implied to be related) — each
    column just fills down independently, *except* Measure+Target and Source+Frequency,
    which do share a row, because that specific pairing is already known from how the
    data was entered (each is one row in `fwMeasureRows`/`fwCollectionRows`), not
    something the Guide is guessing at. The prompt's JSON shape reflects this:
    `unmatched_measures`/`unmatched_collection` are arrays of `{measure,target}`/
    `{source,frequency}` pairs rather than flat strings, so that known pairing survives
    into the export; `unmatched_problems`/`unmatched_objectives`/`unmatched_impacts`/
    `unmatched_analysis` stay plain string arrays since those are genuinely single-column.
  - Filename now also follows the `buildDocFilename()` convention
    (`buildDocFilename('Impact_Framework')`, was the older pre-convention
    `{Project Title}_Impact_Framework.xlsx`), same as the Project Plan export fix above.
  - Verified the same way as the Project Plan export — a functional in-memory mock of
    the `ExcelJS` API (real CDN load is blocked in this sandbox) with a mocked `fetch`
    standing in for the Guide's connection-mapping call, asserting on actual structure:
    font sizes at each tier, the 8 column header colors, a one-Problem/two-Objective test
    case producing exactly the expected Problem-column merge (and confirming Objective/
    Impact correctly do NOT merge when their ancestor chain differs), a mixed unmatched
    case (a lone leftover Problem plus a leftover Measure/Target pair, from otherwise
    fully-connected sections) landing in the correct columns on the correct rows below
    the still-full-width banner, and the corrected filename pattern.

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
- Tab order is now **Project Setup → Timeline → Budget → Impact Framework → Project
  Reports → Ask Guide re: Project**. Project Reports moved to sit right after Budget
  first (was last); Impact Framework then moved from 2nd position to sit right after
  Budget too, ahead of Project Reports (both Laura's explicit requests, two separate
  batches). Only the `<nav class="tabs">` button order changed each time —
  `switchTab()` shows/hides panels by `id`, not DOM position, so the panel `<div>`s
  themselves never needed to move.
- The tab's nav label is now **"Project Reports"** (was "Project Report") — a
  visible-label-only swap, same pattern as the Milestones → Timeline rename: `id="tab-
  btn-report"`, the panel's own card title ("Project Report"), `switchReportMode()`, and
  everything else internal are unchanged. The Follow-ups card's "Go to Project Reports"
  link label (`FOLLOWUP_SOURCE_META.report.tabLabel`) was updated to match.
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

None currently. The Intake Summary `.docx` download issue (previously the sole open
item) was resolved by replacing `.docx` generation for that document with an editable
print/PDF form — see "Intake Summary" above.

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
