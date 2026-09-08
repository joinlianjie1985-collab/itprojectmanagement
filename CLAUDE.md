# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page Kanban board for a fictional internal IT PMO ("UOB IT PMO"), used as a demo/training
tool. The entire app is one file: `index.html`, containing all markup, a `<style>` block and a
`<script>` block.

## Hard constraints

These are requirements of the deliverable, not stylistic preferences. Do not relax them without the
user asking:

- **Vanilla only.** No React/Vue/jQuery/Tailwind, no build step, bundler, or npm. There is no
  `package.json` and there should not be one.
- **One file.** Everything lives in `index.html`. Do not split out `app.js` or `styles.css`.
- **Runs from `file://`.** It must work by double-clicking the file — never assume a dev server.
- **No external resources.** No CDN scripts, no web fonts, no image files. Icons are Unicode glyphs
  or inline SVG; type is a system font stack. `formsubmit.co` is the sole permitted network origin.
- **No persistence of any kind.** No `localStorage`, `sessionStorage`, IndexedDB, or cookies. Board
  state is a plain in-memory array; a refresh resetting to seed data is intended behaviour, and the
  header carries a note saying so.
- **No `alert()` / `confirm()`.** Validation errors render inline under each field; card deletion
  uses an inline "Delete? Yes / No" toggle rendered into the card.
- **No `!important`** anywhere in the CSS.

## Architecture

**Single source of truth.** `state = { tasks, filters, ui, nextIdSeq }`. Everything the board shows
is derived from it, and the whole board is rebuilt by `renderBoard()` on every change. Never mutate
card contents in the DOM outside the render path — mutate `state` and re-render.

`state.ui` holds transient per-card interaction state (`confirmDeleteId`, `moveMenuId`,
`draggingId`) precisely *because* rendering blows the DOM away; open menus and confirm rows survive
a re-render only by living in state. `captureFocusKey()` / `restoreFocus()` bracket the innerHTML
swap in `renderBoard()` so keyboard focus is not lost when a click causes a re-render.

**Function roles.** `renderCard()` returns an HTML string (never a node); `renderBoard()` joins
those into the four columns and also drives `renderSummary()` and `renderFilterStatus()`.
`applyFilters()` is pure — it returns a filtered copy and never touches the DOM. `addTask()`,
`moveTask()` and `deleteTask()` are the only mutators of `state.tasks`, and each ends by
re-rendering.

**Counting rule.** Column count badges reflect the *filtered* view; the header summary strip
(total, per status, overdue) always reflects *all* tasks. Keep that split — it's deliberate.

**Escaping.** Every interpolated value in a template string goes through `escapeHtml()`. Since all
rendering is string concatenation into `innerHTML`, an unescaped interpolation is an XSS hole — if
you add a field to the card, escape it.

**Event wiring is delegated.** Listeners are attached once to `#board` (in `wireCardActions()` and
`wireDragAndDrop()`) and dispatch on `data-action` / `.closest()`, because cards are destroyed on
every render. Never attach a listener to an individual card.

**Dates.** `todayISO()` builds the date from local components rather than `toISOString()` to avoid
UTC off-by-one. Dates are stored and compared as `YYYY-MM-DD` strings (lexicographic comparison is
correct for that format); `formatDate()` is locale-independent by design.

**Optimistic submit.** `handleSubmit()` adds the card, closes the modal and toasts success *before*
awaiting `notifyNewTask()`. The FormSubmit call is wrapped in try/catch and a failure only produces
a warning toast — it must never leave the board or the submit button in a broken state.

## FormSubmit

`FORMSUBMIT_ENDPOINT` at the top of the script is the only place the notification email address
appears; keep it that way. FormSubmit needs a one-time activation — the first submission emails a
confirmation link to that address and nothing delivers until it is clicked. Use the AJAX JSON
endpoint (`/ajax/<email>`), not a form POST, so the page never navigates away.

## Verifying changes

There is no test suite and no Node on this machine. To check the script parses and its logic runs,
extract the `<script>` block and drive it through JavaScriptCore with a DOM stub:

```sh
JSC=/System/Library/Frameworks/JavaScriptCore.framework/Versions/A/Helpers/jsc
# extract the last <script> block to app.js, then in a driver script:
#   new Function(src)                      -> parse check
#   (new Function("document","window","fetch", src))(stubDoc, stubWin, stubFetch)
# stub needs: getElementById/querySelector(All)/createElement/addEventListener on document,
# and per-element innerHTML, textContent, value, dataset, classList, appendChild, focus.
"$JSC" driver.js
```

Assert on `store["board"].innerHTML` and `store["summary-strip"].innerHTML` after init. For the
parts the stub can't cover — drag and drop, focus rings, and the sub-768px stacked layout — drive
a real browser through the Playwright MCP server (see below) rather than checking by hand:
`browser_navigate` to the `file://` path, then `browser_snapshot`, `browser_drag`,
`browser_resize` and `browser_take_screenshot`.

## Local tooling

None of this is an application dependency. The app stays vanilla, single-file and buildless; these
are agent tools that happen to live in the repo. Nothing here may be imported by `index.html`, and
no `package.json` or `node_modules` belongs in the project root.

**Node** is at `~/.local/node` (v24 LTS, installed from the official tarball, `bin` added to
`~/.zshrc`). It exists only so `npx` can run the tools below. It is not installed system-wide.

**Playwright MCP** is configured project-level in `.mcp.json` and used for browser verification and
for the README screenshot. Two flags there are load-bearing and should not be dropped:

- `--browser chromium` — the server's default is real Google Chrome, which is not installed on
  this machine. `chromium` works despite being undocumented in `--help`.
- `--allow-unrestricted-file-access` — Playwright blocks the `file:` protocol outright, and this
  app is meant to be opened from `file://`. Note this grants the browser tool access to any
  `file://` URL on the machine, not just this project.

Its scratch directory `.playwright-mcp/` is gitignored.

**Skills** are installed project-level under `.claude/skills/` but are *not* committed — one of the
upstream repos is archived with no licence, so it is not ours to redistribute. `skills-lock.json`
is committed instead and records each skill's source, path and content hash. To restore them:

```sh
npx skills add https://github.com/rysweet/amplihack --skill cybersecurity-analyst --agent claude-code --copy -y
npx skills add https://github.com/saltbo/agent-kanban --skill agent-kanban --agent claude-code --copy -y
npx skills add https://github.com/nextlevelbuilder/ui-ux-pro-max-skill --skill ui-ux-pro-max --agent claude-code --copy -y
```

Do *not* use `npx skills experimental_install` for this. It reads the lockfile and does fetch all
three, but it ignores `--agent` and always writes to `.agents/skills/`, which Claude Code does not
read. The agent identifier is `claude-code`, not `claude`.

**`/publish-github`** (`.claude/commands/publish-github.md`) is the project-level slash command for
shipping: it scans for secrets first, then pushes, deploys Pages, screenshots the live site, and
updates the README and repo About.
