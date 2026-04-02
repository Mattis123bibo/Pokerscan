<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>AI Kamera</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Space+Mono:wght@400;700&family=DM+Sans:wght@300;400;600&display=swap');

  :root {
    --bg: #0a0a0a;
    --surface: #111111;
    --border: #1e1e1e;
    --accent: #00ff88;
    --accent2: #ff3366;
    --text: #f0f0f0;
    --muted: #555;
    --card: #161616;
  }

  * { margin: 0; padding: 0; box-sizing: border-box; -webkit-tap-highlight-color: transparent; }

  body {
    background: var(--bg);
    color: var(--text);
    font-family: 'DM Sans', sans-serif;
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    overflow-x: hidden;
  }

  header {
    width: 100%;
    max-width: 480px;
    padding: 20px 20px 12px;
  }

  .logo {
    font-family: 'Space Mono', monospace;
    font-size: 13px;
    color: var(--accent);
    letter-spacing: 2px;
    text-transform: uppercase;
  }
  .logo span { color: var(--muted); }

  .main {
    width: 100%;
    max-width: 480px;
    padding: 0 16px;
    display: flex;
    flex-direction: column;
    gap: 16px;
  }

  .viewfinder {
    position: relative;
    width: 100%;
    aspect-ratio: 3/4;
    background: #050505;
    border-radius: 16px;
    overflow: hidden;
    border: 1px solid var(--border);
    -webkit-transform: translateZ(0);
    transform: translateZ(0);
  }

  #video {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
    -webkit-transform: translateZ(0);
    transform: translateZ(0);
    -webkit-backface-visibility: hidden;
    backface-visibility: hidden;
  }

  #canvas { display: none; }

  #preview {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: none;
    position: absolute;
    top: 0; left: 0;
  }

  .corner {
    position: absolute;
    width: 22px; height: 22px;
    border-color: var(--accent);
    border-style: solid;
    opacity: 0.6;
    z-index: 10;
  }
  .corner.tl { top: 14px; left: 14px; border-width: 2px 0 0 2px; border-radius: 4px 0 0 0; }
  .corner.tr { top: 14px; right: 14px; border-width: 2px 2px 0 0; border-radius: 0 4px 0 0; }
  .corner.bl { bottom: 14px; left: 14px; border-width: 0 0 2px 2px; border-radius: 0 0 0 4px; }
  .corner.br { bottom: 14px; right: 14px; border-width: 0 2px 2px 0; border-radius: 0 0 4px 0; }

  .flash-overlay {
    position: absolute; inset: 0;
    background: white; opacity: 0;
    pointer-events: none; z-index: 20;
    transition: opacity 0.05s;
  }

  .scan-line {
    position: absolute;
    left: 0; right: 0; height: 2px;
    background: linear-gradient(90deg, transparent, var(--accent), transparent);
    top: 0; z-index: 15; opacity: 0;
  }
  .scan-line.active { opacity: 1; animation: scan 1.5s linear infinite; }
  @keyframes scan { from { top: 0%; } to { top: 100%; } }

  .status-badge {
    position: absolute;
    top: 14px; left: 50%;
    transform: translateX(-50%);
    background: rgba(0,0,0,0.7);
    border: 1px solid var(--border);
    border-radius: 20px;
    padding: 4px 12px;
    font-family: 'Space Mono', monospace;
    font-size: 10px; color: var(--muted);
    letter-spacing: 1px; white-space: nowrap; z-index: 10;
  }
  .status-badge.live { color: var(--accent); border-color: var(--accent); }
  .status-badge.analyzing { color: #ffaa00; border-color: #ffaa00; animation: pulse-badge 0.8s ease-in-out infinite; }
  @keyframes pulse-badge { 0%,100%{opacity:1} 50%{opacity:0.5} }

  .no-cam {
    display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    height: 100%; gap: 12px; color: var(--muted);
    padding: 20px; text-align: center;
  }
  .no-cam svg { opacity: 0.4; }
  .no-cam p { font-size: 13px; line-height: 1.6; }

  .controls {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 24px;
    padding: 8px 0;
  }

  .btn-retake {
    width: 44px; height: 44px;
    border-radius: 50%;
    border: 1px solid var(--border);
    background: var(--surface);
    color: var(--muted);
    cursor: pointer;
    display: flex; align-items: center; justify-content: center;
    transition: all 0.2s;
    opacity: 0; pointer-events: none;
    -webkit-appearance: none;
  }
  .btn-retake.visible { opacity: 1; pointer-events: all; }

  .btn-shutter {
    width: 72px; height: 72px;
    border-radius: 50%;
    background: white;
    border: 3px solid rgba(255,255,255,0.3);
    cursor: pointer;
    display: flex; align-items: center; justify-content: center;
    box-shadow: 0 0 0 6px rgba(255,255,255,0.07);
    flex-shrink: 0;
    -webkit-appearance: none;
    transition: transform 0.1s;
  }
  .btn-shutter:active { transform: scale(0.92); }
  .btn-shutter.disabled { opacity: 0.4; pointer-events: none; }
  .btn-shutter-inner { width: 56px; height: 56px; border-radius: 50%; background: white; }

  .btn-analyze {
    width: 44px; height: 44px;
    border-radius: 50%;
    border: 1px solid var(--accent);
    background: rgba(0,255,136,0.08);
    color: var(--accent);
    cursor: pointer;
    display: flex; align-items: center; justify-content: center;
    transition: all 0.2s;
    opacity: 0; pointer-events: none;
    -webkit-appearance: none;
  }
  .btn-analyze.visible { opacity: 1; pointer-events: all; }

  .question-row {
    display: flex; gap: 8px;
    opacity: 0; pointer-events: none;
    transition: opacity 0.3s;
  }
  .question-row.visible { opacity: 1; pointer-events: all; }

  .question-input {
    flex: 1;
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 10px;
    padding: 12px 14px;
    color: var(--text);
    font-family: 'DM Sans', sans-serif;
    font-size: 16px;
    outline: none;
    -webkit-appearance: none;
    transition: border-color 0.2s;
  }
  .question-input::placeholder { color: var(--muted); }
  .question-input:focus { border-color: var(--accent); }

  .result-card {
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: 14px;
    overflow: hidden;
    display: none;
  }
  .result-card.visible { display: block; }

  .result-header {
    padding: 12px 16px;
    border-bottom: 1px solid var(--border);
    display: flex; align-items: center; gap: 8px;
  }
  .result-header .tag {
    font-family: 'Space Mono', monospace;
    font-size: 10px; color: var(--accent);
    letter-spacing: 1px; text-transform: uppercase;
  }
  .result-header .model {
    font-family: 'Space Mono', monospace;
    font-size: 10px; color: var(--muted); margin-left: auto;
  }

  .result-body {
    padding: 16px;
    font-size: 14px; line-height: 1.65;
    color: #d0d0d0;
    max-height: 320px; overflow-y: auto;
    white-space: pre-wrap;
    -webkit-overflow-scrolling: touch;
  }

  .cursor {
    display: inline-block; width: 2px; height: 14px;
    background: var(--accent); margin-left: 2px;
    vertical-align: text-bottom;
    animation: blink 0.7s step-end infinite;
  }
  @keyframes blink { 0%,100%{opacity:1} 50%{opacity:0} }

  .error-msg {
    font-family: 'Space Mono', monospace;
    font-size: 11px; color: var(--accent2);
    text-align: center; padding: 8px; display: none;
  }
  .error-msg.visible { display: block; }

  .spacer { height: 20px; }
</style>
</head>
<body>

<header>
  <div class="logo">AI<span>/</span>KAMERA</div>
</header>

<div class="main">
  <div class="viewfinder">
    <div class="corner tl"></div>
    <div class="corner tr"></div>
    <div class="corner bl"></div>
    <div class="corner br"></div>
    <div class="flash-overlay" id="flash"></div>
    <div class="scan-line" id="scanLine"></div>
    <div class="status-badge live" id="statusBadge">LIVE</div>

    <video id="video" autoplay playsinline muted webkit-playsinline></video>
    <canvas id="canvas"></canvas>
    <img id="preview" alt="Aufnahme" />

    <div class="no-cam" id="noCam" style="display:none;">
      <svg width="40" height="40" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
        <path d="M23 19a2 2 0 01-2 2H3a2 2 0 01-2-2V8a2 2 0 012-2h4l2-3h6l2 3h4a2 2 0 012 2z"/>
        <circle cx="12" cy="13" r="4"/>
      </svg>
      <p>Kamera-Zugriff fehlt.<br><br>Safari → Einstellungen → Safari → Kamera → Erlauben, dann Seite neu laden.</p>
    </div>
  </div>

  <div class="controls">
    <button class="btn-retake" id="btnRetake" type="button">
      <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <path d="M3 12a9 9 0 109-9M3 3v6h6"/>
      </svg>
    </button>
    <button class="btn-shutter" id="btnShutter" type="button">
      <div class="btn-shutter-inner"></div>
    </button>
    <button class="btn-analyze" id="btnAnalyze" type="button">
      <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <circle cx="11" cy="11" r="8"/><path d="M21 21l-4.35-4.35"/>
        <path d="M11 8v6M8 11h6"/>
      </svg>
    </button>
  </div>

  <div class="question-row" id="questionRow">
    <input
      class="question-input"
      id="questionInput"
      type="text"
      placeholder="Was willst du wissen? (optional)"
      maxlength="200"
      autocomplete="off"
      autocorrect="off"
    />
  </div>

  <div class="error-msg" id="errorMsg"></div>

  <div class="result-card" id="resultCard">
    <div class="result-header">
      <span class="tag">Claude Analyse</span>
      <span class="model">claude-sonnet-4</span>
    </div>
    <div class="result-body" id="resultBody"></div>
  </div>

  <div class="spacer"></div>
</div>

<script>
const video       = document.getElementById('video');
const canvas      = document.getElementById('canvas');
const preview     = document.getElementById('preview');
const btnShutter  = document.getElementById('btnShutter');
const btnRetake   = document.getElementById('btnRetake');
const btnAnalyze  = document.getElementById('btnAnalyze');
const questionRow = document.getElementById('questionRow');
const questionInput = document.getElementById('questionInput');
const resultCard  = document.getElementById('resultCard');
const resultBody  = document.getElementById('resultBody');
const statusBadge = document.getElementById('statusBadge');
const scanLine    = document.getElementById('scanLine');
const flash       = document.getElementById('flash');
const errorMsg    = document.getElementById('errorMsg');
const noCam       = document.getElementById('noCam');

let capturedBase64 = null;
let state = 'live';

function setStatus(text, type) {
  statusBadge.textContent = text;
  statusBadge.className = 'status-badge ' + (type || '');
}

function showError(msg) {
  errorMsg.textContent = msg;
  errorMsg.classList.add('visible');
  setTimeout(() => errorMsg.classList.remove('visible'), 5000);
}

function showNoCam() {
  video.style.display = 'none';
  noCam.style.display = 'flex';
  btnShutter.classList.add('disabled');
  setStatus('KEIN ZUGRIFF', '');
}

// ---- Camera ----
async function startCamera() {
  if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
    showNoCam(); return;
  }

  // Try with back camera, then fallback to any camera
  const attempts = [
    { video: { facingMode: { ideal: 'environment' } }, audio: false },
    { video: true, audio: false }
  ];

  for (const constraints of attempts) {
    try {
      const stream = await navigator.mediaDevices.getUserMedia(constraints);
      video.srcObject = stream;
      // iOS Safari: must call play() explicitly after setting srcObject
      video.play().catch(() => {});
      return;
    } catch (e) {
      console.warn('Camera attempt failed:', e.name);
    }
  }
  showNoCam();
}

