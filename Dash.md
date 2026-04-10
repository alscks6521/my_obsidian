```dataviewjs
const today = new Date();
const todayStr = today.toISOString().slice(0,10);

let selectedDate = todayStr;
let calRecords = {};
let memoText = "";

const STORAGE_KEY_REC  = "dashboard_calRecords";
const STORAGE_KEY_MEMO = "dashboard_memo";
try {
  const r = localStorage.getItem(STORAGE_KEY_REC);
  if (r) calRecords = JSON.parse(r);
  const m = localStorage.getItem(STORAGE_KEY_MEMO);
  if (m) memoText = m;
} catch(e) {}

function saveRecords() {
  try { localStorage.setItem(STORAGE_KEY_REC, JSON.stringify(calRecords)); } catch(e) {}
}
function saveMemo() {
  try { localStorage.setItem(STORAGE_KEY_MEMO, memoText); } catch(e) {}
}

// ── 날짜 기록 캘린더 ──────────────────────────────────
function renderRecordCal(container, year, month) {
  const firstDay = new Date(year, month, 1).getDay();
  const daysInMonth = new Date(year, month+1, 0).getDate();
  const days = ["일","월","화","수","목","금","토"];

  let html = `<div class="db-rec-cal">`;
  html += `<div class="db-cal-header db-rec-nav">`;
  html += `<button class="db-nav-btn" id="rec-prev">◀</button>`;
  html += `<span>${year}년 ${month+1}월</span>`;
  html += `<button class="db-nav-btn" id="rec-next">▶</button>`;
  html += `</div>`;
  html += `<div class="db-cal-grid">`;
  days.forEach(d => html += `<div class="db-cal-day-label">${d}</div>`);
  for (let i = 0; i < firstDay; i++) html += `<div></div>`;
  for (let d = 1; d <= daysInMonth; d++) {
    const ds = `${year}-${String(month+1).padStart(2,"0")}-${String(d).padStart(2,"0")}`;
    const isToday = ds === todayStr;
    const isSel   = ds === selectedDate;
    const hasRec  = !!calRecords[ds];
    html += `<div class="db-rec-cell${isToday?" db-today":""}${isSel?" db-selected":""}" data-date="${ds}">`;
    html += `${d}${hasRec ? '<span class="db-dot">●</span>' : ""}`;
    html += `</div>`;
  }
  html += `</div>`;
  const recVal = calRecords[selectedDate] || "";
  html += `<div class="db-rec-area">`;
  html += `<div class="db-rec-label">📅 ${selectedDate}</div>`;
  html += `<textarea id="rec-input" class="db-textarea" placeholder="이 날의 기록을 입력하세요...">${recVal}</textarea>`;
  html += `</div></div>`;
  container.innerHTML = html;

  container.querySelectorAll(".db-rec-cell").forEach(cell => {
    cell.addEventListener("click", () => {
      selectedDate = cell.dataset.date;
      renderRecordCal(container, year, month);
    });
  });
  container.querySelector("#rec-prev").addEventListener("click", () => {
    const d = new Date(year, month-1, 1);
    renderRecordCal(container, d.getFullYear(), d.getMonth());
  });
  container.querySelector("#rec-next").addEventListener("click", () => {
    const d = new Date(year, month+1, 1);
    renderRecordCal(container, d.getFullYear(), d.getMonth());
  });
  let recAutoSaveTimer = null;
  container.querySelector("#rec-input").addEventListener("input", (e) => {
    const val = e.target.value.trim();
    if (val) calRecords[selectedDate] = val;
    else delete calRecords[selectedDate];
    clearTimeout(recAutoSaveTimer);
    recAutoSaveTimer = setTimeout(() => saveRecords(), 500);
  });
}

// ── 메모장 ────────────────────────────────────────────
function renderMemo(container) {
  container.innerHTML = `
    <div class="db-memo">
      <div class="db-memo-header">메모</div>
      <textarea id="memo-input" class="db-textarea db-memo-ta" placeholder="고정 메모">${memoText}</textarea>
    </div>`;
  let autoSaveTimer = null;
  container.querySelector("#memo-input").addEventListener("input", (e) => {
    memoText = e.target.value;
    clearTimeout(autoSaveTimer);
    autoSaveTimer = setTimeout(() => saveMemo(), 500);
  });
}

// ── 하위 페이지 목록 ──────────────────────────────────
const STORAGE_KEY_PAGES = "dashboard_subpages";
let subPages = [];
try {
  const s = localStorage.getItem(STORAGE_KEY_PAGES);
  if (s) subPages = JSON.parse(s);
} catch(e) {}

function savePages() {
  try { localStorage.setItem(STORAGE_KEY_PAGES, JSON.stringify(subPages)); } catch(e) {}
}

function renderSubPages(container) {
  let html = `<div class="db-subpages">`;
  html += `<div class="db-memo-header">📁 하위 페이지</div>`;
  html += `<div class="db-page-add-row">`;
  html += `<input id="page-input" class="db-page-input" type="text" placeholder="페이지 이름 입력..." />`;
  html += `<button class="db-page-add-btn" id="page-add">+</button>`;
  html += `</div>`;
  html += `<ul class="db-page-list">`;
  if (subPages.length === 0) {
    html += `<li class="db-page-empty">추가된 페이지가 없습니다.</li>`;
  } else {
    subPages.forEach((name, idx) => {
      const path = `Dash/하위페이지/${name}.md`;
      html += `<li class="db-page-item">`;
      html += `<span class="db-page-link" data-path="${path}">📄 ${name}</span>`;
      html += `<button class="db-page-del" data-idx="${idx}" title="삭제">✕</button>`;
      html += `</li>`;
    });
  }
  html += `</ul></div>`;
  container.innerHTML = html;

  // 추가
  const addBtn = container.querySelector("#page-add");
  const input  = container.querySelector("#page-input");
  const SUB_FOLDER = "Dash/하위페이지";
  const doAdd  = async () => {
    const name = input.value.trim();
    if (!name) return;
    const filePath = `${SUB_FOLDER}/${name}.md`;
    if (!app.vault.getAbstractFileByPath(filePath)) {
      await app.vault.adapter.mkdir(SUB_FOLDER).catch(()=>{});
      await app.vault.create(filePath, `# ${name}\n`).catch(()=>{});
    }
    if (!subPages.includes(name)) { subPages.push(name); savePages(); }
    input.value = "";
    renderSubPages(container);
  };
  addBtn.addEventListener("click", doAdd);
  input.addEventListener("keydown", e => { if (e.key === "Enter") doAdd(); });

  container.querySelectorAll(".db-page-link").forEach(el => {
    el.addEventListener("click", () => {
      app.workspace.openLinkText(el.dataset.path, "", false);
    });
  });

  container.querySelectorAll(".db-page-del").forEach(btn => {
    btn.addEventListener("click", async () => {
      const idx = Number(btn.dataset.idx);
      const name = subPages[idx];
      const filePath = `Dash/하위페이지/${name}.md`;
      const file = app.vault.getAbstractFileByPath(filePath);
      if (file) await app.vault.delete(file);
      subPages.splice(idx, 1);
      savePages();
      renderSubPages(container);
    });
  });
}

