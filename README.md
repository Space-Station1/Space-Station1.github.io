<!DOCTYPE html>
<html lang="no">
<head>
<meta charset="UTF-8">
<title>space-station</title>
<style>
body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; }
canvas { background:#05080f; border:2px solid #4af; }
.ui { position:absolute; top:20px; right:20px; display:flex; flex-direction:column; gap:10px; z-index:2; }
button { padding:10px 15px; font-size:16px; cursor:pointer; }
#loading { position:absolute; inset:0; background:black; display:flex; flex-direction:column; justify-content:center; align-items:center; z-index:5; }
</style>
</head>
<body>

<div id="loading">
<h1>🚀 space-station</h1>
<p>Laster romstasjon…</p>
</div>

<div class="ui">
<button onclick="togglePause()">Pause</button>
<button onclick="restartGame()">Restart</button>
<button id="unlockBtn" onclick="unlockGun()">Unlock Gun</button>
<button onclick="upgradeWeapon()">Upgrade Weapon</button>
<button id="rebirthBtn" onclick="rebirth()" style="display:none;">Rebirth</button>
<button onclick="resetData()">Reset Data</button>
<div id="coins">Coins: 0</div>
<div id="upgrade">Upgrade cost: 0</div>
<div id="gems">Gems: 0</div>
</div>

<canvas id="game" width="500" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");
const GAME_SPEED = 0.9;

let player, enemies, bullets, explosions, stars;
let score, gameOver=false, paused=false;
let keys={};

/* ===== TOUCH: dra til sidene + hold for å skyte ===== */
let touchActive = false;

canvas.addEventListener("touchstart", e => {
  e.preventDefault();
  touchActive = true;
  movePlayerX(e);
  keys[' '] = true;
});

canvas.addEventListener("touchmove", e => {
  e.preventDefault();
  if(touchActive) movePlayerX(e);
});

canvas.addEventListener("touchend", () => {
  touchActive = false;
  keys[' '] = false;
});

function movePlayerX(e){
  const rect = canvas.getBoundingClientRect();
  const touchX = e.touches[0].clientX - rect.left;
  player.x = touchX - player.width / 2;
  if(player.x < 0) player.x = 0;
  if(player.x > canvas.width - player.width)
    player.x = canvas.width - player.width;
}

/* ===== ORIGINAL KODE (URØRT) ===== */

// Lagret data
let coins = Number(localStorage.getItem("coins")) || 100;
let upgradeLevel = Number(localStorage.getItem("upgradeLevel")) || 0;
let highscore = Number(localStorage.getItem("hard_highscore")) || 0;
let gems = Number(localStorage.getItem("gems")) || 0;

let hasGun = false;

// Våpensystem
let bulletSpeed = 8*GAME_SPEED;
let shotsPerGroup = [1,1,2,2,3,3];
let cooldownSettings = [30,9,30,9,30,9];
let shootCooldown = 0;
let groupShooting = false;
let currentShotIndex = 0;

const unlockBtn = document.getElementById("unlockBtn");
const rebirthBtn = document.getElementById("rebirthBtn");

function upgradeCost(){ return 200*upgradeLevel + 100; }
function saveProgress(){
  localStorage.setItem("coins", coins);
  localStorage.setItem("upgradeLevel", upgradeLevel);
  localStorage.setItem("gems", gems);
}

function resetData(){
  localStorage.removeItem("coins");
  localStorage.removeItem("upgradeLevel");
  localStorage.removeItem("hard_highscore");
  localStorage.removeItem("gems");
  coins=100; upgradeLevel=0; highscore=0; gems=0; hasGun=false;
  updateUI(); alert("Data reset!");
}

function init(){
  player={x:180,y:540,width:35,height:35,speed:6*GAME_SPEED};
  enemies=[]; bullets=[]; explosions=[];
  stars = Array.from({length:60},()=>({x:Math.random()*400,y:Math.random()*600,s:1+Math.random()*2}));
  score=0; gameOver=false; paused=false; shootCooldown=0; groupShooting=false; currentShotIndex=0;
  updateUI();
}

function restartGame(){ init(); }
function togglePause(){ if(!gameOver) paused=!paused; }

document.addEventListener("keydown", e=>keys[e.key.toLowerCase()]=true);
document.addEventListener("keyup", e=>keys[e.key.toLowerCase()]=false);

// Fiender
function spawnEnemy(){
  const spawnCount = 3 + Math.floor(Math.random()*3);
  for(let i=0;i<spawnCount;i++){
    enemies.push({x:Math.random()*370,y:-40,w:30,h:30,speedY:2*GAME_SPEED,speedX:0,hp:1,color:'#f44', coins:10});
  }
}

// Skudd
function shoot(){
  bullets.push({x:player.x+player.width/2-3,y:player.y,w:6,h:12,speed:bulletSpeed});
}

function update(){
  if(gameOver||paused) return;

  if((keys['arrowleft']||keys['a']) && player.x>0) player.x-=player.speed;
  if((keys['arrowright']||keys['d']) && player.x<365) player.x+=player.speed;

  if(hasGun && keys[' '] && shootCooldown<=0){
    shoot();
    shootCooldown=15;
  }
  if(shootCooldown>0) shootCooldown--;

  bullets.forEach(b=>b.y-=b.speed);
  bullets = bullets.filter(b=>b.y>-20);

  enemies.forEach(e=>e.y+=e.speedY);

  score+=0.5;
}

function draw(){
  ctx.clearRect(0,0,400,600);
  ctx.fillStyle='#0f0';
  ctx.fillRect(player.x,player.y,player.width,player.height);
  ctx.fillStyle='white';
  bullets.forEach(b=>ctx.fillRect(b.x,b.y,b.w,b.h));
  enemies.forEach(e=>{
    ctx.fillStyle=e.color;
    ctx.fillRect(e.x,e.y,e.w,e.h);
  });
  ctx.fillStyle='white';
  ctx.fillText(`Score: ${Math.floor(score)}`,10,20);
}

init();
setInterval(spawnEnemy,700);
(function loop(){ update(); draw(); requestAnimationFrame(loop); })();
</script>
</body>
</html>
