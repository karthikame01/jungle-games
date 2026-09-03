# Jungle Games — Technical Reference

This document is written for an AI agent (or a new developer) picking up this
repository cold. It describes what exists, how it is built, every piece of
state, every extension point, and every known constraint. Read this before
making changes.

## 1. What this repository is

A small collection of self-contained, single-file HTML browser games for
Grade 1 (CBSE-aligned) learners: math and English phonics. There is no build
step, no bundler, no package.json, and no server-side code. Every file is a
complete, standalone HTML document with inline `<style>` and `<script>` — it
can be opened directly in a browser or served by any static file server.

### Files

| File | Purpose | Status |
|---|---|---|
| `little_jungle_games.html` | The main hub. Contains the landing page (game grid) plus both games embedded as switchable "views" in one document. **This is the file to edit and the one that matters going forward.** | Active, primary |
| `mango_tree_math.html` | An early standalone build of the math game only, predating the hub. Kept for reference/history. | Superseded — not maintained in lockstep with the hub's copy of the same game |
| `mic_check.html` | A standalone feasibility probe for browser speech-to-text (see §7). Not part of the game hub. | Diagnostic tool, not a game |

There is no README. This file is the closest thing to one — treat it as
living documentation and update it when the code changes.

## 2. Runtime model

Everything runs client-side in the browser. There is no backend. Persistence
is entirely `localStorage`, scoped to whatever origin serves the file (so
state does **not** carry over between, say, `file://` and a real domain, or
between different hosting origins).

Text-to-speech uses the browser's built-in `window.speechSynthesis`
(Web Speech API). No audio files ship with the project; all spoken words are
synthesized live by the browser/OS.

## 3. `little_jungle_games.html` — document structure

The file is one HTML document with three `<script>`-adjacent regions that
matter:

```
<style> ... one big stylesheet, all games' CSS concatenated ... </style>

<div id="homeView"> ... landing page / game grid ... </div>
<div class="game-view" id="view-mango" hidden> ... Mango Tree Math markup ... </div>
<div class="game-view" id="view-phonics" hidden> ... Letter Tower Phonics markup ... </div>

<script> ... shared view-routing + game grid + ALL of Mango Tree Math's logic (one IIFE) ... </script>
<script> ... ALL of Letter Tower Phonics' logic (a second, separate IIFE) ... </script>
```

**Critical architectural fact:** there are two independent top-level
`<script>` blocks, each wrapped in its own `(function(){ "use strict"; ... })();`
IIFE. This is deliberate — it was done specifically to stop each game's
internal variables (`score`, `current`, `locked`, `tiles`, etc.) from
colliding with the other game's identically-named variables. **If you add a
third game, give it its own `<script>` block and its own IIFE. Do not append
its logic into an existing script block.**

### 3.1 Cross-game view routing — the `window.LJG` namespace

Because each game's script is isolated in its own IIFE, they cannot call each
other's local functions directly. View switching (which `<div class="game-view">`
is visible, and returning to the home grid) is therefore exposed on a shared
global object, `window.LJG`, defined inside the **first** script block
(the one that also owns the home page and the Mango game):

```js
window.LJG = { showGame: showGame, showHome: showHome, gameViews: gameViews };
```

- `gameViews` is `{ mango: <div#view-mango>, phonics: <div#view-phonics> }`.
- `showGame(id)` hides `#homeView`, shows `gameViews[id]`, hides all other
  game views, and scrolls to top.
- `showHome()` hides every game view and shows `#homeView`.

**To add a new game's view to routing:** add its container div's id to the
`gameViews` object literal in the first script block (search for
`var gameViews = {`), and inside the new game's own script, call
`window.LJG.showHome` from its own "Back to Jungle" button's click handler.
The new game's own script does not need to touch `showGame` — that's only
called by the home page's grid-card click handlers.

### 3.2 The game grid (home page)

Rendered entirely from a JS array literal called `games`, inside the first
script block:

```js
var games = [
  { subject: 'Math', accent: '#E4620C', accentSoft: '#FFE3B8', icon: mangoIcon,
    title: 'Mango Tree Math', desc: '...', status: 'active', viewId: 'mango' },
  { subject: 'Reading', accent: '#5B3E82', accentSoft: '#E7DEF3', icon: owlIcon,
    title: 'Letter Tower Phonics', desc: '...', status: 'active', viewId: 'phonics' },
  { subject: 'Shapes', ..., status: 'soon' },   // placeholder, no real game behind it
  { subject: 'Words', ..., status: 'soon' }     // placeholder, no real game behind it
];
```