// ── 그리드 생성 ───────────────────────────────────────
const root = dv.el("div", "", { cls: "db-root" });

const styles = `<style>
.db-root {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: repeat(2, auto);
  gap: 14px;
  padding: 12px;
}
.db-root-single {
  grid-template-columns: 1fr !important;
}
.db-root-single .db-card-wide {
  grid-column: span 1 !important;
}
.db-card {
  border: 1.5px solid var(--color-base-30);
  border-radius: 12px;
  padding: 14px;
  background: var(--color-base-10);
  min-height: 240px;
  width: 100%;
  box-sizing: border-box;
  overflow: auto;
}
.db-card-wide { grid-column: span 2; }
.db-cal-header {
  text-align: center;
  font-weight: 700;
  margin-bottom: 6px;
  font-size: 1em;
}
.db-rec-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.db-cal-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 2px;
  text-align: center;
}
.db-cal-day-label {
  font-weight: 600;
  color: var(--color-accent);
  font-size: 0.85em;
  padding: 2px 0;
}
.db-rec-cell {
  padding: 5px 1px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9em;
  position: relative;
}
.db-rec-cell:hover { background: var(--color-base-25); }
.db-today {
  background: var(--color-accent) !important;
  color: #fff !important;
  border-radius: 6px;
  font-weight: 700;
}
.db-selected {
  outline: 2px solid var(--color-accent);
  border-radius: 6px;
}
.db-dot {
  position: absolute;
  bottom: 2px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 0.45em;
  color: var(--color-accent);
  line-height: 1;
}
.db-rec-cal { font-size: 0.82em; }
.db-rec-area { margin-top: 10px; }
.db-rec-label { font-weight: 600; margin-bottom: 4px; font-size: 0.95em; }
.db-textarea {
  width: 100%;
  min-height: 70px;
  border: 1px solid var(--color-base-30);
  border-radius: 6px;
  background: var(--color-base-20);
  color: var(--text-normal);
  padding: 6px;
  font-size: 0.95em;
  resize: vertical;
  box-sizing: border-box;
}
.db-memo-ta { flex: 1; min-height: 0; resize: none; }
.db-save-btn {
  margin-top: 6px;
  padding: 4px 14px;
  background: var(--color-accent);
  color: #fff;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.88em;
}
.db-save-btn:hover { opacity: 0.85; }
.db-nav-btn {
  background: none;
  border: 1px solid var(--color-base-30);
  border-radius: 5px;
  cursor: pointer;
  color: var(--text-normal);
  padding: 1px 7px;
  font-size: 0.9em;
}
.db-memo { display: flex; flex-direction: column; height: 100%; }
.db-memo-header { font-weight: 700; margin-bottom: 8px; font-size: 1em; }
/* 하위 페이지 */
.db-subpages { display: flex; flex-direction: column; height: 100%; }
.db-page-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 4px;
  max-height: 240px;
  overflow-y: auto;
}
.db-page-item {
  padding: 7px 10px;
  border-radius: 7px;
  cursor: pointer;
  font-size: 0.9em;
  border: 1px solid var(--color-base-25);
  transition: background 0.15s;
  display: flex;
  align-items: center;
  gap: 6px;
}
.db-page-item:hover {
  background: var(--color-base-25);
  color: var(--color-accent);
}
.db-page-add-row {
  display: flex;
  gap: 6px;
  margin-bottom: 8px;
}
.db-page-input {
  flex: 1;
  border: 1px solid var(--color-base-30);
  border-radius: 6px;
  background: var(--color-base-20);
  color: var(--text-normal);
  padding: 4px 8px;
  font-size: 0.88em;
}
.db-page-add-btn {
  padding: 4px 10px;
  background: var(--color-accent);
  color: #fff;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.85em;
  white-space: nowrap;
}
.db-page-add-btn:hover { opacity: 0.85; }
.db-page-empty {
  font-size: 0.88em;
  color: var(--text-muted);
  padding: 6px 4px;
}
.db-page-link { flex: 1; }
.db-page-del {
  background: none;
  border: none;
  cursor: pointer;
  color: var(--text-muted);
  font-size: 0.8em;
  padding: 1 1px;
  line-height: 1;
  flex-shrink: 0;
}
.db-page-del:hover { color: var(--color-red); }
/* ── 카드 색상 ── */
.db-card-wide      { border-color: #a78bda !important; }
.db-card-subpages  { border-color: #ffb06a !important; }
.db-card-bookmarks { border-color: #6ab8de !important; }
.db-card-memo      { border-color: #f0d44a !important; }
/* 즐겨찾기 링크 */
.db-link-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.db-link-item {
  border: 1px solid var(--color-base-25);
  border-radius: 7px;
  transition: background 0.15s;
}
.db-link-item:hover { background: var(--color-base-25); }
.db-link-a {
  display: block;
  padding: 8px 12px;
  font-size: 0.9em;
  color: var(--text-normal);
  text-decoration: none;
  font-weight: 500;
}
.db-link-a:hover { color: var(--color-accent); }
</style>`;

