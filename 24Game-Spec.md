# Victoria's 24Game: Specification

Snapshot of the game as it exists in `index.html` (as of 2026-11-16, after the players, themes, stickers, daily card, and board-first layout update). Written so another developer or model can rebuild it, extend it, or propose improvements without reading the source.

Live: https://robbbrady.github.io/24Game/ | Repo: https://github.com/robbbrady/24Game | License: MIT

---

## 1. Product summary

A browser puzzle game based on the "24" card game. The player is shown numbers and combines them with + − × ÷, using each number once, to make exactly 24. Built for a child (Victoria) to play on a phone or PC. It runs offline, has no accounts, no server, and no tracking.

Three game modes (Classic, Variables, Negatives), four difficulty levels (Easy, Medium, Hard, Extra Hard), unlimited generated cards, encouragement feedback, and a teacher-style Competition mode. It is used on shared school iPads and personal devices, so it asks who is playing and keeps each player's progress apart. Players pick a theme, collect stickers, share a daily card, and are invited to find more than one way to solve a card.

## 2. Technical constraints

- **One file:** `index.html` with inline CSS and JS. No frameworks, no build step, no external requests, no audio or image files.
- **Hosting:** static, GitHub Pages. Must work when opened from a browser on desktop and mobile Safari/Chrome. (Mobile "file preview" panes that do not run JavaScript are not supported; use the hosted URL.)
- **Persistence:** `localStorage` only, every read and write wrapped in try/catch with in-memory fallback. Keys (all prefixed `g24_`): `players` (list of `{id, name, avatar, theme}`), `current` (last player id), `migrated`, and per player `p<id>_mode`, `p<id>_level`, `p<id>_stats`, `p<id>_vstats`, `p<id>_nstats`, `p<id>_sound`, `p<id>_prog` (`{stickers, daily, modes, levels}`). `prog` also holds `points`; each player record also holds `seen` (last played); `seen` (global) is the time of the last tap. The older un-prefixed `mode`, `level`, `stats`, `vstats`, `nstats`, `sound` are copied into a new player only if that player says the progress is theirs. A guest is held in memory only.
- **Code layout:** pure game logic (solver, generators) lives at top level and is separable from the DOM code, which is inside one `if (typeof document !== 'undefined')` IIFE. Keep this separation: the logic should be testable in Node and portable to other targets (the author has considered an ESP32/C port and a wrapped mobile app).
- **Layout:** single column, max width 520 px, 16 px gutters, no horizontal scroll at phone width. Order top to bottom: title with player chip and menu button; mode tabs and level on one row; a one-line stats strip; a 24-slot collection strip; the board (target badge, tiles, operators, message, steps strip, hint box, four buttons); panels; footer. The whole game fits one phone screen without scrolling. Colors come from CSS variables; the Classic theme follows `prefers-color-scheme`, the other themes set their own palette.

## 3. Core rules and arithmetic

- A card is a list of integers. A move picks two numbers and an operator and replaces both with the result. Repeat until one number is left. **Win = the last value is exactly 24.**
- **All math is exact rational arithmetic** (integer numerator and denominator, reduced by GCD, denominator always positive). No floating point. Cards like 3, 3, 8, 8 (8 ÷ (3 − 8 ÷ 3)) must work.
- Division by zero is rejected with a message; the board is unchanged.
- Display: integers plain, fractions as stacked n over d, negatives with a true minus sign (U+2212). Operators shown as + − × ÷. In steps and expressions, negative operands are wrapped in parentheses, e.g. `4 − (−5)`.
- Order matters for − and ÷: the first number tapped is the left operand.

## 4. Interaction model (Classic and Negatives)

