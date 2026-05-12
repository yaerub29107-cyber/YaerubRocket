<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>NEON STRIKER - Online</title>
<link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700;900&family=Share+Tech+Mono&display=swap" rel="stylesheet">
<!-- PeerJS for P2P WebRTC connections -->
<script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>
<style>
  :root {
    --bg: #0a0a12; --fg: #e0e8ff; --accent: #00ffa3;
    --p1: #ff4466; --p2: #4488ff; --ball: #ffe44d; --muted: #2a2a3a;
  }
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    background: var(--bg); color: var(--fg); font-family: 'Share Tech Mono', monospace;
    overflow: hidden; height: 100vh; width: 100vw; display: flex; flex-direction: column;
    align-items: center; justify-content: center; user-select: none;
  }
  canvas { display: block; position: absolute; top: 0; left: 0; width: 100%; height: 100%; }
  
  .overlay {
    position: fixed; inset: 0; display: flex; flex-direction: column; align-items: center;
    justify-content: center; z-index: 10;
    background: radial-gradient(ellipse at center, rgba(0,255,163,0.02) 0%, rgba(10,10,18,0.97) 70%);
    transition: opacity 0.4s ease;
  }
  .overlay.hidden { opacity: 0; pointer-events: none; }

  .game-title { font-family: 'Orbitron', sans-serif; font-weight: 900; font-size: clamp(2rem, 6vw, 4rem); color: var(--fg); letter-spacing: 0.1em; margin-bottom: 0.5em; }
  .game-subtitle { font-size: clamp(0.7rem, 1.5vw, 0.9rem); color: var(--muted); letter-spacing: 0.3em; text-transform: uppercase; margin-bottom: 2em; }

  .btn {
    font-family: 'Orbitron', sans-serif; font-weight: 700; font-size: clamp(0.8rem, 1.5vw, 1rem);
    color: var(--bg); background: var(--accent); border: none; padding: 0.8em 2em; cursor: pointer;
    letter-spacing: 0.1em; clip-path: polygon(6% 0%, 100% 0%, 94% 100%, 0% 100%); transition: transform 0.15s;
  }
  .btn:hover { transform: scale(1.05); } .btn:active { transform: scale(0.97); }
  .btn:disabled { background: var(--muted); cursor: not-allowed; transform: none; }

  .lobby-section { display: flex; gap: 2em; margin-bottom: 2em; flex-wrap: wrap; justify-content: center; }
  .lobby-box {
    background: rgba(255,255,255,0.03); border: 1px solid rgba(255,255,255,0.08); padding: 1.5em;
    border-radius: 12px; display: flex; flex-direction: column; align-items: center; gap: 1em; width: 280px;
  }
  .lobby-title { font-family: 'Orbitron', sans-serif; font-weight: 700; font-size: 1.1rem; }
  .lobby-title.p1 { color: var(--p1); } .lobby-title.p2 { color: var(--p2); }
  
  input[type="text"] {
    font-family: 'Orbitron', sans-serif; background: rgba(0,0,0,0.5); border: 1px solid var(--muted);
    color: var(--fg); padding: 0.6em; border-radius: 6px; text-align: center; font-size: 1.1rem;
    width: 100%; letter-spacing: 0.1em;
  }
  input:focus { outline: none; border-color: var(--accent); }
  
  .status-text { font-size: 0.8rem; color: var(--muted); text-align: center; min-height: 1.5em; }
  .code-display { font-size: 1.8rem; font-weight: 900; color: var(--accent); letter-spacing: 0.15em; }

  .hud {
    position: fixed; top: 0; left: 0; right: 0; z-index: 5; display: flex; justify-content: center;
    align-items: center; padding: 1em 2em; pointer-events: none; font-family: 'Orbitron', sans-serif; gap: 1.2em;
  }
  .hud.hidden { display: none; }
  .score-box { font-size: clamp(2rem, 6vw, 3.5rem); font-weight: 900; min-width: 1.5em; text-align: center; }
  .score-p1 { color: var(--p1); text-shadow: 0 0 20px rgba(255,68,102,0.4); }
  .score-p2 { color: var(--p2); text-shadow: 0 0 20px rgba(68,136,255,0.4); }
  .score-divider { color: var(--muted); font-size: clamp(1.5rem, 4vw, 2.5rem); }
  .timer-box {
    font-size: clamp(1rem, 3vw, 1.8rem); font-weight: 700; color: var(--fg); background: rgba(0,0,0,0.4);
    padding: 0.3em 0.8em; border-radius: 8px; border: 1px solid rgba(255,255,255,0.1); min-width: 4em; text-align: center;
  }
  .timer-box.danger { color: var(--p1); border-color: rgba(255,68,102,0.5); animation: pulse 1s infinite; }
  @keyframes pulse { 50% { opacity: 0.6; } }

  .goal-flash { position: fixed; inset: 0; z-index: 8; pointer-events: none; opacity: 0; transition: opacity 0.1s; }
  .goal-flash.p1 { background: radial-gradient(circle, rgba(255,68,102,0.3), transparent 70%); }
  .goal-flash.p2 { background: radial-gradient(circle, rgba(68,136,255,0.3), transparent 70%); }
  .goal-flash.active { opacity: 1; }
  .goal-text {
    position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%); z-index: 9;
    font-family: 'Orbitron', sans-serif; font-weight: 900; font-size: clamp(3rem, 10vw, 6rem);
    letter-spacing: 0.15em; pointer-events: none; opacity: 0; transition: opacity 0.15s, transform 0.3s;
  }
  .goal-text.show { opacity: 1; transform: translate(-50%, -50%) scale(1.1); }

  .restart-btn {
    position: fixed; bottom: 1.5em; right: 1.5em; z-index: 6; font-family: 'Orbitron', sans-serif;
    font-weight: 700; font-size: 0.75rem; color: var(--muted); background: rgba(255,255,255,0.05);
    border: 1px solid rgba(255,255,255,0.1); padding: 0.6em 1.2em; border-radius: 6px; cursor: pointer;
    letter-spacing: 0.1em; transition: all 0.2s ease; display: none;
  }
  .restart-btn.visible { display: block; }
  .restart-btn:hover { color: var(--fg); background: rgba(255,255,255,0.1); border-color: var(--accent); }
