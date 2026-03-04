<html lang="no">
<head>
    <meta charset="UTF-8">
    <title>Space Station - Rebirth & Explosions</title>
    <style>
        body { margin:0; background:black; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:Arial; overflow:hidden; }
        canvas { background:#05080f; border:2px solid #4af; cursor: crosshair; }
        .ui { position:absolute; top:10px; right:10px; width: 220px; background: rgba(0,0,0,0.9); padding:15px; border-radius:8px; border:1px solid #4af; z-index:100; max-height: 90vh; overflow-y: auto; }
        .stat-box { font-size: 14px; margin-bottom: 10px; border-bottom: 1px solid #333; padding-bottom: 5px; }
        .wpn-group { margin-bottom: 10px; padding: 8px; background: rgba(255,255,255,0.05); border-radius: 5px; border-left: 3px solid #4af; }
        button { padding:6px; font-size:11px; cursor:pointer; background: #222; color: white; border: 1px solid #4af; border-radius: 4px; width: 100%; margin-top: 3px; }
        button.active { background: #004400; border-color: #0f0; color: #0f0; font-weight: bold; }
        button:disabled { opacity: 0.4; cursor: not-allowed; }
        .section-title { font-size: 10px; color: #4af; font-weight: bold; text-transform: uppercase; }
        #rebirthMenu { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); background: rgba(0,0,0,0.95); padding: 20px; border: 2px solid gold; text-align: center; border-radius: 10px; display: none; z-index: 200; }
    </style>
</head>
<body>

<div id="rebirthMenu">
    <h2 style="color:gold;">REBIRTH?</h2>
    <p>Fortsett spillet for 50 Gems</p>
    <button onclick="executeRebirth()" style="background: gold; color: black; font-weight: bold;">JA, REBIRTH (50 GEMS)</button>
    <button onclick="gameOverFinal()">NEI, AVSLUTT</button>
</div>

<div class="ui">
    <div class="stat-box">
        <div>Score: <span id="scoreVal">0</span></div>
        <div>Coins: <span id="coinsVal">0</span></div>
        <div>Gems: <span id="gemsVal">0</span></div>
        <div style="font-size:11px; color:#4af;">Best Wave: <span id="waveHighVal">1</span></div>
    </div>
    
    <button onclick="paused = !paused">Pause / Fortsett</button>
    
    <div id="shopContent">
        <span class="section-title">Våpen</span>
        <div class="wpn-group">
            <button id="btn-pistol" onclick="selectWpn('pistol')">Pistol (Låst 1k)</button>
        </div>
        <div class="wpn-group">
            <button id="btn-smg" onclick="selectWpn('smg')">SMG (500c)</button>
        </div>
        <div class="wpn-group">
            <button id="btn-shotgun" onclick="selectWpn('shotgun')">Shotgun (750c)</button>
        </div>
        
        <span class="section-title">Status</span>
        <div id="statusList" style="font-size:10px; color: #0ff;"></div>
    </div>
    <button onclick="resetAllData()" style="color:red; font-size:9px; margin-top:10px;">RESET DATA</button>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

let player, enemies, bullets, stars, pickups, explosions, boss;
let score = 0, currentWave = 1, enemiesInWave = 0, waveTimer = 100;
let gameOver = false, paused = false, shootCooldown = 0;
let slowTimer = 0, magnetTimer = 0;

let coins = Number(localStorage.getItem("coins")) || 0;
let gems = Number(localStorage.getItem("gems")) || 0;
let waveHighscore = Number(localStorage.getItem("waveHighscore")) || 1;
let activeWpn = "none";
let ownedWpns = JSON.parse(localStorage.getItem("ownedWpns")) || { pistol: false, smg: false, shotgun: false };

const wpns = {
    pistol: { cost: 0, cooldown: 20, dmg: 1, type: "single" },
    smg: { cost: 500, cooldown: 7, dmg: 0.6, type: "single" },
    shotgun: { cost: 750, cooldown: 40, dmg: 1.2, type: "triple" }
};

function init() {
    player = { x: 185, y: 530, w: 30, h: 30, dead: false };
    enemies = []; bullets = []; pickups = []; explosions = []; boss = null;
    stars = Array.from({length:40}, () => ({x: Math.random()*400, y: Math.random()*600, s: 1+Math.random()*2}));
    if(!player.isRebirthing) { score = 0; currentWave = 1; }
    gameOver = false; document.getElementById("rebirthMenu").style.display = "none";
    updateUI();
}

function updateUI() {
    const set = (id, val) => { if(document.getElementById(id)) document.getElementById(id).innerText = val; };
    set("coinsVal", Math.floor(coins));
    set("gemsVal", gems);
    set("scoreVal", Math.floor(score));
    set("waveHighVal", waveHighscore);

    // Sjekk Pistol-opplåsing (1k score)
    if(score >= 1000 && !ownedWpns.pistol) {
        ownedWpns.pistol = true;
        localStorage.setItem("ownedWpns", JSON.stringify(ownedWpns));
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

function selectWpn(id) {
    if(ownedWpns[id]) activeWpn = id;
    else if(id !== 'pistol' && coins >= wpns[id].cost) {
        coins -= wpns[id].cost; ownedWpns[id] = true; activeWpn = id;
    }
    updateUI();
}

function createExplosion(x, y, color) {
    for(let i=0; i<8; i++) {
        explosions.push({x, y, vx: (Math.random()-0.5)*6, vy: (Math.random()-0.5)*6, life: 20, color});
    }
}

function spawn() {
    if(gameOver || paused || waveTimer > 0 || boss) return;
    if(currentWave === 20 && !boss) boss = { x: 150, y: -100, w: 100, h: 80, hp: 600, maxHp: 600, vx: 2 };

    let x = Math.random()*370;
    let speed = (slowTimer > 0) ? 0.8 : 2 + (currentWave*0.1);
    enemies.push({x, y:-40, w:25, h:25, hp:1, speedY: speed, color:'#f44'});

    enemiesInWave++;
    if(enemiesInWave > 15) {
        currentWave++; enemiesInWave = 0; waveTimer = 100;
        if(currentWave > waveHighscore) waveHighscore = currentWave;
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
            if(c.type === "triple") [-2, 0, 2].forEach(vx => bullets.push({x: player.x+13, y: player.y, vx, vy:-10, dmg: c.dmg}));
            else bullets.push({x: player.x+13, y: player.y, vx:0, vy:-10, dmg: c.dmg});
            shootCooldown = c.cooldown;
        }
        shootCooldown--;
    }

    bullets.forEach((b, i) => {
        b.x += b.vx; b.y += b.vy;
        if(b.y < -20) bullets.splice(i, 1);
        if(boss && b.x > boss.x && b.x < boss.x+boss.w && b.y > boss.y && b.y < boss.y+boss.h) {
            boss.hp -= b.dmg; bullets.splice(i, 1);
            if(boss.hp <= 0) {
                createExplosion(boss.x+50, boss.y+40, "gold");
                gems += 10; coins += 1000; boss = null; currentWave++; updateUI();
            }
        }
    });

    enemies.forEach((e, i) => {
        e.y += e.speedY;
        bullets.forEach((b, bi) => {
            if(b.x > e.x && b.x < e.x+e.w && b.y > e.y && b.y < e.y+e.h) {
                createExplosion(e.x+12, e.y+12, e.color);
                enemies.splice(i, 1); bullets.splice(bi, 1);
                coins += 10; score += 100; updateUI();
                if(Math.random() < 0.1) {
                    let t = Math.random();
                    pickups.push({x:e.x, y:e.y, t: t < 0.3 ? 'slow' : (t < 0.6 ? 'mag' : 'coin')});
                }
            }
        });
        if(!player.dead && player.x < e.x+e.w && player.x+30 > e.x && player.y < e.y+e.h && player.y+30 > e.y) {
            player.dead = true;
            document.getElementById("rebirthMenu").style.display = "block";
            gameOver = true;
        }
    });

    pickups.forEach((p, i) => {
        p.y += 2;
        if(player.x < p.x+20 && player.x+30 > p.x && player.y < p.y+20 && player.y+30 > p.y) {
            if(p.t === 'slow') slowTimer = 300;
            else if(p.t === 'mag') magnetTimer = 400;
            else coins += 50;
            pickups.splice(i, 1); updateUI();
        }
    });
    if(!player.dead) score += 0.2;
}

function draw() {
    ctx.clearRect(0,0,400,600);
    ctx.fillStyle = "white"; stars.forEach(s => ctx.fillRect(s.x, s.y, 1, 1));
    explosions.forEach(p => { ctx.fillStyle = p.color; ctx.fillRect(p.x, p.y, 3, 3); });
    pickups.forEach(p => {
        ctx.fillStyle = p.t==='slow'?'#0ff':(p.t==='mag'?'#f0f':'#ffd700');
        ctx.beginPath(); ctx.arc(p.x+10, p.y+10, 8, 0, 7); ctx.fill();
    });
    if(!player.dead) { ctx.fillStyle = "#0f0"; ctx.fillRect(player.x, player.y, 30, 30); }
    enemies.forEach(e => { ctx.fillStyle = e.color; ctx.fillRect(e.x, e.y, e.w, e.h); });
    if(boss) {
        ctx.fillStyle="#f0f"; ctx.fillRect(boss.x, boss.y, boss.w, boss.h);
        ctx.fillStyle="lime"; ctx.fillRect(boss.x, boss.y-10, boss.w*(boss.hp/boss.maxHp), 5);
    }
    ctx.fillStyle="yellow"; bullets.forEach(b => ctx.fillRect(b.x, b.y, 4, 10));
}

function executeRebirth() {
    if(gems >= 50) {
        gems -= 50; player.isRebirthing = true;
        init(); player.isRebirthing = false;
    } else alert("Ikke nok Gems!");
}

function gameOverFinal() { location.reload(); }
function resetAllData() { localStorage.clear(); location.reload(); }

canvas.onmousemove = e => { let r = canvas.getBoundingClientRect(); player.x = (e.clientX-r.left)*(400/r.width)-15; };

init();
setInterval(spawn, 700);
function loop() { update(); draw(); requestAnimationFrame(loop); }
loop();
</script>
</body>
</html>
