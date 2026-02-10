<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no"/>
  <title>AR Experience</title>

  <!-- Three.js + GLTFLoader via CDN -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/loaders/GLTFLoader.js"></script>

  <style>
    @import url('https://fonts.googleapis.com/css2?family=Space+Mono:wght@400;700&family=Syne:wght@400;700;800&display=swap');

    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    :root {
      --neon:  #00ffe0;
      --neon2: #ff2d78;
      --dark:  #060610;
      --glass: rgba(6,6,22,0.75);
      --glow:  0 0 18px rgba(0,255,224,.5), 0 0 40px rgba(0,255,224,.2);
      --glow2: 0 0 18px rgba(255,45,120,.5);
    }

    html, body { width:100%; height:100%; overflow:hidden; background:#000; font-family:'Syne',sans-serif; }

    /* ── câmera ao fundo ──────────────────────────────────────── */
    #cam {
      position:fixed; inset:0;
      width:100%; height:100%;
      object-fit:cover;
      z-index:1;
      display:none;
      transform: scaleX(-1); /* espelha câmera frontal; remova se usar traseira */
    }

    /* ── canvas Three.js sobre o vídeo ───────────────────────── */
    #c {
      position:fixed; inset:0;
      width:100%; height:100%;
      z-index:2;
      display:none;
      pointer-events:none;
    }

    /* ── Splash ───────────────────────────────────────────────── */
    #splash {
      position:fixed; inset:0; z-index:9999;
      display:flex; flex-direction:column;
      align-items:center; justify-content:center;
      background:var(--dark);
      transition:opacity .7s ease, visibility .7s ease;
    }
    #splash.hidden { opacity:0; visibility:hidden; pointer-events:none; }

    .sg {
      position:absolute; inset:0;
      background-image:
        linear-gradient(rgba(0,255,224,.04) 1px,transparent 1px),
        linear-gradient(90deg,rgba(0,255,224,.04) 1px,transparent 1px);
      background-size:48px 48px;
      mask-image:radial-gradient(ellipse 80% 80% at 50% 50%,black 30%,transparent 100%);
    }
    .sc { position:relative; text-align:center; padding:0 28px; max-width:480px; }

    .badge {
      display:inline-block;
      font-family:'Space Mono',monospace; font-size:10px;
      letter-spacing:4px; text-transform:uppercase;
      color:var(--neon); border:1px solid rgba(0,255,224,.3);
      padding:6px 16px; border-radius:2px; margin-bottom:32px;
      box-shadow:var(--glow); animation:pulse 2.4s ease-in-out infinite;
    }
    @keyframes pulse { 0%,100%{opacity:1} 50%{opacity:.5} }

    h1 {
      font-size:clamp(38px,10vw,70px); font-weight:800;
      line-height:1; color:#fff; letter-spacing:-2px; margin-bottom:14px;
    }
    h1 span { color:var(--neon); text-shadow:var(--glow); }

    p {
      font-family:'Space Mono',monospace; font-size:11px;
      color:rgba(255,255,255,.4); letter-spacing:1px;
      line-height:1.9; margin-bottom:48px;
    }

    #btn-start {
      position:relative; display:inline-flex; align-items:center; gap:14px;
      background:transparent; border:1.5px solid var(--neon);
      color:var(--neon); font-family:'Space Mono',monospace;
      font-size:13px; letter-spacing:3px; text-transform:uppercase;
      padding:18px 40px; cursor:pointer; border-radius:2px;
      box-shadow:var(--glow); transition:all .25s ease; overflow:hidden;
    }
    #btn-start::before {
      content:''; position:absolute; inset:0;
      background:var(--neon); transform:scaleX(0); transform-origin:left;
      transition:transform .3s cubic-bezier(.4,0,.2,1); z-index:0;
    }
    #btn-start:hover::before, #btn-start:active::before { transform:scaleX(1); }
    #btn-start:hover, #btn-start:active { color:var(--dark); box-shadow:none; }
    #btn-start span { position:relative; z-index:1; }
    .arr {
      position:relative; z-index:1;
      width:10px; height:10px;
      border-right:2px solid currentColor; border-top:2px solid currentColor;
      transform:rotate(45deg);
    }

    /* ── HUD ──────────────────────────────────────────────────── */
    #hud {
      position:fixed; inset:0; z-index:8888;
      pointer-events:none; opacity:0;
      transition:opacity .6s ease;
    }
    #hud.on { opacity:1; }
    #hud::after {
      content:''; position:absolute; inset:0;
      background:repeating-linear-gradient(
        0deg,transparent,transparent 2px,
        rgba(0,255,224,.01) 2px,rgba(0,255,224,.01) 4px);
      pointer-events:none;
    }

    /* cantos */
    .co { position:absolute; width:26px; height:26px; }
    .co::before,.co::after { content:''; position:absolute; background:var(--neon); box-shadow:var(--glow); }
    .co::before { width:100%; height:2px; top:0; left:0; }
    .co::after  { width:2px; height:100%; top:0; left:0; }
    .co.tl{top:18px;left:18px}
    .co.tr{top:18px;right:18px;transform:scaleX(-1)}
    .co.bl{bottom:18px;left:18px;transform:scaleY(-1)}
    .co.br{bottom:18px;right:18px;transform:scale(-1)}

    /* topo */
    .htop {
      position:absolute; top:18px; left:50%; transform:translateX(-50%);
      display:flex; align-items:center; gap:10px;
    }
    .hlabel { font-family:'Space Mono',monospace; font-size:10px; letter-spacing:3px; text-transform:uppercase; color:rgba(0,255,224,.7); }
    .hdot   { width:6px; height:6px; border-radius:50%; background:var(--neon2); box-shadow:var(--glow2); animation:blink 1s step-end infinite; }
    @keyframes blink { 50%{opacity:0} }

    /* status */
    #status {
      position:absolute; top:52px; left:50%; transform:translateX(-50%);
      font-family:'Space Mono',monospace; font-size:10px; letter-spacing:2px;
      color:rgba(0,255,224,.5); white-space:nowrap;
    }

    /* barra inferior */
    .hbot {
      position:absolute; bottom:0; left:0; right:0;
      padding:20px 24px;
      display:flex; justify-content:space-between; align-items:center;
      pointer-events:auto;
    }

    /* controles de distância */
    .dist-ctrl { display:flex; gap:10px; }
    .hbtn {
      display:flex; align-items:center; justify-content:center;
      width:46px; height:46px;
      background:var(--glass); backdrop-filter:blur(12px);
      border:1px solid rgba(0,255,224,.25); border-radius:50%;
      color:var(--neon); font-size:22px; font-weight:300;
      cursor:pointer; transition:all .15s ease;
      -webkit-tap-highlight-color:transparent;
    }
    .hbtn:active { background:var(--neon); color:var(--dark); transform:scale(.93); }

    #btn-reset {
      background:var(--glass); backdrop-filter:blur(12px);
      border:1px solid rgba(255,255,255,.15);
      color:rgba(255,255,255,.5);
      font-family:'Space Mono',monospace; font-size:10px;
      letter-spacing:2px; text-transform:uppercase;
      padding:12px 20px; cursor:pointer; border-radius:2px;
      transition:all .2s ease;
      -webkit-tap-highlight-color:transparent;
    }
    #btn-reset:active { border-color:var(--neon2); color:var(--neon2); box-shadow:var(--glow2); }

    /* ── Toast ───────────────────────────────────────────────── */
    #toast {
      position:fixed; top:70px; left:50%;
      transform:translateX(-50%) translateY(-16px);
      background:var(--glass); backdrop-filter:blur(12px);
      border:1px solid var(--neon); color:var(--neon);
      font-family:'Space Mono',monospace; font-size:11px;
      letter-spacing:2px; padding:11px 22px;
      border-radius:2px; box-shadow:var(--glow);
      z-index:9999; opacity:0;
      transition:all .35s cubic-bezier(.4,0,.2,1);
      pointer-events:none; white-space:nowrap;
    }
    #toast.show { opacity:1; transform:translateX(-50%) translateY(0); }

    /* ── loader spinner ──────────────────────────────────────── */
    #loader {
      position:fixed; inset:0; z-index:9998;
      display:none; flex-direction:column;
      align-items:center; justify-content:center;
      background:rgba(6,6,16,.85); backdrop-filter:blur(8px);
    }
    #loader.on { display:flex; }
    .spinner {
      width:44px; height:44px;
      border:2px solid rgba(0,255,224,.2);
      border-top-color:var(--neon);
      border-radius:50%;
      animation:spin .8s linear infinite;
      margin-bottom:16px;
      box-shadow:var(--glow);
    }
    @keyframes spin { to{transform:rotate(360deg)} }
    .loader-txt {
      font-family:'Space Mono',monospace; font-size:11px;
      letter-spacing:3px; color:rgba(0,255,224,.6);
    }
  </style>
