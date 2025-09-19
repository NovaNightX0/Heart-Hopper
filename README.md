<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Adorable Pixel Heart Collector</title>
<style>
  html,body { height:100%; margin:0; display:flex; align-items:center; justify-content:center; background: linear-gradient(180deg,#ffd6e8,#fff); font-family: "Segoe UI", Roboto, Arial; }
  #gameWrap { text-align:center; }
  canvas { 
    image-rendering: pixelated; /* crisp pixels */
    background: #fef7fb;
    border-radius:12px;
    box-shadow: 0 8px 30px rgba(0,0,0,0.10);
    width: 640px; /* scaled up for visibility */
    height: 640px;
  }
  #hud { margin-top:12px; color:#4a2a3a; font-weight:700; letter-spacing:0.5px; }
  button { margin-left:10px; padding:8px 12px; border-radius:8px; border:none; background:#ff8ab0; color:white; cursor:pointer; font-weight:600; }
  button:active { transform:translateY(1px); }
  .small { font-weight:400; color:#6a4050; font-size:13px; margin-top:6px; }
</style>
</head>
<body>
<div id="gameWrap">
  <canvas id="c" width="160" height="160"></canvas>
  <div id="hud">
    Score: <span id="score">0</span>
    <button id="restart">Restart</button>
    <div class="small">Move with arrow keys or WASD — collect hearts! 💖</div>
  </div>
</div>

<script>
// Cute Pixel Heart Collector
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');
const SCORE = document.getElementById('score');
const RESTART = document.getElementById('restart');

const SCALE = 4;            // internal scale for drawing (each pixel drawn is SCALE px)
const VIRTUAL = 40;         // virtual width/height in "pixels" (square)
canvas.width = VIRTUAL;
canvas.height = VIRTUAL;

// Game state
let keys = {};
let score = 0;
let hearts = [];
let particles = [];
let frame = 0;
let gameRunning = true;

// Simple adorable 8x8 player sprite (palette indexes)
// null = transparent
const palette = {
  0: null,
  1: '#ffb3d3', // pink
  2: '#ff6fa1', // darker pink
  3: '#fff3f8', // highlight
  4: '#60304d', // outline / eyes
  5: '#fff'     // white for eyes
};

// 8x8 sprite using palette keys
const playerSprite = [
  [0,0,1,1,1,1,0,0],
  [0,1,2,1,1,2,1,0],
  [1,2,2,2,2,2,2,1],
  [1,2,3,2,2,3,2,1],
  [1,2,2,2,2,2,2,1],
  [0,1,2,4,4,2,1,0],
  [0,0,1,5,5,1,0,0],
  [0,0,0,1,1,0,0,0]
];

// Player
let player = {
  x: VIRTUAL/2 - 4,
  y: VIRTUAL/2 - 4,
  w: 8,
  h: 8,
  speed: 0.9
};

// generate hearts
function spawnHeart() {
  const size = 5;
  const x = Math.random() * (VIRTUAL - size - 2) + 1;
  const y = Math.random() * (VIRTUAL - size - 2) + 1;
  const bob = Math.random()*Math.PI*2;
  hearts.push({x,y,size,bob,angle:0});
}

// draw single 8x8 sprite scaled to pixel grid
function drawSprite(sprite, px, py) {
  for (let r=0;r<sprite.length;r++){
    for (let c=0;c<sprite[r].length;c++){
      const p = sprite[r][c];
      if (!p || palette[p]===null) continue;
      ctx.fillStyle = palette[p];
      ctx.fillRect(px + c, py + r, 1, 1);
    }
  }
}

// small heart shape drawn with colored pixels (5x5)
function drawHeart(x,y,t) {
  // little animate scale by t
  const s = 1 + 0.08 * Math.sin(t*0.2);
  ctx.save();
  ctx.translate(x+2.5, y+2.5);
  ctx.scale(s,s);
  ctx.translate(- (x+2.5), - (y+2.5));
  // manual pixels for a heart (5x5)
  const heart = [
    [0,1,1,0,0],
    [1,1,1,1,0],
    [1,1,1,1,1],
    [0,1,1,1,1],
    [0,0,1,1,0]
  ];
  for (let r=0;r<5;r++){
    for (let c=0;c<5;c++){
      if (heart[r][c]) {
        ctx.fillStyle = '#ff6fa1';
        ctx.fillRect(x + c, y + r, 1, 1);
        // tiny shine
        if ((r+c)%4===0) {
          ctx.fillStyle = '#ffb3d3';
          ctx.fillRect(x + c, y + r, 1, 1);
        }
      }
    }
  }
  ctx.restore();
}

// particles for cute pop effect
function spawnParticles(cx,cy,amount=12) {
  for(let i=0;i<amount;i++){
    const a = Math.random()*Math.PI*2;
    const s = 0.5 + Math.random()*1.5;
    particles.push({
      x: cx + Math.cos(a)*2,
      y: cy + Math.sin(a)*2,
      vx: Math.cos(a)*(0.8+Math.random()*1.6),
      vy: Math.sin(a)*(0.8+Math.random()*1.6) - 0.2,
      life: 30 + Math.random()*30,
      size: s
    });
  }
}

// sound: tiny chime using WebAudio
let audioCtx;
function playChime() {
  try {
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    const t = audioCtx.currentTime;
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.connect(gain); gain.connect(audioCtx.destination);
    osc.type = 'sine';
    osc.frequency.setValueAtTime(880, t);
    osc.frequency.exponentialRampToValueAtTime(1320, t+0.18);
    gain.gain.setValueAtTime(0.001, t);
    gain.gain.exponentialRampToValueAtTime(0.09, t+0.02);
    gain.gain.exponentialRampToValueAtTime(0.001, t+0.25);
    osc.start(t);
    osc.stop(t+0.26);
  } catch(e) { /* ignore if audio blocked */ }
}

// handle keyboard
window.addEventListener('keydown', (e)=>{ keys[e.key.toLowerCase()] = true; if (["ArrowUp","ArrowDown","ArrowLeft","ArrowRight"].includes(e.key)) e.preventDefault(); });
window.addEventListener('keyup', (e)=>{ keys[e.key.toLowerCase()] = false; });

RESTART.addEventListener('click', ()=>{ reset(); });

// basic collision AABB
function coll(a,b){
  return !(a.x + a.w <= b.x || a.x >= b.x + b.size || a.y + a.h <= b.y || a.y >= b.y + b.size);
}

// game reset
function reset(){
  score = 0;
  hearts = [];
  particles = [];
  player.x = VIRTUAL/2 - player.w/2;
  player.y = VIRTUAL/2 - player.h/2;
  for (let i=0;i<5;i++) spawnHeart();
  SCORE.textContent = score;
  gameRunning = true;
}

// update
function update(){
  frame++;
  // movement
  let dx = 0, dy = 0;
  if (keys['arrowleft'] || keys['a']) dx -= 1;
  if (keys['arrowright'] || keys['d']) dx += 1;
  if (keys['arrowup'] || keys['w']) dy -= 1;
  if (keys['arrowdown'] || keys['s']) dy += 1;

  // normalize diagonal
  if (dx !==0 && dy !==0) { dx *= Math.SQRT1_2; dy *= Math.SQRT1_2; }

  player.x += dx * player.speed;
  player.y += dy * player.speed;

  // clamp to bounds
  player.x = Math.max(0, Math.min(VIRTUAL - player.w, player.x));
  player.y = Math.max(0, Math.min(VIRTUAL - player.h, player.y));

  // hearts bob animation
  hearts.forEach(h=>{
    h.bob += 0.06;
    h.angle = Math.sin(h.bob)*0.6;
  });

  // check collisions
  for (let i = hearts.length-1; i>=0; i--) {
    const h = hearts[i];
    const hit = coll(player, h);
    if (hit) {
      hearts.splice(i,1);
      score++;
      SCORE.textContent = score;
      spawnParticles(h.x + h.size/2, h.y + h.size/2, 12);
      playChime();
      // spawn replacement with small chance of 2 hearts
      spawnHeart();
      if (Math.random() < 0.08) spawnHeart();
    }
  }

  // particles update
  for (let i = particles.length-1; i>=0; i--) {
    const p = particles[i];
    p.x += p.vx * 0.12;
    p.y += p.vy * 0.12;
    p.vy += 0.03;
    p.life -= 1;
    if (p.life <= 0) particles.splice(i,1);
  }

  // keep at least 4 hearts
  if (hearts.length < 4 && frame % 60 === 0) spawnHeart();
}

// draw everything (on the VIRTUAL canvas)
function draw(){
  // clear
  ctx.clearRect(0,0,canvas.width,canvas.height);

  // soft vignette background pixels
  for (let y=0;y<VIRTUAL;y+=6){
    for (let x=0;x<VIRTUAL;x+=6){
      // faint pastel dots
      ctx.fillStyle = `rgba(255, 210, 235, ${(0.01*Math.sin((x+y+frame)/20) + 0.02).toFixed(3)})`;
      ctx.fillRect(x, y, 1, 1);
    }
  }

  // hearts
  hearts.forEach(h=>{
    drawHeart(h.x, h.y + Math.sin(h.bob)*0.7, frame + h.bob*10);
  });

  // particles (tiny dots)
  particles.forEach(p=>{
    ctx.fillStyle = `rgba(255,110,150,${Math.max(0, p.life/60)})`;
    ctx.fillRect(p.x, p.y, p.size, p.size);
  });

  // player shadow
  ctx.fillStyle = 'rgba(0,0,0,0.06)';
  ctx.fillRect(Math.round(player.x)+1, Math.round(player.y)+player.h+1, player.w-2, 1);

  // draw the player sprite
  drawSprite(playerSprite, Math.round(player.x), Math.round(player.y));

  // cute floating text
  ctx.fillStyle = '#7a3e52';
  ctx.font = '3px sans-serif';
  ctx.fillText('💕', 1, 4);
}

// main loop (draw to small canvas, browser scales it up)
function loop(){
  update();
  draw();
  requestAnimationFrame(loop);
}

// initialization tune: spawn hearts and start
reset();
for (let i=0;i<3;i++) spawnHeart();
loop();

</script>
</body>
</html>
