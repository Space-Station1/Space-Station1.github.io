<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space Station - Mega Edition Fixed</title>
<style>
    body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; touch-action: none; }
    canvas { background:#05080f; border:2px solid #4af; max-width: 100vw; max-height: 100vh; cursor: crosshair; }
    .ui { position:absolute; top:10px; right:10px; display:flex; flex-direction:column; gap:5px; z-index:10; width: 190px; background: rgba(0,0,0,0.85); padding: 10px; border-radius: 8px; border: 1px solid #4af; max-height: 90vh; overflow-y: auto; }
    button { padding:8px; font-size:11px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; border-radius: 4px; width: 100%; margin-bottom: 2px; }
    button:active { background: #4af; }
    button:disabled { opacity: 0.3; cursor: not-allowed; }
    #shop { margin-top: 10px; border-top: 1px solid #4af; padding-top: 10px; }
    .section-title { font-size: 10px; color: #4af; font-weight: bold; text-transform: uppercase; margin-bottom: 5px; display: block; }
    #stats { font-size: 13px; margin-bottom: 5px; line-height: 1.4; }
    .reset-btn { border-color: #f44; color: #f44; margin-top: 10px; font-size: 9px; }
    .hidden { display: none !important; }
    #bossFightBtn { background: darkred !important; color: white; font-weight: bold; display: none; }
</style>
</head>
<body>

<div class="ui" id="mainUI">
    <button id="toggleUIBtn" onclick="toggleUI()">Skjul UI</button>
    <div id="uiContent">
        <div id="stats">
            <div id="coinsDisplay">Coins: 0</div>
            <div id="gemsDisplay">Gems: 0</div>
            <div id="highscoreDisplay" style="color: #4af; font-size: 11px;">Highscore: 0</div>
        </div>
        <button id="bossFightBtn" onclick="startMegaBoss()">BOSS FIGHT!</button>
        <button onclick="togglePause()">Pause</button>
        <button onclick="restartGame()">Restart</button>
        
        <div id="weaponShop">
            <span class="section-title">Våpen & Upgrades</span>
            <button id="unlockBtn" onclick="buyWeapon('pistol', 0)">Lås opp Pistol (1000 Score)</button>
            <button id="upgradePistolBtn" onclick="upgradeWeapon('pistol')">Oppgrader Pistol</button>
            
            <button id="buyShotgunBtn" onclick="buyWeapon('shotgun', 750)">Kjøp Shotgun (750c)</button>
            <button id="upgradeShotgunBtn" onclick="upgradeWeapon('shotgun')">Oppgrader Shotgun (1000c)</button>

            <button id="buyARBtn" onclick="buyWeapon('ar', 1200)">Kjøp AR (1200c)</button>
            <button id="upgradeARBtn" onclick="upgradeWeapon('ar')">Oppgrader AR (1500c)</button>
            
            <button id="rebirthBtn" onclick="rebirth()" style="display:none; background: gold !important; color: black; font-weight: bold;">REBIRTH (500c)</button>
        </div>

        <div id="shop">
            <span class="section-title">Boosters (Gems)</span>
            <button onclick="buyBooster('armor', 50)">🛡️ Armor (50g)</button>
            <button onclick="buyBooster('doubleDamage', 50)">🔥 2x Dmg (50g)</button>
            <button onclick="buyBooster('slowEnemies', 50)">❄️ Slow (50g)</button>
        </div>
        <button class="reset-btn" onclick="resetGameData()">RESET ALL DATA</button>
    </div>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");
const BASE_SPEED = 1.3;

let player, enemies, bullets, stars, particles, floatingTexts;
let score = 0, gameOver = false, paused = false, shootCooldown = 0;
let keys = {};
let uiVisible = true;
let gemMilestone = 10000;
let enemyExtraSpawn = 0;
let megaBoss = null;

// Tilstand og Lagring
let coins = Number(localStorage.getItem("coins")) || 100;
let gems = Number(localStorage.getItem("gems")) || 10;
let highscore = Number(localStorage.getItem("highscore")) || 0;
let activeWeapon = localStorage.getItem("activeWeapon") || "none";
let weaponsOwned = JSON.parse(localStorage.getItem("weaponsOwned")) || { pistol: false, shotgun: false, ar: false };
let weaponLevels = JSON.parse(localStorage.getItem("weaponLevels")) || { pistol: 0, shotgun: 0, ar: 0 };
let boosters = { armor: false, doubleDamage: false, slowEnemies: false };

const weaponConfigs = {
    pistol: { cooldown: [25, 18, 12], maxLvl: 2, type: "single" },
    shotgun: { cooldown: [45, 30], maxLvl: 1, type: "triple" },
    ar: { cooldown: [10, 6], maxLvl: 1, type: "fast" }
};

function toggleUI() {
    uiVisible = !uiVisible;
    document.getElementById("uiContent").classList.toggle("hidden", !uiVisible);
    document.getElementById("toggleUIBtn").innerText = uiVisible ? "Skjul UI" : "Vis UI";
}

function updateUI() {
    document.getElementById("coinsDisplay").innerText = `Coins: ${Math.floor(coins)}`;
    document.getElementById("gemsDisplay").innerText = `Gems: ${gems}`;
    document.getElementById("highscoreDisplay").innerText = `Highscore: ${Math.floor(highscore)}`;
    
    document.getElementById("bossFightBtn").style.display = (score >= 50000 && !megaBoss) ? "block" : "none";
    
    // Pistol Unlock logikk (Basert på highscore eller nåværende score)
    const unlockBtn = document.getElementById("unlockBtn");
    unlockBtn.style.display = weaponsOwned.pistol ? "none" : "block";
    unlockBtn.disabled = (highscore < 1000 && score < 1000);

    // Oppgradering Pistol
    const upgPistol = document.getElementById("upgradePistolBtn");
    upgPistol.style.display = weaponsOwned.pistol ? "block" : "none";
    let pCost = (weaponLevels.pistol + 1) * 300;
    upgPistol.innerText = weaponLevels.pistol >= 2 ? "Pistol Maxed" : `Oppgrader Pistol (${pCost}c)`;
    upgPistol.disabled = weaponLevels.pistol >= 2 || coins < pCost;

    // Shotgun
    document.getElementById("buyShotgunBtn").style.display = weaponsOwned.shotgun ? "none" : "block";
    document.getElementById("buyShotgunBtn").disabled = coins < 750;
    document.getElementById("upgradeShotgunBtn").style.display = weaponsOwned.shotgun ? "block" : "none";
    document.getElementById("upgradeShotgunBtn").disabled = weaponLevels.shotgun >= 1 || coins < 1000;

    // AR
    document.getElementById("buyARBtn").style.display = weaponsOwned.ar ? "none" : "block";
    document.getElementById("buyARBtn").disabled = coins < 1200;
    document.getElementById("upgradeARBtn").style.display = weaponsOwned.ar ? "block" : "none";
    document.getElementById("upgradeARBtn").disabled = weaponLevels.ar >= 1 || coins < 1500;

    // Rebirth
    document.getElementById("rebirthBtn").style.display = (weaponLevels.pistol >= 2) ? "block" : "none";
    document.getElementById("rebirthBtn").disabled = coins < 500;
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
        coins -= 500; gems += 30;
        weaponLevels = { pistol: 0, shotgun: 0, ar: 0 };
        weaponsOwned = { pistol: false, shotgun: false, ar: false };
        activeWeapon = "none";
        saveProgress(); init();
    }
}

function buyBooster(type, cost) {
    if (gems >= cost) { gems -= cost; boosters[type] = true; updateUI(); saveProgress(); }
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
    player = { x: 180, y: 540, width: 35, height: 35, speed: 7 * BASE_SPEED, armorUsed: false };
    enemies = []; bullets = []; particles = []; floatingTexts = [];
    stars = Array.from({length:50}, () => ({x: Math.random()*400, y: Math.random()*600, s: (1+Math.random()*2) * BASE_SPEED}));
    score = 0; gemMilestone = 10000; enemyExtraSpawn = 0;
    megaBoss = null; gameOver = false; paused = false; shootCooldown = 0;
    updateUI();
}

function createExplosion(x, y, color) {
    for (let i = 0; i < 20; i++) {
        particles.push({x, y, vx: (Math.random()-0.5)*10, vy: (Math.random()-0.5)*10, life: 1, color});
    }
}

function startMegaBoss() {
    megaBoss = { x: 150, y: -100, w: 100, h: 80, hp: 150, maxHp: 150, speedX: 1.5, speedY: 0.3, p1: false, p2: false };
    updateUI();
}

function spawnEnemy(forceWave = false) {
    if (paused || gameOver) return;
    if (megaBoss && !forceWave) return;
    const count = forceWave ? 12 : (Math.floor(Math.random() * 3) + 1 + enemyExtraSpawn);
    for(let i = 0; i < count; i++) {
        enemies.push({x: Math.random()*370, y: -40, w: 30, h: 30, speedY: (2.5 + score/4500) * BASE_SPEED, color: '#f44', coins: 10});
    }
}

function fire() {
    if (activeWeapon === "none") return;
    const config = weaponConfigs[activeWeapon];
    const lvl = weaponLevels[activeWeapon];
    if (config.type === "triple") {
        bullets.push({x: player.x + 15, y: player.y, vx: -2.5, vy: -11});
        bullets.push({x: player.x + 15, y: player.y, vx: 0, vy: -12});
        bullets.push({x: player.x + 15, y: player.y, vx: 2.5, vy: -11});
    } else {
        bullets.push({x: player.x + 15, y: player.y, vx: 0, vy: -12});
    }
    shootCooldown = config.cooldown[lvl];
}

function update() {
    particles.forEach((p, i) => { p.x += p.vx; p.y += p.vy; p.life -= 0.02; if(p.life <= 0) particles.splice(i,1); });
    floatingTexts.forEach((t, i) => { t.y -= 1; t.life -= 0.02; if(t.life <= 0) floatingTexts.splice(i,1); });

    if (gameOver || paused) return;

    if (activeWeapon !== "none" && shootCooldown <= 0) fire();
    if (shootCooldown > 0) shootCooldown--;

    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });

    if ((keys['a'] || keys['arrowleft']) && player.x > 0) player.x -= player.speed;
    if ((keys['d'] || keys['arrowright']) && player.x < 400 - player.width) player.x += player.speed;

    bullets.forEach((b, i) => {
        b.x += b.vx; b.y += b.vy;
        if (b.y < -20 || b.x < -20 || b.x > 420) bullets.splice(i, 1);
    });

    let enemySpeedMult = boosters.slowEnemies ? 0.5 : 1.0;

    if (megaBoss) {
        megaBoss.x += megaBoss.speedX * enemySpeedMult; 
        megaBoss.y += megaBoss.speedY * enemySpeedMult;
        if (megaBoss.x <= 0 || megaBoss.x + megaBoss.w >= 400) megaBoss.speedX *= -1;
        
        bullets.forEach((b, bi) => {
            if (b.x < megaBoss.x + megaBoss.w && b.x + 6 > megaBoss.x && b.y < megaBoss.y + megaBoss.h && b.y + 12 > megaBoss.y) {
                megaBoss.hp -= (boosters.doubleDamage ? 2 : 1);
                bullets.splice(bi, 1);
                if (megaBoss.hp <= 0) { coins += 500; score += 10000; megaBoss = null; updateUI(); }
            }
        });
    }

    enemies.forEach((e, ei) => {
        e.y += e.speedY * enemySpeedMult;
        if (player.x < e.x + e.w && player.x + player.width > e.x && player.y < e.y + e.h && player.y + player.height > e.y) {
            if (boosters.armor && !player.armorUsed) { 
                player.armorUsed = true; enemies.splice(ei, 1); createExplosion(player.x+17, player.y, "#4af");
            } else { 
                gameOver = true; createExplosion(player.x+17, player.y+17, "#0f0");
                if(score > highscore) { highscore = Math.floor(score); saveProgress(); }
            }
        }
        bullets.forEach((b, bi) => {
            if (b.x < e.x + e.w && b.x + 6 > e.x && b.y < e.y + e.h && b.y + 12 > e.y) {
                if (Math.random() < 0.017) { gems += 5; floatingTexts.push({x: e.x, y: e.y, text: "CRIT! +5G", color: "#a4f", life: 1}); }
                coins += e.coins; score += 100; createExplosion(e.x+15, e.y+15, e.color);
                enemies.splice(ei, 1); bullets.splice(bi, 1); updateUI();
            }
        });
        if (e.y > 600) enemies.splice(ei, 1);
    });

    score += 0.3;
    if (score >= gemMilestone) { gems += 2; enemyExtraSpawn++; gemMilestone += 10000; updateUI(); }
}

function draw() {
    ctx.clearRect(0,0,400,600);
    stars.forEach(s => { ctx.fillStyle='white'; ctx.fillRect(s.x,s.y,2,2); });
    particles.forEach(p => { ctx.globalAlpha = p.life; ctx.fillStyle = p.color; ctx.fillRect(p.x, p.y, 4, 4); });
    floatingTexts.forEach(t => { ctx.globalAlpha = t.life; ctx.fillStyle = t.color; ctx.font="bold 14px Arial"; ctx.fillText(t.text, t.x, t.y); });
    ctx.globalAlpha = 1;
    
    if (megaBoss) {
        ctx.fillStyle = "yellow"; ctx.fillRect(megaBoss.x, megaBoss.y, megaBoss.w, megaBoss.h);
        ctx.fillStyle = "red"; ctx.fillRect(50, 10, 300, 12);
        ctx.fillStyle = "yellow"; ctx.fillRect(50, 10, 300 * (megaBoss.hp/megaBoss.maxHp), 12);
    }

    if (!gameOver) {
        ctx.fillStyle = (boosters.armor && !player.armorUsed) ? '#4af' : '#0f0';
        ctx.fillRect(player.x, player.y, player.width, player.height);
    }
    
    bullets.forEach(b => { ctx.fillStyle = 'yellow'; ctx.fillRect(b.x, b.y, 6, 12); });
    enemies.forEach(e => { ctx.fillStyle = e.color; ctx.fillRect(e.x, e.y, e.w, e.h); });
    
    ctx.fillStyle = 'white'; ctx.font = 'bold 16px Arial';
    ctx.fillText(`Score: ${Math.floor(score)}`, 10, 25);
    if(gameOver) { ctx.fillStyle="red"; ctx.font="30px Arial"; ctx.fillText("GAME OVER", 110, 300); }
}

window.addEventListener("keydown", e => { keys[e.key.toLowerCase()] = true; });
window.addEventListener("keyup", e => { keys[e.key.toLowerCase()] = false; });

const handleMove = (e) => {
    if(paused || gameOver) return;
    const rect = canvas.getBoundingClientRect();
    const clientX = e.touches ? e.touches[0].clientX : e.clientX;
    const x = (clientX - rect.left) * (canvas.width / rect.width);
    player.x = x - player.width / 2;
    if (player.x < 0) player.x = 0;
    if (player.x > canvas.width - player.width) player.x = canvas.width - player.width;
};
canvas.addEventListener("mousemove", handleMove);
canvas.addEventListener("touchmove", (e) => { e.preventDefault(); handleMove(e); }, { passive: false });

function togglePause() { paused = !paused; }
function restartGame() { init(); }
function resetGameData() { if(confirm("Slette alt?")) { localStorage.clear(); location.reload(); } }

init();
setInterval(spawnEnemy, 600);
(function loop(){ update(); draw(); requestAnimationFrame(loop); })();
</script>
</body>
</html>
