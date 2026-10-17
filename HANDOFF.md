# Handoff: Pip's Big Leap

## 1. Project goal

The goal is a fun, Mario-style side-scrolling platformer that plays well on an **iPad**, with touch or with a Bluetooth mouse and keyboard. It is an original game, so it uses no Nintendo names or art. The longer-term idea is a website where people can visit and play this game and others like it.

- **Repo:** `NeonNinjaX2/The-Attack`. The work is on branch `claude/lucid-euler-etzz4f`, which is pushed but **not yet merged into `main`**.
- **Playable link (private claude.ai artifact):** https://claude.ai/artifact/FrCN35gCvQTVk7wCKr3inc
- **Files:**
  - `index.html` is the entire game in one file of about 1,320 lines.
  - `README.md` covers controls and how to play.
  - `HANDOFF.md` is this file.

### Answers to the "games website" questions

1. **What kind of games?** Browser games written in plain HTML5 Canvas and JavaScript, with no engine and no build step. They are not Unity, Godot or Scratch, and nothing needs to be downloaded.
2. **Do they already exist?** Yes. Pip's Big Leap is finished and playable. Any more games would be made the same way.

## 2. What's in the game

- **3 worlds**, each with its own tiles, parallax background and music:
  - Sunny Meadow (188 tiles wide)
  - Crystal Caverns (196 tiles wide, stone ceiling)
  - Lava Keep (198 tiles wide; lava, fire bars, moving platforms)
- **Hero:** Pip, a kid in an orange hood. Pip has three forms:
  - small
  - big, which can break bricks
  - fire, which can throw fireballs
- **Enemies:**
  - blob: walks off ledges, can be stomped
  - beetle: turns at ledges; stomping it leaves a shell you can kick, and a kicked shell knocks out other enemies
  - bee: hovers and bobs, can be stomped
  - spiky: can't be stomped, only hit with fire
- **Items and level objects:**
  - `?` blocks give coins; some `?` blocks give power-ups instead
  - Puffcap mushroom makes Pip big; Ember Bloom flower gives fire
  - springs, moving platforms (horizontal and vertical), fire bars, lava
  - a checkpoint flag halfway through each level, and a goal flagpole leading to a castle with fireworks
- **Scoring:**
  - stomp combos (100 → 8000 points, then a 1UP)
  - a 1UP every 100 coins
  - flagpole height bonus
  - best score saved on the device
- **Kid-friendly rules:**
  - 5 lives
  - a hit while in fire form drops Pip to big, not small
  - checkpoint respawns
  - **Try Again** restarts the current world with 5 lives
- **Saved on the device** (`localStorage` keys prefixed `pip.`): unlocked worlds, best score, mute setting.
- **Audio:** everything is generated live with Web Audio, so there are no audio files:
  - sound effects
  - a step sequencer playing 3 original chiptune loops
  - sound starts on the first tap or keypress

## 3. Controls

| Action | Touch | Keyboard | Mouse |
|---|---|---|---|
| Move | ◀ ▶ pad, bottom left | ← → / A D | none |
| Jump (hold for higher) | **A** | Space / ↑ / W / Z / K | none |
| Run (hold) / throw fire (tap) | **B** | X / Shift / J | **Left click** |
| Pause | ⏸ top right | P / Esc | none |
| Mute | Pause menu | M | none |

## 4. Key architectural decisions

1. **One self-contained HTML file.** The only external resource is Google Fonts: Press Start 2P for pixel text and Fredoka for menus. All sprites are drawn in code, and no images are loaded. The file has no `<html>`, `<head>` or `<body>` tags because the claude.ai artifact host wraps it in a page. Browsers also accept it as-is when it's opened directly.
2. **Low-resolution canvas, scaled up.** The game is drawn at 15 tiles × 16 px = **240 px tall**. The width follows the screen shape, between 256 and 448 px. CSS then scales it up with `image-rendering: pixelated` for crisp pixels.
3. **Fixed 60 Hz game updates**, independent of screen refresh rate, so it runs at the same speed on 120 Hz iPads.
4. **Sprites are text grids**, one character per pixel, turned into images at startup. Fire form reuses the same sprites with different colors. Big Pip is built from the small sprite with its body rows repeated, not drawn separately.
5. **Levels are built with small helper functions**, not hand-typed character maps: `ground`, `pipe`, `stairs`, `row`, `coins`, `lava`, `e(...)` for entities, and `goal`. The grid is 15 rows. The ground top is row 13, and anything standing on the ground is placed at `y = 12`.
6. **Physics tuned so levels are always beatable.** A full jump is about 4–5 tiles high and covers about 5 tiles across at walking speed. **Level rule:** gaps ≤ 4 tiles, step-ups ≤ 4 tiles, and pipe height ≤ 3.
7. **Collision is boxes against the tile grid**, resolving X first, then Y. `=` tiles are one-way platforms you can jump up through. Moving platforms are entities that carry the player along.
8. **Input comes from three sources, all mapped to `left/right/jump/b`:**
   - keyboard
   - touch: each finger is tracked against button zones with extra padding, so sliding a thumb between ◀ and ▶ works
   - mouse: left click is B
9. **Full screen and mouse capture** are requested on Play, Try Again and Resume. Both are optional because some browsers refuse them. If the game loses focus or the mouse capture, it **auto-pauses** so stray clicks don't kill Pip.
10. **Game states:** `title` → `intro` → `play` → `dying` / `flag` → `over` / `won`. A `paused` flag sits on top of these, and `freezeT` briefly freezes the game while Pip grows or shrinks.

