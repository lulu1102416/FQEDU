# FQEDU

/** ========= 基本設定 ========= **/
const SPREADSHEET_ID = '1SPyCYVfxBrzOnB8RRtK7VOmZI8IXtr89b0feTf9Sd-I'; // 你的活頁簿 ID
const SHEET_NAME_ANS = 'AnswerKey';  // 題庫＋正解
const SHEET_NAME_RES = 'Responses';  // 學生作答
const SHEET_NAME_SCO = 'Scores';     // 加總排行

// Responses：紀錄五市場 + 總分
const RES_HEADERS = ['Timestamp','Group','QNO','Story','STOCK','BOND','FX','COM','RE','Score'];
const SCO_HEADERS = ['Timestamp','Group','QNO','Score'];

/** ========= 小工具 ========= **/
function jsonOutput(obj){ return ContentService.createTextOutput(JSON.stringify(obj)).setMimeType(ContentService.MimeType.JSON); }
function openSS_(){ return SpreadsheetApp.openById(SPREADSHEET_ID); }
function normKey_(k){ return String(k||'').trim().toUpperCase(); }

// 支援的欄位別名（大小寫與空白都會處理）
const HEADER_ALIASES = {
  QNO: ['QNO','題號','QUESTIONNO','Q_NO','NO'],
  QUESTIONTEXT: ['QUESTIONTEXT','題目','題幹','QUESTION','QUESTION_TEXT'],
  STOCK: ['STOCK','股市'],
  BOND:  ['BOND','債市'],
  FX:    ['FX','外匯','DXY'],
  COM:   ['COM','商品','COMMODITY'],
  RE:    ['RE','房地產','REAL ESTATE','REAL_ESTATE'],
};

/** 找出表頭列（往下掃最多 20 列，遇到含 QNO/題號 或 QuestionText/題目 的就當表頭） */
function detectHeaderRowIndex_(values){
  const MAX_SCAN = Math.min(values.length, 20);
  const allAlias = new Set([].concat(...Object.values(HEADER_ALIASES)).map(normKey_));
  for(let r=0;r<MAX_SCAN;r++){
    const row = values[r].map(normKey_);
    // 有任何一格是別名集合內，就視為表頭
    if(row.some(c => allAlias.has(c))) return r;
  }
  // 找不到就回 0（第 1 列）
  return 0;
}

/** 將實際表頭映射到標準鍵名 */
function buildHeaderMap_(headers){
  const normHeaders = headers.map(normKey_);
  const map = {};
  for(const [canon, aliases] of Object.entries(HEADER_ALIASES)){
    let idx = -1;
    for(const a of aliases){
      const i = normHeaders.indexOf(a);
      if(i !== -1){ idx = i; break; }
    }
    map[canon] = idx;
  }
  return map;
}

/** 讀取表：自動偵測表頭列，回 rows（用 canonical key） */
function readTableCanon_(sh){
  const v = sh.getDataRange().getValues();
  if(v.length < 2) return { rows: [], headerMap: {}, headerRowIndex: 0 };

  const headerRowIndex = detectHeaderRowIndex_(v);
  const headers = v[headerRowIndex];
  const headerMap = buildHeaderMap_(headers);

  // 從表頭下一列開始才是資料
  const dataRows = v.slice(headerRowIndex + 1);

  const rows = dataRows.map(row => {
    const o = {};
    for(const canon of Object.keys(HEADER_ALIASES)){
      const idx = headerMap[canon];
      o[canon] = (idx >= 0 ? row[idx] : '');
    }
    return o;
  });

  return { rows, headerMap, headerRowIndex };
}

/** 依 QNO 尋找（同時容忍文字/數字） */
function findByQno_(rows, qno){
  const wantStr = String(qno).trim();
  const wantNum = Number(wantStr);
  return rows.find(r => {
    const v = r.QNO;
    if(String(v).trim() === wantStr) return true;
    const n = Number(v);
    return Number.isFinite(n) && Number.isFinite(wantNum) && n === wantNum;
  });
}

/** ========= Web App 入口 ========= **/
function doGet(e){
  const action = e && e.parameter ? String(e.parameter.action || '').toLowerCase() : '';

  if(action === 'scores'){
    try{
      const rows = buildScoresRows_();
      return jsonOutput({ ok:true, rows });
    }catch(err){
      return jsonOutput({ ok:false, error:String(err && err.message ? err.message : err) });
    }
  }

  // 健康檢查
  return ContentService.createTextOutput('OK').setMimeType(ContentService.MimeType.TEXT);
}

