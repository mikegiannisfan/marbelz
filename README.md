<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Fluoro Dash — Unlimited Neon Runner</title>
  <meta name="description" content="Fluoro Dash — neon endless runner with unlimited jumps and top/bottom obstacles. Session high score only." />
  <style>
    :root{ --bg:#060616; --panel: rgba(255,255,255,0.03); --accent1:#00ffea; --accent2:#ff00d2; --accent3:#fff700 }
    html,body{height:100%;margin:0;font-family:Inter,system-ui,Segoe UI,Roboto,Arial;background:linear-gradient(180deg,var(--bg),#07071a 70%);color:#e6f7ff}
    .wrap{min-height:100%;display:flex;align-items:center;justify-content:center;padding:20px}
    .card{width:980px;max-width:96vw;background:linear-gradient(180deg, rgba(255,255,255,0.01), rgba(255,255,255,0.01));border-radius:14px;padding:16px;box-shadow:0 12px 50px rgba(0,0,0,.6)}
    header{display:flex;align-items:center;gap:12px}
    h1{font-size:20px;margin:0}
    canvas{display:block;width:100%;height:520px;border-radius:12px;background:radial-gradient(ellipse at center, rgba(10,10,20,0.6), rgba(0,0,0,0.6));box-shadow:0 8px 40px rgba(0,0,0,.6) inset}
    .hud{display:flex;gap:10px;align-items:center;margin-top:12px}
    .chip{background:var(--panel);padding:6px 10px;border-radius:10px;font-weight:700;font-size:14px}
    .btn{background:transparent;border:1px solid rgba(255,255,255,0.06);padding:8px 12px;border-radius:10px;color:#e6f7ff;cursor:pointer}
    .btn.primary{border-color:rgba(0,255,234,0.5)}
    select{background:transparent;border:1px solid rgba(255,255,255,0.06);padding:6px 8px;border-radius:8px;color:#e6f7ff}
    footer{margin-top:12px;font-size:12px;color:#bcd}
    @media(max-width:640px){canvas{height:420px}}
  </style>
</head>
<body>
  <div class="wrap"><div class="card">
    <header>
      <div>
        <h1>Fluoro Dash — Unlimited Neon Runner</h1>
        <div style="font-size:12px;opacity:0.85">Unlimited jumps, logs from top & bottom, session-only high score</div>
      </div>
      <div style="margin-left:auto;text-align:right;font-size:12px;opacity:0.9">Made with ♥ — single file</div>
    </header>

    <canvas id="game"></canvas>

    <div class="hud">
      <div class="chip">Score: <span id="score">0</span></div>
      <div class="chip">High (session): <span id="high">0</span></div>
      <div class="chip">Orbs: <span id="orbs">0</span></div>
      <label class="chip">Difficulty:
        <select id="diffSel"><option value="0">Easy</option><option value="1">Normal</option><option value="2">Hard</option></select>
      </label>
      <button id="startBtn" class="btn primary">Start</button>
      <button id="muteBtn" class="btn">Mute</button>
      <button id="shareBtn" class="btn">Share</button>
    </div>

    <footer>Tip: Tap/click or press Space/Up to chain jumps. High score resets when you refresh.</footer>
  </div></div>

<script>
// Fluoro Dash — merged version
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
function resize(){ canvas.width = canvas.clientWidth; canvas.height = canvas.clientHeight; }
window.addEventListener('resize', resize); resize();

// DOM
const scoreEl = document.getElementById('score');
const highEl = document.getElementById('high');
const orbsEl = document.getElementById('orbs');
const startBtn = document.getElementById('startBtn');
const muteBtn = document.getElementById('muteBtn');
const shareBtn = document.getElementById('shareBtn');
const diffSel = document.getElementById('diffSel');

// state
let running=false, last=0;
let score=0, orbs=0, sessionHigh=0;
let speedBase = 260; // px/sec
let gravity = 1200;
let particles=[];
let obstacles=[];

// player (circle)
const player = { x:120, y:0, r:18, vy:0, alive:true };

// audio
let audioCtx=null, masterGain=null, muted=false;
function initAudio(){ if(audioCtx) return; audioCtx = new (window.AudioContext||window.webkitAudioContext)(); masterGain = audioCtx.createGain(); masterGain.gain.value=0.06; masterGain.connect(audioCtx.destination); }
function beep(freq,dur=0.06){ if(muted) return; initAudio(); const o=audioCtx.createOscillator(); const g=audioCtx.createGain(); o.type='sine'; o.frequency.value=freq; g.gain.value=0.0001; o.connect(g); g.connect(masterGain); o.start(); g.gain.linearRampToValueAtTime(0.6,audioCtx.currentTime+0.01); g.gain.exponentialRampToValueAtTime(0.001,audioCtx.currentTime+dur); o.stop(audioCtx.currentTime+dur+0.02); }

// controls
function jump(){ if(!running) start(); if(!player.alive) return; // unlimited jumps: always allow
  const jumpStrength = -480; // strong jump
  player.vy = jumpStrength; makeParticle(player.x, player.y, 200); beep(900,0.05);
}
window.addEventListener('keydown', e=>{ if(e.code==='Space' || e.key==='ArrowUp'){ e.preventDefault(); jump(); }});
canvas.addEventListener('pointerdown', jump);
window.addEventListener('touchstart', (e)=>{ jump(); });

// spawn obstacles from top or bottom
let spawnTimer=0, orbTimer=0; function spawnObstacle(){ const side = Math.random()<0.5? 'bottom':'top'; const w = rand(28,56); const h = rand(28,110);
  if(side==='bottom'){ const y = canvas.height - 64 - h; obstacles.push({ x: canvas.width + 80, y, w, h, from:'bottom', hue: rand(0,360) }); }
  else { const y = 64; // drop from near top
    obstacles.push({ x: canvas.width + 80, y: y, w, h, from:'top', hue: rand(0,360) }); }
}
function spawnOrb(){ const y = rand(canvas.height*0.2, canvas.height-140); orbTimer = rand(1.8,3.2); orbList.push({ x: canvas.width+40, y, r:10, hue: rand(160,320) }); }

let orbList = [];

function resetGame(){ score=0; orbs=0; obstacles=[]; particles=[]; orbList=[]; player.y = canvas.height - 64 - player.r; player.vy=0; player.alive=true; spawnTimer=0; orbTimer=0; updateHUD(); }

function start(){ resetGame(); running=true; last = performance.now(); loop(last); startBtn.textContent='Restart'; }
startBtn.addEventListener('click', ()=>{ if(!running) start(); else { if(confirm('Restart run?')) start(); }});

muteBtn.addEventListener('click', ()=>{ muted = !muted; muteBtn.textContent = muted? 'Unmute' : 'Mute'; if(!muted) beep(440,0.04); });
shareBtn.addEventListener('click', ()=>{ navigator.clipboard?.writeText(location.href).then(()=>alert('URL copied to clipboard')).catch(()=>alert('Copy failed')); });

diffSel.addEventListener('change', ()=>{/* difficulty applies in loop via factor */});

function rand(a,b){ return a + Math.random()*(b-a); }

function makeParticle(x,y,hue){ for(let i=0;i<8;i++){ particles.push({ x, y, vx: rand(-240,240), vy: rand(-280,40), life: rand(0.5,1.2), t:0, hue, sz: rand(1.5,4) }); } }

function update(now){ const dt = Math.min(0.033, (now-last)/1000); last = now; const diff = Number(diffSel.value); const speed = speedBase + (diff*60) + Math.max(0, score/80);
  // spawn timers
  spawnTimer -= dt; orbTimer -= dt;
  if(spawnTimer<=0){ spawnObstacle(); spawnTimer = rand(0.9,1.6) - Math.min(0.6, speed/1000); }
  if(orbTimer<=0){ if(Math.random()<0.6) orbList.push({ x: canvas.width+40, y: rand(canvas.height*0.25, canvas.height-160), r:10, hue: rand(160,320) }); orbTimer = rand(1.6,3.0); }

  // physics
  player.vy += gravity * dt; player.y += player.vy * dt; const groundY = canvas.height - 64; if(player.y + player.r >= groundY){ player.y = groundY - player.r; player.vy = 0; }
  if(player.y - player.r <= 0){ player.y = player.r; player.vy = Math.max(0, player.vy); }

  // move obstacles
  for(let i=obstacles.length-1;i>=0;i--){ const o = obstacles[i]; o.x -= speed*dt; if(o.x + o.w < -80) obstacles.splice(i,1); }
  for(let i=orbList.length-1;i>=0;i--){ orbList[i].x -= speed*dt; if(orbList[i].x < -40) orbList.splice(i,1); }

  // collisions
  for(let i=obstacles.length-1;i>=0;i--){ const o = obstacles[i]; if(circleRectColl(player.x, player.y, player.r-2, o.x, o.y, o.w, o.h)){
      player.alive = false; running = false; beep(120,0.18); // update session high
      if(score > sessionHigh) sessionHigh = Math.floor(score); highEl.textContent = sessionHigh; setTimeout(()=>{ const s = Math.floor(score); if(confirm(`Game over — score ${s}. Play again?`)) start(); }, 80);
    }
  }
  for(let i=orbList.length-1;i>=0;i--){ const ob = orbList[i]; const dx = player.x - ob.x, dy = player.y - ob.y; if(dx*dx + dy*dy < (player.r + ob.r)*(player.r + ob.r)){
      orbs++; orbList.splice(i,1); score += 60; makeParticle(ob.x, ob.y, ob.hue); beep(1200,0.03);
    } }

  // score
  score += Math.floor(60*dt);
  updateHUD();

  // particles
  for(let i=particles.length-1;i>=0;i--){ const p = particles[i]; p.t += dt; p.x += p.vx*dt; p.y += p.vy*dt; p.vy += 380*dt; if(p.t > p.life) particles.splice(i,1); }
}

function updateHUD(){ scoreEl.textContent = Math.floor(score); orbsEl.textContent = orbs; highEl.textContent = sessionHigh; }

function render(){ ctx.clearRect(0,0,canvas.width,canvas.height);
  // background neon gradient with faint scanlines
  const g = ctx.createLinearGradient(0,0,0,canvas.height); g.addColorStop(0,'#051024'); g.addColorStop(1,'#04030b'); ctx.fillStyle = g; ctx.fillRect(0,0,canvas.width,canvas.height);
  ctx.globalAlpha = 0.06; for(let y=0;y<canvas.height;y+=18){ ctx.fillStyle='rgba(255,255,255,0.02)'; ctx.fillRect(0,y,canvas.width,2); } ctx.globalAlpha = 1;

  // orbs
  orbList.forEach(ob=>{ drawGlowCircle(ob.x, ob.y, ob.r*1.6, `hsl(${ob.hue} 85% 55%)`, 0.26); ctx.beginPath(); ctx.arc(ob.x, ob.y, ob.r,0,Math.PI*2); ctx.fillStyle='#fff'; ctx.fill(); ctx.globalCompositeOperation='lighter'; ctx.fillStyle=`hsl(${ob.hue} 90% 60% / 0.7)`; ctx.fill(); ctx.globalCompositeOperation='source-over'; });

  // obstacles
  obstacles.forEach(o=>{
    const grad = ctx.createLinearGradient(o.x, o.y, o.x, o.y + o.h);
    grad.addColorStop(0, `hsl(${o.hue} 95% 55%)`); grad.addColorStop(1, `hsl(${(o.hue+60)%360} 85% 30%)`);
    drawGlowRect(o.x + o.w/2, o.y + o.h/2, o.w*1.8, o.h*1.8, `hsl(${o.hue} 90% 50%)`, 0.16);
    ctx.fillStyle = grad; roundRect(ctx, o.x, o.y, o.w, o.h, 8); ctx.fill(); ctx.strokeStyle='rgba(255,255,255,0.03)'; ctx.lineWidth=1; ctx.stroke();
  });

  // ground
  ctx.fillStyle='rgba(255,255,255,0.03)'; ctx.fillRect(0, canvas.height-64, canvas.width, 64);

  // player
  drawGlowCircle(player.x, player.y, player.r*2.6, '#00ffd6', 0.2);
  ctx.beginPath(); ctx.arc(player.x, player.y, player.r,0,Math.PI*2); ctx.fillStyle='#e9ffff'; ctx.fill(); ctx.beginPath(); ctx.arc(player.x - player.r*0.34, player.y - player.r*0.34, player.r*0.48,0,Math.PI*2); ctx.fillStyle='rgba(255,255,255,0.5)'; ctx.fill();

  // particles
  particles.forEach(p=>{ ctx.globalAlpha = Math.max(0, 1 - p.t/p.life); ctx.beginPath(); ctx.arc(p.x, p.y, p.sz, 0, Math.PI*2); ctx.fillStyle = `hsl(${p.hue} 90% 60%)`; ctx.fill(); ctx.globalAlpha = 1; });

  // HUD title
  ctx.font = '700 18px system-ui'; ctx.textAlign='left'; ctx.fillStyle='rgba(255,255,255,0.06)'; ctx.fillText('FLUORO DASH', 18, 36);
}

function loop(now){ if(!running) return; update(now); render(); requestAnimationFrame(loop); }

// helpers
function drawGlowCircle(x,y,r,color,alpha=0.2){ ctx.save(); ctx.globalCompositeOperation='lighter'; const g = ctx.createRadialGradient(x,y,0,x,y,r*1.2); g.addColorStop(0, color); g.addColorStop(1,'rgba(0,0,0,0)'); ctx.globalAlpha = alpha; ctx.fillStyle = g; ctx.fillRect(x-r*1.4, y-r*1.4, r*2.8, r*2.8); ctx.restore(); }
function drawGlowRect(cx,cy,w,h,color,alpha=0.14){ ctx.save(); ctx.globalCompositeOperation='lighter'; const g = ctx.createRadialGradient(cx,cy,0,cx,cy,Math.max(w,h)); g.addColorStop(0, color); g.addColorStop(1,'rgba(0,0,0,0)'); ctx.globalAlpha = alpha; ctx.fillStyle = g; ctx.fillRect(cx - w/2, cy - h/2, w, h); ctx.restore(); }
function roundRect(ctx,x,y,w,h,r){ ctx.beginPath(); ctx.moveTo(x+r,y); ctx.arcTo(x+w,y,x+w,y+h,r); ctx.arcTo(x+w,y+h,x,y+h,r); ctx.arcTo(x,y+h,x,y,r); ctx.arcTo(x,y,x+w,y,r); ctx.closePath(); }
function circleRectColl(cx,cy,r, rx, ry, rw, rh){ const testX = Math.max(rx, Math.min(cx, rx+rw)); const testY = Math.max(ry, Math.min(cy, ry+rh)); const dx = cx - testX, dy = cy - testY; return (dx*dx + dy*dy) <= r*r; }

// initial player placement
player.y = canvas.height - 64 - player.r;

// quick hint before start
(function hint(){ const ctx2=ctx; ctx2.font='14px system-ui'; ctx2.fillStyle='rgba(255,255,255,0.06)'; ctx2.fillText('Press Start or Tap to play — Space / Up to jump (unlimited)', 18, 100); })();

// Expose start to buttons
window.Fluoro = { start };

</script>
</body>
</html>
