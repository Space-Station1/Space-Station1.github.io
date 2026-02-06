<!DOCTYPE html>
<html lang="no">
<head>
<meta charset="UTF-8">
<title>space-station</title>
<style>
body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; }
canvas { background:#05080f; border:2px solid #4af; touch-action:none; }
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
<button onclick="restartGame()">Restart (R)</button>
<button onclick="toggleTrigger()">Trigger: <span id="triggerState">OFF</span></button>
<button id="unlockBtn" onclick="unlockGun()">Unlock Gun</button>
<button onclick="upgradeWeapon()">Upgrade Weapon</button>
<button id="rebirthBtn" onclick="rebirth()" style="display:none;">Rebirth</button>
<button onclick="resetData()">Reset Data</button>
<div id="coins">Coins: 0</div>
<div id="upgrade">Upgrade cost: 0</div>
<div id="gems">Gems: 0</div>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");
const GAME_SPEED = 1;

let player, enemies, bullets, explosions, stars;
let score, gameOver=false, paused=false;
let keys={};

// Touch
let touchX = null;
let touchShooting = false;

// Lagret data
let coins = Number(localStorage.getItem("coins")) || 100;
let upgradeLevel = Number(localStorage.getItem("upgradeLevel")) || 0;
let highscore = Number(localStorage.getItem("hard_highscore")) || 0;
let gems = Number(localStorage.getItem("gems")) || 0;
let triggerOn = localStorage.getItem("triggerOn")==="true";

let hasGun = false;

// Våpen
let bulletSpeed = 8*GAME_SPEED;
let shotsPerGroup = [1,1,2,2,3,3];
let cooldownSettings = [30,9,30,9,30,9];
let shootCooldown = 0;
let groupShooting = false;
let currentShotIndex = 0;
let groupCooldown = 0;

// UI
const unlockBtn = document.getElementById("unlockBtn");
const rebirthBtn = document.getElementById("rebirthBtn");
const triggerState = document.getElementById("triggerState");

// Trigger
function toggleTrigger(){
  triggerOn = !triggerOn;
  localStorage.setItem("triggerOn", triggerOn);
  updateUI();
}

// Lagre
function saveProgress(){
  localStorage.setItem("coins", coins);
  localStorage.setItem("upgradeLevel", upgradeLevel);
  localStorage.setItem("gems", gems);
}

// Reset
function resetData(){
  localStorage.clear();
  location.reload();
}

// Init
function init(){
  player={x:180,y:540,width:35,height:35,speed:6*GAME_SPEED};
  enemies=[]; bullets=[]; explosions=[];
  stars = Array.from({length:60},()=>({x:Math.random()*400,y:Math.random()*600,s:1+Math.random()*2}));
  score=0; gameOver=false; paused=false;
  shootCooldown=0; groupShooting=false; currentShotIndex=0;
  updateUI();
}

function restartGame(){ init(); }
function togglePause(){ if(!gameOver) paused=!paused; }

function unlockGun(){
  if(score>=1000 && coins>=100 && !hasGun){
    coins-=100; hasGun=true; saveProgress(); updateUI();
  }
}

function upgradeCost(){ return 200*upgradeLevel + 100; }

function upgradeWeapon(){
  if(!hasGun || upgradeLevel>=5) return;
  const cost = upgradeCost();
  if(coins>=cost){
    coins-=cost; upgradeLevel++;
    bulletSpeed = 8 + upgradeLevel*2;
    if(upgradeLevel===5) rebirthBtn.style.display="block";
    saveProgress(); updateUI();
  }
}

function rebirth(){
  if(upgradeLevel<5 || coins<1000) return;
  coins-=1000;
  upgradeLevel=0;
  hasGun=false;
  gems+=50;
  rebirthBtn.style.display="none";
  saveProgress(); updateUI();
}

// UI
function updateUI(){
  document.getElementById("coins").innerText=`Coins: ${coins}`;
  document.getElementById("upgrade").innerText=hasGun?`Upgrade cost: ${upgradeCost()}`:"Unlock gun first";
  document.getElementById("gems").innerText=`Gems: ${gems}`;
  unlockBtn.style.display=(score>=1000 && !hasGun)?"block":"none";
  triggerState.innerText=triggerOn?"ON":"OFF";
}

// Input
document.addEventListener("keydown", e=>{
  keys[e.key.toLowerCase()]=true;
  if(e.key==='r') restartGame();
});
document.addEventListener("keyup", e=>keys[e.key.toLowerCase()]=false);

// Touch
canvas.addEventListener("touchstart", e=>{
  const t=e.touches[0];
  touchX=t.clientX;
  if(t.clientX>canvas.getBoundingClientRect().left+canvas.width/2) touchShooting=true;
});
canvas.addEventListener("touchmove", e=>{
  touchX=e.touches[0].clientX;
});
canvas.addEventListener("touchend", ()=>{
  touchX=null;
  touchShooting=false;
});

// Enemy spawn
function spawnEnemy(){
  for(let i=0;i<4;i++){
    enemies.push({x:Math.random()*370,y:-40,w:30,h:30,speedY:2,hp:1,color:'#f44',coins:10});
  }
}

// Shoot
function shoot(){
  let spread=[0];
  if(shotsPerGroup[upgradeLevel]===2) spread=[-5,5];
  if(shotsPerGroup[upgradeLevel]===3) spread=[-10,0,10];
  bullets.push({
    x:player.x+player.width/2-3+spread[currentShotIndex],
    y:player.y,w:6,h:12,speed:bulletSpeed
  });
}

// Update
function update(){
  if(gameOver||paused) return;

  if((keys['a']||keys['arrowleft'])&&player.x>0) player.x-=player.speed;
  if((keys['d']||keys['arrowright'])&&player.x<365) player.x+=player.speed;

  if(touchX!==null){
    const rect=canvas.getBoundingClientRect();
    player.x=(touchX-rect.left)-player.width/2;
  }

  const firing = triggerOn || keys[' '] || touchShooting;

  if(hasGun && firing && shootCooldown<=0 && !groupShooting){
    groupShooting=true;
    currentShotIndex=0;
    groupCooldown=cooldownSettings[upgradeLevel];
  }

  if(groupShooting){
    if(currentShotIndex<shotsPerGroup[upgradeLevel]){
      shoot();
      currentShotIndex++;
    } else {
      groupShooting=false;
      shootCooldown=groupCooldown;
    }
  }

  if(shootCooldown>0) shootCooldown--;

  bullets.forEach(b=>b.y-=b.speed);
  bullets=bullets.filter(b=>b.y>-20);

  enemies.forEach(e=>e.y+=e.speedY);

  bullets.forEach((b,bi)=>{
    enemies.forEach((e,ei)=>{
      if(b.x<e.x+e.w&&b.x+b.w>e.x&&b.y<e.y+e.h&&b.y+b.h>e.y){
        bullets.splice(bi,1);
        enemies.splice(ei,1);
        coins+=e.coins;
        score+=200;
      }
    });
  });

  score+=0.5;
  updateUI();
}

function draw(){
  ctx.clearRect(0,0,400,600);
  ctx.fillStyle="#0f0";
  ctx.fillRect(player.x,player.y,player.width,player.height);
  ctx.fillStyle="white";
  bullets.forEach(b=>ctx.fillRect(b.x,b.y,b.w,b.h));
  enemies.forEach(e=>{
    ctx.fillStyle=e.color;
    ctx.fillRect(e.x,e.y,e.w,e.h);
  });
  ctx.fillText(`Score: ${Math.floor(score)}`,10,20);
}

init();
setInterval(spawnEnemy,800);
(function loop(){ update(); draw(); requestAnimationFrame(loop); })();
</script>
</body>
</html>
