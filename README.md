<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>Bat-music · 波形因果音乐实验台</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />

  <style>
    :root {
      --bg-main: #050608;
      --bg-stage: #111218;
      --bg-panel: #050509;
      --bg-panel-soft: #11141c;
      --border-soft: #222534;
      --accent-primary: #007aff;
      --accent-record: #ff3b30;
      --accent-ok: #34c759;
      --text-main: #f5f5f5;
      --text-muted: #8c8f98;
      --radius-lg: 10px;
      --radius-sm: 6px;
    }

    * {
      box-sizing: border-box;
    }

    html,
    body {
      margin: 0;
      padding: 0;
      height: 100%;
      background: var(--bg-main);
      color: var(--text-main);
      font-family: -apple-system, BlinkMacSystemFont, system-ui, sans-serif;
    }

    /* 调试控制台 */
    #debugConsole {
      position: fixed;
      top: 0;
      left: 0;
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

    .bat-app {
      display: flex;
      flex-direction: column;
      height: 100vh;
    }

    /* 顶部：示波器 */
    .stage {
      position: relative;
      flex: 1 1 auto;
      min-height: 180px;
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
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      text-align: center;
      pointer-events: none;
    }

    .timer {
      font-size: 36px;
      font-weight: 600;
      font-family: monospace;
      letter-spacing: 0.06em;
    }

    .status-text {
      font-size: 13px;
      color: var(--text-muted);
      margin-top: 8px;
    }

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
      background: rgba(0, 0, 0, 0.4);
      border: 1px solid rgba(255, 255, 255, 0.08);
      color: var(--text-muted);
    }

    .status-badge--active {
      color: #fff;
      border-color: var(--accent-primary);
      box-shadow: 0 0 10px rgba(0, 122, 255, 0.4);
    }

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
      background: rgba(8, 8, 12, 0.7);
      border: 1px solid rgba(255, 255, 255, 0.06);
      color: var(--text-muted);
    }

    .bwl-layer-tag--focus {
      border-color: var(--accent-ok);
      color: #e0ffe5;
    }

    /* CIEU 面板 */
    .cieu-panel {
      padding: 8px 12px;
      background: var(--bg-panel);
      border-bottom: 1px solid var(--border-soft);
      display: grid;
      grid-template-columns: repeat(5, minmax(0, 1fr));
      gap: 6px;
      font-size: 11px;
    }

    .cieu-item {
      background: var(--bg-panel-soft);
      border-radius: var(--radius-sm);
      padding: 4px 6px;
      border: 1px solid rgba(255, 255, 255, 0.06);
      min-height: 32px;
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
      line-height: 1.3;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }

    .cieu-item--target {
      border-color: var(--accent-primary);
    }

    .cieu-item--reward {
      border-color: var(--accent-ok);
    }

    /* 控制区 */
    .controls {
      flex: 0 0 auto;
      background: var(--bg-panel);
      padding: 10px 12px 14px;
      display: flex;
      flex-direction: column;
      gap: 10px;
    }

    .target-row {
      display: flex;
      flex-wrap: wrap;
      gap: 6px;
      align-items: center;
    }

    .target-label {
      font-size: 11px;
      color: var(--text-muted);
      margin-right: 2px;
    }

    .target-chip {
      padding: 4px 8px;
      border-radius: 999px;
      border: 1px solid rgba(255, 255, 255, 0.08);
      background: rgba(12, 14, 20, 0.85);
      font-size: 11px;
      cursor: pointer;
      user-select: none;
    }

    .target-chip--active {
      border-color: var(--accent-primary);
      background: rgba(0, 122, 255, 0.25);
      color: #e5f1ff;
    }

    #playerLayer {
      max-height: 0;
      opacity: 0;
      overflow: hidden;
      transition: max-height 0.25s ease, opacity 0.25s ease;
      background: #181a20;
      border-radius: var(--radius-lg);
      border: 1px solid transparent;
      margin-top: 4px;
    }

    #playerLayer.show {
      max-height: 80px;
      opacity: 1;
      border-color: var(--border-soft);
    }

    #audioPlayer {
      width: 100%;
      display: block;
      height: 44px;
      outline: none;
    }

    .btn-row {
      display: flex;
      gap: 10px;
    }

    button {
      flex: 1;
      border: none;
      border-radius: var(--radius-lg);
      font-size: 14px;
      font-weight: 600;
      cursor: pointer;
      transition: transform 0.08s ease, opacity 0.12s ease, box-shadow 0.12s ease;
      padding: 10px 4px;
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

    .btn-rec {
      background: var(--accent-record);
      color: #fff;
    }

    .btn-stop-gen {
      background: #3a3a3c;
      color: #fefefe;
    }

    .btn-save {
      background: var(--accent-ok);
      color: #000;
    }

    .btn-share {
      background: #2c2c30;
      color: var(--text-muted);
    }

    .btn-rec.recording {
      box-shadow: 0 0 16px rgba(255, 59, 48, 0.7);
    }

    /* 微信遮罩 */
    #wxMask {
      position: fixed;
      inset: 0;
      background: rgba(0, 0, 0, 0.9);
      z-index: 1000;
      display: none;
      justify-content: center;
      align-items: center;
      text-align: center;
      padding: 24px;
      color: #fff;
    }

    #wxMask.show {
      display: flex;
    }

    .wxMask-content {
      max-width: 320px;
      font-size: 14px;
      line-height: 1.5;
      opacity: 0.95;
    }

    @media (max-height: 640px) {
      .timer {
        font-size: 28px;
      }
      .controls {
        padding-top: 6px;
        padding-bottom: 8px;
      }
    }
  </style>
