<html lang="no">
<head>
    <meta charset="UTF-8">
    <title>Space Station - Persistent Storage</title>
    <style>
        body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; }
        canvas { background:#05080f; border:2px solid #4af; cursor: crosshair; }
        .ui { position:absolute; top:10px; right:10px; width: 220px; background: rgba(0,0,0,0.9); padding:15px; border-radius:8px; border:1px solid #4af; z-index:100; max-height: 90vh; overflow-y: auto; }
        .stat-box { font-size: 14px; margin-bottom: 10px; border-bottom: 1px solid #333; padding-bottom: 5px; }
        .wpn-group { margin-bottom: 10px; padding: 8px; background: rgba(255,255,255,0.05); border-radius: 5px; border-left: 3px solid #4af; }
        button { padding:6px; font-size:11px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; border-radius: 4px; width: 100%; margin-top: 3px; }
        button.active { background: #004400; border-color: #0f0; color: #0f0; font-weight: bold; }
        #deathMenu { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); background: rgba(0,0,0,0.95); padding: 25px; border: 2px solid #4af; text-align: center; border-radius: 10px; display: none; z-index: 200; min-width: 250px; }
        .section-title { font-size: 10px; color: #4af; font-weight: bold; text-transform: uppercase; }
    </style>
</head>
<body>

<div id="deathMenu">
    <h2 style="color:red; margin-top:0;">CRASHED!</h2>
    <p style="font-size: 13px;">Vil du fortsette eller lagre og gå til menyen?</p>
    <button onclick="executeAction('continue', 20)" style="background: #222; color: white;">CONTINUE (20 GEMS)</button>
    <button onclick="executeAction('rebirth', 50)" style="background: gold; color: black; font-weight: bold;">REBIRTH (50 GEMS)</button>
    <hr style="border:0; border-top:1px solid #333; margin: 15px 0;">
    <button onclick="gameOverFinal()" style="background: #444;">NEI, AVSLUTT & LAGRE DATA</button>
</div>

<div class="ui">
    <div class="stat-box">
        <div>Coins: <span id="coinsVal">0</span></div>
        <div>Gems: <span id="gemsVal">0</span></div>
        <div style="font-size:11px; color:#4af;">Best Wave: <span id="waveHighVal">1</span></div>
    </div>
    
    <button onclick="paused = !paused" style="background:#333;">Pause / Fortsett</button>
    <button onclick="restartGame()" style="background:#440000; border-color: #f44;">RESTART SPILLET</button>
    
    <div id="shopContent">
        <span class="section-title" style="margin-top:10px; display:block;">Våpen (1-4)</span>
        <div class="wpn-group">
            <button id="btn-pistol" onclick="selectWpn('pistol')">Pistol (Låst 1k)</button>
        </div>
        <div class="wpn-group">
            <button id="btn-smg" onclick="selectWpn('smg')">SMG (500c)</button>
        </div>
        <div class="wpn-group">
            <button id="btn-shotgun" onclick="selectWpn('shotgun')">Shotgun (750c)</button>
        </div>
    </div>
    <button onclick="resetAllData()" style="color:#666; font-size:9px; margin-top:15px; border-color:#333;">Slett all fremgang</button>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

let player, enemies, bullets, stars, pickups, explosions, boss;
let score = 0, currentWave = 1, enemiesInWave = 0, waveTimer = 100;
let gameOver = false, paused = false, shootCooldown = 0;
let slowTimer = 0, magnetTimer = 0;

// Lasting fra LocalStorage
let coins = Number(localStorage.getItem("coins")) || 0;
let gems = Number(localStorage.getItem("gems")) || 0;
let highscore = Number(localStorage.getItem("highscore")) || 0;
let waveHighscore = Number(localStorage.getItem("waveHighscore")) || 1;
let activeWpn = "none";
let ownedWpns = JSON.parse(localStorage.getItem("ownedWpns")) || { pistol: false, smg: false, shotgun: false };

const wpns = {
    pistol: { cost: 0, cooldown: 20, dmg: 1, type: "single" },
    smg: { cost: 500, cooldown: 7, dmg: 0.6, type: "single" },
    shotgun: { cost: 750, cooldown: 42, dmg: 1.3, type: "triple" }
};

function init(keepScore = false) {
    player = { x: 185, y: 530, w: 30, h: 30, dead: false };
    enemies = []; bullets = []; pickups = []; explosions = []; boss = null;
    stars = Array.from({length:45}, () => ({x: Math.random()*400, y: Math.random()*600, s: 0.8+Math.random()*2}));
    
    if(!keepScore) {
        score = 0;
        currentWave = 1;
        enemiesInWave = 0;
    }
    
    gameOver = false;
    paused = false;
    document.getElementById("deathMenu").style.display = "none";
    updateUI();
}

function updateUI() {
    document.getElementById("coinsVal").innerText = Math.floor(coins);
    document.getElementById("gemsVal").innerText = gems;
    document.getElementById("waveHighVal").innerText = waveHighscore;

    if(score >= 1000 && !ownedWpns.pistol) {
        ownedWpns.pistol = true;
        save();
    }

    for(let id in wpns) {
        let b = document.getElementById("btn-" + id);
        if(!b) continue;
        if(ownedWpns[id]) {
            b.innerText = id.toUpperCase();
            b.className = (activeWpn === id) ? "active" : "";
        } else {
            b.innerText = id === 'pistol' ? "Låst (1k Score)" : `Kjøp ${id.toUpperCase()} (${wpns[id].cost}c)`;
        }
    }
}

function save() {
    localStorage.setItem("coins", coins);
    localStorage.setItem("gems", gems);
    localStorage.setItem("highscore", highscore);
    localStorage.setItem("waveHighscore", waveHighscore);
    localStorage.setItem("ownedWpns", JSON.stringify(ownedWpns));
}

function createExplosion(x, y, color) {
    for(let i=0; i<10; i++) {
        explosions.push({x, y, vx: (Math.random()-0.5)*8, vy: (Math.random()-0.5)*8, life: 25, color});
    }
}

function spawn() {
    if(gameOver || paused || waveTimer > 0 || boss) return;
    if(currentWave === 20 && !boss) boss = { x: 150, y: -100, w: 100, h: 80, hp: 700, maxHp: 700, vx: 2.5 };

    let x = Math.random()*370;
    let speed = (slowTimer > 0) ? 0.7 : 2 + (currentWave*0.12);
    enemies.push({x, y:-40, w:28, h:28, hp:1, speedY: speed, color:'#f44'});

    enemiesInWave++;
    if(enemiesInWave > 18) {
        currentWave++;
        enemiesInWave = 0;
        waveTimer = 120;
        if(currentWave > waveHighscore) waveHighscore = currentWave;
        save();
        updateUI();
    }
}

function update() {
    if(gameOver || paused) return;
    if(waveTimer > 0) waveTimer--;
    if(slowTimer > 0) slowTimer--;
    if(magnetTimer > 0) magnetTimer--;

    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });
    explosions.forEach((p, i) => { p.x += p.vx; p.y += p.vy; p.life--; if(p.life <= 0) explosions.splice(i, 1); });

    if(!player.dead && activeWpn !== "none") {
        if(shootCooldown <= 0) {
            let c = wpns[activeWpn];
            if(c.type === "triple") {
                [-2.5, 0, 2.5].forEach(vx => bullets.push({x: player.x+13, y: player.y, vx, vy:-11, dmg: c.dmg}));
            } else {
                bullets.push({x: player.x+13, y: player.y, vx:0, vy:-11, dmg: c.dmg});
            }
            shootCooldown = c.cooldown;
        }
        shootCooldown--;
    }

    if(boss) {
        if(boss.y < 60) boss.y += 0.6;
        boss.x += boss.vx;
        if(boss.x <= 0 || boss.x + boss.w >= 400) boss.vx *= -1;
    }

    bullets.forEach((b, i) => {
        b.x += b.vx; b.y += b.vy;
        if(b.y < -20) bullets.splice(i, 1);
        if(boss && b.x > boss.x && b.x < boss.x+boss.w && b.y > boss.y && b.y < boss.y+boss.h) {
            boss.hp -= b.dmg; bullets.splice(i, 1);
            if(boss.hp <= 0) {
                createExplosion(boss.x+50, boss.y+40, "orange");
                gems += 12; coins += 1500; boss = null; currentWave++; updateUI();
            }
        }
    });

    enemies.forEach((e, i) => {
        e.y += e.speedY;
        bullets.forEach((b, bi) => {
            if(b.x > e.x && b.x < e.x+e.w && b.y > e.y && b.y < e.y+e.h) {
                createExplosion(e.x+14, e.y+14, "#f44");
                enemies.splice(i, 1); bullets.splice(bi, 1);
                coins += 10; score += 100;
                if(score > highscore) highscore = Math.floor(score);
                updateUI();
                if(Math.random() < 0.12) {
                    let t = Math.random();
                    pickups.push({x:e.x, y:e.y, t: t < 0.3 ? 'slow' : (t < 0.6 ? 'mag' : 'coin')});
                }
            }
        });
        if(!player.dead && player.x < e.x+e.w && player.x+30 > e.x && player.y < e.y+e.h && player.y+30 > e.y) {
            player.dead = true;
            gameOver = true;
            save(); // Lagrer ved død
            document.getElementById("deathMenu").style.display = "block";
        }
        if(e.y > 600) enemies.splice(i, 1);
    });

    pickups.forEach((p, i) => {
        if(magnetTimer > 0) { p.x += (player.x-p.x)*0.12; p.y += (player.y-p.y)*0.12; }
        else p.y += 2.2;
        if(player.x < p.x+22 && player.x+30 > p.x && player.y < p.y+22 && player.y+30 > p.y) {
            if(p.t === 'slow') slowTimer = 350;
            else if(p.t === 'mag') magnetTimer = 450;
            else coins += 60;
            pickups.splice(i, 1); updateUI();
        }
    });
    if(!player.dead) score += 0.25;
}

function draw() {
    ctx.clearRect(0,0,400,600);
    ctx.fillStyle = "white"; stars.forEach(s => ctx.fillRect(s.x, s.y, 1.2, 1.2));
    explosions.forEach(p => { ctx.fillStyle = p.color; ctx.fillRect(p.x, p.y, 3, 3); });
    
    pickups.forEach(p => {
        ctx.fillStyle = p.t==='slow'?'#0ff':(p.t==='mag'?'#f0f':'#ffd700');
        ctx.beginPath(); ctx.arc(p.x+11, p.y+11, 9, 0, 7); ctx.fill();
    });

    if(!player.dead) { 
        ctx.fillStyle = "#0f0"; ctx.fillRect(player.x, player.y, 30, 30); 
    }

    enemies.forEach(e => { ctx.fillStyle = e.color; ctx.fillRect(e.x, e.y, e.w, e.h); });

    if(boss) {
        ctx.fillStyle="#f0f"; ctx.fillRect(boss.x, boss.y, boss.w, boss.h);
        ctx.fillStyle="lime"; ctx.fillRect(boss.x, boss.y-12, boss.w*(boss.hp/boss.maxHp), 6);
    }

    ctx.fillStyle="yellow"; bullets.forEach(b => ctx.fillRect(b.x, b.y, 4, 12));

    ctx.fillStyle = "white";
    ctx.font = "bold 18px Arial";
    ctx.fillText(`SCORE: ${Math.floor(score)}`, 15, 30);
    ctx.font = "12px Arial";
    ctx.fillStyle = "#aaa";
    ctx.fillText(`BEST: ${Math.floor(highscore)}`, 15, 50);

    if(waveTimer > 0) {
        ctx.fillStyle="rgba(255,255,0,0.8)"; ctx.font="bold 35px Arial";
        ctx.fillText(`WAVE ${currentWave}`, 120, 300);
    }
}

function executeAction(type, cost) {
    if(gems >= cost) {
        gems -= cost;
        save();
        init(true);
    } else {
        alert("Ikke nok Gems!");
    }
}

function selectWpn(id) {
    if(ownedWpns[id]) activeWpn = id;
    else if(id !== 'pistol' && coins >= wpns[id].cost) {
        coins -= wpns[id].cost; ownedWpns[id] = true; activeWpn = id;
    }
    save(); updateUI();
}

function restartGame() { if(confirm("Starte helt på nytt?")) { save(); init(false); } }
function gameOverFinal() { save(); location.reload(); } // Denne er nå trygg
function resetAllData() { if(confirm("SLETTE ALT?")) { localStorage.clear(); location.reload(); } }

canvas.onmousemove = e => { 
    let r = canvas.getBoundingClientRect(); 
    player.x = (e.clientX-r.left)*(400/r.width)-15; 
};

init();
setInterval(spawn, 650);
function loop() { update(); draw(); requestAnimationFrame(loop); }
loop();
</script>
</body>
</html>