// ⚠️ 前端不要手動加 Content-Type，避免瀏覽器預檢
function doPost(e){
  const out = ContentService.createTextOutput().setMimeType(ContentService.MimeType.JSON);
  try{
    const data = JSON.parse(e.postData.contents || '{}');
    const action = data.action;
    const p = data.payload || {};
    let result;

    if(action === 'requestQuestion'){
      result = handleRequestQuestion_(p);
    }else if(action === 'submitAnswer'){
      result = handleSubmitAnswer_(p);
    }else{
      throw new Error('Unknown action');
    }

    out.setContent(JSON.stringify({ ok:true, ...result }));
  }catch(err){
    out.setContent(JSON.stringify({ ok:false, message:String(err) }));
  }
  return out;
}

/** ========= 業務邏輯 ========= **/
function handleRequestQuestion_({ group, qno }){
  if(!group) throw new Error('缺少 group');
  if(!qno)   throw new Error('缺少 qno');

  const ss = openSS_();
  const sh = ss.getSheetByName(SHEET_NAME_ANS);
  if(!sh) throw new Error(`找不到工作表 ${SHEET_NAME_ANS}`);

  const { rows } = readTableCanon_(sh);
  const row = findByQno_(rows, qno);
  if(!row) throw new Error(`AnswerKey 中找不到題號 ${qno}`);

  const text = String(row.QUESTIONTEXT || '').trim() || `第 ${qno} 題`;
  return { question: { qno:String(qno), text } };
}

/** 五市場批改 */
function handleSubmitAnswer_({ group, qno, stock, bond, fx, com, re, story }){
  if(!group || !qno) throw new Error('缺少 group 或 qno');
  if(!stock || !bond || !fx || !com || !re) throw new Error('請完成五個市場的 ± 選擇');

  const ss = openSS_();
  const keySh = ss.getSheetByName(SHEET_NAME_ANS);
  if(!keySh) throw new Error(`找不到工作表 ${SHEET_NAME_ANS}`);

  const { rows } = readTableCanon_(keySh);
  const key = findByQno_(rows, qno);
  if(!key) throw new Error(`AnswerKey 中找不到題號 ${qno}`);

  const norm = v => String(v||'').trim().toUpperCase();
  const ans = { STOCK:norm(key.STOCK), BOND:norm(key.BOND), FX:norm(key.FX), COM:norm(key.COM), RE:norm(key.RE) };
  const user= { STOCK:norm(stock),     BOND:norm(bond),     FX:norm(fx),     COM:norm(com),     RE:norm(re) };

  let score = 0;
  Object.keys(ans).forEach(k => { if(ans[k] && user[k] && ans[k] === user[k]) score += 1; });

  const resSh = ensureSheet_(ss, SHEET_NAME_RES, RES_HEADERS);
  resSh.appendRow([
    new Date(), String(group), String(qno), String(story || ''),
    user.STOCK, user.BOND, user.FX, user.COM, user.RE, score
  ]);

  const scoSh = ensureSheet_(ss, SHEET_NAME_SCO, SCO_HEADERS);
  scoSh.appendRow([new Date(), String(group), String(qno), score]);

  return { message: 'submitted' };
}

/** ========= 排行榜：Scores 組別加總 ========= **/
function buildScoresRows_(){
  const ss = openSS_();
  const sh = ss.getSheetByName(SHEET_NAME_SCO);
  if(!sh) throw new Error(`找不到工作表 ${SHEET_NAME_SCO}`);

  const values = sh.getDataRange().getDisplayValues();
  if(values.length < 2) return [];

  const headers = values[0].map(h => (h || '').trim().toUpperCase());
  const idxGroup = headers.indexOf('GROUP');
  const idxScore = headers.indexOf('SCORE');
  if(idxGroup === -1 || idxScore === -1){
    throw new Error(`Scores 首列需包含 'Group' 與 'Score'，目前：${values[0].join(', ')}`);
  }

  const map = new Map();
  for(const row of values.slice(1)){
    const g = String(row[idxGroup] ?? '').trim();
    const n = Number(String(row[idxScore] ?? '').replace(/[, ]/g, ''));
    if(!g || !isFinite(n)) continue;
    map.set(g, (map.get(g) || 0) + n);
  }
  return Array.from(map.entries()).map(([group, score]) => ({ group, score }));
}

/** ========= 其他工具 ========= **/
function ensureSheet_(ss, name, headers){
  let sh = ss.getSheetByName(name);
  if(!sh){
    sh = ss.insertSheet(name);
    sh.appendRow(headers);
  }else if(sh.getLastRow() === 0){
    sh.appendRow(headers);
  }
  return sh;
}
