// build.js — Notion 데이터를 읽어 정적 HTML 생성
const { Client } = require('@notionhq/client');
const fs = require('fs');
const path = require('path');

const notion = new Client({ auth: process.env.NOTION_TOKEN });

// ── Notion DB ID (GitHub Secrets에서 주입) ──
const DB = {
  units:     process.env.UNITS_DB_ID,
  scores:    process.env.SCORES_DB_ID,
  materials: process.env.MATERIALS_DB_ID,
};

// ── 헬퍼: 속성 추출 ──
function getText(prop) {
  if (!prop) return '';
  if (prop.type === 'title')       return prop.title.map(t => t.plain_text).join('');
  if (prop.type === 'rich_text')   return prop.rich_text.map(t => t.plain_text).join('');
  if (prop.type === 'select')      return prop.select?.name || '';
  if (prop.type === 'number')      return prop.number ?? 0;
  if (prop.type === 'checkbox')    return prop.checkbox;
  if (prop.type === 'url')         return prop.url || '';
  return '';
}

// ── Notion DB 전체 조회 (페이지네이션 포함) ──
async function queryAll(dbId, filter) {
  if (!dbId) return [];
  const results = [];
  let cursor;
  do {
    const res = await notion.databases.query({
      database_id: dbId,
      filter,
      start_cursor: cursor,
      page_size: 100,
    });
    results.push(...res.results);
    cursor = res.has_more ? res.next_cursor : null;
  } while (cursor);
  return results;
}

// ── 데이터 가져오기 ──
async function fetchData() {
  console.log('📥 Notion에서 데이터 가져오는 중...');

  // 단원 (Units)
  const unitPages = await queryAll(DB.units);
  const units = { A: [], B: [] };
  for (const p of unitPages) {
    const pr = p.properties;
    const cls = getText(pr['반']) || 'A';
    const vocabRaw = getText(pr['단어목록']);   // "apple:사과,banana:바나나" 형식
    const sentRaw  = getText(pr['문장목록']);   // "I like you.\nShe is nice." 형식
    const vocab = vocabRaw.split(',').map(s => {
      const [en, ko] = s.split(':');
      return { en: (en||'').trim(), ko: (ko||'').trim() };
    }).filter(w => w.en);
    const sentences = sentRaw.split('\n').map(s => s.trim()).filter(Boolean);
    const unit = {
      name:      getText(pr['이름']),
      vocab,
      sentences,
      hw:        getText(pr['숙제여부']),
    };
    if (cls === 'B') units.B.push(unit);
    else             units.A.push(unit);
  }

  // 학생 점수 (Scores)
  const scorePages = await queryAll(DB.scores);
  const students = { A: [], B: [] };
  for (const p of scorePages) {
    const pr = p.properties;
    const cls = getText(pr['반']) || 'A';
    const student = {
      name:   getText(pr['이름']),
      emoji:  getText(pr['이모지']) || '😊',
      quiz:   Number(getText(pr['퀴즈']))   || 0,
      hw:     Number(getText(pr['숙제']))   || 0,
      exam:   Number(getText(pr['내신']))   || 0,
      attend: Number(getText(pr['근태']))   || 0,
    };
    if (cls === 'B') students.B.push(student);
    else             students.A.push(student);
  }

  // 보조교재 (Materials)
  const matPages = await queryAll(DB.materials);
  const materials = { mid3: [], mid2: [], audio: [] };
  for (const p of matPages) {
    const pr = p.properties;
    const cat  = getText(pr['카테고리']) || 'mid3'; // mid3 / mid2 / audio
    const type = getText(pr['종류'])     || 'pdf';
    const mat  = {
      name: getText(pr['이름']),
      type,
      url:  getText(pr['링크']) || '#',
    };
    if (materials[cat]) materials[cat].push(mat);
  }

  console.log(`✅ 단원: A${units.A.length}개 B${units.B.length}개 | 학생: A${students.A.length}명 B${students.B.length}명 | 교재: ${matPages.length}개`);
  return { units, students, materials };
}

