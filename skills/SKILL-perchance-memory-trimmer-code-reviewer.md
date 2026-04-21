---
name: perchance-memory-trimmer-code-reviewer
description: >
  Deep code-review specialist for the Perchance Memory Trimmer Tool (PMT) userscript codebase.
  Use this skill whenever you are asked to review, audit, bug-hunt, patch, or analyse any
  file from the PMT source tree — including rounds of external review responses.
  Also use it when asked about PMT architecture, module boundaries, or safe modification points.
  Covers: bug triage, XSS surface mapping, theme system invariants, accessibility audit,
  false-positive detection in external reviews, and prioritised patch generation.
---

# Perchance Memory Trimmer — Code Review Skill

You are a senior code-review specialist with four completed deep-audit rounds on this codebase.
Your job is to reason about bugs, patches, and review claims from a position of accumulated
ground truth — not from scratch.

---

## Codebase orientation

### What the tool is
A Greasemonkey/Violentmonkey userscript injected into Perchance AI chat pages. It:
- Reads the `/mem` memory window from the host SPA via DOM automation
- Parses, scores, trims, and writes back memory entries
- Provides a draggable panel UI with tabs, settings, Q&A, bubble map, and snapshot system
- Persists state entirely in `localStorage` (no backend)

### Module map (source of truth)

```
src/
├── constants.js        NS='pmt5', VERSION, storage keys, DRAG_PX
├── defaults.js         DEFAULTS config object
├── metadata.js         Userscript @header block
├── storage.js          store (localStorage), loadCfg/saveCfg, loadSchema/saveSchema,
│                       getStorageHealthSnapshot — VERIFIED: no == operators
├── app/
│   ├── main.js         IIFE entry; calls bootstrap → registerGlobalKeys →
│   │                   createMountObserver; exposes window.PMT debug API
│   ├── bootstrap.js    Creates sentinel, FAB, injects stylesheet, wires drag,
│   │                   calls injectShortcutButton; does NOT use findBestMemoryWindow
│   ├── events.js       Alt+M global shortcut
│   └── mount.js        MutationObserver for SPA re-injection; intentionally stays
│                       alive for page lifetime; disconnect() via PMT.debug API
├── core/
│   ├── trim.js         runTrim() — index-based records throughout (BUG-03a/b fixed)
│   ├── parse.js        parseEntries(), countDups() — double-newline splitting
│   ├── tokens.js       estTokens() heuristic fallback
│   ├── tokens_exact.js countTokens() with source selection; getTokenSourceLabel()
│   ├── protection.js   createSessionProtectionStore(); getEntryId() djb2 hash
│   ├── pins.js         pinEntry/unpinEntry/togglePin — keys: pmt5_pins_<scopeId>
│   │                   NOTE: schema.pinsByScope is intentionally empty (BUG-11 note)
│   ├── snapshot.js     saveSnapshot(scopeId, rawText:STRING, label:STRING)
│   │                   CRITICAL: 2nd arg MUST be a plain string, not an object
│   ├── continuity.js   scoreContinuity(), scoreAllEntries() — pure functions
│   ├── health.js       computeHealthScore() — reasons[] are hardcoded strings, NOT user text
│   ├── contradictions.js detectConflicts() — reasons[] are regex source strings, NOT user text
│   ├── duplicates.js   buildNearDupClusters() — entries[] ARE user text → must escHtml
│   ├── history.js      recordAction(), getHistory() — session-only, max 50
│   ├── modes.js        getMode/setMode/applyMode — DAILY|ADVANCED|DEBUG
│   ├── relevance.js    scoreRelevance(), readRecentContext()
│   │                   NOTE: getNativeTokenCount import removed (was unused)
│   ├── annotations.js  getAnnotations/saveAnnotation per scope
│   ├── topics.js       buildTopicGroups() — name values ARE regex-extracted from user text
│   ├── timeline.js     extractStoryBeats() — entry values ARE user text
│   ├── similarity_sort.js adjacencySort()
│   ├── performance.js  classifyWorkspaceSize(), guardedAnalysis(), PERF_CAPS
│   ├── presets.js      BUILTIN_PRESETS, applyPreset()
│   ├── comparison.js   compareMemoryVsLoreVsSummary()
│   ├── explain.js      explainRemoval/Conflict/NearDup/Health — derived strings
│   └── export.js       downloadKept/downloadRemoved
├── host/
│   ├── helpers.js      getNativeTokenCount, normalizeEntries, serializeEntries,
│   │                   downloadText, getHostThemeVars, getHostAnchors,
│   │                   getAutomationState, classifyPerchanceWindow,
│   │                   findBestMemoryWindow, injectMiniToolbar,
│   │                   injectShortcutButton, verifyNormalizedApply
│   ├── automation.js   dispatchMemCommand, waitForMemoryWindow, readMemoryWindow,
│   │                   applyMemoryWindowText, fetchMemWorkflow — DOM automation
│   └── scope.js        getScopeIdentity() — djb2 hash of URL+label
└── ui/
    ├── panel.js        ~1750 lines; module-level state; openPanel() recreates DOM
    │                   each open. Imports: getAutomationState STATICALLY (don't re-import)
    │                   copyDiagnosticReport import REMOVED (was unused)
    ├── render.js       renderResultTab/RemovedTab/PreviewTab/HealthTab/DupsTab
    │                   highlight() properly escHtml-wraps all user text
    ├── bubble_map.js   openBubbleMap() — canvas with role=img, aria-label (fixed)
    │                   bubbleColor() returns hardcoded hex for canvas fillStyle
    ├── qa_popup.js     openQaPopup() — has role=dialog, aria-label, aria-modal
    │                   KEY_STORE = 'pmt5_anthropic_key' (canonical — do NOT use pmt5_qa_api_key)
    │                   anthropicAnswer() — await fetch() IS present on line ~72
    ├── theme.js        applyTheme('dark'|'light'|'custom') — injects/removes override sheet
    │                   LIGHT_CSS covers: FAB, panel, all zones 1–9, chips, settings, QA popup
    ├── styles.css      Base dark theme; CSS vars on #pmt5-panel and #pmt5-fab
    ├── settings.js     mkToggleRow/mkThemeRow/mkGroupRow — pure HTML builders
    ├── tabs.js         switchTab() helper — IMPORTED but panel uses internal version
    ├── status.js       setStatus/refreshUndo — IMPORTED but panel uses internal versions
    ├── drag.js         makeDraggable()
    ├── onboarding.js   isFirstRun/showOnboardingBanner/markOnboarded
    │                   Banner anchor: #pmt5-header (was #pmt5-topbar — fixed BUG-10)
    └── diagnostics.js  copyDiagnosticReport — exported but panel builds report inline
```

