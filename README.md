<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Bat-music: Master Studio</title>
    <style>
        /* === 视觉风格：极简大师黑 === */
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: #000;
            color: #fff;
            margin: 0;
            height: 100vh;
            display: flex;
            flex-direction: column;
            overflow: hidden;
        }

        /* 1. 上半部分：动态示波器 */
        .monitor-screen {
            flex: 1;
            position: relative;
            background: radial-gradient(circle at bottom, #111 0%, #000 100%);
            border-bottom: 1px solid #333;
        }
        canvas { width: 100%; height: 100%; display: block; }

        /* 状态指示器 */
        .status-bar {
            position: absolute;
            top: 20px; left: 0; width: 100%;
            display: flex;
            justify-content: center;
            align-items: center;
            pointer-events: none;
            z-index: 10;
        }
        .rec-tag {
            background: #f00;
            color: #fff;
            padding: 4px 12px;
            border-radius: 12px;
            font-size: 12px;
            font-weight: bold;
            display: flex;
            align-items: center;
            gap: 6px;
            box-shadow: 0 0 15px #f00;
            animation: pulse 1.5s infinite;
            visibility: hidden; /* 默认隐藏 */
        }
        .rec-tag.active { visibility: visible; }
        
        .timer {
            position: absolute;
            right: 20px;
            font-family: 'Monaco', monospace;
            font-size: 18px;
            font-weight: bold;
            color: #888;
        }
        .timer.active { color: #fff; }

        /* HUD 数据显示 */
        .hud {
            position: absolute;
            bottom: 20px; left: 20px;
            font-family: 'Monaco', monospace;
            font-size: 10px;
            color: #666;
            line-height: 1.6;
            pointer-events: none;
        }
        .hud span { color: #0af; }

        @keyframes pulse { 0% { opacity: 1; } 50% { opacity: 0.5; } 100% { opacity: 1; } }

        /* 2. 下半部分：控制台 */
        .console {
            background: #111;
            padding: 20px;
            padding-bottom: env(safe-area-inset-bottom, 30px);
            border-top: 1px solid #222;
            min-height: 160px;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
        }

        /* 按钮容器 */
        .control-deck {
            width: 100%;
            max-width: 450px;
            display: flex;
            flex-direction: column;
            gap: 15px;
        }

        /* 通用按钮样式 */
        button {
            width: 100%;
            padding: 18px;
            border: none;
            border-radius: 12px;
            font-size: 16px;
            font-weight: bold;
            cursor: pointer;
            transition: all 0.2s;
            display: flex;
            justify-content: center;
            align-items: center;
            gap: 8px;
            text-transform: uppercase;
            letter-spacing: 1px;
        }
        button:active { transform: scale(0.98); }

        /* 颜色主题 */
        .btn-start { background: #fff; color: #000; }
        .btn-stop { background: #ff3b30; color: #fff; box-shadow: 0 4px 20px rgba(255, 59, 48, 0.4); }
        .btn-action { flex: 1; font-size: 14px; padding: 15px; }
        
        .btn-reset { background: #222; color: #888; border: 1px solid #333; }
        .btn-save { background: #007aff; color: #fff; }

        /* 回放界面 */
        .playback-ui {
            display: none; /* 初始隐藏 */
            width: 100%;
            flex-direction: column;
            gap: 15px;
        }

        .audio-wrapper {
            background: #222;
            border-radius: 10px;
            padding: 5px;
            border: 1px solid #333;
        }
        audio {
            width: 100%;
            height: 45px;
            outline: none;
        }
        
        .action-row {
            display: flex;
            gap: 10px;
            width: 100%;
        }

    </style>
</head>
<body>

    <div class="monitor-screen">
        <canvas id="canvas"></canvas>
        
        <div class="status-bar">
            <div class="rec-tag" id="recTag">● REC</div>
            <div class="timer" id="timer">00:00</div>
        </div>

        <div class="hud">
            <div>ENGINE: FUNCTIONAL SYNTH</div>
            <div>PITCH (Y*): <span id="pitchVal">--</span></div>
            <div>TIMBRE: <span id="timbreVal">--</span></div>
        </div>
    </div>

    <div class="console">
        <div class="control-deck">

            <button id="btnRecord" class="btn-start" onclick="startRecording()">
                🔴 启动并录音
            </button>

            <button id="btnStop" class="btn-stop" onclick="stopRecording()" style="display:none">
                ⏹ 停止并生成乐曲
            </button>

            <div id="playbackUI" class="playback-ui">
                <div class="audio-wrapper">
                    <audio id="audioPlayer" controls playsinline></audio>
                </div>

                <div class="action-row">
                    <button class="btn-reset btn-action" onclick="resetSystem()">
                        🔄 重录
                    </button>
                    <button class="btn-save btn-action" onclick="downloadMusic()">
                        ⬇️ 导出分享
                    </button>
                </div>
            </div>

        </div>
    </div>

    <script>
        // === 系统全局变量 ===
        let audioCtx, analyser, micSource;
        let destNode, mediaRecorder;
        let audioChunks = [];
        let blobUrl = null;
        let wakeLock = null;

        // 合成器组件
        let osc, filter, gainNode, delay, feedback;
        
        // 状态变量
        let isRecording = false;
        let startTime = 0;
        let timerInt = null;
        let gateOpen = false;

        // 泛函计算变量
        let energy = 0, centroid = 0;
        const SCALE = [130.81, 146.83, 155.56, 174.61, 196.00, 220.00, 233.08, 261.63, 293.66, 311.13, 349.23, 392.00, 440.00, 523.25];

        // 绘图
        let canvas, ctx, w, h;

        // === 1. [🔴 开始录制] 流程 ===
        async function startRecording() {