## 5. Finalized code snippets (excerpts; `index.html` is the source of truth)

**Level builder example (World 1, start):**
```js
function level1() {
  const L = newLevel(188, 'meadow');
  L.ground(0, 40);
  L.set(9, 9, '?');
  L.row(13, 9, 'B?BMB');          // B brick, ? coin block, M power-up block
  L.e('blob', 20, 12);
  L.pipe(23, 2); L.pipe(31, 3);
  L.coins(40, 44, 9);              // coins over a 3-tile gap
  L.ground(44, 72);
  L.e('spring', 69, 12);
  L.e('checkpoint', 94, 12);
  L.stairs(162, 8);
  L.goal(177);                     // flagpole + castle
  return L;
}
// Tile chars: # ground, B brick, S stone, ? coin block, M power-up block, U used,
// [ ] { } pipe parts, = one-way platform, L lava, o coin, . empty
// Entities: blob, beetle, spiky, bee, spring, plat {w,dx|dy,period}, firebar {len,speed}, checkpoint, flag, castle
```

**Player physics (the feel of the game):**
```js
const maxS = input.b ? 2.8 : 1.9, acc = P.onGround ? 0.13 : 0.09;
if (input.jumpP) P.jumpBuf = 7;              // jump buffering
if (P.onGround) P.coyote = 6;                // coyote time
if (P.jumpBuf > 0 && P.coyote > 0) P.vy = -(6.0 + Math.abs(P.vx) * 0.3);
P.vy = Math.min(P.vy + (P.vy < 0 && input.jump ? 0.28 : 0.62), 7); // hold = higher jump
```

**Tile collision (X first, then Y; one-way platforms):**
```js
function moveBody(e, dx, dy) { /* move X, push out of solid tiles; move Y,
  land on solid or '=' if previous bottom was above it; report head hits (hx, hy) */ }
```

**Mouse = B button, with clicks latched so a fast click is never missed:**
```js
let mouseB = false, mouseClick = false;
document.addEventListener('pointerdown', e => {
  if (e.pointerType !== 'mouse' || e.button !== 0) return;
  if (e.target.closest && e.target.closest('.overlay, #pauseBtn')) return;
  e.preventDefault(); initAudio(); tapFlag = true; mouseB = true; mouseClick = true;
  if (state === 'play' || state === 'intro') lockPointer();
});
// in readInput():
input.bP = (input.b && !prevB) || mouseClick; mouseClick = false;
```

**Full screen, mouse capture and auto-pause:**
```js
function immerse() { goFullscreen(); lockPointer(); }   // called on Play / Try Again / Resume
document.addEventListener('pointerlockchange', () => {
  if (!document.pointerLockElement && !paused && (state === 'play' || state === 'intro')) togglePause();
});
addEventListener('blur', () => { mouseB = false; if (!paused && (state === 'play' || state === 'intro')) togglePause(); });
```

## 6. Testing approach

The game was tested with **headless Chromium driven by Playwright**. Each test loaded `index.html`, clicked Play, held keys or clicked the mouse, then read the game state and checked for page errors and screenshots. Tests used a **temporary copy** of the file with debug hooks added (for example, one that forces fire form), so the real file has no debug code. All three worlds rendered with no errors. Nobody has played the whole game end to end yet, and it hasn't been checked on a real iPad.

## 7. Open items / next steps

- [ ] Merge `claude/lucid-euler-etzz4f` into `main`, then turn on **GitHub Pages**. Then use Safari → Share → **Add to Home Screen** on the iPad for real full-screen play. This is the most reliable fix for clicking out.
- [ ] Build the **games website**: a landing page that links to each game, with each game in its own folder, all hosted on GitHub Pages.
- [ ] Ideas offered but not built:
  - a boss fight at the end of Lava Keep
  - more worlds
  - two-player mode
  - choosing a character
  - letting Pip shoot without the power-up (with a cooldown)
- [ ] Play-test the full game on the iPad and tune any jumps that are too hard.

## 8. Recommended starting prompt

> I'm continuing work on **Pip's Big Leap**, an original Mario-style platformer I play on an iPad, usually with a mouse and keyboard. The whole game is one self-contained file, `index.html` (HTML5 Canvas + JavaScript, no libraries, no build step), and `HANDOFF.md` describes how it works. Both are attached to this project. Please read them first.
>
> Ground rules:
> - Keep everything in a single HTML file with no external dependencies except the Google Fonts already used. All sprites are text pixel-grids drawn in code, and all audio is synthesized with Web Audio.
> - The game must work with touch, keyboard and mouse (left click = run/fire). It must stay full screen and auto-pause when it loses focus.
> - Keep levels beatable: gaps ≤ 4 tiles, step-ups ≤ 4 tiles, pipes ≤ 3 high. Build levels with the existing helpers (`ground`, `pipe`, `stairs`, `row`, `coins`, `lava`, `e`, `goal`).
> - Keep the style original. Don't use Nintendo names or characters.
> - When you change the game, give me the complete updated `index.html` so I can replace the file. Tell me in plain language what changed and how to test it on my iPad.
>
> My next goal: [e.g. "build a simple website that lists my games and links to Pip's Big Leap, ready for GitHub Pages" / "add a boss fight at the end of World 3"].