- `status: 'active'` cards render as a real `<a>`/clickable card wired to
  `showGame(viewId)`.
- `status: 'soon'` cards render as a visually-disabled placeholder with a
  "Coming soon" pill and **do not** correspond to any built game. They exist
  purely as a visual promise of what the jungle will grow into. **Do not
  assume "Shape Safari" or "Word Vine" have any code behind them** — they are
  copy and an icon only.

**To add a new active game to the grid:** add an entry to `games` with
`status: 'active'` and a unique `viewId`, add the matching icon SVG string
above it, and make sure `viewId` matches a key you added to `gameViews` (§3.1).

## 4. Game 1 — Mango Tree Math

DOM ids are all prefixed `mtm-` (Mango Tree Math). CSS classes are prefixed
`.mtm-`.

### 4.1 Concept & world

A monkey (Motu) leaps between three mango branches to tap the correct answer
to an arithmetic equation. Two cheering monkeys stand on the ground and wave
on a correct answer, launching colored party balloons; on a wrong answer they
look worried. The whole scene is hand-built from CSS shapes (no image assets,
no SVG path art) — divs with `border-radius` tricks compose the monkeys,
mangoes, tree, and balloons.

### 4.2 Difficulty system

A `LEVELS` array drives auto-progression by cumulative session score:

```js
var LEVELS = [
  { threshold: 0,  maxNumber: 10, types: ['add'],                             name: 'Sprout' },
  { threshold: 5,  maxNumber: 10, types: ['add', 'sub'],                      name: 'Sapling' },
  { threshold: 10, maxNumber: 20, types: ['add', 'sub'],                      name: 'Climber' },
  { threshold: 20, maxNumber: 20, types: ['add', 'sub', 'missing'],           name: 'Tree Top' },
  { threshold: 35, maxNumber: 20, types: ['add', 'sub', 'missing', 'compare'],name: 'Jungle Master' }
];
```

`getLevelIndex(score)` picks the highest threshold the current score has
crossed. Question types:

- `add` / `sub` — standard arithmetic, `a OP b = ?`
- `missing` — one operand hidden, e.g. `6 + ? = 9` or `? − 3 = 6`
- `compare` — "Tap the BIGGER/SMALLER mango!", three number tiles, no equation

`generateProblem(cfg)` builds one question object
`{ type, prompt (HTML string), answer, options: [3 numbers], dots }`.
`dots` (only populated for `add`/`sub`) drives an optional visual counting-dots
aid (see `renderDots`).

### 4.3 Settings (parent override)

A gear icon opens `#mtm-settingsBackdrop`, backed by a `settings` object:

```js
var settings = {
  autoLevel: true,       // if false, ignore LEVELS entirely
  maxNumber: 20,         // used only when autoLevel is false
  ops: { add: true, sub: true, missing: false, compare: false }, // used only when autoLevel is false
  showDots: false
};
```

Persisted as JSON under `localStorage['mtm_settings']`. `getEffectiveConfig()`
is the single source of truth every question generator call goes through —
it returns `{ maxNumber, types, label }` either from the current `LEVELS`
entry (auto mode) or directly from `settings` (manual mode).

### 4.4 Persistence keys (all in `localStorage`)

| Key | Type | Meaning |
|---|---|---|
| `mangoMathMuted` | `'0'`/`'1'` string | sound on/off |
| `mtm_bestStreak` | stringified int | best-ever consecutive-correct streak |
| `mtm_allTimeMangoes` | stringified int | lifetime total correct answers, across all sessions |
| `mtm_settings` | JSON | the `settings` object above |

Session-only (not persisted): `score`, `streak`, `sessionAnswered`,
`sessionCorrect` — these reset to 0 on every page load.

### 4.5 Session recap

Every 10 answered questions (`sessionAnswered % 10 === 0`), a recap modal
(`#mtm-recapBackdrop`) interrupts play to show accuracy for the last 10
questions, best streak, and lifetime total, before continuing.

## 5. Game 2 — Letter Tower Phonics

DOM ids are all prefixed `pt-` (Phonics Tower). CSS classes are prefixed
`.pt-`. This is the more complex of the two games and the one most likely to
keep growing — read this section fully before touching it.

### 5.1 Concept & world

