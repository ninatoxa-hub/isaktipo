<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Roguelike Dungeon</title>
<style>
body{margin:0;padding:0;background:#111;color:white;font-family:Arial;}
canvas{display:block;margin:auto;background:#1a1a1a;}
#minimap{position:absolute;top:10px;right:10px;width:210px;height:210px;display:grid;grid-template-columns:repeat(7,1fr);grid-template-rows:repeat(7,1fr);background:rgba(0,0,0,0.5);border:2px solid #fff;}
.minimap-cell{border:1px solid #333;}
.visited{background:#888;}
.current{background:#0f0;}
.bossroom{background:#f90;}
</style>
</head>
<body>
<canvas id="game" width="800" height="600"></canvas>
<div id="minimap"></div>

<script>
const canvas=document.getElementById('game');
const ctx=canvas.getContext('2d');
const minimap=document.getElementById('minimap');

let keys={};
window.addEventListener('keydown',e=>keys[e.key.toLowerCase()]=true);
window.addEventListener('keyup',e=>keys[e.key.toLowerCase()]=false);

let mouse={x:400,y:300,down:false};
canvas.addEventListener('mousemove',e=>{const r=canvas.getBoundingClientRect();mouse.x=e.clientX-r.left;mouse.y=e.clientY-r.top;});
canvas.addEventListener('mousedown',e=>{if(e.button===0)mouse.down=true;});
canvas.addEventListener('mouseup',e=>{if(e.button===0)mouse.down=false;});

// ======= Игрок =======
let player={x:400,y:300,size:20,hp:6,maxHp:6,speed:3,damage:1,invuln:0,items:[],dirX:0,dirY:-1};

// ======= Карта =======
const mapSize=7;
let map=[],visitedMap=[];
let roomX=3,roomY=3,room=1,MAX_ROOMS=6;
function generateMap(){map=Array.from({length:mapSize},()=>Array(mapSize).fill(0));visitedMap=Array.from({length:mapSize},()=>Array(mapSize).fill(false));map[roomY][roomX]=1;visitedMap[roomY][roomX]=true;}

// ======= Миникарта =======
function updateMinimap(){
 minimap.innerHTML='';
 for(let y=0;y<mapSize;y++){
   for(let x=0;x<mapSize;x++){
     const div=document.createElement('div');div.className='minimap-cell';
     if(visitedMap[y][x])div.classList.add('visited');
     if(x===roomX&&y===roomY)div.classList.add('current');
     if(room===MAX_ROOMS)div.classList.add('bossroom');
     minimap.appendChild(div);
   }
 }
}

// ======= Пули =======
let bullets=[];

// ======= Враги и Босс =======
let enemies=[],boss=null;
const BOSSES=[
 {name:"Red Boss",hp:60,size:36,speed:1.2,color:"#f00",pattern:"bounce"},
 {name:"Blue Boss",hp:80,size:40,speed:0.8,color:"#00f",pattern:"zigzag"},
 {name:"Green Boss",hp:100,size:50,speed:1,color:"#0f0",pattern:"circle"}
];

// ======= Предметы =======
const ITEM_POOL=[
 {name:"Magic Mushroom",icon:"🍄",apply:()=>{player.maxHp++;player.hp++;player.damage++;player.size+=2;}},
 {name:"Wire Coat Hanger",icon:"⚡",apply:()=>player.damage++}
];
let roomItem=null;

// ======= Двери =======
let doors=[];

// ======= Spawn Room =======
function spawnRoom(){
 enemies=[];bullets=[];boss=null;roomItem=null;doors=[];
 if(room===MAX_ROOMS){
   const b=BOSSES[Math.floor(Math.random()*BOSSES.length)];
   boss={...b,x:canvas.width/2,y:canvas.height/2,dirX:1,dirY:1,angle:0,maxHp:b.hp};
 }else{
   const types=[{hp:3,size:16,speed:1.2},{hp:4,size:18,speed:1.5},{hp:5,size:20,speed:0.8}];
   const count=2+Math.floor(Math.random()*4);
   for(let i=0;i<count;i++){
     const t=types[Math.floor(Math.random()*types.length)];
     enemies.push({x:Math.random()*canvas.width*0.8+canvas.width*0.1,y:Math.random()*canvas.height*0.8+canvas.height*0.1,...t});
   }
   if(Math.random()<0.5){
     const it=ITEM_POOL[Math.floor(Math.random()*ITEM_POOL.length)];
     roomItem={x:Math.random()*canvas.width*0.7+canvas.width*0.15,y:Math.random()*canvas.height*0.7+canvas.height*0.15,size:16,icon:it.icon,apply:it.apply};
   }
 }
 // двери
 doors=[];
 if(roomY>0) doors.push({x:canvas.width/2,y:10,width:60,height:20,dir:'up'});
 if(roomY<mapSize-1) doors.push({x:canvas.width/2,y:canvas.height-30,width:60,height:20,dir:'down'});
 if(roomX>0) doors.push({x:10,y:canvas.height/2,width:20,height:60,dir:'left'});
 if(roomX<mapSize-1) doors.push({x:canvas.width-30,y:canvas.height/2,width:20,height:60,dir:'right'});
 visitedMap[roomY][roomX]=true;
 updateMinimap();
}

// ======= Стрельба =======
let shootCooldown=0;
function shoot(dx,dy){bullets.push({x:player.x,y:player.y,dx:dx*6,dy:dy*6,life:60});shootCooldown=15;}

// ======= Update =======
function update(){
 if(player.hp<=0){alert("Вы умерли!");location.reload();}
 if(player.invuln>0)player.invuln--;

 // движение
 if(keys['w']){player.y-=player.speed;player.dirX=0;player.dirY=-1;}
 if(keys['s']){player.y+=player.speed;player.dirX=0;player.dirY=1;}
 if(keys['a']){player.x-=player.speed;player.dirX=-1;player.dirY=0;}
 if(keys['d']){player.x+=player.speed;player.dirX=1;player.dirY=0;}
 player.x=Math.max(20,Math.min(canvas.width-20,player.x));
 player.y=Math.max(20,Math.min(canvas.height-20,player.y));

 if(shootCooldown>0)shootCooldown--;
 if(mouse.down&&shootCooldown===0){
   let dx=mouse.x-player.x,dy=mouse.y-player.y,dist=Math.hypot(dx,dy);
   if(dist>0)shoot(dx/dist,dy/dist);
 }

 // update bullets
 bullets.forEach(b=>{b.x+=b.dx;b.y+=b.dy;b.life--;});
 bullets=bullets.filter(b=>b.life>0);

 // enemies
 enemies.forEach(e=>{
   const dx=player.x-e.x,dy=player.y-e.y,dist=Math.hypot(dx,dy);
   if(dist>0){e.x+=dx/dist*e.speed;e.y+=dy/dist*e.speed;}
   if(dist<player.size+e.size&&player.invuln===0){player.hp--;player.invuln=60;}
 });
 enemies=enemies.filter(e=>e.hp>0);

 // bullets hit enemies
 bullets.forEach(b=>{
   enemies.forEach(e=>{if(Math.hypot(b.x-e.x,b.y-e.y)<e.size){e.hp-=player.damage;b.life=0;}});
   if(boss&&Math.hypot(b.x-boss.x,b.y-boss.y)<boss.size){boss.hp-=player.damage;b.life=0;}
 });

 // boss movement
 if(boss){
   if(boss.pattern==="bounce"){
     boss.x+=boss.dirX*boss.speed;boss.y+=boss.dirY*boss.speed;
     if(boss.x<boss.size||boss.x>canvas.width-boss.size)boss.dirX*=-1;
     if(boss.y<boss.size||boss.y>canvas.height-boss.size)boss.dirY*=-1;
   } else if(boss.pattern==="zigzag"){
     boss.x+=boss.speed*Math.cos(boss.angle);boss.y+=boss.speed*Math.sin(boss.angle);boss.angle+=0.05;
     if(boss.x<boss.size||boss.x>canvas.width-boss.size)boss.angle+=Math.PI;
     if(boss.y<boss.size||boss.y>canvas.height-boss.size)boss.angle+=Math.PI;
   } else if(boss.pattern==="circle"){
     boss.angle+=0.03;boss.x=canvas.width/2+100*Math.cos(boss.angle);boss.y=canvas.height/2+100*Math.sin(boss.angle);
   }
   if(boss.hp<=0){alert("Босс побежден! Игра окончена.");location.reload();}
 }

 // pick up item
 if(roomItem&&Math.hypot(player.x-roomItem.x,player.y-roomItem.y)<player.size+roomItem.size){
   if(roomItem.apply)roomItem.apply();player.items.push(roomItem);roomItem=null;
 }

 // doors
 doors.forEach(d=>{
   if(Math.abs(player.x-(d.x+d.width/2))<player.size+d.width/2 && Math.abs(player.y-(d.y+d.height/2))<player.size+d.height/2){
     if(enemies.length===0){
       if(d.dir==='up')roomY--; else if(d.dir==='down')roomY++; else if(d.dir==='left')roomX--; else if(d.dir==='right')roomX++;
       room++;spawnRoom();
     }
   }
 });

 // cheat codes
 if(keys['z']){roomX=3;roomY=3;room=MAX_ROOMS;spawnRoom();keys['z']=false;}
 if(keys['x']){player.damage+=100;for(let i=0;i<100;i++){player.items.push({icon:"⭐"});}keys['x']=false;}
}

// ======= Draw =======
function draw(){
 ctx.clearRect(0,0,canvas.width,canvas.height);
 ctx.fillStyle="#4CAF50";ctx.beginPath();ctx.arc(player.x,player.y,player.size,0,6.28);ctx.fill();
 ctx.fillStyle="gold";bullets.forEach(b=>{ctx.beginPath();ctx.arc(b.x,b.y,4,0,6.28);ctx.fill();});
 ctx.fillStyle="#f44";enemies.forEach(e=>{ctx.beginPath();ctx.arc(e.x,e.y,e.size,0,6.28);ctx.fill();});
 if(boss){ctx.fillStyle=boss.color;ctx.beginPath();ctx.arc(boss.x,boss.y,boss.size,0,6.28);ctx.fill();ctx.fillStyle="red";ctx.fillRect(50,20,300*(boss.hp/boss.maxHp),12);}
 if(roomItem){ctx.fillStyle="white";ctx.font="30px Arial";ctx.fillText(roomItem.icon,roomItem.x-15,roomItem.y+10);}
 ctx.fillStyle="#888";doors.forEach(d=>{ctx.fillRect(d.x,d.y,d.width,d.height);});
 ctx.fillStyle="white";ctx.font="20px Arial";ctx.fillText("Items:",20,40);
 player.items.forEach((it,i)=>{ctx.fillText(it.icon,20+i*30,70);});
}

// ======= Loop =======
function loop(){update();draw();requestAnimationFrame(loop);}
generateMap();spawnRoom();loop();
</script>
</body>
</html>
