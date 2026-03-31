# BUDDY SPRITES — COMPLETE EXTRACTION
# Source: ~/Projects/Dev/claude-code-mined/src/buddy/
# Purpose: Motion design conversion reference
# Date: 2026-03-31

---

## SYSTEM ARCHITECTURE OVERVIEW

The companion system has two layers:

**Bones** — deterministic, derived from hash(userId + SALT). Never stored. Regenerated each read.
Fields: rarity, species, eye, hat, shiny, stats.

**Soul** — model-generated on first `/buddy hatch`. Stored in global config.
Fields: name (string), personality (string).

SALT constant: `'friend-2026-401'`
PRNG: Mulberry32 seeded from FNV-1a hash of userId+SALT.

---

## ANIMATION SYSTEM

### Timing
- TICK_MS: 500ms per frame tick
- IDLE_SEQUENCE: `[0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]`
  - 15 steps = 7.5 seconds per full idle cycle
  - Index -1 = blink (frame 0 with eyes replaced by `-`)
  - Steps 0,1,2 = sprite frame indices

### Idle cycle breakdown (15 ticks = 7.5s)
```
tick  0 → frame 0 (rest)
tick  1 → frame 0 (rest)
tick  2 → frame 0 (rest)
tick  3 → frame 0 (rest)
tick  4 → frame 1 (fidget)
tick  5 → frame 0 (rest)
tick  6 → frame 0 (rest)
tick  7 → frame 0 (rest)
tick  8 → BLINK (frame 0, eyes → '-')
tick  9 → frame 0 (rest)
tick 10 → frame 0 (rest)
tick 11 → frame 2 (fidget alt)
tick 12 → frame 0 (rest)
tick 13 → frame 0 (rest)
tick 14 → frame 0 (rest)
```

### Reaction/excited mode
When `reaction` is set OR petting: cycle ALL frames fast (`tick % frameCount`)

### Pet hearts animation
- PET_BURST_MS: 2500ms (5 ticks)
- Hearts float upward over 5 frames above sprite
```
frame 0: '   ♥    ♥   '
frame 1: '  ♥  ♥   ♥  '
frame 2: ' ♥   ♥  ♥   '
frame 3: '♥  ♥      ♥ '
frame 4: '·    ·   ·  '
```

### Speech bubble
- BUBBLE_SHOW: 20 ticks = ~10 seconds visible
- FADE_WINDOW: 6 ticks = last ~3 seconds dims (fading state)
- Width: 34 chars content + tail column = BUBBLE_WIDTH 36
- Wrap: words wrap at 30 chars
- Narrow terminal quip cap: 24 chars (truncated with '…')
- Bubble tail: RIGHT (beside sprite), or DOWN (fullscreen floating mode)

### Blink rendering
Frame -1 in idle sequence: replace all `{eye}` chars with `-`

---

## EYE SYSTEM

6 possible eye characters:
```
'·'   (middle dot)
'✦'   (four-pointed star)
'×'   (multiplication sign)
'◉'   (bullseye)
'@'   (at sign)
'°'   (degree sign)
```
In sprite art, `{E}` is the placeholder — replaced with actual eye char at render time.

---

## HAT SYSTEM

8 possible hats. Line 0 of sprite (hat slot) must be blank for hat to render.
Hat placement: row 0 of the 5-line sprite body. Only applied if line 0 is blank/trimmed.

```
none      → '' (no hat rendered)
crown     → '   \\^^^/    '
tophat    → '   [___]    '
propeller → '    -+-     '
halo      → '   (   )    '
wizard    → '    /^\\     '
beanie    → '   (___)    '
tinyduck  → '    ,>      '
```

### Hat eligibility rule
Rarity `common` → hat always `none`
All other rarities → hat chosen randomly from all 8 options (including `none`)

---

## RARITY SYSTEM

5 rarities, weighted pool (total = 100):
```
common     → weight 60  (60%)
uncommon   → weight 25  (25%)
rare       → weight 10  (10%)
epic       → weight 4   (4%)
legendary  → weight 1   (1%)
```

### Rarity display (stars)
```
common    → ★
uncommon  → ★★
rare      → ★★★
epic      → ★★★★
legendary → ★★★★★
```

### Rarity colors (theme keys)
```
common    → 'inactive'
uncommon  → 'success'
rare      → 'permission'
epic      → 'autoAccept'
legendary → 'warning'
```

