<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Survivor or Die - FULL</title>
<style>
body{margin:0;overflow:hidden;font-family:sans-serif;}
canvas{display:block;}
#menu{position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.85);color:white;display:flex;flex-direction:column;align-items:center;justify-content:center;}
input,label,button{margin:5px;padding:5px;}
#inventory,#craft,#ui,#healthBar,#joystick{position:fixed;background:rgba(0,0,0,0.3);color:white;padding:5px;}
#inventory{top:40px;right:10px;width:160px;height:250px;overflow:auto;}
#craft{top:300px;right:10px;width:160px;height:250px;overflow:auto;}
#ui{bottom:10px;width:100%;text-align:center;}
#healthBar{top:10px;left:10px;width:200px;height:20px;background:#000;}
#health{height:100%;background:green;width:100%;}
#joystick{left:20px;bottom:80px;width:100px;height:100px;border-radius:50%;background:rgba(0,0,0,0.2);}
#stick{width:50px;height:50px;background:red;border-radius:50%;position:absolute;left:25px;top:25px;}
</style>
</head>
<body>

<div id="menu">
<h2>Survivor or Die</h2>
<label>İsim: <input id="playerName" type="text" placeholder="İsmin"></label><br>
<label>MP3 Seç: <input id="mp3Input" type="file" accept="audio/mp3"></label><br>
<button onclick="startGame()">Başla</button>
</div>

<div id="inventory"><b>Envanter:</b><div id="items"></div></div>
<div id="craft"><b>Craft:</b><div id="craftButtons"></div><div id="craftResult"></div></div>
<div id="ui">
<button onclick="switchCamera()">Switch Camera</button>
<button onclick="breakBlock()">Break Block</button>
</div>
<div id="healthBar"><div id="health"></div></div>
<div id="joystick"><div id="stick"></div></div>
<audio id="bgm" loop></audio>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r152/three.min.js"></script>
<script>
// --- Menü ve MP3 ---
let playerNameVal;
let audioEl=document.getElementById('bgm');
let gameStarted=false;

function startGame(){
  playerNameVal=document.getElementById('playerName').value||'Survivor';
  let fileInput=document.getElementById('mp3Input').files[0];
  if(fileInput){
    let reader=new FileReader();
    reader.onload=(e)=>{ audioEl.src=e.target.result; audioEl.play(); };
    reader.readAsDataURL(fileInput);
  } else {
    audioEl.src="https://cdn.pixabay.com/download/audio/2023/03/14/audio_a95f44b9e2.mp3?filename=turkish-national-march-1.mp3";
    audioEl.play();
  }
  document.getElementById('menu').style.display='none';
  gameStarted=true;
  initGame();
}

// --- Değişkenler ---
let scene,camera,renderer,playerCube;
let inventory={wood:0,stone:0,iron:0,gold:0,diamond:0,pickaxe:0,sword:0,armor:0};
let recipes=[
  {from:{wood:1},to:{pickaxe:1},name:'Wood → Pickaxe'},
  {from:{stone:3},to:{stonePickaxe:1},name:'3 Stone → Stone Pickaxe'},
  {from:{iron:3},to:{ironPickaxe:1},name:'3 Iron → Iron Pickaxe'},
  {from:{gold:3},to:{goldPickaxe:1},name:'3 Gold → Gold Pickaxe'},
  {from:{diamond:3},to:{diamondPickaxe:1},name:'3 Diamond → Diamond Pickaxe'},
  {from:{wood:2},to:{sword:1},name:'2 Wood → Sword'}
];
let health=100;
let blocks=[];
let enemies=[];

// --- Envanter ve Craft ---
function updateInventory(){
  let inv='';
  for(let k in inventory){ inv+=k+': '+inventory[k]+'<br>'; }
  document.getElementById('items').innerHTML=inv;
}
function updateCraftButtons(){
  let html='';
  recipes.forEach((r,i)=>{ html+=`<button onclick="craft(${i})">${r.name}</button><br>`; });
  document.getElementById('craftButtons').innerHTML=html;
}
function craft(i){
  let r=recipes[i],can=true;
  for(let k in r.from){ if((inventory[k]||0)<r.from[k]) can=false; }
  if(can){
    for(let k in r.from){ inventory[k]-=r.from[k]; }
    for(let k in r.to){ inventory[k]=(inventory[k]||0)+r.to[k]; }
    document.getElementById('craftResult').innerHTML="✔ Craft Successful!";
  } else {
    document.getElementById('craftResult').innerHTML="❌ Not Enough Materials";
  }
  updateInventory();
}

// --- Oyun Başlat ---
function initGame(){
  scene=new THREE.Scene();
  camera=new THREE.PerspectiveCamera(75,window.innerWidth/window.innerHeight,0.1,1000);
  renderer=new THREE.WebGLRenderer();
  renderer.setSize(window.innerWidth,window.innerHeight);
  document.body.appendChild(renderer.domElement);

  // Player cube
  let geo=new THREE.BoxGeometry(1,1,1);
  let mat=new THREE.MeshBasicMaterial({color:0x00ff00});
  playerCube=new THREE.Mesh(geo,mat);
  scene.add(playerCube);
  camera.position.set(0,2,5);

  spawnWorld();
  spawnEnemy();
  animate();
}

function spawnWorld(){
  for(let i=0;i<50;i++){
    let geo=new THREE.BoxGeometry(1,1,1);
    let mat=new THREE.MeshBasicMaterial({color:0x8B4513});
    let block=new THREE.Mesh(geo,mat);
    block.position.set((Math.random()*20)-10, -1, (Math.random()*20)-10);
    scene.add(block);
    blocks.push(block);
  }
}

function spawnEnemy(){
  let geo=new THREE.SphereGeometry(0.5,16,16);
  let mat=new THREE.MeshBasicMaterial({color:0xff0000});
  let enemy=new THREE.Mesh(geo,mat);
  enemy.position.set(5,0,-5);
  scene.add(enemy);
  enemies.push(enemy);
}

function switchCamera(){
  camera.position.z = camera.position.z === 5 ? 2 : 5;
}

function breakBlock(){
  if(blocks.length>0){
    let b=blocks.pop();
    scene.remove(b);
    inventory.wood++;
    updateInventory();
  }
}

function animate(){
  requestAnimationFrame(animate);

  enemies.forEach(e=>{
    e.position.x += (playerCube.position.x - e.position.x)*0.01;
    e.position.z += (playerCube.position.z - e.position.z)*0.01;
  });

  renderer.render(scene,camera);
}

updateInventory();
updateCraftButtons();
</script>
</body>
</html>![Screenshot_20251111-202833](https://github.com/user-attachments/assets/529c80a2-c441-47cb-8349-d7e5af5d04ab)