</head>
<body>

<!-- câmera -->
<video id="cam" autoplay playsinline muted></video>

<!-- Three.js canvas -->
<canvas id="c"></canvas>

<!-- loader -->
<div id="loader">
  <div class="spinner"></div>
  <div class="loader-txt">Carregando modelo...</div>
</div>

<!-- Splash -->
<div id="splash">
  <div class="sg"></div>
  <div class="sc">
    <div class="badge">● Markerless AR</div>
    <h1>Realidade<br/><span>Aumentada</span></h1>
    <p>
      Um clique para ativar a câmera<br/>
      e visualizar o objeto 3D no mundo real.
    </p>
    <button id="btn-start">
      <span>Ver em AR</span>
      <div class="arr"></div>
    </button>
  </div>
</div>

<!-- HUD -->
<div id="hud">
  <div class="co tl"></div><div class="co tr"></div>
  <div class="co bl"></div><div class="co br"></div>
  <div class="htop">
    <div class="hdot"></div>
    <div class="hlabel">AR · Live</div>
  </div>
  <div id="status">Inicializando...</div>
  <div class="hbot">
    <div class="dist-ctrl">
      <button class="hbtn" id="btn-closer"  title="Aproximar">−</button>
      <button class="hbtn" id="btn-farther" title="Afastar">+</button>
    </div>
    <button id="btn-reset">↺ Reset</button>
  </div>