</style>
</head>
<body>

<canvas id="gameCanvas"></canvas>

<div class="goal-flash p1" id="flashP1"></div>
<div class="goal-flash p2" id="flashP2"></div>
<div class="goal-text" id="goalText">GOAL</div>
<button class="restart-btn" id="restartBtn">REMATCH (HOST ONLY)</button>

<!-- Lobby Screen -->
<div class="overlay" id="lobbyScreen">
  <div class="game-title">NEON STRIKER</div>
  <div class="game-subtitle">Online Multiplayer</div>
  
  <div class="lobby-section">
    <!-- Host Box -->
    <div class="lobby-box">
      <div class="lobby-title p1">HOST GAME</div>
      <div class="status-text" id="hostStatus">Create a lobby for others to join.</div>
      <div class="code-display hidden" id="roomCode">----</div>
      <button class="btn" id="hostBtn">CREATE ROOM</button>
    </div>

    <!-- Join Box -->
    <div class="lobby-box">
      <div class="lobby-title p2">JOIN GAME</div>
      <div class="status-text" id="joinStatus">Enter the 4-digit code from the host.</div>
      <input type="text" id="joinInput" placeholder="CODE" maxlength="4">
      <button class="btn" id="joinBtn">JOIN ROOM</button>
    </div>
  </div>
</div>

