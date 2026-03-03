<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space Station - Arsenal Edition</title>
<style>
    body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; }
    canvas { background:#05080f; border:2px solid #4af; max-width: 100vw; max-height: 100vh; }
    .ui { position:absolute; top:10px; right:10px; display:flex; flex-direction:column; gap:5px; z-index:10; width: 180px; background: rgba(0,0,0,0.8); padding: 10px; border-radius: 8px; border: 1px solid #4af; max-height: 90vh; overflow-y: auto; }
    button { padding:8px; font-size:11px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; border-radius: 4px; width: 100%; margin-bottom: 2px; }
    button:disabled { opacity: 0.5; cursor: not-allowed; }
    #shop { margin-top: 10px; border-top: 1px solid #4af; padding-top: 10px; }
    .section-title { font-size: 10px; color: #4af; font-weight: bold; text-transform: uppercase; margin-bottom: 5px; display: block; }
    #stats { font-size: 14px; margin-bottom: 5px; }
    .reset-btn { border-color: #f44; color: #f44; margin-top: 10px; font-size: 9px; }
</style>
</head>
<body>

<div class="ui" id="mainUI">
    <div id="stats">
        <div id="coinsDisplay">Coins: 0</div>
        <div id="gemsDisplay">Gems: 0</div>
    </div>
    <button id="bossFightBtn" onclick="startMegaBoss()" style="display:none; background:darkred;">BOSS FIGHT!</button>
    <button onclick="togglePause()">Pause</button>
    <button onclick="restartGame()">Restart</button>
    
    <div id="weaponShop">
        <span class="section-title">Våpen & Upgrades</span>
        <button id="unlockBtn" onclick="buyWeapon('pistol', 100)">Unlock Pistol (100c)</button>
        <button id="upgradePistolBtn" onclick="upgradeWeapon('pistol')">Upgrade Pistol</button>
        
        <button id="buyShotgunBtn" onclick="buyWeapon('shotgun', 750)">Buy Shotgun (750c)</button>
        <button id="upgradeShotgunBtn" onclick="upgradeWeapon('shotgun')">Upgrade Shotgun (1000c)</button>

        <button id="buyARBtn" onclick="buyWeapon('ar', 1200)">Buy AR (1200c)</button>
        <button id="upgradeARBtn" onclick="upgradeWeapon('ar')">Upgrade AR (1500c)</button>
        
        <button id="rebirthBtn" onclick="rebirth()" style="display:none; background: gold; color: black; font-weight: bold;">REBIRTH (500c)</button>
    </div>

    <div id="shop">
        <span class="section-title">Boosters (Gems)</span>
        <button onclick="buyBooster('armor', 50)">🛡️ Armor (50g)</button>
        <button onclick="buyBooster('doubleDamage', 50)">🔥 2x Dmg (50g)</button>
    </div>
    <button class="reset-btn" onclick="resetGameData()">RESET ALL DATA</button>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");
const BASE_SPEED = 1.3;

let player, enemies, bullets, stars, particles, floatingTexts;
let score = 0, gameOver = false, paused = false, shootCooldown = 0;
let keys = {};
let gemMilestone = 10000;
let enemyExtraSpawn = 0;
let megaBoss = null;

// Lagring og State
let coins = Number(localStorage.getItem("coins")) || 100;
let gems = Number(localStorage.getItem("gems")) || 10;
let highscore = Number(localStorage.getItem("highscore")) || 0;

// Våpen stats
let activeWeapon = localStorage.getItem("activeWeapon") || "none";
let weaponsOwned = JSON.parse(localStorage.getItem("weaponsOwned")) || { pistol: false, shotgun: false, ar: false };
let weaponLevels = JSON.parse(localStorage.getItem("weaponLevels")) || { pistol: 0, shotgun: 0, ar: 0 };

const weaponConfigs = {
    pistol: { cooldown: [25, 18, 12], maxLvl: 2, type: "single" },
    shotgun: { cooldown: [40, 30], maxLvl: 1, type: "triple" },
    ar: { cooldown: [8, 5], maxLvl: 1, type: "fast" }
};

function updateUI() {
    document.getElementById("coinsDisplay").innerText = `Coins: ${Math.floor(coins)}`;
    document.getElementById("gemsDisplay").innerText = `Gems: ${gems}`;
    
    // Vise Boss-knapp
    document.getElementById("bossFightBtn").style.display = (score >= 50000 && !megaBoss) ? "block" : "none";

    // Pistol knapper
    const unlockBtn = document.getElementById("unlockBtn");
    unlockBtn.style.display = weaponsOwned.pistol ? "none" : "block";
    unlockBtn.disabled = score < 1000;
    
    const upgPistol = document.getElementById("upgradePistolBtn");
    upgPistol.style.display = weaponsOwned.pistol ? "block" : "none";
    let pCost = (weaponLevels.pistol + 1) * 300;
    upgPistol.innerText = weaponLevels.pistol >= 2 ? "Pistol Maxed" : `Upgrade Pistol (${pCost}c)`;
    upgPistol.disabled = weaponLevels.pistol >= 2;

    // Shotgun knapper
    document.getElementById("buyShotgunBtn").style.display = weaponsOwned.shotgun ? "none" : "block";
    const upgShotgun = document.getElementById("upgradeShotgunBtn");
    upgShotgun.style.display = weaponsOwned.shotgun ? "block" : "none";
    upgShotgun.innerText = weaponLevels.shotgun >= 1 ? "Shotgun Maxed" : "Upgrade Shotgun (1000c)";
    upgShotgun.disabled = weaponLevels.shotgun >= 1;

    // AR knapper
    document.getElementById("buyARBtn").style.display = weaponsOwned.ar ? "none" : "block";
    const upgAR = document.getElementById("upgradeARBtn");
    upgAR.style.display = weaponsOwned.ar ? "block" : "none";
    upgAR.innerText = weaponLevels.ar >= 1 ? "AR Maxed" : "Upgrade AR (1500c)";
    upgAR.disabled = weaponLevels.ar >= 1;

    // Rebirth Logikk (Kun når pistolen er nivå 2)
    const canRebirth = weaponLevels.pistol >= 2;
    document.getElementById("rebirthBtn").style.display = canRebirth ? "block" : "none";
}

function buyWeapon(type, cost) {
    if (coins >= cost) {
        coins -= cost;
        weaponsOwned[type] = true;
        activeWeapon = type;
        saveProgress();
        updateUI();
    }
}

function upgradeWeapon(type) {
    let cost = type === 'pistol' ? (weaponLevels.pistol + 1) * 300 : (type === 'shotgun' ? 1000 : 1500);
    if (weaponsOwned[type] && weaponLevels[type] < weaponConfigs[type].maxLvl && coins >= cost) {
        coins -= cost;
        weaponLevels[type]++;
        activeWeapon = type;
        saveProgress();
        updateUI();
    }
}

function rebirth() {
    if (coins >= 500) {
        coins -= 500;
        gems += 30;
        weaponLevels = { pistol: 0, shotgun: 0, ar: 0 };
        weaponsOwned = { pistol: false, shotgun: false, ar: false };
        activeWeapon = "none";
        saveProgress();
        init();
        alert("Rebirth utført! +30 Gems.");
    } else {
        alert("Trenger 500 coins for Rebirth!");
    }
}

function saveProgress() {
    localStorage.setItem("coins", coins);
    localStorage.setItem("gems", gems);
    localStorage.setItem("highscore", highscore);
    localStorage.setItem("activeWeapon", activeWeapon);
    localStorage.setItem("weaponsOwned", JSON.stringify(weaponsOwned));
    localStorage.setItem("weaponLevels", JSON.stringify(weaponLevels));
}

function init() {
    player = { x: 180, y: 540, width: 35, height: 35, speed: 7 * BASE_SPEED, armorUsed: false, armorActive: false };
    enemies = []; bullets = []; particles = []; floatingTexts = [];
    stars = Array.from({length:50}, () => ({x: Math.random()*400, y: Math.random()*600, s: (1+Math.random()*2) * BASE_SPEED}));
    score = 0; gemMilestone = 10000; enemyExtraSpawn = 0;
    megaBoss = null; gameOver = false; paused = false;
    updateUI();
}

function spawnEnemy(forceWave = false) {
    if (paused || gameOver) return;
    if (megaBoss && !forceWave) return;
    const count = forceWave ? 10 : (Math.floor(Math.random() * 3) + 1 + enemyExtraSpawn);
    for(let i = 0; i < count; i++) {
        enemies.push({x: Math.random()*370, y: -40, w: 30, h: 30, speedY: (2.5 + score/4000) * BASE_SPEED, hp: 1, color: '#f44', coins: 10});
    }
}

function fire() {
    if (activeWeapon === "none") return;
    const config = weaponConfigs[activeWeapon];
    const lvl = weaponLevels[activeWeapon];
    
    if (config.type === "single" || config.type === "fast") {
        bullets.push({x: player.x + player.width/2 - 3, y: player.y, vx: 0, vy: -12 * BASE_SPEED});
    } else if (config.type === "triple") {
        bullets.push({x: player.x + player.width/2 - 3, y: player.y, vx: -2, vy: -11 * BASE_SPEED});
        bullets.push({x: player.x + player.width/2 - 3, y: player.y, vx: 0, vy: -12 * BASE_SPEED});
        bullets.push({x: player.x + player.width/2 - 3, y: player.y, vx: 2, vy: -11 * BASE_SPEED});
    }
    shootCooldown = config.cooldown[lvl];
}

function update() {
    if (gameOver || paused) {
        particles.forEach((p, i) => { p.x += p.vx; p.y += p.vy; p.life -= 0.02; if(p.life<=0) particles.splice(i,1); });
        return;
    }

    if (activeWeapon !== "none" && shootCooldown <= 0) fire();
    if (shootCooldown > 0) shootCooldown--;

    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });
    if ((keys['a'] || keys['arrowleft']) && player.x > 0) player.x -= player.speed;
    if ((keys['d'] || keys['arrowright']) && player.x < 400 - player.width) player.x += player.speed;

    bullets.forEach((b, i) => {
        b.x += b.vx; b.y += b.vy;
        if (b.y < -20) bullets.splice(i, 1);
    });

    enemies.forEach((e, ei) => {
        e.y += e.speedY;
        if (player.x < e.x + e.w && player.x + player.width > e.x && player.y < e.y + e.h && player.y + player.height > e.y) {
            handlePlayerHit();
        }
        bullets.forEach((b, bi) => {
            if (b.x < e.x + e.w && b.x + 6 > e.x && b.y < e.y + e.h && b.y + 12 > e.y) {
                if (Math.random() < 0.017) { gems += 5; floatingTexts.push({x: e.x, y: e.y, text: "CRIT! +5G", color: "#a4f", life: 1}); }
                coins += e.coins; score += 100;
                createExplosion(e.x+15, e.y+15, e.color);
                enemies.splice(ei, 1); bullets.splice(bi, 1);
                updateUI();
            }
        });
        if (e.y > 600) enemies.splice(ei, 1);
    });

    score += 0.3;
    if (score >= gemMilestone) { gems += 2; enemyExtraSpawn++; gemMilestone += 10000; updateUI(); }
}

function handlePlayerHit() {
    const armorBooster = JSON.parse(localStorage.getItem("boosters"))?.armor; // Forenklet sjekk
    // Siden vi bruker globale boosters variabler i din kode:
    if (boosters.armor && !player.armorUsed) {
        player.armorUsed = true;
        createExplosion(player.x, player.y, "#4af");
    } else {
        gameOver = true;
        createExplosion(player.x, player.y, "#0f0");
        if (score > highscore) { highscore = Math.floor(score); saveProgress(); }
    }
}

function createExplosion(x, y, color) {
    for (let i = 0; i < 15; i++) {
        particles.push({x, y, vx: (Math.random()-0.5)*8, vy: (Math.random()-0.5)*8, life: 1, color});
    }
}

function draw() {
    ctx.clearRect(0,0,400,600);
    stars.forEach(s => { ctx.fillStyle='white'; ctx.fillRect(s.x,s.y,2,2); });
    particles.forEach(p => { ctx.globalAlpha = p.life; ctx.fillStyle = p.color; ctx.fillRect(p.x, p.y, 4, 4); });
    ctx.globalAlpha = 1;
    
    if (!gameOver) {
        ctx.fillStyle = (boosters.armor && !player.armorUsed) ? '#4af' : '#0f0';
        ctx.fillRect(player.x, player.y, player.width, player.height);
    }
    
    bullets.forEach(b => { ctx.fillStyle = 'yellow'; ctx.fillRect(b.x, b.y, 6, 12); });
    enemies.forEach(e => { ctx.fillStyle = e.color; ctx.fillRect(e.x, e.y, e.w, e.h); });
    
    ctx.fillStyle = 'white'; ctx.font = 'bold 16px Arial';
    ctx.fillText(`Score: ${Math.floor(score)}`, 10, 25);
    if(gameOver) ctx.fillText("GAME OVER - Press Restart", 100, 300);
}

// Controls
window.addEventListener("keydown", e => keys[e.key.toLowerCase()] = true);
window.addEventListener("keyup", e => keys[e.key.toLowerCase()] = false);
function togglePause() { paused = !paused; }
function restartGame() { init(); }
function resetGameData() { localStorage.clear(); location.reload(); }
function buyBooster(type, cost) { if(gems >= cost) { gems -= cost; boosters[type] = true; updateUI(); saveProgress(); } }

init();
setInterval(spawnEnemy, 600);
(function loop(){ update(); draw(); requestAnimationFrame(loop); })();
</script>
</body>
</html>