1. Tap a number to select it (tap again to deselect).
2. Tap an operator (tap the same operator again to clear it). An operator tap with no number selected shows "Pick a number first".
3. Tap a second number. The result (dashed border) takes the second tile's place and a copy of the first tile slides into it. Every tile keeps a fixed slot, so nothing else moves.
4. If a number is selected but no operator is, tapping another number moves the selection.
5. When one tile remains: 24 = win; anything else shakes the tile and shows a try-again message with the value reached and an inline **Undo last step** button.
6. Each move is added to a steps strip under the message (for example `7 × 3 = 21`). Tapping a step undoes that step and everything after it.
7. A target badge on the board reads `= 24` and turns into a green `✓ 24` on a win.

## 5. Game modes

### 5.1 Classic
Four positive integers per card, tuned by difficulty (section 6).

### 5.2 Negatives
Four non-zero whole numbers in [−R, R], at least one negative, mixed with positives. Same interaction as Classic.

### 5.3 Variables
Modeled on the "Variables" card product: two number sets share one unknown value.

- A card has two sets ("Left" and "Right") of 3 numbers each, drawn as two wheels, plus one shared unknown `?` from 1 to 9.
- Exactly one value of `?` in 1-9 makes 24 on both sets (each set uses its three numbers plus `?`, each once).
- The two sets are called "sides" in the game (earlier versions said "wheels"; read "wheel" below as "side").
- Flow: the player taps the `?` tile (or the button under the board) and picks a value 1-9 from a dialog with nine large buttons. The button then reads `? = N (tap to change)`. Both wheels then become four-number boards (3 given numbers plus the shared `?`). The player must **prove** the choice by building 24 on each wheel with the normal tap-and-combine mechanic.
- The `?` is a single tile that overlaps both wheels at the bottom center. Each wheel has its own use of it: tap it as an operand on either wheel, once per wheel. It dims once both wheels have used it. It can be tapped first (`? op number`) or second (`number op ?`).
- Before a value is picked, tapping any number opens the picker. Picking a different value resets both sides.
- A finished wheel shows a green "✓ 24" badge. The game is won when both wheels are done. A wheel whose final value is not 24 shows a try-again message naming the wheel.
- Undo restores the last move on whichever wheel it was made; Reset restores both wheels (keeping the chosen `?`).
- Hints: until the correct `?` is chosen, staged one per press (values that work for the left side, then the right, then the answer). After that, the hint ladder in section 8 runs on the first unsolved side.
- The `?` of a new card is never the same as the previous card's `?`.

## 6. Difficulty levels

Difficulty is the number of **distinct** ways a card can reach 24 (fewer = harder). "Distinct" means after canonicalization (section 7.2).

| Level | Classic | Negatives | Variables |
|---|---|---|---|
| Easy | numbers 1-9, 8 or more solutions | range ±5, 8 or more | numbers 1-9, min solutions per wheel 5 or more |
| Medium | 1-9, 3 to 7 solutions | ±9, 3 to 12 | 1-13, 2 to 4 |
| Hard | 1-13, 1 to 2 | ±13, 1 to 2 | 1-13, exactly 1 |
| Extra Hard | 1-25, 1 to 2, at least 2 numbers ≥ 10 | ±25, 1 to 2, at least 2 numbers with absolute value ≥ 10 | 1-25, exactly 1, at least 1 number ≥ 10 in each set |

For Variables, "solutions" means the smaller of the two wheels' distinct-solution counts at the correct `?`.

## 7. Algorithms

### 7.1 Solver
`solveItems(items)` works from any position (the starting numbers or a half-played board with fractions), which is what lets hints follow the player. `solve(nums)` is the same thing from four integers. Recursive brute force over a list of rational items: for every pair, try a+b, a×b, a−b, b−a, a÷b, b÷a (skip division by zero), recurse on the reduced list, record when one item equals 24/1. Each solution keeps a display expression (outer parentheses stripped) and an ordered list of steps (`a op b = r`).

### 7.2 Canonical dedup
Each node carries a canonical string. For + and ×, the two child strings are sorted so commuted forms collapse; − and ÷ keep order. Solutions are stored in a map keyed by the canonical string, and the map size is the difficulty metric. (Associativity variants such as (a+b)+c vs a+(b+c) still count separately; this is accepted noise.)

