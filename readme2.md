<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no"/>
  <title>CardioAR · Monitor</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/loaders/GLTFLoader.js"></script>
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Share+Tech+Mono&family=Barlow:wght@300;400;600;700;900&display=swap');

    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    :root {
      --red:    #ff2d55;
      --red-dim: #7a1528;
      --cyan:   #00e5ff;
      --green:  #00ff88;
      --yellow: #ffd60a;
      --dark:   #040810;
      --panel:  rgba(4,8,16,0.82);
      --border: rgba(0,229,255,0.18);
      --glow-c: 0 0 12px rgba(0,229,255,0.4);
      --glow-r: 0 0 14px rgba(255,45,85,0.5);
      --glow-g: 0 0 12px rgba(0,255,136,0.4);
    }

    html,body { width:100%; height:100%; overflow:hidden; background:#000; font-family:'Barlow',sans-serif; }

    /* ── câmera ──────────────────────────────────────────────── */
    #cam {
      position:fixed; inset:0; width:100%; height:100%;
      object-fit:cover; z-index:1; display:none;
    }

    /* ── Three.js canvas ─────────────────────────────────────── */
    #c {
      position:fixed; inset:0; width:100%; height:100%;
      z-index:2; display:none; pointer-events:auto;
    }

    /* ── Splash ──────────────────────────────────────────────── */
    #splash {
      position:fixed; inset:0; z-index:9999;
      display:flex; flex-direction:column;
      align-items:center; justify-content:center;
      background:var(--dark);
      transition:opacity .6s ease, visibility .6s ease;
    }
    #splash.out { opacity:0; visibility:hidden; pointer-events:none; }

    .splash-bg {
      position:absolute; inset:0; overflow:hidden;
    }
    .splash-bg::before {
      content:'';
      position:absolute; inset:0;
      background:
        radial-gradient(ellipse 60% 50% at 50% 50%, rgba(255,45,85,.08) 0%, transparent 70%),
        repeating-linear-gradient(0deg, transparent, transparent 39px, rgba(0,229,255,.03) 40px),
        repeating-linear-gradient(90deg, transparent, transparent 39px, rgba(0,229,255,.03) 40px);
    }

    /* ECG decorativo no splash */
    .ecg-deco {
      position:absolute; bottom:15%; left:0; right:0; height:60px;
      overflow:hidden; opacity:0.25;
    }
    .ecg-deco svg { animation:ecg-scroll 3s linear infinite; }
    @keyframes ecg-scroll { from{transform:translateX(0)} to{transform:translateX(-50%)} }

    .splash-inner { position:relative; text-align:center; padding:0 28px; max-width:420px; }

    .splash-chip {
      display:inline-flex; align-items:center; gap:8px;
      font-family:'Share Tech Mono',monospace; font-size:10px;
      letter-spacing:3px; text-transform:uppercase;
      color:var(--red); border:1px solid rgba(255,45,85,.35);
      padding:6px 16px; border-radius:2px; margin-bottom:36px;
      box-shadow:var(--glow-r);
    }
    .chip-dot { width:7px;height:7px; border-radius:50%; background:var(--red); box-shadow:var(--glow-r); animation:beat-chip .7s ease-in-out infinite alternate; }
    @keyframes beat-chip { from{transform:scale(1)} to{transform:scale(1.5)} }

    .splash-title {
      font-size:clamp(48px,12vw,80px); font-weight:900;
      line-height:.9; color:#fff; letter-spacing:-3px; margin-bottom:12px;
    }
    .splash-title em { font-style:normal; color:var(--red); text-shadow:var(--glow-r); }

    .splash-sub {
      font-family:'Share Tech Mono',monospace; font-size:11px;
      color:rgba(255,255,255,.35); letter-spacing:1.5px; line-height:1.9;
      margin-bottom:52px;
    }

    #btn-start {
      position:relative; display:inline-flex; align-items:center; gap:16px;
      background:transparent; border:1.5px solid var(--red);
      color:var(--red); font-family:'Share Tech Mono',monospace;
      font-size:12px; letter-spacing:4px; text-transform:uppercase;
      padding:16px 44px; cursor:pointer; border-radius:2px;
      box-shadow:var(--glow-r),inset 0 0 40px rgba(255,45,85,.04);
      transition:all .25s ease; overflow:hidden;
    }
    #btn-start::before {
      content:''; position:absolute; inset:0;
      background:var(--red); transform:scaleX(0); transform-origin:left;
      transition:transform .3s ease; z-index:0;
    }
    #btn-start:hover::before,#btn-start:active::before { transform:scaleX(1); }
    #btn-start:hover,#btn-start:active { color:#fff; box-shadow:none; }
    #btn-start span { position:relative; z-index:1; }
    .pulse-ring {
      position:relative; z-index:1;
      width:14px; height:14px; border-radius:50%;
      background:var(--red); box-shadow:var(--glow-r);
      animation:pulse-ring .9s ease-in-out infinite alternate;
    }
    @keyframes pulse-ring { to { transform:scale(1.6); opacity:.6; } }

    /* ── HUD overlay ─────────────────────────────────────────── */
    #hud {
      position:fixed; inset:0; z-index:8888;
      pointer-events:none; opacity:0;
      transition:opacity .5s ease;
    }
    #hud.on { opacity:1; }

    /* scanlines */
    #hud::after {
      content:''; position:absolute; inset:0;
      background:repeating-linear-gradient(
        0deg,transparent,transparent 2px,
        rgba(0,229,255,.008) 2px,rgba(0,229,255,.008) 4px);
      pointer-events:none;
    }

    /* cantos HUD */
    .hc { position:absolute; width:22px; height:22px; }
    .hc::before,.hc::after { content:''; position:absolute; background:var(--cyan); box-shadow:var(--glow-c); }
    .hc::before { width:100%; height:1.5px; top:0; left:0; }
    .hc::after  { width:1.5px; height:100%; top:0; left:0; }
    .hc.tl{top:16px;left:16px}
    .hc.tr{top:16px;right:16px;transform:scaleX(-1)}
    .hc.bl{bottom:16px;left:16px;transform:scaleY(-1)}
    .hc.br{bottom:16px;right:16px;transform:scale(-1)}

    /* barra topo */
    .hud-top {
      position:absolute; top:0; left:0; right:0;
      padding:10px 20px;
      display:flex; align-items:center; justify-content:space-between;
      border-bottom:1px solid var(--border);
      background:linear-gradient(to bottom,rgba(4,8,16,.7),transparent);
    }
    .hud-logo {
      font-family:'Share Tech Mono',monospace; font-size:11px;
      letter-spacing:4px; text-transform:uppercase; color:var(--cyan);
    }
    .hud-logo span { color:var(--red); }
    .rec-badge {
      display:flex; align-items:center; gap:7px;
      font-family:'Share Tech Mono',monospace; font-size:9px;
      letter-spacing:3px; color:rgba(255,45,85,.8);
    }
    .rec-dot { width:6px;height:6px;border-radius:50%;background:var(--red);box-shadow:var(--glow-r);animation:blink 1s step-end infinite; }
    @keyframes blink{50%{opacity:0}}
    #ts { font-family:'Share Tech Mono',monospace; font-size:9px; color:rgba(0,229,255,.5); letter-spacing:2px; }

    /* ── Painéis laterais ────────────────────────────────────── */
    .panel {
      position:absolute;
      background:var(--panel);
      border:1px solid var(--border);
      backdrop-filter:blur(14px);
      border-radius:4px;
      padding:14px 16px;
      pointer-events:auto;
    }

    /* Painel BPM — esquerda */
    #panel-bpm {
      top:50%; left:12px; transform:translateY(-50%);
      width:130px;
    }

    /* Painel SpO2 — direita */
    #panel-spo2 {
      top:50%; right:12px; transform:translateY(-50%);
      width:130px;
    }

    .p-label {
      font-family:'Share Tech Mono',monospace; font-size:9px;
      letter-spacing:3px; text-transform:uppercase;
      color:rgba(255,255,255,.4); margin-bottom:4px;
    }

    .p-value {
      font-family:'Share Tech Mono',monospace;
      font-size:44px; font-weight:bold; line-height:1;
      margin-bottom:2px; transition:color .4s ease, text-shadow .4s ease;
    }

    .p-unit {
      font-family:'Share Tech Mono',monospace; font-size:10px;
      color:rgba(255,255,255,.35); letter-spacing:2px;
    }

    .p-status {
      display:inline-block; margin-top:10px;
      font-family:'Share Tech Mono',monospace; font-size:9px;
      letter-spacing:2px; text-transform:uppercase;
      padding:3px 8px; border-radius:2px;
      transition:all .4s ease;
    }

    /* Mini ECG dentro do painel BPM */
    .mini-ecg { margin-top:12px; height:36px; overflow:hidden; }
    .mini-ecg canvas { width:100%; height:36px; }

    /* Barra de saturaçao SpO2 */
    .spo2-bar-wrap { margin-top:12px; }
    .spo2-bar-bg {
      width:100%; height:6px; border-radius:3px;
      background:rgba(255,255,255,.08); overflow:hidden;
    }
    .spo2-bar-fill {
      height:100%; border-radius:3px;
      transition:width .6s ease, background .6s ease;
    }
    .spo2-ticks {
      display:flex; justify-content:space-between;
      margin-top:4px;
      font-family:'Share Tech Mono',monospace; font-size:8px;
      color:rgba(255,255,255,.2);
    }

    /* Barra inferior */
    .hud-bot {
      position:absolute; bottom:0; left:0; right:0;
      padding:10px 16px;
      display:flex; align-items:center; justify-content:space-between;
      border-top:1px solid var(--border);
      background:linear-gradient(to top,rgba(4,8,16,.75),transparent);
      pointer-events:auto;
    }

    /* Controles */
    .ctrl-group { display:flex; gap:8px; align-items:center; }
    .cbtn {
      background:rgba(4,8,16,.7); backdrop-filter:blur(8px);
      border:1px solid var(--border); color:rgba(0,229,255,.7);
      font-family:'Share Tech Mono',monospace; font-size:18px;
      width:40px; height:40px; border-radius:50%;
      display:flex; align-items:center; justify-content:center;
      cursor:pointer; transition:all .15s ease;
      -webkit-tap-highlight-color:transparent;
    }
    .cbtn:active { background:var(--cyan); color:var(--dark); transform:scale(.92); }

    #btn-reset-hud {
      background:rgba(4,8,16,.7); backdrop-filter:blur(8px);
      border:1px solid rgba(255,45,85,.25); color:rgba(255,45,85,.7);
      font-family:'Share Tech Mono',monospace; font-size:10px;
      letter-spacing:2px; text-transform:uppercase;
      padding:10px 18px; cursor:pointer; border-radius:2px;
      transition:all .2s ease; -webkit-tap-highlight-color:transparent;
    }
    #btn-reset-hud:active { border-color:var(--red); color:var(--red); box-shadow:var(--glow-r); }

    #sim-label {
      font-family:'Share Tech Mono',monospace; font-size:9px;
      letter-spacing:2px; color:rgba(255,210,10,.5);
    }

    /* ── Toast ───────────────────────────────────────────────── */
    #toast {
      position:fixed; top:58px; left:50%;
      transform:translateX(-50%) translateY(-12px);
      background:rgba(4,8,16,.9); backdrop-filter:blur(12px);
      border:1px solid var(--cyan); color:var(--cyan);
      font-family:'Share Tech Mono',monospace; font-size:10px;
      letter-spacing:2px; padding:10px 20px; border-radius:2px;
      box-shadow:var(--glow-c); z-index:9999; opacity:0;
      transition:all .3s cubic-bezier(.4,0,.2,1);
      pointer-events:none; white-space:nowrap;
    }
    #toast.show { opacity:1; transform:translateX(-50%) translateY(0); }

    /* ── Loader ──────────────────────────────────────────────── */
    #loader {
      position:fixed; inset:0; z-index:9998;
      display:none; flex-direction:column;
      align-items:center; justify-content:center;
      background:rgba(4,8,16,.88); backdrop-filter:blur(8px);
    }
    #loader.on { display:flex; }
    .spin {
      width:40px;height:40px;border-radius:50%;
      border:2px solid rgba(255,45,85,.2); border-top-color:var(--red);
      animation:spin .7s linear infinite; margin-bottom:14px;
      box-shadow:var(--glow-r);
    }
    @keyframes spin{to{transform:rotate(360deg)}}
    .load-txt { font-family:'Share Tech Mono',monospace; font-size:10px; letter-spacing:3px; color:rgba(255,45,85,.6); }

    /* ── Alerta crítico ──────────────────────────────────────── */
    #alert-overlay {
      position:fixed; inset:0; z-index:9990; pointer-events:none;
      border:3px solid var(--red);
      box-shadow:inset 0 0 60px rgba(255,45,85,.15);
      opacity:0; transition:opacity .3s ease;
    }
    #alert-overlay.on { opacity:1; animation:alert-pulse .6s ease-in-out infinite alternate; }
    @keyframes alert-pulse { to { box-shadow:inset 0 0 100px rgba(255,45,85,.35); } }
  </style>
