# Brick Breaker Enhancement Plan

## Design Philosophy

Transform the current single-level daily challenge into a dynamic, escalating survival game where the player fights against a rising tide of bricks. All mechanics are original compositions of general game design patterns.

**Visual principle**: Match the mpOS aesthetic — clean, functional, low graphical complexity. Flat-shaded bricks with simple edge highlights. No particle systems, no screen shake, no glow effects, no animated trails. The game should look like it belongs in this desktop environment.

---

## 1. Color System — 3-Color Palette (OKLCH Perceptual Space)

### Mathematical Basis

Three hues at near-triadic spacing (~120° apart) in OKLCH, adjusted to Paul Tol's CVD-validated anchors. Equal perceptual lightness (L) and chroma (C) per shade tier — no color dominates. All three remain distinguishable under deuteranopia, protanopia, and tritanopia.

### Background

**`#08090e`** — near-black. Maximizes luminance contrast with all three brick colors.

### The Three Colors

Each color has **3 shades** — highlight (top/lit edge), base (brick face), shadow (bottom/dark edge).

| Color | Hue (OKLCH) | Highlight | Base | Shadow |
|-------|------------|-----------|------|--------|
| **Cyan** | h=245° | `#85ccff` | `#279ced` | `#0f68a2` |
| **Amber** | h=80° | `#eabc6e` | `#c68800` | `#865900` |
| **Rose** | h=5° | `#ffa6ba` | `#e16788` | `#99415a` |

- All base shades pass WCAG AA (>5:1) against `#08090e`
- All highlight shades pass WCAG AAA (>8.9:1)
- All pairs distinguishable under simulated color blindness

### Element Colors

- **Bricks**: 3 colors only. Type expressed through markings (lines, diamonds), not extra hues.
- **Ball**: White `#f0f0f0`, solid fill, no glow. Amber `#c68800` during burner mode.
- **Paddle**: Solid `#85ccff` (cyan highlight) with `#0f68a2` (cyan shadow) bottom edge. Simple raised rectangle matching the XP beveled style.
- **Power-up capsules**: Solid base color with a white letter centered.
- **HUD text**: `#d0d4dc` (light neutral gray).
- **Danger line**: Solid `#99415a` (rose shadow), 1px horizontal line.

---

## 2. Core Mechanic — The Rising Tide

### Current State

Single static field of ~20-40 bricks. Clear all to win. No escalation.

### New: Rising Rows

Every **N seconds**, a new row of bricks appears at the bottom of the play field. All existing bricks shift upward by one row height instantly (no animation — just reposition).

- **Game over**: Any brick crosses the danger line near the top
- Replaces "clear all bricks to win" — the game is now **endless survival** scored by points
- The danger line is a thin solid horizontal line in rose-shadow color

### Timing Curve

```
interval = max(4, 12 - (elapsedMinutes * 0.8))  seconds
```

Starts at 12s gaps, shrinks by ~0.8s per minute, floors at 4s. ~90 seconds of comfortable play before real pressure.

### New Row Generation

- 10 columns, ~60% fill rate (gaps for ball passage), rising toward 85% over time
- Color distribution: roughly equal thirds, randomized per cell via daily RNG
- Every 5th row: one tough brick (2 HP)
- Every 8th row: one bonus brick (3× point value)

---

## 3. Chain Cascade System

### Color Matching

When a brick is destroyed by the ball, flood-fill all **orthogonally adjacent** bricks of the **same color**. If the connected group has **3 or more** bricks, they all shatter.

Players who aim at large same-color clusters score far more than random play.

### Gravity Settle

After bricks are destroyed, bricks above the gap fall down to close it. Falling bricks that land adjacent to a new same-color group of 3+ trigger another cascade.

### Chain Counter

Each cascade step increments a chain counter with escalating score multiplier:
- Step 1 (initial hit): 1×
- Step 2 (first cascade): 3×
- Step 3: 6×
- Step 4: 10×
- Step 5+: +5× per link

