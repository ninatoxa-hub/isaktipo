<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Isaac-like FINAL</title>
<style>
body{margin:0;background:#111;display:flex;justify-content:center;align-items:center;height:100vh}
canvas{background:#1a1a1a;border:3px solid #333}
</style>
</head>
<body>
<canvas id="game" width="800" height="600"></canvas>

<script>
const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");

/* ---------- AUDIO ---------- */
const audioCtx=new (window.AudioContext||window.webkitAudioContext)();
function beep(freq,time){
  const o=audioCtx.createOscillator();
  const g=audioCtx.createGain();
  o.frequency.value=freq;
  o.connect(g);g.connect(audioCtx.destination);
  o.start();g.gain.exponentialRampToValueAtTime(0.0001,audioCtx.currentTime+time);
}
function shootSound(){beep(600,0.1)}
function hitSound(){beep(200,0.15)}
function itemSound(){beep(900,0.2)}
setInterval(()=>beep(120,0.05),2000);

/* ---------- INPUT ---------- */
const keys={};
window.addEventListener("keydown",e=>keys[e.key.toLowerCase()]=true);
window.addEventListener("keyup",e=>keys[e.key.toLowerCase()]=false);

/* ---------- SAVE ---------- */
function saveGame(){
 localStorage.setItem("save",JSON.stringify(player));
}
function loadGame(){
 const s=localStorage.getItem("save");
 if(s) Object.assign(player,JSON.parse(s));
}

/* ---------- PLAYER ---------- */
const player={
 x:400,y:300,size:18,
 speed:2,damage:1,
 hp:6,maxHp:6,
 invuln:0
};
loadGame();

/* ---------- GAME ---------- */
let room=1;
const maxRooms=6;
let bullets=[],enemies=[],items=[];
let shootCD=0;

/* ---------- BOSSES ---------- */
function spawnBoss(){
 const t=Math.floor(Math.random()*3);
 if(t===0) enemies.push({type:"tank",x:400,y:300,hp:40,maxHp:40,size:35,speed:0.5});
 if(t===1) enemies.push({type:"shooter",x:400,y:300,hp:25,maxHp:25,size:28,speed:0.8,cd:0});
 if(t===2) enemies.push({type:"rage",x:400,y:300,hp:30,maxHp:30,size:30,speed:0.7});
}

/* ---------- SPAWN ---------- */
function spawnRoom(){
 bullets=[];enemies=[];items=[];
 if(room===maxRooms) spawnBoss();
 else for(let i=0;i<4;i++)
  enemies.push({type:"normal",x:Math.random()*700+50,y:Math.random()*500+50,hp:3,size:20,speed:1});
}
spawnRoom();

/* ---------- UPDATE ---------- */
function update(){
 if(player.invuln>0) player.invuln--;

 if(keys.w) player.y-=player.speed;
 if(keys.s) player.y+=player.speed;
 if(keys.a) player.x-=player.speed;
 if(keys.d) player.x+=player.speed;

 if(shootCD>0) shootCD--;
 const dirs=[
  ["arrowup","w",0,-1],
  ["arrowdown","s",0,1],
  ["arrowleft","a",-1,0],
  ["arrowright","d",1,0]
 ];
 dirs.forEach(d=>{
  if(keys[d[0]]&&shootCD<=0){
   bullets.push({x:player.x,y:player.y,dx:d[2]*6,dy:d[3]*6});
   shootSound();
   shootCD=20;
  }
 });

 bullets.forEach(b=>{b.x+=b.dx;b.y+=b.dy});

 enemies.forEach(e=>{
  let dx=player.x-e.x,dy=player.y-e.y,d=Math.hypot(dx,dy);
  if(e.type==="rage"&&e.hp<e.maxHp/2) e.speed=1.6;
  e.x+=dx/d*e.speed;e.y+=dy/d*e.speed;

  if(e.type==="shooter"){
   e.cd--; if(e.cd<=0){
    bullets.push({x:e.x,y:e.y,dx:dx/d*4,dy:dy/d*4,enemy:true});
    e.cd=60;
   }
  }

  if(d<player.size+e.size&&player.invuln===0){
   player.hp--;player.invuln=60;hitSound();
  }
 });

 bullets.forEach(b=>{
  enemies.forEach(e=>{
   if(Math.hypot(b.x-e.x,b.y-e.y)<e.size){
    e.hp-=player.damage;
    b.x=-999;
   }
  });
 });

 enemies=enemies.filter(e=>e.hp>0);
 bullets=bullets.filter(b=>b.x>-100);

 if(enemies.length===0){
  if(room<maxRooms){items.push({x:400,y:300,type:"damage"});room++;spawnRoom();}
  else alert("ПОБЕДА!");
 }

 saveGame();
}

/* ---------- DRAW ---------- */
function draw(){
 ctx.clearRect(0,0,800,600);

 ctx.fillStyle="green";
 ctx.beginPath();ctx.arc(player.x,player.y,player.size,0,6.28);ctx.fill();

 ctx.fillStyle="gold";
 bullets.forEach(b=>{ctx.beginPath();ctx.arc(b.x,b.y,5,0,6.28);ctx.fill()});

 enemies.forEach(e=>{
  ctx.fillStyle=e.type==="tank"?"#800":e.type==="shooter"?"#08f":e.type==="rage"?"#f00":"#f44";
  ctx.beginPath();ctx.arc(e.x,e.y,e.size,0,6.28);ctx.fill();
  if(room===maxRooms){
   ctx.fillStyle="#000";ctx.fillRect(250,20,300,15);
   ctx.fillStyle="red";ctx.fillRect(250,20,300*(e.hp/e.maxHp),15);
  }
 });

 ctx.fillStyle="white";
 ctx.fillText("HP "+player.hp+"/"+player.maxHp,20,20);
 ctx.fillText("Room "+room+"/"+maxRooms,20,40);
}

function loop(){update();draw();requestAnimationFrame(loop)}
loop();
</script>
</body>
</html>
