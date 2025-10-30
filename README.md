<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Time & Stopwatch</title>
  <style>
    :root{--bg:#0f1724;--card:#0b1220;--accent:#06b6d4;--muted:#94a3b8;--glass: rgba(255,255,255,0.04)}
    *{box-sizing:border-box}
    body{margin:0; font-family:Inter,ui-sans-serif,system-ui,-apple-system,Segoe UI,Roboto,Helvetica,Arial; background:linear-gradient(180deg,#061021 0%, #071627 100%); color:#e6eef6; min-height:100vh; display:flex; align-items:center; justify-content:center; padding:20px}
    .wrap{display:grid; gap:20px; grid-template-columns: 1fr; max-width:900px; width:100%}
    header{display:flex; align-items:center; justify-content:space-between}
    h1{font-size:20px; margin:0}
    .grid{display:grid; grid-template-columns: 1fr 1fr; gap:18px}
    .card{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01)); padding:18px; border-radius:12px; box-shadow:0 6px 18px rgba(2,6,23,0.6); border:1px solid rgba(255,255,255,0.03)}
    .clock-display{font-size:48px; font-weight:600; letter-spacing:1px}
    .sub{color:var(--muted); font-size:13px}
    .row{display:flex; gap:10px; align-items:center}
    button{background:var(--accent); color:#012; padding:10px 12px; border-radius:8px; border:none; font-weight:600; cursor:pointer}
    button.ghost{background:transparent; color:var(--accent); border:1px solid rgba(6,182,212,0.15)}
    .controls{margin-top:14px}
    .stopwatch-time{font-size:36px; font-weight:700}
    .laps{margin-top:12px; max-height:180px; overflow:auto}
    .lap{padding:8px; border-radius:8px; background:var(--glass); display:flex; justify-content:space-between; font-size:14px; margin-bottom:8px}
    .center{display:flex; align-items:center; justify-content:center}
    @media (max-width:720px){.grid{grid-template-columns:1fr} .clock-display{font-size:36px}}
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>Time & Stopwatch</h1>
      <div class="sub">Simple, responsive, single-file web app</div>
    </header>

    <div class="grid">
      <div class="card" id="clock-card">
        <div class="row" style="justify-content:space-between;align-items:flex-end">
          <div>
            <div class="sub">Current time</div>
            <div id="time" class="clock-display">--:--:--</div>
            <div id="date" class="sub">Loading date…</div>
          </div>
          <div style="text-align:right">
            <div class="sub">Timezone</div>
            <select id="tz-select" style="margin-top:6px;padding:8px;border-radius:8px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:inherit">
              <option value="local">Local (browser)</option>
              <option value="UTC">UTC</option>
              <option value="Asia/Kolkata">Asia/Kolkata</option>
              <option value="America/New_York">America/New_York</option>
              <option value="Europe/London">Europe/London</option>
            </select>
          </div>
        </div>
      </div>

      <div class="card" id="stopwatch-card">
        <div class="sub">Stopwatch</div>
        <div style="margin-top:10px;display:flex;align-items:center;justify-content:space-between">
          <div class="stopwatch-time" id="sw-display">00:00:00.000</div>
          <div class="row">
            <button id="sw-start">Start</button>
            <button id="sw-lap" class="ghost">Lap</button>
            <button id="sw-reset" class="ghost">Reset</button>
          </div>
        </div>
        <div class="laps" id="laps"></div>
      </div>
    </div>

    <footer style="text-align:center; color:var(--muted); font-size:13px">Built with plain HTML/CSS/JS — open this file in your browser to use. Refresh will reset the stopwatch.</footer>
  </div>

<script>
// CLOCK
const timeEl = document.getElementById('time');
const dateEl = document.getElementById('date');
const tzSelect = document.getElementById('tz-select');

function updateClock(){
  const tz = tzSelect.value;
  let now = new Date();
  if(tz && tz !== 'local'){
    // Use toLocaleString with timeZone option
    const parts = new Intl.DateTimeFormat('en-US', {timeZone: tz, hour12:false, hour:'2-digit', minute:'2-digit', second:'2-digit'}).formatToParts(now);
    const hh = parts.find(p=>p.type==='hour').value;
    const mm = parts.find(p=>p.type==='minute').value;
    const ss = parts.find(p=>p.type==='second').value;
    timeEl.textContent = `${hh}:${mm}:${ss}`;
    const dateStr = new Intl.DateTimeFormat('en-US', {timeZone: tz, weekday:'short', year:'numeric', month:'short', day:'2-digit'}).format(now);
    dateEl.textContent = dateStr + ' ('+tz+')';
  } else {
    const hh = String(now.getHours()).padStart(2,'0');
    const mm = String(now.getMinutes()).padStart(2,'0');
    const ss = String(now.getSeconds()).padStart(2,'0');
    timeEl.textContent = `${hh}:${mm}:${ss}`;
    dateEl.textContent = now.toDateString();
  }
}
setInterval(updateClock, 250);
updateClock();

// STOPWATCH
const swDisplay = document.getElementById('sw-display');
const btnStart = document.getElementById('sw-start');
const btnLap = document.getElementById('sw-lap');
const btnReset = document.getElementById('sw-reset');
const lapsEl = document.getElementById('laps');

let swStart = 0; // timestamp when running
let swElapsed = 0; // milliseconds accumulated when paused
let swTimer = null;
let running = false;
let lapCount = 0;

function formatTime(ms){
  const milliseconds = ms % 1000;
  const totalSeconds = Math.floor(ms/1000);
  const seconds = totalSeconds % 60;
  const minutes = Math.floor(totalSeconds/60) % 60;
  const hours = Math.floor(totalSeconds/3600);
  return `${String(hours).padStart(2,'0')}:${String(minutes).padStart(2,'0')}:${String(seconds).padStart(2,'0')}.${String(milliseconds).padStart(3,'0')}`;
}

function updateStopwatchDisplay(){
  const now = performance.now();
  const elapsed = swElapsed + (running ? (now - swStart) : 0);
  swDisplay.textContent = formatTime(Math.floor(elapsed));
}

function startStopwatch(){
  if(running) return;
  swStart = performance.now();
  running = true;
  btnStart.textContent = 'Pause';
  btnStart.style.background = 'var(--accent)';
  swTimer = requestAnimationFrame(tick);
}

function tick(){
  updateStopwatchDisplay();
  if(running) swTimer = requestAnimationFrame(tick);
}

function pauseStopwatch(){
  if(!running) return;
  const now = performance.now();
  swElapsed += now - swStart;
  running = false;
  btnStart.textContent = 'Start';
  cancelAnimationFrame(swTimer);
}

function resetStopwatch(){
  running = false;
  swStart = 0; swElapsed = 0; lapCount = 0;
  swDisplay.textContent = '00:00:00.000';
  btnStart.textContent = 'Start';
  lapsEl.innerHTML = '';
  cancelAnimationFrame(swTimer);
}

function lapStopwatch(){
  const now = performance.now();
  const elapsed = swElapsed + (running ? (now - swStart) : 0);
  lapCount++;
  const el = document.createElement('div');
  el.className = 'lap';
  el.innerHTML = `<div>Lap ${lapCount}</div><div>${formatTime(Math.floor(elapsed))}</div>`;
  lapsEl.prepend(el);
}

btnStart.addEventListener('click', ()=>{
  if(!running) startStopwatch(); else pauseStopwatch();
});
btnReset.addEventListener('click', resetStopwatch);
btnLap.addEventListener('click', ()=>{ if(swElapsed>0 || running) lapStopwatch(); });

// Keyboard shortcuts
window.addEventListener('keydown', (e)=>{
  if(e.code === 'Space'){
    e.preventDefault();
    if(!running) startStopwatch(); else pauseStopwatch();
  }
  if(e.key.toLowerCase()==='l') lapStopwatch();
  if(e.key.toLowerCase()==='r') resetStopwatch();
});

// Save timezone selection in localStorage
tzSelect.addEventListener('change', ()=>{ localStorage.setItem('tz', tzSelect.value); updateClock(); });
const savedTz = localStorage.getItem('tz'); if(savedTz) tzSelect.value = savedTz;

</script>
</body>
</html>
