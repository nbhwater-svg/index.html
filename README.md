<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>改修版：1分間英語スピーチトレーナー</title>
    <style>
        :root {
            --primary: #1a73e8;
            --accent: #d93025;
            --success: #188038;
            --bg: #f8f9fa;
            --card: #ffffff;
            --text: #111111; /* 文字色をハッキリとした濃い黒に変更 */
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background-color: var(--bg);
            color: var(--text);
            margin: 0;
            padding: 16px;
            display: flex;
            justify-content: center;
        }

        .container {
            width: 100%;
            max-width: 550px;
            background: var(--card);
            padding: 24px;
            border-radius: 16px;
            box-shadow: 0 4px 16px rgba(0,0,0,0.08);
            box-sizing: border-box;
            text-align: center;
        }

        h1 {
            font-size: 22px;
            margin-top: 0;
            color: var(--primary);
            margin-bottom: 20px;
        }

        .timer-circle {
            width: 140px;
            height: 140px;
            border-radius: 50%;
            border: 6px solid #e8eaed;
            display: flex;
            align-items: center;
            justify-content: center;
            margin: 0 auto 24px;
            position: relative;
            transition: border-color 0.3s;
        }

        .timer-circle.active {
            border-color: var(--accent);
        }

        .timer-display {
            font-size: 36px;
            font-weight: bold;
            font-variant-numeric: tabular-nums;
        }

        .status-badge {
            display: inline-block;
            padding: 6px 12px;
            border-radius: 20px;
            font-size: 14px;
            font-weight: bold;
            background: #e8eaed;
            margin-bottom: 24px;
        }

        .status-badge.recording {
            background: #fce8e6;
            color: var(--accent);
            animation: pulse 1.5s infinite;
        }

        @keyframes pulse {
            0% { opacity: 1; }
            50% { opacity: 0.6; }
            100% { opacity: 1; }
        }

        .btn {
            width: 100%;
            padding: 16px;
            font-size: 18px;
            font-weight: bold;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            margin-bottom: 12px;
            transition: background 0.2s, transform 0.1s;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }

        .btn:active { transform: scale(0.98); }

        .btn-start { background-color: var(--primary); color: white; }
        .btn-start:hover { background-color: #1557b0; }

        .btn-stop { background-color: var(--accent); color: white; display: none; }
        .btn-stop:hover { background-color: #b31412; }

        .audio-section {
            margin-top: 20px;
            padding-top: 20px;
            border-top: 1px solid #e8eaed;
            display: none;
        }

        .audio-section h3 {
            font-size: 15px;
            margin: 0 0 10px;
            color: #5f6368;
            text-align: left;
        }

        audio { width: 100%; margin-bottom: 10px; }

        .transcript-section {
            margin-top: 20px;
            text-align: left;
        }

        .transcript-section h3 {
            font-size: 16px;
            margin: 0 0 10px;
            color: #5f6368;
        }

        /* 文字表示エリアの改善（見やすさ重視） */
        .transcript-box {
            width: 100%;
            height: 240px; /* ボックスを広げて見やすく */
            overflow-y: auto;
            padding: 16px;
            background: #ffffff;
            border-radius: 8px;
            font-size: 18px; /* 文字を大きく */
            line-height: 1.6; /* 行間を広げて読みやすく */
            color: #111111; /* 完全にハッキリした黒文字に固定 */
            box-sizing: border-box;
            white-space: pre-wrap;
            border: 2px solid #9aa0a6;
        }

        .placeholder-text {
            color: #70757a;
            font-style: italic;
            font-size: 16px;
        }
    </style>
</head>
<body>

<div class="container">
    <h1>1分間英語スピーチトレーナー (文字消え対策版)</h1>
    
    <div id="timerCircle" class="timer-circle">
        <div id="timerDisplay" class="timer-display">01:00</div>
    </div>

    <div>
        <div id="statusBadge" class="status-badge">準備完了</div>
    </div>

    <button id="startBtn" class="btn btn-start">スピーチを開始する (Start)</button>
    <button id="stopBtn" class="btn btn-stop">スピーチを終了する (Stop)</button>

    <div id="audioSection" class="audio-section">
        <h3>録音された音声 (Playback):</h3>
        <audio id="audioPlayer" controls></audio>
    </div>

    <div class="transcript-section">
        <h3>文字起こし結果 (English Transcript):</h3>
        <div id="transcriptBox" class="transcript-box">
            <span class="placeholder-text">ここにあなたのスピーチした英語がリアルタイムで記録・消えずに蓄積されます...</span>
        </div>
    </div>
</div>

<script>
    let countdownInterval = null;
    let timeLeft = 60; 

    let mediaRecorder = null;
    let audioChunks = [];

    // 確定したテキストを蓄積・キープしておくための変数
    let finalTranscriptHistory = ""; 

    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
    let recognition = null;

    if (SpeechRecognition) {
        recognition = new SpeechRecognition();
        recognition.continuous = true;       
        recognition.interimResults = true;    
        recognition.lang = 'en-US';           
    }

    const timerDisplay = document.getElementById('timerDisplay');
    const timerCircle = document.getElementById('timerCircle');
    const statusBadge = document.getElementById('statusBadge');
    const startBtn = document.getElementById('startBtn');
    const stopBtn = document.getElementById('stopBtn');
    const audioSection = document.getElementById('audioSection');
    const audioPlayer = document.getElementById('audioPlayer');
    const transcriptBox = document.getElementById('transcriptBox');

    function startTimer() {
        timeLeft = 60;
        updateTimerDisplay();
        timerCircle.classList.add('active');

        countdownInterval = setInterval(() => {
            timeLeft--;
            updateTimerDisplay();

            if (timeLeft <= 0) {
                endSpeech(true); 
            }
        }, 1000);
    }

    function updateTimerDisplay() {
        const minutes = Math.floor(timeLeft / 60);
        const seconds = timeLeft % 60;
        timerDisplay.textContent = `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`;
    }

    startBtn.addEventListener('click', async () => {
        try {
            const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
            mediaRecorder = new MediaRecorder(stream);
            audioChunks = [];

            mediaRecorder.ondataavailable = (event) => {
                audioChunks.push(event.data);
            };

            mediaRecorder.onstop = () => {
                const audioBlob = new Blob(audioChunks, { type: 'audio/mp3' });
                const audioUrl = URL.createObjectURL(audioBlob);
                audioPlayer.src = audioUrl;
                audioSection.style.display = 'block';
                stream.getTracks().forEach(track => track.stop());
            };

            // スタート時に記録用変数をリセット
            finalTranscriptHistory = "";

            startBtn.style.display = 'none';
            stopBtn.style.display = 'block';
            statusBadge.textContent = '録音＆文字起こし中...';
            statusBadge.classList.add('recording');
            transcriptBox.innerHTML = ''; 

            mediaRecorder.start();
            if (recognition) {
                recognition.start();
            } else {
                transcriptBox.innerHTML = '<span class="placeholder-text" style="color:red;">お使いのブラウザは文字起こしに対応していません。(PCのGoogle Chromeを推奨します)</span>';
            }

            startTimer();

        } catch (err) {
            alert('マイクのアクセスが拒否されたか、マイクが見つかりません。');
            console.error(err);
        }
    });

    // 改良した文字起こしエンジン
    if (recognition) {
        recognition.onresult = (event) => {
            let interimTranscript = '';
            let currentSessionFinal = '';

            for (let i = event.resultIndex; i < event.results.length; ++i) {
                if (event.results[i].isFinal) {
                    currentSessionFinal += event.results[i].transcript + ' ';
                } else {
                    interimTranscript += event.results[i].transcript;
                }
            }

            // 確定したテキストは今までの履歴(finalTranscriptHistory)にどんどん追加
            if (currentSessionFinal !== "") {
                finalTranscriptHistory += currentSessionFinal;
            }

            // 過去のすべての確定テキスト(黒) ＋ 今話している最中のテキスト(薄いグレー)を表示
            // これにより、通信の瞬きで文字がリセットされて消えるのを完全に防ぎます
            transcriptBox.innerHTML = `<span style="color: #111111; font-weight: bold;">${finalTranscriptHistory}</span><span style="color: #70757a;">${interimTranscript}</span>`;
            
            transcriptBox.scrollTop = transcriptBox.scrollHeight;
        };

        // ブラウザ側で音声認識が勝手に途切れた場合は自動で再スタートさせる処理を追加
        recognition.onend = () => {
            if (timeLeft > 0 && startBtn.style.display === 'none') {
                recognition.start();
            }
        };

        recognition.onerror = (event) => {
            console.error('Speech recognition error', event.error);
        };
    }

    stopBtn.addEventListener('click', () => {
        endSpeech(false);
    });

    function endSpeech(isTimeUp) {
        clearInterval(countdownInterval);
        timerCircle.classList.remove('active');

        if (mediaRecorder && mediaRecorder.state !== 'inactive') {
            mediaRecorder.stop();
        }
        if (recognition) {
