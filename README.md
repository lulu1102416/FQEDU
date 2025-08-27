# FQEDU

<html lang="zh-Hant">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>本週挑戰｜作答系統</title>
  <style>
    :root { --radius: 14px; --shadow: 0 10px 30px rgba(0,0,0,.08); }
    body { margin:0; padding:32px; font-family: -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,"Noto Sans TC","Microsoft JhengHei",sans-serif; background:#f6f7fb; color:#222; }
    .wrap { max-width: 860px; margin: 0 auto; }
    .card { background:#fff; border-radius:var(--radius); box-shadow:var(--shadow); padding:24px; }
    h1 { margin:0 0 8px; font-size:28px; }
    .sub { color:#667085; margin:0 0 20px; }
    label { display:block; font-weight:600; margin:10px 0 6px; }
    select, input, textarea {
      width:100%; padding:12px 14px; border:1px solid #e5e7eb; border-radius:10px; background:#fff; font-size:16px; outline:none;
    }
    textarea { min-height:120px; resize:vertical; }
    .row { display:flex; gap:12px; flex-wrap:wrap; }
    .col { flex:1 1 240px; }
    .btn { display:inline-flex; align-items:center; justify-content:center; gap:8px; padding:12px 16px; border:0; border-radius:10px; font-weight:700; cursor:pointer; }
    .btn.primary { background:#111827; color:#fff; }
    .btn.secondary { background:#eef2f7; color:#111827; }
    .btn[disabled]{ opacity:.6; cursor:not-allowed; }
    .actions { display:flex; gap:10px; margin-top:14px; }
    .hidden { display:none; }
    .qbox { padding:14px; border:1px solid #eef0f4; background:#fafbff; border-radius:10px; }
    .radio-group { display:flex; gap:8px; flex-wrap:wrap; }
    .chip { border:1px solid #d1d5db; background:#fff; border-radius:999px; padding:8px 14px; cursor:pointer; user-select:none; }
    .chip input { display:none; }
    .chip.active { border-color:#111827; box-shadow: inset 0 0 0 2px #111827; }
    .note { font-size:13px; color:#6b7280; margin-top:10px; }
    .alert { margin-top:10px; padding:12px 14px; border-radius:10px; }
    .alert.ok { background:#effaf1; border:1px solid #b7ebc6; color:#166534; }
    .alert.err { background:#fff1f0; border:1px solid #ffcdc7; color:#a8071a; }
    .badge { display:inline-block; padding:4px 10px; border-radius:999px; background:#eef2ff; color:#334155; font-weight:600; font-size:12px; }
  </style>
</head>
<body>
<div class="wrap">
  <!-- Step 1 -->
  <section id="step1" class="card">
    <h1>本週挑戰</h1>
    <p class="sub">請先選擇 <b>組別</b> 與輸入 <b>第幾題</b>，送出後才會顯示題目與作答區。</p>

    <div class="row">
      <div class="col">
        <label for="group">請填組別</label>
        <select id="group" required>
          <option value="" selected disabled>— 請選擇組別 —</option>
        </select>
      </div>
      <div class="col">
        <label for="qno">本週挑戰｜請輸入第幾題</label>
        <input id="qno" type="number" min="1" step="1" placeholder="例如：3" required />
      </div>
    </div>

    <div class="actions">
      <button id="btnFetch" class="btn primary">送出並顯示題目</button>
      <button id="btnReset1" class="btn secondary" type="button">清除</button>
    </div>
    <div id="msg1"></div>
  </section>

  <!-- Step 2 -->
  <section id="step2" class="card hidden" style="margin-top:18px;">
    <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:6px;">
      <span class="badge" id="badgeInfo">組別 − 題號</span>
    </div>

    <label>題目內容</label>
    <div id="qbox" class="qbox">（載入中…）</div>

    <label for="story" style="margin-top:12px;">請填答故事（觀點/理由/情境）</label>
    <textarea id="story" placeholder="請在此輸入你的說明…" ></textarea>

    <div class="row" style="margin-top:10px;">
      <div class="col">
        <label>美元指數（DXY）趨勢</label>
        <div id="optDxy" class="radio-group"></div>
      </div>
      <div class="col">
        <label>房地產趨勢</label>
        <div id="optRe" class="radio-group"></div>
      </div>
      <div class="col">
        <label>原物料趨勢</label>
        <div id="optCom" class="radio-group"></div>
      </div>
    </div>

    <div class="actions">
      <button id="btnSubmit" class="btn primary">送出作答</button>
      <button id="btnBack" class="btn secondary" type="button">返回上一步</button>
    </div>
    <div id="msg2"></div>

    <p class="note">系統於 <b>後端</b> 批改並記錄分數；前台不顯示正解與分數。</p>
  </section>
</div>

<script>
  // TODO：換成你部署出的 Apps Script Web App EXEC URL
  const GAS_ENDPOINT = 'https://script.google.com/macros/s/AKfycbwQ51qgWZ6zTSgHbNgVJOe-H7JXEdTtbadQONuTY0xJOs5b5BmoSmlqd3EBfruaJl5V/exec';
  const MAX_GROUP = 100;

  const $ = (id)=>document.getElementById(id);
  function setHidden(id, b){ $(id).classList.toggle('hidden', b); }
  function toast(id, text, ok=true){
    $(id).innerHTML = text ? `<div class="alert ${ok?'ok':'err'}">${text}</div>` : '';
  }

  // 建立 ± chips
  function buildChips(container, name){
    const opts=[{v:'+',t:'正面'},{v:'-',t:'負面'}];
    container.innerHTML='';
    opts.forEach(o=>{
      const chip=document.createElement('label');
      chip.className='chip';
      chip.innerHTML=`<input type="radio" name="${name}" value="${o.v}"><span>${o.t}</span>`;
      chip.addEventListener('click',()=>{
        [...container.querySelectorAll('.chip')].forEach(c=>c.classList.remove('active'));
        chip.classList.add('active');
      });
      container.appendChild(chip);
    });
  }
  function getChip(container){ const i=container.querySelector('input:checked'); return i?i.value:''; }
  function clearChips(container){
    [...container.querySelectorAll('input')].forEach(i=>i.checked=false);
    [...container.querySelectorAll('.chip')].forEach(c=>c.classList.remove('active'));
  }

  // 初始化 group 選單
  (function(){
    const sel=$('group');
    for(let i=1;i<=MAX_GROUP;i++){
      const o=document.createElement('option');
      o.value=String(i); o.textContent=`第 ${i} 組`; sel.appendChild(o);
    }
  })();
  buildChips($('optDxy'),'dxy'); buildChips($('optRe'),'re'); buildChips($('optCom'),'com');

  // Step1: 送出並顯示題目
  $('btnFetch').addEventListener('click', async ()=>{
    toast('msg1','');
    const group=$('group').value;
    const qno=($('qno').value||'').trim();
    if(!group) return toast('msg1','請先選擇組別。',false);
    if(!qno || Number(qno)<1) return toast('msg1','請輸入有效的題號（>=1）。',false);

    $('btnFetch').disabled=true;
    try{
      const res = await fetch(GAS_ENDPOINT, {
        method:'POST',
        // 重要：不要加 headers 的 Content-Type，避免預檢
        body: JSON.stringify({ action:'requestQuestion', payload:{ group, qno } })
      });
      const data = await res.json();
      if(!data.ok) throw new Error(data.message||'取得題目失敗');
      $('qbox').textContent = data.question?.text || `第 ${qno} 題`;
      $('badgeInfo').textContent = `第 ${group} 組｜第 ${qno} 題`;
      // 保存狀態
      window.__ctx = { group, qno };
      // 切換畫面
      setHidden('step1',true); setHidden('step2',false);
      $('story').value=''; clearChips($('optDxy')); clearChips($('optRe')); clearChips($('optCom'));
    }catch(err){
      toast('msg1', err.message || '連線失敗', false);
    }finally{ $('btnFetch').disabled=false; }
  });

  $('btnReset1').addEventListener('click', ()=>{ $('group').value=''; $('qno').value=''; toast('msg1',''); });

  $('btnBack').addEventListener('click', ()=>{ setHidden('step2',true); setHidden('step1',false); toast('msg2',''); });

  // Step2: 送出作答
  $('btnSubmit').addEventListener('click', async ()=>{
    toast('msg2','');
    const story=($('story').value||'').trim();
    const dxy=getChip($('optDxy')), re=getChip($('optRe')), com=getChip($('optCom'));
    if(!dxy || !re || !com) return toast('msg2','請完成三個市場的 ± 選擇。',false);

    $('btnSubmit').disabled=true;
    try{
      const {group,qno} = window.__ctx || {};
      const res = await fetch(GAS_ENDPOINT, {
        method:'POST',
        body: JSON.stringify({ action:'submitAnswer', payload:{ group,qno,story,dxy,re,com } })
      });
      const data = await res.json();
      if(!data.ok) throw new Error(data.message||'送出失敗');
      toast('msg2','已送出！感謝作答。',true);
    }catch(err){
      toast('msg2', err.message || '連線失敗', false);
    }finally{ $('btnSubmit').disabled=false; }
  });
</script>
</body>
</html>