</div>

<div id="toast"></div>

<script>
/* ─────────────────────────────────────────────────────────
   Configuração — ajuste o nome do arquivo GLB aqui
───────────────────────────────────────────────────────── */
const GLB_PATH   = 'realistic_human_heart.glb'; // caminho relativo ao index.html
const MODEL_SCALE = 0.3;   // escala inicial (ajuste se muito grande/pequeno)
let   modelDist  = -2.5;   // distância em Z (negativo = à frente)

/* ─── elementos DOM ─────────────────────────────────────── */
const camEl     = document.getElementById('cam');
const canvas    = document.getElementById('c');
const splash    = document.getElementById('splash');
const hud       = document.getElementById('hud');
const loader    = document.getElementById('loader');
const statusEl  = document.getElementById('status');
const toastEl   = document.getElementById('toast');
const btnStart  = document.getElementById('btn-start');
const btnReset  = document.getElementById('btn-reset');
const btnCloser = document.getElementById('btn-closer');
const btnFarther= document.getElementById('btn-farther');

/* ─── toast ─────────────────────────────────────────────── */
let toastTimer;
function toast(msg, dur=2500){
  clearTimeout(toastTimer);
  toastEl.textContent = msg;
  toastEl.classList.add('show');
  toastTimer = setTimeout(()=>toastEl.classList.remove('show'), dur);
}

/* ─── Three.js globals ──────────────────────────────────── */
let renderer, scene, camera, model, animId;
let rotY = 0;

function initThree(){
  renderer = new THREE.WebGLRenderer({ canvas, alpha:true, antialias:true });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2));
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setClearColor(0x000000, 0);   // fundo transparente → vídeo aparece

  scene = new THREE.Scene();

  camera = new THREE.PerspectiveCamera(70, window.innerWidth/window.innerHeight, 0.01, 100);
  camera.position.set(0,0,0);

  // Iluminação
  scene.add(new THREE.AmbientLight(0xffffff, 0.8));
  const dir = new THREE.DirectionalLight(0x00ffe0, 1.2);
  dir.position.set(2, 4, 3);
  scene.add(dir);
  const dir2 = new THREE.DirectionalLight(0xff2d78, 0.4);
  dir2.position.set(-2, -1, -2);
  scene.add(dir2);

  window.addEventListener('resize', onResize);
}