// ── HTML 생성 ──
function buildHTML(data) {
  const dataJson = JSON.stringify(data);
  return `<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>📚 영어교실</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Jua&family=Nanum+Gothic:wght@400;700;800&family=Gaegu:wght@700&display=swap" rel="stylesheet">
<style>
:root{
  --primary:#6C63FF;--primary-light:#EEEEFF;--accent:#FF6584;
  --accent2:#43C6AC;--accent3:#FFB347;
  --bg:#F8F9FF;--white:#fff;--text:#2D2D3A;--muted:#8888AA;
  --border:#E8E8F2;--radius:18px;--shadow:0 4px 24px rgba(108,99,255,.10);
}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:'Nanum Gothic',sans-serif;background:var(--bg);color:var(--text);min-height:100vh}

/* HEADER */
.hd{background:var(--white);border-bottom:2px solid var(--border);
  padding:0 32px;display:flex;align-items:center;justify-content:space-between;
  height:64px;position:sticky;top:0;z-index:100;
  box-shadow:0 2px 12px rgba(108,99,255,.07)}
.logo{font-family:'Jua',sans-serif;font-size:1.5rem;color:var(--primary);
  display:flex;align-items:center;gap:8px}
.nav{display:flex;gap:4px;background:var(--bg);border-radius:50px;padding:4px}
.nav button{font-family:'Jua',sans-serif;font-size:.9rem;padding:7px 18px;
  border-radius:50px;border:none;background:transparent;color:var(--muted);cursor:pointer;transition:.2s}
.nav button.on{background:var(--primary);color:#fff;box-shadow:0 2px 10px rgba(108,99,255,.3)}
.nav button:hover:not(.on){background:var(--primary-light);color:var(--primary)}
.badge-sync{font-size:.72rem;color:var(--muted);display:flex;align-items:center;gap:4px}
.badge-sync span{color:var(--accent2);font-weight:700}

/* PAGES */
.pg{display:none}.pg.on{display:block}

/* HERO */
.hero{text-align:center;padding:52px 32px 28px}
.hero h1{font-family:'Jua',sans-serif;font-size:2.6rem;line-height:1.2;margin-bottom:10px}
.hero h1 em{color:var(--primary);font-style:normal}
.hero p{color:var(--muted);max-width:460px;margin:0 auto;line-height:1.7;font-size:1rem}

/* CLASS TABS */
.ctabs{display:flex;align-items:center;gap:8px;padding:0 32px 20px}
.ctab{font-family:'Jua',sans-serif;padding:8px 28px;border-radius:50px;
  border:2px solid var(--border);background:var(--white);color:var(--muted);cursor:pointer;transition:.2s}
.ctab.a-on{background:#6C63FF;color:#fff;border-color:#6C63FF}
.ctab.b-on{background:#FF6584;color:#fff;border-color:#FF6584}
.ctab:hover:not(.a-on):not(.b-on){border-color:var(--primary);color:var(--primary)}

/* FILTER */
.filters{display:flex;gap:8px;padding:0 32px 26px;flex-wrap:wrap}
.fpill{font-family:'Nanum Gothic',sans-serif;font-weight:700;font-size:.88rem;
  padding:6px 16px;border-radius:50px;border:2px solid var(--border);
  background:var(--white);color:var(--muted);cursor:pointer;transition:.2s}
.fpill.on{background:var(--primary);color:#fff;border-color:var(--primary)}
.fpill:hover:not(.on){border-color:var(--primary);color:var(--primary)}

/* SECTION TITLE */
.stitle{font-family:'Jua',sans-serif;font-size:1.3rem;padding:0 32px 16px}

/* CARDS GRID */
.grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(280px,1fr));
  gap:22px;padding:0 32px 48px}

/* UNIT CARD */
.card{background:var(--white);border-radius:var(--radius);box-shadow:var(--shadow);
  overflow:hidden;cursor:pointer;border:2px solid transparent;transition:.22s}
.card:hover{transform:translateY(-5px);box-shadow:0 12px 36px rgba(108,99,255,.18);border-color:var(--primary)}
.thumb{width:100%;height:150px;display:flex;align-items:center;justify-content:center;
  font-size:3.8rem;position:relative;overflow:hidden}
.thumb::after{content:'';position:absolute;inset:0;background:linear-gradient(180deg,transparent 55%,rgba(0,0,0,.08))}
.tbadges{position:absolute;bottom:10px;left:12px;display:flex;gap:5px;z-index:1}
.tb{font-family:'Jua',sans-serif;font-size:.72rem;padding:2px 10px;border-radius:50px;color:#fff}
.tb-v{background:var(--primary)}.tb-s{background:var(--accent2)}.tb-hw{background:var(--accent3);color:#333}
.cb{padding:14px 16px 16px}
.ctitle{font-family:'Jua',sans-serif;font-size:1.1rem;margin-bottom:7px;line-height:1.4}
.cmeta{display:flex;align-items:center;gap:7px;font-size:.8rem;color:var(--muted);margin-bottom:6px}
.cdot{width:3px;height:3px;border-radius:50%;background:var(--border)}
.cdesc{font-size:.85rem;color:var(--muted);line-height:1.6}

/* DETAIL OVERLAY */
.ov{display:none;position:fixed;inset:0;background:rgba(0,0,0,.45);z-index:200;
  align-items:center;justify-content:center}
.ov.open{display:flex}
.panel{background:var(--white);border-radius:var(--radius);width:92%;max-width:660px;
  max-height:88vh;overflow-y:auto;box-shadow:0 20px 60px rgba(0,0,0,.2);animation:up .25s ease}
@keyframes up{from{transform:translateY(28px);opacity:0}to{transform:translateY(0);opacity:1}}
.ph{display:flex;justify-content:space-between;align-items:center;
  padding:20px 24px 14px;border-bottom:2px solid var(--border);
  position:sticky;top:0;background:var(--white);z-index:1;
  border-radius:var(--radius) var(--radius) 0 0}
.phtitle{font-family:'Jua',sans-serif;font-size:1.2rem}
.pclose{width:32px;height:32px;border-radius:50%;border:none;background:var(--bg);
  cursor:pointer;font-size:1rem;display:flex;align-items:center;justify-content:center;transition:.2s}
.pclose:hover{background:var(--border)}
.acc-item{border-bottom:1px solid var(--border)}
.acc-btn{width:100%;background:none;border:none;padding:14px 24px;
  display:flex;justify-content:space-between;align-items:center;
  cursor:pointer;font-family:'Jua',sans-serif;font-size:.97rem;color:var(--text);transition:.15s}
.acc-btn:hover{background:var(--primary-light)}
.arr{transition:.25s;font-size:.78rem}
.acc-btn.open .arr{transform:rotate(180deg)}
.acc-body{display:none;padding:0 24px 16px;animation:fd .2s}
.acc-body.open{display:block}
@keyframes fd{from{opacity:0}to{opacity:1}}
.wlist{list-style:none;display:grid;grid-template-columns:1fr 1fr;gap:7px;margin-top:7px}
.witem{background:var(--bg);border-radius:10px;padding:8px 12px;
  display:flex;justify-content:space-between;align-items:center;font-size:.88rem}
.wen{font-weight:700;color:var(--primary)}.wko{color:var(--muted);font-size:.82rem}
.slist{list-style:none;display:flex;flex-direction:column;gap:7px;margin-top:7px}
.sitem{background:var(--bg);border-radius:10px;padding:10px 13px;font-size:.88rem;
  line-height:1.6;border-left:3px solid var(--accent2)}

/* SCORE PAGE */
.sgrid{display:grid;grid-template-columns:repeat(auto-fill,minmax(250px,1fr));gap:18px;padding:0 32px 48px}
.sc{background:var(--white);border-radius:var(--radius);padding:20px;
  box-shadow:var(--shadow);border:2px solid transparent;transition:.2s}
.sc:hover{transform:translateY(-3px);box-shadow:0 10px 28px rgba(108,99,255,.15)}
.sc-top{display:flex;align-items:center;gap:11px;margin-bottom:14px}
.av{width:46px;height:46px;border-radius:50%;background:var(--primary-light);
  display:flex;align-items:center;justify-content:center;font-size:1.5rem}
.sn{font-family:'Jua',sans-serif;font-size:1.05rem}
.st{font-size:.8rem;color:var(--muted);margin-top:2px}
.bars{display:flex;flex-direction:column;gap:7px}
.brow{display:flex;align-items:center;gap:7px;font-size:.8rem}
.blbl{width:68px;color:var(--muted);flex-shrink:0}
.btrack{flex:1;height:7px;background:var(--bg);border-radius:50px;overflow:hidden}
.bfill{height:100%;border-radius:50px;transition:width .6s cubic-bezier(.4,0,.2,1)}
.bq{background:var(--primary)}.bh{background:var(--accent2)}
.be{background:var(--accent3)}.ba{background:var(--accent)}
.bval{width:22px;text-align:right;font-weight:700;color:var(--text);font-size:.83rem}

/* MATERIALS */
.mgrid{display:grid;grid-template-columns:repeat(auto-fill,minmax(220px,1fr));gap:16px;padding:0 32px 48px}
.mc{background:var(--white);border-radius:var(--radius);padding:20px;
  box-shadow:var(--shadow);display:flex;flex-direction:column;gap:10px;
  border:2px solid transparent;transition:.2s;cursor:pointer}
.mc:hover{transform:translateY(-3px);box-shadow:0 10px 28px rgba(108,99,255,.15);border-color:var(--primary)}
.mc a{text-decoration:none;color:inherit}
.mi{font-size:2.2rem}.mt{font-family:'Jua',sans-serif;font-size:.97rem}.mtp{font-size:.78rem;color:var(--muted)}
.msub{display:flex;gap:7px;padding:0 32px 24px;flex-wrap:wrap}
.mst{font-family:'Jua',sans-serif;font-size:.92rem;padding:7px 20px;border-radius:10px;
  border:2px solid var(--border);background:var(--white);color:var(--muted);cursor:pointer;transition:.2s}
.mst.on{background:var(--primary);color:#fff;border-color:var(--primary)}
.mst:hover:not(.on){border-color:var(--primary);color:var(--primary)}

/* EMPTY */
.empty{grid-column:1/-1;text-align:center;padding:56px 32px;color:var(--muted)}
.empty .ei{font-size:3rem;margin-bottom:14px}
.empty p{font-family:'Jua',sans-serif;font-size:1.05rem}

/* NOTION BADGE */
.notion-badge{
  display:inline-flex;align-items:center;gap:6px;
  background:#fff;border:1.5px solid var(--border);border-radius:50px;
  padding:5px 14px 5px 10px;font-size:.78rem;color:var(--muted);
  text-decoration:none;transition:.2s;margin:0 32px 32px;
}
.notion-badge:hover{border-color:var(--primary);color:var(--primary)}
.notion-badge svg{width:16px;height:16px;flex-shrink:0}

@media(max-width:680px){
  .hd{padding:0 12px;height:auto;flex-wrap:wrap;gap:6px;padding:10px 12px}
  .nav button{padding:5px 11px;font-size:.8rem}
  .grid,.sgrid,.mgrid{grid-template-columns:1fr;padding:0 14px 32px}
  .hero,.stitle,.ctabs,.filters,.msub{padding-left:14px;padding-right:14px}
  .wlist{grid-template-columns:1fr}
  .notion-badge{margin:0 14px 24px}
}
</style>
</head>
<body>

<header class="hd">
  <div class="logo">📚 영어교실</div>
  <nav class="nav">
    <button class="on" onclick="gp('quiz',this)">퀴즈/단어</button>
    <button onclick="gp('mat',this)">보조교재</button>
    <button onclick="gp('score',this)">점수판</button>
  </nav>
  <div class="badge-sync">🔄 Notion 연동 · 마지막 업데이트 <span id="upd-time">방금</span></div>
</header>

<!-- ══ QUIZ PAGE ══ -->
<section class="pg on" id="pg-quiz">
  <div class="hero">
    <h1>단어 &amp; <em>퀴즈</em></h1>
    <p>Notion에서 관리되는 단원별 단어와 문장을 확인하세요</p>
  </div>
  <div class="ctabs">
    <span style="font-family:'Jua',sans-serif;color:var(--muted);margin-right:4px">반 선택:</span>
    <button class="ctab a-on" id="qtab-A" onclick="setQC('A')">A반</button>
    <button class="ctab" id="qtab-B" onclick="setQC('B')">B반</button>
  </div>
  <div class="filters">
    <button class="fpill on" onclick="setQF('all',this)">전체</button>
    <button class="fpill" onclick="setQF('vocab',this)">단어</button>
    <button class="fpill" onclick="setQF('sentence',this)">문장</button>
    <button class="fpill" onclick="setQF('hw',this)">숙제</button>
  </div>
  <p class="stitle">단원 목록</p>
  <div class="grid" id="qgrid"></div>
</section>

<!-- ══ MATERIALS PAGE ══ -->
<section class="pg" id="pg-mat">
  <div class="hero">
    <h1>📂 <em>보조교재</em></h1>
    <p>PDF 자료와 오디오 파일을 확인하세요</p>
  </div>
  <div class="msub">
    <button class="mst on" onclick="setMT('mid3',this)">중3 내신</button>
    <button class="mst" onclick="setMT('mid2',this)">중2 내신</button>
    <button class="mst" onclick="setMT('audio',this)">오디오</button>
  </div>
  <div class="mgrid" id="mgrid"></div>
</section>

<!-- ══ SCORE PAGE ══ -->
<section class="pg" id="pg-score">
  <div class="hero">
    <h1>🏆 <em>점수판</em></h1>
    <p>학생별 퀴즈, 숙제, 내신, 근태 점수를 확인하세요</p>
  </div>
  <div class="ctabs">
    <span style="font-family:'Jua',sans-serif;color:var(--muted);margin-right:4px">반 선택:</span>
    <button class="ctab a-on" id="stab-A" onclick="setSC('A')">A반</button>
    <button class="ctab" id="stab-B" onclick="setSC('B')">B반</button>
  </div>
  <div class="sgrid" id="sgrid"></div>
</section>

<!-- Detail overlay -->
<div class="ov" id="ov" onclick="closeOv(event)">
  <div class="panel" id="panel">
    <div class="ph">
      <span class="phtitle" id="ptitle">단원 상세</span>
      <button class="pclose" onclick="closePanel()">✕</button>
    </div>
    <div id="pbody"></div>
  </div>
</div>

<script>
// ── 데이터 주입 ──
const DATA = ${dataJson};

// ── 상태 ──
let qClass='A', sClass='A', qFilter='all', matTab='mid3';
const COLORS=['#EEF0FF','#FFF0F4','#EDFBF7','#FFF8EE','#F0EFFF','#FFFAEE'];
const EMOJIS=['📖','✏️','🔤','💬','📝','🗒️','📚','🌟'];
const ICONS={pdf:'📄',audio:'🎵',other:'📎'};
const CATS={quiz:'퀴즈',hw:'숙제',exam:'내신',attend:'근태'};
const BAR={quiz:'bq',hw:'bh',exam:'be',attend:'ba'};

// ── 페이지 전환 ──
function gp(id, btn) {
  document.querySelectorAll('.pg').forEach(p=>p.classList.remove('on'));
  document.querySelectorAll('.nav button').forEach(b=>b.classList.remove('on'));
  document.getElementById('pg-'+id).classList.add('on');
  btn.classList.add('on');
  if(id==='quiz') renderQ();
  if(id==='mat')  renderM();
  if(id==='score') renderS();
}

// ── QUIZ ──
function setQC(c){ qClass=c; document.getElementById('qtab-A').className='ctab'+(c==='A'?' a-on':''); document.getElementById('qtab-B').className='ctab'+(c==='B'?' b-on':''); renderQ(); }
function setQF(f, el){ qFilter=f; document.querySelectorAll('.fpill').forEach(p=>p.classList.remove('on')); el.classList.add('on'); renderQ(); }

function renderQ(){
  const units = DATA.units[qClass]||[];
  const filtered = units.filter(u=>{
    if(qFilter==='all') return true;
    if(qFilter==='vocab') return u.vocab?.length;
    if(qFilter==='sentence') return u.sentences?.length;
    if(qFilter==='hw') return u.hw;
    return true;
  });
  const g = document.getElementById('qgrid');
  if(!filtered.length){ g.innerHTML='<div class="empty"><div class="ei">📭</div><p>단원이 없습니다</p></div>'; return; }
  g.innerHTML = filtered.map((u,i)=>{
    const bg=COLORS[i%COLORS.length], em=EMOJIS[i%EMOJIS.length];
    const vc=u.vocab?.length||0, sc=u.sentences?.length||0;
    return \`<div class="card" onclick="openDetail(\${i})">
      <div class="thumb" style="background:\${bg}">\${em}
        <div class="tbadges">
          \${vc?'<span class="tb tb-v">단어 '+vc+'</span>':''}
          \${sc?'<span class="tb tb-s">문장 '+sc+'</span>':''}
          \${u.hw?'<span class="tb tb-hw">숙제</span>':''}
        </div>
      </div>
      <div class="cb">
        <div class="ctitle">\${u.name}</div>
        <div class="cmeta">
          <span class="tb \${qClass==='A'?'tb-v':'tb-s'}" style="font-size:.68rem">\${qClass}반</span>
          <span class="cdot"></span><span>\${vc+sc}개 항목</span>
        </div>
        <div class="cdesc">\${vc?'단어: '+u.vocab.slice(0,2).map(w=>w.en).join(', ')+(vc>2?'…':''):'문장 위주 단원'}</div>
      </div>
    </div>\`;
  }).join('');
}

function openDetail(i){
  const u = (DATA.units[qClass]||[])[i];
  if(!u) return;
  document.getElementById('ptitle').textContent = u.name;
  let html='';
  if(u.vocab?.length){
    html+=\`<div class="acc-item">
      <button class="acc-btn" onclick="toggleAcc(this)">📗 단어 목록 <span class="arr">▼</span></button>
      <div class="acc-body">
        <ul class="wlist">\${u.vocab.map(w=>'<li class="witem"><span class="wen">'+w.en+'</span><span class="wko">'+w.ko+'</span></li>').join('')}</ul>
      </div></div>\`;
  }
  if(u.sentences?.length){
    html+=\`<div class="acc-item">
      <button class="acc-btn" onclick="toggleAcc(this)">💬 문장 목록 <span class="arr">▼</span></button>
      <div class="acc-body">
        <ul class="slist">\${u.sentences.map(s=>'<li class="sitem">'+s+'</li>').join('')}</ul>
      </div></div>\`;
  }
  if(!html) html='<div style="padding:30px;text-align:center;color:var(--muted)">내용이 없습니다</div>';
  document.getElementById('pbody').innerHTML = html;
  document.getElementById('ov').classList.add('open');
}
function closePanel(){ document.getElementById('ov').classList.remove('open'); }
function closeOv(e){ if(e.target===document.getElementById('ov')) closePanel(); }
function toggleAcc(btn){ btn.classList.toggle('open'); btn.nextElementSibling.classList.toggle('open'); }

// ── MATERIALS ──
function setMT(t, el){ matTab=t; document.querySelectorAll('.mst').forEach(b=>b.classList.remove('on')); el.classList.add('on'); renderM(); }
function renderM(){
  const mats=DATA.materials[matTab]||[];
  const g=document.getElementById('mgrid');
  if(!mats.length){ g.innerHTML='<div class="empty"><div class="ei">📭</div><p>자료가 없습니다</p></div>'; return; }
  g.innerHTML=mats.map(m=>\`
    <div class="mc">
      <a href="\${m.url}" target="_blank" rel="noopener">
        <div class="mi">\${ICONS[m.type]||'📎'}</div>
        <div class="mt">\${m.name}</div>
        <div class="mtp">\${m.type.toUpperCase()}</div>
      </a>
    </div>\`).join('');
}

// ── SCORES ──
function setSC(c){ sClass=c; document.getElementById('stab-A').className='ctab'+(c==='A'?' a-on':''); document.getElementById('stab-B').className='ctab'+(c==='B'?' b-on':''); renderS(); }
function renderS(){
  const students=DATA.students[sClass]||[];
  const g=document.getElementById('sgrid');
  if(!students.length){ g.innerHTML='<div class="empty"><div class="ei">👥</div><p>학생 데이터가 없습니다</p></div>'; return; }
  g.innerHTML=students.map(s=>{
    const grand=s.quiz+s.hw+s.exam+s.attend;
    const max=Math.max(grand,40);
    return \`<div class="sc">
      <div class="sc-top">
        <div class="av">\${s.emoji}</div>
        <div><div class="sn">\${s.name}</div><div class="st">총 \${grand}점</div></div>
      </div>
      <div class="bars">
        \${Object.entries({quiz:s.quiz,hw:s.hw,exam:s.exam,attend:s.attend}).map(([cat,val])=>\`
        <div class="brow">
          <span class="blbl">\${CATS[cat]}</span>
          <div class="btrack"><div class="bfill \${BAR[cat]}" style="width:\${Math.min(val/max*100,100)}%"></div></div>
          <span class="bval">\${val}</span>
        </div>\`).join('')}
      </div>
    </div>\`;
  }).join('');
}

// ── 업데이트 시간 ──
const buildTime = new Date('${new Date().toISOString()}');
const diff = Math.round((Date.now()-buildTime)/60000);
document.getElementById('upd-time').textContent = diff<2?'방금':diff<60?diff+'분 전':Math.round(diff/60)+'시간 전';

// ── 초기 렌더 ──
renderQ(); renderM(); renderS();
</script>
</body>
</html>`;
}

// ── 메인 ──
async function main() {
  fs.mkdirSync('./dist', { recursive: true });
  const data = await fetchData();
  const html = buildHTML(data);
  fs.writeFileSync('./dist/index.html', html, 'utf8');
  console.log('🚀 dist/index.html 생성 완료!');
}

main().catch(err => { console.error('❌ 빌드 실패:', err); process.exit(1); });
