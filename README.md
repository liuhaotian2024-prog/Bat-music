// main.js

// ========== 0. 全局状态 ==========

const batState = {
    phase: 'idle',          // idle | recording | tokenizing | cieu | generating | playing
    targetStyle: null,      // 交响 / 爵士 / rap 等
    recordedBlob: null,     // 原始录音
    generatedBlob: null,    // 生成音乐
    audioBuffer: null,      // AudioBuffer（用于画波形 / 提取特征）
    cieu: {
        x: null,            // 状态特征
        u: null,            // 干预描述
        yStar: null,        // 目标
        yOutcome: null,     // 结果
        reward: null        // 奖励
    }
};

let mediaRecorder = null;
let recordedChunks = [];
let audioContext = null;
let recordingStartTime = null;
let timerInterval = null;

// DOM 引用
let elTimer, elStatusText, elWaveCanvas, waveCtx;
let elStatusBadges, elBwlTags;
let elCieuX, elCieuU, elCieuYStar, elCieuYOutcome, elCieuR;
let btnRecord, btnStop, btnSave, btnReset;
let elPlayerLayer, elAudioPlayer;
let targetChips;
let debugConsole;

// ========== 1. 初始化 ==========

window.addEventListener('DOMContentLoaded', () => {
    // 基本元素
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
    btnStop = document.getElementById('btnStop');
    btnSave = document.getElementById('btnSave');
    btnReset = document.getElementById('btnReset');

    elPlayerLayer = document.getElementById('playerLayer');
    elAudioPlayer = document.getElementById('audioPlayer');

    targetChips = document.querySelectorAll('.target-chip');
    debugConsole = document.getElementById('debugConsole');

    attachEventHandlers();
    resizeCanvas();
    drawIdleWave();

    setPhase('idle');
    updateCIEUPanel();

    window.addEventListener('resize', () => {
        resizeCanvas();
        drawCurrentWave();
    });

    // 简单错误捕获 → 调试控制台
    window.addEventListener('error', (e) => {
        logDebug(`Error: ${e.message}\n${e.filename}:${e.lineno}`);
    });
});

// ========== 2. 事件绑定 ==========

function attachEventHandlers() {
    btnRecord.addEventListener('click', handleStartRecording);
    btnStop.addEventListener('click', handleStopRecording);
    btnSave.addEventListener('click', handleGenerateMusic);
    btnReset.addEventListener('click', handleReset);

    targetChips.forEach(chip => {
        chip.addEventListener('click', () => {
            targetChips.forEach(c => c.classList.remove('target-chip--active'));
            chip.classList.add('target-chip--active');
            const style = chip.getAttribute('data-style');
            batState.targetStyle = style;
            batState.cieu.yStar = style;
            updateCIEUPanel();
        });
    });

    if (elAudioPlayer) {
        elAudioPlayer.addEventListener('play', () => setPhase('playing'));
        elAudioPlayer.addEventListener('pause', () => {
            if (batState.phase === 'playing') setPhase('cieu');
        });
        elAudioPlayer.addEventListener('ended', () => setPhase('cieu'));
    }
}

// ========== 3. Phase / UI 状态管理 ==========

function setPhase(phase) {
    batState.phase = phase;

    // 更新状态文本
    switch (phase) {
        case 'idle':
            elStatusText.textContent = '等待录制呼噜声 / 怪声音…';
            break;
        case 'recording':
            elStatusText.textContent = '录制中：请发出呼噜声 / 怪声音…';
            break;
        case 'tokenizing':
            elStatusText.textContent = '对波形进行词元化 / 语法检查…';
            break;
        case 'cieu':
            elStatusText.textContent = 'CIEU 已构建，可进行音乐生成或调参。';
            break;
        case 'generating':
            elStatusText.textContent = '基于 y* 生成音乐片段…';
            break;
        case 'playing':
            elStatusText.textContent = '播放生成的音乐片段…';
            break;
    }

    // 更新状态徽章
    elStatusBadges.forEach(badge => {
        const p = badge.getAttribute('data-phase');
        if (p === phase) {
            badge.classList.add('status-badge--active');
        } else {
            badge.classList.remove('status-badge--active');
        }
    });

    // 映射到 BWL 层
    const focusLayers = [];
    if (phase === 'recording') focusLayers.push('L0');
    if (phase === 'tokenizing') focusLayers.push('L1', 'L2');
    if (phase === 'cieu') focusLayers.push('L3');
    if (phase === 'generating' || phase === 'playing') focusLayers.push('L4');

    elBwlTags.forEach(tag => {
        const layer = tag.getAttribute('data-layer');
        if (focusLayers.includes(layer)) {
            tag.classList.add('bwl-layer-tag--focus');
        } else {
            tag.classList.remove('bwl-layer-tag--focus');
        }
    });

    updateButtons();
}

