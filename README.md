import html
from IPython.display import HTML, display

GAME = r"""
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Lantern Drift</title>
<style>
:root{--ink:#120d24;--text:#f4eefc;--amber:#ffb347;--pink:#ff4f81;--teal:#1ec8b0}
html,body{height:100%;margin:0;background:var(--ink);color:var(--text);font-family:"Trebuchet MS","Segoe UI",sans-serif;overflow:hidden;touch-action:none}
#c{display:block;width:100%;height:100%}
#hud{position:fixed;left:0;right:0;top:0;display:flex;justify-content:space-between;padding:14px 18px;font-size:20px;pointer-events:none;text-shadow:0 2px 8px #000a}
#hud b{color:var(--amber)}
#msg{position:fixed;inset:0;display:flex;flex-direction:column;align-items:center;justify-content:center;text-align:center;padding:24px;background:#120d24cc;gap:14px}
#msg h1{margin:0;font-size:clamp(30px,6vw,56px);color:var(--amber)}
#msg p{margin:0;max-width:34ch;line-height:1.5;opacity:.85}
button{font:inherit;font-size:18px;padding:12px 28px;border:0;border-radius:999px;background:var(--amber);color:var(--ink);cursor:pointer}
#msg.hide{display:none}
</style>
</head>
<body>
<canvas id="c"></canvas>
<div id="hud"><span>Orbs <b id="s">0</b></span><span>Lives <b id="l">3</b></span></div>
<div id="msg"><h1 id="t">Lantern Drift</h1><p id="d">Gather the amber orbs and dodge the pink drifters. Move with WASD or arrow keys, or drag with mouse/touch.</p><button id="go">Start</button></div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
(function(){
const cv=document.getElementById('c');
const R=new THREE.WebGLRenderer({canvas:cv,antialias:true});
R.setPixelRatio(Math.min(devicePixelRatio,2));
const sc=new THREE.Scene();
sc.background=new THREE.Color(0x2a1b4d);
sc.fog=new THREE.Fog(0x2a1b4d,18,48);
const cam=new THREE.PerspectiveCamera(60,1,.1,100);
function resize(){R.setSize(innerWidth,innerHeight,false);cam.aspect=innerWidth/innerHeight;cam.updateProjectionMatrix()}
addEventListener('resize',resize);resize();

sc.add(new THREE.HemisphereLight(0xb9a6ff,0x1a1030,.9));
const sun=new THREE.DirectionalLight(0xffc98a,.8);sun.position.set(8,14,6);sc.add(sun);

const A=22;
const sea=new THREE.Mesh(new THREE.PlaneGeometry(A*2,A*2,28,28),new THREE.MeshStandardMaterial({color:0x1b7f8f,flatShading:true,roughness:.7}));
sea.rotation.x=-Math.PI/2;sc.add(sea);
const base=sea.geometry.attributes.position.array.slice();

const P=new THREE.Mesh(new THREE.OctahedronGeometry(.8),new THREE.MeshStandardMaterial({color:0xfff2d6,emissive:0xffb347,emissiveIntensity:.6,flatShading:true}));
P.position.y=1.1;sc.add(P);
P.add(new THREE.PointLight(0xffb347,1.4,10));

const orbs=[],foes=[];
const orbG=new THREE.SphereGeometry(.45,14,14),orbM=new THREE.MeshStandardMaterial({color:0xffb347,emissive:0xffb347,emissiveIntensity:.9});
const foeG=new THREE.ConeGeometry(.8,1.8,6),foeM=new THREE.MeshStandardMaterial({color:0xff4f81,emissive:0x7a1038,flatShading:true});
const rnd=()=>(Math.random()*2-1)*(A-3);
function spawnOrb(){const o=new THREE.Mesh(orbG,orbM);o.position.set(rnd(),1,rnd());sc.add(o);orbs.push(o)}
function spawnFoe(){const f=new THREE.Mesh(foeG,foeM);f.position.set(rnd(),1,rnd());const a=Math.random()*6.28,v=2+Math.random()*1.5;f.userData.v=new THREE.Vector3(Math.cos(a)*v,0,Math.sin(a)*v);sc.add(f);foes.push(f)}

const keys={};let drag=null;
addEventListener('keydown',e=>{keys[e.code]=true;if(e.code.startsWith('Arrow'))e.preventDefault()});
addEventListener('keyup',e=>keys[e.code]=false);
cv.addEventListener('pointerdown',e=>drag={x:e.clientX,y:e.clientY,dx:0,dy:0});
addEventListener('pointermove',e=>{if(drag){drag.dx=Math.max(-1,Math.min(1,(e.clientX-drag.x)/60));drag.dy=Math.max(-1,Math.min(1,(e.clientY-drag.y)/60))}});
addEventListener('pointerup',()=>drag=null);

let score=0,lives=3,running=false,inv=0,vel=new THREE.Vector3();
const $=id=>document.getElementById(id);
function reset(){
  orbs.concat(foes).forEach(m=>sc.remove(m));orbs.length=foes.length=0;
  score=0;lives=3;inv=0;vel.set(0,0,0);P.position.set(0,1.1,0);
  for(let i=0;i<8;i++)spawnOrb();for(let i=0;i<3;i++)spawnFoe();
  $('s').textContent=0;$('l').textContent=3;
}
function end(){running=false;$('t').textContent='Lights out';$('d').textContent='You gathered '+score+' orbs. Try for more.';$('go').textContent='Play again';$('msg').classList.remove('hide')}
$('go').onclick=()=>{reset();running=true;$('msg').classList.add('hide');window.focus()};
reset();

let last=performance.now();
function loop(now){
  requestAnimationFrame(loop);
  const dt=Math.min((now-last)/1000,.05);last=now;const t=now/1000;
  const pos=sea.geometry.attributes.position;
  for(let i=0;i<pos.count;i++)pos.setZ(i,Math.sin(base[i*3]*.4+t)*.35+Math.cos(base[i*3+1]*.4+t*.8)*.35);
  pos.needsUpdate=true;sea.geometry.computeVertexNormals();
  P.rotation.y+=dt*1.5;
  orbs.forEach((o,i)=>{o.position.y=1+Math.sin(t*2+i)*.25});
  if(running){
    let ix=(keys.KeyD||keys.ArrowRight?1:0)-(keys.KeyA||keys.ArrowLeft?1:0);
    let iz=(keys.KeyS||keys.ArrowDown?1:0)-(keys.KeyW||keys.ArrowUp?1:0);
    if(drag){ix+=drag.dx;iz+=drag.dy}
    vel.x+=ix*30*dt;vel.z+=iz*30*dt;vel.multiplyScalar(1-2.2*dt);
    P.position.addScaledVector(vel,dt);
    P.position.x=Math.max(-A+1,Math.min(A-1,P.position.x));
    P.position.z=Math.max(-A+1,Math.min(A-1,P.position.z));
    for(let i=orbs.length-1;i>=0;i--)if(orbs[i].position.distanceTo(P.position)<1.3){
      sc.remove(orbs[i]);orbs.splice(i,1);spawnOrb();score++;$('s').textContent=score;
      if(score%5===0)spawnFoe();
    }
    inv=Math.max(0,inv-dt);P.visible=inv<=0||Math.floor(t*12)%2===0;
    foes.forEach(f=>{
      f.position.addScaledVector(f.userData.v,dt);f.rotation.y+=dt*2;
      if(Math.abs(f.position.x)>A-1)f.userData.v.x*=-1;
      if(Math.abs(f.position.z)>A-1)f.userData.v.z*=-1;
      if(!inv&&f.position.distanceTo(P.position)<1.5){
        lives--;inv=1.5;$('l').textContent=lives;vel.multiplyScalar(-1);
        if(lives<=0)end();
      }
    });
  }
  cam.position.x+=(P.position.x-cam.position.x)*3*dt;
  cam.position.z+=(P.position.z+11-cam.position.z)*3*dt;
  cam.position.y=9;
  cam.lookAt(P.position.x,0,P.position.z-1);
  R.render(sc,cam);
}
requestAnimationFrame(loop);
})();
</script>
</body>
</html>
"""
with open("lantern-drift.html", "w") as f:
    f.write(GAME)

display(HTML(
    f'<iframe srcdoc="{html.escape(GAME, quote=True)}" '
    'width="100%" height="600" style="border:0;border-radius:8px" '
    'allow="fullscreen"></iframe>'
))
