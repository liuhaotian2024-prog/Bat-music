<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>Bat-music · 从呼噜到地狱作曲（PCM版）</title>
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

    #debugConsole {
      position: fixed;
      top: 0; left: 0;
      width: 100%;
      min-height: 18px;
      max-height: 80px;
      background: #200;
      color: #ff6b6b;
      font-size: 10px;
      font-family: monospace;
      z-index: 999;
      display: block;
      padding: 2px 6px;
      overflow-y: auto;
      white-space: pre-wrap;
    }

    .app {
      display: flex;
      flex-direction: column;
      height: 100vh;
      padding-top: 80px;
      box-sizing: border-box;
    }

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

    .summary {
      padding: 10px 14px 6px;
      background: var(--bg-panel);
      border-bottom: 1px solid var(--border-soft);
      font-size: 13px;
      color: var(--text-muted);
    }

    .summary-title {
      font-size: 12px;
      color: #aaa;
      margin-bottom: 4px;
    }

    .summary-text {
      font-size: 13px;
      color: #f5f5f5;
      line-height: 1.4;
      min-height: 2.6em;
    }

    .controls {
      flex: 0 0 auto;
      background: var(--bg-panel);
      padding: 10px 12px 14px;
      display: flex;
      flex-direction: column;
      gap: 10px;
    }

    #playerLayer {
      max-height: 0;
      opacity: 0;
      overflow: hidden;
      transition: max-height .25s ease, opacity .25s ease;
      background: #181a20;
      border-radius: var(--radius-lg);
      border: 1px solid transparent;
    }

    #playerLayer.show {
      max-height: 90px;
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
      gap: 8px;
      margin-top: 4px;
    }

    button {
      flex: 1;
      border: none;
      border-radius: var(--radius-lg);
      font-size: 14px;
      font-weight: 600;
      cursor: pointer;
      padding: 10px 4px;
      transition: transform .08s ease, opacity .12s ease, box-shadow .12s ease;
    }

    button:active {
      opacity: 0.85;
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
      box-shadow: 0 0 16px rgba(255,59,48,0.7);
    }

    #wxMask {
      position: fixed;
      inset: 0;
      background: rgba(0,0,0,0.9);
      z-index: 1000;
      display: none;
      justify-content: center;
      align-items: center;
      text-align: center;
      padding: 24px;
      color: #fff;
    }

    #wxMask.show { display: flex; }

    .wxMask-content {
      max-width: 320px;
      font-size: 14px;
      line-height: 1.5;
      opacity: 0.95;
    }

    @media (max-height: 640px) {
      .timer { font-size: 30px; }
      .controls { padding-bottom: 10px; }
    }
  </style>