An owl (Ollie) perched in a "Letter Tower" (a twilight/moonlit tower scene,
visually distinct from the math game's daytime jungle) reacts to correct/
wrong taps. The stage map is a scrollable list of stage cards below a
decorative tower banner (the banner is pure atmosphere, not interactive).

### 5.2 Stage configuration — `PT_STAGES`

Every stage is one entry in a single array, `PT_STAGES`. This is the
**primary extension point** for this game — a new phonics topic is, in the
simple case, just a new entry plus a new word bank array.

```js
var PT_STAGES = [
  { id: 1, mode: 'letter',   title: 'Sound Start',       subtitle: '...', iconText: 'Aa',
    bank: PT_BANK_SOUNDS,    promptLabel: '...' },
  { id: 2, mode: 'word',     title: 'Blend It',          subtitle: '...', iconText: 'cat',
    bank: PT_BANK_CVC,       promptLabel: '...' },
  { id: 3, mode: 'word',     title: 'Digraph Duo',       ..., bank: PT_BANK_DIGRAPH },
  { id: 4, mode: 'word',     title: 'Blend Buddies',     ..., bank: PT_BANK_BLENDS },
  { id: 5, mode: 'word',     title: 'Magic E Spell',     ..., bank: PT_BANK_MAGICE,   hasDemo: true },
  { id: 6, mode: 'word',     title: 'Vowel Team Trail',  ..., bank: PT_BANK_VOWELTEAMS },
  { id: 7, mode: 'syllable', title: 'Syllable Steps',    subtitle: '...', iconText: 'ta·co',
    promptLabel: '...', hasSlider: true }   // no `bank` — see §5.5, uses SYLLABLE_TIERS instead
];
```

Fields:
- `id` — 1-based, must match the entry's position in the array (index + 1).
  Code like `PT_STAGES[activeStage.id]` (used to find "the next stage") relies
  on this. If you ever reorder or delete a stage, renumber `id`s to stay
  contiguous or fix that lookup.
- `mode` — `'letter'`, `'word'`, or `'syllable'`. Determines which question
  generator and which tile-rendering/click-handling path is used (see §5.3
  and §5.5 — these are genuinely different code paths, not configuration of
  one path).
- `title`, `subtitle`, `iconText` — display only. `iconText` is plain text/
  emoji shown in a small badge on the stage card (no icon image assets).
- `promptLabel` — shown above the answer tiles when TTS is available.
- `bank` — the word/letter data array for `letter`/`word` modes (§5.4).
  Omitted for `mode: 'syllable'`, which pulls from `SYLLABLE_TIERS` instead.
- `hasDemo: true` — (Stage 5 only) triggers the one-time interactive "Magic E"
  teaching modal before the first playthrough of that stage (§5.6).
- `hasSlider: true` — (Stage 7 only) shows a gear icon in the play-view HUD
  that opens the complexity-slider settings modal (§5.5).

`QUESTIONS_PER_STAGE = 8` — every stage session is exactly 8 questions,
regardless of mode.

### 5.3 `letter` / `word` mode — the multiple-choice pipeline

This is the pipeline for stages 1–6. Three fixed `<button>` elements exist in
the HTML (`#pt-tile0`, `#pt-tile1`, `#pt-tile2`, inside container
`#pt-tiles`), referenced once into a `tiles` array. Every question just
rewrites their `textContent`/`dataset.value`, it never creates/destroys tile
elements.

- `generateQuestion()` calls `drawEntry()` (pulls one entry from a shuffled,
  self-refilling `deck` built from `activeStage.bank`), then:
  - `mode === 'letter'`: the question is "which letter does this word start
    with?" — `{ spoken: entry.word, answer: entry.letter, options: [correct letter + 2 random distinct letters] }`.
  - `mode === 'word'`: the question is "which spelling matches what you
    heard?" — `{ spoken: entry.word, answer: entry.word, options: shuffle([entry.word, ...entry.distractors]) }`.
- `renderQuestion()` writes the three options into the three tiles, updates
  the prompt text and progress bar, and after 250ms calls `speak(current.spoken)`.
- `handleTileClick(tile)` is bound once, at script init, to all three tiles.
  It locks input, checks `tile.dataset.value === current.answer`, plays the
  correct/wrong owl reaction + confetti/speech, then after 1500ms either
  calls `finishStage()` (if `qIndex >= QUESTIONS_PER_STAGE`) or
  `renderQuestion()` for the next question.

### 5.4 Word bank shape

Two shapes, matching the two non-syllable `mode`s:

```js
// mode: 'letter'  (only PT_BANK_SOUNDS uses this shape)
{ word: 'apple', letter: 'A' }

// mode: 'word'    (PT_BANK_CVC, PT_BANK_DIGRAPH, PT_BANK_BLENDS, PT_BANK_MAGICE, PT_BANK_VOWELTEAMS)
{ word: 'cat', distractors: ['cot', 'cap'] }
```

All distractors are **hand-curated real words**, deliberately chosen to be
close/plausible confusions (off-by-one-letter, swapped vowel, etc.), not
algorithmically mutated strings. **Keep this convention** if you add words —
do not auto-generate distractors by string manipulation, hand-pick them, and
avoid accidental homophones between the target and a distractor (a homophone
pair is unfair in an audio-only quiz — the child can't distinguish them by
ear even if they know the spelling; this was deliberately avoided when
writing `PT_BANK_VOWELTEAMS`, see the comment-free but deliberate absence of
"sea" as a distractor for "see"-sounding words).

### 5.5 `syllable` mode — Stage 7's different mechanic

This is **not** multiple choice. The child taps syllable tiles, shuffled out
of reading order, in the correct sequence to "build" the word aloud. It uses
a second, parallel tile container (`#pt-syllableTiles`, initially empty,
built fresh by JS every question) and a text readout (`#pt-syllableReadout`)
showing progress like `pen · _`. The original three fixed tiles (`#pt-tiles`)
are hidden while this stage is active; `beginQuestions()` branches on
`activeStage.mode` to decide which container to show and which render/click
pipeline to use.

**Data** — three tiers by syllable count, each an array of syllable-string
arrays (no `word`/`distractors` object; the word *is* the array):

```js
var PT_SYLLABLE_TIER2 = [ ['rab','bit'], ['nap','kin'], ... ];   // 24 words
var PT_SYLLABLE_TIER3 = [ ['ba','na','na'], ['el','e','phant'], ... ]; // 18 words
var PT_SYLLABLE_TIER4 = [ ['al','li','ga','tor'], ... ];         // 14 words
var SYLLABLE_TIERS = { 2: PT_SYLLABLE_TIER2, 3: PT_SYLLABLE_TIER3, 4: PT_SYLLABLE_TIER4 };
```

56 words total across the three tiers — deliberately large so the pool
doesn't feel repetitive (this was an explicit user requirement: "not like 10
or 20 words").

**Difficulty — two modes, one settings object:**

```js
var syllableSettings = { auto: true, maxTier: 2 };  // persisted: localStorage['pt_syllableSettings']
```

- **Auto (default):** `syllableAutoTierFor(sessionCorrect)` walks
  `SYLLABLE_AUTO_PROGRESSION` (`[{threshold:0,maxTier:2},{threshold:3,maxTier:3},{threshold:6,maxTier:4}]`)
  and raises the ceiling as the *current session's* correct count rises. This
  resets every time the stage is (re)started — it is not a lifetime
  progression like Mango Math's `LEVELS`.
- **Manual (slider):** when `syllableSettings.auto === false`, the ceiling is
  fixed at `syllableSettings.maxTier` (2–4), set by an `<input type="range">`
  in `#pt-syllableSettingsBackdrop`, opened via the gear icon that only
  appears for stages with `hasSlider: true`.

Either way, `effectiveSyllableMaxTier()` returns the current ceiling, and
`syllablePool(maxTier)` returns the **union** of every tier from 2 up to that
ceiling (not just the top tier) — so turning the slider up both raises
difficulty *and* enlarges the pool of words in rotation, satisfying "if user
wishes to increase the complexity they can use slider and add more words."
This means even at maxTier 4 you will still sometimes see a 2-syllable word;
that's intentional, not a bug.

**Interaction sequencing** — `handleSyllableTileClick(tile)`:
1. Tiles are tagged `data-origIndex` = their position in the *correct* word
   order (not their displayed/shuffled position). This matters because some
   words have a repeated syllable (e.g. "ba·na·na" → two tiles both read
   "na") — matching must be done by original index, never by comparing tile
   text, or a repeated syllable becomes ambiguous.
2. A tap is accepted only if `origIndex === syllableProgress` (the next
   expected index, starting at 0). Correct: tile turns green + disabled,
   that syllable is spoken via TTS, `syllableProgress` increments, readout
   updates.
3. An out-of-order tap flashes red, sets `syllableMistake = true` for this
   question (so it won't count toward `sessionCorrect` even if the child
   eventually finishes it via retries — but retries are unlimited and never
   block progress), and does not advance.
4. When `syllableProgress` reaches the syllable count, `completeSyllableWord()`
   fires: speaks the *whole* word, confetti, `sessionCorrect += (syllableMistake ? 0 : 1)`,
   `qIndex += 1`, then after 1700ms either `finishStage()` or the next question.
5. `syllableTapLocked` is a short-lived debounce (separate from the
   coarser `locked` flag used between questions) that prevents a second tap
   from registering while the previous tap's 550ms feedback animation is
   still playing. **Do not remove this** — without it, rapid double-taps can
   register two syllables in the same tick.

The HUD's existing "hear it again" speaker button still works in this mode —
it speaks `current.spoken` (the *whole* target word), which acts as an
optional hint/fallback if a child gets stuck, at the cost of partially
giving away the puzzle. This was a deliberate tradeoff, not an oversight.

### 5.6 Magic E demo (Stage 5 only)

The first time Stage 5 is entered (guarded by `progress.seenMagicEDemo`,
persisted inside `pt_progress`, see §5.7), `showMagicDemo(onDone)` shows a
modal (`#pt-magicBackdrop`) with the word "cap". Tapping the wand button
(`#pt-wandBtn`) triggers `sparkleBurst()` (particle divs positioned relative
to `#pt-magicWord`, which **requires** `.pt-modal-panel` to have
`position: relative` — this was a real bug once, see §8) and morphs the text
to `c<span class="glow-vowel">a</span>p<span class="silent-e">e</span>`,
speaking "cape". A "Start Stage 5" button then proceeds into the normal
8-question loop. This demo never repeats after the first time, in any
session, on that browser/device (it's a `localStorage` flag, not
per-session).

### 5.7 Progress & stage unlocking

```js
function isUnlocked(){ return true; }  // ALL stages are unlocked, always — see history note below
```

**Important history note:** stages were originally gated (stage *N* required
finishing stage *N−1*). The user explicitly asked to unlock everything, so
`isUnlocked` was simplified to always return `true` and the stage-map
rendering code (`renderStageMap`) was correspondingly simplified to remove
the now-dead "locked card" branch. **Do not reintroduce gating without also
reverting/rewriting `renderStageMap`** (it no longer builds a lock-pill UI at
all).

Stars, not unlocking, are now the only "progress" signal: `finishStage()`
computes `stars = sessionCorrect>=7 ? 3 : sessionCorrect>=5 ? 2 : sessionCorrect>=1 ? 1 : 0`
and stores the **best-ever** stars per stage:

```js
// localStorage['pt_progress'], JSON:
{ "1": { "cleared": true, "stars": 3 }, "2": { "cleared": true, "stars": 2 }, ..., "seenMagicEDemo": true }
```

Note the `seenMagicEDemo` flag lives in the *same* object as the per-stage
star records (a bit of a mixed bag — it's not keyed by stage id, it's a
top-level sibling key). Keep this in mind if you ever iterate
`Object.keys(progress)` expecting only numeric stage ids.

### 5.8 Voice picker

A "Choose Ollie's Voice" button (visible on both the map and play views,
lives in the shared `.pt-header`) opens `#pt-voiceBackdrop`, listing every
voice `window.speechSynthesis.getVoices()` returns, filtered to `lang`
starting with `en` (falls back to showing all voices if that filter yields
zero). Each voice is tagged Female/Male/Voice by a **substring name-match
heuristic** (`FEMALE_HINTS`/`MALE_HINTS` arrays of lowercase substrings like
`'zira'`, `'hazel'`, `'david'`) — there is no real gender metadata exposed by
the Web Speech API, this is a best-effort guess based on common voice names
from Windows/Chrome/Apple TTS engines. Female-tagged voices sort first.

Selection persists as `localStorage['pt_voiceKey']`, a string
`"<voice.name>|<voice.lang>"`. `getSelectedVoice()` re-resolves that key
against the current `getVoices()` list every time `speak()` runs (voice
objects themselves aren't serializable, and the browser's voice list can
change between page loads or load asynchronously — see `onvoiceschanged`
wiring near the voice-picker code). If the previously-selected voice is no
longer present, `speak()` silently falls back to the browser default.

**This voice picker is Phonics-only.** Mango Tree Math has no TTS and no
voice picker.

### 5.9 Persistence keys (all in `localStorage`)

| Key | Type | Meaning |
|---|---|---|
| `phonicsMuted` | `'0'`/`'1'` string | sound on/off |
| `pt_progress` | JSON | per-stage `{cleared, stars}` + `seenMagicEDemo` flag (§5.7) |
| `pt_voiceKey` | string | `"<voice name>\|<voice lang>"` of the chosen TTS voice |
| `pt_syllableSettings` | JSON | `{ auto: bool, maxTier: 2\|3\|4 }` (§5.5) |

Session-only: `activeStage`, `deck`, `current`, `locked`, `qIndex`,
`sessionCorrect`, and (syllable mode only) `syllableProgress`,
`syllableMistake`, `syllableTapLocked`.

## 6. Shared conventions worth knowing before you edit

- **CSS custom properties** are declared once in the top-level `:root` block.
  Mango Math's tokens (`--mango`, `--leaf-*`, `--bark*`, `--monkey-*`, …) and
  Phonics Tower's tokens (`--pt-sky-*`, `--pt-owl-*`, `--pt-tower*`,
  `--pt-parchment*`, …) coexist in the same `:root` — they're namespaced by
  prefix, not by scope, because both games render in the same document.
  `--gold` and `--berry` are shared/reused across both games as the
  semantic "star" and "wrong/error" colors.
