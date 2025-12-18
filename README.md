<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Bat-music: Grid Layout</title>
    <style>
        /* === 全局样式 === */
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background-color: #000;
            color: #fff;
            margin: 0;
            height: 100vh;
            display: flex;
            flex-direction: column;
            overflow: hidden;
        }

        /* 1. 上半部：示波器与状态 */
        .monitor-area {
            flex: 1;
            position: relative;
            background: #080808;
            border-bottom: 1px solid #333;
        }
        canvas { width: 100%; height: 100%; display: block; }

        /* 状态栏 */
        .status-header {
            position: absolute;
            top: 0; left: 0; width: 100%;
            padding: 15px;
            box-sizing: border-box;
            display: flex;
            justify-content: space-between;
            align-items: center;
            pointer-events: none;
        }
        .status-badge {
            font-size: 12px;
            font-weight: bold;
            background: rgba(255,255,255,0.1);
            padding: 4px 10px;
            border-radius: 4px;
            color: #888;
        }
        .status-badge.active { background: #f00; color: #fff; box-shadow: 0 0 10px #f00; }
        .timer { font-family: 'Monaco', monospace; font-size: 18px; font-weight: bold; }

        /* HUD 参数 */
        .hud-params {
            position: absolute;
            bottom: 10px; left: 10px;
            font-size: 10px;
            color: #666;
            line-height: 1.5;
            pointer-events: none;
        }
        .hud-val { color: #0af; font-weight: bold; }

        /* 2. 下半部：控制面板 */
        .control-panel {
            background: #121212;
            padding: 20px;
            padding-bottom: env(safe-area-inset-bottom, 20px);
            border-top: 1px solid #333;
            display: flex;
            flex-direction: column;
            gap: 15px;
        }

        /* 音频播放器 (默认隐藏，生成后显示) */
        .player-box {
            height: 0;
            overflow: hidden;
            transition: height 0.3s;
            background: #222;
            border-radius: 8px;
        }
        .player-box.show { height: 40px; border: 1px solid #444; }
        audio { width: 100%; height: 100%; outline: none; }

        /* === 核心：2x2 按钮网格 === */
        .grid-container {
            display: grid;
            grid-template-columns: 1fr 1fr; /* 两列 */
            grid-template-rows: 1fr 1fr;    /* 两行 */
            gap: 12px;
            width: 100%;
        }

        button {
            border: none;
            border-radius: 10px;
            padding: 16px 5px;
            font-size: 14px;
            font-weight: bold;
            cursor: pointer;
            transition: all 0.2s;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            gap: 5px;
            color: #fff;
            background: #222; /* 默认禁用色 */
            opacity: 0.5;
            pointer-events: none; /* 默认不可点 */
        }
        
        button i { font-size: 18px; font-style: normal; }

        /* 激活状态的按钮样式 */
        button.enabled { opacity: 1; pointer-events: auto; }
        
        /* 按钮 1: 开始 */
        #btnStart.enabled { background: #007aff; }
        #btnStart.recording { background: #333; color: #666; opacity: 0.5; pointer-events: none; } /* 录制时变灰 */

        /* 按钮 2: 停止 */
        #btnStop.enabled { background: #ff3b30; box-shadow: 0 0 15px rgba(255, 59, 48, 0.3); }

        /* 按钮 3: 存储 */
        #btnSave.enabled { background: #34c759; color: #000; }

        /* 按钮 4: 分享 */
        #btnShare.enabled { background: #5856d6; }

    </style>
</head>
<body>

    <div class="monitor-area">
        <canvas id="canvas"></canvas>
        
        <div class="status-header">
            <div class="status-badge" id="recBadge">● REC</div>
            <div class="timer" id="timer">00:00</div>
        </div>

        <div class="hud-params">
            <div>TARGET: <span id="valPitch" class="hud-val">--</span></div>
            <div>TIMBRE: <span id="valTimbre" class="hud-val">--</span></div>
            <div>ENERGY: <span id="valEnergy" class="hud-val">0</span></div>
        </div>
    </div>

    <div class="control-panel">
        
        <div class="player-box" id="playerBox">
            <audio id="audioPlayer" controls playsinline></audio>
        </div>

        <div class="grid-container">
            <button id="btnStart" class="enabled" onclick="startRecording()">
                <i>▶</i> 开始录制
            </button>
            <button id="btnStop" onclick="stopRecording()">
                <i>⏹</i> 停止生成
            </button>

            <button id="btnSave" onclick="saveFile()">
                <i>💾</i> 存储乐曲
            </button>
            <button id="btnShare" onclick="shareFile()">
                <i>🔗</i> 分享乐曲
            </button>
        </div>
    </div>

    <script>
        // === 全局变量 ===
        let audioCtx, analyser, micSource;
        let destNode, mediaRecorder;
        let audioChunks = [];
        let blobUrl = null;
        let finalBlob = null; // 用于分享的文件对象
        let wakeLock = null;

        // 合成器组件
        let osc, filter, gainNode, delay, feedback;
        
        // 状态
        let isRecording = false;
        let startTime = 0;
        let timerInt = null;
        let gateOpen = false;

        // 绘图
        let canvas, ctx, w, h;
        let energy = 0, centroid = 0;

        // C Dorian 音阶
        const SCALE = [130.81, 146.83, 155.56, 174.61, 196.00, 220.00, 233.08, 261.63, 293.66, 311.13, 349.23, 392.00, 440.00, 523.25];

        // === 1. 按钮逻辑：开始 ===
        async function startRecording() {
            try {
                // UI 状态更新
                setButtonState('btnStart', false); // 禁用开始
                setButtonState('btnStop', true);   // 启用停止
                setButtonState('btnSave', false);
                setButtonState('btnShare', false);
                
                document.getElementById('recBadge').classList.add('active');
                document.getElementById('playerBox').classList.remove('show'); // 隐藏播放器
                
                // 计时器
                startTime = Date.now();
                timerInt = setInterval(updateTimer, 1000);

                // 唤醒锁
                if ('wakeLock' in navigator) { try { wakeLock = await navigator.wakeLock.request('screen'); } catch(e){} }

                // 初始化音频
                if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                if (audioCtx.state === 'suspended') await audioCtx.resume();

                await initEngine();

                // 录音路由
                destNode = audioCtx.createMediaStreamDestination();
                audioChunks = [];

                // 断开旧连接，建立双路路由 (监听 + 录制)
                gainNode.disconnect();
                delay.disconnect();

                gainNode.connect(audioCtx.destination);
                gainNode.connect(destNode);
                
                delay.connect(audioCtx.destination);
                delay.connect(destNode);
                delay.connect(feedback); // 保持回声闭环

                // 启动录音机
                let mimeType = 'audio/webm';
                if (!MediaRecorder.isTypeSupported(mimeType)) mimeType = 'audio/mp4';
                
                mediaRecorder = new MediaRecorder(destNode.stream, mimeType ? {mimeType} : undefined);
                mediaRecorder.ondataavailable = e => { if (e.data.size > 0) audioChunks.push(e.data); };
                mediaRecorder.start();

                isRecording = true;
                computeLoop();

            } catch (e) {
                alert("启动失败: " + e);
                resetUI();
            }
        }

        // === 2. 按钮逻辑：停止 ===
        function stopRecording() {
            if (!mediaRecorder || mediaRecorder.state === 'inactive') return;

            mediaRecorder.stop();
            isRecording = false;
            clearInterval(timerInt);
            
            // 物理静音
            gateOpen = false;
            if(gainNode) gainNode.gain.setTargetAtTime(0, audioCtx.currentTime, 0.2);

            // UI 更新
            setButtonState('btnStop', false);
            document.getElementById('recBadge').classList.remove('active');

            mediaRecorder.onstop = () => {
                finalBlob = new Blob(audioChunks, { type: 'audio/webm' });
                if (blobUrl) URL.revokeObjectURL(blobUrl);
                blobUrl = URL.createObjectURL(finalBlob);

                // 加载播放器
                const player = document.getElementById('audioPlayer');
                player.src = blobUrl;
                document.getElementById('playerBox').classList.add('show');

                // 启用第二排按钮
                setButtonState('btnStart', true); // 允许重新开始
                setButtonState('btnSave', true);
                setButtonState('btnShare', true);
            };
        }

        // === 3. 按钮逻辑：存储 ===
        function saveFile() {
            if (!blobUrl) return;
            const link = document.createElement('a');
            link.href = blobUrl;
            const timeStr = new Date().toISOString().slice(11,16).replace(':','');
            link.download = `Bat-Music_${timeStr}.webm`;
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
        }

        // === 4. 按钮逻辑：分享 ===
        async function shareFile() {
            if (!finalBlob) return;
            
            // 构建文件对象
            const file = new File([finalBlob], "bat-music.webm", { type: "audio/webm" });

            if (navigator.share && navigator.canShare && navigator.canShare({ files: [file] })) {
                try {
                    await navigator.share({
                        files: [file],
                        title: 'Bat-music Creation',
                        text: 'Listen to my sleep symphony!',
                    });
                } catch (err) {
                    console.log('Share failed:', err);
                }
            } else {
                alert("您的浏览器不支持直接分享文件，请使用“存储”按钮。");
            }
        }

        // === 辅助逻辑 ===
        function setButtonState(id, enabled) {
            const btn = document.getElementById(id);
            if (enabled) {
                btn.classList.add('enabled');
                if(id === 'btnStart') btn.classList.remove('recording');
            } else {
                btn.classList.remove('enabled');
                if(id === 'btnStart') btn.classList.add('recording'); // 特殊样式
            }
        }

        function updateTimer() {
            let s = Math.floor((Date.now() - startTime) / 1000);
            let m = Math.floor(s / 60).toString().padStart(2, '0');
            s = (s % 60).toString().padStart(2, '0');
            document.getElementById('timer').innerText = `${m}:${s}`;
        }

        function resetUI() {
            setButtonState('btnStart', true);
            setButtonState('btnStop', false);
            setButtonState('btnSave', false);
            setButtonState('btnShare', false);
        }

        // === 内核引擎 (保持 v2.3 的物理逻辑) ===
        async function initEngine() {
            if (osc) return;
            osc = audioCtx.createOscillator(); osc.type = 'sine';
            filter = audioCtx.createBiquadFilter(); filter.type = 'lowpass'; filter.Q.value = 5;
            gainNode = audioCtx.createGain(); gainNode.gain.value = 0;
            delay = audioCtx.createDelay(); delay.delayTime.value = 0.4;
            feedback = audioCtx.createGain(); feedback.gain.value = 0.35;
            
            feedback.connect(delay);
            osc.connect(filter);
            filter.connect(gainNode); // 后续路由在 Start 中动态连接

            osc.start();

            const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
            analyser = audioCtx.createAnalyser(); analyser.fftSize = 2048;
            micSource = audioCtx.createMediaStreamSource(stream); micSource.connect(analyser);

            canvas = document.getElementById('canvas');
            ctx = canvas.getContext('2d');
            resizeCanvas();
            window.addEventListener('resize', resizeCanvas);
            drawLoop();
        }

        function computeLoop() {
            if (!isRecording) return;
            requestAnimationFrame(computeLoop);
            const len = analyser.frequencyBinCount;
            const data = new Uint8Array(len);
            analyser.getByteFrequencyData(data);
            
            let sum=0, wSum=0;
            for(let i=0; i<len; i++) { sum+=data[i]; wSum+=i*data[i]; }
            energy = sum/len; centroid = sum>0 ? wSum/sum : 0;

            if (energy > 12) {
                if (!gateOpen) {
                    gainNode.gain.cancelScheduledValues(audioCtx.currentTime);
                    gainNode.gain.linearRampToValueAtTime(0.5, audioCtx.currentTime + 0.1);
                    gateOpen = true;
                }
                
                let type='sine'; let tName='SINE';
                if(centroid>60){type='sawtooth'; tName='SAW';}
                else if(centroid>25){type='triangle'; tName='TRI';}
                if(osc.type!==type) osc.type=type;

                let idx = Math.floor((energy/60)*SCALE.length);
                if(idx>=SCALE.length) idx=SCALE.length-1;
                let target=SCALE[idx];

                osc.frequency.setTargetAtTime(target, audioCtx.currentTime, 0.15);
                filter.frequency.setTargetAtTime(500+energy*30, audioCtx.currentTime, 0.1);
                
                document.getElementById('valPitch').innerText = target.toFixed(0)+'Hz';
                document.getElementById('valTimbre').innerText = tName;
                document.getElementById('valEnergy').innerText = energy.toFixed(1);
            } else {
                if (gateOpen) {
                    gainNode.gain.cancelScheduledValues(audioCtx.currentTime);
                    gainNode.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 1.0);
                    gateOpen = false;
                }
            }
        }

        function drawLoop() {
            requestAnimationFrame(drawLoop);
            if (!ctx) return;
            ctx.fillStyle = 'rgba(0,0,0,0.2)'; ctx.fillRect(0,0,w,h);
            if (!isRecording && energy<2) return;
            
            let cx=w/2, cy=h/2; let r=50+energy*1.5;
            ctx.beginPath();
            ctx.strokeStyle = gateOpen ? `hsl(${energy*3},100%,60%)` : '#333';
            ctx.lineWidth = 3; ctx.arc(cx, cy, r, 0, Math.PI*2); ctx.stroke();
            
            if(gateOpen) {
                ctx.beginPath(); ctx.fillStyle='#fff';
                let off = Math.sin(Date.now()/200)*centroid;
                ctx.arc(cx+off, cy-off, 5, 0, Math.PI*2); ctx.fill();
            }
        }

        function resizeCanvas() {
            w = canvas.width = canvas.parentElement.clientWidth;
            h = canvas.height = canvas.parentElement.clientHeight;
        }
    </script>
</body>
</html>
