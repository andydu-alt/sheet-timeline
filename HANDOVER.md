# Sheet Timeline — handover

A drag-and-drop Gantt timeline on top of Google Sheets. Owner: Andy Du. Live at
https://andydu-alt.github.io/sheet-timeline/ (GitHub Pages, repo `andydu-alt/sheet-timeline`).

Everything is in one file: `index.html` (HTML + CSS + vanilla JS, no build step, no dependencies
except Google Identity Services for sign-in).

## How to work on it

- Edit `index.html`, commit, push to `main`. GitHub Pages republishes in 1–2 minutes.
- Bump `BUILD_ID` (top of the script) on every change. The status line shows it, so users can
  confirm after a hard refresh (Ctrl+Shift+R) that they run the new version.
- Syntax check before pushing:
  `node -e "const s=require('fs').readFileSync('index.html','utf8');new Function(s.match(/<script>([\s\S]*)<\/script>/)[1])"`
- Never commit tokens, sheet dumps or test data — the repo is published as-is.

## What it does

- Each **product line** (`SOURCES` config) is one tab of the workbook "PT resource management"
  (`1mM_6BOF2_XsmXJudqXMic95eAmxAShVBfbILxm7F86I`): Core (General), Database, IT DR, Physical.
- Rows become tasks; one lane per Assignee. Bars read "Feature • Task" (or "Feature • Activity"
  when the source sets `BAR_DETAIL: "activity"` — Database and Physical).
- Drag = move, handles = resize, vertical drag = reorder priority in the lane (saved per browser
  in localStorage). Every drag asks for confirmation first.
- Right-click a bar: **Add task above/below**, **Remove dependencies**, **Delete task**.
- Toolbar: product line, view (Product Timeline / Dev Comparison), filters (Assignees,
  Activities, Features), zoom, undo/redo, ⚙ Settings (legend, Apply FRD/Spec Formulas,
  Align Waterfall, Re-authenticate).
- **Dev Comparison** (Database only): read-only plan-vs-actual chart per feature, from each
  feature's own "Development Ganttchart" tab (`COMPARE_FEATURES`: Oracle v3, MSSQL). Plan bar on
  top (white, solid border, labelled), actual bar below (filled). Columns are found by header
  name (TASK TITLE, ASSIGNEE/TASK OWNER, START DATE, DUE DATE, ACTUAL START DATE, ACTUAL DUE
  DATE, …); role rows ("Dev Director", "QA Manual", "MILESTONES") are skipped.

## Rules the code relies on — keep them

1. **Columns are found by header name** (`HEADERS` per source, alias arrays allowed). A missing
   header is a hard error, never a guess. Row 1 is re-read on every load.
2. **The numeric tab id is looked up by tab name** (`ensureSheetId`). Tabs get re-created from
   backups under the same name, which changes the id — never hard-code it for API writes.
   (`SHEET_ID` in `SOURCES` only namespaces localStorage.)
3. **Never overwrite a formula cell with a literal** unless that field is the one the user
   dragged, or the user explicitly chose it (Remove dependencies, declined chain shift, delete).
4. **Chains**: a dependent Start is `=WORKDAY(H<prevRow>,1,Holidays!$B$2:$B)`; End is
   `=WORKDAY(G<row>,F<row>*100/<workload>,…)`. Sheets truncates the day count — use
   `estimateForSpan()` when back-solving an estimate. Never offer to chain a task behind one of
   its own dependents (`startFormulaDescendants`) — that is a circular reference.
5. **Group rows**: a row with `=MIN(G7:G78)`-style Start/End is a Feature summary only if its
   Assignee is a group label ("Feature", "MAINTENANCE", …). A person's row with such a formula is
   a real task spanning its sibling rows — shown, but locked (`isDerivedTask`).
6. **Undo history is never renumbered.** Both stacks are LIFO and every row insert/delete is on
   the stack, so each entry replays against the layout it was recorded in. If a structural step
   fails part-way, `historyFailed()` clears history and reloads.
7. **One structural change at a time** — insert, delete, undo/redo, bulk formulas run through
   `exclusive()`; drags and the context menu are blocked meanwhile.
8. **Delete never leaves #REF!**: before removing row R, every formula in the tab that reads R is
   re-pointed (to the predecessor, if the user re-chains) or frozen to its shown value; range
   edges are recorded so undo restores them. It also checks row R still holds that task.
9. **Load speed**: one parallel round of requests (values + formulas + holidays). Permission
   probe and tab-id lookup run in the background. Keep new calls off the critical path — each
   Sheets call on this workbook costs ~1 s.
10. Only the first load jumps to Today; reloads keep the scroll position.

## Testing

There is no real-sheet test environment. The approach used so far:
- Pure-function unit tests in Node: extract functions from `index.html` by name and run them
  (estimateForSpan, parseDate, parseWorkdayFormula, delete-reference detection, cmpParseDate).
- End-to-end in a browser with a **fake Sheets backend** that replaces `window.fetch`: a grid
  seeded from a dump of the Database tab (values + formulas), supporting values get/put/
  batchUpdate and row insert/delete with Sheets-style reference shifting. Scenarios: delete,
  delete with re-chain, delete twice + undo twice, insert above/below a rollup block, double
  undo, failed delete, drag/resize via synthetic pointer events. The check that matters: after
  undo, every formula in the tab equals the original exactly, with no new `#REF!`.
  Stub `requestAnimationFrame` if the browser tab is hidden. Keep dumps out of the repo.
- Still try one action + Undo on a throwaway task in the real sheet after risky changes.

## Access

- Users need edit access to the workbook (and view access to the Dev Comparison sheets).
- Google sign-in uses Andy's OAuth client (`CLIENT_ID`); if it is in "Testing" mode, new users
  must be added as Test users in Google Cloud Console.

## Known gaps

- Formulas in OTHER tabs that point at a deleted row still become `#REF!`.
- No automated test suite is committed; tests above were run ad hoc.