<!-- Win Screen -->
<div class="overlay hidden" id="winScreen">
  <div class="game-title" id="winTitle">TIME UP!</div>
  <div class="game-subtitle" id="winSub">Player 1 Wins</div>
  <button class="btn" id="retryBtn">REMATCH</button>
</div>

<!-- HUD -->
<div class="hud hidden" id="hud">
  <div class="score-box score-p1" id="scoreP1">0</div>
  <div class="score-divider">:</div>
  <div class="score-box score-p2" id="scoreP2">0</div>
  <div class="timer-box" id="timerBox">3:00</div>
</div>

<script>
// ============================================================
// NEON STRIKER — Peer-to-Peer Online Edition
// ============================================================

const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// DOM Refs
const lobbyScreen = document.getElementById('lobbyScreen');
const winScreen = document.getElementById('winScreen');
const hud = document.getElementById('hud');
const scoreP1El = document.getElementById('scoreP1');
const scoreP2El = document.getElementById('scoreP2');
const flashP1 = document.getElementById('flashP1');
const flashP2 = document.getElementById('flashP2');
const goalText = document.getElementById('goalText');
const winTitle = document.getElementById('winTitle');
const winSub = document.getElementById('winSub');
const restartBtn = document.getElementById('restartBtn');
const timerBox = document.getElementById('timerBox');
const hostStatus = document.getElementById('hostStatus');
const joinStatus = document.getElementById('joinStatus');
const roomCodeEl = document.getElementById('roomCode');
const joinInput = document.getElementById('joinInput');

let W, H;
function resize() { W = canvas.width = window.innerWidth; H = canvas.height = window.innerHeight; setupField(); }
window.addEventListener('resize', resize);

// ---- Game Constants ----
const FIELD_W = 1200; const FIELD_H = 700; const GOAL_DEPTH = 40; const GOAL_WIDTH = 200;
const CAR_W = 40; const CAR_H = 22; const BALL_R = 12;
const CAR_ACCEL = 650; const CAR_FRICTION = 0.94; const CAR_TURN_SPEED = 3.5;
const BALL_FRICTION = 0.995; const CAR_BOUNCE = 0.4; const BALL_BOUNCE = 0.8; const HIT_FORCE = 350;

// ---- State ----
let state = 'menu'; // menu, playing, goal, gameover
let scores = [0, 0]; let goalPauseTimer = 0; let particles = [];
let matchDuration = 180; let timeRemaining = 0;
let fieldX = 0, fieldY = 0, scale = 1;
let isHost = false; let peer = null; let conn = null;
let networkKeys = { up: false, down: false, left: false, right: false };

const cars = [
  { x: 0, y: 0, vx: 0, vy: 0, angle: 0, color: '#ff4466', glow: 'rgba(255,68,102,0.4)' },
  { x: 0, y: 0, vx: 0, vy: 0, angle: Math.PI, color: '#4488ff', glow: 'rgba(68,136,255,0.4)' }
];
const ball = { x: 0, y: 0, vx: 0, vy: 0 };
const spawnPoints = [ { x: -FIELD_W/4, y: 0, angle: 0 }, { x: FIELD_W/4, y: 0, angle: Math.PI } ];

function setupField() {
  const scaleX = W / (FIELD_W + 100); const scaleY = H / (FIELD_H + 100);
  scale = Math.min(scaleX, scaleY); fieldX = (W - FIELD_W * scale) / 2; fieldY = (H - FIELD_H * scale) / 2;
}

// ---- Networking (PeerJS) ----
function generateCode() { return Math.random().toString(36).substring(2, 6).toUpperCase(); }