</head>
<body>

<video id="cam" autoplay playsinline muted></video>
<canvas id="c"></canvas>

<!-- Loader -->
<div id="loader"><div class="spin"></div><div class="load-txt">Carregando...</div></div>

<!-- Alerta crítico -->
<div id="alert-overlay"></div>

<!-- ═══ SPLASH ════════════════════════════════════════════════ -->
<div id="splash">
  <div class="splash-bg">
    <!-- ECG decorativo -->
    <div class="ecg-deco">
      <svg width="200%" height="60" viewBox="0 0 1200 60" preserveAspectRatio="none" fill="none">
        <polyline points="
          0,30 100,30 120,30 130,5 140,55 150,30 160,30
          260,30 280,30 290,5 300,55 310,30 320,30
          420,30 440,30 450,5 460,55 470,30 480,30
          580,30 600,30 610,5 620,55 630,30 640,30
          740,30 760,30 770,5 780,55 790,30 800,30
          900,30 920,30 930,5 940,55 950,30 960,30
          1060,30 1080,30 1090,5 1100,55 1110,30 1200,30
        " stroke="rgba(255,45,85,0.6)" stroke-width="1.5"/>
      </svg>
    </div>
  </div>
  <div class="splash-inner">
    <div class="splash-chip">
      <div class="chip-dot"></div>
      Monitor AR · Tempo Real
    </div>
    <h1 class="splash-title">Cardio<em>AR</em></h1>
    <p class="splash-sub">
      Oximetria · Frequência Cardíaca<br/>
      Visualização 3D em Realidade Aumentada
    </p>
    <button id="btn-start">
      <span>Iniciar Monitor</span>
      <div class="pulse-ring"></div>
    </button>
  </div>
