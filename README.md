<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Roguelike Game</title>
<style>
/* ======== CSS ======== */
body {
    margin: 0;
    padding: 20px;
    background-color: #111;
    color: white;
    font-family: Arial, sans-serif;
    display: flex;
    flex-direction: column;
    align-items: center;
    min-height: 100vh;
}
canvas { display: block; background: #1a1a1a; border: 3px solid #333; }
.key { display: inline-block; padding: 6px 12px; margin: 4px; background: #333; border-radius: 6px; border: 1px solid #666; text-align: center; }
.game-overlay { position: absolute; top:0; left:0; width:100%; height:100%; background: rgba(0,0,0,0.85); display:none; justify-content:center; align-items:center; z-index:50; }
.overlay-content { background:#222; padding:40px; border-radius:20px; border:3px solid #FFD700; text-align:center; max-width:600px; }
</style>
</head>
<body>
<h1>ROGUELIKE DUNGEON</h1>
<canvas id="game" width="800" height="600"></canvas>
<div id="gameOverOverlay" class="game-overlay">
  <div class="overlay-content">
    <h2 style="color:#f44;">GAME OVER</h2>
    <p>Нажмите ENTER для рестарта</p>
  </div>
</div>
<div id="victoryOverlay" class="game-overlay">
  <div class="overlay-content">
    <h2 style="color:#FFD700;">ПОБЕДА!</h2>
    <p>Вы победили босса!</p>
    <p>Нажмите ENTER для новой игры</p>
  </div>
</div>

<script>
// ======== JS ========

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const gameOverOverlay = document.getElementById('gameOverOverlay');
const victoryOverlay = document.getElementById('victoryOverlay');

let keys = {}, mouseX=400, mouseY=300, mouseDown=false;
canvas.addEventListener('mousemove', e=>{ const rect=canvas.getBoundingClientRect(); mouseX=e.clientX-rect.left; mouseY=e.clientY-rect.top; });
canvas.addEventListener('mousedown', e=>{ if(e.button===0) mouseDown=true; });
canvas.addEventListener('mouseup', e=>{ if(e.button===0) mouseDown=false; });
window.addEventListener('keydown', e=>keys[e.key.toLowerCase()]=true);
window.addEventListener('keyup', e=>keys[e.key.toLowerCase()]=false);

const player = {x:400, y:300, size:20, speed:2, hp:6, damage:1, invuln:0, dirX:0, dirY:-1};
let bullets=[], enemies=[], gameOver=false, bossDefeated=false;

const playerImg = (()=>{ const c=document.createElement('canvas'); c.width=c.height=60; const ctx=c.getContext('2d'); ctx.fillStyle="#4CAF50"; ctx.beginPath(); ctx.arc(30,30,20,0,Math.PI*2); ctx.fill(); ctx.fillStyle="#FFF"; ctx.beginPath(); ctx.arc(25,25,4,0,Math.PI*2); ctx.arc(35,25,4,0,Math.PI*2); ctx.fill(); const img=new Image(); img.src=c.toDataURL(); return img; })();
const bossImg = (()=>{ const c=document.createElement('canvas'); c.width=c.height=80; const ctx=c.getContext('2d'); ctx.fillStyle="#FF4444"; ctx.beginPath(); ctx.arc(40,40,30,0,Math.PI*2); ctx.fill(); const img=new Image(); img.src=c.toDataURL(); return img; })();

function shoot(dx,dy){ bullets.push({x:player.x,y:player.y,dx:dx*8,dy:dy*8,size:6}); }

function update(){
    if(gameOver || bossDefeated){ if(keys['enter']) restartGame(); return; }
    if(player.hp<=0) return triggerGameOver();
    if(player.invuln>0) player.invuln--;
    if(keys['w']) {player.y-=player.speed;player.dirX=0;player.dirY=-1;}
    if(keys['s']) {player.y+=player.speed;player.dirX=0;player.dirY=1;}
    if(keys['a']) {player.x-=player.speed;player.dirX=-1;player.dirY=0;}
    if(keys['d']) {player.x+=player.speed;player.dirX=1;player.dirY=0;}
    player.x=Math.max(20,Math.min(780,player.x));
    player.y=Math.max(20,Math.min(580,player.y));
    if(mouseDown){ const dx=mouseX-player.x; const dy=mouseY-player.y; const dist=Math.sqrt(dx*dx+dy*dy); if(dist>0) shoot(dx/dist,dy/dist); }
    bullets.forEach(b=>{ b.x+=b.dx; b.y+=b.dy; });
    bullets=bullets.filter(b=>b.x>-50&&b.x<850&&b.y>-50&&b.y<650);
}

function draw(){
    ctx.clearRect(0,0,canvas.width,canvas.height);
    // Игрок
    ctx.drawImage(playerImg,player.x-30,player.y-30);
    // Пули
    ctx.fillStyle="#FFD700";
    bullets.forEach(b=>ctx.beginPath(),ctx.arc(b.x,b.y,b.size,0,Math.PI*2),ctx.fill());
}

function gameLoop(){ update(); draw(); requestAnimationFrame(gameLoop); }
function restartGame(){ player.hp=6; bullets=[]; enemies=[]; gameOver=false; bossDefeated=false; gameOverOverlay.style.display='none'; victoryOverlay.style.display='none'; }
function triggerGameOver(){ gameOver=true; gameOverOverlay.style.display='flex'; }

gameLoop();
</script>
</body>
</html>

