<!DOCTYPE html>
<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space-Station</title>
<style>
  body { margin:0; background:black; color:white; font-family:Arial; text-align:center; }
  canvas { background:#020824; display:block; margin:0 auto; }
  #ui { margin-top:8px; }
  button { padding:6px 12px; margin:4px; font-size:14px; }
</style>
</head>
<body>
<h2>🚀 Space‑Station</h2>
<canvas id="game" width="400" height="600"></canvas>
<div id="ui">
  <span id="info"></span><br>
  <button onclick="rebirth()">Rebirth (1000 coins)</button>
</div>

<script>
// ====== CANVAS ======
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

// ====== GAME STATE ======
let keys = {};
let gameOver = false;
let paused = false;

let score = 0;
let coins = Number(localStorage.getItem("coins")) || 100;
let gems  = Number(localStorage.getItem("gems")) || 0;

let weaponUnlocked = localStorage.getItem("weaponUnlocked") === "true";
let upgradeLevel = Number(localStorage.getItem("upgradeLevel")) || 0;

// ====== PLAYER ======
const player = {
  x: 180, y: 540, w: 40, h: 20, speed: 5
};

// ====== ARRAYS ======
let enemies = [];
let bullets = [];

// ====== SHOOTING ======
let lastShot = 0;

function getShootData() {
  switch(upgradeLevel) {
    case 0: return { shots:1, delay:500 };
    case 1: return { shots:1, delay:150 };
    case 2: return { shots:2, delay:500 };
    case 3: return { shots:2, delay:150 };
    case 4: return { shots:3, delay:500 };
    case 5: return { shots:3, delay:150 };
  }
}

function shoot() {
  if(!weaponUnlocked) return;
  const data = getShootData();
  const now = Date.now();
  if(now - lastShot < data.delay) return;
  lastShot = now;

  for(let i=0;i<data.shots;i++){
    bullets.push({
      x: player.x + player.w/2 - 2 + (i-(data.shots-1)/2)*10,
      y: player.y,
      vy: -7
    });
  }
}

// ====== INPUT ======
document.addEventListener("keydown", e=>{
  keys[e.key.toLowerCase()] = true;
  if(e.key === " ") shoot();
  if(e.key === "r" || e.key === "R") restartRound();
});
document.addEventListener("keyup", e=>keys[e.key.toLowerCase()] = false);

// ====== ENEMIES ======
function spawnEnemy(){
  enemies.push({
    x: Math.random()*360,
    y: -20,
    w: 30,
    h: 30,
    vy: 1.5
  });
}

// ====== COLLISION ======
function hit(a,b){
  return a.x < b.x+b.w && a.x+a.w > b.x && a.y < b.y+b.h && a.y+a.h > b.y;
}

// ====== GAME LOOP ======
function update(){
  if(gameOver || paused) return;

  // movement
  if(keys["arrowleft"] || keys["a"]) player.x -= player.speed;
  if(keys["arrowright"] || keys["d"]) player.x += player.speed;
  player.x = Math.max(0, Math.min(canvas.width-player.w, player.x));

  // bullets
  bullets.forEach(b=>b.y += b.vy);
  bullets = bullets.filter(b=>b.y > -10);

  // enemies
  enemies.forEach(e=>e.y += e.vy);

  // bullet vs enemy
  bullets.forEach(b=>{
    enemies.forEach(e=>{
      if(hit({x:b.x,y:b.y,w:4,h:8},e)){
        e.dead = true;
        b.dead = true;
        score += 10;
        coins += 10;
      }
    });
  });

  bullets = bullets.filter(b=>!b.dead);
  enemies = enemies.filter(e=>!e.dead);

  // player vs enemy = GAME OVER
  enemies.forEach(e=>{
    if(hit(player,e)){
      gameOver = true;
      save();
    }
  });

  // spawn
  if(Math.random() < 0.03) spawnEnemy();
}

function draw(){
  ctx.clearRect(0,0,canvas.width,canvas.height);

  // player
  ctx.fillStyle="cyan";
  ctx.fillRect(player.x,player.y,player.w,player.h);

  // bullets
  ctx.fillStyle="yellow";
  bullets.forEach(b=>ctx.fillRect(b.x,b.y,4,8));

  // enemies
  ctx.fillStyle="red";
  enemies.forEach(e=>ctx.fillRect(e.x,e.y,e.w,e.h));

  // text
  document.getElementById("info").innerText =
    `Score: ${score} | Coins: ${coins} | Gems: ${gems} | Upgrade: ${upgradeLevel}`;

  if(gameOver){
    ctx.fillStyle="white";
    ctx.font="20px Arial";
    ctx.fillText("GAME OVER – press R",90,300);
  }
}

function loop(){
  update();
  draw();
  requestAnimationFrame(loop);
}

// ====== RESTART ROUND ======
function restartRound(){
  enemies = [];
  bullets = [];
  score = 0;
  gameOver = false;
}

// ====== REBIRTH ======
function rebirth(){
  if(upgradeLevel < 5) return alert("Må ha maks oppgraderinger!");
  if(coins < 1000) return alert("For få coins!");

  coins -= 1000;
  gems += 50;
  upgradeLevel = 0;
  weaponUnlocked = false;
  save();
}

function save(){
  localStorage.setItem("coins",coins);
  localStorage.setItem("gems",gems);
  localStorage.setItem("upgradeLevel",upgradeLevel);
  localStorage.setItem("weaponUnlocked",weaponUnlocked);
}

loop();
</script>
</body>
</html>