function onResize(){
  if(!renderer) return;
  camera.aspect = window.innerWidth/window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}

function loadModel(){
  loader.classList.add('on');
  statusEl.textContent = 'Carregando modelo…';

  const gltfLoader = new THREE.GLTFLoader();
  gltfLoader.load(
    GLB_PATH,
    (gltf) => {
      loader.classList.remove('on');
      model = gltf.scene;

      // Centraliza e escala o modelo automaticamente
      const box  = new THREE.Box3().setFromObject(model);
      const size = new THREE.Vector3();
      box.getSize(size);
      const maxDim = Math.max(size.x, size.y, size.z);
      const scale  = MODEL_SCALE / maxDim;
      model.scale.setScalar(scale);

      // Centraliza pivot
      const center = new THREE.Vector3();
      box.getCenter(center).multiplyScalar(scale);
      model.position.sub(center);

      // Posiciona à frente da câmera
      const pivot = new THREE.Group();
      pivot.add(model);
      pivot.position.set(0, 0, modelDist);
      scene.add(pivot);
      window._pivot = pivot;

      statusEl.textContent = 'Modelo ativo';
      toast('✦ Objeto posicionado');
      startLoop();
    },
    (xhr) => {
      const pct = Math.round(xhr.loaded / xhr.total * 100);
      statusEl.textContent = `Carregando… ${pct}%`;
    },
    (err) => {
      loader.classList.remove('on');
      console.error('Erro ao carregar GLB:', err);
      statusEl.textContent = 'Erro no modelo — usando fallback';
      toast('⚠ Modelo não encontrado — exibindo fallback');
      useFallback();
    }
  );
}

/* ─── Fallback se o .glb não carregar ──────────────────── */
function useFallback(){
  const group = new THREE.Group();

  const sphere = new THREE.Mesh(
    new THREE.SphereGeometry(0.18, 32, 32),
    new THREE.MeshStandardMaterial({ color:0x00ffe0, metalness:.9, roughness:.1, emissive:0x004040, emissiveIntensity:.4 })
  );
  group.add(sphere);

  const torus = new THREE.Mesh(
    new THREE.TorusGeometry(0.28, 0.008, 16, 100),
    new THREE.MeshStandardMaterial({ color:0xff2d78, metalness:.8, roughness:.2, emissive:0x400020, emissiveIntensity:.6 })
  );
  torus.rotation.x = Math.PI/2.2;
  group.add(torus);

  const dot = new THREE.Mesh(
    new THREE.SphereGeometry(0.03, 8, 8),
    new THREE.MeshStandardMaterial({ color:0xffffff, emissive:0xffffff, emissiveIntensity:1 })
  );
  dot.position.set(0.36, 0, 0);
  group.add(dot);

  group.position.set(0, 0, modelDist);
  scene.add(group);
  window._pivot = group;

  statusEl.textContent = 'Fallback ativo';
  toast('✦ Objeto posicionado');
  startLoop();
}

/* ─── Loop de renderização ───────────────────────────────── */
function startLoop(){
  if(animId) cancelAnimationFrame(animId);
  function tick(){
    animId = requestAnimationFrame(tick);
    if(window._pivot){
      window._pivot.rotation.y += 0.008;
    }
    renderer.render(scene, camera);
  }
  tick();
}

/* ─── Câmera do dispositivo ─────────────────────────────── */
async function startCamera(){
  // Tenta câmera traseira primeiro (melhor para AR)
  const constraints = [
    { video:{ facingMode:{ exact:'environment' }, width:{ ideal:1920 }, height:{ ideal:1080 } } },
    { video:{ facingMode:'environment' } },
    { video:true }
  ];
  for(const c of constraints){
    try {
      const stream = await navigator.mediaDevices.getUserMedia(c);
      camEl.srcObject = stream;
      // Remove espelhamento para câmera traseira
      if(c.video.facingMode === 'environment' || (c.video.facingMode && c.video.facingMode.exact === 'environment')){
        camEl.style.transform = 'none';
      }
      return true;
    } catch(e){ /* tenta próximo */ }
  }
  return false;
}

