<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space Station - Balanced Edition</title>
<style>
    body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; }
    canvas { background:#05080f; border:2px solid #4af; max-width: 100vw; max-height: 100vh; }
    .ui { position:absolute; top:10px; right:10px; display:flex; flex-direction:column; gap:5px; z-index:10; width: 170px; background: rgba(0,0,0,0.7); padding: 10px; border-radius: 8px; border: 1px solid #4af; }
    button { padding:8px; font-size:12px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; border-radius: 4px; width: 100%; }
    button:active { background: #4af; }
    #shop { margin-top: 10px; border-top: 1px solid #4af; padding-top: 10px; display: flex; flex-direction: column; gap: 5px; }
    .hidden { display: none !important; }
    #stats { font-size: 14px; margin-bottom: 5px; }
    .reset-btn { border-color: #f44; color: #f44; margin-top: 10px; font-size: 10px; }
    #bossFightBtn { background: darkred; color: white; font-weight: bold; animation: pulse 1s infinite; display: none; }
    @keyframes pulse { 0% { opacity: 0.7; } 50% { opacity: 1; } 100% { opacity: 0.7; } }
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
        <button id="bossFightBtn" onclick="startMegaBoss()">BOSS FIGHT!</button>
        <button onclick="togglePause()">Pause</button>
        <button onclick="restartGame()">Restart</button>
        <button id="unlockBtn" onclick="unlockGun()">Unlock Gun</button>
        <button id="upgradeBtn" onclick="upgradeWeapon()">Upgrade Gun</button>
        <button id="rebirthBtn" onclick="rebirth()" style="display:none; background: gold; color: black; font-weight: bold;">REBIRTH (+30 Gems)</button>
        
        <div id="shop">
            <strong style="font-size: 10px; color: #4af;">SHOP (1 RUNDE)</strong>
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

// Grunnleggende farts-multiplikator (fast verdi)
const BASE_SPEED = 1.3;

let player, enemies, bullets, stars, particles, floatingTexts;
let score = 0, gameOver = false, paused = false;
let keys = {};
let uiVisible = true;
let gemMilestone = 10000;
let enemyExtraSpawn = 0;
let megaBoss = null;

if (localStorage.getItem("hasPlayedBefore") === null) {
    localStorage.setItem("coins", 100);
    localStorage.setItem("gems", 10);
    localStorage.setItem("hasPlayedBefore", "true");
}

let coins = Number(localStorage.getItem("coins"));
let gems = Number(localStorage.getItem("gems"));
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
    
    const bossBtn = document.getElementById("bossFightBtn");
    if (score >= 50000 && !megaBoss) {
        bossBtn.style.display = "block";
    } else {
        bossBtn.style.display = "none";
    }

    const unlockBtn = document.getElementById("unlockBtn");
    if (hasGun) {
        unlockBtn.style.display = "none";
    } else {
        unlockBtn.style.display = "block";
        unlockBtn.innerText = score < 1000 ? "Lås opp (Trenger 1000 score)" : "Unlock Gun (100c)";
        unlockBtn.style.opacity = score < 1000 ? "0.6" : "1";
    }
    
    const upgradeCost = 500 * upgradeLevel + 100;
    const upgradeBtn = document.getElementById("upgradeBtn");
    if (upgradeLevel >= 5) {
        upgradeBtn.innerText = "Max Upgrade";
    } else {
        upgradeBtn.innerText = `Upgrade Gun (${upgradeCost}c)`;
    }
    
    document.getElementById("rebirthBtn").style.display = (upgradeLevel >= 5) ? "block" : "none";
}

function saveProgress() {
    localStorage.setItem("coins", coins);
    localStorage.setItem("gems", gems);
    localStorage.setItem("upgradeLevel", upgradeLevel);
    localStorage.setItem("hasGun", hasGun);
    localStorage.setItem("highscore", highscore);
}

function resetGameData() {
    if(confirm("Vil du slette ALT? Dette kan ikke angres.")) {
        localStorage.clear();
        location.reload();
    }
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

function createExplosion(x, y, color) {
    for (let i = 0; i < 20; i++) {
        particles.push({
            x: x, y: y,
            vx: (Math.random() - 0.5) * 8,
            vy: (Math.random() - 0.5) * 8,
            life: 1.0, color: color
        });
    }
}

function init() {
    boosters = { armor: false, doubleDamage: false, slowEnemies: false };
    player = { x: 180, y: 540, width: 35, height: 35, speed: 7 * BASE_SPEED, armorUsed: false };
    enemies = []; bullets = []; particles = []; floatingTexts = [];
    stars = Array.from({length:50}, () => ({x: Math.random()*400, y: Math.random()*600, s: (1+Math.random()*2) * BASE_SPEED}));
    score = 0; gemMilestone = 10000; enemyExtraSpawn = 0;
    megaBoss = null;
    gameOver = false; paused = false;
    updateUI();
}

function startMegaBoss() {
    megaBoss = {
        x: 150, y: -100, w: 100, h: 80, 
        hp: 150, maxHp: 150, 
        speedX: 1.5, speedY: 0.3,
        phase1Wave: false, phase2Wave: false
    };
    updateUI();
}

function togglePause() { paused = !paused; }
function restartGame() { init(); }

function unlockGun() {
    if (score < 1000) return alert("Nå 1000 poeng i runden først!");
    if (coins >= 100) { coins -= 100; hasGun = true; saveProgress(); updateUI(); }
}

function upgradeWeapon() {
    let cost = 500 * upgradeLevel + 100;
    if (hasGun && upgradeLevel < 5 && coins >= cost) {
        coins -= cost; upgradeLevel++; saveProgress(); updateUI();
    }
}

function rebirth() {
    if (upgradeLevel >= 5) {
        gems += 30; upgradeLevel = 0; hasGun = false; coins = 100;
        saveProgress(); updateUI(); init();
        alert("Rebirth! +30 Gems.");
    }
}

function spawnEnemy(forceWave = false) {
    if (paused || gameOver) return;
    if (megaBoss && !forceWave) return;

    const count = forceWave ? 10 : (Math.floor(Math.random() * 3) + 1 + enemyExtraSpawn);
    for(let i = 0; i < count; i++) {
        enemies.push({x: Math.random()*370, y: -40, w: 30, h: 30, speedY: (2.5 + score/4000) * BASE_SPEED, speedX: 0, hp: 1, maxHp: 1, color: '#f44', coins: 10, isBoss: false});
    }
}

function update() {
    particles.forEach((p, pi) => {
        p.x += p.vx; p.y += p.vy; p.life -= 0.02;
        if (p.life <= 0) particles.splice(pi, 1);
    });

    floatingTexts.forEach((t, ti) => {
        t.y -= 1; t.life -= 0.02;
        if (t.life <= 0) floatingTexts.splice(ti, 1);
    });

    if (gameOver || paused) return;

    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });

    if ((keys['a'] || keys['arrowleft']) && player.x > 0) player.x -= player.speed;
    if ((keys['d'] || keys['arrowright']) && player.x < 400 - player.width) player.x += player.speed;

    if (hasGun && shootCooldown <= 0) {
        bullets.push({x: player.x + player.width/2 - 3, y: player.y, w: 6, h: 12, speed: 12 * BASE_SPEED});
        shootCooldown = cooldownSettings[upgradeLevel];
    }
    if (shootCooldown > 0) shootCooldown--;

    bullets.forEach((b, bi) => {
        b.y -= b.speed;
        if (b.y < -20) bullets.splice(bi, 1);
    });

    if (megaBoss) {
        megaBoss.y += megaBoss.speedY;
        megaBoss.x += megaBoss.speedX;
        if (megaBoss.x <= 0 || megaBoss.x + megaBoss.w >= 400) megaBoss.speedX *= -1;

        if (megaBoss.hp <= megaBoss.maxHp * 0.5 && !megaBoss.phase1Wave) {
            spawnEnemy(true);
            megaBoss.phase1Wave = true;
        }
        if (megaBoss.hp <= megaBoss.maxHp * 0.1 && !megaBoss.phase2Wave) {
            spawnEnemy(true);
            megaBoss.phase2Wave = true;
        }

        if (player.x < megaBoss.x + megaBoss.w && player.x + player.width > megaBoss.x && player.y < megaBoss.y + megaBoss.h && player.y + player.height > megaBoss.y) {
            if (boosters.armor && !player.armorUsed) {
                player.armorUsed = true;
                megaBoss.y -= 50; // Skyv bossen litt opp ved treff
                createExplosion(player.x + player.width/2, player.y, '#4af');
            } else {
                gameOver = true;
                createExplosion(player.x + player.width/2, player.y + player.height/2, '#0f0');
            }
        }

        bullets.forEach((b, bi) => {
            if (b.x < megaBoss.x + megaBoss.w && b.x + b.w > megaBoss.x && b.y < megaBoss.y + megaBoss.h && b.y + b.h > megaBoss.y) {
                megaBoss.hp -= (boosters.doubleDamage ? 2 : 1);
                bullets.splice(bi, 1);
                if (megaBoss.hp <= 0) {
                    score += 10000;
                    coins += 500;
                    createExplosion(megaBoss.x + megaBoss.w/2, megaBoss.y + megaBoss.h/2, "yellow");
                    megaBoss = null;
                    updateUI();
                }
            }
        });
    }

    let speedMult = boosters.slowEnemies ? 0.5 : 1;
    enemies.forEach((e, ei) => {
        e.y += e.speedY * speedMult;
        e.x += e.speedX * speedMult;
        
        if (player.x < e.x + e.w && player.x + player.width > e.x && player.y < e.y + e.h && player.y + player.height > e.y) {
            if (boosters.armor && !player.armorUsed) {
                player.armorUsed = true;
                enemies.splice(ei, 1);
                createExplosion(e.x + e.w/2, e.y + e.h/2, '#4af');
            } else {
                gameOver = true;
                createExplosion(player.x + player.width/2, player.y + player.height/2, '#0f0');
            }
        }

        bullets.forEach((b, bi) => {
            if (b.x < e.x + e.w && b.x + b.w > e.x && b.y < e.y + e.h && b.y + b.h > e.y) {
                if (Math.random() < 0.017) {
                    gems += 5;
                    floatingTexts.push({x: e.x, y: e.y, text: "CRIT! +5G", color: "#a4f", life: 1.0});
                    createExplosion(e.x + e.w/2, e.y + e.h/2, "#a4f");
                    updateUI(); saveProgress();
                }
                e.hp -= (boosters.doubleDamage ? 2 : 1);
                bullets.splice(bi, 1);
                if (e.hp <= 0) {
                    coins += e.coins;
                    score += e.isBoss ? 500 : 100;
                    createExplosion(e.x + e.w/2, e.y + e.h/2, e.color);
                    enemies.splice(ei, 1);
                    updateUI();
                }
            }
        });
        if (e.y > 600 || e.x > 500 || e.x < -100) enemies.splice(ei, 1);
    });

    score += 0.3 * BASE_SPEED;
    if (score >= gemMilestone) { 
        gems += 2; enemyExtraSpawn += 1; gemMilestone += 10000; 
        updateUI(); saveProgress(); 
    }
}

function draw() {
    ctx.clearRect(0,0,400,600);
    stars.forEach(s => { ctx.fillStyle='white'; ctx.fillRect(s.x,s.y,2,2); });

    particles.forEach(p => {
        ctx.globalAlpha = p.life; ctx.fillStyle = p.color; ctx.fillRect(p.x, p.y, 4, 4);
    });

    floatingTexts.forEach(t => {
        ctx.globalAlpha = t.life; ctx.fillStyle = t.color;
        ctx.font = "bold 16px Arial"; ctx.fillText(t.text, t.x, t.y);
    });
    ctx.globalAlpha = 1.0;

    if (megaBoss) {
        ctx.fillStyle = "yellow";
        ctx.fillRect(megaBoss.x, megaBoss.y, megaBoss.w, megaBoss.h);
        ctx.fillStyle = "grey";
        ctx.fillRect(50, 20, 300, 15);
        ctx.fillStyle = "yellow";
        ctx.fillRect(50, 20, 300 * (megaBoss.hp / megaBoss.maxHp), 15);
        ctx.strokeStyle = "white";
        ctx.strokeRect(50, 20, 300, 15);
    }

    if (!gameOver) {
        ctx.fillStyle = (boosters.armor && !player.armorUsed) ? '#4af' : '#0f0';
        ctx.fillRect(player.x, player.y, player.width, player.height);
    }
    
    bullets.forEach(b => { ctx.fillStyle = boosters.doubleDamage ? 'orange' : 'yellow'; ctx.fillRect(b.x, b.y, b.w, b.h); });
    enemies.forEach(e => {
        ctx.fillStyle = e.color; ctx.fillRect(e.x, e.y, e.w, e.h);
        if(e.isBoss) {
            ctx.fillStyle = 'red'; ctx.fillRect(e.x, e.y - 10, e.w, 5);
            ctx.fillStyle = '#0f0'; ctx.fillRect(e.x, e.y - 10, e.w * (e.hp / e.maxHp), 5);
        }
    });

    ctx.fillStyle = 'white'; ctx.font = 'bold 18px Arial';
    ctx.fillText(`Score: ${Math.floor(score)}`, 15, 30);
    ctx.font = '14px Arial'; ctx.fillText(`Highscore: ${highscore}`, 15, 50);
    
    if (paused) { ctx.font='30px Arial'; ctx.fillText('PAUSE', 150, 300); }
    if (gameOver) { ctx.fillStyle='red'; ctx.font='30px Arial'; ctx.fillText('GAME OVER', 110, 300); }
}

window.addEventListener("keydown", e => keys[e.key.toLowerCase()] = true);
window.addEventListener("keyup", e => keys[e.key.toLowerCase()] = false);
canvas.addEventListener("touchmove", e => {
    e.preventDefault();
    let rect = canvas.getBoundingClientRect();
    let x = e.touches[0].clientX - rect.left;
    player.x = x - player.width / 2;
    if(player.x < 0) player.x = 0; if(player.x > 365) player.x = 365;
}, {passive: false});

init();
setInterval(spawnEnemy, 600);
(function loop(){ update(); draw(); requestAnimationFrame(loop); })();
</script>
</body>
</html>