document.getElementById('hostBtn').addEventListener('click', () => {
  const code = generateCode();
  peer = new Peer('neonstriker-' + code); // Create peer with custom ID
  
  peer.on('open', (id) => {
    isHost = true;
    roomCodeEl.textContent = code;
    roomCodeEl.classList.remove('hidden');
    hostStatus.textContent = "Waiting for opponent...";
    document.getElementById('hostBtn').disabled = true;
  });

  peer.on('connection', (connection) => {
    conn = connection;
    setupConnection();
    hostStatus.textContent = "Opponent connected! Starting match...";
    setTimeout(startGame, 1000);
  });

  peer.on('error', (err) => {
    hostStatus.textContent = "Error: " + err.type;
    if(err.type === 'unavailable-id') hostStatus.textContent = "Room code taken. Try again.";
  });
});

document.getElementById('joinBtn').addEventListener('click', () => {
  const code = joinInput.value.trim().toUpperCase();
  if (code.length !== 4) { joinStatus.textContent = "Please enter a valid 4-digit code."; return; }
  
  peer = new Peer(); // Create peer with random ID
  joinStatus.textContent = "Connecting...";

  peer.on('open', () => {
    conn = peer.connect('neonstriker-' + code);
    isHost = false;
    setupConnection();
  });

  peer.on('error', (err) => {
    joinStatus.textContent = "Error: " + err.type;
    if(err.type === 'peer-unavailable') joinStatus.textContent = "Room not found!";
  });
});

function setupConnection() {
  conn.on('open', () => { if(!isHost) joinStatus.textContent = "Connected!"; });
  
  conn.on('data', (data) => {
    if (isHost) {
      // Host receives opponent inputs
      if (data.type === 'input') { networkKeys = data.keys; }
      if (data.type === 'restart') { startGame(); }
    } else {
      // Client receives game state from host
      if (data.type === 'start') { startGame(); }
      if (data.type === 'state') { applyNetworkState(data.payload); }
      if (data.type === 'goal') { triggerGoalVisual(data.playerIndex); }
      if (data.type === 'gameover') { showWinner(data.winnerIndex); }
    }
  });

  conn.on('close', () => {
    alert("Opponent disconnected.");
    location.reload(); // Simple reset on disconnect
  });
}

// ---- Input System ----
const localKeys = {};
window.addEventListener('keydown', e => {
  localKeys[e.code] = true;
  if (e.code === 'KeyR' && isHost && (state === 'playing' || state === 'goal')) { startGame(); sendRestart(); }
  if(['ArrowUp','ArrowDown','ArrowLeft','ArrowRight','Space'].includes(e.code)) e.preventDefault();
});
window.addEventListener('keyup', e => { localKeys[e.code] = false; });

function getLocalInput() {
  return { up: !!localKeys['KeyW'], down: !!localKeys['KeyS'], left: !!localKeys['KeyA'], right: !!localKeys['KeyD'] };
}

// ---- Physics & Collisions ----
function updateCar(car, inputs, dt) {
  let moveInput = 0, turnInput = 0;
  if (inputs.up) moveInput = 1; if (inputs.down) moveInput = -1;
  if (inputs.left) turnInput = -1; if (inputs.right) turnInput = 1;

  if (moveInput !== 0) car.angle += turnInput * CAR_TURN_SPEED * dt * moveInput;
  car.vx += Math.cos(car.angle) * moveInput * CAR_ACCEL * dt;
  car.vy += Math.sin(car.angle) * moveInput * CAR_ACCEL * dt;
  car.vx *= CAR_FRICTION; car.vy *= CAR_FRICTION;
  car.x += car.vx * dt; car.y += car.vy * dt;

  if (Math.abs(moveInput) > 0 && Math.random() < 0.5) {
    spawnParticle(car.x - Math.cos(car.angle) * CAR_W/2, car.y - Math.sin(car.angle) * CAR_W/2, car.color, 1.5);
  }
}

