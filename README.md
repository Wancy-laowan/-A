[two_player_duel.html](https://github.com/user-attachments/files/32158533/two_player_duel.html)
# <!doctype html>
<html lang="zh-CN">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no" />
<title>霓虹双人决斗</title>
<style>
  :root{
    --bg:#090b14;
    --panel:#12182b;
    --text:#f7f9ff;
    --muted:#aeb8d6;
    --p1:#5cf2ff;
    --p2:#ff6bd6;
    --gold:#ffe66d;
  }
  *{box-sizing:border-box}
  html,body{margin:0;min-height:100%;background:radial-gradient(circle at 50% 20%,#18213d 0,#0b0e1a 45%,#060810 100%);color:var(--text);font-family:Inter,ui-sans-serif,system-ui,-apple-system,"PingFang SC","Microsoft YaHei",sans-serif}
  body{display:grid;place-items:center;padding:18px}
  .shell{width:min(1100px,100%);display:grid;gap:14px}
  .topbar{display:flex;align-items:center;justify-content:space-between;gap:16px}
  .brand{font-size:clamp(24px,4vw,42px);font-weight:900;letter-spacing:.04em}
  .brand span{opacity:.55;font-weight:700}
  .hint{color:var(--muted);font-size:14px;text-align:right}
  .game-wrap{position:relative;overflow:hidden;border-radius:24px;border:1px solid rgba(255,255,255,.13);box-shadow:0 24px 80px rgba(0,0,0,.45);background:#070a13}
  canvas{display:block;width:100%;height:auto;aspect-ratio:16/9;background:#0a0d18}
  .overlay{position:absolute;inset:0;display:grid;place-items:center;padding:24px;background:linear-gradient(180deg,rgba(5,7,15,.35),rgba(5,7,15,.78));backdrop-filter:blur(6px);transition:.2s}
  .overlay.hidden{opacity:0;pointer-events:none}
  .card{width:min(560px,92%);padding:28px;border-radius:24px;background:rgba(15,20,38,.92);border:1px solid rgba(255,255,255,.12);box-shadow:0 20px 70px rgba(0,0,0,.38);text-align:center}
  h1{margin:0 0 10px;font-size:clamp(30px,5vw,54px)}
  .sub{margin:0 auto 20px;color:var(--muted);line-height:1.6}
  .controls{display:grid;grid-template-columns:1fr 1fr;gap:12px;margin:18px 0}
  .control{padding:14px;border-radius:16px;background:rgba(255,255,255,.05);border:1px solid rgba(255,255,255,.08);text-align:left}
  .control b{display:block;margin-bottom:5px}
  .p1 b{color:var(--p1)} .p2 b{color:var(--p2)}
  kbd{display:inline-grid;place-items:center;min-width:26px;height:26px;padding:0 7px;border-radius:8px;background:#222a43;border:1px solid rgba(255,255,255,.16);box-shadow:inset 0 -2px rgba(0,0,0,.35);font-weight:800;font-size:12px}
  button{appearance:none;border:0;border-radius:14px;padding:12px 20px;font-weight:900;font-size:16px;cursor:pointer;background:linear-gradient(135deg,#7cf7ff,#9e83ff 55%,#ff7ddc);color:#08101a;box-shadow:0 10px 30px rgba(91,213,255,.22)}
  .footer{display:flex;justify-content:space-between;gap:12px;flex-wrap:wrap;color:var(--muted);font-size:13px}
  .pill{padding:9px 12px;border-radius:999px;background:rgba(255,255,255,.05);border:1px solid rgba(255,255,255,.08)}
  @media (max-width:650px){.controls{grid-template-columns:1fr}.hint{display:none}.card{padding:20px}}
</style>
</head>
<body>
<div class="shell">
  <div class="topbar">
    <div class="brand">霓虹决斗 <span>2P</span></div>
    <div class="hint">同屏双人 · 能量弹 · 跳跃 · 三局两胜</div>
  </div>

  <div class="game-wrap">
    <canvas id="game" width="960" height="540"></canvas>

    <div class="overlay" id="overlay">
      <div class="card">
        <h1 id="overlayTitle">双人决斗</h1>
        <p class="sub" id="overlayText">把对手的能量值打到 0。每局胜者得 1 分，先拿到 2 分获胜。</p>
        <div class="controls">
          <div class="control p1">
            <b>玩家 1 · 青蓝</b>
            <kbd>A</kbd> <kbd>D</kbd> 移动　<kbd>W</kbd> 跳跃　<kbd>F</kbd> 发射
          </div>
          <div class="control p2">
            <b>玩家 2 · 粉紫</b>
            <kbd>←</kbd> <kbd>→</kbd> 移动　<kbd>↑</kbd> 跳跃　<kbd>L</kbd> 发射
          </div>
        </div>
        <button id="startBtn">开始决斗</button>
      </div>
    </div>
  </div>

  <div class="footer">
    <div class="pill">⚡ 发射有短暂冷却，边跳边打更灵活</div>
    <div class="pill">↻ 按 <b>R</b> 可重新开始整场比赛</div>
  </div>
</div>

<script>
(() => {
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const overlay = document.getElementById('overlay');
  const overlayTitle = document.getElementById('overlayTitle');
  const overlayText = document.getElementById('overlayText');
  const startBtn = document.getElementById('startBtn');

  const W = canvas.width, H = canvas.height;
  const GROUND = 458;
  const keys = new Set();
  let running = false;
  let last = performance.now();
  let particles = [];
  let projectiles = [];
  let round = 1;
  let matchOver = false;

  const stars = Array.from({length:95}, (_,i)=>({
    x:(i*83)%W, y:(i*47)%360, r: 0.5+(i%4)*0.35, a:.2+(i%7)*.08
  }));

  const platforms = [
    {x:180,y:360,w:180,h:14},
    {x:600,y:360,w:180,h:14},
    {x:390,y:285,w:180,h:14}
  ];

  function player(x, color, facing, controls, name){
    return {
      x,y:GROUND-52,w:36,h:52,vx:0,vy:0,speed:255,jump:565,
      color,facing,controls,name,hp:100,score:0,cooldown:0,onGround:false,
      invuln:0
    };
  }

  const p1 = player(120,'#5cf2ff',1,{left:'KeyA',right:'KeyD',jump:'KeyW',shoot:'KeyF'},'P1');
  const p2 = player(804,'#ff6bd6',-1,{left:'ArrowLeft',right:'ArrowRight',jump:'ArrowUp',shoot:'KeyL'},'P2');

  function resetRound(){
    Object.assign(p1,{x:120,y:GROUND-52,vx:0,vy:0,hp:100,facing:1,cooldown:0,invuln:0});
    Object.assign(p2,{x:804,y:GROUND-52,vx:0,vy:0,hp:100,facing:-1,cooldown:0,invuln:0});
    projectiles = [];
    particles = [];
    running = true;
    matchOver = false;
    overlay.classList.add('hidden');
  }

  function resetMatch(){
    p1.score = 0; p2.score = 0; round = 1; matchOver = false;
    overlayTitle.textContent = '双人决斗';
    overlayText.textContent = '把对手的能量值打到 0。每局胜者得 1 分，先拿到 2 分获胜。';
    startBtn.textContent = '开始决斗';
    resetRound();
  }

  function showRoundResult(winner){
    running = false;
    winner.score++;
    const wonMatch = winner.score >= 2;
    if(wonMatch){
      matchOver = true;
      overlayTitle.textContent = `${winner.name} 获得胜利！`;
      overlayText.textContent = `最终比分 ${p1.score} : ${p2.score}。按按钮或 R 开始新比赛。`;
      startBtn.textContent = '再来一场';
    } else {
      overlayTitle.textContent = `${winner.name} 赢下第 ${round} 局`;
      overlayText.textContent = `当前比分 ${p1.score} : ${p2.score}。下一局即将继续。`;
      startBtn.textContent = '下一局';
      round++;
    }
    overlay.classList.remove('hidden');
  }

  function rectHit(a,b){
    return a.x < b.x+b.w && a.x+a.w > b.x && a.y < b.y+b.h && a.y+a.h > b.y;
  }

  function burst(x,y,color,count=14){
    for(let i=0;i<count;i++){
      const a=Math.random()*Math.PI*2, s=70+Math.random()*170;
      particles.push({x,y,vx:Math.cos(a)*s,vy:Math.sin(a)*s,life:.35+Math.random()*.35,max:.7,color,r:2+Math.random()*3});
    }
  }

  function shoot(p){
    if(p.cooldown>0) return;
    p.cooldown=.34;
    const dir=p.facing;
    projectiles.push({
      x:p.x+p.w/2+dir*24,y:p.y+22,w:18,h:8,vx:dir*520,owner:p,
      color:p.color,life:1.65
    });
    burst(p.x+p.w/2+dir*24,p.y+26,p.color,5);
  }

  function resolveWorld(p, dt){
    p.vy += 1450*dt;
    p.x += p.vx*dt;
    p.x = Math.max(18, Math.min(W-p.w-18,p.x));

    const oldY=p.y;
    p.y += p.vy*dt;
    p.onGround=false;

    if(p.y+p.h >= GROUND){
      p.y=GROUND-p.h; p.vy=0; p.onGround=true;
    }

    for(const plat of platforms){
      const wasAbove = oldY+p.h <= plat.y+4;
      const nowCross = p.y+p.h >= plat.y && p.y+p.h <= plat.y+24;
      const horizontal = p.x+p.w>plat.x && p.x<plat.x+plat.w;
      if(p.vy>=0 && wasAbove && nowCross && horizontal){
        p.y=plat.y-p.h; p.vy=0; p.onGround=true;
      }
    }
  }

  function updatePlayer(p, opponent, dt){
    const left=keys.has(p.controls.left), right=keys.has(p.controls.right);
    p.vx = (right-left)*p.speed;
    if(left&&!right)p.facing=-1;
    if(right&&!left)p.facing=1;
    if(keys.has(p.controls.shoot)) shoot(p);

    if(p.cooldown>0)p.cooldown-=dt;
    if(p.invuln>0)p.invuln-=dt;
    resolveWorld(p,dt);

    if(rectHit(p,opponent)){
      const mid=(p.x+p.w/2)-(opponent.x+opponent.w/2);
      const push=mid<0?-1:1;
      p.x+=push*2.2;
      opponent.x-=push*2.2;
    }
  }

  function jump(p){
    if(running && p.onGround){
      p.vy=-p.jump; p.onGround=false;
      burst(p.x+p.w/2,p.y+p.h,p.color,8);
    }
  }

  addEventListener('keydown', e=>{
    if(['ArrowLeft','ArrowRight','ArrowUp','Space'].includes(e.code)) e.preventDefault();
    if(!keys.has(e.code)){
      if(e.code===p1.controls.jump) jump(p1);
      if(e.code===p2.controls.jump) jump(p2);
      if(e.code==='KeyR') resetMatch();
    }
    keys.add(e.code);
  },{passive:false});
  addEventListener('keyup', e=>keys.delete(e.code));

  startBtn.addEventListener('click',()=>{
    if(matchOver){ p1.score=0; p2.score=0; round=1; }
    resetRound();
  });

  function update(dt){
    if(!running) return;
    updatePlayer(p1,p2,dt);
    updatePlayer(p2,p1,dt);

    for(let i=projectiles.length-1;i>=0;i--){
      const b=projectiles[i];
      b.x+=b.vx*dt; b.life-=dt;
      const target=b.owner===p1?p2:p1;
      if(b.life<=0 || b.x<-40 || b.x>W+40){
        projectiles.splice(i,1); continue;
      }
      if(target.invuln<=0 && rectHit(b,target)){
        target.hp=Math.max(0,target.hp-14);
        target.invuln=.12;
        target.vx += Math.sign(b.vx)*85;
        burst(b.x,b.y,b.color,16);
        projectiles.splice(i,1);
        if(target.hp<=0){
          showRoundResult(b.owner);
          break;
        }
      }
    }

    for(let i=particles.length-1;i>=0;i--){
      const p=particles[i];
      p.x+=p.vx*dt; p.y+=p.vy*dt; p.vy+=380*dt; p.life-=dt;
      if(p.life<=0) particles.splice(i,1);
    }
  }

  function roundRect(x,y,w,h,r){
    ctx.beginPath();
    ctx.roundRect(x,y,w,h,r);
  }

  function drawBackground(t){
    const g=ctx.createLinearGradient(0,0,0,H);
    g.addColorStop(0,'#0b1130'); g.addColorStop(.55,'#11132b'); g.addColorStop(1,'#070a13');
    ctx.fillStyle=g; ctx.fillRect(0,0,W,H);

    for(const s of stars){
      const pulse=.55+.45*Math.sin(t*.001+s.x);
      ctx.globalAlpha=s.a*pulse;
      ctx.fillStyle='#d9e8ff';
      ctx.beginPath();ctx.arc(s.x,s.y,s.r,0,Math.PI*2);ctx.fill();
    }
    ctx.globalAlpha=1;

    ctx.save();
    ctx.globalAlpha=.16;
    ctx.strokeStyle='#7b8cff';
    ctx.lineWidth=1;
    for(let x=0;x<W;x+=48){ctx.beginPath();ctx.moveTo(x,405);ctx.lineTo(W/2+(x-W/2)*1.45,H);ctx.stroke()}
    for(let y=405;y<H;y+=22){ctx.beginPath();ctx.moveTo(0,y);ctx.lineTo(W,y);ctx.stroke()}
    ctx.restore();

    const city=[45,82,123,166,205,252,302,349,404,450,505,554,603,657,705,758,812,865,918];
    ctx.fillStyle='rgba(70,84,142,.18)';
    city.forEach((x,i)=>{
      const h=50+(i*37)%110,w=30+(i*13)%28;
      ctx.fillRect(x-w/2,GROUND-h,w,h);
    });

    ctx.fillStyle='#11172a';
    ctx.fillRect(0,GROUND,W,H-GROUND);
    ctx.fillStyle='rgba(123,140,255,.18)';
    ctx.fillRect(0,GROUND,W,2);

    for(const p of platforms){
      ctx.fillStyle='rgba(27,35,63,.94)';
      roundRect(p.x,p.y,p.w,p.h,7);ctx.fill();
      const lg=ctx.createLinearGradient(p.x,0,p.x+p.w,0);
      lg.addColorStop(0,'rgba(92,242,255,.55)'); lg.addColorStop(.5,'rgba(158,131,255,.65)'); lg.addColorStop(1,'rgba(255,107,214,.55)');
      ctx.fillStyle=lg;ctx.fillRect(p.x+8,p.y,p.w-16,2);
    }
  }

  function drawPlayer(p){
    ctx.save();
    const blink=p.invuln>0 && Math.floor(p.invuln*35)%2===0;
    ctx.globalAlpha=blink?.45:1;
    const cx=p.x+p.w/2, cy=p.y+p.h/2;

    ctx.shadowColor=p.color; ctx.shadowBlur=18;
    ctx.fillStyle=p.color;
    roundRect(p.x,p.y+7,p.w,p.h-7,10);ctx.fill();

    ctx.shadowBlur=0;
    ctx.fillStyle='#0a1020';
    roundRect(p.x+7,p.y+14,p.w-14,15,6);ctx.fill();

    ctx.fillStyle='#eef8ff';
    const eyeX=p.facing>0?p.x+24:p.x+10;
    ctx.fillRect(eyeX,p.y+19,5,4);

    ctx.strokeStyle=p.color;ctx.lineWidth=5;ctx.lineCap='round';
    ctx.beginPath();ctx.moveTo(cx,p.y+42);ctx.lineTo(cx+p.facing*17,p.y+34);ctx.stroke();

    ctx.fillStyle='rgba(255,255,255,.75)';
    ctx.fillRect(p.x+5,p.y+p.h-4,10,4);
    ctx.fillRect(p.x+p.w-15,p.y+p.h-4,10,4);
    ctx.restore();
  }

  function drawProjectile(b){
    ctx.save();
    ctx.shadowColor=b.color;ctx.shadowBlur=16;
    ctx.fillStyle=b.color;
    roundRect(b.x,b.y,b.w,b.h,5);ctx.fill();
    ctx.globalAlpha=.35;
    ctx.fillRect(b.x-(Math.sign(b.vx)*18),b.y+2,Math.sign(b.vx)*18,b.h-4);
    ctx.restore();
  }

  function drawHUD(){
    ctx.save();
    ctx.font='800 16px system-ui';
    ctx.textBaseline='middle';

    const margin=34, barW=300, barH=18, top=30;

    ctx.fillStyle='rgba(0,0,0,.35)';
    roundRect(margin,top,barW,barH,9);ctx.fill();
    roundRect(W-margin-barW,top,barW,barH,9);ctx.fill();

    ctx.fillStyle=p1.color;
    roundRect(margin,top,barW*(p1.hp/100),barH,9);ctx.fill();
    ctx.fillStyle=p2.color;
    const w2=barW*(p2.hp/100);
    roundRect(W-margin-w2,top,w2,barH,9);ctx.fill();

    ctx.fillStyle='#fff';
    ctx.fillText(`P1  ${p1.hp}`,margin,top+34);
    ctx.textAlign='right';ctx.fillText(`${p2.hp}  P2`,W-margin,top+34);
    ctx.textAlign='center';
    ctx.fillStyle='#c8d2ef';
    ctx.fillText(`第 ${round} 局　${p1.score} : ${p2.score}`,W/2,top+8);

    ctx.restore();
  }

  function drawParticles(){
    for(const p of particles){
      ctx.globalAlpha=Math.max(0,p.life/p.max);
      ctx.fillStyle=p.color;
      ctx.beginPath();ctx.arc(p.x,p.y,p.r,0,Math.PI*2);ctx.fill();
    }
    ctx.globalAlpha=1;
  }

  function render(t){
    drawBackground(t);
    drawHUD();
    projectiles.forEach(drawProjectile);
    drawPlayer(p1); drawPlayer(p2);
    drawParticles();

    if(running){
      ctx.save();
      ctx.textAlign='center';
      ctx.font='700 13px system-ui';
      ctx.fillStyle='rgba(228,234,255,.55)';
      ctx.fillText('F',p1.x+p1.w/2,p1.y-11);
      ctx.fillText('L',p2.x+p2.w/2,p2.y-11);
      ctx.restore();
    }
  }

  function loop(now){
    const dt=Math.min(.025,(now-last)/1000); last=now;
    update(dt); render(now); requestAnimationFrame(loop);
  }

  render(performance.now());
  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
