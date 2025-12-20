<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Isaac Enhanced Rooms</title>
<style>
body{margin:0;background:#000;color:#fff;font-family:Arial;}
canvas{display:block;margin:auto;background:#111;}
#minimap{position:absolute;top:10px;right:10px;width:150px;height:150px;background:rgba(0,0,0,0.5);border:2px solid #fff;display:grid;grid-template-columns:repeat(7,1fr);grid-template-rows:repeat(7,1fr);}
.minimap-cell{width:20px;height:20px;border:1px solid #333;}
.visited{background:#888;}
.current{background:#0f0;}
.bossroom{background:#f90;}
</style>
</head>
<body>
<canvas id="game" width="900" height="600"></canvas>
<div id="minimap"></div>

<script>
const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");
const W=canvas.width,H=canvas.height;

// ====== PLAYER ======
const player={x:450,y:300,size:14,hp:6,maxHp:6,speed:2.2,damage:1,tearSpeed:6,items:[],invuln:0};

// ====== INPUT ======
const K={},M={x:0,y:0,down:false};
addEventListener("keydown",e=>K[e.key.toLowerCase()]=true);
addEventListener("keyup",e=>K[e.key.toLowerCase()]=false);
canvas.addEventListener("mousemove",e=>{const r=canvas.getBoundingClientRect();M.x=e.clientX-r.left;M.y=e.clientY-r.top;});
canvas.addEventListener("mousedown",e=>{if(e.button===0)M.down=true;});
canvas.addEventListener("mouseup",e=>{if(e.button===0)M.down=false;});

// ====== MAP ======
const mapSize=7;
let map=[],visitedMap=[];
let roomX=3,roomY=3,room=1,MAX_ROOMS=6;

function generateMap(){
 map=Array.from({length:mapSize},()=>Array(mapSize).fill(0));
 visitedMap=Array.from({length:mapSize},()=>Array(mapSize).fill(false));
 map[roomY][roomX]=1;visitedMap[roomY][roomX]=true;
}

// ====== MINIMAP ======
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

// ====== ITEMS ======
const ITEM_POOL=[
 {name:"Magic Mushroom",icon:"🍄",apply:()=>{player.maxHp++;player.hp++;player.damage++;player.size+=2;}},
 {name:"Wire Coat Hanger",icon:"⚡",apply:()=>player.tearSpeed++}
];
let roomItem=null;

// ====== BULLETS ======
let bullets=[];

// ====== ENEMIES ======
let enemies=[],boss=null;

// ====== DOORS ======
let doors=[];

// ====== SPAWN ROOM ======
function spawnRoom(){
 enemies=[];bullets=[];boss=null;roomItem=null;doors=[];
 // случайные враги
 if(room===MAX_ROOMS){
  boss={x:450,y:300,hp:60,maxHp:60,size:36,speed:1,dirX:1,dirY:1};
 }else{
  const count=2+Math.floor(Math.random()*4);
  for(let i=0;i<count;i++){
   enemies.push({x:Math.random()*800+50,y:Math.random()*500+50,hp:4,size:18,speed:1.5});
  }
  // предмет
  if(Math.random()<0.5){
   roomItem={x:Math.random()*700+100,y:Math.random()*500+50,size:16,icon:ITEM_POOL[Math.floor(Math.random()*ITEM_POOL.length)].icon};
  }
}

// создаем двери
 doors=[];
 if(roomY>0) doors.push({x:W/2,y:10,width:60,height:20,dir:'up'});
 if(roomY<mapSize-1) doors.push({x:W/2,y:H-10-20,width:60,height:20,dir:'down'});
 if(roomX>0) doors.push({x:10,y:H/2,width:20,height:60,dir:'left'});
 if(roomX<mapSize-1) doors.push({x:W-10-20,y:H/2,width:20,height:60,dir:'right'});
 visitedMap[roomY][roomX]=true;
 updateMinimap();
}

// ====== SHOOT ======
let shootCooldown=0;
function shoot(dx,dy){bullets.push({x:player.x,y:player.y,dx:dx*player.tearSpeed,dy:dy*player.tearSpeed,life:60});shootCooldown=15;}

// ====== UPDATE ======
function update(){
 if(player.hp<=0){alert("Вы умерли!");location.reload();}
 if(player.invuln>0)player.invuln--;

 if(K.w)player.y-=player.speed;
 if(K.s)player.y+=player.speed;
 if(K.a)player.x-=player.speed;
 if(K.d)player.x+=player.speed;
 player.x=Math.max(20,Math.min(W-20,player.x));
 player.y=Math.max(20,Math.min(H-20,player.y));

 if(shootCooldown>0)shootCooldown--;
 if(M.down&&shootCooldown===0){
  let dx=M.x-player.x,dy=M.y-player.y,dist=Math.hypot(dx,dy);
  if(dist>0)shoot(dx/dist,dy/dist);
 }

 // обновление пуль
 bullets.forEach(b=>{b.x+=b.dx;b.y+=b.dy;b.life--;});
 bullets=bullets.filter(b=>b.life>0);

 // обновление врагов
 enemies.forEach(e=>{
   const dx=player.x-e.x,dy=player.y-e.y,dist=Math.hypot(dx,dy);
   if(dist>0){e.x+=dx/dist*e.speed;e.y+=dy/dist*e.speed;}
   if(dist<player.size+e.size&&player.invuln===0){player.hp--;player.invuln=60;}
 });

 // попадания пуль
 bullets.forEach(b=>{
   enemies.forEach(e=>{
     if(Math.hypot(b.x-e.x,b.y-e.y)<e.size){e.hp-=player.damage;b.life=0;}
   });
   if(boss&&Math.hypot(b.x-boss.x,b.y-boss.y)<boss.size){boss.hp-=player.damage;b.life=0;}
 });
 enemies=enemies.filter(e=>e.hp>0);

 // движение босса
 if(boss){
  boss.x+=boss.dirX*boss.speed;
  boss.y+=boss.dirY*boss.speed;
  if(boss.x<boss.size||boss.x>W-boss.size)boss.dirX*=-1;
  if(boss.y<boss.size||boss.y>H-boss.size)boss.dirY*=-1;
 }

 // подбор предмета
 if(roomItem&&Math.hypot(player.x-roomItem.x,player.y-roomItem.y)<player.size+roomItem.size){
  player.items.push(roomItem);roomItem=null;
 }

 // двери
 doors.forEach(d=>{
   if(Math.abs(player.x-(d.x+d.width/2))<player.size+d.width/2 && Math.abs(player.y-(d.y+d.height/2))<player.size+d.height/2){
     if(enemies.length===0){ // только если комната очищена
       if(d.dir==='up')roomY--; else if(d.dir==='down')roomY++; else if(d.dir==='left')roomX--; else if(d.dir==='right')roomX++;
       room++;spawnRoom();
     }
   }
 });
}

// ====== DRAW ======
function draw(){
 ctx.clearRect(0,0,W,H);
 // player
 ctx.fillStyle="#4CAF50";ctx.beginPath();ctx.arc(player.x,player.y,player.size,0,6.28);ctx.fill();
 // bullets
 ctx.fillStyle="gold";bullets.forEach(b=>{ctx.beginPath();ctx.arc(b.x,b.y,4,0,6.28);ctx.fill();});
 // enemies
 ctx.fillStyle="#f44";enemies.forEach(e=>{ctx.beginPath();ctx.arc(e.x,e.y,e.size,0,6.28);ctx.fill();});
 // boss
 if(boss){ctx.fillStyle="#f00";ctx.beginPath();ctx.arc(boss.x,boss.y,boss.size,0,6.28);ctx.fill();ctx.fillStyle="red";ctx.fillRect(300,20,300*(boss.hp/boss.maxHp),12);}
 // room item
 if(roomItem){ctx.fillStyle="white";ctx.font="30px Arial";ctx.fillText(roomItem.icon,roomItem.x-15,roomItem.y+10);}
 // doors
 ctx.fillStyle="#888";doors.forEach(d=>{ctx.fillRect(d.x,d.y,d.width,d.height);});
 // items
 ctx.fillStyle="white";ctx.font="20px Arial";ctx.fillText("Items:",20,40);
 player.items.forEach((it,i)=>{ctx.fillText(it.icon,20+i*30,70);});
}

// ====== GAME LOOP ======
function loop(){update();draw();requestAnimationFrame(loop);}
generateMap();spawnRoom();loop();
</script>
</body>
</html>
