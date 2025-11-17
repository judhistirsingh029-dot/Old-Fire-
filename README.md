# Old-Fire-
<!doctype html>
<html lang="hi">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Gem Catcher — Simple HTML5 Game</title>
<style>
  :root{--bg:#0b1220;--panel:#0f1724;--accent:#7be4ff;--good:#7cff7c;--bad:#ff7c7c;}
  html,body{height:100%;margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,"Helvetica Neue",Arial;}
  body{background:linear-gradient(180deg,var(--bg),#071018);display:flex;align-items:center;justify-content:center;color:#dfeffd;}
  #game-wrap{width:360px;max-width:96vw;background:linear-gradient(180deg,var(--panel),#0b1530);border-radius:12px;padding:12px;box-shadow:0 10px 30px rgba(0,0,0,0.6);}
  header{display:flex;justify-content:space-between;align-items:center;margin-bottom:8px}
  header h1{font-size:16px;margin:0}
  .hud{display:flex;gap:8px;align-items:center}
  .badge{background:rgba(255,255,255,0.04);padding:6px 8px;border-radius:8px;font-size:13px}
  canvas{background:linear-gradient(180deg,#08131f, #06202c);display:block;border-radius:8px;width:100%;height:auto;touch-action:none}
  .controls{display:flex;gap:6px;margin-top:8px;justify-content:center}
  .btn{background:linear-gradient(180deg,rgba(255,255,255,0.02),rgba(255,255,255,0.01));border:1px solid rgba(255,255,255,0.03);padding:8px 10px;border-radius:8px;font-size:13px;cursor:pointer}
  .small{font-size:12px;padding:6px 8px}
  footer{margin-top:8px;font-size:12px;color:rgba(223,239,253,0.6);text-align:center}
  .overlay{position:fixed;left:0;top:0;right:0;bottom:0;display:flex;align-items:center;justify-content:center;pointer-events:none}
  @media (min-width:700px){#game-wrap{width:420px}}
</style>
</head>
<body>
<div id="game-wrap" role="application" aria-label="Gem Catcher game">
  <header>
    <h1>Gem Catcher</h1>
    <div class="hud">
      <div class="badge">Score: <span id="score">0</span></div>
      <div class="badge">Lives: <span id="lives">3</span></div>
      <div class="badge">Level: <span id="level">1</span></div>
    </div>
  </header>

  <canvas id="game" width="720" height="480" aria-label="Game canvas"></canvas>

  <div class="controls">
    <button id="left" class="btn small">←</button>
    <button id="pause" class="btn small">Pause</button>
    <button id="right" class="btn small">→</button>
    <button id="restart" class="btn small">Restart</button>
  </div>

  <footer>Use ← → keys or touch (tap left/right) to move. Catch green gems, avoid red skulls.</footer>
</div>

<script>
/* Gem Catcher — simple HTML5 canvas game
   Single file. Change constants below to tweak gameplay.
*/

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const scoreEl = document.getElementById('score');
const livesEl = document.getElementById('lives');
const levelEl = document.getElementById('level');

let W = canvas.width;
let H = canvas.height;

const PLAYER = {
  x: W/2, y: H - 60, w: 80, h: 18, speed: 6
};

let state = {
  score: 0,
  lives: 3,
  level: 1,
  running: true,
  spawnTimer: 0,
  spawnInterval: 70, // frames
  items: [], // falling things
  frame: 0
};

function resizeCanvasToDisplay() {
  const rect = canvas.getBoundingClientRect();
  const ratio = canvas.width / canvas.height;
  let newW = Math.floor(rect.width * devicePixelRatio);
  let newH = Math.floor((rect.width / ratio) * devicePixelRatio);
  if (newW !== canvas.width || newH !== canvas.height) {
    canvas.width = newW;
    canvas.height = newH;
    W = canvas.width;
    H = canvas.height;
    PLAYER.y = H - 60*devicePixelRatio;
    PLAYER.w = 80*devicePixelRatio;
    PLAYER.h = 18*devicePixelRatio;
  }
}
resizeCanvasToDisplay();

function rand(min,max){return Math.random()*(max-min)+min}

function spawnItem() {
  // type: "gem" (good) or "skull" (bad)
  const t = Math.random() < 0.78 ? 'gem' : 'skull';
  const speed = rand(1.2, 2.6) + state.level*0.2;
  state.items.push({
    x: rand(30, W-30),
    y: -40,
    r: 12 + Math.random()*8,
    type: t,
    vy: speed
  });
}

function drawPlayer() {
  ctx.save();
  ctx.translate(PLAYER.x, PLAYER.y);
  // base
  ctx.fillStyle = '#9bdcff';
  roundRect(ctx, -PLAYER.w/2, -PLAYER.h/2, PLAYER.w, PLAYER.h, 6);
  ctx.fill();
  // highlight
  ctx.fillStyle = 'rgba(255,255,255,0.12)';
  roundRect(ctx, -PLAYER.w/2+6, -PLAYER.h/2+2, PLAYER.w-12, PLAYER.h-6,4);
  ctx.fill();
  ctx.restore();
}

function drawItem(it) {
  ctx.save();
  ctx.translate(it.x, it.y);
  if (it.type === 'gem') {
    // diamond-like gem
    ctx.beginPath();
    ctx.moveTo(0,-it.r);
    ctx.lineTo(it.r*0.6,0);
    ctx.lineTo(0,it.r);
    ctx.lineTo(-it.r*0.6,0);
    ctx.closePath();
    ctx.fillStyle = '#7cff7c';
    ctx.fill();
    ctx.strokeStyle = 'rgba(255,255,255,0.08)';
    ctx.stroke();
  } else {
    // skull simple circle with crossbones (stylized)
    ctx.beginPath();
    ctx.arc(0,0,it.r,0,Math.PI*2);
    ctx.fillStyle = '#ff7c7c';
    ctx.fill();
    ctx.fillStyle = '#fff';
    ctx.fillRect(-it.r*0.45, -it.r*0.1, it.r*0.9, it.r*0.18);
    ctx.fillStyle = 'rgba(0,0,0,0.6)';
    ctx.fillRect(-it.r*0.25, -it.r*0.15, it.r*0.15, it.r*0.15);
    ctx.fillRect(it.r*0.1, -it.r*0.15, it.r*0.15, it.r*0.15);
  }
  ctx.restore();
}

function update() {
  if (!state.running) return;
  state.frame++;
  // spawn logic
  state.spawnTimer++;
  if (state.spawnTimer >= Math.max(10, state.spawnInterval - state.level*3)) {
    spawnItem();
    state.spawnTimer = 0;
  }

  // move items
  for (let i = state.items.length-1; i >= 0; i--) {
    const it = state.items[i];
    it.y += it.vy + Math.sin((state.frame+it.x)*0.02)*0.6;
    // collision with player
    const px = PLAYER.x;
    const py = PLAYER.y - PLAYER.h/2;
    const dx = Math.abs(it.x - px);
    const dy = Math.abs(it.y - py);
    if (dx < it.r + PLAYER.w/2 && dy < it.r + PLAYER.h/2) {
      // caught
      if (it.type === 'gem') {
        state.score += 10 + Math.floor(state.level*2);
      } else {
        state.lives -= 1;
        // small penalty
        state.score = Math.max(0, state.score - 8);
      }
      state.items.splice(i,1);
      continue;
    }
    // off screen
    if (it.y - it.r > H + 60) {
      // missed good gem => penalty; skull simply disappears
      if (it.type === 'gem') {
        state.lives -= 1;
      }
      state.items.splice(i,1);
    }
  }

  // level up
  const newLevel = 1 + Math.floor(state.score / 200);
  if (newLevel !== state.level) {
    state.level = newLevel;
    // speed up spawn a bit
    state.spawnInterval = Math.max(20, state.spawnInterval - 6);
  }

  // check game over
  if (state.lives <= 0) {
    state.running = false;
  }

  // update HUD
  scoreEl.textContent = state.score;
  livesEl.textContent = Math.max(0, state.lives);
  levelEl.textContent = state.level;
}

function render() {
  // clear
  ctx.clearRect(0,0,W,H);

  // background subtle grid
  const gSize = 36 * devicePixelRatio;
  ctx.save();
  ctx.globalAlpha = 0.06;
  ctx.strokeStyle = '#7be4ff';
  for (let x=0;x<W;x+=gSize){
    ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,H); ctx.stroke();
  }
  for (let y=0;y<H;y+=gSize){
    ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(W,y); ctx.stroke();
  }
  ctx.restore();

  // draw items
  for (const it of state.items) drawItem(it);

  // draw player
  drawPlayer();

  // top text
  if (!state.running) {
    ctx.save();
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(W/2-220*devicePixelRatio, H/2-50*devicePixelRatio, 440*devicePixelRatio, 100*devicePixelRatio);
    ctx.fillStyle = '#fff';
    ctx.textAlign = 'center';
    ctx.font = `${24*devicePixelRatio}px system-ui`;
    ctx.fillText('Game Over', W/2, H/2 - 6*devicePixelRatio);
    ctx.font = `${14*devicePixelRatio}px system-ui`;
    ctx.fillText(`Final Score: ${state.score}`, W/2, H/2 + 22*devicePixelRatio);
    ctx.restore();
  }
}

function loop() {
  update();
  render();
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);

// helpers
function roundRect(ctx, x, y, w, h, r){
  ctx.beginPath();
  ctx.moveTo(x+r,y);
  ctx.arcTo(x+w,y,x+w,y+h,r);
  ctx.arcTo(x+w,y+h,x,y+h,r);
  ctx.arcTo(x,y+h,x,y,r);
  ctx.arcTo(x,y,x+w,y,r);
  ctx.closePath();
}

// controls
let leftDown=false, rightDown=false;
window.addEventListener('keydown', e => {
  if (e.key === 'ArrowLeft') leftDown = true;
  if (e.key === 'ArrowRight') rightDown = true;
  if (e.key === ' ' || e.key === 'Spacebar') { state.running = !state.running; }
});
window.addEventListener('keyup', e => {
  if (e.key === 'ArrowLeft') leftDown = false;
  if (e.key === 'ArrowRight') rightDown = false;
});

document.getElementById('left').addEventListener('pointerdown', ()=>{ leftDown=true });
document.getElementById('left').addEventListener('pointerup', ()=>{ leftDown=false });
document.getElementById('right').addEventListener('pointerdown', ()=>{ rightDown=true });
document.getElementById('right').addEventListener('pointerup', ()=>{ rightDown=false });

document.getElementById('pause').addEventListener('click', ()=>{
  state.running = !state.running;
  document.getElementById('pause').textContent = state.running ? 'Pause' : 'Resume';
});

document.getElementById('restart').addEventListener('click', ()=>{
  resetGame();
});

function resetGame(){
  state.score = 0;
  state.lives = 3;
  state.level = 1;
  state.spawnInterval = 70;
  state.items = [];
  state.running = true;
  state.spawnTimer = 0;
  document.getElementById('pause').textContent = 'Pause';
}

// pointer (touch) support on canvas
canvas.addEventListener('pointerdown', function(e){
  const rect = canvas.getBoundingClientRect();
  const x = (e.clientX - rect.left) * (canvas.width / rect.width);
  if (x < canvas.width/2) {
    leftDown = true;
    setTimeout(()=>{ leftDown=false }, 150);
  } else {
    rightDown = true;
    setTimeout(()=>{ rightDown=false }, 150);
  }
});

// apply movement each frame
setInterval(()=> {
  if (leftDown) PLAYER.x -= PLAYER.speed * devicePixelRatio;
  if (rightDown) PLAYER.x += PLAYER.speed * devicePixelRatio;
  // clamp
  PLAYER.x = Math.max(PLAYER.w/2, Math.min(W-PLAYER.w/2, PLAYER.x));
}, 16);

// ensure canvas scales on resize/orientation change
window.addEventListener('resize', () => {
  // keep logical size stable but adapt physical pixels
  resizeCanvasToDisplay();
});

// initial player setup
PLAYER.x = W/2;
PLAYER.y = H - 60*devicePixelRatio;

// small tutorial spawn so player sees items
for (let i=0;i<3;i++){
  state.items.push({x: 80 + i*120, y: 40 + i*30, r: 14, type: 'gem', vy: 1.4});
}
</script>
</body>
</html>