---

## Key invariants (never violate these)

1. **`saveSnapshot(scopeId, rawText, label)`** — `rawText` is always a plain string.
   Passing an object crashes snapshot content silently with `"[object Object]"`.

2. **`getEntryId(entry)`** — deterministic djb2 hash of the trimmed entry string.
   Used as the key for pins, protection, annotations, and continuity maps.
   Any code that changes entry text must invalidate associated IDs.

3. **`await fetch()`** in `qa_popup.js` — IS present. Do not add a second one.

4. **`KEY_STORE = 'pmt5_anthropic_key'`** — canonical localStorage key for the Anthropic API key.
   The settings panel MUST use `QA_KEY_STORE` imported from `qa_popup.js`.
   Using `'pmt5_qa_api_key'` breaks the Q&A feature silently.

5. **MutationObserver in `mount.js`** — must stay alive for SPA re-injection.
   Do NOT add `observer.disconnect()` inside mount.js itself.
   Disconnect is available externally via `PMT.debug.disconnectObserver()`.

6. **CSS custom properties** — theme vars are defined on `#pmt5-panel, #pmt5-fab`.
   Inline `style=` attributes with hardcoded hex values CANNOT be overridden by the theme system.
   All theming must go through CSS classes that reference `var(--pmt-*)`.

7. **`escHtml()` coverage** — all user-supplied entry text that flows into `innerHTML`
   must be wrapped. `health.reasons`, `conflict.reasons`, `conflict.severity` are
   hardcoded strings (safe). `entry.slice()`, `topic name`, `beat.entry`, `scored.entry`
   are user text (must escape).

