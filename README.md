<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>Bat-music · 从呼噜到地狱爵士 / 摇滚（PCM版）</title>
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
      margin: 0; padding: 0; height: 100%;
      background: var(--bg-main); color: var(--text-main);
      font-family: -apple-system, BlinkMacSystemFont, system-ui, sans-serif;
    }
    #debugConsole {
      position: fixed; top: 0; left: 0; width: 100%;
      min-height: 18px; max-height: 80px;
      background: #200; color: #ff6b6b;
      font-size: 10px; font-family: monospace;
      z-index: 999; display: block;
      padding: 2px 6px; overflow-y: auto; white-space: pre-wrap;
    }
    .app {
      display: flex; flex-direction: column; height: 100vh;
      padding-top: 80px; box-sizing: border-box;
    }
    .stage {
      position: relative; flex: 1 1 auto; min-height: 200px;
      background: radial-gradient(circle at top, #1a1c24 0, var(--bg-stage) 55%);
      border-bottom: 1px solid var(--border-soft); overflow: hidden;
    }
    .wave-canvas { width: 100%; height: 100%; display: block; }
    .center-status {
      position: absolute; top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      text-align: center; pointer-events: none;
    }
    .timer {
      font-size: 36px; font-weight: 600;
      font-family: monospace; letter-spacing: 0.06em;
    }
    .status-text { font-size: 13px; color: var(--text-muted); margin-top: 8px; }
    .status-badge-row {
      position: absolute; left: 12px; top: 12px;
      display: flex; gap: 6px; font-size: 10px;
    }
    .status-badge {
      padding: 2px 6px; border-radius: 999px;
      background: rgba(0,0,0,0.35);
      border: 1px solid rgba(255,255,255,0.08);
      color: #999;
    }
    .status-badge--active {
      color: #fff; border-color: var(--accent-primary);
      box-shadow: 0 0 10px rgba(0,122,255,0.5);
    }
    .summary {
      padding: 10px 14px 6px;
      background: var(--bg-panel);
      border-bottom: 1px solid var(--border-soft);
      font-size: 13px; color: var(--text-muted);
    }
    .summary-title { font-size: 12px; color: #aaa; margin-bottom: 4px; }
    .summary-text { font-size: 13px; color: #f5f5f5; line-height: 1.4; min-height: 2.6em; }
    .controls {
      flex: 0 0 auto; background: var(--bg-panel);
      padding: 10px 12px 14px; display: flex;
      flex-direction: column; gap: 10px;
    }
    #playerLayer {
      max-height: 0; opacity: 0; overflow: hidden;
      transition: max-height .25s ease, opacity .25s ease;
      background: #181a20; border-radius: var(--radius-lg);
      border: 1px solid transparent;
    }
    #playerLayer.show { max-height: 90px; opacity: 1; border-color: var(--border-soft); }
    #audioPlayer { width: 100%; display: block; height: 44px; outline: none; }
    .btn-row { display: flex; gap: 8px; margin-top: 4px; }
    button {
      flex: 1; border: none; border-radius: var(--radius-lg);
      font-size: 14px; font-weight: 600;
      cursor: pointer; padding: 10px 4px;
      transition: transform .08s ease, opacity .12s ease, box-shadow .12s ease;
    }
    button:active { opacity: 0.85; transform: translateY(1px); }
    button:disabled { background: #222; color: #555; cursor: default; box-shadow: none; }
    .btn-rec { background: var(--accent-record); color: #fff; }
    .btn-stop-gen { background: #3a3a3c; color: #fefefe; }
    .btn-save { background: var(--accent-ok); color: #000; }
    .btn-share { background: #2c2c30; color: var(--text-muted); }
    .btn-rec.recording { box-shadow: 0 0 16px rgba(255,59,48,0.7); }
    #wxMask {
      position: fixed; inset: 0; background: rgba(0,0,0,0.9);
      z-index: 1000; display: none;
      justify-content: center; align-items: center;
      text-align: center; padding: 24px; color: #fff;
    }
    #wxMask.show { display: flex; }
    .wxMask-content {
      max-width: 320px; font-size: 14px;
      line-height: 1.5; opacity: 0.95;
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
        还没有乐曲。录一段呼噜 / 怪声，我会把它“炼”成一段真正的地狱爵士 / 摇滚曲子。
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
          elStatusText.textContent = '根据噪音场与乐音场的张力，在地狱熔炉中作曲（爵士 / 摇滚骨架）…';
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

    /********** 录音管线 **********/
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

    /********** 地狱作曲：爵士 / 摇滚曲式 **********/
    async function generateFromRecording(floatData, sampleRate) {
      try {
        const analysis = analyzeAndDescribe(floatData, sampleRate);
        elSummary.textContent = analysis.summary;

        const composed = composeSongPCM(analysis.features, analysis.styleId, floatData, sampleRate);
        const mastered = applyHellMaster(composed, sampleRate, analysis.styleId, analysis.features);
        batState.floatData = mastered;

        const wavBlob = encodeWAV(mastered, sampleRate);
        batState.generatedBlob = wavBlob;

        const url = URL.createObjectURL(wavBlob);
        elAudioPlayer.src = url;
        elPlayerLayer.classList.add('show');

        drawCurrentWave();
        logDebug('作曲样本数=' + mastered.length + ', wavSize=' + wavBlob.size + ' 字节');

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

    /********** 噪音场分析 **********/
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

      let styleId = 'hell_jazz'; // 只在 jazz / rock 上玩，其他合并到这两种
      let styleName = '地狱爵士';

      if (zcr > 2500 && rms > 0.08) {
        styleId = 'hell_rock';
        styleName = '地狱摇滚';
      } else if (zcr > 1800 && rms > 0.05) {
        styleId = 'hell_rock';
        styleName = '地狱摇滚';
      } else if (zcr > 1200 && rms <= 0.05) {
        styleId = 'hell_jazz';
        styleName = '地狱爵士';
      } else if (zcr <= 1200 && rms > 0.06) {
        styleId = 'hell_jazz';
        styleName = '地狱慢爵士';
      } else if (reward > 0.8) {
        styleId = 'hell_jazz';
        styleName = '地狱冥想爵士';
      }

      const tech =
        `（特征: len=${duration.toFixed(2)}s, rms=${rms.toFixed(3)}, ` +
        `peak=${maxAmp.toFixed(3)}, zcr≈${zcr.toFixed(0)}次/秒, Y*·r≈${reward.toFixed(2)}）`;

      const summary = `我把这段呼噜 / 怪声视为一个“地狱噪音场”，识别为「${styleName}」素材。${tech}`;

      return {
        summary,
        styleId,
        features: { rms, peak:maxAmp, zcr, duration, reward }
      };
    }

    /********** 曲式作曲引擎 **********/
    function composeSongPCM(features, styleId, sourceData, sampleRate) {
      const style = getSongStyle(styleId, features);
      const spb = 60 / style.bpm;
      const beatsPerBar = 4;
      const bars = style.bars;
      const beats = bars * beatsPerBar;
      const totalSeconds = beats * spb;
      const maxSeconds = 16;
      const lengthSeconds = Math.min(totalSeconds, maxSeconds);
      const length = Math.floor(lengthSeconds * sampleRate);

      const out = new Float32Array(length);

      const motif = deriveMotifFromNoise(sourceData || batState.floatData, sampleRate, 16);
      const form = buildSongForm(bars, styleId, features.reward);

      if (styleId === 'hell_jazz') {
        addJazzDrums(out, sampleRate, style, features, form);
        addJazzBass(out, sampleRate, style, features, form, motif);
        addJazzPiano(out, sampleRate, style, features, form, motif);
        addJazzLead(out, sampleRate, style, features, form, motif);
      } else {
        addRockDrums(out, sampleRate, style, features, form);
        addRockBass(out, sampleRate, style, features, form, motif);
        addRockRiff(out, sampleRate, style, features, form, motif);
        addRockLead(out, sampleRate, style, features, form, motif);
      }

      return normalizeFloat(out);
    }

    function getSongStyle(styleId, f) {
      if (styleId === 'hell_jazz') {
        return {
          styleId: 'hell_jazz',
          bpm: 110 + 20 * f.reward, // 110-130
          scaleRoot: 50,            // D3 小调
          bars: 32,                 // 32 小节：AABA
          swing: 0.16
        };
      } else {
        return {
          styleId: 'hell_rock',
          bpm: 120 + 20 * (1 - f.reward), // 120-140
          scaleRoot: 45,                  // A2
          bars: 24,                       // verse-chorus-verse-chorus-bridge-outro 大致
          swing: 0.0
        };
      }
    }

    function deriveMotifFromNoise(data, sr, steps) {
      if (!data || data.length === 0) return [];
      const n = data.length;
      const segLen = Math.floor(n / steps);
      const motif = [];
      for (let s = 0; s < steps; s++) {
        const start = s * segLen;
        const end = (s === steps-1) ? n : Math.min(n, (s+1)*segLen);
        if (end <= start) break;
        let absSum = 0, sqSum = 0, zeroCross = 0;
        for (let i = start; i < end; i++) {
          const v = data[i];
          absSum += Math.abs(v);
          sqSum += v * v;
          if (i>start && ((data[i-1]>0 && v<=0)||(data[i-1]<0 && v>=0))) zeroCross++;
        }
        const len = end - start;
        const meanAbs = absSum / len;
        const localZCR = zeroCross / len * sr;
        const vel = Math.min(1, Math.max(0.2, meanAbs*10));
        let offset = 0;
        if (localZCR > 2500) offset = 3;
        else if (localZCR > 1800) offset = 2;
        else if (localZCR > 1200) offset = 1;
        else if (localZCR < 400) offset = -2;
        else if (localZCR < 800) offset = -1;
        motif.push({ offset, vel, zcr:localZCR });
      }
      return motif;
    }

    /********** 曲式：Jazz AABA / Rock Verse-Chorus **********/
    function buildSongForm(bars, styleId, reward) {
      const form = [];
      if (styleId === 'hell_jazz') {
        // 32 bars: A1(0-7), A2(8-15), B(16-23), A3(24-31)
        for (let b = 0; b < bars; b++) {
          let section = 'A';
          if (b < 8) section = 'A1';
          else if (b < 16) section = 'A2';
          else if (b < 24) section = 'B';
          else section = 'A3';

          let base = 0.4 + (1 - reward) * 0.3;
          let bump = 0;
          if (section === 'A1') bump = 0.1;
          if (section === 'A2') bump = 0.2;
          if (section === 'B')  bump = 0.5;
          if (section === 'A3') bump = 0.35;

          let tension = base + bump;
          tension += (Math.random() - 0.5) * 0.15;
          tension = Math.max(0, Math.min(1, tension));
          form.push({ section, tension });
        }
      } else {
        // Rock: roughly [verse1(0-5), chorus1(6-11), verse2(12-17), chorus2(18-21), outro(22-23)]
        for (let b = 0; b < bars; b++) {
          let section = 'verse';
          if (b < 6) section = 'verse1';
          else if (b < 12) section = 'chorus1';
          else if (b < 18) section = 'verse2';
          else if (b < 22) section = 'chorus2';
          else section = 'outro';

          let base = 0.5 + (1 - reward) * 0.4;
          let bump = 0;
          if (section.startsWith('verse'))  bump = 0.15;
          if (section.startsWith('chorus')) bump = 0.5;
          if (section === 'outro')          bump = 0.25;

          let tension = base + bump;
          tension += (Math.random() - 0.5) * 0.1;
          tension = Math.max(0, Math.min(1, tension));
          form.push({ section, tension });
        }
      }
      return form;
    }

    /********** PCM 原语 **********/
    function timeToIndex(t, sr){ return Math.floor(t*sr); }

    function addSine(out,sr,tStart,dur,freq,amp,decay=3.0){
      const start=timeToIndex(tStart,sr);
      const len=Math.min(out.length-start,Math.floor(dur*sr));
      const w=2*Math.PI*freq/sr;
      for(let i=0;i<len;i++){
        const env=Math.exp(-decay*i/len);
        out[start+i]+=amp*env*Math.sin(w*i);
      }
    }

    function addNoiseHit(out,sr,tStart,dur,amp,highpass=false){
      const start=timeToIndex(tStart,sr);
      const len=Math.min(out.length-start,Math.floor(dur*sr));
      let prev=0;
      for(let i=0;i<len;i++){
        let n=(Math.random()*2-1);
        if(highpass){
          const hp=n-prev*0.7;
          prev=n;
          n=hp;
        }
        const env=Math.exp(-6*i/len);
        out[start+i]+=amp*env*n;
      }
    }

    /********** Jazz Drums：swing ride + snare comp **********/
    function addJazzDrums(out,sr,style,f,form){
      const spb=60/style.bpm;
      const beatsPerBar=4;
      const swing=style.swing;

      for(let bar=0; bar<style.bars; bar++){
        const {section,tension:T}=form[bar];
        const barStart=bar*beatsPerBar*spb;

        // ride pattern：1 &a 2 &a 3 &a 4 &a（近似）
        for(let beat=0; beat<beatsPerBar; beat++){
          const tBeat = barStart + beat*spb;
          const mainT = tBeat; // downbeat
          const swingT = tBeat + spb*(2/3)*(1+swing*0.3); // swing-ish
          addNoiseHit(out,sr,mainT,0.06,0.10*(0.7+0.6*T),true);
          addNoiseHit(out,sr,swingT,0.04,0.07*(0.7+0.6*T),true);
        }

        // hi-hat on 2 and 4
        addNoiseHit(out,sr,barStart+spb*1,0.04,0.11*(0.6+0.4*T),true);
        addNoiseHit(out,sr,barStart+spb*3,0.04,0.11*(0.6+0.4*T),true);

        // snare comp：A 段简单，B 段密一点
        const compHits = section==='B' ? 3 : 2;
        for(let i=0;i<compHits;i++){
          const pos = (Math.random()*4)*spb;
          const t=barStart+pos;
          addNoiseHit(out,sr,t,0.08,0.13*(0.7+0.6*T),true);
        }

        // occasional kick on 1 & 3
        addSine(out,sr,barStart,0.22,60,0.18*(0.7+0.6*T),4.0);
        addSine(out,sr,barStart+2*spb,0.22,60,0.16*(0.7+0.6*T),4.0);
      }
    }

    /********** Jazz Bass：walking / motif 驱动 **********/
    function addJazzBass(out,sr,style,f,form,motif){
      const spb=60/style.bpm;
      const beatsPerBar=4;
      const scale=makeMinorScale(style.scaleRoot); // D 小调
      const baseStrength=0.23+0.4*f.rms/0.12;

      for(let bar=0; bar<style.bars; bar++){
        const {section,tension:T}=form[bar];
        for(let beat=0; beat<beatsPerBar; beat++){
          const motifIdx=(bar*beatsPerBar+beat)%Math.max(1,motif.length);
          const m = motif[motifIdx] || {offset:0,vel:0.6};

          const barStart=bar*beatsPerBar*spb;
          const t = barStart + beat*spb*(0.98); // 稍微靠前一点
          // 简单 ii–V–i 轮流：假设 Dm-G7-Dm
          const iiDeg = 2; const vDeg = 7; const iDeg = 0;
          let deg = iDeg;
          if (section==='A1' || section==='A3') {
            deg = (beat%4===1)?iiDeg:(beat%4===2?vDeg:iDeg);
          } else if (section==='A2') {
            deg = (beat%4===0?iiDeg:(beat%4===2?vDeg:iDeg));
          } else { // B 段更乱一点
            deg = (beat%4===0?iiDeg:(beat%4===1?vDeg:(Math.random()<0.5?iDeg:5)));
          }
          deg += m.offset;
          const idx=((deg%scale.length)+scale.length)%scale.length;
          const midi=scale[idx];
          const freq=midiToFreq(midi-12);

          const dur=spb*0.9;
          const amp=baseStrength*(0.7+0.7*T)*m.vel;
          addSine(out,sr,t,dur,freq,amp,3.5);
          addSine(out,sr,t,dur,freq*2,amp*0.25,4.0);
        }
      }
    }

    /********** Jazz Piano：和弦枕 + B 段对比 **********/
    function addJazzPiano(out,sr,style,f,form,motif){
      const spb=60/style.bpm;
      const beatsPerBar=4;
      const scale=makeMinorScale(style.scaleRoot+12);
      const baseLevel=0.09+0.12*f.reward;

      for(let bar=0; bar<style.bars; bar++){
        const {section,tension:T}=form[bar];
        const barStart=bar*beatsPerBar*spb;

        const motifIdx=bar%Math.max(1,motif.length);
        const m=motif[motifIdx] || {offset:0,vel:0.5};

        // 简单小七和弦 + 扩展：i7、iv7、bVII7 轮流
        const chordPattern=[0,5,10,3]; // i, iv, bVII, iii-ish
        let deg=chordPattern[bar%chordPattern.length]+m.offset;
        let rootIdx=((deg%scale.length)+scale.length)%scale.length;

        const chord=[scale[rootIdx],
                     scale[(rootIdx+2)%scale.length],
                     scale[(rootIdx+4)%scale.length]];
        if(T>0.4) chord.push(scale[(rootIdx+6)%scale.length]);   // 7度
        if(T>0.6) chord.push(scale[(rootIdx+1)%scale.length]);   // 9度擦音

        const level=
          baseLevel *
          (section==='B'?1.2:1.0) *
          (0.5+0.8*(1-f.reward)) *
          (0.7+0.6*T);

        const sustain = 4*spb*(0.8+0.3*T);

        chord.forEach((midi,i)=>{
          const freq=midiToFreq(midi);
          const start=barStart+spb*0.0 + i*0.02;
          addSine(out,sr,start,sustain,freq,level,2.0);
        });
      }
    }

    /********** Jazz Lead：呼应 + free 段 **********/
    function addJazzLead(out,sr,style,f,form,motif){
      const spb=60/style.bpm;
      const beatsPerBar=4;
      const scale=makeMinorScale(style.scaleRoot+19); // 高一点
      const densityBase=0.5+0.5*f.zcr/3000;

      for(let bar=0; bar<style.bars; bar++){
        const {section,tension:T}=form[bar];
        const barStart=bar*beatsPerBar*spb;

        let density =
          section==='A1'? densityBase*0.4 :
          section==='A2'? densityBase*0.6 :
          section==='B' ? densityBase*1.0 :
          densityBase*0.7;
        if(Math.random()>density)continue;

        const phrases = section==='B' ? 3 : 2;
        for(let p=0;p<phrases;p++){
          const motifIdx=(bar*phrases+p)%Math.max(1,motif.length);
          const m=motif[motifIdx] || {offset:0,vel:0.5};

          const localPos = Math.random()*4*spb;
          let t=barStart+localPos;
          const baseMidi=scale[Math.floor(Math.random()*scale.length)]+m.offset;
          let midi=baseMidi;
          if(section==='B' || T>0.6){
            midi+=(Math.random()<0.5?1:-1); // chromatic
          }
          if(T>0.7 && Math.random()<0.5){
            midi+=(Math.random()<0.5?7:-5); // 大跳
          }

          const freq=midiToFreq(midi);
          const dur=spb*(0.15+0.4*Math.random());
          const amp=0.08+0.16*(1-f.reward)*(0.5+T)*m.vel;
          addSine(out,sr,t,dur,freq,amp,4.0);
          addSine(out,sr,t,dur,freq*2,amp*0.5,5.0);
        }
      }
    }

    /********** Rock Drums **********/
    function addRockDrums(out,sr,style,f,form){
      const spb=60/style.bpm;
      const beatsPerBar=4;

      for(let bar=0; bar<style.bars; bar++){
        const {section,tension:T}=form[bar];
        const barStart=bar*beatsPerBar*spb;

        for(let beat=0; beat<beatsPerBar; beat++){
          const tBeat=barStart+beat*spb;

          const sectionScale =
            section.startsWith('verse')?0.8:
            section.startsWith('chorus')?1.2:
            0.9;
          const energy=style.drumEnergy*sectionScale*(0.7+0.6*T)*(0.6+0.6*f.rms/0.1);

          // Kick: 1 & 3
          if(beat===0 || beat===2){
            addSine(out,sr,tBeat,0.18,60+20*T,0.7*energy,5.0);
          }

          // Snare: 2 & 4
          if(beat===1 || beat===3){
            addNoiseHit(out,sr,tBeat,0.14,0.7*energy,true);
          }

          // Hats 8 分
          const hatT1=tBeat;
          const hatT2=tBeat+spb*0.5;
          addNoiseHit(out,sr,hatT1,0.05,0.18*energy,true);
          addNoiseHit(out,sr,hatT2,0.05,0.16*energy,true);
        }

        // 简单 fill：每小节末尾有小概率加 tom/noise 滚奏
        if(section.startsWith('chorus') && Math.random()<0.3){
          const fillStart=barStart+3*spb;
          const hits=4;
          for(let k=0;k<hits;k++){
            addNoiseHit(out,sr,fillStart+k*spb*0.25,0.09,0.3*style.drumEnergy,true);
          }
        }
      }
    }

    /********** Rock Bass / Riff / Lead **********/
    function addRockBass(out,sr,style,f,form,motif){
      const spb=60/style.bpm;
      const beatsPerBar=4;
      const scale=makeMinorScale(style.scaleRoot);
      const baseStrength=0.25+0.4*f.rms/0.12;

      for(let bar=0; bar<style.bars; bar++){
        const {section,tension:T}=form[bar];
        const barStart=bar*beatsPerBar*spb;
        const motifIdx=bar%Math.max(1,motif.length);
        const m=motif[motifIdx] || {offset:0,vel:0.6};
        // 常见 rock 进程：i - bVII - IV - i
        const prog=[0,10,5,0];
        const progDeg=prog[bar%prog.length]+m.offset;
        const idx=((progDeg%scale.length)+scale.length)%scale.length;
        const midi=scale[idx];
        const freq=midiToFreq(midi-12);

        for(let beat=0; beat<beatsPerBar; beat++){
          if(section==='verse2' && beat===2 && Math.random()<0.5)continue;
          const t=barStart+beat*spb;
          const dur=spb*(0.9);
          const amp=baseStrength*(0.8+0.8*T)*m.vel;
          addSine(out,sr,t,dur,freq,amp,3.5);
          addSine(out,sr,t,dur,freq*2,amp*0.3,4.0);
        }
      }
    }

    function addRockRiff(out,sr,style,f,form,motif){
      const spb=60/style.bpm;
      const beatsPerBar=4;
      const scale=makeMinorScale(style.scaleRoot+12);
      for(let bar=0; bar<style.bars; bar++){
        const {section,tension:T}=form[bar];
        if(!section.startsWith('chorus'))continue;
        const barStart=bar*beatsPerBar*spb;

        const steps=8;
        for(let i=0;i<steps;i++){
          const pos=(i/steps)*4*spb;
          const t=barStart+pos;
          const motifIdx=(bar*steps+i)%Math.max(1,motif.length);
          const m=motif[motifIdx] || {offset:0,vel:0.6};
          let midi=scale[(i%scale.length)];
          midi+=m.offset;
          const freq=midiToFreq(midi);
          const dur=spb*0.25;
          const amp=0.12+0.16*(1-f.reward)*(0.5+T)*m.vel;
          addSine(out,sr,t,dur,freq,amp,4.0);
        }
      }
    }

    function addRockLead(out,sr,style,f,form,motif){
      const spb=60/style.bpm;
      const beatsPerBar=4;
      const scale=makeMinorScale(style.scaleRoot+19);
      const densityBase=0.4+0.4*f.zcr/3000;

      for(let bar=0; bar<style.bars; bar++){
        const {section,tension:T}=form[bar];
        const barStart=bar*beatsPerBar*spb;
        let density =
          section==='verse1'||section==='verse2'? densityBase*0.4 :
          section.startsWith('chorus')? densityBase*(0.9+0.7*T) :
          densityBase*0.3;
        if(Math.random()>density)continue;

        const hits=section.startsWith('chorus')?3:2;
        for(let h=0;h<hits;h++){
          const motifIdx=(bar*hits+h)%Math.max(1,motif.length);
          const m=motif[motifIdx] || {offset:0,vel:0.6};
          const localPos=Math.random()*4*spb;
          const t=barStart+localPos;
          let midi=scale[Math.floor(Math.random()*scale.length)]+m.offset;
          if(T>0.6 && Math.random()<0.4)midi+=(Math.random()<0.5?12:-12);
          const freq=midiToFreq(midi);
          const dur=spb*(0.15+0.4*Math.random());
          const amp=0.12+0.18*(1-f.reward)*(0.5+T)*m.vel;
          addSine(out,sr,t,dur,freq,amp,4.0);
          addSine(out,sr,t,dur,freq*2,amp*0.6,5.0);
        }
      }
    }

    /********** Hell Mastering：整体失真 **********/
    function applyHellMaster(samples, sampleRate, styleId, features) {
      const n = samples.length;
      const out = new Float32Array(n);
      const duration = n / sampleRate;

      let driveBase = 3.0;
      if (styleId === 'hell_rock') driveBase = 7.0;
      else if (styleId === 'hell_jazz') driveBase = 4.0;

      for (let i = 0; i < n; i++) {
        const t = i / sampleRate;
        const x = samples[i];

        let env = 1.0;
        if (t < 0.4) env *= t / 0.4;
        if (duration - t < 0.4) env *= (duration - t) / 0.4;

        const mid = t / duration;
        env *= 0.9 + 0.2 * Math.sin(Math.PI * mid);

        const drive = driveBase * (1 + (1 - features.reward) * 1.5);
        const z = x * env * drive;

        let y = Math.tanh(z);
        y = Math.round(y * 24) / 24; // 粗一点的量化
        out[i] = y;
      }
      return normalizeFloat(out);
    }

    /********** 工具 **********/
    function makeMinorScale(rootMidi){
      const intervals=[0,2,3,5,7,8,10];
      return intervals.map(i=>rootMidi+i);
    }

    function midiToFreq(midi){
      return 440*Math.pow(2,(midi-69)/12);
    }

    function normalizeFloat(samples){
      let max=0;
      for(let i=0;i<samples.length;i++){
        const v=Math.abs(samples[i]);
        if(v>max)max=v;
      }
      const target=0.9;
      const gain=(max>0 && max<target)?(target/max):1;
      if(gain===1)return samples;
      const out=new Float32Array(samples.length);
      for(let i=0;i<samples.length;i++)out[i]=samples[i]*gain;
      return out;
    }

    function encodeWAV(samples,sampleRate){
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

    function startTimer(){
      recordingStartTime=performance.now();
      clearInterval(timerInterval);
      timerInterval=setInterval(()=>{
        const elapsed=(performance.now()-recordingStartTime)/1000;
        const m=Math.floor(elapsed/60);
        const s=Math.floor(elapsed%60);
        elTimer.textContent=`${String(m).padStart(2,'0')}:${String(s).padStart(2,'0')}`;
      },200);
    }

    function stopTimer(){
      clearInterval(timerInterval);
      timerInterval=null;
    }

    function saveGenerated(){
      if(!batState.generatedBlob)return;
      const url=URL.createObjectURL(batState.generatedBlob);
      const a=document.createElement('a');
      const ts=new Date().toISOString().replace(/[:.]/g,'-');
      a.href=url;
      a.download=`bat-hell-song-${ts}.wav`;
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(url);
    }

    async function sharePage(){
      if(!batState.generatedBlob)return;
      const shareData={
        title:'Bat-music 实验',
        text:'我刚刚用呼噜 / 怪声在 Bat-music 里炼出了一段地狱爵士 / 摇滚乐曲。',
        url:window.location.href
      };
      if(navigator.share){
        try{ await navigator.share(shareData); }
        catch(err){ logDebug('分享失败或被取消: '+err.message); }
      }else{
        alert('当前浏览器不支持系统分享，请手动复制地址栏链接。');
      }
    }

    function showWxMask(){
      const mask=document.getElementById('wxMask');
      if(mask)mask.classList.add('show');
    }

    function hideWxMask(){
      const mask=document.getElementById('wxMask');
      if(mask)mask.classList.remove('show');
    }

    function logDebug(msg){
      if(!debugConsole)return;
      const time=new Date().toISOString().split('T')[1].split('.')[0];
      debugConsole.textContent+=`[${time}] ${msg}\n`;
      debugConsole.scrollTop=debugConsole.scrollHeight;
    }
  </script>
</body>
</html>
