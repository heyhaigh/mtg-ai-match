# AI MTG Match

**Head-to-head autonomous Magic: The Gathering battles with no human players.**

A browser-based AI-vs-AI MTG simulator that fetches real cards from [Scryfall](https://scryfall.com/), renders actual card art, and plays full games with strategic combat, spell casting, and land-based mana — all on autopilot.

![Format: Standard & Commander](https://img.shields.io/badge/format-Standard%20%7C%20Commander-blueviolet)
![Cards: Scryfall API](https://img.shields.io/badge/cards-Scryfall%20API-orange)
![Players: AI only](https://img.shields.io/badge/players-AI%20vs%20AI-green)

---

## Features

### Real MTG Cards
- All cards fetched live from the **Scryfall API** at page load
- Real card names, art, power/toughness, oracle text, and keywords
- 6 deck archetypes: Mono-White, Mono-Blue, Mono-Black, Mono-Red, Mono-Green, Boros Aggro
- Basic lands (Plains, Island, Swamp, Mountain, Forest) with authentic Scryfall art

### Game Formats
- **Standard**: 60-card decks (20 unique cards × 3 copies + 20 lands), 20 starting life
- **Commander**: 100-card singleton decks (63 unique cards + 37 lands), 40 starting life
- Toggle between formats with a dropdown — decks rebuild automatically from Scryfall

### AI Strategy Engine
- **Strategic turn phasing**: Main Phase 1 (removal + haste creatures) → Combat → Main Phase 2 (deploy)
- **Aggressive combat**: creatures attack by default, only holding back tiny creatures when heavily outnumbered
- **Lethal detection**: AI goes all-in when it can swing for the kill
- **Smart blocking**: favorable trades, even trades for big threats, chump-blocking when facing lethal
- **Card evaluation**: context-aware scoring — removal when opponent has threats, draw when hand is low, board wipes when behind

### Card Mechanics
- **Keywords parsed from oracle text**: haste, flying, vigilance, trample, deathtouch, lifelink, menace, first strike, double strike, reach, hexproof, indestructible, defender, flash, ward
- **Haste**: creatures attack the turn they enter
- **Summoning sickness**: properly tracked and cleared at start of controller's turn
- **ETB triggers**: draw, tokens, life gain, damage, destroy, bounce
- **Attack triggers**: "when ~ attacks" abilities fire during combat
- **Spell effects**: damage (targeted + AoE), destroy, exile, bounce, draw, life gain/drain, discard, buffs, tokens, board wipes
- **Combat keywords**: flying (only blocked by flying/reach), trample (excess damage to player), deathtouch (any damage is lethal), lifelink (gain life equal to damage dealt), first strike, menace

### Land & Mana System
- AI plays one land per turn (first action in Main Phase 1)
- Mana generated from lands on the battlefield — no auto-mana
- Lands displayed in their own row with real Scryfall card art
- Multi-color decks split lands between their colors

### Visual Experience
- **Card art**: real Scryfall art crops rendered as card backgrounds
- **Card play announcements**: full-art overlay (250×349px, MTG aspect ratio) with player badge
- **Game start animation**: "⚔️ BATTLE START" overlay with deck matchup
- **Victory screen**: "🏆 VICTORY" overlay with winner, game number, and turn count
- **10-second auto-rematch**: countdown with randomized deck pairs for variety
- **Match log**: timestamped event feed showing plays, attacks, damage, and deaths
- **Responsive layout**: scrollable, left-aligned controls, board + log side by side

### Auto-Play Loop
- Press **Start match** and the AI plays indefinitely
- After each game: results overlay → 10s countdown → random new decks → next game
- Pause/Resume and Reset controls
- Series tracking (games played, wins per deck)

---

## Quick Start

### Option 1: Local file
```bash
# Just open index.html in a browser
open index.html
```
> Note: Requires internet access for Scryfall API calls.

### Option 2: Local server
```bash
cd mtg-ai-match
python3 -m http.server 8080
# Open http://localhost:8080
```

### Option 3: Any static host
Upload the 3 files (`index.html`, `app.js`, `styles.css`) to any static hosting:
- GitHub Pages
- Netlify
- Vercel
- Cloudflare Pages
- S3 + CloudFront

---

## Project Structure

```
mtg-ai-match/
├── index.html      # Page shell — controls, board, log panel
├── app.js          # Game engine, AI brain, Scryfall loader, renderer
├── styles.css      # Dark theme, card rendering, overlays, responsive layout
└── README.md       # This file
```

### `app.js` Architecture

| Section | Description |
|---------|-------------|
| **Scryfall helpers** | Throttled fetch, search, image URL extraction, caching |
| **Card simplifier** | Converts raw Scryfall JSON → engine card objects |
| **Oracle parsers** | Regex-based extraction of spell effects, ETB triggers, attack triggers, keywords |
| **Deck archetypes** | 6 color-based deck definitions with Scryfall search queries |
| **Card DB + Deck pools** | Runtime card catalog populated from Scryfall at boot |
| **Land system** | Basic land loading, 1-per-turn play, mana from battlefield lands |
| **AI evaluation** | `evalCard()` — context-aware card scoring for play decisions |
| **AI targeting** | `chooseTarget()` — smart target selection for spells (lethal check, value trades) |
| **AI combat** | `chooseAttackers()` — aggressive by default with keyword awareness |
| **AI blocking** | `chooseBlockers()` — favorable trades, chump-blocking, threat assessment |
| **Spell resolution** | `resolveSpell()` — handles 15+ spell effect types |
| **Combat damage** | `combatDamage()` — deathtouch, trample, lifelink, first strike |
| **Turn loop** | `takeTurn()` — Main 1 → Combat → Main 2 → End step |
| **Match runner** | `runAutoMatch()` — game loop with overlays, countdown, auto-rematch |
| **Renderer** | `render()` → `renderSeat()` → `drawCardNode()` — full board state display |

---

## Scryfall API Usage

This app makes **~15 API calls** on first load:
- 6 card search queries (one per deck archetype, up to 2 pages each)
- 5 basic land lookups (`named?exact=Plains`, etc.)

Results are cached in memory for the session. Scryfall's rate limit is respected with a 120ms throttle between requests.

**No API key required** — Scryfall's public API is used with standard rate limiting.

---

## Customization

### Add a new deck archetype
Add an entry to `DECK_ARCHETYPES`:
```js
simic: {
  name: 'Simic Ramp',
  color: 'UG',
  aggression: 0.5,
  plan: 'ramp',
  query: 'c>=ug -t:land f:standard'
}
```

### Change deck size
Modify `formatDeckSize()` and `formatCopies()` to adjust card counts per format.

### Adjust AI aggression
Each deck has an `aggression` value (0.0–1.0) that influences attack decisions. Higher = more aggressive.

### Change game speed
Edit the `sleep()` durations in `takeTurn()`:
- `600` — turn start pause
- `500` — combat phase pause
- `300` — between card plays
- `400` — end of turn
- `1100` — card announcement overlay duration

---

## Known Limitations

- **No stack/priority system** — spells resolve immediately, no responses
- **No planeswalkers** — only creatures, instants, sorceries, enchantments, artifacts, and lands
- **No mana colors** — any land pays for any spell (generic mana only)
- **Simplified combat** — one blocker per attacker, no damage assignment order
- **Oracle text parsing is regex-based** — complex or multi-mode cards may not resolve correctly
- **No graveyard recursion** — cards that return from graveyard are not supported
- **No counters** — +1/+1 counters, loyalty counters, etc. are not tracked

---

## Tech Stack

- **Vanilla JavaScript** — no frameworks, no build step, no dependencies
- **Scryfall API** — real MTG card data and artwork
- **CSS custom properties** — dark theme with responsive layout
- **Seeded PRNG** — deterministic games from seed input (Mulberry32)

---

## Credits

- Card data and artwork: [Scryfall](https://scryfall.com/) (unofficial MTG API)
- Game rules inspired by: [Magic: The Gathering](https://magic.wizards.com/)
- Original engine concept inspired by a Sup MTG lobby app

---

## License

MIT — use it however you want. Card artwork is property of Wizards of the Coast / Scryfall.