### Cascade Visuals (Minimal)

- Bricks targeted for cascade briefly **invert** (swap highlight and shadow colors) for 200ms, then disappear
- Bricks falling to settle simply reposition (no easing animation — instant, like the rest of the UI)
- Chain multiplier displayed as plain text ("×3") at the HUD, same font as score — no floating text, no scaling effects

---

## 4. Power-Up Overhaul

### Current: 4 types (Widen, Shrink, Multiball, Burner)

### Revised: 7 types

Drop rate stays at 12%. Duration stays at 8s for timed effects. Power-ups are small solid-color rectangles with a centered white letter, falling at constant speed.

| # | Name | Letter | Capsule Color | Effect |
|---|------|--------|---------------|--------|
| 1 | **Widen** | W | Cyan base | Paddle grows +24px (max 120px). Timed 8s. |
| 2 | **Multiball** | M | Cyan base | Spawns 2 extra balls at ±30°. |
| 3 | **Burner** | B | Amber base | Ball passes through bricks without bouncing. Timed 8s. |
| 4 | **Blast** | X | Amber base | Destroys all bricks in a 3×3 area around the last-hit brick. One-shot. Triggers cascades. |
| 5 | **Freeze** | F | Rose base | Pauses the rising tide timer for 15s. |
| 6 | **Magnetize** | G | Rose base | Ball curves gently toward nearest brick cluster. Timed 8s. |
| 7 | **Shrink** | S | `#404040` (dark) | Paddle shrinks -20px (min 40px). Timed 8s. Negative. |

### Drop Weights

```
Widen: 20%    Multiball: 15%   Burner: 15%
Blast: 10%    Freeze: 15%      Magnetize: 10%
Shrink: 15%
```

### Debris Meter

Destroyed bricks fill a **debris meter** — a thin horizontal bar at the bottom of the canvas, below the paddle. Solid fill, no color transitions (uses cyan base color). Fills left-to-right. Requires ~25 bricks to fill.

When full, the player presses **D** (or double-taps the bottom edge on mobile) to trigger a **Wave Clear**: the entire bottom row of bricks is destroyed instantly, triggering cascades. Meter resets.

---

## 5. Brick Types

| Type | HP | Points | Visual | Frequency |
|------|-----|--------|--------|-----------|
| **Normal** | 1 | 10 | Solid base color, 1px highlight top edge, 1px shadow bottom edge | ~65% |
| **Tough** | 2 | 25 | Same as normal. After first hit: a single horizontal crack line (1px, shadow color) across the middle | ~20% |
| **Bonus** | 1 | 50 | Small diamond outline (highlight color) centered on face | ~8% |
| **Hazard** | 1 | 0 | Diagonal X (shadow color) on face. Falls when hit. Damages paddle on contact. | ~7% |

All types come in all 3 colors and participate in cascade matching.

---

## 6. Scoring

### Point Sources

| Event | Points |
|-------|--------|
| Brick destroyed (by ball) | brick.points × chain multiplier |
| Brick destroyed (by cascade) | brick.points × chain multiplier |
| Wave Clear | 100 × bricks destroyed in the row |
| Survival bonus (on game over) | 100 × elapsed minutes |

### Combo Streak

Consecutive brick hits without the ball touching the paddle: each hit gets +5 bonus (hit 1 = +0, hit 2 = +5, hit 3 = +10...). Resets on paddle contact.

### Daily Seed

Same FNV-1a + Mulberry32 system. The seed determines the initial layout AND the sequence of rising rows. All players face identical challenges each day.

---

## 7. Rendering Approach

Everything drawn with `fillRect`, `strokeRect`, and `fillText`. No `createRadialGradient`, no `createLinearGradient`, no `arc` (except the ball circle), no `shadow*` canvas properties. Match the flat, beveled look of the XP-style UI.

### Bricks

```
fillRect(base color)           ← brick face
fillRect(highlight, top 1px)   ← lit edge
fillRect(shadow, bottom 1px)   ← dark edge
```

