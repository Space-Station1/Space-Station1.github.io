<!DOCTYPE html>
<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space Station - Fixed</title>
<style>
    body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; }
    canvas { background:#05080f; border:2px solid #4af; }
    .ui { position:absolute; top:20px; right:20px; display:flex; flex-direction:column; gap:5px; z-index:2; max-width: 200px; }
    button { padding:8px; font-size:14px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; }
    button:hover { background: #4af; }
    #loading { position:absolute; inset:0; background:black; display:flex; flex-direction:column; justify-content:center; align-items:center; z-index:5; }
    #shop { margin-top: 10px; border-top: 1px solid #4af; padding-top: 10px; }
</style>
</head>
<body>

<div id="loading">
    <h1>🚀 space-station</h1>
    <p>Laster inn systemer...</p>
</div>

<div class="ui" id="mainUI">
    <button onclick="togglePause()">Pause</button>
    <button onclick="restartGame()">Restart</button>
    <button id="unlockBtn" onclick="unlockGun()">Unlock Gun</button>
    <button onclick="upgradeWeapon()">Upgrade Weapon</button>
    <button id="rebirthBtn" onclick="rebirth()" style="display:none;">Rebirth</button>
    <button onclick="resetData()">Reset Data</button>
    <div id="coins">Coins: 0</div>
    <div id="upgrade">Upgrade cost: 0</div>
    <div id="gems">Gems: 0</div>
    
    <div id="shop">
        <strong>Shop</strong>
        <button onclick="buyBooster('armor')">Armor (50c)</button>
        <button onclick="buyBooster('doubleDamage')">2x Dmg (75c)</button>
        <button onclick="buyBooster('slowEnemies')">Slow (100c)</button>
    </div>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");
const GAME_SPEED = 1.0;

// Spill-variabler
let player, enemies, bullets, explosions, stars;
let score, gameOver=false, paused=false;
let keys={};
let isTouching = false;
let fireOn = false;

// Lagret data
let coins = Number(localStorage.getItem("coins")) || 100;
let upgradeLevel = Number(localStorage.getItem("upgradeLevel")) || 0;
let highscore = Number(localStorage.getItem("hard_highscore")) || 0;
let gems = Number(localStorage.getItem("gems")) || 0;
let hasGun = localStorage.getItem("hasGun") === "true";

// Boosters
let boosters = { armor: false, doubleDamage: false, slowEnemies: false };

// Våpensystem
let bulletSpeed = 8 * GAME_SPEED;
let shotsPerGroup = [1, 1, 2, 2, 3, 3];
let cooldownSettings = [30, 20, 15, 10, 8, 5];
let shootCooldown = 0;
let groupShooting = false;
let currentShotIndex = 0;

// UI elementer
const ui = document.getElementById("mainUI");
const unlockBtn = document.getElementById("unlockBtn");
const rebirthBtn = document.getElementById("rebirthBtn");

// Touch kontroll
canvas.addEventListener("touchstart", e => { isTouching = true; movePlayerToTouch(e); });
canvas.addEventListener("touchmove", e => { if(isTouching) movePlayerToTouch(e); });
canvas.addEventListener("touchend", () => { isTouching = false; });

function movePlayerToTouch(e){
    e.preventDefault();
    const rect = canvas.getBoundingClientRect();
    const touchX = e.touches[0].clientX - rect.left;
    player.x = touchX - player.width / 2;
    if(player.x < 0) player.x = 0;
    if(player.x > canvas.width - player.width) player.x = canvas.width - player.width;
}

// UI Funksjoner
function updateUI(){
    document.getElementById("coins").innerText = `Coins: ${coins}`;
    document.getElementById("upgrade").innerText = hasGun ? `Upgrade: ${200 * upgradeLevel + 100}` : "Unlock gun first";
    document.getElementById("gems").innerText = `Gems: ${gems}`;
    unlockBtn.style.display = hasGun ? "none" : "block";
    if(upgradeLevel >= 5) rebirthBtn.style.display = "block";
}

function saveProgress(){
    localStorage.setItem("coins", coins);
    localStorage.setItem("upgradeLevel", upgradeLevel);
    localStorage.setItem("gems", gems);
    localStorage.setItem("hasGun", hasGun);
}

function buyBooster(type) {
    let costs = { armor: 50, doubleDamage: 75, slowEnemies: 100 };
    if(boosters[type]) return alert("Allerede kjøpt!");
    if(coins < costs[type]) return alert("For lite penger!");
    
    coins -= costs[type];
    boosters[type] = true;
    updateUI();
    saveProgress();
    alert(type + " aktivert!");
}

function resetData(){
    localStorage.clear();
    location.reload();
}

function init(){
    player={x:180,y:540,width:35,height:35,speed:6*GAME_SPEED, armorUsed: false};
    enemies=[]; bullets=[]; explosions=[];
    stars = Array.from({length:60},()=>({x:Math.random()*400,y:Math.random()*600,s:1+Math.random()*2}));
    score=0; gameOver=false; paused=false; 
    updateUI();
}

function togglePause(){ if(!gameOver) paused=!paused; }
function restartGame(){ init(); }

function unlockGun(){
    if(coins >= 100){
        coins -= 100; hasGun = true; saveProgress(); updateUI();
    } else { alert("Trenger 100 coins!"); }
}

function upgradeWeapon(){
    let cost = 200 * upgradeLevel + 100;
    if(hasGun && upgradeLevel < 5 && coins >= cost){
        coins -= cost; upgradeLevel++; saveProgress(); updateUI();
    }
}

function rebirth(){
    if(upgradeLevel >= 5 && coins >= 1000){
        coins -= 1000; upgradeLevel = 0; gems += 50; hasGun = false;
        saveProgress(); updateUI(); rebirthBtn.style.display="none";
    }
}

function spawnEnemy(){
    if(paused || gameOver) return;
    const spawnCount = 1 + Math.floor(score/5000); 
    for(let i=0; i<spawnCount; i++){
        let r = Math.random();
        if(r < 0.8){
            enemies.push({x:Math.random()*370, y:-40, w:30, h:30, speedY:(1.8 + score/5000), speedX:0, hp:1, color:'#f44', coins:10});
        } else {
            enemies.push({x:150, y:-80, w:100, h:80, speedY:0.5, speedX:0.5, hp:10, isBoss:true, color:'#a4f', coins:50});
        }
    }
}

function update(){
    if(gameOver || paused) return;

    // Bakgrunn
    stars.forEach(s=>{ s.y += s.s; if(s.y>600) s.y=0; });

    // Bevegelse
    if((keys['arrowleft']||keys['a']) && player.x>0) player.x-=player.speed;
    if((keys['arrowright']||keys['d']) && player.x<365) player.x+=player.speed;

    // Skyting
    if(hasGun){
        if(shootCooldown <= 0 && (keys[' '] || fireOn || isTouching)){
            bullets.push({x:player.x+player.width/2-3, y:player.y, w:6, h:12, speed:bulletSpeed});
            shootCooldown = cooldownSettings[upgradeLevel];
        }
        if(shootCooldown > 0) shootCooldown--;
    }

    // Oppdater kuler
    bullets.forEach(b => b.y -= b.speed);
    bullets = bullets.filter(b => b.y > -20);

    // Oppdater fiender (inkludert Slow-booster)
    let speedMult = boosters.slowEnemies ? 0.6 : 1;
    enemies.forEach(e => {
        e.y += e.speedY * speedMult;
        if(e.isBoss) {
            e.x += e.speedX;
            if(e.x < 0 || e.x > 300) e.speedX *= -1;
        }
    });

    // Kollisjon Kule -> Fiende
    bullets.forEach((b, bi) => {
        enemies.forEach((e, ei) => {
            if(b.x < e.x + e.w && b.x + b.w > e.x && b.y < e.y + e.h && b.y + b.h > e.y){
                for(let i=0; i<5; i++) explosions.push({x:b.x, y:b.y, dx:(Math.random()-0.5)*4, dy:(Math.random()-0.5)*4, life:10});
                bullets.splice(bi, 1);
                e.hp -= boosters.doubleDamage ? 2 : 1;
                if(e.hp <= 0){
                    coins += e.coins;
                    score += e.isBoss ? 1000 : 200;
                    enemies.splice(ei, 1);
                    updateUI();
                }
            }
        });
    });

    // Kollisjon Fiende -> Spiller (inkludert Armor-booster)
    enemies.forEach((e, ei) => {
        if(player.x < e.x + e.w && player.x + player.width > e.x && player.y < e.y + e.h && player.y + player.height > e.y){
            if(boosters.armor && !player.armorUsed){
                player.armorUsed = true;
                enemies.splice(ei, 1); // Fjern fienden som traff
                alert("Armor absorbert!");
            } else {
                gameOver = true;
                if(score > highscore) localStorage.setItem('hard_highscore', Math.floor(score));
            }
        }
    });

    explosions.forEach(p=>{p.x+=p.dx; p.y+=p.dy; p.life--;});
    explosions = explosions.filter(p=>p.life>0);
    score += 0.2;
}

function draw(){
    ctx.clearRect(0,0,400,600);
    stars.forEach(s=>{ ctx.fillStyle='white'; ctx.fillRect(s.x,s.y,2,2); });
    
    // Tegn spiller (blå hvis armor er aktiv)
    ctx.fillStyle = (boosters.armor && !player.armorUsed) ? '#4af' : '#0f0';
    ctx.fillRect(player.x,player.y,player.width,player.height);
    
    ctx.fillStyle='yellow'; bullets.forEach(b=>ctx.fillRect(b.x,b.y,b.w,b.h));
    enemies.forEach(e=>{
        ctx.fillStyle=e.color; ctx.fillRect(e.x,e.y,e.w,e.h);
        if(e.isBoss){ 
            ctx.fillStyle='red'; ctx.fillRect(e.x, e.y-10, e.w, 5);
            ctx.fillStyle='#0f0'; ctx.fillRect(e.x, e.y-10, e.w*(e.hp/10), 5);
        }
    });
    explosions.forEach(p=>{ ctx.fillStyle='orange'; ctx.fillRect(p.x,p.y,3,3); });
    
    ctx.fillStyle='white'; ctx.font='16px Arial';
    ctx.fillText(`Score: ${Math.floor(score)}`, 10, 25);
    if(paused) ctx.fillText('PAUSE', 170, 300);
    if(gameOver) ctx.fillText('GAME OVER - Trykk Restart', 100, 300);
}

document.addEventListener("keydown", e => {
    keys[e.key.toLowerCase()] = true;
    if(e.key === 'p') togglePause();
});
document.addEventListener("keyup", e => keys[e.key.toLowerCase()] = false);

// Start spill
init();
setInterval(spawnEnemy, 1000);
document.getElementById('loading').style.display='none';

(function loop(){
    update();
    draw();
    requestAnimationFrame(loop);
})();
</script>
</body>
</html>
