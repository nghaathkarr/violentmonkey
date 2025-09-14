// ==UserScript==
// @name         Wingo30s Watcher & Logger
// @namespace    http://tampermonkey.net/
// @version      1.2
// @description  Watch Wingo 30s draws, record server id + numbers, analyze (small/big, colors), store per-day and export CSV.
// @match        https://777bigwingame.vip/*
// @grant        none
// ==/UserScript==

(function(){
  'use strict';

  // ----- config -----
  const POLL_INTERVAL = 1000; // ms
  const STORAGE_PREFIX = 'wingo-watcher-'; // will append YYYYMMDD
  const MAX_KEEP = 2000;

  // ----- helpers -----
  function todayKey(){
    const d = new Date();
    const s = d.toISOString().slice(0,10); // YYYY-MM-DD
    return STORAGE_PREFIX + s;
  }

  function loadHistory(){
    try { return JSON.parse(localStorage.getItem(todayKey()) || '[]'); }
    catch(e){ return []; }
  }
  function saveHistory(arr){
    localStorage.setItem(todayKey(), JSON.stringify(arr.slice(-MAX_KEEP)));
  }

  // robust serverId finder: look for long digit sequences (13-20 digits like screenshot)
  function findServerId(){
    const all = document.querySelectorAll('body *');
    for(let el of all){
      if(!el || !el.innerText) continue;
      const m = el.innerText.match(/\b\d{8,20}\b/);
      if(m) return m[0];
    }
    return null;
  }

  // try to find number container by common patterns
  function findNumberContainer(){
    // try specific common class names
    const tries = [
      '.result-balls', '.results', '.draw-numbers', '.win-numbers', '.win-go', '.draw-area', '.ball-row'
    ];
    for(const s of tries){
      const el = document.querySelector(s);
      if(el && el.innerText && /[0-9]/.test(el.innerText)) return el;
    }
    // fallback: look for a node that contains 3-10 separated single digits
    const nodes = document.querySelectorAll('body *');
    for(let n of nodes){
      if(!n || !n.innerText) continue;
      const digits = n.innerText.match(/[0-9]/g);
      if(digits && digits.length >= 1 && digits.length <= 20){
        // require that text length isn't huge and contains some separators or small chunk
        if(n.innerText.length < 200) return n;
      }
    }
    return null;
  }

  // parse numbers from node -> returns array of digit strings (0-9)
  function parseNumbersFromNode(node){
    if(!node) return [];
    // try to find explicit ball elements
    const ballSel = node.querySelectorAll && node.querySelectorAll('.ball, .num, .number, .digit, span');
    if(ballSel && ballSel.length){
      const out = [];
      ballSel.forEach(el=>{
        const t = (el.innerText||'').trim();
        if(/^[0-9]$/.test(t)) out.push(t);
      });
      if(out.length) return out;
    }
    // fallback: extract digits in order
    const txt = node.innerText || node.textContent || '';
    const matches = txt.match(/[0-9]/g);
    return matches ? matches : [];
  }

  // analysis rules per user's spec
  function analyzeDigit(ch){
    const n = parseInt(ch,10);
    if(Number.isNaN(n)) return null;
    const size = (n <= 4) ? 'အသေး' : 'အကြီး';
    let color;
    if(n === 0) color = 'အနီ+ခရမ်း';
    else if(n === 5) color = 'အစိမ်း+ခရမ်း';
    else if(n % 2 === 0) color = 'အနီ';
    else color = 'အစိမ်း';
    return {n, size, color};
  }

  // UI overlay
  const overlay = document.createElement('div');
  overlay.id = 'wingo-overlay';
  Object.assign(overlay.style, {
    position: 'fixed', right: '12px', top: '70px',
    width: '320px', maxHeight: '60vh', overflowY: 'auto',
    background: 'rgba(255,255,255,0.98)', border: '2px solid #2b8a3e',
    borderRadius: '8px', padding: '10px', zIndex: 9999999, fontFamily: 'sans-serif',
    boxShadow: '0 4px 10px rgba(0,0,0,0.15)'
  });
  overlay.innerHTML = `
    <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:8px">
      <strong>Wingo Watcher</strong>
      <div>
        <button id="ww-export" style="margin-right:6px;padding:4px 6px">Export CSV</button>
        <button id="ww-clear" style="background:#e74c3c;color:#fff;border:none;padding:4px 6px">Clear</button>
      </div>
    </div>
    <div style="font-size:13px;margin-bottom:6px">Server ID: <span id="ww-server">-</span></div>
    <div style="font-size:13px;margin-bottom:6px">Last: <span id="ww-last">-</span></div>
    <div style="font-size:12px;color:#666;margin-bottom:6px">Today entries: <span id="ww-count">0</span></div>
    <div id="ww-history" style="font-size:12px"></div>
  `;
  document.body.appendChild(overlay);

  document.getElementById('ww-clear').addEventListener('click', ()=>{
    localStorage.removeItem(todayKey());
    renderHistory([]);
  });

  document.getElementById('ww-export').addEventListener('click', ()=>{
    const hist = loadHistory();
    if(!hist.length) return alert('No data to export for today.');
    const rows = [['time','serverId','numbers','analysisJSON']];
    for(const r of hist) rows.push([r.time, r.serverId || '', r.numbers.join(' '), JSON.stringify(r.analysis)]);
    const csv = rows.map(r => r.map(c => `"${String(c).replace(/"/g,'""')}"`).join(',')).join('\n');
    const blob = new Blob([csv], {type:'text/csv;charset=utf-8;'});
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = `wingo-${new Date().toISOString().slice(0,10)}.csv`;
    document.body.appendChild(a); a.click(); a.remove();
    URL.revokeObjectURL(url);
  });

  function renderHistory(h){
    document.getElementById('ww-count').textContent = h.length;
    document.getElementById('ww-server').textContent = h.length ? (h[h.length-1].serverId || '-') : '-';
    document.getElementById('ww-last').textContent = h.length ? h[h.length-1].numbers.join(' ') : '-';
    const container = document.getElementById('ww-history');
    if(!h.length){ container.innerHTML = '<em style="color:#666">No entries yet.</em>'; return; }
    container.innerHTML = h.slice().reverse().slice(0,20).map(it=>{
      const a = it.analysis.map(x=>`${x.n}(${x.size[0]||''},${x.color})`).join(' ');
      return `<div style="padding:6px;border-bottom:1px solid #eee"><strong>${it.time}</strong><div style="font-size:13px">${it.numbers.join(' ')}</div><div style="font-size:12px;color:#444">${a}</div></div>`;
    }).join('');
  }

  // main loop: look for serverId + numbers, record when new
  let lastRecordedSerialized = null;
  function pollOnce(){
    const serverId = findServerId();
    const node = findNumberContainer();
    const numbers = parseNumbersFromNode(node);
    if(!numbers || !numbers.length) return;
    const analysis = numbers.map(n => analyzeDigit(n));
    const now = new Date().toLocaleString();
    const entry = {time: now, serverId: serverId, numbers: numbers, analysis: analysis};
    const serialized = serverId + '|' + numbers.join(',');
    if(serialized !== lastRecordedSerialized){
      // new draw - save
      lastRecordedSerialized = serialized;
      const hist = loadHistory();
      hist.push(entry);
      saveHistory(hist);
      renderHistory(hist);
      console.log('[WingoWatcher] new entry', entry);
    }
  }

  // init
  try { renderHistory(loadHistory()); } catch(e){ console.warn(e); }
  setInterval(pollOnce, POLL_INTERVAL);

})();
