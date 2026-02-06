<!DOCTYPE html>
<html lang="no">
<head>
<meta charset="UTF-8">
<title>space-station</title>
<style>
body{
  margin:0; background:black; color:white;
  display:flex; justify-content:center; align-items:center;
  height:100vh; font-family:Arial; overflow:hidden;
}
canvas{ background:#05080f; border:2px solid #4af; }
.ui{
  position:absolute; top:20px; right:20px;
  display:flex; flex-direction:column; gap:8px; z-index:2;
}
button{ padding:8px 12px; font-size:14px; cursor:pointer; }
</style>
</head>
<body>

<div class="ui">
  <button onclick="togglePause()">Pause</button>
  <button onclick="restartGame()">Restart (R)</button>
  <button onclick="toggleTrigger()">Trigger: <span id="triggerTxt">OFF</span></button>
  <button id="unlockBtn" onclick="unlockGun()">Unlock Gun</button>
  <button onclick="upgradeWeapon()">Upgrade Weapon</button>
  <button id="rebirthBtn" onclick="rebirth()" style="display:none;">Rebirth</button>
  <button onclick="resetData()">Reset Data</button>
  <div id="coins">Coins: 0</div>
  <div id="gems">Gems: 0</div>
  <div id="upgrade">Upgrade: 0</div>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
/* =======================
   CANVAS & STATE
======================= */
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

let player, enemies, bullets, explosions, stars;
let score=0, gameOver=false, paused=false;
let keys={};

/* =======================
   SAVED DATA
======================= */
let coins = +localStorage.getItem("coins") || 100;
let gems = +localStorage.getItem("gems") || 0;
let upgradeLevel = +localStorage.getItem("upgradeLevel") || 0;
let highscore = +localStorage.getItem("highscore") || 0;
let hasGun = localStorage.getItem("hasGun")==="yes";

/* =======================
   WEAPON SYSTEM
======================= */
const shotsPerGroup = [1,1,2,2,3,3];
const cooldowns = [30,9,30,9,30,9]; // frames
let shootCooldown = 0;
let groupShooting = false;
let currentShot = 0;
let triggerOn = false;

/* =======================
   TOUCH
======================= */
let touchX=null;
let touchFire=false;

canvas.addEventListener("touchstart",e=>{
  e.preventDefault();
  touchFire=true;
  touchX=e.touches[0].clientX;
},{passive:false});

canvas.addEventListener("touchmove",e=>{
  e.preventDefault();
  touchX=e.touches[0].clientX;
},{passive:false});

canvas.addEventListener("touchend",()=>{
  touchFire=false;
  touchX=null;
});

/* =======================
   INIT
======================= */
function init(){
  player={x:180,y:540,w:35,h:35,speed:6};
  enemies=[]; bullets=[]; explosions=[];
  stars=Array.from({length:60},()=>({x:Math.random()*400,y:Math.random()*600,s:1+Math.random()*2}));
  score=0; gameOver=false; paused=false;
}
init();

/* =======================
   INPUT
======================= */
document.addEventListener("keydown",e=>{
  keys[e.key.toLowerCase()]=true;
  if(e.key==="r"||e.key==="R") restartGame();
});
document.addEventListener("keyup",e=>keys[e.key.toLowerCase()]=false);

/* =======================
   UI ACTIONS
======================= */
function togglePause(){ paused=!paused; }
function restartGame(){ init(); }
function toggleTrigger(){
  triggerOn=!triggerOn;
  document.getElementById("triggerTxt").innerText=triggerOn?"ON":"OFF";
}

function unlockGun(){
  if(score>=1000 && coins>=100 && !hasGun){
    coins-=100; hasGun=true;
    localStorage.setItem("hasGun","yes");
    save();
  }
}

function upgradeWeapon(){
  if(!hasGun || upgradeLevel>=5) return;
  let cost=200*(upgradeLevel+1);
  if(coins>=cost){
    coins-=cost;
    upgradeLevel++;
    if(upgradeLevel===5)
      document.getElementById("rebirthBtn").style.display="block";
    save();
  }
}

function rebirth(){
  if(upgradeLevel<5 || coins<1000) return;
  coins-=1000;
  upgradeLevel=0;
  hasGun=false;
  gems+=50;
  localStorage.setItem("hasGun","no");
  document.getElementById("rebirthBtn").style.display="none";
  save();
}

function resetData(){
  localStorage.clear();
  location.reload();
}

function save(){
  localStorage.setItem("coins",coins);
  localStorage.setItem("gems",gems);
  localStorage.setItem("upgradeLevel",upgradeLevel);
}

/* =======================
   GAME LOGIC
======================= */
function spawnEnemy(){
  for(let i=0;i<3;i++){
    let r=Math.random();
    if(r<0.8){
      enemies.push({x:Math.random()*370,y:-40,w:30,h:30,sy:1.6,hp:1,coins:10});
    }else{
      enemies.push({x:Math.random()<0.5?-40:440,y:Math.random()*250,w:35,h:35,sx:3,hp:1,coins:10});
    }
  }
}

function shoot(){
  let spread=[0];
  if(shotsPerGroup[upgradeLevel]==2) spread=[-6,6];
  if(shotsPerGroup[upgradeLevel]==3) spread=[-10,0,10];
  let off=spread[currentShot%spread.length];
  bullets.push({x:player.x+player.w/2+off,y:player.y,s:8});
}

function explode(x,y){
  for(let i=0;i<6;i++)
    explosions.push({x,y,dx:(Math.random()-.5)*3,dy:(Math.random()-.5)*3,l:15});
}

/* =======================
   UPDATE
======================= */
function update(){
  if(paused||gameOver) return;

  stars.forEach(s=>{s.y+=s.s; if(s.y>600)s.y=0;});

  if(keys["arrowleft"]||keys["a"]) player.x-=player.speed;
  if(keys["arrowright"]||keys["d"]) player.x+=player.speed;

  if(touchX!==null){
    const r=canvas.getBoundingClientRect();
    player.x=touchX-r.left-player.w/2;
  }
  player.x=Math.max(0,Math.min(365,player.x));

  let firing = (keys[" "]||triggerOn||touchFire);

  if(hasGun){
    if(!groupShooting && firing && shootCooldown<=0){
      groupShooting=true;
      currentShot=0;
    }
    if(groupShooting){
      if(currentShot<shotsPerGroup[upgradeLevel]){
        shoot(); currentShot++;
      }else{
        groupShooting=false;
        shootCooldown=cooldowns[upgradeLevel];
      }
    }
    if(shootCooldown>0) shootCooldown--;
  }

  bullets.forEach(b=>b.y-=b.s);
  bullets=bullets.filter(b=>b.y>-20);

  enemies.forEach(e=>{
    e.y+=e.sy||0; e.x+=e.sx||0;
  });

  bullets.forEach((b,bi)=>{
    enemies.forEach((e,ei)=>{
      if(b.x>e.x&&b.x<e.x+e.w&&b.y>e.y&&b.y<e.y+e.h){
        bullets.splice(bi,1); e.hp--; explode(e.x,e.y);
        if(e.hp<=0){
          coins+=e.coins; enemies.splice(ei,1);
        }
      }
    });
  });

  explosions.forEach(p=>{p.x+=p.dx;p.y+=p.dy;p.l--;});
  explosions=explosions.filter(p=>p.l>0);

  score+=0.5;
  if(score>highscore){ highscore=score; localStorage.setItem("highscore",highscore); }

  document.getElementById("coins").innerText="Coins: "+coins;
  document.getElementById("gems").innerText="Gems: "+gems;
  document.getElementById("upgrade").innerText="Upgrade: "+upgradeLevel;
  document.getElementById("unlockBtn").style.display=
    (!hasGun && score>=1000)?"block":"none";
}

/* =======================
   DRAW
======================= */
function draw(){
  ctx.clearRect(0,0,400,600);
  stars.forEach(s=>ctx.fillRect(s.x,s.y,2,2));
  ctx.fillStyle="#0f0";
  ctx.fillRect(player.x,player.y,player.w,player.h);
  ctx.fillStyle="#fff";
  bullets.forEach(b=>ctx.fillRect(b.x,b.y,4,8));
  enemies.forEach(e=>ctx.fillRect(e.x,e.y,e.w,e.h));
  explosions.forEach(p=>ctx.fillRect(p.x,p.y,3,3));
}

/* =======================
   LOOP
======================= */
setInterval(spawnEnemy,800);
(function loop(){
  update(); draw();
  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
