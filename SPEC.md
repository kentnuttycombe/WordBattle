# Territory Words - Game Specification

## Overview

A territory capture word game on an 11x11 grid of random letters. Players select letters that spell valid English words; the selected letters form vertices of a polygon, and the area inside is claimed as territory. Used letters disappear from the board after each word. Supports three game modes: Player vs Player (local), Player vs Computer (AI), and a guided Tutorial.

## Technology

- Single-file HTML/JavaScript/Canvas application
- No build step or dependencies (except a dictionary API call)
- Runs in any modern browser
- Embedded word list (~2500 words) for AI opponent

---

## Core Requirements

### R1 - Game Board

- R1.1: Display an 11x11 grid of uppercase random letters (A-Z).
- R1.2: Letters are evenly spaced and rendered as a grid on an HTML Canvas (600x600 pixels: 11×50 + 25×2 padding).
- R1.3: Each letter cell has a clearly defined center point that serves as the vertex when selected.
- R1.4: The board is generated once at game start. Used letters disappear from the board after each word is played (see R3.12).
- R1.5: Letter distribution should be weighted toward common English letters (E, T, A, O, I, N, S, R, H, L appear more frequently) to make word formation more feasible.

### R2 - Turn System

- R2.1: Two players alternate turns: **Blue** goes first, then **Red**.
- R2.2: Each player gets **8 turns** each (16 turns total).
- R2.3: The current player and remaining turns are displayed in the UI.
- R2.4: A player may **pass** their turn if they cannot form a word.
- R2.5: After all 16 turns, the game ends and a winner is declared.

### R3 - Letter Selection & Word Formation

