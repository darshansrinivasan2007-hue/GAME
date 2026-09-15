<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Sky Hopper</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  html, body {
    height: 100%;
    overflow: hidden;
    font-family: 'Trebuchet MS', 'Segoe UI', sans-serif;
    background: linear-gradient(180deg, #7ecbff 0%, #a0e8ff 40%, #ffe9a8 100%);
  }
  #game-wrap {
    position: relative;
    width: 100%;
    height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  canvas {
    display: block;
    border-radius: 16px;
    box-shadow: 0 12px 40px rgba(0,0,0,0.25);
    background: linear-gradient(180deg, #7ecbff 0%, #a0e8ff 60%, #ffe9a8 100%);
    touch-action: none;
    cursor: pointer;
  }
  #hud {
    position: absolute;
    top: 24px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 40px;
    font-weight: 900;
    color: #fff;
    text-shadow: 0 3px 0 rgba(0,0,0,0.2), 0 0 8px rgba(0,0,0,0.15);
    pointer-events: none;
    user-select: none;
    letter-spacing: 2px;
  }
  #best {
    position: absolute;
    top: 72px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 15px;
    font-weight: 700;
    color: #fff;
    text-shadow: 0 2px 0 rgba(0,0,0,0.2);
    pointer-events: none;
    user-select: none;
    opacity: 0.9;
  }
  #overlay {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
    color: #fff;
    background: rgba(30, 20, 60, 0.35);
    backdrop-filter: blur(2px);
    border-radius: 16px;
    pointer-events: none;
  }
  #overlay h1 {
    font-size: 44px;
    text-shadow: 0 4px 0 rgba(0,0,0,0.25);
    margin-bottom: 10px;
  }
  #overlay p {
    font-size: 18px;
    font-weight: 600;
    text-shadow: 0 2px 0 rgba(0,0,0,0.2);
    margin-bottom: 4px;
  }
  #overlay .hint {
    margin-top: 18px;
    font-size: 15px;
    opacity: 0.85;
    font-weight: 500;
  }
</style>
</head>
<body>
<div id="game-wrap">
  <div id="hud">0</div>
  <div id="best">Best: 0</div>
  <canvas id="c" width="420" height="620"></canvas>
  <div id="overlay">
    <h1 id="overlay-title">Sky Hopper 🐦</h1>
    <p id="overlay-sub">Tap, click, or press Space to flap</p>
    <p class="hint">Weave through the clouds and rack up points!</p>
  </div>
</div>

<script>
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

const hud = document.getElementById('hud');
const bestEl = document.getElementById('best');
const overlay = document.getElementById('overlay');
const overlayTitle = document.getElementById('overlay-title');
const overlaySub = document.getElementById('overlay-sub');

let best = 0;
try {
  const saved = window.name.match(/skyhopper_best=(\d+)/);
  if (saved) best = parseInt(saved[1], 10);
} catch(e) {}
bestEl.textContent = 'Best: ' + best;

const GRAVITY = 0.45;
const FLAP = -8;
const PIPE_GAP = 165;
const PIPE_WIDTH = 66;
const PIPE_SPEED = 2.6;
const PIPE_SPACING = 210;

const COLORS = ['#ff8ba0', '#ffd166', '#8ee3c8', '#8ab4ff', '#c9a2ff'];

let state = 'ready'; // ready, playing, dead
let bird, pipes, score, frame, particles;

function reset() {
  bird = { x: 110, y: H/2, vy: 0, r: 16, rot: 0 };
  pipes = [];
  score = 0;
  frame = 0;
  particles = [];
  hud.textContent = '0';
}
reset();

function spawnPipe() {
  const margin = 70;
  const gapY = margin + Math.random() * (H - margin*2 - PIPE_GAP);
  const color = COLORS[Math.floor(Math.random()*COLORS.length)];
  pipes.push({ x: W + 20, gapY, passed: false, color });
}

function flap() {
  if (state === 'ready') {
    state = 'playing';
    overlay.style.display = 'none';
  }
  if (state === 'dead') {
    reset();
    state = 'playing';
    overlay.style.display = 'none';
    return;
  }
  if (state === 'playing') {
    bird.vy = FLAP;
    for (let i=0;i<6;i++){
      particles.push({x:bird.x-10,y:bird.y+8,vx:-Math.random()*2-1,vy:(Math.random()-0.5)*2,life:20,r:Math.random()*3+2});
    }
  }
}

function die() {
  state = 'dead';
  if (score > best) {
    best = score;
    bestEl.textContent = 'Best: ' + best;
    try { window.name = 'skyhopper_best=' + best; } catch(e) {}
  }
  overlay.style.display = 'flex';
  overlayTitle.textContent = 'Score: ' + score;
  overlaySub.textContent = best === score && score > 0 ? '🎉 New best!' : 'Tap to try again';
}

document.addEventListener('keydown', e => {
  if (e.code === 'Space') { e.preventDefault(); flap(); }
});
canvas.addEventListener('mousedown', flap);
canvas.addEventListener('touchstart', e => { e.preventDefault(); flap(); }, {passive:false});

