<!doctype html>
<html lang="ar">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no" />
<title>English × Mouse & Cheese — Mobile</title>
<style>
  html,body{margin:0;height:100%;background:#0a0e1a;overflow:hidden;touch-action:none;}
  canvas{display:block;width:100vw;height:100vh;}
</style>
</head>
<body>
<canvas id="app"></canvas>
<script>
// ===== Canvas & DPI =====
const canvas = document.getElementById('app');
const ctx = canvas.getContext('2d');
function resize(){
  const dpr = Math.max(1, Math.floor(devicePixelRatio||1));
  canvas.width = Math.floor(innerWidth*dpr);
  canvas.height= Math.floor(innerHeight*dpr);
  ctx.setTransform(dpr,0,0,dpr,0,0);
}
addEventListener('resize', resize); resize();

// ===== Game Constants =====
const GRID = 30;
const PRAISES = ["شاطر يا سطا!","طرششش!"];
const BANK = [
  {ar:"قطة", en:"cat"}, {ar:"كلب", en:"dog"}, {ar:"تفاح", en:"apple"},
  {ar:"ماء", en:"water"}, {ar:"خبز", en:"bread"}, {ar:"بيت", en:"house"},
  {ar:"سيارة", en:"car"}, {ar:"كتاب", en:"book"}, {ar:"مدرسة", en:"school"},
  {ar:"باب", en:"door"}, {ar:"نافذة", en:"window"}, {ar:"مستشفى", en:"hospital"},
  {ar:"حديقة", en:"park"}, {ar:"أزرق", en:"blue"}, {ar:"أحمر", en:"red"},
  {ar:"طعام", en:"food"}, {ar:"شارع", en:"street"}, {ar:"كرسي", en:"chair"},
  {ar:"طاولة", en:"table"}, {ar:"حليب", en:"milk"},
];

// ===== Speech (mobile-safe) =====
let audioEnabled = false;
function enableAudio(){
  // One user gesture to unlock TTS on iOS/Android
  try{
    speechSynthesis.cancel();
    const warm = new SpeechSynthesisUtterance("جاهزين؟");
    warm.lang = "ar-EG";
    speechSynthesis.speak(warm);
    audioEnabled = true;
  }catch{}
}
function speakAR(text){
  if(!audioEnabled) return;
  try{
    const u = new SpeechSynthesisUtterance(text);
    u.lang = "ar-EG";
    u.rate = 1.02;
    const v = speechSynthesis.getVoices().find(v=> (v.lang||"").toLowerCase().startsWith("ar"));
    if (v) u.voice = v;
    speechSynthesis.cancel(); speechSynthesis.speak(u);
  }catch{}
}
function speakEN(text){
  if(!audioEnabled) return;
  try{
    const u = new SpeechSynthesisUtterance(text);
    u.lang = "en-US";
    u.rate = 0.98;
    const v = speechSynthesis.getVoices().find(v=> /^en-/.test((v.lang||"").toLowerCase()));
    if (v) u.voice = v;
    speechSynthesis.speak(u);
  }catch{}
}
setTimeout(()=>speechSynthesis.getVoices(), 500);

// ===== State =====
let state;
function newState(){
  const cols = Math.floor(innerWidth/GRID)-2;
  const rows = Math.floor(innerHeight/GRID)-9; // leave room for HUD + Dpad
  state = {
    cols, rows,
    snake:[{x:Math.floor(cols/2), y:Math.floor(rows/2)}],
    dir:{x:1,y:0}, queue:[],
    speed:140, last:0, over:false, paused:false, score:0,
    q:null, options:[], correctIndex:0, cheeses:[],
    playX: innerWidth/2 - (cols*GRID)/2,
    playY: Math.max(90, innerHeight*0.18),
    // D-pad layout
    dpad:{cx: innerWidth/2, cy: innerHeight-110, r: 85},
    // swipe
    touchStart:null
  };
  newRound();
}
function randInt(a,b){ return a + Math.floor(Math.random()*(b-a+1)); }
function shuffle(a){ for(let i=a.length-1;i>0;i--){ const j=Math.floor(Math.random()*(i+1)); [a[i],a[j]]=[a[j],a[i]]; } }
function onSnake(cell){ return state.snake.some(s=>s.x===cell.x && s.y===cell.y); }

function newRound(){
  const q = BANK[randInt(0,BANK.length-1)];
  const d1 = BANK[randInt(0,BANK.length-1)];
  let d2; do { d2 = BANK[randInt(0,BANK.length-1)]; } while (d2.en===q.en || d2.en===d1.en);
  const opts = [q.en, d1.en, d2.en]; shuffle(opts);
  const correctIndex = opts.indexOf(q.en);

  const pos=[];
  for(let i=0;i<3;i++){
    let c;
    do{
      c = {x:randInt(0,state.cols-1), y:randInt(0,state.rows-1)};
    } while (onSnake(c) || pos.some(p=>p.x===c.x&&p.y===c.y));
    pos.push(c);
  }
  state.q=q;
  state.options=opts;
  state.correctIndex=correctIndex;
  state.cheeses=pos.map((p,i)=>({x:p.x,y:p.y,label:opts[i]}));
}

// ===== Input: Keys (optional for desktop) =====
addEventListener('keydown', e=>{
  const map = {ArrowUp:[0,-1], ArrowDown:[0,1], ArrowLeft:[-1,0], ArrowRight:[1,0]};
  if (map[e.key]){ e.preventDefault(); enqueueDir(map[e.key][0], map[e.key][1]); }
  else if (e.key===' '){ e.preventDefault(); state.paused=!state.paused; }
  else if (e.key==='r'||e.key==='R'){ e.preventDefault(); newState(); }
});

// ===== Input: Touch (swipe + D-pad) =====
canvas.addEventListener('touchstart', (e)=>{
  e.preventDefault();
  const t = e.changedTouches[0];
  const p = {x:t.clientX, y:t.clientY, time:Date.now()};
  state.touchStart = p;

  // Audio unlock on first tap (on top banner button)
  const btn = getSoundBtnRect();
  if (pointInRect(p.x,p.y,btn)) { enableAudio(); return; }

  // D-pad immediate direction
  const dir = dpadDirection(p.x,p.y);
  if (dir) enqueueDir(dir.x,dir.y);
}, {passive:false});

canvas.addEventListener('touchmove', e=>{ e.preventDefault(); }, {passive:false});

canvas.addEventListener('touchend', (e)=>{
  e.preventDefault();
  const t = e.changedTouches[0];
  const end = {x:t.clientX, y:t.clientY, time:Date.now()};
  // D-pad tap?
  const dir = dpadDirection(end.x,end.y);
  if (dir){ enqueueDir(dir.x,dir.y); return; }
  // Swipe?
  const s = state.touchStart;
  if (!s) return;
  const dx = end.x - s.x, dy = end.y - s.y;
  const dt = end.time - s.time;
  if (dt < 500 && Math.hypot(dx,dy) > 30){
    if (Math.abs(dx) > Math.abs(dy)){
      enqueueDir(dx>0?1:-1, 0);
    } else {
      enqueueDir(0, dy>0?1:-1);
    }
  }
  state.touchStart = null;
}, {passive:false});

function enqueueDir(dx,dy){
  const cur = state.dir;
  if (cur.x+dx===0 && cur.y+dy===0) return; // no reverse
  state.queue.push({x:dx,y:dy});
}

// ===== Render helpers =====
function bg(){
  const w = innerWidth, h = innerHeight;
  const g = ctx.createLinearGradient(0,0,0,h);
  g.addColorStop(0,'#0a0e1a'); g.addColorStop(1,'#161b2e');
  ctx.fillStyle=g; ctx.fillRect(0,0,w,h);
  ctx.fillStyle='#e7ecff';
  ctx.font='700 22px system-ui,-apple-system,Segoe UI,Roboto';
  ctx.fillText('English × Mouse & Cheese (Mobile)', 14, 34);
  ctx.fillStyle='#b9c3ff';
  ctx.font='500 13px system-ui,-apple-system,Segoe UI,Roboto';
  ctx.fillText('اسحب أو استخدم الأسهم على الشاشة • لمس زر الصوت لتفعيل التشجيع', 14, 54);

  // Sound button
  const r = getSoundBtnRect();
  roundedRect(r.x, r.y, r.w, r.h, 10, audioEnabled?'#2aa36b':'#4b4f78', '#ffffff22');
  ctx.fillStyle='#eaf0ff';
  ctx.font='600 13px system-ui,-apple-system';
  ctx.textAlign='center'; ctx.textBaseline='middle';
  ctx.fillText(audioEnabled?'الصوت مُفعّل 🔊':'تفعيل الصوت 🔊', r.x+r.w/2, r.y+r.h/2);
  ctx.textAlign='left';
}
function getSoundBtnRect(){
  const w = 130, h = 32;
  return {x: innerWidth-14-w, y: 18, w, h};
}
function roundedRect(x,y,w,h,r,fill,stroke){
  ctx.beginPath();
  ctx.moveTo(x+r,y);
  ctx.arcTo(x+w,y,x+w,y+h,r);
  ctx.arcTo(x+w,y+h,x,y+h,r);
  ctx.arcTo(x,y+h,x,y,r);
  ctx.arcTo(x,y,x+w,y,r);
  if (fill){ ctx.fillStyle=fill; ctx.fill(); }
  if (stroke){ ctx.strokeStyle=stroke; ctx.stroke(); }
}
function drawCheese(px,py,label){
  roundedRect(px+3,py+3,GRID-6,GRID-6,6,'#ffd94d',null);
  ctx.fillStyle='#e5b700';
  ctx.beginPath(); ctx.arc(px+7,py+8,2,0,Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.arc(px+GRID-9,py+GRID-10,3,0,Math.PI*2); ctx.fill();
  ctx.fillStyle='#eaf0ff';
  ctx.font='600 12px system-ui,-apple-system';
  ctx.textAlign='center'; ctx.textBaseline='top';
  ctx.fillText(label, px+GRID/2, py+GRID+4);
}
function drawMouse(px,py,isHead){
  ctx.fillStyle = isHead ? '#ffd38a' : '#f0b56a';
  roundedRect(px+2,py+2,GRID-4,GRID-4,6,ctx.fillStyle,null);
  if (isHead){
    ctx.fillStyle='#6b4e2e';
    ctx.fillRect(px+GRID/2-3, py+4, 2, 2);
    ctx.fillRect(px+GRID/2+3, py+4, 2, 2);
    ctx.fillRect(px+GRID/2-1, y=py+GRID-8, 2, 2);
  }
}
function drawDpad(){
  const {cx,cy,r} = state.dpad;
  // base ring
  ctx.beginPath(); ctx.arc(cx,cy,r,0,Math.PI*2);
  ctx.fillStyle='rgba(255,255,255,0.07)'; ctx.fill();
  ctx.strokeStyle='#ffffff22'; ctx.stroke();
  // arrows
  arrow(cx, cy-r*0.6, 0,-1, '↑');
  arrow(cx, cy+r*0.6, 0, 1, '↓');
  arrow(cx-r*0.6, cy, -1,0, '←');
  arrow(cx+r*0.6, cy, 1, 0, '→');
}
function arrow(x,y,dx,dy,label){
  roundedRect(x-30,y-22,60,44,10,'rgba(255,255,255,0.10)','#ffffff22');
  ctx.fillStyle='#eaf0ff'; ctx.font='700 16px system-ui';
  ctx.textAlign='center'; ctx.textBaseline='middle';
  ctx.fillText(label, x, y);
}
function dpadDirection(x,y){
  const {cx,cy,r} = state.dpad;
  const dx = x-cx, dy = y-cy;
  if (Math.hypot(dx,dy) > r+10) return null;
  // pick nearest direction
  if (Math.abs(dx) > Math.abs(dy)){
    return {x: dx>0?1:-1, y:0};
  } else {
    return {x:0, y: dy>0?1:-1};
  }
}
function pointInRect(x,y,r){ return x>=r.x && x<=r.x+r.w && y>=r.y && y<=r.y+r.h; }

// ===== Game Loop =====
function step(ts){
  ctx.clearRect(0,0,canvas.width,canvas.height);
  bg();

  const px = state.playX, py = state.playY;
  const pw = state.cols*GRID, ph = state.rows*GRID;

  // Frame
  roundedRect(px-10,py-10,pw+20,ph+20,12,'rgba(255,255,255,0.06)','#ffffff22');

  // Word prompt
  ctx.fillStyle='#eaf0ff';
  ctx.font='700 26px system-ui,-apple-system';
  ctx.textAlign='center';
  ctx.fillText(state.q.ar, innerWidth/2, py-28);
  ctx.fillStyle='#b9c3ff';
  ctx.font='500 13px system-ui,-apple-system';
  ctx.fillText('كل قطعة جبنة تحمل ترجمة إنجليزية — الصحيحة تُعطي نقطة وتشجيع صوتي', innerWidth/2, py-8);

  // Grid
  ctx.strokeStyle='rgba(255,255,255,0.05)';
  for(let x=0;x<=state.cols;x++){ ctx.beginPath(); ctx.moveTo(px+x*GRID, py); ctx.lineTo(px+x*GRID, py+ph); ctx.stroke(); }
  for(let y=0;y<=state.rows;y++){ ctx.beginPath(); ctx.moveTo(px, py+y*GRID); ctx.lineTo(px+pw, py+y*GRID); ctx.stroke(); }

  // Tick
  if (!state.paused && !state.over && ts - state.last >= state.speed){
    state.last = ts;
    if (state.queue.length){
      const nd = state.queue.shift();
      const cur = state.dir;
      if (!(cur.x+nd.x===0 && cur.y+nd.y===0)) state.dir = nd;
    }
    const head = {x:state.snake[0].x+state.dir.x, y:state.snake[0].y+state.dir.y};
    // wrap
    if (head.x<0) head.x=state.cols-1; if (head.x>=state.cols) head.x=0;
    if (head.y<0) head.y=state.rows-1; if (head.y>=state.rows) head.y=0;

    if (state.snake.some(s=>s.x===head.x && s.y===head.y)){
      state.over = true;
    } else {
      state.snake.unshift(head);
      const hit = state.cheeses.findIndex(c=>c.x===head.x && c.y===head.y);
      if (hit>-1){
        const correct = (hit===state.correctIndex);
        if (correct){
          state.score++;
          if (audioEnabled){
            speakAR(PRAISES[randInt(0,PRAISES.length-1)]);
            setTimeout(()=>speakEN(state.q.en), 350);
          }
          state.speed = Math.max(70, state.speed-2);
          newRound(); // grow
        } else {
          // wrong: no grow (pop)
          state.snake.pop();
        }
      } else {
        state.snake.pop();
      }
    }
  }

  // Draw cheeses
  state.cheeses.forEach(c=>{
    drawCheese(px + c.x*GRID, py + c.y*GRID, c.label);
  });

  // Draw snake
  state.snake.forEach((s,i)=> drawMouse(px + s.x*GRID, py + s.y*GRID, i===0));

  // HUD
  ctx.textAlign='left'; ctx.fillStyle='#c8cff8'; ctx.font='600 15px system-ui,-apple-system';
  ctx.fillText(`Score: ${state.score}`, 14, innerHeight-18);

  // D-pad
  drawDpad();

  // Game Over Overlay
  if (state.over){
    ctx.fillStyle='rgba(0,0,0,0.55)';
    ctx.fillRect(px,py,pw,ph);
    ctx.fillStyle='#ffe9a8'; ctx.font='700 28px system-ui,-apple-system';
    ctx.textAlign='center';
    ctx.fillText('Game Over', px+pw/2, py+ph/2-8);
    ctx.fillStyle='#eaf0ff'; ctx.font='500 16px system-ui,-apple-system';
    ctx.fillText('لمسة واحدة لإعادة البدء', px+pw/2, py+ph/2+22);
  }

  requestAnimationFrame(step);
}
canvas.addEventListener('touchend', ()=>{
  if (state.over){ newState(); }
});

// ===== Start =====
newState();
requestAnimationFrame(step);
</script>
</body>
</html>
