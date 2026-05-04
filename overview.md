# Bingo Card Generator — Project Overview

A fully static, serverless bingo card generator hosted on GitHub Pages. No backend, no database — all state lives in URLs and localStorage.

---

## Core Concept

Three distinct pages, zero server dependencies:

| Route | Purpose |
|---|---|
| `/` | **Generator** — input words, configure card, produce N shareable links |
| `/card` | **Player view** — loads a fixed card from URL, handles clicks, detects wins |
| `/host` | **Host view** — loads all N cards from URL, previews each one |

Everything is encoded in the URL. The generator outputs links. The links contain the card. The host link contains all cards.

---

## URL Encoding Scheme

All data is stored in the URL hash (`#`) not query params, to avoid server-side handling and keep URLs clean.

### Card URL
```
/card#<base64url(lz-compressed(JSON))>
```

Card payload:
```json
{
  "seed": "abc123",
  "grid": 5,
  "items": ["item1", "item2", ...],
  "freeSpace": true,
  "winCondition": 3,
  "title": "Office Bingo"
}
```

- `seed` — unique identifier per card, used in win images for attribution
- `grid` — 3, 4, or 5
- `items` — the specific subset assigned to this card, pre-shuffled into position
- `freeSpace` — boolean, centre square override
- `winCondition` — integer 1 to max lines (1 = first line wins, max = blackout)
- `title` — optional display label

### Host URL
```
/host#<base64url(lz-compressed(JSON))>
```

Host payload:
```json
{
  "title": "Game Night",
  "cards": [
    { "seed": "abc123", "url": "/card#..." },
    { "seed": "def456", "url": "/card#..." }
  ]
}
```

The host view reconstructs and renders each card from its embedded URL payload.

---

## Page 1: Generator (`/`)

### Input
- Large text area accepting:
  - Plain list (one item per line)
  - CSV (comma-separated, single row or column)
  - TSV (tab-separated)
  - Parser auto-detects format on paste
- Item count shown live (e.g. "32 items — enough for 6 unique 5×5 cards")

### Configuration
| Setting | Options |
|---|---|
| Grid size | 3×3, 4×4, 5×5 |
| Free space | Toggle (centre square) |
| Win condition | Slider 1 → max lines |
| Number of cards | 1–N (constrained by item pool) |
| Card set title | Optional text label |

### Card Generation Logic

Cards are generated so that:
1. Each card draws a **random subset** of items from the pool (no guaranteed overlap)
2. Items are **shuffled into random positions** within each card
3. The seed is a **short random ID** (e.g. nanoid, 8 chars) unique to each card
4. If the pool is large enough, subsets can have **zero overlap** between cards

Minimum items required:
- 3×3 (no free space): 9 items
- 4×4: 16 items
- 5×5 (with free space): 24 items
- 5×5 (no free space): 25 items

For non-overlapping subsets: `num_cards × grid²` items needed (minus free spaces).

### Output
- List of N card links (copyable individually)
- **Copy all links** button
- **Open host view** button → generates and opens the `/host` URL
- Small preview thumbnail of each card inline

---

## Page 2: Player Card (`/card`)

### First Visit — Name Prompt
On the **base `/card` URL** (no hash), show a name entry screen:
- Input: "Enter your name to receive your card"
- Name is stored in **localStorage** under key `bingo_player_name`
- Once set, the name persists across all cards opened on that device
- Name is shown in a small footer on the card and embedded in the win image

> **Implementation note:** The name prompt only appears on `/card` with no hash. If a hash is present, the card loads directly. This means the host sends the direct card URLs. The name is captured from localStorage — set when the user first lands on the base URL or prompted inline if not yet set.

### Card Rendering
- Renders from the decoded URL payload
- Grid is drawn as a CSS grid (3×3, 4×4, or 5×5)
- Free space (if enabled) is the centre cell, pre-marked, non-clickable
- Each cell shows the item text, sized to fit

### Click Behaviour
- Clicking an unmarked cell **marks it** with a subtle visual (colour fill + checkmark)
- A small ISO 8601 timestamp is written into the cell on click (e.g. `2024-11-03T14:32:07Z`)
  - Displayed in a small, low-contrast font at the bottom of the cell
  - Not user-removable (audit trail)
- Clicking a marked cell **unmarks it** (removes mark and timestamp)

### State Persistence
Click state is saved to localStorage keyed by card seed:
```
bingo_state_<seed> = { "item_index": "ISO_timestamp", ... }
```
State is restored on page reload automatically.

### Win Detection
Win is checked after every click. Win lines checked:
- All rows
- All columns  
- Both diagonals
- Free space counts as marked

