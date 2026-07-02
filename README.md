# 🗼 VOID TIDE

> **Hold the light. Survive the tide.**

A serverless, peer-to-peer, isometric co-op twin-stick shooter in a **single HTML file**. You are a lighthouse keeper on a lonely island in a dark sea. Creatures of the void tide surface offshore, swim for your island, crawl onto the beach, and lay siege to the beacon at its heart. Survive **10 tides** and dawn breaks. Lose the beacon — or every keeper — and the tide claims all.

Built with **Phaser 3** + **PeerJS**. No build step, no server, no assets — every visual is procedural vector art drawn with canvas primitives.

---

## 🚀 Quick Start

### Play locally

```bash
# any static file server works
python -m http.server 8137
# or
npx http-server -p 8137
```

Open `http://localhost:8137` — that's it.

> ⚠️ Serve over `http(s)://`, not `file://` — PeerJS signaling and the clipboard API behave best in a proper origin.

### Play with a friend

1. **Host**: open the game, click **PLAY SOLO / START HOSTING** (or just wait — the match auto-starts when someone joins you)
2. Copy your connection code (`void-tide-xxxxxx`) and send it to a friend
3. **Guest**: paste the code, click **CONNECT TO HOST**

The connection is direct browser-to-browser (WebRTC). The public PeerJS cloud is used only for the initial handshake. For play over the internet, host the file on any static HTTPS host (GitHub Pages works perfectly).

---

## 🎮 Controls

| Action  | PC                  | Mobile                             |
|---------|---------------------|-------------------------------------|
| Move    | `WASD` / arrow keys | Left-thumb virtual joystick         |
| Aim     | Mouse               | Right-thumb touch (anywhere right)  |
| Fire    | Left click (hold)   | Held automatically while aiming     |
| Reload  | `R`                 | On-screen RELOAD button             |
| Restart | `Space` (game over) | RESTART MATCH button                |

---

## 🌊 How It Plays

- **Defend the Beacon** — the lighthouse at the island's center has 600 HP. If it falls, you lose *even if you're alive*. It mends +60 HP after every cleared tide.
- **The light fights with you** — the lighthouse's twin beams sweep the island and **slow any creature caught in the light by 50%**.
- **Creatures swim ashore** — spawns surface offshore and swim in half-submerged. Snipe them in the water, but sea kills sink their loot.
- **Two creature types** — fast void-green **wisps** and hulking violet **dreadmaws** (dreadmaws always march on the beacon).
- **Pickups** — kills on land can drop **embers** (+25 HP), **ammo caches** (+30 reserve), and **power sigils** (8s of RAPID FIRE or SPREAD SHOT).
- **Combo multiplier** — chain kills within 3 seconds to build up to ×5 score. *RAMPAGE. UNSTOPPABLE.*
- **Co-op respawns** — if your partner is still standing, you get back up after 6 seconds.

---

## 🏗️ Architecture

Everything lives in [`index.html`](index.html) (~1,600 lines):

```
HOST BROWSER (Player 1)                    GUEST BROWSER (Player 2)
+---------------------------------+        +---------------------------------+
| - Runs the authoritative sim    |        | - Sends inputs @ 20 Hz          |
|   (AI, collisions, tides,       |        |   (move vector, aim, fire)      |
|    beacon, pickups, combo)      |        | - Predicts own movement locally |
| - Broadcasts snapshots @ 15 Hz  |        | - Interpolates all other        |
|                                 |        |   entities from snapshots       |
+---------------------------------+        +---------------------------------+
                |                                          ^
                |  [game-state snapshots + fx events]      |
                v                                          |
                +============ PeerJS WebRTC data channel ==+
                                                           |
                +<=========== [input structs] =============+
```

Key design points:

- **Host-authoritative**: the host simulates everything; the guest is a renderer with client-side prediction + reconciliation for its own player.
- **World-space isometric renderer**: logical grid coords → world pixels via the classic `screenX = (x−y)·W½`, `screenY = (x+y)·H½` transform; a smooth-follow Phaser camera (with aim look-ahead, zoom, and shake) does the rest.
- **fx event stream**: one-shot effects (explosions, splashes, muzzle flashes, score popups) are queued as small numbered events, run instantly on the host, and shipped to the guest inside the next snapshot.
- **100% procedural rendering**: island terrain, water, creatures, lighthouse, particles, and HUD are all drawn per-frame with `Phaser.GameObjects.Graphics` — zero image/audio assets. Sound is a tiny WebAudio oscillator synth.

The full original design document is in [`isometric_game_specification.md`](isometric_game_specification.md).

---

## 🛠️ Tech Stack

| Layer      | Choice                                   |
|------------|------------------------------------------|
| Engine     | [Phaser 3.80](https://phaser.io/) (CDN)  |
| Networking | [PeerJS 1.5](https://peerjs.com/) (CDN)  |
| Audio      | Web Audio API (procedural synth)         |
| Assets     | None — pure vector graphics              |
| Build      | None — one HTML file                     |

---

## 📦 Deployment

Any static host works. For GitHub Pages:

1. Push this repo to GitHub
2. Settings → Pages → deploy from the `main` branch
3. Share the URL — HTTPS is required for reliable WebRTC on mobile browsers

---

*Made with Claude Code. The tide rises. Hold the light.* 🗼