For tough bricks after first hit, add a 1px horizontal line at brick midpoint in shadow color.
For bonus bricks, draw a small diamond outline (4 lines) in highlight color at center.
For hazard bricks, draw an X in shadow color.

### Ball

```
arc + fill(#f0f0f0)   ← solid white circle, no glow
```

In burner mode: `fill(#c68800)` instead.

### Paddle

```
fillRect(#85ccff)              ← paddle face (cyan highlight)
fillRect(#0f68a2, bottom 2px)  ← dark edge (cyan shadow)
```

### HUD

Plain `fillText` in `#d0d4dc`, 13px bold, same font as the rest of mpOS. Score left-aligned, lives (dots) right-aligned. Debris meter: single `fillRect` bar below paddle.

### Danger Line

Single 1px `fillRect` in `#99415a` spanning the full width at the danger-zone Y coordinate.

---

## 8. Controls

- **Arrow keys / pointer**: Move paddle (unchanged)
- **Space / tap**: Launch ball (unchanged)
- **D key / double-tap bottom edge**: Activate Wave Clear (only when debris meter is full)

---

## 9. Game Flow

1. **Rules dialog** — updated text explaining rising rows, cascades, debris meter
2. **Initial layout** — ~25 bricks from daily seed, 3 colors
3. **Ball attached to paddle** — space/tap to launch
4. **Play phase** — ball bounces, bricks cascade, power-ups drop
5. **Rising tide starts** after 10 seconds
6. **New rows rise** on accelerating timer
7. **Game over** — brick crosses danger line OR all lives lost
8. **Score display** — final score, survival time, longest chain

### Lives: 3, displayed as dots in HUD.

---

## 10. Implementation Order

### Phase 1 — Color & Visual Foundation
1. Replace 4-color brick palette with 3-color system (Cyan/Amber/Rose shades)
2. Change background from `#1a1f2e` to `#08090e`
3. Update paddle rendering to flat cyan style
4. Update ball to solid white (remove radial gradient)
5. Update HUD text color to `#d0d4dc`
6. Simplify brick rendering to flat rects with 1px edge highlights

### Phase 2 — Rising Tide
7. Add rising row generation (new row at bottom, all bricks shift up)
8. Add danger line rendering
9. Change win condition from "clear all" to endless survival
10. Add timing curve (12s → 4s interval)
11. Update game-over screen to show survival time + score

### Phase 3 — Cascade System
12. Implement flood-fill same-color adjacency check
13. Add cascade clearing with chain counter and multiplier
14. Add gravity settle (bricks fall to fill gaps)
15. Add cascade visual feedback (color inversion flash, then disappear)
16. Add cascade timing delays (200ms between chain steps)

### Phase 4 — Power-up Expansion
17. Add Blast power-up (3×3 area destroy)
18. Add Freeze power-up (pause rising tide 15s)
19. Add Magnetize power-up (ball curves toward bricks)
20. Update power-up capsule visuals to flat colored rects with white letters
21. Add debris meter bar and Wave Clear activation

### Phase 5 — Scoring & Polish
22. Implement chain multiplier scoring
23. Add combo streak counter
24. Add survival time bonus
25. Update rules dialog text
26. Final consistency pass

---

## What Stays the Same

- Daily seed system (FNV-1a + Mulberry32)
- Canvas coordinate system (480×640 virtual)
- Ball physics (speed, angle, bounce, delta-time loop)
- Paddle movement (keyboard + pointer)
- Touch/mobile support
- Sound effects via mpAudio
- Window integration (titlebar, minimize, close)
- requestAnimationFrame game loop with dt capping
- Hazard falling mechanic
- Life system (3 lives)

## What's Removed

- "You Win!" condition (replaced by endless survival)
- 4th brick color (3-color system replaces it)
- Static grid assumption (bricks shift vertically now)
- Radial gradient on ball (solid fill instead)
- Paddle gradient (flat beveled rect instead)
- Existing particle system for brick destruction (replaced by simple color-inversion flash)