function drawCloud(x,y,scale,alpha) {
  ctx.save();
  ctx.globalAlpha = alpha;
  ctx.fillStyle = '#fff';
  ctx.beginPath();
  ctx.ellipse(x, y, 22*scale, 14*scale, 0, 0, Math.PI*2);
  ctx.ellipse(x+18*scale, y+4*scale, 16*scale, 11*scale, 0, 0, Math.PI*2);
  ctx.ellipse(x-18*scale, y+5*scale, 15*scale, 10*scale, 0, 0, Math.PI*2);
  ctx.fill();
  ctx.restore();
}

let bgClouds = [];
for (let i=0;i<6;i++){
  bgClouds.push({x: Math.random()*W, y: 40+Math.random()*300, scale: 0.6+Math.random()*0.8, speed: 0.2+Math.random()*0.3, alpha: 0.5+Math.random()*0.3});
}

function drawBackground() {
  const grad = ctx.createLinearGradient(0,0,0,H);
  grad.addColorStop(0, '#7ecbff');
  grad.addColorStop(0.6, '#a0e8ff');
  grad.addColorStop(1, '#ffe9a8');
  ctx.fillStyle = grad;
  ctx.fillRect(0,0,W,H);

  bgClouds.forEach(c => {
    drawCloud(c.x, c.y, c.scale, c.alpha);
    if (state === 'playing') c.x -= c.speed;
    if (c.x < -60) c.x = W + 60;
  });
}

function drawPipe(p) {
  ctx.save();
  ctx.fillStyle = p.color;
  const topH = p.gapY - PIPE_GAP/2;
  const botY = p.gapY + PIPE_GAP/2;
  // top pipe
  roundRectPipe(p.x, 0, PIPE_WIDTH, topH, true);
  // bottom pipe
  roundRectPipe(p.x, botY, PIPE_WIDTH, H - botY, false);
  ctx.restore();
}

function roundRectPipe(x,y,w,h, isTop) {
  ctx.fillStyle = ctx.fillStyle;
  ctx.beginPath();
  ctx.fillRect(x, y, w, h);
  // cap
  const capH = 18;
  ctx.save();
  ctx.globalAlpha = 0.9;
  ctx.fillStyle = 'rgba(255,255,255,0.35)';
  if (isTop) {
    ctx.fillRect(x-3, h-capH, w+6, capH);
  } else {
    ctx.fillRect(x-3, y, w+6, capH);
  }
  ctx.restore();
}

function drawBird() {
  ctx.save();
  ctx.translate(bird.x, bird.y);
  ctx.rotate(bird.rot);
  // body
  ctx.fillStyle = '#ffcf4d';
  ctx.beginPath();
  ctx.ellipse(0,0,bird.r,bird.r*0.85,0,0,Math.PI*2);
  ctx.fill();
  // wing
  ctx.fillStyle = '#ffb020';
  ctx.beginPath();
  ctx.ellipse(-4, 4, 9, 6, Math.sin(frame*0.3)*0.3, 0, Math.PI*2);
  ctx.fill();
  // eye
  ctx.fillStyle = '#2b2b2b';
  ctx.beginPath();
  ctx.arc(7, -4, 2.6, 0, Math.PI*2);
  ctx.fill();
  // beak
  ctx.fillStyle = '#ff7a3d';
  ctx.beginPath();
  ctx.moveTo(bird.r-2, -2);
  ctx.lineTo(bird.r+9, 1);
  ctx.lineTo(bird.r-2, 5);
  ctx.closePath();
  ctx.fill();
  ctx.restore();
}

function drawParticles() {
  particles.forEach(p => {
    ctx.save();
    ctx.globalAlpha = Math.max(p.life/20, 0);
    ctx.fillStyle = '#fff';
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.r, 0, Math.PI*2);
    ctx.fill();
    ctx.restore();
  });
}

function update() {
  frame++;
  if (state === 'playing') {
    bird.vy += GRAVITY;
    bird.y += bird.vy;
    bird.rot = Math.max(-0.5, Math.min(1.2, bird.vy/10));

    if (frame % Math.round(PIPE_SPACING/PIPE_SPEED) === 0) spawnPipe();

    pipes.forEach(p => p.x -= PIPE_SPEED);
    pipes = pipes.filter(p => p.x > -PIPE_WIDTH - 20);

    pipes.forEach(p => {
      if (!p.passed && p.x + PIPE_WIDTH < bird.x) {
        p.passed = true;
        score++;
        hud.textContent = score;
      }
      const topH = p.gapY - PIPE_GAP/2;
      const botY = p.gapY + PIPE_GAP/2;
      const inX = bird.x + bird.r > p.x && bird.x - bird.r < p.x + PIPE_WIDTH;
      if (inX) {
        if (bird.y - bird.r < topH || bird.y + bird.r > botY) {
          die();
        }
      }
    });

    if (bird.y + bird.r > H || bird.y - bird.r < 0) {
      bird.y = Math.max(bird.r, Math.min(H - bird.r, bird.y));
      die();
    }

    particles.forEach(p => { p.x += p.vx; p.y += p.vy; p.life--; });
    particles = particles.filter(p => p.life > 0);
  }
}

function render() {
  drawBackground();
  pipes.forEach(drawPipe);
  drawParticles();
  drawBird();
}

function loop() {
  update();
  render();
  requestAnimationFrame(loop);
}
loop();
</script>
</body>
</html>