Win condition (set at generation time) determines threshold:
- `winCondition: 1` → first completed line triggers win
- `winCondition: 3` → three lines needed
- `winCondition: max` → full blackout

### Win State
On win:
1. All winning cells are highlighted distinctly
2. A **win banner** appears with confetti or similar animation
3. A **"Save win image"** button is prominently shown

### Win Image (Canvas-rendered PNG)
Exported via HTML Canvas (no external library dependency):

Image contents:
- Card title (if set)
- Full bingo grid with all marked cells shown (including timestamps in small text)
- Free space marked
- Player name (from localStorage)
- Card seed
- Win timestamp (ISO 8601, UTC)
- Simple decorative border

Image is exported as a downloadable PNG via `canvas.toBlob()` + an `<a download>` trigger. Filename: `bingo-win-<seed>-<name>.png`.

---

## Page 3: Host View (`/host`)

### Layout
- Page title from payload
- Grid of card previews (2 or 3 columns)
- Each card preview shows:
  - Card seed label
  - Miniature rendered grid (CSS, not canvas)
  - Copy link button
  - Open card button (new tab)

### Purpose
A quick visual reference for the game organiser. No interaction with game state — purely a distribution and preview tool. The host view URL itself is shareable.

---

## Sample Cards

Baked into the site as named presets selectable in the generator:

| Name | Description |
|---|---|
| Office Meeting Bingo | "Circle back", "Let's take this offline", etc. |
| Movie Night Bingo | Tropes and clichés |
| Tech Conference Bingo | "Disruptive", "AI-powered", "Scalable", etc. |
| Road Trip Bingo | Visual/event items |
| Custom | Blank — user provides all items |

Presets are a JS array in a `presets.js` file, easily extensible.

---

## Technical Stack

**Fully static — no build step required.**

| Concern | Solution |
|---|---|
| Compression | `lz-string` (CDN) — compresses JSON before base64 encoding |
| Unique IDs | `nanoid` (CDN or inline 50-line implementation) |
| Canvas export | Native browser Canvas API |
| Styling | Vanilla CSS with CSS variables for theming |
| No framework | Plain JS — maximally forkable |

All external dependencies loaded from `jsDelivr` CDN with integrity hashes.

### File Structure
```
/
├── index.html          # Generator
├── card.html           # Player card
├── host.html           # Host view
├── style.css           # Shared styles
├── bingo.js            # Shared logic (encoding, card gen, canvas export)
├── presets.js          # Sample card data
└── README.md
```

---

## GitHub Pages Deployment

### Setup
1. Push to `main` branch (files at repo root or `/docs`)
2. Enable GitHub Pages in repo Settings → Pages → Source: main branch
3. Site live at `https://<username>.github.io/<repo>/`

### Optional: Custom Domain
Add a `CNAME` file at repo root containing your domain. Configure DNS as per GitHub Pages docs. With a custom domain the URLs become much shareable (e.g. `bingo.yourdomain.com/card#...`).

### README Strategy
The README should:
- Embed 2–3 live example card links (pre-generated, real URLs)
- Show a screenshot of the generator, a card in play, and a win image
- Include a one-click "Generate your own" button linking to the live site
- Document the preset format so contributors can add new sample sets

### "Use this template" Setup
Mark the repo as a template in GitHub settings. Forks get their own hosted instance. Document in README how to add custom presets to `presets.js`.

---

## Implementation Phases

### Phase 1 — Core
- [ ] URL encode/decode utilities (`bingo.js`)
- [ ] Card generation algorithm (subset + shuffle)
- [ ] `card.html` — render, click, localStorage state

### Phase 2 — Generator
- [ ] `index.html` — input parsing, configuration UI
- [ ] Multi-card generation with seed output
- [ ] Host URL generation

### Phase 3 — Win + Export
- [ ] Win detection logic
- [ ] Canvas win image renderer
- [ ] Win animation

### Phase 4 — Host View + Polish
- [ ] `host.html` — preview grid
- [ ] Presets (`presets.js`)
- [ ] README with live examples
- [ ] Mobile responsive pass

---

## Key Constraints & Decisions

- **No server, no auth, no database** — all state in URL + localStorage
- **URLs are immutable** — a generated card link always produces the same card
- **Name is device-local** — stored in localStorage, not in the card URL (privacy)
- **Win images are self-contained** — seed + name + timestamp make each image uniquely attributable without any server verification
- **lz-string over encryption** — data is compressed + base64url encoded, not encrypted; it is non-trivially human-readable but not secure. This is intentional — the goal is compact URLs, not secrecy.
- **No QR codes in win image** — kept simple; the seed on the image is the verification mechanism