</div>

<!-- ═══ HUD ══════════════════════════════════════════════════ -->
<div id="hud">
  <div class="hc tl"></div><div class="hc tr"></div>
  <div class="hc bl"></div><div class="hc br"></div>

  <!-- Topo -->
  <div class="hud-top">
    <div class="hud-logo">Cardio<span>AR</span></div>
    <div class="rec-badge"><div class="rec-dot"></div>SIMULADO</div>
    <div id="ts">--:--:--</div>
  </div>

  <!-- Painel BPM -->
  <div class="panel" id="panel-bpm">
    <div class="p-label">Freq. Cardíaca</div>
    <div class="p-value" id="val-bpm">--</div>
    <div class="p-unit">BPM</div>
    <div class="p-status" id="st-bpm">—</div>
    <div class="mini-ecg"><canvas id="ecg-canvas" height="36"></canvas></div>
  </div>

  <!-- Painel SpO2 -->
  <div class="panel" id="panel-spo2">
    <div class="p-label">SpO₂</div>
    <div class="p-value" id="val-spo2">--</div>
    <div class="p-unit">% SAT O₂</div>
    <div class="p-status" id="st-spo2">—</div>
    <div class="spo2-bar-wrap">
      <div class="spo2-bar-bg"><div class="spo2-bar-fill" id="spo2-bar" style="width:0%"></div></div>
      <div class="spo2-ticks"><span>80</span><span>90</span><span>100</span></div>
    </div>
  </div>

  <!-- Barra inferior -->
  <div class="hud-bot">
    <div class="ctrl-group">
      <button class="cbtn" id="btn-minus" title="Afastar">−</button>
      <button class="cbtn" id="btn-plus"  title="Aproximar">+</button>
    </div>
    <div id="sim-label">⚠ DADOS SIMULADOS</div>
    <button id="btn-reset-hud">↺ Reset</button>
  </div>
