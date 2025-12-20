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
<canvas id="game" width="900" height="600"></canvas>
<script>
const c=document.getElementById("game"),ctx=c.getContext("2d");

/* ================= AUDIO ================= */
const AC=new (window.AudioContext||window.webkitAudioContext)();
function sound(f,t=0.12){
 let o=AC.createOscillator(),g=AC.createGain();
 o.frequency.value=f;o.connect(g);g.connect(AC.destination);
 o.start();g.gain.exponentialRampToValueAtTime(0.0001,AC.currentTime+t);
}
setInterval(()=>sound(110,0.05),2500);

/* ================= INPUT ================= */
const K={};
addEventListener("keydown",e=>K[e.key.toLowerCase()]=true);
addEventListener("keyup",e=>K[e.key.toLowerCase()]=false);

/* ================= SAVE ================= */
function save(){localStorage.setItem("isaac_save",JSON.stringify(player))}
function load(){
 let s=localStorage.getItem("isaac_save");
 if(s) Object.assign(player,JSON.parse(s));
}

/* ================= PLAYER ================= */
const player={
 x:450,y:300,size:16,
 hp:6,maxHp:6,
 speed:2,damage:1,
 tears:1,tearSpeed:6,
 mods:{multi:0,split:0,fire:0,poison:0,slow:0,crit:0},
 inv:0,room:1
};
load();

/* ================= GAME ================= */
const MAX_ROOMS=6;
let bullets=[],enemies=[],items=[],boss=null;

/* ================= BULLET ================= */
function shoot(dx,dy){
 bullets.push({
  x:player.x,y:player.y,
  dx:dx*player.tearSpeed,dy:dy*player.tearSpeed,
  life:80,mods:{...player.mods}
 });
 sound(600);
}

/* ================= ENEMY ================= */
function spawnEnemy(type,x,y){
 let e={x,y,type,hp:5,size:18,speed:1,cd:0};
 if(type==="fast"){e.speed=1.6;e.hp=3}
 if(type==="tank"){e.speed=0.6;e.hp=10;e.size=24}
 enemies.push(e);
}

/* ================= BOSSES ================= */
function spawnBoss(){
 const t=["tank","shooter","rage","split"][Math.floor(Math.random()*4)];
 boss={type:t,x:450,y:300,hp:50,maxHp:50,size:35,speed:0.7,cd:0};
}

/* ================= ROOM ================= */
function spawnRoom(){
 bullets=[];enemies=[];items=[];boss=null;
 if(player.room===MAX_ROOMS) spawnBoss();
 else{
  for(let i=0;i<5;i++){
   const t=Math.random()<0.3?"fast":Math.random()<0.2?"tank":"normal";
   spawnEnemy(t,Math.random()*800+50,Math.random()*500+50);
  }
 }
}
spawnRoom();

/* ================= MODIFIERS ================= */
function giveRandomMod(){
 const m=Object.keys(player.mods);
 player.mods[m[Math.floor(Math.random()*m.length)]]++;
}

/* ================= UPDATE ================= */
function update(){
 if(player.inv>0)player.inv--;

 /* movement */
 if(K.w)player.y-=player.speed;
 if(K.s)player.y+=player.speed;
 if(K.a)player.x-=player.speed;
 if(K.d)player.x+=player.speed;

 /* shooting */
 if(K.arrowup)shoot(0,-1);
 if(K.arrowdown)shoot(0,1);
 if(K.arrowleft)shoot(-1,0);
 if(K.arrowright)shoot(1,0);

 /* cheats */
 if(K.z){player.room=MAX_ROOMS;spawnRoom();K.z=false}
 if(K.x){for(let i=0;i<100;i++)giveRandomMod();K.x=false}

 /* bullets */
 bullets.forEach(b=>{
  b.x+=b.dx;b.y+=b.dy;b.life--;
  enemies.forEach(e=>{
   if(Math.hypot(b.x-e.x,b.y-e.y)<e.size){
    e.hp-=player.damage;b.life=0;
   }
  });
  if(boss && Math.hypot(b.x-boss.x,b.y-boss.y)<boss.size){
   boss.hp-=player.damage;b.life=0;
  }
 });
 bullets=bullets.filter(b=>b.life>0);

 /* enemies */
 enemies.forEach(e=>{
  let dx=player.x-e.x,dy=player.y-e.y,d=Math.hypot(dx,dy);
  e.x+=dx/d*e.speed;e.y+=dy/d*e.speed;
  if(d<player.size+e.size&&player.inv===0){
   player.hp--;player.inv=60;sound(200);
  }
 });
 enemies=enemies.filter(e=>e.hp>0);

 /* boss */
 if(boss){
  let dx=player.x-boss.x,dy=player.y-boss.y,d=Math.hypot(dx,dy);
  boss.x+=dx/d*boss.speed;boss.y+=dy/d*boss.speed;
  if(boss.hp<boss.maxHp/2&&boss.type==="rage")boss.speed=1.5;
 }

 /* room clear */
 if(!boss && enemies.length===0){
  items.push({x:450,y:300});
 }
 if(boss && boss.hp<=0){
  alert("ПОБЕДА");
 }

 save();
}

/* ================= DRAW ================= */
function draw(){
 ctx.clearRect(0,0,900,600);

 /* player */
 ctx.fillStyle="#4CAF50";
 ctx.beginPath();ctx.arc(player.x,player.y,player.size,0,6.28);ctx.fill();

 /* bullets */
 ctx.fillStyle="gold";
 bullets.forEach(b=>{ctx.beginPath();ctx.arc(b.x,b.y,4,0,6.28);ctx.fill()});

 /* enemies */
 enemies.forEach(e=>{
  ctx.fillStyle=e.type==="tank"?"#800":e.type==="fast"?"#fa0":"#f44";
  ctx.beginPath();ctx.arc(e.x,e.y,e.size,0,6.28);ctx.fill();
 });

 /* boss */
 if(boss){
  ctx.fillStyle="#f00";
  ctx.beginPath();ctx.arc(boss.x,boss.y,boss.size,0,6.28);ctx.fill();
  ctx.fillStyle="#000";ctx.fillRect(300,20,300,16);
  ctx.fillStyle="red";ctx.fillRect(300,20,300*(boss.hp/boss.maxHp),16);
 }

 /* UI */
 ctx.fillStyle="white";
 ctx.fillText("HP "+player.hp+"/"+player.maxHp,20,20);
 ctx.fillText("Room "+player.room+"/"+MAX_ROOMS,20,40);
 ctx.fillText("Mods "+JSON.stringify(player.mods),20,60);
}

function loop(){update();draw();requestAnimationFrame(loop)}
loop();
</script>
</body>
</html>
