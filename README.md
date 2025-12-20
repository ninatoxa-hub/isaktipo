<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Isaac Deluxe</title>
<style>
body{margin:0;background:#000;color:#fff;font-family:Arial}
canvas{display:block;margin:auto;background:#111}
#menu{
 position:absolute;inset:0;display:flex;
 flex-direction:column;align-items:center;justify-content:center;
 background:#000;gap:20px
}
button,input{font-size:20px;padding:10px 20px}
</style>
</head>
<body>

<div id="menu">
<h1>ISAAC DELUXE</h1>
<input id="seedInput" placeholder="SEED (optional)">
<button onclick="startGame()">START</button>
<button onclick="continueGame()">CONTINUE</button>
</div>

<canvas id="game" width="900" height="600"></canvas>

<script>
/* ================= RNG (SEED) ================= */
let SEED=Date.now();
function rnd(){
 SEED=(SEED*9301+49297)%233280;
 return SEED/233280;
}

/* ================= CANVAS ================= */
const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");
const W=canvas.width,H=canvas.height;

/* ================= STATE ================= */
let state="menu";

/* ================= PLAYER ================= */
const player={
 x:450,y:300,size:14,
 hp:6,maxHp:6,
 speed:2.2,damage:1,
 tearSpeed:6,
 mods:{multi:0,fire:0,poison:0},
 items:[]
};

/* ================= ITEMS ================= */
const ITEM_POOL=[
 {name:"Magic Mushroom",icon:"🍄",apply:()=>{player.maxHp++;player.hp++;player.damage++;player.size+=2}},
 {name:"Third Eye",icon:"👁️",apply:()=>player.mods.multi++},
 {name:"Burning Tear",icon:"🔥",apply:()=>player.mods.fire++},
 {name:"Poison Gland",icon:"☠️",apply:()=>player.mods.poison++},
 {name:"Wire Coat Hanger",icon:"⚡",apply:()=>player.tearSpeed++}
];

let roomItem=null;

/* ================= MENU ================= */
function startGame(){
 const s=document.getElementById("seedInput").value;
 SEED=s?parseInt(s):Date.now();
 localStorage.removeItem("save");
 state="game";
 document.getElementById("menu").style.display="none";
 room=1;
 spawnRoom();
}

function continueGame(){
 const d=localStorage.getItem("save");
 if(!d)return;
 Object.assign(player,JSON.parse(d));
 state="game";
 document.getElementById("menu").style.display="none";
 room=JSON.parse(localStorage.getItem("room"))||1;
 spawnRoom();
}

/* ================= ROOM ================= */
let enemies=[],bullets=[],boss=null;
let room=1,MAX_ROOMS=6;

function spawnRoom(){
 enemies=[];bullets=[];boss=null;roomItem=null;

 if(room===MAX_ROOMS){
  boss={x:450,y:300,hp:60,maxHp:60,size:36};
 }else{
  const types=['fast','slow','tank'];
  for(let i=0;i<4;i++){
   const t=types[Math.floor(rnd()*types.length)];
   enemies.push({x:rnd()*800+50,y:rnd()*500+50,hp:t==='tank'?6:4,size:t==='tank'?22:18,speed:t==='fast'?2:t==='slow'?1:1.5});
  }
 }
}

/* ================= INPUT ================= */
const K={};
addEventListener("keydown",e=>K[e.key.toLowerCase()]=true);
addEventListener("keyup",e=>K[e.key.toLowerCase()]=false);

/* ================= SHOOT ================= */
function shoot(dx,dy){
 let count=1+player.mods.multi;
 for(let i=0;i<count;i++)
  bullets.push({x:player.x,y:player.y,dx:dx*player.tearSpeed,dy:dy*player.tearSpeed,life:60});
}

/* ================= UPDATE ================= */
function update(){
 if(state!=="game")return;

 // Движение игрока
 if(K.w)player.y-=player.speed;
 if(K.s)player.y+=player.speed;
 if(K.a)player.x-=player.speed;
 if(K.d)player.x+=player.speed;

 // Чит-коды
 if(K.z){room=MAX_ROOMS;spawnRoom();K.z=false;}
 if(K.x){player.damage+=100;player.items.push({name:'Cheat',icon:'💥'});K.x=false;}

 // Двигаем врагов к игроку
 enemies.forEach(e=>{
   const dx = player.x - e.x;
   const dy = player.y - e.y;
   const dist = Math.hypot(dx, dy);
   if(dist>0){
     e.x += dx/dist * e.speed;
     e.y += dy/dist * e.speed;
   }
 });

 // Проверка попаданий пуль по врагам и боссу
 bullets.forEach(b=>{
  b.x+=b.dx;b.y+=b.dy;b.life--;
  enemies.forEach(e=>{
   if(Math.hypot(b.x-e.x,b.y-e.y)<e.size){
    e.hp-=player.damage;b.life=0;
   }
  });
  if(boss&&Math.hypot(b.x-boss.x,b.y-boss.y)<boss.size){
   boss.hp-=player.damage;b.life=0;
  }
 });
 bullets=bullets.filter(b=>b.life>0);
 enemies=enemies.filter(e=>e.hp>0);

 // Спавн предмета, если комната очищена
 if(!boss&&enemies.length===0&&!roomItem){
  roomItem=ITEM_POOL[Math.floor(rnd()*ITEM_POOL.length)];
 }

 // Подбор предмета
 if(roomItem&&Math.hypot(player.x-450,player.y-300)<20){
  roomItem.apply();
  player.items.push(roomItem);
  room++;
  spawnRoom();
 }

 // Победа над боссом
 if(boss&&boss.hp<=0){alert("ПОБЕДА!");state="menu";location.reload()}

 save();
}

/* ================= SAVE ================= */
function save(){
 localStorage.setItem("save",JSON.stringify(player));
 localStorage.setItem("room",room);
}

/* ================= DRAW ================= */
function draw(){
 ctx.clearRect(0,0,W,H);

 if(state!=="game")return;

 // Игрок
 ctx.fillStyle="#4CAF50";
 ctx.beginPath();ctx.arc(player.x,player.y,player.size,0,6.28);ctx.fill();

 // Пули
 ctx.fillStyle="gold";
 bullets.forEach(b=>{ctx.beginPath();ctx.arc(b.x,b.y,4,0,6.28);ctx.fill()});

 // Враги
 ctx.fillStyle="#f44";
 enemies.forEach(e=>{ctx.beginPath();ctx.arc(e.x,e.y,e.size,0,6.28);ctx.fill()});

 // Босс
 if(boss){
  ctx.fillStyle="#f00";
  ctx.beginPath();ctx.arc(boss.x,boss.y,boss.size,0,6.28);ctx.fill();
  // HP бар босса
  ctx.fillStyle="red";
  ctx.fillRect(300,20,300*(boss.hp/boss.maxHp),12);
 }

 // Предмет
 if(roomItem){
  ctx.fillStyle="white";
  ctx.font="30px Arial";
  ctx.fillText(roomItem.icon,440,300);
 }

 // Панель предметов
 ctx.fillStyle="white";
 ctx.font="20px Arial";
 ctx.fillText("Items:",20,40);
 player.items.forEach((it,i)=>{
  ctx.fillText(it.icon,20+i*30,70);
 });
}

/* ================= GAME LOOP ================= */
function loop(){update();draw();requestAnimationFrame(loop)}
loop();
</script>
</body>
</html>