- R3.1: On their turn, a player clicks letters on the grid one at a time.
- R3.2: Each selected letter is visually highlighted with the current player's color.
- R3.3: As letters are selected, lines are drawn connecting them in order (vertex 1 -> 2 -> 3 -> ...).
- R3.4: A player can **undo** their last selection (backtrack one letter at a time).
- R3.5: A player can **clear** all selections to start their turn over.
- R3.6: When the player submits, the word is validated against a free dictionary API (https://api.dictionaryapi.dev/api/v2/entries/en/<word>).
- R3.7: If the API returns a valid result, the word is accepted. If it returns 404, the word is rejected and the player must try again.
- R3.8: The same board letter (same row, column) cannot be selected twice in one turn.
- R3.9: The minimum word length is **3 letters**.
- R3.10: **Selection range restriction (Manhattan distance):** The first letter of a word may be selected from anywhere on the board. Each subsequent letter must be within a **Manhattan distance of 4 cells** from the previously selected letter. For example, if the last selected letter is at (3, 4), the next letter must be at a cell (r, c) where `|r - 3| + |c - 4| <= 4`. The reachable area forms a diamond shape.
- R3.11: **Out-of-range graying:** Once at least one letter has been selected, all cells that are outside the valid range from the most recently selected letter are visually **grayed out** (dimmed text and a dark overlay) to indicate they cannot be selected. Clicking a grayed-out cell has no effect.
- R3.12: **Letter consumption:** After a word is played (whether it captures territory or is collinear), the used letters are removed from the board. Empty cells cannot be clicked, show no letter, and are skipped by the AI. Letters are a finite resource — plan carefully.

### R4 - Polygon, Territory & Strength

- R4.1: Upon word acceptance, a polygon is formed by connecting the selected letter positions in order, then closing the shape (last point connects back to first).
- R4.2: The interior of the polygon is filled/shaded with the current player's color. The **opacity/intensity** of the color reflects the territory's current strength level (see R4.7–R4.9). Letters beneath must remain readable at all strength levels.
- R4.3: Territory is tracked at the **pixel level** on a dedicated offscreen data layer. Each pixel stores: **owner** (none, blue, red, or neutral) and **strength** (0–5).
- R4.4: If the selected points are **collinear** (form a line with zero area), the word is accepted as a valid turn but no territory is awarded. The turn is consumed.
- R4.5: Polygons may be **concave** or **convex** — the Canvas `fill()` with the standard winding rule handles this.
- R4.6: Self-intersecting polygons are allowed; the Canvas fill rule determines which pixels are interior.
- R4.7: **Capture strength** is determined by word length: a 3-letter word captures at strength **1**, a 4-letter word at strength **2**, a 5-letter word at strength **3**, a 6-letter word at strength **4**, and a 7+-letter word at strength **5** (maximum). Formula: `strength = min(word.length - 2, 5)`.
- R4.8: There are **5 degrees of strength**, visually represented by progressively darker/more opaque shading of the player's color. Strength 1 is the lightest tint; strength 5 is the most saturated.
- R4.9: Suggested opacity mapping (per strength level): 1 → 0.12, 2 → 0.24, 3 → 0.36, 4 → 0.48, 5 → 0.60. These values keep letters readable while clearly differentiating strength levels.

### R5 - Territory Overlap & Conflict

Territory conflict uses a **strength subtraction** model. When territories collide, the attacking strength chips away at the defending strength.

- R5.1: Each pixel on the board is tracked with an **owner** (unclaimed, blue, red, or neutral/purple) and a **strength** (0–5).
- R5.2: **Claiming unclaimed territory:** The pixel becomes the attacker's color at the attacking strength.
- R5.3: **Attacking opponent territory (attack < defense):** The defender keeps the pixel, but their strength is reduced.
- R5.4: **Attacking opponent territory (attack = defense):** Strengths cancel out. The pixel becomes **purple (neutral)** at strength 0.
- R5.5: **Attacking opponent territory (attack > defense):** The attacker captures the pixel. The new strength is attack minus defense.
- R5.6: **Claiming own territory (stacking):** Strengths **add together**, capped at 5.
- R5.7: **Claiming neutral/purple territory:** The pixel becomes the attacker's color at the full attacking strength (purple has strength 0).
- R5.8: The visual rendering reflects all states: blue and red at varying intensities (5 levels), and purple for neutral pixels.

### R6 - Scoring

- R6.1: Score = total number of pixels of the player's color on the territory canvas.
- R6.2: Blue and Red scores are displayed at the top of the screen, updated after each turn with smooth animated counters.
- R6.3: Purple pixels count for **neither** player.
- R6.4: At game end, the player with the higher score wins. If scores are equal, the game is a draw.

### R7 - UI & Display

- R7.1: The game board is centered on screen with a clean layout.
- R7.2: A header area shows: current player's turn, Blue score, Red score, turns remaining.
- R7.3: The currently-being-spelled word is displayed as text so players can see what they're forming.
- R7.4: Buttons: **Submit Word**, **Undo**, **Clear**, **Pass Turn**.
- R7.5: Feedback messages for: invalid word, word too short, API errors, successful territory claim.
- R7.6: At game end, a polished overlay declares the winner with final scores, glowing title text, gradient background, and a **New Game** button that returns to the menu.
- R7.7: Canvas has a colored glow (box-shadow) matching the current player's color.
- R7.8: Turn indicator pulses briefly when switching players.
- R7.9: Buttons have enhanced hover shadows and transitions.

### R8 - Menu System

- R8.1: On load, a full-screen menu overlay is displayed with the gradient title "Territory Words" and three mode cards.
- R8.2: **Player vs Player (PvP):** Two players alternate turns locally on the same device.
- R8.3: **Player vs Computer (PvC):** Player controls Red; computer controls Blue and goes first.
- R8.4: **Tutorial:** Guided step-by-step walkthrough on a pre-set board.
- R8.5: Menu fades out with a smooth transition when a mode is selected.
- R8.6: **New Game** always returns to the menu.

### R9 - Player vs Computer (AI)

- R9.1: An embedded word list of ~2500 common 3-5 letter English words is stored as a space-separated string, parsed into a `Set` for lookup and a prefix `Set` for DFS pruning.
- R9.2: **AI word search** (`aiFindWords()`): DFS from each starting cell, exploring paths up to 5 letters, respecting Manhattan distance constraint, pruned by prefix set.
- R9.3: **AI word selection** (`aiChooseWord()`): Tuned for moderate difficulty. Strongly favors 4-letter words (bonus 500) over 3-letter (250) or 5-letter (100). Rewards proximity (closer letters score higher). Area bonus capped at 500. Overlap bonus reduced to 150. High randomness (0-600) for variety. Picks from top 8 candidates.
- R9.4: **AI turn execution** (`aiTakeTurn()`): Shows "Computer is thinking..." message, delays 800-1500ms, animates letter selection one at a time (250ms each), validates against dictionary API (accepts on network error).
- R9.5: Computer plays as Blue (goes first); player plays as Red.
- R9.6: All click/button handlers are guarded with `aiThinking` flag.
- R9.7: Game over shows "You Win!" / "Computer Wins!" in PvC mode. Scores display as "CPU" and "You".
- R9.8: Falls back to passing if no words found or API rejects all candidates.

### R10 - Tutorial Mode

- R10.1: Uses a pre-set 11x11 grid with known letter positions.
- R10.2: Tutorial overlay is positioned at the bottom of the screen (doesn't block the board).
- R10.3: Shows step title, description, and Next/Skip buttons.
- R10.4: 8 guided steps: welcome, click H, spell HELP (E/L/P forced clicks), submit & capture territory, letters disappear explanation, range & distance, strength/overlap/scoring, completion.
- R10.5: Tutorial word is **HELP** — H(0,0) → E(0,4) → L(4,4) → P(4,0). Each pair is exactly Manhattan distance 4 (= MAX_RANGE). The polygon forms a 200×200 pixel square capturing ~40,000 pixels of territory.
- R10.6: Tutorial click handler only allows clicking highlighted target cells. Shows hint message on wrong clicks.
- R10.7: Tutorial skips dictionary API validation for word submission.
- R10.8: Skip button and completion both return to the menu.

### R11 - Visual Effects & Animations

- R11.1: **Particle effects**: Burst of ~30 colored particles at polygon centroid on territory capture, fading over ~1s with gravity.
- R11.2: **Floating score text**: "+N px" floats upward and fades after territory capture.
- R11.3: **Animated score counters**: Scores interpolate smoothly to new values instead of jumping.
- R11.4: **Turn pulse**: Turn indicator briefly scales up on player switch.
- R11.5: **Canvas glow**: Subtle colored box-shadow on canvas matching current player's color.
- R11.6: **Enhanced buttons**: Better hover shadows and smooth transitions.
- R11.7: **Polished game over**: Gradient background and glowing title text.
- R11.8: Each animation system runs its own independent `requestAnimationFrame` loop.

---

## Milestones

### Milestone 1 — Playable Board with Territory Claiming

**Goal:** Two players can take turns selecting letters, forming polygons, and visually claiming territory with strength-based shading.

### Milestone 2 — Scoring, Overlap Rules & Dictionary Validation

**Goal:** Full game mechanics — validated words, strength-based territory conflict, live scoring, and turn limits.

### Milestone 3 — Polish & Quality of Life

**Goal:** Visual refinements, better UX, and edge-case handling. Turn history sidebar, sound effects, hover effects, responsive layout.

### Milestone 4 — Menu, Game Modes & Visual Modernization

**Goal:** Menu system with PvP/PvC/Tutorial modes, AI opponent, guided tutorial, particle effects, animated scores, Manhattan distance metric, and polished visual design.
