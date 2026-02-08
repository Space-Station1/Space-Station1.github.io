<html lang="no">
<head>
<meta charset="UTF-8">
<title>space-station</title>
<style>
body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; }
canvas { background:#05080f; border:2px solid #4af; }
.ui { position:absolute; top:40px; right:20px; display:flex; flex-direction:column; gap:10px; z-index:2; }
button { padding:10px 15px; font-size:16px; cursor:pointer; }
#loading { position:absolute; inset:0; background:black; display:flex; flex-direction:column; justify-content:center; align-items:center; z-index:5; }
#shop {
  position: absolute; l
  left: -800px;
  top: 80px;
  width: 220px;
  background: #8b5a2b;
  border: 4px solid #000;
  padding: 10px;
  font-family: monospace;
  color: #000;
}

#shop h2 {
  text-align: center;
  margin: 4px 0 10px;
  border-bottom: 4px solid #000;
}

.section {
  margin-bottom: 12px;
}

.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 6px;
}

.item {
  background: #c68642;
  border: 3px solid #000;
  padding: 4px;
  cursor: pointer;
  text-align: center;
  font-size: 11px;
}

.item:hover {
  background: #d89b5a;
}

.skin {
  width: 32px;
  height: 32px;
  margin: 0 auto 4px;
  border: 2px solid #000;
}