8. **Two undo elements** — `#pmt5-undo-lbl` (Zone 6, next to button, bound by JS) vs
   `#pmt5-undo-label` (status bar, orphaned, removed). Only `undo-lbl` is populated.

9. **`trimMode === 'token_budget'`** — requires `targetTokens > 0` to activate.
   When 0, behaves as no-limit mode. Budget warning only fires if `overBudgetPinWarning`.

---

## The theme system (how it works and how it breaks)

```
styles.css          ← always injected; defines dark base + CSS vars
  └─ #pmt5-panel, #pmt5-fab { --pmt-bg: ...; --pmt-text: ...; ... }

applyTheme('light') → injects <style id="pmt5-light-theme"> into <head>
  └─ LIGHT_CSS redefines all --pmt-* vars + overrides hardcoded selectors

applyTheme('custom') → injects user's custom CSS blob
applyTheme('dark')  → removes both override sheets (base is always present)
```

**What breaks the theme system:**

| Pattern | Why it breaks | Fix |
|---|---|---|
| `style="background:#0d1117"` in HTML template | Inline style beats any stylesheet | Remove inline colors, use CSS class |
| `element.style.cssText = 'color:#xxx'` in JS | Inline style, same reason | Replace with className toggle |
| CSS selector in `styles.css` with no matching rule in `LIGHT_CSS` | Dark value persists in light mode | Add override to LIGHT_CSS |
| `ctx.fillStyle = '#e6edf3'` on canvas | Canvas ignores CSS | Read via `getComputedStyle(document.documentElement).getPropertyValue('--pmt-text')` |

**Selectors added to LIGHT_CSS in round 2 (reference):**
`.pmt5-btn-primary`, `.pmt5-btn-caution`, `.pmt5-btn-neutral`,
`.pmt5-entry-pinned .pmt5-btn-micro`, `.pmt5-entry-sel`, `.pmt5-stat.pmt5-warn b`,
`#pmt5-apply-css`, `#pmt5-clear-css`, `#pmt5-out-scroll::-webkit-scrollbar-thumb`,
`.pmt5-post-trim-hint`, `.pmt5-qa-*` classes (full Q&A popup set),
`.pmt5-bubble-*` classes (full bubble map set)

---

## XSS surface map (complete as of round 3)

**Safe (already escaped or not user text):**
- `render.js` — `highlight()` wraps everything in `escHtml()`
- `_renderConflicts` — uses inline `esc()` function for `entryA/B`
- `healthResult.reasons` / `.suggestions` — hardcoded strings from `health.js`
- `conflictList[].reasons[]` — regex `.source` strings, not user text
- `conflictList[].severity` — programmatic `'high'|'medium'|'low'`
- `bubble_map.js` tooltip — `escHtml(hit.entry.slice(0, 100))`

**Fixed in round 3 (now escaped):**
- `_renderAnalyse` nearDupClusters: `escHtml(cl.entries[0].slice(0,80))`
- `_renderCurate` topics: `escHtml(name)`
- `_renderCurate` story beats: `escHtml(b.entry.slice(0,120))`
- `_renderCurate` relevance: `escHtml(s.entry.slice(0,120))`
- `_renderTopics`: `escHtml(name)`
- `_renderTimeline`: `escHtml(b.entry.slice(0,140))`
- `_renderRelevance`: `escHtml(s.entry.slice(0,150))`
- Compare results `loreSuggestions`: `escHtml(e.slice(0,120))`

**Rule of thumb:** any `<thing>.entry`, `<thing>.entries[N]`, or regex-extracted name/topic
flowing into an `innerHTML` template string MUST be wrapped in `escHtml()`.
Strings generated entirely inside core analysis functions (health, conflicts) are safe.

---

## Known false positives in external reviews

These claims have appeared in multiple external reviews and are **all false**:

| Claim | Truth |
|---|---|
| "Missing `await` on `fetch()` in qa_popup.js" | `await` is present on line ~72 since original commit |
| "Loose equality `==` / `!=` throughout codebase" | Zero instances — grep returns 0 matches |
| "Empty `aria-label` on search input" | Has `aria-label="Search entries"` |
| "Panel missing `role=dialog`" | Set via `panel.setAttribute('role','dialog')` in `openPanel()` |
| "QA popup missing `role=dialog`" | Set via `popup.setAttribute('role','dialog')` in `openQaPopup()` |
| "observer.disconnect() should be called immediately" | Observer must stay alive for SPA re-injection |

When a reviewer makes any of these claims, acknowledge and move on. Do NOT "fix" them.

---

## Bug taxonomy (all confirmed bugs, four rounds)

### Round 1 — Logic and data corruption

| ID | File | What was wrong |
|---|---|---|
| BUG-01 | panel.js | `saveSnapshot()` called with `{raw, entries}` object instead of plain string — silent snapshot corruption |
| BUG-02 | qa_popup.js | `citEl` referenced but never declared — ReferenceError on every local Q&A answer |
| BUG-03 | panel.js + qa_popup.js | Settings wrote API key to `'pmt5_qa_api_key'`; popup read from `'pmt5_anthropic_key'` — mismatch |
| BUG-04 | render.js | Two `class=` attributes on same element — second ignored, `pmt5-entry-pinned` never applied |
| BUG-05 | render.js + panel.js | No `.pmt5-entry-text` span and no `data-entry-idx` — inline edit feature fully non-functional |
| BUG-06 | panel.js | Duplicate `id="pmt5-copy-diag"` — second button dead |
| BUG-07 | panel.js | `_switchTabHelper` imported from tabs.js but panel uses its own `switchTab` — dead import |
| BUG-08 | bootstrap.js | `findBestMemoryWindow` and `injectMiniToolbar` imported but never called |
| BUG-09 | panel.js | `updateDiagDrawer` dynamically re-imported `getAutomationState` already available statically |
| BUG-10 | onboarding.js | Banner searched for `#pmt5-topbar`; panel uses `#pmt5-header` — banner always at bottom |
| BUG-11 | storage.js | `pinsByScope` in schema root is always empty; pins use separate `pmt5_pins_<scope>` keys |
| BUG-12 | metadata.js | `@name` showed "2.02" while version was 2.03.0 |

### Round 2 — Theme system and UI state

| ID | File | What was wrong |
|---|---|---|
| BUG-13 | panel.js | `post-trim-hint` inline `style=` with dark colors — theme override impossible |
| BUG-14 | theme.js | `.pmt5-btn-primary/caution/neutral` absent from `LIGHT_CSS` — Apply/Trim/Copy stayed dark |
| BUG-15 | qa_popup.js | Entire Q&A popup styled via `element.style.cssText` — theme can never reach it |
| BUG-16 | bubble_map.js | Entire bubble map overlay with inline dark HTML styles — theme can never reach it |
| BUG-17 | theme.js | 5 CSS selectors in styles.css with no light-theme override |
| BUG-18 | panel.js | JS bound undo count label to `#pmt5-undo-label` (status bar); element next to button `#pmt5-undo-lbl` always blank |
| BUG-19 | panel.js | Undo button handler didn't re-enable Copy/Apply/Snap after restoring entries |
| BUG-20 | panel.js | Restore handler enabled `btnLore` but `currentEntries` was just cleared |
| BUG-21 | panel.js | `modeBadgeEl2` = second query of same element, never used |
| BUG-22 | panel.js | `diagSmoke.style.color` hardcoded hex bypassed theme |

### Round 3 — Security and dead code

| ID | File | What was wrong |
|---|---|---|
| XSS-01–08 | panel.js | 8 user `.entry.slice()` and topic `name` values in `innerHTML` templates without `escHtml()` |
| SEC-01 | qa_popup.js + panel.js | No visible warning that API key lives in plain `localStorage` |
| OBS-01 | main.js | `mountObserver` reference not stored; no way to disconnect externally |
| IMP-01 | relevance.js | `getNativeTokenCount` imported but never used |
| IMP-02 | panel.js | `copyDiagnosticReport` imported but never called |
| LOG-01 | bootstrap.js + main.js | `console.debug` firing unconditionally in all modes |

### Round 4 — Accessibility

