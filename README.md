<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Bat-music: Symphony</title>
    <style>
        body { background: #000; color: #d4af37; margin: 0; height: 100vh; display: flex; flex-direction: column; font-family: 'Times New Roman', serif; overflow: hidden; }
        
        /* 错误提示 */
        #debug { position: fixed; top: 0; left: 0; background: #500; color: #fff; font-size: 10px; z-index: 999; display: none; }

        /* 1. 舞台：金色大厅风格 */
        .stage {
            flex: 1; position: relative; background: #050505;
            background-image: radial-gradient(circle at center, #222 0%, #000 100%);
        }
        canvas { width: 100%; height: 100%; display: block; }
        
        .title {
            position: absolute; top: 20px; width: 100%; text-align: center;
            font-size: 14px; letter-spacing: 3px; color: #888; pointer-events: none;
        }
        .main-text {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            text-align: center; pointer-events: none;
        }
        .timer { font-size: 48px; font-weight: bold; color: #d4af37; text-shadow: 0 0 20px #d4af37; }
        .status { font-size: 14px; margin-top: 10px; color: #666; font-style: italic; }

        /* 2. 控制台 */
        .controls {
            background: #111; padding: 20px; border-top: 1px solid #333;
            display: flex; flex-direction: column; gap: 15px;
        }
        .player-box {
            height: 0; overflow: hidden; transition: 0.3s; background: #222; border-radius: 4px;
        }
        .player-box.show { height: 50px; border: 1px solid #444; }
        audio { width: 100%; height: 100%; }

        .btn-row { display: flex; gap: 15px; }
        button {
            flex: 1; padding: 18px; border: 1px solid #333; background: #000;
            color: #888; font-family: inherit; font-size: 14px; font-weight: bold;
            text-transform: uppercase; letter-spacing: 1px; cursor: pointer;
            transition: 0.2s;
        }
        button:active { background: #222; }
        button:disabled { opacity: 0.3; pointer-events: none; }

        /* 激活色 */
        .btn-start { border-color: #d4af37; color: #d4af37; }
        .btn-start:active { background: #d4af37; color: #000; }
        
        .btn-stop { border-color: #b00; color: #b00; }
        .btn-stop:active { background: #b00; color: #fff; }

        .btn-save { border-color: #fff; color: #fff; }

        /* 微信遮罩 */
        #wxMask {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.95); z-index: 1000; display: none;
            justify-content: center; align-items: center; text-align: center;
        }
    </style>
</head>
<body>

    <div id="debug"></div>

    <div id="wxMask" onclick="this.style.display='none'">
        <div>
            <h2 style="color:#d4af37">保存乐曲</h2>
            <p style="color:#999; line-height:1.8">
                微信不支持直接下载。<br>
                请点击 <strong>存储文件</strong> 后<br>
                长按播放器选择“保存视频/音频”<br>
                或点击右上角 ... 在浏览器打开
            </p>
        </div>
    </div>

    <div class="stage">
        <canvas id="canvas"></canvas>
        <div class="title">BAT-MUSIC SYMPHONY</div>
        <div class="main-text">
            <div class="timer" id="timer">00:00</div>
            <div class="status" id="status">PRESS START</div>
        </div>
    </div>

    <div class="controls">
        <div class="player-box" id="playerBox">
            <audio id="audioPlayer" controls playsinline></audio>
        </div>
        <div class="btn-row">
            <button id="btnStart" class="btn-start" onclick="app.start()">▶ 开始交响</button>
            <button id="btnStop" class="btn-stop" onclick="app.stop()" disabled>■ 停止生成</button>
        </div>
        <div class="btn-row">
            <button id="btnSave" class="btn-save" onclick="app.save()" disabled>💾 存储文件</button>
            <button onclick="location.reload()">🔄 重置</button>
        </div>
    </div>

    <script>
        // 错误捕捉
        window.onerror = function(msg, url, line) {
            let d = document.getElementById('debug');
            d.style.display = 'block';
            d.innerText = `Error: ${msg} (Line ${line})`;
        };

        const app = {
            ctx: null,
            isRecording: false,
            mediaRecorder: null,
            chunks: [],
            blobUrl: null,
            timer: null,
            startTime: 0,
            
            // 音频节点
            dest: null,
            mic: null,
            analyser: null,
            masterGain: null,

            // 交响乐团组件
            strings: [], // 弦乐群
            bass: null,
            drums: null, // 战鼓
            reverb: null, // 大厅混响

            // 乐理：C Minor Harmonic (史诗感)
            // C3, D3, Eb3, F3, G3, Ab3, B3, C4
            scale: [130.8, 146.8, 155.6, 174.6, 196.0, 207.7, 246.9, 261.6],
            nextNoteTime: 0,
            beatCount: 0,

            init: async function() {
                const AC = window.AudioContext || window.webkitAudioContext;
                this.ctx = new AC();
                if(this.ctx.state === 'suspended') await this.ctx.resume();

                // 混响 (模拟金色大厅)
                this.reverb = await this.createReverb();

                // 麦克风
                const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                this.mic = this.ctx.createMediaStreamSource(stream);
                this.analyser = this.ctx.createAnalyser();
                this.analyser.fftSize = 1024;
                this.mic.connect(this.analyser);

                // 总线
                this.dest = this.ctx.createMediaStreamDestination();
                this.masterGain = this.ctx.createGain();
                this.masterGain.gain.value = 0.8;
                
                // 混响接入总线
                this.reverb.connect(this.masterGain);
                this.masterGain.connect(this.ctx.destination);
                this.masterGain.connect(this.dest);

                // 启动视觉和逻辑
                this.loop();
                this.clock(); // 启动节奏时钟
                return true;
            },

            // 创建卷积混响 (Convolution Reverb) - 制造大厅感的核心
            createReverb: async function() {
                let convolver = this.ctx.createConvolver();
                // 生成一个简单的脉冲响应
                let rate = this.ctx.sampleRate;
                let length = rate * 2.0; // 2秒混响
                let decay = 2.0;
                let buffer = this.ctx.createBuffer(2, length, rate);
                for (let c = 0; c < 2; c++) {
                    let channel = buffer.getChannelData(c);
                    for (let i = 0; i < length; i++) {
                        // 指数衰减的白噪声
                        channel[i] = (Math.random() * 2 - 1) * Math.pow(1 - i / length, decay);
                    }
                }
                convolver.buffer = buffer;
                
                // 混响增益
                let gain = this.ctx.createGain();
                gain.gain.value = 0.6; // 60% 湿声
                convolver.connect(gain);
                return gain;
            },

            start: async function() {
                if(this.isRecording) return;
                try {
                    await this.init();
                    this.isRecording = true;
                    this.chunks = [];

                    // 录音机兼容性
                    let mime = 'audio/webm';
                    if(!MediaRecorder.isTypeSupported(mime)) mime = 'audio/mp4';
                    try {
                        this.mediaRecorder = new MediaRecorder(this.dest.stream, {mimeType: mime});
                    } catch(e) {
                        this.mediaRecorder = new MediaRecorder(this.dest.stream);
                    }

                    this.mediaRecorder.ondataavailable = e => { if(e.data.size>0) this.chunks.push(e.data); };
                    this.mediaRecorder.start();

                    // UI
                    this.startTime = Date.now();
                    this.timer = setInterval(()=>this.updateTimer(), 1000);
                    this.updateUI('rec');

                } catch(e) {
                    alert("启动失败: " + e.message);
                }
            },

            stop: function() {
                this.isRecording = false;
                clearInterval(this.timer);
                
                // 停止所有声音
                if(this.masterGain) this.masterGain.gain.setTargetAtTime(0, this.ctx.currentTime, 0.5);

                if(this.mediaRecorder && this.mediaRecorder.state !== 'inactive') {
                    this.mediaRecorder.stop();
                    this.mediaRecorder.onstop = () => {
                        let blob = new Blob(this.chunks, {type: this.mediaRecorder.mimeType});
                        if(this.blobUrl) URL.revokeObjectURL(this.blobUrl);
                        this.blobUrl = URL.createObjectURL(blob);
                        let p = document.getElementById('audioPlayer');
                        p.src = this.blobUrl;
                        this.updateUI('stop');
                    }
                } else {
                    this.updateUI('stop');
                }
            },

            save: function() {
                if(/MicroMessenger/i.test(navigator.userAgent)) {
                    document.getElementById('wxMask').style.display = 'flex';
                    return;
                }
                if(!this.blobUrl) return;
                let a = document.createElement('a');
                a.href = this.blobUrl;
                a.download = `Bat-Symphony_${Date.now()}.webm`;
                document.body.appendChild(a);
                a.click();
                document.body.removeChild(a);
            },

            // === 核心：虚拟管弦乐团 ===
            
            // 1. 超级锯齿波 (SuperSaw) - 模拟弦乐群
            playStrings: function(freq, vol) {
                let now = this.ctx.currentTime;
                // 3个振荡器微走调，制造"一群人拉琴"的感觉
                let detunes = [-10, 0, 10]; 
                
                let env = this.ctx.createGain();
                env.gain.value = 0;
                env.connect(this.reverb); // 进混响
                env.connect(this.ctx.destination); // 干声 (保持清晰度)
                
                // 包络：慢起慢落 (Legato)
                env.gain.linearRampToValueAtTime(vol * 0.3, now + 0.5);
                env.gain.setTargetAtTime(0, now + 0.6, 0.5);

                detunes.forEach(d => {
                    let osc = this.ctx.createOscillator();
                    osc.type = 'sawtooth';
                    osc.frequency.value = freq;
                    osc.detune.value = d;
                    osc.connect(env);
                    osc.start(now);
                    osc.stop(now + 2.0);
                });
            },

            // 2. 史诗战鼓 (Epic Drums)
            playDrum: function(time) {
                // Kick
                let osc = this.ctx.createOscillator();
                let gain = this.ctx.createGain();
                osc.connect(gain);
                gain.connect(this.reverb);
                
                osc.frequency.setValueAtTime(100, time);
                osc.frequency.exponentialRampToValueAtTime(0.01, time + 0.5);
                
                gain.gain.setValueAtTime(0.8, time);
                gain.gain.exponentialRampToValueAtTime(0.01, time + 0.5);
                
                osc.start(time);
                osc.stop(time + 0.5);
            },

            // === 节奏时钟与编曲逻辑 ===
            clock: function() {
                // 100 BPM
                let lookahead = 0.1; 
                let interval = 0.15; // 1/4拍 
                
                let schedule = () => {
                    if(!this.isRecording) return;
                    let now = this.ctx.currentTime;
                    
                    // 调度未来音符
                    if (now >= this.nextNoteTime - lookahead) {
                        // 1. 播放背景鼓点 (4/4拍)
                        if (this.beatCount % 4 === 0) {
                            this.playDrum(this.nextNoteTime); // 重拍
                            this.drawPulse(); // 视觉跳动
                        }

                        // 2. 检测呼噜声并触发弦乐
                        this.checkMicAndPlay(this.nextNoteTime);

                        this.nextNoteTime += 0.6; // 每0.6秒一拍
                        this.beatCount++;
                    }
                    requestAnimationFrame(schedule);
                };
                requestAnimationFrame(schedule);
            },

            checkMicAndPlay: function(time) {
                let data = new Uint8Array(this.analyser.frequencyBinCount);
                this.analyser.getByteFrequencyData(data);
                let sum=0; for(let i=0; i<data.length; i++) sum+=data[i];
                let energy = sum / data.length;

                // 门限：只有声音够大才触发
                if (energy > 15) {
                    // 量化音高 (Quantize)
                    let idx = Math.floor((energy / 60) * this.scale.length);
                    idx = Math.min(idx, this.scale.length-1);
                    let note = this.scale[idx];
                    
                    // 随机升降八度，增加丰富度
                    if (Math.random()>0.7) note *= 2; 
                    if (Math.random()>0.8) note /= 2;

                    // 演奏！
                    this.playStrings(note, Math.min(energy/100, 1.0));
                    
                    // 更新UI文字
                    document.getElementById('status').innerText = "GENERATING: C-MINOR STRINGS";
                    document.getElementById('status').style.color = "#d4af37";
                } else {
                    document.getElementById('status').innerText = "LISTENING...";
                    document.getElementById('status').style.color = "#666";
                }
            },

            // UI与工具
            updateTimer: function() {
                let s = Math.floor((Date.now() - this.startTime)/1000);
                let m = Math.floor(s/60).toString().padStart(2,'0');
                s = (s%60).toString().padStart(2,'0');
                document.getElementById('timer').innerText = `${m}:${s}`;
            },

            updateUI: function(state) {
                const start = document.getElementById('btnStart');
                const stop = document.getElementById('btnStop');
                const save = document.getElementById('btnSave');
                const box = document.getElementById('playerBox');

                if (state === 'rec') {
                    start.disabled = true; start.style.opacity = '0.3';
                    stop.disabled = false; stop.style.opacity = '1';
                    save.disabled = true;
                    box.classList.remove('show');
                } else {
                    start.disabled = false; start.innerText = "▶ 再来一首"; start.style.opacity = '1';
                    stop.disabled = true; stop.style.opacity = '0.3';
                    save.disabled = false;
                    box.classList.add('show');
                    document.getElementById('status').innerText = "SYMPHONY COMPLETE";
                }
            },

            drawPulse: function() {
                const c = document.getElementById('canvas');
                const cx = c.getContext('2d');
                cx.fillStyle = 'rgba(212, 175, 55, 0.1)';
                cx.fillRect(0,0,c.width,c.height);
            },
            
            loop: function() {
                requestAnimationFrame(()=>this.loop());
                if(!this.isRecording) return;
                // 简单的淡出
                const c = document.getElementById('canvas');
                const cx = c.getContext('2d');
                cx.fillStyle = 'rgba(0,0,0,0.05)';
                cx.fillRect(0,0,c.width,c.height);
            }
        };
    </script>
</body>
</html>
