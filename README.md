<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space Station - HP Bar & Booster Fix</title>
<style>
    body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; touch-action: none; }
    canvas { background:#05080f; border:2px solid #4af; max-width: 100vw; max-height: 100vh; cursor: crosshair; }
    .ui { position:absolute; top:10px; right:10px; display:flex; flex-direction:column; gap:5px; z-index:10; width: 195px; background: rgba(0,0,0,0.85); padding: 10px; border-radius: 8px; border: 1px solid #4af; max-height: 90vh; overflow-y: auto; }
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
            <span class="section-title">Våpen (1-4)</span>
            <div class="wpn-group">
                <button id="pistolSelect" onclick="selectWeapon('pistol')">Pistol</button>
                <button id="unlockBtn" onclick="buyWeapon('pistol', 0)">Lås opp (1k Score)</button>
                <button id="upgradePistolBtn" onclick="upgradeWeapon('pistol')">Oppgrader</button>
            </div>
            <div class="wpn-group">
                <button id="smgSelect" onclick="selectWeapon('smg')">SMG</button>
                <button id="buySMGBtn" onclick="buyWeapon('smg', 1000)">Kjøp (1000c)</button>
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
let shake = 0;

let coins = Number(localStorage.getItem("coins")) || 0;
let gems = Number(localStorage.getItem("gems")) || 0;
let highscore = Number(localStorage.getItem("highscore")) || 0;
let activeWeapon = localStorage.getItem("activeWeapon") || "none";
let weaponsOwned = JSON.parse(localStorage.getItem("weaponsOwned")) || { pistol: false, smg: false, shotgun: false, ar: false };
let weaponLevels = JSON.parse(localStorage.getItem("weaponLevels")) || { pistol: 0, smg: 0, shotgun: 0, ar: 0 };
let boosters = { armor: false, doubleDamage: false, slowEnemies: false };

const weaponConfigs = {
    pistol: { cooldown: [25, 18, 12], maxLvl: 2, type: "single", dmg: 1 },
    smg: { cooldown: [8, 5], maxLvl: 1, type: "single", dmg: 0.3 },
    shotgun: { cooldown: [45, 30], maxLvl: 1, type: "triple", dmg: 1 },
    ar: { cooldown: [10, 6], maxLvl: 1, type: "fast", dmg: 1 }
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
    
    document.getElementById("unlockBtn").style.display = weaponsOwned.pistol ? "none" : "block";
    document.getElementById("unlockBtn").disabled = (highscore < 1000 && score < 1000);
    
    ["pistol", "smg", "shotgun", "ar"].forEach(w => {
        const btn = document.getElementById(w + "Select");
        if(btn) {
            btn.style.display = weaponsOwned[w] ? "block" : "none";
            btn.className = activeWeapon === w ? "active-wpn" : "";
            btn.innerText = activeWeapon === w ? w.toUpperCase() + " (VALGT)" : "Bruk " + w;
        }
        const upgBtn = document.getElementById("upgrade" + w.charAt(0).toUpperCase() + w.slice(1) + "Btn");
        if(upgBtn) {
            upgBtn.style.display = weaponsOwned[w] ? "block" : "none";
        }
    });
}

function buyWeapon(type, cost) {
    if (coins >= cost) { coins -= cost; weaponsOwned[type] = true; activeWeapon = type; saveProgress(); updateUI(); }
}

function upgradeWeapon(type) {
    let cost = 500; // Forenklet kostnad for oppgradering
    if (weaponsOwned[type] && weaponLevels[type] < weaponConfigs[type].maxLvl && coins >= cost) {
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
    player = { x: 180, y: 540, width: 35, height: 35, speed: 7 * BASE_SPEED, armorUsed: false, alive: true };
    enemies = []; bullets = []; particles = []; floatingTexts = [];
    stars = Array.from({length:50}, () => ({x: Math.random()*400, y: Math.random()*600, s: (1+Math.random()*2) * BASE_SPEED}));
    boosters = { armor: false, doubleDamage: false, slowEnemies: false };
    score = 0; gemMilestone = 10000; shake = 0;
    gameOver = false; paused = false; shootCooldown = 0;
    updateUI();
}

function createExplosion(x, y, color, count = 20) {
    for (let i = 0; i < count; i++) {
        particles.push({x, y, vx: (Math.random()-0.5)*15, vy: (Math.random()-0.5)*15, life: 1, color});
    }
}

function spawnEnemy() {
    if (paused || gameOver) return;
    let extraChance = Math.min(0.2, score / 100000);
    let r = Math.random();
    if (r < 0.15 && score > 2000) {
        let hp = 5 + Math.floor(score / 10000);
        enemies.push({x: Math.random()*350, y: -50, w: 45, h: 45, speedY: 1.2 * BASE_SPEED, color: '#800', coins: 50, hp: hp, maxHp: hp, isHeavy: true, type: 'heavy'});
    } else if (r < 0.30 + extraChance && score > 1000) {
        enemies.push({x: Math.random()*300 + 50, y: -40, w: 25, h: 25, speedY: 2 * BASE_SPEED, color: '#a0f', coins: 20, hp: 1, isHeavy: false, type: 'sinus', centerX: 0, angle: 0});
    } else {
        enemies.push({x: Math.random()*370, y: -40, w: 30, h: 30, speedY: (2.5 + score/5000) * BASE_SPEED, color: '#f44', coins: 10, hp: 1, isHeavy: false, type: 'normal'});
    }
}

function fire() {
    if (!player.alive || activeWeapon === "none") return;
    const config = weaponConfigs[activeWeapon];
    const lvl = weaponLevels[activeWeapon];
    const damage = config.dmg * (boosters.doubleDamage ? 2 : 1);
    if (config.type === "triple") {
        bullets.push({x: player.x + 15, y: player.y, vx: -2.5, vy: -11, dmg: damage});
        bullets.push({x: player.x + 15, y: player.y, vx: 0, vy: -12, dmg: damage});
        bullets.push({x: player.x + 15, y: player.y, vx: 2.5, vy: -11, dmg: damage});
    } else {
        bullets.push({x: player.x + 15, y: player.y, vx: 0, vy: -12, dmg: damage});
    }
    shootCooldown = config.cooldown[lvl];
}

function update() {
    if (shake > 0) shake *= 0.9;
    particles.forEach((p, i) => { p.x += p.vx; p.y += p.vy; p.life -= 0.02; if(p.life <= 0) particles.splice(i,1); });
    floatingTexts.forEach((t, i) => { t.y -= 1; t.life -= 0.02; if(t.life <= 0) floatingTexts.splice(i,1); });

    if (gameOver || paused) return;

    if (player.alive && activeWeapon !== "none" && shootCooldown <= 0) fire();
    if (shootCooldown > 0) shootCooldown--;
    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });

    if (player.alive) {
        if ((keys['a'] || keys['arrowleft']) && player.x > 0) player.x -= player.speed;
        if ((keys['d'] || keys['arrowright']) && player.x < 400 - player.width) player.x += player.speed;
    }

    let enemySpeedMult = boosters.slowEnemies ? 0.5 : 1.0;

    enemies.forEach((e, ei) => {
        if (e.type === 'sinus') {
            if (!e.centerX) e.centerX = e.x; e.angle += 0.05; e.x = e.centerX + Math.sin(e.angle) * 50;
            e.y += e.speedY * enemySpeedMult;
        } else { e.y += e.speedY * enemySpeedMult; }

        if (player.alive && player.x < e.x + e.w && player.x + player.width > e.x && player.y < e.y + e.h && player.y + player.height > e.y) {
            if (boosters.armor && !player.armorUsed) { 
                player.armorUsed = true; enemies.splice(ei, 1); createExplosion(player.x+17, player.y, "#4af"); 
            } else { 
                player.alive = false; shake = 20;
                createExplosion(player.x+17, player.y+17, "#0f0", 60); createExplosion(player.x+17, player.y+17, "orange", 40);
                setTimeout(() => { gameOver = true; if(score > highscore) { highscore = Math.floor(score); saveProgress(); } }, 1000);
            }
        }

        bullets.forEach((b, bi) => {
            if (b.x < e.x + e.w && b.x + 6 > e.x && b.y < e.y + e.h && b.y + 12 > e.y) {
                e.hp -= b.dmg; bullets.splice(bi, 1);
                if (e.hp <= 0) {
                    if (Math.random() < 0.02) { gems += 5; floatingTexts.push({x: e.x, y: e.y, text: "GEMS! +5", color: "#a4f", life: 1}); }
                    coins += (e.coins || 10); score += (e.isHeavy ? 500 : 100); createExplosion(e.x+e.w/2, e.y+e.h/2, e.color);
                    enemies.splice(ei, 1); updateUI();
                }
            }
        });
        if (e.y > 600) enemies.splice(ei, 1);
    });

    bullets.forEach((b, i) => { b.y += b.vy; b.x += b.vx; if(b.y < -20) bullets.splice(i,1); });
    score += 0.3;
    if (score >= gemMilestone) { gems += 2; gemMilestone += 10000; updateUI(); }
}

function draw() {
    ctx.save();
    if (shake > 1) ctx.translate((Math.random()-0.5)*shake, (Math.random()-0.5)*shake);
    ctx.clearRect(0,0,400,600);
    stars.forEach(s => { ctx.fillStyle='white'; ctx.fillRect(s.x,s.y,2,2); });
    particles.forEach(p => { ctx.globalAlpha = p.life; ctx.fillStyle = p.color; ctx.fillRect(p.x, p.y, 4, 4); });
    floatingTexts.forEach(t => { ctx.globalAlpha = t.life; ctx.fillStyle = t.color; ctx.font="bold 14px Arial"; ctx.fillText(t.text, t.x, t.y); });
    ctx.globalAlpha = 1;
    
    if (player.alive) {
        ctx.fillStyle = (boosters.armor && !player.armorUsed) ? '#4af' : '#0f0';
        ctx.fillRect(player.x, player.y, player.width, player.height);
    }
    
    bullets.forEach(b => { ctx.fillStyle = boosters.doubleDamage ? 'orange' : 'yellow'; ctx.fillRect(b.x, b.y, 6, 12); });
    
    enemies.forEach(e => { 
        ctx.fillStyle = e.color; ctx.fillRect(e.x, e.y, e.w, e.h); 
        // HP BARER TILBAKE HER:
        if (e.isHeavy) {
            ctx.fillStyle = "red"; ctx.fillRect(e.x, e.y - 10, e.w, 5);
            ctx.fillStyle = "lime"; ctx.fillRect(e.x, e.y - 10, e.w * (e.hp / e.maxHp), 5);
        }
    });
    
    ctx.restore();
    ctx.fillStyle = 'white'; ctx.font = 'bold 16px Arial';
    ctx.fillText(`Score: ${Math.floor(score)}`, 10, 25);
    if(gameOver) { ctx.fillStyle="red"; ctx.font="30px Arial"; ctx.fillText("GAME OVER", 110, 300); }
}

window.addEventListener("keydown", e => { 
    keys[e.key.toLowerCase()] = true; 
    if (e.key === '1') selectWeapon('pistol'); if (e.key === '2') selectWeapon('smg'); if (e.key === '3') selectWeapon('shotgun'); if (e.key === '4') selectWeapon('ar');
});
window.addEventListener("keyup", e => { keys[e.key.toLowerCase()] = false; });

const handleMove = (e) => {
    if(paused || gameOver || !player.alive) return;
    const rect = canvas.getBoundingClientRect();
    const clientX = e.touches ? e.touches[0].clientX : e.clientX;
    const x = (clientX - rect.left) * (canvas.width / rect.width);
    player.x = x - player.width / 2;
    if (player.x < 0) player.x = 0; if (player.x > 365) player.x = 365;
};
canvas.addEventListener("mousemove", handleMove);
canvas.addEventListener("touchmove", (e) => { e.preventDefault(); handleMove(e); }, { passive: false });

function restartGame() { init(); }
function resetGameData() { if(confirm("Slette alt?")) { localStorage.clear(); location.reload(); } }

init();
setInterval(spawnEnemy, 500);
(function loop(){ update(); draw(); requestAnimationFrame(loop); })();
</script>
</body>
</html>
