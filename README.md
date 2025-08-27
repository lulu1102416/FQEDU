# FQEDU
<!doctype html>
<html lang="zh-Hant">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>本週挑戰｜作答系統</title>
  <style>
    :root {
      --radius: 12px;
      --shadow: 0 6px 22px rgba(0,0,0,.08);
    }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Noto Sans TC", "PingFang TC", "Microsoft JhengHei", sans-serif;
      margin: 0; padding: 24px; background: #f7f7fb; color:#222;
    }
    .card {
      max-width: 720px; margin: 0 auto 20px; background: #fff;
      border-radius: var(--radius); box-shadow: var(--shadow); padding: 20px;
    }
    h1 { font-size: 22px; margin: 0 0 8px }
    p.hint { margin: 4px 0 16px; color:#666 }
    label { display:block; font-weight:600; margin:12px 0 6px }
    select, input[type="number"], textarea {
      width: 100%; padding: 12px 14px; border:1px solid #ddd; border-radius: 10px; font-size: 16px;
      background: #fff; outline: none;
    }
    select { max-height: 240px; overflow-y: auto; }
    textarea { min-height: 120px; resize: vertical; }
    .row { display:flex; gap:12px; flex-wrap: wrap; }
    .col { flex:1 1 220px; }
    .btn {
      display:inline-flex; align-items:center; justify-content:center; gap:8px;
      padding: 12px 16px; border:0; border-radius: 10px; cursor: pointer; font-weight:700;
      background:#222; color:#fff;
    }
    .btn.secondary { background:#f0f0f3; color:#222; }
    .btn[disabled] { opacity:.6; cursor:not-allowed }
    .actions { display:flex; gap:10px; margin-top: 16px; }
    .badge { display:inline-block; padding:4px 10px; border-radius: 999px; background:#eef2ff; color:#334; font-weight:600; font-size:12px }
    .qbox {
      padding:14px; border-radius: 10px; background:#fafafa; border:1px solid #eee; line-height:1.6;
    }
    .radio-group { display:flex; gap:8px; flex-wrap:wrap; }
    .chip {
      border:1px solid #ddd; border-radius: 999px; padding: 8px 12px; cursor: pointer; user-select:none;
      background:#fff;
    }
    .chip input { display:none }
    .chip.active { border-color:#222; box-shadow: inset 0 0 0 2px #222; }
    .footer-note { font-size:13px; color:#777; }
    .center { text-align:center }
    .hidden { display:none }
    .success {
      background:#effaf1; border:1px solid #b7ebc6; color:#146c2e; padding:12px 14px; border-radius:10px; margin-top:10px;
    }
    .error {
      background:#fff1f0; border:1px solid #ffccc7; color:#a8071a; padding:12px 14px; border-radius:10px; margin-top:10px;
    }
  </style>
</head>
<body>

  <!-- Step 1 -->
  <section id="step1" class="card">
    <h1>本週挑戰</h1>
    <p class="hint">請先選擇 <b>組別</b> 與輸入 <b>第幾題</b>，送出後才會顯示題目與作答區。</p>

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
      <button id="btnFetch" class="btn">送出並顯示題目</button>
      <button id="btnReset1" class="btn secondary" type="button">清除</button>
    </div>
    <div id="msg1"></div>
  </section>

  <!-- Step 2 -->
  <section id="step2" class="card hidden">
    <div class="row" style="align-items:center; justify-content:space-between">
      <div><span class="badge" id="badgeInfo">組別 − 題號</span></div>
    </div>

    <label>題目內容</label>
    <div id="qbox" class="qbox">（載入中…）</div>

    <label for="story">請填答故事（可簡述觀點、理由、情境）</label>
    <textarea id="story" placeholder="請在此輸入你的故事或說明…" ></textarea>

    <div class="row">
      <div class="col">
        <label>美元指數（DXY）趨勢</label>
        <div id="optDxy" class="radio-group" role="radiogroup" aria-label="DXY">
          <!-- radio chips injected by JS -->
        </div>
      </div>
      <div class="col">
        <label>房地產趨勢</label>
        <div id="optRe" class="radio-group" role="radiogroup" aria-label="RE">
        </div>
      </div>
      <div class="col">
        <label>原物料趨勢</label>
        <div id="optCom" class="radio-group" role="radiogroup" aria-label="COM">
        </div>
      </div>
    </div>

    <div class="actions">
      <button id="btnSubmit" class="btn">送出作答</button>
      <button id="btnBack" class="btn secondary" type="button">返回上一步</button>
    </div>
    <div id="msg2"></div>

    <p class="footer-note">
      系統會在 <b>後端</b> 完成批改並記錄成績；前台不會顯示正解以避免外流。
    </p>
  </section>

<script>
  // 1) 把你的 Google Apps Script Web App URL 貼在這裡
  const GAS_ENDPOINT = 'https://script.google.com/macros/s/REPLACE_WITH_YOUR_DEPLOYED_URL/exec';

  // 2) 可調整的組別數量（例如 1~100 組）
  const MAX_GROUP = 100;

  // =============== 小工具 ===============
  function el(id) { return document.getElementById(id); }
  function setHidden(id, hidden) { el(id).classList.toggle('hidden', hidden); }
  function msg(containerId, text, kind='success') {
    const box = el(containerId);
    box.innerHTML = text ? `<div class="${kind}">${text}</div>` : '';
  }
  function buildChips(container, name) {
    const opts = [
      { value: '+', label: '正面' },
      { value: '-', label: '負面' },
    ];
    container.innerHTML = '';
    opts.forEach(o => {
      const chip = document.createElement('label');
      chip.className = 'chip';
      chip.innerHTML = `<input type="radio" name="${name}" value="${o.value}"><span>${o.label}</span>`;
      chip.addEventListener('click', () => {
        // 樣式切換
        [...container.querySelectorAll('.chip')].forEach(c => c.classList.remove('active'));
        chip.classList.add('active');
      });
      container.appendChild(chip);
    });
  }
  function getChipValue(container) {
    const input = container.querySelector('input[type="radio"]:checked');
    return input ? input.value : '';
  }
  function clearChips(container) {
    [...container.querySelectorAll('input[type="radio"]')].forEach(i => i.checked=false);
    [...container.querySelectorAll('.chip')].forEach(c => c.classList.remove('active'));
  }

  // =============== 初始化 UI ===============
  // 填充組別下拉 (1~MAX_GROUP)
  (function initGroups() {
    const sel = el('group');
    for (let i=1; i<=MAX_GROUP; i++) {
      const opt = document.createElement('option');
      opt.value = String(i);
      opt.textContent = `第 ${i} 組`;
      sel.appendChild(opt);
    }
  })();

  // 建立三組 ± chips
  buildChips(el('optDxy'), 'dxy');
  buildChips(el('optRe'),  're');
  buildChips(el('optCom'), 'com');

  // =============== 狀態記錄 ===============
  let current = { group:'', qno:'', qtext:'' };

  // =============== 事件綁定 ===============
  el('btnFetch').addEventListener('click', async () => {
    msg('msg1','');
    const group = el('group').value;
    const qno = el('qno').value.trim();
    if (!group) return msg('msg1','請先選擇組別。','error');
    if (!qno || Number(qno) < 1) return msg('msg1','請輸入有效的題號（>= 1）。','error');

    el('btnFetch').disabled = true;

    try {
      const res = await fetch(GAS_ENDPOINT, {
        method: 'POST',
        headers: {'Content-Type':'application/json'},
        body: JSON.stringify({
          action: 'requestQuestion',
          payload: { group, qno }
        })
      });
      const data = await res.json();
      if (!data.ok) throw new Error(data.message || '取得題目失敗');
      // 顯示題目
      current = { group, qno, qtext: data.question?.text || `第 ${qno} 題（題幹）` };
      el('qbox').textContent = current.qtext;
      el('badgeInfo').textContent = `第 ${group} 組｜第 ${qno} 題`;
      // 切換到第 2 步
      setHidden('step1', true);
      setHidden('step2', false);
      // 清空作答
      el('story').value = '';
      clearChips(el('optDxy'));
      clearChips(el('optRe'));
      clearChips(el('optCom'));
    } catch (err) {
      console.error(err);
      msg('msg1', err.message || '伺服器連線失敗','error');
    } finally {
      el('btnFetch').disabled = false;
    }
  });

  el('btnReset1').addEventListener('click', () => {
    el('group').value = '';
    el('qno').value = '';
    msg('msg1','');
  });

  el('btnBack').addEventListener('click', () => {
    // 返回上一步
    setHidden('step2', true);
    setHidden('step1', false);
    msg('msg2','');
  });

  el('btnSubmit').addEventListener('click', async () => {
    msg('msg2','');
    const story = el('story').value.trim();
    const dxy = getChipValue(el('optDxy'));
    const re  = getChipValue(el('optRe'));
    const com = getChipValue(el('optCom'));

    if (!dxy || !re || !com) {
      return msg('msg2','請完成三個市場的 ± 選擇。','error');
    }

    el('btnSubmit').disabled = true;
    try {
      const res = await fetch(GAS_ENDPOINT, {
        method: 'POST',
        headers: {'Content-Type':'application/json'},
        body: JSON.stringify({
          action: 'submitAnswer',
          payload: {
            group: current.group,
            qno: current.qno,
            story, dxy, re, com
          }
        })
      });
      const data = await res.json();
      if (!data.ok) throw new Error(data.message || '送出失敗');

      msg('msg2','已送出！感謝作答（成績由後端批改，不在前台顯示）。','success');
      // 可在此重置或導回步驟一
      // setTimeout(() => { location.reload(); }, 800);
    } catch (err) {
      console.error(err);
      msg('msg2', err.message || '伺服器連線失敗','error');
    } finally {
      el('btnSubmit').disabled = false;
    }
  });
</script>
</body>
</html>