// iOS: video can go blank after tab switch — resume on visibility change
document.addEventListener('visibilitychange', () => {
  if (!document.hidden && video.srcObject && state === 'live') {
    video.play().catch(() => {});
  }
});

// ---- Shutter ----
btnShutter.addEventListener('click', () => {
  if (state !== 'live') return;

  flash.style.opacity = '1';
  setTimeout(() => { flash.style.opacity = '0'; }, 150);

  const w = video.videoWidth  || 640;
  const h = video.videoHeight || 480;
  canvas.width = w;
  canvas.height = h;
  canvas.getContext('2d').drawImage(video, 0, 0, w, h);

  const dataUrl = canvas.toDataURL('image/jpeg', 0.85);
  capturedBase64 = dataUrl.split(',')[1];

  preview.src = dataUrl;
  preview.style.display = 'block';
  video.style.display = 'none';

  state = 'captured';
  setStatus('AUFGENOMMEN', 'live');
  btnRetake.classList.add('visible');
  btnAnalyze.classList.add('visible');
  questionRow.classList.add('visible');
  resultCard.classList.remove('visible');
  resultBody.textContent = '';
});

// ---- Retake ----
btnRetake.addEventListener('click', () => {
  preview.style.display = 'none';
  video.style.display = 'block';
  video.play().catch(() => {});

  capturedBase64 = null;
  state = 'live';
  setStatus('LIVE', 'live');
  btnRetake.classList.remove('visible');
  btnAnalyze.classList.remove('visible');
  questionRow.classList.remove('visible');
  resultCard.classList.remove('visible');
  resultBody.textContent = '';
  scanLine.classList.remove('active');
});