- **No dark-mode theming.** Both games commit to one fixed, fully-painted
  visual world each (Mango = daytime jungle, Phonics = twilight tower) and
  intentionally do not respond to `prefers-color-scheme`. This was a
  deliberate choice (an art-directed "single visual world"), not an
  oversight — do not add a dark-theme media query without redesigning both
  games' palettes to have a coherent second state.
- **Fonts:** `Baloo 2` (display/headings/numbers/buttons) + `Nunito` (body
  text), loaded once via a single Google Fonts `<link>` in the `<head>`,
  shared by the whole document including both games and the home page. Keep
  using this pairing for consistency if you add a third game, unless there's
  a strong reason to differentiate it.
- **No canvas, no SVG path art, no image assets.** Every character (Motu the
  monkey, the cheering monkeys, Ollie the owl) is built from plain `<div>`s
  styled with `border-radius`/gradients/transforms. The few SVGs in the file
  are simple icon glyphs (stars, arrows, a mic icon, etc.), not illustration.
  Keep this build technique if you add new characters — it keeps the file
  self-contained and avoids binary asset management.
- **`randInt`/`shuffle` helpers are duplicated**, once per script block
  (once for Mango, once for Phonics) — this is intentional given the
  IIFE-isolation design (§3), not an accidental copy-paste that needs
  deduplicating.
