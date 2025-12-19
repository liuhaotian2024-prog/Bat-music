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

    /* 底部：播放器 + 按钮 */
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

    /* 提示遮罩（微信 / 权限问题） */
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
  <div id="debugConsole"></div>

  <div class="app">
    <!-- 示波器 -->
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
        <div class="status-badge" data-phase="generated">已生成乐曲</div>
        <div class="status-badge" data-phase="playing">播放中</div>
      </div>
    </div>

    <!-- 本次乐曲气质说明（内部用 CIEU/Y* 计算） -->
    <div class="summary">
      <div class="summary-title">本次乐曲的“气质”：</div>
      <div id="summaryText" class="summary-text">
        还没有乐曲。录一段呼噜 / 怪声，我会把它“炼”成一段独一无二的音乐。
      </div>
    </div>

    <!-- 播放器 + 按钮 -->
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

  <!-- 不支持录音环境提示 -->
  <div id="wxMask">
    <div class="wxMask-content">
      <p>抱歉，当前浏览器可能不支持实时录音。</p>
      <p>建议在手机系统浏览器（Safari / Chrome）中打开本页面，并确认授权麦克风权限。</p>
    </div>
  </div>

  <script>
    /***** 全局状态 *****/
    const batState = {
      phase: 'idle',           // idle | recording | generated | playing
      recordedBlob: null,      // 原始录音 WAV Blob
      generatedBlob: null,     // 当前乐曲 Blob（目前 = 原录音，占位）
      floatData: null,         // 录音的 PCM float32
      sampleRate: 44100,
    };

    // Web Audio 相关
    let audioContext = null;
    let micStream = null;
    let sourceNode = null;
    let analyser = null;
    let processor = null;

    // 录音缓存
    let recordedBuffers = [];
    let recordingLength = 0;

    // 波形绘制
    let elCanvas, canvasCtx;
    let drawing = false;

    // DOM
    let elTimer, elStatusText, elStatusBadges;
    let btnRecord, btnStopGen, btnSave, btnShare;
    let elPlayerLayer, elAudioPlayer, elSummary, debugConsole;

    let timerInterval = null;
    let recordingStartTime = null;

    /***** 初始化 *****/
    window.addEventListener('DOMContentLoaded', () => {
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
      debugConsole = document.getElementById('debugConsole');

      attachEvents();
      resizeCanvas();
      drawIdleWave();
      setPhase('idle');

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

      elAudioPlayer.addEventListener('play', () => setPhase('playing'));
      elAudioPlayer.addEventListener('pause', () => {
        if (batState.phase === 'playing') setPhase('generated');
      });
      elAudioPlayer.addEventListener('ended', () => setPhase('generated'));
    }

    /***** Phase & UI *****/
    function setPhase(phase) {
      batState.phase = phase;

      switch (phase) {
        case 'idle':
          elStatusText.textContent = '等待录制呼噜声 / 口哨 / 敲桌子等怪声音…';
          break;
        case 'recording':
          elStatusText.textContent = '录制中：请发出你想“炼成乐曲”的声音…';
          break;
        case 'generated':
          elStatusText.textContent = '已生成乐曲，可以反复播放 / 保存 / 分享。';
          break;
        case 'playing':
          elStatusText.textContent = '播放当前乐曲中…';
          break;
      }

      elStatusBadges.forEach(badge => {
        const p = badge.getAttribute('data-phase');
        badge.classList.toggle('status-badge--active', p === phase);
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

      btnRecord.classList.toggle('recording', batState.phase === 'recording');
    }

    /***** Canvas 相关 *****/
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
      drawWaveformFromFloat(batState.floatData, batState.sampleRate);
    }

    function drawWaveformFromFloat(data, sampleRate) {
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

    /***** 录音：ScriptProcessor 版 *****/
    async function startRecording() {
      try {
        if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
          showWxMask();
          return;
        }

        if (!audioContext) {
          audioContext = new (window.AudioContext || window.webkitAudioContext)();
        }
        await audioContext.resume();

        micStream = await navigator.mediaDevices.getUserMedia({ audio: true });
        batState.sampleRate = audioContext.sampleRate;

        // mic → analyser（画波形）+ processor（录数据）
        sourceNode = audioContext.createMediaStreamSource(micStream);

        analyser = audioContext.createAnalyser();
        analyser.fftSize = 2048;
        sourceNode.connect(analyser);

        processor = audioContext.createScriptProcessor(4096, 1, 1);
        sourceNode.connect(processor);
        // ScriptProcessor 必须接到 destination 上才能触发回调
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
        logDebug('录音启动失败: ' + err.message);
        showWxMask();
      }
    }

    function stopAndGenerate() {
      if (batState.phase !== 'recording') return;
      stopTimer();
      stopDrawing();
      stopAudioGraph();

      if (recordingLength === 0) {
        setPhase('idle');
        return;
      }

      // 合并所有 buffer
      const merged = new Float32Array(recordingLength);
      let offset = 0;
      for (const buf of recordedBuffers) {
        merged.set(buf, offset);
        offset += buf.length;
      }

      // 简单归一化，确保不至于太小声
      const normalized = normalizeFloat(merged);
      batState.floatData = normalized;

      const wavBlob = encodeWAV(normalized, batState.sampleRate);
      batState.recordedBlob = wavBlob;
      logDebug('WAV 大小：' + wavBlob.size + ' 字节');

      generateFromRecording(wavBlob, normalized, batState.sampleRate);
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

    /***** 核心：生成乐曲 + CIEU/Y* 分析 *****/
    function generateFromRecording(blob, floatData, sampleRate) {
      // 1）当前占位实现：乐曲 = 原始录音
      const generatedBlob = blob;
      batState.generatedBlob = generatedBlob;
      const url = URL.createObjectURL(generatedBlob);
      elAudioPlayer.src = url;
      elPlayerLayer.classList.add('show');

      // 2）内部用 CIEU/Y* 做简单分析 → 文本描述
      const summary = analyzeAndDescribe(floatData, sampleRate);
      elSummary.textContent = summary;

      setPhase('generated');
      drawCurrentWave();
    }

    function analyzeAndDescribe(data, sampleRate) {
      const n = data.length;
      if (n === 0) return '录音为空。重新试一次吧。';

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

      const rms = Math.sqrt(sqSum / n);
      const zcr = zeroCross / n * sampleRate;

      // 理想助眠目标场 Y*
      const yStar = { rms: 0.02, zcr: 500 };

      const er = Math.abs(rms - yStar.rms) / (yStar.rms + 1e-6);
      const ez = Math.abs(zcr - yStar.zcr) / (yStar.zcr + 1e-6);
      const e = er * 0.7 + ez * 0.3;
      const reward = Math.max(0, Math.min(1, 1 - e * 0.5));

      let mood;
      if (reward > 0.8) {
        mood = '这段声音非常接近“助眠理想场”，乐曲整体偏安静、平滑，适合作为入睡背景。';
      } else if (reward > 0.5) {
        mood = '这段声音在混乱与秩序之间，生成的乐曲会带一点起伏，有点故事感。';
      } else if (reward > 0.2) {
        mood = '原始声音比较躁动，乐曲里会带明显节奏感，像一段不太安分的即兴。';
      } else {
        mood = '这是一段非常“野”的原始噪音，乐曲整体会偏实验 / 噪音风格，非常适合作为艺术素材。';
      }

      const tech =
        `（内部指标：rms=${rms.toFixed(3)}, peak=${maxAmp.toFixed(3)}, ` +
        `zcr≈${zcr.toFixed(0)} 次/秒, reward≈${reward.toFixed(2)}）`;

      return mood + ' ' + tech;
    }

    /***** WAV 编码 & 归一化 *****/
    function normalizeFloat(samples) {
      let max = 0;
      for (let i = 0; i < samples.length; i++) {
        const v = Math.abs(samples[i]);
        if (v > max) max = v;
      }
      // 如果本来就不小，就不放大；如果很小，就放到 0.9
      const target = 0.9;
      const gain = max > 0 && max < target ? (target / max) : 1.0;
      if (gain === 1.0) return samples;

      const out = new Float32Array(samples.length);
      for (let i = 0; i < samples.length; i++) {
        out[i] = samples[i] * gain;
      }
      return out;
    }

    function encodeWAV(samples, sampleRate) {
      const buffer = new ArrayBuffer(44 + samples.length * 2);
      const view = new DataView(buffer);

      function writeString(offset, str) {
        for (let i = 0; i < str.length; i++) {
          view.setUint8(offset + i, str.charCodeAt(i));
        }
      }

      const numChannels = 1;
      const bytesPerSample = 2;
      const blockAlign = numChannels * bytesPerSample;
      const byteRate = sampleRate * blockAlign;

      // RIFF header
      writeString(0, 'RIFF');
      view.setUint32(4, 36 + samples.length * 2, true);
      writeString(8, 'WAVE');

      // fmt chunk
      writeString(12, 'fmt ');
      view.setUint32(16, 16, true);
      view.setUint16(20, 1, true); // PCM
      view.setUint16(22, numChannels, true);
      view.setUint32(24, sampleRate, true);
      view.setUint32(28, byteRate, true);
      view.setUint16(32, blockAlign, true);
      view.setUint16(34, 16, true); // bits per sample

      // data chunk
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

    /***** 计时 *****/
    function startTimer() {
      recordingStartTime = performance.now();
      clearInterval(timerInterval);
      timerInterval = setInterval(() => {
        const elapsed = (performance.now() - recordingStartTime) / 1000;
        const m = Math.floor(elapsed / 60);
        const s = Math.floor(elapsed % 60);
        elTimer.textContent =
          `${String(m).padStart(2,'0')}:${String(s).padStart(2,'0')}`;
      }, 200);
    }

    function stopTimer() {
      clearInterval(timerInterval);
      timerInterval = null;
    }

    /***** 保存 & 分享 *****/
    function saveGenerated() {
      if (!batState.generatedBlob) return;
      const url = URL.createObjectURL(batState.generatedBlob);
      const a = document.createElement('a');
      const ts = new Date().toISOString().replace(/[:.]/g, '-');
      a.href = url;
      a.download = `bat-music-${ts}.wav`;
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(url);
    }

    async function sharePage() {
      if (!batState.generatedBlob) return;
      const shareData = {
        title: 'Bat-music 实验',
        text: '我刚刚用呼噜 / 怪声在 Bat-music 里生成了一段乐曲。',
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

    /***** 提示 & 调试 *****/
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
      debugConsole.style.display = 'block';
      const time = new Date().toISOString().split('T')[1].split('.')[0];
      debugConsole.textContent += `[${time}] ${msg}\n`;
    }
  </script>
</body>
</html>