</div>

<div id="toast"></div>

<!-- ═══ SCRIPTS ══════════════════════════════════════════════ -->
<script>
/* ───────────────────────────────────────────────────────────
   CONFIGURAÇÃO
─────────────────────────────────────────────────────────── */
const GLB_PATH    = 'realistic_human_heart.glb';
const BASE_SCALE  = 0.3;
let   modelDist   = -2.2;

/* ── DOM refs ─────────────────────────────────────────────── */
const camEl     = document.getElementById('cam');
const canvasEl  = document.getElementById('c');
const splash    = document.getElementById('splash');
const hud       = document.getElementById('hud');
const loaderEl  = document.getElementById('loader');
const toastEl   = document.getElementById('toast');
const alertEl   = document.getElementById('alert-overlay');
const tsEl      = document.getElementById('ts');

const valBpm    = document.getElementById('val-bpm');
const stBpm     = document.getElementById('st-bpm');
const valSpo2   = document.getElementById('val-spo2');
const stSpo2    = document.getElementById('st-spo2');
const spo2Bar   = document.getElementById('spo2-bar');
const ecgCanvas = document.getElementById('ecg-canvas');

/* ── Toast ─────────────────────────────────────────────────── */
let toastTimer;
function toast(msg, dur=2500){
  clearTimeout(toastTimer);
  toastEl.textContent=msg; toastEl.classList.add('show');
  toastTimer=setTimeout(()=>toastEl.classList.remove('show'),dur);
}