### 7.3 Classic and Negatives generation
Rejection sampling: draw numbers (uniform), apply the level's extra filters (at least one negative; double-digit count), skip cards already seen this session (key = sorted numbers), solve, accept if the solution count is within the level's range. Up to 5000 tries per card. If nothing is found and the seen-set is non-empty, clear it and retry once (small pools).

### 7.4 Variables generation
Random rejection sampling is too slow, so build a pool once per max value (lazy, shows "Building cards..."):
- Pool = every sorted triple 1..max together with the list of `?` values in 1-9 that make 24 with it; keep triples with 2 or more such values.
- Checking "can these 4 numbers make 24" uses a fast method: memoized sets of values reachable from 1-3 numbers, then for each split of the four numbers, look up the value the other half would need (`needs(u)` returns the six inverses giving 24). This replaced a plain recursive check to make the 1-25 pool affordable.
- Generate: pick `?` (not the previous card's), filter pool triples containing it (and the double-digit rule), pick A, pick B such that the two value lists intersect in exactly `{?}`, require the level's solution-count range on both, randomly swap left/right, shuffle display order within each set.

## 8. Controls

| Control | Behavior |
|---|---|
| Undo | Restore the previous board state (stack of snapshots). Disabled with no history or after the card ends. |
| Reset | Restore the original numbers (Variables: both wheels, keeps chosen `?`). Does not reset the timer. |
| Hint | Marks the card as helped. A ladder worked out by solving from the **current board**: (1) what to aim for, e.g. "Try to make 2 and 12. Then 2 × 12 = 24."; (2) the same plus two glowing tiles to start with; (3) the next step in text; (4) every step from here. Once at level 3 it gives one step per move; a second press on the same board gives the rest. If the board cannot reach 24 it says so and suggests undoing. The solver prefers a solution with whole, positive intermediate values. |
| Skip / Next card | One button. While a card is in play it reads Skip, resets the streak to 0, and deals a new card. After a win (or while looking for another way) it reads Next card and has no penalty. There is no separate Show answer; hint level 4 covers it. |
| Steps strip | Each move as a chip; tapping one undoes that step and everything after it. |
| Menu | Header button: Daily card, Sticker book, Theme, Sound, Competition, Share, Switch player. |
| Player chip | Header button showing the current player; opens "Who's playing?". |
| Level select | Easy / Medium / Hard / Extra Hard. Saves the choice, clears the seen-set, deals a new card. |
| Mode toggle | Classic / Variables / Negatives segmented control. Same effects as level change. |
| Sound | Menu item, on/off, saved per player. |
| Share | Menu item; opens the share dialog (section 12). |

Timer: starts when a card is dealt, shows m:ss, ticks twice a second, stops on win or Show answer.

## 9. Feedback

- **Win:** random message plus time, e.g. "Good job! Solved in 0:14" (Variables adds " ? = 4."). A short pop animation and a soft chime. A win after any hint or Show answer uses a gentler message and the suffix " (with help)".
- **Messages (kid-friendly, never the same twice in a row):**
  - Win: Good job! / Nice work! / You got it! / Great thinking! / Excellent! / Way to go! / Math whiz! / Spot on!
  - Helped: Nice finish! / You figured it out! / Good job sticking with it!
  - Try again: Not quite, try again! / So close! Give it another go. / Almost there, keep trying! / Good effort. Try a different way! / Hang in there. Undo and try again. (followed by "That makes X." or, in Variables, "The left wheel makes X, not 24.")
  - Variables, one wheel done: "Left wheel done, nice! Now the other wheel."
- **Streak:** counts clean wins (no hint, no answer) per mode. Streak of 3 and every multiple of 5 shows a special message ("3 in a row, you're on fire!", "5 in a row! Amazing!", "10 in a row! Unstoppable!", otherwise "N in a row! Incredible!") with confetti and a fanfare.
- **New best time:** a clean win faster than an existing best for that mode and level adds "New best time!" plus confetti and fanfare (the very first win does not count).
- **Streak challenge prompt:** after a clean win with streak a multiple of 5, a panel asks "N in a row! Ready for a challenge?" with buttons: Harder level (next level up, hidden at Extra Hard), Try each other mode, Keep going. It disappears when the next card is dealt. Not shown in Competition mode.
- **Sound (Web Audio, synthesized, no files):** win = two sine notes (659 Hz, 880 Hz); big = four triangle notes (523, 659, 784, 1047 Hz) plus a held 1319 Hz; try-again = a low tone sliding 300 to 200 Hz. Sound needs a user tap (always true here). On iPhone the ring/silent switch mutes it.
- **Win effect:** canvas overlay, about 2.2 s, pointer-events off, skipped when the device prefers reduced motion. The theme sets its look: confetti (Classic), stars (Space), rising bubbles (Ocean), falling leaves (Jungle), chalk dust (Chalkboard), pixels (Arcade), embers (Volcano). It plays on streak milestones, new best times, new stickers, and a full world. ("Confetti" above means this effect.)
- **Name:** about four wins in ten, the message uses the player's name ("Nice work, Sam!").

## 10. Competition mode

Teacher-style "prove it yourself" mode, toggled by a button above the mode toggle.

- **Entering:** Menu, Start Competition. A red frame is drawn around the screen (it works with every theme) and a "COMPETITION: NO HELP" bar with an Exit button appears. Entering always deals a fresh card.
- **Disabled while active:** Undo, step-strip undo, Hint, Skip, the level select, the mode toggle, Daily card, and Switch player. **Reset is allowed** but each one counts as a new attempt; a wrong final number offers a "Try again" button that does the same.
- **Exiting:** the Exit button or the menu; if a card is in progress it asks "Exit competition? This attempt will be lost." then deals a normal card.
- **On a win:** the usual encouragement plays, and a green summary panel ("Competition: solved without help") shows the player's name, mode, level, the card (Variables: both sets and the `?`), solve time, attempt number, date and time, and every step the player made (Variables grouped per side). "Copy proof" copies it as text; "New competition card" deals the next card.
- Steps shown are the moves actually made, recorded per board as they happen.

## 11. Stats and persistence

- Stored per player. Per mode (Classic, Variables, Negatives, stored separately): solved count, current streak, best time per level (easy, medium, hard, xhard). The stats bar shows Time, Solved, Streak, and Best for the current mode and level.
- A solved card always increments Solved. Helped solves reset the streak and never set a best time. Skip and Show answer reset the streak.
- Stats are device and browser local; there is no sync.
- Missing keys (for example `xhard` in stats saved before Extra Hard existed) must be tolerated.

## 12. Share and QR

- "Share this game" in the menu opens a modal: heading "Scan to play", a QR code on a white tile (always white, even in dark mode), the URL, and Copy link / Close buttons. Closes by button, backdrop tap, or Esc.
- The QR is an inline SVG built from the module matrix (29 modules, QR version 3, error correction M, 4-module quiet zone) encoding `https://robbbrady.github.io/24Game/`. It was decode-tested.
- On mobile browsers with the Web Share API, Copy link becomes "Share link" and opens the system share sheet; elsewhere it copies to the clipboard and shows "Copied!".
- Repo also contains `qr.png` and a print-ready `qr.svg` of the same code.

## 12a. Players

- **One rule for shared and personal devices** (the game cannot tell which it is on). `g24_seen` holds the time of the last tap. On load: no remembered player, show **Who's playing?**; remembered player and last tap under 10 minutes ago (a reload), go straight in; otherwise show **Welcome back** with "Continue as NAME" and "I'm someone else". The same sheet appears after 10 minutes with no tap or key press, checked every 15 seconds and when the page becomes visible again. It cannot be dismissed. Continue keeps the card and does not count the time away; "I'm someone else" goes to Who's playing and starts a fresh session.
- **Who's playing?**: a tile per player (avatar, name, points, last played), most recent first, plus **Add player**, **Just play (nothing saved)**, **Remove a player** (with a confirm; deletes that player's keys), and, when any player has not played for 30 days, **Tidy up** (lists them, confirm, never automatic).
- **Bringing progress over is asked, never automatic.** When a new player is saved: if they were just playing as a guest and earned anything, ask "Keep what you just earned?"; otherwise, if the device has un-prefixed stats from before players existed and nobody has claimed them, ask "Is this saved progress yours?". "No, start fresh" leaves the old stats for a later player to claim.
- Add player: a first name or nickname (max 12 characters) and one of 12 avatar emoji. The screen says "Stays on this device. Nothing is sent anywhere." Names are only ever written with `textContent` or escaped.
- Right after a new player is saved, the theme picker opens as "Pick your world, NAME" with a "Start playing" button.
- The player chip in the header shows the avatar, the name, and the newest stickers by time earned: 1 below 430 px wide, 3 from 430 px, 5 from 600 px. It updates when a sticker is earned.
- Switching player starts a fresh session: new card, session counters cleared, that player's mode, level, theme, sound, stats, and stickers loaded.

