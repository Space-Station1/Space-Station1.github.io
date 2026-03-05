// Finn spawnEnemy-funksjonen din og erstatt den med denne:
function spawnEnemy() {
    if (paused || gameOver) return;
    if (megaBoss) return;

    let r = Math.random();
    if (r < 0.15 && score > 2000) {
        // Heavy Enemy
        let hp = 5 + Math.floor(score / 10000);
        enemies.push({x: Math.random()*350, y: -50, w: 45, h: 45, speedY: 1.2 * BASE_SPEED, color: '#800', coins: 50, hp: hp, maxHp: hp, isHeavy: true, type: 'heavy'});
    } else if (r < 0.30 && score > 1000) {
        // NY: Sinus Enemy (Bølgemønster)
        enemies.push({
            x: Math.random()*300 + 50, 
            y: -40, w: 25, h: 25, 
            speedY: 2 * BASE_SPEED, 
            color: '#a0f', 
            coins: 20, hp: 1, 
            isHeavy: false, 
            type: 'sinus',
            centerX: 0, // Blir satt i første update
            angle: 0
        });
    } else {
        // Vanlig Enemy
        enemies.push({x: Math.random()*370, y: -40, w: 30, h: 30, speedY: (2.5 + score/5000) * BASE_SPEED, color: '#f44', coins: 10, hp: 1, isHeavy: false, type: 'normal'});
    }
}

// Finn delen i update() der fiender beveger seg (enemies.forEach), 
// og oppdater bevegelseslogikken slik:
enemies.forEach((e, ei) => {
    // Bevegelse
    if (e.type === 'sinus') {
        if (e.centerX === 0) e.centerX = e.x;
        e.angle += 0.05;
        e.x = e.centerX + Math.sin(e.angle) * 50; // 50 er bredden på bølgen
        e.y += e.speedY * enemySpeedMult;
    } else {
        e.y += e.speedY * enemySpeedMult;
    }

    // (Resten av kollisjonshåndteringen din fortsetter herfra som før...)
    if (player.x < e.x + e.w && player.x + player.width > e.x && player.y < e.y + e.h && player.y + player.height > e.y) {
        // ... kollisjonskode
    }
});
