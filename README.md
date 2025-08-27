# FQEDU

/** ========= 基本設定 ========= **/
const SPREADSHEET_ID = '1SPyCYVfxBrzOnB8RRtK7VOmZI8IXtr89b0feTf9Sd-I'; // 你的活頁簿 ID
const SHEET_NAME_ANS = 'AnswerKey';  // 題庫＋正解（後端專用）
const SHEET_NAME_RES = 'Responses';  // 學生作答
const SHEET_NAME_SCO = 'Scores';     // 批改分數（排行榜來源）

/** 回答表頭（已擴充成五市場 + 總分） */
const RES_HEADERS = ['Timestamp','Group','QNO','Story','STOCK','BOND','FX','COM','RE','Score'];
const SCO_HEADERS = ['Timestamp','Group','QNO','Score'];

/** ========= 小工具 ========= **/
function jsonOutput(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
function openSS_() { return SpreadsheetApp.openById(SPREADSHEET_ID); }

/** 讀表工具（強韌版）
 * - 自動正規化表頭：trim、轉大寫
 * - 支援同義欄位（QNO / 題號、QUESTIONTEXT / 題目 / 題幹）
 * - QNO 比對同時支援「字串」與「數字」相等（1 == "1"）
 */
const HEADER_ALIASES = {
  QNO: ['QNO', '題號', 'questionno', 'q_no', 'no'],
  QUESTIONTEXT: ['QUESTIONTEXT', '題目', '題幹', 'question', 'question_text'],
  STOCK: ['STOCK', '股市'],
  BOND:  ['BOND', '債市'],
  FX:    ['FX', '外匯', 'DXY'],
  COM:   ['COM', '商品', 'COMMODITY'],
  RE:    ['RE', '房地產', 'REAL ESTATE', 'REAL_ESTATE'],
};

function normKey_(k) { return String(k || '').trim().toUpperCase(); }

/** 將實際表頭映射成標準鍵名 */
function buildHeaderMap_(headers) {
  const normHeaders = headers.map(normKey_);
  const map = {}; // { canonicalKey: index }
  for (const [canon, aliases] of Object.entries(HEADER_ALIASES)) {
    let idx = -1;
    for (const a of aliases) {
      const i = normHeaders.indexOf(a);
      if (i !== -1) { idx = i; break; }
    }
    map[canon] = idx; // 若為 -1 代表缺欄
  }
  return map;
}

/** 讀取整張表，回傳物件陣列（鍵名為 canonical key） */
function readTableCanon_(sh) {
  const v = sh.getDataRange().getValues();
  if (v.length < 2) return { rows: [], headerMap: {} };

  const headers = v[0];
  const headerMap = buildHeaderMap_(headers);

  const rows = v.slice(1).map(row => {
    const obj = {};
    for (const canon of Object.keys(HEADER_ALIASES)) {
      const idx = headerMap[canon];
      obj[canon] = (idx >= 0 ? row[idx] : '');
    }
    return obj;
  });
  return { rows, headerMap };
}

/** 以 QNO 查找（容忍數字/字串差異） */
function findByQno_(rows, qno) {
  const wantStr = String(qno).trim();
  const wantNum = Number(wantStr);
  return rows.find(r => {
    const v = r.QNO;
    // 同時檢查字串相等與數字相等
    if (String(v).trim() === wantStr) return true;
    const n = Number(v);
    return Number.isFinite(n) && Number.isFinite(wantNum) && n === wantNum;
  });
}

/** ========= Web App 入口 ========= **/
// GET：健康檢查；若帶 action=scores 則輸出排行榜 JSON
function doGet(e) {
  const action = e && e.parameter ? String(e.parameter.action || '').toLowerCase() : '';

  if (action === 'scores') {
    try {
      const rows = buildScoresRows_(); // ➜ [{ group, score }]
      return jsonOutput({ ok: true, rows });
    } catch (err) {
      return jsonOutput({ ok: false, error: String(err && err.message ? err.message : err) });
    }
  }

  // 默認健康檢查
  return ContentService.createTextOutput('OK').setMimeType(ContentService.MimeType.TEXT);
}

// POST：requestQuestion / submitAnswer
// ⚠️ 不要在前端手動加 Content-Type，避免瀏覽器送 OPTIONS 預檢
function doPost(e) {
  const out = ContentService.createTextOutput().setMimeType(ContentService.MimeType.JSON);
  try {
    const data = JSON.parse(e.postData.contents || '{}');
    const action = data.action;
    const p = data.payload || {};
    let result;

    if (action === 'requestQuestion') {
      result = handleRequestQuestion_(p);
    } else if (action === 'submitAnswer') {
      result = handleSubmitAnswer_(p);
    } else {
      throw new Error('Unknown action');
    }

    out.setContent(JSON.stringify({ ok: true, ...result }));
  } catch (err) {
    out.setContent(JSON.stringify({ ok: false, message: String(err) }));
  }
  return out;
}

/** ========= 業務邏輯 ========= **/
function handleRequestQuestion_({ group, qno }) {
  if (!group) throw new Error('缺少 group');
  if (!qno)   throw new Error('缺少 qno');

  const ss = openSS_();
  const sh = ss.getSheetByName(SHEET_NAME_ANS);
  if (!sh) throw new Error(`找不到工作表 ${SHEET_NAME_ANS}`);

  const { rows } = readTableCanon_(sh);
  const row = findByQno_(rows, qno);
  if (!row) throw new Error(`AnswerKey 中找不到題號 ${qno}`);

  // 題幹：有 QuestionText 用 QuestionText，否則 fallback
  const text = String(row.QUESTIONTEXT || '').trim() || `第 ${qno} 題`;

  return {
    question: { qno: String(qno), text }
  };
}

/** 五市場批改版（STOCK / BOND / FX / COM / RE 都要 + 或 -） */
function handleSubmitAnswer_({ group, qno, stock, bond, fx, com, re, story }) {
  if (!group || !qno) throw new Error('缺少 group 或 qno');
  if (!stock || !bond || !fx || !com || !re) throw new Error('請完成五個市場的 ± 選擇');

  const ss = openSS_();

  // 讀取正解
  const keySh = ss.getSheetByName(SHEET_NAME_ANS);
  if (!keySh) throw new Error(`找不到工作表 ${SHEET_NAME_ANS}`);

  const { rows } = readTableCanon_(keySh);
  const key = findByQno_(rows, qno);
  if (!key) throw new Error(`AnswerKey 中找不到題號 ${qno}`);

  function norm(v){ return String(v || '').trim().toUpperCase(); }
  const ans = {
    STOCK: norm(key.STOCK),
    BOND:  norm(key.BOND),
    FX:    norm(key.FX),
    COM:   norm(key.COM),
    RE:    norm(key.RE),
  };
  const user = {
    STOCK: norm(stock),
    BOND:  norm(bond),
    FX:    norm(fx),
    COM:   norm(com),
    RE:    norm(re),
  };

  let score = 0;
  Object.keys(ans).forEach(k => { if (ans[k] && user[k] && ans[k] === user[k]) score += 1; });

  // Responses：寫入五市場 + 總分
  const resSh = ensureSheet_(ss, SHEET_NAME_RES, RES_HEADERS);
  resSh.appendRow([
    new Date(), String(group), String(qno), String(story || ''),
    user.STOCK, user.BOND, user.FX, user.COM, user.RE, score
  ]);

  // Score

