<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space Station - Boss Wave Edition</title>
<style>
    body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; touch-action: none; }
    canvas { background:#05080f; border:2px solid #4af; max-width: 100vw; max-height: 100vh; cursor: crosshair; }
    .ui { position:absolute; top:10px; right:10px; display:flex; flex-direction:column; gap:5px; z-index:10; width: 200px; background: rgba(0,0,0,0.85); padding: 10px; border-radius: 8px; border: 1px solid #4af; max-height: 90vh; overflow-y: auto; }
    button { padding:8px; font-size:11px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; border-radius: 4px; width: 100%; margin-bottom: 2px; }
    button.active-wpn { background: #004400; border-color: #0f0; color: #0f0; }
    button:disabled { opacity: 0.3; cursor: not-allowed; }
    .section-title { font-size: 10px; color: #4af; font-weight: bold; text-transform: uppercase; margin-bottom: 5px; display: block; }
    #stats { font-size: 13px; margin-bottom: 5px; line-height: 1.4; }
    .wpn-group { margin-bottom: 10px; padding: 5px; border-radius: 4px; background: rgba(255,255,255,0.05); }
    .hidden { display: none !important; }
</style>
</head>
<body>

<div class="ui" id="mainUI">
    <button id="toggleUIBtn" onclick="toggleUI()">Skjul UI</button>
    <div id="uiContent">
        <div id="stats">
            <div id="coinsDisplay">Coins: 0</div>
            <div id="highscoreDisplayUI">Score Best: 0</div>
            <div id="waveHighscoreDisplayUI" style="color: #4af;">Wave Best: 1</div>
        </div>
        <button onclick="togglePause()">Pause</button>
        <button onclick="restartGame()">Restart</button>
        
        <div id="weaponShop">
            <span class="section-title">Arsenal (1-4)</span>
            <div class="wpn-group">
                <button id="pistolSelect" onclick="selectWeapon('pistol')">Pistol</button>
                <button id="unlockBtn" onclick="buyWeapon('pistol', 0)">Lås opp (1k Score)</button>
                <button id="upgradePistolBtn" onclick="upgradeWeapon('pistol')">Upg</button>
            </div>
            <div class="wpn-group">
                <button id="smgSelect" onclick="selectWeapon('smg')">SMG</button>
                <button id="buySMGBtn" onclick="buyWeapon('smg', 500)">Kjøp SMG (500c)</button>
                <button id="upgradeSMGBtn" onclick="upgradeWeapon('smg')">Upg</button>
            </div>
            <div class="wpn-group">
                <button id="shotgunSelect" onclick="selectWeapon('shotgun')">Shotgun</button>
                <button id="buyShotgunBtn" onclick="buyWeapon('shotgun', 750)">Kjøp (750c)</button>
                <button id="upgradeShotgunBtn" onclick="upgradeWeapon('shotgun')">Upg</button>
            </div>
            <div class="wpn-group">
                <button id="arSelect" onclick="selectWeapon('ar')">AR</button>
                <button id="buyARBtn" onclick="buyWeapon('ar', 1200)">Kjøp (1200c)</button>
                <button id="upgradeARBtn" onclick="upgradeWeapon('ar')">Upg</button>
            </div>
        </div>
        <button onclick="resetGameData()" style="color:red; font-size:9px;">RESET DATA</button>
    </div>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

let player, enemies, bullets, stars, pickups, boss;
let score = 0, gameOver = false, paused = false, shootCooldown = 0;
let keys = {};
let uiVisible = true;
let currentWave = 1, enemiesInWave = 0, waveTimer = 100;
let magnetTimer = 0;

let coins = Number(localStorage.getItem("coins")) || 0;
let highscore = Number(localStorage.getItem("highscore")) || 0;
let waveHighscore = Number(localStorage.getItem("waveHighscore")) || 1;
let activeWeapon = localStorage.getItem("activeWeapon") || "none";
let weaponsOwned = JSON.parse(localStorage.getItem("weaponsOwned")) || { pistol: false, smg: false, shotgun: false, ar: false };
let weaponLevels = JSON.parse(localStorage.getItem("weaponLevels")) || { pistol: 0, smg: 0, shotgun: 0, ar: 0 };

const weaponConfigs = {
    pistol: { cooldown: [25, 15], dmg: 1, type: "single" },
    smg: { cooldown: [7, 4], dmg: 0.5, type: "single" },
    shotgun: { cooldown: [45, 30], dmg: 1, type: "triple" },
    ar: { cooldown: [12, 7], dmg: 1.3, type: "single" }
};

function toggleUI() {
    uiVisible = !uiVisible;
    document.getElementById("uiContent").classList.toggle("hidden", !uiVisible);
}

function selectWeapon(type) {
    if (weaponsOwned[type]) { activeWeapon = type; updateUI(); saveProgress(); }
}

function updateUI() {
    document.getElementById("coinsDisplay").innerText = `Coins: ${Math.floor(coins)}`;
    document.getElementById("highscoreDisplayUI").innerText = `Score Best: ${Math.floor(highscore)}`;
    document.getElementById("waveHighscoreDisplayUI").innerText = `Wave Best: ${waveHighscore}`;
    
    ['pistol', 'smg', 'shotgun', 'ar'].forEach(t => {
        const buyBtn = (t === 'pistol') ? document.getElementById('unlockBtn') : document.getElementById(`buy${t.toUpperCase()}Btn`);
        const upgBtn = document.getElementById(`upgrade${t.toUpperCase()}Btn`);
        const selBtn = document.getElementById(`${t}Select`);

        if(buyBtn) buyBtn.style.display = weaponsOwned[t] ? "none" : "block";
        upgBtn.style.display = weaponsOwned[t] ? "block" : "none";
        selBtn.style.display = weaponsOwned[t] ? "block" : "none";
        selBtn.className = activeWeapon === t ? "active-wpn" : "";
    });
}

function buyWeapon(type, cost) {
    if (coins >= cost) { coins -= cost; weaponsOwned[type] = true; activeWeapon = type; updateUI(); saveProgress(); }
}

function upgradeWeapon(type) {
    if (coins >= 500 && weaponLevels[type] < 1) { coins -= 500; weaponLevels[type]++; updateUI(); saveProgress(); }
}

function saveProgress() {
    localStorage.setItem("coins", coins);
    localStorage.setItem("weaponsOwned", JSON.stringify(weaponsOwned));
    localStorage.setItem("weaponLevels", JSON.stringify(weaponLevels));
    localStorage.setItem("activeWeapon", activeWeapon);
    localStorage.setItem("highscore", highscore);
    localStorage.setItem("waveHighscore", waveHighscore);
}

function init() {
    player = { x: 180, y: 540, w: 35, h: 35, speed: 6, dead: false };
    enemies = []; bullets = []; pickups = []; boss = null;
    stars = Array.from({length:40}, () => ({x: Math.random()*400, y: Math.random()*600, s: 1 + Math.random()*2}));
    score = 0; currentWave = 1; enemiesInWave = 0; waveTimer = 100;
    updateUI();
}

function spawnEnemy() {
    if (paused || gameOver || waveTimer > 0 || boss) return;

    // Sjekk om vi skal spawne boss (Wave 20 ELLER score 100k)
    if (currentWave === 20 || score >= 100000) {
        if (!boss) {
            boss = { x: 150, y: -100, w: 100, h: 80, hp: 500, maxHp: 500, speedX: 2, speedY: 0.5 };
        }
        return;
    }

    let x = Math.random() * 360;
    let roll = Math.random();
    
    if (roll < 0.2) { 
        enemies.push({x, y: -40, w: 30, h: 30, speedY: 2, color: '#44f', hp: 1, isSinus: true, offset: Math.random()*100});
    } else if (roll < 0.3 && score > 3000) { 
        enemies.push({x, y: -50, w: 45, h: 45, speedY: 1, color: '#800', hp: 5, maxHp: 5, isHeavy: true});
    } else { 
        enemies.push({x, y: -40, w: 30, h: 30, speedY: 2 + (currentWave*0.2), color: '#f44', hp: 1});
    }
    
    enemiesInWave++;
    if (enemiesInWave > 15) { 
        currentWave++; 
        enemiesInWave = 0; 
        waveTimer = 150; 
        if (currentWave > waveHighscore) waveHighscore = currentWave;
    }
}

function fire() {
    if (activeWeapon === "none" || player.dead) return;
    const conf = weaponConfigs[activeWeapon];
    const dmg = conf.dmg;
    if (conf.type === "triple") {
        [-2, 0, 2].forEach(vx => bullets.push({x: player.x+15, y: player.y, vx, vy: -10, dmg}));
    } else {
        bullets.push({x: player.x+15, y: player.y, vx: 0, vy: -10, dmg});
    }
    shootCooldown = conf.cooldown[weaponLevels[activeWeapon]];
}

function update() {
    if (paused || gameOver) return;
    if (waveTimer > 0) waveTimer--;
    if (magnetTimer > 0) magnetTimer--;

    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });
    
    if (!player.dead) {
        if (keys['a'] || keys['arrowleft']) player.x -= player.speed;
        if (keys['d'] || keys['arrowright']) player.x += player.speed;
        player.x = Math.max(0, Math.min(365, player.x));
        if (shootCooldown <= 0) fire();
        shootCooldown--;
    }

    // Boss Logikk
    if (boss) {
        boss.y += (boss.y < 50) ? boss.speedY : 0;
        boss.x += boss.speedX;
        if (boss.x <= 0 || boss.x + boss.w >= 400) boss.speedX *= -1;

        bullets.forEach((b, bi) => {
            if (b.x < boss.x + boss.w && b.x + 5 > boss.x && b.y < boss.y + boss.h && b.y + 10 > boss.y) {
                boss.hp -= b.dmg; bullets.splice(bi, 1);
                if (boss.hp <= 0) {
                    coins += 1000; score += 20000; boss = null; currentWave++; updateUI();
                }
            }
        });
    }

    bullets.forEach((b, i) => {
        b.x += b.vx; b.y += b.vy;
        if (b.y < -20) bullets.splice(i, 1);
    });

    pickups.forEach((p, i) => {
        if (magnetTimer > 0) {
            p.x += (player.x - p.x) * 0.1; p.y += (player.y - p.y) * 0.1;
        } else { p.y += 2; }
        if (player.x < p.x+20 && player.x+35 > p.x && player.y < p.y+20 && player.y+35 > p.y) {
            if (p.t === 'm') magnetTimer = 400; else coins += 100;
            pickups.splice(i, 1); updateUI();
        }
    });

    enemies.forEach((e, i) => {
        e.y += e.speedY;
        if (e.isSinus) e.x += Math.sin(e.y/30) * 3;
        
        if (!player.dead && player.x < e.x+e.w && player.x+35 > e.x && player.y < e.y+e.h && player.y+35 > e.y) {
            player.dead = true;
            setTimeout(() => { gameOver = true; if(score > highscore) highscore = Math.floor(score); saveProgress(); }, 1000);
        }

        bullets.forEach((b, bi) => {
            if (b.x < e.x+e.w && b.x+5 > e.x && b.y < e.y+e.h && b.y+10 > e.y) {
                e.hp -= b.dmg; bullets.splice(bi, 1);
                if (e.hp <= 0) {
                    if (Math.random() < 0.05) pickups.push({x: e.x, y: e.y, t: Math.random()>0.5?'m':'g'});
                    coins += 10; score += 100; enemies.splice(i, 1); updateUI();
                }
            }
        });
        if (e.y > 600) enemies.splice(i, 1);
    });
    if (!player.dead) score += 0.2;
}

function draw() {
    ctx.clearRect(0, 0, 400, 600);
    stars.forEach(s => { ctx.fillStyle='white'; ctx.fillRect(s.x, s.y, 2, 2); });
    
    pickups.forEach(p => { ctx.fillStyle = p.t==='m'?'#f0f':'gold'; ctx.beginPath(); ctx.arc(p.x+10, p.y+10, 8, 0, 7); ctx.fill(); });

    if (!player.dead) {
        ctx.fillStyle = '#0f0'; ctx.fillRect(player.x, player.y, player.w, player.h);
    }

    if (boss) {
        ctx.fillStyle = '#a0f'; ctx.fillRect(boss.x, boss.y, boss.w, boss.h);
        ctx.fillStyle = 'red'; ctx.fillRect(boss.x, boss.y - 10, boss.w, 5);
        ctx.fillStyle = 'lime'; ctx.fillRect(boss.x, boss.y - 10, boss.w * (boss.hp/boss.maxHp), 5);
    }

    bullets.forEach(b => { ctx.fillStyle='yellow'; ctx.fillRect(b.x, b.y, 4, 10); });
    enemies.forEach(e => {
        ctx.fillStyle = e.color; ctx.fillRect(e.x, e.y, e.w, e.h);
        if (e.isHeavy) {
            ctx.fillStyle='red'; ctx.fillRect(e.x, e.y-5, e.w, 3);
            ctx.fillStyle='lime'; ctx.fillRect(e.x, e.y-5, e.w*(e.hp/e.maxHp), 3);
        }
    });

    ctx.fillStyle='white'; ctx.font='16px Arial';
    ctx.fillText(`Score: ${Math.floor(score)}`, 10, 25);
    ctx.fillText(`Wave: ${currentWave}`, 10, 45);
    if (waveTimer > 0) ctx.fillText(`WAVE ${currentWave} STARTS`, 130, 300);
    if (boss) ctx.fillText(`BOSS FIGHT!`, 150, 30);
    if (gameOver) { ctx.fillStyle='red'; ctx.font='30px Arial'; ctx.fillText("GAME OVER", 110, 300); }
}

window.addEventListener("keydown", e => { keys[e.key.toLowerCase()] = true; if(e.key==='1')selectWeapon('pistol'); if(e.key==='2')selectWeapon('smg'); if(e.key==='3')selectWeapon('shotgun'); if(e.key==='4')selectWeapon('ar'); });
window.addEventListener("keyup", e => keys[e.key.toLowerCase()] = false);

canvas.addEventListener("mousemove", e => {
    if (gameOver || player.dead) return;
    const rect = canvas.getBoundingClientRect();
    player.x = (e.clientX - rect.left) * (400/rect.width) - 17;
});

function togglePause() { paused = !paused; }
function restartGame() { init(); }
function resetGameData() { localStorage.clear(); location.reload(); }

init();
setInterval(spawnEnemy, 600);
function loop() { update(); draw(); requestAnimationFrame(loop); }
loop();
</script>
</body>
</html>
