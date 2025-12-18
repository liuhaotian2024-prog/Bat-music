/* ========== 0. 全局主题与布局 ========== */

:root {
    --bg-main: #050608;
    --bg-stage: #111218;
    --bg-panel: #050509;
    --bg-panel-soft: #11141c;
    --border-soft: #222534;
    --accent-primary: #007aff;
    --accent-record: #ff3b30;
    --accent-ok: #34c759;
    --accent-warn: #f5a623;
    --text-main: #f5f5f5;
    --text-muted: #8c8f98;
    --radius-lg: 10px;
    --radius-sm: 6px;
}

html, body {
    margin: 0;
    padding: 0;
    height: 100%;
    background: var(--bg-main);
    color: var(--text-main);
    font-family: -apple-system, BlinkMacSystemFont, system-ui, sans-serif;
}

/* 整个 Bat-music 应用容器 */
.bat-app {
    display: flex;
    flex-direction: column;
    height: 100vh;
}

/* ========== 1. 调试控制台 ========== */

#debugConsole {
    position: fixed;
    top: 0; left: 0;
    width: 100%;
    min-height: 20px;
    max-height: 60px;
    background: #200;
    color: #ff6b6b;
    font-size: 10px;
    font-family: monospace;
    z-index: 999; /* 避免压过微信遮罩 */
    display: none;
    padding: 2px 6px;
    box-sizing: border-box;
    overflow-y: auto;
    white-space: pre-wrap;
}

/* ========== 2. 示波器 + BWL 层视图 ========== */

/* 上半部分：示波器区域（L0 波形 + L1 词元条等） */
.stage {
    position: relative;
    flex: 1 1 auto;
    min-height: 180px;
    background: radial-gradient(circle at top, #1a1c24 0, var(--bg-stage) 55%);
    border-bottom: 1px solid var(--border-soft);
    overflow: hidden;
}

/* 建议为主画布加类 .wave-canvas，而不是全局 canvas */
.wave-canvas {
    width: 100%;
    height: 100%;
    display: block;
}

/* 中央状态（计时 + 流程提示） */
.center-status {
    position: absolute;
    top: 50%; left: 50%;
    transform: translate(-50%, -50%);
    text-align: center;
    pointer-events: none;
}

.timer {
    font-size: 40px;
    font-weight: 600;
    font-family: monospace;
    letter-spacing: 0.06em;
}

.status-text {
    font-size: 13px;
    color: var(--text-muted);
    margin-top: 8px;
}

/* 状态小标签：录制中 / 词元化 / CIEU / 合成中… */
.status-badges {
    position: absolute;
    left: 16px;
    top: 14px;
    display: flex;
    gap: 6px;
    font-size: 10px;
}

.status-badge {
    padding: 2px 6px;
    border-radius: 999px;
    background: rgba(0,0,0,0.4);
    border: 1px solid rgba(255,255,255,0.08);
    color: var(--text-muted);
}

.status-badge--active {
    color: #fff;
    border-color: var(--accent-primary);
    box-shadow: 0 0 12px rgba(0,122,255,0.35);
}

/* BWL 层标签（L0~L4）可叠加在底部 */
.bwl-layers {
    position: absolute;
    left: 16px;
    bottom: 12px;
    display: flex;
    gap: 4px;
    font-size: 9px;
}

.bwl-layer-tag {
    padding: 2px 6px;
    border-radius: 999px;
    background: rgba(8,8,12,0.7);
    border: 1px solid rgba(255,255,255,0.06);
    color: var(--text-muted);
}

.bwl-layer-tag--focus {
    border-color: var(--accent-ok);
    color: #e0ffe5;
}

/* ========== 3. CIEU 因果面板（预留） ========== */

.cieu-panel {
    position: relative;
    padding: 10px 16px 6px;
    background: var(--bg-panel);
    border-bottom: 1px solid var(--border-soft);
    display: grid;
    grid-template-columns: repeat(5, minmax(0, 1fr));
    gap: 6px;
    font-size: 11px;
    box-sizing: border-box;
}

.cieu-item {
    background: var(--bg-panel-soft);
    border-radius: var(--radius-sm);
    padding: 4px 6px;
    border: 1px solid rgba(255,255,255,0.06);
    min-height: 32px;
    box-sizing: border-box;
}

.cieu-item__label {
    font-size: 10px;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: var(--text-muted);
    margin-bottom: 2px;
}

.cieu-item__value {
    font-size: 11px;
    color: #f0f0f0;
    line-height: 1.2;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

/* 高亮当前被优化的目标 Y*、奖励 r */
.cieu-item--target {
    border-color: var(--accent-primary);
}

.cieu-item--reward {
    border-color: var(--accent-ok);
}

/* ========== 4. 控制区（录制 + 目标选择 + 播放） ========== */

.controls {
    flex: 0 0 auto;
    background: var(--bg-panel);
    padding: 12px 16px 18px;
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    gap: 10px;
}

/* 4.1 目标风格 / Y* 选项 */
.target-row {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 4px;
}

.target-label {
    font-size: 11px;
    color: var(--text-muted);
    margin-right: 4px;
}

.target-chip {
    padding: 4px 8px;
    border-radius: 999px;
    border: 1px solid rgba(255,255,255,0.08);
    background: rgba(12,14,20,0.85);
    font-size: 11px;
    cursor: pointer;
    user-select: none;
}

.target-chip--active {
    border-color: var(--accent-primary);
    background: rgba(0,122,255,0.2);
    color: #e5f1ff;
}

/* 4.2 播放器层（展开/收起） */
#playerLayer {
    max-height: 0;
    opacity: 0;
    overflow: hidden;
    transition: max-height 0.25s ease, opacity 0.25s ease;
    background: #181a20;
    border-radius: var(--radius-lg);
    border: 1px solid transparent;
    box-sizing: border-box;
}

#playerLayer.show {
    max-height: 70px;
    opacity: 1;
    border-color: var(--border-soft);
}