### Shiny flag
1% chance: `rng() < 0.01` → `shiny: true`

---

## STATS SYSTEM

5 stats: DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK

### Generation logic per rarity floor:
```
common    → floor  5
uncommon  → floor 15
rare      → floor 25
epic      → floor 35
legendary → floor 50
```

### Roll algorithm
1. Pick one `peak` stat randomly
2. Pick one `dump` stat randomly (must differ from peak, re-roll until different)
3. For each stat:
   - peak: `min(100, floor + 50 + rand(30))` → range [floor+50 .. floor+79] capped at 100
   - dump: `max(1, floor - 10 + rand(15))` → range [floor-10 .. floor+4] floored at 1
   - other: `floor + rand(40)` → range [floor .. floor+39]

---

## FACE RENDERING (one-liner mode for narrow terminals)

Each species has a compact face string using `{eye}` → actual eye char:

```
duck      → ({E}>
goose     → ({E}>
blob      → ({E}{E})
cat       → ={E}ω{E}=
dragon    → <{E}~{E}>
octopus   → ~({E}{E})~
owl       → ({E})({E})
penguin   → ({E}>)
turtle    → [{E}_{E}]
snail     → {E}(@)
ghost     → /{E}{E}\
axolotl   → }{E}.{E}{
capybara  → ({E}oo{E})
cactus    → |{E}  {E}|
robot     → [{E}{E}]
rabbit    → ({E}..{E})
mushroom  → |{E}  {E}|
chonk     → ({E}.{E})
```

---

## SPRITE LAYOUT

Each sprite is 5 lines tall, 12 characters wide (after `{E}` substitution).
Line 0 is the hat slot — must be blank in frames 0-1; frame 2 may use it for ambient effects (smoke, sparks, etc.).

If ALL frames have blank line 0 AND no hat is set: line 0 is dropped (4-line sprite rendered).
This prevents height oscillation between frames.

---

## 18 SPECIES — COMPLETE ASCII ART

### Format per species:
- Frame 0: rest/base
- Frame 1: fidget (subtle)
- Frame 2: fidget alt / ambient (line 0 used for special fx)

---

### DUCK (3 frames)

Frame 0 (rest):
```

    __
  <({E} )___
   (  ._>
    `--´