// ---- Analyze ----
btnAnalyze.addEventListener('click', async () => {
  if (!capturedBase64 || state === 'analyzing') return;

  state = 'analyzing';
  setStatus('ANALYSE...', 'analyzing');
  scanLine.classList.add('active');
  btnAnalyze.style.opacity = '0.4';
  btnAnalyze.style.pointerEvents = 'none';
  btnRetake.style.opacity = '0.4';
  btnRetake.style.pointerEvents = 'none';

  resultBody.innerHTML = '<span class="cursor"></span>';
  resultCard.classList.add('visible');

  const question = questionInput.value.trim();
  const prompt = question
    ? `Analysiere dieses Bild. Der Nutzer fragt: "${question}". Antworte auf Deutsch, klar und präzise.`
    : `Beschreibe und analysiere dieses Bild auf Deutsch. Was ist zu sehen? Was fällt auf? Sei prägnant aber informativ.`;

  try {
    const resp = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 1000,
        messages: [{
          role: 'user',
          content: [
            { type: 'image', source: { type: 'base64', media_type: 'image/jpeg', data: capturedBase64 } },
            { type: 'text', text: prompt }
          ]
        }]
      })
    });

    if (!resp.ok) {
      const err = await resp.json().catch(() => ({}));
      throw new Error(err.error?.message || `API Fehler ${resp.status}`);
    }

    const data = await resp.json();
    const text = data.content?.map(b => b.text || '').join('') || '(Keine Antwort)';

    // Typewriter
    resultBody.innerHTML = '';
    let i = 0;
    const cursor = document.createElement('span');
    cursor.className = 'cursor';
    function type() {
      if (i < text.length) {
        resultBody.textContent = text.slice(0, ++i);
        resultBody.appendChild(cursor);
        resultBody.scrollTop = resultBody.scrollHeight;
        setTimeout(type, 12);
      } else { cursor.remove(); }
    }
    type();

  } catch (e) {
    resultBody.textContent = '';
    resultCard.classList.remove('visible');
    showError('Fehler: ' + e.message);
  } finally {
    state = 'captured';
    setStatus('AUFGENOMMEN', 'live');
    scanLine.classList.remove('active');
    btnAnalyze.style.opacity = '';
    btnAnalyze.style.pointerEvents = '';
    btnRetake.style.opacity = '';
    btnRetake.style.pointerEvents = '';
  }
});

questionInput.addEventListener('keydown', e => {
  if (e.key === 'Enter' && state === 'captured') btnAnalyze.click();
});

startCamera();
</script>
</body>
</html>
