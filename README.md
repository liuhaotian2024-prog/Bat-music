<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Bat-music</title>
    <style>
        /* === Bat-music 艺术视觉风格 === */
        body {
            font-family: 'Courier New', monospace;
            background-color: #0f0014; /* 深邃紫夜色 */
            color: #d8b4fe;
            margin: 0;
            height: 100vh;
            display: flex;
            flex-direction: column;
            overflow: hidden;
        }

        /* 1. 核心视觉区：沉浸式画布 */
        .stage {
            flex: 1;
            position: relative;
            background: radial-gradient(circle at bottom, #2a0055 0%, #000 80%);
            display: flex;
            justify-content: center;
            align-items: center;
        }
        canvas {
            position: absolute;
            top: 0; left: 0;
            width: 100%; height: 100%;
        }

        /* 2. 抬头显示 (HUD) */
        .hud {
            position: absolute;
            top: 20px; left: 20px;
            pointer-events: none;
            text-shadow: 0 0 10px #d4aaff;
            z-index: 10;
        }
        .brand {
            font-size: 24px;
            font-weight: bold;
            color: #fff;
            letter-spacing: 2px;
            margin-bottom: 5px;
        }
        .info { font-size: 12px; opacity: 0.8; }

        /* 3. 中央音符显示 */
        .current-note {
            position: relative;
            z-index: 5;
            font-size: 80px;
            font-weight: 900;
            color: rgba(255, 255, 255, 0.1);
            transition: all 0.1s;
        }

        /* 4. 底部控制台 */
        .controls {
            padding: 20px;
            background: #1a0524;
            border-top: 1px solid #440066;
            text-align: center;
            /* 适配 iPhone 底部安全区 */
            padding-bottom: env(safe-area-inset-bottom, 20px);
        }

        button {
            background: linear-gradient(135deg, #b000ff, #0044ff);
            border: none;
            padding: 18px 40px;
            color: #fff;
            font-size: 16px;
            font-weight: bold;
            border-radius: 50px;
            box-shadow: 0 0 25px rgba(176, 0, 255, 0.4);
            cursor: pointer;
            width: 80%;
            max-width: 300px;
            text-transform: uppercase;
            letter-spacing: 1px;
        }
        button:active { transform: scale(0.96); opacity: 0.9; }

    </style>
</head>
<body>

    <div class="stage">
        <canvas id="canvas"></canvas>
        
        <div class="hud">
            <div class="brand">Bat-music</div>
            <div class="info">
                MODE: SLEEP JAZZ<br>
                FREQ: <span id="freqVal">--</span> Hz<br>
                GAIN: <span id="gainVal">0</span>%
            </div>
        </div>

        <div class="current-note" id="visualNote">♫</div>
    </div>

    <div class="controls">
        <button onclick="startMusic()" id="btnStart">启动 (Start Session)</button>
    </div>

    <script>
        // === 全局变量 ===
        let audioCtx, analyser, micSource;
        let osc, gainNode, delayNode, feedbackNode, filterNode;
        let isRunning = false;
        let wakeLock = null;

        // 画布相关
        let canvas, ctx, w, h;
        
        // --- 乐理核心：C小调五声音阶 (适合深夜的神秘感) ---
        // C3, Eb3, F3, G3, Bb3, C4, Eb4...
        const SCALE = [
            130.81, 155.56, 174.61, 196.00, 233.08, 
            261.63, 311.13, 349.23, 392.00, 466.16, 
            523.25, 622.25
        ];

        // === 1. 启动 Bat-music ===
        async function startMusic() {
            if (isRunning) return;
            
            try {
                // A. 屏幕常亮
                if ('wakeLock' in navigator) {
                    try { wakeLock = await navigator.wakeLock.request('screen'); } catch(e){}
                }

                audioCtx = new (window.AudioContext || window.webkitAudioContext)();

                // B. 构建合成器链路 (Synth Chain)
                // 1. 振荡器 (声音源) - 使用正弦波，最纯净，像玻璃琴
                osc = audioCtx.createOscillator();
                osc.type = 'sine'; 
                
                // 2. 音量门 (VCA)
                gainNode = audioCtx.createGain();
                gainNode.gain.value = 0; // 默认静音

                // 3. 效果器链：延时 (Delay) + 混响感
                delayNode = audioCtx.createDelay();
                delayNode.delayTime.value = 0.4; // 400ms 回声
                
                feedbackNode = audioCtx.createGain();
                feedbackNode.gain.value = 0.4; // 回声长短

                filterNode = audioCtx.createBiquadFilter();
                filterNode.type = 'lowpass'; // 让回声更暗淡，不抢主音
                filterNode.frequency.value = 1000;

                // C. 连接线路
                // 声音 -> 音量 -> 输出
                osc.connect(gainNode);
                gainNode.connect(audioCtx.destination); 
                
                // 声音 -> 音量 -> 延时 -> 滤波 -> 反馈 -> 回到延时 (形成回声) -> 输出
                gainNode.connect(delayNode);
                delayNode.connect(filterNode);
                filterNode.connect(feedbackNode);
                feedbackNode.connect(delayNode);
                delayNode.connect(audioCtx.destination);

                osc.start();

                // D. 麦克风输入 (Input)
                const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                analyser = audioCtx.createAnalyser();
                analyser.fftSize = 1024; // 适中的精度
                analyser.smoothingTimeConstant = 0.5; // 让视觉更柔和
                micSource = audioCtx.createMediaStreamSource(stream);
                micSource.connect(analyser);

                // E. 视觉初始化
                canvas = document.getElementById('canvas');
                ctx = canvas.getContext('2d');
                resizeCanvas();
                window.addEventListener('resize', resizeCanvas);

                // UI 更新
                document.getElementById('btnStart').style.display = 'none'; // 隐藏按钮，沉浸体验
                isRunning = true;
                
                // 启动循环
                loop(); 

            } catch (e) {
                alert("无法启动：请允许麦克风权限。Bat-music 需要聆听您的呼吸。");
            }
        }

        // === 2. 核心转换逻辑 ===
        function loop() {
            requestAnimationFrame(loop);
            
            // 获取音频数据
            const data = new Uint8Array(analyser.frequencyBinCount);
            analyser.getByteFrequencyData(data);
            
            // 计算当前总音量 (作为能量值)
            let sum = 0;
            for(let i=0; i<data.length; i++) sum += data[i];
            let vol = sum / data.length;

            // --- 音乐映射 (Mapping) ---
            // 阈值设为 10，过滤掉极其微小的底噪
            if (vol > 10) { 
                // 1. 音量 -> 音高
                // 呼噜声越大，音调越高。映射到 SCALE 数组中
                let index = Math.floor((vol / 80) * SCALE.length);
                if (index >= SCALE.length) index = SCALE.length - 1;
                let targetFreq = SCALE[index];

                // 2. 滑音 (Portamento)
                // 让音高变化平滑，不要太跳跃，像大提琴一样
                osc.frequency.setTargetAtTime(targetFreq, audioCtx.currentTime, 0.2);
                
                // 3. 音量 -> 增益
                // 限制最大音量为 0.6，保护听力
                let targetGain = Math.min(vol/120, 0.6);
                gainNode.gain.setTargetAtTime(targetGain, audioCtx.currentTime, 0.1);

                // HUD 更新
                document.getElementById('freqVal').innerText = targetFreq.toFixed(0);
                document.getElementById('gainVal').innerText = (targetGain * 100).toFixed(0);

                // 视觉中心音符律动
                let noteEl = document.getElementById('visualNote');
                noteEl.style.opacity = 0.2 + (vol/100);
                noteEl.style.transform = `scale(${1 + vol/30})`;
                noteEl.style.color = `hsl(${260 + vol}, 100%, 70%)`; // 颜色随力度变亮

            } else {
                // 没声音时，缓慢静音 (Release)
                gainNode.gain.setTargetAtTime(0, audioCtx.currentTime, 1.0);
            }

            // --- 视觉渲染 ---
            drawVisuals(data);
        }

        // === 3. 艺术视觉渲染 ===
        function drawVisuals(data) {
            // 制造拖影效果 (Trails)
            ctx.fillStyle = 'rgba(15, 0, 20, 0.2)'; 
            ctx.fillRect(0, 0, w, h);

            let barWidth = (w / data.length) * 2;
            let x = 0;

            for(let i = 0; i < data.length; i++) {
                let val = data[i];
                let barHeight = val * 1.5;

                // 颜色算法：基于紫色的渐变
                // 底部深紫 -> 顶部亮粉
                let hue = 270 + (i / data.length) * 60; 
                let light = 30 + (val / 255) * 50;
                
                ctx.fillStyle = `hsl(${hue}, 80%, ${light}%)`;
                
                // 绘制镜像波形 (像湖面倒影)
                let centerY = h / 2;
                ctx.fillRect(x, centerY - barHeight/2, barWidth, barHeight);

                x += barWidth + 1;
            }
        }

        function resizeCanvas() {
            w = canvas.width = canvas.parentElement.clientWidth;
            h = canvas.height = canvas.parentElement.clientHeight;
        }

        // 唤醒锁维持
        document.addEventListener('visibilitychange', async () => {
            if (wakeLock !== null && document.visibilityState === 'visible') {
                try { wakeLock = await navigator.wakeLock.request('screen'); } catch(e){}
            }
        });
    </script>
</body>
</html>
