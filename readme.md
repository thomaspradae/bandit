<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Three‑Armed Bandit – 15‑Pull Challenge</title>
  <!-- lightweight, tailwind‑ish vibes → all inline to stay GitHub‑Pages‑portable -->
  <style>
    :root{--blue:#2563eb;--slate:#1e293b;--bg:#f1f5f9;--rose:#f43f5e}
    *{box-sizing:border-box;margin:0;padding:0}
    body{font-family:system-ui,sans-serif;background:var(--bg);color:var(--slate);display:flex;flex-direction:column;align-items:center;gap:1.8rem;padding:2rem}
    h1{font-size:2.35rem;text-align:center;font-weight:700;line-height:1.2}
    h1 small{font-size:.7em;font-weight:400;color:#64748b}
    .machines{display:flex;gap:1.5rem;flex-wrap:wrap;margin-top:.4rem}
    button.machine{position:relative;isolation:isolate;padding:1.25rem 2.4rem;font-size:1.35rem;font-weight:600;border:none;border-radius:1rem;background:var(--blue);color:#fff;box-shadow:0 6px 14px rgba(0,0,0,.18);cursor:pointer;transition:transform .12s,filter .12s}
    button.machine:hover{filter:brightness(1.15)}
    button.machine:active{transform:scale(.93)}
    button.machine:disabled{opacity:.4;cursor:not-allowed;filter:none}
    #status{font-size:1.15rem;min-height:1.6rem;text-align:center}
    #summary{font-size:1.5rem;font-weight:700;margin-top:1.4rem;text-align:center}
    table{margin-top:1.1rem;width:100%;max-width:480px;border-collapse:collapse;background:#fff;border-radius:.6rem;overflow:hidden;box-shadow:0 4px 12px rgba(0,0,0,.07)}
    th,td{padding:.55rem .7rem;text-align:center;border-bottom:1px solid #e2e8f0;font-size:.95rem}
    th{background:#e0e7ff;font-size:1rem;font-weight:600}
    tr.flash{animation:flashBg 1s ease-in-out}
    @keyframes flashBg{0%{background:#fef08a}100%{background:transparent}}
    /* progress bar */
    #barWrap{position:relative;width:100%;max-width:480px;height:.65rem;background:#cbd5e1;border-radius:1rem;overflow:hidden;margin-top:-.6rem}
    #bar{position:absolute;inset:0;background:var(--blue);transition:width .4s}
    /* floating payout label */
    .float{position:absolute;left:50%;top:0;transform:translate(-50%,0);font-size:1.1rem;font-weight:700;pointer-events:none;color:var(--rose);animation:floatUp 1.4s forwards}
    @keyframes floatUp{0%{opacity:0;transform:translate(-50%,0) scale(.8)}20%{opacity:1;transform:translate(-50%,-10px) scale(1)}100%{opacity:0;transform:translate(-50%,-60px) scale(1.1)}}
    /* modal */
    .modal{position:fixed;inset:0;background:rgba(0,0,0,.6);display:flex;align-items:center;justify-content:center;opacity:0;pointer-events:none;transition:opacity .25s;z-index:60}
    .modal.show{opacity:1;pointer-events:auto}
    .modal-box{background:#fff;border-radius:1rem;padding:2rem 2.4rem;max-width:380px;width:90%;text-align:center;box-shadow:0 20px 44px rgba(0,0,0,.25);animation:pop .35s cubic-bezier(.21,1.02,.73,1.04)}
    @keyframes pop{0%{transform:scale(.7);opacity:0}100%{transform:scale(1);opacity:1}}
  </style>
  <script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.5.1/dist/confetti.browser.min.js"></script>
</head>
<body>
  <h1>Three‑Armed Bandit <small>(15‑Pull Challenge)</small></h1>
  <div id="status"></div>
  <div class="machines">
    <button class="machine" data-id="A">Pull A</button>
    <button class="machine" data-id="B">Pull B</button>
    <button class="machine" data-id="C">Pull C</button>
  </div>
  <div id="barWrap"><div id="bar"></div></div>
  <div id="summary"></div>
  <table id="log" hidden>
    <thead><tr><th>#</th><th>Machine</th><th>Payout</th><th>Net</th></tr></thead>
    <tbody></tbody>
  </table>

<!-- modal & confetti canvas -->
<div id="modal" class="modal"><div class="modal-box" id="modalBox"></div></div>

<script>
// ------- MATH: Normal RNG via Box-Muller -------
function randn(mean=0,std=1){let u=0,v=0;while(u===0)u=Math.random();while(v===0)v=Math.random();return mean+std*Math.sqrt(-2*Math.log(u))*Math.cos(2*Math.PI*v)}

// ------- GAME PARAMETERS -------
const params={A:{mean:2,std:1},B:{mean:5,std:8},C:{mean:3,std:1}};
let pulls=15, bank=0, history=[], confettiFired=false;

// ------- DOM SHORTHANDS -------
const statusEl=document.getElementById('status');
const summaryEl=document.getElementById('summary');
const logTbl=document.getElementById('log');
const logBody=logTbl.querySelector('tbody');
const buttons=document.querySelectorAll('.machine');
const bar=document.getElementById('bar');
const modal=document.getElementById('modal');
const modalBox=document.getElementById('modalBox');

// ------- RENDER LOOP -------
function render(){
  statusEl.textContent=`Pulls left: ${pulls}\xa0—\xa0Bankroll: $${bank.toFixed(2)}`;
  bar.style.width=`${(15-pulls)/15*100}%`;
  if(history.length) logTbl.hidden=false;
  // render newest row
  if(logBody.children.length<history.length){
    const row=history[history.length-1];
    const tr=document.createElement('tr');
    tr.classList.add('flash');
    tr.innerHTML=`<td>${history.length}</td><td>${row.machine}</td><td>$${row.payout.toFixed(2)}</td><td>$${row.net.toFixed(2)}</td>`;
    logBody.appendChild(tr);
  }
  if(pulls===0) endGame();
}

// ------- END GAME -------
function endGame(){
  summaryEl.textContent=`👏 FINAL BANKROLL: $${bank.toFixed(2)}`;
  buttons.forEach(b=>b.disabled=true);
  celebrate();
}

function celebrate(){
  if(confettiFired) return; confettiFired=true;
  const duration=2800, end=Date.now()+duration;
  (function frame(){
    confetti({particleCount:8,spread:70,origin:{y:0.7}});
    if(Date.now()<end) requestAnimationFrame(frame);
  })();
  modalBox.innerHTML=`<h2 style='font-size:1.7rem;margin-bottom:.4rem'>Game Over</h2><p style='font-size:1.3rem;margin-bottom:.9rem'>Your bankroll:<br><strong>$${bank.toFixed(2)}</strong></p><button id='replay' style='padding:.65rem 1.3rem;border:none;border-radius:.6rem;background:var(--blue);color:#fff;font-size:1rem;cursor:pointer;'>Play Again</button>`;
  modal.classList.add('show');
  document.getElementById('replay').onclick=resetGame;
}

// ------- RESET -------
function resetGame(){
  pulls=15; bank=0; history=[]; confettiFired=false;
  logBody.innerHTML=''; summaryEl.textContent='';
  buttons.forEach(b=>b.disabled=false); modal.classList.remove('show');
  render();
}

// ------- FLOATING LABEL -------
function floatLabel(btn,txt){
  const span=document.createElement('span'); span.className='float'; span.textContent=txt;
  btn.appendChild(span);
  setTimeout(()=>span.remove(),1400);
}

// ------- CLICK HANDLER -------
function pull(e){
  if(pulls===0) return;
  const id=e.currentTarget.dataset.id;
  const {mean,std}=params[id];
  const payout=Math.max(0,randn(mean,std));
  bank+=payout-1; pulls--; history.push({machine:id,payout,net:bank});
  floatLabel(e.currentTarget,`$${payout.toFixed(2)}`);
  render();
}

buttons.forEach(b=>b.addEventListener('click',pull));
render();
</script>
</body>
</html>
