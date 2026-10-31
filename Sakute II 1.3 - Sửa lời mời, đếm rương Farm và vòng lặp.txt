// ==UserScript==
// @name         Sakute II 1.3.1
// @namespace    hh3d.me-cung.automation
// @version      1.3.1-min-players-conditional-kick-settings
// @description  1.3.1: Đưa số người tối thiểu ra ngoài Setting; ẩn điều kiện trục xuất và thiết lập phụ khi chưa bật
// @include      https://hoathinh3d.*/me-cung*
// @run-at       document-idle
// @grant        none
// ==/UserScript==

(() => {
  'use strict';
  if (window.__HH3D_ME_CUNG_AUTO__) return;
  window.__HH3D_ME_CUNG_AUTO__ = true;

  const $ = (s, root = document) => root.querySelector(s);
  const $$ = (s, root = document) => [...root.querySelectorAll(s)];
  const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));
  // Luôn lấy origin của tab chính (trang người dùng đang mở), tránh nhầm origin nếu Mê Cung chạy trong iframe.
  const SITE_ORIGIN = (() => { try { return window.top.location.origin; } catch (_) { return window.location.origin; } })();
  const norm = s => (s || '').normalize('NFC').replace(/\s+/g, ' ').trim().toLowerCase();
  const visible = el => !!el && !!(el.offsetWidth || el.offsetHeight || el.getClientRects().length);
  const textOf = el => norm(el?.innerText || el?.textContent || '');
  const words = {
    difficulty: ['thường', 'khó', 'ác mộng'],
    elements: ['kim', 'mộc', 'thủy', 'hỏa', 'thổ']
  };

  const ROOM_SETTINGS_KEY = 'hh3d_maze_config_v2';
  const PROTECTED_IDS_KEY = 'hh3d_maze_protected_ids_v1';
  let protectedPlayers = [];
  try { protectedPlayers = JSON.parse(localStorage.getItem(PROTECTED_IDS_KEY) || '[]'); if (!Array.isArray(protectedPlayers)) protectedPlayers = []; } catch (_) { protectedPlayers = []; }
  function saveProtectedPlayers() { try { localStorage.setItem(PROTECTED_IDS_KEY, JSON.stringify(protectedPlayers)); } catch (_) {} }
  function rowPlayerIdentity(row) {
    const nodes = [row, ...$$('[data-user-id],[data-uid],[data-id], [data-userid]', row)];
    let id = '';
    for (const el of nodes) { id = el.getAttribute('data-user-id') || el.getAttribute('data-uid') || el.getAttribute('data-userid') || ''; if (!id && el.getAttribute('data-id') && /^\d+$/.test(el.getAttribute('data-id'))) id = el.getAttribute('data-id'); if (id) break; }
    const name = String(row?.innerText || row?.textContent || '').split('\n').map(x=>x.trim()).find(Boolean) || '';
    return { id: String(id || '').trim(), name };
  }
  function isSupportRow(row) {
    const t = norm(row?.innerText || row?.textContent || '');
    return /trợ\s*chiến|tro\s*chien/.test(t);
  }
  function isProtectedRow(row) {
    if (!row) return false;
    // Thành viên có nhãn Trợ Chiến luôn được xem như đã khóa ID.
    if (isSupportRow(row)) return true;
    const identity = rowPlayerIdentity(row);
    // Trường hợp trang có ID trực tiếp trong DOM: so sánh ID tuyệt đối.
    if (identity.id && protectedPlayers.some(x => String(x.id).trim() === identity.id)) return true;

    // Một số hàng thành viên không chứa data-user-id. Khi đó ánh xạ tên hiển thị
    // của hàng sang user-status đã bắt được từ request thành công của trang.
    const lines = String(row.innerText || row.textContent || '')
      .split(/\n|\r/).map(x => norm(x)).filter(Boolean);
    const namesInRow = new Set(lines);
    for (const member of (typeof cachedUserStatusMembers !== 'undefined' ? cachedUserStatusMembers : [])) {
      if (!member?.id || !member?.name) continue;
      if (!namesInRow.has(norm(member.name))) continue;
      if (protectedPlayers.some(x => String(x.id).trim() === String(member.id).trim())) return true;
    }
    // Dự phòng nếu tên nằm trong một dòng kèm thông tin trạng thái.
    const rowText = norm(row.innerText || row.textContent || '');
    for (const member of (typeof cachedUserStatusMembers !== 'undefined' ? cachedUserStatusMembers : [])) {
      if (!member?.id || !member?.name || !protectedPlayers.some(x => String(x.id).trim() === String(member.id).trim())) continue;
      const n = norm(member.name);
      if (n && rowText.includes(n)) return true;
    }
    return false;
  }
  const RUN_PROGRESS_KEY = 'hh3d_maze_run_progress_v1';
  const INVITE_SENT_KEY = 'hh3d_maze_invite_last_event_v1';
  const USER_OPTIONS_KEY = 'hh3d_maze_user_options_v1';
  const PLAN_KEY = 'hh3d_maze_stage_plan_v1';
  const userOptions = { autoInvite: true, surrenderKimNightmare: true };
  try { Object.assign(userOptions, JSON.parse(localStorage.getItem(USER_OPTIONS_KEY) || '{}')); } catch (_) {}
  userOptions.autoInvite = userOptions.autoInvite !== false;
  // Bật mặc định: gặp boss Kim ở độ khó Ác Mộng thì tự đầu hàng.
  userOptions.surrenderKimNightmare = userOptions.surrenderKimNightmare !== false;
  function saveUserOptions() { try { localStorage.setItem(USER_OPTIONS_KEY, JSON.stringify(userOptions)); } catch (_) {} }
  const INVITE_EVENT_COOLDOWN_MS = 8000;
  const INVITE_INTERVAL_MS = 10 * 60 * 1000;
  const ROOM_RELOCK_INTERVAL_MS = 30 * 60 * 1000;
  let underfilledLobbySince = 0;
  let lastUnderfilledInviteAt = 0;
  let lastInviteRosterFingerprint = '';
  let inviteSendInFlight = false;
  const CHAT_THREAD_ID = 22239;
  const ROOM_DEFAULTS = { requiredPlayers: 5, autoKickEnabled: true, autoKickIdle: true, autoKickUnlockedIds: false, idleKickSeconds: 5 };
  let roomSettings = { ...ROOM_DEFAULTS };
  try {
    const savedRoom = JSON.parse(localStorage.getItem(ROOM_SETTINGS_KEY) || '{}');
    roomSettings = { ...ROOM_DEFAULTS, ...savedRoom };
    roomSettings.requiredPlayers = clampInt(roomSettings.requiredPlayers, 1, 5, 5);
    roomSettings.idleKickSeconds = clampInt(roomSettings.idleKickSeconds, 1, 60, 5);
    roomSettings.autoKickEnabled = roomSettings.autoKickEnabled !== false;
    roomSettings.autoKickIdle = roomSettings.autoKickIdle !== false;
    roomSettings.autoKickUnlockedIds = !!roomSettings.autoKickUnlockedIds;
  } catch (_) {}
  const idleKickTracker = new WeakMap();
  const ROOM_PLAYER_SELECTORS = [
    '.room-member','.member-item','.player-item','.room-user','.user-item',
    '.lobby-player','.player-card','.char-card','.member','.room-players .item',
    '.players .item','.player-list .item','.member-list .member','.user-list .user'
  ];
  const ROOM_COUNT_SELECTORS = [
    '#room-player-count','#player-count','.room-player-count','.player-count',
    '.room-member-count','.member-count','.room-count','.room-user-count'
  ];
  function clampInt(value, min, max, fallback) {
    const n = Number(value);
    return Number.isFinite(n) ? Math.min(Math.max(Math.round(n), min), max) : fallback;
  }
  function saveRoomSettings() {
    localStorage.setItem(ROOM_SETTINGS_KEY, JSON.stringify(roomSettings));
  }

  const state = {
    running: false, paused: false, busy: false, mode: 'grass',
    plan: [{type:'grass',difficulty:'Khó',count:1},{type:'skip',difficulty:'Ác Mộng',count:1},{type:'skip',difficulty:'Thường',count:1}], planIndex:0, chestCount:0, difficultyApplied:false,
    difficulty: '', stage: 0, lastAction: '', lastActionAt: 0,
    seenStage: '', startedAt: 0, log: [], stoppedReason: '',
    farmElements: [['Kim'], ['Mộc'], ['Thủy'], ['Hỏa'], ['Thổ']],
    stageDone: new Set(), retryAfter: 0, lastStartAt: 0, lastCombatStage: 0, farmScanUntil: 0, farmScanStage: 0, perfectStage: 1, perfectStartStage: 1, perfectRewardHandled: false, perfectAdvancePending: false, deadRetreatPending: false, lobbyWaitUntil: 0,
    perfectStage3ChestCount: 0, perfectStopAfterReturn: false, perfectKickIssued: false, perfectKickPending: false,
    matchStartPlayerCount: 0, interruptionResetApplied: false, planCompletedAwaitingKick: false
  };
  let lastRoomFingerprint = '';
  let panelOpen = true;

  function saveRunProgress() {
    try {
      if (!state.running) return;
      localStorage.setItem(RUN_PROGRESS_KEY, JSON.stringify({
        running: true, paused: !!state.paused, mode: state.mode, plan: state.plan, planIndex: state.planIndex, chestCount: state.chestCount, difficultyApplied: state.difficultyApplied,
        difficulty: state.difficulty, stage: state.stage,
        startedAt: state.startedAt, log: state.log.slice(0, 30),
        farmElements: state.farmElements, stageDone: [...state.stageDone],
        retryAfter: state.retryAfter, lastStartAt: state.lastStartAt,
        lastCombatStage: state.lastCombatStage, farmScanUntil: state.farmScanUntil,
        farmScanStage: state.farmScanStage, perfectStage: state.perfectStage, perfectStartStage: state.perfectStartStage,
        perfectRewardHandled: state.perfectRewardHandled,
        perfectAdvancePending: state.perfectAdvancePending,
        deadRetreatPending: state.deadRetreatPending, lobbyWaitUntil: state.lobbyWaitUntil,
        seenStage: state.seenStage, lastAction: state.lastAction,
        lastActionAt: state.lastActionAt,
        perfectStage3ChestCount: state.perfectStage3ChestCount, perfectStopAfterReturn: state.perfectStopAfterReturn,
        perfectKickIssued: state.perfectKickIssued, perfectKickPending: state.perfectKickPending,
        matchStartPlayerCount: state.matchStartPlayerCount, interruptionResetApplied: state.interruptionResetApplied,
        planCompletedAwaitingKick: state.planCompletedAwaitingKick
      }));
    } catch (_) {}
  }
  function clearSavedRunProgress() {
    try { localStorage.removeItem(RUN_PROGRESS_KEY); } catch (_) {}
  }
  const USER_STARTED_RUN_KEY = 'hh3d_maze_user_started_run_v1';
  function restoreRunProgress() {
    try {
      // Không tự khởi chạy khi người dùng chỉ vừa mở panel/trang. Chỉ phục hồi sau reload
      // nếu phiên tab này trước đó đã được người dùng bấm Auto.
      if (sessionStorage.getItem(USER_STARTED_RUN_KEY) !== '1') return false;
      const saved = JSON.parse(localStorage.getItem(RUN_PROGRESS_KEY) || 'null');
      if (!saved || saved.running !== true) { sessionStorage.removeItem(USER_STARTED_RUN_KEY); return false; }
      state.mode = ['grass','achievement','farm'].includes(saved.mode) ? saved.mode : 'grass';
      state.plan = normalizePlan(saved.plan || loadPlanFromUI());
      state.planIndex = clampInt(saved.planIndex,0,3,0); state.chestCount = clampInt(saved.chestCount,0,999,0); state.difficultyApplied=!!saved.difficultyApplied;
      applyPlanToUI(state.plan); syncActivePlanStep();
      state.paused = false; // yêu cầu: tự chạy tiếp sau khi trang tải lại
      state.running = true; state.busy = false;
      state.difficulty = saved.difficulty || '';
      state.stage = Number(saved.stage) || 0;
      state.startedAt = Number(saved.startedAt) || Date.now();
      state.log = Array.isArray(saved.log) ? saved.log.slice(0, 30) : [];
      state.farmElements = Array.isArray(saved.farmElements) ? saved.farmElements : state.farmElements;
      state.stageDone = new Set(Array.isArray(saved.stageDone) ? saved.stageDone : []);
      state.retryAfter = Number(saved.retryAfter) || 0;
      state.lastStartAt = Number(saved.lastStartAt) || 0;
      state.lastCombatStage = Number(saved.lastCombatStage) || 0;
      state.farmScanUntil = Number(saved.farmScanUntil) || 0;
      state.farmScanStage = Number(saved.farmScanStage) || 0;
      state.perfectStage = [1,2,3].includes(Number(saved.perfectStage)) ? Number(saved.perfectStage) : 1;
      state.perfectStartStage = [1,2].includes(Number(saved.perfectStartStage)) ? Number(saved.perfectStartStage) : state.perfectStage === 2 ? 2 : 1;
      state.perfectRewardHandled = !!saved.perfectRewardHandled;
      state.perfectAdvancePending = !!saved.perfectAdvancePending;
      state.deadRetreatPending = !!saved.deadRetreatPending;
      state.lobbyWaitUntil = Number(saved.lobbyWaitUntil) || 0;
      state.seenStage = saved.seenStage || '';
      state.lastAction = saved.lastAction || '';
      state.lastActionAt = Number(saved.lastActionAt) || 0;
      state.perfectStage3ChestCount = clampInt(saved.perfectStage3ChestCount, 0, 5, 0);
      state.perfectStopAfterReturn = !!saved.perfectStopAfterReturn;
      state.perfectKickIssued = !!saved.perfectKickIssued; state.perfectKickPending = !!saved.perfectKickPending;
      state.matchStartPlayerCount = clampInt(saved.matchStartPlayerCount, 0, 5, 0);
      state.interruptionResetApplied = !!saved.interruptionResetApplied;
      state.planCompletedAwaitingKick = !!saved.planCompletedAwaitingKick;
      ui.mode.value = state.mode;
      $$('.mc-stage-elements[data-stage]', panel).forEach((box, i) => {
        const chosen = state.farmElements[i] || ['Kim'];
        $$('input[data-element]', box).forEach(el => { el.checked = chosen.includes(el.dataset.element); });
      });
      ui.elements.style.display = state.mode === 'grass' ? 'block' : 'none';
      setStatus('Đã khôi phục · tự chạy tiếp');
      log('Đã khôi phục tiến trình sau khi tải lại trang; tự động tiếp tục.');
      saveRunProgress();
      return true;
    } catch (_) { return false; }
  }

  const panel = document.createElement('section');
  panel.id = 'hh3d-mc-auto-panel';
  panel.innerHTML = `
    <header class="mc-head"><b>神-Sakute</b><span class="mc-head-actions"><button id="mc-settings-toggle" type="button" title="Cài đặt">⚙</button><button id="mc-min" title="Thu gọn">−</button></span></header>
    <div id="mc-body">
      <section id="mc-settings" hidden>
        <div class="mc-settings-tree">
          <div class="mc-tree-root">
            <label class="mc-setting-option"><input id="mc-setting-auto-invite" type="checkbox"> Gửi lời mời</label>
          </div>
          <div class="mc-tree-root">
            <label class="mc-setting-option"><input id="mc-setting-surrender-kim-nightmare" type="checkbox"> Đầu hàng khi gặp boss Kim ở Ác Mộng</label>
          </div>
          <div class="mc-tree-root mc-tree-kick">
            <label class="mc-setting-option"><input id="mc-kick-enabled" type="checkbox"> Trục xuất</label>
            <div id="mc-kick-options" class="mc-tree-children" hidden>
              <div class="mc-tree-child">
                <label class="mc-setting-option"><input id="mc-kick-unready" type="checkbox"> Chưa sẵn sàng</label>
                <label id="mc-kick-delay-row" class="mc-tree-detail-row">Thời gian chờ (giây)<input id="mc-kick-delay" type="number" min="1" max="60" value="5"></label>
              </div>
              <div class="mc-tree-child mc-tree-id">
                <div class="mc-tree-branch-title">Khóa ID</div>
                <label class="mc-setting-option"><input id="mc-kick-unlocked-ids" type="checkbox"> Trục xuất người chưa khóa ID <span class="mc-tree-detail">(bỏ qua Trợ Chiến)</span></label>
                <div id="mc-protect-id-options" class="mc-protect-id" hidden>
                  <label>ID người chơi <input id="mc-protect-value-input" type="text" inputmode="numeric" placeholder="Nhập user_id"></label>
                  <button id="mc-protect-add" type="button">+ Tìm và khóa ID</button>
                  <div id="mc-protected-list"></div>
                  <p class="mc-note">ID đã khóa và thành viên có nhãn Trợ Chiến sẽ không bị trục xuất. Danh sách ID lưu qua lần tải lại trang.</p>
                </div>
              </div>
            </div>
          </div>
        </div>
        <p class="mc-note">Các tùy chọn được lưu lại sau khi tải lại trang.</p>
      </section>
      <label id="mc-required-players-row">Người tối thiểu để bắt đầu
        <select id="mc-required-players">${[1,2,3,4,5].map(n=>`<option value="${n}">${n}</option>`).join('')}</select>
      </label>
      <div id="mc-plan-editor">
        <div class="mc-subtitle">Kế hoạch tối đa 3 giai đoạn</div>
        ${[1,2,3].map((n) => `<fieldset class="mc-plan-stage" data-plan-stage="${n}"><legend>Giai đoạn ${n}</legend>
          <label>Chế độ
            <select class="mc-plan-type" data-plan-type="${n}">
              ${n===1 ? '<option value="grass">Cỏ</option><option value="achievement">Chiến tích</option><option value="farm">Farm</option>' : '<option value="skip">--/-- · Chuyển tiếp</option><option value="achievement">Chiến tích</option><option value="farm">Farm</option>'}
            </select>
          </label>
          <label>Độ khó<select class="mc-plan-difficulty" data-plan-difficulty="${n}"><option>Khó</option><option>Ác Mộng</option><option>Thường</option></select></label>
          <label class="mc-plan-count-wrap" data-plan-count-wrap="${n}">Số rương Farm <input class="mc-plan-count" data-plan-count="${n}" type="number" min="1" max="999" value="1"></label>
        </fieldset>`).join('')}
        <p class="mc-note">Mỗi giai đoạn bắt đầu ở độ khó đã chọn và kết thúc khi nhận đủ rương chinh phục. Cỏ dò hệ boss; Chiến tích nhận 1 rương; Farm nhận số rương đã đặt. --/-- bỏ qua giai đoạn đó.</p>
      </div>
      <select id="mc-mode" hidden><option value="grass">Cỏ</option><option value="achievement">Chiến tích</option><option value="farm">Farm</option></select>
      <div id="mc-elements">
        <div class="mc-subtitle">Hệ boss cần đánh theo từng ải (chọn Bỏ để không dò)</div>
        <div class="mc-element-grid">
          ${[1,2,3,4,5].map((n) => `<fieldset class="mc-stage-elements" data-stage="${n}"><legend>Ải ${n}</legend>${['Bỏ', ...words.elements.map(e=>e[0].toUpperCase()+e.slice(1))].map((e,i)=>`<label><input type="checkbox" data-element="${e}" ${n===1&&e==='Kim'?'checked':''}>${e}</label>`).join('')}</fieldset>`).join('')}
        </div>
        <p class="mc-note">Mỗi ải chọn 1 hoặc 2 ngũ hành. Chọn Bỏ thì không thể chọn hệ khác ở ải đó. Khi dò, chỉ cần boss khớp ít nhất 1 hệ đã chọn là tiếp tục đánh; sai cả hai hệ thì đầu hàng. Tiến trình các ải còn lại được ghi nhớ.</p>
      </div>
      <div class="mc-status"><span>Trạng thái</span><b id="mc-status">Đang chờ</b></div>
      <div class="mc-status"><span>Độ khó</span><b id="mc-diff">Chưa xác định</b></div>
      <div class="mc-status"><span>Ải / hệ boss</span><b id="mc-stage">—</b></div>
      <div class="mc-status"><span>Thời gian</span><b id="mc-time">00:00</b></div>
      <div class="mc-buttons">
        <button id="mc-start" class="primary">▶ Auto</button>
        <button id="mc-pause">⏸ Tạm dừng</button>
        <button id="mc-stop" class="danger">■ Dừng</button>
      </div>
      <div class="mc-subtitle">Nhật ký</div><div id="mc-log" aria-live="polite"></div>
      <div class="mc-note">Cỏ: đầu hàng ngay khi sai hệ; khoảng cách bắt đầu tối thiểu 11 giây. Ải đã hoàn thành trong lượt này không đầu hàng.</div>
    </div>`;
  document.body.appendChild(panel);
  // Nút mở panel xuất hiện khi là chủ phòng; mở panel không tự khởi chạy tool.
  const launcher = document.createElement('button');
  launcher.id = 'hh3d-mc-auto-launcher';
  launcher.type = 'button';
  launcher.textContent = '神-Sakute';
  launcher.style.cssText = 'position:fixed;right:16px;top:90px;z-index:2147483647;padding:10px 14px;border:1px solid #7944bd;border-radius:10px;background:#24113c;color:#fff;font:600 13px system-ui,sans-serif;cursor:pointer;box-shadow:0 6px 20px #0008;display:none';
  document.body.appendChild(launcher);
  panel.style.display = '';
  panel.classList.add('mc-collapsed');
  $('#mc-min', panel).textContent = '＋';
  $('#mc-min', panel).title = 'Mở rộng panel';

  const style = document.createElement('style');
  style.textContent = `
    #hh3d-mc-auto-panel{position:fixed;z-index:2147483646;right:16px;top:90px;width:330px;max-height:82vh;overflow:auto;background:#100c22;color:#eee8ff;border:1px solid #7944bd;border-radius:14px;box-shadow:0 10px 40px #0009;font:13px/1.45 system-ui,sans-serif;pointer-events:none} #hh3d-mc-auto-panel *{pointer-events:auto}
    #hh3d-mc-auto-panel *{box-sizing:border-box} #hh3d-mc-auto-panel .mc-head{display:flex;align-items:center;justify-content:space-between;gap:8px;padding:11px 13px;background:#24113c;border-radius:13px 13px 0 0;font-size:15px} #hh3d-mc-auto-panel .mc-head-actions{display:flex;align-items:center;gap:5px} #hh3d-mc-auto-panel #mc-settings-toggle,#hh3d-mc-auto-panel #mc-min{min-width:38px;padding:7px 9px;text-align:center}
    #hh3d-mc-auto-panel button,#hh3d-mc-auto-panel select{font:inherit;color:inherit;background:#211735;border:1px solid #61418e;border-radius:8px;padding:7px 9px}
    #hh3d-mc-auto-panel button{cursor:pointer} #hh3d-mc-auto-panel button:hover{filter:brightness(1.2)}
    #hh3d-mc-auto-panel #mc-body{padding:12px;display:grid;gap:10px} #hh3d-mc-auto-panel #mc-settings{display:grid;gap:8px;padding:9px;background:#1b142b;border-radius:8px} #hh3d-mc-auto-panel #mc-settings[hidden]{display:none} #hh3d-mc-auto-panel .mc-settings-tree{display:grid;gap:9px} #hh3d-mc-auto-panel .mc-tree-root{position:relative;padding-left:2px} #hh3d-mc-auto-panel .mc-tree-root>.mc-setting-option{font-weight:600} #hh3d-mc-auto-panel .mc-tree-children{display:grid;gap:8px;margin:7px 0 0 12px;padding:2px 0 2px 15px;border-left:1px solid #70548f} #hh3d-mc-auto-panel .mc-tree-children[hidden]{display:none} #hh3d-mc-auto-panel .mc-tree-child{position:relative;display:grid;gap:5px} #hh3d-mc-auto-panel .mc-tree-child:before{content:"";position:absolute;left:-15px;top:10px;width:10px;border-top:1px solid #70548f} #hh3d-mc-auto-panel .mc-tree-branch-title{font-weight:600;color:#d4adff} #hh3d-mc-auto-panel .mc-tree-detail{font-size:11px;font-weight:400;color:#b9accd} #hh3d-mc-auto-panel .mc-tree-detail-row{display:flex;align-items:center;justify-content:space-between;gap:8px;padding-left:24px;font-size:11px;color:#b9accd} #hh3d-mc-auto-panel .mc-tree-detail-row input{width:70px} #hh3d-mc-auto-panel .mc-setting-option{display:flex;align-items:flex-start;justify-content:flex-start;gap:8px;line-height:1.4} #hh3d-mc-auto-panel .mc-setting-option input{flex:0 0 auto;margin-top:3px} #hh3d-mc-auto-panel #mc-body>label{display:grid;gap:5px}
    #hh3d-mc-auto-panel select{width:100%;min-width:0} #hh3d-mc-auto-panel .mc-element-grid{display:grid;grid-template-columns:repeat(5,minmax(0,1fr));gap:5px}
    #hh3d-mc-auto-panel .mc-stage-elements{min-width:0;margin:0;padding:5px;border:1px solid #49306d;border-radius:7px}
    #hh3d-mc-auto-panel .mc-stage-elements legend{font-size:11px;padding:0 3px} #hh3d-mc-auto-panel .mc-stage-elements label{font-size:10px;display:flex;align-items:center;gap:3px;margin:3px 0}
    #hh3d-mc-auto-panel .mc-stage-elements input{width:auto;margin:0}
    #hh3d-mc-auto-panel #mc-kick-options[hidden],#hh3d-mc-auto-panel #mc-kick-delay-row[hidden],#hh3d-mc-auto-panel #mc-protect-id-options[hidden]{display:none!important}
    #hh3d-mc-auto-panel #mc-required-players-row{display:flex;align-items:center;justify-content:space-between;gap:8px;padding:7px 9px;background:#1b142b;border-radius:8px}
    #hh3d-mc-auto-panel .mc-protect-id{display:grid;gap:6px;margin-top:8px}
    #hh3d-mc-auto-panel .mc-protect-id label{display:grid;gap:4px}
    #hh3d-mc-auto-panel .mc-protect-id input{width:100%;min-width:0;background:#100c22;color:#eee8ff;border:1px solid #61418e;border-radius:6px;padding:6px}
    #hh3d-mc-auto-panel .mc-protected-item{display:flex;align-items:center;justify-content:space-between;gap:6px;padding:5px;background:#100c22;border-radius:6px;margin-top:4px}
    #hh3d-mc-auto-panel .mc-protected-item button{padding:3px 6px;background:#70202b}
    #hh3d-mc-auto-panel .mc-subtitle{font-weight:700;color:#d4adff;margin-top:2px}
    #hh3d-mc-auto-panel .mc-plan-stage{border:1px solid #49306d;border-radius:7px;padding:7px;margin:6px 0;display:grid;gap:6px;min-width:0}
    #hh3d-mc-auto-panel .mc-plan-stage legend{color:#d4adff;font-weight:700;padding:0 4px}
    #hh3d-mc-auto-panel .mc-plan-stage label{display:grid;grid-template-columns:1fr 1fr;align-items:center;gap:6px;margin:0}
    #hh3d-mc-auto-panel .mc-plan-stage select,#hh3d-mc-auto-panel .mc-plan-stage input{width:100%;min-width:0}
 #hh3d-mc-auto-panel .mc-note{font-size:11px;color:#b9accd;margin:0}
    #hh3d-mc-auto-panel .mc-status{display:flex;justify-content:space-between;gap:8px;padding:6px 8px;background:#1b142b;border-radius:7px}
    #hh3d-mc-auto-panel .mc-status span{color:#b9accd} #hh3d-mc-auto-panel .mc-status b{text-align:right}
    #hh3d-mc-auto-panel .mc-buttons{display:grid;grid-template-columns:1fr 1fr;gap:6px} #hh3d-mc-auto-panel .primary{background:#7040e8;border-color:#9268ff;font-weight:700}
    #hh3d-mc-auto-panel .danger{background:#70202b;border-color:#b33c4a} #hh3d-mc-auto-panel #mc-log{max-height:145px;overflow:auto;background:#0b0815;border-radius:8px;padding:8px;font-size:11px;white-space:normal}
    #hh3d-mc-auto-panel .log-line{padding:3px 0;border-bottom:1px solid #241a35} #hh3d-mc-auto-panel .good{color:#6de6a4} #hh3d-mc-auto-panel .warn{color:#ffca74}
    #hh3d-mc-auto-panel.mc-collapsed{width:auto;max-width:48px;max-height:none;overflow:visible;border-radius:10px} #hh3d-mc-auto-panel.mc-collapsed #mc-body{display:none} #hh3d-mc-auto-panel.mc-collapsed .mc-head{padding:4px;border-radius:10px;background:#24113c} #hh3d-mc-auto-panel.mc-collapsed .mc-head>b,#hh3d-mc-auto-panel.mc-collapsed #mc-settings-toggle{display:none} #hh3d-mc-auto-panel.mc-collapsed .mc-head-actions{gap:0} #hh3d-mc-auto-panel.mc-collapsed #mc-min{min-width:32px;width:32px;height:32px;padding:0;font-size:18px;border:0;background:transparent}
  `;
  document.head.appendChild(style);

  const ui = {
    mode: $('#mc-mode', panel), elements: $('#mc-elements', panel),
    status: $('#mc-status', panel), diff: $('#mc-diff', panel),
    stage: $('#mc-stage', panel), time: $('#mc-time', panel), log: $('#mc-log', panel)
  };
  function initUserSettingsUI() {
    const toggle = $('#mc-settings-toggle', panel);
    const settings = $('#mc-settings', panel);
    const autoInvite = $('#mc-setting-auto-invite', panel);
    const surrenderKimNightmare = $('#mc-setting-surrender-kim-nightmare', panel);
    autoInvite.checked = userOptions.autoInvite;
    surrenderKimNightmare.checked = userOptions.surrenderKimNightmare;
    toggle.addEventListener('click', () => { settings.hidden = !settings.hidden; });
    autoInvite.addEventListener('change', () => {
      userOptions.autoInvite = autoInvite.checked;
      saveUserOptions();
      log(`Tự gửi lời mời: ${userOptions.autoInvite ? 'CÓ' : 'KHÔNG'}`);
    });
    surrenderKimNightmare.addEventListener('change', () => {
      userOptions.surrenderKimNightmare = surrenderKimNightmare.checked;
      saveUserOptions();
      log(`Ác Mộng gặp boss Kim: tự đầu hàng ${userOptions.surrenderKimNightmare ? 'BẬT' : 'TẮT'}`);
    });
  }
  initUserSettingsUI();
  function initRoomSettingsUI() {
    const required = $('#mc-required-players', panel);
    const enabled = $('#mc-kick-enabled', panel);
    const unready = $('#mc-kick-unready', panel);
    const unlocked = $('#mc-kick-unlocked-ids', panel);
    const delay = $('#mc-kick-delay', panel);
    const kickOptions = $('#mc-kick-options', panel);
    const delayRow = $('#mc-kick-delay-row', panel);
    const protectOptions = $('#mc-protect-id-options', panel);
    function refreshKickSettingsVisibility() {
      kickOptions.hidden = !enabled.checked;
      delayRow.hidden = !enabled.checked || !unready.checked;
      protectOptions.hidden = !enabled.checked || !unlocked.checked;
    }
    required.value = String(roomSettings.requiredPlayers);
    enabled.checked = roomSettings.autoKickEnabled;
    unready.checked = roomSettings.autoKickIdle;
    unlocked.checked = roomSettings.autoKickUnlockedIds;
    delay.value = String(roomSettings.idleKickSeconds);
    required.addEventListener('change', () => {
      roomSettings.requiredPlayers = clampInt(required.value, 1, 5, 5);
      saveRoomSettings();
      log(`Số người tối thiểu để bắt đầu: ${roomSettings.requiredPlayers}`);
    });
    refreshKickSettingsVisibility();
    enabled.addEventListener('change', () => {
      roomSettings.autoKickEnabled = enabled.checked;
      refreshKickSettingsVisibility();
      saveRoomSettings();
      log(`Tự trục xuất thành viên: ${enabled.checked ? 'BẬT' : 'TẮT'}`);
    });
    unready.addEventListener('change', () => {
      roomSettings.autoKickIdle = unready.checked;
      refreshKickSettingsVisibility();
      saveRoomSettings();
      log(`Trục xuất người chưa sẵn sàng: ${unready.checked ? 'BẬT' : 'TẮT'}`);
    });
    unlocked.addEventListener('change', () => {
      roomSettings.autoKickUnlockedIds = unlocked.checked;
      refreshKickSettingsVisibility();
      saveRoomSettings();
      log(`Trục xuất người chưa khóa ID: ${unlocked.checked ? 'BẬT' : 'TẮT'} (bỏ qua Trợ Chiến)`);
    });
    delay.addEventListener('change', () => {
      roomSettings.idleKickSeconds = clampInt(delay.value, 1, 60, 5);
      delay.value = String(roomSettings.idleKickSeconds);
      saveRoomSettings();
      log(`Thời gian chờ trục xuất người chưa sẵn sàng: ${roomSettings.idleKickSeconds}s`);
    });
  }
  function isVisibleRoomElement(el) {
    return !!el && !el.hidden && !!(el.offsetWidth || el.offsetHeight || el.getClientRects().length)
      && getComputedStyle(el).display !== 'none' && getComputedStyle(el).visibility !== 'hidden';
  }
  function roomPlayerCount() {
    // Ở sảnh, dạng "4/5 Người" là số thành viên, không phải số ải.
    const bodyText = String(document.body?.innerText || '').replace(panel?.innerText || '', '');
    const explicit = bodyText.match(/(\d+)\s*\/\s*5\s*người\b/i);
    if (explicit) return Number(explicit[1]);
    for (const selector of ROOM_COUNT_SELECTORS) {
      const el = $(selector);
      if (!isVisibleRoomElement(el)) continue;
      const m = String(el.textContent || el.value || el.innerText || '').match(/\d+/);
      if (m) return Number(m[0]);
    }
    let count = 0;
    for (const selector of ROOM_PLAYER_SELECTORS) {
      const nodes = $$(selector).filter(isVisibleRoomElement);
      count = Math.max(count, nodes.length);
    }
    return count;
  }
  function roomReadyCount() {
    // Đọc nội bộ số "? Sẵn Sàng"; không hiển thị thêm bộ đếm trên panel.
    const bodyText = String(document.body?.innerText || '').replace(panel?.innerText || '', '');
    const m = bodyText.match(/(\d+)\s*sẵn\s*sàng\b/i);
    return m ? Number(m[1]) : null;
  }
  function isUnreadyRow(row) {
    const t = norm(row?.innerText || row?.textContent || '');
    return /chưa|chua|chờ sẵn sàng|cho san sang|not ready|waiting|đang chờ|dang cho|idle/.test(t);
  }
  function findKickButton(row) {
    const selectors = '.kick-btn,.btn-kick,.kick-player,.member-kick,.player-kick,.remove-user,.btn-remove,.icon-kick,[data-action="kick"],[data-role="kick"],[title*="kick" i],[title*="trục xuất" i],[title*="truc xuat" i],.btn-delete,button,a,[role="button"]';
    return $$(selectors, row).find(el => {
      if (!isVisibleRoomElement(el)) return false;
      const t = norm(`${el.textContent||''} ${el.getAttribute('title')||''} ${el.getAttribute('aria-label')||''}`);
      return /trục xuất|truc xuat|kick|remove|loại|loai/.test(t);
    });
  }
  function handleUnreadyPlayers(allowKick = false) {
    const rows = [...new Set(ROOM_PLAYER_SELECTORS.flatMap(sel => $$(sel)))].filter(isVisibleRoomElement);
    let unready = 0;
    let pending = 0;
    for (const row of rows) {
      const waiting = isUnreadyRow(row);
      if (!waiting) idleKickTracker.delete(row);
      if (waiting) unready++;
      // Trợ Chiến và ID đã khóa luôn nằm ngoài danh sách trục xuất.
      if (isProtectedRow(row)) { idleKickTracker.delete(row); continue; }
      if (!allowKick || !roomSettings.autoKickEnabled) continue;

      const shouldKickUnlocked = roomSettings.autoKickUnlockedIds;
      let shouldKickUnready = false;
      if (waiting && roomSettings.autoKickIdle) {
        const now = Date.now();
        if (!idleKickTracker.has(row)) {
          idleKickTracker.set(row, now);
          log(`Phát hiện người chưa sẵn sàng; bắt đầu đếm ${roomSettings.idleKickSeconds}s.`);
        } else shouldKickUnready = (now - idleKickTracker.get(row)) / 1000 >= roomSettings.idleKickSeconds;
        if (!shouldKickUnready) pending++;
      }
      if (!shouldKickUnlocked && !shouldKickUnready) continue;
      // Kiểm tra lại ngay trước khi bấm để tránh trục xuất người vừa được khóa ID/nhận diện Trợ Chiến.
      if (isProtectedRow(row)) { idleKickTracker.delete(row); continue; }
      const btn = findKickButton(row);
      if (btn) {
        const reason = shouldKickUnlocked ? 'chưa khóa ID' : `chưa sẵn sàng sau ${roomSettings.idleKickSeconds}s`;
        btn.click(); idleKickTracker.delete(row);
        if (!state.planCompletedAwaitingKick) {
          resetPlanToStage1(`Tool tự trục xuất thành viên ${reason}`);
          log(`Đã trục xuất thành viên ${reason}; đặt lại kế hoạch từ giai đoạn 1.`, 'warn');
        } else {
          log(`Đã trục xuất thành viên ${reason} sau khi hoàn thành toàn bộ giai đoạn.`, 'warn');
        }
        lastRoomFingerprint = '';
        return { unready, kicked: true };
      }
    }
    return { unready, kicked: false, pending };
  }

  initRoomSettingsUI();
  function renderProtectedPlayers() {
    const list = $('#mc-protected-list', panel); if (!list) return;
    list.innerHTML = protectedPlayers.map(p => `<div class="mc-protected-item"><span>${escapeHtml(p.id)}: ${escapeHtml(p.name || 'Chưa rõ tên')}</span><button type="button" data-unprotect-id="${escapeHtml(p.id)}">Gỡ</button></div>`).join('') || '<p class="mc-note">Chưa có ID được bảo vệ.</p>';
  }
  // Cache the actual user-status response used by the page. Re-fetching its URL as a
  // plain GET can lose the nonce/headers used by the original request and return 401.
  let cachedUserStatusMembers = [];
  function collectUserStatusMembers(data, out = []) {
    if (!data || typeof data !== 'object') return out;
    if (Array.isArray(data)) { for (const item of data) collectUserStatusMembers(item, out); return out; }
    if (data.user_id != null && data.display_name != null && /^\d+$/.test(String(data.user_id))) {
      out.push({ id: String(data.user_id), name: String(data.display_name) });
    }
    // API shape can change; traverse nested objects/arrays instead of relying on a
    // short allow-list of property names.
    for (const value of Object.values(data)) {
      if (value && typeof value === 'object') collectUserStatusMembers(value, out);
    }
    return out;
  }
  function cacheUserStatusPayload(data) {
    const members = collectUserStatusMembers(data);
    if (members.length) cachedUserStatusMembers = [...new Map(members.map(p => [p.id, p])).values()];
  }
  function isUserStatusUrl(url) { return /\/wp-json\/me-cung\/v1\/user-status(?:[?#]|$)/i.test(String(url || '')); }
  // Capture future fetch-based requests without changing the page's response.
  try {
    const originalFetch = window.fetch;
    if (originalFetch && !originalFetch.__hh3dUserStatusCapture) {
      const wrappedFetch = function(input, init) {
        const url = typeof input === 'string' ? input : input?.url;
        const promise = originalFetch.apply(this, arguments);
        if (isUserStatusUrl(url)) promise.then(response => {
          if (response && response.ok) response.clone().json().then(cacheUserStatusPayload).catch(() => {});
        }).catch(() => {});
        return promise;
      };
      wrappedFetch.__hh3dUserStatusCapture = true;
      window.fetch = wrappedFetch;
    }
  } catch (_) {}
  // Also capture XMLHttpRequest-based page requests.
  try {
    const xhrOpen = XMLHttpRequest.prototype.open;
    const xhrSend = XMLHttpRequest.prototype.send;
    if (!XMLHttpRequest.prototype.__hh3dUserStatusCapture) {
      XMLHttpRequest.prototype.open = function(method, url) {
        this.__hh3dUserStatusUrl = isUserStatusUrl(url);
        return xhrOpen.apply(this, arguments);
      };
      XMLHttpRequest.prototype.send = function() {
        if (this.__hh3dUserStatusUrl) this.addEventListener('load', function() {
          if (this.status >= 200 && this.status < 300) {
            try { cacheUserStatusPayload(JSON.parse(this.responseText)); } catch (_) {}
          }
        });
        return xhrSend.apply(this, arguments);
      };
      XMLHttpRequest.prototype.__hh3dUserStatusCapture = true;
    }
  } catch (_) {}
  async function findPlayerForProtection(query) {
    if (!/^\d+$/.test(query)) throw new Error('Chỉ nhập ID dạng số.');
    const rows = roomRows();
    const rowMatch = rows.map(row => rowPlayerIdentity(row)).find(p => p.id === query);
    let apiMatch = cachedUserStatusMembers.find(p => p.id === query);
    if (!apiMatch && !rowMatch) {
      const nonce = findRestNonceInDocument(document);
      try {
        const headers = { 'Accept': 'application/json' };
        if (nonce) headers['X-WP-Nonce'] = nonce;
        const response = await fetch(new URL('/wp-json/me-cung/v1/user-status', location.origin).href,
          { method: 'GET', credentials: 'same-origin', cache: 'no-store', headers });
        if (response.ok) {
          cacheUserStatusPayload(await response.json());
          apiMatch = cachedUserStatusMembers.find(p => p.id === query);
        }
      } catch (_) {}
    }
    return { id: query, name: rowMatch?.name || apiMatch?.name || '' };
  }
  $('#mc-protect-add', panel).addEventListener('click', async () => {
    const input = $('#mc-protect-value-input', panel);
    const query = String(input.value || '').trim();
    if (!query) { log('Hãy nhập ID người chơi.', 'warn'); return; }
    const button = $('#mc-protect-add', panel);
    button.disabled = true;
    try {
      const player = await findPlayerForProtection(query);
      if (!/^\d+$/.test(player.id)) throw new Error('Không lấy được user_id hợp lệ; chưa thêm bảo vệ.');
      if (!protectedPlayers.some(p => String(p.id) === player.id)) protectedPlayers.push({ id: player.id, name: player.name || 'Chưa rõ tên' });
      saveProtectedPlayers(); renderProtectedPlayers(); input.value = '';
      log(`Đã khóa bảo vệ ${player.id}: ${player.name || 'Chưa rõ tên'}.`, 'good');
    } catch (err) { log(err?.message || 'Không thể tìm ID người chơi.', 'warn'); }
    finally { button.disabled = false; }
  });
  $('#mc-protected-list', panel).addEventListener('click', ev => {
    const btn = ev.target.closest('[data-unprotect-id]'); if (!btn) return;
    const id = btn.getAttribute('data-unprotect-id'); protectedPlayers = protectedPlayers.filter(p => String(p.id) !== id);
    saveProtectedPlayers(); renderProtectedPlayers(); log(`Đã gỡ bảo vệ ID ${id}.`, 'warn');
  });
  renderProtectedPlayers();

  function log(message, kind = '') {
    const time = new Date().toLocaleTimeString();
    state.log.unshift({time, message, kind});
    state.log = state.log.slice(0, 60);
    ui.log.innerHTML = state.log.map(x => `<div class="log-line ${x.kind}">${x.time} — ${escapeHtml(x.message)}</div>`).join('');
  }
  function escapeHtml(s) {
    return String(s).replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
  }
  function setStatus(s) { ui.status.textContent = s; }
  function getVisibleText() { return norm(document.body?.innerText || ''); }

  function findClickable(re, scope = document) {
    const candidates = $$('button,[role="button"],a,input[type="button"],input[type="submit"]', scope).filter(visible);
    return candidates.find(el => re.test(textOf(el) || norm(el.getAttribute('aria-label')) || norm(el.value)));
  }
  function click(el, description) {
    if (!el || !visible(el)) return false;
    el.click();
    state.lastAction = description;
    state.lastActionAt = Date.now();
    log(`Đã bấm: ${description}`, 'warn');
    return true;
  }
  function autoAttackControl() {
    // Giao diện hiện tại hiển thị nhãn "TỰ ĐỘNG" cạnh công tắc dạng viên thuốc.
    // Tìm phần tử nhãn trước, sau đó ưu tiên công tắc/nút nằm cùng cụm giao diện.
    const all = $$('body *').filter(el => el !== panel && !panel.contains(el) && visible(el));
    const label = all
      .filter(el => /^(tự động|auto\s*attack|autoattack|tự động chiến đấu)$/.test(norm(textOf(el))) && el.children.length <= 2)
      .sort((a, b) => (a.innerText || a.textContent || '').length - (b.innerText || b.textContent || '').length)[0];
    if (label) {
      let parent = label;
      for (let depth = 0; depth < 4 && parent && parent !== document.body; depth++, parent = parent.parentElement) {
        const toggles = $$('button,[role="switch"],[role="checkbox"],input[type="checkbox"],label,[tabindex], [class*="switch"], [class*="toggle"]', parent)
          .filter(el => el !== label && visible(el));
        if (toggles.length) {
          // Công tắc thực tế thường là phần tử nhỏ nằm bên phải nhãn.
          const rightSide = toggles.filter(el => el.getBoundingClientRect().left >= label.getBoundingClientRect().left);
          const pool = rightSide.length ? rightSide : toggles;
          return pool.sort((a, b) => {
            const ar = a.getBoundingClientRect(), br = b.getBoundingClientRect();
            return (ar.width * ar.height) - (br.width * br.height);
          })[0];
        }
      }
      // Nếu không tìm thấy công tắc con, bấm cụm nhãn nhỏ nhất có thể tương tác.
      const clickable = label.closest('button,[role="button"],label,[tabindex]');
      if (clickable && clickable !== panel && !panel.contains(clickable)) return clickable;
    }
    const candidates = all.filter(el => /tự động|auto\s*attack|autoattack|tự động chiến đấu/i.test(
      [textOf(el), el.getAttribute('aria-label') || '', el.getAttribute('title') || ''].join(' ')
    ));
    return candidates.sort((a, b) => (a.innerText || a.textContent || '').length - (b.innerText || b.textContent || '').length)[0] || null;
  }
  function autoAttackIsOn(el) {
    if (!el) return false;
    const nodes = [el, ...$$('*', el)].slice(0, 80);
    for (const node of nodes) {
      const ariaChecked = node.getAttribute?.('aria-checked');
      const ariaPressed = node.getAttribute?.('aria-pressed');
      if (ariaChecked === 'true' || ariaPressed === 'true') return true;
      if (node.matches?.('input[type="checkbox"]') && node.checked) return true;
      const cls = norm(typeof node.className === 'string' ? node.className : '');
      if (/(^|\s)(active|enabled|on|checked|is-on|toggle-on)(\s|$)/.test(cls)) return true;
      // Kiểm tra màu nền xanh của công tắc đang bật (như ảnh người dùng gửi).
      try {
        const bg = getComputedStyle(node).backgroundColor;
        const m = bg.match(/rgba?\((\d+),\s*(\d+),\s*(\d+)/i);
        if (m) {
          const r = +m[1], g = +m[2], b = +m[3];
          if (g > 100 && g > r * 1.25 && g > b * 1.05) return true;
        }
      } catch (_) {}
    }
    const label = norm([textOf(el), el.getAttribute('aria-label') || '', el.getAttribute('title') || ''].join(' '));
    return /đang tự động|tắt tự động|auto\s*attack\s*on|autoattack\s*on|dừng tự động/.test(label);
  }

  function settingToggleState(el) {
    if (!el) return null;
    const nodes = [el, ...$$('*', el)].slice(0, 80);
    let sawState = false;
    for (const node of nodes) {
      const ariaChecked = node.getAttribute?.('aria-checked');
      const ariaPressed = node.getAttribute?.('aria-pressed');
      if (ariaChecked === 'true' || ariaPressed === 'true') return true;
      if (ariaChecked === 'false' || ariaPressed === 'false') sawState = true;
      if (node.matches?.('input[type="checkbox"],input[type="radio"]')) {
        sawState = true;
        if (node.checked) return true;
      }
      const cls = norm(typeof node.className === 'string' ? node.className : '');
      if (/(^|\s)(active|enabled|on|checked|is-on|toggle-on)(\s|$)/.test(cls)) return true;
      if (/(^|\s)(inactive|disabled|off|unchecked|is-off|toggle-off)(\s|$)/.test(cls)) sawState = true;
      try {
        const bg = getComputedStyle(node).backgroundColor;
        const m = bg.match(/rgba?\((\d+),\s*(\d+),\s*(\d+)/i);
        if (m) {
          const r = +m[1], g = +m[2], b = +m[3];
          if (g > 100 && g > r * 1.25 && g > b * 1.05) return true;
          if (r > 100 && r > g * 1.25 && r > b * 1.05) sawState = true;
        }
      } catch (_) {}
    }
    const label = norm([textOf(el), el.getAttribute?.('aria-label') || '', el.getAttribute?.('title') || ''].join(' '));
    if (/^(bật|đang bật|on|enabled)$/.test(label)) return true;
    if (/^(tắt|đang tắt|off|disabled)$/.test(label)) return false;
    return sawState ? false : null;
  }

  function findNamedSettingToggle(name) {
    const outside = $$('body *').filter(el => el !== panel && !panel.contains(el) && visible(el));
    const label = outside
      .filter(el => norm(textOf(el)) === norm(name) && el.children.length <= 3)
      .sort((a, b) => (a.innerText || a.textContent || '').length - (b.innerText || b.textContent || '').length)[0];
    if (!label) return null;
    for (let parent = label, depth = 0; depth < 4 && parent && parent !== document.body; depth++, parent = parent.parentElement) {
      const toggles = $$('button,[role="switch"],[role="checkbox"],input[type="checkbox"],label,[tabindex],[class*="switch"],[class*="toggle"]', parent)
        .filter(el => el !== label && visible(el) && el !== panel && !panel.contains(el));
      if (toggles.length) {
        const lr = label.getBoundingClientRect();
        const right = toggles.filter(el => el.getBoundingClientRect().left >= lr.left);
        const pool = right.length ? right : toggles;
        return pool.sort((a, b) => {
          const ar = a.getBoundingClientRect(), br = b.getBoundingClientRect();
          return (ar.width * ar.height) - (br.width * br.height);
        })[0];
      }
      const clickable = label.closest('button,[role="button"],label,[tabindex]');
      if (clickable && clickable !== panel && !panel.contains(clickable)) return clickable;
    }
    return null;
  }

  function keepGameSettingsEnabled() {
    for (const name of ['Tự động chuyển ải', 'Tự động mở rương']) {
      const toggle = findNamedSettingToggle(name);
      if (!toggle) continue;
      const state = settingToggleState(toggle);
      // Chỉ bấm khi trạng thái tắt được nhận diện rõ ràng; không mạo hiểm đảo công tắc đang bật.
      if (state === false) {
        toggle.click();
        log(`Tự bật lại cài đặt "${name}".`, 'good');
      }
    }
  }

  function isRoomHost() {
    // 1) Sảnh: nhận diện nhãn chữ "Chủ phòng" như các bản trước.
    const candidates = $$('body *').filter(el =>
      el !== panel && !panel.contains(el) && visible(el) &&
      /^(chủ phòng|chủ phòng 👑|host|room host)$/.test(norm(textOf(el))) &&
      el.children.length <= 3
    );
    if (candidates.length) return true;

    // 2) Trong trận: nhãn chữ không còn xuất hiện; game đánh dấu chủ phòng
    // bằng huy hiệu vương miện trên thẻ nhân vật. Tìm biểu tượng/nhãn liên quan
    // tới crown/host trong DOM và kiểm tra nó đang hiển thị.
    if (isCombat()) {
      const crownNodes = $$('body *').filter(el => {
        if (el === panel || panel.contains(el) || !visible(el)) return false;
        const meta = norm([
          el.textContent,
          el.getAttribute?.('aria-label'), el.getAttribute?.('title'),
          el.getAttribute?.('alt'), el.getAttribute?.('src'),
          el.getAttribute?.('class'), el.getAttribute?.('data-icon')
        ].filter(Boolean).join(' '));
        const crownMark = /👑|crown|king|host|owner|leader|chu[-_ ]?phong/.test(meta);
        // Hạn chế bắt nhầm cả khung thẻ chứa nội dung dài.
        return crownMark && el.children.length <= 4 && (el.tagName === 'IMG' || el.children.length <= 2 || /👑/.test(meta));
      });
      if (crownNodes.length) return true;
    }
    return false;
  }


  const EXPIRED_RELOAD_KEY = 'hh3d_maze_expired_reload_once_v1';

  function checkExpiredSession() {
    const bodyText = norm(document.body?.innerText || '');
    const expired = /phiên đã hết hạn/.test(bodyText) &&
                    /tải lại trang/.test(bodyText);

    if (!expired) {
      // Xóa khóa khi màn hết hạn đã biến mất, để lần hết hạn sau vẫn tự xử lý.
      try { sessionStorage.removeItem(EXPIRED_RELOAD_KEY); } catch (_) {}
      return false;
    }

    // Lưu tiến trình trước khi tải lại; restoreRunProgress() sẽ tự tiếp tục sau khi trang mở lại.
    if (state.running) saveRunProgress();

    try {
      // Chống reload lặp vô hạn nếu máy chủ vẫn trả về màn "Phiên đã hết hạn".
      if (sessionStorage.getItem(EXPIRED_RELOAD_KEY) === '1') return true;
      sessionStorage.setItem(EXPIRED_RELOAD_KEY, '1');
    } catch (_) {}

    setStatus('Phiên hết hạn — đang tải lại và khôi phục tiến trình…');
    window.setTimeout(() => {
      try { window.location.reload(); } catch (_) {}
    }, 900);
    return true;
  }

  function roomRows() {
    const all = [...new Set(ROOM_PLAYER_SELECTORS.flatMap(sel => $$(sel)))].filter(isVisibleRoomElement);
    return all.filter(el => !all.some(other => other !== el && other.contains(el)))
      .sort((a, b) => a.getBoundingClientRect().top - b.getBoundingClientRect().top);
  }
  function roomRosterNames() {
    return roomRows().map(row => norm((row.innerText || row.textContent).split('\n')[0]).slice(0, 50)).filter(Boolean);
  }
  function roomFingerprint() {
    const names = roomRosterNames();
    if (names.length) return [...new Set(names)].sort().join('|');
    const count = roomPlayerCount();
    return count ? `count:${count}` : '';
  }
  function findInviteCopyButton() {
    return $$('button,a,[role="button"],[title],[aria-label]').find(el => isVisibleRoomElement(el) && !panel.contains(el) &&
      /sao chép link mời|sao chep link moi|copy invite|invite link/i.test(
        `${el.innerText || ''} ${el.getAttribute('title') || ''} ${el.getAttribute('aria-label') || ''} ${el.getAttribute('data-tooltip') || ''}`));
  }
  function findRestNonceInDocument(doc = document) {
    const candidates = [window.wpApiSettings?.nonce, window._wpnonce, window.wp_rest_nonce,
      window.bm?.nonce, window.BetterMessages?.nonce, window.better_messages?.nonce,
      doc.querySelector('meta[name="wp-rest-nonce"]')?.content, doc.querySelector('input[name="_wpnonce"]')?.value];
    for (const value of candidates) if (typeof value === 'string' && value.length >= 8) return value;
    const scripts = [...doc.querySelectorAll('script')].map(el => el.textContent || '').join('\n');
    const patterns = [/(?:wpApiSettings|restNonce|rest_nonce|wp_rest_nonce)["'\s:=,]+["']([a-zA-Z0-9_-]{8,})["']/i, /["']nonce["']\s*:\s*["']([a-zA-Z0-9_-]{8,})["']/i, /X-WP-Nonce.{0,80}?["']([a-zA-Z0-9_-]{8,})["']/i];
    for (const re of patterns) { const m = scripts.match(re); if (m) return m[1]; }
    return '';
  }
  async function obtainRestNonce() {
    let nonce = findRestNonceInDocument(document);
    if (nonce) return nonce;
    // Nonce thường chỉ được nhúng ở trang Better Messages. Tải HTML cùng origin nền, không mở tab.
    const page = await fetch(`${SITE_ORIGIN}/danh-sach-tin-nhan`, { credentials: 'include', cache: 'no-store', mode: 'same-origin' });
    if (!page.ok) throw new Error(`Không tải được trang tin nhắn để lấy nonce (HTTP ${page.status}).`);
    const html = await page.text();
    const doc = new DOMParser().parseFromString(html, 'text/html');
    nonce = findRestNonceInDocument(doc);
    if (!nonce) throw new Error('Không tìm thấy REST nonce trong trang Mê Cung hoặc HTML trang tin nhắn.');
    return nonce;
  }
  async function sendInviteMessage(inviteText) {
    // Dùng cùng origin với trang Mê Cung; site có thể được mở qua nhiều domain.
    // Giữ nguyên nội dung copy, không trim hoặc tự thêm chữ vào lời mời.
    const message = String(inviteText ?? '');
    if (!message.trim()) throw new Error('Nội dung lời mời rỗng.');
    let lastError = null;
    for (let attempt = 0; attempt < 2; attempt++) {
      const nonce = await obtainRestNonce();
      const now = Date.now();
      const payload = {
        message,
        temp_id: `tmp_${CHAT_THREAD_ID}_${Math.floor(Math.random() * 900000000 + 100000000)}`,
        temp_time: now * 10,
        include_update: 1
      };
      const sendUrl = `${SITE_ORIGIN}/wp-json/better-messages/v1/thread/${CHAT_THREAD_ID}/send?nocache=${now}`;
      let response;
      try {
        response = await fetch(sendUrl, {
          method: 'POST', credentials: 'include', cache: 'no-store', mode: 'same-origin',
          headers: { 'Content-Type': 'application/json', 'X-WP-Nonce': nonce, 'Accept': 'application/json' },
          body: JSON.stringify(payload)
        });
      } catch (networkError) {
        if (attempt === 0) { await sleep(1200); continue; }
        throw new Error(`Không kết nối được API gửi tin tại ${sendUrl}. Kiểm tra miền/API hoặc kết nối mạng. Chi tiết: ${networkError?.message || networkError}`);
      }
      let data = null;
      const raw = await response.text();
      try { data = raw ? JSON.parse(raw) : null; } catch (_) {}
      const apiError = data && (data.result === false || data.success === false || data.error || data.code && data.message && response.status >= 400);
      if (response.ok && !apiError) return data;
      lastError = new Error(`API gửi tin nhắn HTTP ${response.status}${data?.message || data?.data?.message ? `: ${data.message || data.data.message}` : raw ? `: ${raw.slice(0, 180)}` : ''}`);
      // Nonce có thể hết hạn; lần thử thứ hai sẽ đọc lại nonce mới.
      if (attempt === 0 && (response.status === 401 || response.status === 403)) continue;
      throw lastError;
    }
    throw lastError || new Error('Không gửi được lời mời.');
  }
  function isRoomLocked() {
    // Khi phòng đang khóa, nút hiện là "Mở Khóa"; khi đang mở, nút hiện "Khóa Phòng".
    return !!findClickable(/^mở khóa phòng$|^mở khóa$/i);
  }
  async function refreshRoomIfLocked() {
    if (!isRoomLocked()) return false;
    const unlock = findClickable(/^mở khóa phòng$|^mở khóa$/i);
    if (!unlock) return false;
    click(unlock, 'Mở khóa phòng để làm mới');
    await sleep(500);
    const lock = findClickable(/^khóa phòng$/i);
    if (!lock) { log('Đã mở khóa nhưng không tìm thấy nút Khóa Phòng; chưa gửi lời mời mới.', 'warn'); return false; }
    click(lock, 'Khóa lại phòng sau 0,5 giây');
    await sleep(1000);
    return true;
  }
  async function maybeSendInviteMessage(forceForRosterChange = false, initialAfterAuto = false, afterLock = false) {
    if (!userOptions.autoInvite || !state.running || state.paused || !isRoomHost()) return;
    if (inviteSendInFlight) return;
    const players = roomPlayerCount();
    if (players >= roomSettings.requiredPlayers || players <= 0) return;
    let lastSent = 0; try { lastSent = Number(localStorage.getItem(INVITE_SENT_KEY)) || 0; } catch (_) {}
    if (!initialAfterAuto && !afterLock && forceForRosterChange && Date.now() - lastSent < INVITE_EVENT_COOLDOWN_MS && Date.now() - (state.startedAt || Date.now()) < ROOM_RELOCK_INTERVAL_MS) return;
    inviteSendInFlight = true;
    try {
      // Phòng khóa thì làm mới trước; phòng đang mở thì chỉ lấy/gửi lời mời.
      const refreshed = await refreshRoomIfLocked();
      const button = findInviteCopyButton();
      if (!button) { log('Không tìm thấy nút lấy lời mời ở sảnh.', 'warn'); return; }
      let copiedText = '';
      const onCopy = ev => { const text = ev.clipboardData?.getData('text/plain'); if (text) copiedText = text; };
      document.addEventListener('copy', onCopy, true);
      try {
        // Một số lần nút copy cập nhật clipboard chậm; thử lại tối đa 3 lần.
        for (let attempt = 0; attempt < 3 && !copiedText; attempt++) {
          button.click();
          await sleep(350 + attempt * 250);
          if (!copiedText && navigator.clipboard?.readText) {
            try { copiedText = await navigator.clipboard.readText(); } catch (_) {}
          }
        }
      } finally { document.removeEventListener('copy', onCopy, true); }
      copiedText = String(copiedText || '');
      if (!copiedText.trim()) { log('Đã bấm sao chép nhưng không đọc được nội dung lời mời sau 3 lần thử; chưa gửi để tránh gửi sai.', 'warn'); return; }
      await sendInviteMessage(copiedText);
      const sentAt = Date.now();
      localStorage.setItem(INVITE_SENT_KEY, String(sentAt));
      log(refreshed ? 'Đã làm mới phòng và gửi lời mời vào tin nhắn thành công.' : 'Đã gửi lời mời vào tin nhắn thành công.', 'good');
    } catch (e) { log(`Chưa gửi được lời mời: ${e?.message || e}`, 'warn'); }
    finally { inviteSendInFlight = false; }
  }
  function updatePanelVisibility() {
    const host = isRoomHost();
    const inCombat = isCombat();
    // Khi đang chạy, giữ panel hiển thị trong trận kể cả khi giao diện trận đấu
    // không còn chứa nhãn Chủ phòng. Thu gọn/mở rộng chỉ đổi cách hiển thị,
    // không ảnh hưởng vòng lặp tự động.
    const keepVisibleInCombat = state.running && inCombat;
    const showPanel = true;
    panel.style.display = showPanel ? '' : 'none';
    launcher.style.display = 'none';
  }

  function stopAutoAttackForScan() {
    const control = autoAttackControl();
    if (control && autoAttackIsOn(control)) {
      click(control, 'Tắt AutoAttack để dò hệ boss');
      return true;
    }
    return false;
  }
  function currentDifficulty() {
    const t = getVisibleText();
    // Tránh đọc nhầm các lựa chọn trong panel; chỉ dùng nhãn hiển thị ngoài panel.
    const outside = norm(document.body.innerText.replace(panel.innerText, ''));
    if (/độ khó.{0,20}ác mộng|ác mộng/.test(outside)) return 'Ác Mộng';
    if (/độ khó.{0,20}khó|độ khó phòng.{0,20}khó/.test(outside)) return 'Khó';
    if (/độ khó.{0,20}thường|độ khó phòng.{0,20}thường/.test(outside)) return 'Thường';
    const m = outside.match(/độ khó\s*([^\n]+)/);
    if (m) {
      const v = m[1];
      if (v.includes('ác mộng')) return 'Ác Mộng';
      if (v.includes('khó')) return 'Khó';
      if (v.includes('thường')) return 'Thường';
    }
    return '';
  }
  function currentStage() {
    if (!isCombat()) return 0;

    // Ưu tiên số ải hiển thị trực tiếp trong nhãn "Ải X/5".
    // Loại panel của tool để không đọc nhầm trạng thái nội bộ.
    const textNodes = $$('body *')
      .filter(el => el !== panel && !panel.contains(el) && isVisibleRoomElement(el))
      .filter(el => el.children.length === 0 || (el.children.length <= 2 && (el.innerText || '').length < 40))
      .map(el => ({ el, txt: (el.innerText || el.textContent || '').replace(/\s+/g, ' ').trim() }))
      .filter(x => x.txt);
    for (const item of textNodes) {
      const m = item.txt.match(/(?:^|[^\p{L}])ải\s*([1-5])\s*\/\s*5(?:$|[^0-9])/iu);
      if (m) return Number(m[1]);
    }

    // Không có nhãn "Ải X/5": đếm các vòng tròn màu liền kề ở góc trên trái.
    // Không xét phần tử chứa chữ; chỉ nhận phần tử nhỏ, gần tròn và có màu sáng.
    const dots = [];
    for (const el of $$('body *')) {
      if (el === panel || panel.contains(el) || !isVisibleRoomElement(el) || el.children.length > 0) continue;
      const txt = (el.innerText || el.textContent || '').trim();
      if (txt) continue;
      const r = el.getBoundingClientRect();
      if (r.left < 12 || r.left > 125 || r.top < 8 || r.top > 62) continue;
      if (r.width < 7 || r.height < 7 || r.width > 22 || r.height > 22) continue;
      if (Math.abs(r.width - r.height) > 5) continue;
      const cs = getComputedStyle(el);
      const radius = parseFloat(cs.borderTopLeftRadius) || 0;
      if (radius < Math.min(r.width, r.height) * 0.3) continue;
      const colors = [cs.backgroundColor, cs.borderTopColor, cs.borderRightColor, cs.borderBottomColor, cs.borderLeftColor];
      let colored = false;
      for (const color of colors) {
        const nums = String(color).match(/[\d.]+/g)?.map(Number) || [];
        if (nums.length < 3 || (nums[3] !== undefined && nums[3] <= 0.15)) continue;
        const [rr, gg, bb] = nums;
        // Vàng sáng hoặc xanh/cyan sáng; bỏ tím tối và vòng chỉ có viền.
        if ((rr >= 155 && gg >= 105 && bb <= 155 && rr - bb >= 35) ||
            (gg >= 125 && bb >= 125 && gg - rr >= 20 && bb - rr >= 15)) {
          colored = true; break;
        }
      }
      if (!colored) continue;
      const x = Math.round(r.left + r.width / 2);
      const y = Math.round(r.top + r.height / 2);
      if (!dots.some(d => Math.abs(d.x - x) <= 4 && Math.abs(d.y - y) <= 4)) dots.push({x, y});
    }
    if (!dots.length) return 0;
    dots.sort((a,b) => a.x - b.x);
    // Chỉ đếm dãy liền kề trên cùng một hàng, tránh lấy các chấm ở nơi khác.
    const rows = [];
    for (const dot of dots) {
      let row = rows.find(row => Math.abs(row.y - dot.y) <= 6);
      if (!row) { row = {y: dot.y, dots: []}; rows.push(row); }
      row.dots.push(dot);
    }
    rows.sort((a,b) => b.dots.length - a.dots.length);
    const row = rows[0];
    row.dots.sort((a,b) => a.x - b.x);
    const adjacent = [];
    for (const dot of row.dots) {
      if (!adjacent.length || dot.x - adjacent[adjacent.length - 1].x <= 28) adjacent.push(dot);
    }
    return Math.min(5, adjacent.length);
  }
  function currentBossElement() {
    // Đọc hệ boss theo nhiều kiểu nhãn, không phụ thuộc đúng một chuỗi
    // "Ngũ hành boss: ...". Ưu tiên văn bản có ngữ cảnh boss/hệ/ngũ hành.
    const outsideRaw = document.body?.innerText || '';
    const outside = norm(outsideRaw.replace(panel.innerText, ''));
    const elements = [
      ['Kim', /\bkim\b/i],
      ['Mộc', /\bmộc\b/i],
      ['Thủy', /\bthủy\b/i],
      ['Hỏa', /\bhỏa\b/i],
      ['Thổ', /\bthổ\b/i]
    ];

    const patterns = [
      /ngũ hành\s*boss\s*[:：\-]?\s*(kim|mộc|thủy|hỏa|thổ)(?:$|[^\p{L}])/iu,
      /boss\s*(?:ngũ hành|hệ)?\s*[:：\-]?\s*(kim|mộc|thủy|hỏa|thổ)(?:$|[^\p{L}])/iu,
      /(?:hệ|ngũ hành)\s*(?:của\s*)?boss\s*[:：\-]?\s*(kim|mộc|thủy|hỏa|thổ)(?:$|[^\p{L}])/iu,
      /(?:thuộc|hệ)\s*[:：\-]?\s*(kim|mộc|thủy|hỏa|thổ)\s*(?:hành)?\s*boss(?:$|[^\p{L}])/iu
    ];
    for (const re of patterns) {
      const m = outside.match(re);
      if (m) return m[1][0].toUpperCase() + m[1].slice(1);
    }
    // Fallback cho giao diện tách nhãn và giá trị thành các dòng kế tiếp.
    const lines = outside.split(/[\n\r]+/).map(x => x.trim()).filter(Boolean);
    for (let i = 0; i < lines.length; i++) {
      if (/(?:ngũ hành\s*boss|boss(?:\s+ngũ hành)?|hệ\s*boss|thuộc tính boss)/i.test(lines[i])) {
        const joined = `${lines[i]} ${lines[i + 1] || ''}`;
        const m = joined.match(/(?:^|[^\p{L}])(kim|mộc|thủy|hỏa|thổ)(?:$|[^\p{L}])/iu);
        if (m) return m[1][0].toUpperCase() + m[1].slice(1);
      }
    }

    // Một số giao diện tách nhãn và giá trị thành các phần tử DOM riêng.
    const candidates = $$('body *').filter(el =>
      el !== panel && !panel.contains(el) && visible(el) &&
      el.children.length <= 2
    );
    for (const el of candidates) {
      const label = norm([
        el.innerText, el.textContent, el.getAttribute('aria-label'),
        el.getAttribute('title'), el.getAttribute('alt'),
        el.getAttribute('data-testid'), el.className
      ].filter(Boolean).join(' '));
      if (!/(boss|ngũ hành|thuộc tính|hệ)/i.test(label)) continue;
      for (const [name, re] of elements) {
        if (re.test(label)) return name;
      }
    }

    // Thử thuộc tính ảnh/icon nằm trong vùng có nhắc đến boss.
    const bossContainers = candidates.filter(el =>
      /(boss|ngũ hành|thuộc tính|hệ)/i.test(norm(el.innerText || el.getAttribute('aria-label') || el.getAttribute('title') || ''))
    );
    for (const el of bossContainers) {
      const nodes = [el, ...$$('img,[title],[aria-label],[alt]', el)];
      for (const node of nodes) {
        const meta = norm([
          node.getAttribute?.('alt'), node.getAttribute?.('title'),
          node.getAttribute?.('aria-label'), node.getAttribute?.('src'),
          node.getAttribute?.('class')
        ].filter(Boolean).join(' '));
        for (const [name, re] of elements) {
          if (re.test(meta)) return name;
        }
      }
    }
    return '';
  }
  function isCombat() {
    const t = norm((document.body?.innerText || '').replace(panel.innerText, ''));
    // Dấu hiệu sảnh phải được ưu tiên: "?/5" tại đây là số thành viên.
    const lobbyMarkers = /sao chép link mời|trục xuất|truc xuat|mở khóa|pháp tướng|sẵn sàng/.test(t);
    const battleMarkers = /đầu hàng|trạng thái:\s*(bình thường|đang chiến đấu)|liệt điểm ma cung|đang chiến đấu|xác nhận đầu hàng/.test(t);
    if (lobbyMarkers && !battleMarkers) return false;
    return battleMarkers || !!currentBossElement();
  }
  function isRewardScreen() {
    const t = getVisibleText().replace(norm(panel.innerText), '');
    if (state.mode === 'grass') {
      // Sau khi nhận rương, trang có thể bỏ nhãn hoàn thành nhưng vẫn hiện trạng thái đã nhận.
      // Nhận diện trạng thái đã nhận độc lập để bộ đếm kịp chuyển sang giai đoạn kế tiếp.
      return /thưởng vật phẩm hôm nay đã nhận/.test(t) ||
        (/bấm vào rương để nhận thưởng|bấm để mở rương|mở rương|nhận thưởng/.test(t) &&
         /chinh phục mê cung|đội hình đã vượt qua 5 ải/.test(t));
    }
    // Chiến tích: màn hình rương chinh phục sẵn sàng đã đủ tính là nhận 1 rương.
    // Không đợi thao tác mở rương hoặc dòng xác nhận sau đó.
    if (state.mode === 'achievement') {
      return /chinh phục mê cung/.test(t) && /rương đã sẵn sàng|bấm để mở rương|bấm vào rương để nhận thưởng/.test(t);
    }
    // Farm: ghi nhận ngay khi đã tới màn hình kết quả/rương của một lượt.
    // Các dấu hiệu được giới hạn ở ngữ cảnh hoàn thành để tránh đếm chữ "phần thưởng"
    // xuất hiện ở các khu vực giao diện khác.
    return /thưởng vật phẩm hôm nay đã nhận|rương đã sẵn sàng|bấm để mở rương|bấm vào rương để nhận thưởng|nhận thưởng|rương.*(?:cuối|phần thưởng)|chúc mừng.*(?:hoàn thành|chiến thắng)|hoàn thành.*mê cung|đội hình đã vượt qua 5 ải|chinh phục mê cung|phần thưởng/.test(t);
  }
  function hasConfirmation() {
    const t = getVisibleText().replace(norm(panel.innerText), '');
    return /xác nhận hành động|xác nhận đầu hàng|xác nhận trục xuất|trục xuất thành viên|bắt đầu chinh phục mê cung/.test(t);
  }
  function confirmIfNeeded() {
    const t = getVisibleText().replace(norm(panel.innerText), '');
    if (/xác nhận đầu hàng/.test(t)) {
      // Farm: xác nhận đầu hàng để thoát lượt và quay về sảnh chạy vòng mới.
      if (['farm','achievement'].includes(state.mode)) return click(findClickable(/^xác nhận$/i), `${state.mode}: xác nhận đầu hàng`);
      // Chỉ xác nhận đầu hàng nếu chế độ Cỏ và boss sai hệ.
      if (state.mode === 'grass' && isCombat()) {
        const step = activePlanStep();
        const boss = currentBossElement();
        if (userOptions.surrenderKimNightmare && step?.difficulty === 'Ác Mộng' && boss === 'Kim') {
          return click(findClickable(/^xác nhận$/i), 'Ác Mộng gặp boss Kim: xác nhận đầu hàng');
        }
        const stage = currentStage();
        const expected = state.farmElements[stage - 1] || [];
        if (stage >= 1 && stage <= 5 && boss && expected.length && !expected.includes('Bỏ') && !expected.some(e => e.toLowerCase() === boss.toLowerCase())) {
          return click(findClickable(/^xác nhận$/i), 'Xác nhận đầu hàng (khác hệ)');
        }
      }
      log('Đang có hộp xác nhận đầu hàng; không tự xác nhận để tránh thao tác nhầm.', 'warn');
      setStatus('Chờ xác nhận an toàn');
      return false;
    }
    // Hộp xác nhận bắt đầu dùng chung cho mọi giai đoạn.
    if (/bắt đầu chinh phục mê cung/.test(t)) {
      const confirmStart = findClickable(/^xác nhận$/i);
      if (confirmStart) return click(confirmStart, `Xác nhận bắt đầu Mê Cung · ${state.mode}`);
      return false;
    }
    if (['farm','achievement','grass'].includes(state.mode) && /xác nhận hành động/.test(t)) return click(findClickable(/^xác nhận$/i), 'Farm: xác nhận thao tác');
    return false;
  }
  async function waitWhilePaused() {
    while (state.running && state.paused) await sleep(250);
  }
  async function safeAction(fn) {
    if (!state.running || state.paused || state.busy || Date.now() < state.retryAfter) return;
    state.busy = true;
    try { await fn(); } catch (e) { log(`Lỗi: ${e?.message || e}`, 'warn'); }
    finally { state.busy = false; }
  }

  async function chooseDifficulty(name) {
    const text = getVisibleText().replace(norm(panel.innerText), '');
    if (isCombat() || /đang chiến đấu|combat/.test(text)) {
      log('Không đổi độ khó khi đang trong ải.', 'warn'); return false;
    }
    const change = findClickable(/đổi/i);
    if (change) {
      click(change, 'Mở đổi độ khó');
      await sleep(500);
    }
    const option = findClickable(new RegExp('^' + name.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') + '$', 'i'));
    if (option) {
      click(option, `Chọn độ khó ${name}`);
      await sleep(500);
      // Một số giao diện yêu cầu xác nhận riêng sau khi chọn độ khó.
      const bodyAfterSelect = getVisibleText().replace(norm(panel.innerText), '');
      if (/xác nhận.{0,25}(?:độ khó|thay đổi)|thay đổi.{0,25}độ khó/i.test(bodyAfterSelect)) {
        const confirmDifficulty = findClickable(/^xác nhận$/i);
        if (confirmDifficulty) {
          click(confirmDifficulty, `Xác nhận thay đổi độ khó ${name}`);
          await sleep(500);
        }
      }
      const close = findClickable(/^đóng$/i);
      if (close) click(close, 'Đóng hộp độ khó');
      state.difficulty = name;
      return true;
    }
    log(`Không tìm thấy lựa chọn độ khó "${name}". Hãy kiểm tra giao diện.`, 'warn');
    return false;
  }

  function resetPlanToStage1(reason) {
    if (!state.running || !Array.isArray(state.plan)) return;
    state.planCompletedAwaitingKick = false;
    let first = state.plan.findIndex(step => step && step.type !== 'skip');
    if (first < 0) first = 0;
    state.planIndex = first;
    state.chestCount = 0;
    state.difficultyApplied = false;
    state.matchStartPlayerCount = 0;
    state.perfectRewardHandled = false;
    state.stageDone.clear();
    state.lastCombatStage = 0;
    state.farmScanStage = 0;
    state.farmScanUntil = 0;
    const step = state.plan[first];
    if (step && step.type !== 'skip') {
      state.mode = step.type;
      state.difficulty = step.difficulty;
      if (ui.mode) ui.mode.value = step.type;
    }
    state.lobbyWaitUntil = Date.now() + 3000;
    state.lastCombatRoster = '';
    log(`${reason} · tự động quay lại giai đoạn ${first + 1}.`, 'warn');
    saveRunProgress();
  }

  async function handleLobby() {
    const lobbyCount = roomPlayerCount();
    if (state.running && lobbyCount > 0 && lobbyCount < roomSettings.requiredPlayers) {
      if (!underfilledLobbySince) { underfilledLobbySince = Date.now(); lastUnderfilledInviteAt = Date.now(); }
      else if (Date.now() - underfilledLobbySince >= INVITE_INTERVAL_MS && Date.now() - lastUnderfilledInviteAt >= INVITE_INTERVAL_MS) {
        lastUnderfilledInviteAt = Date.now(); await maybeSendInviteMessage(true, false, false); return;
      }
    } else { underfilledLobbySince = 0; lastUnderfilledInviteAt = 0; }
    state.perfectRewardHandled = false;
    // So sánh số người trước khi vào trận với số người khi quay lại sảnh.
    // Chỉ xác nhận thành viên đã rời giữa trận khi số người hiện tại thấp hơn mốc ban đầu.
    if (state.matchStartPlayerCount > 0 && lobbyCount > 0 && lobbyCount < state.matchStartPlayerCount) {
      const before = state.matchStartPlayerCount;
      state.matchStartPlayerCount = 0;
      lastRoomFingerprint = roomFingerprint();
      resetPlanToStage1(`Có người thoát giữa trận (${before} → ${lobbyCount} người)`);
      if (lobbyCount > 0 && lobbyCount < roomSettings.requiredPlayers) await maybeSendInviteMessage(true, false, false);
      return;
    }
    if (state.matchStartPlayerCount > 0 && lobbyCount >= state.matchStartPlayerCount) state.matchStartPlayerCount = 0;
    const fingerprint = roomFingerprint();
    if (fingerprint) {
      if (lastRoomFingerprint && fingerprint !== lastRoomFingerprint && lobbyCount > 0) {
        lastRoomFingerprint = fingerprint;
        resetPlanToStage1('Phát hiện thành viên rời phòng hoặc bị trục xuất');
        if (lobbyCount > 0 && lobbyCount < roomSettings.requiredPlayers) await maybeSendInviteMessage(true, false, false);
        return;
      } else lastRoomFingerprint = fingerprint;
    }
    if (state.planCompletedAwaitingKick) {
      const cleanup = handleUnreadyPlayers(true);
      if (cleanup.kicked) return;
      if (cleanup.pending > 0) {
        setStatus(`Đã xong các giai đoạn · chờ ${cleanup.pending} người chưa sẵn sàng hết thời gian để trục xuất`);
        return;
      }
      state.planCompletedAwaitingKick = false;
      resetPlanToStage1('Đã hoàn thành toàn bộ kế hoạch; kết thúc lượt trục xuất');
      syncActivePlanStep();
    }
    syncActivePlanStep();
    if (!state.running) return;
    const step = activePlanStep();
    if (!step) { stop('Đã hoàn thành toàn bộ kế hoạch 3 giai đoạn'); return; }
    if (!state.lobbyWaitUntil) state.lobbyWaitUntil = Date.now() + 3000;
    if (Date.now() < state.lobbyWaitUntil) {
      setStatus(`Giai đoạn ${state.planIndex+1} · chờ ${Math.ceil((state.lobbyWaitUntil-Date.now())/1000)}s`); return;
    }
    const actual = currentDifficulty();
    const desired = step.difficulty;
    ui.diff.textContent = actual || desired || 'Chưa xác định';
    if (actual && actual !== desired) {
      state.difficultyApplied = false;
      const changed = await chooseDifficulty(desired);
      if (changed) { state.difficultyApplied = true; await sleep(500); return; }
      state.retryAfter=Date.now()+1500; return;
    }
    if (actual === desired) state.difficultyApplied = true;
    if (!actual && !state.difficultyApplied) {
      const changed = await chooseDifficulty(desired);
      if (changed) { state.difficultyApplied = true; await sleep(500); return; }
      log(`Không xác định được độ khó hiện tại hoặc không thể chọn ${desired}; tạm dừng để tránh chạy sai giai đoạn.`, 'warn');
      setStatus('Không xác định được độ khó'); state.running = false; return;
    }
    const start = findClickable(/^bắt đầu$/i);
    if (!start) { setStatus('Đang chờ giao diện sảnh hoặc nút Bắt đầu.'); state.retryAfter=Date.now()+2500; return; }
    const players = roomPlayerCount();
    const readiness = handleUnreadyPlayers(); if (readiness.kicked) return;
    if (players < roomSettings.requiredPlayers) { setStatus(`Đợi đủ người: ${players}/${roomSettings.requiredPlayers} · tối thiểu ${roomSettings.requiredPlayers}`); return; }
    if (readiness.unready > 0) { setStatus(`Đợi ${readiness.unready} người sẵn sàng`); return; }
    const waitMs = 11001 - (Date.now() - state.lastStartAt);
    if (state.lastStartAt && waitMs > 0) { setStatus(`Chờ ${Math.ceil(waitMs/1000)}s trước lượt tiếp theo`); state.retryAfter=Date.now()+waitMs; return; }
    // Lưu mốc số người ngay trước khi bắt đầu để phát hiện người thoát giữa trận.
    state.matchStartPlayerCount = players;
    state.interruptionResetApplied = false;
    state.lastStartAt=Date.now(); state.lobbyWaitUntil=0;
    saveRunProgress();
    click(start, `Bắt đầu · Giai đoạn ${state.planIndex+1} · ${step.type} · ${step.difficulty}`);
    for (let attempt=0; attempt<5; attempt++) { await sleep(500); const t=getVisibleText().replace(norm(panel.innerText),''); if (/bắt đầu chinh phục mê cung/.test(t)) { const c=findClickable(/^xác nhận$/i); if(c) click(c,`Xác nhận bắt đầu giai đoạn ${state.planIndex+1}`); break; } }
    setStatus(`Đang vào ải · Giai đoạn ${state.planIndex+1}`); await sleep(900);
  }

  async function handleCombat() {
    const stage = currentStage();
    const boss = currentBossElement();
    ui.stage.textContent = `${stage || '?'} / 5${boss ? ' · ' + boss : ''}`;
    const activeStep = activePlanStep();
    // Quy tắc dùng chung cho mọi chế độ/giai đoạn: nếu độ khó của giai đoạn là Ác Mộng
    // và nhận diện được boss Kim, đầu hàng ngay khi bật tùy chọn trong Setting.
    if (userOptions.surrenderKimNightmare && activeStep?.difficulty === 'Ác Mộng' && boss === 'Kim') {
      const bodyText = getVisibleText().replace(norm(panel.innerText), '');
      setStatus(`Ác Mộng · gặp boss Kim ở ải ${stage || '?'}: đang đầu hàng`);
      if (/xác nhận đầu hàng/.test(bodyText)) {
        const confirm = findClickable(/^xác nhận$/i);
        if (confirm) { click(confirm, 'Ác Mộng gặp boss Kim: xác nhận đầu hàng'); await sleep(400); }
      } else {
        const abandon = findClickable(/đầu hàng/i);
        if (abandon) { click(abandon, 'Ác Mộng gặp boss Kim: đầu hàng'); await sleep(400); }
      }
      return;
    }
    if (state.mode === 'grass') {
      if (!stage || stage < 1 || stage > 5) {
        setStatus('Đang dò số ải'); return;
      }

      // Khi thấy số ải tăng, ải trước đã được vượt qua. Ghi nhớ qua các lần đầu hàng
      // và bắt đầu lại; chỉ xóa danh sách này khi nhận rương chinh phục cuối chu trình.
      const previousStage = state.lastCombatStage;
      if (previousStage >= 1 && previousStage < stage && !(state.farmElements[previousStage - 1] || []).includes('Bỏ') && !state.stageDone.has(previousStage)) {
        state.stageDone.add(previousStage);
        log(`Ghi nhớ ải ${previousStage} đã vượt qua; sẽ bỏ qua dò hệ ở lần bắt đầu lại.`, 'good');
      }
      state.lastCombatStage = stage;

      // Ải được chọn 'Bỏ': không dò ngũ hành, không đầu hàng theo hệ; cứ bật AutoAttack.
      // Không đưa ải này vào danh sách stageDone vì đây là lựa chọn bỏ dò, không phải xác nhận hoàn thành.
      if ((state.farmElements[stage - 1] || []).includes('Bỏ')) {
        state.farmScanStage = 0;
        setStatus(`Ải ${stage}: đã chọn Bỏ — bỏ qua dò ngũ hành`);
        const auto = autoAttackControl() || findClickable(/tự động/i);
        if (auto && !autoAttackIsOn(auto)) click(auto, `Cỏ: bỏ dò ngũ hành ải ${stage}, bật AutoAttack`);
        return;
      }

      // Ải đã vượt qua: không dò hệ và không đầu hàng, bật AutoAttack để đi tiếp.
      if (state.stageDone.has(stage)) {
        state.farmScanStage = 0;
        setStatus(`Ải ${stage} đã hoàn thành trước đó; bỏ qua dò hệ`);
        const auto = autoAttackControl();
        if (auto && !autoAttackIsOn(auto)) click(auto, `Bật AutoAttack bỏ qua ải ${stage} đã hoàn thành`);
        return;
      }

      // Mỗi lần gặp ải chưa hoàn thành: tắt AutoAttack và chờ đủ 0.3 giây để đọc nhãn boss.
      if (state.farmScanStage !== stage) {
        state.farmScanStage = stage;
        state.farmScanUntil = Date.now() + 300;
        stopAutoAttackForScan();
        setStatus(`Tạm tắt AutoAttack · dò hệ boss 0.3 giây (ải ${stage})`);
        log(`Ải ${stage}: tắt AutoAttack và chờ 0.3 giây để nhận diện ngũ hành boss.`);
        return;
      }
      if (Date.now() < state.farmScanUntil) {
        stopAutoAttackForScan();
        setStatus(`Đang nhận diện hệ boss: ${Math.ceil((state.farmScanUntil - Date.now()) / 1000)}s`);
        return;
      }

      const detectedBoss = currentBossElement();
      if (!detectedBoss) {
        setStatus('Chưa nhận diện được ngũ hành boss; tiếp tục dò');
        return;
      }
      const expected = state.farmElements[stage - 1] || [];
      if (!expected.some(e => e.toLowerCase() === detectedBoss.toLowerCase())) {
        setStatus(`Sai hệ: ${detectedBoss} không thuộc ${expected.join(' / ')}`);
        // Không lặp click đầu hàng khi hộp xác nhận đã mở; chờ đúng hộp thoại rồi xác nhận.
        const bodyText = getVisibleText().replace(norm(panel.innerText), '');
        if (/xác nhận đầu hàng/.test(bodyText)) {
          const confirm = findClickable(/^xác nhận$/i);
          if (confirm) {
            click(confirm, `Xác nhận đầu hàng ải ${stage} sai hệ`); await sleep(500);
            state.farmScanStage = 0;
            state.retryAfter = Math.max(Date.now() + 1000, state.lastStartAt + 11001);
            setStatus(`Đã đầu hàng ải ${stage}; chờ rồi bắt đầu dò lại ải này`);
          }
          return;
        }
        const abandon = findClickable(/đầu hàng/i);
        if (abandon) {
          click(abandon, `Đầu hàng ngay ải ${stage} (${detectedBoss}, cần ${expected.join(' / ')})`);
          await sleep(250);
          const afterText = getVisibleText().replace(norm(panel.innerText), '');
          if (/xác nhận đầu hàng/.test(afterText)) {
            const confirm = findClickable(/^xác nhận$/i);
            if (confirm) {
              click(confirm, `Xác nhận đầu hàng ải ${stage} sai hệ`); await sleep(500); await sleep(500);
              state.farmScanStage = 0;
              state.retryAfter = Math.max(Date.now() + 1000, state.lastStartAt + 11001);
              setStatus(`Đã đầu hàng ải ${stage}; chờ rồi bắt đầu dò lại ải này`);
            }
          } else {
            state.retryAfter = Date.now() + 800;
            log('Đã yêu cầu đầu hàng; đang chờ giao diện xác nhận.', 'warn');
          }
        } else {
          setStatus('Sai hệ nhưng chưa thấy nút đầu hàng');
        }
        return;
      }

      setStatus(`Đúng hệ ${detectedBoss}; bật AutoAttack`);
      const auto = autoAttackControl() || findClickable(/tự động/i);
      if (auto && !autoAttackIsOn(auto)) click(auto, 'Bật AutoAttack sau khi xác nhận đúng hệ');
      return;
    }
    setStatus(`Đang chiến đấu${stage ? ` — ải ${stage}` : ''}`);
    const auto = findClickable(/tự động/i);
    if (auto && !/đang tự động|tắt tự động/.test(textOf(auto))) click(auto, 'Bật tự động chiến đấu');
  }

  async function handleReward() {
    const t = getVisibleText().replace(norm(panel.innerText), '');
    setStatus(`Giai đoạn ${state.planIndex+1}: xử lý rương chinh phục`);
    const chestReady = state.mode === 'achievement' && /chinh phục mê cung/.test(t) &&
      /rương đã sẵn sàng|bấm để mở rương|bấm vào rương để nhận thưởng/.test(t);
    // Với Chiến tích, chỉ cần thấy màn hình rương chinh phục sẵn sàng là tính đã nhận rương.
    const farmChestReady = state.mode === 'farm' && (
      /thưởng vật phẩm hôm nay đã nhận|rương đã sẵn sàng|bấm để mở rương|bấm vào rương để nhận thưởng/.test(t) ||
      (/chúc mừng.*(?:hoàn thành|chiến thắng)|đội hình đã vượt qua 5 ải|hoàn thành.*mê cung/.test(t) && /rương|thưởng|chinh phục/.test(t))
    );
    const received = /thưởng vật phẩm hôm nay đã nhận/.test(t) || chestReady || farmChestReady;
    const completed = /đội hình đã vượt qua 5 ải|chinh phục mê cung|hoàn thành.*mê cung/.test(t);
    const returnButton = findClickable(/quay về sảnh|về sảnh|trở về sảnh/i);
    const chest = findClickable(/bấm vào rương để nhận thưởng|bấm để mở rương|mở rương|nhận thưởng/i);
    // Dòng 'đã nhận thưởng' là tín hiệu xác nhận rương đã được nhận. Không bắt buộc
    // phải còn thấy dòng 'chinh phục mê cung' cùng lúc, vì giao diện có thể đổi trạng thái sau khi nhận.
    if (received) {
      if (!state.perfectRewardHandled) {
        const step = activePlanStep();
        state.perfectRewardHandled = true;
        state.chestCount = (Number(state.chestCount)||0) + 1;
        const target = step ? (step.type === 'farm' ? Math.max(1, Number(step.count)||1) : 1) : 1;
        log(`Giai đoạn ${state.planIndex+1}: đã nhận rương ${state.chestCount}/${target}.`, 'good');
        if (state.chestCount >= target) {
          const finished = state.planIndex + 1;
          state.planIndex++;
          state.chestCount = 0; state.difficultyApplied=false;
          while (state.planIndex < state.plan.length && state.plan[state.planIndex].type === 'skip') {
            log(`Giai đoạn ${state.planIndex+1} đặt --/--: bỏ qua và chuyển tiếp.`, 'good'); state.planIndex++;
          }
          if (state.planIndex >= state.plan.length) {
            state.planIndex = state.plan.length;
            state.planCompletedAwaitingKick = true;
            log(`Đã hoàn tất giai đoạn ${finished}; toàn bộ kế hoạch đã đủ rương. Chờ về sảnh để xử lý trục xuất.`, 'good');
          } else {
            state.mode = state.plan[state.planIndex].type === 'grass' ? 'grass' : state.plan[state.planIndex].type;
            state.difficulty = state.plan[state.planIndex].difficulty;
            state.stageDone.clear(); state.lastCombatStage=0; state.farmScanStage=0;
            log(`Chuyển sang giai đoạn ${state.planIndex+1}: ${planTypeLabel(state.mode)} · ${state.difficulty}.`, 'good');
          }
        }
        saveRunProgress();
      }
      if (returnButton) {
        click(returnButton, 'Quay về sảnh sau khi nhận rương chinh phục');
        await sleep(700);
        if (state.planIndex >= state.plan.length) {
          state.planCompletedAwaitingKick = true;
          state.lobbyWaitUntil = 0;
          saveRunProgress();
        } else { state.lobbyWaitUntil=Date.now()+3000; state.perfectRewardHandled=false; syncActivePlanStep(); }
        return;
      }
      setStatus('Đã nhận rương; chờ nút Quay về sảnh'); state.retryAfter=Date.now()+700; return;
    }
    if (completed && chest && state.mode !== 'achievement') { click(chest, 'Mở/nhận rương chinh phục'); await sleep(700); return; }
    if (returnButton && /rương|thưởng|hoàn thành|chinh phục/.test(t)) { click(returnButton,'Quay về sảnh sau màn thưởng'); await sleep(700); return; }
    setStatus('Chờ xác nhận nhận rương chinh phục'); state.retryAfter=Date.now()+1000;
  }

  function countDeadPlayers() {
    const deadRe = /(?:đã chết|đang chết|tử vong|đã tử vong|bị hạ gục|đã bị hạ|dead|defeated)/i;
    const selectors = [
      '.room-member','.member-item','.player-item','.room-user','.user-item',
      '.lobby-player','.player-card','.char-card','.member','.players .item',
      '.player-list .item','.member-list .member','.user-list .user',
      '[class*="player"]','[class*="member"]'
    ];
    const containers = [...new Set(selectors.flatMap(sel => $$(sel)))].filter(el =>
      el !== panel && !panel.contains(el) && visible(el) && el.children.length < 12
    );
    const marked = new Set();
    for (const el of containers) {
      const t = norm([el.innerText, el.getAttribute('aria-label'), el.getAttribute('title'), el.className].join(' '));
      if (deadRe.test(t)) {
        // Avoid counting a nested status and its enclosing player card twice.
        const hasDeadAncestor = [...marked].some(parent => parent.contains(el));
        if (!hasDeadAncestor) {
          for (const child of [...marked]) if (el.contains(child)) marked.delete(child);
          marked.add(el);
        }
      }
    }
    if (marked.size) return marked.size;
    // Fallback for interfaces that render each death as a standalone status label.
    const body = getVisibleText().replace(norm(panel.innerText), '');
    const matches = body.match(/(?:đã chết|đang chết|tử vong|đã tử vong|bị hạ gục|đã bị hạ|dead|defeated)/gi);
    return matches ? matches.length : 0;
  }

  async function handleDeadPlayerRetreat() {
    const t = getVisibleText().replace(norm(panel.innerText), '');
    // Đã bỏ cơ chế đầu hàng tự động theo số người chết.
    // Không tự đầu hàng theo số người chết.
    if (!state.deadRetreatPending) return false;
    if (!state.deadRetreatPending) return false;

    const confirm = findClickable(/^xác nhận$/i);
    if (/xác nhận đầu hàng|xác nhận hành động/.test(t) && confirm) {
      click(confirm, 'Xác nhận đầu hàng khi có hơn 2 người chết');
      await sleep(400);
      return true;
    }
    if (/thất bại|đầu hàng thành công|đã đầu hàng|kết thúc lượt/.test(t)) {
      const back = findClickable(/quay về phòng|về phòng|quay về sảnh|về sảnh|trở về phòng|trở về sảnh|xác nhận|đồng ý/i);
      if (back) {
        click(back, 'Quay về phòng sau khi đầu hàng do hơn 2 người chết');
        state.deadRetreatPending = false;
        state.retryAfter = Date.now() + 1000;
        await sleep(700);
        return true;
      }
    }
    if (isCombat()) {
      const abandon = findClickable(/đầu hàng/i);
      if (abandon && !/xác nhận đầu hàng/.test(t)) {
        click(abandon, 'Đầu hàng khi có hơn 2 người chết');
        await sleep(350);
        const after = getVisibleText().replace(norm(panel.innerText), '');
        if (/xác nhận đầu hàng|xác nhận hành động/.test(after)) {
          const yes = findClickable(/^xác nhận$/i);
          if (yes) click(yes, 'Xác nhận đầu hàng khi có hơn 2 người chết');
        }
        return true;
      }
    }
    state.retryAfter = Date.now() + 800;
    return true;
  }

  async function handleFarmEndState() {
    if (!['farm','achievement','grass'].includes(state.mode)) return false;
    const t = getVisibleText().replace(norm(panel.innerText), '');
    if (!/(thất bại|that bai|chinh phục thất bại|khiêu chiến thất bại|đã thất bại|bạn đã thua|đầu hàng thành công|đã đầu hàng)/i.test(t)) return false;
    const confirm = findClickable(/^xác nhận$/i);
    if (/xác nhận đầu hàng|xác nhận hành động/.test(t) && confirm) {
      click(confirm, `${state.mode}: xác nhận kết quả/đầu hàng`); await sleep(500); return true;
    }
    const back = findClickable(/^(quay về phòng|về phòng|quay về sảnh|về sảnh|trở về phòng|trở về sảnh|xác nhận|đồng ý)$/i);
    if (back) {
      click(back, `${state.mode}: xác nhận kết thúc lượt và quay về sảnh`);
      setStatus(`${state.mode}: kết thúc lượt, quay về sảnh để thử lại giai đoạn hiện tại`);
      await sleep(700); return true;
    }
    setStatus(`${state.mode}: chờ nút xác nhận/quay về sảnh sau khi kết thúc lượt`);
    state.retryAfter = Date.now() + 1000; return true;
  }

  async function tick() {
    if (!state.running || state.paused) return;
    ui.diff.textContent = currentDifficulty() || state.difficulty || 'Chưa xác định';
    state.deadRetreatPending = false;
    if (hasConfirmation()) {
      confirmIfNeeded();
      return;
    }
    if (isRewardScreen()) { await handleReward(); return; }
    if (await handleFarmEndState()) return;
    if (isCombat()) {
      const battleText = norm(getVisibleText().replace(norm(panel.innerText), ''));
      if (/(?:đã rời|vừa rời|thoát khỏi|bị trục xuất|đã thoát).{0,45}(?:phòng|trận|đội)|(?:phòng|trận|đội).{0,45}(?:đã rời|vừa rời|thoát khỏi|bị trục xuất|đã thoát)/i.test(battleText)) {
        if (!state.interruptionResetApplied) {
          state.interruptionResetApplied = true;
          resetPlanToStage1('Có thành viên rời/thoát hoặc bị trục xuất giữa trận');
        }
      }
      await handleCombat(); return;
    }
    state.interruptionResetApplied = false;
    await handleLobby();
  }

  function loadPlanFromUI() {
    return [1,2,3].map(n => ({
      type: $(`[data-plan-type="${n}"]`, panel)?.value || (n===1?'grass':'skip'),
      difficulty: $(`[data-plan-difficulty="${n}"]`, panel)?.value || 'Khó',
      count: clampInt($(`[data-plan-count="${n}"]`, panel)?.value,1,999,1)
    }));
  }
  function normalizePlan(plan) {
    const allowed = ['grass','achievement','farm','skip'];
    return [1,2,3].map((n,i) => {
      const x=plan?.[i]||{};
      let type=allowed.includes(x.type)?x.type:(i===0?'grass':'skip');
      // Cỏ chỉ được phép ở giai đoạn 1; chuẩn hóa cấu hình cũ không hợp lệ.
      if (i > 0 && type === 'grass') type = 'achievement';
      return {type, difficulty:['Thường','Khó','Ác Mộng'].includes(x.difficulty)?x.difficulty:'Khó', count:clampInt(x.count,1,999,1)};
    });
  }
  function applyPlanToUI(plan) {
    normalizePlan(plan).forEach((x,i) => {
      const n=i+1, type=$(`[data-plan-type="${n}"]`,panel), diff=$(`[data-plan-difficulty="${n}"]`,panel), count=$(`[data-plan-count="${n}"]`,panel);
      if(type) type.value=x.type; if(diff) diff.value=x.difficulty; if(count) count.value=String(x.count);
      const wrap=$(`[data-plan-count-wrap="${n}"]`,panel); if(wrap) wrap.style.display=x.type==='farm'?'grid':'none';
      const diffLabel=$(`[data-plan-difficulty="${n}"]`,panel)?.closest('label'); if(diffLabel) diffLabel.style.display=x.type==='skip'?'none':'grid';
    });
    // Chỉ hiển thị lựa chọn Ngũ hành khi giai đoạn 1 đang chọn Cỏ.
    if (ui.elements) ui.elements.style.display = normalizePlan(plan)[0]?.type === 'grass' ? 'block' : 'none';
  }
  function planTypeLabel(type) { return type==='grass'?'Cỏ':type==='achievement'?'Chiến tích':type==='farm'?'Farm':'--/--'; }
  function activePlanStep() { return state.plan?.[state.planIndex] && state.plan[state.planIndex].type!=='skip' ? state.plan[state.planIndex] : null; }
  function syncActivePlanStep() {
    while(state.planIndex < (state.plan||[]).length && state.plan[state.planIndex].type==='skip') state.planIndex++;
    const step=activePlanStep();
    if(!step) return null;
    state.mode=step.type; ui.mode.value=step.type; state.difficulty=step.difficulty;
    ui.elements.style.display=step.type==='grass'?'block':'none';
    applyPlanToUI(state.plan);
    return step;
  }
  function savePlan() { try { localStorage.setItem(PLAN_KEY, JSON.stringify(loadPlanFromUI())); } catch(_){} }
  function start() {
    try { sessionStorage.setItem(USER_STARTED_RUN_KEY, '1'); } catch (_) {}
    state.plan=normalizePlan(loadPlanFromUI()); state.planIndex=0; state.chestCount=0;
    while(state.planIndex<state.plan.length && state.plan[state.planIndex].type==='skip') state.planIndex++;
    if(state.planIndex>=state.plan.length) { log('Hãy chọn ít nhất một giai đoạn khác --/--.','warn'); return; }
    try { localStorage.setItem(PLAN_KEY,JSON.stringify(state.plan)); } catch(_){}
    state.mode=state.plan[state.planIndex].type; ui.mode.value=state.mode;
    state.farmElements = $$('.mc-stage-elements[data-stage]', panel).map(box => { const chosen = $$('input[data-element]:checked', box).map(el => el.dataset.element); return chosen.length ? chosen : ['Kim']; });
    state.running=true; state.paused=false; state.busy=false; state.planCompletedAwaitingKick=false;
    state.difficulty=state.plan[state.planIndex].difficulty; state.difficultyApplied=false;
    state.startedAt=Date.now(); state.stageDone.clear(); state.lastStartAt=0; state.lastCombatStage=0; state.farmScanUntil=0; state.farmScanStage=0; state.retryAfter=0; state.lobbyWaitUntil=0; state.stoppedReason=''; state.perfectRewardHandled=false;
    lastRoomFingerprint=roomFingerprint(); saveRunProgress(); syncActivePlanStep();
    setStatus('Đang chạy'); log(`Bắt đầu kế hoạch: GĐ ${state.planIndex+1} · ${planTypeLabel(state.mode)} · ${state.difficulty}${state.mode==='farm'?` · ${state.plan[state.planIndex].count} rương`:''}.`);
    void maybeSendInviteMessage(false,true); tick();
  }

  function clearRunProgress() {
    // Tiến trình chỉ thuộc một lần bấm Bắt đầu; Dừng sẽ xóa sạch để lần sau chạy mới.
    state.stageDone.clear();
    state.planIndex=0; state.chestCount=0;
    state.lastCombatStage = 0;
    state.farmScanUntil = 0;
    state.farmScanStage = 0;
    state.lastStartAt = 0;
    state.perfectStage = 1; state.perfectStartStage = 1;
    state.perfectRewardHandled = false; state.perfectAdvancePending = false; state.deadRetreatPending = false;
    state.perfectStage3ChestCount = 0; state.perfectStopAfterReturn = false; state.perfectKickIssued = false; state.perfectKickPending = false;
    state.retryAfter = 0;
    state.lobbyWaitUntil = 0;
    state.seenStage = '';
    state.lastAction = '';
    state.lastActionAt = 0;
    state.matchStartPlayerCount = 0;
    state.interruptionResetApplied = false;
  }

  function stop(reason = 'Đã dừng theo yêu cầu') {
    state.running = false; state.paused = false; state.busy = false;
    try { sessionStorage.removeItem(USER_STARTED_RUN_KEY); } catch (_) {}
    clearRunProgress();
    clearSavedRunProgress();
    state.stoppedReason = reason;
    setStatus('Đã dừng');
    if (ui.stage) ui.stage.textContent = '—';
    log(`${reason} · Đã xóa toàn bộ tiến trình; lần Bắt đầu sau sẽ ghi nhớ mới.`, 'warn');
  }

  $('#mc-start', panel).addEventListener('click', () => { panelOpen = true; start(); updatePanelVisibility(); });
  launcher.addEventListener('click', () => { panelOpen = true; updatePanelVisibility(); log('Panel đã mở. Chọn chế độ rồi bấm Auto để bắt đầu.'); });
  $('#mc-pause', panel).addEventListener('click', () => {
    if (!state.running) { log('Chưa bắt đầu chạy.', 'warn'); return; }
    state.paused = !state.paused;
    setStatus(state.paused ? 'Đã tạm dừng' : 'Đang chạy');
    $('#mc-pause', panel).textContent = state.paused ? '▶ Tiếp tục' : '⏸ Tạm dừng';
    log(state.paused ? 'Tạm dừng thao tác tự động bắt đầu; giữ nguyên giai đoạn hiện tại và tiến trình.' : 'Tiếp tục từ tiến trình đã giữ.');
    saveRunProgress();
  });
  $('#mc-stop', panel).addEventListener('click', () => stop());
  // Mỗi ải chọn tối đa hai hệ; Bỏ là lựa chọn độc quyền.
  $$('.mc-stage-elements', panel).forEach(box => {
    box.addEventListener('change', ev => {
      const input = ev.target;
      if (!input.matches('input[data-element]')) return;
      const checks = $$('input[data-element]', box);
      const skip = checks.find(el => el.dataset.element === 'Bỏ');
      const chosen = checks.filter(el => el.checked && el !== skip);
      if (input === skip && skip.checked) {
        checks.forEach(el => { if (el !== skip) el.checked = false; });
      } else if (input !== skip && input.checked) {
        if (skip) skip.checked = false;
        const others = checks.filter(el => el.checked && el !== skip);
        if (others.length > 2) input.checked = false;
      }
      if (!checks.some(el => el.checked)) {
        const first = checks.find(el => el.dataset.element !== 'Bỏ');
        if (first) first.checked = true;
      }
    });
  });

  [1,2,3].forEach(n => {
    const type=$(`[data-plan-type="${n}"]`,panel);
    type?.addEventListener('change', () => { applyPlanToUI(loadPlanFromUI()); savePlan(); });
    $(`[data-plan-difficulty="${n}"]`,panel)?.addEventListener('change',savePlan);
    $(`[data-plan-count="${n}"]`,panel)?.addEventListener('change',savePlan);
  });
  // Thu gọn/mở rộng chỉ thay đổi giao diện; không được phát sinh sự kiện tới game
  // hoặc thay đổi state.running/state.paused/tiến trình hiện tại.
  panel.addEventListener('click', ev => { ev.stopPropagation(); });
  $('#mc-min', panel).addEventListener('click', ev => {
    ev.preventDefault();
    ev.stopPropagation();
    panel.classList.toggle('mc-collapsed');
    $('#mc-min', panel).textContent = panel.classList.contains('mc-collapsed') ? '＋' : '−';
    $('#mc-min', panel).title = panel.classList.contains('mc-collapsed') ? 'Mở rộng panel' : 'Thu gọn tối đa';
  });

  ui.elements.style.display = 'none';
  let initialPlan; try { initialPlan=JSON.parse(localStorage.getItem(PLAN_KEY)||'null'); } catch(_) {}
  applyPlanToUI(normalizePlan(initialPlan || loadPlanFromUI()));
  // Chỉ khôi phục phiên chạy sau reload nếu người dùng đã bấm Auto trong tab này;
  // mở panel lần đầu không tự bắt đầu.
  const resumed = restoreRunProgress();
  if (!resumed) log('Panel đã sẵn sàng. Hãy chọn chế độ và bấm Auto để bắt đầu.');
  if (resumed) panelOpen = true;
  panel.classList.add('mc-collapsed');
  $('#mc-min', panel).textContent = '＋';
  $('#mc-min', panel).title = 'Mở rộng panel';
  // Vòng lặp tự động chạy độc lập với trạng thái thu gọn/mở rộng của panel.
  setInterval(() => {
    try {
      if (checkExpiredSession()) return;
      try { updatePanelVisibility(); } catch (e) { console.warn('[Sakute] panel visibility:', e); }
      try { keepGameSettingsEnabled(); } catch (e) { console.warn('[Sakute] game settings:', e); }
      // Auto tick is deliberately independent of panel open/collapsed layout.
      if (state.running && !state.paused) {
        Promise.resolve(tick()).catch(e => { console.error('[Sakute] tick error:', e); log(`Lỗi vòng Auto: ${e?.message || e}`, 'warn'); });
        saveRunProgress();
      }
      if (state.startedAt && state.running) {
        const sec = Math.floor((Date.now() - state.startedAt) / 1000);
        ui.time.textContent = `${String(Math.floor(sec/60)).padStart(2,'0')}:${String(sec%60).padStart(2,'0')}`;
      }
    } catch (e) { console.error('[Sakute] auto loop error:', e); }
  }, 1200);
})();
