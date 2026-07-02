# Isometric Multiplayer Vector Shooter (Phaser 3 + PeerJS)
## Complete Game Specification & Architecture Guide

This document defines the comprehensive scope, core features, required interface screens, and mathematical specifications for building a single-file multiplayer isometric shooting game. It serves as a blueprint for Claude or any LLM to generate the entire code structure in a clean, maintainable format using **Phaser 3** and **PeerJS**.

---

## 1. Game Overview & Scope

* **Genre:** Twin-stick Isometric Action Shooter (Multiplayer).
* **Perspective:** True Isometric view ($45^{\circ}$ rotation, $30^{\circ}$ tilt).
* **Architecture:** Single HTML file using standard CDNs for zero-setup, serverless deployment.
* **Networking:** Serverless Peer-to-Peer (P2P) WebRTC architecture managed by PeerJS. One browser runs as **Host** (coordinates game state, enemies, collisions), and one browser runs as **Guest** (sends input, interpolates remote entities).
* **Visual Direction:** Clean, crisp, procedural vector art drawn entirely using canvas-native graphics primitives (`Phaser.GameObjects.Graphics`).

---

## 2. Core Features (The Must-Haves)

### A. Serverless P2P Matchmaking
* **No Dedicated Server Setup:** Uses the free, public PeerJS cloud server solely for signaling and swapping connection handshakes.
* **Unique Peer ID:** Every session generates an alphanumeric ID (e.g., `iso-shoot-83a9`).
* **Direct Pipeline:** Once connected, data channel transmission transfers binary/JSON payloads with low latency over direct WebRTC data pipes.

### B. Two-Dimensional Isometric Vector Grid
* **Logical vs. Screen Coordinates:** The game separates the gameplay logic (a flat 2D grid array) from spatial layout rendering (the visual diamond perspective).
* **Tile Boundaries:** Explicit grid boundaries prevent entities from walking off the diamond plane.

### C. Twin-Stick Mechanics & Vector Target Lines
* **Decoupled Movement & Aiming:** Players move along the diagonal tile axes using `W`, `A`, `S`, `D` but aim independently using the mouse pointer anywhere on the screen plane.
* **Laser Sight:** A continuous, clean vector line draws from the player's weapon center directly to the converted crosshair target point to help players compensate for isometric distortion.

### D. Host-Authoritative Entity Synchronization
* **Host Responsibilities:** Spawns and drives AI enemy paths, calculates projectile-to-hitbox collisions, and broadcasts complete snapshot arrays.
* **Guest Responsibilities:** Listens to snapshot arrays, smoothly updates local proxies, captures user inputs (`WASD` state + mouse vector data), and updates the host at a high refresh frequency.

---

## 3. Required User Interface Screens

To function as a complete game, the single-file application flows sequentially through three visual UI layer states managed via HTML overlays or separate Phaser Canvas states:

### Screen 1: The Connection & Matchmaking Lobby
* **Visual State:** Plain HTML modal form overlaid precisely on top of an uninitialized canvas canvas background.
* **Elements:**
    * **Title Header:** "Isometric Vector Strike" (Stylized block font).
    * **Your Connection Code:** A read-only text field displaying the generated Peer ID with a "Copy to Clipboard" utility button.
    * **Join Session Input:** A text input box to paste a friend's Peer ID.
    * **Action Buttons:** A primary button labeled "Connect to Host" and a fallback button "Play Solo / Start Hosting".
    * **Status Text Box:** Live messaging field updating from `"Initializing PeerJS..."` to `"Waiting for peer..."` or `"Connection Established!"`.

### Screen 2: Main Tactical Gameplay Canvas
* **Visual State:** Full viewport HTML5 canvas rendering the true isometric game engine and the overlapping HUD layers described in Section 5.
* **Elements:**
    * Active map grid, player entities, active projectile paths, enemy path tracers, and dynamic explosion particle lines.