/* Skins */
.green { background: #00ff00; }
.purple { background: #a020f0; }
.gold { background: gold; }

#shop button {
  width: 100%;
  margin-top: 4px;
  font-family: monospace;
  border: 3px solid #000;
  background: #c68642;
  cursor: pointer;
}
</style>
</head>
<body>

<div id="loading">
<h1>🚀Space-Station1</h1>
<p>Sorry we're having some issues, please try again later…</p>
</div>

<div class="ui">
<div id="shop">
  <h2>SHOP</h2>

  <div class="section">
    <h3>Skins</h3>
    <div class="grid">
      <div class="item" onclick="buySkin('green')">
        <div class="skin green"></div>
        <span>Green<br>10💎</span>
      </div>
      <div class="item" onclick="buySkin('purple')">
        <div class="skin purple"></div>
        <span>Purple<br>15💎</span>
      </div>
      <div class="item" onclick="buySkin('gold')">
        <div class="skin gold"></div>
        <span>Gold<br>25💎</span>
      </div>
    </div>
  </div>

  <div class="section">
    <h3>Boosters</h3>
    <button onclick="buyBooster('armor')">Armor (20💎)</button>
    <button onclick="buyBooster('damage')">2x Damage (30💎)</button>
    <button onclick="buyBooster('speed')">0,7x Enemy speed (25💎)</button>
  </div>
</div>
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

<canvas id="game" width="400" height="600"></canvas>

<script>
// BOOSTERS
let armorOn = false;
let doubleDamageOn = false;
let slowEnemiesOn = false;
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");
const GAME_SPEED = 1.0;

  /* ===== TOUCH FOLLOW FINGER ===== */

let isTouching = false;

canvas.addEventListener("touchstart", e => {
  isTouching = true;
  movePlayerToTouch(e);
});
canvas.addEventListener("touchmove", e => {
  if(isTouching) movePlayerToTouch(e);
});

canvas.addEventListener("touchend", () => {
  isTouching = false;
});

function movePlayerToTouch(e){
  e.preventDefault();

  const rect = canvas.getBoundingClientRect();
  const touchX = e.touches[0].clientX - rect.left;

  // Sentrer kuben på fingeren
  player.x = touchX - player.width / 2;

  // Begrens innenfor banen
  if(player.x < 0) player.x = 0;
  if(player.x > canvas.width - player.width)
    player.x = canvas.width - player.width;
}
const ui = document.querySelector(".ui");

const hideBtn = document.createElement("button");
hideBtn.textContent = "Hide UI";
ui.prepend(hideBtn);

let uiHidden = false;

hideBtn.onclick = () => {
  uiHidden = !uiHidden;

  [...ui.children].forEach(el => {
    if (el !== hideBtn) {
      el.style.display = uiHidden ? "none" : "";
    }
  });

  hideBtn.textContent = uiHidden ? "Show UI" : "Hide UI";
};  
/* ===== SHOP DATA ===== */

let ownedSkins = JSON.parse(localStorage.getItem("ownedSkins")) || ["green"];
let activeSkin = localStorage.getItem("activeSkin") || "green";

let boosters = JSON.parse(localStorage.getItem("boosters")) || {
  armor: false,
  damage: false,
  speed: false
};

/* ===== SKINS ===== */

function buySkin(name){
  const prices = { green:10, purple:15, gold:25 };

  if (ownedSkins.includes(name)) {
    activeSkin = name;
    localStorage.setItem("activeSkin", activeSkin);
    return;
  }

  if (gems >= prices[name]) {
    gems -= prices[name];
    ownedSkins.push(name);
    activeSkin = name;

    localStorage.setItem("ownedSkins", JSON.stringify(ownedSkins));
    localStorage.setItem("activeSkin", activeSkin);
    localStorage.setItem("gems", gems);

    updateUI();
  } else {
    alert("Not enough gems!");
  }
}

/* ===== BOOSTERS ===== */

function buyBooster(type){
  const prices = { armor:20, damage:30, speed:25 };

  if (boosters[type]) return;

  if (gems >= prices[type]) {
    gems -= prices[type];
    boosters[type] = true;

    localStorage.setItem("boosters", JSON.stringify(boosters));
    localStorage.setItem("gems", gems);

    applyBoosters();
    updateUI();
  } else {
    alert("Not enough gems!");
  }
}

/* ===== APPLY EFFECTS ===== */

function applyBoosters() {
    if (!player || !enemies) return; // sikkerhet, krasjer ikke hvis init ikke ferdig

    // Armor
    if (boosters.armor) {
        player.armor = true;
        armorOn = true;
    }

    // 2x Damage
    if (boosters.damage) {
        doubleDamageOn = true;
    }

    // Slow enemies
    if (boosters.speed) {
        slowEnemiesOn = true;
        enemies.forEach(enemy => {
            enemy.speedY *= 0.7;
            enemy.speedX *= 0.7;
        });
    }
}
let player, enemies, bullets, explosions, stars;
let score, gameOver=false, paused=false;
let keys={};

// Lagret data
let coins = Number(localStorage.getItem("coins")) || 100;
let upgradeLevel = Number(localStorage.getItem("upgradeLevel")) || 0;
let highscore = Number(localStorage.getItem("hard_highscore")) || 0;
let gems = Number(localStorage.getItem("gems")) || 0;

let hasGun = localStorage.getItem("hasGun") === "true";

// Våpensystem
let bulletSpeed = 8*GAME_SPEED;
let shotsPerGroup = [1,1,2,2,3,3]; // antall skudd i gruppen per oppgr
let cooldownSettings = [30,9,30,9,30,9]; // frames mellom grupper per oppgr
let shootCooldown = 0; // vent før neste gruppe
let groupShooting = false;
let currentShotIndex = 0;

// UI
const unlockBtn = document.getElementById("unlockBtn");
const rebirthBtn = document.getElementById("rebirthBtn");

/* ===== FIRE ON / OFF BUTTON ===== */

// Lag knapp i UI (samme stil som andre)
const fireBtn = document.createElement("button");
fireBtn.innerText = "Fire: OFF";
document.querySelector(".ui").appendChild(fireBtn);

// Fire-status
let fireOn = false;

// Toggle
fireBtn.onclick = () => {
  fireOn = !fireOn;
  fireBtn.innerText = fireOn ? "Fire: ON" : "Fire: OFF";
};
  
// Oppgraderingskostnad
function upgradeCost(){ return 200*upgradeLevel + 100; }
function saveProgress(){
  localStorage.setItem("coins", coins);
  localStorage.setItem("upgradeLevel", upgradeLevel);
  localStorage.setItem("gems", gems);
}

// Reset alt data
function resetData(){
  localStorage.removeItem("coins");
  localStorage.removeItem("upgradeLevel");
  localStorage.removeItem("hard_highscore");
  localStorage.removeItem("gems");
  coins=100; upgradeLevel=0; highscore=0; gems=0; hasGun=false; bulletsPerGroup=1; bulletSpeed=8*GAME_SPEED;
  updateUI(); alert("Data reset!");
}

// Init spill
function init(){
  player = {
    x: 180,
    y: 540,
    width: 35,
    height: 35,
    speed: 6*GAME_SPEED,
    hp: armorOn ? 3 : 1
  };

  enemies = [];
  bullets = [];
  explosions = [];
  stars = Array.from({length:60},()=>({x:Math.random()*400,y:Math.random()*600,s:1+Math.random()*2}));
  score = 0;
  gameOver = false;
  paused = false;
  shootCooldown = 0;
  groupShooting = false;
  currentShotIndex = 0;

  updateUI();
}

// Restart runden
function restartGame(){ init(); }

// Pause
function togglePause(){ if(!gameOver) paused=!paused; }

// Unlock pistol
function unlockGun(){
  if(score>=1000 && coins>=100 && !hasGun){
    coins-=100; hasGun=true; function saveProgress(){
  localStorage.setItem("coins", coins);
  localStorage.setItem("upgradeLevel", upgradeLevel);
  localStorage.setItem("gems", gems);
  localStorage.setItem("hasGun", hasGun);
    }; updateUI();
    alert("Pistol unlocked!");
  } else if(score<1000) alert("Score 1000 needed!");
  else if(coins<100) alert("Not enough coins!");
}

// Oppgrader pistol
function upgradeWeapon(){
  if(!hasGun) { alert("You need to unlock the gun first!"); return; }
  if(upgradeLevel>=5){ alert("Max upgrade reached!"); return; }
  const cost = upgradeCost();
  if(coins>=cost){
    coins-=cost; upgradeLevel++;
    bulletSpeed = 8 + upgradeLevel*2;
    saveProgress(); updateUI();
    if(upgradeLevel==5) rebirthBtn.style.display="block";
  }
}

// Rebirth
function rebirth(){
  if(upgradeLevel<5){ alert("Need max upgrade to rebirth!"); return; }
  if(coins<1000){ alert("Need 1000 coins to rebirth!"); return; }
  coins-=1000; hasGun=false; upgradeLevel=0; bulletSpeed=8*GAME_SPEED; gems+=50;
  saveProgress(); updateUI();
  alert("Rebirth complete! +50 gems!");
  rebirthBtn.style.display="none";
}
function buyArmor(){
  if(gems < 20) return alert("Not enough gems!");
  if(armorOn) return alert("Armor already active!");
  gems -= 20;
  armorOn = true;
  saveProgress();
  updateUI();
}

function buyDoubleDamage(){
  if(gems < 30) return alert("Not enough gems!");
  if(doubleDamageOn) return alert("Already active!");
  gems -= 30;
  doubleDamageOn = true;
  saveProgress();
  updateUI();
}

function buySlowEnemies(){
  if(gems < 25) return alert("Not enough gems!");
  if(slowEnemiesOn) return alert("Already active!");
  gems -= 25;
  slowEnemiesOn = true;
  saveProgress();
  updateUI();
}

// Oppdater UI
function updateUI(){
  document.getElementById("coins").innerText=`Coins: ${coins}`;
  document.getElementById("upgrade").innerText = hasGun ? `Upgrade cost: ${upgradeCost()}` : "Unlock gun first";
  document.getElementById("gems").innerText=`Gems: ${gems}`;
  unlockBtn.style.display = (score>=1000 && !hasGun) ? "block" : "none";
}
document.getElementById("boosters").innerText =
  `Armor: ${armorOn ? "ON" : "OFF"} | 2x DMG: ${doubleDamageOn ? "ON" : "OFF"} | Slow: ${slowEnemiesOn ? "ON" : "OFF"}`;

// Input
document.addEventListener("keydown", e=>{
  keys[e.key.toLowerCase()]=true;
  if(e.key==='p'||e.key==='P') togglePause();
  if(e.key==='r'||e.key==='R') restartGame();
});
document.addEventListener("keyup", e=>keys[e.key.toLowerCase()]=false);

// Fiender
function spawnEnemy(){
  const spawnCount = 3 + Math.floor(Math.random()*3);
  for(let i=0;i<spawnCount;i++){
    const r=Math.random();
    if(r<0.75){
      enemies.push({x:Math.random()*370,y:-40,w:30,h:30,speedY: (1.8 + score/2000) * GAME_SPEED * (slowEnemiesOn ? 0.7 : 1),speedX:0,hp:1,color:'#f44', coins:10});
    } else if(r<0.95){
      const left=Math.random()<0.5;
      enemies.push({x:left?-40:440,y:Math.random()*250,w:35,h:35,speedY:1.5*GAME_SPEED,speedX:left?2.5*GAME_SPEED:-2.5*GAME_SPEED,hp:1,color:'#fa0', coins:10});
    } else {
      enemies.push({x:150,y:-80,w:100,h:80,speedY:1*GAME_SPEED,speedX:1*GAME_SPEED,hp:20,isBoss:true,color:'#a4f', coins:50});
    }
  }
}

// Skudd
function shoot(){
  let off=0;
  const groupShots = shotsPerGroup[upgradeLevel];
  if(groupShots>1){
    const spread=10;
    if(groupShots==2) off=[-spread/2,spread/2][currentShotIndex%2];
    if(groupShots==3) off=[-spread,0,spread][currentShotIndex%3];
  }
  bullets.push({x:player.x+player.width/2-3+off,y:player.y,w:6,h:12,speed:bulletSpeed});
}

// Eksplosjon
function explode(x,y){ for(let i=0;i<8;i++) explosions.push({x,y,dx:(Math.random()-0.5)*4,dy:(Math.random()-0.5)*4,life:15}); }

function update(){
  if(gameOver||paused) return;

  stars.forEach(s=>{ s.y += s.s*GAME_SPEED; if(s.y>600) s.y=0; });

  if((keys['arrowleft']||keys['a']) && player.x>0) player.x-=player.speed;
  if((keys['arrowright']||keys['d']) && player.x<365) player.x+=player.speed;

  // Skudd-grupper
  if(hasGun){
    if(!groupShooting && (keys[' '] || fireOn) && shootCooldown<=0){
      groupShooting=true;
      groupCooldown=cooldownSettings[upgradeLevel];
      currentShotIndex=0;
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
  }

  bullets.forEach(b=>b.y-=b.speed);
  bullets = bullets.filter(b=>b.y>-20);

  enemies.forEach(e=>{ e.y+=e.speedY; e.x+=e.speedX; });

  bullets.forEach((b,bi)=>{
    enemies.forEach((e,ei)=>{
      if(b.x<e.x+e.w && b.x+b.w>e.x && b.y<e.y+e.h && b.y+b.h>e.y){
        explode(e.x+e.w/2,e.y+e.h/2);
        e.hp -= doubleDamageOn ? 2 : 1; bullets.splice(bi,1);
        if(e.hp<=0){
          enemies.splice(ei,1);
          coins += e.coins;
          score += e.isBoss?1000:200;
          saveProgress(); updateUI();
        }
      }
    });
  });

  enemies.forEach(e=>{
    if(player.x<e.x+e.w && player.x+player.width>e.x && player.y<e.y+e.h && player.y+player.height>e.y){
      player.hp--;

if(player.hp <= 0){
  gameOver = true;
}
      if(score>highscore){ highscore=score; localStorage.setItem('hard_highscore',highscore); }
    }
  });

  explosions.forEach(p=>{p.x+=p.dx;p.y+=p.dy;p.life--;});
  explosions = explosions.filter(p=>p.life>0);

  score+=0.5*GAME_SPEED;
  updateUI();
}

function draw(){
  ctx.fillRect(player.x,player.y,player.width,player.height);
  ctx.clearRect(0,0,400,600);
  stars.forEach(s=>{ ctx.fillStyle='white'; ctx.fillRect(s.x,s.y,2,2); });
  const skinColors = {
  green: "#00ff00",
  purple: "#a020f0",
  gold: "gold"
};
ctx.fillStyle = skinColors[activeSkin] || "#0f0"; ctx.fillRect(player.x,player.y,player.width,player.height);
  ctx.fillStyle='white'; bullets.forEach(b=>ctx.fillRect(b.x,b.y,b.w,b.h));
  enemies.forEach(e=>{
    ctx.fillStyle=e.color; ctx.fillRect(e.x,e.y,e.w,e.h);
    if(e.isBoss){ ctx.fillStyle='black'; ctx.fillRect(e.x,e.y-10,e.w,5); ctx.fillStyle='#0f0'; ctx.fillRect(e.x,e.y-10,e.w*(e.hp/20),5); }
  });
  explosions.forEach(p=>{ ctx.fillStyle='yellow'; ctx.fillRect(p.x,p.y,3,3); });
  ctx.fillStyle='white'; ctx.font='16px Arial';
  ctx.fillText(`Score: ${Math.floor(score)}`,10,20);
  ctx.fillText(`Highscore: ${highscore}`,10,40);
  if(paused){ ctx.font='30px Arial'; ctx.fillText('PAUSE',140,300); }
  if(gameOver){ ctx.font='30px Arial'; ctx.fillText('GAME OVER',100,300); }
}

// Loading screen
if(!localStorage.getItem('visited')){
  setTimeout(()=>{ document.getElementById('loading').style.display='none'; localStorage.setItem('visited','yes'); },2000);
}else{ document.getElementById('loading').style.display='none'; }

init();
setInterval(spawnEnemy,700);
(function loop(){ update(); draw(); requestAnimationFrame(loop); })();
</script>
</body>
</html>
