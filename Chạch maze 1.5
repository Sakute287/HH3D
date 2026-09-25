// ==UserScript==
// @name         Ngồi IM teo lái 👌🏻
// @namespace    *https://hoathinh3d.*/
// @version      1.0.0
// @description  Tự động xử lý Mê Cung
// @author       ChachHere
// @include      https://hoathinh3d.*/me-cung*
// @run-at       document-start
// @grant        none
// @updateURL   https://raw.githubusercontent.com/ChachHere/Chach-coco/main/Chanh.js
// @downloadURL https://raw.githubusercontent.com/ChachHere/Chach-coco/main/Chanh.js
// ==/UserScript==

(() => {
    'use strict';


    const LOG = '[Ngồi IM teo lái 👌🏻';
    function isMeCungDomain() {
        return /^hoathinh3d\.[^.]+$/i.test(
            window.location.hostname
        );
    }

    function isMeCungPage() {
        if (!isMeCungDomain()) {
            return false;
        }

        const path = window.location.pathname.replace(/\/+$/, '');

        if (path !== '/me-cung') {
            return false;
        }

        const signatures = [
            '#btn-start',
            '#battle-frame',
            '#player-list',
            '#auto-ready-track',
            '#modal-b5-reward'
        ];

        const matched = signatures.filter(
            selector => document.querySelector(selector)
        ).length;

        return matched >= 2;
    }
    const CHECK_INTERVAL = 1000;
    const ACTION_COOLDOWN = 700;

    const SELECTORS = {
        startButton: '#btn-start',
        confirmButton: '.confirm-btn-ok',
        battleFrame: '#battle-frame',
        autoAttackTrack: '#auto-attack-track',
        autoAttackToggle: '#auto-attack-toggle',
        autoReadyTrack: '#auto-ready-track',
        autoReadyToggle: '#auto-ready-row button',
        autoNextStageTrack: '#auto-next-stage-track',
        autoNextStageToggle: '#auto-next-stage-row button',
        autoOpenChestTrack: '#auto-open-chest-track',
        autoOpenChestToggle: '#auto-open-chest-row button',
        chestInplace: '#b5-chest-inplace',
        chestCountdown: '#b5-chest-countdown',
        chestWrap: '#b5-chest-wrap',
        rewardBox: '#modal-b5-reward',
        backLobbyButton: '#modal-b5-reward .btn-back-lobby',
        failModal: '#modal-fail',
        failBackLobbyButton: '#modal-fail .btn-next-stage',
        nextStageButton: '#btn-next-stage'
    };

    let lastActionAt = 0;
    let startedCurrentCycle = false;
    const MC_SETTINGS_KEY = 'hh3d_me_cung_settings';

    const mcSettings = {
        toolEnabled: false,
        minMembers: 5,
        kickUnreadyEnabled: false,
        kickUnreadyMinutes: 5
    };

    try {
        const saved = JSON.parse(
            localStorage.getItem(MC_SETTINGS_KEY) || '{}'
        );

        // Lần đầu chưa có setting → mặc định TẮT.
        if (typeof saved.toolEnabled === 'boolean') {
            mcSettings.toolEnabled = saved.toolEnabled;
        }

        if (
            Number.isInteger(saved.minMembers) &&
            saved.minMembers >= 1 &&
            saved.minMembers <= 5
        ) {
            mcSettings.minMembers = saved.minMembers;
        }

        if (typeof saved.kickUnreadyEnabled === 'boolean') {
            mcSettings.kickUnreadyEnabled = saved.kickUnreadyEnabled;
        }

        if (
            Number.isFinite(saved.kickUnreadyMinutes) &&
            saved.kickUnreadyMinutes >= 1
        ) {
            mcSettings.kickUnreadyMinutes = Math.floor(
                saved.kickUnreadyMinutes
            );
        }
    } catch (error) {
        console.warn(
            `${LOG} ⚠️ Không đọc được Settings Mê Cung:`,
            error
        );
    }

    function saveMcSettings() {
        localStorage.setItem(
            MC_SETTINGS_KEY,
            JSON.stringify(mcSettings)
        );
    }
    let confirmHandled = false;
    let battleAutoHandled = false;
    let chestHandled = false;
    let rewardHandled = false;
    let lastState = 'unknown';
    let observerScheduled = false;
    let failHandled = false;
    function isVisible(element) {
        if (!element) return false;
        if (element.hidden) return false;
        if (element.getClientRects().length === 0) return false;

        const style = getComputedStyle(element);
        return style.display !== 'none' && style.visibility !== 'hidden';
    }

    function canAct() {
        const now = Date.now();
        if (now - lastActionAt < ACTION_COOLDOWN) return false;
        lastActionAt = now;
        return true;
    }

    function clickOnce(selector, label, ignoreCooldown = false) {
        const element = document.querySelector(selector);
        if (!isVisible(element)) return false;

        if (!ignoreCooldown && !canAct()) return false;

        element.click();
        console.log(`${LOG} ${label}`);
        return true;
    }

    function trackIsOn(trackSelector) {
        const track = document.querySelector(trackSelector);
        return !!track && track.classList.contains('on');
    }

    function getCurrentMemberCount() {
        return document.querySelectorAll('#player-list .player-card.filled').length;
    }

    function ensureToggle(trackSelector, buttonSelector, label) {
        const track = document.querySelector(trackSelector);
        const button = document.querySelector(buttonSelector);

        if (!isVisible(track) || !isVisible(button)) return false;
        if (track.classList.contains('on')) return false;
        if (!canAct()) return false;

        button.click();
        console.log(`${LOG} 🤖 Đã bật ${label}`);
        return true;
    }

    function getState() {
        const rewardBox = document.querySelector(SELECTORS.rewardBox);
        if (isVisible(rewardBox)) return 'reward';

        const failModal = document.querySelector(SELECTORS.failModal);
        if (isVisible(failModal)) return 'fail';

        const confirm = document.querySelector(SELECTORS.confirmButton);
        if (isVisible(confirm)) return 'confirm';

        const nextStageButton = document.querySelector(SELECTORS.nextStageButton);
        if (
            isVisible(nextStageButton) &&
            !nextStageButton.disabled
        ) {
            return 'next-stage';
        }

        const chest = document.querySelector(SELECTORS.chestInplace);
        if (isVisible(chest)) return 'chest';

        const battle = document.querySelector(SELECTORS.battleFrame);
        if (isVisible(battle)) return 'battle';

        const start = document.querySelector(SELECTORS.startButton);
        const readyTrack = document.querySelector(SELECTORS.autoReadyTrack);
        if (isVisible(start) || isVisible(readyTrack)) return 'waiting-room';

        return 'lobby';
    }

    function resetCycleFlags() {
        startedCurrentCycle = false;
        confirmHandled = false;
        battleAutoHandled = false;
        chestHandled = false;
        rewardHandled = false;
        failHandled = false;
    }

    function announceState(state) {
        if (state === lastState) return;
        lastState = state;
        console.log(`${LOG} 📍 State: ${state}`);
    }

    function handleWaitingRoom() {
        // 1) Bảo đảm 3 Auto của phòng chờ đều ON.
        ensureToggle(
            SELECTORS.autoReadyTrack,
            SELECTORS.autoReadyToggle,
            'Tự động sẵn sàng'
        );

        ensureToggle(
            SELECTORS.autoNextStageTrack,
            SELECTORS.autoNextStageToggle,
            'Tự động chuyển ải'
        );

        const autoOpenTrack = document.querySelector(SELECTORS.autoOpenChestTrack);
        const autoOpenToggle = document.querySelector(SELECTORS.autoOpenChestToggle);

        if (
            isVisible(autoOpenTrack) &&
            isVisible(autoOpenToggle) &&
            autoOpenTrack.classList.contains('on')
        ) {
            clickOnce(
                SELECTORS.autoOpenChestToggle,
                '🛑 Đã tắt Tự động mở rương → tool tự mở rương'
            );
        }

        // 2) Chỉ bấm Bắt đầu khi nút thật sự đang hiện và chưa xử lý ở cycle này.
        const startBtn = document.querySelector(SELECTORS.startButton);
        const currentMemberCount = getCurrentMemberCount();

        if (
            !startedCurrentCycle &&
            currentMemberCount >= mcSettings.minMembers &&
            isVisible(startBtn) &&
            !startBtn.disabled
        ) {
            if (clickOnce(SELECTORS.startButton, '▶️ Đã bấm BẮT ĐẦU')) {
                confirmHandled = false;

                setTimeout(() => {
                    const confirmButton = document.querySelector(SELECTORS.confirmButton);

                    if (isVisible(confirmButton)) {
                        startedCurrentCycle = true;
                    } else {
                        console.log(
                            `${LOG} ⏳ Chưa đủ điều kiện bắt đầu → chờ thành viên sẵn sàng rồi kiểm tra lại.`
                        );
                    }
                }, 300);
            }
        }
        if (rewardHandled || failHandled) {
            resetCycleFlags();
            console.log(`${LOG} 🔄 Đã quay lại phòng chờ → bắt đầu cycle mới.`);
        }
    }

    function handleConfirm() {
        // Confirm bắt đầu trận
        if (confirmHandled) return;

        if (
            clickOnce(
                SELECTORS.confirmButton,
                '✅ Đã bấm XÁC NHẬN',
                true
            )
        ) {
            confirmHandled = true;
        }
    }

    function handleNextStage() {
        const nextStageButton = document.querySelector(
            SELECTORS.nextStageButton
        );

        if (!isVisible(nextStageButton)) return;
        if (nextStageButton.disabled) return;

        if (
            clickOnce(
                SELECTORS.nextStageButton,
                '⏩ Đã bấm ĐẾN ẢI KẾ TIẾP',
                true
            )
        ) {
            console.log(`${LOG} ⚡ Chuyển ải ngay`);
        }
    }

    function handleBattle() {
        // Battle đã hiện thật sự -> bật Auto Tấn Công ngay nếu OFF.
        if (battleAutoHandled) return;

        const track = document.querySelector(SELECTORS.autoAttackTrack);
        const toggle = document.querySelector(SELECTORS.autoAttackToggle);

        if (!isVisible(track) || !isVisible(toggle)) return;

        if (track.classList.contains('on')) {
            console.log(`${LOG} 🤖 TỰ ĐỘNG TẤN CÔNG đã ON sẵn.`);
            battleAutoHandled = true;
            return;
        }

        if (clickOnce(SELECTORS.autoAttackToggle, '🤖 Đã bật TỰ ĐỘNG TẤN CÔNG')) {
            battleAutoHandled = true;
        }
    }

    function handleChest() {
        if (chestHandled) return;

        const chest = document.querySelector(SELECTORS.chestInplace);
        const countdown = document.querySelector(SELECTORS.chestCountdown);
        const chestWrap = document.querySelector(SELECTORS.chestWrap);

        if (!isVisible(chest) || !isVisible(countdown) || !isVisible(chestWrap)) return;

        const text = countdown.textContent.trim();
        if (text !== '✅ Rương đã sẵn sàng!') return;

        if (clickOnce(SELECTORS.chestWrap, '🎁 Rương đã sẵn sàng → Đã click mở rương')) {
            chestHandled = true;
        }
    }

    function handleReward() {
        if (rewardHandled) return;

        const backButton = document.querySelector(SELECTORS.backLobbyButton);
        if (!backButton) return;

        backButton.click();
        rewardHandled = true;

        console.log(`${LOG} 🏠 Đã bấm QUAY VỀ SẢNH → loop lại`);
    }

    function handleFail() {
        if (failHandled) return;

        const backButton = document.querySelector(SELECTORS.failBackLobbyButton);
        if (!backButton) return;

        backButton.click();
        failHandled = true;

        console.log(`${LOG} ☠️ Thất bại → Đã bấm QUAY VỀ SẢNH → loop lại`);
    }

    let lastStateMachineRun = 0;

    function runStateMachine() {
        const state = getState();
        const now = Date.now();

        // State confirm là phản hồi trực tiếp sau một thao tác,
        // nên cho chạy ngay, không chờ cooldown 3 giây.
        const isUrgentState =
              state === 'confirm' ||
              state === 'next-stage';

        if (!isUrgentState && now - lastStateMachineRun < 3000) {
            return;
        }

        lastStateMachineRun = now;

        if (!mcSettings.toolEnabled) return;

        announceState(state);

        switch (state) {
            case 'waiting-room':
                handleWaitingRoom();
                break;

            case 'confirm':
                handleConfirm();
                break;

            case 'next-stage':
                handleNextStage();
                break;

            case 'battle':
                handleBattle();
                break;

            case 'chest':
                handleChest();
                break;

            case 'reward':
                handleReward();
                break;

            case 'fail':
                handleFail();
                break;

            case 'lobby':
            default:
                // Sảnh chính để thủ công.
                break;
        }
    }

    function scheduleRun() {
        if (observerScheduled) return;
        observerScheduled = true;

        setTimeout(() => {
            observerScheduled = false;
            try {
                runStateMachine();
            } catch (error) {
                console.error(`${LOG} ❌ Lỗi state machine:`, error);
            }
        }, 200);
    }

    function createMcPanel() {
        if (document.querySelector('#hh3d-me-cung-panel')) return;

        const style = document.createElement('style');

        style.textContent = `
            #hh3d-me-cung-panel {
                position: fixed;
                top: 90px;
                right: 18px;
                width: 320px;
                z-index: 999999;
                box-sizing: border-box;
                padding: 18px;
                border: 1px solid rgba(238, 192, 73, .55);
                border-radius: 16px;
                background:
                    radial-gradient(circle at 50% 0%, rgba(255, 214, 92, .10), transparent 38%),
                    linear-gradient(145deg, rgba(30, 31, 34, .97), rgba(18, 19, 21, .97));
                color: #eee;
                font: 13px/1.45 -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
                box-shadow:
                    0 0 0 1px rgba(255, 214, 92, .05),
                    0 14px 40px rgba(0, 0, 0, .55),
                    inset 0 0 24px rgba(255, 214, 92, .025);
                backdrop-filter: blur(10px);
            }

            #hh3d-me-cung-panel .mc-panel-header {
                display: flex;
                align-items: center;
                justify-content: space-between;
                gap: 10px;
                margin-bottom: 14px;
            }

            #hh3d-me-cung-panel .mc-panel-title {
                color: #f1f1f1;
                font-size: 15px;
                font-weight: 800;
                letter-spacing: .02em;
            }

            #hh3d-me-cung-panel .mc-panel-minimize {
                width: 30px;
                height: 30px;
                padding: 0;
                border: 1px solid rgba(255, 214, 92, .20);
                border-radius: 9px;
                background: rgba(255,255,255,.06);
                color: #ddd;
                cursor: pointer;
                font-size: 18px;
                line-height: 28px;
            }

            #hh3d-me-cung-panel .mc-panel-row {
                width: 100% !important;
                box-sizing: border-box !important;

                display: grid !important;
                grid-template-columns: minmax(0, 1fr) auto !important;

                align-items: center !important;
                column-gap: 12px !important;

                margin: 7px 0 !important;
            }

            #hh3d-me-cung-panel .mc-panel-label {
                min-width: 0 !important;
                width: 100% !important;
                box-sizing: border-box !important;

                color: #cbd3e4 !important;
                font-size: 12px !important;
                font-weight: 500 !important;
                line-height: 1.35 !important;
            }

            #hh3d-me-cung-panel .mc-panel-value {
                color: #ffd34d;
                font-weight: 800;
            }

            #hh3d-me-cung-panel .mc-panel-separator {
                height: 1px;
                margin: 15px 0 13px;
                background: linear-gradient(
                    90deg,
                    transparent,
                    rgba(255, 214, 92, .28),
                    transparent
                );
            }

            #hh3d-me-cung-panel .mc-panel-section-title {
                margin-bottom: 9px;
                color: #f0c94d;
                font-size: 11px;
                font-weight: 800;
                letter-spacing: .08em;
                text-transform: uppercase;
            }

            #hh3d-me-cung-panel select,
            #hh3d-me-cung-panel input[type="number"] {
                width: 92px;
                box-sizing: border-box;
                padding: 7px 9px;
                border: 1px solid rgba(255, 214, 92, .22);
                border-radius: 8px;
                background: rgba(255,255,255,.06);
                color: #fff;
                outline: none;
                font: inherit;
            }

            #hh3d-me-cung-panel .mc-setting-toggle {
                min-width: 72px;
                padding: 8px 11px;
                border: 1px solid rgba(255,255,255,.08);
                border-radius: 9px;
                background: linear-gradient(180deg, #4a4a4a, #333);
                color: #fff;
                cursor: pointer;
                font-size: 11px;
                font-weight: 800;
                box-shadow: inset 0 1px 0 rgba(255,255,255,.05);
            }

            #hh3d-me-cung-panel .mc-setting-toggle.on {
                border-color: rgba(255, 214, 92, .35);
                background: linear-gradient(180deg, #ffce43, #e7a91f);
                color: #211b0d;
                box-shadow:
                    0 0 16px rgba(255, 196, 54, .20),
                    inset 0 1px 0 rgba(255,255,255,.35);
            }

            #hh3d-me-cung-panel .mc-tool-card {
                margin-bottom: 9px !important;
                padding: 8px 10px !important;

                border: 1px solid rgba(245, 197, 66, 0.16) !important;
                border-radius: 10px !important;

                background: rgba(255, 255, 255, 0.018) !important;

                box-shadow: none !important;
            }

            #hh3d-me-cung-panel .mc-tool-status {
                width: 100% !important;
                box-sizing: border-box !important;

                display: grid !important;
                grid-template-columns: minmax(0, 1fr) auto !important;

                align-items: center !important;
                column-gap: 14px !important;

                min-height: 34px !important;
            }

            #hh3d-me-cung-panel .mc-tool-state {
                display: inline-flex;
                align-items: center;
                gap: 8px;
                color: #d8d8d8;
                font-size: 14px;
                font-weight: 800;
            }

            #hh3d-me-cung-panel .mc-tool-state-dot {
                display: inline-block;
                width: 10px;
                height: 10px;
                flex: 0 0 10px;
                border-radius: 50%;
                background: #ff5252;
                box-shadow:
                    0 0 8px rgba(255, 82, 82, .45),
                    0 0 14px rgba(255, 82, 82, .20);
                transition:
                    background .25s ease,
                    box-shadow .25s ease,
                    transform .25s ease;
            }

            #hh3d-me-cung-panel .mc-tool-state.on .mc-tool-state-dot {
                background: #35e7a4;
                box-shadow:
                    0 0 8px rgba(53, 231, 164, .75),
                    0 0 18px rgba(53, 231, 164, .55),
                    0 0 30px rgba(53, 231, 164, .20);
                animation: hh3dMcToolBreath 2s ease-in-out infinite;
            }

            #hh3d-me-cung-panel .mc-tool-state.off .mc-tool-state-dot {
                background: #ff5252;
                box-shadow:
                    0 0 8px rgba(255, 82, 82, .45),
                    0 0 14px rgba(255, 82, 82, .20);
            }

            #hh3d-me-cung-panel .mc-tool-button {
                min-width: 88px !important;
                height: 30px !important;
                padding: 4px 10px !important;

                border-radius: 7px !important;

                font-family: inherit !important;
                font-size: 10px !important;
                font-weight: 600 !important;
                letter-spacing: .015em !important;

                box-shadow: none !important;
            }

            #hh3d-me-cung-panel .mc-tool-button:active {
                transform: translateY(1px);
            }

            #hh3d-me-cung-panel .mc-tool-button.on {
                border-color: rgba(53, 231, 164, .45);
                background: linear-gradient(180deg, #35d99b, #159b68);
                color: #fff;
                box-shadow:
                    0 0 8px rgba(53, 231, 164, .22),
                    0 0 16px rgba(53, 231, 164, .12),
                    inset 0 1px 0 rgba(255,255,255,.22);
                animation: hh3dMcButtonBreath 2.4s ease-in-out infinite;
            }

            @keyframes hh3dMcButtonBreath {
                0%, 100% {
                    box-shadow:
                        0 0 7px rgba(53, 231, 164, .20),
                        0 0 14px rgba(53, 231, 164, .10),
                        inset 0 1px 0 rgba(255,255,255,.20);
                }

                50% {
                    box-shadow:
                        0 0 10px rgba(53, 231, 164, .34),
                        0 0 20px rgba(53, 231, 164, .16),
                        inset 0 1px 0 rgba(255,255,255,.25);
                }
            }

            #hh3d-me-cung-panel .mc-tool-button.off {
                border-color: rgba(255, 82, 82, .35);
                background: linear-gradient(180deg, #ff5d66, #d92e3b);
                box-shadow:
                    0 0 10px rgba(255, 82, 82, .18),
                    inset 0 1px 0 rgba(255,255,255,.16);
            }

            @keyframes hh3dMcToolBreath {
                0%, 100% {
                    transform: scale(1);
                    opacity: .82;
                    box-shadow:
                        0 0 8px rgba(53, 231, 164, .70),
                        0 0 18px rgba(53, 231, 164, .48),
                        0 0 28px rgba(53, 231, 164, .18);
                }

                50% {
                    transform: scale(1.16);
                    opacity: 1;
                    box-shadow:
                        0 0 10px rgba(53, 231, 164, .90),
                        0 0 24px rgba(53, 231, 164, .66),
                        0 0 38px rgba(53, 231, 164, .28);
                }
            }

            #hh3d-me-cung-panel .mc-tool-button {
                min-width: 92px;
                padding: 9px 14px;
                border: 0;
                border-radius: 9px;
                cursor: pointer;
                background: linear-gradient(180deg, #575757, #3b3b3b);
                color: #fff;
                font-size: 12px;
                font-weight: 900;
            }

            #hh3d-me-cung-panel .mc-tool-button.on {
                background: linear-gradient(180deg, #ff405f, #eb2450);
                box-shadow: 0 0 18px rgba(255, 55, 90, .18);
            }

            #hh3d-me-cung-panel .mc-tool-current-state {
                margin-top: 9px;
                color: #aaa;
                font-size: 12px;
            }

            #hh3d-me-cung-panel .mc-panel-footer {
                width: 100% !important;
                box-sizing: border-box !important;

                margin-top: 10px !important;
                padding-top: 10px !important;

                border-top: 1px solid rgba(255,255,255,0.10) !important;

                color: #d0d8f0 !important;

                font-family: inherit !important;
                font-size: 20px !important;
                font-weight: 600 !important;
                letter-spacing: .02em !important;

                text-align: center !important;
            }

            #hh3d-me-cung-fab {
                position: fixed;
                top: 90px;
                right: 18px;
                width: 46px;
                height: 46px;
                z-index: 999999;
                display: none;
                align-items: center;
                justify-content: center;
                border: 1px solid rgba(255, 214, 92, .35);
                border-radius: 13px;
                background: rgba(24, 24, 26, .96);
                color: #fff;
                cursor: pointer;
                font-size: 20px;
                box-shadow:
                    0 0 18px rgba(0,0,0,.35),
                    inset 0 0 12px rgba(255,214,92,.04);
                backdrop-filter: blur(10px);
            }

            #hh3d-me-cung-panel button#mc-tool-toggle.mc-tool-button.on {
                border-color: rgba(53, 231, 164, .45) !important;
                background: linear-gradient(180deg, #35d99b, #159b68) !important;
                color: #fff !important;
                box-shadow:
                    0 0 8px rgba(53, 231, 164, .22),
                    0 0 16px rgba(53, 231, 164, .12),
                    inset 0 1px 0 rgba(255,255,255,.22) !important;
                animation: hh3dMcButtonBreath 2.4s ease-in-out infinite !important;
            }

            @keyframes hh3dMcButtonBreath {
                0%, 100% {
                    box-shadow:
                        0 0 7px rgba(53, 231, 164, .20),
                        0 0 14px rgba(53, 231, 164, .10),
                        inset 0 1px 0 rgba(255,255,255,.20) !important;
                }

                50% {
                    box-shadow:
                        0 0 10px rgba(53, 231, 164, .34),
                        0 0 20px rgba(53, 231, 164, .16),
                        inset 0 1px 0 rgba(255,255,255,.25) !important;
                }
            }

            /* ===== Ngồi IM teo lái 👌🏻 - Final Skin ===== */

            #hh3d-me-cung-panel {
                width: 310px !important;
                box-sizing: border-box !important;

                padding: 12px !important;

                border: 1px solid rgba(245, 197, 66, 0.14) !important;
                border-radius: 10px !important;

                background: rgba(36, 35, 35, 0.68) !important;

                color: #d0d8f0 !important;

                font-family: inherit !important;

                box-shadow:
                    0 8px 32px rgba(0, 0, 0, 0.30) !important;

                backdrop-filter: blur(12px) !important;
                -webkit-backdrop-filter: blur(12px) !important;
            }

            /* Header */

            #hh3d-me-cung-panel .mc-panel-header {
                margin-bottom: 10px !important;
            }

            #hh3d-me-cung-panel .mc-panel-title {
                color: #e0e6f0 !important;

                font-size: 15px !important;
                font-weight: 600 !important;

                letter-spacing: 0 !important;
            }

            /* Nút thu nhỏ */

            #hh3d-me-cung-panel .mc-panel-minimize {
                width: 29px !important;
                height: 29px !important;

                padding: 0 !important;

                border: 1px solid rgba(245, 197, 66, 0.16) !important;
                border-radius: 8px !important;

                background: rgba(255, 255, 255, 0.035) !important;
                color: #d0d8f0 !important;

                font-size: 16px !important;
                font-weight: 500 !important;

                box-shadow: none !important;
            }

            /* Card Trạng thái Tool */

            #hh3d-me-cung-panel .mc-tool-card {
                margin-bottom: 9px !important;
                padding: 9px 10px !important;

                border: 1px solid rgba(245, 197, 66, 0.18) !important;
                border-radius: 9px !important;

                background: rgba(255, 255, 255, 0.025) !important;

                box-shadow: none !important;
            }

            #hh3d-me-cung-panel .mc-tool-status {
                width: 100% !important;
                min-height: 32px !important;

                display: grid !important;
                grid-template-columns: minmax(0, 1fr) auto !important;

                align-items: center !important;
                column-gap: 14px !important;
            }

            #hh3d-me-cung-panel .mc-tool-state {
                display: inline-flex !important;
                align-items: center !important;
                gap: 8px !important;

                color: #d0d8f0 !important;

                font-size: 13px !important;
                font-weight: 600 !important;
            }

            /* LED */

            #hh3d-me-cung-panel .mc-tool-state-dot {
                width: 9px !important;
                height: 9px !important;

                flex: 0 0 9px !important;

                border-radius: 50% !important;

                transition:
                    background .25s ease,
                    box-shadow .25s ease,
                    transform .25s ease !important;
            }

            #hh3d-me-cung-panel .mc-tool-state.off .mc-tool-state-dot {
                background: #e74c3c !important;

                box-shadow:
                    0 0 5px rgba(231, 76, 60, .45),
                    0 0 10px rgba(231, 76, 60, .16) !important;

                animation: none !important;
            }

            #hh3d-me-cung-panel .mc-tool-state.on .mc-tool-state-dot {
                background: #22d3a0 !important;

                animation: hh3dMcLedBreathFinal 2.4s ease-in-out infinite !important;
            }

            /* Nút ĐANG CHẠY / DỪNG */

            #hh3d-me-cung-panel button#mc-tool-toggle {
                min-width: 92px !important;
                height: 32px !important;

                padding: 4px 12px !important;

                border-radius: 6px !important;

                font-family: inherit !important;
                font-size: 10px !important;
                font-weight: 600 !important;

                letter-spacing: .3px !important;

                box-shadow: none !important;

                transition:
                    background .2s ease,
                    border-color .2s ease,
                    filter .2s ease !important;
            }

            #hh3d-me-cung-panel button#mc-tool-toggle.on {
                background: linear-gradient(
                    135deg,
                    #22d3a0,
                    #10b981
                ) !important;

                border: 1px solid rgba(34, 211, 160, .30) !important;

                color: #fff !important;

                box-shadow:
                    0 0 6px rgba(34, 211, 160, .12) !important;

                animation: hh3dMcRunningButtonFinal 2.8s ease-in-out infinite !important;
            }

            #hh3d-me-cung-panel button#mc-tool-toggle.off {
                background: linear-gradient(
                    135deg,
                    #df5a60,
                    #c9424c
                ) !important;

                border: 1px solid rgba(231, 76, 60, .24) !important;

                color: #fff !important;

                box-shadow: none !important;

                animation: none !important;
            }

            /* Tiêu đề Cài đặt */

            #hh3d-me-cung-panel .mc-panel-section-title {
                margin-bottom: 7px !important;

                color: #f5c542 !important;

                font-size: 10px !important;
                font-weight: 600 !important;

                letter-spacing: .08em !important;
            }

            /* Chữ Settings */

            #hh3d-me-cung-panel .mc-panel-label {
                color: #cbd3e4 !important;

                font-size: 12px !important;
                font-weight: 500 !important;

                line-height: 1.35 !important;
            }

            /* Select + Number */

            #hh3d-me-cung-panel select,
            #hh3d-me-cung-panel input[type="number"] {
                width: 84px !important;
                height: 31px !important;

                padding: 4px 8px !important;

                border: 1px solid rgba(245, 197, 66, .17) !important;
                border-radius: 7px !important;

                background: rgba(255, 255, 255, .045) !important;
                color: #d0d8f0 !important;

                font-family: inherit !important;
                font-size: 11px !important;
                font-weight: 500 !important;

                box-shadow: none !important;
            }

            /* Toggle Trục xuất */

            #hh3d-me-cung-panel .mc-setting-toggle {
                min-width: 64px !important;
                height: 30px !important;

                padding: 4px 10px !important;

                border-radius: 7px !important;

                font-family: inherit !important;
                font-size: 10px !important;
                font-weight: 600 !important;

                box-shadow: none !important;
            }

            #hh3d-me-cung-panel .mc-setting-toggle.on {
                background: #22d3a0 !important;

                border-color: rgba(34, 211, 160, .25) !important;

                color: #fff !important;

                box-shadow: none !important;
            }

            #hh3d-me-cung-panel .mc-setting-toggle:not(.on) {
                background: rgba(255, 255, 255, .07) !important;

                border-color: rgba(255, 255, 255, .10) !important;

                color: #d0d8f0 !important;

                box-shadow: none !important;
            }

            /* Đường phân cách */

            #hh3d-me-cung-panel .mc-panel-separator {
                background: rgba(255, 255, 255, .075) !important;
            }

            /* Footer */

            #hh3d-me-cung-panel .mc-panel-footer {
                width: 100% !important;
                box-sizing: border-box !important;

                margin-top: 10px !important;
                padding-top: 10px !important;

                border-top: 1px solid rgba(255, 255, 255, .08) !important;

                text-align: center !important;

                color: #d0d8f0 !important;

                font-family: inherit !important;
                font-size: 20px !important;
                font-weight: 500 !important;

                letter-spacing: .02em !important;
            }

            /* Phòng trường hợp CSS cũ còn sót */

            #hh3d-me-cung-panel .mc-tool-current-state {
                display: none !important;
            }

            /* Animation LED */

            @keyframes hh3dMcLedBreathFinal {
                0%, 100% {
                    transform: scale(1);
                    opacity: .82;

                    box-shadow:
                        0 0 5px rgba(34, 211, 160, .48),
                        0 0 10px rgba(34, 211, 160, .16);
                }

                50% {
                    transform: scale(1.10);
                    opacity: 1;

                    box-shadow:
                        0 0 7px rgba(34, 211, 160, .66),
                        0 0 14px rgba(34, 211, 160, .22);
                }
            }

            /* Animation nút chạy - nhẹ hơn LED */

            @keyframes hh3dMcRunningButtonFinal {
                0%, 100% {
                    filter: brightness(1);
                }

                50% {
                    filter: brightness(1.025);
                }
            }
    `;

        document.head.appendChild(style);

        const panel = document.createElement('div');
        panel.id = 'hh3d-me-cung-panel';

        panel.innerHTML = `
            <div class="mc-panel-header">
                <div class="mc-panel-title">⚔️ Ngồi IM teo lái 👌🏻</div>

                <button
                    type="button"
                    class="mc-panel-minimize"
                    id="mc-panel-minimize"
                    title="Thu nhỏ panel"
                >−</button>
            </div>

            <div class="mc-tool-card">
                <div class="mc-tool-status">
                    <div class="mc-tool-state off" id="mc-tool-state">
                        <span class="mc-tool-state-dot"></span>
                        <span>Trạng thái Tool</span>
                    </div>

                    <button
                        type="button"
                        class="mc-tool-button off"
                        id="mc-tool-toggle"
                    >DỪNG</button>
                </div>
            </div>

            <div class="mc-panel-separator"></div>

            <div class="mc-panel-section-title">
                Cài đặt
            </div>

            <div class="mc-panel-row">
                <span class="mc-panel-label">
                    👥 Người tối thiểu để bắt đầu
                </span>

                <select id="mc-setting-min-members">
                    <option value="1">1 người</option>
                    <option value="2">2 người</option>
                    <option value="3">3 người</option>
                    <option value="4">4 người</option>
                    <option value="5">5 người</option>
                </select>
            </div>

            <div class="mc-panel-row">
                <span class="mc-panel-label">
                    🚪 Trục xuất người chưa sẵn sàng
                </span>

                <button
                    type="button"
                    class="mc-setting-toggle"
                    id="mc-setting-kick"
                >TẮT</button>
            </div>

            <div class="mc-panel-row">
                <span class="mc-panel-label">
                    ⏱ Thời gian chờ trục xuất
                </span>

                <input
                    id="mc-setting-kick-minutes"
                    type="number"
                    min="1"
                    step="1"
                    value="5"
                >
            </div>

            <div class="mc-panel-footer">
                Heny
            </div>
        `;

        document.body.appendChild(panel);

        const fab = document.createElement('button');
        fab.id = 'hh3d-me-cung-fab';
        fab.type = 'button';
        fab.title = 'Mở Ngồi IM teo lái 👌🏻';
        fab.textContent = '⚔️';

        document.body.appendChild(fab);

        const minimizeButton = panel.querySelector('#mc-panel-minimize');

        minimizeButton.addEventListener('click', () => {
            panel.style.display = 'none';
            fab.style.display = 'flex';
        });

        fab.addEventListener('click', () => {
            panel.style.display = 'block';
            fab.style.display = 'none';
        });

        const minMembersSelect = panel.querySelector('#mc-setting-min-members');
        const kickToggle = panel.querySelector('#mc-setting-kick');
        const kickMinutesInput = panel.querySelector('#mc-setting-kick-minutes');
        const toolToggle = panel.querySelector('#mc-tool-toggle');

        const updateToolToggle = () => {
            const isRunning = mcSettings.toolEnabled;

            toolToggle.classList.toggle('on', isRunning);
            toolToggle.classList.toggle('off', !isRunning);

            toolToggle.textContent = isRunning
                ? 'ĐANG CHẠY'
            : 'DỪNG';

            const toolState = panel.querySelector('#mc-tool-state');

            if (toolState) {
                toolState.classList.remove('on', 'off');
                toolState.classList.add(isRunning ? 'on' : 'off');
            }
        };

        updateToolToggle();

        toolToggle.addEventListener('click', () => {
            mcSettings.toolEnabled = !mcSettings.toolEnabled;
            saveMcSettings();
            updateToolToggle();

            console.log(
                `${LOG} ⚙️ Tool Mê Cung:`,
                mcSettings.toolEnabled ? 'BẬT' : 'TẮT'
            );

            if (mcSettings.toolEnabled) {
                lastStateMachineRun = 0;
                runStateMachine();
            }
        });

        kickToggle.addEventListener('click', () => {
            mcSettings.kickUnreadyEnabled = !mcSettings.kickUnreadyEnabled;

            kickToggle.classList.toggle('on', mcSettings.kickUnreadyEnabled);
            kickToggle.textContent = mcSettings.kickUnreadyEnabled
                ? 'BẬT'
            : 'TẮT';

            saveMcSettings();

            console.log(
                `${LOG} ⚙️ Trục xuất chưa sẵn sàng:`,
                mcSettings.kickUnreadyEnabled ? 'BẬT' : 'TẮT'
            );
        });

        kickMinutesInput.addEventListener('change', () => {
            const value = Math.max(
                1,
                Math.floor(Number(kickMinutesInput.value) || 1)
            );

            mcSettings.kickUnreadyMinutes = value;
            kickMinutesInput.value = String(value);

            saveMcSettings();

            console.log(
                `${LOG} ⚙️ Thời gian chờ trục xuất: ${value} phút`
            );
        });

        minMembersSelect.addEventListener('change', () => {
            mcSettings.minMembers = Number(minMembersSelect.value);
            saveMcSettings();

            console.log(
                `${LOG} ⚙️ Số người tối thiểu để bắt đầu: ${mcSettings.minMembers}`
            );
        });

        minMembersSelect.value = String(mcSettings.minMembers);
        kickMinutesInput.value = String(mcSettings.kickUnreadyMinutes);

        kickToggle.classList.toggle('on', mcSettings.kickUnreadyEnabled);
        kickToggle.textContent = mcSettings.kickUnreadyEnabled ? 'BẬT' : 'TẮT';
    }

    function start() {
        if (!isMeCungPage()) {
            return;
        }

        console.log(
            `${LOG} 🚀 Đã khởi động trên ${window.location.origin}`
        );

        createMcPanel();

        const observer = new MutationObserver(() => {
            scheduleRun();
        });

        observer.observe(document.documentElement || document, {
            subtree: true,
            childList: true,
            attributes: true,
            attributeFilter: ['class', 'style', 'hidden', 'disabled']
        });

        setInterval(runStateMachine, CHECK_INTERVAL);
        scheduleRun();
    }

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', start, { once: true });
    } else {
        start();
    }
})();
