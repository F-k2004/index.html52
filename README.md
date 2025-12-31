<!DOCTYPE html>
<html lang="fa">
<head>
<meta charset="UTF-8">
<title>🛰️ Multi-Station DSN Network</title>
<style>
html,body{margin:0;overflow:hidden;background:#00030a;font-family:system-ui}
canvas{display:block}
.hud{
  position:absolute;left:16px;top:16px;
  padding:14px 18px;border-radius:14px;
  background:rgba(255,255,255,0.06);
  backdrop-filter:blur(10px);
  color:#d9f3ff;font-size:13px;min-width:280px;
}
.good{color:#9ff0ff}
.mid{color:#ffd29f}
.bad{color:#ff9f9f}
.warn{color:#ffb366}
</style>
</head>
<body>

<canvas id="c"></canvas>
<div class="hud" id="hud"></div>

<script>
const c=document.getElementById("c");
const ctx=c.getContext("2d");
let w,h;
function resize(){w=c.width=innerWidth;h=c.height=innerHeight}
resize();addEventListener("resize",resize);

// Earth
const earth={x:w/2,y:h/2,r:70,mu:9000};

// Satellite
const sat={x:earth.x,y:earth.y-240,vx:2.25,vy:0};

// Sun & Storm
const sun={angle:0};
let storm={active:false,intensity:0,timer:0};

// DSN stations
const stations=[
  {name:"Goldstone (USA)",angle:0.2,color:"#00ffcc"},
  {name:"Madrid (EU)",angle:2.2,color:"#66aaff"},
  {name:"Canberra (AUS)",angle:4.1,color:"#ffaa66"}
];

// Constants
const txPower=130;
const maxDist=420;

// Physics
function gravity(){
  const dx=earth.x-sat.x,dy=earth.y-sat.y;
  const d=Math.hypot(dx,dy);
  const f=earth.mu/(d*d);
  sat.vx+=f*dx/d;
  sat.vy+=f*dy/d;
}

// LOS
function hasLOS(st){
  const dx=sat.x-st.x,dy=sat.y-st.y;
  const d=Math.hypot(dx,dy);
  if(d>maxDist) return false;
  const t=((earth.x-st.x)*dx+(earth.y-st.y)*dy)/(d*d);
  const px=st.x+t*dx,py=st.y+t*dy;
  return Math.hypot(px-earth.x,py-earth.y)>earth.r;
}

// Shadow
function inShadow(){
  const lx=Math.cos(sun.angle),ly=Math.sin(sun.angle);
  const dx=sat.x-earth.x,dy=sat.y-earth.y;
  const proj=dx*lx+dy*ly;
  if(proj<0) return false;
  const perp=Math.abs(-dx*ly+dy*lx);
  return perp<earth.r;
}

// Solar storm
function updateStorm(){
  if(!storm.active && Math.random()<0.001){
    storm.active=true;
    storm.intensity=1+Math.random()*3;
    storm.timer=300+Math.random()*400;
  }
  if(storm.active){
    storm.timer--;
    storm.intensity*=0.994;
    if(storm.timer<=0 || storm.intensity<0.1){
      storm.active=false;
    }
  }
}

// Noise
function plasmaNoise(){
  let n=0.04+Math.random()*0.05;
  if(storm.active) n+=storm.intensity*0.25;
  return n;
}

// Signal
function linkQuality(st){
  if(!hasLOS(st)) return null;
  const dx=sat.x-st.x,dy=sat.y-st.y;
  const dist=Math.hypot(dx,dy);
  let signal=txPower/(dist*dist);
  if(inShadow()) signal*=0.25;
  const noise=plasmaNoise();
  const snr=signal/noise;
  return {signal,noise,snr,dist};
}

function updateStations(){
  stations.forEach(s=>{
    s.angle+=0.0007;
    s.x=earth.x+Math.cos(s.angle)*earth.r;
    s.y=earth.y+Math.sin(s.angle)*earth.r;
  });
}

function update(){
  gravity();
  sat.x+=sat.vx;
  sat.y+=sat.vy;
  updateStations();
  updateStorm();
  sun.angle+=0.0003;
}

function draw(){
  ctx.fillStyle="rgba(0,3,10,0.35)";
  ctx.fillRect(0,0,w,h);

  update();

  // Earth
  ctx.beginPath();
  ctx.arc(earth.x,earth.y,earth.r,0,Math.PI*2);
  ctx.fillStyle="#0b3d91";
  ctx.fill();

  // Satellite
  ctx.beginPath();
  ctx.arc(sat.x,sat.y,4,0,Math.PI*2);
  ctx.fillStyle="#e6f7ff";
  ctx.fill();

  let best=null;

  stations.forEach(st=>{
    ctx.beginPath();
    ctx.arc(st.x,st.y,4,0,Math.PI*2);
    ctx.fillStyle=st.color;
    ctx.fill();

    const q=linkQuality(st);
    if(q && (!best || q.snr>best.q.snr)){
      best={st,q};
    }
  });

  let hud=`🛰️ DSN NETWORK<br>`;
  if(storm.active) hud+=`🌞 <span class="warn">SOLAR STORM</span><br>`;

  if(best){
    let cls=best.q.snr>12?"good":best.q.snr>6?"mid":"bad";
    ctx.strokeStyle=
      best.q.snr>12?"rgba(120,220,255,0.9)":
      best.q.snr>6 ?"rgba(255,200,120,0.7)":
                    "rgba(255,120,120,0.6)";
    ctx.lineWidth=1.8;
    ctx.beginPath();
    ctx.moveTo(best.st.x,best.st.y);
    ctx.lineTo(sat.x,sat.y);
    ctx.stroke();

    hud+=`
📡 Active Station:<br>
<b>${best.st.name}</b><br>
📶 SNR: <span class="${cls}">${best.q.snr.toFixed(1)}</span><br>
📏 Distance: ${best.q.dist.toFixed(0)}<br>
🌑 Shadow: ${inShadow()?"YES":"NO"}<br>
🔁 Auto Handover: ENABLED
`;
  }else{
    hud+=`📡 <span class="bad">NO STATION AVAILABLE</span>`;
  }

  document.getElementById("hud").innerHTML=hud;
  requestAnimationFrame(draw);
}

draw();
</script>
</body>
</html>