</head>
<body>
  <div id="debugConsole">Bat-music 调试输出：<br/></div>

  <div class="app">
    <div class="stage">
      <canvas id="waveCanvas" class="wave-canvas"></canvas>

      <div class="center-status">
        <div id="timer" class="timer">00:00</div>
        <div id="statusText" class="status-text">
          等待录制呼噜声 / 口哨 / 敲桌子等怪声音…
        </div>
      </div>

      <div class="status-badge-row">
        <div class="status-badge" data-phase="idle">空闲</div>
        <div class="status-badge" data-phase="recording">录制中</div>
        <div class="status-badge" data-phase="generating">作曲中</div>
        <div class="status-badge" data-phase="generated">已生成乐曲</div>
        <div class="status-badge" data-phase="playing">播放中</div>
      </div>
    </div>

    <div class="summary">
      <div class="summary-title">本次乐曲的“气质”：</div>
      <div id="summaryText" class="summary-text">
        还没有乐曲。录一段呼噜 / 怪声，我会把它“炼”成一段真正的地狱乐曲。
      </div>
    </div>

    <div class="controls">
      <div id="playerLayer">
        <audio id="audioPlayer" controls></audio>
      </div>

      <div class="btn-row">
        <button id="btnRecord" class="btn-rec">开始录音</button>
        <button id="btnStopGen" class="btn-stop-gen" disabled>停止并生成乐曲</button>
        <button id="btnSave" class="btn-save" disabled>保存到本地</button>
        <button id="btnShare" class="btn-share" disabled>分享链接</button>
      </div>
    </div>
  </div>

  <div id="wxMask">
    <div class="wxMask-content">
      <p>当前浏览器可能不支持实时录音。</p>
      <p>建议在系统浏览器（Safari / Chrome）中打开本页面，并确认授权麦克风。</p>
    </div>
  </div>

  <script>
    /********** 全局状态 **********/
    const batState = {
      phase: 'idle',
      floatData: null,
      sampleRate: 44100,
      generatedBlob: null
    };

    let audioContext = null;
    let micStream = null;
    let sourceNode = null;
    let analyser = null;
    let processor = null;

    let recordedBuffers = [];
    let recordingLength = 0;

    let elCanvas, canvasCtx;
    let drawing = false;

    let elTimer, elStatusText, elStatusBadges;
    let btnRecord, btnStopGen, btnSave, btnShare;
    let elPlayerLayer, elAudioPlayer, elSummary, debugConsole;

    let timerInterval = null;
    let recordingStartTime = null;

    /********** 初始化 **********/
    window.addEventListener('DOMContentLoaded', () => {
      debugConsole = document.getElementById('debugConsole');
      logDebug('DOMContentLoaded');

      elCanvas = document.getElementById('waveCanvas');
      canvasCtx = elCanvas.getContext('2d');

      elTimer = document.getElementById('timer');
      elStatusText = document.getElementById('statusText');
      elStatusBadges = document.querySelectorAll('.status-badge');

      btnRecord = document.getElementById('btnRecord');
      btnStopGen = document.getElementById('btnStopGen');
      btnSave = document.getElementById('btnSave');
      btnShare = document.getElementById('btnShare');

      elPlayerLayer = document.getElementById('playerLayer');
      elAudioPlayer = document.getElementById('audioPlayer');
      elSummary = document.getElementById('summaryText');

      attachEvents();
      resizeCanvas();
      drawIdleWave();
      setPhase('idle');

      window.addEventListener('resize', () => {
        resizeCanvas();
        drawCurrentWave();
      });

      window.addEventListener('error', (e) => {
        logDebug('window error: ' + e.message);
      });
    });

    function attachEvents() {
      btnRecord.addEventListener('click', () => {
        logDebug('点击开始录音');
        startRecording();
      });
      btnStopGen.addEventListener('click', () => {
        logDebug('点击停止并生成乐曲');
        stopAndGenerate();
      });
      btnSave.addEventListener('click', saveGenerated);
      btnShare.addEventListener('click', sharePage);

      elAudioPlayer.addEventListener('play', () => setPhase('playing'));
      elAudioPlayer.addEventListener('pause', () => {
        if (batState.phase === 'playing') setPhase('generated');
      });
      elAudioPlayer.addEventListener('ended', () => setPhase('generated'));

      logDebug('事件绑定完成');
    }

    /********** Phase & UI **********/
    function setPhase(phase) {
      batState.phase = phase;
      logDebug('Phase -> ' + phase);

      switch (phase) {
        case 'idle':
          elStatusText.textContent = '等待录制呼噜声 / 口哨 / 敲桌子等怪声音…';
          break;
        case 'recording':
          elStatusText.textContent = '录制中：请发出你想“炼成乐曲”的声音…';
          break;
        case 'generating':
          elStatusText.textContent = '根据噪音场与乐音场的张力，在地狱熔炉中作曲（PCM 合成）…';
          break;
        case 'generated':
          elStatusText.textContent = '已生成地狱乐曲，可以播放 / 保存 / 分享。';
          break;
        case 'playing':
          elStatusText.textContent = '播放当前地狱乐曲中…';
          break;
      }

      elStatusBadges.forEach(badge => {
        const p = badge.getAttribute('data-phase');
        badge.classList.toggle('status-badge--active', p === phase);
      });

      updateButtons();
    }

    function updateButtons() {
      const hasGenerated = !!batState.generatedBlob;

      if (batState.phase === 'recording') {
        btnRecord.disabled = true;
        btnStopGen.disabled = false;
      } else {
        btnRecord.disabled = false;
        btnStopGen.disabled = (batState.phase !== 'recording');
      }

      btnSave.disabled = !hasGenerated;
      btnShare.disabled = !hasGenerated;

      btnRecord.classList.toggle('recording', batState.phase === 'recording');
    }

    /********** Canvas **********/
    function resizeCanvas() {
      const rect = elCanvas.getBoundingClientRect();
      const dpr = window.devicePixelRatio || 1;
      elCanvas.width = rect.width * dpr;
      elCanvas.height = rect.height * dpr;
      canvasCtx.setTransform(dpr, 0, 0, dpr, 0, 0);
    }

    function drawIdleWave() {
      const ctx = canvasCtx;
      const { width, height } = elCanvas.getBoundingClientRect();
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
      if (!batState.floatData) {
        drawIdleWave();
        return;
      }
      drawWaveformFromFloat(batState.floatData);
    }

    function drawWaveformFromFloat(data) {
      const ctx = canvasCtx;
      const { width, height } = elCanvas.getBoundingClientRect();
      ctx.clearRect(0, 0, width, height);

      const samplesPerPixel = Math.max(1, Math.floor(data.length / width));
      const amp = height / 2 * 0.9;
      ctx.beginPath();
      ctx.moveTo(0, height / 2);

      for (let x = 0; x < width; x++) {
        const start = x * samplesPerPixel;
        const end = Math.min(start + samplesPerPixel, data.length);
        let sum = 0;
        for (let i = start; i < end; i++) sum += Math.abs(data[i]);
        const avg = sum / (end - start || 1);
        const sign = data[start] >= 0 ? 1 : -1;
        const y = height / 2 - avg * amp * sign;
        ctx.lineTo(x, y);
      }

      ctx.strokeStyle = '#4cd964';
      ctx.lineWidth = 1.5;
      ctx.stroke();
    }

    function startDrawingFromAnalyser() {
      if (!analyser) return;
      drawing = true;
      const { width, height } = elCanvas.getBoundingClientRect();
      const bufferLength = analyser.fftSize;
      const dataArray = new Uint8Array(bufferLength);

      function draw() {
        if (!drawing || !analyser) {
          drawIdleWave();
          return;
        }
        analyser.getByteTimeDomainData(dataArray);
        canvasCtx.clearRect(0, 0, width, height);
        canvasCtx.lineWidth = 1.5;
        canvasCtx.strokeStyle = '#4cd964';
        canvasCtx.beginPath();
        const sliceWidth = width / bufferLength;
        let x = 0;
        for (let i = 0; i < bufferLength; i++) {
          const v = dataArray[i] / 128.0;
          const y = v * height / 2;
          if (i === 0) canvasCtx.moveTo(x, y);
          else canvasCtx.lineTo(x, y);
          x += sliceWidth;
        }
        canvasCtx.stroke();
        requestAnimationFrame(draw);
      }
      draw();
    }

    function stopDrawing() {
      drawing = false;
    }

    /********** 录音管线（保留原来稳定版本） **********/
    async function startRecording() {
      try {
        if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
          logDebug('浏览器不支持 getUserMedia');
          showWxMask();
          return;
        }

        if (!audioContext) {
          audioContext = new (window.AudioContext || window.webkitAudioContext)();
          logDebug('创建 AudioContext, sampleRate=' + audioContext.sampleRate);
        }
        await audioContext.resume();

        micStream = await navigator.mediaDevices.getUserMedia({ audio: true });
        batState.sampleRate = audioContext.sampleRate;

        sourceNode = audioContext.createMediaStreamSource(micStream);

        analyser = audioContext.createAnalyser();
        analyser.fftSize = 2048;
        sourceNode.connect(analyser);

        processor = audioContext.createScriptProcessor(4096, 1, 1);
        sourceNode.connect(processor);
        processor.connect(audioContext.destination);

        recordedBuffers = [];
        recordingLength = 0;

        processor.onaudioprocess = (event) => {
          const input = event.inputBuffer.getChannelData(0);
          const buffer = new Float32Array(input.length);
          buffer.set(input);
          recordedBuffers.push(buffer);
          recordingLength += buffer.length;
        };

        startTimer();
        startDrawingFromAnalyser();
        setPhase('recording');
      } catch (err) {
        logDebug('startRecording 失败: ' + err.message);
        showWxMask();
      }
    }

    async function stopAndGenerate() {
      if (batState.phase !== 'recording') return;
      stopTimer();
      stopDrawing();
      stopAudioGraph();

      if (recordingLength === 0) {
        logDebug('录音长度为 0');
        setPhase('idle');
        return;
      }

      const merged = new Float32Array(recordingLength);
      let offset = 0;
      for (const buf of recordedBuffers) {
        merged.set(buf, offset);
        offset += buf.length;
      }

      const normalized = normalizeFloat(merged);
      batState.floatData = normalized;

      setPhase('generating');
      await generateFromRecording(normalized, batState.sampleRate);
    }

    function stopAudioGraph() {
      if (processor) {
        processor.disconnect();
        processor.onaudioprocess = null;
        processor = null;
      }
      if (sourceNode) {
        sourceNode.disconnect();
        sourceNode = null;
      }
      if (analyser) {
        analyser.disconnect();
        analyser = null;
      }
      if (micStream) {
        micStream.getTracks().forEach(t => t.stop());
        micStream = null;
      }
    }

    /********** 地狱作曲：纯 PCM 合成 **********/
    async function generateFromRecording(floatData, sampleRate) {
      try {
        const analysis = analyzeAndDescribe(floatData, sampleRate);
        elSummary.textContent = analysis.summary;

        const composed = composePCM(analysis.features, analysis.styleId, floatData, sampleRate);
        batState.floatData = composed;
        const wavBlob = encodeWAV(composed, sampleRate);
        batState.generatedBlob = wavBlob;

        const url = URL.createObjectURL(wavBlob);
        elAudioPlayer.src = url;
        elPlayerLayer.classList.add('show');

        drawCurrentWave();
        logDebug('作曲样本数=' + composed.length + ', wavSize=' + wavBlob.size + ' 字节');

        setPhase('generated');
      } catch (err) {
        logDebug('generateFromRecording 失败: ' + err.message);
        const blob = encodeWAV(floatData, sampleRate);
        batState.generatedBlob = blob;
        const url = URL.createObjectURL(blob);
        elAudioPlayer.src = url;
        elPlayerLayer.classList.add('show');
        setPhase('generated');
      }
    }

    /********** 噪音场分析 + 风格 / 特征 **********/
    function analyzeAndDescribe(data, sampleRate) {
      const n = data.length;
      if (n === 0) {
        return {
          summary: '录音为空。重新试一次吧。',
          styleId: 'hell_ambient',
          features: { rms:0, peak:0, zcr:0, duration:0, reward:0 }
        };
      }

      let absSum = 0, sqSum = 0, maxAmp = 0, zeroCross = 0;
      for (let i = 0; i < n; i++) {
        const v = data[i];
        const av = Math.abs(v);
        absSum += av;
        sqSum += v * v;
        if (av > maxAmp) maxAmp = av;
        if (i > 0 && ((data[i-1] > 0 && v <= 0) || (data[i-1] < 0 && v >= 0))) {
          zeroCross++;
        }
      }

      const duration = n / sampleRate;
      const rms = Math.sqrt(sqSum / n);
      const zcr = zeroCross / n * sampleRate;

      const yStar = { rms: 0.02, zcr: 500 };
      const er = Math.abs(rms - yStar.rms) / (yStar.rms + 1e-6);
      const ez = Math.abs(zcr - yStar.zcr) / (yStar.zcr + 1e-6);
      const e = er * 0.7 + ez * 0.3;
      const reward = Math.max(0, Math.min(1, 1 - e * 0.5));

      let styleId = 'hell_ambient';
      let styleName = '地狱氛围噪音';
      if (zcr > 2500 && rms > 0.08) {
        styleId = 'hell_metal';
        styleName = '地狱重金属';
      } else if (zcr > 1800 && rms > 0.05) {
        styleId = 'hell_rock';
        styleName = '地狱摇滚';
      } else if (zcr > 1200 && rms <= 0.05) {
        styleId = 'hell_jazz';
        styleName = '地狱爵士';
      } else if (zcr <= 1200 && rms > 0.06) {
        styleId = 'hell_rap';
        styleName = '地狱说唱节奏底';
      } else if (reward > 0.8) {
        styleId = 'hell_ambient';
        styleName = '地狱交响冥想';
      }

      let mood;
      if (reward > 0.8) {
        mood = '噪音接近静态场，我会用暗黑和弦与稀疏的碎片，让它变成“阴间交响”的序章。';
      } else if (reward > 0.5) {
        mood = '噪音在混乱与秩序之间，我会做多段结构，把它写成一首有故事的 ' + styleName + '。';
      } else if (reward > 0.2) {
        mood = '噪音偏躁动，我会在高潮段拉满鼓、贝斯和 Lead，把它推向高压地狱。';
      } else {
        mood = '噪音极度粗糙，我会彻底打碎时间线，用大量 glitch 粒子写成实验性的“地狱碎片乐章”。';
      }

      const tech =
        `（特征: len=${duration.toFixed(2)}s, rms=${rms.toFixed(3)}, ` +
        `peak=${maxAmp.toFixed(3)}, zcr≈${zcr.toFixed(0)}次/秒, Y*·r≈${reward.toFixed(2)}）`;

      const summary = `我把这段呼噜 / 怪声视为一个“地狱噪音场”，识别为「${styleName}」素材。${mood} ${tech}`;

      return {
        summary,
        styleId,
        features: { rms, peak:maxAmp, zcr, duration, reward }
      };
    }

    /********** PCM 合成作曲核心 **********/
    function composePCM(features, styleId, driverData, sampleRate) {
      const style = getHellStyle(styleId, features);
      const spb = 60 / style.bpm;
      const bars = style.bars;
      const beats = bars * 4;
      const totalSeconds = beats * spb;
      const maxSeconds = 12; // 防止太长
      const lengthSeconds = Math.min(totalSeconds, maxSeconds);
      const length = Math.floor(lengthSeconds * sampleRate);

      const out = new Float32Array(length);

      // 段落模板 + 张力曲线（“泛函优化”）
      const form = buildOptimizedForm(bars, features.reward, styleId);
      const tensionCurve = form.map(f => f.tension);

      // 背景噪音场（低电平）
      if (driverData && driverData.length > 0) {
        for (let i = 0; i < length; i++) {
          const v = driverData[i % driverData.length];
          out[i] += v * style.noiseLevel * 0.3;
        }
      }

      // 鼓 / Bass / Lead / Pad / Arp / Glitch
      addDrumsPCM(out, sampleRate, style, features, form);
      addBassPCM(out, sampleRate, style, features, form);
      addLeadPCM(out, sampleRate, style, features, form);
      addPadPCM(out, sampleRate, style, features, form);
      addArpPCM(out, sampleRate, style, features, form);
      addGlitchPCM(out, sampleRate, style, features, form, driverData);

      return normalizeFloat(out);
    }

    function getHellStyle(styleId, f) {
      const base = {
        noiseLevel: 0.25 + 0.35 * (1 - f.reward),
        bars: f.duration > 8 ? 10 : (f.duration > 4 ? 8 : 6)
      };
      switch (styleId) {
        case 'hell_metal':
          return { ...base, styleId, bpm: 150, scaleRoot: 42, drumEnergy: 1.3, leadDensity: 0.9 };
        case 'hell_rock':
          return { ...base, styleId, bpm: 130, scaleRoot: 45, drumEnergy: 1.0, leadDensity: 0.8 };
        case 'hell_jazz':
          return { ...base, styleId, bpm: 115, scaleRoot: 48, drumEnergy: 0.8, leadDensity: 0.9 };
        case 'hell_rap':
          return { ...base, styleId, bpm: 90,  scaleRoot: 40, drumEnergy: 0.9, leadDensity: 0.6 };
        case 'hell_ambient':
        default:
          return { ...base, styleId:'hell_ambient', bpm: 75, scaleRoot: 52, drumEnergy: 0.5, leadDensity: 0.4 };
      }
    }

    // “泛函优化”的张力曲线（简化版，多模板中选最优）
    function buildOptimizedForm(bars, reward, styleId) {
      const templates = [
        ['intro','groove','groove','break','climax','climax','outro'],
        ['intro','groove','groove','break','groove','climax','break','climax','outro'],
        ['intro','intro','groove','climax','climax','groove','outro']
      ];

      let bestForm = null;
      let bestScore = -Infinity;

      templates.forEach((tpl) => {
        const form = [];
        for (let b = 0; b < bars; b++) {
          const x = b / Math.max(1, bars - 1);
          const tIdx = Math.min(tpl.length - 1, Math.floor(x * tpl.length));
          const section = tpl[tIdx];

          let base = 0.3 + (1 - reward) * 0.4;
          let bump = 0;
          if (section === 'intro')  bump = 0.1 * x;
          if (section === 'groove') bump = 0.25;
          if (section === 'break')  bump = 0.15;
          if (section === 'climax') bump = 0.5;
          if (section === 'outro')  bump = 0.15 * (1 - x);

          let tension = base + bump;
          tension += (Math.random() - 0.5) * 0.12;
          tension = Math.max(0, Math.min(1, tension));
          form.push({ section, tension });
        }

        const ts = form.map(f => f.tension);
        const meanT = ts.reduce((a,b)=>a+b,0) / ts.length;
        const varT = ts.reduce((a,b)=>a+Math.pow(b-meanT,2),0) / ts.length;
        const maxT = Math.max(...ts);
        const climaxIndex = ts.indexOf(maxT);
        const climaxPos = climaxIndex / Math.max(1, ts.length-1);
        const posScore = 1 - Math.abs(climaxPos - 0.7);

        let styleScore = 0;
        if (styleId === 'hell_jazz') {
          styleScore = -Math.abs(meanT - 0.6) + 0.8 * varT;
        } else if (styleId === 'hell_metal' || styleId === 'hell_rock') {
          styleScore = -Math.abs(meanT - 0.75) + 0.5 * varT;
        } else {
          styleScore = -Math.abs(meanT - 0.5) + 0.3 * varT;
        }

        const J = 1.2 * varT + 0.8 * maxT + 0.8 * posScore + styleScore;
        if (J > bestScore) {
          bestScore = J;
          bestForm = form;
        }
      });

      return bestForm;
    }

    /********** PCM 层：鼓 / Bass / Lead / Pad / Arp / Glitch **********/
    function timeToIndex(t, sr) {
      return Math.floor(t * sr);
    }

    function addSine(out, sr, tStart, dur, freq, amp, decay = 3.0) {
      const start = timeToIndex(tStart, sr);
      const len = Math.min(out.length - start, Math.floor(dur * sr));
      const w = 2 * Math.PI * freq / sr;
      for (let i = 0; i < len; i++) {
        const env = Math.exp(-decay * i / len);
        out[start + i] += amp * env * Math.sin(w * i);
      }
    }

    function addNoiseHit(out, sr, tStart, dur, amp, highpass=false) {
      const start = timeToIndex(tStart, sr);
      const len = Math.min(out.length - start, Math.floor(dur * sr));
      for (let i = 0; i < len; i++) {
        let n = (Math.random() * 2 - 1);
        if (highpass) {
          // 简单高通：减去邻居平均
          const prev = (i>0)? (Math.random()*2-1) : 0;
          n = n - prev * 0.5;
        }
        const env = Math.exp(-6 * i / len);
        out[start + i] += amp * env * n;
      }
    }

    function addDrumsPCM(out, sr, style, f, form) {
      const spb = 60 / style.bpm;
      const beats = style.bars * 4;
      const isJazz = style.styleId === 'hell_jazz';

      for (let b = 0; b < beats; b++) {
        const barIdx = Math.floor(b / 4);
        const { section, tension: T } = form[barIdx];
        const tBase = b * spb;
        const beatInBar = b % 4;

        const sectionScale =
          section === 'intro' ? 0.4 :
          section === 'outro' ? 0.6 :
          section === 'break' ? 0.5 :
          1.0;

        const energyScale =
          style.drumEnergy * sectionScale * (0.6 + 0.8 * f.rms / 0.1) * (0.7 + 0.6 * T);

        // Kick
        if (section !== 'break') {
          if (beatInBar === 0 || beatInBar === 2 ||
              (T > 0.6 && Math.random() < 0.6)) {
            addSine(out, sr, tBase, 0.25, 55 + 25*T, 0.5 * energyScale, 4.0);
          }
        } else if (Math.random() < 0.4) {
          addSine(out, sr, tBase, 0.2, 45 + 15*T, 0.3 * energyScale, 4.0);
        }

        // Snare
        if (beatInBar === 1 || beatInBar === 3) {
          if (section !== 'intro') {
            let t = tBase;
            if (isJazz) {
              const swing = 0.18;
              t += spb * swing;
            }
            addNoiseHit(out, sr, t, 0.18, 0.35 * energyScale, true);
            if (T > 0.7 && Math.random() < 0.5) {
              addNoiseHit(out, sr, t - spb*0.12, 0.12, 0.15*energyScale, true);
            }
          }
        }

        // Hat / ride
        const hatBase = f.zcr > 2000 ? 2 : 1;
        const hatDensity =
          section === 'intro' ? 1 :
          section === 'break' ? 1 + (T > 0.6 ? 1 : 0) :
          section === 'outro' ? hatBase :
          hatBase + (T > 0.6 ? 1 : 0);

        for (let i = 0; i < hatDensity; i++) {
          let hatT = tBase + spb * (i/hatDensity + 0.5/(hatDensity+1));
          if (isJazz) {
            const pos = (hatT - tBase) / spb;
            if (pos > 0.5) hatT += spb * 0.15 * (0.5 + 0.8*T);
          }
          addNoiseHit(out, sr, hatT, 0.08, 0.12 * energyScale, true);
        }
      }
    }

    function addBassPCM(out, sr, style, f, form) {
      const spb = 60 / style.bpm;
      const beats = style.bars * 4;
      const scale = makeMinorScale(style.scaleRoot);
      const baseStrength = 0.18 + 0.4 * f.rms / 0.12;
      const isJazz = style.styleId === 'hell_jazz';

      for (let b = 0; b < beats; b++) {
        const barIdx = Math.floor(b / 4);
        const { section, tension: T } = form[barIdx];

        if (section === 'intro' && Math.random() < 0.5) continue;
        const t = b * spb;

        let idx;
        if (isJazz) {
          const direction = (Math.random() < 0.5) ? 1 : -1;
          const step = (Math.random() < 0.2) ? 3 : 1;
          const baseIdx = (b + (direction>0?0:scale.length)) % scale.length;
          idx = (baseIdx + direction * step + scale.length) % scale.length;
          if (Math.random() < 0.1) idx = (idx + 3) % scale.length; // tritone
        } else {
          const chordPattern = [0, 5, 3, 4];
          const chordIdx = chordPattern[barIdx % chordPattern.length];
          const leap = T > 0.65 && Math.random() < 0.6 ? (Math.random() < 0.5 ? -2 : 2) : 0;
          idx = (chordIdx + leap + scale.length) % scale.length;
        }

        const midi = scale[idx];
        const freq = midiToFreq(midi - 12);
        const dur = isJazz ? spb*0.9 : spb*(1.4+0.6*T);
        const amp = baseStrength * (0.7 + 0.9*T);

        // 用少量正弦叠加 saw-ish 感
        addSine(out, sr, t, dur, freq, amp, 3.0);
        addSine(out, sr, t, dur, freq*2, amp*0.3, 3.5);
      }
    }

    function addLeadPCM(out, sr, style, f, form) {
      const spb = 60 / style.bpm;
      const beats = style.bars * 4;
      const scale = makeMinorScale(style.scaleRoot + 12);
      const densityBase = style.leadDensity * (0.3 + 0.7 * f.zcr / 3000);
      const isJazz = style.styleId === 'hell_jazz';

      for (let b = 0; b < beats; b++) {
        const barIdx = Math.floor(b / 4);
        const { section, tension: T } = form[barIdx];

        let density =
          section === 'intro' ? densityBase * 0.3 :
          section === 'break' ? densityBase * 0.6 :
          section === 'outro' ? densityBase * 0.4 :
          densityBase * (0.6 + 0.9 * T);

        if (isJazz && (section === 'groove' || section === 'climax')) {
          density *= 1.3;
        }

        if (Math.random() > density) continue;

        const offsetBeat = isJazz && section !== 'intro'
          ? (Math.random() < 0.7 ? 0.33 : Math.random() * 0.5)
          : Math.random() * 0.5;

        const t = b * spb + spb * offsetBeat;

        const baseMidi = scale[Math.floor(Math.random() * scale.length)];
        let midi = baseMidi;
        if (section === 'climax' || (section === 'groove' && T > 0.6)) {
          midi += (Math.random() < 0.5 ? -1 : 1); // 半音擦音
        }
        if (isJazz && Math.random() < 0.4) {
          midi += (Math.random() < 0.5 ? 9 : -6); // 大跳
        }

        const freq = midiToFreq(midi);
        const dur = (section === 'break')
          ? spb * (0.15 + 0.25*T)
          : spb * (0.18 + 0.4*(0.5+T));
        const amp = 0.07 + 0.12*(1-f.reward)*(0.4+T);

        // 简单方波 lead
        addSine(out, sr, t, dur, freq, amp, 4.0);
      }
    }

    function addPadPCM(out, sr, style, f, form) {
      const spb = 60 / style.bpm;
      const bars = style.bars;
      const scale = makeMinorScale(style.scaleRoot + (style.styleId === 'hell_jazz' ? 7 : 5));
      const baseLevel = 0.08 + 0.12 * f.reward;

      const chordPatternA = [0, 5, 3, 4];
      const chordPatternB = [2, 4, 6, 1];
      const isJazz = style.styleId === 'hell_jazz';

      for (let bar = 0; bar < bars; bar++) {
        const { section, tension: T } = form[bar];
        if (section === 'break' && Math.random() < 0.6) continue;

        const barStart = bar*4*spb;
        const usePattern = (section === 'climax' || (isJazz && section==='break'))
          ? chordPatternB : chordPatternA;
        const chordDegree = usePattern[bar % usePattern.length];
        const rootIdx = chordDegree % scale.length;

        const chord = [
          scale[rootIdx],
          scale[(rootIdx+2)%scale.length],
          scale[(rootIdx+4)%scale.length]
        ];
        if (T>0.5) chord.push(scale[(rootIdx+6)%scale.length]);
        if (isJazz && T>0.6) chord.push(scale[(rootIdx+1)%scale.length]);

        const level =
          baseLevel *
          (section==='intro'||section==='outro'?0.8:1.0) *
          (0.5+0.8*(1-f.reward)) *
          (0.7+0.6*T);

        chord.forEach((midi,i)=>{
          const freq = midiToFreq(midi);
          const sustain = (section==='break'
            ? spb*(1.0+0.4*(1-T))
            : 4*spb*(0.9+0.4*T));
          const start = barStart + i*0.02;
          addSine(out, sr, start, sustain, freq, level, 2.0);
        });
      }
    }

    function addArpPCM(out, sr, style, f, form) {
      const spb = 60 / style.bpm;
      const bars = style.bars;
      const scale = makeMinorScale(style.scaleRoot + 12);
      const baseDensity = 0.4 + 0.6*(1-f.reward);
      const isJazz = style.styleId === 'hell_jazz';

      for (let bar=0; bar<bars; bar++) {
        const { section, tension:T } = form[bar];
        const barStart = bar*4*spb;
        let density =
          section==='intro'? baseDensity*0.3 :
          section==='break'? baseDensity*0.5 :
          section==='outro'? baseDensity*0.4 :
          baseDensity*(0.6+0.7*T);

        if (isJazz && (section==='groove'||section==='climax')) density *= 1.3;

        const stepsPerBar = Math.round(8+16*T);
        for (let i=0;i<stepsPerBar;i++){
          if (Math.random()>density) continue;
          const pos = (i/stepsPerBar)*4*spb;
          let t = barStart + pos;

          if (isJazz) {
            const posBeat = (pos/spb)%1;
            if (posBeat>0.5) t += spb*0.12*(0.5+0.8*T);
          }

          let midi = scale[Math.floor(Math.random()*scale.length)];
          if (isJazz && Math.random()<0.4) midi += (Math.random()<0.5?1:-1);
          if (T>0.7 && Math.random()<0.5) midi += (Math.random()<0.5?12:-12);

          const freq = midiToFreq(midi);
          const dur = spb*(0.07+0.18*(0.3+T));
          const amp = 0.04+0.08*(1-f.reward)*(0.4+T);
          addSine(out, sr, t, dur, freq, amp, 5.0);
        }
      }
    }

    function addGlitchPCM(out, sr, style, f, form, driverData) {
      if (!driverData || driverData.length===0) return;
      const spb = 60 / style.bpm;
      const bars = style.bars;

      const minGrain = Math.floor(sr*0.03);
      const maxGrain = Math.floor(sr*0.09);

      const grains = [];
      let ptr=0;
      while(ptr+minGrain<driverData.length && grains.length<300){
        const gLen = minGrain + Math.floor(Math.random()*(maxGrain-minGrain));
        const g = new Float32Array(gLen);
        g.set(driverData.subarray(ptr,ptr+gLen));
        grains.push(g);
        ptr += gLen;
      }
      if (grains.length===0) return;

      for (let bar=0; bar<bars; bar++){
        const {section, tension:T} = form[bar];
        const barStart = bar*4*spb;

        const baseEvents =
          section==='intro'?2:
          section==='break'?4:
          section==='outro'?3:5;
        const events = Math.round(baseEvents+10*T*(1+(section==='climax'?0.7:0)));

        for(let i=0;i<events;i++){
          const grain = grains[Math.floor(Math.random()*grains.length)];
          const localPos = Math.random()*4*spb;
          const t = barStart+localPos;
          const destStart = timeToIndex(t,sr);
          const rate = 0.6+1.4*Math.random();

          const step = Math.max(1,Math.floor(rate));
          const len = grain.length;
          for(let j=0;j<len;j+=step){
            const idx=destStart+Math.floor(j/step);
            if(idx>=out.length) break;
            const env=Math.exp(-4*j/len);
            out[idx]+=grain[j]*env*(0.04+0.11*T*(1-f.reward));
          }
        }
      }
    }

    /********** 工具函数 **********/
    function makeMinorScale(rootMidi) {
      const intervals = [0,2,3,5,7,8,10];
      return intervals.map(i=>rootMidi+i);
    }

    function midiToFreq(midi) {
      return 440*Math.pow(2,(midi-69)/12);
    }

    function normalizeFloat(samples) {
      let max=0;
      for(let i=0;i<samples.length;i++){
        const v=Math.abs(samples[i]);
        if(v>max)max=v;
      }
      const target=0.9;
      const gain = max>0 && max<target ? (target/max):1.0;
      if(gain===1.0)return samples;
      const out=new Float32Array(samples.length);
      for(let i=0;i<samples.length;i++)out[i]=samples[i]*gain;
      return out;
    }

    function makeDistortionCurve(amount) {
      const k=typeof amount==='number'?amount:50;
      const n=44100;
      const curve=new Float32Array(n);
      const deg=Math.PI/180;
      for(let i=0;i<n;++i){
        const x=i*2/n-1;
        curve[i]=(3+k)*x*20*deg/(Math.PI+k*Math.abs(x));
      }
      return curve;
    }

    function encodeWAV(samples, sampleRate) {
      const buffer=new ArrayBuffer(44+samples.length*2);
      const view=new DataView(buffer);

      function writeString(offset,str){
        for(let i=0;i<str.length;i++)view.setUint8(offset+i,str.charCodeAt(i));
      }

      const numChannels=1;
      const bytesPerSample=2;
      const blockAlign=numChannels*bytesPerSample;
      const byteRate=sampleRate*blockAlign;

      writeString(0,'RIFF');
      view.setUint32(4,36+samples.length*2,true);
      writeString(8,'WAVE');

      writeString(12,'fmt ');
      view.setUint32(16,16,true);
      view.setUint16(20,1,true);
      view.setUint16(22,numChannels,true);
      view.setUint32(24,sampleRate,true);
      view.setUint32(28,byteRate,true);
      view.setUint16(32,blockAlign,true);
      view.setUint16(34,16,true);

      writeString(36,'data');
      view.setUint32(40,samples.length*2,true);

      let offset=44;
      for(let i=0;i<samples.length;i++,offset+=2){
        let s=samples[i];
        s=Math.max(-1,Math.min(1,s));
        view.setInt16(offset,s<0?s*0x8000:s*0x7fff,true);
      }
      return new Blob([view],{type:'audio/wav'});
    }

    function startTimer() {
      recordingStartTime=performance.now();
      clearInterval(timerInterval);
      timerInterval=setInterval(()=>{
        const elapsed=(performance.now()-recordingStartTime)/1000;
        const m=Math.floor(elapsed/60);
        const s=Math.floor(elapsed%60);
        elTimer.textContent=`${String(m).padStart(2,'0')}:${String(s).padStart(2,'0')}`;
      },200);
    }

    function stopTimer() {
      clearInterval(timerInterval);
      timerInterval=null;
    }

    function saveGenerated() {
      if(!batState.generatedBlob)return;
      const url=URL.createObjectURL(batState.generatedBlob);
      const a=document.createElement('a');
      const ts=new Date().toISOString().replace(/[:.]/g,'-');
      a.href=url;
      a.download=`bat-hell-music-${ts}.wav`;
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(url);
    }

    async function sharePage() {
      if(!batState.generatedBlob)return;
      const shareData={
        title:'Bat-music 实验',
        text:'我刚刚用呼噜 / 怪声在 Bat-music 里炼出了一段地狱乐曲。',
        url:window.location.href
      };
      if(navigator.share){
        try{ await navigator.share(shareData); }
        catch(err){ logDebug('分享失败或被取消: '+err.message); }
      }else{
        alert('当前浏览器不支持系统分享，请手动复制地址栏链接。');
      }
    }

    function showWxMask() {
      const mask=document.getElementById('wxMask');
      if(mask)mask.classList.add('show');
    }

    function hideWxMask() {
      const mask=document.getElementById('wxMask');
      if(mask)mask.classList.remove('show');
    }

    function logDebug(msg) {
      if(!debugConsole)return;
      const time=new Date().toISOString().split('T')[1].split('.')[0];
      debugConsole.textContent+=`[${time}] ${msg}\n`;
      debugConsole.scrollTop=debugConsole.scrollHeight;
    }
  </script>
</body>
</html>