/* ═══════════════════════════════════════════════════════════
   MOCK ENGINE — sensor simulado
═══════════════════════════════════════════════════════════ */
const sensor = {
  bpm:  70,
  spo2: 98,
  ts:   Date.now(),

  /* limites fisiológicos */
  BPM_MIN: 40, BPM_MAX: 130,
  SPO2_MIN: 82, SPO2_MAX: 100,

  /* variação por ciclo */
  bpmDrift:  0,
  spo2Drift: 0,

  tick() {
    const now = Date.now();
    this.ts = Math.floor(now / 1000);

    /* BPM — caminhada aleatória suave */
    this.bpmDrift  += (Math.random()-.5) * 1.2;
    this.bpmDrift   = Math.max(-3, Math.min(3, this.bpmDrift));
    this.bpm       += this.bpmDrift + (Math.random()-.5)*.8;
    this.bpm        = Math.max(this.BPM_MIN, Math.min(this.BPM_MAX, this.bpm));

    /* SpO2 — deriva mais lenta */
    this.spo2Drift += (Math.random()-.5) * .3;
    this.spo2Drift  = Math.max(-.6, Math.min(.6, this.spo2Drift));
    this.spo2      += this.spo2Drift + (Math.random()-.5)*.2;
    this.spo2       = Math.max(this.SPO2_MIN, Math.min(this.SPO2_MAX, this.spo2));

    return {
      timestamp: this.ts,
      bpm:  Math.round(this.bpm),
      spo2: parseFloat(this.spo2.toFixed(1))
    };
  },

  /* classificação de status */
  classifyBpm(v) {
    if (v < 60)  return { label:'BAIXO',  color:'#ffd60a', state:'warn' };
    if (v > 100) return { label:'ALTO',   color:'#ff9500', state:'warn' };
    return            { label:'NORMAL',  color:'#00ff88', state:'ok'   };
  },
  classifySpo2(v) {
    if (v < 90)  return { label:'CRÍTICO', color:'#ff2d55', state:'crit' };
    if (v < 95)  return { label:'ALERTA',  color:'#ffd60a', state:'warn' };
    return            { label:'NORMAL',   color:'#00ff88', state:'ok'   };
  }
};

/* ═══════════════════════════════════════════════════════════
   MINI ECG — canvas simples
═══════════════════════════════════════════════════════════ */
const ECG_W = 200, ECG_H = 36;
ecgCanvas.width  = ECG_W;
ecgCanvas.height = ECG_H;
const ectx = ecgCanvas.getContext('2d');

const ecgBuffer = new Array(ECG_W).fill(ECG_H/2);
let ecgPhase = 0;

function ecgWaveform(phase) {
  /* simula onda PQRST simplificada */
  const t = phase % 1;
  if (t < .1)  return ECG_H/2;
  if (t < .15) return ECG_H/2 - (t-.1)/.05 * 4;
  if (t < .2)  return ECG_H/2 - 4 + (t-.15)/.05 * 4;
  if (t < .22) return ECG_H/2 - (t-.2)/.02 * 14;
  if (t < .25) return ECG_H/2 - 14 + (t-.22)/.03 * 28;
  if (t < .30) return ECG_H/2 + 14 - (t-.25)/.05 * 20;
  if (t < .38) return ECG_H/2 - 6 + (t-.30)/.08 * 6;
  if (t < .55) return ECG_H/2 - (t-.38)/.17 * 3;
  if (t < .7)  return ECG_H/2 - 3 + (t-.55)/.15 * 3;
  return ECG_H/2;
}

function drawEcg(color='#ff2d55') {
  ectx.clearRect(0, 0, ECG_W, ECG_H);
  ectx.strokeStyle = color;
  ectx.lineWidth   = 1.5;
  ectx.shadowColor = color;
  ectx.shadowBlur  = 4;
  ectx.beginPath();
  ecgBuffer.forEach((v,i) => i===0 ? ectx.moveTo(i,v) : ectx.lineTo(i,v));
  ectx.stroke();
}