## 12f. Points

One running total per player (`prog.points`), shown in the stats strip and on player tiles. Speed never earns points.

- Base by level: Easy 10, Medium 20, Hard 30, Extra Hard 50. Mode multiplier: Classic ×1, Negatives ×1.2, Variables ×1.5 (rounded).
- Clean solve (no hint): base × streak multiplier, where the multiplier is 1.0 on the first card of a streak and rises 0.1 per card to a cap of 2.0 at 11 in a row.
- Solve with a hint: half the base, no multiplier (and the streak resets, as before).
- Each extra way found on a card: +5. First solve of the daily card: +20.
- The win message shows the gain, e.g. "+22 points (streak ×1.1)".
- Cards solved before points existed are credited once at a flat 15 each.

## 12b. Themes and the living world

- Themes: Classic, Space, Ocean, Jungle, Chalkboard, Arcade (unlocks at 10 stickers), Volcano (unlocks at 20). Chosen per player in Menu, Theme. Themes are topics, not color preferences.
- A theme is a `:root[data-theme]` block of CSS variables, a `.scene` background drawn with CSS gradients (Jungle adds an inline SVG data URI), an optional tile treatment, a set of six "things" emoji, and a win effect.
- **Living world:** each clean solve (no hint) in a session adds one of the theme's things to the 24-slot strip under the stats and at a random spot in the background. At 24 the game celebrates, awards a sticker, and empties the world to start again. Switching player clears it.

