<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Roguelike</title>
<style>
body { margin:0; background:#111; display:flex; justify-content:center; align-items:center; height:100vh; }
canvas { background:#1a1a1a; border:3px solid #333; }
</style>
</head>
<body>

<canvas id="game" width="800" height="600"></canvas>

<script>
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');

/* ---------- УПРАВЛЕНИЕ ---------- */
const keys = {};
window.addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
window.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);

let mouseX = 400, mouseY = 300, mouseDown = false;
canvas.addEventListener('mousemove', e => {
    const r = canvas.getBoundingClientRect();
    mouseX = e.clientX - r.left;
    mouseY = e.clientY - r.top;
});
canvas.addEventListener('mousedown', e => e.button === 0 && (mouseDown = true));
canvas.addEventListener('mouseup', e => e.button === 0 && (mouseDown = false));

/* ---------- ИГРОК ---------- */
const player = {
    x:400, y:300,
    size:18,
    speed:2,
    hp:6,
    damage:1,
    invuln:0
};

/* ---------- КОМНАТЫ ---------- */
const maxRooms = 6;
let currentRoom = 1;
let roomType = 'enemy';
let roomCleared = false;
let bossDefeated = false;

/* ---------- ОБЪЕКТЫ ---------- */
let bullets = [];
let enemies = [];

/* ---------- СТРЕЛЬБА ---------- */
let shootCooldown = 0;
const SHOOT_DELAY = 35;

/* ---------- ДВЕРИ ---------- */
const doors = [
    {x:380,y:0,w:40,h:20},
    {x:380,y:580,w:40,h:20},
    {x:0,y:280,w:20,h:40},
    {x:780,y:280,w:20,h:40}
];

/* ---------- СПАВН ---------- */
function spawnRoom() {
    enemies = [];
    bullets = [];
    roomCleared = false;

    if (currentRoom === maxRooms) {
        roomType = 'boss';
        enemies.push({ x:400,y:300,size:30,hp:25,speed:0.6 });
    } else {
        roomType = 'enemy';
        for (let i=0;i<5;i++) {
            enemies.push({
                x:Math.random()*700+50,
                y:Math.random()*500+50,
                size:20,
                hp:3,
                speed:0.8
            });
        }
    }
}
spawnRoom();

/* ---------- ПЕРЕХОД ---------- */
function nextRoom() {
    if (!roomCleared) return;
    currentRoom++;
    if (currentRoom > maxRooms) return;
    player.x = 400;
    player.y = 300;
    spawnRoom();
}

/* ---------- ОБНОВЛЕНИЕ ---------- */
function update() {
    if (bossDefeated) return;

    if (player.invuln > 0) player.invuln--;

    /* движение */
    if (keys.w) player.y -= player.speed;
    if (keys.s) player.y += player.speed;
    if (keys.a) player.x -= player.speed;
    if (keys.d) player.x += player.speed;

    player.x = Math.max(20, Math.min(780, player.x));
    player.y = Math.max(20, Math.min(580, player.y));

    /* стрельба */
    if (shootCooldown > 0) shootCooldown--;
    if (mouseDown && shootCooldown <= 0) {
        const dx = mouseX - player.x;
        const dy = mouseY - player.y;
        const d = Math.hypot(dx,dy);
        bullets.push({
            x:player.x,y:player.y,
            dx:(dx/d)*7, dy:(dy/d)*7,
            size:5, alive:true
        });
        shootCooldown = SHOOT_DELAY;
    }

    /* пули */
    bullets.forEach(b => { b.x+=b.dx; b.y+=b.dy; });
    bullets = bullets.filter(b => b.x>-50&&b.x<850&&b.y>-50&&b.y<650&&b.alive);

    /* враги */
    enemies.forEach(e => {
        const dx = player.x - e.x;
        const dy = player.y - e.y;
        const d = Math.hypot(dx,dy);
        e.x += dx/d * e.speed;
        e.y += dy/d * e.speed;

        if (d < player.size + e.size && player.invuln === 0) {
            player.hp--;
            player.invuln = 60;
        }
    });

    /* урон */
    bullets.forEach(b => {
        enemies.forEach(e => {
            if (!b.alive) return;
            if (Math.hypot(b.x-e.x,b.y-e.y) < b.size+e.size) {
                e.hp -= player.damage;
                b.alive = false;
            }
        });
    });

    enemies = enemies.filter(e => e.hp > 0);

    if (enemies.length === 0 && !roomCleared) {
        roomCleared = true;
        if (roomType === 'boss') bossDefeated = true;
    }

    /* двери */
    if (roomCleared) {
        doors.forEach(d => {
            if (
                player.x > d.x-10 && player.x < d.x+d.w+10 &&
                player.y > d.y-10 && player.y < d.y+d.h+10
            ) nextRoom();
        });
    }
}

/* ---------- ОТРИСОВКА ---------- */
function draw() {
    ctx.clearRect(0,0,800,600);

    /* игрок */
    ctx.fillStyle='#4CAF50';
    ctx.beginPath();
    ctx.arc(player.x,player.y,player.size,0,Math.PI*2);
    ctx.fill();

    /* пули */
    ctx.fillStyle='#FFD700';
    bullets.forEach(b=>{
        ctx.beginPath();
        ctx.arc(b.x,b.y,b.size,0,Math.PI*2);
        ctx.fill();
    });

    /* враги */
    ctx.fillStyle='#FF4444';
    enemies.forEach(e=>{
        ctx.beginPath();
        ctx.arc(e.x,e.y,e.size,0,Math.PI*2);
        ctx.fill();
    });

    /* двери */
    doors.forEach(d=>{
        ctx.fillStyle = roomCleared ? '#FFD700' : '#444';
        ctx.fillRect(d.x,d.y,d.w,d.h);
    });

    /* HUD */
    ctx.fillStyle='white';
    ctx.fillText(`Комната: ${currentRoom}/${maxRooms}`,20,20);
    ctx.fillText(`HP: ${player.hp}`,20,40);

    if (bossDefeated) {
        ctx.fillStyle='gold';
        ctx.font='48px Arial';
        ctx.fillText('ПОБЕДА!',300,300);
    }
}

/* ---------- ЦИКЛ ---------- */
function loop() {
    update();
    draw();
    requestAnimationFrame(loop);
}
loop();
</script>

</body>
</html>