#playerLayer audio {
    width: 100%;
    display: block;
    height: 40px;
    outline: none;
}

/* 4.3 按钮组 */
.btn-row {
    display: flex;
    gap: 12px;
}

button {
    flex: 1;
    border: none;
    border-radius: var(--radius-lg);
    font-size: 15px;
    font-weight: 600;
    cursor: pointer;
    transition: transform 0.08s ease, opacity 0.12s ease, box-shadow 0.12s ease;
    padding: 10px 0;
}

button:active {
    opacity: 0.8;
    transform: translateY(1px);
}

button:disabled {
    background: #222;
    color: #555;
    cursor: default;
    box-shadow: none;
}

/* 状态颜色 */
.btn-rec    { background: var(--accent-record); color: #fff; }
.btn-stop   { background: #3a3a3c; color: #fefefe; }
.btn-save   { background: var(--accent-ok); color: #000; }
.btn-reset  { background: #2c2c30; color: var(--text-muted); }

/* 可选：给录制中的按钮加跳动光晕 */
.btn-rec.recording {
    box-shadow: 0 0 18px rgba(255,59,48,0.6);
}

/* ========== 5. 微信遮罩 ========== */

#wxMask {
    position: fixed;
    inset: 0;
    background: rgba(0,0,0,0.9);
    z-index: 1000;
    display: none; /* 默认隐藏 */
    justify-content: center;
    align-items: center;
    text-align: center;
    padding: 24px;
    box-sizing: border-box;
    color: #fff;
}

/* 显示遮罩时使用 flex 以启用居中对齐 */
#wxMask.show {
    display: flex;
}

.wxMask-content {
    max-width: 320px;
    font-size: 14px;
    line-height: 1.5;
    opacity: 0.95;
}

/* ========== 6. 响应式微调 ========== */

@media (max-height: 640px) {
    .controls {
        padding-top: 8px;
        padding-bottom: 10px;
        gap: 8px;
    }
    .timer {
        font-size: 30px;
    }
}
