<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space Station - Mobile Optimized</title>
<style>
    body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; }
    canvas { background:#05080f; border:2px solid #4af; max-width: 100vw; max-height: 100vh; }
    .ui { position:absolute; top:10px; right:10px; display:flex; flex-direction:column; gap:5px; z-index:2; max-width: 160px; background: rgba(0,0,0,0.5); padding: 5px; border-radius: 5px; }
    button { padding:8px; font-size:12px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; border-radius: 4px; }
    button:active { background: #4af; }
    #shop { margin-top: 5px; border-top: 1px solid #4af; padding-top: 5px; display: flex; flex-direction: column; gap: 3px; }
    .hidden { display: none !important; }
</style>
</head>
<body>

<div class="ui" id="mainUI">
    <button id="toggleUIBtn" onclick="toggleUI()">Skjul UI</button>
    
    <div id="uiContent">
        <button onclick="togglePause()">Pause</button>
        <button onclick="restartGame()">Restart</button>
        <button id="unlockBtn" onclick="unlockGun()">Unlock Gun</button>
        <button onclick="upgradeWeapon()">Upgrade Weapon</button>
        <button id="rebirthBtn" onclick="rebirth()" style="display:none;">Rebirth</button>
        <div id="coins">Coins: 0</div>
        <div id="gems">Gems: 0</div>
        
        <div id="shop">
            <strong style="font-size: 11px;">Shop (Varer 1 runde)</strong>
            <button onclick="buyBooster('armor', 5)">Armor (5 Gems)</button>
            <button onclick="buyBooster('doubleDamage', 10)">2x Dmg (10 Gems)</button>
            <button onclick="buyBooster('slowEnemies', 15)">Slow (15 Gems)</button>
        </div>
    </div>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

// Spill-tilstand
let player, enemies, bullets, explosions, stars;
let score, gameOver=false, paused=false;
let keys={};
let uiVisible = true;

// Lagret progresjon
let coins = Number(localStorage.getItem("coins")) || 0;
let gems = Number(localStorage.getItem("gems")) || 0;
let upgradeLevel = Number(localStorage.getItem("upgradeLevel")) || 0;
let hasGun = localStorage.getItem("hasGun") === "true";

// Boosters (disse nullstilles hver runde i init)
let boosters = { armor: false, doubleDamage: false, slowEnemies: false };

function toggleUI() {
    uiVisible = !uiVisible;
    document.getElementById("uiContent").classList.toggle("hidden", !uiVisible);
    document.getElementById("toggleUIBtn").innerText = uiVisible ? "Skjul UI" : "Vis UI";
}

function buyBooster(type, cost) {
    if (boosters[type]) {
        alert("Allerede aktiv i denne runden!");
        return;
    }
    if (gems >= cost) {
        gems -= cost;
        boosters[type] = true;
        saveProgress();
        updateUI();
        alert(type + " aktivert for denne runden!");
    } else {
        alert("Ikke nok Gems!");
    }
}

function init() {
    // VIKTIG: Her nullstilles boosters for den nye runden
    boosters = { armor: false, doubleDamage: false, slowEnemies: false };
    
    player = { x: 180, y: 540, width: 35, height: 35, speed: 6, armorUsed: false };
    enemies = [];
    bullets = [];
    explosions = [];
    stars = Array.from({length:60}, () => ({x: Math.random()*400, y: Math.random()*600, s: 1+Math.random()*2}));
    score = 0;
    gameOver = false;
    paused = false;
    updateUI();
}

function saveProgress() {
    localStorage.setItem("coins", coins);
    localStorage.setItem("gems", gems);
    localStorage.setItem("upgradeLevel", upgradeLevel);
    localStorage.setItem("hasGun", hasGun);
}

function updateUI() {
    document.getElementById("coins").innerText = `Coins: ${Math.floor(coins)}`;
    document.getElementById("gems").innerText = `Gems: ${gems}`;
    document.getElementById("unlockBtn").style.display = hasGun ? "none" : "block";
    if (upgradeLevel >= 5) document.getElementById("rebirthBtn").style.display = "block";
}

// ... resten av spill-logikken (spawnEnemy, update, draw) som før ...
// Husk å kalle updateUI() når fiender dør for å se coins øke.

function update() {
    if (gameOver || paused) return;

    // Bakgrunn
    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });

    // Bevegelse (Touch/Mus)
    canvas.ontouchmove = (e) => {
        let rect = canvas.getBoundingClientRect();
        let x = e.touches[0].clientX - rect.left;
        player.x = x - player.width / 2;
    };

    // Fiender bevegelse (Slow booster påvirker her)
    let speedMult = boosters.slowEnemies ? 0.5 : 1;
    enemies.forEach(e => {
        e.y += e.speedY * speedMult;
    });

    // Kollisjon med Armor sjekk
    enemies.forEach((e, index) => {
        if (checkCollision(player, e)) {
            if (boosters.armor && !player.armorUsed) {
                player.armorUsed = true;
                enemies.splice(index, 1);
                // Armor er nå brukt opp for denne runden
            } else {
                gameOver = true;
            }
        }
    });

    // Skade med Double Damage sjekk
    bullets.forEach((b, bIdx) => {
        enemies.forEach((e, eIdx) => {
            if (checkCollision(b, e)) {
                e.hp -= (boosters.doubleDamage ? 2 : 1);
                bullets.splice(bIdx, 1);
                if (e.hp <= 0) {
                    coins += e.coins;
                    enemies.splice(eIdx, 1);
                    saveProgress();
                    updateUI();
                }
            }
        });
    });

    score += 0.1;
}

function checkCollision(a, b) {
    return a.x < b.x + b.w && a.x + a.width > b.x && a.y < b.y + b.h && a.y + a.height > b.y;
}

// Start spillet
init();
(function loop(){ update(); draw(); requestAnimationFrame(loop); })();
setInterval(spawnEnemy, 1000);
</script>
</body>
</html>
