<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Bat-music: Quantum Composer</title>
    <style>
        /* === 视觉：量子网格 === */
        body {
            font-family: 'Courier New', monospace;
            background: #000;
            color: #0f0;
            margin: 0;
            height: 100vh;
            display: flex;
            flex-direction: column;
            overflow: hidden;
        }

        /* 1. 核心舞台：音乐场 */
        .grid-stage {
            flex: 1;
            position: relative;
            background: #050505;
            background-image: 
                linear-gradient(rgba(0, 255, 0, 0.1) 1px, transparent 1px),
                linear-gradient(90deg, rgba(0, 255, 0, 0.1) 1px, transparent 1px);
            background-size: 40px 40px;
            overflow: hidden;
        }
        canvas { width: 100%; height: 100%; display: block; }

        /* 顶部信息 */
        .top-hud {
            position: absolute;
            top: 0; left: 0; width: 100%;
            padding: 15px;
            box-sizing: border-box;
            display: flex;
            justify-content: space-between;
            pointer-events: none;
            text-shadow: 0 0 5px #0f0;
        }
        .beat-indicator {
            width: 10px; height: 10px; background: #333; border-radius: 50%;
        }
        .beat-indicator.active { background: #0f0; box-shadow: 0 0 10px #0f0; }

        /* 乐理参数 */
        .theory-hud {
            position: absolute;
            bottom: 15px; left: 15px;
            font-size: 10px;
            color: #0a0;
            background: rgba(0,0,0,0.8);
            padding: 5px;
            border: 1px solid #040;
        }

        /* 2. 控制台 */
        .controls {
            background: #111;
            padding: 20px;
            padding-bottom: env(safe-area-inset-bottom, 20px);
            border-top: 2px solid #040;
            display: flex;
            flex-direction: column;
            gap: 15px;
        }

        /* 播放器提示 */
        .download-tip {
            font-size: 12px; color: #888; text-align: center; display: none; margin-bottom: 5px;
        }

        /* 2x2 按钮网格 */
        .btn-grid {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 12px;
        }

        button {
            background: #000;
            color: #0f0;
            border: 1px solid #0f0;
            padding: 15px;
            font-family: inherit;
            font-weight: bold;
            font-size: 14px;
            cursor: pointer;
            text-transform: uppercase;
            transition: all 0.1s;
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 8px;
        }
        button:active { background: #0f0; color: #000; }
        button:disabled { border-color: #333; color: #333; pointer-events: none; }

        /* 特殊按钮色 */
        .btn-stop { border-color: #f00; color: #f00; }
        .btn-stop:active { background: #f00; color: #fff; }
        .btn-share { border-color: #0af; color: #0af; }
        .btn-share:active { background: #0af; color: #000; }

    </style>
</head>
<body>

    <div class="grid-stage">
        <canvas id="canvas"></canvas>
        
        <div class="top-hud">
            <div>QUANTUM COMPOSER v5.0</div>
            <div class="beat-indicator" id="beatLed"></div>
        </div>

        <div class="theory-hud">
            <div>Y* FIELD: C-MINOR JAZZ</div>
            <div>LAYER: <span id="layerName">BASE</span></div>
            <div>EVENTS: <span id="eventCount">0</span></div>
        </div>
    </div>

    <div class="controls">
        <div class="download-tip" id="dlTip">⚠️ 微信用户请点击“存储”后选择“保存到文件”</div>
        
        <audio id="hiddenPlayer" style="display:none"></audio>

        <div class="btn-grid">
            <button id="btnStart" onclick="startComposition()">
                <span>▶ 叠加录制</span>
            </button>
            <button id="btnStop" class="btn-stop" onclick="stopComposition()" disabled>
                <span>⏹ 生成乐曲</span>
            </button>
            <button id="btnSave" onclick="saveFile()" disabled>
                <span>💾 存储文件</span>
            </button>
            <button id="btnReset" class="btn-share" onclick="resetAll()">
                <span>🔄 清空重来</span>
            </button>
        </div>
    </div>

    <script>
        // === 核心变量 ===
        let audioCtx;
        let isRunning = false;
        let isRecording = false;
        let wakeLock = null;

        // 音序器 (The Sequencer)
        let bpm = 110;
        let nextNoteTime = 0;
        let noteQueue = []; // 待播放队列
        let beatCount = 0;

        // 录音机
        let destNode, mediaRecorder;
        let chunks = [];
        let blobUrl = null;

        // 场叠加层 (Loops)
        // 我们不录声音，我们录"事件"。
        // { time: 0.0, type: 'kick', note: 200 }
        let patternLayers = []; 
        let currentLayer = [];
        
        // 乐理：泛函 Y*
        // C Minor Pentatonic + Blue Note
        const SCALE = [130.81, 155.56, 174.61, 185.00, 196.00, 233.08, 261.63, 311.13, 349.23, 392.00];

        // 信号分析
        let analyser, micSource;
        let lastEnergy = 0;
        let triggerCooldown = 0;

        // 绘图
        let canvas, ctx, w, h;
        let visuals = [];

        // === 1. 启动与叠加 (Layering) ===
        async function startComposition() {
            try {
                if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                if (audioCtx.state === 'suspended') await audioCtx.resume();
                
                // 唤醒锁
                if ('wakeLock' in navigator) { try { wakeLock = await navigator.wakeLock.request('screen'); } catch(e){} }

                // 如果是第一次启动，建立基础
                if (!isRunning) {
                    await initAudioChain();
                    // 启动时钟
                    nextNoteTime = audioCtx.currentTime + 0.1;
                    requestAnimationFrame(scheduler);
                    
                    // 启动录音机 (录制总输出)
                    startMasterRecorder();
                    
                    isRunning = true;
                }

                // 开启"录入模式"
                isRecording = true;
                currentLayer = []; // 准备新的一层
                
                // UI
                document.getElementById('btnStart').innerText = "🔴 正在叠加...";
                document.getElementById('btnStart').disabled = true;
                document.getElementById('btnStop').disabled = false;
                document.getElementById('layerName').innerText = `LAYER ${patternLayers.length + 1}`;

            } catch (e) {
                alert("启动失败: " + e);
            }
        }

        // === 2. 停止与生成 (Compose) ===
        function stopComposition() {
            isRecording = false;
            
            // 将当前录入的事件合并到主循环中
            if (currentLayer.length > 0) {
                patternLayers.push(currentLayer);
            }

            // 停止总录音机
            if (mediaRecorder && mediaRecorder.state === 'recording') {
                mediaRecorder.stop();
            }

            // UI
            document.getElementById('btnStart').innerText = "▶ 再次叠加";
            document.getElementById('btnStart').disabled = false;
            document.getElementById('btnStop').disabled = true;
            document.getElementById('btnSave').disabled = false;
            
            document.getElementById('layerName').innerText = "PLAYBACK";
        }

        // === 3. 音频链与录音 ===
        async function initAudioChain() {
            // 麦克风输入 (仅用于触发，不直接听到)
            const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
            micSource = audioCtx.createMediaStreamSource(stream);
            analyser = audioCtx.createAnalyser();
            analyser.fftSize = 512;
            micSource.connect(analyser);

            // 总线 (Master Bus)
            destNode = audioCtx.createMediaStreamDestination();
            
            // 绘图
            canvas = document.getElementById('canvas');
            ctx = canvas.getContext('2d');
            resize();
            window.addEventListener('resize', resize);
            drawLoop();
            analyzeLoop(); // 启动信号监听
        }

        function startMasterRecorder() {
            chunks = [];
            // 兼容性
            let mime = MediaRecorder.isTypeSupported('audio/webm') ? 'audio/webm' : 'audio/mp4';
            mediaRecorder = new MediaRecorder(destNode.stream, { mimeType: mime });
            
            mediaRecorder.ondataavailable = e => { if (e.data.size > 0) chunks.push(e.data); };
            
            mediaRecorder.onstop = () => {
                let blob = new Blob(chunks, { type: mediaRecorder.mimeType });
                if (blobUrl) URL.revokeObjectURL(blobUrl);
                blobUrl = URL.createObjectURL(blob);
                
                // 可以在这里自动播放预览
                let p = document.getElementById('hiddenPlayer');
                p.src = blobUrl;
                // p.play(); 
            };
            
            mediaRecorder.start();
        }

        // === 4. 信号切片与触发 (The Slicer) ===
        function analyzeLoop() {
            requestAnimationFrame(analyzeLoop);
            if (!isRecording) return;

            let data = new Uint8Array(analyser.frequencyBinCount);
            analyser.getByteFrequencyData(data);

            // 计算瞬时能量
            let sum = 0;
            let weightedSum = 0; // 用于计算质心
            for(let i=0; i<data.length; i++) {
                sum += data[i];
                weightedSum += i * data[i];
            }
            let energy = sum / data.length;
            let centroid = sum > 0 ? weightedSum / sum : 0;

            // 瞬态检测 (Transient Detection)
            // 能量突然增加，且超过阈值
            if (triggerCooldown > 0) triggerCooldown--;

            if (energy > 20 && (energy - lastEnergy) > 5 && triggerCooldown <= 0) {
                // >>> 触发事件 <<<
                
                // 1. 量化时间 (Quantize Time)
                // 记录相对于当前小节的位置
                // 简化处理：直接记录触发，但在播放时量化
                
                // 2. 泛函计算 Y* (确定乐器和音高)
                let type, note;
                
                if (centroid < 20) {
                    type = 'kick'; // 低沉呼噜 -> 鼓
                    note = 50;
                } else if (centroid < 50) {
                    type = 'bass'; // 中频哼唱 -> 贝斯
                    note = SCALE[Math.floor((energy/100)*4)]; // 低音区
                } else {
                    type = 'synth'; // 高频怪声 -> 旋律
                    let idx = Math.floor((energy/80) * SCALE.length);
                    note = SCALE[Math.min(idx, SCALE.length-1)];
                }

                // 3. 立即演奏 (Feedback)
                playNote(type, note, audioCtx.currentTime);
                
                // 4. 记录到层 (Record)
                current
