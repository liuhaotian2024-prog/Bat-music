<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>Bat-music · 从呼噜到地狱作曲</title>
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

    /* 顶部调试条：默认打开，方便排错与观察 */
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
      padding-top: 80px; /* 给调试条留空间 */
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
          elStatusText.textContent = '根据噪音场与乐音场的张力，在地狱熔炉中作曲…';
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

    /********** 录音管线（已验证可用） **********/
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

    /********** 地狱作曲：噪音场 + 乐音场 + 时间线粉碎 **********/
    async function generateFromRecording(floatData, sampleRate) {
      try {
        const analysis = analyzeAndDescribe(floatData, sampleRate);
        elSummary.textContent = analysis.summary;

        const rendered = await renderHellComposition(
          analysis.features,
          analysis.styleId,
          floatData,
          sampleRate
        );

        batState.floatData = rendered.processedData;
        batState.generatedBlob = rendered.blob;

        const url = URL.createObjectURL(rendered.blob);
        elAudioPlayer.src = url;
        elPlayerLayer.classList.add('show');

        drawCurrentWave();
        logDebug('作曲样本数=' + rendered.processedData.length +
                 ', wavSize=' + rendered.blob.size + ' 字节');

        setPhase('generated');
      } catch (err) {
        logDebug('generateFromRecording 失败，fallback 原始录音: ' + err.message);
        const blob = encodeWAV(floatData, sampleRate);
        batState.generatedBlob = blob;
        const url = URL.createObjectURL(blob);
        elAudioPlayer.src = url;
        elPlayerLayer.classList.add('show');
        setPhase('generated');
      }
    }

    // 噪音场分析 + Y* 张力（reward）→ 风格 + 特征
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

      // Y*：理想助眠场；reward 越低，说明“地狱噪音成分”越重
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
        mood = '噪音接近静态场，我会用大跨度但稀疏的和弦，让它变成“阴间交响”的底色。';
      } else if (reward > 0.5) {
        mood = '噪音在混乱与秩序之间，我会用适中密度的鼓和贝斯，让乐曲像一场危险的梦境。';
      } else if (reward > 0.2) {
        mood = '噪音偏躁动，我会把节奏和 Lead 推向高张力区，形成攻击性很强的 ' + styleName + '。';
      } else {
        mood = '噪音极度粗糙，我会强行把时间线粉碎重组，让它变成一段实验性“地狱碎片乐章”。';
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

    /********** 乐音场作曲 + 时间线粉碎 **********/
    function renderHellComposition(features, styleId, driverData, sampleRate) {
      return new Promise((resolve, reject) => {
        try {
          const style = getHellStyle(styleId, features);
          const spb = 60 / style.bpm;
          const bars = style.bars;
          const beats = bars * 4;
          const totalSeconds = beats * spb;
          const length = Math.floor(totalSeconds * sampleRate);

          const offline = new OfflineAudioContext(1, length, sampleRate);
          const master = offline.createGain();
          master.gain.value = style.masterGain;
          master.connect(offline.destination);

          // 根据 Y* reward 构造每小节的张力曲线 T(b)
          const tensionCurve = buildTensionCurve(bars, features.reward);

          // 1. 噪音纹理：长背景层
          if (driverData && driverData.length > 0) {
            const driverBuffer = offline.createBuffer(1, length, sampleRate);
            const dest = driverBuffer.getChannelData(0);
            for (let i = 0; i < length; i++) {
              dest[i] = driverData[i % driverData.length];
            }
            const src = offline.createBufferSource();
            src.buffer = driverBuffer;

            const shaper = offline.createWaveShaper();
            shaper.curve = makeDistortionCurve(style.noiseDist);
            shaper.oversample = '4x';

            const filter = offline.createBiquadFilter();
            filter.type = style.noiseFilterType;
            filter.frequency.value = style.noiseFreq;
            filter.Q.value = style.noiseQ;

            const g = offline.createGain();
            g.gain.value = style.noiseLevel;

            src.connect(shaper);
            shaper.connect(filter);
            filter.connect(g);
            g.connect(master);

            src.start(0);
          }

          // 2. 鼓 / 贝斯 / Lead （受每小节 T(b) 调制）
          scheduleDrums(offline, master, style, features, tensionCurve);
          scheduleBass(offline, master, style, features, tensionCurve);
          scheduleLead(offline, master, style, features, tensionCurve);

          // 3. 时间线粉碎的 glitch 层：从原始噪音中抓 grain 重排
          scheduleGlitchFromDriver(
            offline, master, style, features, driverData, sampleRate, tensionCurve
          );

          offline.startRendering().then(buffer => {
            const out = new Float32Array(buffer.length);
            buffer.copyFromChannel(out, 0, 0);
            const normalized = normalizeFloat(out);
            const blob = encodeWAV(normalized, sampleRate);
            resolve({ processedData: normalized, blob });
          }).catch(reject);
        } catch (err) {
          reject(err);
        }
      });
    }

    function buildTensionCurve(bars, reward) {
      // 这里可以看成一个简单的泛函：我们希望张力曲线从低 → 高 → 释放
      // reward 越低，整体越“地狱”，基线张力越高
      const base = 0.3 + (1 - reward) * 0.5; // 0.3~0.8
      const curve = [];
      for (let b = 0; b < bars; b++) {
        const phase = Math.sin(Math.PI * b / Math.max(1, bars - 1));
        let t = base * (0.7 + 0.6 * phase); // 中间段张力更高
        t += (Math.random() - 0.5) * 0.15;  // 小扰动
        t = Math.max(0, Math.min(1, t));
        curve.push(t);
      }
      return curve;
    }

    function getHellStyle(styleId, f) {
      const base = {
        masterGain: 0.9,
        noiseLevel: 0.28 + 0.35 * (1 - f.reward),
        noiseDist: 80 + 220 * (1 - f.reward),
        noiseFilterType: 'bandpass',
        noiseFreq: 1500,
        noiseQ: 1.4,
        bars: f.duration > 6 ? 8 : 4
      };
      switch (styleId) {
        case 'hell_metal':
          return { ...base, bpm: 150 + 40 * f.reward, scaleRoot: 42, dist: 250, drumEnergy: 1.3, leadDensity: 0.9 };
        case 'hell_rock':
          return { ...base, bpm: 130 + 20 * f.reward, scaleRoot: 45, dist: 180, drumEnergy: 1.0, leadDensity: 0.75 };
        case 'hell_jazz':
          return { ...base, bpm: 115 + 15 * f.reward, scaleRoot: 48, dist: 110, drumEnergy: 0.75, leadDensity: 0.7 };
        case 'hell_rap':
          return { ...base, bpm: 90  + 15 * (1 - f.reward), scaleRoot: 40, dist: 140, drumEnergy: 0.9, leadDensity: 0.55 };
        case 'hell_ambient':
        default:
          return { ...base, bpm: 70  + 25 * f.reward, scaleRoot: 52, dist: 80,  drumEnergy: 0.45, leadDensity: 0.4 };
      }
    }

    /********** 鼓 / Bass / Lead：受张力曲线调制 **********/
    function scheduleDrums(ctx, master, style, f, tensionCurve) {
      const spb = 60 / style.bpm;
      const beats = style.bars * 4;

      for (let b = 0; b < beats; b++) {
        const barIdx = Math.floor(b / 4);
        const T = tensionCurve[barIdx];
        const energyScale =
          style.drumEnergy * (0.6 + 0.8 * f.rms / 0.1) * (0.7 + 0.6 * T);

        const t = b * spb;
        const beatInBar = b % 4;

        // Kick：张力高时 ghost note 更多
        if (beatInBar === 0 || beatInBar === 2 ||
            (T > 0.6 && Math.random() < 0.5)) {
          scheduleKick(ctx, master, t, 55 + T * 20, 0.5 * energyScale);
        }

        // Snare：2/4 强拍不变，张力高时加提前/延后 ghost
        if (beatInBar === 1 || beatInBar === 3) {
          scheduleSnare(ctx, master, t, 0.35 * energyScale);
          if (T > 0.7 && Math.random() < 0.4) {
            scheduleSnare(ctx, master, t - spb * 0.12, 0.15 * energyScale);
          }
        }

        // Hat：张力越高越密
        const hatBase = f.zcr > 2000 ? 2 : 1;
        const hatDensity = hatBase + (T > 0.6 ? 1 : 0);
        for (let i = 0; i < hatDensity; i++) {
          const hatT = t + spb * (i / hatDensity + 0.5 / (hatDensity + 1));
          scheduleHat(ctx, master, hatT, 0.12 * energyScale);
        }
      }
    }

    function scheduleKick(ctx, master, time, baseFreq, gainLevel) {
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();

      osc.type = 'sine';
      osc.frequency.setValueAtTime(baseFreq, time);
      osc.frequency.exponentialRampToValueAtTime(baseFreq * 0.25, time + 0.12);

      gain.gain.setValueAtTime(gainLevel, time);
      gain.gain.exponentialRampToValueAtTime(0.001, time + 0.25);

      osc.connect(gain);
      gain.connect(master);
      osc.start(time);
      osc.stop(time + 0.3);
    }

    function scheduleSnare(ctx, master, time, gainLevel) {
      const noiseBuffer = ctx.createBuffer(1, ctx.sampleRate * 0.2, ctx.sampleRate);
      const data = noiseBuffer.getChannelData(0);
      for (let i = 0; i < data.length; i++) data[i] = (Math.random() * 2 - 1);
      const noise = ctx.createBufferSource();
      noise.buffer = noiseBuffer;

      const filter = ctx.createBiquadFilter();
      filter.type = 'highpass';
      filter.frequency.value = 1900;

      const gain = ctx.createGain();
      gain.gain.setValueAtTime(gainLevel, time);
      gain.gain.exponentialRampToValueAtTime(0.001, time + 0.18);

      noise.connect(filter);
      filter.connect(gain);
      gain.connect(master);

      noise.start(time);
      noise.stop(time + 0.2);
    }

    function scheduleHat(ctx, master, time, gainLevel) {
      const noiseBuffer = ctx.createBuffer(1, ctx.sampleRate * 0.1, ctx.sampleRate);
      const data = noiseBuffer.getChannelData(0);
      for (let i = 0; i < data.length; i++) data[i] = (Math.random() * 2 - 1);
      const noise = ctx.createBufferSource();
      noise.buffer = noiseBuffer;

      const filter = ctx.createBiquadFilter();
      filter.type = 'highpass';
      filter.frequency.value = 6500;

      const gain = ctx.createGain();
      gain.gain.setValueAtTime(gainLevel, time);
      gain.gain.exponentialRampToValueAtTime(0.001, time + 0.08);

      noise.connect(filter);
      filter.connect(gain);
      gain.connect(master);

      noise.start(time);
      noise.stop(time + 0.1);
    }

    function scheduleBass(ctx, master, style, f, tensionCurve) {
      const spb = 60 / style.bpm;
      const beats = style.bars * 4;
      const scale = makeMinorScale(style.scaleRoot);
      const baseStrength = 0.18 + 0.4 * f.rms / 0.12;

      for (let b = 0; b < beats; b += 2) {
        const barIdx = Math.floor(b / 4);
        const T = tensionCurve[barIdx];
        const t = b * spb;

        const idxBase = (b / 2) % scale.length;
        const leap = T > 0.6 && Math.random() < 0.5 ? (Math.random() < 0.5 ? -2 : 2) : 0;
        const idx = (idxBase + leap + scale.length) % scale.length;

        const midi = scale[idx];
        const freq = midiToFreq(midi - 12); // 降一八度

        const osc = ctx.createOscillator();
        const gain = ctx.createGain();

        osc.type = 'sawtooth';
        osc.frequency.setValueAtTime(freq, t);

        const dur = spb * (1.4 + 0.5 * T);
        const strength = baseStrength * (0.7 + 0.8 * T);

        gain.gain.setValueAtTime(strength, t);
        gain.gain.exponentialRampToValueAtTime(0.001, t + dur);

        osc.connect(gain);
        gain.connect(master);

        osc.start(t);
        osc.stop(t + dur + 0.05);
      }
    }

    function scheduleLead(ctx, master, style, f, tensionCurve) {
      const spb = 60 / style.bpm;
      const beats = style.bars * 4;
      const baseScale = makeMinorScale(style.scaleRoot + 12);
      const densityBase = style.leadDensity * (0.3 + 0.7 * f.zcr / 3000);

      // 额外的“擦音”半音阶，用于地狱爵士感
      function pickPitchWithTension(scale, T) {
        const baseMidi = scale[Math.floor(Math.random() * scale.length)];
        if (Math.random() < T) {
          const offset = (Math.random() < 0.5 ? -1 : 1); // 半音偏移
          return baseMidi + offset;
        }
        return baseMidi;
      }

      for (let b = 0; b < beats; b++) {
        const barIdx = Math.floor(b / 4);
        const T = tensionCurve[barIdx];
        const density = densityBase * (0.3 + 0.9 * T);

        if (Math.random() > density) continue;

        const t = b * spb + spb * Math.random() * 0.5;
        const midi = pickPitchWithTension(baseScale, T);
        const freq = midiToFreq(midi);

        const osc = ctx.createOscillator();
        const gain = ctx.createGain();

        osc.type = (T > 0.6 ? 'sawtooth' : 'square');
        osc.frequency.setValueAtTime(freq, t);
        if (T > 0.7) {
          // 高张力时做一点滑音
          osc.frequency.linearRampToValueAtTime(freq * (1 + (Math.random() - 0.5) * 0.2),
                                                t + spb * 0.25);
        }

        const dur = spb * (0.18 + 0.4 * (0.5 + T));
        const level = 0.07 + 0.12 * (1 - f.reward) * (0.5 + T);

        gain.gain.setValueAtTime(level, t);
        gain.gain.exponentialRampToValueAtTime(0.001, t + dur);

        const shaper = ctx.createWaveShaper();
        shaper.curve = makeDistortionCurve(style.dist);
        shaper.oversample = '4x';

        osc.connect(shaper);
        shaper.connect(gain);
        gain.connect(master);

        osc.start(t);
        osc.stop(t + dur + 0.05);
      }
    }

    /********** 时间线粉碎：从原始噪音中取 grain 做 glitch **********/
    function scheduleGlitchFromDriver(ctx, master, style, f, driverData, sampleRate, tensionCurve) {
      if (!driverData || driverData.length === 0) return;

      const spb = 60 / style.bpm;
      const bars = style.bars;

      // 粒度 40–80ms 之间
      const minGrain = Math.floor(sampleRate * 0.04);
      const maxGrain = Math.floor(sampleRate * 0.08);

      // 预切一堆 grain
      const grains = [];
      let ptr = 0;
      while (ptr + minGrain < driverData.length && grains.length < 256) {
        const gLen = minGrain + Math.floor(Math.random() * (maxGrain - minGrain));
        const g = new Float32Array(gLen);
        g.set(driverData.subarray(ptr, ptr + gLen));
        grains.push(g);
        ptr += gLen;
      }
      if (grains.length === 0) return;

      for (let bar = 0; bar < bars; bar++) {
        const T = tensionCurve[bar];
        const barStart = bar * 4 * spb;

        // 张力越高，grains 越多
        const events = Math.round(2 + 8 * T);
        for (let i = 0; i < events; i++) {
          const grain = grains[Math.floor(Math.random() * grains.length)];
          const buf = ctx.createBuffer(1, grain.length, sampleRate);
          buf.getChannelData(0).set(grain);

          const src = ctx.createBufferSource();
          src.buffer = buf;

          // 时间线粉碎：随机时间 + 随机回放速度（0.6~1.8）
          const localPos = Math.random() * 4 * spb;
          const t = barStart + localPos;
          src.playbackRate.value = 0.6 + 1.2 * Math.random();

          // 滤波：高张力时更尖锐
          const filter = ctx.createBiquadFilter();
          filter.type = (T > 0.6 ? 'highpass' : 'bandpass');
          filter.frequency.value = 1500 + 3500 * T;
          filter.Q.value = 0.7 + 1.2 * T;

          const g = ctx.createGain();
          g.gain.value = 0.06 + 0.12 * T * (1 - f.reward);

          src.connect(filter);
          filter.connect(g);
          g.connect(master);

          src.start(t);
          src.stop(t + grain.length / sampleRate / src.playbackRate.value + 0.02);
        }
      }
    }

    /********** 工具函数：音阶 / WAV / 归一化 / 调试 **********/
    function makeMinorScale(rootMidi) {
      const intervals = [0, 2, 3, 5, 7, 8, 10];
      return intervals.map(i => rootMidi + i);
    }

    function midiToFreq(midi) {
      return 440 * Math.pow(2, (midi - 69) / 12);
    }

    function normalizeFloat(samples) {
      let max = 0;
      for (let i = 0; i < samples.length; i++) {
        const v = Math.abs(samples[i]);
        if (v > max) max = v;
      }
      const target = 0.9;
      const gain = max > 0 && max < target ? (target / max) : 1.0;
      if (gain === 1.0) return samples;
      const out = new Float32Array(samples.length);
      for (let i = 0; i < samples.length; i++) out[i] = samples[i] * gain;
      return out;
    }

    function makeDistortionCurve(amount) {
      const k = typeof amount === 'number' ? amount : 50;
      const n = 44100;
      const curve = new Float32Array(n);
      const deg = Math.PI / 180;
      for (let i = 0; i < n; ++i) {
        const x = i * 2 / n - 1;
        curve[i] = (3 + k) * x * 20 * deg / (Math.PI + k * Math.abs(x));
      }
      return curve;
    }

    function encodeWAV(samples, sampleRate) {
      const buffer = new ArrayBuffer(44 + samples.length * 2);
      const view = new DataView(buffer);

      function writeString(offset, str) {
        for (let i = 0; i < str.length; i++) view.setUint8(offset + i, str.charCodeAt(i));
      }

      const numChannels = 1;
      const bytesPerSample = 2;
      const blockAlign = numChannels * bytesPerSample;
      const byteRate = sampleRate * blockAlign;

      writeString(0, 'RIFF');
      view.setUint32(4, 36 + samples.length * 2, true);
      writeString(8, 'WAVE');

      writeString(12, 'fmt ');
      view.setUint32(16, 16, true);
      view.setUint16(20, 1, true);
      view.setUint16(22, numChannels, true);
      view.setUint32(24, sampleRate, true);
      view.setUint32(28, byteRate, true);
      view.setUint16(32, blockAlign, true);
      view.setUint16(34, 16, true);

      writeString(36, 'data');
      view.setUint32(40, samples.length * 2, true);

      let offset = 44;
      for (let i = 0; i < samples.length; i++, offset += 2) {
        let s = samples[i];
        s = Math.max(-1, Math.min(1, s));
        view.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7fff, true);
      }

      return new Blob([view], { type: 'audio/wav' });
    }

    /********** 计时 / 保存 / 分享 / 调试 **********/
    function startTimer() {
      recordingStartTime = performance.now();
      clearInterval(timerInterval);
      timerInterval = setInterval(() => {
        const elapsed = (performance.now() - recordingStartTime) / 1000;
        const m = Math.floor(elapsed / 60);
        const s = Math.floor(elapsed % 60);
        elTimer.textContent = `${String(m).padStart(2,'0')}:${String(s).padStart(2,'0')}`;
      }, 200);
    }

    function stopTimer() {
      clearInterval(timerInterval);
      timerInterval = null;
    }

    function saveGenerated() {
      if (!batState.generatedBlob) return;
      const url = URL.createObjectURL(batState.generatedBlob);
      const a = document.createElement('a');
      const ts = new Date().toISOString().replace(/[:.]/g, '-');
      a.href = url;
      a.download = `bat-hell-music-${ts}.wav`;
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(url);
    }

    async function sharePage() {
      if (!batState.generatedBlob) return;
      const shareData = {
        title: 'Bat-music 实验',
        text: '我刚刚用呼噜 / 怪声在 Bat-music 里炼出了一段地狱乐曲。',
        url: window.location.href
      };
      if (navigator.share) {
        try {
          await navigator.share(shareData);
        } catch (err) {
          logDebug('分享失败或被取消: ' + err.message);
        }
      } else {
        alert('当前浏览器不支持系统分享，请手动复制地址栏链接。');
      }
    }

    function showWxMask() {
      const mask = document.getElementById('wxMask');
      if (mask) mask.classList.add('show');
    }

    function hideWxMask() {
      const mask = document.getElementById('wxMask');
      if (mask) mask.classList.remove('show');
    }

    function logDebug(msg) {
      if (!debugConsole) return;
      const time = new Date().toISOString().split('T')[1].split('.')[0];
      debugConsole.textContent += `[${time}] ${msg}\n`;
      debugConsole.scrollTop = debugConsole.scrollHeight;
    }
  </script>
</body>
</html>
