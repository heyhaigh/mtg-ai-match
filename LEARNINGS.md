# Learnings & Pitfalls

Hard-won lessons from building the AI MTG Match simulator. Reference this before making changes to avoid re-introducing solved bugs.

---

## 🔴 Critical Bugs We Hit

### 1. Summoning Sickness Never Cleared
**Symptom**: Creatures never attacked — games went 30+ turns with full boards staring at each other.
**Root cause**: `untapAll()` reset `tapped` but never cleared `summoningSick`. Every creature permanently thought it just entered the battlefield.
**Fix**: Clear `summoningSick = false` at the start of the controller's turn in `untapAll()`.
**Rule**: Any game state flag that's set on entry MUST have a corresponding clear path.

### 2. CSS `background` Shorthand Overrides Inline `backgroundImage`
**Symptom**: Card art from Scryfall never appeared — cards showed only colored gradients.
**Root cause**: `.card.creature { background: ... }` is a CSS shorthand that resets ALL background properties, including `background-image` set by JavaScript inline styles.
**Fix**: Use `:not(.has-art)` selectors so gradient backgrounds only apply to cards without art.
**Rule**: Never use CSS `background` shorthand on elements that might have JS-set `backgroundImage`. Use `background-color` instead, or scope with `:not()`.

### 3. `appState.running = true` Before Calling `runAutoMatch()`
**Symptom**: Start button did nothing — match never began.
**Root cause**: The click handler set `running = true`, then `runAutoMatch()` checked `if (running) return;` and immediately exited.
**Fix**: Let `runAutoMatch()` set its own running state internally.
**Rule**: Don't pre-set guard flags that the callee also checks. Let the function own its own state transitions.

### 4. Removed UI Element Still Referenced in Code
**Symptom**: Victory overlay and auto-rematch countdown silently broke — no error visible.
**Root cause**: Speed slider (`#speedInput`) was removed from HTML but JS still called `$('#speedInput').value`, returning `null`. `Number(null)` → `0`, which caused issues downstream.
**Fix**: Remove all JS references when removing HTML elements. Added global error catchers to surface these.
**Rule**: When removing a UI element, grep the entire codebase for its ID/selector.

### 5. Deck Card Arrays Contained Tuples Instead of IDs
**Symptom**: `Error: Unknown card: goblin,7` — game crashed on boot.
**Root cause**: `DECKS.red.cards` was `[['goblin', 7], ...]` (recipe format), but `createMatch` passed it directly to `makeInstance()` which expected string IDs.
**Fix**: Added `expandRecipe()` and `normalizeCardId()` to flatten recipe arrays into ID lists.
**Rule**: Always validate the shape of data flowing between layers — deck recipes ≠ card ID arrays.

---

## 🟡 Design Mistakes

### 6. Auto-Mana Hides a Core Game Mechanic
**Symptom**: User noticed lands were completely missing from the game experience.
**Root cause**: Original design used invisible auto-mana (increment +1/turn) to simplify gameplay, but it made the game feel nothing like MTG.
**Fix**: Added real basic lands to decks, land-playing phase in turns, mana from battlefield lands.
**Lesson**: Don't abstract away mechanics that define the identity of the game you're simulating.

### 7. 30-Card Decks End Games via Deck-Out
**Symptom**: Games ended in 1-2 turns — nobody took damage, someone just ran out of cards.
**Root cause**: 30-card deck, 7-card opening hand, draw spells accelerating — library empty by turn 5. `drawCard()` set `life = 0` on empty library.
**Fix**: 60-card decks (Standard) / 100-card (Commander). Deck-out no longer kills — you just stop drawing.
**Lesson**: Deck size must match the game length you want. Do the math: `deckSize - handSize = maximum turns`.

### 8. Conservative AI = Boring Games
**Symptom**: Both AIs built huge boards but never attacked.
**Root cause**: Attack logic required specific keyword conditions (flying, trample, etc.) or board advantage. Vanilla creatures with no keywords never qualified.
**Fix**: Flipped the logic — **attack by default**, only hold back if a tiny creature would die for nothing while heavily outnumbered.
**Lesson**: In game AI, the default action should be the exciting one. Hold-back should be the exception, not the rule.

