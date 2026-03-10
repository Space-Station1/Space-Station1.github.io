<!DOCTYPE html>
<html lang="no">
<head>
<meta charset="UTF-8">
<title>Space Station - Creeper Edition</title>
<style>
    body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; touch-action: none; }
    canvas { background:#05080f; border:2px solid #4af; max-width: 100vw; max-height: 100vh; cursor: crosshair; }
    .ui { position:absolute; top:10px; right:10px; display:flex; flex-direction:column; gap:5px; z-index:10; width: 195px; background: rgba(0,0,0,0.85); padding: 10px; border-radius: 8px; border: 1px solid #4af; }
    button { padding:8px; font-size:11px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; border-radius: 4px; width: 100%; margin-bottom: 2px; }
    .section-title { font-size: 10px; color: #4af; font-weight: bold; text-transform: uppercase; margin-bottom: 5px; display: block; }
</style>
</head>
<body>

<div class="ui" id="mainUI">
    <div id="stats">
        <div id="coinsDisplay">Coins: 0</div>
        <div id="gemsDisplay">Gems: 0</div>
        <div id="highscoreDisplayUI" style="color: #4af; font-size: 11px;">Best: 0</div>
    </div>
    <button onclick="restartGame()">Restart</button>
    <div id="shop">
        <span class="section-title">Boosters</span>
        <button id="armorBtn" onclick="buyBooster('armor', 50)">🛡️ Armor (50g)</button>
        <button id="dmgBtn" onclick="buyBooster('doubleDamage', 50)">🔥 2x Dmg (50g)</button>
    </div>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

let player, enemies, bullets, stars;
let score = 0, gameOver = false, shootCooldown = 0;
let keys = {};
let gameDifficulty = 1;

// Skin-oppsett
const creeperUrl = "https://minecraftfaces.com/wp-content/themes/minecraftfaces/faces/creeper-face.png";
const creeperImg = new Image();
creeperImg.src = creeperUrl;

let coins = Number(localStorage.getItem("coins")) || 100;
let gems = Number(localStorage.getItem("gems")) || 10;
let highscore = Number(localStorage.getItem("highscore")) || 0;
let boosters = { armor: false, doubleDamage: false };

function updateUI() {
    document.getElementById("coinsDisplay").innerText = `Coins: ${Math.floor(coins)}`;
    document.getElementById("gemsDisplay").innerText = `Gems: ${gems}`;
    document.getElementById("highscoreDisplayUI").innerText = `Best: ${Math.floor(highscore)}`;
    document.getElementById("armorBtn").style.borderColor = boosters.armor ? "#0f0" : "#4af";
    document.getElementById("dmgBtn").style.borderColor = boosters.doubleDamage ? "#0f0" : "#4af";
}

function buyBooster(type, cost) {
    if (gems >= cost && !boosters[type]) {
        gems -= cost; boosters[type] = true; updateUI();
    }
}

function init() {
    player = { x: 180, y: 540, w: 35, h: 35, speed: 6, alive: true, armorUsed: false };
    enemies = []; bullets = []; 
    stars = Array.from({length:40}, () => ({x: Math.random()*400, y: Math.random()*600, s: 1+Math.random()*2}));
    score = 0; gameDifficulty = 1; gameOver = false;
    updateUI();
}

function spawnEnemy() {
    if (gameOver) return;
    enemies.push({ x: Math.random()*370, y: -30, w: 30, h: 30, speed: 3 * gameDifficulty, hp: 1 });
}

function fire() {
    if (!player.alive) return;
    let dmg = 1 * (boosters.doubleDamage ? 2 : 1);
    bullets.push({x: player.x+15, y: player.y, vy: -12, dmg: dmg});
    shootCooldown = 15; 
}

function update() {
    if (gameOver) return;
    gameDifficulty = 1 + (score / 25000);
    if (shootCooldown > 0) shootCooldown--;
    if (player.alive && shootCooldown <= 0) fire();

    if (keys['a'] || keys['arrowleft']) player.x -= player.speed;
    if (keys['d'] || keys['arrowright']) player.x += player.speed;
    player.x = Math.max(0, Math.min(400 - player.w, player.x));

    enemies.forEach((e, i) => {
        e.y += e.speed;
        if (player.alive && player.x < e.x + e.w && player.x + player.w > e.x && player.y < e.y + e.h && player.y + player.h > e.y) {
            if (boosters.armor && !player.armorUsed) {
                player.armorUsed = true; enemies.splice(i, 1);
            } else {
                player.alive = false;
                boosters.armor = false; boosters.doubleDamage = false;
                setTimeout(() => { 
                    gameOver = true; 
                    if(score > highscore) highscore = Math.floor(score);
                    localStorage.setItem("highscore", highscore);
                    updateUI();
                }, 1000);
            }
        }
    });

    bullets.forEach((b, bi) => {
        b.y += b.vy;
        enemies.forEach((e, ei) => {
            if (b.x > e.x && b.x < e.x + e.w && b.y > e.y && b.y < e.y + e.h) {
                enemies.splice(ei, 1); bullets.splice(bi, 1);
                coins += 5; score += 100; updateUI();
            }
        });
    });

    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });
    score += 0.5;
}

function draw() {
    ctx.clearRect(0,0,400,600);
    ctx.fillStyle = "white"; stars.forEach(s => ctx.fillRect(s.x, s.y, 2, 2));
    
    // Tegn spiller som Creeper
    if (player.alive) {
        ctx.drawImage(creeperImg, player.x, player.y, player.w, player.h);
    }
    
    ctx.fillStyle = "yellow"; bullets.forEach(b => ctx.fillRect(b.x, b.y, 5, 10));
    ctx.fillStyle = "#f44"; enemies.forEach(e => ctx.fillRect(e.x, e.y, e.w, e.h));
    
    ctx.fillStyle = "white"; ctx.font = "16px Arial";
    ctx.fillText(`Score: ${Math.floor(score)}`, 10, 25);
    
    if (gameOver) {
        ctx.fillStyle = "red"; ctx.font = "30px Arial";
        ctx.fillText("GAME OVER", 110, 300);
    }
}

window.addEventListener("keydown", e => keys[e.key.toLowerCase()] = true);
window.addEventListener("keyup", e => keys[e.key.toLowerCase()] = false);
canvas.addEventListener("mousemove", e => {
    const rect = canvas.getBoundingClientRect();
    player.x = (e.clientX - rect.left) * (400 / rect.width) - 17;
});

function restartGame() { init(); }
init();
setInterval(spawnEnemy, 600);
function loop() { update(); draw(); requestAnimationFrame(loop); }
requestAnimationFrame(loop);
</script>
</body>
</html>
