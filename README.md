# FQEDU
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>本週挑戰｜作答系統</title>
  <style>
    :root { --radius: 14px; --shadow: 0 10px 30px rgba(0,0,0,.08); }
    body { margin:0; padding:32px; font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,"Noto Sans TC","Microsoft JhengHei",sans-serif; background:#f6f7fb; color:#222; }
    .wrap { max-width: 960px; margin: 0 auto; }
    .card { background:#fff; border-radius:var(--radius); box-shadow:var(--shadow); padding:24px; }
    h1 { margin:0 0 8px; font-size:28px; }
    .sub { color:#667085; margin:0 0 20px; }
    label { display:block; font-weight:600; margin:10px 0 6px; }
    select, input { width:100%; padding:12px 14px; border:1px solid #e5e7eb; border-radius:10px; font-size:16px; }
    .row { display:grid; grid-template-columns: repeat(2,minmax(240px,1fr)); gap:12px; }
    .btn { display:inline-flex; align-items:center; justify-content:center; padding:12px 16px; border:0; border-radius:10px; font-weight:700; cursor:pointer; }
    .btn.primary { background:#111827; color:#fff; }
    .btn.secondary { background:#eef2f7; color:#111827; }
    .btn[disabled]{ opacity:.6; cursor:not-allowed; }
    .actions { display:flex; gap:10px; margin-top:14px; flex-wrap:wrap; }
    .hidden { display:none; }
    .qbox { padding:14px; border:1px solid #eef0f4; background:#fafbff; border-radius:10px; margin-top:8px; white-space: pre-wrap; }
    .grid-5 { display:grid; grid-template-columns: repeat(auto-fit, minmax(140px,1fr)); gap:12px; margin-top:12px; }
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
      <div>
        <label for="group">請填組別</label>
        <select id="group" required>
          <option value="" selected disabled>— 請選擇組別 —</option>
        </select>
      </div>
      <div>
        <label for="qno">本週挑戰｜請輸入第幾題（1–52）</label>
        <input id="qno" type="number" min="1" max="52" step="1" placeholder="例如：3" required />
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
    <div id="qbox" class="qbox" aria-live="polite">（載入中…）</div>

    <!-- 五市場 ± 選擇 -->
    <div class="grid-5">
      <div><label>股市</label><div id="optStock" class="radio-group"></div></div>
      <div><label>債市</label><div id="optBond" class="radio-group"></div></div>
      <div><label>美元指數</label><div id="optFx" class="radio-group"></div></div>
      <div><label>原物料</label><div id="optCom" class="radio-group"></div></div>
      <div><label>房地產</label><div id="optRe" class="radio-group"></div></div>
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
  // 後端 Apps Script Web App URL（你的最新部署）
  const GAS_ENDPOINT = 'https://script.google.com/macros/s/AKfycbyBhvNpUrTzfbWTgodkTuK05aDZnwY2v01sQPRBHZLP-Uf5lGnck7wxdDx1CTXExgIe/exec';
  const MAX_GROUP = 100, MAX_QNO = 52;

  const $ = id => document.getElementById(id);
  const setHidden = (id, b)=> $(id).classList.toggle('hidden', b);
  function toast(id, text, ok=true){ $(id).innerHTML = text ? `<div class="alert ${ok?'ok':'err'}">${text}</div>` : ''; }

  function buildChips(container, name){
    const opts=[{v:'+',t:'正面(＋)'},{v:'-',t:'負面(－)'}];
    container.innerHTML='';
    opts.forEach(o=>{
      const chip=document.createElement('label');
      chip.className='chip';
      chip.innerHTML=`<input type="radio" name="${name}" value="${o.v}"><span>${o.t}</span>`;
      chip.addEventListener('click',()=>{ 
        [...container.querySelectorAll('.chip')].forEach(c=>c.classList.remove('active'));
        chip.classList.add('active');
        chip.querySelector('input').checked=true;
      });
      container.appendChild(chip);
    });
  }
  const getChip = c => (c.querySelector('input:checked')||{}).value||'';
  const clearChips = c => { c.querySelectorAll('input').forEach(i=>i.checked=false); c.querySelectorAll('.chip').forEach(ch=>ch.classList.remove('active')); };

  // 初始化選單與選項
  (function init(){
    const sel=$('group');
    for(let i=1;i<=MAX_GROUP;i++){
      const o=document.createElement('option');
      o.value=String(i); o.textContent=`第 ${i} 組`; sel.appendChild(o);
    }
    buildChips($('optStock'),'stock');
    buildChips($('optBond'),'bond');
    buildChips($('optFx'),'fx');
    buildChips($('optCom'),'com');
    buildChips($('optRe'),'re');
  })();

  async function postJSON(url,payload){
    // 不帶 Content-Type，避免瀏覽器預檢
    const res = await fetch(url,{method:'POST',body:JSON.stringify(payload)});
    return await res.json();
  }

  // 送出並顯示題目
  $('btnFetch').addEventListener('click', async ()=>{
    toast('msg1','');
    const group=$('group').value;
    const qnoN=Number(($('qno').value||'').trim());
    if(!group) return toast('msg1','請先選擇組別。',false);
    if(!qnoN || qnoN<1 || qnoN>MAX_QNO) return toast('msg1','題號必須 1–52。',false);

    try{
      const data=await postJSON(GAS_ENDPOINT,{action:'requestQuestion',payload:{group,qno:qnoN}});
      if(!data.ok) throw new Error(data.message||'取得題目失敗');
      $('qbox').textContent=data.question?.text||`第 ${qnoN} 題`;
      $('badgeInfo').textContent=`第 ${group} 組｜第 ${qnoN} 題`;
      window.__ctx={group:String(group),qno:String(qnoN)};
      setHidden('step1',true); setHidden('step2',false);
      ['optStock','optBond','optFx','optCom','optRe'].forEach(id=>clearChips($(id)));
    }catch(err){ toast('msg1',err.message||'連線失敗',false); }
  });

  // 清除/返回
  $('btnReset1').addEventListener('click',()=>{ $('group').value=''; $('qno').value=''; toast('msg1',''); });
  $('btnBack').addEventListener('click',()=>{ setHidden('step2',true); setHidden('step1',false); toast('msg2',''); });

  // 送出作答
  $('btnSubmit').addEventListener('click', async ()=>{
    toast('msg2','');
    const stock=getChip($('optStock')), bond=getChip($('optBond')), fx=getChip($('optFx')), com=getChip($('optCom')), re=getChip($('optRe'));
    if(!stock||!bond||!fx||!com||!re) return toast('msg2','請完成五個市場的 ± 選擇。',false);

    try{
      const {group,qno}=window.__ctx||{};
      const payload={group,qno,stock,bond,fx,com,re};
      const data=await postJSON(GAS_ENDPOINT,{action:'submitAnswer',payload});
      if(!data.ok) throw new Error(data.message||'送出失敗');
      toast('msg2','已送出！感謝作答。',true);
    }catch(err){ toast('msg2',err.message||'連線失敗',false); }
  });
</script>
</body>
</html>