function stepEcg(bpm) {
  const speed = bpm / 60 / 30; /* avança mais rápido com BPM maior */
  ecgPhase += speed;
  ecgBuffer.shift();
  ecgBuffer.push(ecgWaveform(ecgPhase));
}

/* ═══════════════════════════════════════════════════════════
   UI UPDATE
═══════════════════════════════════════════════════════════ */
function updateUI(data) {
  const { bpm, spo2 } = data;
  const cb = sensor.classifyBpm(bpm);
  const cs = sensor.classifySpo2(spo2);

  /* timestamp */
  const d = new Date(data.timestamp * 1000);
  tsEl.textContent = d.toLocaleTimeString('pt-BR');

  /* BPM */
  valBpm.textContent = bpm;
  valBpm.style.color = cb.color;
  valBpm.style.textShadow = `0 0 16px ${cb.color}88`;
  stBpm.textContent  = cb.label;
  stBpm.style.color  = cb.color;
  stBpm.style.background = cb.color + '22';
  stBpm.style.borderColor = cb.color + '44';
  stBpm.style.border = `1px solid ${cb.color}44`;

  /* SpO2 */
  valSpo2.textContent = spo2.toFixed(1);
  valSpo2.style.color = cs.color;
  valSpo2.style.textShadow = `0 0 16px ${cs.color}88`;
  stSpo2.textContent  = cs.label;
  stSpo2.style.color  = cs.color;
  stSpo2.style.background = cs.color + '22';
  stSpo2.style.border = `1px solid ${cs.color}44`;

  /* barra SpO2: mapeia 80–100 → 0–100% */
  const barPct = Math.max(0, Math.min(100, (spo2 - 80) / 20 * 100));
  spo2Bar.style.width = barPct + '%';
  spo2Bar.style.background = cs.color;

  /* alerta crítico na borda */
  if (cs.state === 'crit' || cb.state === 'crit') {
    alertEl.classList.add('on');
  } else {
    alertEl.classList.remove('on');
  }
}

/* ═══════════════════════════════════════════════════════════
   THREE.JS
═══════════════════════════════════════════════════════════ */
let renderer, scene, threeCamera, pivot;
let modelLoaded = false;
let lastBeat = 0;
let beatDur  = 0; /* ms por batimento */
let currentBpm = 70;

function initThree() {
  renderer = new THREE.WebGLRenderer({ canvas: canvasEl, alpha: true, antialias: true });
  renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
  renderer.setSize(innerWidth, innerHeight);
  renderer.setClearColor(0x000000, 0);

  scene = new THREE.Scene();

  threeCamera = new THREE.PerspectiveCamera(70, innerWidth/innerHeight, 0.01, 100);
  threeCamera.position.set(0, 0, 0);

  /* Luzes */
  scene.add(new THREE.AmbientLight(0xffffff, 0.6));
  const d1 = new THREE.DirectionalLight(0xff4466, 1.4);
  d1.position.set(2,3,3); scene.add(d1);
  const d2 = new THREE.DirectionalLight(0xff9999, 0.5);
  d2.position.set(-2,-1,-1); scene.add(d2);

  window.addEventListener('resize', () => {
    threeCamera.aspect = innerWidth/innerHeight;
    threeCamera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
  });
}

function loadModel() {
  loaderEl.classList.add('on');
  const loader3d = new THREE.GLTFLoader();
  loader3d.load(
    GLB_PATH,
    (gltf) => {
      loaderEl.classList.remove('on');
      const model = gltf.scene;

      /* auto-escala */
      const box  = new THREE.Box3().setFromObject(model);
      const size = new THREE.Vector3();
      box.getSize(size);
      const s = BASE_SCALE / Math.max(size.x, size.y, size.z);
      model.scale.setScalar(s);
      const center = new THREE.Vector3();
      box.getCenter(center).multiplyScalar(s);
      model.position.sub(center);

      pivot = new THREE.Group();
      pivot.add(model);
      pivot.position.set(0, 0, modelDist);
      scene.add(pivot);
      modelLoaded = true;

      toast('✦ Coração 3D carregado');
      startLoop();
    },
    null,
    () => {
      loaderEl.classList.remove('on');
      toast('⚠ GLB não encontrado — fallback ativo');
      useFallback();
    }
  );
}

