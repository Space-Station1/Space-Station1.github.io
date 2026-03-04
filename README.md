<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space Station - Wave Master Edition</title>
<style>
    body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; touch-action: none; }
    canvas { background:#05080f; border:2px solid #4af; max-width: 100vw; max-height: 100vh; cursor: crosshair; }
    .ui { position:absolute; top:10px; right:10px; display:flex; flex-direction:column; gap:5px; z-index:10; width: 200px; background: rgba(0,0,0,0.85); padding: 10px; border-radius: 8px; border: 1px solid #4af; max-height: 90vh; overflow-y: auto; }
    button { padding:8px; font-size:11px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; border-radius: 4px; width: 100%; margin-bottom: 2px; }
    button:active { background: #4af; }
    button.active-wpn { background: #004400; border-color: #0f0; color: #0f0; }
    button:disabled { opacity: 0.3; cursor: not-allowed; }
    #shop { margin-top: 10px; border-top: 1px solid #4af; padding-top: 10px; }
    .section-title { font-size: 10px; color: #4af; font-weight: bold; text-transform: uppercase; margin-bottom: 5px; display: block; }
    #stats { font-size: 13px; margin-bottom: 5px; line-height: 1.4; }
    .reset-btn { border-color: #f44; color: #f44; margin-top: 10px; font-size: 9px; }
    .hidden { display: none !important; }
    .wpn-group { margin-bottom: 10px; padding: 5px; border-radius: 4px; background: rgba(255,255,255,0.05); }
</style>
</head>
<body>

<div class="ui" id="mainUI">
    <button id="toggleUIBtn" onclick="toggleUI()">Skjul UI</button>
    <div id="uiContent">
        <div id="stats">
            <div id="coinsDisplay">Coins: 0</div>
            <div id="gemsDisplay">Gems: 0</div>
            <div id="highscoreDisplayUI" style="color: #4af; font-size: 11px;">Best: 0</div>
        </div>
        <button onclick="togglePause()">Pause</button>
        <button onclick="restartGame()">Restart</button>
        
        <div id="weaponShop">
            <span class="section-title">Arsenal (1-4)</span>
            <div class="wpn-group">
                <button id="pistolSelect" onclick="selectWeapon('pistol')">Pistol</button>
                <button id="unlockBtn" onclick="buyWeapon('pistol', 0)">Lås opp (1k Score)</button>
                <button id="upgradePistolBtn" onclick="upgradeWeapon('pistol')">Oppgrader</button>
            </div>
            <div class="wpn-group">
                <button id="smgSelect" onclick="selectWeapon('smg')">SMG</button>
                <button id="buySMGBtn" onclick="buyWeapon('smg', 500)">Kjøp SMG (500c)</button>
                <button id="upgradeSMGBtn" onclick="upgradeWeapon('smg')">Oppgrader</button>
            </div>
            <div class="wpn-group">
                <button id="shotgunSelect" onclick="selectWeapon('shotgun')">Shotgun</button>
                <button id="buyShotgunBtn" onclick="buyWeapon('shotgun', 750)">Kjøp (750c)</button>
                <button id="upgradeShotgunBtn" onclick="upgradeWeapon('shotgun')">Oppgrader</button>
            </div>
            <div class="wpn-group">
                <button id="arSelect" onclick="selectWeapon('ar')">AR</button>
                <button id="buyARBtn" onclick="buyWeapon('ar', 1200)">Kjøp (1200c)</button>
                <button id="upgradeARBtn" onclick="upgradeWeapon('ar')">Oppgrader</button>
            </div>
        </div>

        <div id="shop">
            <span class="section-title">Boosters</span>
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

let player, enemies, bullets, stars, particles, floatingTexts, pickups;
let score = 0, gameOver = false, paused = false, shootCooldown = 0;
let keys = {};
let uiVisible = true;
let currentWave = 1, enemiesInWave = 0, waveTimer = 0;
let magnetTimer = 0;
let megaBoss = null;

let coins = Number(localStorage.getItem("coins")) || 100;
let gems = Number(localStorage.getItem("gems")) || 10;
let highscore = Number(localStorage.getItem("highscore")) || 0;
let activeWeapon = localStorage.getItem("activeWeapon") || "none";
let weaponsOwned = JSON.parse(localStorage.getItem("weaponsOwned")) || { pistol: false, smg: false, shotgun: false, ar: false };
let weaponLevels = JSON.parse(localStorage.getItem("weaponLevels")) || { pistol: 0, smg: 0, shotgun: 0, ar: 0 };
let boosters = { armor: false, doubleDamage: false, slowEnemies: false };

const weaponConfigs = {
    pistol: { cooldown: [25, 18, 12], type: "single", dmg: 1 },
    smg: { cooldown: [6, 4], type: "single", dmg: 0.5 },
    shotgun: { cooldown: [45, 30], type: "triple", dmg: 1 },
    ar: { cooldown: [10, 6], type: "single", dmg: 1.2 }
};

function toggleUI() {
    uiVisible = !uiVisible;
    document.getElementById("uiContent").classList.toggle("hidden", !uiVisible);
    document.getElementById("toggleUIBtn").innerText = uiVisible ? "Skjul UI" : "Vis UI";
}

function selectWeapon(type) {
    if (weaponsOwned[type]) { activeWeapon = type; saveProgress(); updateUI(); }
}

function updateUI() {
    document.getElementById("coinsDisplay").innerText = `Coins: ${Math.floor(coins)}`;
    document.getElementById("gemsDisplay").innerText = `Gems: ${gems}`;
    document.getElementById("highscoreDisplayUI").innerText = `Best: ${Math.floor(highscore)}`;
    
    // Oppdaterer alle kjøp/oppgraderingsknapper dynamisk
    const wTypes = ['pistol', 'smg', 'shotgun', 'ar'];
    wTypes.forEach(t => {
        const buyBtn = document.getElementById(`buy${t.toUpperCase()}Btn`) || document.getElementById('unlockBtn');
        const upgBtn = document.getElementById(`upgrade${t.toUpperCase()}Btn`);
        const selBtn = document.getElementById(`${t}Select`);

        if (t === 'pistol') {
            buyBtn.style.display = weaponsOwned.pistol ? "none" : "block";
            buyBtn.disabled = highscore < 1000 && score < 1000;
        } else {
            buyBtn.style.display = weaponsOwned[t] ? "none" : "block";
        }

        upgBtn.style.display = weaponsOwned[t] ? "block" : "none";
        upgBtn.innerText = weaponLevels[t] >= 1 ? "Maxed" : "Oppgrader";
        upgBtn.disabled = weaponLevels[t] >= 1;

        selBtn.style.display = weaponsOwned[t] ? "block" : "none";
        selBtn.className = activeWeapon === t ? "active-wpn" : "";
    });
}

function buyWeapon(type, cost) {
    if (coins >= cost) { coins -= cost; weaponsOwned[type] = true; activeWeapon = type; saveProgress(); updateUI(); }
}

function upgradeWeapon(type) {
    let cost = 500;
    if (coins >= cost && weaponLevels[type] < 1) {
        coins -= cost; weaponLevels[type]++; saveProgress(); updateUI();
    }
}

function buyBooster(type, cost) {
    if (gems >= cost) { gems -= cost; boosters[type] = true; updateUI(); saveProgress(); }
}

function saveProgress() {
    localStorage.setItem("coins", coins); localStorage.setItem("gems", gems);
    localStorage.setItem("highscore", highscore); localStorage.setItem("activeWeapon", activeWeapon);
    localStorage.setItem("weaponsOwned", JSON.stringify(weaponsOwned));
    localStorage.setItem("weaponLevels", JSON.stringify(weaponLevels));
}

function init() {
    player = { x: 180, y: 540, width: 35, height: 35, speed: 7 * BASE_SPEED, armorUsed: false, dead: false };
    enemies = []; bullets = []; particles = []; floatingTexts = []; pickups = [];
    stars = Array.from({length:50}, () => ({x: Math.random()*400, y: Math.random()*600, s: (1+Math.random()*2) * BASE_SPEED}));
    score = 0; currentWave = 1; enemiesInWave = 0; waveTimer = 100;
    megaBoss = null; gameOver = false; paused = false; shootCooldown = 0; magnetTimer = 0;
    updateUI();
}

function createExplosion(x, y, color, amount = 15) {
    for (let i = 0; i < amount; i++) {
        particles.push({x, y, vx: (Math.random()-0.5)*10, vy: (Math.random()-0.5)*10, life: 1, color});
    }
}

function spawnEnemy() {
    if (paused || gameOver || megaBoss || waveTimer > 0) return;
    
    let typeRoll = Math.random();
    let x = Math.random() * 350;
    
    if (typeRoll < 0.15 && score > 5000) { // Heavy
        enemies.push({x, y: -50, w: 45, h: 45, speedY: 1, color: '#800', coins: 50, hp: 5 + currentWave, maxHp: 5 + currentWave, isHeavy: true});
    } else if (typeRoll < 0.40) { // Sinus (Blå)
        enemies.push({x, y: -40, w: 30, h: 30, speedY: 2, color: '#44f', coins: 20, hp: 1, isSinus: true, offset: Math.random() * 100});
    } else { // Vanlig
        enemies.push({x, y: -40, w: 30, h: 30, speedY: 2.5 + (currentWave * 0.2), color: '#f44', coins: 10, hp: 1});
    }
    
    enemiesInWave++;
    if (enemiesInWave > 10 + (currentWave * 5)) {
        currentWave++;
        enemiesInWave = 0;
        waveTimer = 150; // Pause mellom waves
    }
}

function fire() {
    if (activeWeapon === "none" || player.dead) return;
    const config = weaponConfigs[activeWeapon];
    const lvl = weaponLevels[activeWeapon];
    const d = config.dmg * (boosters.doubleDamage ? 2 : 1);

    if (config.type === "triple") {
        [ -2.5, 0, 2.5 ].forEach(vx => bullets.push({x: player.x + 15, y: player.y, vx, vy: -12, dmg: d}));
    } else {
        bullets.push({x: player.x + 15, y: player.y, vx: 0, vy: -12, dmg: d});
    }
    shootCooldown = config.cooldown[lvl] || config.cooldown[0];
}

function update() {
    if (paused) return;

    particles.forEach((p, i) => { p.x += p.vx; p.y += p.vy; p.life -= 0.02; if(p.life <= 0) particles.splice(i,1); });
    floatingTexts.forEach((t, i) => { t.y -= 1; t.life -= 0.02; if(t.life <= 0) floatingTexts.splice(i,1); });

    if (gameOver) return;
    if (waveTimer > 0) waveTimer--;
    if (magnetTimer > 0) magnetTimer--;

    if (activeWeapon !== "none" && shootCooldown <= 0) fire();
    if (shootCooldown > 0) shootCooldown--;
    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });

    if (!player.dead) {
        if ((keys['a'] || keys['arrowleft']) && player.x > 0) player.x -= player.speed;
        if ((keys['d'] || keys['arrowright']) && player.x < 400 - player.width) player.x += player.speed;
    }

    // Pickups logikk
    pickups.forEach((p, i) => {
        if (magnetTimer > 0) {
            let dx = player.x - p.x; let dy = player.y - p.y;
            p.x += dx * 0.1; p.y += dy * 0.1;
        } else { p.y += 2; }

        if (player.x < p.x + 20 && player.x + player.width > p.x && player.y < p.y + 20 && player.y + player.height > p.y) {
            if (p.type === 'magnet') magnetTimer = 500;
            if (p.type === 'gold') coins += 100;
            pickups.splice(i, 1);
            updateUI();
        }
    });

    bullets.forEach((b, i) => {
        b.x += b.vx; b.y += b.vy;
        if (b.y < -20) bullets.splice(i, 1);
    });

    let enemySpeedMult = boosters.slowEnemies ? 0.5 : 1.0;

    enemies.forEach((e, ei) => {
        e.y += e.speedY * enemySpeedMult;
        if (e.isSinus) e.x += Math.sin((e.y + e.offset) / 20) * 3;

        // Kollisjon spiller
        if (!player.dead && player.x < e.x + e.w && player.x + player.width > e.x && player.y < e.y + e.h && player.y + player.height > e.y) {
            if (boosters.armor && !player.armorUsed) { player.armorUsed = true; enemies.splice(ei, 1); createExplosion(e.x, e.y, "#4af"); }
            else { player.dead = true; createExplosion(player.x, player.y, "#0f0", 40); setTimeout(() => { gameOver = true; if(score > highscore) highscore = Math.floor(score); saveProgress(); }, 1000); }
        }

        // Treff av kuler
        bullets.forEach((b, bi) => {
            if (b.x < e.x + e.w && b.x + 6 > e.x && b.y < e.y + e.h && b.y + 12 > e.y) {
                e.hp -= b.dmg;
                bullets.splice(bi, 1);
                if (e.hp <= 0) {
                    createExplosion(e.x + e.w/2, e.y + e.h/2, e.color);
                    if (Math.random() < 0.05) pickups.push({x: e.x, y: e.y, type: Math.random() > 0.5 ? 'magnet' : 'gold'});
                    coins += e.coins; score += 100;
                    enemies.splice(ei, 1); updateUI();
                }
            }
        });
        if (e.y > 600) enemies.splice(ei, 1);
    });

    if (!player.dead) score += 0.2;
}

function draw() {
    ctx.clearRect(0,0,400,600);
    stars.forEach(s => { ctx.fillStyle='white'; ctx.fillRect(s.x,s.y,2,2); });
    
    pickups.forEach(p => {
        ctx.fillStyle = p.type === 'magnet' ? '#f0f' : '#ffd700';
        ctx.beginPath(); ctx.arc(p.x + 10, p.y + 10, 8, 0, Math.PI*2); ctx.fill();
    });

    particles.forEach(p => { ctx.globalAlpha = p.life; ctx.fillStyle = p.color; ctx.fillRect(p.x, p.y, 4, 4); });
    ctx.globalAlpha = 1;

    if (!player.dead) {
        if (boosters.armor && !player.armorUsed) {
            ctx.strokeStyle = "#4af"; ctx.lineWidth = 2; ctx.beginPath();
            ctx.arc(player.x+17, player.y+17, 25, 0, Math.PI*2); ctx.stroke();
        }
        ctx.fillStyle = '#0f0'; ctx.fillRect(player.x, player.y, player.width, player.height);
    }

    bullets.forEach(b => { ctx.fillStyle = 'yellow'; ctx.fillRect(b.x, b.y, 4, 10); });
    
    enemies.forEach(e => { 
        ctx.fillStyle = e.color; ctx.fillRect(e.x, e.y, e.w, e.h); 
        if (e.isHeavy) {
            ctx.fillStyle = "red"; ctx.fillRect(e.x, e.y - 8, e.w, 5);
            ctx.fillStyle = "lime"; ctx.fillRect(e.x, e.y - 8, e.w * (e.hp/e.maxHp), 5);
        }
    });

    // UI Overlay på Canvas
    ctx.fillStyle = 'white'; ctx.font = 'bold 16px Arial';
    ctx.fillText(`Score: ${Math.floor(score)}`, 10, 25);
    ctx.fillStyle = '#4af'; ctx.fillText(`Wave: ${currentWave}`, 10, 45);
    
    if (waveTimer > 0) {
        ctx.fillStyle = "yellow"; ctx.font = "20px Arial";
        ctx.fillText(`WAVE ${currentWave} STARTS...`, 110, 300);
    }
    if (magnetTimer > 0) {
        ctx.fillStyle = "#f0f"; ctx.font = "12px Arial";
        ctx.fillText(`MAGNET ACTIVE!`, 10, 65);
    }

    if(gameOver) { ctx.fillStyle="red"; ctx.font="30px Arial"; ctx.fillText("GAME OVER", 110, 300); }
}

window.addEventListener("keydown", e => { 
    keys[e.key.toLowerCase()] = true; 
    if (e.key === '1') selectWeapon('pistol');
    if (e.key === '2') selectWeapon('smg');
    if (e.key === '3') selectWeapon('shotgun');
    if (e.key === '4') selectWeapon('ar');
});
window.addEventListener("keyup", e => { keys[e.key.toLowerCase()] = false; });

const handleMove = (e) => {
    if(paused || gameOver || player.dead) return;
    const rect = canvas.getBoundingClientRect();
    const clientX = e.touches ? e.touches[0].clientX : e.clientX;
    const x = (clientX - rect.left) * (canvas.width / rect.width);
    player.x = x - player.width / 2;
};
canvas.addEventListener("mousemove", handleMove);
canvas.addEventListener("touchmove", (e) => { e.preventDefault(); handleMove(e); }, { passive: false });

function togglePause() { paused = !paused; }
function restartGame() { init(); }
function resetGameData() { if(confirm("Reset alt?")) { localStorage.clear(); location.reload(); } }

init();
setInterval(spawnEnemy, 600);
(function loop(){ update(); draw(); requestAnimationFrame(loop); })();
</script>
</body>
</html>