### Screen 3: Game Over / Match Summary State
* **Visual State:** Semi-transparent dark overlay fade directly over the final frame of the game canvas.
* **Elements:**
    * **Match Result Header:** "VICTORY" or "DEFEAT".
    * **Performance Metrics:** Displays Total Waves Cleared, Enemies Destroyed, and Accuracy Percentages.
    * **Reset Tool:** A prominent "Restart Match" button that clears entity containers, resets health thresholds, and syncs the new game state between connected peers.

---

## 4. Architectural & Mathematical Foundations

### A. Coordinate Transformation Math
To transform logical vector data positions $(X, Y)$ into isometric display positions on the screen $(Screen_X, Screen_Y)$, use the following mathematical conversions:

$$Screen_X = (X - Y) \times W_{half} + Center_X$$

$$Screen_Y = (X + Y) \times H_{half} + Center_Y$$

*Where:*
* $X, Y$: Flat, logical float tracking units on the map grid.
* $W_{half}, H_{half}$: Semicolon boundaries defining tile geometric scaling (typically $W=64$, $H=32$, so $W_{half}=32$, $H_{half}=16$).
* $Center_X, Center_Y$: Layout displacement vectors centering the coordinate origin on the middle of the viewport.

### B. Network Topology (Host vs Guest State Matrix)

```
        HOST BROWSER (Player 1)                   GUEST BROWSER (Player 2)
   +---------------------------------+       +---------------------------------+
   | - Computes AI Enemy Paths       |       | - Captures WASD Keys & Mouse Pos|
   | - Resolves Projectile Collisions|       | - Fires Local Input Structs     |
   | - Manages Game Wave Cycle       |       | - Interpolates Proxy Positions  |
   +---------------------------------+       +---------------------------------+
                   |                                         ^
                   | [Sends Game State Snapshot]             |
                   v                                         |
                   +================= (PeerJS WebRTC) =======+
                                                             |
                   +<================ [Sends Inputs / Pos] ===+
```

---

## 5. Complete HUD Layout Requirements

The Heads-Up Display must be rendered cleanly over the canvas layer using crisp geometric bars, flat fills, and highly legible monochromatic text fields.

```
+-------------------------------------------------------------------------+
| [HP/SHIELD METER]                                            [MINIMAP]  |
| Health [██████████░░░] 85%                                    +-------+ |
| Shield [█████████████] 100%                                   | .   o | |
|                                                               |   x   | |
|                                                               +-------+ |
|                                                                         |
|                                                                         |
|                                                                         |
|                                                                         |
| [NETWORK MONITOR]                                         [AMMO/STATS]  |
| Ping: 32ms                                                Wave: 03      |
| Role: HOST                                                Score: 01450  |
| Connected Peers: 1                                        Ammo: 30 / 90 |
+-------------------------------------------------------------------------+
```

### HUD Component Breakdown:
1.  **Top-Left Asset: Health & Shield Core Module**
    * A clean horizontal layout. Uses a thick red container block for health and a secondary electric-cyan block directly below it for defensive armor/shields.
    * Overlaid with digital text percentage readings.
2.  **Top-Right Asset: Minimalist Vector Mini-map**
    * A cropped circular or square outline frame rendering a scaled-down wireframe representation of the grid.
    * **Legend Indicators:** A distinct bright blue vector dot for Player 1, a deep red dot for Player 2, and flashing white dots tracking active enemy positions.
3.  **Bottom-Right Asset: Weapon Systems & Wave Statistics**
    * **Ammunition Multi-slot:** High-visibility layout text (e.g., `AMMO: 30 / 90`) indicating rounds in the current magazine versus remaining reserve stocks.
    * **Score & Progression Data:** Tracks running metrics (`SCORE: 01450`) and global progression indexes (`WAVE: 03`).
4.  **Bottom-Left Asset: Real-Time Network Diagnostic Diagnostics**
    * A small, low-impact telemetry readout showing immediate network link health: `PING: 32ms`, local session validation status (`ROLE: HOST` or `ROLE: GUEST`), and peer payload verifications.