function updateBall(dt) {
  ball.vx *= BALL_FRICTION; ball.vy *= BALL_FRICTION;
  ball.x += ball.vx * dt; ball.y += ball.vy * dt;

  if (ball.y < -FIELD_H/2 + BALL_R) { ball.y = -FIELD_H/2 + BALL_R; ball.vy *= -BALL_BOUNCE; }
  if (ball.y > FIELD_H/2 - BALL_R) { ball.y = FIELD_H/2 - BALL_R; ball.vy *= -BALL_BOUNCE; }

  const goalTop = -GOAL_WIDTH/2; const goalBot = GOAL_WIDTH/2;
  if (ball.x < -FIELD_W/2 + BALL_R) {
    if (ball.y > goalTop && ball.y < goalBot) { return 1; } // Goal P2
    else { ball.x = -FIELD_W/2 + BALL_R; ball.vx *= -BALL_BOUNCE; }
  }
  if (ball.x > FIELD_W/2 - BALL_R) {
    if (ball.y > goalTop && ball.y < goalBot) { return 0; } // Goal P1
    else { ball.x = FIELD_W/2 - BALL_R; ball.vx *= -BALL_BOUNCE; }
  }
  return -1;
}

function carBallCollision(car) {
  const dx = ball.x - car.x; const dy = ball.y - car.y; const dist = Math.sqrt(dx*dx + dy*dy);
  const minDist = BALL_R + CAR_W/2;
  if (dist < minDist && dist > 0) {
    const overlap = minDist - dist; const nx = dx / dist; const ny = dy / dist;
    ball.x += nx * overlap; ball.y += ny * overlap;
    ball.vx = ball.vx*0.2 + (car.vx + nx*HIT_FORCE)*0.8; ball.vy = ball.vy*0.2 + (car.vy + ny*HIT_FORCE)*0.8;
    for(let i=0; i<8; i++) spawnParticle(ball.x, ball.y, '#ffe44d', 2);
  }
}

function carWallCollision(car) {
  const hw = CAR_W/2; const hh = CAR_H/2;
  if (car.x < -FIELD_W/2 + hw) { car.x = -FIELD_W/2 + hw; car.vx *= -CAR_BOUNCE; }
  if (car.x > FIELD_W/2 - hw) { car.x = FIELD_W/2 - hw; car.vx *= -CAR_BOUNCE; }
  if (car.y < -FIELD_H/2 + hh) { car.y = -FIELD_H/2 + hh; car.vy *= -CAR_BOUNCE; }
  if (car.y > FIELD_H/2 - hh) { car.y = FIELD_H/2 - hh; car.vy *= -CAR_BOUNCE; }
}

function resetPositions() {
  cars[0].x = spawnPoints[0].x; cars[0].y = spawnPoints[0].y; cars[0].vx = 0; cars[0].vy = 0; cars[0].angle = 0;
  cars[1].x = spawnPoints[1].x; cars[1].y = spawnPoints[1].y; cars[1].vx = 0; cars[1].vy = 0; cars[1].angle = Math.PI;
  ball.x = 0; ball.y = 0; ball.vx = 0; ball.vy = 0;
}

// ---- Score & Timer ----
function triggerGoalVisual(playerIndex) {
  state = 'goal'; goalPauseTimer = 1.5;
  scoreP1El.textContent = scores[0]; scoreP2El.textContent = scores[1];
  const flashEl = playerIndex === 0 ? flashP1 : flashP2;
  flashEl.classList.add('active');
  goalText.style.color = playerIndex === 0 ? 'var(--p1)' : 'var(--p2)';
  goalText.classList.add('show');
  for(let i=0; i<40; i++) spawnParticle(ball.x, ball.y, playerIndex === 0 ? '#ff4466' : '#4488ff', 3);
  setTimeout(() => { flashEl.classList.remove('active'); goalText.classList.remove('show'); }, 800);
}

function updateTimer(dt) {
  if (state === 'playing') {
    timeRemaining -= dt;
    if (timeRemaining <= 0) { timeRemaining = 0; endGame(); }
  }
  const mins = Math.floor(timeRemaining / 60); const secs = Math.floor(timeRemaining % 60);
  timerBox.textContent = `${mins}:${secs < 10 ? '0' : ''}${secs}`;
  if (timeRemaining <= 30 && timeRemaining > 0) timerBox.classList.add('danger');
  else timerBox.classList.remove('danger');
}

