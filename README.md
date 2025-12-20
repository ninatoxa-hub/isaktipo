<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Roguelike Dungeon</title>
<style>
body {
    margin: 0;
    background: #111;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
}
canvas {
    background: #1a1a1a;
    border: 3px solid #333;
}
</style>
</head>
<body>

<canvas id="game" width="800" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

/* ---------- INPUT ---------- */
const keys = {};
window.addEventListener("keydown", e => keys[e.key.toLowerCase()] = true);
window.addEventListener("keyup", e => keys[e.key.toLowerCase()] = false);

let mouseX = 400, mouseY = 300, mouseDown = false;
canvas.addEventListener("mousemove", e => {
    const r = canvas.getBoundingClientRect();
    mouseX = e.clientX - r.left;
    mouseY = e.clientY - r.top;
});
canvas.addEventListener("mousedown", e => e.button === 0 && (mouseDown = true));
canvas.addEventListener("mouseup", e => e.button === 0 && (mouseDown = false));

/* ---------- PLAYER ---------- */
const player = {
    x: 400, y: 300,
    size: 18,
    speed: 2,
    hp: 6,
    maxHp: 6,
    damage: 1,
    invuln: 0
};

/* ---------- GAME ---------- */
const maxRooms = 6;
let currentRoom = 1;
let roomCleared = false;
let bossDefeated = false;

/* ---------- OBJECTS ---------- */
let bullets = [];
let enemies = [];
let item = null;

/* ---------- SHOOTING ---------- */
let shootCooldown = 0;
const SHOOT_DELAY = 35;

/* ---------- DOORS ---------- */
const doors = [
    {x:380,y:0,w:40,h:20},
    {x:380,y:580,w:40,h:20},
    {x:0,y:280,w:20,h:40},
    {x:780,y:280,w:20,h:40}
];

/* ---------- MAP ---------- */
const mapSize = 7;
let roomX = 3, roomY = 3;
const visited = Array.from({length:mapSize},()=>Array(mapSize).fill(false));
visited[roomY][roomX] = true;

/* ---------- SPAWN ---------- */
function spawnRoom() {
    enemies = [];
    bullets = [];
    item = null;
    roomCleared = false;

    if (currentRoom === maxRooms) {
        enemies.push({x:400,y:300,size:30,hp:25,maxHp:25,speed:0.6,type:"boss"});
    } else {
        const count = 4 + Math.floor(Math.random()*2);
        for (let i=0;i<count;i++) {
            const type = Math.random()<0.3 ? "fast" : Math.random()<0.2 ? "tank" : "normal";
            enemies.push({
                x:Math.random()*700+50,
                y:Math.random()*500+50,
                size:type==="tank"?26:20,
                hp:type==="tank"?6:3,
                speed:type==="fast"?1.4:0.8,
                type
            });
        }
    }
}
spawnRoom();

/* ---------- ITEMS ---------- */
function spawnItem() {
    const types = ["damage","hp","speed"];
    item = {
        x:400,y:300,size:12,
        type: types[Math.floor(Math.random()*types.length)]
    };
}

/* ---------- NEXT ROOM ---------- */
function nextRoom() {
    if (!roomCleared) return;
    currentRoom++;
    if (currentRoom > maxRooms) return;
    player.x = 400;
    player.y = 300;
    spawnRoom();
}

/* ---------- UPDATE ---------- */
function update() {
    if (bossDefeated) return;

    if (player.invuln > 0) player.invuln--;

    /* movement */
    if (keys.w) player.y -= player.speed;
    if (keys.s) player.y += player.speed;
    if (keys.a) player.x -= player.speed;
    if (keys.d) player.x += player.speed;

    player.x = Math.max(20, Math.min(780, player.x));
    player.y = Math.max(20, Math.min(580, player.y));

    /* shooting */
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

    bullets.forEach(b => { b.x+=b.dx; b.y+=b.dy; });
    bullets = bullets.filter(b => b.alive);

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
        if (currentRoom === maxRooms) bossDefeated = true;
        else spawnItem();
    }

    if (item && Math.hypot(player.x-item.x,player.y-item.y)<player.size+item.size) {
        if (item.type==="damage") player.damage++;
        if (item.type==="hp") { player.maxHp++; player.hp++; }
        if (item.type==="speed") player.speed+=0.3;
        item = null;
    }

    if (roomCleared) {
        doors.forEach(d => {
            if (player.x>d.x-10 && player.x<d.x+d.w+10 &&
                player.y>d.y-10 && player.y<d.y+d.h+10) {
                nextRoom();
            }
        });
    }
}

/* ---------- DRAW ---------- */
function draw() {
    ctx.clearRect(0,0,800,600);

    /* player */
    ctx.fillStyle="#4CAF50";
    ctx.beginPath();
    ctx.arc(player.x,player.y,player.size,0,Math.PI*2);
    ctx.fill();

    /* bullets */
    ctx.fillStyle="#FFD700";
    bullets.forEach(b=>{
        ctx.beginPath();
        ctx.arc(b.x,b.y,b.size,0,Math.PI*2);
        ctx.fill();
    });

    /* enemies */
    enemies.forEach(e=>{
        ctx.fillStyle = e.type==="boss"?"#FF0000":e.type==="fast"?"#FF8800":"#FF4444";
        ctx.beginPath();
        ctx.arc(e.x,e.y,e.size,0,Math.PI*2);
        ctx.fill();
    });

    /* item */
    if (item) {
        ctx.fillStyle = item.type==="damage"?"gold":item.type==="hp"?"lime":"cyan";
        ctx.beginPath();
        ctx.arc(item.x,item.y,item.size,0,Math.PI*2);
        ctx.fill();
    }

    /* doors */
    doors.forEach(d=>{
        ctx.fillStyle = roomCleared?"gold":"#444";
        ctx.fillRect(d.x,d.y,d.w,d.h);
    });

    /* boss hp */
    enemies.forEach(e=>{
        if (e.type==="boss") {
            ctx.fillStyle="#000";
            ctx.fillRect(250,20,300,20);
            ctx.fillStyle="red";
            ctx.fillRect(250,20,300*(e.hp/e.maxHp),20);
        }
    });

    /* UI */
    ctx.fillStyle="white";
    ctx.fillText(`Комната ${currentRoom}/${maxRooms}`,20,20);
    ctx.fillText(`HP ${player.hp}/${player.maxHp}`,20,40);
    ctx.fillText(`DMG ${player.damage}`,20,60);

    if (bossDefeated) {
        ctx.fillStyle="gold";
        ctx.font="48px Arial";
        ctx.fillText("ПОБЕДА!",300,300);
    }
}

/* ---------- LOOP ---------- */
function loop() {
    update();
    draw();
    requestAnimationFrame(loop);
}
loop();
</script>
</body>
</html>