- **Every text-bearing HTML file must start with `<meta charset="UTF-8">`
  before `<title>`.** This was missed twice during development (once on
  `little_jungle_games.html`, once on `mic_check.html`) and caused visible
  mojibake (e.g. "·" rendering as "Â·", em dashes breaking) when the file was
  served without an explicit encoding header. Always check for this first if
  you see garbled punctuation.

## 7. `mic_check.html` — speech-recognition feasibility probe

A **standalone, non-game** diagnostic page, built to answer one question
before committing to building a read-aloud phonics stage: *can this
environment's browser/mic combination even do speech recognition at all?*

- Uses `window.SpeechRecognition || window.webkitSpeechRecognition`
  (the Web Speech API's recognition side — this is a **different** API from
  the `speechSynthesis` used elsewhere in this project; recognition support
  is much narrower).
- Immediately reports browser support in a status banner on load, with no
  user action needed — Chrome/Edge (desktop and Android) support it; Firefox
  does not; Safari/iOS support is inconsistent.
- A big mic button starts/stops `recognition`, showing live interim +
  final transcripts.
- A `normalize()` + `isFuzzyMatch()` pair does lenient substring matching
  between what was heard and a selected target word (chips: cat, banana,
  elephant, rabbit, alligator — these mirror real Phonics word-bank entries
  on purpose) — not exact-string matching, because kids' pronunciation and
  STT accuracy are both imperfect.
