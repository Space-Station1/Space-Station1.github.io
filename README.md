<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space Station - Fixed Edition</title>
<style>
    body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; touch-action: none; }
    canvas { background:#05080f; border:2px solid #4af; max-width: 100vw; max-height: 100vh; cursor: crosshair; display: block; }
    
    /* UI Styling */
    .ui { position:absolute; top:10px; right:10px; width: 220px; background: rgba(0,0,0,0.9); padding: 15px; border-radius: 8px; border: 1px solid #4af; z-index: 100; }
    #stats { margin-bottom: 10px; font-size: 14px; border-bottom: 1px solid #333; padding-bottom: 5px; }
    .wpn-group { margin-bottom: 12px; padding: 8px; background: rgba(255,255,255,0.05); border-radius: 5px; }
    
    button { padding:8px; font-size:12px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; border-radius: 4px; width: 100%; margin-top: 3px; }
    button:hover { background: #333; }
    button.active-wpn { background: #004400; border-color: #0f0; color: #0f0; font-weight: bold; }
    .section-title { font-size: 11px; color: #4af; font-weight: bold; text-transform: uppercase; display: block; }
    .hidden { display: none !important; }
</style>
</head>
<body>

<div class="ui" id="mainUI">
    <div id="stats">
        <div id="coinsDisplay">Coins: 0</div>
        <div id="highscoreDisplayUI">Best Score: 0</div>
        <div id="waveHighscoreDisplayUI" style="color: #4af;">Best Wave: 1</div>
    </div>
    
    <button onclick="restartGame()">Restart Spill</button>
    <button id="toggleUIBtn" onclick="toggleUI()">Skjul Shop</button>

    <div id="uiContent">
        <span class="section-title" style="margin-top:10px;">Arsenal (Tast 1-4)</span>
        
        <div class="wpn-group">
            <button id="pistolSelect" onclick="selectWeapon('pistol')">Pistol</button>
            <button id="upgradePistolBtn" onclick="upgradeWeapon('pistol')">Oppgrader (500c)</button>
        </div>

        <div class="wpn-group">
            <button id="smgSelect" onclick="selectWeapon('smg')">Kjøp SMG (500c)</button>
            <button id="upgradeSMGBtn" onclick="upgradeWeapon('smg')">Oppgrader (500c)</button>
        </div>

        <div class="wpn-group">
            <button id="shotgunSelect" onclick="selectWeapon('shotgun')">Kjøp Shotgun (750c)</button>
            <button id="upgradeShotgunBtn" onclick="upgradeWeapon('shotgun')">Oppgrader (500c)</button>
        </div>

        <div class="wpn-group">
            <button id="arSelect" onclick="selectWeapon('ar')">Kjøp AR (1200c)</button>
            <button id="upgradeARBtn" onclick="upgradeWeapon('ar')">Oppgrader (500c)</button>
        </div>
        
        <button onclick="resetGameData()" style="color:#f44; font-size:10px; margin-top:10px;">Slett Lagring</button>
    </div>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

// Spillvariabler
let player, enemies, bullets, stars, pickups, boss;
let score = 0, gameOver = false, paused = false, shootCooldown = 0;
let keys = {};
let currentWave = 1, enemiesInWave = 0, waveTimer = 100, magnetTimer = 0;

// Lagring
let coins = Number(localStorage.getItem("coins")) || 0;
let highscore = Number(localStorage.getItem("highscore")) || 0;
let waveHighscore = Number(localStorage.getItem("waveHighscore")) || 1;
let activeWeapon = localStorage.getItem("activeWeapon") || "pistol";
let weaponsOwned = JSON.parse(localStorage.getItem("weaponsOwned")) || { pistol: true, smg: false, shotgun: false, ar: false };
let weaponLevels = JSON.parse(localStorage.getItem("weaponLevels")) || { pistol: 0, smg: 0, shotgun: 0, ar: 0 };

const weaponConfigs = {
    pistol: { cooldown: [25, 15], dmg: 1, type: "single", cost: 0 },
    smg: { cooldown: [8, 5], dmg: 0.5, type: "single", cost: 500 },
    shotgun: { cooldown: [45, 30], dmg: 1, type: "triple", cost: 750 },
    ar: { cooldown: [12, 8], dmg: 1.3, type: "single", cost: 1200 }
};

function init() {
    player = { x: 180, y: 520, w: 30, h: 30, speed: 6, dead: false };
    enemies = []; bullets = []; pickups = []; boss = null;
    stars = Array.from({length:50}, () => ({x: Math.random()*400, y: Math.random()*600, s: 1 + Math.random()*2}));
    score = 0; currentWave = 1; enemiesInWave = 0; waveTimer = 100;
    gameOver = false;
    updateUI();
}

function updateUI() {
    document.getElementById("coinsDisplay").innerText = `Coins: ${Math.floor(coins)}`;
    document.getElementById("highscoreDisplayUI").innerText = `Best Score: ${Math.floor(highscore)}`;
    document.getElementById("waveHighscoreDisplayUI").innerText = `Best Wave: ${waveHighscore}`;

    for (let t in weaponConfigs) {
        let selBtn = document.getElementById(t + "Select");
        let upgBtn = document.getElementById("upgrade" + t.charAt(0).toUpperCase() + t.slice(1) + "Btn");
        
        if (weaponsOwned[t]) {
            selBtn.innerText = t.toUpperCase();
            selBtn.className = (activeWeapon === t) ? "active-wpn" : "";
            upgBtn.innerText = weaponLevels[t] >= 1 ? "Max Nivå" : "Oppgrader (500c)";
            upgBtn.disabled = weaponLevels[t] >= 1;
        } else {
            selBtn.innerText = `Kjøp ${t.toUpperCase()} (${weaponConfigs[t].cost}c)`;
            upgBtn.innerText = "Låst";
            upgBtn.disabled = true;
        }
    }
}

function selectWeapon(type) {
    if (weaponsOwned[type]) {
        activeWeapon = type;
    } else {
        if (coins >= weaponConfigs[type].cost) {
            coins -= weaponConfigs[type].cost;
            weaponsOwned[type] = true;
            activeWeapon = type;
        }
    }
    saveProgress();
    updateUI();
}

function upgradeWeapon(type) {
    if (weaponsOwned[type] && coins >= 500 && weaponLevels[type] < 1) {
        coins -= 500;
        weaponLevels[type]++;
        saveProgress();
        updateUI();
    }
}

function saveProgress() {
    localStorage.setItem("coins", coins);
    localStorage.setItem("weaponsOwned", JSON.stringify(weaponsOwned));
    localStorage.setItem("weaponLevels", JSON.stringify(weaponLevels));
    localStorage.setItem("activeWeapon", activeWeapon);
    localStorage.setItem("highscore", highscore);
    localStorage.setItem("waveHighscore", waveHighscore);
}

function spawnEnemy() {
    if (gameOver || waveTimer > 0 || boss) return;

    // BOSS Trigger: Wave 20 eller 100k score
    if (currentWave === 20 || score >= 100000) {
        if (!boss) boss = { x: 150, y: -100, w: 100, h: 80, hp: 600, maxHp: 600, speedX: 2 };
        return;
    }

    let x = Math.random() * 360;
    let roll = Math.random();
    if (roll < 0.2) { // Sinus (Blå)
        enemies.push({x, y: -40, w: 25, h: 25, speedY: 2, color: '#44f', hp: 1, isSinus: true});
    } else if (roll < 0.3 && score > 5000) { // Heavy (Mørkerød)
        enemies.push({x, y: -50, w: 40, h: 40, speedY: 1, color: '#800', hp: 6, maxHp: 6, isHeavy: true});
    } else { // Vanlig (Rød)
        enemies.push({x, y: -40, w: 25, h: 25, speedY: 2 + (currentWave * 0.1), color: '#f44', hp: 1});
    }

    enemiesInWave++;
    if (enemiesInWave > 15) {
        currentWave++;
        enemiesInWave = 0;
        waveTimer = 120;
        if (currentWave > waveHighscore) waveHighscore = currentWave;
        saveProgress();
        updateUI();
    }
}

function update() {
    if (gameOver) return;
    if (waveTimer > 0) waveTimer--;
    if (magnetTimer > 0) magnetTimer--;

    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });

    // Spillermovement
    if (!player.dead) {
        if (keys['a'] || keys['arrowleft']) player.x -= player.speed;
        if (keys['d'] || keys['arrowright']) player.x += player.speed;
        player.x = Math.max(0, Math.min(370, player.x));

        if (shootCooldown <= 0) {
            const conf = weaponConfigs[activeWeapon];
            const dmg = conf.dmg;
            if (conf.type === "triple") {
                [-2, 0, 2].forEach(vx => bullets.push({x: player.x+13, y: player.y, vx, vy: -10, dmg}));
            } else {
                bullets.push({x: player.x+13, y: player.y, vx: 0, vy: -10, dmg});
            }
            shootCooldown = conf.cooldown[weaponLevels[activeWeapon]];
        }
        shootCooldown--;
    }

    // Boss logikk
    if (boss) {
        if (boss.y < 50) boss.y += 0.5;
        boss.x += boss.speedX;
        if (boss.x <= 0 || boss.x + boss.w >= 400) boss.speedX *= -1;
    }

    // Bullets
    bullets.forEach((b, i) => {
        b.x += b.vx; b.y += b.vy;
        if (b.y < -20) bullets.splice(i, 1);
        
        if (boss && b.x > boss.x && b.x < boss.x + boss.w && b.y > boss.y && b.y < boss.y + boss.h) {
            boss.hp -= b.dmg;
            bullets.splice(i, 1);
            if (boss.hp <= 0) { coins += 1000; score += 20000; boss = null; currentWave++; updateUI(); }
        }
    });

    // Fiender
    enemies.forEach((e, i) => {
        e.y += e.speedY;
        if (e.isSinus) e.x += Math.sin(e.y/30) * 3;

        // Kollisjon med kuler
        bullets.forEach((b, bi) => {
            if (b.x > e.x && b.x < e.x + e.w && b.y > e.y && b.y < e.y + e.h) {
                e.hp -= b.dmg;
                bullets.splice(bi, 1);
                if (e.hp <= 0) {
                    if (Math.random() < 0.08) pickups.push({x: e.x, y: e.y, t: Math.random() > 0.5 ? 'm' : 'c'});
                    enemies.splice(i, 1); score += 100; coins += 5; updateUI();
                }
            }
        });

        // Kollisjon med spiller
        if (!player.dead && player.x < e.x + e.w && player.x + player.w > e.x && player.y < e.y + e.h && player.y + player.h > e.y) {
            player.dead = true;
            setTimeout(() => { gameOver = true; if(score > highscore) highscore = Math.floor(score); saveProgress(); }, 800);
        }
        if (e.y > 600) enemies.splice(i, 1);
    });

    // Pickups
    pickups.forEach((p, i) => {
        if (magnetTimer > 0) { p.x += (player.x - p.x) * 0.1; p.y += (player.y - p.y) * 0.1; }
        else { p.y += 2; }
        if (player.x < p.x + 20 && player.x + 30 > p.x && player.y < p.y + 20 && player.y + 30 > p.y) {
            if (p.t === 'm') magnetTimer = 500; else coins += 50;
            pickups.splice(i, 1); updateUI();
        }
    });

    if (!player.dead) score += 0.2;
}

function draw() {
    ctx.clearRect(0, 0, 400, 600);
    
    // Stjerner
    ctx.fillStyle = "white";
    stars.forEach(s => ctx.fillRect(s.x, s.y, 1.5, 1.5));

    // Pickups
    pickups.forEach(p => {
        ctx.fillStyle = p.t === 'm' ? "#f0f" : "#ffd700";
        ctx.beginPath(); ctx.arc(p.x+10, p.y+10, 7, 0, Math.PI*2); ctx.fill();
    });

    // Spiller
    if (!player.dead) {
        ctx.fillStyle = "#0f0";
        ctx.fillRect(player.x, player.y, player.w, player.h);
    }

    // Fiender
    enemies.forEach(e => {
        ctx.fillStyle = e.color;
        ctx.fillRect(e.x, e.y, e.w, e.h);
        if (e.isHeavy) {
            ctx.fillStyle = "red"; ctx.fillRect(e.x, e.y-5, e.w, 3);
            ctx.fillStyle = "lime"; ctx.fillRect(e.x, e.y-5, e.w * (e.hp/e.maxHp), 3);
        }
    });

    // Boss
    if (boss) {
        ctx.fillStyle = "#ff00ff"; ctx.fillRect(boss.x, boss.y, boss.w, boss.h);
        ctx.fillStyle = "red"; ctx.fillRect(boss.x, boss.y-10, boss.w, 5);
        ctx.fillStyle = "lime"; ctx.fillRect(boss.x, boss.y-10, boss.w * (boss.hp/boss.maxHp), 5);
    }

    // Bullets
    ctx.fillStyle = "yellow";
    bullets.forEach(b => ctx.fillRect(b.x, b.y, 4, 10));

    // UI på canvas
    ctx.fillStyle = "white"; ctx.font = "16px Arial";
    ctx.fillText(`Score: ${Math.floor(score)}`, 10, 25);
    ctx.fillStyle = "#4af"; ctx.fillText(`Wave: ${currentWave}`, 10, 45);

    if (waveTimer > 0) {
        ctx.fillStyle = "yellow"; ctx.font = "30px Arial";
        ctx.fillText(`WAVE ${currentWave}`, 130, 300);
    }

    if (gameOver) {
        ctx.fillStyle = "red"; ctx.font = "40px Arial";
        ctx.fillText("GAME OVER", 85, 300);
    }
}

// Input
window.addEventListener("keydown", e => { 
    keys[e.key.toLowerCase()] = true; 
    if(e.key === '1') selectWeapon('pistol');
    if(e.key === '2') selectWeapon('smg');
    if(e.key === '3') selectWeapon('shotgun');
    if(e.key === '4') selectWeapon('ar');
});
window.addEventListener("keyup", e => keys[e.key.toLowerCase()] = false);

// Mus-styring
canvas.addEventListener("mousemove", e => {
    if (gameOver || player.dead) return;
    const rect = canvas.getBoundingClientRect();
    player.x = (e.clientX - rect.left) * (400 / rect.width) - 15;
});

function toggleUI() {
    let content = document.getElementById("uiContent");
    content.classList.toggle("hidden");
    document.getElementById("toggleUIBtn").innerText = content.classList.contains("hidden") ? "Vis Shop" : "Skjul Shop";
}

function restartGame() { init(); }
function resetGameData() { if(confirm("Slette alt?")) { localStorage.clear(); location.reload(); } }

// Start
init();
setInterval(spawnEnemy, 700);
function loop() { update(); draw(); requestAnimationFrame(loop); }
loop();
</script>
</body>
</html>
