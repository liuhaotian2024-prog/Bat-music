<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>Bat-music · 从呼噜到地狱乐曲</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />

  <style>
    :root {
      --bg-main: #050608;
      --bg-stage: #111218;
      --bg-panel: #050509;
      --border-soft: #222534;
      --accent-primary: #007aff;
      --accent-record: #ff3b30;
      --accent-ok: #34c759;
      --text-main: #f5f5f5;
      --text-muted: #8c8f98;
      --radius-lg: 10px;
    }

    * { box-sizing: border-box; }

    html, body {
      margin: 0;
      padding: 0;
      height: 100%;
      background: var(--bg-main);
      color: var(--text-main);
      font-family: -apple-system, BlinkMacSystemFont, system-ui, sans-serif;
    }

    /* 调试条，调试时改成 display:block */
    #debugConsole {
      position: fixed;
      top: 0; left: 0;
      width: 100%;
      min-height: 18px;
      max-height: 60px;
      background: #200;
      color: #ff6b6b;
      font-size: 10px;
      font-family: monospace;
      z-index: 999;
      display: none;
      padding: 2px 6px;
      overflow-y: auto;
      white-space: pre-wrap;
    }

    .app {
      display: flex;
      flex-direction: column;
      height: 100vh;
    }

    /* 顶部：示波器 */
    .stage {
      position: relative;
      flex: 1 1 auto;
      min-height: 200px;
      background: radial-gradient(circle at top, #1a1c24 0, var(--bg-stage) 55%);
      border-bottom: 1px solid var(--border-soft);
      overflow: hidden;
    }

    .wave-canvas {
      width: 100%;
      height: 100%;
      display: block;
    }

    .center-status {
      position: absolute;
      top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      text-align: center;
      pointer-events: none;
    }

    .timer {
      font-size: 38px;
      font-weight: 600;
      font-family: monospace;
      letter-spacing: 0.06em;
    }

    .status-text {
      font-size: 13px;
      color: var(--text-muted);
      margin-top: 8px;
    }

    .status-badge-row {
      position: absolute;
      left: 12px;
      top: 12px;
      display: flex;
      gap: 6px;
      font-size: 10px;
    }

    .status-badge {
      padding: 2px 6px;
      border-radius: 999px;
      background: rgba(0,0,0,0.35);
      border: 1px solid rgba(255,255,255,0.08);
      color: #999;
    }

    .status-badge--active {
      color: #fff;
      border-color: var(--accent-primary);
      box-shadow: 0 0 10px rgba(0,122,255,0.5);
    }

    /* 中部：本次乐曲气质说明 */
    .summary {
      pad
