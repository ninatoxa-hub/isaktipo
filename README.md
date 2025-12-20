<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Isaac Deluxe Enhanced</title>
<style>
body{margin:0;background:#000;color:#fff;font-family:Arial}
canvas{display:block;margin:auto;background:#111}
#menu{
 position:absolute;inset:0;display:flex;
 flex-direction:column;align-items:center;justify-content:center;
 background:#000;gap:20px
}
button,input{font-size:20px;padding:10px 20px}
#minimap{
 position:absolute;top:10px;right:10px;width:150px;height:150px;
 background:rgba(0,0,0,0.5);border:2px solid #fff;display:grid;
 grid-template-columns:repeat(7,1fr);grid-template-rows:repeat(7,1fr);
}
.minimap-cell{width:20px;height:20px;border:1px solid #333;}
.visited{background:#888;}
.current{background:#0f0;}
.bossroom{background:#f90;}
</style>
</head>
<body>

<div id="menu">
<h1>ISAAC DELUXE ENHANCED</h1>
<input id="seedInput" placeholder="SEED (optional)">
<button onclick="startGame()">NEW GAME</button>
<button onclick="continueGame()">CONTINUE</button>
</div>

<div id="minimap"></div>
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

/* ================= AUDIO ================= */
const audioContext=new (window.AudioContext||window.webkitAudioContext)();
let bgMusic=null;
function playSound(freq,duration=0.2,type='sine',volume=0.3){
 const osc=audioContext.createOscillator();
 const gain=audioContext.createGain();
 osc.connect(gain);gain.connect(audioContext.destination);
 osc.type=type;osc.frequency.setValueAtTime(freq,audioContext.currentTime);
 gain.gain.setValueAtTime(volume,audioContext.currentTime);
 osc.start();osc.stop(audioContext.currentTime+duration);
}

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
 {name:"Magic Mushroom",icon:"🍄",apply:()=>{player.maxHp++;player.hp++;player.damage++;player.size+=2;playSound(440,0.3);}},
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
 updateMinimap();
 playBgMusic();
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
 updateMinimap();
 playBgMusic();
}

/* ================= BACKGROUND MUSIC ================= */
function playBgMusic(){
 if(bgMusic)bgMusic.stop();
 const osc=audioContext.createOscillator();
 const gain=audioContext.createGain();
 osc.type='sine';osc.frequency.setValueAtTime(220,audioContext.currentTime);
 gain.gain.setValueAtTime(0.05,audioContext.currentTime);
 osc.connect(gain);gain.connect(audioContext.destination);
 osc.start();
 bgMusic=osc;
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
let enemies=[],bullets=[],boss=null,doors=[];

function spawnRoom(){
 enemies=[];bullets=[];boss=null;roomItem=null;doors=[];

 if(room===MAX_ROOMS){
  const types=['fire','poison','ice'];
  const t=types[Math.floor(rnd()*types.length)];
  boss={x:450,y:300,hp:60,maxHp:60,size:36,type:t};
  playSound(100,0.5,'triangle');
 }else{
  const enemyTypes=['fast','slow','tank'];
  const count=Math.floor(rnd()*4)+2;
  for(let i=0;i<count;i++){
   const t=enemyTypes[Math.floor(rnd()*enemyTypes.length)];
   enemies.push({x:rnd()*800+50,y:rnd()*500+50,hp:t==='tank'?6:4,size:t==='tank'?22:18,speed:t==='fast'?2:t==='slow'?1:1.5,type:t});
  }
 }
}

/* ================= MINIMAP ================= */
const minimap=document.getElementById("minimap");
function updateMinimap(){
 minimap.innerHTML="";
 for(let y=0;y<mapSize;y++){
  for(let x=0;x<mapSize;x++){
   const div=document.createElement("div");div.className="minimap-cell";
   if(visitedMap[y][x])div.classList.add("visited");
   if(x===roomX&&y===roomY)div.classList.add("current");
   if(room===MAX_ROOMS)div.classList.add("bossroom");
   minimap.appendChild(div);
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
 let tearType="normal";
 player.items.forEach(it=>{
   if(it.name==="Burning Tear") tearType="fire";
   if(it.name==="Poison Gland") tearType="poison";
 });
 bullets.push({x:player.x,y:player.y,dx:dx*player.tearSpeed,dy:dy*player.tearSpeed,life:60,type:tearType});
 shootCooldown=15;
 playSound(800,0.1,'sine',0.2);
}

/* ================= UPDATE ================= */
function update(){
 if(state!=="game")return;

 if(player.hp<=0){alert("Вы умерли!");state="menu";document.getElementById("menu").style.display="flex";return;}

 if(player.invuln>0)player.invuln--;

 if(K.w)player.y-=player.speed;
 if(K.s)player.y+=player.speed;
 if(K.a)player.x-=player.speed;
 if(K.d)player.x+=player.speed;

player.x=Math.max(20,Math.min(W-20,player.x));
player.y=Math.max(20,Math.min(H-20,player.y));

 if(shootCooldown>0)shootCooldown--;
 if(M.down&&shootCooldown===0){
  let dx=M.x-player.x;
  let dy=M.y-player.y;
  let dist=Math.hypot(dx,dy);
  if(dist>0)shoot(dx/dist,dy/dist);
 }

 enemies.forEach(e=>{
   const dx=player.x-e.x;
   const dy=player.y-e.y;
   const dist=Math.hypot(dx,dy);
   if(dist>0){e.x+=dx/dist*e.speed;e.y+=dy/dist*e.speed;}
   if(dist<player.size+e.size&&player.invuln===0){
     player.hp--;player.invuln=60;
     playSound(150,0.2,'sawtooth');
   }
 });

 bullets.forEach(b=>{b.x+=b.dx;b.y+=b.dy;b.life--;});
 bullets.forEach(b=>{
  enemies.forEach(e=>{
   if(Math.hypot(b.x-e.x,b.y-e.y)<e.size){e.hp-=player.damage;b.life=0;playSound(600,0.1);}});
  if(boss&&Math.hypot(b.x-boss.x,b.y-boss.y)<boss.size){boss.hp-=player.damage;b.life=0;playSound(400,0.2);}});
 bullets=bullets.filter(b=>b.life>0);
 enemies=enemies.filter(e=>e.hp>0);

 if(!boss&&enemies.length===0&&!roomItem){
  roomItem={x:450,y:300,size:16,icon:"🚪",name:"Door"};
 }

 if(roomItem&&roomItem.name==="Door"&&Math.hypot(player.x-roomItem.x,player.y-roomItem.y)<30){
  if(room<MAX_ROOMS){room++;roomX+=Math.floor(rnd()*3)-1;roomY+=Math.floor(rnd()*3)-1;spawnRoom();visitedMap[roomY][roomX]=true;updateMinimap();}
 }

 save();

 if(boss&&boss.hp<=0){alert("ПОБЕДА! Вы победили босса!");state="menu";document.getElementById("menu").style.display="flex";}
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

 // player
 ctx.fillStyle="#4CAF50";ctx.beginPath();ctx.arc(player.x,player.y,player.size,0,6.28);ctx.fill();

 // tears
 bullets.forEach(b=>{
   if(b.type==="fire")ctx.fillStyle="orange";
   else if(b.type==="poison")ctx.fillStyle="green";
   else ctx.fillStyle="gold";
   ctx.beginPath();ctx.arc(b.x,b.y,4,0,6.28);ctx.fill();
 });

 // enemies
 ctx.fillStyle="#f44";enemies.forEach(e=>{ctx.beginPath();ctx.arc(e.x,e.y,e.size,0,6.28);ctx.fill();});

 // boss
 if(boss){
  ctx.fillStyle="#f00";ctx.beginPath();ctx.arc(boss.x,boss.y,boss.size,0,6.28);ctx.fill();
  ctx.fillStyle="red";ctx.fillRect(300,20,300*(boss.hp/boss.maxHp),12);
 }

 // room item
 if(roomItem){ctx.fillStyle="white";ctx.font="30px Arial";ctx.fillText(roomItem.icon,roomItem.x-15,roomItem.y+10);}
 ctx.fillStyle="white";ctx.font="20px Arial";ctx.fillText("Items:",20,40);
 player.items.forEach((it,i)=>{ctx.fillText(it.icon,20+i*30,70);});
}

/* ================= GAME LOOP ================= */
function loop(){update();draw();requestAnimationFrame(loop)}
loop();
</script>
</body>
</html>
