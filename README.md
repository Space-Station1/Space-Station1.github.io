<html lang="no">
<head>
    <meta charset="UTF-8">
    <title>Space Station - Fixed & Final</title>
    <style>
        body { margin:0; background:#000; color:white; display:flex; justify-content:center; align-items:center; height:100vh; font-family:sans-serif; overflow:hidden; }
        canvas { background:#05080f; border:2px solid #4af; cursor: none; }
        .ui { position:absolute; top:10px; right:10px; width: 220px; background: rgba(0,0,0,0.9); padding:15px; border-radius:8px; border:1px solid #4af; }
        button { width:100%; padding:8px; margin-top:5px; cursor:pointer; background:#222; color:white; border:1px solid #4af; border-radius:4px; font-size:11px; }
        button.active { background:#004400; border-color:#0f0; color:#0f0; font-weight:bold; }
        #deathMenu { position:absolute; top:50%; left:50%; transform:translate(-50%,-50%); background:rgba(0,0,0,0.95); padding:30px; border:3px solid #4af; text-align:center; display:none; z-index:200; }
    </style>
</head>
<body>

<div id="deathMenu">
    <h2 style="color:#f44;">SKIPET ER ØDELAGT</h2>
    <p id="finalScore">Score: 0</p>
    <button onclick="useGemAction(20)">CONTINUE (20 GEMS)</button>
    <button onclick="useGemAction(50)" style="background:gold; color:black;">REBIRTH (50 GEMS)</button>
    <button onclick="location.reload()" style="background:#444;">AVSLUTT</button>
</div>

<div class="ui">
    <div style="font-size:14px; border-bottom:1px solid #333; padding-bottom:5px; margin-bottom:10px;">
        Coins: <span id="cVal">0</span> | Gems: <span id="gVal">0</span>
    </div>
    <button onclick="paused = !paused">Pause</button>
    <button onclick="resetGame()" style="background:#400;">Restart Runde</button>
    <div style="margin-top:10px; font-size:10px; color:#4af;">VÅPEN (1-3)</div>
    <button id="w-pistol" onclick="buyOrSelect('pistol')">Pistol (1k Score)</button>
    <button id="w-smg" onclick="buyOrSelect('smg')">SMG (500c)</button>
    <button id="w-shotgun" onclick="buyOrSelect('shotgun')">Shotgun (750c)</button>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game"), ctx = canvas.getContext("2d");
let player, enemies, bullets, stars, explosions, pickups, boss;
let score = 0, coins = Number(localStorage.getItem("coins"))||0, gems = Number(localStorage.getItem("gems"))||0;
let currentWave = 1, waveTimer = 0, enemiesInWave = 0, gameOver = false, paused = false;
let activeWpn = localStorage.getItem("activeWpn") || "none", shootCd = 0;
let owned = JSON.parse(localStorage.getItem("owned")) || {pistol:false, smg:false, shotgun:false};

const wpnData = {
    pistol: {cost:0, cd:18, dmg:1, type:"single"},
    smg: {cost:500, cd:6, dmg:0.6, type:"single"},
    shotgun: {cost:750, cd:40, dmg:1.4, type:"triple"}
};

function init(cont = false) {
    player = {x:185, y:530, w:30, h:30};
    enemies = []; bullets = []; explosions = []; pickups = []; boss = null;
    stars = Array.from({length:50}, () => ({x:Math.random()*400, y:Math.random()*600, s:Math.random()*2+1}));
    if(!cont) { score = 0; currentWave = 1; enemiesInWave = 0; }
    gameOver = false; document.getElementById("deathMenu").style.display = "none";
    updateUI();
}

function updateUI() {
    document.getElementById("cVal").innerText = Math.floor(coins);
    document.getElementById("gVal").innerText = gems;
    if(score >= 1000 && !owned.pistol) { owned.pistol = true; save(); }
    for(let id in wpnData) {
        let btn = document.getElementById("w-"+id);
        if(owned[id]) { btn.innerText = id.toUpperCase(); btn.className = activeWpn === id ? "active" : ""; }
        else { btn.innerText = id === 'pistol' ? "Låst (1k)" : "Kjøp "+id.toUpperCase()+" ("+wpnData[id].cost+")"; }
    }
}

function save() {
    localStorage.setItem("coins", coins); localStorage.setItem("gems", gems);
    localStorage.setItem("owned", JSON.stringify(owned)); localStorage.setItem("activeWpn", activeWpn);
}

function buyOrSelect(id) {
    if(owned[id]) activeWpn = id;
    else if(id !== 'pistol' && coins >= wpnData[id].cost) { coins -= wpnData[id].cost; owned[id] = true; activeWpn = id; }
    save(); updateUI();
}

function explode(x, y, color) {
    for(let i=0; i<10; i++) explosions.push({x, y, vx:(Math.random()-0.5)*6, vy:(Math.random()-0.5)*6, l:30, c:color});
}

function update() {
    if(gameOver || paused) return;
    stars.forEach(s => { s.y += s.s; if(s.y > 600) s.y = 0; });
    explosions.forEach((p,i) => { p.x += p.vx; p.y += p.vy; p.l--; if(p.l<=0) explosions.splice(i,1); });

    if(activeWpn !== "none" && shootCd-- <= 0) {
        let w = wpnData[activeWpn];
        if(w.type === "triple") [-2,0,2].forEach(vx => bullets.push({x:player.x+13, y:player.y, vx, vy:-10, d:w.dmg}));
        else bullets.push({x:player.x+13, y:player.y, vx:0, vy:-10, d:w.dmg});
        shootCd = w.cd;
    }

    bullets.forEach((b,i) => {
        b.x += b.vx; b.y += b.vy;
        if(b.y < -20) bullets.splice(i,1);
        enemies.forEach((e,ei) => {
            if(b.x > e.x && b.x < e.x+e.w && b.y > e.y && b.y < e.y+e.h) {
                explode(e.x+13, e.y+13, "#f44");
                enemies.splice(ei,1); bullets.splice(i,1);
                score += 100; coins += 10; updateUI();
                if(Math.random() < 0.1) pickups.push({x:e.x, y:e.y, t:Math.random()<0.5?'gem':'coin'});
            }
        });
    });

    if(Math.random() < 0.02 + (currentWave*0.005)) enemies.push({x:Math.random()*370, y:-30, w:26, h:26, s:2+(currentWave*0.1)});
    enemies.forEach((e,i) => {
        e.y += e.s;
        if(e.y > 600) enemies.splice(i,1);
        if(player.x < e.x+26 && player.x+30 > e.x && player.y < e.y+26 && player.y+30 > e.y) {
            gameOver = true; save(); document.getElementById("deathMenu").style.display = "block";
            document.getElementById("finalScore").innerText = "Score: " + Math.floor(score);
        }
    });

    pickups.forEach((p,i) => {
        p.y += 2;
        if(player.x < p.x+20 && player.x+30 > p.x && player.y < p.y+20 && player.y+30 > p.y) {
            if(p.t === 'gem') gems++; else coins += 50;
            pickups.splice(i,1); updateUI();
        }
    });

    if(enemiesInWave++ > 20) { currentWave++; enemiesInWave = 0; waveTimer = 100; }
    if(waveTimer > 0) waveTimer--;
    score += 0.1;
}

function draw() {
    ctx.clearRect(0,0,400,600);
    ctx.fillStyle = "white"; stars.forEach(s => ctx.fillRect(s.x, s.y, 1, 1));
    explosions.forEach(p => { ctx.fillStyle = p.c; ctx.fillRect(p.x, p.y, 2, 2); });
    pickups.forEach(p => { ctx.fillStyle = p.t==='gem'?'#0ff':'#ffd700'; ctx.beginPath(); ctx.arc(p.x+10,p.y+10,7,0,7); ctx.fill(); });
    ctx.fillStyle = "#0f0"; ctx.fillRect(player.x, player.y, 30, 30);
    ctx.fillStyle = "#f44"; enemies.forEach(e => ctx.fillRect(e.x, e.y, e.w, e.h));
    ctx.fillStyle = "yellow"; bullets.forEach(b => ctx.fillRect(b.x, b.y, 4, 10));
    ctx.fillStyle = "white"; ctx.font = "bold 16px Arial"; ctx.fillText("SCORE: "+Math.floor(score), 10, 25);
    if(waveTimer > 0) { ctx.fillStyle = "yellow"; ctx.font = "bold 40px Arial"; ctx.fillText("WAVE "+currentWave, 120, 300); }
}

function useGemAction(cost) {
    if(gems >= cost) { gems -= cost; save(); init(true); } else alert("Ikke nok Gems!");
}

canvas.onmousemove = e => { player.x = (e.clientX - canvas.getBoundingClientRect().left) - 15; };
window.onkeydown = e => { if(e.key === '1') buyOrSelect('pistol'); if(e.key === '2') buyOrSelect('smg'); if(e.key === '3') buyOrSelect('shotgun'); };
function loop() { update(); draw(); requestAnimationFrame(loop); }
init(); loop();
</script>
</body>
</html>