// ---- Particles ----
function spawnParticle(x, y, color, size) {
  const angle = Math.random() * Math.PI * 2; const speed = 50 + Math.random() * 150;
  particles.push({ x, y, vx: Math.cos(angle)*speed, vy: Math.sin(angle)*speed, life: 1, decay: 1+Math.random()*2, size: size||2, color });
}
function updateParticles(dt) {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]; p.x += p.vx*dt; p.y += p.vy*dt; p.vx *= 0.95; p.vy *= 0.95; p.life -= p.decay*dt;
    if (p.life <= 0) particles.splice(i, 1);
  }
}

// ---- Drawing ----
function toScreen(x, y) { return { x: fieldX + (x + FIELD_W/2)*scale, y: fieldY + (y + FIELD_H/2)*scale }; }

function drawField() {
  ctx.fillStyle = '#0e1420'; const tl = toScreen(-FIELD_W/2, -FIELD_H/2);
  ctx.fillRect(tl.x, tl.y, FIELD_W*scale, FIELD_H*scale);
  ctx.strokeStyle = 'rgba(255,255,255,0.15)'; ctx.lineWidth = 2*scale;
  ctx.strokeRect(tl.x, tl.y, FIELD_W*scale, FIELD_H*scale);
  ctx.beginPath(); const cl = toScreen(0, -FIELD_H/2); const cb = toScreen(0, FIELD_H/2);
  ctx.moveTo(cl.x, cl.y); ctx.lineTo(cb.x, cb.y); ctx.stroke();
  const cc = toScreen(0, 0); ctx.beginPath(); ctx.arc(cc.x, cc.y, 60*scale, 0, Math.PI*2); ctx.stroke();
  
  const goalTopS = toScreen(-FIELD_W/2, -GOAL_WIDTH/2); const goalTopE = toScreen(-FIELD_W/2, GOAL_WIDTH/2);
  ctx.fillStyle = 'rgba(68,136,255,0.1)'; ctx.fillRect(goalTopS.x - GOAL_DEPTH*scale, goalTopS.y, GOAL_DEPTH*scale, (GOAL_WIDTH)*scale);
  ctx.strokeStyle = 'rgba(68,136,255,0.5)'; ctx.lineWidth = 3*scale; ctx.beginPath();
  ctx.moveTo(goalTopS.x, goalTopS.y); ctx.lineTo(goalTopS.x - GOAL_DEPTH*scale, goalTopS.y); ctx.lineTo(goalTopE.x - GOAL_DEPTH*scale, goalTopE.y); ctx.lineTo(goalTopE.x, goalTopE.y); ctx.stroke();
  
  const goalBR = toScreen(FIELD_W/2, GOAL_WIDTH/2); ctx.fillStyle = 'rgba(255,68,102,0.1)'; ctx.fillRect(goalBR.x, goalTopS.y, GOAL_DEPTH*scale, GOAL_WIDTH*scale);
  ctx.strokeStyle = 'rgba(255,68,102,0.5)'; ctx.beginPath(); ctx.moveTo(goalTopS.x + FIELD_W*scale, goalTopS.y); ctx.lineTo(goalTopS.x + (FIELD_W+GOAL_DEPTH)*scale, goalTopS.y); ctx.lineTo(goalTopE.x + (FIELD_W+GOAL_DEPTH)*scale, goalTopE.y); ctx.lineTo(goalTopE.x + FIELD_W*scale, goalTopE.y); ctx.stroke();
}

