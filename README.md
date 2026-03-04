<html lang="no">
<head>
    <meta charset="UTF-8">
    <title>Space Station - Fixed</title>
    <style>
        body { margin:0; background:#05080f; color:white; font-family:Arial, sans-serif; overflow:hidden; display:flex; justify-content:center; align-items:center; height:100vh; }
        canvas { background:#000; border:2px solid #4af; display:block; cursor: crosshair; }
        .ui { position:absolute; top:10px; right:10px; width:200px; background:rgba(0,0,0,0.85); padding:15px; border:1px solid #4af; border-radius:8px; z-index:100; }
        button { width:100%; margin-top:5px; padding:8px; cursor:pointer; background:#222; color:white; border:1px solid #4af; border-radius:4px; }
        button:hover { background:#333; }
        .stat-line { margin-bottom: 5px; font-size: 14px; }
    </style>
</head>
<body>

<div class="ui">
    <div class="stat-line">Coins: <span id="coinsDisplay">0</span></div>
    <div class="stat-line">Wave: <span id="waveDisplay">1</span></div>
    <div class="stat-line" style="color:#4af">Best: <span id="highscoreDisplay">0</span></div>
    <hr border="0" style="border-top:1px solid #333">
    <button onclick="restartGame()">Restart</button>
    <button onclick="clearData()" style="color:#f44; font-size:10px;">Slett Lagring</button>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

// Spill-tilstand
let player, enemies, bullets, stars;
let coins = Number(localStorage.getItem("coins")) || 0;
let highscore = Number(localStorage.getItem("highscore")) || 0;
let currentWave = 1;
let score = 0;
let gameOver = false;

function init() {
    player = { x: 185, y: 530, w: 30, h: 30 };
    enemies = [];
    bullets = [];
    stars = Array.from({length:40}, () => ({x: Math.random()*400, y: Math.random()*600, s: 1+Math.random()*2}));
    gameOver = false;
    score = 0;
    currentWave = 1;
    updateUI();
}

function updateUI() {
    // Fail-safe oppdatering av tekst
    const setTxt = (id, txt) => {
        const el = document.getElementById(id);
        if (el) el.innerText = txt;
    };
    
    setTxt("coinsDisplay", Math.floor(coins));
    setTxt("waveDisplay", currentWave);
    setTxt("highscoreDisplay", Math.floor(highscore));
}

function update() {
    if (gameOver) return;

    // Flytt stjerner
    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });

    // Flytt kuler
    bullets.forEach((b, i) => {
        b.y -= 8;
        if (b.y < -20) bullets.splice(i, 1);
    });

    // Flytt og sjekk fiender
    enemies.forEach((e, i) => {
        e.y += 2 + (currentWave * 0.2);
        
        // Kollisjon med kuler
        bullets.forEach((b, bi) => {
            if (b.x > e.x && b.x < e.x + 25 && b.y > e.y && b.y < e.y + 25) {
                enemies.splice(i, 1);
                bullets.splice(bi, 1);
                coins += 10;
                score += 100;
                updateUI();
            }
        });

        // Kollisjon med spiller
        if (player.x < e.x + 25 && player.x + 30 > e.x && player.y < e.y + 25 && player.y + 30 > e.y) {
            gameOver = true;
            if (score > highscore) {
                highscore = score;
                localStorage.setItem("highscore", highscore);
            }
            localStorage.setItem("coins", coins);
        }

        if (e.y > 600) enemies.splice(i, 1);
    });

    // Spawn fiender
    if (Math.random() < 0.02 + (currentWave * 0.005)) {
        enemies.push({ x: Math.random() * 375, y: -30 });
    }

    // Enkel wave-logikk basert på score
    let nextWave = Math.floor(score / 2000) + 1;
    if (nextWave > currentWave) {
        currentWave = nextWave;
        updateUI();
    }
}

function draw() {
    ctx.clearRect(0, 0, 400, 600);

    // Bakgrunn
    ctx.fillStyle = "white";
    stars.forEach(s => ctx.fillRect(s.x, s.y, 1.5, 1.5));

    // Spiller
    ctx.fillStyle = "#0f0";
    ctx.fillRect(player.x, player.y, player.w, player.h);

    // Fiender
    ctx.fillStyle = "#f44";
    enemies.forEach(e => ctx.fillRect(e.x, e.y, 25, 25));

    // Kuler
    ctx.fillStyle = "yellow";
    bullets.forEach(b => ctx.fillRect(b.x, b.y, 4, 10));

    if (gameOver) {
        ctx.fillStyle = "rgba(0,0,0,0.7)";
        ctx.fillRect(0,0,400,600);
        ctx.fillStyle = "white";
        ctx.font = "30px Arial";
        ctx.fillText("GAME OVER", 110, 300);
    }
}

// Kontroller
canvas.addEventListener("mousemove", (e) => {
    const rect = canvas.getBoundingClientRect();
    player.x = (e.clientX - rect.left) * (400 / rect.width) - 15;
});

canvas.addEventListener("mousedown", () => {
    if (!gameOver) bullets.push({ x: player.x + 13, y: player.y });
});

function restartGame() { init(); }
function clearData() { localStorage.clear(); location.reload(); }

function loop() {
    update();
    draw();
    requestAnimationFrame(loop);
}

init();
loop();
</script>
</body>
</html>