/* ─── BOTÃO ÚNICO — inicia tudo ─────────────────────────── */
btnStart.addEventListener('click', async () => {
  splash.classList.add('hidden');
  hud.classList.add('on');
  canvas.style.display = 'block';
  camEl.style.display  = 'block';
  statusEl.textContent = 'Ativando câmera…';

  initThree();

  const ok = await startCamera();
  if(!ok){
    toast('⚠ Câmera bloqueada. Permita o acesso.');
    statusEl.textContent = 'Câmera bloqueada';
    return;
  }

  camEl.onloadedmetadata = () => {
    camEl.play();
    toast('✦ Câmera ativa — carregando modelo');
    loadModel();
  };
});

/* ─── Reset — reposiciona à frente ──────────────────────── */
btnReset.addEventListener('click', ()=>{
  if(!window._pivot) return;
  window._pivot.position.set(0, 0, modelDist);
  window._pivot.rotation.set(0, 0, 0);
  toast('↺ Objeto reposicionado');
  statusEl.textContent = 'Objeto reposicionado';
});

/* ─── Aproximar / Afastar ────────────────────────────────── */
btnCloser.addEventListener('click', ()=>{
  if(!window._pivot) return;
  modelDist = Math.min(modelDist + 0.3, -0.5);
  window._pivot.position.z = modelDist;
});
btnFarther.addEventListener('click', ()=>{
  if(!window._pivot) return;
  modelDist = Math.max(modelDist - 0.3, -8);
  window._pivot.position.z = modelDist;
});

/* ─── Arraste para rotacionar (touch + mouse) ────────────── */
let dragging=false, lastX=0, lastY=0;
canvas.style.pointerEvents = 'auto';

canvas.addEventListener('touchstart', e=>{
  dragging=true;
  lastX=e.touches[0].clientX;
  lastY=e.touches[0].clientY;
},{passive:true});
canvas.addEventListener('touchmove', e=>{
  if(!dragging||!window._pivot) return;
  const dx=(e.touches[0].clientX-lastX)*0.01;
  const dy=(e.touches[0].clientY-lastY)*0.01;
  window._pivot.rotation.y += dx;
  window._pivot.rotation.x += dy;
  lastX=e.touches[0].clientX;
  lastY=e.touches[0].clientY;
},{passive:true});
canvas.addEventListener('touchend', ()=>dragging=false);

canvas.addEventListener('mousedown', e=>{ dragging=true; lastX=e.clientX; lastY=e.clientY; });
canvas.addEventListener('mousemove', e=>{
  if(!dragging||!window._pivot) return;
  window._pivot.rotation.y += (e.clientX-lastX)*0.01;
  window._pivot.rotation.x += (e.clientY-lastY)*0.01;
  lastX=e.clientX; lastY=e.clientY;
});
canvas.addEventListener('mouseup', ()=>dragging=false);

/* ─── Pinch para escalar (mobile) ───────────────────────── */
let lastPinch = 0;
canvas.addEventListener('touchstart', e=>{
  if(e.touches.length===2){
    lastPinch = Math.hypot(
      e.touches[0].clientX - e.touches[1].clientX,
      e.touches[0].clientY - e.touches[1].clientY
    );
  }
},{passive:true});
canvas.addEventListener('touchmove', e=>{
  if(e.touches.length===2 && window._pivot){
    const dist = Math.hypot(
      e.touches[0].clientX - e.touches[1].clientX,
      e.touches[0].clientY - e.touches[1].clientY
    );
    const delta = dist - lastPinch;
    const s = window._pivot.scale.x + delta*0.002;
    window._pivot.scale.setScalar(Math.max(0.05, Math.min(s, 5)));
    lastPinch = dist;
  }
},{passive:true});
</script>
</body>
</html>