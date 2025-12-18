<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Bat-music: Functional Core</title>
    <style>
        /* === 视觉风格：量子场论黑 === */
        body {
            font-family: 'Menlo', 'Monaco', monospace;
            background-color: #000;
            color: #0f0;
            margin: 0;
            height: 100vh;
            display: flex;
            flex-direction: column;
            overflow: hidden;
        }

        /* 1. 动态场显示区 */
        .field-monitor {
            flex: 1;
            position: relative;
            background: #050505;
        }
        canvas { width: 100%; height: 100%; display: block; }
        
        .hud {
            position: absolute;
            top: 20px; left: 20px;
            pointer-events: none;
            background: rgba(0,0,0,0.7);
            padding: 10px;
            border: 1px solid #333;
        }
        .hud-item { margin-bottom: 5px; font-size: 11px; }
        .val-highlight { color: #fff; font-weight: bold; }
        
        /* 泛函计算指示器 */
        .math-overlay {
            position: absolute;
            bottom: 20px; right: 20px;
            text-align: right;
            font-size: 10px;
            color: #666;
            line-height: 1.4;
        }

        /* 2. 底部控制台 */
        .controls {
            padding: 20px;
            background: #111;
            border-top: 1px solid #222;
            display: flex;
            justify-content: center;
            padding-bottom: env(safe-area-inset-bottom, 20px);
        }

        button {
            background: #004400;
            border: 1px solid #0f0;
            color: #0f0;
            padding: 15px 30px;
            font-size: 14px;
            text-transform: uppercase;
            letter-spacing: 1px;
            cursor: pointer;
            width: 100%;
            max-width: 400px;
        }
        button:active { background: #006600; color: #fff; }

        /* 状态灯 */
        .indicator {
            display: inline-block; width: 8px; height: 8px; border-radius: 50%; background: #333; margin-right: 5px;
        }
        .indicator.active { background: #0f0; box-shadow: 0 0 10px #0f0; }

    </style>
</head>
<body>

    <div class="field-monitor">
        <canvas id="canvas"></canvas>
        
        <div class="hud">
            <div class="hud-item"><span class="indicator" id="gateLed"></span> GATE: <span id="gateState" class="val-highlight">CLOSED</span></div>
            <div class="hud-item">TIMBRE (C): <span id="timbreVal" class="val-highlight">--</span></div>
            <div class="hud-item">INSTRUMENT: <span id="instName" style="color:#0af">--</span></div>
            <div class="hud-item">ENERGY (E): <span id="energyVal">0</span></div>
        </div>

        <div class="math-overlay">
            Bat-music v2.0 Kernel<br>
            Y* Calculation: Functional Harmonic Path<br>
            Δ(t) -> Pitch Correction & LFO
        </div>
    </div>

    <div class="controls">
        <button onclick="initEngine()" id="btnStart">Initialize Field (启动场)</button>
    </div>

    <script>
        // === 1. 物理引擎与全局变量 ===
        let audioCtx, analyser, micSource;
        let isRunning = false;
        let wakeLock = null;

        // 合成器组件 (Synthesizer Nodes)
        let osc, filter, gainNode, lfo, delay, feedback;
        
        // CIEU 状态变量
        let energy = 0;         // 能量
        let centroid = 0;       // 频谱质心 (决定音色)
        let targetFreq = 0;     // Y* (目标频率)
        let currentFreq = 0;    // 当前实际频率
        let gateOpen = false;   // 门限状态

        // 画布
        let canvas, ctx, w, h;
        let particles = []; // 可视化粒子

        // --- 音乐理论：C Dorian Scale (神秘、爵士感) ---
        // 频率表: C3, D3, Eb3, F3, G3, A3, Bb3, C4...
        const SCALE = [
            130.81, 146.83, 155.56, 174.61, 196.00, 220.00, 233.08, 
            261.63, 293.66, 311.13, 349.23, 392.00, 440.00, 466.16, 
            523.25
        ];

        // === 2. 泛函初始化 ===
        async function initEngine() {
            if (isRunning) return;
            try {
                if ('wakeLock' in navigator) { try { wakeLock = await navigator.wakeLock.request('screen'); } catch(e){} }

                audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                
                // --- A. 合成器架构 (The Instrument) ---
                
                // 1. 振荡器 (源)
                osc = audioCtx.createOscillator();
                osc.type = 'sine'; // 初始波形
                
                // 2. 滤波器 (塑形) - 低通滤波
                filter = audioCtx.createBiquadFilter();
                filter.type = 'lowpass';
                filter.Q.value = 5; // 共振峰，增加电子感

                // 3. LFO (颤音/张力) - 用于表现 Δ 误差
                lfo = audioCtx.createOscillator();
                lfo.type = 'sine';
                lfo.frequency.value = 0; // 初始无颤音
                let lfoGain = audioCtx.createGain();
                lfoGain.gain.value = 10; // 颤音深度
                lfo.connect(lfoGain);
                lfoGain.connect(osc.frequency); // 调制音高
                lfo.start();

                // 4. 音量包络 (ADSR Gate)
                gainNode = audioCtx.createGain();
                gainNode.gain.value = 0;

                // 5. 空间效果 (Delay)
                delay = audioCtx.createDelay();
                delay.delayTime.value = 0.4;
                feedback = audioCtx.createGain();
                feedback.gain.value = 0.3;

                // 连线
                osc.connect(filter);
                filter.connect(gainNode);
                gainNode.connect(audioCtx.destination); // 干声
                gainNode.connect(delay);                // 湿声
                delay.connect(feedback);
                feedback.connect(delay);
                delay.connect(audioCtx.destination);

                osc.start();

                // --- B. 传感器架构 (The Ear) ---
                const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                analyser = audioCtx.createAnalyser();
                analyser.fftSize = 2048; // 高精度 FFT 以计算质心
                analyser.smoothingTimeConstant = 0.3;
                micSource = audioCtx.createMediaStreamSource(stream);
                micSource.connect(analyser);

                // --- C. 视觉 ---
                canvas = document.getElementById('canvas');
                ctx = canvas.getContext('2d');
                resizeCanvas();
                window.addEventListener('resize', resizeCanvas);

                // UI
                document.getElementById('btnStart').style.display = 'none';
                isRunning = true;
                
                // 启动 CIEU 循环
                computeLoop();

            } catch (e) {
                alert("启动失败: 请检查麦克风权限。" + e);
            }
        }

        // === 3. 泛函动态计算内核 (Core Loop) ===
        function computeLoop() {
            requestAnimationFrame(computeLoop);
            
            // 1. 获取频域数据
            const bufferLength = analyser.frequencyBinCount;
            const dataArray = new Uint8Array(bufferLength);
            analyser.getByteFrequencyData(dataArray);

            // 2. 计算五元组参数
            // E (Energy): 总体积
            let sum = 0;
            let weightedSum = 0;
            for(let i = 0; i < bufferLength; i++) {
                sum += dataArray[i];
                weightedSum += i * dataArray[i];
            }
            energy = sum / bufferLength;
            
            // C (Centroid): 频谱质心 = 加权平均频率 / 总能量
            // 质心决定了声音是“闷”(低) 还是 “亮”(高)
            centroid = sum > 0 ? weightedSum / sum : 0;

            // --- 动态决策层 ---

            // A. 门限判定 (Gate Logic)
            // 只有能量 > 10 才启动，否则进入 Release 状态 (解决"停不下来"的问题)
            if (energy > 15) {
                if (!gateOpen) {
                    // Attack (起音)
                    gainNode.gain.cancelScheduledValues(audioCtx.currentTime);
                    gainNode.gain.linearRampToValueAtTime(0.5, audioCtx.currentTime + 0.1);
                    gateOpen = true;
                    updateHUD(true);
                }
                
                // B. 音色映射 (Timbre Mapping) - 不同的怪声对应不同乐器
                // 质心 < 20: 呼噜/深沉 -> 正弦波 (Bass)
                // 质心 20-60: 人声/哼唱 -> 三角波 (Cello)
                // 质心 > 60: 尖叫/敲击 -> 锯齿波 (Synth Lead)
                let type = 'sine';
                let instName = 'DEEP BASS';
                let cutoffBase = 500;

                if (centroid > 60) { 
                    type = 'sawtooth'; 
                    instName = 'ACID LEAD'; 
                    cutoffBase = 2000;
                } else if (centroid > 20) { 
                    type = 'triangle'; 
                    instName = 'ELECTRIC CELLO'; 
                    cutoffBase = 1000;
                }

                if (osc.type !== type) osc.type = type;

                // C. Y* 寻路 (Target Finding)
                // 能量越大，音高越高。将能量映射到音阶索引。
                let noteIndex = Math.floor((energy / 60) * SCALE.length);
                if (noteIndex >= SCALE.length) noteIndex = SCALE.length - 1;
                targetFreq = SCALE[noteIndex];

                // D. 路径积分 (Smooth Transition)
                // 使用指数平滑逼近 Y*
                osc.frequency.setTargetAtTime(targetFreq, audioCtx.currentTime, 0.2);

                // E. Δ 误差表现 (Tension)
                // 滤波器开度随能量动态变化
                filter.frequency.setTargetAtTime(cutoffBase + (energy * 20), audioCtx.currentTime, 0.1);
                
                // 如果声音极其不稳定(质心抖动大)，增加 LFO 颤音
                lfo.frequency.value = (centroid / 10); 

            } else {
                if (gateOpen) {
                    // Release (释音) - 优雅淡出
                    gainNode.gain.cancelScheduledValues(audioCtx.currentTime);
                    gainNode.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 1.0); // 1秒长尾
                    gateOpen = false;
                    updateHUD(false);
                }
            }

            // 更新数据显示
            document.getElementById('energyVal').innerText = energy.toFixed(1);
            document.getElementById('timbreVal').innerText = centroid.toFixed(1);
            document.getElementById('instName').innerText = gateOpen ? instName : "--";

            // 渲染视觉
            drawField(dataArray);
        }

        function updateHUD(isOpen) {
            const led = document.getElementById('gateLed');
            const txt = document.getElementById('gateState');
            if (isOpen) {
                led.className = "indicator active";
                txt.innerText = "OPEN (PLAYING)";
                txt.style.color = "#0f0";
            } else {
                led.className = "indicator";
                txt.innerText = "CLOSED (IDLE)";
                txt.style.color = "#666";
            }
        }

        // === 4. 波形场可视化 ===
        function drawField(data) {
            ctx.fillStyle = 'rgba(0, 0, 0, 0.2)'; // 拖影残像
            ctx.fillRect(0, 0, w, h);

            if (!gateOpen && energy < 5) return;

            let cx = w / 2;
            let cy = h / 2;
            let radius = 50 + energy;

            ctx.beginPath();
            ctx.strokeStyle = gateOpen ? `hsl(${centroid * 4}, 100%, 50%)` : '#333';
            ctx.lineWidth = 2;

            // 绘制极坐标波形 (像一只眼睛或雷达)
            for (let i = 0; i < data.length; i+=5) {
                let val = data[i];
                let angle = (i / data.length) * Math.PI * 2;
                
                // 半径随频谱能量波动
                let r = radius + val; 
                let x = cx + Math.cos(angle) * r;
                let y = cy + Math.sin(angle) * r;

                if (i === 0) ctx.moveTo(x, y);
                else ctx.lineTo(x, y);
            }
            ctx.closePath();
            ctx.stroke();

            // 绘制 Y* 目标核心
            ctx.beginPath();
            ctx.fillStyle = '#fff';
            ctx.arc(cx, cy, energy / 5, 0, Math.PI * 2);
            ctx.fill();
        }

        function resizeCanvas() {
            w = canvas.width = canvas.parentElement.clientWidth;
            h = canvas.height = canvas.parentElement.clientHeight;
        }

        document.addEventListener('visibilitychange', async () => {
            if (wakeLock !== null && document.visibilityState === 'visible') {
                try { wakeLock = await navigator.wakeLock.request('screen'); } catch(e){}
            }
        });
    </script>
</body>
</html>
