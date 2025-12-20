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
function rnd(){SEED=(SEED*9301+49297)%233280;return SEED/233280;}

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
 items:[],
 invuln:0
};

/* ================= ITEMS ================= */
const ITEM_POOL=[
 {name:"Magic Mushroom",icon:"🍄",apply:()=>{player.maxHp++;player.hp++;player.damage++;player.size+=2}},
 {name:"Third Eye",icon:"👁️",apply:()=>player.items.push({name:"Third Eye",icon:"👁️"})},
 {name:"Burning Tear",icon:"🔥",apply:()=>player.items.push({name:"Burning Tear",icon:"🔥"})},
 {name:"Poison Gland",icon:"☠️",apply:()=>player.items.push({name:"Poison Gland",icon:"☠️"})},
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
 roomX=3;roomY=3;room=1;
 generateMap();
 spawnRoom();
}

function continueGame(){
 const d=localStorage.getItem("save");
 if(!d)return;
 Object.assign(player,JSON.parse(d));
 state="game";
 document.getElementById("menu").style.display="none";
 roomX=parseInt(localStorage.getItem("roomX"))||3;
 roomY=parseInt(localStorage.getItem("roomY"))||3;
 room=parseInt(localStorage.getItem("room"))||1;
 generateMap();
 spawnRoom();
}

/* ================= MAP ================= */
const mapSize=7;
let map=[],visitedMap=[];
let roomX=3,roomY=3,room=1,MAX_ROOMS=6;

function generateMap(){
 map=Array.from({length:mapSize},()=>Array(mapSize).fill(0));
 visitedMap=Array.from({length:mapSize},()=>Array(mapSize).fill(false));
 map[roomY][roomX]=1;visitedMap[roomY][roomX]=true;
}

/* ================= ROOM ================= */
let enemies=[],bullets=[],boss=null;

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
const K={},M={x:0,y:0,down:false};
addEventListener("keydown",e=>K[e.key.toLowerCase()]=true);
addEventListener("keyup",e=>K[e.key.toLowerCase()]=false);
canvas.addEventListener("mousemove",e=>{const r=canvas.getBoundingClientRect();M.x=e.clientX-r.left;M.y=e.clientY-r.top;});
canvas.addEventListener("mousedown",e=>{if(e.button===0)M.down=true;});
canvas.addEventListener("mouseup",e=>{if(e.button===0)M.down=false;});

/* ================= SHOOT ================= */
let shootCooldown=0;
function shoot(dx,dy){
 bullets.push({x:player.x,y:player.y,dx:dx*player.tearSpeed,dy:dy*player.tearSpeed,life:60});
 shootCooldown=12; // уменьшенная скорострельность
}

/* ================= UPDATE ================= */
function update(){
 if(state!=="game")return;

 // Смерть игрока
 if(player.hp<=0){alert("Вы умерли!");state="menu";document.getElementById("menu").style.display="flex";return;}

 // Неуязвимость
 if(player.invuln>0)player.invuln--;

 // Движение игрока
 if(K.w)player.y-=player.speed;
 if(K.s)player.y+=player.speed;
 if(K.a)player.x-=player.speed;
 if(K.d)player.x+=player.speed;

 // Ограничение по комнате
player.x=Math.max(20,Math.min(W-20,player.x));
player.y=Math.max(20,Math.min(H-20,player.y));

 // Стрельба
 if(shootCooldown>0)shootCooldown--;
 if(M.down&&shootCooldown===0){
  let dx=M.x-player.x;
  let dy=M.y-player.y;
  let dist=Math.hypot(dx,dy);
  if(dist>0)shoot(dx/dist,dy/dist);
 }

 // Двигаем врагов к игроку
 enemies.forEach(e=>{
   const dx=player.x-e.x;
   const dy=player.y-e.y;
   const dist=Math.hypot(dx,dy);
   if(dist>0){e.x+=dx/dist*e.speed;e.y+=dy/dist*e.speed;}
   // Урон игроку
   if(dist<player.size+e.size&&player.invuln===0){
     player.hp--;player.invuln=60;
   }
 });

 // Пули
 bullets.forEach(b=>{b.x+=b.dx;b.y+=b.dy;b.life--;});
 bullets.forEach(b=>{
  enemies.forEach(e=>{
   if(Math.hypot(b.x-e.x,b.y-e.y)<e.size){e.hp-=player.damage;b.life=0;}
  });
  if(boss&&Math.hypot(b.x-boss.x,b.y-boss.y)<boss.size){boss.hp-=player.damage;b.life=0;}
 });
 bullets=bullets.filter(b=>b.life>0);
 enemies=enemies.filter(e=>e.hp>0);

 // Если комната пуста, показываем дверь
 if(!boss&&enemies.length===0&&!roomItem){
  roomItem={x:450,y:300,size:16,icon:"🚪",name:"Door"};
 }

 // Вход в дверь
 if(roomItem&&roomItem.name==="Door"&&Math.hypot(player.x-roomItem.x,player.y-roomItem.y)<30){
  if(room<MAX_ROOMS){room++;roomX+=Math.floor(rnd()*3)-1;roomY+=Math.floor(rnd()*3)-1;spawnRoom();}
 }

 // Подбор предметов (если предмет не дверь)
 if(roomItem&&roomItem.name!=="Door"&&Math.hypot(player.x-450,player.y-300)<20){roomItem.apply();player.items.push(roomItem);roomItem=null;}

 // Победа над боссом
 if(boss&&boss.hp<=0){alert("ПОБЕДА!");state="menu";document.getElementById("menu").style.display="flex";}

 save();
}

/* ================= SAVE ================= */
function save(){
 localStorage.setItem("save",JSON.stringify(player));
 localStorage.setItem("roomX",roomX);
 localStorage.setItem("roomY",roomY);
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
 if(boss){ctx.fillStyle="#f00";ctx.beginPath();ctx.arc(boss.x,boss.y,boss.size,0,6.28);ctx.fill();
 ctx.fillStyle="red";ctx.fillRect(300,20,300*(boss.hp/boss.maxHp),12);}

 // Предмет/дверь
 if(roomItem){ctx.fillStyle="white";ctx.font="30px Arial";ctx.fillText(roomItem.icon,roomItem.x-15,roomItem.y+10);}

 // Панель предметов
 ctx.fillStyle="white";ctx.font="20px Arial";ctx.fillText("Items:",20,40);
 player.items.forEach((it,i)=>{ctx.fillText(it.icon,20+i*30,70);});
}

/* ================= GAME LOOP ================= */
function loop(){update();draw();requestAnimationFrame(loop)}
loop();
</script>
</body>
</html>
