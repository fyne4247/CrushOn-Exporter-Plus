// ==UserScript==
// @name         CrushOn Exporter v2
// @namespace    co-export-plus
// @version      2.5.1
// @description  Export chats and SillyTavern cards, restore avatars, calibrate speakers, and safely export HTML.
// @match        https://crushon.ai/*
// @match        https://www.crushon.ai/*
// @run-at       document-idle
// @grant        GM_addStyle
// @grant        GM_setValue
// @grant        GM_getValue
// @grant        GM_download
// @grant        GM_xmlhttpRequest
// @grant        GM_info
// @connect      *
// ==/UserScript==

(function () {
  'use strict';

  // ---------- cross-manager settings ----------
  // Older script managers expose synchronous GM_getValue/GM_setValue.
  // Safari Userscripts may provide only asynchronous GM.* methods, so use
  // origin-local storage when those synchronous APIs are absent.
  const CO_STORE_PREFIX = 'co-exporter-v2:';

  function coGetValue(key, fallback){
    try {
      if(typeof GM_getValue === 'function') return GM_getValue(key, fallback);
    }catch(_){}
    try {
      const stored = localStorage.getItem(CO_STORE_PREFIX + key);
      return stored === null ? fallback : JSON.parse(stored);
    }catch(_) { return fallback; }
  }

  function coSetValue(key, value){
    try {
      if(typeof GM_setValue === 'function') return GM_setValue(key, value);
    }catch(_){}
    try {
      localStorage.setItem(CO_STORE_PREFIX + key, JSON.stringify(value));
    }catch(_){}
  }

  // ---------- CSS ----------
  const CSS = `

  #coBox{position:fixed;right:12px;bottom:12px;z-index:2147483647;display:flex;gap:4px;padding:4px;background:rgba(15,15,16,.82);backdrop-filter:blur(10px);-webkit-backdrop-filter:blur(10px);border:1px solid #2a2a2a;border-radius:10px;transform-origin:bottom right;opacity:.65;transition:opacity .15s ease;touch-action:none}
  #coBox:hover{opacity:1}
  #coBox button{width:32px;height:32px;padding:0;display:flex;align-items:center;justify-content:center;border-radius:6px;border:1px solid #333;background:#1a1a1b;color:#aaa;font:9px/1 ui-monospace,SFMono-Regular,Menlo,monospace;letter-spacing:.4px;font-weight:600;cursor:pointer;touch-action:manipulation;transition:background .12s,border-color .12s,color .12s,transform .06s;user-select:none;-webkit-user-select:none}
  #coBox button:hover{border-color:#555;color:#eee;background:#222}
  #coBox button:active{transform:scale(.92)}
  #coBox button.on{border-color:#2e7;background:rgba(20,80,40,.55);color:#9f9}
  #coBox button.coGo{border-color:#2e7;background:linear-gradient(#1e5,#0a3);color:#fff;font-size:16px;font-weight:400;letter-spacing:0}
  #coBox button.coGo:hover{background:linear-gradient(#2f6,#1b4)}
  #coBox button.coGo.busy{background:linear-gradient(#a83,#640);border-color:#c95;color:#ffd}
  #coBox button.coScope{font-size:10px}
  #coBox button.coScope.one{border-color:#a6f;background:rgba(95,45,160,.65);color:#eadcff}
  #coBox button.coScope.one:hover{background:rgba(115,60,190,.8)}
  #coBox button.coScope.all{border-color:#f55;background:rgba(160,30,30,.65);color:#ffdede}
  #coBox button.coScope.all:hover{background:rgba(190,40,40,.8)}
  #coBox button#coCfg{font:22px/1 system-ui,-apple-system,"Segoe UI Symbol",sans-serif;color:#ddd}

  #coHUD{position:fixed;left:12px;bottom:12px;z-index:2147483647;background:#0f0f10;color:#e6e6e6;border:1px solid #333;border-radius:10px;padding:10px 12px;font:12px/1.4 monospace;min-width:320px;transform-origin:bottom left}
  #coHUD .ok{color:#7bff7b} #coHUD .warn{color:#ffd479} #coHUD .muted{opacity:.8}
  #coTab{position:fixed;right:12px;bottom:12px;width:28px;height:28px;border-radius:8px;background:#111;border:1px solid #333;color:#fff;z-index:2147483647;display:none;align-items:center;justify-content:center;font:14px monospace}

  .coMenu{position:fixed;background:#151515;color:#fff;border:1px solid #3a3a3a;border-radius:10px;box-shadow:0 10px 32px #000b;display:none;min-width:240px;padding:6px;z-index:2147483647}
  .coMenu button{display:block;width:100%;text-align:left;margin:2px 0;padding:8px 10px;background:#1b1b1c;color:#fff;border:1px solid #333;border-radius:8px;cursor:pointer}

  .coFloating{position:fixed;z-index:2147483647;background:#0f0f10;color:#e6e6e6;border:1px solid #333;border-radius:10px;transform-origin:top left}

  #coPrompt.coFloating{display:none;max-width:460px;width:min(92vw,460px);border-radius:12px;padding:12px 14px;font:13px/1.45 system-ui,-apple-system,Segoe UI,Roboto,Arial;box-shadow:0 18px 48px #000a}
  #coPrompt .title{font-weight:700;margin:0 0 6px 0}
  #coPrompt .msg{white-space:pre-wrap;word-break:break-word}
  #coPrompt .row{display:flex;gap:8px;justify-content:flex-end;margin-top:12px}
  #coPrompt button{padding:10px 12px;border-radius:8px;border:1px solid #3a3a3a;background:#1b1b1c;color:#fff;cursor:pointer}
  #coPrompt .accent{border-color:#2e7;background-image:linear-gradient(#1e5,#0a3)}
  #coPrompt .danger{border-color:#b44;background-image:linear-gradient(#d44,#700)}
  #coPrompt .neutral{border-color:#777;background-image:linear-gradient(#555,#333)}
  input#coScaleSlider{width:100%}

  #coHelp.coFloating{display:none;min-width:300px;max-width:min(92vw,680px)}
  #coHelp header{display:flex;gap:8px;align-items:center;justify-content:space-between;padding:8px 10px;border-bottom:1px solid #333;background:#111;border-radius:10px 10px 0 0}
  #coHelpClose{padding:6px 8px;font-size:12px;border:1px solid #333;color:#fff;background:#222;cursor:pointer}

  .coIcon{position:absolute;top:6px;width:22px;height:22px;display:flex;align-items:center;justify-content:center;font-size:14px;border:1px solid #3a3a3a;border-radius:50%;background:#1b1b1c;color:#bbb;cursor:pointer;transition:opacity .15s ease,color .15s ease,background .15s ease,border-color .15s ease}
  .coHandle{left:6px;opacity:.35;touch-action:none}
  .coIcon:hover{opacity:.7}
  `;
  (typeof GM_addStyle === 'function'
    ? GM_addStyle
    : (css => { const s=document.createElement('style'); s.textContent=css; (document.head||document.documentElement).appendChild(s); }))(CSS);

  // ---------- utils ----------
  const $  = (s,r)=> (r||document).querySelector(s);
  const qsa= (s,r)=> Array.from((r||document).querySelectorAll(s));
  const el = (t,a)=>{const e=document.createElement(t); if(a) for(const k in a){k==='textContent'?e.textContent=a[k]:e.setAttribute(k,a[k])} return e};
  const ts = ()=> new Date().toISOString().replace(/[:.]/g,'-');
  const sleep = ms => new Promise(r=>setTimeout(r,ms));
  const escapeHtml = s => String(s).replace(/[&<>"']/g, c=>({"&":"&amp;","<":"&lt;",">":"&gt;","\"":"&quot;","'":"&#39;"}[c]));
  const stripHTML = s => { const d=document.createElement('div'); d.innerHTML=s; return (d.textContent||'').replace(/\r/g,'').replace(/[ \t]+\n/g,'\n').replace(/\n{3,}/g,'\n\n').trim(); };

  // ---------- CrushOn CDN image proxy bypass ----------
  // CrushOn sometimes asks img.cocdn.co to resize an image that still exists
  // at cdn.crushon.ai. Rewrite ONLY these known proxy URLs to the original.
  // Runs locally; does not replace or re-upload the character's stored image.
  const IMAGE_ATTRS = [
    'srcset', 'src', 'data-srcset', 'data-src', 'data-original',
    'data-lazy-srcset', 'data-lazy-src', 'poster', 'style'
  ];
  const IMAGE_PROXY_RE = /(?:https?:)?\/\/img\.cocdn\.co\/_image\/(?:cover|optimize)\/((?:https?:)?\/\/cdn\.crushon\.ai\/[^\s"'()<>,]+)/gi;
  const IMAGE_SELECTOR = IMAGE_ATTRS.map(
    a => '[' + a + '*="img.cocdn.co"]'
  ).join(',');

  // q and w belong to the image transformer rather than the stored picture.
  function directImageUrl(url){
    if(typeof url !== 'string' || !url.includes('img.cocdn.co')) return url;
    return url.replace(IMAGE_PROXY_RE, (proxy, embedded) => {
      try {
        const unescaped = decodeURIComponent(embedded);
        const direct = new URL(unescaped.startsWith('//') ? 'https:' + unescaped : unescaped);
        if(direct.protocol !== 'https:' || direct.hostname !== 'cdn.crushon.ai') return proxy;
        direct.searchParams.delete('q');
        direct.searchParams.delete('w');
        return direct.href;
      }catch(_){ return proxy; }
    });
  }

  let IMAGE_FIX_ENABLED = coGetValue('co_image_proxy_fix', true) !== false;

  function fixImageElement(element){
    if(!IMAGE_FIX_ENABLED || !element || element.nodeType !== 1) return;
    for(const attr of IMAGE_ATTRS){
      const current = element.getAttribute(attr);
      if(!current || !current.includes('img.cocdn.co')) continue;
      const fixed = directImageUrl(current);
      if(fixed !== current) element.setAttribute(attr, fixed);
    }
  }

  function fixImageTree(root){
    if(!IMAGE_FIX_ENABLED || !root) return;
    if(root.nodeType === 1) fixImageElement(root);
    if(typeof root.querySelectorAll === 'function'){
      for(const child of root.querySelectorAll(IMAGE_SELECTOR)) fixImageElement(child);
    }
  }

  function setImageFixEnabled(on){
    IMAGE_FIX_ENABLED = !!on;
    coSetValue('co_image_proxy_fix', IMAGE_FIX_ENABLED);
    updateToolsMenuLabels();
    if(IMAGE_FIX_ENABLED) fixImageTree(document.documentElement);
    // Disabling leaves successfully restored images intact; subsequent proxy
    // URLs from navigation/lazy loading are no longer rewritten.
  }

  function startImageFix(){
    const root = document.documentElement;
    if(!root) return;
    // The browser may already have loaded the page (document-idle). Catch both
    // existing pictures and React/Next.js's later DOM and lazy-load updates.
    const observer = new MutationObserver(changes => {
      if(!IMAGE_FIX_ENABLED) return;
      for(const change of changes){
        if(change.type === 'attributes') fixImageElement(change.target);
        else for(const added of change.addedNodes) fixImageTree(added);
      }
    });
    observer.observe(root, {
      attributes: true,
      attributeFilter: IMAGE_ATTRS,
      childList: true,
      subtree: true
    });
    fixImageTree(root);
  }

  if(document.documentElement) startImageFix();
  else document.addEventListener('DOMContentLoaded', startImageFix, {once:true});

  // ---------- scale ----------
  let UI_SCALE = (function(){ try{ const v=coGetValue('co_ui_scale', 1); return (typeof v==='number' && v>0)? v : 1; }catch(_){ return 1; } })();
  function applyScale(k){
    UI_SCALE = k;
    ['#coBox','#coHUD','#coTab','#coPrompt','#coHelp','#coMenuExport','#coMenuTools'].forEach(sel=>{
      const n=$(sel); if(n) n.style.transform=`scale(${k})`;
    });
  }

  // ---------- status bar toggle ----------
  let HUD_ENABLED = (function(){ try{ return !!coGetValue('co_hud_enabled', false); }catch(_){ return false; } })();
  function updateToolsMenuLabels(){
    const btn=$('#coToggleHUD'); if(btn) btn.textContent = HUD_ENABLED ? 'Hide status bar' : 'Show status bar';
    const imgBtn=$('#coToggleImages');
    if(imgBtn) imgBtn.textContent = 'Restore missing pictures: ' + (IMAGE_FIX_ENABLED ? 'ON' : 'OFF');
    const regBtn=$('#coToggleRegens');
    if(regBtn) regBtn.textContent = 'Regenerations as branches: ' + (INCLUDE_REGENS ? 'ON' : 'OFF');
    const contBtn=$('#coToggleContinue');
    if(contBtn) contBtn.textContent = 'Skip "continue" messages: ' + (SKIP_CONTINUE ? 'ON' : 'OFF');
  }

  // ---------- export options ----------
  // Off: a regenerated reply that nothing continues from is kept only as a
  // SillyTavern swipe, not as its own branch file.
  let INCLUDE_REGENS = (function(){ try{ return coGetValue('co_include_regens', false) === true; }catch(_){ return false; } })();
  // On: user messages that only say "continue" are left out of every export.
  let SKIP_CONTINUE = (function(){ try{ return coGetValue('co_skip_continue', true) !== false; }catch(_){ return true; } })();
  function setIncludeRegens(on){
    INCLUDE_REGENS=!!on;
    try{ coSetValue('co_include_regens', INCLUDE_REGENS); }catch(_){}
    updateToolsMenuLabels();
  }
  function setSkipContinue(on){
    SKIP_CONTINUE=!!on;
    try{ coSetValue('co_skip_continue', SKIP_CONTINUE); }catch(_){}
    updateToolsMenuLabels();
  }
  // "continue", any case, optionally wrapped in */_/quotes/brackets and
  // followed by . ! ? or … — and nothing else.
  function isContinueText(s){
    const t=String(s||'').replace(/[*_~`"'“”‘’()[\]]/g,'').replace(/[\s.!?…]+$/,'').trim().toLowerCase();
    return t==='continue';
  }
  function updateHUDVisibility(){
    const hud=$('#coHUD'); const box=$('#coBox');
    if(!hud) return;
    if(box?.dataset.minimized==='1'){ hud.style.display='none'; return; }
    hud.style.display = HUD_ENABLED ? '' : 'none';
  }
  function setHUDEnabled(on){
    HUD_ENABLED=!!on;
    try{ coSetValue('co_hud_enabled', HUD_ENABLED); }catch(_){}
    updateHUDVisibility();
    updateToolsMenuLabels();
  }

  // ---------- drag + persistence ----------
  function clamp(v,min,max){ return Math.max(min, Math.min(max, v)); }
  function viewportWH(){ return { w:(window.visualViewport?.width||window.innerWidth), h:(window.visualViewport?.height||window.innerHeight) }; }
  function centerInViewport(el){
    const {w,h}=viewportWH();
    const prev = el.style.display;
    if(prev==='none') el.style.display='block';
    const r=el.getBoundingClientRect();
    const L = window.scrollX + Math.max(8, (w - r.width)/2);
    const T = window.scrollY + Math.max(8, (h - r.height)/2);
    el.style.left=L+'px'; el.style.top=T+'px'; el.style.right='auto'; el.style.bottom='auto';
    if(prev==='none') el.style.display='none';
  }
  function isWayOffscreen(el){
    const {w,h}=viewportWH();
    const r=el.getBoundingClientRect();
    return (r.right < 32) || (r.bottom < 32) || (r.left > w-32) || (r.top > h-32);
  }
  function ensureInViewport(el){
    const pad=8, {w,h}=viewportWH();
    const r=el.getBoundingClientRect();
    if(isWayOffscreen(el)){ centerInViewport(el); return; }
    const left = clamp(r.left, pad, Math.max(pad, w - r.width - pad));
    const top  = clamp(r.top , pad, Math.max(pad, h - r.height - pad));
    el.style.left = left + window.scrollX + 'px';
    el.style.top  = top  + window.scrollY + 'px';
    el.style.right='auto'; el.style.bottom='auto';
  }
  function savePos(key, el){ try{ const r = el.getBoundingClientRect(); coSetValue(key, {left: r.left + window.scrollX, top: r.top + window.scrollY}); }catch{} }
  function restorePos(key, el){
    try{ const v = coGetValue(key, null);
      if(v && typeof v.left==='number' && typeof v.top==='number'){
        el.style.left = v.left + 'px'; el.style.top  = v.top  + 'px'; el.style.right='auto'; el.style.bottom='auto'; return true;
      }
    }catch{}
    return false;
  }
  function restorePosOrCenter(key, el){ if(!restorePos(key, el)) centerInViewport(el); ensureInViewport(el); }

  function unpinWindow(_keyBase, el){ if(el){ el.dataset.pinned='0'; const handle=el.querySelector('.coHandle'); if(handle) handle.style.pointerEvents='auto'; } }
  function addOverlayControls(el, _keyBase, posKey){
    const h=document.createElement('button');
    h.type='button'; h.className='coIcon coHandle'; h.textContent='⠿'; h.title='Drag'; h.setAttribute('aria-label','Drag'); el.appendChild(h);
    return {handle:h};
  }
  function makeDraggable(el, posKey, handle){
    if(!el || !handle) return;
    let startX=0, startY=0, offX=0, offY=0, down=false;
    const start = (cx, cy)=>{ const r=el.getBoundingClientRect(); startX=cx; startY=cy; offX=r.left+window.scrollX; offY=r.top+window.scrollY; down=true; };
    const move = (cx, cy)=>{ if(!down) return; el.style.left=(offX + (cx-startX))+'px'; el.style.top =(offY + (cy-startY))+'px'; el.style.right='auto'; el.style.bottom='auto'; };
    const end = ()=>{ if(!down) return; down=false; ensureInViewport(el); if(posKey) savePos(posKey, el); };
    handle.addEventListener('mousedown', e=>{ e.preventDefault(); start(e.clientX, e.clientY); });
    document.addEventListener('mousemove', e=>move(e.clientX, e.clientY));
    document.addEventListener('mouseup', end);
    handle.addEventListener('touchstart', e=>{ const t=e.touches[0]; e.preventDefault(); start(t.clientX, t.clientY); }, {passive:false});
    document.addEventListener('touchmove', e=>{ const t=e.touches?.[0]; if(!t) return; move(t.clientX, t.clientY); }, {passive:false});
    document.addEventListener('touchend', end);
  }

  // Whole-bar drag (background only; buttons still click normally)
  function makeBarDraggable(el, posKey){
    if(!el) return;
    let sx=0, sy=0, ox=0, oy=0, down=false, moved=false;

    const start = (cx,cy)=>{
      const r = el.getBoundingClientRect();
      sx=cx; sy=cy;
      ox = r.left + window.scrollX;
      oy = r.top  + window.scrollY;
      down = true; moved = false;
    };
    const move = (cx,cy,ev)=>{
      if(!down) return;
      const dx = cx - sx, dy = cy - sy;
      if(!moved && Math.abs(dx) + Math.abs(dy) < 4) return;
      moved = true;
      el.style.left   = (ox + dx) + 'px';
      el.style.top    = (oy + dy) + 'px';
      el.style.right  = 'auto';
      el.style.bottom = 'auto';
      if(ev && ev.cancelable) ev.preventDefault();
    };
    const end = ()=>{
      if(!down) return;
      down = false;
      if(moved){
        ensureInViewport(el);
        if(posKey) savePos(posKey, el);
      }
    };

    el.addEventListener('mousedown', e=>{
      if(e.target.closest('button')) return;
      e.preventDefault();
      start(e.clientX, e.clientY);
    });
    el.addEventListener('touchstart', e=>{
      if(e.target.closest('button')) return;
      const t = e.touches?.[0]; if(!t) return;
      start(t.clientX, t.clientY);
    }, {passive:true});

    document.addEventListener('mousemove', e=>move(e.clientX, e.clientY, e));
    document.addEventListener('touchmove', e=>{
      const t = e.touches?.[0]; if(!t) return;
      move(t.clientX, t.clientY, e);
    }, {passive:false});

    document.addEventListener('mouseup', end);
    document.addEventListener('touchend', end);
  }

  function clampVisiblePopups(){
    ['#coPrompt','#coHelp'].forEach(sel=>{
      const n=$(sel);
      if(n && n.style.display!=='none'){ ensureInViewport(n); }
    });
  }
  window.addEventListener('resize', clampVisiblePopups);
  window.addEventListener('orientationchange', clampVisiblePopups);
  try{ window.visualViewport?.addEventListener('resize', clampVisiblePopups); }catch(_){}

  // ---------- robust UI boot ----------
  function ensureUI(){
    if(!$('#coBox')){ try{ buildMainUI(); }catch(e){} return; }
    const box=$('#coBox'); const hud=$('#coHUD'); const tab=$('#coTab');
    if(box?.dataset.minimized==='1'){ if(tab) tab.style.display='flex'; applyScale(UI_SCALE); return; }
    if(getComputedStyle(box).display==='none') box.style.display='';
    if(hud){ hud.style.display = HUD_ENABLED ? '' : 'none'; }
    if(tab) tab.style.display='none';
    applyScale(UI_SCALE);
  }
  if(document.readyState==='complete' || document.readyState==='interactive') ensureUI();
  else document.addEventListener('DOMContentLoaded', ensureUI, {once:true});
  // UI reboots only when CrushOn replaces/removes our toolbar or page body.
  // Don't scan the full chat DOM on every token, image, or message mutation.
  let uiRebootPending = false;
  function scheduleUIReboot(){
    if(uiRebootPending) return;
    uiRebootPending = true;
    requestAnimationFrame(()=>{ uiRebootPending=false; ensureUI(); });
  }
  const uiRootObserver = new MutationObserver(changes=>{
    if(changes.some(m => [...m.removedNodes].some(n =>
      n.nodeType===1 && (n.id==='coBox' || n.contains?.($('#coBox')) || n.tagName==='BODY')
    ))) scheduleUIReboot();
  });
  try{ uiRootObserver.observe(document.documentElement,{childList:true}); }catch(_){}
  function observeUIBody(){
    if(!document.body) return;
    const toolbar = $('#coBox');
    const watch = new MutationObserver(changes=>{
      if(changes.some(m=>[...m.removedNodes].some(n=>n===toolbar || n.contains?.(toolbar)))) scheduleUIReboot();
    });
    watch.observe(document.body,{childList:true});
  }
  if(document.body) observeUIBody();
  else document.addEventListener('DOMContentLoaded', observeUIBody, {once:true});
  setTimeout(scheduleUIReboot,1200);

  // ---------- state ----------
  let CHAT_CONTAINER = null;
  const records=[]; const seen=new Set();
  // character profile captured alongside the chat log: {c, id, source, gate, q}
  let CHAR = null;
  let CAPTURE_SOURCE = 'none';
  let CAPTURE_NOTE = '';
  let LAST_TREE_NOTE = '';
  // ERP/SillyTavern capture facts for chat_metadata.crushon (see exportSillyTavernJSONL).
  let CAPTURE_META = {capture:'none', complete:false, pages_fetched:0, stopped_early:false};
  let SPEAKER_SIG = (function(){ try{ return coGetValue('co_speaker_sig', null);}catch(_){ return null; } })();
  let AI_NAME_SNAPSHOT = (function(){ try{ return coGetValue('co_ai_name_snapshot', null);}catch(_){ return null; } })();

  // Format toggles for the top bar
  const EXPORT_FORMATS = {
    png:   (()=>{ try{ return coGetValue('co_fmt_png',   true); }catch(_){ return true; } })(),
    jsonl: (()=>{ try{ return coGetValue('co_fmt_jsonl', true); }catch(_){ return true; } })(),
    txt:   (()=>{ try{ return coGetValue('co_fmt_txt',   true); }catch(_){ return true; } })(),
  };
  // ONE = ↓ exports the branch on screen; ALL = every branch of this chat.
  let EXPORT_SCOPE = (()=>{ try{ return coGetValue('co_scope', 'one')==='all' ? 'all' : 'one'; }catch(_){ return 'one'; } })();
  function setScope(scope){
    EXPORT_SCOPE = scope==='all' ? 'all' : 'one';
    try{ coSetValue('co_scope', EXPORT_SCOPE); }catch(_){}
    const b = $('#coScope');
    if(b){
      b.textContent = EXPORT_SCOPE==='all' ? 'ALL' : 'ONE';
      b.className = 'coScope '+EXPORT_SCOPE;
      b.title = EXPORT_SCOPE==='all'
        ? 'ALL: ↓ exports every branch of this chat (.zip). Click for ONE.'
        : 'ONE: ↓ exports the branch on screen. Click for ALL.';
    }
  }

  function setFormat(key, on){
    EXPORT_FORMATS[key] = !!on;
    try{ coSetValue('co_fmt_'+key, EXPORT_FORMATS[key]); }catch(_){}
    const b = $('#coBtn_'+key);
    if(b) b.classList.toggle('on', EXPORT_FORMATS[key]);
  }

  // ---------- selectors ----------
  const SEL_CONTAINER = '#chat-container';
  // Updated: descendant matching so AI rows nested under div.w-full are found
  const SEL_GROUPS = '#chat-container :is(div.relative.flex.w-full.items-start, div.group.relative.flex.w-full.items-start)';
  // Updated: include .prose fallback
  const SEL_MARKDOWN_BLOCK = "[class*='MarkdownText_'], .prose";

  // ---------- UI ----------
  buildMainUI();

  function buildMainUI(){
    if($('#coBox')) return;

    // ---- the bar: 5 small squares ----
    const bar = el('div',{id:'coBox'});
    bar.setAttribute('role','toolbar');
    bar.setAttribute('aria-label','CrushOn Exporter');

    const mkTog = (key,label,title)=>{
      const b = el('button',{type:'button',title});
      b.id = 'coBtn_'+key;
      b.textContent = label;
      if(EXPORT_FORMATS[key]) b.classList.add('on');
      b.onclick = ()=>setFormat(key, !EXPORT_FORMATS[key]);
      return b;
    };

    const bPng   = mkTog('png',   'PNG',   'Export character card as PNG');
    const bJsonl = mkTog('jsonl', 'JSONL', 'Export chat as SillyTavern .jsonl');
    const bTxt   = mkTog('txt',   'TXT',   'Export chat as plain .txt');

    const bGo = el('button',{id:'coGo',type:'button',title:'Capture and export selected formats'});
    bGo.className = 'coGo';
    bGo.textContent = '↓';
    bGo.onclick = exportByScope;

    const bScope = el('button',{id:'coScope',type:'button'});
    bScope.onclick = ()=>setScope(EXPORT_SCOPE==='all' ? 'one' : 'all');

    const bCfg = el('button',{id:'coCfg',type:'button',title:'More tools'});
    bCfg.textContent = '⚙';
    bCfg.onclick = ()=>toggleMenu('export', bCfg);

    bar.append(bPng, bJsonl, bTxt, bGo, bScope, bCfg);
    document.body.appendChild(bar);
    setScope(EXPORT_SCOPE);

    // ---- HUD (unchanged, opt-in) ----
    const hud = el('div',{id:'coHUD'});
    hud.innerHTML = `<div><b>Status:</b> <span id=coStatus class=muted>idle</span></div>
    <div>Captured: <span id=coMsgs>0</span> | Groups: <span id=coGroups>0</span> | Split → <span id=coSplit>0U/0A</span></div>
    <div>Chat log: <span id=coHasLog class=muted>none</span> | Character: <span id=coHasCard class=muted>none</span></div>`;
    document.body.appendChild(hud);

    // ---- reopen tab after Hide panel ----
    const tab = el('div',{id:'coTab', textContent:'≡'});
    document.body.appendChild(tab);
    tab.onclick = ()=>{
      const b = $('#coBox'), h = $('#coHUD');
      if(b){ b.dataset.minimized='0'; b.style.display=''; }
      if(h){ h.dataset.minimized='0'; h.style.display = HUD_ENABLED ? '' : 'none'; }
      tab.style.display='none';
    };

    // ---- simplified settings menu ----
    const mExp = el('div',{id:'coMenuExport', class:'coMenu'});

    const btnHudToggle = document.createElement('button');
    btnHudToggle.id = 'coToggleHUD';
    btnHudToggle.textContent = HUD_ENABLED ? 'Hide status bar' : 'Show status bar';
    btnHudToggle.onclick = ()=>setHUDEnabled(!HUD_ENABLED);

    const btnImgToggle = document.createElement('button');
    btnImgToggle.id = 'coToggleImages';
    btnImgToggle.textContent = 'Restore missing pictures: ' + (IMAGE_FIX_ENABLED ? 'ON' : 'OFF');
    btnImgToggle.onclick = ()=>setImageFixEnabled(!IMAGE_FIX_ENABLED);

    const btnRegenToggle = document.createElement('button');
    btnRegenToggle.id = 'coToggleRegens';
    btnRegenToggle.title = 'Off: regenerated replies nothing continues from stay as swipes instead of separate branch files';
    btnRegenToggle.onclick = ()=>setIncludeRegens(!INCLUDE_REGENS);

    const btnContinueToggle = document.createElement('button');
    btnContinueToggle.id = 'coToggleContinue';
    btnContinueToggle.title = 'On: your messages that only say "continue" are left out of exports';
    btnContinueToggle.onclick = ()=>setSkipContinue(!SKIP_CONTINUE);

    mExp.append(
      btnHudToggle,
      btnImgToggle,
      btnRegenToggle,
      btnContinueToggle,
      divider(),
      mk('Export .html (bonus)', exportHTML),
      mk('This chat: all branches (separate files)', ()=>exportAllBranches('separate')),
      mk('ALL chats: every branch (.zip)', exportAllChats),
      divider(),
      mk('Calibrate speakers (fallback)', manualCalibrate),
      mk('Clear captured', ()=>clearCaptured(false)),
      divider(),
      mk('Resize UI…', showResizeUI),
      mk('Read me', ()=>{
        const h = $('#coHelp');
        h.style.display='block';
        closeMenus();
        restorePosOrCenter('co_pos_help', h);
      }),
      mk('Dump character data', probeCharacterData),
      mk('Dump history response', dumpHistoryResponse),
      divider(),
      mk('Hide panel', ()=>hidePanel(true))
    );
    document.body.appendChild(mExp);

    document.body.appendChild(buildHelp());
    document.body.appendChild(buildPrompt());

    document.addEventListener('click', e=>{
      if(!(e.target.closest && (
        e.target.closest('#coBox') ||
        e.target.closest('.coMenu') ||
        e.target.closest('#coPrompt') ||
        e.target.closest('#coHelp')
      ))) closeMenus();
    }, true);

    window.addEventListener('keydown', e=>{
      if(e.altKey && String(e.key).toLowerCase()==='e'){
        const b = $('#coBox'), h = $('#coHUD'), t = $('#coTab');
        if(b){ b.style.display=''; b.dataset.minimized='0'; }
        if(h){ h.style.display = HUD_ENABLED ? '' : 'none'; h.dataset.minimized='0'; }
        if(t) t.style.display='none';
      }
    }, true);

    CHAT_CONTAINER = $(SEL_CONTAINER) || CHAT_CONTAINER;
    setCounts('0U/0A');
    setStatus('idle','muted');
    applyScale(UI_SCALE);

    makeBarDraggable(bar, 'co_pos_box');
    restorePos('co_pos_box', bar);

    const c2 = addOverlayControls(hud, 'co_hud', 'co_pos_hud');
    makeDraggable(hud, 'co_pos_hud', c2.handle);
    restorePos('co_pos_hud', hud);

    updateHUDVisibility();
    updateToolsMenuLabels();
  }

  function divider(){
    const d = document.createElement('div');
    d.style.cssText = 'height:1px;background:#3a3a3a;margin:7px 4px';
    return d;
  }

  function mk(t,fn){const b=document.createElement('button'); b.textContent=t; b.onclick=fn; return b;}

  function toggleMenu(which,btn){
    const id = which==='export' ? '#coMenuExport' : '#coMenuTools';
    const menu=$(id); if(!menu) return;
    const other= which==='export' ? '#coMenuTools' : '#coMenuExport';
    const om=$(other); if(om) om.style.display='none';
    if(menu.style.display==='block'){menu.style.display='none'; return;}
    menu.style.display='block';
    const r=btn.getBoundingClientRect();
    const mw = menu.offsetWidth || 260;
    menu.style.left = Math.max(8, Math.min((window.visualViewport?.width||window.innerWidth) - mw - 8, r.left + window.scrollX))+'px';
    menu.style.top  = Math.max(8, r.top + window.scrollY - menu.offsetHeight - 8)+'px';
  }

  function closeMenus(){ ['#coMenuExport','#coMenuTools'].forEach(s=>{const n=$(s); if(n) n.style.display='none';}); }

  function hidePanel(h=true){
    const b=$('#coBox'); const hUD=$('#coHUD');
    if(b){ b.dataset.minimized = h ? '1' : '0'; b.style.display = h ? 'none' : ''; }
    if(hUD){
      hUD.dataset.minimized = h ? '1' : '0';
      hUD.style.display = h ? 'none' : (HUD_ENABLED ? '' : 'none');
    }
    const t=$('#coTab'); if(t) t.style.display = h ? 'flex' : 'none';
    closeMenus();
  }

  function setStatus(t,c){ const s=$('#coStatus'); if(s){ s.className=c||'muted'; s.textContent=t; } }
  function setCounts(split){
    const gg=getGroups(), groups=$('#coGroups'), msgs=$('#coMsgs'), role=$('#coSplit');
    if(groups) groups.textContent=String(gg.length);
    if(msgs) msgs.textContent=String(records.length);
    if(split!=null && role) role.textContent=split;
  }

  // Reflect what's actually in hand: a chat log, a character profile, or both.
  function setCardBadge(){
    const log=$('#coHasLog');
    if(log){
      const n=records.length;
      log.textContent = n ? `${n} msgs` : 'none';
      log.className = n ? 'ok' : 'muted';
    }
    const el=$('#coHasCard'); if(!el) return;
    if(!CHAR){ el.textContent='none'; el.className='muted'; return; }
    if(!CHAR.gate.allowed){ el.textContent='blocked'; el.className='warn'; return; }
    if(!CHAR.q.ok){ el.textContent='unusable'; el.className='warn'; return; }
    const name = (cleanAiNameExact(pick(CHAR.c, CARD_KEYS.name))||'?').slice(0,18);
    el.textContent = CHAR.q.defHidden ? `${name} (partial)` : `${name} (full)`;
    el.className = CHAR.q.defHidden ? 'warn' : 'ok';
  }

  // Fetch the character profile for the current page. Never throws.
  async function captureCharacter(onStep){
    try{
      const found = await findCharacter(onStep);
      if(!found) { CHAR=null; setCardBadge(); return null; }
      const gate = await accessCheck(found.c);
      const q = cardQuality(found.c, found.source);
      CHAR = { c:found.c, id:found.id, source:found.source, gate, q };
    }catch(_){ CHAR=null; }
    setCardBadge();
    return CHAR;
  }

  // One-line human summary of the character half of a capture.
  function charSummary(){
    if(!CHAR) return 'Character profile: not found on this page.';
    if(!CHAR.gate.allowed) return `Character profile: blocked — ${CHAR.gate.why}.`;
    if(!CHAR.q.ok) return `Character profile: unusable — ${CHAR.q.why}.`;
    const name = cleanAiNameExact(pick(CHAR.c, CARD_KEYS.name)) || '?';
    const got = CHAR.q.filled.join(', ');
    return CHAR.q.defHidden
      ? `Character profile: “${name}” — partial (${got}).\nThe creator hid the definition, so personality/scenario/examples aren't available.`
      : `Character profile: “${name}” — full (${got}).`;
  }

  // ---------- help ----------
  function buildHelp(){
    const p=el('div',{id:'coHelp', class:'coFloating'});
    p.innerHTML=`<header><b>CrushOn Exporter • Read Me</b><button id=coHelpClose>Close</button></header>
      <div style="padding:12px 14px;font:13px/1.55 system-ui,-apple-system,Segoe UI,Roboto,Arial">
        <p><b>Toggles:</b> PNG / JSONL / TXT pick what every export produces: <b>↓</b>, all branches, and all chats. They persist across reloads. In branch and bulk exports, TXT adds a readable transcript per branch and PNG adds the character card.</p>
        <p><b>Regenerations as branches</b> (⚙, off by default): when off, a regenerated reply that nothing continues from is kept as a SillyTavern swipe on its branch instead of becoming its own branch file. Continued alternatives are always branches.</p>
        <p><b>Skip "continue" messages</b> (⚙, on by default): your messages that only say "continue" (any case, with or without punctuation or asterisks) are left out of every export. The next message's parent ID points past them.</p>
        <p><b>↓ Export:</b> Reads the branch shown on the page from CrushOn's message tree when it can be identified; otherwise tries a history request observed in this tab, then visible chat rows. When no history request is discoverable, the chat export may be partial. Unknown timestamps are left blank. Character lookup runs only when PNG is selected.</p>
        <p><b>Formats:</b> .txt and .html carry [user]/[ai] labels; .jsonl imports back into SillyTavern; .png is a SillyTavern v2 card (JSON embedded in the avatar PNG). HTML recaptures the current chat automatically. Public/unlisted profiles and characters you own can export as PNG; private characters that aren't yours are blocked. If CrushOn returns no real definition, the .png is refused rather than saved with placeholder text. Creators who hide their definition export as description + greeting only, and the card says so.</p>
        <p><b>ONE / ALL</b> (next to ↓): <b>ONE</b> (purple) = ↓ exports the branch on screen. <b>ALL</b> (red) = ↓ exports every branch of this chat as one .zip. The choice persists.</p>
        <p><b>What gets exported:</b> <b>ONE + ↓</b> = this chat, the branch on screen. <b>ALL + ↓</b> = every branch of this chat. <b>⚙ → ALL chats</b> = every branch of every chat.</p>
        <p><b>Export all branches:</b> ALL + ↓ (or ⚙ → separate files) reads CrushOn's full message tree (read-only) and saves one SillyTavern .jsonl per branch (root → leaf, original order and timestamps) plus manifest.json describing forks and shared ancestry. The .zip option is one download (best on phones); "separate files" saves each file individually. Missing or partial history is reported, never filled in.</p>
        <p><b>Export ALL chats:</b> ⚙ → lists every chat in your Recent list and saves every branch of every chat into one .zip (a folder per character + manifest.json). Requests go one at a time about every 2 s; you can stop at any point and keep what was collected. Best on desktop: phones may run out of memory on large archives.</p>
        <p><b>Restore missing pictures:</b> Enabled by default. Bypasses CrushOn's failing img.cocdn.co transformer for existing cdn.crushon.ai images, including lazy-loaded pictures. Switch it on/off from ⚙. This is a local display fix, not a re-upload.</p>
        <p><b>⚙ More:</b> Status bar, image fix, regenerations, skip continue, HTML export, all branches as separate files, export all chats, manual calibration, clear, resize, read me, character data dump, history dump, hide panel.</p>
        <p><b>Drag:</b> Grab the bar's background. Buttons still click normally.</p>
      </div>`;
    p.querySelector('#coHelpClose').onclick=()=>{ unpinWindow('co_help', p); p.style.display='none'; };
    const c=addOverlayControls(p, 'co_help', 'co_pos_help'); makeDraggable(p, 'co_pos_help', c.handle);
    return p;
  }

  // ---------- centered prompt ----------
  function buildPrompt(){
    const p=el('div',{id:'coPrompt', class:'coFloating'});
    p.innerHTML=`<div class="title">CrushOn Exporter</div>
      <div class="msg" id="coPromptMsg">…</div>
      <div class="row">
        <button id="coPromptExtra" class="neutral" style="display:none">Start over</button>
        <button id="coPromptCancel" class="danger">Cancel</button>
        <button id="coPromptOK" class="accent">ok</button>
      </div>`;
    p.querySelector('#coPromptCancel').onclick=()=>hidePrompt();
    p.querySelector('#coPromptOK').onclick=()=>hidePrompt();
    p.querySelector('#coPromptExtra').onclick=()=>{};
    const c=addOverlayControls(p,'co_prompt','co_pos_prompt'); makeDraggable(p,'co_pos_prompt', c.handle);
    return p;
  }

  function showPrompt(msg,opts={}){
    updatePrompt(msg,opts);
    const n=$('#coPrompt');
    n.style.display='block';
    restorePosOrCenter('co_pos_prompt', n);
  }

  function updatePrompt(message,opts={}){
    $('#coPromptMsg').textContent=message;
    const ok=$('#coPromptOK'), cancel=$('#coPromptCancel'), extra=$('#coPromptExtra');
    const setBtn=(b,on,txt,cls,show=true)=>{ b.onclick=on||(()=>hidePrompt()); b.textContent=txt??b.textContent; b.className=cls||b.className; b.style.display=show?'':'none'; };
    setBtn(ok,opts.onOK,opts.okText??'ok','accent',opts.ok!==false);
    setBtn(cancel,opts.onCancel,opts.cancelText??'Cancel','danger',opts.cancel!==false);
    setBtn(extra,opts.onExtra,opts.extraText??'Start over','neutral',opts.extra===true);
  }

  function hidePrompt(){
    const n=$('#coPrompt');
    savePos('co_pos_prompt', n);
    unpinWindow('co_prompt', n);
    n.style.display='none';
  }

  // ---------- Resize UI ----------
  function showResizeUI(){
    const prev=UI_SCALE;
    showPrompt('',{
      okText:'accept',
      cancelText:'cancel',
      extraText:'default',
      extra:true,
      onOK:()=>{ try{ coSetValue('co_ui_scale', UI_SCALE); }catch{} hidePrompt(); },
      onCancel:()=>{ applyScale(prev); hidePrompt(); },
      onExtra:()=>{
        const s=$('#coScaleSlider');
        if(s){ s.value='100'; }
        setPct(100);
        applyScale(1);
        try{ coSetValue('co_ui_scale', 1); }catch{}
      }
    });

    const msg=$('#coPromptMsg');
    msg.innerHTML = `<div style="display:flex;align-items:center;gap:10px;min-width:260px">
        <div style="min-width:88px">Scale: <b id="coScalePct"></b></div>
        <input id="coScaleSlider" type="range" min="70" max="160" step="5" />
      </div>`;

    const slider=$('#coScaleSlider');
    const pct=$('#coScalePct');

    function setPct(v){ pct.textContent=v+'%'; }

    const start=Math.round(prev*100);
    slider.value=String(start);
    setPct(start);

    slider.oninput=()=>{
      const v=Number(slider.value);
      setPct(v);
      applyScale(v/100);
      clampVisiblePopups();
    };
  }

  // ---------- DOM helpers ----------
  function resolveContainer(){
    // CrushOn navigates chats as an SPA. Refresh a cached container if the old
    // node was detached, otherwise a later export can read the previous route.
    const live = $(SEL_CONTAINER);
    if(live && live !== CHAT_CONTAINER) CHAT_CONTAINER = live;
    if(CHAT_CONTAINER && CHAT_CONTAINER.isConnected) return CHAT_CONTAINER;
    CHAT_CONTAINER = live || null;
    return CHAT_CONTAINER || document.body;
  }
  function getGroups(){ return qsa(SEL_GROUPS, resolveContainer()); }
  function getBlock(g){ return g.querySelector(SEL_MARKDOWN_BLOCK) || g; }
  function alignRight(el){ const root=resolveContainer().getBoundingClientRect(); const b=(getBlock(el)).getBoundingClientRect(); return (b.left + b.width/2) > (root.left + root.width/2); }
  function hasCharacterLink(el){ return !!el.querySelector('a[href*="/character/"], [data-character-id]'); }
  function hasNameChip(el){ return !!el.querySelector('.ml-2 span, [class*="name"], [class*="Name"]'); }
  function featureFromRow(row){ return { right: alignRight(row), charLink: hasCharacterLink(row), nameChip: hasNameChip(row) }; }
  function similarity(f,sig){ let s=0; if(f.right===sig.right) s+=3; if(f.charLink===sig.charLink) s+=3; if(f.nameChip===sig.nameChip) s+=1; return s; }

  // ---------- AI name ----------
  function cleanAiNameExact(x){
    if(!x) return 'AI';
    let t=String(x).trim();
    t=t.replace(/^\s*(chat\s*with|with)\s+/i,'');
    t=t.replace(/\s*\|.*$/,'');
    t=t.replace(/\s*[-–—·•]\s*.+$/,'');
    t=t.replace(/\s*\((?:.*?)\)\s*$/,'');
    t=t.replace(/^[#:\s]+|[\s#•|]+$/g,'');
    t=t.replace(/\s{2,}/g,' ').trim();
    if(t.length>64) t=t.slice(0,64).trim();
    return t || 'AI';
  }

  function extractAiNameFromRow(row){
    const cands=[
      row.querySelector('a[href*="/character/"]')?.textContent,
      row.querySelector('[data-character-name]')?.getAttribute('data-character-name'),
      row.querySelector('.ml-2 span, [class*="name"], [class*="Name"]')?.textContent,
      document.querySelector('meta[property="og:title"]')?.content,
      document.title
    ].filter(Boolean);

    for(const raw of cands){
      const t=cleanAiNameExact(raw);
      if(t && t!=='AI') return t;
    }

    return 'AI';
  }

  const currentAIName = ()=> (SPEAKER_SIG && SPEAKER_SIG.aiName) || AI_NAME_SNAPSHOT || 'AI';

  function setAIName(name){
    const n = cleanAiNameExact(name||'AI');
    SPEAKER_SIG = Object.assign({}, SPEAKER_SIG||{}, { aiName: n });
    try{ coSetValue('co_speaker_sig', SPEAKER_SIG); }catch{}
    AI_NAME_SNAPSHOT = n;
    try{ coSetValue('co_ai_name_snapshot', AI_NAME_SNAPSHOT); }catch{}
  }

  // ---------- calibration ----------
  function clearCalibration(src){
    SPEAKER_SIG = null;
    AI_NAME_SNAPSHOT = null;
    try{
      coSetValue('co_speaker_sig', null);
      coSetValue('co_ai_name_snapshot', null);
    }catch{}
    if(src) setStatus(`calibration reset (${src})`,'warn');
  }

  const labelFor = who => who==='USER' ? 'user' : 'ai';

  function autoCalibrateTry(){
    const rows=getGroups();
    if(!rows.length) return null;

    const candUser = rows.find(r=>alignRight(r) && !hasCharacterLink(r)) || rows.find(r=>alignRight(r));
    const candAI   = rows.find(r=>hasCharacterLink(r)) || rows.find(r=>!alignRight(r));

    if(!candUser || !candAI || candUser===candAI) return null;

    const u=featureFromRow(candUser);
    const a=featureFromRow(candAI);
    const aiName = extractAiNameFromRow(candAI);

    return { sig:{ user:u, ai:a, aiName: cleanAiNameExact(aiName), scope:location.pathname } };
  }

  async function manualCalibrate(){
    clearCaptured(true);

    try{
      await promptOKCancel('Calibrate: Select YOUR bubble.\nPress ok to start.','ok','Cancel');
      const userNode = await pickOneRow();

      await promptOKCancel('Calibrate: Select AI bubble.\nPress ok to start.','ok','Cancel');
      const aiNode = await pickOneRow();

      const u=featureFromRow(userNode);
      const a=featureFromRow(aiNode);
      const aiName=cleanAiNameExact(extractAiNameFromRow(aiNode));

      showPrompt(`Confirm calibration:\nUser = USER\nAI = ${aiName}`,{
        okText:'ok',
        extra:true,
        extraText:'Start over',

        onOK: async ()=>{
          SPEAKER_SIG={user:u, ai:a, aiName, manual:true, scope:location.pathname};
          try{ coSetValue('co_speaker_sig', SPEAKER_SIG); }catch{}
          setAIName(aiName);

          updatePrompt('Capturing…',{cancel:false, ok:false});
          await doCaptureAfterCalibrated();
          await finishCapture();
        },

        onExtra: ()=>{ hidePrompt(); manualCalibrate(); },
        onCancel: ()=>{ cancelAndClear(); }
      });
    }catch{
      cancelAndClear();
    }
  }

  function pickOneRow(){
    return new Promise((resolve,reject)=>{
      hidePrompt();

      const selector=SEL_GROUPS;

      const onClick=e=>{
        const row=e.target.closest?.(selector);
        if(!row) return;
        cleanup();
        resolve(row);
      };

      const onKey=e=>{
        if(e.key==='Escape'){
          cleanup();
          reject(new Error('cancel'));
        }
      };

      function cleanup(){
        document.removeEventListener('click',onClick,true);
        document.removeEventListener('keydown',onKey,true);
      }

      document.addEventListener('click',onClick,true);
      document.addEventListener('keydown',onKey,true);
    });
  }

  // Shared tail for every capture path. Character lookup can be skipped for
  // chat-only exports so PNG is the only format that pays that cost.
  async function finishCapture(afterCapture, opts={}){
    const captureCharacterProfile = opts.captureCharacterProfile !== false;
    const silentSummary = !!opts.silentSummary;

    if(captureCharacterProfile){
      updatePrompt(
        'Captured chat. Now reading the character profile…',
        {cancel:false, ok:false}
      );

      await captureCharacter(
        s => updatePrompt(
          'Reading character profile… ('+s+')',
          {cancel:false, ok:false}
        )
      );
    }else{
      // captureNow() clears CHAR at the start. Keep it cleared rather than
      // accidentally reusing a character from a previous SPA route.
      CHAR=null;
    }

    setCardBadge();

    const nLog = records.length;
    const cardOK = !!(CHAR && CHAR.gate.allowed && CHAR.q.ok);
    const both = nLog>0 && cardOK;

    const head = both
      ? 'Captured: chat log + character profile.'
      : nLog>0
        ? 'Captured: chat log only.'
        : cardOK
          ? 'Captured: character profile only.'
          : 'Captured: nothing usable.';

    const msg = [
      head,
      '',
      `Chat log: ${nLog} messages (split ${$('#coSplit').textContent}; source: ${CAPTURE_SOURCE}).`,
      CAPTURE_NOTE,
      captureCharacterProfile ? charSummary() : 'Character profile: not requested.',
      '',
      cardOK
        ? 'You can export .txt / .html / SillyTavern .jsonl and the character .png.'
        : 'You can export .txt / .html / SillyTavern .jsonl. The character .png is unavailable.'
    ].join('\n');

    setStatus(
      both
        ? 'captured log + character'
        : nLog>0
          ? 'captured log'
          : cardOK
            ? 'captured character'
            : 'nothing captured',
      (nLog>0||cardOK) ? 'ok' : 'warn'
    );

    if(typeof afterCapture==='function'){
      await afterCapture();
      return;
    }

    if(silentSummary){
      hidePrompt();
      return;
    }

    updatePrompt(msg, {
      okText:'ok',
      onOK:hidePrompt,
      cancel:false
    });
  }

  // ---------- authenticated conversation-history capture ----------
  // Prefer the site's own history responses over a virtualized chat DOM.
  // CrushOn does not publish a stable history API contract: only REPLAY a
  // matching request already made by the current page, using its exact URL
  // and the user's existing session. No endpoint or conversation ID guesses.
  function conversationScopeIds(){
    const ids=new Set();
    const path=location.pathname.split('/').filter(Boolean);
    for(let i=0;i+1<path.length;i++){
      if(/^(chat|chats|conversation|conversations|session|sessions)$/i.test(path[i])){
        ids.add(decodeURIComponent(path[i+1]));
      }
    }
    const params=new URLSearchParams(location.search);
    for(const k of ['chatId','conversationId','sessionId','chat_id','conversation_id','session_id']){
      const v=params.get(k);
      if(v) ids.add(v);
    }
    return [...ids].filter(x=>x.length>=4);
  }

  function parseObservedInput(url){
    try{
      const raw=new URL(url).searchParams.get('input');
      return raw ? JSON.parse(raw) : null;
    }catch(_){return null;}
  }

  function historyRequestCandidates(){
    const ids=conversationScopeIds();
    if(!ids.length || !performance.getEntriesByType) return [];
    const urls=performance.getEntriesByType('resource')
      .map(x=>x.name)
      .filter(Boolean).reverse();
    const seen=new Set(), found=[];
    for(const link of urls){
      if(seen.has(link)) continue;
      seen.add(link);
      try{
        const url=new URL(link);
        if(url.origin!==location.origin || !url.pathname.startsWith('/api/trpc/')) continue;
        const procedure=decodeURIComponent(url.pathname.split('/api/trpc/')[1]||'');
        if(!/^(?:chat|chatroom|message|messages|conversation|conversations|history|room|session)[A-Za-z]*\./i.test(procedure)) continue;
        if(!/(?:get|list|query|history|messages|fetch)/i.test(procedure)) continue;
        const input=parseObservedInput(url.href);
        if(!input) continue;
        const serialized=JSON.stringify(input);
        // Accept only requests explicitly scoped to the current page. Do not
        // accidentally export a different chat included in another tab's UI.
        if(!ids.some(id=>serialized.includes(JSON.stringify(id)))) continue;
        found.push({url:url.href,input});
        if(found.length>=8) break;
      }catch(_){}
    }
    return found;
  }

  function apiMessageRole(m){
    if(typeof m?.is_user==='boolean') return m.is_user?'USER':'AI';
    if(typeof m?.isUser==='boolean') return m.isUser?'USER':'AI';
    if(typeof m?.is_user_message==='boolean') return m.is_user_message?'USER':'AI';
    if(typeof m?.isAI==='boolean') return m.isAI?'AI':'USER';
    const who=m?.role ?? m?.senderType ?? m?.senderRole ?? m?.authorRole ??
      m?.messageRole ?? m?.sender?.role ?? m?.sender?.type ?? m?.author?.role;
    const role=String(who??'').trim().toLowerCase();
    if(/^(user|human|player|me)$/i.test(role)) return 'USER';
    if(/^(ai|assistant|character|bot|model)$/i.test(role)) return 'AI';
    return null;
  }

  function apiMessageBody(m){
    let v=m?.content ?? m?.message ?? m?.text ?? m?.body ?? m?.messageText;
    if(v && typeof v==='object'){
      v=v.text ?? v.content ?? v.body ?? v.message;
    }
    if(Array.isArray(v)){
      v=v.map(x=>typeof x==='string'?x:(x?.text??'')).filter(Boolean).join('\n');
    }
    return typeof v==='string' ? v : '';
  }

  function normalizeHistoryMessages(messages){
    if(!Array.isArray(messages) || !messages.length) return [];
    const rows=[];
    for(const m of messages){
      if(!m || typeof m!=='object') continue;
      const who=apiMessageRole(m), content=apiMessageBody(m);
      if(!who || !content.trim()) continue;
      // API strings are text unless explicitly marked as HTML; never treat
      // arbitrary user/model text containing '<' as trusted markup.
      const html= m?.contentType==='text/html' || m?.format==='html'
        ? safeExportHtml(content)
        : escapeHtml(content).replace(/\r?\n/g,'<br>');
      const isHtml=m?.contentType==='text/html' || m?.format==='html';
      const parentId=m.parent ?? m.parentId ?? m.parent_id ?? null;
      rows.push({
        id:String(m.id ?? m.messageId ?? m._id ?? ''),
        seq:rows.length,
        who, html,
        ...(isHtml ? {} : {raw:content}),
        crushon:{
          message_id:String(m.id ?? m.messageId ?? m._id ?? '') || null,
          parent_message_id:parentId!=null && parentId!=='' ? String(parentId) : null,
          edited:m.isEdited===true || m.is_edited===true
        },
        t:parseMessageTime(m.createdAt ?? m.created_at ?? m.sentAt ??
          m.sendTime ?? m.timestamp ?? m.time ?? m.created ?? m.date)
      });
    }
    // Never mistake an unrelated array for chat history.
    if(rows.length<2 || rows.length<Math.ceil(messages.length*0.75)) return [];
    return rows;
  }

  function extractHistoryPage(response){
    let best=null;
    const seen=new Set();
    function visit(value,depth=0){
      if(!value || typeof value!=='object' || depth>12 || seen.has(value)) return;
      seen.add(value);
      if(Array.isArray(value)){
        const rows=normalizeHistoryMessages(value);
        if(rows.length && (!best || rows.length>best.rows.length)){
          best={rows,meta:null};
        }
        if(value.length<100) value.forEach(v=>visit(v,depth+1));
        return;
      }
      for(const [k,v] of Object.entries(value)){
        if(['messages','history','items','records','edges','nodes','data'].includes(k) && Array.isArray(v)){
          const arr=k==='edges'?v.map(e=>e?.node??e):v;
          const rows=normalizeHistoryMessages(arr);
          if(rows.length && (!best || rows.length>best.rows.length)){
            best={rows,meta:value};
          }
        }
      }
      for(const v of Object.values(value)) visit(v,depth+1);
    }
    visit(response);
    return best;
  }

  function nextHistoryUrl(url,responseMeta){
    const page=responseMeta?.pageInfo ?? responseMeta?.pagination ?? responseMeta;
    if(!page || typeof page!=='object') return null;
    const next=page.nextCursor ?? page.next_cursor ?? page.nextPageCursor;
    const more=page.hasNextPage ?? page.hasMore ?? page.has_more;
    let u;
    try{u=new URL(url);}catch(_){return null;}
    const value=parseObservedInput(u.href);
    if(!value || typeof value!=='object') return null;
    // Existing tRPC batch inputs are in {0:{json:{...}}}; single inputs
    // can be {json:{...}} or plain input. Keep the original shape.
    const payload=value?.['0']?.json ?? value?.json ?? value;
    if(!payload || typeof payload!=='object') return null;
    if(next!==undefined && next!==null && next!==''){
      const key=['cursor','before','nextCursor','lastMessageId'].find(k=>Object.hasOwn(payload,k));
      if(!key) return null;
      payload[key]=next;
    }else if(more===true && Number.isInteger(payload.page) && payload.page>=0){
      payload.page+=1;
    }else{
      return null;
    }
    u.searchParams.set('input',JSON.stringify(value));
    return u.href;
  }

  async function requestHistoryPage(url){
    const controller=new AbortController();
    const timeout=setTimeout(()=>controller.abort(),8000);
    try{
      const response=await fetch(url,{
        credentials:'include', cache:'no-store', signal:controller.signal,
        headers:{accept:'application/json','trpc-accept':'application/json'}
      });
      if(!response.ok) return null;
      return await response.json();
    }finally{clearTimeout(timeout);}
  }

  async function captureHistoryFromObservedRequest(){
    for(const candidate of historyRequestCandidates()){
      const combined=[], ids=new Set(), visited=new Set();
      let url=candidate.url, page=0;
      for(; page<12 && url && !visited.has(url);page++){
        visited.add(url);
        let response;
        try{response=await requestHistoryPage(url);}catch(_){break;}
        if(!response) break;
        const found=extractHistoryPage(response);
        if(!found?.rows?.length) break;
        for(const row of found.rows){
          // Dedup by stable server ID when provided; retain repeated text.
          const key=row.id || `page:${page}:row:${row.seq}`;
          if(ids.has(key)) continue;
          ids.add(key);
          combined.push({...row,seq:combined.length});
        }
        url=nextHistoryUrl(url,found.meta);
      }
      if(combined.length){
        if(combined.every(r=>r.t!==null)) combined.sort((a,b)=>a.t-b.t || a.seq-b.seq);
        // Completeness is unknowable for this generic path: never claim it.
        combined.meta={capture:'observed_request', complete:false, pages_fetched:page, stopped_early:page>=12 && !!url};
        return combined;
      }
    }
    return null;
  }

  // ---------- full conversation tree, branch + bulk export ----------
  // Verified against crushon.ai (Oct 2026). The chat page loads its history
  // with a POST tRPC call, chat.getMessageHistory, input
  //   {conversationId, voiceAudioOption:{voiceId, withAudio, withNarration}, pager?}
  // The result holds the WHOLE message tree, including branches the UI is not
  // showing, as `mapping`:
  //   {<id>: {id, parent, children:[ids], role, content, createdAt, isEdited,
  //           messageType, …}}
  // role 1 = user, 2 = character, 3 = empty system anchor at the root;
  // messageType 5 = the greeting. Later pages are requested by echoing the
  // returned `pager`; an empty page ends the history. Branch switching in the
  // UI is client-side only. The chat list comes from the same calls CrushOn's
  // inbox uses: character.queryRecentCharacters + chat.listConversations.
  // Everything here only READS. Requests go one at a time through a pacing
  // gate with back-off on 429/5xx so bulk runs don't hammer the site.
  // This section reads only. It never sends, edits, deletes or switches anything.
  const TREE_PROC = 'chat.getMessageHistory';
  const TREE_MAX_PAGES = 25;
  const SCRIPT_VERSION = (()=>{ try{ return GM_info.script.version; }catch(_){ return '2.5.1'; } })();

  // ---- request pacing ----
  const PACE = { last:0, minGap:400, jitter:0, cancelled:false };
  function setPace(minGap, jitter){ PACE.minGap=minGap; PACE.jitter=jitter||0; }
  async function paceWait(){
    const gap=PACE.minGap + Math.random()*PACE.jitter;
    const wait=PACE.last + gap - Date.now();
    if(wait>0) await sleep(wait);
    PACE.last=Date.now();
  }
  function isRetryable(res){ return res.status===0 || res.status===429 || res.status===502 || res.status===503 || res.status===504; }

  // One paced tRPC request with retries. method 'GET' (query) or 'POST'.
  async function trpcCall(proc, input, method){
    let attempt=0;
    for(;;){
      if(PACE.cancelled) return {ok:false, status:0, error:'cancelled', cancelled:true};
      await paceWait();
      const res=await trpcOnce(proc, input, method);
      if(res.ok || !isRetryable(res) || attempt>=3) return res;
      const backoff=(res.retryAfter ? res.retryAfter*1000 : 8000*Math.pow(2,attempt));
      attempt++;
      console.warn(`[CrushOn Exporter] ${proc} → ${res.status||res.error}; retry ${attempt} in ${Math.round(backoff/1000)}s`);
      await sleep(Math.min(backoff, 90000));
    }
  }

  async function trpcOnce(proc, input, method){
    const controller=new AbortController();
    const timer=setTimeout(()=>controller.abort(), 25000);
    try{
      const batch=JSON.stringify({0:{json:input}});
      const url=`${location.origin}/api/trpc/${proc}?batch=1` + (method==='GET' ? '&input='+encodeURIComponent(batch) : '');
      const r=await fetch(url,{
        method:method==='GET' ? 'GET' : 'POST', credentials:'include', cache:'no-store', signal:controller.signal,
        headers:{'content-type':'application/json','trpc-accept':'application/json',accept:'application/json'},
        body:method==='GET' ? undefined : batch
      });
      let j=null;
      try{ j=await r.json(); }catch(_){}
      const slot=Array.isArray(j) ? j[0] : j;
      if(!r.ok || !slot || slot.error){
        const msg=slot?.error?.json?.message || slot?.error?.message || ('HTTP '+r.status);
        const ra=Number(r.headers.get('retry-after'));
        return {ok:false, status:r.status, error:String(msg), retryAfter:Number.isFinite(ra)&&ra>0 ? ra : null};
      }
      return {ok:true, status:r.status, data:slot?.result?.data?.json ?? slot?.result?.data ?? null};
    }catch(e){
      return {ok:false, status:0, error:e?.name==='AbortError' ? 'request timed out' : String(e?.message||e)};
    }finally{ clearTimeout(timer); }
  }

  function chatRouteIds(){
    const m=location.pathname.match(/\/character\/([^/?#]+)\/chat\/([^/?#]+)/i);
    if(!m) return null;
    try{
      return {characterId:decodeURIComponent(m[1]), conversationId:decodeURIComponent(m[2])};
    }catch(_){ return null; }
  }

  // Fetch every page of a conversation's message tree. Returns
  // {ok, nodes:Map, memory:[], complete, stoppedEarly, pages, notes:[], server, raw:[]}
  // or {ok:false, error}.
  async function fetchConversationTree(conversationId, onStep, keepRaw){
    const nodes=new Map(), notes=[], raw=[];
    let memory=[], pager=null, complete=false, pages=0, server=null;

    while(pages<TREE_MAX_PAGES){
      onStep && onStep(`reading message tree, page ${pages+1}…`);
      const input={conversationId, voiceAudioOption:{voiceId:'', withAudio:false, withNarration:false}};
      if(pager) input.pager=pager;
      const res=await trpcCall(TREE_PROC, input, 'POST');
      pages++;
      if(keepRaw) raw.push({request:input, response:res.ok ? res.data : {error:res.error, status:res.status}});

      if(!res.ok){
        if(pages===1) return {ok:false, error:`${TREE_PROC} unavailable (${res.error})`, cancelled:!!res.cancelled, raw};
        notes.push(`Page ${pages} of the history failed (${res.error}); older messages may be missing.`);
        break;
      }

      const d=res.data;
      const map=d && d.mapping;
      if(!map || typeof map!=='object' || Array.isArray(map)){
        if(pages===1) return {ok:false, error:`${TREE_PROC} returned no message tree ("mapping"). CrushOn may have changed its API`, raw};
        notes.push(`Page ${pages} had no message tree; older messages may be missing.`);
        break;
      }

      let added=0;
      for(const [k,v] of Object.entries(map)){
        if(!v || typeof v!=='object') continue;
        const id=String(v.id ?? k);
        if(!nodes.has(id)){ nodes.set(id, v); added++; }
      }
      if(Array.isArray(d.memoryMessages)) memory=memory.concat(d.memoryMessages);
      server={
        hasHistory:d.hasHistory ?? null,
        historyMessageLimit:d.historyMessageLimit ?? null,
        pager:d.pager && typeof d.pager==='object'
          ? {offset:d.pager.offset ?? null, count:d.pager.count ?? null, total:d.pager.total ?? null}
          : null
      };

      const more=d.hasHistory===true || (d.nextCursor!==null && d.nextCursor!==undefined && d.nextCursor!=='');
      if(!added || !more){ complete=true; break; }
      if(!d.pager || typeof d.pager!=='object'){
        notes.push('CrushOn reported older history but exposed no pager to request it; older messages may be missing.');
        break;
      }
      if(pager && JSON.stringify(d.pager)===JSON.stringify(pager)){
        notes.push('History pager stopped advancing; older messages may be missing.');
        break;
      }
      pager=d.pager;
    }

    if(!complete && pages>=TREE_MAX_PAGES){
      notes.push(`Stopped after ${TREE_MAX_PAGES} history pages; older messages may be missing.`);
    }
    if(memory.length){
      notes.push(`CrushOn returned ${memory.length} "memory" message(s) (carried-over context outside the tree); they open every branch file, flagged memory:true.`);
    }
    return {ok:true, nodes, memory, complete, stoppedEarly:!complete, pages, notes, server, raw};
  }

  function treeRole(m){
    const r=m?.role;
    if(r===1 || r==='1') return 'USER';
    if(r===2 || r==='2') return 'AI';
    if(r===3 || r==='3') return 'SYSTEM';
    return apiMessageRole(m);
  }

  function nodeText(m){
    return typeof m?.content==='string' ? m.content : apiMessageBody(m);
  }

  // A user message that only says "continue" (when that option is on).
  function isSkippedNode(m){
    return SKIP_CONTINUE && treeRole(m)==='USER' && isContinueText(nodeText(m));
  }

  // Empty system anchors (CrushOn's tree root) are structural, not messages.
  function isEmittedNode(m){
    return !(treeRole(m)==='SYSTEM' && !String(nodeText(m)).trim()) && !isSkippedNode(m);
  }

  function nodeTime(m){
    return parseMessageTime(m?.createdAt ?? m?.created_at ?? m?.timestamp ?? null);
  }

  // Inferred from CrushOn's data: AI alternatives are regenerations; a user
  // alternative flagged isEdited came from editing; otherwise "make branch".
  function altKind(m){
    const who=treeRole(m);
    if(who==='AI') return 'regen';
    if(who==='USER') return m?.isEdited===true ? 'edit' : 'branch';
    return 'branch';
  }

  // Turn the raw node map into ordered branches. Child order follows the
  // server's `children` arrays (CrushOn's own order); a node's `parent` field is
  // treated as authoritative. Nothing is invented: gaps are reported as flags.
  function analyzeTree(nodes, opts={}){
    const warnings=[];
    const parentOf=id=>{
      const p=nodes.get(id)?.parent;
      return (p===null || p===undefined || p==='') ? null : String(p);
    };
    const byTime=(a,b)=>((nodeTime(nodes.get(a))??0)-(nodeTime(nodes.get(b))??0)) || (a<b?-1:a>b?1:0);

    const children=new Map();
    const missingChildren=new Map();
    for(const id of nodes.keys()){ children.set(id, []); }
    for(const [id,m] of nodes){
      const listed=Array.isArray(m.children) ? m.children.map(String) : [];
      for(const c of listed){
        if(!nodes.has(c)){
          if(!missingChildren.has(id)) missingChildren.set(id, []);
          missingChildren.get(id).push(c);
        }else if(parentOf(c)===id){
          if(!children.get(id).includes(c)) children.get(id).push(c);
        }else{
          warnings.push(`Message ${c} is listed under ${id} but names ${parentOf(c)||'no parent'} as its parent; kept under its own parent.`);
        }
      }
    }
    const unlisted=[];
    for(const id of nodes.keys()){
      const p=parentOf(id);
      if(p && nodes.has(p) && !children.get(p).includes(id)) unlisted.push(id);
    }
    unlisted.sort(byTime).forEach(id=>children.get(parentOf(id)).push(id));
    if(unlisted.length) warnings.push(`${unlisted.length} message(s) were not listed by their parent; appended in time order.`);

    // Which children get their own branches. With regenerations off, an AI
    // reply with siblings and no children of its own is a dead-end regeneration:
    // it stays in `children` (so it is still written as a swipe) but gets no
    // branch. If every sibling is such a dead end, the newest one is kept.
    const includeRegens=opts.includeRegens ?? INCLUDE_REGENS;
    const walk=new Map(), folded=new Set();
    let foldedRegens=0;
    for(const [id,kids] of children){
      if(includeRegens || kids.length<2){ walk.set(id, kids); continue; }
      const dead=kids.filter(c=>treeRole(nodes.get(c))==='AI' && !children.get(c).length && !missingChildren.has(c));
      if(!dead.length){ walk.set(id, kids); continue; }
      let keep=kids.filter(c=>!dead.includes(c));
      if(!keep.length) keep=[dead.slice().sort(byTime).pop()];
      foldedRegens+=kids.length-keep.length;
      kids.forEach(c=>{ if(!keep.includes(c)) folded.add(c); });
      walk.set(id, keep);
    }

    const roots=[...nodes.keys()].filter(id=>parentOf(id)===null).sort(byTime);
    const detached=[...nodes.keys()].filter(id=>{ const p=parentOf(id); return p && !nodes.has(p); }).sort(byTime);
    if(detached.length) warnings.push(`${detached.length} message chain(s) reference a parent that CrushOn did not return; exported as detached branches flagged "incomplete_ancestry".`);
    if(roots.length>1) warnings.push(`The tree has ${roots.length} separate roots.`);

    // Iterative DFS → one branch per leaf.
    const depth=new Map(), branches=[], visited=new Set();
    for(const start of [...roots, ...detached]){
      const stack=[[start, [start]]];
      while(stack.length){
        const [id, path]=stack.pop();
        if(visited.has(id)){ warnings.push(`Cycle or duplicate reference at message ${id}; skipped.`); continue; }
        visited.add(id);
        depth.set(id, path.length-1);
        const kids=walk.get(id).filter(c=>!visited.has(c));
        if(!kids.length){
          branches.push({
            leaf:id, path,
            detachedRoot:detached.includes(start),
            truncatedTail:missingChildren.has(id) || (walk.get(id).length>0)
          });
          continue;
        }
        for(let i=kids.length-1;i>=0;i--) stack.push([kids[i], path.concat(kids[i])]);
      }
    }
    const unreached=[...nodes.keys()].filter(id=>!visited.has(id) && !folded.has(id));
    if(unreached.length) warnings.push(`${unreached.length} message(s) could not be placed in the tree (cyclic references) and were left out.`);
    for(const [id,list] of missingChildren){
      warnings.push(`Message ${id} lists ${list.length} child message(s) that CrushOn did not return.`);
    }

    const forks=[...nodes.keys()].filter(id=>visited.has(id) && walk.get(id).length>1);
    const pad=Math.max(2, String(branches.length).length);
    branches.forEach((b,i)=>{
      b.number=i+1;
      b.id='branch-'+String(i+1).padStart(pad,'0');
      b.forkPoints=[];
      for(let k=0;k<b.path.length-1;k++){
        const kids=walk.get(b.path[k]);
        if(kids.length>1){
          b.forkPoints.push({message_id:b.path[k], depth:k, choice:kids.indexOf(b.path[k+1])+1, of:kids.length, chosen_message_id:b.path[k+1], kind:altKind(nodes.get(b.path[k+1]))});
        }
      }
    });
    // Shared ancestry: longest common prefix with any earlier branch = parent branch.
    branches.forEach((b,i)=>{
      let best=null;
      for(let j=0;j<i;j++){
        const o=branches[j].path; let n=0;
        while(n<o.length && n<b.path.length && o[n]===b.path[n]) n++;
        if(n && (!best || n>best.nodes)) best={
          branch_id:branches[j].id,
          nodes:n,
          jsonl_messages:b.path.slice(0,n).filter(id=>isEmittedNode(nodes.get(id))).length,
          diverges_after_message_id:b.path[n-1],
          first_unique_message_id:b.path[n] ?? null
        };
      }
      b.sharedPrefix=best;
      b.forkKind=best && best.first_unique_message_id ? altKind(nodes.get(best.first_unique_message_id)) : null;
    });
    const nodeBranches=new Map();
    branches.forEach(b=>b.path.forEach(id=>{
      if(!nodeBranches.has(id)) nodeBranches.set(id, []);
      nodeBranches.get(id).push(b.id);
    }));

    let newest=null;
    for(const b of branches){
      const t=nodeTime(nodes.get(b.leaf));
      if(t!==null && (!newest || t>newest.t)) newest={b, t};
    }

    return {nodes, branches, forks, roots, detached, children, walk, folded, foldedRegens, includeRegens, depth, nodeBranches, missingChildren, warnings, newestLeaf:newest?.b.leaf ?? null};
  }

  // Which branch is the page showing? Compares rendered message text with each
  // branch. Returns a leaf id only for a clear winner, otherwise null.
  function renderedLeaf(nodes, branches){
    const key=s=>String(s||'').replace(/\{\{(?:user|char)\}\}/gi,'').replace(/\(edited\)\s*$/i,'').toLowerCase().replace(/[^\p{L}\p{N}]+/gu,'').slice(0,80);
    let dom;
    try{ dom=getGroups().map(g=>key(getBlock(g).textContent)).filter(Boolean); }catch(_){ return null; }
    if(!dom.length || !branches.length) return null;
    const scored=branches.map(b=>{
      const keys=b.path.map(id=>nodes.get(id)).filter(m=>m && String(nodeText(m)).trim()).map(m=>key(nodeText(m)));
      const set=new Set(keys);
      let hits=0; dom.forEach(x=>{ if(set.has(x)) hits++; });
      let tail=0;
      for(let i=1;i<=Math.min(keys.length, dom.length);i++){
        if(keys[keys.length-i]===dom[dom.length-i]) tail++; else break;
      }
      return {leaf:b.leaf, hits, tail, score:tail*100000+hits};
    }).sort((a,b)=>b.score-a.score);
    const top=scored[0], second=scored[1];
    if(top.tail<1 || top.hits/dom.length<0.6) return null;
    if(second && second.score===top.score) return null;
    return top.leaf;
  }

  // With regens off, the page may be showing a regeneration that has no branch
  // of its own. Match against the full tree, then return the branch that
  // carries it as a swipe: the one sharing the longest history with it.
  function renderedLeafFor(nodes, tree){
    const direct=renderedLeaf(nodes, tree.branches);
    if(direct || tree.includeRegens || !tree.foldedRegens) return direct;
    const full=analyzeTree(nodes, {includeRegens:true});
    const leaf=renderedLeaf(nodes, full.branches);
    return leaf ? closestBranch(tree, full.branches.find(b=>b.leaf===leaf).path)?.leaf ?? null : null;
  }

  function closestBranch(tree, path){
    let best=null, bestN=-1;
    for(const b of tree.branches){
      let n=0;
      while(n<b.path.length && n<path.length && b.path[n]===path[n]) n++;
      if(n>bestN){ best=b; bestN=n; }
    }
    return best;
  }

  function exportAIName(){
    const n=currentAIName();
    if(n && n!=='AI') return n;
    return cleanAiNameExact(document.querySelector('meta[property="og:title"]')?.content || document.title || 'AI');
  }

  function safeFilePart(s, fallback){
    // Readable, filesystem-safe, deterministic.
    return String(s||'').replace(/[\\/:*?"<>|\u0000-\u001f]+/g,'_').replace(/\s+/g,' ').trim().replace(/^\.+/,'').slice(0,48) || fallback;
  }

  function isoOrBlank(t){ return t!==null && t!==undefined && Number.isFinite(t) ? new Date(t).toISOString() : ''; }

  // CrushOn's "memory" messages: real user/character messages returned beside
  // the tree (no parent links, all stamped at the chat's creation). They are the
  // context the chat started from, so every branch file opens with them.
  function memoryRows(fetched){
    return (fetched.memory||[]).filter(m=>m && typeof m==='object' && !isSkippedNode(m)).map(m=>({id:String(m.id ?? ''), m}));
  }
  function forkIndexOf(b, fetched){
    return b.sharedPrefix ? memoryRows(fetched).length + b.sharedPrefix.jsonl_messages : null;
  }

  // Nearest ancestor that is actually written out, stepping over skipped
  // "continue" messages so parent_message_id never points at a missing line.
  function emittedParent(m, tree){
    const up=x=>(x?.parent!=null && x.parent!=='') ? String(x.parent) : null;
    let p=up(m);
    const skipped=[];
    while(p && tree?.nodes?.has(p) && isSkippedNode(tree.nodes.get(p))){
      skipped.push(p);
      p=up(tree.nodes.get(p));
    }
    return {id:p, skipped};
  }

  // Per-message ERP/SillyTavern fields shared by every API-based export.
  function crushonMessageExtra(m, id, tree){
    const t=nodeTime(m);
    const ep=emittedParent(m, tree);
    const sibs=m.parent!=null && tree ? tree.children.get(String(m.parent)) : null;
    const who=treeRole(m);
    const isGreeting=m.messageType===5 || m.messageType==='5';
    return {
      message_id:id,
      parent_message_id:ep.id,
      ...(ep.skipped.length ? {skipped_continue_ids:ep.skipped} : {}),
      created_at:isoOrBlank(t),
      time_source:t!==null ? 'server' : 'none',
      edited:m.isEdited===true,
      ...(isGreeting ? {greeting:true} : {}),
      role:m.role ?? null,
      message_type:m.messageType ?? null,
      ...(sibs && sibs.length>1 ? {alternative:{index:sibs.indexOf(id)+1, of:sibs.length, kind:altKind(m)}} : {}),
      ...(tree && (tree.children.get(id)||[]).length>1 ? {branch_point:true} : {}),
      ...(who ? {} : {unknown_role:true}),
      ...(Array.isArray(m.extendedContents) && m.extendedContents.length ? {extended_contents:m.extendedContents.length} : {})
    };
  }

  // Regenerated character replies → standard SillyTavern swipes (all of them,
  // whether or not they have their own branch).
  function swipeFields(m, id, tree){
    const nodes=tree.nodes;
    const sibs=m.parent!=null ? (tree.children.get(String(m.parent))||[]) : [];
    const aiSibs=sibs.filter(s=>treeRole(nodes.get(s))==='AI');
    if(treeRole(m)!=='AI' || aiSibs.length<2 || aiSibs.length!==sibs.length) return {};
    return {
      swipes:aiSibs.map(s=>String(nodeText(nodes.get(s))||'')),
      swipe_id:aiSibs.indexOf(id),
      swipe_info:aiSibs.map(s=>({send_date:isoOrBlank(nodeTime(nodes.get(s))), extra:{crushon:{message_id:s}}}))
    };
  }

  // One SillyTavern JSONL chat for one root→leaf path.
  function branchJsonl(b, ctx){
    const {nodes, tree, aiName, route, exportedAt, fetched, renderedLeafId, sourceUrl}=ctx;
    const treeComplete=fetched.complete && !tree.detached.length && !tree.missingChildren.size;
    const branchComplete=treeComplete && !b.detachedRoot && !b.truncatedTail;
    const firstTime=b.path.map(id=>nodeTime(nodes.get(id))).find(t=>t!==null);
    const header={
      user_name:'unused',
      character_name:'unused',
      create_date:isoOrBlank(firstTime ?? null),
      chat_metadata:{
        source:'CrushOn Exporter PLUS',
        capture_source:'CrushOn history API (chat.getMessageHistory, full message tree)',
        capture_note:branchComplete
          ? `Branch ${b.number} of ${tree.branches.length}. Complete: CrushOn returned the whole tree for this chat.`
          : `Branch ${b.number} of ${tree.branches.length}. INCOMPLETE: CrushOn's history had gaps; see crushon.warnings.`,
        source_url:sourceUrl,
        imported_from:'crushon.ai',
        exported_at:exportedAt,
        crushon:{
          exporter_version:SCRIPT_VERSION,
          character_id:route.characterId,
          chat_id:route.conversationId,
          ...(ctx.chatName ? {chat_name:ctx.chatName} : {}),
          capture:'message_tree',
          complete:branchComplete,
          pages_fetched:fetched.pages,
          stopped_early:!!fetched.stoppedEarly,
          branch_id:b.id,
          branch_number:b.number,
          branch_count:tree.branches.length,
          parent_branch_id:b.sharedPrefix?.branch_id ?? null,
          fork_index:forkIndexOf(b, fetched),
          fork_kind:b.forkKind,
          fork_kind_basis:b.forkKind ? 'inferred: AI alternative = regen; user alternative with isEdited = edit; otherwise branch' : null,
          divergence_message_id:b.sharedPrefix?.first_unique_message_id ?? null,
          leaf_message_id:b.leaf,
          memory_messages:memoryRows(fetched).length,
          rendered_in_page:renderedLeafId===null || renderedLeafId===undefined ? null : renderedLeafId===b.leaf,
          newest_leaf:tree.newestLeaf===b.leaf,
          fork_points:b.forkPoints,
          incomplete_ancestry:b.detachedRoot,
          truncated_tail:b.truncatedTail,
          regens_as_branches:tree.includeRegens,
          skip_continue:SKIP_CONTINUE,
          ...(branchComplete ? {} : {warnings:tree.warnings.concat(fetched.notes)})
        }
      }
    };
    const rows=[header];
    let unknownRoles=0;
    for(const {id, m} of memoryRows(fetched)){
      const who=treeRole(m);
      if(!who) unknownRoles++;
      const t=nodeTime(m);
      rows.push({
        name:who==='USER' ? 'User' : (who==='AI' ? aiName : 'System'),
        is_user:who==='USER',
        is_system:who==='SYSTEM' || !who,
        send_date:isoOrBlank(t),
        mes:String(nodeText(m)||''),
        extra:{crushon:{
          message_id:id || null, parent_message_id:null,
          created_at:isoOrBlank(t), time_source:t!==null ? 'server' : 'none',
          edited:m.isEdited===true, memory:true,
          role:m.role ?? null, context_type:m.contextType ?? null,
          ...(who ? {} : {unknown_role:true})
        }}
      });
    }
    for(const id of b.path){
      const m=nodes.get(id);
      if(!isEmittedNode(m)) continue;
      const who=treeRole(m);
      if(!who) unknownRoles++;
      const isUser=who==='USER';
      const row={
        name:isUser ? 'User' : (who==='AI' ? aiName : 'System'),
        is_user:isUser,
        is_system:who==='SYSTEM' || !who,
        send_date:isoOrBlank(nodeTime(m)),
        mes:String(nodeText(m)||''),       // verbatim server text
        extra:{crushon:crushonMessageExtra(m, id, tree)}
      };
      Object.assign(row, swipeFields(m, id, tree));
      rows.push(row);
    }
    return {text:rows.map(r=>JSON.stringify(r)).join('\n'), rows, messages:rows.length-1, unknownRoles, complete:branchComplete};
  }

  // The same branch as a readable [user]/[ai] transcript (the ↓ .txt format).
  function branchTxt(b, ctx, rows){
    const meta=rows[0].chat_metadata, cr=meta.crushon;
    const split=cr.parent_branch_id
      ? `, splits from ${cr.parent_branch_id} at message ${cr.fork_index+1} (${cr.fork_kind})`
      : '';
    const lines=[
      `# ${ctx.aiName} — ${ctx.chatName || ctx.route.conversationId} — ${b.id}`,
      'URL: '+meta.source_url,
      'AI: '+ctx.aiName,
      'Exported: '+meta.exported_at,
      'Source: '+meta.capture_source,
      'Coverage: '+meta.capture_note,
      `Branch: ${b.id} of ${cr.branch_count}${split}`,
      ''
    ];
    for(const r of rows.slice(1)){
      if(r.is_system) continue;
      lines.push(`[${r.is_user ? 'user' : 'ai'}] `+r.mes);
      lines.push('');
    }
    return lines.join('\n');
  }

  function exportOptions(tree){
    return {
      regens_as_branches:tree ? tree.includeRegens : INCLUDE_REGENS,
      folded_regens:tree ? tree.foldedRegens : null,
      skip_continue:SKIP_CONTINUE,
      formats:['jsonl','txt','png'].filter(k=>EXPORT_FORMATS[k])
    };
  }

  // Manifest path of a branch: the .jsonl when JSONL is on, else the .txt
  // (null when neither was written).
  function branchFileRefs(f, folder){
    if(!EXPORT_FORMATS.jsonl) return {file:EXPORT_FORMATS.txt ? folder+f.txtName : null};
    return {file:folder+f.name, ...(EXPORT_FORMATS.txt ? {txt_file:folder+f.txtName} : {})};
  }

  function branchFileName(charName, chatId, branchId, ext='jsonl'){
    return `${safeFilePart(charName,'character')} - ${safeFilePart(chatId,'chat')} - ${branchId}.${ext}`;
  }

  // Build every branch file for one chat from an already-fetched tree.
  function buildChatBranchFiles(fetched, ctxBase){
    const tree=analyzeTree(fetched.nodes);
    const ctx={...ctxBase, nodes:fetched.nodes, tree, fetched};
    const files=tree.branches.map(b=>{
      const j=branchJsonl(b, ctx);
      return {
        name:branchFileName(ctx.aiName, ctx.route.conversationId, b.id),
        data:j.text,
        txtName:branchFileName(ctx.aiName, ctx.route.conversationId, b.id, 'txt'),
        txt:EXPORT_FORMATS.txt ? branchTxt(b, ctx, j.rows) : null,
        messages:j.messages, unknownRoles:j.unknownRoles, complete:j.complete, branch:b
      };
    });
    return {tree, files, ctx};
  }

  function chatManifestEntries(files, ctx, folder){
    return files.map(f=>({
      ...branchFileRefs(f, folder),
      character_id:ctx.route.characterId,
      character_name:ctx.aiName,
      chat_id:ctx.route.conversationId,
      ...(ctx.chatName ? {chat_name:ctx.chatName} : {}),
      branch_id:f.branch.id,
      parent_branch_id:f.branch.sharedPrefix?.branch_id ?? null,
      fork_index:forkIndexOf(f.branch, ctx.fetched),
      fork_kind:f.branch.forkKind,
      leaf_message_id:f.branch.leaf,
      messages:f.messages,
      complete:f.complete,
      newest_leaf:ctx.tree.newestLeaf===f.branch.leaf,
      rendered_in_page:ctx.renderedLeafId===null || ctx.renderedLeafId===undefined ? null : ctx.renderedLeafId===f.branch.leaf
    }));
  }

  // Detailed tree description for a single chat (Export all branches).
  function buildBranchManifest(ctx, files){
    const {nodes, tree, fetched, route, aiName, exportedAt, renderedLeafId}=ctx;
    const roleCount={USER:0, AI:0, SYSTEM:0, unknown:0};
    for(const m of nodes.values()) roleCount[treeRole(m)||'unknown']++;
    const nodeList=[], listed=new Set();
    for(const b of tree.branches) for(const id of b.path){
      if(listed.has(id)) continue;
      listed.add(id);
      const m=nodes.get(id);
      nodeList.push({
        id,
        parent:m.parent!=null ? String(m.parent) : null,
        children:tree.children.get(id)||[],
        role:treeRole(m) || 'unknown',
        message_type:m.messageType ?? null,
        created_at:isoOrBlank(nodeTime(m)),
        edited:m.isEdited===true,
        depth:tree.depth.get(id) ?? null,
        content_chars:String(nodeText(m)||'').length,
        branches:tree.nodeBranches.get(id)||[]
      });
    }
    return {
      format:'crushon-exporter-plus/branch-tree',
      format_version:2,
      exporter_version:SCRIPT_VERSION,
      exported_at:exportedAt,
      source_url:ctx.sourceUrl,
      character_id:route.characterId,
      chat_id:route.conversationId,
      character_name:aiName,
      options:exportOptions(tree),
      capture:{
        procedure:TREE_PROC,
        pages_fetched:fetched.pages,
        stopped_early:!!fetched.stoppedEarly,
        complete:fetched.complete && !tree.detached.length && !tree.missingChildren.size,
        server:fetched.server,
        memory_messages:fetched.memory.length,
        notes:fetched.notes
      },
      counts:{
        nodes:nodes.size,
        user_messages:roleCount.USER,
        character_messages:roleCount.AI,
        system_nodes:roleCount.SYSTEM,
        unknown_role:roleCount.unknown,
        roots:tree.roots.length,
        detached_chains:tree.detached.length,
        fork_points:tree.forks.length,
        branches:tree.branches.length
      },
      rendered_branch:renderedLeafId ? (tree.branches.find(b=>b.leaf===renderedLeafId)?.id ?? null) : null,
      rendered_branch_note:renderedLeafId
        ? 'Matched by comparing the messages rendered in the page with each branch.'
        : 'Could not confidently tell which branch the page is showing; no branch is marked as rendered.',
      files:chatManifestEntries(files, ctx, ''),
      branches:tree.branches.map((b,i)=>({
        id:b.id,
        number:b.number,
        ...branchFileRefs(files[i], ''),
        messages:files[i].messages,
        complete:files[i].complete,
        leaf_message_id:b.leaf,
        leaf_created_at:isoOrBlank(nodeTime(nodes.get(b.leaf))),
        newest_leaf:tree.newestLeaf===b.leaf,
        rendered_in_page:renderedLeafId===null ? null : renderedLeafId===b.leaf,
        parent_branch_id:b.sharedPrefix?.branch_id ?? null,
        fork_index:forkIndexOf(b, fetched),
        fork_kind:b.forkKind,
        shared_prefix:b.sharedPrefix,
        fork_points:b.forkPoints,
        incomplete_ancestry:b.detachedRoot,
        truncated_tail:b.truncatedTail
      })),
      forks:tree.forks.slice()
        .sort((a,b)=>(tree.depth.get(a)??0)-(tree.depth.get(b)??0))
        .map(id=>({
          message_id:id,
          role:treeRole(nodes.get(id)) || 'unknown',
          depth:tree.depth.get(id) ?? null,
          created_at:isoOrBlank(nodeTime(nodes.get(id))),
          alternatives:(tree.children.get(id)||[]).map((c,k)=>({
            choice:k+1,
            message_id:c,
            role:treeRole(nodes.get(c)) || 'unknown',
            kind:altKind(nodes.get(c)),
            created_at:isoOrBlank(nodeTime(nodes.get(c))),
            branches:tree.nodeBranches.get(c)||[]
          }))
        })),
      nodes:nodeList,
      warnings:tree.warnings.concat(fetched.notes)
    };
  }

  // Minimal store-only ZIP writer (no compression; UTF-8 names).
  function makeZip(entries){
    const enc=new TextEncoder();
    const now=new Date();
    const dosTime=((now.getHours()<<11)|(now.getMinutes()<<5)|(now.getSeconds()>>1))&0xFFFF;
    const dosDate=(((now.getFullYear()-1980)<<9)|((now.getMonth()+1)<<5)|now.getDate())&0xFFFF;
    const chunks=[], central=[];
    let offset=0;
    const u16=n=>[n&255,(n>>>8)&255];
    const u32=n=>[n&255,(n>>>8)&255,(n>>>16)&255,(n>>>24)&255];
    for(const e of entries){
      const name=enc.encode(e.name);
      const data=typeof e.data==='string' ? enc.encode(e.data) : e.data;
      const crc=crc32(data);
      const local=new Uint8Array([
        ...u32(0x04034b50), ...u16(20), ...u16(0x0800), ...u16(0), ...u16(dosTime), ...u16(dosDate),
        ...u32(crc), ...u32(data.length), ...u32(data.length), ...u16(name.length), ...u16(0)
      ]);
      chunks.push(local, name, data);
      central.push(new Uint8Array([
        ...u32(0x02014b50), ...u16(20), ...u16(20), ...u16(0x0800), ...u16(0), ...u16(dosTime), ...u16(dosDate),
        ...u32(crc), ...u32(data.length), ...u32(data.length), ...u16(name.length), ...u16(0), ...u16(0),
        ...u16(0), ...u16(0), ...u32(0), ...u32(offset)
      ]), name);
      offset+=local.length+name.length+data.length;
    }
    if(entries.length>65535 || offset>0xFFFFFFFF) throw new Error('Export too large for a single ZIP file.');
    const cdSize=central.reduce((n,c)=>n+c.length,0);
    const end=new Uint8Array([
      ...u32(0x06054b50), ...u16(0), ...u16(0), ...u16(entries.length), ...u16(entries.length),
      ...u32(cdSize), ...u32(offset), ...u16(0)
    ]);
    // A Blob avoids copying everything into one giant array (helps mobile).
    return new Blob(chunks.concat(central,[end]), {type:'application/zip'});
  }

  // ---------- Export all branches (current chat) ----------
  async function buildBranchExport(onStep){
    const route=chatRouteIds();
    if(!route) return {ok:false, error:'Open a specific CrushOn conversation (…/character/<id>/chat/<conversation id>) first.'};
    setPace(400, 200);
    PACE.cancelled=false;
    const fetched=await fetchConversationTree(route.conversationId, onStep);
    if(!fetched.ok) return {ok:false, error:fetched.error};
    if(!fetched.nodes.size) return {ok:false, error:'CrushOn returned an empty message tree for this conversation.'};

    onStep && onStep('analysing branches…');
    const pre=analyzeTree(fetched.nodes);
    if(!pre.branches.length) return {ok:false, error:'No branch could be built from the returned messages.'};
    const ctxBase={
      route,
      aiName:exportAIName(),
      exportedAt:new Date().toISOString(),
      sourceUrl:`${location.origin}/character/${encodeURIComponent(route.characterId)}/chat/${encodeURIComponent(route.conversationId)}`,
      renderedLeafId:renderedLeafFor(fetched.nodes, pre)
    };
    const {tree, files, ctx}=buildChatBranchFiles(fetched, ctxBase);
    const manifest=buildBranchManifest(ctx, files);
    return {ok:true, ctx, tree, fetched, files, manifest, base:safeFilePart(ctx.aiName,'character')};
  }

  function noFormatPicked(){
    if(['jsonl','txt','png'].some(k=>EXPORT_FORMATS[k])) return false;
    showPrompt('Pick at least one format — PNG, JSONL, or TXT — then try again.', {okText:'ok', cancel:false, onOK:hidePrompt});
    setStatus('no format selected','warn');
    return true;
  }

  async function exportAllBranches(mode){
    closeMenus();
    if(noFormatPicked()) return false;
    showPrompt('Reading the full conversation tree…', {ok:false, cancel:false});
    setStatus('reading conversation tree…','muted');
    let result;
    try{
      result=await buildBranchExport(s=>{ updatePrompt(s,{ok:false,cancel:false}); setStatus(s,'muted'); });
    }catch(e){
      result={ok:false, error:String(e?.message||e)};
    }
    if(!result.ok){
      setStatus('branch export unavailable','warn');
      updatePrompt('Export all branches is unavailable:\n'+result.error+'\n\nNothing was downloaded. The regular ↓ export still works.',
        {okText:'ok', cancel:false, onOK:hidePrompt});
      return false;
    }

    const {files, manifest, base, tree, ctx}=result;

    // Character card for this chat, when PNG is on.
    let card=null, cardNote='';
    if(EXPORT_FORMATS.png){
      updatePrompt('Reading the character profile…', {ok:false, cancel:false});
      CHAR=null;
      await captureCharacter(s=>updatePrompt('Reading the character profile… ('+s+')', {ok:false, cancel:false}));
      if(CHAR && CHAR.gate.allowed && CHAR.q.ok){
        try{
          const built=await characterPngBytes({c:CHAR.c, id:CHAR.id, source:CHAR.source}, ctx.aiName, true);
          card={name:`${base}.png`, data:built.bytes};
          manifest.card={file:card.name, ok:true, included:CHAR.q.filled};
        }catch(e){ cardNote='PNG failed: '+(e?.message||e); }
      }else{
        cardNote='PNG skipped — '+(CHAR ? (CHAR.gate.allowed ? CHAR.q.why : CHAR.gate.why) : 'no character data on this page');
      }
      if(!card) manifest.card={file:null, ok:false, why:cardNote};
    }

    const outputs=[];
    for(const f of files){
      if(EXPORT_FORMATS.jsonl) outputs.push({name:f.name, data:f.data, mime:'application/x-ndjson;charset=utf-8'});
      if(EXPORT_FORMATS.txt) outputs.push({name:f.txtName, data:f.txt, mime:'text/plain;charset=utf-8'});
    }
    if(card) outputs.push({name:card.name, data:card.data, mime:'image/png'});

    const stamp=ts();
    const manifestText=JSON.stringify(manifest, null, 2);
    const wasMulti=MULTI_DOWNLOAD;
    try{
      if(mode==='separate'){
        MULTI_DOWNLOAD=true;
        download(`${base} - ${safeFilePart(ctx.route.conversationId,'chat')} - manifest.json`, manifestText, 'application/json;charset=utf-8');
        for(const o of outputs){
          await sleep(350); // browsers throttle bursts of downloads
          download(o.name, o.data, o.mime);
        }
      }else{
        const folder=`${base} - ${safeFilePart(ctx.route.conversationId,'chat')}/`;
        const zip=makeZip([{name:folder+'manifest.json', data:manifestText}]
          .concat(outputs.map(o=>({name:folder+o.name, data:o.data}))));
        download(`${base} - ${safeFilePart(ctx.route.conversationId,'chat')} - branches - ${stamp}.zip`, zip, 'application/zip');
      }
    }finally{
      setTimeout(()=>{ MULTI_DOWNLOAD=wasMulti; }, 0);
    }

    const kinds=[EXPORT_FORMATS.jsonl && '.jsonl', EXPORT_FORMATS.txt && '.txt'].filter(Boolean).join(' + ');
    const saved=(kinds ? `a ${kinds} per branch` : 'no chat files')+(card ? ', the character .png' : '')+' + manifest.json';
    const lines=[
      `Exported ${tree.branches.length} branch${tree.branches.length===1?'':'es'} (${manifest.counts.nodes} messages in the tree, ${tree.forks.length} fork point${tree.forks.length===1?'':'s'}).`,
      mode==='separate' ? `Saved ${saved}, as separate files.` : `Saved one .zip: ${saved}.`,
      tree.foldedRegens ? `${tree.foldedRegens} regenerated repl${tree.foldedRegens===1?'y':'ies'} kept as swipes, not branches (⚙ → Regenerations as branches).` : '',
      cardNote,
      manifest.capture.complete ? 'Coverage: CrushOn returned the complete tree.' : 'Coverage: INCOMPLETE. See the warnings below.',
      manifest.rendered_branch ? `The branch shown on this page is ${manifest.rendered_branch}.` : 'Could not tell which branch this page is showing.',
      ''
    ].filter((x,i,a)=>x!=='' || i===a.length-1);
    tree.branches.forEach((b,i)=>{
      const sp=b.sharedPrefix ? `, splits from ${b.sharedPrefix.branch_id} at message ${b.sharedPrefix.jsonl_messages+1} (${b.forkKind})` : '';
      lines.push(`• ${b.id}: ${files[i].messages} msgs${sp}${tree.newestLeaf===b.leaf?' (newest)':''}`);
    });
    if(manifest.warnings.length){
      lines.push('', 'Warnings:');
      manifest.warnings.slice(0,6).forEach(w=>lines.push('– '+w));
      if(manifest.warnings.length>6) lines.push(`– …and ${manifest.warnings.length-6} more in manifest.json`);
    }
    updatePrompt(lines.join('\n'), {okText:'ok', cancel:false, onOK:hidePrompt});
    setStatus(`exported ${tree.branches.length} branches`,'ok');
    return true;
  }

  // ---------- Dump history response (diagnostics) ----------
  async function dumpHistoryResponse(){
    closeMenus();
    const route=chatRouteIds();
    if(!route){
      showPrompt('Open a specific conversation first.', {okText:'ok', cancel:false, onOK:hidePrompt});
      return;
    }
    showPrompt('Reading raw history response…', {ok:false, cancel:false});
    setPace(400, 200);
    PACE.cancelled=false;
    const fetched=await fetchConversationTree(route.conversationId, s=>updatePrompt(s,{ok:false,cancel:false}), true);
    const out={
      tool:'CrushOn Exporter PLUS: history dump',
      exporter_version:SCRIPT_VERSION,
      exported_at:new Date().toISOString(),
      source_url:location.href,
      procedure:TREE_PROC,
      ok:fetched.ok,
      error:fetched.error ?? null,
      complete:fetched.complete ?? false,
      notes:fetched.notes ?? [],
      pages:fetched.raw
    };
    download(`crushon-history-dump - ${safeFilePart(route.conversationId,'chat')} - ${ts()}.json`, JSON.stringify(out,null,2), 'application/json;charset=utf-8');
    updatePrompt(
      (fetched.ok ? `Saved the raw response (${fetched.pages} page${fetched.pages===1?'':'s'}, ${fetched.nodes.size} messages).` : 'The history request failed; the dump records the error.') +
      '\nIt contains your full message text. Handle it like the chat itself.',
      {okText:'ok', cancel:false, onOK:hidePrompt});
  }

  // ---------- Export ALL chats (every character, every chat, every branch) ----------
  async function listAllChats(onStep){
    const rc=await trpcCall('character.queryRecentCharacters', {nsfw:true, locale:(document.documentElement.lang||'en').slice(0,2)||'en'}, 'GET');
    if(!rc.ok) return {ok:false, error:`character.queryRecentCharacters unavailable (${rc.error})`};
    const chars=Array.isArray(rc.data?.characters) ? rc.data.characters : null;
    if(!chars) return {ok:false, error:'character.queryRecentCharacters returned no character list.'};
    const chats=[], problems=[];
    for(let i=0;i<chars.length;i++){
      if(PACE.cancelled) return {ok:false, error:'cancelled', cancelled:true};
      const c=chars[i];
      const cid=String(c?.id||'');
      if(!cid) continue;
      onStep && onStep(`listing chats: character ${i+1} of ${chars.length}…`);
      const lc=await trpcCall('chat.listConversations', {characterId:cid}, 'GET');
      if(!lc.ok){ problems.push({character_id:cid, character_name:c.name||'', error:lc.error}); if(lc.cancelled) break; continue; }
      const infos=Array.isArray(lc.data?.conversationInfos) ? lc.data.conversationInfos : [];
      for(const info of infos){
        const conv=info?.conversation;
        if(conv?.id) chats.push({characterId:cid, characterName:cleanAiNameExact(c.name||'') , conversationId:String(conv.id), chatName:conv.name||''});
      }
      if(typeof c.conversationCount==='number' && c.conversationCount!==infos.length){
        problems.push({character_id:cid, character_name:c.name||'', note:`character says ${c.conversationCount} chats; list returned ${infos.length}`});
      }
    }
    return {ok:true, characters:chars.length, chats, problems};
  }

  async function exportAllChats(){
    closeMenus();
    if(noFormatPicked()) return false;
    PACE.cancelled=false;
    setPace(1500, 1000); // ≈1.5–2.5 s between requests, one at a time
    const cancelBtn={cancel:true, cancelText:'Stop', ok:false, onCancel:()=>{ PACE.cancelled=true; updatePrompt('Stopping after the current request…',{ok:false,cancel:false}); }};
    showPrompt('Listing your chats…', cancelBtn);
    setStatus('bulk: listing chats…','muted');

    const list=await listAllChats(s=>{ updatePrompt(s, cancelBtn); setStatus('bulk: '+s,'muted'); });
    if(!list.ok){
      setStatus('bulk export stopped','warn');
      updatePrompt(list.cancelled ? 'Stopped. Nothing was downloaded.' : 'Could not list your chats:\n'+list.error+'\n\nNothing was downloaded.', {okText:'ok', cancel:false, onOK:hidePrompt});
      return false;
    }
    const mins=Math.max(1, Math.round(list.chats.length*2.2/60));
    const go=await new Promise(resolve=>showPrompt(
      `Found ${list.chats.length} chats across ${list.characters} characters.\n\n`+
      `Downloading every branch of every chat takes about ${mins} minute${mins===1?'':'s'} (one request every ~2 s, so CrushOn isn't flooded). `+
      (EXPORT_FORMATS.png ? 'PNG is on, so each character\'s card is looked up too, which adds a little time. ' : '')+
      'Keep this tab open. You can stop at any time and keep what has been collected.\n\nStart?',
      {okText:'Start', cancelText:'Cancel', onOK:()=>resolve(true), onCancel:()=>resolve(false)}));
    if(!go){ hidePrompt(); setStatus('bulk export cancelled','warn'); return false; }

    const exportedAt=new Date().toISOString();
    const root=`crushon-all-chats - ${ts()}/`;
    const entries=[], manifestFiles=[], chatResults=[], cards=[];
    const charFolder=new Map();
    let done=0, foldedRegens=0;
    for(const chat of list.chats){
      if(PACE.cancelled) break;
      done++;
      const label=`chat ${done} of ${list.chats.length} (${chat.characterName})`;
      updatePrompt(`Exporting ${label}…\n${manifestFiles.length} branch files so far.`, cancelBtn);
      setStatus('bulk: '+label,'muted');
      const fetched=await fetchConversationTree(chat.conversationId);
      if(!fetched.ok){
        chatResults.push({character_id:chat.characterId, character_name:chat.characterName, chat_id:chat.conversationId, ok:false, error:fetched.error});
        if(fetched.cancelled) break;
        continue;
      }
      if(!fetched.nodes.size){
        chatResults.push({character_id:chat.characterId, character_name:chat.characterName, chat_id:chat.conversationId, ok:true, branches:0, note:'empty conversation'});
        continue;
      }
      if(!charFolder.has(chat.characterId)){
        charFolder.set(chat.characterId, `${safeFilePart(chat.characterName,'character')} (${safeFilePart(chat.characterId.slice(0,8),'id')})/`);
        if(EXPORT_FORMATS.png && !PACE.cancelled){
          updatePrompt(`Exporting ${label}…\nReading ${chat.characterName}'s character card…`, cancelBtn);
          const cardName=`${safeFilePart(chat.characterName,'character')}.png`;
          let r;
          try{ r=await characterPngById(chat.characterId, chat.characterName); }
          catch(e){ r={ok:false, why:String(e?.message||e)}; }
          if(r.ok) entries.push({name:root+charFolder.get(chat.characterId)+cardName, data:r.bytes});
          cards.push({character_id:chat.characterId, character_name:chat.characterName,
            file:r.ok ? charFolder.get(chat.characterId)+cardName : null, ok:r.ok,
            ...(r.ok ? {included:r.filled} : {why:r.why})});
        }
      }
      const folder=root+charFolder.get(chat.characterId);
      const route={characterId:chat.characterId, conversationId:chat.conversationId};
      const {tree, files, ctx}=buildChatBranchFiles(fetched, {
        route, aiName:chat.characterName || 'AI', chatName:chat.chatName, exportedAt,
        sourceUrl:`${location.origin}/character/${encodeURIComponent(chat.characterId)}/chat/${encodeURIComponent(chat.conversationId)}`,
        renderedLeafId:null
      });
      for(const f of files){
        if(EXPORT_FORMATS.jsonl) entries.push({name:folder+f.name, data:f.data});
        if(EXPORT_FORMATS.txt) entries.push({name:folder+f.txtName, data:f.txt});
      }
      foldedRegens+=tree.foldedRegens;
      manifestFiles.push(...chatManifestEntries(files, ctx, charFolder.get(chat.characterId)));
      chatResults.push({
        character_id:chat.characterId, character_name:chat.characterName, chat_id:chat.conversationId,
        ...(chat.chatName ? {chat_name:chat.chatName} : {}),
        ok:true, messages:fetched.nodes.size, memory_messages:fetched.memory.length, branches:tree.branches.length, fork_points:tree.forks.length,
        folded_regens:tree.foldedRegens,
        pages_fetched:fetched.pages, stopped_early:!!fetched.stoppedEarly,
        complete:fetched.complete && !tree.detached.length && !tree.missingChildren.size,
        warnings:tree.warnings.concat(fetched.notes)
      });
      fetched.nodes.clear(); // free memory as we go
    }

    const stopped=PACE.cancelled;
    PACE.cancelled=false;
    const failed=chatResults.filter(r=>!r.ok);
    const incomplete=chatResults.filter(r=>r.ok && r.complete===false);
    const manifest={
      format:'crushon-exporter-plus/bulk-export',
      format_version:1,
      exporter_version:SCRIPT_VERSION,
      exported_at:exportedAt,
      finished:!stopped,
      note:stopped ? 'Stopped by the user before every chat was exported; chats not listed under "chats" were not attempted.' : 'Every listed chat was attempted.',
      listing:{source:'character.queryRecentCharacters + chat.listConversations', characters:list.characters, chats_listed:list.chats.length, problems:list.problems},
      options:{...exportOptions(null), folded_regens:foldedRegens},
      totals:{chats_attempted:chatResults.length, chats_failed:failed.length, chats_incomplete:incomplete.length, files:manifestFiles.length,
        ...(EXPORT_FORMATS.png ? {cards:cards.filter(c=>c.ok).length} : {})},
      files:manifestFiles,
      ...(EXPORT_FORMATS.png ? {cards} : {}),
      chats:chatResults
    };
    if(!manifestFiles.length){
      setStatus('bulk export: nothing collected','warn');
      updatePrompt('Nothing was collected, so nothing was downloaded.'+(failed.length?`\n${failed.length} chat(s) failed, first error: ${failed[0].error}`:''), {okText:'ok', cancel:false, onOK:hidePrompt});
      return false;
    }
    updatePrompt('Packing the .zip…', {ok:false, cancel:false});
    await sleep(50);
    let zip;
    try{
      zip=makeZip([{name:root+'manifest.json', data:JSON.stringify(manifest,null,2)}].concat(entries));
    }catch(e){
      updatePrompt('Could not build the .zip: '+(e?.message||e), {okText:'ok', cancel:false, onOK:hidePrompt});
      return false;
    }
    download(root.slice(0,-1)+'.zip', zip, 'application/zip');
    const lines=[
      stopped ? `Stopped early. Saved what was collected (${chatResults.length} of ${list.chats.length} chats attempted).` : `Attempted all ${list.chats.length} chats.`,
      foldedRegens ? `${foldedRegens} regenerated repl${foldedRegens===1?'y':'ies'} kept as swipes, not branches.` : '',
      EXPORT_FORMATS.png ? `${cards.filter(c=>c.ok).length} of ${cards.length} character cards saved${cards.some(c=>!c.ok) ? '; see manifest.json → cards for the rest' : ''}.` : '',
      `${manifestFiles.length} branches across ${charFolder.size} characters, ${zip.size>=1048576 ? (Math.round(zip.size/1048576*10)/10)+' MB' : Math.max(1,Math.round(zip.size/1024))+' KB'}.`,
      incomplete.length ? `${incomplete.length} chat(s) came back incomplete; see manifest.json → chats.` : 'Every exported chat came back complete.',
      failed.length ? `${failed.length} chat(s) failed; see manifest.json → chats.` : '',
      list.problems.length ? `${list.problems.length} listing note(s) in manifest.json → listing.problems.` : ''
    ].filter(Boolean);
    updatePrompt(lines.join('\n'), {okText:'ok', cancel:false, onOK:hidePrompt});
    setStatus(`bulk export: ${manifestFiles.length} files`,'ok');
    return true;
  }

  // ---------- regular ↓ export: the rendered branch from the tree ----------
  // Returns {rows, note, meta} or {rows:null, note}. Used only when the visible
  // branch can be identified with confidence.
  async function captureRenderedBranchFromTree(){
    const route=chatRouteIds();
    if(!route) return {rows:null, note:''};
    setPace(400, 200);
    PACE.cancelled=false;
    const fetched=await fetchConversationTree(route.conversationId);
    if(!fetched.ok || !fetched.nodes.size) return {rows:null, note:''};
    // Find what the page shows in the full tree (a regeneration may be on
    // screen), but name branches as the branch exports do.
    const full=analyzeTree(fetched.nodes, {includeRegens:true});
    const tree=analyzeTree(fetched.nodes);
    let leaf=null, why='';
    if(full.branches.length===1){ leaf=full.branches[0].leaf; why='single-branch conversation'; }
    else{
      leaf=renderedLeaf(fetched.nodes, full.branches);
      why='branch matched to the messages rendered in the page';
    }
    if(!leaf){
      return {rows:null, note:`This conversation has ${tree.branches.length} branches and the visible one could not be identified; switch ONE → ALL and press ↓ for the full tree.`};
    }
    const shown=full.branches.find(x=>x.leaf===leaf);
    const b=tree.branches.find(x=>x.leaf===leaf) || closestBranch(tree, shown.path);
    const shownRegen=b.leaf!==leaf;
    const rows=[];
    for(const {id, m} of memoryRows(fetched)){
      const who=treeRole(m);
      if(who!=='USER' && who!=='AI') continue;
      const text=String(nodeText(m)||'');
      const t=nodeTime(m);
      rows.push({id, seq:rows.length, who, raw:text, html:escapeHtml(text).replace(/\r?\n/g,'<br>'), t,
        crushon:{message_id:id||null, parent_message_id:null, created_at:isoOrBlank(t), time_source:t!==null?'server':'none', edited:m.isEdited===true, memory:true}});
    }
    for(const id of shown.path){
      const m=fetched.nodes.get(id);
      const who=treeRole(m);
      if(who!=='USER' && who!=='AI') continue;
      if(isSkippedNode(m)) continue;
      const text=String(nodeText(m)||'');
      rows.push({
        id, seq:rows.length, who, raw:text,
        html:escapeHtml(text).replace(/\r?\n/g,'<br>'),
        t:nodeTime(m),
        crushon:crushonMessageExtra(m, id, tree),
        swipe:swipeFields(m, id, tree)
      });
    }
    const complete=fetched.complete && !b.detachedRoot && !b.truncatedTail && !tree.missingChildren.size;
    const branchNote=(tree.branches.length>1
      ? ` This is ${b.id} of ${tree.branches.length} (${why}); switch ONE → ALL and press ↓ to save the others.`
      : '')+(shownRegen ? ` The page shows a regenerated reply; in branch exports it is a swipe on ${b.id}.` : '');
    return {
      rows,
      meta:{
        capture:'message_tree', complete, pages_fetched:fetched.pages, stopped_early:!!fetched.stoppedEarly,
        branch_id:b.id, branch_count:tree.branches.length,
        parent_branch_id:b.sharedPrefix?.branch_id ?? null,
        fork_index:forkIndexOf(b, fetched),
        fork_kind:b.forkKind, leaf_message_id:leaf, memory_messages:memoryRows(fetched).length,
        regens_as_branches:tree.includeRegens, skip_continue:SKIP_CONTINUE,
        ...(shownRegen ? {shown_regeneration:true} : {})
      },
      note:(complete ? 'Complete history for this branch from CrushOn\'s message tree.' : 'CrushOn\'s message tree was incomplete; older messages may be missing.')+branchNote
    };
  }

  // ---------- capture ----------
  // Returns true when chat capture completed, false when capture completed but
  // found no messages, and null when the user canceled manual calibration.
  async function captureNow(afterCapture, opts={}){
    // Keep an explicitly calibrated role signature for this chat. Clearing
    // exported records must not silently invalidate manual calibration.
    clearCaptured(true, false);
    if(SPEAKER_SIG?.scope && SPEAKER_SIG.scope!==location.pathname){
      clearCalibration('different chat route');
    }

    // Replays only an existing, same-conversation authenticated history
    // request seen in this tab. Never fabricate an API endpoint or ID.
    setStatus('checking accessible conversation history…','muted');
    // 1) CrushOn's own message tree (chat.getMessageHistory), when the visible
    //    branch can be identified with confidence.
    let treeNote='';
    LAST_TREE_NOTE='';
    CAPTURE_META={capture:'none', complete:false, pages_fetched:0, stopped_early:false};
    try{
      const fromTree=await captureRenderedBranchFromTree();
      treeNote=fromTree.note||'';
      LAST_TREE_NOTE=treeNote;
      if(fromTree.rows?.length){
        for(const it of fromTree.rows) records.push(it);
        CAPTURE_SOURCE='CrushOn history API (message tree)';
        CAPTURE_NOTE=treeNote;
        CAPTURE_META=fromTree.meta;
        const u=fromTree.rows.filter(x=>x.who==='USER').length;
        setCounts(`${u}U/${fromTree.rows.length-u}A`);
        await finishCapture(afterCapture, opts);
        return true;
      }
    }catch(e){ console.warn('[CrushOn Exporter] message tree unavailable:',e); }

    // 2) A matching history request already observed in this tab.
    try{
      const history=await captureHistoryFromObservedRequest();
      if(history?.length){
        for(const it of history) records.push(it);
        CAPTURE_SOURCE='CrushOn history API';
        CAPTURE_NOTE='History API response captured. Older pages are included only when the observed endpoint exposes supported pagination; full-history coverage cannot otherwise be guaranteed.';
        CAPTURE_META=history.meta || {capture:'observed_request', complete:false, pages_fetched:0, stopped_early:false};
        if(treeNote) CAPTURE_NOTE+=' '+treeNote;
        const u=history.filter(x=>x.who==='USER').length;
        setCounts(`${u}U/${history.length-u}A`);
        await finishCapture(afterCapture, opts);
        return true;
      }
    }catch(e){ console.warn('[CrushOn Exporter] history request unavailable:',e); }

    if(!getGroups().length){
      CAPTURE_SOURCE='none';
      CAPTURE_NOTE='No rendered chat messages or matching history request were found.';
      return false;
    }
    showPrompt('Attempting auto-calibration…',{cancel:false, ok:false});
    await sleep(100);

    const auto=(SPEAKER_SIG?.manual && SPEAKER_SIG.scope===location.pathname)
      ? {sig:SPEAKER_SIG} : autoCalibrateTry();

    if(auto){
      SPEAKER_SIG=auto.sig;
      try{ coSetValue('co_speaker_sig', SPEAKER_SIG); }catch{}

      setAIName(auto.sig.aiName||'AI');

      updatePrompt(
        `Auto-calibration succeeded as [${currentAIName()}]. Capturing…`,
        {cancel:false, ok:false}
      );

      await sleep(60);
      await doCaptureAfterCalibrated();
      await finishCapture(afterCapture, opts);
      return records.length>0;
    }

    try{
      await promptOKCancel(
        'Auto failed.\nCalibrate: Select YOUR bubble.\nPress ok to start.',
        'ok',
        'Cancel'
      );

      const userNode=await pickOneRow();

      await promptOKCancel(
        'Calibrate: Select AI bubble.\nPress ok to start.',
        'ok',
        'Cancel'
      );

      const aiNode=await pickOneRow();

      const u=featureFromRow(userNode);
      const a=featureFromRow(aiNode);
      const aiName=cleanAiNameExact(extractAiNameFromRow(aiNode));

      // IMPORTANT: wait for the confirmation callback itself. The previous
      // v2.1.0 implementation returned from captureNow() as soon as this prompt
      // was displayed, so the unified ↓ flow believed capture had been canceled
      // before the user could finish calibration.
      return await new Promise(resolve=>{
        showPrompt(`Confirm calibration:\nUser = USER\nAI = ${aiName}`,{
          okText:'ok',
          extra:true,
          extraText:'Start over',

          onOK: async ()=>{
            try{
              SPEAKER_SIG={user:u, ai:a, aiName, manual:true, scope:location.pathname};
              try{ coSetValue('co_speaker_sig', SPEAKER_SIG); }catch{}

              setAIName(aiName);

              updatePrompt('Capturing…',{cancel:false, ok:false});
              await doCaptureAfterCalibrated();
              await finishCapture(afterCapture, opts);
              resolve(records.length>0);
            }catch(err){
              setStatus('capture failed','warn');
              showPrompt('Capture failed: ' + (err?.message || err),
                {okText:'ok', cancel:false, onOK:hidePrompt});
              resolve(false);
            }
          },

          onExtra: async ()=>{
            hidePrompt();
            resolve(await captureNow(afterCapture, opts));
          },

          onCancel: ()=>{
            cancelAndClear();
            resolve(null);
          }
        });
      });
    }catch{
      cancelAndClear();
      return null;
    }
  }

  function fixSkew(items){
    // Re-evaluate all rows, not only chats that looked nearly all USER.
    // A manual or auto-calibration signature must actually determine roles.
    for(const it of items){
      it.who = decideFromFeatures(it);
    }
  }

  function decideFromFeatures(f){
    if(SPEAKER_SIG?.user && SPEAKER_SIG?.ai){
      const u=SPEAKER_SIG.user, a=SPEAKER_SIG.ai;
      // The side is the strongest discriminator when calibration found
      // distinctly opposed chat bubbles (including bubbles without links).
      if(typeof u.right==='boolean' && typeof a.right==='boolean' && u.right!==a.right){
        return f.right===u.right ? 'USER' : 'AI';
      }
      const us=similarity(f,u), as=similarity(f,a);
      if(us!==as) return us>as ? 'USER' : 'AI';
    }
    return f.charLink ? 'AI' : (f.right ? 'USER' : 'AI');
  }

  async function doCaptureAfterCalibrated(){
    const items=collectFromDOM();
    CAPTURE_SOURCE='visible DOM';
    CAPTURE_META={capture:'visible_dom', complete:false, pages_fetched:0, stopped_early:false};
    CAPTURE_NOTE='Only currently rendered messages are available; older messages may be missing.'+(LAST_TREE_NOTE?' '+LAST_TREE_NOTE:'');
    fixSkew(items);

    let u=0,a=0;

    for(const it of items){
      if(it.who==='USER') u++;
      else a++;
    }

    for(const it of items){
      // Repeated identical messages are valid separate messages.
      const key=it.id ? 'id:'+it.id : 'seq:'+it.seq;
      if(seen.has(key)) continue;

      seen.add(key);
      records.push(it);
    }

    setCounts(`${u}U/${a}A`);

    setStatus(
      `calibrated as [${currentAIName()}] and captured ${items.length} items`,
      'ok'
    );
  }

  // ---------- speaker roles + authentic DOM timestamps ----------
  function decideWho(g){ return decideFromFeatures(featureFromRow(g)); }

  function parseMessageTime(value){
    if(value===null || value===undefined || value==='') return null;
    if(typeof value==='object' && typeof value.toDate==='function') return parseMessageTime(value.toDate());
    if(value instanceof Date) return Number.isFinite(value.getTime()) ? value.getTime() : null;
    if(typeof value==='number' || /^\d{10,13}$/.test(String(value))){
      const n=Number(value), ms=n<1e11 ? n*1000 : n;
      return Number.isFinite(ms) && ms>=946684800000 && ms<=4102444800000 ? ms : null;
    }
    const ms=Date.parse(String(value));
    return Number.isFinite(ms) ? ms : null;
  }

  function domMessageTime(row){
    const nodes=[row,...row.querySelectorAll('time[datetime], [data-timestamp], [data-created-at], [data-sent-at]')];
    for(const node of nodes){
      for(const attr of ['datetime','data-timestamp','data-created-at','data-sent-at']){
        const time=parseMessageTime(node.getAttribute(attr));
        if(time!==null) return time;
      }
    }
    return null; // Never invent a send time from the moment of export.
  }

  function orderedRecords(){
    // Applies to the observed-request and on-screen captures too.
    const items=SKIP_CONTINUE
      ? records.filter(it=>!(it.who==='USER' && isContinueText(messageText(it))))
      : records.slice();
    // Message-tree captures are already in parent→child order; keep it.
    if(CAPTURE_META.capture==='message_tree') return items;
    // Only timestamp-sort when *all* entries have authentic timestamps.
    // Otherwise preserve server order / message order in the DOM.
    if(items.length && items.every(it=>it.t!==null && Number.isFinite(it.t))){
      items.sort((a,b)=>a.t-b.t || (a.seq||0)-(b.seq||0));
    }
    return items;
  }

  // CrushOn renders an "(edited)" tag inside the message block. It is UI, not
  // message text: remove it and report edited:true instead.
  const EDITED_LABEL_RE=/^\s*\(?\s*edited\s*\)?\s*$/i;
  function stripEditedLabel(root){
    let edited=false;
    for(const n of [...root.querySelectorAll('span,small,div,i,em')]){
      if(n.children.length===0 && EDITED_LABEL_RE.test(n.textContent||'')){ n.remove(); edited=true; }
    }
    return edited;
  }

  // Flatten rendered markdown back to text, restoring *em* / **strong** so a
  // DOM capture reads like CrushOn's raw text instead of losing asterisks.
  function domToText(root){
    const BLOCK=/^(p|div|li|blockquote|pre|h[1-6]|tr|ul|ol|table|section|article)$/;
    let out='';
    const walk=n=>{
      if(n.nodeType===3){ out+=n.nodeValue; return; }
      if(n.nodeType!==1) return;
      const tag=n.localName;
      if(tag==='script' || tag==='style' || tag==='button' || tag==='svg') return;
      if(tag==='br'){ out+='\n'; return; }
      const block=BLOCK.test(tag);
      if(block && out && !out.endsWith('\n')) out+='\n';
      const mark=(tag==='strong'||tag==='b') ? '**' : (tag==='em'||tag==='i') ? '*' : (tag==='del'||tag==='s') ? '~~' : '';
      if(tag==='li') out+='- ';
      if(mark) out+=mark;
      n.childNodes.forEach(walk);
      if(mark) out+=mark;
      if(block && !out.endsWith('\n')) out+='\n';
    };
    walk(root);
    // Paragraphs in CrushOn are separated by a blank line in the raw text.
    return out.replace(/\r/g,'').replace(/[ \t]+\n/g,'\n').replace(/\n{3,}/g,'\n\n').trim();
  }

  function collectFromDOM(){
    const groups=getGroups();
    const out=[];

    for(const g of groups){
      const body=getBlock(g);
      const clone=body.cloneNode(true);
      const edited=stripEditedLabel(clone);
      const html=(clone.innerHTML||'').trim();
      if(!html) continue;
      const f=featureFromRow(g);
      out.push({
        t:domMessageTime(g),
        seq:out.length,
        who:decideWho(g),
        html,
        text:domToText(clone),
        crushon:{edited},
        right:f.right,
        charLink:f.charLink,
        nameChip:f.nameChip
      });
    }
    return out;
  }

  // ---------- clear ----------
  function clearCaptured(silent=false, resetCalibration=true){
    records.length=0;
    seen.clear();
    CHAR=null;
    CAPTURE_SOURCE='none';
    CAPTURE_NOTE='';
    setCardBadge();

    const msgs=$('#coMsgs');
    if(msgs) msgs.textContent='0';

    const groups=$('#coGroups');
    if(groups) groups.textContent='0';

    const split=$('#coSplit');
    if(split) split.textContent='0U/0A';

    if(resetCalibration) clearCalibration('clearCaptured');

    if(!silent){
      showPrompt(
        'Captured set and calibration cleared.',
        {cancel:false, okText:'ok', onOK:hidePrompt}
      );
    }
  }

  // ---------- download/export ----------
  // Batch mode suppresses per-export prompts so ↓ can show one summary.
  // MULTI_DOWNLOAD separately suppresses the mobile share sheet only when more
  // than one file will be emitted. A one-format ↓ export can still use Share.
  let LITE_BATCH=false;
  let MULTI_DOWNLOAD=false;

  const download=(name,data,mime)=>{
    const type = mime || 'text/plain;charset=utf-8';
    const blob = new Blob([data], {type: type});

    const tryShare = async ()=>{
      if(MULTI_DOWNLOAD) return false;

      try{
        if(navigator.canShare && typeof File!=='undefined'){
          const f=new File([blob], name, {type});

          if(navigator.canShare({files:[f]})){
            await navigator.share({files:[f], title:name});
            return true;
          }
        }
      }catch(_){ }

      return false;
    };

    const tryAnchor = ()=>{
      try{
        const url=URL.createObjectURL(blob);
        const a=document.createElement('a');

        a.href=url;
        a.download=name;
        a.rel='noopener';
        a.target='_self';

        document.body.appendChild(a);
        a.click();

        setTimeout(()=>{
          URL.revokeObjectURL(url);
          a.remove();
        }, 1500);

        return true;
      }catch(_){
        return false;
      }
    };

    const tryGM = ()=>{
      if(typeof GM_download==='function'){
        try{
          const url = (typeof data==='string')
            ? ('data:'+type+','+encodeURIComponent(data))
            : URL.createObjectURL(blob);

          GM_download({
            url,
            name,
            saveAs:true
          });

          return true;
        }catch(_){
          return false;
        }
      }

      return false;
    };

    const tryMs = ()=>{
      try{
        if('msSaveOrOpenBlob' in navigator){
          navigator.msSaveOrOpenBlob(blob, name);
          return true;
        }
      }catch(_){}

      return false;
    };

    (async()=>{
      if(await tryShare()) return;
      if(tryAnchor()) return;
      if(tryGM()) return;
      if(tryMs()) return;

      const url=URL.createObjectURL(blob);
      window.open(url,'_blank');

      setTimeout(
        ()=>URL.revokeObjectURL(url),
        2000
      );
    })();
  };

  // Unified capture + export triggered by the ↓ button.
  async function exportByScope(){
    if(EXPORT_SCOPE!=='all') return captureAndExport();
    const go = $('#coGo');
    if(go){ if(go.disabled) return; go.disabled = true; go.classList.add('busy'); }
    try{ await exportAllBranches('zip'); }
    finally{ if(go){ go.disabled = false; go.classList.remove('busy'); } }
  }

  async function captureAndExport(){
    closeMenus();

    const requested = ['txt','jsonl','png'].filter(k=>EXPORT_FORMATS[k]);
    if(!requested.length){
      showPrompt(
        'Pick at least one format — PNG, JSONL, or TXT — then press ↓ again.',
        {okText:'ok', cancel:false, onOK:hidePrompt}
      );
      setStatus('no format selected','warn');
      return;
    }

    const go = $('#coGo');
    if(go){ go.disabled = true; go.classList.add('busy'); }

    const wasLite = LITE_BATCH;
    const wasMulti = MULTI_DOWNLOAD;
    LITE_BATCH = true;
    MULTI_DOWNLOAD = false;

    try{
      const picks = requested.slice();
      const skipped = [];
      const completed = [];
      const wantsChat = picks.includes('txt') || picks.includes('jsonl');

      // Always recapture chat-dependent formats. Reusing records here can export
      // stale messages after new replies arrive or after CrushOn changes SPA routes.
      if(wantsChat){
        {
          setStatus('capturing current chat…','muted');
          const captured = await captureNow(null, {
            captureCharacterProfile:false,
            silentSummary:true
          });

          if(captured===null){
            // Explicit Cancel means cancel the whole ↓ operation.
            return;
          }

          if(!records.length){
            for(const fmt of ['txt','jsonl']){
              const i=picks.indexOf(fmt);
              if(i>=0){
                picks.splice(i,1);
                skipped.push(`${fmt.toUpperCase()} skipped — chat capture returned no messages.`);
              }
            }
          }
        }
      }else{
        // PNG-only exports must not inherit stale chat/character state from a
        // previous route.
        records.length=0;
        seen.clear();
        CHAR=null;
        setCardBadge();
        const msgs=$('#coMsgs'); if(msgs) msgs.textContent='0';
        const split=$('#coSplit'); if(split) split.textContent='0U/0A';
      }

      // Character lookup is performed only when PNG is actually requested, and
      // it is refreshed every time so SPA navigation cannot reuse a stale CHAR.
      let pngReady = false;
      if(picks.includes('png')){
        CHAR=null;
        setCardBadge();
        setStatus('reading current character…','muted');
        await captureCharacter(s=>setStatus('character: '+s,'muted'));

        if(CHAR && CHAR.gate.allowed && CHAR.q.ok){
          pngReady = true;
        }else{
          const i = picks.indexOf('png');
          if(i >= 0) picks.splice(i, 1);
          skipped.push(
            'PNG skipped — ' +
            (CHAR ? (CHAR.gate.allowed ? CHAR.q.why : CHAR.gate.why) : 'no character data on this page')
          );
        }
      }

      if(!picks.length){
        setStatus('nothing to export','warn');
        showPrompt(
          skipped.length ? skipped.join('\n') : 'Nothing exportable was found on this page.',
          {okText:'ok', cancel:false, onOK:hidePrompt}
        );
        return;
      }

      // More than one actual output suppresses the one-file mobile share sheet.
      MULTI_DOWNLOAD = picks.length > 1;

      if(picks.includes('txt')){
        setStatus('exporting .txt…','muted');
        exportTxt();
        completed.push('• chat .txt');
      }

      if(picks.includes('jsonl')){
        setStatus('exporting .jsonl…','muted');
        exportSillyTavernJSONL();
        completed.push('• SillyTavern .jsonl');
      }

      if(picks.includes('png') && pngReady){
        setStatus('exporting character .png…','muted');
        const pngOK = await exportCharacterPNG();
        if(pngOK){
          completed.push(`• character .png — ${CHAR.q.filled.join(', ')}`);
        }else{
          skipped.push('PNG failed during card generation or download preparation.');
        }
      }

      const lines=[];
      if(completed.length){
        const exportedChat = picks.includes('txt') || picks.includes('jsonl');
        lines.push(exportedChat
          ? `Exported current chat (${records.length} message${records.length===1?'':'s'}).`
          : 'Export complete.');
        if(exportedChat){
          lines.push(`Source: ${CAPTURE_SOURCE}`);
          lines.push(CAPTURE_NOTE);
        }
        lines.push(...completed);
      }else{
        lines.push('Nothing was exported.');
      }

      if(skipped.length){
        lines.push('');
        lines.push(...skipped);
      }

      showPrompt(lines.join('\n'), {okText:'ok', cancel:false, onOK:hidePrompt});
      setStatus(
        completed.length ? 'exported: ' + completed.map(x=>x.replace(/^•\s*/, '')).join(' + ') : 'export failed',
        completed.length ? 'ok' : 'warn'
      );
    }catch(e){
      setStatus('export failed','warn');
      showPrompt('Export failed: ' + (e?.message || e),
        {okText:'ok', cancel:false, onOK:hidePrompt});
    }finally{
      if(go){ go.disabled = false; go.classList.remove('busy'); }
      // Keep the batch flags through currently queued download attempts.
      setTimeout(()=>{
        LITE_BATCH = wasLite;
        MULTI_DOWNLOAD = wasMulti;
      }, 0);
    }
  }

  // API captures keep the server's text verbatim; DOM captures use the
  // markdown-preserving flattening; older records fall back to stripHTML.
  function messageText(it){
    if(typeof it.raw==='string') return it.raw;
    if(typeof it.text==='string') return it.text;
    return stripHTML(it.html);
  }

  function exportTxt(){
    if(!records.length){
      showPrompt(
        'No captured chat is available to export.',
        {cancel:false, okText:'ok', onOK:hidePrompt}
      );
      return;
    }

    const items=orderedRecords();

    const lines=[
      '# '+(document.title||'CrushOn Chat'),
      'URL: '+location.href,
      'AI: '+exportAIName(),
      'Exported: '+new Date().toISOString(),
      'Source: '+CAPTURE_SOURCE,
      'Coverage: '+CAPTURE_NOTE,
      ''
    ];

    for(const it of items){
      lines.push(`[${labelFor(it.who)}] `+messageText(it));
      lines.push('');
    }

    download(
      'crushon-chat-'+ts()+'.txt',
      lines.join('\n')
    );

    setStatus('exported .txt','ok');
  }

  function exportSillyTavernJSONL(){
    if(!records.length){
      showPrompt(
        'No captured chat is available to export.',
        {cancel:false, okText:'ok', onOK:hidePrompt}
      );
      return;
    }

    closeMenus();

    const items=orderedRecords();
    const aiName=exportAIName() || 'AI';
    const userName='User';
    const exportedAt=new Date().toISOString();

    // SillyTavern's native chat format is JSON Lines:
    // one header object followed by one JSON object per message.
    const rows=[{
      chat_metadata:{
        source:'CrushOn Exporter PLUS',
        capture_source:CAPTURE_SOURCE,
        capture_note:CAPTURE_NOTE,
        source_url:location.href,
        imported_from:'crushon.ai',
        exported_at:exportedAt,
        crushon:(()=>{
          const r=chatRouteIds();
          return {
            exporter_version:SCRIPT_VERSION,
            character_id:r?.characterId ?? null,
            chat_id:r?.conversationId ?? null,
            ...CAPTURE_META
          };
        })()
      },
      user_name:'unused',
      character_name:'unused'
    }];

    for(const it of items){
      const isUser=it.who==='USER';
      const text=messageText(it);
      const hasTime=it.t!==null && it.t!==undefined && Number.isFinite(it.t);
      const fromServer=CAPTURE_META.capture==='message_tree' || CAPTURE_META.capture==='observed_request';

      rows.push({
        ...(it.swipe||{}),
        name:isUser ? userName : aiName,
        is_user:isUser,
        is_system:false,
        // Empty rather than inventing a timestamp at export time.
        send_date:hasTime ? new Date(it.t).toISOString() : '',
        mes:text,
        extra:{
          crushon:{
            created_at:hasTime ? new Date(it.t).toISOString() : '',
            time_source:hasTime ? (fromServer ? 'server' : 'page') : 'none',
            edited:false,
            ...(it.id ? {message_id:it.id} : {}),
            ...(it.crushon||{})
          }
        }
      });
    }

    const jsonl=rows
      .map(row=>JSON.stringify(row))
      .join('\n');

    const safe=aiName
      .replace(/[^\w\-]+/g,'_')
      .slice(0,48) || 'character';

    download(
      `${safe}-crushon-chat-${ts()}.jsonl`,
      jsonl,
      'application/x-ndjson;charset=utf-8'
    );

    setStatus(
      'exported SillyTavern .jsonl',
      'ok'
    );

    if(!LITE_BATCH){
      showPrompt(
        `Saved a SillyTavern-importable JSONL chat with ${items.length} messages.\n\n` +
        'Import it from SillyTavern’s chat-file import control for the character you want this history attached to.',
        {
          cancel:false,
          okText:'ok',
          onOK:hidePrompt
        }
      );
    }
  }

  function safeExportHtml(markup){
    // Build an inert tree, then copy only explicitly approved elements and
    // attributes. Imported chat content is NEVER inserted into the export raw.
    const parsed=new DOMParser().parseFromString('<div id="co-clean">'+String(markup||'')+'</div>','text/html');
    const root=parsed.getElementById('co-clean');
    if(!root) return escapeHtml(stripHTML(markup));
    const allowed=new Set('p br div span strong b em i u s del strike blockquote pre code ul ol li h1 h2 h3 h4 h5 h6 hr table thead tbody tr th td a img sup sub'.split(' '));
    const blocked=new Set('script style iframe object embed svg math link meta base form input button textarea select video audio source canvas template noscript'.split(' '));
    const doc=parsed.createElement('div');
    function appendSafe(src,dest){
      for(const child of [...src.childNodes]){
        if(child.nodeType===Node.TEXT_NODE){
          dest.appendChild(parsed.createTextNode(child.textContent||''));
          continue;
        }
        if(child.nodeType!==Node.ELEMENT_NODE) continue;
        const tag=child.localName.toLowerCase();
        if(blocked.has(tag)) continue;
        if(!allowed.has(tag)){ appendSafe(child,dest); continue; }
        const node=parsed.createElement(tag);
        if(tag==='a'){
          const raw=child.getAttribute('href')||'';
          try{
            const url=new URL(raw,location.origin);
            if(['http:','https:','mailto:'].includes(url.protocol)){
              node.setAttribute('href',url.href);
              node.setAttribute('rel','noopener noreferrer');
              node.setAttribute('target','_blank');
            }
          }catch(_){}
        }
        if(tag==='img'){
          const raw=directImageUrl(child.getAttribute('src')||'');
          try{
            const url=new URL(raw,location.origin);
            if(['https:','http:'].includes(url.protocol)) node.setAttribute('src',url.href);
          }catch(_){}
          node.setAttribute('alt',(child.getAttribute('alt')||'').slice(0,200));
        }
        if(tag==='th' || tag==='td'){
          const colspan=Number(child.getAttribute('colspan'));
          if(Number.isInteger(colspan) && colspan>=1 && colspan<=16) node.setAttribute('colspan',String(colspan));
        }
        appendSafe(child,node);
        dest.appendChild(node);
      }
    }
    appendSafe(root,doc);
    return doc.innerHTML;
  }

  async function exportHTML(){
    closeMenus();

    setStatus('capturing current chat for HTML…','muted');
    const captured = await captureNow(null, {
      captureCharacterProfile:false,
      silentSummary:true
    });

    if(captured===null) return false;

    if(!records.length){
      showPrompt(
        'Chat capture returned no messages, so no HTML file was written.',
        {cancel:false, okText:'ok', onOK:hidePrompt}
      );
      setStatus('HTML export unavailable','warn');
      return false;
    }

    const items=orderedRecords();

    const head=`<!doctype html><html><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><style>
:root{color-scheme:dark light}
body{margin:24px;font-family:ui-sans-serif,system-ui,-apple-system,Segoe UI,Roboto,Arial,sans-serif;background:#0b0b0b;color:#e6e6e6}
a{color:#7bb7ff}.chat{max-width:920px;margin:0 auto}.meta{font:12px/1.4 monospace;opacity:.8;margin-bottom:12px}.title{font:16px/1.4 monospace;margin:0 0 6px 0}
article.msg{position:relative;background:#121212;border:1px solid #333;border-radius:12px;padding:18px 16px 14px;margin:12px 0}
article.msg .cap{position:absolute;top:-10px;left:12px;font:11px/1 monospace;letter-spacing:.3px;background:#1c1c1c;border:1px solid #3a3a3a;border-radius:8px;padding:3px 6px}
article.msg .cap.user{background:#291a2d;border-color:#5a3a66}article.msg .cap.ai{background:#182028;border-color:#334655}
article.msg .body img{max-width:100%;height:auto}pre,code{background:#0e0e0e;border:1px solid #2a2a2a;border-radius:6px;padding:2px 4px}
</style></head><body><div class="chat"><div class="title"># ${escapeHtml(document.title||'CrushOn Chat')}</div><div class="meta">URL: ${escapeHtml(location.href)}<br>AI: ${escapeHtml(exportAIName())}<br>Exported: ${new Date().toISOString()}<br>Source: ${escapeHtml(CAPTURE_SOURCE)}<br>Coverage: ${escapeHtml(CAPTURE_NOTE)}</div>`;

    const body=items.map(it=>{
      const capCls=(it.who==='USER')?'user':'ai';
      const capTxt='['+escapeHtml(labelFor(it.who))+']';

      return `<article class="msg"><div class="cap ${capCls}">${capTxt}</div><div class="body">${safeExportHtml(it.html)}</div></article>`;
    }).join('');

    const html=head+body+'</div></body></html>';

    download(
      'crushon-chat-'+ts()+'.html',
      html,
      'text/html;charset=utf-8'
    );

    setStatus('exported .html','ok');

    return true;
  }

  // ---------- SillyTavern character card export ----------
  // Builds a chara_card_v2 PNG
  // (tEXt "chara" = base64 JSON)
  // using the character's avatar.

  const CARD_KEYS = {
    name: [
      'name',
      'characterName',
      'charName',
      'title',
      'displayName'
    ],

    description: [
      'description',
      'intro',
      'bio',
      'summary',
      'about',
      'shortDescription',
      'profile'
    ],

    personality: [
      'personality',
      'persona',
      'definition',
      'characterDefinition',
      'prompt',
      'systemPrompt',
      'instructions',
      'longDescription'
    ],

    scenario: [
      'scenario',
      'worldScenario',
      'world',
      'setting',
      'context'
    ],

    first_mes: [
      'greeting',
      'firstMessage',
      'first_mes',
      'firstMes',
      'initialMessage',
      'openingMessage',
      'welcomeMessage'
    ],

    mes_example: [
      'example_conversation',
      'exampleDialogs',
      'exampleDialogue',
      'exampleConversation',
      'example_dialogs',
      'mes_example',
      'examples',
      'sampleDialog'
    ],

    creator: [
      'creator',
      'author',
      'userName',
      'username',
      'creatorName',
      'ownerName',
      'nickName'
    ],

    avatar: [
      'avatar',
      'avatarUrl',
      'avatar_url',
      'image',
      'imageUrl',
      'image_url',
      'picture',
      'photo',
      'cover',
      'coverUrl',
      'thumbnail',
      'profileImage'
    ]
  };

  const pick = (obj, keys) => {
    for(const k of keys){
      const v = obj?.[k];

      if(typeof v === 'string' && v.trim()){
        return v.trim();
      }

      if(v && typeof v === 'object'){
        const inner = v.url || v.src || v.text || v.value;

        if(typeof inner === 'string' && inner.trim()){
          return inner.trim();
        }
      }
    }

    return '';
  };

  function characterIdFromPage(){
    const m = location.pathname.match(
      /\/character\/([^/?#]+)/i
    );

    if(m){
      return decodeURIComponent(m[1]);
    }

    const a = document.querySelector(
      'a[href*="/character/"]'
    );

    const m2 = a && a.getAttribute('href').match(
      /\/character\/([^/?#]+)/i
    );

    return m2
      ? decodeURIComponent(m2[1])
      : '';
  }

  // ---------- deep search ----------
  function scoreCandidate(o){
    if(!o || typeof o!=='object' || Array.isArray(o)){
      return 0;
    }

    if(!pick(o, CARD_KEYS.name)){
      return 0;
    }

    let s = 1;

    if(pick(o, CARD_KEYS.personality)) s += 4;
    if(pick(o, CARD_KEYS.first_mes))   s += 3;
    if(pick(o, CARD_KEYS.description)) s += 2;
    if(pick(o, CARD_KEYS.scenario))    s += 1;
    if(pick(o, CARD_KEYS.avatar))      s += 1;

    return s;
  }

  function deepBest(
    root,
    best={score:0, obj:null},
    depth=0,
    seen=new Set()
  ){
    if(
      !root ||
      typeof root!=='object' ||
      depth>12 ||
      seen.has(root)
    ){
      return best;
    }

    seen.add(root);

    const s = scoreCandidate(root);

    if(s > best.score){
      best = {
        score:s,
        obj:root
      };
    }

    for(const k in root){
      let v;

      try{
        v = root[k];
      }catch(_){
        continue;
      }

      if(v && typeof v==='object'){
        best = deepBest(
          v,
          best,
          depth+1,
          seen
        );
      }
    }

    return best;
  }

  // Pull JSON objects out of Next.js flight/embedded payloads.
  function jsonEndFrom(str, start){
    let depth=0;
    let inStr=false;
    let esc=false;

    for(let i=start;i<str.length;i++){
      const c = str[i];

      if(inStr){
        if(esc) esc=false;
        else if(c==='\\') esc=true;
        else if(c==='"') inStr=false;
        continue;
      }

      if(c==='"'){
        inStr=true;
      }else if(c==='{'){
        depth++;
      }else if(c==='}'){
        depth--;

        if(!depth){
          return i+1;
        }
      }
    }

    return -1;
  }

  function objectsAroundMarkers(text){
    const out=[];

    const markers=/"(?:personality|persona|definition|greeting|firstMessage|first_mes|scenario|exampleDialogs)"\s*:/g;

    let m;
    let guard=0;

    while(
      (m = markers.exec(text)) &&
      guard++ < 60
    ){
      let starts=[];
      let i=m.index;

      while(i>=0 && starts.length<25){
        i = text.lastIndexOf('{', i-1);

        if(i<0) break;

        starts.push(i);
      }

      for(const st of starts){
        const end = jsonEndFrom(text, st);

        if(end > m.index){
          try{
            out.push(
              JSON.parse(
                text.slice(st, end)
              )
            );
          }catch(_){}

          break;
        }
      }
    }

    return out;
  }

  function embeddedPayloads(){
    const blobs=[];

    try{
      if(window.__NEXT_DATA__){
        blobs.push(window.__NEXT_DATA__);
      }
    }catch(_){}

    try{
      const f = self.__next_f;

      if(Array.isArray(f)){
        const text = f
          .map(x=>Array.isArray(x)? x[1] : '')
          .filter(x=>typeof x==='string')
          .join('');

        if(text){
          objectsAroundMarkers(
            text
              .replace(/\\"/g,'"')
              .replace(/\\n/g,'\n')
          ).forEach(
            o=>blobs.push(o)
          );
        }
      }
    }catch(_){}

    qsa(
      'script[type="application/json"], script[type="application/ld+json"]'
    ).forEach(s=>{
      try{
        blobs.push(
          JSON.parse(s.textContent)
        );
      }catch(_){}
    });

    return blobs;
  }

  // ---------- tRPC character lookup ----------
  function mergeChar(a, b){
    if(
      !b ||
      typeof b!=='object' ||
      Array.isArray(b)
    ){
      return a;
    }

    if(!a){
      return Object.assign({}, b);
    }

    for(const k in b){
      let v;

      try{
        v = b[k];
      }catch(_){
        continue;
      }

      const cur = a[k];

      if(
        v === null ||
        v === undefined ||
        v === ''
      ){
        continue;
      }

      if(typeof v === 'string'){
        if(
          typeof cur !== 'string' ||
          v.length > cur.length
        ){
          a[k] = v;
        }

        continue;
      }

      if(Array.isArray(v)){
        if(
          !Array.isArray(cur) ||
          v.length > cur.length
        ){
          a[k] = v;
        }

        continue;
      }

      if(typeof v === 'object'){
        a[k] = mergeChar(
          cur &&
          typeof cur==='object' &&
          !Array.isArray(cur)
            ? cur
            : null,
          v
        );

        continue;
      }

      if(
        cur === undefined ||
        cur === null ||
        cur === ''
      ){
        a[k] = v;
      }
    }

    return a;
  }

  function unwrapChar(j){
    if(!j || typeof j!=='object'){
      return null;
    }

    if(pick(j, CARD_KEYS.name)){
      return j;
    }

    for(const k of [
      'character',
      'oc',
      'data',
      'info',
      'characterInfo',
      'detail'
    ]){
      const v = j[k];

      if(
        v &&
        typeof v==='object' &&
        pick(v, CARD_KEYS.name)
      ){
        return v;
      }
    }

    const best = deepBest(j);

    return best.obj;
  }

  async function trpcCharacterLookup(id, onStep){
    if(!id){
      return null;
    }

    const step = onStep || (()=>{});

    let acc = null;
    let hits = [];

    const attempt = async (proc, input) => {
      try{
        const j = await trpc(proc, input);
        const c = unwrapChar(j);

        if(c){
          acc = mergeChar(acc, c);
          hits.push(proc);
          return true;
        }
      }catch(_){}

      return false;
    };

    step('getCharacterForNewChat');

    await attempt(
      'character.getCharacterForNewChat',
      {characterId:id}
    );

    step('getOriginalCharacterInfo');

    await attempt(
      'character.getOriginalCharacterInfo',
      {characterId:id}
    );

    if(
      !pick(acc||{}, CARD_KEYS.personality) ||
      !pick(acc||{}, CARD_KEYS.first_mes)
    ){
      step('getCharacter');

      for(const dp of [
        2,5,6,8,9,
        0,1,3,4,7,
        10,11,12
      ]){
        if(
          await attempt(
            'character.getCharacter',
            {
              characterId:id,
              displayPosition:dp
            }
          )
        ){
          break;
        }
      }
    }

    if(!acc){
      step('getCharacter4Memory');

      await attempt(
        'character.getCharacter4Memory',
        {characterId:id}
      );
    }

    // Owner-only details are not part of normal character lookups. The
    // server is the authority: if it denies a request we never retry with
    // forged ownership or merge any unauthorized data.
    const me=await currentUserId();
    if(me && (!acc || !characterCreatorId(acc) || characterCreatorId(acc)===me)){
      step('queryOwnCharacter');
      await attempt('character.queryOwnCharacter',{characterId:id});
    }

    step('queryRecentCharacters');

    try{
      const j = await trpc(
        'character.queryRecentCharacters',
        {
          nsfw:true,
          locale:'en'
        }
      );

      const list =
        j?.characters ||
        j?.items ||
        (
          Array.isArray(j)
            ? j
            : []
        );

      const row = Array.isArray(list)
        ? list.find(
            x =>
              x &&
              String(x.id)===String(id)
          )
        : null;

      if(row){
        acc = mergeChar(acc, row);
        hits.push('queryRecentCharacters');
      }
    }catch(_){}

    if(acc){
      acc.__procs = hits;
    }

    return acc;
  }

  // ---------- DOM fallback ----------
  function domFallback(){
    const meta = n =>
      document.querySelector(
        `meta[property="${n}"], meta[name="${n}"]`
      )?.content || '';

    const name = cleanAiNameExact(
      meta('og:title') ||
      document.title
    );

    if(
      !name ||
      name==='AI'
    ){
      return null;
    }

    return {
      name,
      description:
        meta('og:description') || '',

      avatar:
        meta('og:image') ||
        document.querySelector(
          'img[src*="avatar"], #chat-container img'
        )?.src ||
        '',

      __fallback:true
    };
  }

  // ---------- access gate ----------
  let _meId = null;

  async function currentUserId(){
    if(_meId !== null){
      return _meId;
    }

    _meId = '';

    try{
      const r = await fetch(
        location.origin + '/api/auth/session',
        {
          credentials:'include',
          headers:{
            accept:'application/json'
          }
        }
      );

      if(r.ok){
        const j = await r.json();
        const u = j?.user || j;

        const v =
          u?.id ||
          u?.userId ||
          u?.sub ||
          j?.userId ||
          '';

        if(v){
          _meId = String(v);
        }
      }
    }catch(_){}

    if(!_meId){
      const hunt = (
        o,
        d=0,
        seen=new Set()
      ) => {
        if(
          !o ||
          typeof o!=='object' ||
          d>8 ||
          seen.has(o)
        ){
          return '';
        }

        seen.add(o);

        if(
          (o.id || o.userId || o._id) &&
          (
            o.email ||
            o.userName ||
            o.username ||
            o.nickName
          )
        ){
          return String(
            o.id ||
            o.userId ||
            o._id
          );
        }

        for(const k in o){
          let v;

          try{
            v=o[k];
          }catch(_){
            continue;
          }

          if(v && typeof v==='object'){
            const r=hunt(
              v,
              d+1,
              seen
            );

            if(r){
              return r;
            }
          }
        }

        return '';
      };

      for(const b of embeddedPayloads()){
        const r = hunt(b);

        if(r){
          _meId = r;
          break;
        }
      }
    }

    return _meId;
  }

  function characterCreatorId(c){
    const v =
      (
        c &&
        c.user &&
        (
          c.user.id ??
          c.user.userId
        )
      )
      ??
      (
        c &&
        c.creator &&
        (
          c.creator.id ??
          c.creator.userId
        )
      )
      ??
      c?.creatorId
      ??
      c?.userId
      ??
      c?.authorId
      ??
      c?.ownerId;

    return v
      ? String(v)
      : '';
  }

  async function accessCheck(c){
    const owner = characterCreatorId(c);
    const me = await currentUserId();

    if(
      owner &&
      me &&
      owner === me
    ){
      return {
        allowed:true,
        why:'you own this character'
      };
    }

    const vis = c?.visibility;

    const visStr = String(
      vis ??
      c?.status ??
      c?.privacy ??
      ''
    ).toLowerCase();

    const isUnlisted = vis === 1 || visStr === '1' || visStr === 'unlisted';
    const isPrivate =
      c?.isPrivate === true ||
      c?.private === true ||
      vis === 2 ||
      visStr === '2' ||
      /^(private|draft|hidden)$/.test(visStr) ||
      (!isUnlisted && (c?.isPublic === false || c?.public === false));

    if(isPrivate){
      return {
        allowed:false,
        why:'this character is private and not yours — export blocked'
      };
    }

    return {
      allowed:true,
      why:isUnlisted ? 'unlisted profile accessible to you' : 'public profile'
    };
  }

  // ---------- PNG writing ----------
  const PNG_SIG = [
    137,80,78,71,
    13,10,26,10
  ];

  const CRC_TABLE = (()=>{
    const t=new Uint32Array(256);

    for(let n=0;n<256;n++){
      let c=n;

      for(let k=0;k<8;k++){
        c = (c&1)
          ? (0xEDB88320 ^ (c>>>1))
          : (c>>>1);
      }

      t[n]=c>>>0;
    }

    return t;
  })();

  function crc32(bytes){
    let c=0xFFFFFFFF;

    for(let i=0;i<bytes.length;i++){
      c = CRC_TABLE[
        (c ^ bytes[i]) & 255
      ] ^ (c>>>8);
    }

    return (
      c ^ 0xFFFFFFFF
    )>>>0;
  }

  function be32(n){
    return [
      (n>>>24)&255,
      (n>>>16)&255,
      (n>>>8)&255,
      n&255
    ];
  }

  function latin1(str){
    const a=new Uint8Array(str.length);

    for(let i=0;i<str.length;i++){
      a[i]=str.charCodeAt(i)&255;
    }

    return a;
  }

  function makeChunk(type, dataBytes){
    const body = new Uint8Array(
      4 + dataBytes.length
    );

    body.set(
      latin1(type),
      0
    );

    body.set(
      dataBytes,
      4
    );

    const out = new Uint8Array(
      4 + body.length + 4
    );

    out.set(
      be32(dataBytes.length),
      0
    );

    out.set(
      body,
      4
    );

    out.set(
      be32(crc32(body)),
      4 + body.length
    );

    return out;
  }

  function textChunk(keyword, text){
    const k = latin1(keyword);
    const t = latin1(text);

    const d = new Uint8Array(
      k.length + 1 + t.length
    );

    d.set(k,0);
    d[k.length]=0;
    d.set(t, k.length+1);

    return makeChunk(
      'tEXt',
      d
    );
  }

  function b64utf8(str){
    const bytes = new TextEncoder()
      .encode(str);

    let s='';
    const CH=0x8000;

    for(
      let i=0;
      i<bytes.length;
      i+=CH
    ){
      s += String.fromCharCode.apply(
        null,
        bytes.subarray(
          i,
          i+CH
        )
      );
    }

    return btoa(s);
  }

  function embedCharaChunk(png, jsonStr){
    if(
      !PNG_SIG.every(
        (b,i)=>png[i]===b
      )
    ){
      throw new Error(
        'not a PNG'
      );
    }

    const keep=[];
    let end=null;
    let p=8;

    const dv = new DataView(
      png.buffer,
      png.byteOffset,
      png.byteLength
    );

    while(p + 8 <= png.length){
      const len = dv.getUint32(p);

      const type =
        String.fromCharCode(
          png[p+4],
          png[p+5],
          png[p+6],
          png[p+7]
        );

      const chunk = png.subarray(
        p,
        p + 12 + len
      );

      if(type === 'IEND'){
        end = chunk;
        break;
      }

      const isOldCard =
        type==='tEXt' &&
        /^(chara|ccv3)\0/.test(
          String.fromCharCode(
            ...chunk.subarray(
              8,
              Math.min(
                8+8,
                chunk.length
              )
            )
          )
        );

      if(!isOldCard){
        keep.push(chunk);
      }

      p += 12 + len;
    }

    const b64 = b64utf8(jsonStr);

    const parts = [
      new Uint8Array(PNG_SIG),
      ...keep,
      textChunk('chara', b64),
      end ||
      makeChunk(
        'IEND',
        new Uint8Array(0)
      )
    ];

    const total = parts.reduce(
      (n,a)=>n+a.length,
      0
    );

    const out = new Uint8Array(total);

    let o=0;

    for(const a of parts){
      out.set(a, o);
      o += a.length;
    }

    return out;
  }

  // ---------- image fetching / conversion ----------
  function gmFetchBlob(url){
    return new Promise((resolve,reject)=>{
      if(
        typeof GM_xmlhttpRequest !== 'function'
      ){
        return reject(
          new Error(
            'no GM_xmlhttpRequest'
          )
        );
      }

      GM_xmlhttpRequest({
        method:'GET',
        url,
        responseType:'blob',

        onload:r=>{
          if(
            r.status>=200 &&
            r.status<300 &&
            r.response
          ){
            resolve(r.response);
          }else{
            reject(
              new Error(
                'http '+r.status
              )
            );
          }
        },

        onerror:()=>{
          reject(
            new Error('gm error')
          );
        },

        ontimeout:()=>{
          reject(
            new Error('timeout')
          );
        }
      });
    });
  }

  async function getImageBlob(url){
    if(!url){
      throw new Error(
        'no avatar url'
      );
    }

    const abs = directImageUrl(new URL(
      url,
      location.href
    ).href);

    try{
      const r = await fetch(
        abs,
        {
          mode:'cors',
          credentials:'omit'
        }
      );

      if(r.ok){
        return await r.blob();
      }
    }catch(_){}

    return await gmFetchBlob(abs);
  }

  async function blobToPngBytes(blob){
    const head = new Uint8Array(
      await blob
        .slice(0,8)
        .arrayBuffer()
    );

    if(
      PNG_SIG.every(
        (b,i)=>head[i]===b
      )
    ){
      return new Uint8Array(
        await blob.arrayBuffer()
      );
    }

    let src;

    try{
      src = await createImageBitmap(blob);
    }catch(_){
      src = await new Promise(
        (res,rej)=>{
          const u=URL.createObjectURL(blob);
          const im=new Image();

          im.onload=()=>{
            URL.revokeObjectURL(u);
            res(im);
          };

          im.onerror=()=>{
            URL.revokeObjectURL(u);
            rej(
              new Error(
                'decode failed'
              )
            );
          };

          im.src=u;
        }
      );
    }

    const cv = document.createElement(
      'canvas'
    );

    cv.width = src.width || 400;
    cv.height = src.height || 600;

    cv.getContext('2d')
      .drawImage(
        src,
        0,
        0
      );

    const out = await new Promise(
      res => cv.toBlob(
        res,
        'image/png'
      )
    );

    return new Uint8Array(
      await out.arrayBuffer()
    );
  }

  async function placeholderPngBytes(name){
    const cv = document.createElement(
      'canvas'
    );

    cv.width=400;
    cv.height=600;

    const g = cv.getContext('2d');

    g.fillStyle='#15151a';
    g.fillRect(
      0,
      0,
      400,
      600
    );

    g.fillStyle='#e6e6e6';
    g.font='bold 120px system-ui,sans-serif';
    g.textAlign='center';
    g.textBaseline='middle';

    g.fillText(
      (name||'?')
        .trim()
        .slice(0,2)
        .toUpperCase(),
      200,
      280
    );

    g.fillStyle='#9a9aa8';
    g.font='22px system-ui,sans-serif';

    g.fillText(
      String(name||'').slice(0,24),
      200,
      400
    );

    const blob = await new Promise(
      res => cv.toBlob(
        res,
        'image/png'
      )
    );

    return new Uint8Array(
      await blob.arrayBuffer()
    );
  }

  // ---------- card assembly ----------
  function normalizeExamples(v){
    if(Array.isArray(v)){
      return v
        .map(
          x =>
            typeof x==='string'
              ? x
              : (
                  x?.text ||
                  x?.content ||
                  ''
                )
        )
        .filter(Boolean)
        .join('\n');
    }

    return typeof v==='string'
      ? v
      : '';
  }

  function exampleTextOf(c){
    const s = stripHTML(
      normalizeExamples(
        c?.example_conversation
        ??
        c?.exampleDialogs
        ??
        c?.exampleDialogue
        ??
        c?.exampleConversation
        ??
        c?.mes_example
        ??
        c?.examples
      )
    );

    if(s && s.trim()){
      return s;
    }

    const msgs =
      c?.newExampleConversation
        ?.messages;

    if(
      Array.isArray(msgs) &&
      msgs.length
    ){
      const out = msgs
        .map(m => {
          const txt = stripHTML(
            (
              m?.content
              ??
              m?.text
              ??
              m?.message
              ??
              ''
            ).toString()
          );

          if(!txt){
            return '';
          }

          const who = String(
            m?.role
            ??
            m?.who
            ??
            ''
          ).toLowerCase();

          const tag =
            /user|human/.test(who)
              ? '{{user}}'
              : '{{char}}';

          return tag + ': ' + txt;
        })
        .filter(Boolean);

      if(out.length){
        return '<START>\n' +
          out.join('\n');
      }
    }

    return '';
  }

  function cardQuality(c, source){
    const fromApi =
      /^trpc/.test(
        String(source||'')
      );

    const f = {
      name:
        !!cleanAiNameExact(
          pick(
            c,
            CARD_KEYS.name
          )
        ),

      description:
        !!stripHTML(
          pick(
            c,
            CARD_KEYS.description
          )
        ),

      personality:
        !!String(
          c?.personality || ''
        ).trim(),

      scenario:
        !!String(
          c?.scenario || ''
        ).trim(),

      first_mes:
        !!String(
          pick(
            c,
            CARD_KEYS.first_mes
          ) || ''
        ).trim(),

      mes_example:
        !!exampleTextOf(c)
    };

    const filled = [
      'description',
      'personality',
      'scenario',
      'first_mes',
      'mes_example'
    ].filter(k=>f[k]);

    const defMissing =
      !f.personality &&
      !f.scenario &&
      !f.mes_example;

    const explicitHidden =
      c?.definitionVisibility === 2 ||
      c?.definitionVisibility === '2' ||
      c?.definitionHidden === true ||
      c?.hideDefinition === true ||
      c?.isDefinitionHidden === true;
    const defHidden = defMissing && explicitHidden;

    return {
      fromApi,
      fields:f,
      filled,
      defHidden,

      ok:
        fromApi &&
        f.name &&
        filled.length > 0,

      why:
        !fromApi
          ? 'only page text was available (no API data)'
          : !f.name
            ? 'no character name'
            : !filled.length
              ? 'the character record came back empty'
              : ''
    };
  }

  function creatorName(c){
    const cand = [
      c?.creator,
      c?.user,
      c?.author
    ];

    for(const v of cand){
      if(
        typeof v === 'string' &&
        v.trim()
      ){
        return v.trim();
      }

      if(
        v &&
        typeof v === 'object'
      ){
        const n =
          v.name ||
          v.userName ||
          v.username ||
          v.nickName;

        if(
          typeof n === 'string' &&
          n.trim()
        ){
          return n.trim();
        }
      }
    }

    return pick(
      c,
      CARD_KEYS.creator
    );
  }

  function avatarUrlOf(c){
    const ca = c?.characterAvatar;

    const cands = [
      ca?.avatar,
      ca?.originalAvatar,
      ca?.thumbnailAvatar,
      c?.avatar,
      c?.avatarUrl,
      c?.image,
      c?.imageUrl,
      c?.cover
    ];

    for(const v of cands){
      if(
        typeof v==='string' &&
        v.trim()
      ){
        return v.trim();
      }
    }

    return pick(
      c,
      CARD_KEYS.avatar
    );
  }

  function buildV2Card(c, meta){
    const data = {
      name:
        cleanAiNameExact(
          pick(
            c,
            CARD_KEYS.name
          )
        ),

      description:
        stripHTML(
          pick(
            c,
            CARD_KEYS.description
          )
        ),

      personality:
        stripHTML(
          pick(
            c,
            CARD_KEYS.personality
          )
        ),

      scenario:
        stripHTML(
          pick(
            c,
            CARD_KEYS.scenario
          )
        ),

      first_mes:
        stripHTML(
          pick(
            c,
            CARD_KEYS.first_mes
          )
        ),

      mes_example:
        exampleTextOf(c),

      creator_notes:
        `Exported from ${location.href} on ${new Date().toISOString()}`,

      system_prompt:'',
      post_history_instructions:'',

      alternate_greetings:
        Array.isArray(
          c.alternateGreetings
        )
          ? c.alternateGreetings.map(String)
          : Array.isArray(c.greetings)
            ? c.greetings
                .slice(1)
                .map(String)
            : [],

      tags:
        Array.isArray(c.tags)
          ? c.tags
              .map(
                t =>
                  typeof t==='string'
                    ? t
                    : (
                        t?.slug ||
                        t?.name ||
                        t?.title ||
                        ''
                      )
              )
              .filter(Boolean)
          : [],

      creator:
        creatorName(c),

      character_version:
        String(
          c.version ??
          '1.0'
        ),

      extensions:{
        crushon:{
          id:meta.id || '',
          url:location.href,
          source:meta.source
        }
      }
    };

    const extra = [];

    const appearance =
      String(
        c?.appearance || ''
      ).trim();

    if(appearance){
      extra.push(
        'Appearance: ' +
        stripHTML(appearance)
      );
    }

    if(c?.age){
      extra.push(
        'Age: ' + c.age
      );
    }

    if(extra.length){
      data.description =
        (
          data.description
            ? data.description + '\n\n'
            : ''
        )
        +
        extra.join('\n');
    }

    const q = cardQuality(
      c,
      meta.source
    );

    if(q.defHidden){
      data.creator_notes +=
        '\nNOTE: the creator hid this character’s definition on CrushOn, ' +
        'so personality / scenario / example dialogue were not available.';
    }

    const card = {
      spec:'chara_card_v2',
      spec_version:'2.0',
      data
    };

    return Object.assign(
      {
        name:data.name,
        description:data.description,
        personality:data.personality,
        scenario:data.scenario,
        first_mes:data.first_mes,
        mes_example:data.mes_example
      },
      card
    );
  }

  async function findCharacter(onStep){
    const id = characterIdFromPage();

    if(id){
      const api =
        await trpcCharacterLookup(
          id,
          onStep
        );

      if(api){
        return {
          c:api,
          id,
          source:
            'trpc:' +
            (
              api.__procs ||
              []
            ).join('+')
        };
      }
    }

    let best = {
      score:0,
      obj:null
    };

    for(const b of embeddedPayloads()){
      best = deepBest(
        b,
        best
      );
    }

    if(best.obj){
      return {
        c:best.obj,
        id,
        source:'page-data'
      };
    }

    const dom = domFallback();

    if(dom){
      return {
        c:dom,
        id,
        source:'dom'
      };
    }

    return null;
  }

  // SillyTavern v2 card PNG for a looked-up character ({c, id, source}).
  // The page's og:image is only a valid avatar fallback for the character on
  // screen. Returns {bytes, name}.
  async function characterPngBytes(found, nameFallback, onScreen){
    const card = buildV2Card(found.c, {id:found.id, source:found.source});
    if(!card.name || card.name==='AI'){
      card.name = card.data.name = nameFallback || 'Character';
    }
    const avatarUrl =
      avatarUrlOf(found.c) ||
      (onScreen ? document.querySelector('meta[property="og:image"]')?.content : '') ||
      '';
    let png;
    try{
      png = await blobToPngBytes(await getImageBlob(avatarUrl));
    }catch(err){
      png = await placeholderPngBytes(card.name);
    }
    return {bytes:embedCharaChunk(png, JSON.stringify(card)), name:card.name};
  }

  // Card for a character that isn't on screen (bulk export). Same access and
  // quality gates as the ↓ PNG export. Returns {ok, bytes, name, filled} or {ok:false, why}.
  async function characterPngById(characterId, nameFallback, onStep){
    await paceWait();
    const c = await trpcCharacterLookup(characterId, onStep);
    if(!c) return {ok:false, why:'CrushOn returned no character data'};
    const gate = await accessCheck(c);
    if(!gate.allowed) return {ok:false, why:gate.why};
    const source = 'trpc:' + (c.__procs || []).join('+');
    const q = cardQuality(c, source);
    if(!q.ok) return {ok:false, why:q.why};
    const built = await characterPngBytes({c, id:characterId, source}, nameFallback, false);
    return {ok:true, ...built, filled:q.filled};
  }

  async function exportCharacterPNG(){
    closeMenus();

    setStatus(
      'reading character…',
      'muted'
    );

    if(!LITE_BATCH){
      showPrompt(
        'Looking up the character…',
        {
          ok:false,
          cancel:false
        }
      );
    }

    try{
      if(!CHAR){
        await captureCharacter(
          s => {
            if(!LITE_BATCH){
              updatePrompt(
                'Looking up the character… ('+s+')',
                {
                  ok:false,
                  cancel:false
                }
              );
            }
          }
        );
      }

      if(!CHAR){
        if(!LITE_BATCH){
          updatePrompt(
            'Could not find character data on this page. Open the character’s profile or a chat with them, then try again.',
            {
              cancel:false,
              okText:'ok',
              onOK:hidePrompt
            }
          );
        }

        setStatus(
          'character not found',
          'warn'
        );

        return false;
      }

      const {
        c,
        id,
        source,
        gate,
        q
      } = CHAR;

      if(!gate.allowed){
        if(!LITE_BATCH){
          updatePrompt(
            `Export blocked: ${gate.why}.`,
            {
              cancel:false,
              okText:'ok',
              onOK:hidePrompt
            }
          );
        }

        setStatus(
          'export blocked',
          'warn'
        );

        return false;
      }

      if(!q.ok){
        if(!LITE_BATCH){
          updatePrompt(
            `Can't build a character card: ${q.why}.\n\n` +
            'Open the character’s profile or a chat with them, then try again.',
            {
              cancel:false,
              okText:'ok',
              onOK:hidePrompt
            }
          );
        }

        setStatus(
          'no character data',
          'warn'
        );

        return false;
      }

      if(!LITE_BATCH){
        updatePrompt(
          `Building card (${gate.why})…`,
          {
            ok:false,
            cancel:false
          }
        );
      }

      const built = await characterPngBytes(
        {c, id, source},
        currentAIName(),
        true
      );
      const card = {name:built.name};
      const out = built.bytes;

      const safe =
        card.name
          .replace(
            /[^\w\-]+/g,
            '_'
          )
          .slice(0,48)
        ||
        'character';

      download(
        `${safe}.png`,
        out,
        'image/png'
      );

      if(!LITE_BATCH){
        updatePrompt(
          [
            `Saved ${safe}.png — SillyTavern v2 card.`,
            '',
            `Included: ${q.filled.join(', ')}.`,
            q.defHidden
              ? 'The creator hid this character’s definition on CrushOn, so personality, scenario and example dialogue were not available.'
              : '',
            `Source: ${source}.`
          ]
            .filter(Boolean)
            .join('\n'),

          {
            cancel:false,
            okText:'ok',
            onOK:hidePrompt
          }
        );
      }

      setStatus(
        'exported character .png',
        'ok'
      );
      return true;
    }catch(err){
      if(!LITE_BATCH){
        updatePrompt(
          'Character export failed: ' +
          (
            err &&
            err.message ||
            err
          ),
          {
            cancel:false,
            okText:'ok',
            onOK:hidePrompt
          }
        );
      }

      setStatus(
        'character export failed',
        'warn'
      );
      return false;
    }
  }

  // ---------- CrushOn tRPC + RSC access ----------
  function trpcInput(v){
    return (
      v === undefined ||
      v === null
    )
      ? {
          json:null,
          meta:{
            values:[
              'undefined'
            ]
          }
        }
      : {
          json:v
        };
  }

  async function trpc(proc, input){
    const url =
      `${location.origin}/api/trpc/${proc}?batch=1&input=` +
      encodeURIComponent(
        JSON.stringify({
          0:trpcInput(input)
        })
      );

    const r = await fetch(
      url,
      {
        credentials:'include',
        headers:{
          'trpc-accept':'application/json',
          'content-type':'application/json'
        }
      }
    );

    let j = null;

    try{
      j = await r.json();
    }catch(_){}

    const slot =
      Array.isArray(j)
        ? j[0]
        : j;

    if(
      slot &&
      slot.error
    ){
      throw new Error(
        slot.error?.json?.message ||
        slot.error?.message ||
        'trpc error'
      );
    }

    return (
      slot?.result?.data?.json
      ??
      slot?.result?.data
      ??
      null
    );
  }

  async function rscFetch(path){
    const r = await fetch(
      location.origin + path,
      {
        credentials:'include',
        headers:{
          'RSC':'1',
          'Next-Router-Prefetch':'1',
          'accept':'*/*'
        }
      }
    );

    if(!r.ok){
      throw new Error(
        'http ' + r.status
      );
    }

    return await r.text();
  }

  // ---------- character data probe ----------
  async function trpcRaw(proc, input){
    const url =
      `${location.origin}/api/trpc/${proc}?batch=1&input=` +
      encodeURIComponent(
        JSON.stringify({
          0:trpcInput(input)
        })
      );

    const r = await fetch(
      url,
      {
        credentials:'include',
        headers:{
          'trpc-accept':'application/json',
          'content-type':'application/json'
        }
      }
    );

    const t = await r.text();

    return {
      status:r.status,
      body:t
    };
  }

  async function probeCharacterData(){
    closeMenus();

    setStatus(
      'probing…',
      'muted'
    );

    showPrompt(
      'Probing…',
      {
        ok:false,
        cancel:false
      }
    );

    const id =
      characterIdFromPage();

    const out = [];

    out.push(
      'URL: ' + location.href
    );

    out.push(
      'character id: ' +
      (
        id ||
        '(none found)'
      )
    );

    try{
      out.push(
        'your user id: ' +
        (
          await currentUserId()
          ||
          '(unknown)'
        )
      );
    }catch(e){
      out.push(
        'your user id: error ' +
        e.message
      );
    }

    out.push('');

    const PROCS = [
      [
        'character.getCharacterForNewChat',
        {characterId:id}
      ],
      [
        'character.getOriginalCharacterInfo',
        {characterId:id}
      ],
      [
        'character.getCharacter4Memory',
        {characterId:id}
      ],
      [
        'character.queryOwnCharacter',
        {characterId:id}
      ]
    ];

    for(let dp=0; dp<=12; dp++){
      PROCS.push(
        [
          'character.getCharacter',
          {
            characterId:id,
            displayPosition:dp
          }
        ]
      );
    }

    for(const [proc, input] of PROCS){
      updatePrompt(
        'Probing ' + proc + '…',
        {
          ok:false,
          cancel:false
        }
      );

      try{
        const {
          status,
          body
        } = await trpcRaw(
          proc,
          input
        );

        const tag =
          proc +
          ' ' +
          JSON.stringify(input);

        out.push(
          '--- ' +
          tag +
          ' -> HTTP ' +
          status +
          ' ---'
        );

        out.push(
          String(body)
            .slice(0, 20000)
        );

        out.push('');
      }catch(e){
        out.push(
          '--- ' +
          proc +
          ' -> threw: ' +
          e.message +
          ' ---'
        );

        out.push('');
      }
    }

    updatePrompt(
      'Merging…',
      {
        ok:false,
        cancel:false
      }
    );

    out.push(
      '########## merged character object ##########'
    );

    try{
      const found =
        await findCharacter();

      if(!found){
        out.push(
          '(findCharacter returned nothing)'
        );
      }else{
        out.push(
          'source: ' +
          found.source
        );

        const gate =
          await accessCheck(
            found.c
          );

        out.push(
          'access: ' +
          (
            gate.allowed
              ? 'ALLOWED'
              : 'BLOCKED'
          )
          +
          ' — ' +
          gate.why
        );

        out.push(
          'avatar url: ' +
          (
            avatarUrlOf(found.c)
            ||
            '(none)'
          )
        );

        out.push('');

        out.push(
          JSON.stringify(
            found.c,
            null,
            2
          ).slice(
            0,
            60000
          )
        );

        out.push('');

        out.push(
          '########## card that would be written ##########'
        );

        out.push(
          JSON.stringify(
            buildV2Card(
              found.c,
              {
                id,
                source:found.source
              }
            ),
            null,
            2
          ).slice(
            0,
            40000
          )
        );
      }
    }catch(e){
      out.push(
        'merge failed: ' +
        e.message
      );
    }

    const text =
      out.join('\n');

    const stamp =
      new Date()
        .toISOString()
        .replace(
          /[:.]/g,
          '-'
        );

    download(
      'crushon-probe4-' +
      stamp +
      '.txt',
      text,
      'text/plain'
    );

    try{
      await navigator.clipboard
        .writeText(text);
    }catch(_){}

    updatePrompt(
      'Probe saved as crushon-probe4-' +
      stamp +
      '.txt',
      {
        cancel:false,
        okText:'ok',
        onOK:hidePrompt
      }
    );

    setStatus(
      'probe done',
      'ok'
    );
  }

  // ---------- simple prompt helpers ----------
  function promptOKCancel(
    message,
    okText='ok',
    cancelText='Cancel'
  ){
    return new Promise(
      (resolve,reject)=>{
        showPrompt(
          message,
          {
            okText,
            cancelText,

            onOK:()=>{
              resolve(true);
            },

            onCancel:()=>{
              reject(
                new Error(
                  'cancel'
                )
              );
            }
          }
        );
      }
    );
  }

  function cancelAndClear(){
    clearCaptured(true);
    hidePrompt();
    setStatus(
      'canceled',
      'warn'
    );
  }

})();