- Every attempt (heard text, target, matched boolean) is appended to an
  on-page log; nothing is persisted to `localStorage` or sent anywhere by
  this project's own code.

**Known unresolved question as of this writing:** whether microphone
permission can even be granted inside a Claude "Artifact" (a sandboxed
iframe embedded in claude.ai) was **not yet confirmed** — that requires a
live human test with real hardware and a real permission prompt, which
can't be done from an automated/headless session. `mic_check.html` was
built and handed to the human user specifically to answer that question
before any read-aloud stage is built on this foundation. If you're an AI
picking this up later: check with the user whether that test happened and
what the result was before assuming microphone-based features will work in
this hosting context.

## 8. Known quirks encountered during development (read if debugging weirdness)

- **CSS `border-width` does not accept percentage values.** An earlier
  attempt to build the Phonics Tower's roof as a CSS-triangle using
  percentage `border-left`/`border-right`/`border-bottom` silently collapsed
  to nothing (invalid values are dropped, not rounded/clamped). It was
  rebuilt using `clip-path: polygon(...)` on a percentage-sized `<div>`
  instead — percentages *are* valid for `width`/`height`. If you need a
  CSS-only triangle that scales with a percentage-sized parent, use
  `clip-path`, not a border hack.
- **`position: relative` is required on any container you spawn
  absolutely-positioned particles into.** The Magic E demo's sparkle burst
  was originally positioned wrong because `.pt-modal-panel` had no explicit
  `position`, so the sparkles' `position: absolute` resolved against the
  nearest positioned ancestor (the `position: fixed` backdrop) instead of the
  panel itself, offsetting them. Fixed by adding `position: relative` to
  `.pt-modal-panel`.