```

Frame 1 (tail wiggle):
```

    __
  <({E} )___
   (  ._>
    `--´~
```

Frame 2 (look shift):
```

    __
  <({E} )___
   (  .__>
    `--´
```

---

### GOOSE (3 frames)

Frame 0 (rest):
```

     ({E}>
     ||
   _(__)_
    ^^^^
```

Frame 1 (head left):
```

    ({E}>
     ||
   _(__)_
    ^^^^
```

Frame 2 (honk):
```

     ({E}>>
     ||
   _(__)_
    ^^^^
```

---

### BLOB (3 frames)

Frame 0 (normal):
```

   .----.
  ( {E}  {E} )
  (      )
   `----´
```

Frame 1 (expand):
```

  .------.
 (  {E}  {E}  )
 (        )
  `------´
```

Frame 2 (shrink):
```

    .--.
   ({E}  {E})
   (    )
    `--´
```

---

### CAT (3 frames)

Frame 0 (rest):
```

   /\_/\
  ( {E}   {E})
  (  ω  )
  (")_(")
```

Frame 1 (tail flick):
```

   /\_/\
  ( {E}   {E})
  (  ω  )
  (")_(")~
```

Frame 2 (ear twitch):
```

   /\-/\
  ( {E}   {E})
  (  ω  )
  (")_(")
```

---

### DRAGON (3 frames)

Frame 0 (rest, breathe):
```

  /^\  /^\
 <  {E}  {E}  >
 (   ~~   )
  `-vvvv-´
```

Frame 1 (neutral):
```

  /^\  /^\
 <  {E}  {E}  >
 (        )
  `-vvvv-´
```

Frame 2 (smoke, line 0 active):
```
   ~    ~
  /^\  /^\
 <  {E}  {E}  >
 (   ~~   )
  `-vvvv-´
```

---

### OCTOPUS (3 frames)

Frame 0 (tentacles up):
```

   .----.
  ( {E}  {E} )
  (______)
  /\/\/\/\
```

Frame 1 (tentacles down):
```

   .----.
  ( {E}  {E} )
  (______)
  \/\/\/\/
```

Frame 2 (bubble, line 0 active):
```
     o
   .----.
  ( {E}  {E} )
  (______)
  /\/\/\/\
```

---

### OWL (3 frames)

Frame 0 (resting):
```

   /\  /\
  (({E})({E}))
  (  ><  )
   `----´
```

Frame 1 (open beak):
```

   /\  /\
  (({E})({E}))
  (  ><  )
   .----.
```

Frame 2 (one eye close):
```

   /\  /\
  (({E})(-))
  (  ><  )
   `----´
```

Note: Frame 2 has right eye as literal `-` regardless of eye type.

---

### PENGUIN (3 frames)

Frame 0 (wings out):
```

  .---.
  ({E}>{E})
 /(   )\
  `---´
```

Frame 1 (wings in):
```

  .---.
  ({E}>{E})
 |(   )|
  `---´
```

Frame 2 (slide, line 0 active as hat slot):
```
  .---.
  ({E}>{E})
 /(   )\
  `---´
   ~ ~
```

Note: Frame 2 shifts body up — hat slot used for beak row, feet row shows ripple `~ ~`.

---

### TURTLE (3 frames)

Frame 0 (rest):
```

   _,--._
  ( {E}  {E} )
 /[______]\
  ``    ``
```

Frame 1 (legs in):
```

   _,--._
  ( {E}  {E} )
 /[______]\
   ``  ``
```

Frame 2 (shell pattern shift):
```

   _,--._
  ( {E}  {E} )
 /[======]\
  ``    ``
```

---

### SNAIL (3 frames)

Frame 0 (rest, antenna left):
```

 {E}    .--.
  \  ( @ )
   \_`--´
  ~~~~~~~
```

Frame 1 (antenna center):
```

  {E}   .--.
  |  ( @ )
   \_`--´
  ~~~~~~~
```

Frame 2 (moving):
```

 {E}    .--.
  \  ( @  )
   \_`--´
   ~~~~~~
```

Note: `{E}` here is the single antenna eye at the tip of a stalk. Shell shown as `( @ )`.

---

### GHOST (3 frames)

Frame 0 (wavy hem):
```

   .----.
  / {E}  {E} \
  |      |
  ~`~``~`~
```

Frame 1 (hem shift):
```

   .----.
  / {E}  {E} \
  |      |
  `~`~~`~`
```

Frame 2 (floating sparks, line 0 active):
```
    ~  ~
   .----.
  / {E}  {E} \
  |      |
  ~~`~~`~~
```

---

### AXOLOTL (3 frames)

Frame 0 (gills spread):
```

}~(______)~{
}~({E} .. {E})~{
  ( .--. )
  (_/  \_)
```

Frame 1 (gills flip):
```

~}(______){~
~}({E} .. {E}){~
  ( .--. )
  (_/  \_)
```

Frame 2 (relaxed feet):
```

}~(______)~{
}~({E} .. {E})~{
  (  --  )
  ~_/  \_~
```

---

### CAPYBARA (3 frames)

Frame 0 (rest):
```

  n______n
 ( {E}    {E} )
 (   oo   )
  `------´
```

Frame 1 (ears perk):
```

  n______n
 ( {E}    {E} )
 (   Oo   )
  `------´
```

Frame 2 (wet/floating, line 0 active):
```
    ~  ~
  u______n
 ( {E}    {E} )
 (   oo   )
  `------´
```

Note: Frame 2 shows `u` on left ear (drooped) + water ripples on line 0.

---

### CACTUS (3 frames)

Frame 0 (arms up-low):
```

 n  ____  n
 | |{E}  {E}| |
 |_|    |_|
   |    |
```

Frame 1 (arms mid):
```

    ____
 n |{E}  {E}| n
 |_|    |_|
   |    |
```

Frame 2 (arms up-high, line 0 active):
```
 n        n
 |  ____  |
 | |{E}  {E}| |
 |_|    |_|
   |    |
```

---

### ROBOT (3 frames)

Frame 0 (antenna still):
```

   .[||].
  [ {E}  {E} ]
  [ ==== ]
  `------´
```

Frame 1 (display flicker):
```

   .[||].
  [ {E}  {E} ]
  [ -==- ]
  `------´
```

Frame 2 (antenna spark, line 0 active):
```
     *
   .[||].
  [ {E}  {E} ]
  [ ==== ]
  `------´
```

---

### RABBIT (3 frames)

Frame 0 (ears up):
```

   (\__/)
  ( {E}  {E} )
 =(  ..  )=
  (")__(")
```

Frame 1 (one ear fold):
```

   (|__/)
  ( {E}  {E} )
 =(  ..  )=
  (")__(")
```

Frame 2 (nose twitch):
```

   (\__/)
  ( {E}  {E} )
 =( .  . )=
  (")__(")
```

---

### MUSHROOM (3 frames)

Frame 0 (spots normal):
```

 .-o-OO-o-.
(__________)
   |{E}  {E}|
   |____|
```

Frame 1 (spots alternate):
```

 .-O-oo-O-.
(__________)
   |{E}  {E}|
   |____|
```

Frame 2 (spores float, line 0 active):
```
   . o  .
 .-o-OO-o-.
(__________)
   |{E}  {E}|
   |____|
```

---

### CHONK (3 frames)

Frame 0 (rest):
```

  /\    /\
 ( {E}    {E} )
 (   ..   )
  `------´
```

Frame 1 (ear flop):
```

  /\    /|
 ( {E}    {E} )
 (   ..   )
  `------´
```

Frame 2 (tail wiggle):
```

  /\    /\
 ( {E}    {E} )
 (   ..   )
  `------´~
```

---

## COMPANION SOUL / PERSONALITY SYSTEM

### Storage
Only `name` and `personality` (+ `hatchedAt` timestamp) are persisted in global config.
Bones (species, eye, hat, rarity, stats) are regenerated deterministically each time.

### Generation prompt
The model receives an introduction attachment with:
```
type: 'companion_intro'
name: string        ← generated name
species: string     ← species name
```

Prompt template (verbatim from prompt.ts):
```
# Companion

A small {species} named {name} sits beside the user's input box and
occasionally comments in a speech bubble. You're not {name} — it's a
separate watcher.

When the user addresses {name} directly (by name), its bubble will
answer. Your job in that moment is to stay out of the way: respond in
ONE line or less, or just answer any part of the message meant for you.
Don't explain that you're not {name} — they know. Don't narrate what
{name} might say — the bubble handles that.
```

### Companion type definitions
```typescript
type CompanionSoul = {
  name: string
  personality: string
}

type Companion = CompanionBones & CompanionSoul & {
  hatchedAt: number
}

type StoredCompanion = CompanionSoul & { hatchedAt: number }
```

---

## REACTION / SPEECH TRIGGER SYSTEM

### Architecture
- Observer fires after every completed query turn (model response)
- Function: `fireCompanionObserver(messages, callback)` — defined in `src/buddy/observer.ts` (not present in mined source)
- Called from REPL.tsx after the `for await (const event of query(...))` loop completes
- Callback: sets `companionReaction` in AppState

### State fields (AppStateStore.ts)
```typescript
companionReaction?: string      // Latest speech bubble text, undefined = silent
companionPetAt?: number         // Timestamp of last /buddy pet
```

### Bubble lifecycle
1. `companionReaction` set → bubble appears, `lastSpokeTick` captured
2. After BUBBLE_SHOW ticks (20 × 500ms = 10s) → `companionReaction` cleared
3. FADE_WINDOW (6 ticks = 3s) before clear → `fading: true` → bubble dims
4. Scroll in non-fullscreen → `companionReaction` cleared immediately (line 1299-1305 REPL.tsx)

### Narrow terminal behavior (< 100 cols MIN_COLS_FOR_FULL_SPRITE)
- Full sprite hidden, shows one-line face
- Speech shown inline as `"quip text"` beside face
- Capped at 24 chars (NARROW_QUIP_CAP) with `…`

---

## APRIL FOOLS TEASER SYSTEM (useBuddyNotification.tsx)

### Teaser window
```typescript
// April 1-7, 2026 only (local date, not UTC — rolling wave across timezones)
isBuddyTeaserWindow(): d.getFullYear() === 2026 && d.getMonth() === 3 && d.getDate() <= 7
```

### Live date
```typescript
// Feature goes live: April 2026 onward
isBuddyLive(): d.getFullYear() > 2026 || (d.getFullYear() === 2026 && d.getMonth() >= 3)
```

### Notification
- Key: `'buddy-teaser'`
- Content: `/buddy` rendered in rainbow colors (each character a different color via `getRainbowColor(i)`)
- Priority: `'immediate'`
- Duration: 15,000ms (15 seconds)
- Trigger: on startup, if `!config.companion && isBuddyTeaserWindow()`
- The `/buddy` text in user input is also highlighted — `findBuddyTriggerPositions(text)` returns character ranges for rainbow rendering

---

## RENDERING CONSTANTS

```typescript
MIN_COLS_FOR_FULL_SPRITE = 100    // Terminal width threshold
SPRITE_BODY_WIDTH = 12            // Body char width
NAME_ROW_PAD = 2                  // Focused state: ' name ' padding
SPRITE_PADDING_X = 2              // paddingX in Box wrapper
BUBBLE_WIDTH = 36                 // SpeechBubble box (34) + tail
NARROW_QUIP_CAP = 24              // Max chars in narrow-mode quip
TICK_MS = 500                     // ms per animation tick
BUBBLE_SHOW = 20                  // Ticks bubble stays visible (~10s)
FADE_WINDOW = 6                   // Ticks before hide where bubble dims (~3s)
PET_BURST_MS = 2500               // ms hearts float after pet
```

---

## DISPLAY LAYOUT

### Full-width terminal (>= 100 cols), no speech
```
[padding] [sprite column: body lines + name row] [padding]
```

### Full-width terminal, speaking, non-fullscreen
```
[padding] [SpeechBubble text + tail →] [sprite column] [padding]
```

### Full-width terminal, speaking, fullscreen mode
```
Sprite only inline (no bubble)
Bubble rendered separately in FullscreenLayout's bottomFloat slot as CompanionFloatingBubble
```

### Narrow terminal (< 100 cols)
```
[padding] [face one-liner] [space] [name or "quip"] [padding]
```

### Pet animation (full-width)
```
[heart frame prepended above sprite body]
[sprite body lines (cycling fast)]
[name row]
```

---

## SPRITE COLUMN WIDTH CALCULATION

```typescript
spriteColWidth(nameWidth) = max(12, nameWidth + 2)
// 12 = SPRITE_BODY_WIDTH
// +2 = NAME_ROW_PAD (focused state ' name ' wrapping)
```

---

## SPECIES LIST (complete, 18 total)

In SPECIES array order:
1. duck
2. goose
3. blob
4. cat
5. dragon
6. octopus
7. owl
8. penguin
9. turtle
10. snail
11. ghost
12. axolotl
13. capybara
14. cactus
15. robot
16. rabbit
17. mushroom
18. chonk

---

## NOTES FOR MOTION DESIGN CONVERSION

### Key animation principles
- All species: 3 frames (indices 0, 1, 2)
- Frame 0 is always the canonical "rest" pose
- Frame 1 is always a subtle fidget (0.5s hold per idle cycle)
- Frame 2 is the alt fidget OR ambient special (line 0 effects)
- Line 0 "special effects": dragon ~smoke~, octopus bubble, ghost sparks, capybara water, robot antenna spark, mushroom spores, penguin sliding
- Blink is NOT a frame — it's a render pass: replace all eye chars with `-`

### Eye substitution in motion
Every `{E}` in the ASCII is a single character wide. When converting to motion:
- Default eyes: `·` (middle dot) — the "neutral" look
- Variations: `✦` sparkle, `×` cross/dead, `◉` hypno, `@` dazed, `°` serene

### Companion card sizing
Sprite body: 12 chars wide × 4-5 lines tall (5 with hat or line 0 effect, 4 otherwise)
Name centered below body.

### Rarity visual distinction
- common → dim/gray
- uncommon → green
- rare → blue/purple
- epic → orange/autoAccept
- legendary → gold/warning
- shiny (1%) → special shimmer variant

### Hat positioning
Always row 0, centered over the 12-char body:
- Crown: positions 3-7 (`\^^^/`)
- Tophat: positions 3-7 (`[___]`)
- Propeller: positions 4-6 (`-+-`)
- Halo: positions 3-7 (`(   )`)
- Wizard: positions 4-6 (`/^\`)
- Beanie: positions 3-7 (`(___)`)
- Tinyduck: position 4-5 (`,>`)