function useFallback() {
  pivot = new THREE.Group();

  /* coração simplificado com esferas */
  const mat = new THREE.MeshStandardMaterial({ color:0xcc2244, metalness:.3, roughness:.6, emissive:0x440011, emissiveIntensity:.4 });
  const s1 = new THREE.Mesh(new THREE.SphereGeometry(.14,16,16), mat);
  s1.position.set(-.08,.06,0);
  const s2 = new THREE.Mesh(new THREE.SphereGeometry(.14,16,16), mat);
  s2.position.set(.08,.06,0);
  const s3 = new THREE.Mesh(new THREE.SphereGeometry(.18,16,16), mat);
  s3.position.set(0,-.04,0);

  pivot.add(s1,s2,s3);
  pivot.position.set(0,0,modelDist);
  scene.add(pivot);
  modelLoaded = true;
  startLoop();
}

/* ─── pulsação sincronizada com BPM ─────────────────────── */
function heartbeatScale(now, bpm) {
  beatDur = 60000 / bpm; /* ms por batimento */
  const phase = ((now - lastBeat) % beatDur) / beatDur;
  /* sístole rápida (0–20%) depois diástole (20–100%) */
  let scale;
  if (phase < .2) {
    scale = 1 + Math.sin(phase/.2 * Math.PI) * .12;
  } else {
    scale = 1 + Math.sin((phase-.2)/.8 * Math.PI) * .03;
  }
  return scale;
}

/* ─── cor do modelo por SpO2 ─────────────────────────────── */
let lastSpo2Color = null;
function applyBloodColor(spo2) {
  let hex;
  if      (spo2 >= 95) hex = 0xcc2244; /* vermelho vivo — normal */
  else if (spo2 >= 90) hex = 0xaa3322; /* alaranjado — alerta */
  else                 hex = 0x552244; /* roxo escuro — crítico */

  if (hex === lastSpo2Color) return;
  lastSpo2Color = hex;

  pivot.traverse(obj => {
    if (obj.isMesh) {
      obj.material.color.setHex(hex);
      obj.material.emissive.setHex(hex >> 2);
    }
  });
}

/* ─── loop de renderização (único loop, sem duplicatas) ──── */
let loopRunning = false;
function startLoop() {
  if (loopRunning) return;
  loopRunning = true;
  function tick() {
    requestAnimationFrame(tick);
    if (!pivot || !renderer) return;
    const now = performance.now();
    const sc = heartbeatScale(now, currentBpm);
    pivot.scale.setScalar(sc);
    renderer.render(scene, threeCamera);
  }
  tick();
}

/* ═══════════════════════════════════════════════════════════
   CÂMERA — com limpeza de stream anterior
═══════════════════════════════════════════════════════════ */
function stopCamera() {
  /* para todas as tracks para liberar o hardware */
  if (camEl.srcObject) {
    camEl.srcObject.getTracks().forEach(t => t.stop());
    camEl.srcObject = null;
  }
}

async function startCamera() {
  stopCamera(); /* garante que não há stream preso */

  const tries = [
    { video: { facingMode: { exact: 'environment' }, width: { ideal: 1280 }, height: { ideal: 720 } } },
    { video: { facingMode: 'environment' } },
    { video: true }
  ];

  for (const c of tries) {
    try {
      const stream = await navigator.mediaDevices.getUserMedia(c);
      camEl.srcObject = stream;
      /* câmera traseira: sem espelhamento */
      const isRear = JSON.stringify(c).includes('environment');
      camEl.style.transform = isRear ? 'none' : 'scaleX(-1)';
      return true;
    } catch(e) { /* tenta próxima opção */ }
  }
  return false;
}

/* ═══════════════════════════════════════════════════════════
   LOOP DE MOCK — 750ms (com limpeza de intervalo anterior)
═══════════════════════════════════════════════════════════ */
let mockInterval = null;
function stopMock() {
  if (mockInterval) { clearInterval(mockInterval); mockInterval = null; }
}
function startMock() {
  stopMock();
  mockInterval = setInterval(() => {
    const data = sensor.tick();
    currentBpm = data.bpm;
    updateUI(data);
    stepEcg(data.bpm);
    drawEcg(sensor.classifyBpm(data.bpm).color);
    if (pivot) applyBloodColor(data.spo2);
  }, 750);
}

/* ═══════════════════════════════════════════════════════════
   BOTÃO ÚNICO — inicia tudo (robusto a múltiplos cliques)
═══════════════════════════════════════════════════════════ */
let started = false;