### 9. Scryfall Keywords Array Misses Granted Keywords
**Symptom**: Cards with "haste" in oracle text but not in `keywords[]` didn't get haste behavior.
**Root cause**: Scryfall's `keywords` array only lists innate keywords, not ones granted by abilities ("creatures you control have haste").
**Fix**: Parse oracle text for all 15 core MTG keywords and add them to the card's keyword list. Also check oracle text at runtime in `hasKW()`.
**Lesson**: Don't trust a single data source for derived properties. Cross-reference multiple fields.

---

## 🟢 Architecture Lessons

### 10. File Permissions in Sandboxed Environments
**Problem**: Files created by one user (root/agent) couldn't be edited by another (user).
**Workaround**: Edit in `/home/user/`, then `sudo cp` to `/code/`. Always verify with `curl` that the served file matches your edit.
**Lesson**: Work in a writable directory, deploy to the served directory as a separate step.

### 11. Browser Caching Masks Code Changes
**Problem**: User refreshed but saw old behavior because browser cached old JS/CSS.
**Workaround**: Hard refresh (`Ctrl+Shift+R`), or add cache-busting query params.
**Lesson**: During development, tell users to hard-refresh. Or serve with `Cache-Control: no-cache`.

### 12. Headless Browser Screenshots Don't Wait for Async
**Problem**: `--virtual-time-budget` doesn't wait for real network requests (Scryfall API calls).
**Workaround**: Use CDP (Chrome DevTools Protocol) with explicit waits, or test with a real browser.
**Lesson**: Headless screenshots are only useful for synchronous page states. Async-loaded content needs explicit wait logic.

### 13. Global Error Catchers Are Essential
**Problem**: Errors in async code (promises, event handlers) silently failed — UI just stopped working with no indication.
**Fix**: Added `window.onerror` and `window.onunhandledrejection` handlers that write errors to the status bar.
**Lesson**: Always add visible error surfaces during development. Silent failures are the hardest bugs.

### 14. Single Responsibility for State Transitions
**Problem**: Start button, reset button, runAutoMatch, and the auto-rematch countdown all manipulated `running`, `match`, and `runId` — leading to race conditions.
**Fix**: Each function owns specific state transitions. Start button creates the match, `runAutoMatch` owns `running`, reset increments `runId` to cancel loops.
**Lesson**: Map out which function owns which state field. Don't let multiple code paths fight over the same flag.

---

## 🔵 Scryfall API Notes

- **Rate limit**: 120ms between requests minimum (we throttle client-side)
- **No API key needed**: public API
- **Basic lands**: Use `named?exact=Mountain` — always returns a valid card with `image_uris`
- **Card search**: `order=edhrec` gives popular/playable cards first — great for deck building
- **Keywords array**: Incomplete for granted keywords — always cross-reference oracle text
- **Art crop vs normal**: `art_crop` is just the illustration. For lands, use `normal` (full card frame) since the color border makes them recognizable
- **Dual-faced cards**: Check `card_faces[0].image_uris` as fallback when `image_uris` is null
- **~15 API calls per session**: 6 searches (2 pages each max) + 5 land lookups. Cache in memory.

---

## 📋 Testing Checklist

Before deploying changes, verify:

- [ ] `node --check app.js` passes (syntax)
- [ ] Page loads without errors in status bar
- [ ] Scryfall cards load (status shows "X real cards loaded")
- [ ] Deck dropdowns populate
- [ ] Start match launches the game loop
- [ ] Creatures attack within first 3-4 turns
- [ ] Life totals decrease from combat
- [ ] Card art appears on battlefield and hand cards
- [ ] Land cards appear with correct art (Mountain looks like a Mountain)
- [ ] Card play overlay shows art + player badge
- [ ] Victory overlay appears when a player hits 0 HP
- [ ] 10-second countdown appears after victory
- [ ] Next game starts with randomized decks
- [ ] Format toggle rebuilds decks correctly
- [ ] No `#speedInput` or other dead element references remain
- [ ] Global error handlers catch and display any runtime errors