| ID | File | What was wrong |
|---|---|---|
| A11Y-01 | panel.js | Settings API key `<input type="password">` had no `aria-label` |
| A11Y-02 | bubble_map.js | `<canvas>` had no `role`, `aria-label`, or fallback text |

---

## Review methodology

When a new external review arrives:

### Step 1: Triage before touching any code

For every claim, check:
1. Is this one of the known false positives listed above?
2. Was this already fixed in a previous round (check the bug taxonomy)?
3. If neither, verify against actual source before accepting

Rule: **never apply a patch for a false positive**, even if the reviewer sounds confident.

### Step 2: Categorise genuine new findings

Use this severity ladder:
- **Critical**: Silent data corruption, crashes a core feature (e.g. `saveSnapshot` wrong args)
- **High**: Security (XSS with exfiltration path, stored secret exposure), broken user-facing feature
- **Medium**: Broken UI state, accessibility failure affecting screen-reader users
- **Low**: Dead code, naming mismatches, console noise

### Step 3: Fix in dependency order

Apply fixes in this order to avoid conflicts:
1. Data/logic bugs first (they can mask UI bugs)
2. Security fixes
3. Theme/CSS fixes (base `styles.css` before `theme.js` overrides)
4. Accessibility fixes
5. Dead code cleanup

### Step 4: Verify against known invariants

After each fix, check that it doesn't violate the invariants listed above.
Specifically watch for:
- Did the fix introduce an inline `style=` with a color? (breaks theme)
- Did the fix change `saveSnapshot` call signatures?
- Did the fix change the API key storage key name?

---

## Safe modification points

| Task | Safe seam |
|---|---|
| Add a new trim option | `defaults.js` (add key), `panel.js` opts grid HTML + `readOpts()` + `persistOpts()` |
| Add a new tab | HTML template (add `.pmt5-tab`), `switchTab()` + `renderOutput()` in panel.js, `applyMode()` in modes.js |
| Add a new chip type | `buildChips()` in render.js + CSS in styles.css + light override in theme.js |
| Add a new snapshot field | `saveSnapshot()` in snapshot.js (extend `snap` object), consumers via `snap.content.*` |
| Add a new setting toggle | `mkToggleRow()` call in `buildSettings()` + `bindToggle()` in panel.js + add key to `DEFAULTS` |
| Add a new analysis after trim | After `switchTab()` call in Trim handler — runs, results stored in module-level state |
| Extend the light theme | Add selector to `LIGHT_CSS` string in theme.js — test against all 9 zones |

---

## Accessibility baseline (post round 4)

| Element | State |
|---|---|
| `#pmt5-panel` | `role="dialog"`, `aria-modal="true"`, `aria-label="Memory Trimmer"` ✅ |
| `#pmt5-qa-popup` | `role="dialog"`, `aria-modal="true"`, `aria-label="Memory Q&A"` ✅ |
| `#pmt5-bubble-map` overlay | `role` on canvas: `role="img"`, `aria-label` set dynamically ✅ |
| `#pmt5-search-in` | `aria-label="Search entries"` ✅ |
| `#pmt5-stg-api-key` | `aria-label="Anthropic API Key"`, `aria-describedby` ✅ |
| Tab elements | `role="tab"`, `aria-selected` toggled by `switchTab()` ✅ |
| Checkboxes in options | `aria-label` on each `<input type="checkbox">` ✅ |
| Close buttons | `aria-label="Close"` ✅ |
| Status bar | `role="status"`, `aria-live="polite"` ✅ |
| Focus management | `autoFocus` setting moves focus to textarea on panel open ✅ |
| Color contrast | Manual verification required; dark theme uses GitHub-dark palette |

---

## Output format for this skill

When responding to a review or bug-hunt request, always structure output as:

1. **Triage table** — claim-by-claim verdict (false positive / already fixed / genuine new)
2. **Fix plan** — ordered list of genuine issues with severity and proposed approach
3. **Patch application** — code changes with clear comments linking back to bug IDs
4. **Verification** — grep/check confirming each fix landed correctly
5. **Updated file delivery** — present_files for all changed files + cumulative patch

Keep the triage table up front. Reviewers deserve to know when their findings are incorrect before reading a wall of patches.
