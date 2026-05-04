# Bingo Card Generator — AI Build Prompt

Use this prompt (with the relevant phase section) when building each file. Always include the Style Block and the relevant Phase section.

---

## Style Block (include in every prompt)

```
Follow this style guide precisely:

VISUAL MODE: Builder UI for generator/host pages. Stage (dark) for the active card page.
FONT: Montserrat from Google Fonts — body/inputs 0.85rem, labels 0.8rem muted, buttons 1rem, H1 2rem 800 weight.
PALETTE:
  Page bg:       #f3efed
  Card/panel bg: #edeae8 / #fff
  Borders:       1px solid #d4ceca, border-radius 10–12px cards, 5–6px inputs
  Buttons:       bg #efa551, hover #d48a3a, white text, border-radius 5px
  Accent/focus:  #ffd966 (range, checkbox, input focus ring)
  Muted text:    #7a6f6a
  Dark stage bg: #222034
  Section titles: 0.7rem, uppercase, letter-spacing 0.1em, color #7a6f6a, border-bottom #d4ceca
LAYOUT: Two-column (260px panel + 1fr preview) for generator. Centered single-column for host. Fullscreen stage for card.
ACCESSIBILITY: viewport meta, lang="en", labelled inputs, <button> elements only.
SINGLE FILE: All CSS in <style>, all JS in <script> at end of <body>. CDN links fine. No build tools.
CSS VARIABLES: Use :root custom properties for any value used 3+ times.
```

---

## Phase 1 — `card.html`

```
Build card.html — the player-facing bingo card page.

DEPENDENCIES (CDN):
- lz-string: https://cdn.jsdelivr.net/npm/lz-string/libs/lz-string.min.js

DATA FORMAT: Card state is in window.location.hash, base64url-decoded then LZ-decompressed to JSON:
{
  seed: string,        // short unique ID e.g. "abc12345"
  grid: 3|4|5,
  items: string[],     // pre-shuffled, length = grid²  (free space replaces centre if freeSpace:true)
  freeSpace: boolean,
  winCondition: number // 1 to max possible lines
  title: string
}

FEATURES:
1. NAME PROMPT: On page load, if localStorage key `bingo_player_name` is absent AND no hash present, show a centered name entry screen (warm builder style). Store name to localStorage on submit.
   If hash IS present but name not set, show a small dismissible name prompt overlay before revealing the card.

2. CARD RENDER: CSS grid (3×3/4×4/5×5). Each cell shows item text sized to fit.
   Free space: centre cell, pre-marked, no click, labelled "FREE".

3. CLICK BEHAVIOUR:
   - Click unmarked cell → mark it (fill accent colour + checkmark overlay).
   - Write ISO 8601 UTC timestamp into cell in 0.6rem, low-contrast text at cell bottom. Not removable.
   - Click marked cell → unmark (removes mark and timestamp).
   - State saved to localStorage key `bingo_state_<seed>` as { itemIndex: isoTimestamp }.
   - Restore state on load.

4. WIN DETECTION: Check after every click. Count completed lines (rows + cols + 2 diagonals). Free space counts as marked.
   Trigger win state when completed lines >= winCondition.

5. WIN STATE:
   - Highlight winning cells distinctly (purple #45004D bg, white text).
   - Show win banner overlay with confetti (CSS keyframe animation, no library).
   - Show "Save win image" button prominently.

6. WIN IMAGE (native Canvas, no html2canvas):
   Draw onto a canvas element (hidden): full grid, all cell text, marked cells filled, timestamps in small text,
   player name (from localStorage), card seed, card title, win timestamp (ISO UTC).
   Export via canvas.toBlob() → <a download="bingo-win-<seed>-<name>.png">.

STAGE STYLE: Card page uses dark #222034 background, white text, Montserrat.
```

---

## Phase 2 — `index.html` (Generator)