- **Difficulty changes the screen** (`data-level` on the root element): a band of the theme's level color rises from the bottom of the background (none on Easy, 40% on Medium, 70% on Hard, full height on Extra Hard); the board gets an accent border on Hard and a thicker accent edge on Extra Hard; and the top left of the board shows dots and the level name, e.g. "●●○○ Medium" (replaced by the daily, find-another-way, or attempt label when one applies).

## 12c. Stickers

30 emoji stickers per player, checked on every win and on every new way found. The book (Menu, Sticker book) shows earned ones in color and the rest in gray with a clue; three are secret until earned. A new sticker shows a toast and the win effect.

- **Getting started:** First 24, Hat Trick (3 streak), High Five (5), Perfect Ten (10), Two Dozen (24 solved across modes), Century (100).
- **Math skills (clean wins only):** Fraction Hero (a fraction on the way), Below Zero (negative on the way, Classic), Division Star (used ÷), Mix Master (three different operators), One Trick (one operator for every step), No Times (no ×), Big Numbers (Extra Hard), Detective (Variables, right `?` on the first pick).
- **Grit and speed:** Lightning (under 10 s, clean), Slow and Steady (3 minutes or more, clean), Comeback (hit a wrong final number, then solved it clean), No Help Needed (10 clean solves in one session).
- **Explorer:** Another Way (2 ways on one card), Way Finder (5 ways), Every Way (all ways on a card with 3 or more), Mode Explorer (all three modes), Level Climber (all four levels), Full World (24 things collected).
- **Habits:** Daily Player, School Week (daily card five times in a row; gaps of up to 3 days allowed so weekends do not break it), Champion (competition win).
- **Secret:** Double Dozen (last step 12 + 12), The Famous One (3, 3, 8, 8), Right on Time (solved in exactly 0:24).

