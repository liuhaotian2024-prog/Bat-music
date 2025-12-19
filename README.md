<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>Bat-music · 从呼噜到乐曲</title>
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

    /* 调试条，默认隐藏，如需查看日志改成 block */
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
      overflow
