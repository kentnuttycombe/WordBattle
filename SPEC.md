# Territory Words - Game Specification

## Overview

A two-player local multiplayer game where players compete to capture territory on a 12x12 grid of random letters. Each turn, a player selects letters from across the board that spell a valid English word. The selected letters form vertices of a polygon, and the area inside that polygon is claimed as that player's territory. The player with the most territory at the end wins.

## Technology

- Single-file HTML/JavaScript/Canvas application
- No build step or dependencies (except a dictionary API call)
- Runs in any modern browser

---

## Core Requirements

### R1 - Game Board

- R1.1: Display a 12x12 grid of uppercase random letters (A-Z).
- R1.2: Letters are evenly spaced and rendered as a grid on an HTML Canvas.
- R1.3: Each letter cell has a clearly defined center point that serves as the vertex when selected.
- R1.4: The board is generated once at game start and remains fixed for the entire game.
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
- R3.10: **Selection range restriction:** The first letter of a word may be selected from anywhere on the board. Each subsequent letter must be within a **maximum distance of 4 cells** from the previously selected letter (measured as Chebyshev distance — the max of the row difference and column difference). For example, if the last selected letter is at (3, 4), the next letter must be at a cell (r, c) where `max(|r - 3|, |c - 4|) <= 4`.
- R3.11: **Out-of-range graying:** Once at least one letter has been selected, all cells that are outside the valid 4-cell range from the most recently selected letter are visually **grayed out** (dimmed text and a dark overlay) to indicate they cannot be selected. Clicking a grayed-out cell has no effect.

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
- R5.2: **Claiming unclaimed territory:** The pixel becomes the attacker's color at the attacking strength. E.g., a 5-letter word (strength 3) on empty land → player's color at strength 3.
- R5.3: **Attacking opponent territory (attack < defense):** The defender keeps the pixel, but their strength is reduced. E.g., Blue at strength 3 attacked by Red at strength 2 → Blue remains at strength 1.
- R5.4: **Attacking opponent territory (attack = defense):** Strengths cancel out. The pixel becomes **purple (neutral)** at strength 0. Neither player scores for it.
- R5.5: **Attacking opponent territory (attack > defense):** The attacker captures the pixel. The new strength is attack minus defense. E.g., Blue at strength 2 attacked by Red at strength 4 → Red at strength 2.
- R5.6: **Claiming own territory (stacking):** Strengths **add together**, capped at 5. E.g., player owns a pixel at strength 2 and claims it with a 4-letter word (strength 2) → player's color at strength 4.
- R5.7: **Claiming neutral/purple territory:** The pixel becomes the attacker's color at the full attacking strength (purple has strength 0).
- R5.8: The visual rendering reflects all states: blue and red at varying intensities (5 levels), and purple for neutral pixels. Strength differences must be visually distinguishable.

### R6 - Scoring

- R6.1: Score = total number of pixels of the player's color on the territory canvas.
- R6.2: Blue and Red scores are displayed at the top of the screen, updated after each turn.
- R6.3: Purple pixels count for **neither** player.
- R6.4: At game end, the player with the higher score wins. If scores are equal, the game is a draw.

### R7 - UI & Display

- R7.1: The game board is centered on screen with a clean layout.
- R7.2: A header area shows: current player's turn, Blue score, Red score, turns remaining.
- R7.3: The currently-being-spelled word is displayed as text (e.g., "C-A-T") so players can see what they're forming.
- R7.4: Buttons: **Submit Word**, **Undo**, **Clear**, **Pass Turn**.
- R7.5: Feedback messages for: invalid word, word too short, API errors, successful territory claim.
- R7.6: At game end, a clear overlay/message declares the winner with final scores and a **New Game** button.

---

## Milestones

### Milestone 1 — Playable Board with Territory Claiming

**Goal:** Two players can take turns selecting letters, forming polygons, and visually claiming territory with strength-based shading. No word validation or scoring yet.

**Features:**
- Generate and render the 12x12 letter grid on a Canvas
- Click-to-select letters with visual highlights and connecting lines
- Undo and Clear selection functionality
- Submit closes the polygon and fills it with the current player's color (blue or red)
- Strength-based shading: longer words produce darker territory (5 visual intensity levels)
- Turn alternation between Blue and Red
- All words accepted (no dictionary validation yet)
- Minimum 3-letter requirement enforced
- Basic header showing whose turn it is

**Playable as:** Players take turns drawing polygons on the board. Territory appears at different intensities based on word length. No score tracking or overlap rules yet.

---

### Milestone 2 — Scoring, Overlap Rules & Dictionary Validation

**Goal:** Full game mechanics — validated words, strength-based territory conflict with subtraction model, live scoring, and turn limits.

**Features:**
- Dictionary API integration (validate words on submit)
- Feedback for invalid/valid words
- Full strength subtraction overlap model:
  - Attack < defense: defender weakened but holds
  - Attack = defense: territory becomes neutral (purple)
  - Attack > defense: attacker captures at remaining strength
  - Self-overlap stacks strength (capped at 5)
  - Claiming purple territory at full attack strength
- Pixel-level score counting displayed in the header
- Turn counter (10 turns each, 20 total)
- Pass turn button
- Collinear detection (accept but award zero territory)
- Game end condition after all turns with winner display

**Playable as:** A complete game with rules. Players form real words, contest and fortify territory using the strength system, and compete for the highest score over 16 turns.

---

### Milestone 3 — Polish & Quality of Life

**Goal:** Visual refinements, better UX, and edge-case handling for a finished product.

**Features:**
- Weighted letter distribution favoring common English letters
- Animated polygon fill (brief fade-in when territory is claimed)
- Preview of the polygon shape before submitting (dashed line from last selected letter back to first)
- Hover effects on grid letters
- Game-end overlay with final scores, winner banner, and New Game button
- Responsive layout (adapts to different window sizes)
- Loading indicator while dictionary API call is in progress
- Error handling for network failures (allow retry or accept on honor system)
- Turn history sidebar showing each word played and territory gained
- Sound effects (optional toggle) for letter selection, word accept/reject, and game end

**Playable as:** A polished, visually appealing game ready to share. Smooth interactions, clear feedback, and a satisfying game-end experience.