// 按钮状态
function updateButtons() {
    const hasRecording = !!batState.recordedBlob;
    const hasGenerated = !!batState.generatedBlob;

    if (batState.phase === 'recording') {
        btnRecord.disabled = true;
        btnStop.disabled = false;
    } else {
        btnRecord.disabled = false;
        btnStop.disabled = true;
    }

    btnSave.disabled = !hasRecording;
    btnReset.disabled = !(hasRecording || hasGenerated);

    // 录制按钮视觉
    if (batState.phase === 'recording') {
        btnRecord.classList.add('recording');
        btnRecord.textContent = '录制中…';
    } else {
        btnRecord.classList.remove('recording');
        btnRecord.textContent = '开始录制';
    }
}

// ========== 4. Canvas 波形渲染 ==========

function resizeCanvas() {
    const rect = elWaveCanvas.getBoundingClientRect();
    const dpr = window.devicePixelRatio || 1;
    elWaveCanvas.width = rect.width * dpr;
    elWaveCanvas.height = rect.height * dpr;
    waveCtx.setTransform(dpr, 0, 0, dpr, 0, 0); // 保持绘制坐标是 CSS 像素
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
    const amp = height / 2 * 0.85;

    ctx.beginPath();
    ctx.moveTo(0, height / 2);

    for (let x = 0; x < width; x++) {
        let sum = 0;
        const start = x * step;
        const end = Math.min(start + step, channelData.length);
        for (let i = start; i < end; i++) sum += Math.abs(channelData[i]);
        const avg = sum / (end - start || 1);
        const y = height / 2 - avg * amp * (channelData[start] >= 0 ? 1 : -1);
        ctx.lineTo(x, y);
    }

    ctx.strokeStyle = '#4cd964';
    ctx.lineWidth = 1.5;
    ctx.stroke();
}

// ========== 5. 录音流程 (L0) ==========

async function handleStartRecording() {
    try {
        if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
            showWxMask();
            return;
        }

        if (!mediaRecorder) {
            const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
            mediaRecorder = new MediaRecorder(stream);

            mediaRecorder.ondataavailable = (e) => {
                if (e.data.size > 0) recordedChunks.push(e.data);
            };

            mediaRecorder.onstop = handleRecordingStop;
        }

        recordedChunks = [];
        mediaRecorder.start();
        recordingStartTime = performance.now();
        startTimer();

        batState.recordedBlob = null;
        batState.generatedBlob = null;
        batState.audioBuffer = null;

        setPhase('recording');
        updateCIEUPanel();
        drawIdleWave();
    } catch (err) {
        logDebug('录音启动失败: ' + err.message);
        showWxMask();
    }
}

function handleStopRecording() {
    if (mediaRecorder && mediaRecorder.state === 'recording') {
        mediaRecorder.stop();
        stopTimer();
        setPhase('tokenizing');
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

        // 提取简单特征，写入 CIEU.x
        extractFeaturesToCIEU(audioBuffer);

        // 干预 uₜ：这里先用「原声录制 + 目标风格」的描述
        batState.cieu.u = buildInterventionDescription();

        setPhase('cieu');
        updateCIEUPanel();
    } catch (err) {
        logDebug('处理录音失败: ' + err.message);
        setPhase('idle');
    }
}

// ========== 6. 简单特征提取 → 填 CIEU.x ==========

