<!doctype html>
<html lang="fr">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Course Luxe — BMW (démo)</title>
<style>
  :root{
    --bg:#0b0f13;
    --panel:#0f1720;
    --accent:#0ea5ad;
    --muted:#94a3b8;
    --danger:#ef4444;
    --glass: rgba(255,255,255,0.04);
  }
  html,body{height:100%;margin:0;background:linear-gradient(180deg,var(--bg),#071019);font-family:Inter,Segoe UI,Roboto,Arial,sans-serif;color:#e6eef6}
  .container{display:grid;grid-template-columns:1fr 340px;gap:18px;padding:18px;height:100%}
  canvas{width:100%;height:100%;background:linear-gradient(180deg,#071021 0%, #071018 60%);border-radius:12px;display:block;box-shadow:0 10px 30px rgba(2,6,23,0.8) inset}
  .sidebar{background:var(--panel);border-radius:12px;padding:16px;box-shadow:0 8px 24px rgba(2,6,23,0.6)}
  h1{font-size:18px;margin:0 0 8px}
  .controls{display:flex;flex-direction:column;gap:8px;margin-bottom:12px}
  .btn{background:var(--glass);border:1px solid rgba(255,255,255,0.04);padding:8px 10px;border-radius:8px;color:var(--muted);cursor:pointer}
  .btn.primary{background:linear-gradient(90deg,var(--accent),#60a5fa);color:#02121b;border:0}
  .row{display:flex;gap:8px;align-items:center}
  .stat{display:flex;justify-content:space-between;padding:8px 10px;background:rgba(255,255,255,0.02);border-radius:8px;font-size:14px;color:var(--muted)}
  .label{font-size:12px;color:#9fb6c0}
  .big{font-size:20px;color:#e6eef6}
  .footer{margin-top:12px;font-size:12px;color:var(--muted)}
  .touch-controls{display:flex;gap:8px;flex-wrap:wrap;margin-top:10px}
  .touch-btn{flex:1;padding:10px;border-radius:8px;background:rgba(255,255,255,0.03);text-align:center;user-select:none}
  .range{width:100%}
  .small{font-size:12px;color:var(--muted)}
  @media (max-width:900px){
    .container{grid-template-columns:1fr}
    .sidebar{order:2}
  }
</style>
</head>
<body>
<div class="container">
  <div style="display:flex;flex-direction:column;gap:12px">
    <canvas id="game" width="1200" height="800"></canvas>
    <div style="display:flex;gap:12px">
      <button class="btn primary" id="restartBtn">Redémarrer</button>
      <div style="flex:1" class="stat"><div class="label">Mode</div><div id="modeText">Normal</div></div>
      <div style="width:160px" class="stat"><div class="label">Difficulté</div><select id="difficulty" style="background:transparent;border:0;color:var(--muted);outline:none">
        <option value="easy">Facile</option>
        <option value="normal" selected>Normal</option>
        <option value="hard">Difficile</option>
      </select></div>
    </div>
  </div>

  <aside class="sidebar">
    <h1>Course Luxe — BMW (démo)</h1>
    <div class="small">Contrôles: flèches / ZQSD (accélérer, freiner, tourner). Sur mobile, utilisez les boutons tactiles.</div>

    <div style="height:12px"></div>
    <div class="controls">
      <div class="stat"><div class="label">Vitesse</div><div id="speed">0 km/h</div></div>
      <div class="stat"><div class="label">Chrono</div><div id="timer">00:00.00</div></div>
      <div class="stat"><div class="label">Tours</div><div id="laps">0 / 3</div></div>
      <div class="stat"><div class="label">Position</div><div id="pos">1</div></div>
    </div>

    <div class="row" style="margin-top:6px">
      <div style="flex:1"><button class="btn" id="toggleAI">AI: ON</button></div>
      <div style="flex:1"><button class="btn" id="toggleHud">HUD</button></div>
    </div>

    <div style="margin-top:12px">
      <div class="label">Boost (space)</div>
      <input id="boostRange" type="range" min="0" max="100" value="30" class="range">
    </div>

    <div style="margin-top:12px">
      <div class="label">Touches tactiles (mobile)</div>
      <div class="touch-controls">
        <div class="touch-btn" id="tLeft">←</div>
        <div class="touch-btn" id="tUp">↑</div>
        <div class="touch-btn" id="tDown">↓</div>
        <div class="touch-btn" id="tRight">→</div>
        <div class="touch-btn" id="tBoost">Boost</div>
      </div>
    </div>

    <div class="footer">Fait avec ❤️ — Démo éducative. Si tu veux, j'ajoute des voitures ou une IA plus avancée.</div>
  </aside>
</div>

<script>
/*
  Simple top-down racing demo.
  - Single track drawn by path
  - Player car with simple physics
  - Collision with walls
  - Laps / timer
*/

// Canvas & context
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');

let W = canvas.width;
let H = canvas.height;

window.addEventListener('resize', () => {
  // maintain internal resolution but scale via CSS
  // nothing to do; canvas CSS handles size
});

// Track definition (outer and inner polygon) simple oval-ish with notches
const track = {
  outer: createRoundedRectPath(120, 80, W-240, H-160, 80),
  inner: createRoundedRectPath(280, 160, W-560, H-320, 60),
  startLine: { x: W/2 - 8, y: 150, w:16, h:60 }
};

// Player car state
const player = {
  x: W/2, y: H/2 + 220, angle: -Math.PI/2,
  speed: 0, maxSpeed: 7, accel: 0.18, brakePower: 0.4,
  turnSpeed: 0.04, width: 44, height: 80,
  laps: 0, lastCrossTime:0
};

// Game state
let keys = {};
let startTime = null;
let running = true;
let showHUD = true;
let aiOn = true;
let totalLaps = 3;
let difficulty = 'normal';

const otherCars = []; // simple AI cars (basic)
initAIcars(3);

// Input handling
addEventListener('keydown', e => { keys[e.key.toLowerCase()] = true; if(e.key===' '){ e.preventDefault(); }});
addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

document.getElementById('restartBtn').addEventListener('click', resetGame);
document.getElementById('toggleAI').addEventListener('click', ()=>{
  aiOn = !aiOn; document.getElementById('toggleAI').textContent = 'AI: ' + (aiOn ? 'ON':'OFF');
});
document.getElementById('toggleHud').addEventListener('click', ()=>{
  showHUD = !showHUD; document.getElementById('toggleHud').textContent = 'HUD ' + (showHUD ? 'ON':'OFF');
});
document.getElementById('difficulty').addEventListener('change', (e)=>{
  difficulty = e.target.value;
  document.getElementById('modeText').textContent = difficulty[0].toUpperCase()+difficulty.slice(1);
  adjustDifficulty();
});

// Touch buttons
['tLeft','tUp','tDown','tRight','tBoost'].forEach(id=>{
  const el = document.getElementById(id);
  if(!el) return;
  el.addEventListener('touchstart', ()=> el.classList.add('active'));
  el.addEventListener('touchend', ()=> el.classList.remove('active'));
  // map to keys while pressed
  el.addEventListener('touchstart', (e)=>{
    e.preventDefault();
    if(id==='tLeft') keys['arrowleft']=true;
    if(id==='tRight') keys['arrowright']=true;
    if(id==='tUp') keys['arrowup']=true;
    if(id==='tDown') keys['arrowdown']=true;
    if(id==='tBoost') keys[' ']=true;
  });
  el.addEventListener('touchend', (e)=>{
    e.preventDefault();
    if(id==='tLeft') keys['arrowleft']=false;
    if(id==='tRight') keys['arrowright']=false;
    if(id==='tUp') keys['arrowup']=false;
    if(id==='tDown') keys['arrowdown']=false;
    if(id==='tBoost') keys[' ']=false;
  });
});

// Utilities
function createRoundedRectPath(x,y,w,h,r){
  // returns array of points approximating a rounded rect using bezier-like points (simple)
  // We'll generate polygon points around the border
  const pts = [];
  const step = 18;
  // top edge
  for(let i=0;i<=step;i++){
    const t = i/step;
    const px = x + r + (w-2*r)*t;
    const py = y - r * Math.cos(Math.PI*t/2);
    pts.push({x:px,y:py});
  }
  // right edge
  for(let i=0;i<=step;i++){
    const t = i/step;
    const px = x + w + r * Math.sin(Math.PI*t/2);
    const py = y + r + (h-2*r)*t;
    pts.push({x:px,y:py});
  }
  // bottom edge
  for(let i=0;i<=step;i++){
    const t = i/step;
    const px = x + w - r - (w-2*r)*t;
    const py = y + h + r * Math.cos(Math.PI*t/2);
    pts.push({x:px,y:py});
  }
  // left edge
  for(let i=0;i<=step;i++){
    const t = i/step;
    const px = x - r * Math.sin(Math.PI*t/2);
    const py = y + h - r - (h-2*r)*t;
    pts.push({x:px,y:py});
  }
  return pts;
}

function pointInPolygon(x,y,poly){
  // ray casting
  let inside=false;
  for(let i=0,j=poly.length-1;i<poly.length;j=i++){
    const xi=poly[i].x, yi=poly[i].y;
    const xj=poly[j].x, yj=poly[j].y;
    const intersect = ((yi>y) != (yj>y)) && (x < (xj-xi)*(y-yi)/(yj-yi)+xi);
    if(intersect) inside=!inside;
  }
  return inside;
}

function resetGame(){
  player.x = W/2; player.y = H/2 + 220; player.angle = -Math.PI/2; player.speed = 0; player.laps=0;
  startTime = performance.now();
  otherCars.length = 0; initAIcars(3);
}

// Difficulty adjust
function adjustDifficulty(){
  if(difficulty==='easy'){ player.maxSpeed=7; player.accel=0.2; otherCarSpeedFactor=0.85; }
  if(difficulty==='normal'){ player.maxSpeed=8; player.accel=0.18; otherCarSpeedFactor=1.0; }
  if(difficulty==='hard'){ player.maxSpeed=9.3; player.accel=0.16; otherCarSpeedFactor=1.12; }
}

let otherCarSpeedFactor = 1.0;

// AI cars
function initAIcars(n){
  for(let i=0;i<n;i++){
    let a = {
      x: player.x + (i+1)*60,
      y: player.y - (i+1)*40,
      angle: -Math.PI/2,
      speed: 0,
      color: `hsl(${(i*60)%360} 80% 60%)`,
      targetIndex: 0
    };
    otherCars.push(a);
  }
}

// Track helper: testing point inside track area (between outer and inner)
function isOnTrack(x,y){
  return pointInPolygon(x,y,track.outer) && !pointInPolygon(x,y,track.inner);
}

// Simple function to detect crossing start line
function crossedStartLine(prev,cur){
  const s = track.startLine;
  // If segment intersects the vertical rect near start line
  // We'll use a simple check: crossed from bottom to top near x range
  if((prev.y > s.y + s.h/2) && (cur.y <= s.y + s.h/2) && Math.abs(cur.x - (s.x + s.w/2)) < 120){
    return true;
  }
  return false;
}

// Main loop
function loop(ts){
  if(!startTime) startTime = ts;
  const dt = 16/1000; // approximate fixed dt for consistency
  update(dt);
  render();
  requestAnimationFrame(loop);
}

// Update physics & state
function update(dt){
  // controls
  const forward = keys['arrowup'] || keys['z'] || keys['w'];
  const back = keys['arrowdown'] || keys['s'];
  const left = keys['arrowleft'] || keys['q'] || keys['a'];
  const right = keys['arrowright'] || keys['d'];
  const boostPressed = keys[' '];

  const boostPower = parseInt(document.getElementById('boostRange').value)/100;

  // acceleration
  if(forward){
    player.speed += player.accel * (1 + (boostPressed?boostPower:0));
  } else if(back){
    player.speed -= player.brakePower;
  } else {
    // natural friction
    player.speed *= 0.985;
  }

  // clamp speed
  const maxSpeed = player.maxSpeed * (boostPressed ? (1+boostPower*0.6) : 1);
  if(player.speed > maxSpeed) player.speed = maxSpeed;
  if(player.speed < -2.5) player.speed = -2.5;

  // turning
  const turn = (right?1:0) - (left?1:0);
  // more turning when slow, less when at high speed (simulate understeer)
  const speedFactor = Math.max(0.25, 1 - Math.abs(player.speed)/ (player.maxSpeed*1.6));
  player.angle += turn * player.turnSpeed * speedFactor;

  // drift effect (slight lateral offset)
  const vx = Math.cos(player.angle) * player.speed;
  const vy = Math.sin(player.angle) * player.speed;
  const prev = {x:player.x, y:player.y};
  player.x += vx;
  player.y += vy;

  // collision with track edges: push back if outside
  if(!isOnTrack(player.x, player.y)){
    // simple collision response: push back along negative velocity and reduce speed
    player.x -= vx * 1.6;
    player.y -= vy * 1.6;
    player.speed *= -0.35; // bounce and slow down
  }

  // start line detection for laps (from bottom->top)
  if(crossedStartLine(prev, player)){
    const now = performance.now();
    if(now - player.lastCrossTime > 800){ // debounce
      player.laps++;
      player.lastCrossTime = now;
      if(player.laps >= totalLaps){
        // finished
        running = false;
        // show final time in HUD
      }
    }
  }

  // update other cars (very basic)
  if(aiOn){
    otherCars.forEach((c,i)=>{
      // aim at a point along center of track (simple orbit)
      const targetAngle = Math.atan2(track.startLine.y - c.y, (W/2) - c.x);
      const angDiff = normalizeAngle(targetAngle - c.angle);
      c.angle += angDiff * 0.02;
      // accelerate toward speed
      const desired = 4.2 * otherCarSpeedFactor;
      c.speed += (desired - c.speed) * 0.02;
      c.x += Math.cos(c.angle) * c.speed;
      c.y += Math.sin(c.angle) * c.speed;
      // keep on track
      if(!isOnTrack(c.x,c.y)){
        c.x -= Math.cos(c.angle) * c.speed * 1.6;
        c.y -= Math.sin(c.angle) * c.speed * 1.6;
        c.angle += 0.9 * (Math.random()-0.5);
      }
    });
  }

  // Update HUD
  updateHUD();
}

function normalizeAngle(a){
  while(a > Math.PI) a -= Math.PI * 2;
  while(a < -Math.PI) a += Math.PI * 2;
  return a;
}

function updateHUD(){
  const speedEl = document.getElementById('speed');
  const timerEl = document.getElementById('timer');
  const lapsEl = document.getElementById('laps');
  const posEl = document.getElementById('pos');

  const kmh = Math.round(player.speed * 18);
  speedEl.textContent = `${kmh} km/h`;

  const elapsed = (performance.now() - startTime) / 1000;
  timerEl.textContent = formatTime(elapsed);
  lapsEl.textContent = `${player.laps} / ${totalLaps}`;

  // simple position calculation: compare player vs others by laps then y
  let pos = 1;
  otherCars.forEach(c=>{
    // naive: if car's y is less than player's y (ahead on track) then pos++
    if(c.y < player.y) pos++;
  });
  posEl.textContent = pos;
}

function formatTime(s){
  const mm = Math.floor(s/60);
  const ss = Math.floor(s%60).toString().padStart(2,'0');
  const cs = Math.floor((s*100)%100).toString().padStart(2,'0');
  return `${mm}:${ss}.${cs}`;
}

// Rendering
function render(){
  // clear
  ctx.clearRect(0,0,W,H);

  // background gradient
  const g = ctx.createLinearGradient(0,0,0,H);
  g.addColorStop(0,'#071018');
  g.addColorStop(1,'#041018');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,W,H);

  // draw track outer
  ctx.save();
  ctx.translate(0,0);
  drawPolygon(track.outer, {fill:'#0d2230', stroke:'#07121a'});
  // inner cutout (erase)
  ctx.globalCompositeOperation = 'destination-out';
  drawPolygon(track.inner, {fill:'rgba(0,0,0,1)'});
  ctx.globalCompositeOperation = 'source-over';
  // track border lines
  drawPolygon(track.outer, {stroke:'#123241', lineWidth:3, fill:null});
  drawPolygon(track.inner, {stroke:'#123241', lineWidth:3, fill:null});

  // start line
  ctx.fillStyle = '#fff';
  ctx.fillRect(track.startLine.x, track.startLine.y, track.startLine.w, track.startLine.h);
  // dashed center line
  ctx.beginPath();
  ctx.setLineDash([12,10]);
  ctx.strokeStyle = 'rgba(255,255,255,0.06)';
  ctx.lineWidth = 3;
  ctx.moveTo(W/2, track.inner[0].y + 40);
  ctx.lineTo(W/2, track.outer[0].y - 40);
  ctx.stroke();
  ctx.setLineDash([]);
  ctx.restore();

  // draw other cars
  otherCars.forEach(c => {
    drawCar(c.x, c.y, c.angle, c.color);
  });

  // draw player car with a luxurious style
  drawLuxuryCar(player.x, player.y, player.angle);

  // HUD overlays (mini map)
  if(showHUD){
    drawMiniMap();
    drawSpeedometer();
  }

  // finish message
  if(!running){
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(0, H/2 - 60, W, 120);
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 36px Inter, Arial';
    ctx.textAlign = 'center';
    ctx.fillText('Course terminée 🎉', W/2, H/2 - 6);
    ctx.font = '18px Inter, Arial';
    ctx.fillText('Appuie sur "Redémarrer" pour rejouer', W/2, H/2 + 26);
  }
}

function drawPolygon(pts, opts={}){
  if(pts.length===0) return;
  ctx.beginPath();
  ctx.moveTo(pts[0].x, pts[0].y);
  for(let i=1;i<pts.length;i++) ctx.lineTo(pts[i].x, pts[i].y);
  ctx.closePath();
  if(opts.fill !== undefined && opts.fill !== null){
    ctx.fillStyle = opts.fill;
    ctx.fill();
  }
  if(opts.stroke){
    ctx.strokeStyle = opts.stroke;
    ctx.lineWidth = opts.lineWidth || 1;
    ctx.stroke();
  }
}

function drawCar(x,y,angle,color='#f00'){
  ctx.save();
  ctx.translate(x,y);
  ctx.rotate(angle);
  // simple rectangle car
  ctx.fillStyle = color;
  ctx.fillRect(-18, -36, 36, 72);
  // windows
  ctx.fillStyle = 'rgba(255,255,255,0.12)';
  ctx.fillRect(-14, -28, 28, 22);
  ctx.restore();
}

function drawLuxuryCar(x,y,angle){
  ctx.save();
  ctx.translate(x,y);
  ctx.rotate(angle);

  // shadow
  ctx.beginPath();
  ctx.ellipse(0,40,34,12,0,0,Math.PI*2);
  ctx.fillStyle = 'rgba(0,0,0,0.35)';
  ctx.fill();

  // body
  const grd = ctx.createLinearGradient(-22,-28,22,28);
  grd.addColorStop(0,'#e6eef6');
  grd.addColorStop(0.5,'#dbeef3');
  grd.addColorStop(1,'#bcd9df');

  ctx.fillStyle = grd;
  roundRect(-22,-40,44,80,10,true,false);

  // sleek stripe
  ctx.fillStyle = 'rgba(6,24,34,0.85)';
  ctx.fillRect(-22,-12,44,8);

  // headlights
  ctx.fillStyle = '#fff';
  ctx.fillRect(-18,-30,8,6);
  ctx.fillRect(10,-30,8,6);

  // tinted windshield
  ctx.fillStyle = 'rgba(4,20,30,0.7)';
  ctx.fillRect(-16,-24,32,18);

  // wheels (simple)
  drawWheel(-16,28);
  drawWheel(16,28);
  drawWheel(-16,-28);
  drawWheel(16,-28);

  // Grill hint (luxury feel)
  ctx.fillStyle = 'rgba(2,10,14,0.9)';
  ctx.fillRect(-10,6,20,6);

  ctx.restore();
}

function drawWheel(cx,cy){
  ctx.save();
  ctx.translate(cx,cy);
  ctx.beginPath();
  ctx.ellipse(0,0,6,10,0,0,Math.PI*2);
  ctx.fillStyle = '#0e1114';
  ctx.fill();
  ctx.beginPath();
  ctx.ellipse(0,0,3,6,0,0,Math.PI*2);
  ctx.fillStyle = '#666';
  ctx.fill();
  ctx.restore();
}

// helper to draw rounded rect
function roundRect(x,y,w,h,r, fill, stroke){
  if (typeof r === 'undefined') r = 5;
  ctx.beginPath();
  ctx.moveTo(x+r,y);
  ctx.arcTo(x+w,y,x+w,y+h,r);
  ctx.arcTo(x+w,y+h,x,y+h,r);
  ctx.arcTo(x,y+h,x,y,r);
  ctx.arcTo(x,y,x+w,y,r);
  ctx.closePath();
  if (fill) ctx.fill();
  if (stroke) ctx.stroke();
}

// Mini-map rendering (small top-right)
function drawMiniMap(){
  const mw = 220, mh = 120;
  const mx = W - mw - 20, my = 20;
  ctx.save();
  ctx.globalAlpha = 0.95;
  // bg
  roundRect(mx,my,mw,mh,12,true,false);
  ctx.fillStyle = 'rgba(3,9,12,0.6)';
  ctx.fill();

  // draw scaled track
  const scale = 0.12;
  ctx.translate(mx + 12, my + 12);
  ctx.scale(scale, scale);

  // track outer filled
  ctx.beginPath();
  track.outer.forEach((p,i)=> i===0?ctx.moveTo(p.x,p.y):ctx.lineTo(p.x,p.y));
  ctx.closePath();
  ctx.fillStyle = '#071b25';
  ctx.fill();

  // inner cutout
  ctx.globalCompositeOperation = 'destination-out';
  ctx.beginPath();
  track.inner.forEach((p,i)=> i===0?ctx.moveTo(p.x,p.y):ctx.lineTo(p.x,p.y));
  ctx.closePath();
  ctx.fill();

  ctx.globalCompositeOperation = 'source-over';
  // draw player dot
  ctx.fillStyle = '#7ef2ff';
  ctx.beginPath();
  ctx.arc(player.x, player.y, 10,0,Math.PI*2);
  ctx.fill();

  // other cars
  otherCars.forEach(c=>{
    ctx.fillStyle = c.color;
    ctx.beginPath();
    ctx.arc(c.x, c.y, 8,0,Math.PI*2);
    ctx.fill();
  });

  ctx.resetTransform();
  ctx.globalAlpha = 1;
  ctx.restore();
}

function drawSpeedometer(){
  const w = 140, h = 84;
  const x = 20, y = H - h - 20;
  ctx.save();
  // bg
  roundRect(x,y,w,h,12,true,false);
  ctx.fillStyle = 'rgba(3,9,12,0.6)';
  ctx.fill();

  // speed big
  ctx.fillStyle = '#fff';
  ctx.font = '24px Inter, Arial';
  ctx.textAlign = 'left';
  ctx.fillText(Math.round(player.speed*18) + ' km/h', x+12, y+36);

  // mini bar
  const pct = Math.min(1, Math.max(0, player.speed / player.maxSpeed));
  ctx.fillStyle = '#223';
  ctx.fillRect(x+12, y+46, w-24, 10);
  ctx.fillStyle = '#4ee0d6';
  ctx.fillRect(x+12, y+46, (w-24)*pct, 10);

  ctx.restore();
}

// initial setup
resetGame();
adjustDifficulty();
requestAnimationFrame(loop);
</script>
</body>
</html>
