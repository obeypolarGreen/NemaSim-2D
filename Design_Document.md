# NemaSim-2D — Technical Design Document
### Version 3.0 (Consolidated) · *C. elegans* Lifecycle & Mortality Simulation

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture & Constants](#2-architecture--constants)
3. [Environment Grid](#3-environment-grid)
4. [Agent Architecture — The Brain](#4-agent-architecture--the-brain)
5. [Physical Simulation — The Body](#5-physical-simulation--the-body)
6. [Reproduction & Lifecycle](#6-reproduction--lifecycle)
7. [Mortality System](#7-mortality-system)
8. [Visual & Rendering System](#8-visual--rendering-system)
9. [HUD & Telemetry](#9-hud--telemetry)
10. [Controls & User Interaction](#10-controls--user-interaction)
11. [Known Bugs Fixed](#11-known-bugs-fixed)
12. [Parameter Reference](#12-parameter-reference)

---

## 1. System Overview

**NemaSim-2D** is a JavaScript (Canvas 2D API) multi-agent simulation modelling the complete lifecycle of *Caenorhabditis elegans* nematode worms. The simulation abstracts the 302-neuron connectome into three functional layers — **Sensory Integration**, **Neuromodulatory State**, and **Locomotor Actuation** — while adding a full reproductive system, sexual dimorphism, pheromone-driven mate-seeking, and a mortality model.

**Canvas:** 700 × 590 px  
**Grid cell size:** 4 px (175 × 147 cells)  
**Target platform:** Browser (single HTML file, no dependencies)

---

## 2. Architecture & Constants

### 2.1 Lifecycle Constants

| Constant | Value | Description |
|---|---|---|
| `LARVA_TICKS` | 600 | Ticks for larva to mature into adult |
| `SPERM_START` | 300 | Initial self-sperm count for hermaphrodites |
| `EGG_LAY_RATE` | 90 | Ticks between eggs when on food (standard) |
| `EGG_LAY_RATE` (mated) | 45 | Ticks between eggs after male mating (2× rate) |
| `EGG_HATCH` | 400–500 | Ticks for egg to hatch (+ up to 100 random jitter) |
| `MATING_DIST` | 14 px | Contact distance for mating trigger |
| `MATING_TICKS` | 120 | Duration of sperm transfer event |
| `MAX_POP` | 60 (adjustable) | Hard population ceiling |

### 2.2 Death Constants

| Constant | Value | Description |
|---|---|---|
| `LIFESPAN_ADULT` | 2500 ticks | Base adult lifespan |
| `LIFESPAN_JITTER` | ±500 ticks | Random variance per individual |
| `STARVATION_DEATH` | 1800 ticks | Hunger ticks before starvation kills |
| `OVERCROWD_RADIUS` | 20 px | Neighbor detection radius for crowding |
| `OVERCROWD_LIMIT` | 8 | Neighbor count threshold for crowding stress |
| `OVERCROWD_TICKS` | 600 | Ticks under crowding before death |
| `NUTRIENT_RADIUS` | 16 px | Food patch radius released on death |

### 2.3 Physics Constants

| Constant | Value | Description |
|---|---|---|
| `N_SEG` | 12 | Body segments per worm |
| `SPEED_BASE` | 1.2 px/tick | Base movement speed |
| Oscillation freq (off food) | 0.085 rad/tick | Head sine frequency roaming |
| Oscillation freq (on food) | 0.028 rad/tick | Head sine frequency feeding |
| Amplitude (off food) | 0.42 | Lateral swing magnitude |
| Amplitude (on food) | 0.22 | Reduced swing on food |

---

## 3. Environment Grid

The environment is a flat `Float32Array` / `Uint8Array` grid (COLS × ROWS cells, 4 px each). Three separate layers are maintained simultaneously.

### 3.1 Grid Layers

| Layer | Type | Description |
|---|---|---|
| `food` | `Uint8Array` | Binary bacteria patches. Boolean per cell. |
| `pheromone` | `Float32Array` | General aggregation signal. Emitted by all worms. |
| `ascr3` | `Float32Array` | Sex pheromone emitted by hermaphrodites. Tracked by males. |
| `ascr10` | `Float32Array` | Sex pheromone emitted by males. Tracked by sperm-depleted hermaphrodites. |

### 3.2 Diffusion

All three pheromone layers are diffused every 5 simulation ticks using a 2D discrete diffusion kernel:

```
next[i] = ((layer[i]×4 + N + S + E + W) / 8) × decay
```

| Layer | Decay Rate |
|---|---|
| `pheromone` | 0.995 per diffusion pass |
| `ascr3` | 0.993 per diffusion pass |
| `ascr10` | 0.993 per diffusion pass |

Sex pheromones decay faster than general pheromone, creating sharper and more transient mating gradients.

### 3.3 Food Mechanics

- Food patches are placed as circles of filled cells at initialisation and on demand.
- Food is **not** consumed every tick. One cell is consumed per egg laid, causing gradual, biologically realistic depletion.
- If total food drops below 3% of grid area, a new patch is automatically added every 300 ticks.
- Worm death releases a circular food patch (`NUTRIENT_RADIUS` = 16 px) — the body decomposes into nutrients.

### 3.4 Chemotaxis Gradient Sampling

Agents sample the 4-directional finite difference gradient at their head position:

```
gradient = ((E - W) + (S - N)) × 0.5
```

- **Positive gradient** → suppress turning (run straight toward concentration)
- **Negative gradient** → increase turn probability (pirouette to reorient)

---

## 4. Agent Architecture — The Brain

### 4.1 Sensory System (Inputs)

Agents query the environment at the head segment's `(x, y)` coordinates each tick.

| Modality | Biological Correlate | Implementation |
|---|---|---|
| Chemotaxis (general) | ASE neurons (ASEL/ASER) | Gradient of `pheromone` layer. Positive = suppress turn. Negative = trigger pirouette. |
| Chemotaxis (sex) | ADL/ASK neurons | Males track `ascr3` gradient. Sperm-depleted hermaphrodites track `ascr10` gradient. |
| Food sense | NSM neurons | Boolean `checkFood(head.x, head.y)`. Triggers dopamine rise. |
| Mechanosensation (wall) | ALM/AVM, PLM neurons | Boundary check at 10 px margin. Triggers reversal. |
| Social/aggregation mode | ADL/ASK neurons | `social` gene flag: `true` = positive chemotaxis, `false` = repelled by pheromone. |

### 4.2 Neuromodulatory State

Three neuromodulators are tracked as continuous floats [0.0–1.0] and updated each tick.

#### Dopamine — Food Response
```
On food:   dopamine = min(1, dopamine + 0.05)
Off food:  dopamine *= 0.97
```
- Reduces locomotion speed by up to 50%
- Increases turn probability
- Triggers `DWELL` state when > 0.4

#### Serotonin — Satiety
```
On food:   serotonin = min(1, serotonin + 0.01)
Off food:  serotonin *= 0.99
```
- Accumulates slowly with sustained feeding
- Does not directly gate behaviours in current implementation; reserved for feasting enhancement

#### Octopamine — Hunger / Stress
```
On food:   octopamine = max(0, octopamine - 0.02)
Off food:  octopamine = min(1, octopamine + 0.002)
```
- Inhibits egg laying when > 0.7
- Boosts roaming speed by up to 35% (`speed × (1 + octopamine × 0.35)`)
- Suppresses egg-laying drive during starvation

### 4.3 Neural Control Unit — State Machine

The 302-neuron connectome is abstracted to 6 discrete behavioural states, each mapped to a biological command interneuron circuit.

| State | Biological Analogue | Trigger Condition |
|---|---|---|
| `FORWARD` | AVB command interneuron | Default roaming state |
| `REVERSAL` | AVA command interneuron | Wall contact; anterior touch |
| `OMEGA_TURN` | AIB/AIZ interneurons | Post-reversal; negative chemotaxis gradient |
| `DWELL` | NSM/AIY circuits | `onFood = true` AND `dopamine > 0.4` |
| `MATING` | Male-specific circuit | Tail-to-head contact with partner |
| `MATE_SEEK` | Sperm-depleted circuit | `sperm == 0` AND `maleSpermBoost == 0` (hermaphrodite only) |

#### State Transition Logic

```
if (reversal timer > 0)         → REVERSAL
else if (omega timer > 0)       → OMEGA_TURN
else if (near wall)             → REVERSAL (timer: 35–60 ticks)
else if (onFood && dopa > 0.4)  → DWELL
else if (sperm depleted)        → MATE_SEEK  (tracks ascr10 gradient)
else if (sex == male)           → FORWARD    (tracks ascr3 gradient)
else                            → FORWARD    (probabilistic turn check)

Turn probability:
  pTurn = max(0, turn_bias × 0.015 − gradient × 0.008)
  if (random() < pTurn) → OMEGA_TURN (timer: 20–45 ticks)
```

#### Post-Reversal Sequence
```
REVERSAL (35–60 ticks) → OMEGA_TURN (25 ticks) → FORWARD
```

---

## 5. Physical Simulation — The Body

### 5.1 Segment Chain

Each worm is a chain of **12 rigid segments**. The head segment is driven by the locomotion engine; subsequent segments copy the position of the segment ahead of them with a one-tick propagation delay, creating natural undulatory wave propagation.

```javascript
Segment[i].x = Segment[i-1].x  // previous tick position
Segment[i].y = Segment[i-1].y
```

### 5.2 Head Oscillator

The head angle advances as a sinusoidal oscillator:

```
oscPhase += frequency
headAngle += sin(oscPhase) × amplitude × 0.08
```

Parameters are modulated by state and neuromodulator levels:

| Condition | Frequency | Amplitude | Speed Multiplier |
|---|---|---|---|
| Off food (roaming) | 0.085 | 0.42 | 1.0 − dopamine×0.45 |
| On food (feeding) | 0.028 | 0.22 | 0.25 (DWELL) |
| Reversal | 0.085 | 0.42×(1−tyramine×0.6) | 0.55 |
| Omega Turn | — | — | 0.18 |
| Mating | — | — | 0.10 |
| Mate Seeking | 0.085 | 0.42 | 1.30 |

### 5.3 Sex-Based Speed Modifiers

- Males receive an additional **1.2× speed boost** (biased roaming)
- Larvae move at **0.5× speed** (smaller body)
- Octopamine hunger adds up to **+35% speed** when starving

### 5.4 Omega Turn Mechanics

During `OMEGA_TURN`, the normal sinusoidal oscillator is replaced with a sharp ventral swing:

```javascript
omegaPhase += 0.11
headAngle  += sin(omegaPhase) × 0.14
```

This produces a >90° reorientation over 25 ticks, mimicking the biological pirouette.

---

## 6. Reproduction & Lifecycle

### 6.1 Sex Determination

| Sex | Genotype | Frequency | Role |
|---|---|---|---|
| Hermaphrodite | XX | ~97% | Self-fertile; lays eggs; seeks mates when sperm-depleted |
| Male | XO | ~3% (4% of hatchlings) | Roving mate-seeker; transfers sperm to hermaphrodites |

At initialisation, at least one male is guaranteed. If the random assignment produces none, the first worm is converted to male.

### 6.2 Hermaphrodite Lifecycle

```
Egg → Larva (0–600 ticks) → Adult (600+ ticks)
         ↓                        ↓
   Random walk            Self-fertile: lays eggs
   Half speed             Sperm: 300 → 0
   No reproduction        When sperm = 0: MATE_SEEK state
                          After mating: +1000 male sperm boost
```

**Egg laying conditions (all must be true):**
1. `sex == 'herm'`
2. `isAdult == true`
3. `onFood == true`
4. `octopamine < 0.7`
5. `totalFertility > 0`
6. `eggLayTimer >= EGG_LAY_RATE`

### 6.3 Egg Class

Eggs are persistent objects in the simulation with their own tick loop.

| Property | Value |
|---|---|
| Hatch time | 400–500 ticks |
| Visual size | Grows from r=2 to r=3.5 as it develops |
| Color (self-fertilised) | `#c8ffd4` (pale green) |
| Color (male-fertilised) | `#FFB0C0` (pale pink) |
| Hatch condition | `age >= hatchTime` AND `worms.length < MAX_POP` |
| Sex of hatchling | 96% hermaphrodite / 4% male |

### 6.4 Mating System

#### Trigger
Male tail collider contacts hermaphrodite head within `MATING_DIST` (14 px).

#### Mating Dance Sequence
1. Both worms enter `MATING` state for `MATING_TICKS` (120 ticks)
2. Both slow to 10% normal speed
3. Both glow with pink ring indicator
4. On completion: `_completeMating()` is called

#### Sperm Transfer (`_completeMating`)
```javascript
partner.maleSpermBoost = 1000   // male sperm precedence
partner.sperm          = 0      // clears self-sperm
```
Male sperm overwrites hermaphrodite self-sperm, increasing total fertility from ~300 to 1,000 offspring. Egg-laying rate doubles to `EGG_LAY_RATE × 0.5`.

#### Pheromone Emission
- Hermaphrodites emit `ascr3` (0.005/tick) — tracked by males
- Males emit `ascr10` (0.005/tick) — tracked by sperm-depleted hermaphrodites
- All worms emit general `pheromone` (0.003/tick)

---

## 7. Mortality System

All three mortality causes result in the same death sequence. Dead worms **release a food nutrient patch** at their position (radius 16 px), decomposing their body back into the ecosystem.

### 7.1 Death Causes

#### 1. Natural Lifespan
```
if (adultAge >= lifespan) → die('old age')

lifespan = LIFESPAN_ADULT + random(-LIFESPAN_JITTER, +LIFESPAN_JITTER)
         = 2500 ± 500 ticks
```
Each worm gets an individually randomised lifespan at birth, creating natural population turnover rather than synchronised die-off.

#### 2. Starvation
```
if (hunger >= STARVATION_DEATH) → die('starvation')
  hunger ticks ++  every tick off food
  hunger = 0       when food found
```
The hunger timer is the existing `brain.timers.hunger` counter. Starvation threshold is 1,800 continuous ticks without food contact.

#### 3. Overcrowding / Competition
```
neighbors = count of living worms within OVERCROWD_RADIUS (20 px)
if (neighbors >= OVERCROWD_LIMIT):
    crowdStress++
    if (crowdStress >= OVERCROWD_TICKS) → die('overcrowding')
else:
    crowdStress = max(0, crowdStress - 2)   // slow recovery
```
Crowding stress accumulates when ≥8 neighbors are within 20 px and dissipates at 2× recovery rate when conditions improve. A worm must sustain overcrowding for 600 ticks before dying, preventing immediate population crashes from brief clustering.

### 7.2 Death Sequence

```
die(cause) called
  → dead = true
  → dying = true (fade begins)
  → env.addFoodPatch(head position, radius=16)   ← nutrient release
  → totalDeaths++
  → log event

Per-frame: dyingAlpha -= 0.015   (fade over ~67 frames)
When dyingAlpha ≤ 0:
  → dying = false
  → worm removed from array on next prune pass
```

### 7.3 Population Dynamics

The combination of reproduction and mortality creates a self-regulating population cycle:

- Birth rate is food-dependent (egg laying requires food contact)
- Death rate rises with age, starvation, and density
- Nutrient recycling from corpses sustains new food patches, preventing total collapse
- The `MAX_POP` cap acts as a hard ceiling (default: 60, user-adjustable up to 150)

---

## 8. Visual & Rendering System

### 8.1 Render Order

1. Background fill (`#020705`)
2. Pheromone field (ImageData pixel pass)
3. Food patch cells + border glow
4. Eggs
5. Larvae (drawn first, below adults)
6. Adult worms

### 8.2 Pheromone Rendering

The three pheromone layers are composited into a single ImageData pass per frame:

```
pixel.R = ascr10 × 0.9 × 255       (male pheromone → red channel)
pixel.G = pheromone × 0.7 + ascr3 × 0.5  (general + herm → green channel)
pixel.B = ascr3 × 0.4 × 255        (herm pheromone → blue tint)
pixel.A = max(general, ascr3, ascr10) × 0.5 × 255
```

This produces a visible tri-channel field: green for general aggregation, pinkish-red where males have been, with overlapping zones appearing as mixed colours.

### 8.3 Sexual Dimorphism & State Colors

| Condition | Color | Hex |
|---|---|---|
| Hermaphrodite (healthy adult) | Pale green | `#E0FFE0` |
| Hermaphrodite (sperm-depleted / MATE_SEEK) | Yellow-green | `#D4E157` |
| Hermaphrodite (elderly, >80% lifespan) | Pale yellow | `#ffffcc` |
| Hermaphrodite (overcrowding stress >50%) | Pale red | `#ffcccc` |
| Male (XO) | Light pink | `#FFD1DC` |
| Larva | Grey | `#A0A0A0` |
| Dying worm (fading) | Dark grey | `#404040` |

### 8.4 Tail Dimorphism — Males

Males have a **fan-shaped tail** rendered as 6 radial lines emanating from the tail segment, spanning ±0.6 radians. This distinguishes them visually from hermaphrodites at a glance, matching the biological fan-tail morphology of *C. elegans* males.

### 8.5 Egg Rendering

Eggs are drawn as filled ellipses (width 1.5× height) that grow and brighten as they develop:
- `alpha = 0.4 + (age/hatchTime) × 0.5`
- `radius = 2 + (age/hatchTime) × 1.5`
- Self-fertilised: green tones; Male-fertilised: pink tones

### 8.6 State Indicators

| Indicator | Trigger | Visual |
|---|---|---|
| Mating glow ring | `state == MATING` | Pink ring (r=6) around head |
| Mate-seeking pulse | `state == MATE_SEEK` | Yellow-green blinking ring (r=5), 15-tick blink |
| Dopamine glow | `dopamine > 0.5` | Shadow blur on body colour |

### 8.7 Scanline Overlay

A CSS `repeating-linear-gradient` scanline effect is layered over the canvas as a fixed `pointer-events: none` div, giving the simulation a CRT/lab-monitor aesthetic.

---

## 9. HUD & Telemetry

### 9.1 Population Statistics Panel

| Stat | Source |
|---|---|
| Total | `worms.length` |
| Hermaphrodites | `worms.filter(w => w.sex=='herm' && w.isAdult).length` |
| Males | `worms.filter(w => w.sex=='male' && w.isAdult).length` |
| Larvae | `worms.filter(w => !w.isAdult).length` |
| Eggs Laid | `totalEggsLaid` (cumulative) |
| Deaths | `totalDeaths` (cumulative) |
| Tick | simulation tick counter |
| Food | `countFood() / 10` (normalised cell count) |
| FPS | rolling 500ms frame count |

### 9.2 Neuromodulator Bars

Average values across all living adults displayed as animated progress bars:
- **Dopamine** — green
- **Serotonin** — cyan
- **Octopamine** — amber

### 9.3 Behavioural State Counts

Live count of agents in each state: Forward, Reversal, Omega Turn, Dwelling, Mating, Mate Seeking.

### 9.4 Event Log

Prepended log (max 60 entries, auto-cleared) with colour-coded categories:

| Category | Colour | Events |
|---|---|---|
| `info` | Dim green | Init, food added, pheromone toggle |
| `warn` | Amber | Deaths (with cause), social gene changes |
| `mate` | Pink | Mating initiated, sperm transfer complete |
| `birth` | Yellow-green | Larvae hatched, eggs laid (with remaining fertility) |

---

## 10. Controls & User Interaction

### 10.1 Button Controls

| Button | Action |
|---|---|
| ⏸ Pause / ▶ Resume | Toggle simulation |
| ↺ Reset | Re-initialise with current slider values |
| + Food | Place random food patch |
| + Herm. | Spawn adult hermaphrodite at random location |
| + Male | Spawn adult male at random location |
| Pheromones | Toggle pheromone field rendering |

### 10.2 Sliders

| Slider | Range | Default | Effect |
|---|---|---|---|
| Initial Worms | 2–40 | 12 | Worm count on Reset |
| Food Patches | 1–20 | 6 | Patch count on Reset |
| Sim Speed | 1–5× | 1× | Ticks per animation frame |
| Pop. Cap | 10–150 | 60 | `MAX_POP` hard ceiling |

### 10.3 Canvas Interaction

**Click** on the canvas to place a food patch of radius 22 px at the clicked location.

---

## 11. Known Bugs Fixed

### Bug 1 — Egg Laying Never Fired (Operator Precedence)

**Version:** v2.0 → v2.1

**Symptom:** Zero eggs laid after 75,000+ ticks with multiple hermaphrodites on food.

**Root cause:** JavaScript operator precedence error in the egg-laying gate condition:

```javascript
// BROKEN — !nm.octopamine evaluates first (boolean), then false > 0.7 = false always
if (this.sex === 'herm' && this.isAdult && onFood && !nm.octopamine > 0.7)

// FIXED
if (this.sex === 'herm' && this.isAdult && onFood && nm.octopamine < 0.7)
```

### Bug 2 — Food Patches Destroyed Instantly

**Version:** v2.0 → v2.1

**Symptom:** Food cells were erased each tick a worm stood on them, causing `onFood` to be nearly always `false` and starving worms even when "on" a patch.

**Root cause:** `env.consumeFood()` was called every tick inside the sensory block rather than only at egg-laying events.

**Fix:** Removed per-tick consumption. Food is now only consumed at the moment an egg is laid (`_layEgg()` calls `consumeFood()` once per egg). This makes food depletion gradual and proportional to reproduction activity.

---

## 12. Parameter Reference

### Quick-Tune Cheat Sheet

| Want to... | Change |
|---|---|
| Longer-lived worms | Increase `LIFESPAN_ADULT` |
| Faster population growth | Decrease `EGG_LAY_RATE` or increase `SPERM_START` |
| More starvation deaths | Decrease `STARVATION_DEATH` |
| Looser crowding tolerance | Increase `OVERCROWD_LIMIT` or `OVERCROWD_TICKS` |
| Faster larvae | Increase larva speed `spd` in `_larvaTick()` |
| Males find mates faster | Increase `ascr3` deposit rate or decrease decay |
| More ecological nutrients on death | Increase `NUTRIENT_RADIUS` |
| Richer pheromone field | Decrease diffusion decay rates |

### Biological Reference Points

| Parameter | C. elegans Biology | Simulation Equivalent |
|---|---|---|
| Lifespan | ~20 days at 20°C | 2500 adult ticks |
| Self-progeny | ~300 offspring | 300 self-sperm |
| Cross-progeny (mated) | ~1,400 offspring | 1000 male sperm boost |
| Egg hatch time | ~12–14 hours | 400–500 ticks |
| L4 → adult | ~8 hours | 600 ticks |
| Body length | 1 mm | ~50 px at 1:1 scale |
| Locomotion speed | 0.2 mm/sec | `SPEED_BASE = 1.2` px/tick |

---

*NemaSim-2D — Built iteratively from the original connectome abstraction design through reproduction, lifecycle, and mortality systems. All mechanics reflect functional approximations of C. elegans biology rather than neuron-level simulation.*