root.innerHTML = styles;

// 반응형: ResizeObserver로 너비 감지
const ro = new ResizeObserver(entries => {
  for (const entry of entries) {
    const w = entry.contentRect.width;
    root.classList.toggle("db-root-single", w <= 900);
  }
});
ro.observe(root);

// 1번(wide): 날짜 기록 캘린더
const calCard = document.createElement("div");
calCard.className = "db-card db-card-wide";
root.appendChild(calCard);
renderRecordCal(calCard, today.getFullYear(), today.getMonth());

// 2번: 메모장
const memoCard = document.createElement("div");
memoCard.className = "db-card db-card-memo";
root.appendChild(memoCard);
renderMemo(memoCard);

// 3번: 하위 페이지 목록
const subCard = document.createElement("div");
subCard.className = "db-card db-card-subpages";
root.appendChild(subCard);
renderSubPages(subCard);

// 4번: 즐겨찾기 링크
const bookmarks = [
  { label: "🏢 포털 메인",        url: "http://portal.kiotcom.co.kr/portal/main/portalMain.do" },
  { label: "📋 개발 TASK",        url: "https://www.notion.so/2223cdec4891812491c7d919de4293ed" },
  { label: "🏃 개발 스프린트",    url: "https://www.notion.so/2233cdec48918046a528eb767751742c?v=2233cdec489180749eeb000c319aa38e" },
  { label: "📝 개발 릴리즈 노트", url: "https://www.notion.so/2233cdec4891805aa8c0da4d3ecc7c4a" },
  { label: "🦊 GitLab",           url: "http://git.kiotnas.synology.me/kiotservice/seeguard" },
];

const linkCard = document.createElement("div");
linkCard.className = "db-card db-card-bookmarks";
let linkHtml = `<div class="db-memo-header">🔗 즐겨찾기</div><ul class="db-link-list">`;
bookmarks.forEach(({ label, url }) => {
  linkHtml += `<li class="db-link-item"><a class="db-link-a" href="${url}" target="_blank">${label}</a></li>`;
});
linkHtml += `</ul>`;
linkCard.innerHTML = linkHtml;
root.appendChild(linkCard);

// 5번: 빈 카드
const emptyCard = document.createElement("div");
emptyCard.className = "db-card";
root.appendChild(emptyCard);
```