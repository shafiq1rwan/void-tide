# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

**Void Tide** — a single-file P2P isometric co-op twin-stick shooter (Phaser 3 + PeerJS). The entire game is `index.html`; there is no build step, no package.json, no assets. `isometric_game_specification.md` is the original design doc (kept for reference; the theme has since evolved from "Isometric Vector Strike" to Void Tide).

## Commands

```bash
# serve (required — file:// breaks PeerJS/clipboard)
python -m http.server 8137

# syntax-check the inline JS after editing (the script is wrapped in parse markers)
node -e "
const m = require('fs').readFileSync('index.html','utf8')
  .match(/\/\/ ===JS-START===([\s\S]*)\/\/ ===JS-END===/);
new Function(m[1]); console.log('JS parses OK');"
```

There are no automated tests. Verify changes by opening two browser tabs (host + guest) and playing a tide.

## File layout (all inside index.html)

1. `<style>` — lobby / game-over overlay CSS (deep-sea palette, amber `#ffd98a` lighthouse accents)
2. HTML overlays — `#lobby` (screen 1), `#gameover` (screen 3), `#reloadBtn` (touch only)
3. `<script>` between `// ===JS-START===` and `// ===JS-END===` markers, organized by comment banners:
   - CONFIG — tuning constants (`GRID`, `MAXWAVE`, `BEACON_MAX`, speeds, rates)
   - AUDIO — tiny WebAudio oscillator synth (`sfx(name)`)
   - PEERJS NETWORKING — connection lifecycle + message router `onData`
   - GAME MODEL (HOST) — `freshGame`, `spawnEnemy`, `applyInput`, `updateHost` (the authoritative sim)
   - SNAPSHOTS — `buildSnapshot` / `applySnapshot`
   - PARTICLES & VFX — `PARTS`, `FLOATS`, `runFx`
   - PHASER SCENE — `Main`: input, camera, all rendering, HUD

## Architecture invariants — do not break these

- **Host-authoritative**: only `updateHost` mutates game state (`APP.G`). The guest renders proxies (`APP.R`) interpolated from snapshots, with local prediction + reconciliation for its own player only (`guestPredict`).
- **Every gameplay-visible field must ride the snapshot.** If you add state the guest needs (new entity type, player flag, etc.), update *both* `buildSnapshot` and `applySnapshot`, and keep keys terse (bandwidth: 15 Hz).
- **One-shot effects go through the fx queue**, not ad-hoc calls: `pushFx(G, k, x, y, extra)` runs the effect immediately on the host *and* queues it for the guest's next snapshot. fx kinds: `1` boom/kill (+`s` score popup, `c` color), `2` player hit (`pi`), `3` muzzle (`a` angle), `4` bullet spark, `5` pickup (`pi`, `pw`), `6` beacon hit, `7` shore splash. Add new kinds as numbers in `runFx`.
- **Coordinates**: logic runs on a flat `GRID×GRID` (14×14) float grid; island center `(GRID/2, GRID/2)` maps to world `(0,0)` via `isoX/isoY`; `unIso` inverts. Positions **outside** `[0, GRID]` are open water (swimming enemies live there). Player movement clamps with `clampGrid`; enemies only clamp once landed (`e.sw === 0`).
- **Camera**: the Phaser main camera provides zoom/pan/shake. World drawing uses world coords (scrollFactor 1); the HUD graphics + texts are pinned with `setScrollFactor(0)` and positioned in screen coords (`V.w`/`V.h`). Screen→logical conversion must go through `camera.getWorldPoint` then `unIso`.
- **Rendering is 100% procedural** (`Phaser.GameObjects.Graphics` redrawn per frame + a pooled text array for score floaters). Never introduce image/audio assets or additional CDNs — single-file, zero-asset is the core constraint.
- **Roles converge, code paths split**: host draws from `APP.G` (`drawWorldHost`), guest from `APP.R` (`drawWorldGuest`); both funnel into `drawEntities(...)`. HUD reads from whichever model matches `APP.role` — keep both branches updated.

## Conventions

- Plain ES2017-ish JavaScript, no modules, no TypeScript. Two-space indent, section comment banners (`// ===== NAME =====`), terse single-letter snapshot keys.
- Colors live in the `COL` table; theme palette is deep-sea blues + lighthouse amber (`0xffd98a`). Player 1 is always blue, player 2 red (minimap legend contract from the spec).
- Timers count down in seconds; the frame delta is clamped (`dt ≤ 0.05`).
- Touch input: pointer ownership is tracked by id in `TOUCH` (left 44% of screen = joystick, rest = aim/fire). Test any input change against multi-touch.

## Gotchas

- `Peer` ids are `void-tide-` + 6 chars; on `unavailable-id` the code regenerates recursively — keep that path intact.
- The guest never hears `sfx` for its own gameplay events directly; sounds arrive via fx events (≈70 ms later). Local-only sounds (reload click) are triggered in `gatherInput`.
- The grid/island terrain is drawn **once** (`this.gridDrawn` flag) into `gridGfx`; animated water lives in `drawWater` (per-frame). If you change terrain, remember it won't redraw until reload.
- `applyInput` runs on the host for *both* players (guest inputs arrive as `APP.remIn`); don't reference `APP.myIndex` inside it for gameplay decisions.
- Accuracy can exceed 100% with SPREAD (3 bullets per counted shot) — display paths clamp with `Math.min(100, ...)`; keep that when touching stats.
- After any edit, run the node parse check above — a syntax error produces a silent black screen in the browser.