- **A decorative element positioned behind another element in the same
  stacking context will be silently hidden if it's earlier in DOM order and
  neither has an explicit `z-index`.** The moon on the Phonics stage-map
  banner was invisible because the tower silhouette (later in the DOM,
  larger, same default stacking level) painted over it. Fixed by giving the
  moon and tower explicit, non-conflicting `z-index` values and moving the
  moon horizontally clear of the tower's footprint.
- **Browser caching during local iteration can make you debug a stale
  file.** While iterating locally against a `python -m http.server` (see
  §9), the same URL was sometimes served from the browser's own HTTP cache
  even after the on-disk file changed, producing confusing "the fix didn't
  work" symptoms that were actually "you're looking at the pre-fix version."
  Append a throwaway query string (`?v=2`, `?v=3`, …) to the URL on every
  reload during local testing to force a fresh fetch.
- **A very-online automated browser tab can end up sharing JS/DOM state
  with a previous tab in the same debugging session** in some sandboxed
  preview environments, making `location.reload()` or opening a "new" tab
  not actually reset in-memory variables the way a real fresh page load
  would. If state looks impossibly stale during automated testing (e.g. a
  view is visible that nothing in the code should have made visible), don't
  assume it's a code bug before verifying with `localStorage.clear()` plus a
  hard, cache-busted navigation.

## 9. How to run this locally

There is no build step. Any static file server works:

```bash
python -m http.server 8743
# then open http://localhost:8743/little_jungle_games.html
```

Opening the file directly via `file://` also works for casual checking, but
some automated browser/testing tools cannot access `file://` URLs — use a
local static server for anything scripted.

## 10. Extension checklist

**Adding a new question type to an existing Phonics word-mode stage:**
just add entries to that stage's bank array (`{word, distractors}`). No code
changes needed.

**Adding a new Phonics stage using the existing `letter`/`word` mechanic:**
1. Add a new `PT_BANK_*` array (`{word, letter}` or `{word, distractors}`
   shape, §5.4).
2. Add a new entry to `PT_STAGES` with the next sequential `id`, `mode:
   'letter'` or `'word'`, and `bank: <your new array>`.
3. Nothing else — the existing pipeline (§5.3), stage map, star system, and
   HUD all pick it up automatically because they iterate `PT_STAGES`
   generically.

**Adding a new Phonics stage with a genuinely new mechanic** (like Stage 7's
syllable-tapping): expect to add a new `mode` value, branch `beginQuestions()`
on it, and write parallel `renderXQuestion()`/`handleXTileClick()` functions,
following the pattern in §5.5. Reuse `finishStage()`, `qIndex`,
`sessionCorrect`, and the stars/progress system as-is — those are
mode-agnostic already.

**Adding a third game entirely:** see §3 — new `<script>` block, own IIFE,
register its view in `gameViews`, add a `status: 'active'` entry to `games`,
wire its own back button to `window.LJG.showHome`.