## 12d. Daily card

- `dailyCard(date)` runs the Classic Medium generator with a random source seeded from the local date (`YYYY-MM-DD`), so every device deals the same card that day with no server.
- Menu, Daily card shows today's board: every player on this device who has solved it, with time (clean solves listed first, helped ones marked), then Play today's card. Playing it switches to Classic Medium for that one card and returns to the player's own mode and level afterwards.
- The first solve of the day is the recorded one; later plays are practice. Share result gives a short text with the date and time.

## 12e. Find another way

- Classic and Negatives only. Every board tile carries the same canonical key the solver uses (section 7.2), so a finished board maps to one of the card's distinct solutions.
- After a win a panel reads "You found 1 of N ways. Can you find another?" and lists the ways found. Find another way resets the board on the same card. A repeat is turned away ("You already found that way"); a new one is added to the list. Extra ways do not change Solved, streak, or best time. Hints in this mode skip ways already found.

## 13. Repo files

`index.html` (the game), `README.md`, `LICENSE` (MIT, Robb Brady 2026), `qr.png`, `qr.svg`, `.gitignore` (excludes the older `SPEC.md` and a screenshot), `24Game-Spec.md` (this file), `.claude/launch.json` (local preview server setting, not part of the game).

---

## 14. Known gaps and open questions (for the next model to weigh in on)

1. **Competition dead end:** resolved. Reset is allowed and counted, and the proof shows the attempt number. A player who is truly stuck still has to exit competition to get a different card.
2. **Competition and stats:** the code comment says competition is "never saved", but a competition win currently goes through the normal win path and does increment Solved, streak, and best time for that mode and level. Decide whether competition results should count toward normal stats, be kept separate, or not be recorded.
3. **Proof:** can now be copied as text and names the player. Still no print stylesheet or tamper-resistant code.
4. **Cheating surface:** hints are disabled in the UI only. Anyone can open the console, and state is in plain JavaScript and `localStorage`.
5. **Extra Hard Variables performance:** the 1-25 pool is built on first use on the player's device and has not been timed on a low-end phone. Consider precomputing, a web worker, or caching the pool.
6. **Difficulty metric:** counting distinct solutions ignores how hard the solution is to find (for example, solutions needing fractions or negatives mid-way). A weighted difficulty could feel fairer.
7. **No automated tests.** The logic is portable and testable in Node; a small test suite (solver counts, generator ranges, fraction math, Variables uniqueness) would protect future changes.
8. **No tutorial or first-run help:** the Variables mode in particular is unexplained beyond the footer line and the picker dialog.
12. **Shared-device limits:** players are a convenience, not accounts. Anyone can pick another player's tile, and clearing the browser's site data removes everyone's progress. School iPads that wipe Safari data between sessions will not keep players.
13. **Guest progress is not saved** (by design). On a personal device a returning guest starts from zero each visit unless they add a name.
14. **Ways count includes order variants** (see 7.2), so "1 of 34 ways" overstates the truly different ideas on easy cards.
9. **Accessibility:** tiles have aria-labels, but there is no keyboard play, no screen-reader announcement design for the board, and color alone marks selected tiles and success.
10. **Not installable:** no PWA manifest or service worker, so "Add to Home Screen" works but offline support relies on browser cache.
11. **Zero is excluded** from Negatives cards. Including it could be an optional level.

## 15. Suggested brief for the receiving model

"Read this spec. (A) Rebuild the game as a single HTML file that matches it, or (B) review it and propose the most valuable improvements, starting with section 14. State any assumptions. Keep it one file, vanilla JavaScript, exact fractions, no external requests."
