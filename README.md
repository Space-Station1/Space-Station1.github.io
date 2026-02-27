<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space Station - Extreme Mode</title>
<style>
    body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; }
    canvas { background:#05080f; border:2px solid #4af; max-width: 100vw; max-height: 100vh; }
    .ui { position:absolute; top:10px; right:10px; display:flex; flex-direction:column; gap:5px; z-index:10; width: 150px; background: rgba(0,0,0,0.7); padding: 10px; border-radius: 8px; border: 1px solid #4af; }
    button { padding:8px; font-size:12px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; border-radius: 4px; width: 100%; }
    button:active { background: #4af; }
    #shop { margin-top: 10px; border-top: 1px solid #4af; padding-top: 10px; display: flex; flex-direction: column; gap: 5px; }
    .hidden { display: none !important; }
    #stats { font-size: 14px; margin-bottom: 5px; }
</style>
</head>
<body>

<div class="ui" id="mainUI">
    <button id="toggleUIBtn" onclick="toggleUI()">Skjul UI</button>
    <div id="uiContent">
        <div id="stats">
            <div id="coinsDisplay">Coins: 0</div>
            <div id="gemsDisplay">Gems: 0</div>
        </div>
        <button onclick="togglePause()">Pause</button>
        <button onclick="restartGame()">Restart</button>
        <button id="unlockBtn" onclick="unlockGun()">Unlock Gun (100c)</button>
        <button onclick="upgradeWeapon()">Upgrade Gun</button>
        <div id="shop">
            <strong style="font-size: 10px; color: #4af;">SHOP (1 RUNDE)</strong>
            <button onclick="buyBooster('armor', 5)">🛡️ Armor (5g)</button>
            <button onclick="buyBooster('doubleDamage', 10)">🔥 2x Dmg (10g)</button>
            <button onclick="buyBooster('slowEnemies', 15)">❄️ Slow (15g)</button>
        </div>
    </div>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

const SPEED_BOOST = 1.3;

let player, enemies, bullets, stars;
let score = 0, gameOver = false, paused = false;
let keys = {};
let uiVisible = true;
let gemMilestone = 1000;

let coins = Number(localStorage.getItem("coins")) || 0;
let gems = Number(localStorage.getItem("gems")) || 10;
let upgradeLevel = Number(localStorage.getItem("upgradeLevel")) || 0;
let hasGun = localStorage.getItem("hasGun") === "true";
let highscore = Number(localStorage.getItem("highscore")) || 0;

let boosters = { armor: false, doubleDamage: false, slowEnemies: false };
const cooldownSettings = [25, 18, 12, 8, 5, 3];
let shootCooldown = 0;

function toggleUI() {
    uiVisible = !uiVisible;
    document.getElementById("uiContent").classList.toggle("hidden", !uiVisible);
    document.getElementById("toggleUIBtn").innerText = uiVisible ? "Skjul UI" : "Vis UI";
}

function updateUI() {
    document.getElementById("coinsDisplay").innerText = `Coins: ${Math.floor(coins)}`;
    document.getElementById("gemsDisplay").innerText = `Gems: ${gems}`;
    document.getElementById("unlockBtn").style.display = hasGun ? "none" : "block";
}

function saveProgress() {
    localStorage.setItem("coins", coins);
    localStorage.setItem("gems", gems);
    localStorage.setItem("upgradeLevel", upgradeLevel);
    localStorage.setItem("hasGun", hasGun);
    localStorage.setItem("highscore", highscore);
}

function buyBooster(type, cost) {
    if (boosters[type]) return alert("Allerede aktiv!");
    if (gems >= cost) {
        gems -= cost;
        boosters[type] = true;
        updateUI();
        saveProgress();
    } else { alert("Ikke nok Gems!"); }
}

function init() {
    boosters = { armor: false, doubleDamage: false, slowEnemies: false };
    player = { x: 180, y: 540, width: 35, height: 35, speed: 7 * SPEED_BOOST, armorUsed: false };
    enemies = []; bullets = [];
    stars = Array.from({length:50}, () => ({x: Math.random()*400, y: Math.random()*600, s: (1+Math.random()*2) * SPEED_BOOST}));
    score = 0; gemMilestone = 1000;
    gameOver = false; paused = false;
    updateUI();
}

function togglePause() { paused = !paused; }
function restartGame() { init(); }

function unlockGun() {
    if (coins >= 100) { coins -= 100; hasGun = true; saveProgress(); updateUI(); }
}

function upgradeWeapon() {
    let cost = 200 * upgradeLevel + 100;
    if (hasGun && upgradeLevel < 5 && coins >= cost) {
        coins -= cost; upgradeLevel++; saveProgress(); updateUI();
    }
}

function spawnEnemy() {
    if (paused || gameOver) return;
    
    // Økt antall fiender som kan komme samtidig (1-3 fiender per spawn)
    const count = Math.floor(Math.random() * 3) + 1;
    
    for(let i = 0; i < count; i++) {
        const r = Math.random();
        if (r < 0.85) {
            enemies.push({x: Math.random()*370, y: -40, w: 30, h: 30, speedY: (2.5 + score/4000) * SPEED_BOOST, hp: 1, maxHp: 1, color: '#f44', coins: 10});
        } else {
            enemies.push({x: Math.random()*300, y: -80, w: 60, h: 50, speedY: 1.2 * SPEED_BOOST, hp: 5, maxHp: 5, color: '#a4f', coins: 50, isBoss: true});
        }
    }
}

function update() {
    if (gameOver || paused) return;

    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });

    if ((keys['a'] || keys['arrowleft']) && player.x > 0) player.x -= player.speed;
    if ((keys['d'] || keys['arrowright']) && player.x < 400 - player.width) player.x += player.speed;

    if (hasGun && shootCooldown <= 0) {
        bullets.push({x: player.x + player.width/2 - 3, y: player.y, w: 6, h: 12, speed: 12 * SPEED_BOOST});
        shootCooldown = cooldownSettings[upgradeLevel];
    }
    if (shootCooldown > 0) shootCooldown--;

    bullets.forEach((b, bi) => {
        b.y -= b.speed;
        if (b.y < -20) bullets.splice(bi, 1);
    });

    let speedMult = boosters.slowEnemies ? 0.5 : 1;
    enemies.forEach((e, ei) => {
        e.y += e.speedY * speedMult;
        
        if (player.x < e.x + e.w && player.x + player.width > e.x && player.y < e.y + e.h && player.y + player.height > e.y) {
            if (boosters.armor && !player.armorUsed) {
                player.armorUsed = true;
                enemies.splice(ei, 1);
            } else {
                gameOver = true;
                if (score > highscore) { highscore = Math.floor(score); saveProgress(); }
            }
        }

        bullets.forEach((b, bi) => {
            if (b.x < e.x + e.w && b.x + b.w > e.x && b.y < e.y + e.h && b.y + b.h > e.y) {
                e.hp -= (boosters.doubleDamage ? 2 : 1);
                bullets.splice(bi, 1);
                if (e.hp <= 0) {
                    coins += e.coins;
                    score += e.isBoss ? 500 : 100;
                    enemies.splice(ei, 1);
                    updateUI();
                }
            }
        });
        if (e.y > 600) enemies.splice(ei, 1);
    });

    score += 0.3 * SPEED_BOOST;
    if (score >= gemMilestone) { gems += 1; gemMilestone += 1000; updateUI(); saveProgress(); }
}

function draw() {
    ctx.clearRect(0,0,400,600);
    stars.forEach(s => { ctx.fillStyle='white'; ctx.fillRect(s.x,s.y,2,2); });
    
    ctx.fillStyle = (boosters.armor && !player.armorUsed) ? '#4af' : '#0f0';
    ctx.fillRect(player.x, player.y, player.width, player.height);
    
    bullets.forEach(b => { ctx.fillStyle = boosters.doubleDamage ? 'orange' : 'yellow'; ctx.fillRect(b.x, b.y, b.w, b.h); });
    
    enemies.forEach(e => {
        ctx.fillStyle = e.color;
        ctx.fillRect(e.x, e.y, e.w, e.h);
        
        // HP Bar for store fiender (Bosses)
        if(e.isBoss) {
            ctx.fillStyle = 'red';
            ctx.fillRect(e.x, e.y - 10, e.w, 5);
            ctx.fillStyle = '#0f0';
            ctx.fillRect(e.x, e.y - 10, e.w * (e.hp / e.maxHp), 5);
        }
    });

    ctx.fillStyle = 'white';
    ctx.font = 'bold 18px Arial';
    ctx.fillText(`Score: ${Math.floor(score)}`, 15, 30);
    ctx.font = '14px Arial';
    ctx.fillText(`Highscore: ${highscore}`, 15, 50);
    
    if (paused) { 
        ctx.fillStyle='white';
        ctx.font='30px Arial'; 
        ctx.fillText('PAUSE', 150, 300); 
    }
    if (gameOver) { 
        ctx.fillStyle='red'; 
        ctx.font='30px Arial'; 
        ctx.fillText('GAME OVER', 110, 300); 
    }
}

window.addEventListener("keydown", e => keys[e.key.toLowerCase()] = true);
window.addEventListener("keyup", e => keys[e.key.toLowerCase()] = false);

canvas.addEventListener("touchmove", e => {
    e.preventDefault();
    let rect = canvas.getBoundingClientRect();
    let x = e.touches[0].clientX - rect.left;
    player.x = x - player.width / 2;
    if(player.x < 0) player.x = 0;
    if(player.x > 365) player.x = 365;
}, {passive: false});

init();
// Spawner fiender oftere (hvert 600ms)
setInterval(spawnEnemy, 600);
(function loop(){ update(); draw(); requestAnimationFrame(loop); })();
</script>
</body>
</html>