```
Build index.html — the bingo card generator.

DEPENDENCIES (CDN):
- lz-string: https://cdn.jsdelivr.net/npm/lz-string/libs/lz-string.min.js
- nanoid (inline 50-line implementation, no CDN needed — use Math.random + base36)

LAYOUT: Two-column — 260px left panel (config), 1fr right panel (output/preview).
Collapse to single column below 600px.

LEFT PANEL — CONFIG:
- Textarea: paste words (auto-detects CSV, TSV, or newline-separated). Show live count: "32 items".
- Preset selector: <select> with options: Office Meeting Bingo / Movie Night Bingo / Tech Conference Bingo / Road Trip Bingo / Custom. Loading a preset fills the textarea.
- Grid size: radio or segmented control — 3×3 / 4×4 / 5×5.
- Free space: checkbox toggle.
- Win condition: range slider 1 → max lines (update max dynamically when grid changes).
  Show label: "First line" at 1, "Blackout" at max, numeric otherwise.
- Number of cards: number input 1–20. Show warning if item pool too small for non-overlapping subsets.
- Card set title: text input (optional).
- "Generate Cards" CTA button (#efa551).

RIGHT PANEL — OUTPUT (shown after generation):
- List of N card rows: seed label + full URL (readonly input) + Copy button.
- "Copy all links" button.
- "Open host view" button → encodes host payload → opens /host.html#<encoded>.
- Small card preview thumbnail per card (CSS grid, no interaction, items shown as small text).

CARD GENERATION LOGIC:
- Shuffle full item pool.
- Slice into non-overlapping subsets of grid² items (minus 1 if freeSpace).
- For each subset, shuffle into card position order.
- Assign a seed: 8-char alphanumeric random ID.
- Encode each card as: btoa(LZString.compress(JSON.stringify(payload))) → card URL = /card.html#<encoded>.
- If pool too small for non-overlapping: warn user, allow overlap with a confirmation.

SAMPLE PRESETS (hardcoded JS array):
- Office Meeting Bingo: "Let's take this offline", "Circle back", "Action item", "Bandwidth", "Synergy", "Deep dive", "Pivot", "Ecosystem", "Deliverable", "Touch base", "Move the needle", "Boil the ocean", "Low-hanging fruit", "Paradigm shift", "Blue-sky thinking", "Stakeholder buy-in", "Agile", "KPI", "Drill down", "Cadence", "Alignment", "Roadmap", "At the end of the day", "Leverage", "Value add"
- Movie Night Bingo: "Jump scare", "Love interest", "Comic relief", "Mentor dies", "Training montage", "Third act breakup", "Secret identity", "Car chase", "Surprise twist", "Chosen one", "Sidekick saves day", "Villain monologue", "Post-credits scene", "Slow clap", "Hero's doubt", "Flashback", "Damsel rescued", "Time pressure", "One-liner before kill", "Redemption arc", "False ending", "Reluctant hero", "The real villain", "Sacrifice", "They kiss in the rain"
- Tech Conference Bingo: "Disruptive", "AI-powered", "Scalable", "Machine learning", "The cloud", "Blockchain", "Move fast", "10x engineer", "Unicorn", "Iterate", "Frictionless", "Democratise", "Hacker", "Full stack", "Data-driven", "Platform", "Ecosystem", "API-first", "Series A", "Founder story", "Pain point", "User journey", "We're hiring", "Impact", "Change the world"
- Road Trip Bingo: "Petrol station snacks", "Wrong turn", "Someone needs the toilet", "Speed camera", "Argument about music", "Fog", "Roadworks", "Rest stop selfie", "Nearly missed exit", "Unexpected detour", "Service station coffee", "License plate game", "Falling asleep", "Phone dies", "Are we there yet", "Rain starts", "Sun visor down", "Motorway jam", "Radio ad twice", "Cow field", "Wind farm", "Someone's cold", "Bridge", "Overtaken by a van", "Roundabout confusion"
```

---

## Phase 3 — `host.html`

```
Build host.html — the host/organiser view.

DEPENDENCIES (CDN):
- lz-string: https://cdn.jsdelivr.net/npm/lz-string/libs/lz-string.min.js

DATA FORMAT: window.location.hash decoded to:
{
  title: string,
  cards: [{ seed: string, url: string }]
}

LAYOUT: Centered single-column. Max-width 960px.
Show page title from payload.title (fallback: "Bingo Host View").

CARD GRID: CSS grid, auto-fill columns min 240px.
Each card tile (panel style: #edeae8, border-radius 12px, border #d4ceca):
- Seed label (0.7rem uppercase muted)
- Mini card preview: render the card payload from the card URL hash (decode + LZ decompress) as a small read-only CSS grid. Items as small text (0.5rem), free space cell labelled "FREE".
- "Copy link" button
- "Open card" button (opens card URL in new tab)

If hash is absent or malformed, show a friendly error state: "No host data found. Generate cards from the generator."
```

---

## Shared Encoding Utility (include in every file's `<script>`)

```js
// Encode payload to URL hash
function encodePayload(obj) {
  return btoa(LZString.compress(JSON.stringify(obj)))
    .replace(/\+/g,'-').replace(/\//g,'_').replace(/=+$/,'');
}
// Decode URL hash to payload
function decodePayload(hash) {
  try {
    const b64 = hash.replace(/-/g,'+').replace(/_/g,'/');
    return JSON.parse(LZString.decompress(atob(b64)));
  } catch { return null; }
}
```
