# D.snake
Jeu Snake néon avec stickers et sons arcade
index.html
<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Destin snake</title>
<link rel="manifest" href="manifest.webmanifest">
<meta name="theme-color" content="#000000">
<link rel="apple-touch-icon" href="icon-192.png">
<style>
:root{--bg:#000;--fg:#fff;--muted:#8a8a8a;--btn:#1a1a1a;box-sizing:border-box;padding-top:env(safe-area-inset-top,0px);padding-bottom:env(safe-area-inset-bottom,0px)}
html{scroll-padding-top:env(safe-area-inset-top,0px)}
html,body{height:100%}
body{background:var(--bg);color:var(--fg);margin:0;overflow:hidden;display:flex;flex-direction:column;justify-content:center;align-items:center;gap:10px;font-family:system-ui,-apple-system,Arial,sans-serif;touch-action:none;user-select:none}
#score{font-weight:bold;font-size:18px}
#mute{position:absolute;top:calc(env(safe-area-inset-top,0px) + 8px);right:10px;font-size:20px;border:0;border-radius:8px;background:var(--btn);padding:6px 8px}
#wrap{position:relative}
canvas{display:block;background:#000;border:2px solid #ff4fd8;border-radius:6px;max-width:94vw;box-shadow:0 0 24px rgba(255,79,216,.55)}
#msg{position:absolute;inset:0;display:flex;flex-direction:column;align-items:center;justify-content:center;background:rgba(0,0,0,.8);color:#fff;border-radius:6px;text-align:center;cursor:pointer;font-weight:bold;gap:6px}
#msg small{font-weight:normal;color:#aaa}
#pad{display:grid;grid-template-columns:repeat(3,56px);grid-template-rows:repeat(2,56px);gap:6px}
#pad button{font-size:22px;border:0;border-radius:10px;background:var(--btn);color:var(--fg)}
#pad button:active{background:#ff4fd8;color:#000}
#up{grid-column:2}#left{grid-column:1;grid-row:2}#down{grid-column:2;grid-row:2}#right{grid-column:3;grid-row:2}
.hint{color:var(--muted);font-size:12px}
</style>
</head>
<body>
<div id="score">Score: 0</div>
<button id="mute" aria-label="Son">🔊</button>
<div id="wrap">
  <canvas id="canvas"></canvas>
  <div id="msg"><div>Destin snake</div><small>Touche pour jouer</small></div>
</div>
<div id="pad">
  <button id="up" aria-label="Haut">▲</button>
  <button id="left" aria-label="Gauche">◀</button>
  <button id="down" aria-label="Bas">▼</button>
  <button id="right" aria-label="Droite">▶</button>
</div>
<div class="hint">Flèches, ZQSD/WASD ou balayage · Espace = pause</div>
<script>
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const scoreEl = document.getElementById('score');
const msg = document.getElementById('msg');

const COLS = 20, ROWS = 20;
const CELL = Math.max(14, Math.min(24, Math.floor(Math.min(innerWidth * 0.94, innerHeight - 260) / COLS)));
canvas.width = COLS * CELL;
canvas.height = ROWS * CELL;
msg.style.width = canvas.width + 'px';
msg.style.height = canvas.height + 'px';

let snake, dir, queue, food, score, best = 0, timer, state = 'idle', lastScore = -1;
// Sons arcade (synthétisés, aucun fichier)
let actx = null, muted = false;
function beep(freq, dur, type = 'square', when = 0, slideTo = null, vol = 0.07) {
  if (muted) return;
  try {
    actx = actx || new (window.AudioContext || window.webkitAudioContext)();
    if (actx.state === 'suspended') actx.resume();
    const t = actx.currentTime + when, o = actx.createOscillator(), g = actx.createGain();
    o.type = type; o.frequency.setValueAtTime(freq, t);
    if (slideTo) o.frequency.exponentialRampToValueAtTime(slideTo, t + dur);
    g.gain.setValueAtTime(vol, t); g.gain.exponentialRampToValueAtTime(0.0001, t + dur);
    o.connect(g).connect(actx.destination); o.start(t); o.stop(t + dur);
  } catch (e) {}
}
const sfx = {
  start: () => [523, 659, 784, 1047].forEach((f, i) => beep(f, 0.1, 'square', i * 0.08)),
  eat: () => { beep(880, 0.07, 'square'); beep(1320, 0.1, 'square', 0.06); },
  die: () => { beep(440, 0.5, 'sawtooth', 0, 70, 0.09); beep(220, 0.5, 'square', 0.1, 50, 0.06); },
  turn: () => beep(300, 0.03, 'square', 0, null, 0.02)
};
document.getElementById('mute').addEventListener('pointerdown', e => {
  e.stopPropagation(); muted = !muted; e.currentTarget.textContent = muted ? '🔇' : '🔊';
});

const DIRS = { up: [0, -1], down: [0, 1], left: [-1, 0], right: [1, 0] };

function reset() {
  snake = [{ x: 10, y: 10 }, { x: 9, y: 10 }, { x: 8, y: 10 }];
  dir = DIRS.right; queue = []; score = 0;
  placeFood();
}

function placeFood() {
  const taken = new Set(snake.map(s => s.y * COLS + s.x));
  if (taken.size >= COLS * ROWS) { food = null; return; }
  let i;
  do { i = Math.floor(Math.random() * COLS * ROWS); } while (taken.has(i));
  food = { x: i % COLS, y: (i / COLS) | 0 };
}

function setDir(name) {
  const d = DIRS[name];
  const ref = queue.length ? queue[queue.length - 1] : dir;
  if (d[0] === -ref[0] && d[1] === -ref[1]) return; // pas de demi-tour
  if (d[0] === ref[0] && d[1] === ref[1]) return;
  if (queue.length < 2) { queue.push(d); sfx.turn(); }
}

function speed() { return Math.max(60, 140 - score * 3); }

function schedule() { clearTimeout(timer); timer = setTimeout(tick, speed()); }

function tick() {
  if (state !== 'play') return;
  if (queue.length) dir = queue.shift();
  const head = { x: snake[0].x + dir[0], y: snake[0].y + dir[1] };
  const eating = food && head.x === food.x && head.y === food.y;
  const body = eating ? snake : snake.slice(0, -1);
  if (head.x < 0 || head.y < 0 || head.x >= COLS || head.y >= ROWS ||
      body.some(s => s.x === head.x && s.y === head.y)) return gameOver();
  snake.unshift(head);
  if (eating) { score++; placeFood(); sfx.eat(); } else snake.pop();
  schedule();
}

function start() {
  reset(); state = 'play'; sfx.start(); msg.style.display = 'none'; schedule();
}

function gameOver() {
  state = 'over'; sfx.die();
  best = Math.max(best, score);
  msg.innerHTML = `<div>Perdu ! Score : ${score}</div><small>Record : ${best}</small><small>Touche pour rejouer</small>`;
  msg.style.display = 'flex';
}

function togglePause() {
  if (state === 'play') { state = 'pause'; clearTimeout(timer); msg.innerHTML = '<div>Pause</div><small>Touche pour reprendre</small>'; msg.style.display = 'flex'; }
  else if (state === 'pause') { state = 'play'; msg.style.display = 'none'; schedule(); }
}

// Fond animé : stickers qui flottent et tournent
const EMOJIS = ['🎮','👾','🕹️','⭐','🍕','🚀','🦄','🔥','💎','🍩','🎧','👻','🌈','🍒','⚡','🐸','🎲','🛸'];
const dots = Array.from({ length: 16 }, () => ({
  x: Math.random() * canvas.width, y: Math.random() * canvas.height,
  size: CELL * (1 + Math.random() * 1.2), v: 10 + Math.random() * 20,
  e: EMOJIS[(Math.random() * EMOJIS.length) | 0],
  rot: Math.random() * 6.28, spin: (Math.random() - 0.5) * 1.2, p: Math.random() * 6.28
}));
let prev = performance.now();
const CYCLE = [[47, 107, 255], [255, 225, 0], [255, 40, 40]]; // bleu, jaune, rouge
function borderColor(now) {
  const t = (now / 1200) % CYCLE.length, i = Math.floor(t), f = t - i;
  const a = CYCLE[i], b = CYCLE[(i + 1) % CYCLE.length];
  return `rgb(${a.map((v, k) => Math.round(v + (b[k] - v) * f)).join(',')})`;
}

function draw(now = performance.now()) {
  const { width, height } = canvas;
  const half = CELL / 2, dt = Math.min(0.05, (now - prev) / 1000); prev = now;

  ctx.shadowBlur = 0;
  ctx.fillStyle = '#000';
  ctx.fillRect(0, 0, width, height);

  // halo central qui respire
  const pulse = 0.5 + 0.5 * Math.sin(now / 900);
  const g = ctx.createRadialGradient(width / 2, height / 2, 0, width / 2, height / 2, width * 0.7);
  g.addColorStop(0, `rgba(255,0,60,${0.10 + 0.08 * pulse})`);
  g.addColorStop(1, 'rgba(0,0,0,0)');
  ctx.fillStyle = g;
  ctx.fillRect(0, 0, width, height);

  // stickers
  ctx.textBaseline = 'middle';
  ctx.textAlign = 'center';
  for (const d of dots) {
    d.y -= d.v * dt; d.rot += d.spin * dt;
    if (d.y < -d.size) { d.y = height + d.size; d.x = Math.random() * width; d.e = EMOJIS[(Math.random() * EMOJIS.length) | 0]; }
    ctx.save();
    ctx.globalAlpha = 0.28 + 0.1 * Math.sin(now / 700 + d.p);
    ctx.translate(d.x + Math.sin(now / 1500 + d.p) * 8, d.y);
    ctx.rotate(d.rot);
    ctx.font = `${d.size}px sans-serif`;
    ctx.fillText(d.e, 0, 0);
    ctx.restore();
  }
  ctx.textBaseline = 'alphabetic';

  // titre discret (rouge vif, faible opacité)
  ctx.shadowBlur = 0;
  ctx.fillStyle = 'rgba(255, 26, 26, 0.16)';
  ctx.font = `bold ${CELL * 1.8}px Arial`;
  ctx.textAlign = 'center';
  ctx.fillText('<Destin snake>', width / 2, height / 2);

  // point vert lumineux (pulse)
  if (food) {
    ctx.shadowColor = '#39ff14';
    ctx.shadowBlur = 18 + 10 * pulse;
    ctx.fillStyle = '#39ff14';
    ctx.beginPath();
    ctx.arc(food.x * CELL + half, food.y * CELL + half, CELL / 2.2 * (0.92 + 0.08 * pulse), 0, Math.PI * 2);
    ctx.fill();
  }

  // serpent bleu, bordure qui cycle bleu / jaune / rouge
  const size = CELL - 2;
  const bc = borderColor(now);
  ctx.shadowColor = bc;
  ctx.strokeStyle = bc;
  ctx.lineWidth = 2;
  for (let i = snake.length - 1; i >= 0; i--) {
    const x = snake[i].x * CELL + 1, y = snake[i].y * CELL + 1;
    ctx.shadowBlur = i === 0 ? 18 : 12;
    ctx.fillStyle = i === 0 ? '#4d8dff' : '#1f5cff';
    ctx.fillRect(x, y, size, size);
    ctx.strokeRect(x + 1, y + 1, size - 2, size - 2);
  }
  ctx.shadowBlur = 0;

  if (score !== lastScore) { scoreEl.textContent = `Score: ${score}`; lastScore = score; }
}

reset();
(function loop(t) { draw(t); requestAnimationFrame(loop); })(performance.now());

// Contrôles
const KEYS = { ArrowUp:'up', w:'up', z:'up', ArrowDown:'down', s:'down', ArrowLeft:'left', a:'left', q:'left', ArrowRight:'right', d:'right' };
addEventListener('keydown', e => {
  if (e.key === ' ') { e.preventDefault(); state === 'play' || state === 'pause' ? togglePause() : start(); return; }
  const k = KEYS[e.key.length === 1 ? e.key.toLowerCase() : e.key];
  if (!k) return;
  e.preventDefault();
  if (state !== 'play') { if (state !== 'pause') start(); return; }
  setDir(k);
});
for (const id of ['up', 'down', 'left', 'right']) {
  document.getElementById(id).addEventListener('pointerdown', e => {
    e.preventDefault();
    if (state === 'play') setDir(id); else if (state !== 'pause') start();
  });
}
msg.addEventListener('pointerdown', () => state === 'pause' ? togglePause() : start());

let tx, ty;
addEventListener('touchstart', e => { tx = e.touches[0].clientX; ty = e.touches[0].clientY; }, { passive: true });
addEventListener('touchend', e => {
  if (tx == null || state !== 'play') return;
  const dx = e.changedTouches[0].clientX - tx, dy = e.changedTouches[0].clientY - ty;
  if (Math.max(Math.abs(dx), Math.abs(dy)) < 24) return;
  setDir(Math.abs(dx) > Math.abs(dy) ? (dx > 0 ? 'right' : 'left') : (dy > 0 ? 'down' : 'up'));
  tx = null;
}, { passive: true });

if ('serviceWorker' in navigator) addEventListener('load', () => navigator.serviceWorker.register('sw.js').catch(() => {}));
</script>
</body>
</html>
manifest.webmanifest
{
  "name": "Destin snake",
  "short_name": "Snake",
  "start_url": "./index.html",
  "scope": "./",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#000000",
  "theme_color": "#000000",
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
sw.js
const CACHE = 'destin-snake-v1';
const FILES = ['./', './index.html', './manifest.webmanifest', './icon-192.png', './icon-512.png'];
self.addEventListener('install', e => { e.waitUntil(caches.open(CACHE).then(c => c.addAll(FILES))); self.skipWaiting(); });
self.addEventListener('activate', e => { e.waitUntil(caches.keys().then(k => Promise.all(k.filter(n => n !== CACHE).map(n => caches.delete(n))))); self.clients.claim(); });
self.addEventListener('fetch', e => { e.respondWith(caches.match(e.request).then(r => r || fetch(e.request))); });