function drawCar(car) {
  const p = toScreen(car.x, car.y); ctx.save(); ctx.translate(p.x, p.y); ctx.rotate(car.angle);
  ctx.shadowBlur = 15*scale; ctx.shadowColor = car.glow;
  ctx.fillStyle = car.color; ctx.fillRect(-CAR_W/2*scale, -CAR_H/2*scale, CAR_W*scale, CAR_H*scale);
  ctx.fillStyle = 'rgba(255,255,255,0.3)'; ctx.fillRect(CAR_W/4*scale, -CAR_H/2*scale, CAR_W/5*scale, CAR_H*scale);
  ctx.shadowBlur = 0; ctx.restore();
}

function drawBall() {
  const p = toScreen(ball.x, ball.y); const grad = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, Math.max(1, BALL_R*3*scale));
  grad.addColorStop(0, 'rgba(255,228,77,0.3)'); grad.addColorStop(1, 'rgba(255,228,77,0)');
  ctx.fillStyle = grad; ctx.beginPath(); ctx.arc(p.x, p.y, Math.max(1, BALL_R*3*scale), 0, Math.PI*2); ctx.fill();
  ctx.fillStyle = '#ffe44d'; ctx.beginPath(); ctx.arc(p.x, p.y, Math.max(1, BALL_R*scale), 0, Math.PI*2); ctx.fill();
}

function drawParticles() {
  for (const p of particles) { const sp = toScreen(p.x, p.y); ctx.globalAlpha = Math.max(0, p.life); ctx.fillStyle = p.color; ctx.beginPath(); ctx.arc(sp.x, sp.y, Math.max(0.5, p.size*p.life*scale), 0, Math.PI*2); ctx.fill(); }
  ctx.globalAlpha = 1;
}

// ---- Game Flow ----
function startGame() {
  scores = [0, 0]; scoreP1El.textContent = '0'; scoreP2El.textContent = '0';
  timeRemaining = matchDuration; timerBox.classList.remove('danger');
  resetPositions(); particles = []; state = 'playing';
  lobbyScreen.classList.add('hidden'); winScreen.classList.add('hidden');
  hud.classList.remove('hidden'); restartBtn.classList.add('visible');
}

function endGame() {
  let winnerIndex = -1; // Draw
  if (scores[0] > scores[1]) winnerIndex = 0;
  else if (scores[1] > scores[0]) winnerIndex = 1;
  
  if (isHost) { sendGameOver(winnerIndex); }
  showWinner(winnerIndex);
}

function showWinner(winnerIndex) {
  state = 'gameover'; let winnerText = ''; let winnerColor = '';
  if (winnerIndex === 0) { winnerText = 'Player 1 (Red) Wins!'; winnerColor = 'var(--p1)'; }
  else if (winnerIndex === 1) { winnerText = 'Player 2 (Blue) Wins!'; winnerColor = 'var(--p2)'; }
  else { winnerText = 'Draw!'; winnerColor = 'var(--accent)'; }
  winTitle.textContent = 'TIME UP!'; winTitle.style.color = winnerColor; winTitle.style.textShadow = `0 0 30px ${winnerColor}`;
  winSub.textContent = winnerText; hud.classList.add('hidden'); restartBtn.classList.remove('visible'); winScreen.classList.remove('hidden');
}

// ---- Network Sync ----
function sendInput() {
  if (!isHost && conn && conn.open) { conn.send({ type: 'input', keys: getLocalInput() }); }
}

function sendState() {
  if (isHost && conn && conn.open) {
    conn.send({
      type: 'state',
      payload: { c0: { x: cars[0].x, y: cars[0].y, vx: cars[0].vx, vy: cars[0].vy, a: cars[0].angle },
                 c1: { x: cars[1].x, y: cars[1].y, vx: cars[1].vx, vy: cars[1].vy, a: cars[1].angle },
                 b: { x: ball.x, y: ball.y, vx: ball.vx, vy: ball.vy },
                 s: scores, t: timeRemaining, st: state }
    });
  }
}