</head>
<body>
  <!-- 调试控制台 -->
  <div id="debugConsole"></div>

  <div class="bat-app">
    <!-- 1. 示波器 + BWL 层 -->
    <div class="stage">
      <canvas id="waveCanvas" class="wave-canvas"></canvas>

      <div class="center-status">
        <div id="timer" class="timer">00:00</div>
        <div id="statusText" class="status-text">等待录制呼噜声 / 口哨 / 敲桌子等怪声音…</div>
      </div>

      <div class="status-badges">
        <div class="status-badge" data-phase="idle">空闲</div>
        <div class="status-badge" data-phase="recording">录制中</div>
        <div class="status-badge" data-phase="cieu">CIEU 编码</div>
        <div class="status-badge" data-phase="generating">生成乐曲</div>
        <div class="status-badge" data-phase="playing">播放中</div>
      </div>

      <div class="bwl-layers">
        <div class="bwl-layer-tag" data-layer="L0">L0 物理</div>
        <div class="bwl-layer-tag" data-layer="L1">L1 词元</div>
        <div class="bwl-layer-tag" data-layer="L2">L2 句法</div>
        <div class="bwl-layer-tag" data-layer="L3">L3 语义</div>
        <div class="bwl-layer-tag" data-layer="L4">L4 语用</div>
      </div>
    </div>

    <!-- 2. CIEU 面板 -->
    <div class="cieu-panel">
      <div class="cieu-item" data-cieu="x">
        <div class="cieu-item__label">xₜ · 状态</div>
        <div class="cieu-item__value" id="cieu-x">—</div>
      </div>
      <div class="cieu-item" data-cieu="u">
        <div class="cieu-item__label">uₜ · 干预</div>
        <div classc="cieu-item__value" id="cieu-u">等待操作</div>
      </div>
      <div class="cieu-item cieu-item--target" data-cieu="yStar">
        <div class="cieu-item__label">y*ₜ · 目标</div>
        <div class="cieu-item__value" id="cieu-y-star">未选择</div>
      </div>
      <div class="cieu-item" data-cieu="yOutcome">
        <div class="cieu-item__label">yₜ+Δ · 结果</div>
        <div class="cieu-item__value" id="cieu-y-outcome">—</div>
      </div>
      <div class="cieu-item cieu-item--reward" data-cieu="r">
        <div class="cieu-item__label">rₜ+Δ · 奖励</div>
        <div class="cieu-item__value" id="cieu-r">—</div>
      </div>
    </div>

    <!-- 3. 控制区 -->
    <div class="controls">
      <!-- Y* 目标选择 -->
      <div class="target-row">
        <span class="target-label">音乐风格 (y*):</span>
        <div class="target-chip" data-style="symphony">交响</div>
        <div class="target-chip" data-style="jazz">爵士</div>
        <div class="target-chip" data-style="rap">说唱</div>
        <div class="target-chip" data-style="ambient">氛围</div>
        <div class="target-chip" data-style="lofi">LoFi</div>
        <div class="target-chip" data-style="sleep">助眠白噪</div>
      </div>

      <!-- 播放器 -->
      <div id="playerLayer">
        <audio id="audioPlayer" controls></audio>
      </div>

      <!-- 四个按钮：开始录音 / 停止并生成乐曲 / 保存 / 分享 -->
      <div class="btn-row">
        <button id="btnRecord" class="btn-rec">开始录音</button>
        <button id="btnStopGen" class="btn-stop-gen" disabled>停止并生成乐曲</button>
        <button id="btnSave" class="btn-save" disabled>保存到本地</button>
        <button id="btnShare" class="btn-share" disabled>分享链接</button>
      </div>
    </div>
  </div>

  <!-- 微信遮罩（在不支持录音的环境使用） -->
  <div id="wxMask">
    <div class="wxMask-content">
      <p>当前环境可能限制音频录制或播放。</p>
      <p>请在系统浏览器中打开本页面，或检查麦克风权限设置。</p>
    </div>
  </div>

  <script>
    const batState = {
      phase: 'idle', // idle | recording | cieu | generating | playing
      targetStyle: null,
      recordedBlob: null,
      generatedBlob: null,
      audioBuffer: null,
      cieu: {
        x: null,
        u: null,
        yStar: null,
        yOutcome: null,
        reward: null
      }
    };

    let mediaRecorder = null;
    let recordedChunks = [];
    let audioContext = null;
    let analyser = null;
    let micSource = null;
    let drawing = false;
    let recordingStartTime = null;
    let timerInterval = null;

    let elTimer,
      elStatusText,
      elWaveCanvas,
      waveCtx,
      elStatusBadges,
      elBwlTags,
      elCieuX,
      elCieuU,
      elCieuYStar,
      elCieuYOutcome,
      elCieuR,
      btnRecord,
      btnStopGen,
      btnSave,
      btnShare,
      elPlayerLayer,
      elAudioPlayer,
      targetChips,
      debugConsole;

    window.addEventListener('DOMContentLoaded', () => {
      elTimer = document.getElementById('timer');
      elStatusText = document.getElementById('statusText');
      elWaveCanvas = document.getElementById('waveCanvas');
      waveCtx = elWaveCanvas.getContext('2d');

      elStatusBadges = document.querySelectorAll('.status-badge');
      elBwlTags = document.querySelectorAll('.bwl-layer-tag');

      elCieuX = document.getElementById('cieu-x');
      elCieuU = document.getElementById('cieu-u');
      elCieuYStar = document.getElementById('cieu-y-star');
      elCieuYOutcome = document.getElementById('cieu-y-outcome');
      elCieuR = document.getElementById('cieu-r');

      btnRecord = document.getElementById('btnRecord');
      btnStopGen = document.getElementById('btnStopGen');
      btnSave = document.getElementById('btnSave');
      btnShare = document.getElementById('btnShare');

      elPlayerLayer = document.getElementById('playerLayer');
      elAudioPlayer = document.getElementById('audioPlayer');

      targetChips = document.querySelectorAll('.target-chip');
      debugConsole = document.getElementById('debugConsole');

      attachEvents();
      resizeCanvas();
      drawIdleWave();
      setPhase('idle');
      updateCIEUPanel();

      window.addEventListener('resize', () => {
        resizeCanvas();
        drawCurrentWave();
      });

      window.addEventListener('error', (e) => {
        logDebug('Error: ' + e.message);
      });
    });

    function attachEvents() {
      btnRecord.addEventListener('click', startRecording);
      btnStopGen.addEventListener('click', stopAndGenerate);
      btnSave.addEventListener('click', saveGenerated);
      btnShare.addEventListener('click', sharePage);

      targetChips.forEach((chip) => {
        chip.addEventListener('click', () => {
          targetChips.forEach((c) => c.classList.remove('target-chip--active'));
          chip.classList.add('target-chip--active');
          const style = chip.getAttribute('data-style');
          batState.targetStyle = style;
          batState.cieu.yStar = style;
          updateCIEUPanel();
        });
      });

      elAudioPlayer.addEventListener('play', () => setPhase('playing'));
      elAudioPlayer.addEventListener('pause', () => {
        if (batState.phase === 'playing') setPhase('cieu');
      });
      elAudioPlayer.addEventListener('ended', () => setPhase('cieu'));
    }

    function setPhase(phase) {
      batState.phase = phase;

      switch (phase) {
        case 'idle':
          elStatusText.textContent = '等待录制呼噜声 / 怪声音…';
          break;
        case 'recording':
          elStatusText.textContent = '录制中：请发出你希望“变乐曲”的声音…';
          break;
        case 'cieu':
          elStatusText.textContent = '已构建 CIEU，可反复生成 / 调参。';
          break;
        case 'generating':
          elStatusText.textContent = '基于 y* 生成乐曲（当前为占位逻辑）…';
          break;
        case 'playing':
          elStatusText.textContent = '播放当前乐曲…';
          break;
      }

      elStatusBadges.forEach((badge) => {
        const p = badge.getAttribute('data-phase');
        badge.classList.toggle('status-badge--active', p === phase);
      });

      const focusLayers = [];
      if (phase === 'recording') focusLayers.push('L0');
      if (phase === 'cieu') focusLayers.push('L1', 'L2', 'L3');
      if (phase === 'generating' || phase === 'playing') focusLayers.push('L4');

      elBwlTags.forEach((tag) => {
        const layer = tag.getAttribute('data-layer');
        tag.classList.toggle('bwl-layer-tag--focus', focusLayers.includes(layer));
      });

      updateButtons();
    }

    function updateButtons() {
      const hasRecording = !!batState.recordedBlob;
      const hasGenerated = !!batState.generatedBlob;

      if (batState.phase === 'recording') {
        btnRecord.disabled = true;
        btnStopGen.disabled = false;
      } else {
        btnRecord.disabled = false;
        btnStopGen.disabled = !hasRecording && batState.phase !== 'recording';
      }

      btnSave.disabled = !hasGenerated;
      btnShare.disabled = !hasGenerated;

      if (batState.phase === 'recording') {
        btnRecord.classList.add('recording');
      } else {
        btnRecord.classList.remove('recording');
      }
    }

    function resizeCanvas() {
      const rect = elWaveCanvas.getBoundingClientRect();
      const dpr = window.devicePixelRatio || 1;
      elWaveCanvas.width = rect.width * dpr;
      elWaveCanvas.height = rect.height * dpr;
      waveCtx.setTransform(dpr, 0, 0, dpr, 0, 0);
    }

    function drawIdleWave() {
      const ctx = waveCtx;
      const { width, height } = elWaveCanvas.getBoundingClientRect();
      ctx.clearRect(0, 0, width, height);
      ctx.strokeStyle = '#333';
      ctx.lineWidth = 1;
      ctx.beginPath();
      for (let x = 0; x < width; x++) {
        const y = height / 2 + Math.sin(x / 20) * 4;
        if (x === 0) ctx.moveTo(x, y);
        else ctx.lineTo(x, y);
      }
      ctx.stroke();
    }

    function drawCurrentWave() {
      if (!batState.audioBuffer) {
        drawIdleWave();
        return;
      }
      drawWaveformFromBuffer(batState.audioBuffer);
    }

    function drawWaveformFromBuffer(buffer) {
      const ctx = waveCtx;
      const { width, height } = elWaveCanvas.getBoundingClientRect();
      ctx.clearRect(0, 0, width, height);
      const channelData = buffer.getChannelData(0);
      const step = Math.floor(channelData.length / width) || 1;
      const amp = (height / 2) * 0.85;

      ctx.beginPath();
      ctx.moveTo(0, height / 2);

      for (let x = 0; x < width; x++) {
        let sum = 0;
        const start = x * step;
        const end = Math.min(start + step, channelData.length);
        for (let i = start; i < end; i++) sum += Math.abs(channelData[i]);
        const avg = sum / (end - start || 1);
        const sign = channelData[start] >= 0 ? 1 : -1;
        const y = height / 2 - avg * amp * sign;
        ctx.lineTo(x, y);
      }

      ctx.strokeStyle = '#4cd964';
      ctx.lineWidth = 1.5;
      ctx.stroke();
    }

    async function startRecording() {
      try {
        if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
          showWxMask();
          return;
        }

        if (!audioContext) {
          audioContext = new (window.AudioContext || window.webkitAudioContext)();
        }

        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });

        micSource = audioContext.createMediaStreamSource(stream);
        analyser = audioContext.createAnalyser();
        analyser.fftSize = 2048;
        micSource.connect(analyser);

        mediaRecorder = new MediaRecorder(stream);
        recordedChunks = [];

        mediaRecorder.ondataavailable = (e) => {
          if (e.data.size > 0) recordedChunks.push(e.data);
        };

        mediaRecorder.onstop = handleRecordingStop;

        mediaRecorder.start();
        startTimer();
        startDrawing();

        batState.recordedBlob = null;
        batState.generatedBlob = null;
        batState.audioBuffer = null;
        batState.cieu = {
          x: null,
          u: null,
          yStar: batState.targetStyle,
          yOutcome: null,
          reward: null
        };
        updateCIEUPanel();

        setPhase('recording');
      } catch (err) {
        logDebug('录音启动失败: ' + err.message);
        showWxMask();
      }
    }

    function stopAndGenerate() {
      if (mediaRecorder && mediaRecorder.state === 'recording') {
        mediaRecorder.stop();
        stopTimer();
        stopDrawing();
        setPhase('generating'); // 先切到 generating，onstop 里完成 CIEU 填写
      }
    }

    async function handleRecordingStop() {
      try {
        const blob = new Blob(recordedChunks, { type: 'audio/webm' });
        batState.recordedBlob = blob;

        if (!audioContext) {
          audioContext = new (window.AudioContext || window.webkitAudioContext)();
        }

        const arrayBuffer = await blob.arrayBuffer();
        const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);
        batState.audioBuffer = audioBuffer;
        drawWaveformFromBuffer(audioBuffer);

        extractFeaturesToCIEU(audioBuffer);
        batState.cieu.u = buildInterventionDescription();

        const generatedBlob = await fakeGenerateFromRecorded(blob);
        batState.generatedBlob = generatedBlob;

        const url = URL.createObjectURL(generatedBlob);
        elAudioPlayer.src = url;
        elPlayerLayer.classList.add('show');

        batState.cieu.yOutcome = {
          source: 'placeholder',
          desc: '当前用原始录音作为乐曲 yₜ+Δ（占位版）'
        };
        const reward = batState.cieu.yStar ? 1.0 : 0.2;
        batState.cieu.reward = reward;

        updateCIEUPanel();
        setPhase('cieu');
      } catch (err) {
        logDebug('录音处理/生成失败: ' + err.message);
        setPhase('idle');
      }
    }

    function startDrawing() {
      drawing = true;
      const { width, height } = elWaveCanvas.getBoundingClientRect();
      const bufferLength = analyser.fftSize;
      const dataArray = new Uint8Array(bufferLength);

      function draw() {
        if (!drawing || !analyser) {
          drawIdleWave();
          return;
        }
        analyser.getByteTimeDomainData(dataArray);
        waveCtx.clearRect(0, 0, width, height);
        waveCtx.lineWidth = 1.5;
        waveCtx.strokeStyle = '#4cd964';
        waveCtx.beginPath();
        const sliceWidth = width / bufferLength;
        let x = 0;

        for (let i = 0; i < bufferLength; i++) {
          const v = dataArray[i] / 128.0;
          const y = (v * height) / 2;
          if (i === 0) waveCtx.moveTo(x, y);
          else waveCtx.lineTo(x, y);
          x += sliceWidth;
        }
        waveCtx.stroke();

        requestAnimationFrame(draw);
      }

      draw();
    }

    function stopDrawing() {
      drawing = false;
    }

    function extractFeaturesToCIEU(buffer) {
      const duration = buffer.duration;
      const sampleRate = buffer.sampleRate;
      const data = buffer.getChannelData(0);
      let absSum = 0;
      let maxAmp = 0;
      for (let i = 0; i < data.length; i++) {
        const v = data[i];
        const av = Math.abs(v);
        absSum += av;
        if (av > maxAmp) maxAmp = av;
      }
      const meanAmp = absSum / data.length;
      const text =
        'len=' +
        duration.toFixed(2) +
        's, sr=' +
        sampleRate +
        'Hz, meanAmp=' +
        meanAmp.toFixed(3) +
        ', peak=' +
        maxAmp.toFixed(3);

      batState.cieu.x = {
        duration,
        sampleRate,
        meanAmp,
        maxAmp,
        text
      };
      elCieuX.textContent = text;
    }

    function buildInterventionDescription() {
      const style = batState.targetStyle ? batState.targetStyle : 'none';
      return 'op=mic_record, style=' + style;
    }

    function fakeGenerateFromRecorded(blob) {
      // 这里将来可以替换为真正的 Bat-mind / 波形因果生成调用
      return new Promise((resolve) => {
        setTimeout(() => resolve(blob), 300);
      });
    }

    function updateCIEUPanel() {
      if (batState.cieu.x && batState.cieu.x.text) {
        elCieuX.textContent = batState.cieu.x.text;
      } else {
        elCieuX.textContent = '—';
      }

      elCieuU.textContent = batState.cieu.u || '等待操作';

      if (batState.cieu.yStar) {
        elCieuYStar.textContent = batState.cieu.yStar;
      } else {
        elCieuYStar.textContent = '未选择';
      }

      if (batState.cieu.yOutcome && batState.cieu.yOutcome.desc) {
        elCieuYOutcome.textContent = batState
