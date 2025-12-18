<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Bat-music v6.0</title>
    <style>
        /* === 极简高对比度风格 (防止渲染错误) === */
        body {
            background: #000;
            color: #fff;
            margin: 0;
            height: 100vh;
            display: flex;
            flex-direction: column;
            font-family: sans-serif;
            overflow: hidden;
        }

        /* 调试控制台 (如果不幸出错，这里会显示红字) */
        #debugConsole {
            position: fixed; top: 0; left: 0; width: 100%; height: 20px;
            background: #200; color: #f00; font-size: 10px;
            z-index: 9999; display: none; padding: 2px;
        }

        /* 1. 示波器 */
        .stage {
            flex: 1;
            position: relative;
            background: #111;
            border-bottom: 2px solid #333;
        }
        canvas { width: 100%; height: 100%; display: block; }

        .center-status {
            position: absolute;
            top: 50%; left: 50%;
            transform: translate(-50%, -50%);
            text-align: center;
            pointer-events: none;
        }
        .timer { font-size: 40px; font-weight: bold; font-family: monospace; }
        .status-text { font-size: 14px; color: #888; margin-top: 10px; }

        /* 2. 控制区 */
        .controls {
            height: 220px;
            background: #000;
            padding: 20px;
            box-sizing: border-box;
            display: flex;
            flex-direction: column;
            gap: 15px;
        }

        /* 播放器 */
        #playerLayer {
            height: 0; opacity: 0; overflow: hidden; transition: all 0.3s;
            background: #222; border-radius: 8px;
        }
        #playerLayer.show { height: 50px; opacity: 1; border: 1px solid #444; }
        audio { width: 100%; height: 100%; outline: none; }

        /* 按钮组 */
        .btn-row { display: flex; gap: 15px; flex: 1; }
        
        button {
            flex: 1;
            border: none;
            border-radius: 8px;
            font-size: 16px;
            font-weight: bold;
            cursor: pointer;
            transition: opacity 0.2s;
        }
        button:active { opacity: 0.7; }
        button:disabled { background: #222 !important; color: #555 !important; }

        /* 颜色定义 */
        .btn-rec { background: #007aff; color: #fff; }
        .btn-stop { background: #ff3b30; color: #fff; }
        .btn-save { background: #34c759; color: #000; }
        .btn-reset { background: #333; color: #ccc; }

        /* 微信遮罩 */
        #wxMask {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.9); z-index: 1000;
            display: none; justify-content: center; align-items: center; text-align: center;
        }
    </style>
</head>
<body>
    
    <div id="debugConsole"></div>

    <div id="wxMask" onclick="this.style.display='none'">
        <div style="padding: 20px;">
            <h2 style="color:#0f0">微信无法直接分享</h2>
            <p style="color:#ccc">请点击 <strong>存储文件</strong><br>然后长按选择“保存”<br>或点击右上角 ... 在浏览器打开</p>
        </div>
    </div>

    <div class="stage">
        <canvas id="canvas"></canvas>
        <div class="center-status">
            <div class="timer" id="timer">00:00</div>
            <div class="status-text" id="statusText">READY</div>
        </div>
    </div>

    <div class="controls">
        <div id="playerLayer">
            <audio id="audioPlayer" controls playsinline></audio>
        </div>

        <div class="btn-row">
            <button id="btnStart" class="btn-rec" onclick="app.start()">● 开始录制</button>
            <button id="btnStop" class="btn-stop" onclick="app.stop()" disabled>■ 停止</button>
        </div>
        <div class="btn-row">
            <button id="btnSave" class="btn-save" onclick="app.save()" disabled>💾 存储文件</button>
            <button id="btnReset" class="btn-reset" onclick="app.reset()">🔄 重置</button>
        </div>
    </div>

    <script>
        // === 全局错误捕捉 (防止崩了没人知道) ===
        window.onerror = function(msg, source, lineno) {
            let el = document.getElementById('debugConsole');
            el.style.display = 'block';
            el.innerText = `Error: ${msg} (Line ${lineno})`;
            return false;
        };

        // === 应用命名空间 (防止变量污染) ===
        const app = {
            // 状态
            isRecording: false,
            ctx: null,
            mediaRecorder: null,
            chunks: [],
            blobUrl: null,
            timerInt: null,
            startTime: 0,
            
            // 音频节点
            dest: null,
            mic: null,
            analyser: null,
            osc: null,
            gain: null,
            
            // 乐理常量 (C Minor Pentatonic)
            scale: [130.8, 155.6, 174.6, 196.0, 233.1, 261.6, 311.1, 392.0],

            // 1. 初始化引擎
            init: async function() {
                try {
                    // 兼容 AudioContext
                    const AudioContext = window.AudioContext || window.webkitAudioContext;
                    if (!this.ctx) this.ctx = new AudioContext();
                    if (this.ctx.state === 'suspended') await this.ctx.resume();

                    // 麦克风
                    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                    this.mic = this.ctx.createMediaStreamSource(stream);
                    this.analyser = this.ctx.createAnalyser();
                    this.analyser.fftSize = 1024;
                    this.mic.connect(this.analyser);

                    // 合成器 (振荡器 + 增益)
                    this.osc = this.ctx.createOscillator();
                    this.osc.type = 'sine';
                    this.gain = this.ctx.createGain();
                    this.gain.gain.value = 0;
                    
                    this.osc.connect(this.gain);
                    this.osc.start();

                    // 总线 (用于录音和回放)
                    this.dest = this.ctx.createMediaStreamDestination();
                    
                    // 路由：合成器 -> (耳朵 + 录音机)
                    this.gain.connect(this.ctx.destination);
                    this.gain.connect(this.dest);

                    // 启动视觉和逻辑循环
                    this.loop();
                    return true;
                } catch (e) {
                    alert("麦克风启动失败: " + e.message);
                    return false;
                }
            },

            // 2. 开始录制
            start: async function() {
                if (this.isRecording) return;
                
                // 确保引擎启动
                const ready = await this.init();
                if (!ready) return;

                this.isRecording = true;
                this.chunks = [];
                
                // 兼容 Safari 的录音格式
                let mime = 'audio/webm';
                if (!MediaRecorder.isTypeSupported(mime)) mime = 'audio/mp4';
                
                try {
                    this.mediaRecorder = new MediaRecorder(this.dest.stream, { mimeType: mime });
                } catch(e) {
                    // 如果指定格式失败，让浏览器自己选
                    this.mediaRecorder = new MediaRecorder(this.dest.stream);
                }

                this.mediaRecorder.ondataavailable = e => {
                    if(e.data.size > 0) this.chunks.push(e.data);
                };

                this.mediaRecorder.start();

                // 计时器
                this.startTime = Date.now();
                this.timerInt = setInterval(() => this.updateTimer(), 1000);

                // UI 更新
                this.updateUI('recording');
            },

            // 3. 停止 (暴力强制版)
            stop: function() {
                this.isRecording = false;
                clearInterval(this.timerInt);
                
                // 物理静音
                if(this.gain) this.gain.gain.setTargetAtTime(0, this.ctx.currentTime, 0.1);

                // 强制停止录音机
                if (this.mediaRecorder && this.mediaRecorder.state !== 'inactive') {
                    this.mediaRecorder.stop();
                    
                    this.mediaRecorder.onstop = () => {
                        const blob = new Blob(this.chunks, { type: this.mediaRecorder.mimeType });
                        if (this.blobUrl) URL.revokeObjectURL(this.blobUrl);
                        this.blobUrl = URL.createObjectURL(blob);
                        
                        // 加载播放器
                        let p = document.getElementById('audioPlayer');
                        p.src = this.blobUrl;
                        
                        this.updateUI('stopped');
                    };
                } else {
                    // 如果录音机本来就没动，也要重置UI
                    this.updateUI('stopped');
                }
            },

            // 4. 存储 (微信兼容版)
            save: function() {
                // 检测是否微信
                const isWx = /MicroMessenger/i.test(navigator.userAgent);
                if (isWx) {
                    document.getElementById('wxMask').style.display = 'flex';
                    return; // 微信里不能直接调用下载，只能提示
                }

                if (!this.blobUrl) return;
                const a = document.createElement('a');
                a.href = this.blobUrl;
                a.download = 'Bat-music_' + Date.now() + '.webm';
                document.body.appendChild(a);
                a.click();
                document.body.removeChild(a);
            },

            reset: function() {
                location.reload(); // 最彻底的重置
            },

            // 5. 核心逻辑循环 (AI 作曲)
            loop: function() {
                requestAnimationFrame(() => this.loop());
                if (!this.isRecording) return;

                const data = new Uint8Array(this.analyser.frequencyBinCount);
                this.analyser.getByteFrequencyData(data);

                // 计算能量
                let sum = 0; for(let i=0; i<data.length; i++) sum+=data[i];
                let energy = sum / data.length;

                // 简单的门限逻辑
                const now = this.ctx.currentTime;
                if (energy > 15) {
                    // 有声音 -> 映射音高
                    let idx = Math.floor((energy / 60) * this.scale.length);
                    idx = Math.min(idx, this.scale.length - 1);
                    let freq = this.scale[idx];

                    this.osc.frequency.setTargetAtTime(freq, now, 0.1);
                    this.gain.gain.setTargetAtTime(0.5, now, 0.1);
                    
                    this.draw(energy, '#0f0');
                } else {
                    // 没声音 -> 静音
                    this.gain.gain.setTargetAtTime(0, now, 0.2);
                    this.draw(energy, '#333');
                }
            },

            // 绘图
            draw: function(energy, color) {
                const c = document.getElementById('canvas');
                const cx = c.getContext('2d');
                // 适配高清屏
                if (c.width !== c.offsetWidth) {
                    c.width = c.offsetWidth;
                    c.height = c.offsetHeight;
                }

                cx.fillStyle = 'rgba(0,0,0,0.1)';
                cx.fillRect(0, 0, c.width, c.height);

                let r = 50 + energy * 2;
                cx.beginPath();
                cx.arc(c.width/2, c.height/2, r, 0, Math.PI*2);
                cx.strokeStyle = color;
                cx.lineWidth = 5;
                cx.stroke();
            },

            updateTimer: function() {
                let s = Math.floor((Date.now() - this.startTime) / 1000);
                let m = Math.floor(s/60).toString().padStart(2,'0');
                s = (s%60).toString().padStart(2,'0');
                document.getElementById('timer').innerText = `${m}:${s}`;
            },

            updateUI: function(state) {
                const btnStart = document.getElementById('btnStart');
                const btnStop = document.getElementById('btnStop');
                const btnSave = document.getElementById('btnSave');
                const playerLayer = document.getElementById('playerLayer');
                const statusText = document.getElementById('statusText');

                if (state === 'recording') {
                    btnStart.disabled = true;
                    btnStop.disabled = false;
                    btnStop.style.opacity = '1';
                    btnSave.disabled = true;
                    playerLayer.classList.remove('show');
                    statusText.innerText = "RECORDING (录音中)...";
                    statusText.style.color = "#f00";
                } else if (state === 'stopped') {
                    btnStart.disabled = false;
                    btnStart.innerText = "● 继续录制";
                    btnStop.disabled = true;
                    btnSave.disabled = false;
                    playerLayer.classList.add('show');
                    statusText.innerText = "COMPLETED (已完成)";
                    statusText.style.color = "#0f0";
                }
            }
        };
    </script>
</body>
</html>