function applyNetworkState(s) {
  // Interpolate to smooth out network jitter
  const lerpF = 0.4;
  cars[0].x = lerp(cars[0].x, s.c0.x, lerpF); cars[0].y = lerp(cars[0].y, s.c0.y, lerpF);
  cars[0].vx = s.c0.vx; cars[0].vy = s.c0.vy; cars[0].angle = s.c0.a;
  cars[1].x = lerp(cars[1].x, s.c1.x, lerpF); cars[1].y = lerp(cars[1].y, s.c1.y, lerpF);
  cars[1].vx = s.c1.vx; cars[1].vy = s.c1.vy; cars[1].angle = s.c1.a;
  ball.x = lerp(ball.x, s.b.x, lerpF); ball.y = lerp(ball.y, s.b.y, lerpF);
  ball.vx = s.b.vx; ball.vy = s.b.vy;
  scores = s.s; scoreP1El.textContent = scores[0]; scoreP2El.textContent = scores[1];
  timeRemaining = s.t; state = s.st;
  const mins = Math.floor(timeRemaining/60); const secs = Math.floor(timeRemaining%60);
  timerBox.textContent = `${mins}:${secs < 10 ? '0' : ''}${secs}`;
}

function sendGoal(playerIndex) { if (isHost && conn && conn.open) conn.send({ type: 'goal', playerIndex }); }
function sendGameOver(winnerIndex) { if (isHost && conn && conn.open) conn.send({ type: 'gameover', winnerIndex }); }
function sendRestart() { if (isHost && conn && conn.open) conn.send({ type: 'restart' }); }

// Main loop timers for network sync
let inputTimer = 0; let stateTimer = 0;

document.getElementById('retryBtn').addEventListener('click', () => {
  if (isHost) { startGame(); sendRestart(); }
  else { alert("Only the host can restart the match."); }
});
restartBtn.addEventListener('click', () => {
  if (isHost) { startGame(); sendRestart(); }
  else { alert("Only the host can restart the match."); }
});

// ---- Main Loop ----
let lastTime = performance.now();

function gameLoop(timestamp) {
  const dt = Math.min(0.05, (timestamp - lastTime) / 1000);
  lastTime = timestamp;
  ctx.fillStyle = '#06060c'; ctx.fillRect(0, 0, W, H);

  if (state === 'playing' || state === 'goal') {
    if (isHost) {
      // Host runs physics
      if (state === 'playing') {
        updateCar(cars[0], getLocalInput(), dt); // Host controls Red (P1)
        updateCar(cars[1], networkKeys, dt);     // Client controls Blue (P2)
        const goalScorer = updateBall(dt);
        if (goalScorer !== -1) {
          scores[goalScorer]++; triggerGoalVisual(goalScorer); sendGoal(goalScorer);
        }
        carBallCollision(cars[0]); carBallCollision(cars[1]);
        carWallCollision(cars[0]); carWallCollision(cars[1]);
        updateTimer(dt);
      } else if (state === 'goal') {
        goalPauseTimer -= dt; updateTimer(dt);
        if (goalPauseTimer <= 0) {
          if (timeRemaining <= 0) endGame();
          else { resetPositions(); state = 'playing'; }
        }
      }
      // Send state to client periodically
      stateTimer += dt;
      if (stateTimer > 0.05) { sendState(); stateTimer = 0; } // ~20 ticks/sec
    } else {
      // Client just sends inputs
      inputTimer += dt;
      if (inputTimer > 0.05) { sendInput(); inputTimer = 0; }
      // Client predicts its own car slightly to hide latency
      updateCar(cars[1], getLocalInput(), dt);
    }
    updateParticles(dt);
  }

  drawField(); drawParticles();
  if (state !== 'menu') { drawBall(); drawCar(cars[0]); drawCar(cars[1]); }
  requestAnimationFrame(gameLoop);
}

resize();
requestAnimationFrame(gameLoop);
</script>
</body>
</html>