document.getElementById('btn-start').addEventListener('click', async () => {
  if (started) return; /* evita dupla execução */
  started = true;

  splash.classList.add('out');
  hud.classList.add('on');
  canvasEl.style.display = 'block';
  camEl.style.display    = 'block';

  /* inicializa Three.js apenas uma vez */
  if (!renderer) initThree();

  const ok = await startCamera();
  if (!ok) {
    toast('⚠ Câmera bloqueada — verifique as permissões');
    started = false; /* permite tentar de novo */
    return;
  }

  /* função que dispara quando o vídeo está pronto para reprodução */
  function onReady() {
    camEl.removeEventListener('loadedmetadata', onReady);
    camEl.removeEventListener('loadeddata',     onReady);
    camEl.play().catch(() => {});
    toast('✦ Monitor ativo — sincronizando');
    if (!pivot) loadModel(); /* carrega modelo só uma vez */
    startMock();
  }

  /* readyState >= 1 = metadados já disponíveis (cache/reconexão) */
  if (camEl.readyState >= 1) {
    onReady();
  } else {
    /* espera tanto loadedmetadata quanto loadeddata para garantir */
    camEl.addEventListener('loadedmetadata', onReady, { once: true });
    camEl.addEventListener('loadeddata',     onReady, { once: true });
  }
});

/* ═══════════════════════════════════════════════════════════
   CONTROLES HUD
═══════════════════════════════════════════════════════════ */
document.getElementById('btn-plus').addEventListener('click', () => {
  if (!pivot) return;
  modelDist = Math.min(modelDist + .3, -.5);
  pivot.position.z = modelDist;
});
document.getElementById('btn-minus').addEventListener('click', () => {
  if (!pivot) return;
  modelDist = Math.max(modelDist - .3, -8);
  pivot.position.z = modelDist;
});
document.getElementById('btn-reset-hud').addEventListener('click', () => {
  if (!pivot) return;
  modelDist = -2.2;
  pivot.position.set(0, 0, modelDist);
  pivot.rotation.set(0, 0, 0);
  toast('↺ Posição resetada');
});

/* ─── Arraste para rotacionar ─────────────────────────────── */
let drag=false, lx=0, ly=0;
canvasEl.addEventListener('touchstart', e=>{ drag=true; lx=e.touches[0].clientX; ly=e.touches[0].clientY; },{passive:true});
canvasEl.addEventListener('touchmove',  e=>{
  if(!drag||!pivot) return;
  pivot.rotation.y += (e.touches[0].clientX-lx)*.012;
  pivot.rotation.x += (e.touches[0].clientY-ly)*.012;
  lx=e.touches[0].clientX; ly=e.touches[0].clientY;
},{passive:true});
canvasEl.addEventListener('touchend', ()=>drag=false);

canvasEl.addEventListener('mousedown', e=>{ drag=true; lx=e.clientX; ly=e.clientY; });
canvasEl.addEventListener('mousemove', e=>{
  if(!drag||!pivot) return;
  pivot.rotation.y += (e.clientX-lx)*.012;
  pivot.rotation.x += (e.clientY-ly)*.012;
  lx=e.clientX; ly=e.clientY;
});
canvasEl.addEventListener('mouseup', ()=>drag=false);

/* ─── Pinch zoom ──────────────────────────────────────────── */
let lp=0;
canvasEl.addEventListener('touchstart', e=>{ if(e.touches.length===2) lp=Math.hypot(e.touches[0].clientX-e.touches[1].clientX,e.touches[0].clientY-e.touches[1].clientY); },{passive:true});
canvasEl.addEventListener('touchmove', e=>{
  if(e.touches.length===2 && pivot){
    const d=Math.hypot(e.touches[0].clientX-e.touches[1].clientX,e.touches[0].clientY-e.touches[1].clientY);
    const base = pivot.scale.x + (d-lp)*.003;
    pivot.scale.setScalar(Math.max(.05,Math.min(base,4)));
    lp=d;
  }
},{passive:true});

/* ─── Reativa câmera ao voltar ao app (iOS/Android matam stream) ── */
document.addEventListener('visibilitychange', async () => {
  if (document.visibilityState !== 'visible' || !started) return;
  const tracks = camEl.srcObject ? camEl.srcObject.getVideoTracks() : [];
  const streamDead = tracks.length === 0 || tracks[0].readyState === 'ended';
  if (streamDead) {
    toast('↺ Reconectando câmera...');
    const ok = await startCamera();
    if (ok && camEl.readyState >= 1) camEl.play().catch(() => {});
  }
});
</script>
</body>
</html>