function extractFeaturesToCIEU(buffer) {
    const duration = buffer.duration; // 秒
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

    const text = `len=${duration.toFixed(2)}s, sr=${sampleRate}Hz, meanAmp=${meanAmp.toFixed(3)}, peak=${maxAmp.toFixed(3)}`;

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
    // 可以在这里加入更多「干预」信息：例如未来加节奏拉伸、滤波参数等
    const style = batState.targetStyle ? batState.targetStyle : 'none';
    return `op=mic_record, style=${style}`;
}

// ========== 7. 生成音乐 (y*, y, r) ==========

async function handleGenerateMusic() {
    if (!batState.recordedBlob) return;

    setPhase('generating');

    try {
        // TODO: 这里将来替换为调用 Bat-mind / 服务器端模型的逻辑
        // 现在先用「原始录音」作为“生成音乐”的占位版本。
        const generatedBlob = await fakeGenerateFromRecorded(batState.recordedBlob);

        batState.generatedBlob = generatedBlob;
        const url = URL.createObjectURL(generatedBlob);
        elAudioPlayer.src = url;

        // 显示播放器
        elPlayerLayer.classList.add('show');

        // 填充 CIEU.y 和 CIEU.r（这里 reward 只是占位）
        batState.cieu.yOutcome = {
            source: 'placeholder',
            desc: '当前使用原始录音作为 yₜ+Δ（占位版本）'
        };

        // 一个简单的「伪 reward」：如果设置过 y*，给 1.0，否则 0.2
        const reward = batState.cieu.yStar ? 1.0 : 0.2;
        batState.cieu.reward = reward;

        updateCIEUPanel();
        setPhase('cieu');
    } catch (err) {
        logDebug('生成音乐失败: ' + err.message);
        setPhase('cieu');
    }
}

// 占位：当前只是返回原始录音，方便先打通 UI 流程
function fakeGenerateFromRecorded(blob) {
    return new Promise((resolve) => {
        setTimeout(() => resolve(blob), 300);
    });
}

// ========== 8. CIEU 面板展示 ==========

function updateCIEUPanel() {
    // xₜ
    if (batState.cieu.x && batState.cieu.x.text) {
        elCieuX.textContent = batState.cieu.x.text;
    } else {
        elCieuX.textContent = '—';
    }

    // uₜ
    elCieuU.textContent = batState.cieu.u || '等待操作';

    // y*ₜ
    if (batState.cieu.yStar) {
        elCieuYStar.textContent = batState.cieu.yStar;
    } else {
        elCieuYStar.textContent = '未选择';
    }

    // yₜ+Δ
    if (batState.cieu.yOutcome && batState.cieu.yOutcome.desc) {
        elCieuYOutcome.textContent = batState.cieu.yOutcome.desc;
    } else {
        elCieuYOutcome.textContent = '—';
    }

    // rₜ+Δ
    if (typeof batState.cieu.reward === 'number') {
        elCieuR.textContent = batState.cieu.reward.toFixed(2);
    } else {
        elCieuR.textContent = '—';
    }

    updateButtons();
}

// ========== 9. Timer（录音计时） ==========

function startTimer() {
    recordingStartTime = performance.now();
    clearInterval(timerInterval);

    timerInterval = setInterval(() => {
        const elapsed = (performance.now() - recordingStartTime) / 1000;
        const m = Math.floor(elapsed / 60);
        const s = Math.floor(elapsed % 60);
        elTimer.textContent = `${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`;
    }, 200);
}

function stopTimer() {
    clearInterval(timerInterval);
    timerInterval = null;
}

// ========== 10. Reset / 微信遮罩 / 调试 ==========

function handleReset() {
    batState.phase = 'idle';
    batState.targetStyle = null;
    batState.recordedBlob = null;
    batState.generatedBlob = null;
    batState.audioBuffer = null;
    batState.cieu = { x: null, u: null, yStar: null, yOutcome: null, reward: null };

    targetChips.forEach(c => c.classList.remove('target-chip--active'));
    elAudioPlayer.src = '';
    elPlayerLayer.classList.remove('show');
    elTimer.textContent = '00:00';

    drawIdleWave();
    updateCIEUPanel();
    setPhase('idle');
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
    debugConsole.style.display = 'block';
    const time = new Date().toISOString().split('T')[1].split('.')[0];
    debugConsole.textContent += `[${time}] ${msg}\n`;
}
