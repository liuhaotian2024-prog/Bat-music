<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Bat-music: AI Composer</title>
    <style>
        /* === 视觉：深空指挥台 === */
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background: #000;
            color: #fff;
            margin: 0;
            height: 100vh;
            display: flex;
            flex-direction: column;
            overflow: hidden;
        }

        /* 1. 上半部：AI 编曲可视化 */
        .stage {
            flex: 1;
            position: relative;
            background: linear-gradient(180deg, #1a0b2e 0%, #000 100%);
            border-bottom: 1px solid #333;
        }
        canvas { width: 100%; height: 100%; display: block; }

        /* 状态信息 */
        .header-info {
            position: absolute;
            top: 0; left: 0; width: 100%;
            padding: 15px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            pointer-events: none;
            box-sizing: border-box;
            z-index: 10;
        }
        .mode-badge {
            background: rgba(255,255,255,0.1);
            padding: 4px 10px; border-radius: 4px; font-size: 11px;
            color: #aaa; border: 1px solid #333;
            backdrop-filter: blur(4px);
        }
        .timer { font-family: 'Monaco', monospace; font-size: 16px; font-weight: bold; color: #666; }
        .timer.active { color: #f00; text-shadow: 0 0 10px #f00; }

        /* 乐理参数 HUD */
        .theory-hud {
            position: absolute;
            bottom: 15px; left: 15px;
            font-family: 'Monaco', monospace;
            font-size: 10px;
            color: #666;
            line-height: 1.6;
            pointer-events: none;
        }
        .val { color: #d0f; font-weight: bold; }
        .chord { color: #0af; font-weight: bold; font-size: 14px; }

        /* 2. 下半部：控制面板 (2x2 Grid) */
        .control-deck {
            background: #111;
            padding: 20px;
            padding-bottom: env(safe-area-inset-bottom, 20px);
            border-top: 1px solid #222;
        }

        /* 播放器容器 */
        .player-wrapper {
            height: 0; opacity: 0;
            transition: all 0.3s;
            background: #222;
            border-radius: 8px;
            margin-bottom: 15px;
            overflow: hidden;
        }
        .player-wrapper.show { height: 50px; opacity: 1; border: 1px solid #444; }
        audio { width: 100%; height: 100%; outline: none; }

        /* 按钮网格 */
        .grid {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 12px;
        }

        button {
            border: none;
            border-radius: 12px;
            padding: 18px 5px;
            font-size: 14px;
            font-weight: bold;
            color: #fff;
            background: #222;
            cursor: pointer;
            transition: all 0.2s;
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 6px;
            opacity: 0.4;
            pointer-events: none;
        }
        
        /* 激活状态 */
        button.active { opacity: 1; pointer-events: auto; }
        
        #btnStart.active { background: #007aff; }
        #btnStop.active { background: #ff3b30; box-shadow: 0 0 20px rgba(255, 59, 48, 0.2); }
        #btnSave.active { background: #30d158; color: #000; }
        #btnShare.active { background: #5e5ce6; }

        /* 图标 */
        i { font-style: normal; font-size: 18px; }

    </style>
</head>
<body>

    <div class="stage">
        <canvas id="canvas"></canvas>
        
        <div class="header-info">
            <div class="mode-badge">CIEU: JAZZ ENSEMBLE</div>
            <div class="timer" id="timer">00:00</div>
        </div>

        <div class="theory-hud">
            <div>HARMONY: <span id="hudChord" class="chord">--</span></div>
            <div>MELODY (Y*): <span id="hudNote" class="val">--</span></div>
            <div>DYNAMICS (Δ): <span id="hudDelta" class="val">0%</span></div>
        </div>
    </div>

    <div class="control-deck">
        <div class="player-wrapper" id="playerBox">
            <audio id="audioPlayer" controls playsinline></audio>
        </div>

        <div class="grid">
            <button id="btnStart" class="active" onclick="startSession()">
                <i>▶</i> 开始创作
            </button>
            <button id="btnStop" onclick="stopSession()">
                <i>⏹</i> 完成乐曲
            </button>
            <button id="btnSave" onclick="saveFile()">
                <i>💾</i> 存储文件
            </button>
            <button id="btnShare" onclick="shareFile()">
                <i>🔗</i> 分享
            </button>
        </div>
    </div>

    <script>
        // === 全局状态 ===
        let audioCtx;
        let isRecording = false;
        let startTime = 0, timerInt;
        let wakeLock = null;

        // 录音相关
        let mediaRecorder;
        let audioChunks = [];
        let blobUrl = null;
        let finalBlob = null;
        let destNode; // 总线节点

        // --- 因果AI作曲引擎 (The Causal Composer) ---
        // 乐理基础：C Minor Dorian (爵士感)
        const SCALE = [130.81, 155.56, 174.61, 196.00, 233.08, 261.63, 311.13, 349.23, 392.00, 466.16];
        // 和弦进行：Cm7 - Fm7 - Gm7 - BbMaj7 (循环)
        const CHORDS = [
            [130.81, 155.56, 196.00, 233.08], // Cm7
            [174.61, 207.65, 261.63, 311.13], // Fm7
            [196.00, 233.08, 293.66, 349.23], // Gm7
            [233.08, 293.66, 349.23, 415.30]  // BbMaj7
        ];
        const CHORD_NAMES = ["Cm7 (i)", "Fm7 (iv)", "Gm7 (v)", "BbMaj7 (VII)"];

        // 乐器组件
        let padOscs = [];    // 和声铺底 (4个振荡器)
        let padGain;
        let leadOsc;         // 旋律 (你的声音)
        let leadFilter;
        let leadGain;
        let bassOsc;         // 低音
        let bassGain;
        
        let analyser, micSource;
        
        // 动态变量
        let energy = 0;      // 能量
        let chordIndex = 0;  // 当前和弦索引
        let progressionTimer = 0; // 换和弦计时器

        // 绘图
        let canvas, ctx, w, h;
        let visualNotes = []; // 视觉粒子

        // === 1. 开始创作 (Start) ===
        async function startSession() {
            try {
                // UI 锁定
                toggleBtn('btnStart', false);
                toggleBtn('btnStop', true);
                toggleBtn('btnSave', false);
                toggleBtn('btnShare', false);
                document.getElementById('playerBox').classList.remove('show');
                document.getElementById('timer').classList.add('active');

                // 唤醒锁
                if ('wakeLock' in navigator) { try { wakeLock = await navigator.wakeLock.request('screen'); } catch(e){} }

                // 初始化音频上下文
                if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                if (audioCtx.state === 'suspended') await audioCtx.resume();

                // 初始化录音总线
                destNode = audioCtx.createMediaStreamDestination();
                audioChunks = [];

                // 启动引擎 (包含乐器创建和连线)
                await initComposer();

                // 启动录音机 (关键修正：录制 destNode)
                let mime = MediaRecorder.isTypeSupported('audio/webm') ? 'audio/webm' : 'audio/mp4';
                mediaRecorder = new MediaRecorder(destNode.stream, { mimeType: mime });
                
                mediaRecorder.ondataavailable = e => {
                    if (e.data.size > 0) audioChunks.push(e.data);
                };
                mediaRecorder.start();

                // 计时器
                startTime = Date.now();
                timerInt = setInterval(() => {
                    let s = Math.floor((Date.now() - startTime)/1000);
                    let m = Math.floor(s/60).toString().padStart(2,'0');
                    s = (s%60).toString().padStart(2,'0');
                    document.getElementById('timer').innerText = `${m}:${s}`;
                }, 1000);

                isRecording = true;
                composerLoop(); // 启动AI作曲循环

            } catch (e) {
                alert("启动失败: " + e);
                resetUI();
            }
        }

        // === 2. 完成乐曲 (Stop) ===
        function stopSession() {
            if (!mediaRecorder) return;

            // 优雅淡出 (Fade Out)
            let now = audioCtx.currentTime;
            padGain.gain.setTargetAtTime(0, now, 0.5);
            leadGain.gain.setTargetAtTime(0, now, 0.5);
            bassGain.gain.setTargetAtTime(0, now, 0.5);

            // 延迟停止录音，确保尾音被录入
            setTimeout(() => {
                mediaRecorder.stop();
                isRecording = false;
                clearInterval(timerInt);
                
                // 停止振荡器
                padOscs.forEach(o => o.stop());
                leadOsc.stop();
                bassOsc.stop();

                document.getElementById('timer').classList.remove('active');
                toggleBtn('btnStop', false);

                mediaRecorder.onstop = () => {
                    finalBlob = new Blob(audioChunks, { type: mediaRecorder.mimeType });
                    if (blobUrl) URL.revokeObjectURL(blobUrl);
                    blobUrl = URL.createObjectURL(finalBlob);

                    // 加载播放器
                    let player = document.getElementById('audioPlayer');
                    player.src = blobUrl;
                    document.getElementById('playerBox').classList.add('show');

                    // 启用分享
                    toggleBtn('btnStart', true); // 允许重录
                    toggleBtn('btnSave', true);
                    toggleBtn('btnShare', true);
                };
            }, 1000); // 等待1秒尾音
        }

        // === 3. 存储 (Save) ===
        function saveFile() {
            if (!blobUrl) return;
            let a = document.createElement('a');
            a.href = blobUrl;
            let time = new Date().toISOString().slice(11,16).replace(':','');
            a.download = `Bat-Music_Jazz_${time}.webm`; // 默认 webm
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
        }

        // === 4. 分享 (Share) ===
        async function shareFile() {
            if (!finalBlob) return;
            
            // 尝试调用原生分享
            let file = new File([finalBlob], "bat-jazz.webm", { type: finalBlob.type });
            
            if (navigator.canShare && navigator.canShare({ files: [file] })) {
                try {
                    await navigator.share({
                        files: [file],
                        title: 'Bat-music Composition',
                        text: 'Listen to my AI-generated sleep jazz!'
                    });
                } catch (e) { console.log('Share canceled'); }
            } else {
                alert("微信或当前浏览器不支持直接分享。请点击“存储”保存文件后手动发送。");
            }
        }

        // === 核心引擎：AI Composer ===
        async function initComposer() {
            // 1. 创建麦克风输入
            const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
            micSource = audioCtx.createMediaStreamSource(stream);
            analyser = audioCtx.createAnalyser();
            analyser.fftSize = 1024;
            micSource.connect(analyser);

            // 2. 音频路由总线 (Master Bus)
            // 所有乐器 -> Master Gain -> (Destination & Recorder)
            let masterGain = audioCtx.createGain();
            masterGain.gain.value = 0.8; 
            masterGain.connect(audioCtx.destination); // 监听
            masterGain.connect(destNode);             // 录音

            // 3. 乐器一：和声铺底 (The Field)
            // 4个振荡器组成一个和弦
            padGain = audioCtx.createGain();
            padGain.gain.value = 0; // 初始静音，随呼吸渐入
            padGain.connect(masterGain);
            
            padOscs = [];
            for(let i=0; i<4; i++) {
                let o = audioCtx.createOscillator();
                o.type = 'sine'; // 柔和的 Pad
                o.frequency.value = CHORDS[0][i]; // 初始和弦
                o.connect(padGain);
                o.start();
                padOscs.push(o);
            }

            // 4. 乐器二：低音 (Bass)
            bassGain = audioCtx.createGain();
            bassGain.gain.value = 0.3;
            bassGain.connect(masterGain);
            bassOsc = audioCtx.createOscillator();
            bassOsc.type = 'triangle'; // 有质感的低音
            bassOsc.frequency.value = CHORDS[0][0] / 2; // 根音低八度
            bassOsc.connect(bassGain);
            bassOsc.start();

            // 5. 乐器三：旋律 (The Causal Lead)
            // 你的声音转化为主奏
            leadGain = audioCtx.createGain();
            leadGain.gain.value = 0;
            
            // 增加混响/延时效果
            let delay = audioCtx.createDelay();
            delay.delayTime.value = 0.35;
            let fb = audioCtx.createGain();
            fb.gain.value = 0.4;
            delay.connect(fb);
            fb.connect(delay);
            
            leadGain.connect(masterGain);
            leadGain.connect(delay); // 发送至延时
            delay.connect(masterGain);

            leadOsc = audioCtx.createOscillator();
            leadOsc.type = 'sawtooth'; // 类似小号/萨克斯
            
            leadFilter = audioCtx.createBiquadFilter();
            leadFilter.type = 'lowpass';
            leadFilter.Q.value = 3; // 增加共振，更有电子味

            leadOsc.connect(leadFilter);
            leadFilter.connect(leadGain);
            leadOsc.start();

            // 视觉初始化
            canvas = document.getElementById('canvas');
            ctx = canvas.getContext('2d');
            resizeCanvas();
            window.addEventListener('resize', resizeCanvas);
            drawLoop();
        }

        // === 作曲循环 (The Loop) ===
        function composerLoop() {
            if (!isRecording) return;
            requestAnimationFrame(composerLoop);

            // 获取感知数据
            let data = new Uint8Array(analyser.frequencyBinCount);
            analyser.getByteFrequencyData(data);
            
            let sum = 0;
            for(let i=0; i<data.length; i++) sum += data[i];
            energy = sum / data.length; // 0 - 255

            let now = audioCtx.currentTime;

            // --- 逻辑层：泛函和声驱动 ---
            
            // 1. 和声推进 (CIEU Progression)
            // 如果能量持续积累，推动和弦变化
            if (energy > 20) {
                progressionTimer++;
                // 呼吸积累到一定程度，切换下一个和弦
                if (progressionTimer > 200) { // 约3-4秒换一次
                    chordIndex = (chordIndex + 1) % CHORDS.length;
                    changeChord(chordIndex, now);
                    progressionTimer = 0;
                }
                
                // 和声淡入 (Pad Swell)
                padGain.gain.setTargetAtTime(0.3, now, 0.5);
            } else {
                // 安静时，和声淡出但不断
                padGain.gain.setTargetAtTime(0.05, now, 1.0);
            }

            // 2. 旋律生成 (Causal Melody)
            if (energy > 15) {
                // 将能量映射为音阶 (Quantization)
                // 能量越大，音越高，但必须落在 C Dorian 音阶内
                let noteIdx = Math.floor((energy / 60) * SCALE.length);
                if (noteIdx >= SCALE.length) noteIdx = SCALE.length - 1;
                let targetFreq = SCALE[noteIdx];

                // 平滑变调 (Portamento) - 像爵士滑音
                leadOsc.frequency.setTargetAtTime(targetFreq, now, 0.1);
                
                // 滤波器跟随力度 (Dynamics)
                leadFilter.frequency.setTargetAtTime(300 + energy * 20, now, 0.1);
                
                // 音量包络
                leadGain.gain.setTargetAtTime(Math.min(energy/200, 0.4), now, 0.1);

                // HUD 更新
                document.getElementById('hudNote').innerText = targetFreq.toFixed(0) + " Hz";
                document.getElementById('hudDelta').innerText = Math.floor(energy/2.5) + "%";
                
                // 视觉粒子生成
                if (Math.random() > 0.8) addVisualNote(noteIdx);

            } else {
                // 释音 (Release)
                leadGain.gain.setTargetAtTime(0, now, 0.2);
            }

            document.getElementById('hudChord').innerText = CHORD_NAMES[chordIndex];
        }

        function changeChord(idx, time) {
            // 平滑改变 4 个 Pad 的音高
            let freqs = CHORDS[idx];
            for(let i=0; i<4; i++) {
                padOscs[i].frequency.setTargetAtTime(freqs[i], time, 0.2);
            }
            // 改变 Bass 音高
            bassOsc.frequency.setTargetAtTime(freqs[0]/2, time, 0.2);
        }

        // === 视觉层 ===
        function drawLoop() {
            requestAnimationFrame(drawLoop);
            if (!ctx) return;

            // 拖影背景
            ctx.fillStyle = 'rgba(0,0,0,0.2)';
            ctx.fillRect(0,0,w,h);

            // 绘制和声场 (The Field)
            let gradient = ctx.createLinearGradient(0, h, 0, 0);
            gradient.addColorStop(0, '#1a0b2e');
            gradient.addColorStop(1, 'transparent');
            ctx.fillStyle = gradient;
            let barH = (energy / 255) * h * 0.8;
            ctx.fillRect(0, h - barH, w, barH);

            // 绘制旋律粒子
            for (let i = visualNotes.length - 1; i >= 0; i--) {
                let p = visualNotes[i];
                p.y -= p.speed;
                p.alpha -= 0.02;
                
                ctx.beginPath();
                ctx.arc(p.x, p.y, p.size, 0, Math.PI*2);
                ctx.fillStyle = `rgba(0, 255, 255, ${p.alpha})`;
                ctx.fill();

                if (p.alpha <= 0) visualNotes.splice(i, 1);
            }
        }

        function addVisualNote(idx) {
            visualNotes.push({
                x: (idx / SCALE.length) * w + (Math.random()*20-10),
                y: h - 50,
                size: Math.random() * 5 + 2,
                speed: Math.random() * 2 + 1,
                alpha: 1
            });
        }

        function resizeCanvas() {
            w = canvas.width = canvas.parentElement.clientWidth;
            h = canvas.height = canvas.parentElement.clientHeight;
        }

        function toggleBtn(id, enable) {
            let btn = document.getElementById(id);
            if (enable) btn.classList.add('active');
            else btn.classList.remove('active');
        }
        
        function resetUI() {
            toggleBtn('btnStart', true);
            toggleBtn('btnStop', false);
        }

    </script>
</body>
</html>
