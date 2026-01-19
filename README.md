<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">
    <title>AR English Eat - Easy Speech Edition</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/face_mesh.js" crossorigin="anonymous"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, onSnapshot, query, orderBy, deleteDoc, getDocs, limit } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        let firebaseConfig = {};
        try { firebaseConfig = JSON.parse(__firebase_config); } catch (e) { console.error(e); }
        
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'eng-eat-easy-speech';

        window.fb = { db, auth, appId, collection, doc, setDoc, onSnapshot, query, orderBy, deleteDoc, getDocs, limit };
        
        onAuthStateChanged(auth, (user) => { window.fbUser = user; });
        (async () => {
            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) await signInWithCustomToken(auth, __initial_auth_token);
            else await signInAnonymously(auth);
        })();
    </script>

    <style>
        body { overscroll-behavior: none; background-color: #0f172a; color: white; }
        canvas { width: 100%; height: 100%; }
        .glass { background: rgba(15, 23, 42, 0.85); backdrop-filter: blur(12px); border: 1px solid rgba(255,255,255,0.1); }
        .mic-active { animation: pulse-purple 1.2s infinite; }
        @keyframes pulse-purple {
            0% { box-shadow: 0 0 0 0 rgba(168, 85, 247, 0.6); transform: scale(1); }
            70% { box-shadow: 0 0 0 15px rgba(168, 85, 247, 0); transform: scale(1.05); }
            100% { box-shadow: 0 0 0 0 rgba(168, 85, 247, 0); transform: scale(1); }
        }
        #loading-overlay { transition: opacity 0.5s ease-out; }
    </style>
</head>
<body class="h-screen w-screen overflow-hidden font-sans select-none">

    <div id="loading-overlay" class="fixed inset-0 z-[200] bg-slate-900 flex flex-col justify-center items-center pointer-events-none opacity-0">
        <div class="w-12 h-12 border-4 border-purple-500 border-t-transparent rounded-full animate-spin mb-4"></div>
        <p class="text-purple-300 font-bold">กำลังเปิดกล้อง...</p>
    </div>

    <div id="screen-menu" class="absolute inset-0 z-[100] bg-slate-900 flex flex-col justify-center items-center p-6 text-center">
        <h1 class="text-5xl font-black mb-10 text-transparent bg-clip-text bg-gradient-to-r from-purple-400 to-pink-500">AR English Eat</h1>
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6 w-full max-w-2xl">
            <button onclick="showLogin()" class="bg-purple-600 p-8 rounded-3xl shadow-xl hover:bg-purple-500 active:scale-95 transition-all">
                <div class="text-5xl mb-3">🎮</div>
                <div class="text-xl font-bold">โหมดนักเรียน</div>
            </button>
            <button onclick="openPasscodeModal()" class="bg-slate-800 p-8 rounded-3xl shadow-xl hover:bg-slate-700 active:scale-95 transition-all">
                <div class="text-5xl mb-3">👨‍🏫</div>
                <div class="text-xl font-bold">โหมดคุณครู</div>
            </button>
        </div>
    </div>

    <div id="modal-passcode" class="fixed inset-0 z-[150] flex items-center justify-center p-6 hidden">
        <div class="absolute inset-0 bg-black/60 backdrop-blur-sm" onclick="closePasscodeModal()"></div>
        <div class="glass w-full max-w-sm p-8 rounded-[2rem] relative shadow-2xl">
            <h3 class="text-2xl font-black text-center mb-4">ระบุรหัสผ่าน</h3>
            <input id="teacher-pass-input" type="password" placeholder="••••" class="w-full bg-slate-800 border-2 border-slate-700 rounded-2xl p-4 text-center text-3xl font-mono focus:border-purple-500 outline-none mb-6">
            <div class="grid grid-cols-2 gap-3">
                <button onclick="closePasscodeModal()" class="bg-slate-800 py-3 rounded-xl font-bold text-slate-400">ยกเลิก</button>
                <button onclick="verifyTeacherPass()" class="bg-purple-600 py-3 rounded-xl font-bold text-white">ตกลง</button>
            </div>
        </div>
    </div>

    <div id="screen-login" class="absolute inset-0 bg-slate-900 z-[90] hidden flex flex-col justify-center items-center p-6 text-center">
        <h2 class="text-3xl font-bold mb-6 text-purple-300">ยินดีต้อนรับ!</h2>
        <input id="student-name" type="text" maxlength="15" placeholder="ชื่อเล่นของคุณ..." class="w-full max-w-xs p-4 rounded-xl bg-slate-800 border-2 border-purple-500 text-center text-xl mb-6 outline-none text-white">
        <button onclick="startAction()" class="bg-gradient-to-r from-purple-600 to-pink-600 px-12 py-4 rounded-full font-black text-lg shadow-lg active:scale-95 transition-all">เริ่มภารกิจ!</button>
    </div>

    <div id="screen-teacher" class="absolute inset-0 bg-slate-900 z-[95] hidden flex flex-col p-6 overflow-hidden">
        <div class="flex justify-between items-center mb-6">
            <h2 class="text-3xl font-black text-white">👨‍🏫 Score Board</h2>
            <button onclick="location.reload()" class="bg-slate-800 px-4 py-2 rounded-xl text-xs">ออก</button>
        </div>
        <div id="teacher-list" class="flex-1 overflow-y-auto space-y-2 pr-2"></div>
    </div>

    <div id="game-view" class="relative w-full h-full hidden">
        <video id="input-video" class="hidden" playsinline muted autoplay></video>
        <canvas id="output-canvas" class="absolute inset-0 w-full h-full object-cover"></canvas>
        
        <div class="absolute inset-0 pointer-events-none p-4">
            <div id="hud" class="bg-black/50 backdrop-blur-md rounded-2xl p-4 border border-white/10 opacity-0 transition-opacity">
                <div id="question-text" class="text-yellow-400 text-lg font-bold text-center mb-2 italic">...</div>
                <div class="flex justify-between items-center px-4">
                    <span id="player-display" class="text-xs bg-purple-600 px-3 py-1 rounded-full text-white font-bold">👤 -</span>
                    <span id="score-display" class="text-white font-mono text-2xl font-bold">0</span>
                </div>
            </div>
        </div>

        <!-- Speech Layer (Improved) -->
        <div id="screen-speech" class="absolute inset-0 bg-black/95 z-[120] hidden flex flex-col justify-center items-center p-6 text-center">
            <div class="text-purple-400 text-sm font-bold tracking-widest mb-2 uppercase">ออกเสียงดังๆ:</div>
            <div id="speech-target" class="text-7xl font-black mb-4 text-white uppercase scale-110 tracking-tight">...</div>
            <div id="speech-hint" class="text-slate-400 text-sm italic mb-10">...</div>
            
            <div id="mic-indicator" class="w-28 h-28 bg-purple-600 rounded-full flex items-center justify-center mic-active mb-6 shadow-2xl relative">
                <span id="mic-emoji" class="text-5xl">🎙️</span>
            </div>

            <!-- Audio Power Bar -->
            <div class="w-full max-w-xs bg-slate-800/50 rounded-full h-2 mb-4 overflow-hidden border border-slate-700">
                <div id="audio-bar" class="h-full bg-purple-400 w-0 transition-all duration-75"></div>
            </div>
            
            <div id="speech-feedback" class="h-8 text-xl font-bold text-slate-300">รอฟังเสียง...</div>
            <button onclick="skipSpeech()" class="mt-12 text-slate-600 text-xs underline">กดข้ามหากไมค์ไม่ทำงาน</button>
        </div>
    </div>

    <div id="screen-end" class="absolute inset-0 bg-slate-900 z-[90] hidden flex flex-col justify-center items-center p-6 text-center">
        <div class="bg-slate-800 p-8 rounded-3xl border border-slate-700 max-w-sm w-full">
            <h2 class="text-4xl font-black mb-6">เก่งมาก!</h2>
            <div id="final-score" class="text-6xl font-black text-yellow-400 mb-8">0</div>
            <button onclick="location.reload()" class="w-full bg-purple-600 py-4 rounded-xl font-bold">กลับหน้าหลัก</button>
        </div>
    </div>

<script>
const CONFIG = { mouthThreshold: 10, eatDist: 85, gravity: 2.8, teacherPass: "1234" };
const questions = [
    { q: "She ___ to school every day.", c: "Goes", w: ["Go", "Going"], hint: "โกส์" },
    { q: "The apple is ___ the table.", c: "On", w: ["In", "Under"], hint: "ออน" },
    { q: "A person who teaches is a ___.", c: "Teacher", w: ["Doctor", "Nurse"], hint: "ที-เชอร์" }
];

let state = {
    active: false, score: 0, qIndex: 0, items: [], 
    mouth: { x: 0, y: 0, open: false },
    playerName: "", isSpeech: false,
    recognition: null, isSuccessProcessed: false
};

const ctx = document.getElementById('output-canvas').getContext('2d');

function nav(id) {
    document.querySelectorAll('body > div:not(#loading-overlay)').forEach(d => {
        if(d.id !== 'game-view') d.classList.add('hidden');
    });
    if(id) document.getElementById(id).classList.remove('hidden');
}

window.showLogin = () => nav('screen-login');
window.openPasscodeModal = () => document.getElementById('modal-passcode').classList.remove('hidden');
window.closePasscodeModal = () => document.getElementById('modal-passcode').classList.add('hidden');

window.verifyTeacherPass = () => {
    if (document.getElementById('teacher-pass-input').value === CONFIG.teacherPass) {
        closePasscodeModal(); nav('screen-teacher'); loadTeacherData();
    } else alert("รหัสผ่านไม่ถูกต้อง");
};

window.startAction = async () => {
    const name = document.getElementById('student-name').value.trim();
    if (!name) return alert("กรุณาพิมพ์ชื่อ");
    state.playerName = name;
    
    const loader = document.getElementById('loading-overlay');
    loader.style.opacity = "1";
    
    state.recognition = initSpeechRecognition();
    document.getElementById('player-display').innerText = `👤 ${name}`;
    
    nav('game-view');
    document.getElementById('game-view').classList.remove('hidden');
    
    await initAR();
    loader.style.opacity = "0";
};

async function initAR() {
    const videoElement = document.getElementById('input-video');
    const faceMesh = new FaceMesh({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}` });
    
    faceMesh.setOptions({ maxNumFaces: 1, refineLandmarks: true, minDetectionConfidence: 0.5, minTrackingConfidence: 0.5 });
    faceMesh.onResults(onResults);
    
    const camera = new Camera(videoElement, {
        onFrame: async () => { await faceMesh.send({image: videoElement}); },
        width: 640, height: 480 
    });

    const canv = document.getElementById('output-canvas');
    canv.width = window.innerWidth; canv.height = window.innerHeight;

    return camera.start().then(() => {
        state.active = true;
        document.getElementById('hud').style.opacity = "1";
        updateUI();
    });
}

function onResults(results) {
    ctx.save();
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
    ctx.translate(ctx.canvas.width, 0); ctx.scale(-1, 1);
    ctx.drawImage(results.image, 0, 0, ctx.canvas.width, ctx.canvas.height);
    ctx.restore();

    if (results.multiFaceLandmarks?.[0]) {
        const lm = results.multiFaceLandmarks[0];
        const dist = Math.abs(lm[13].y - lm[14].y) * ctx.canvas.height;
        state.mouth.x = (1 - lm[13].x) * ctx.canvas.width;
        state.mouth.y = lm[13].y * ctx.canvas.height;
        state.mouth.open = dist > CONFIG.mouthThreshold;
        
        ctx.beginPath();
        ctx.arc(state.mouth.x, state.mouth.y, 14, 0, 7);
        ctx.fillStyle = state.mouth.open ? "#a855f7" : "rgba(255,255,255,0.7)";
        ctx.fill();
        if(state.mouth.open) { ctx.strokeStyle = "white"; ctx.lineWidth = 2; ctx.stroke(); }
    }
    if (state.active && !state.isSpeech) updateGame();
}

function updateGame() {
    if (Math.random() < 0.04 && state.items.length < 3) spawnItem();
    state.items.forEach((item, idx) => {
        item.y += CONFIG.gravity;
        
        ctx.save();
        ctx.fillStyle = "rgba(15, 23, 42, 0.85)";
        ctx.beginPath(); ctx.roundRect(item.x - 70, item.y - 30, 140, 60, 20); ctx.fill();
        ctx.strokeStyle = item.isCorrect ? "#a855f7" : "#334155";
        ctx.lineWidth = 3; ctx.stroke();
        ctx.fillStyle = "white"; ctx.font = "bold 24px sans-serif"; ctx.textAlign = "center";
        ctx.fillText(item.text, item.x, item.y + 8);
        ctx.restore();

        if (Math.hypot(item.x - state.mouth.x, item.y - state.mouth.y) < CONFIG.eatDist && state.mouth.open) {
            state.items.splice(idx, 1);
            if (item.isCorrect) {
                state.score += 10; syncScore(); startSpeechPhase(item.text);
            } else {
                state.score = Math.max(0, state.score - 5); syncScore();
            }
            updateUI();
        }
        if (item.y > ctx.canvas.height + 50) state.items.splice(idx, 1);
    });
}

function spawnItem() {
    const q = questions[state.qIndex]; if(!q) return;
    const pool = [q.c, ...q.w];
    const text = pool[Math.floor(Math.random() * pool.length)];
    state.items.push({ x: 80 + Math.random()*(ctx.canvas.width-160), y: -50, text, isCorrect: text === q.c });
}

function initSpeechRecognition() {
    const R = window.SpeechRecognition || window.webkitSpeechRecognition;
    if (!R) return null;
    const rec = new R();
    rec.lang = 'en-US'; rec.continuous = true; rec.interimResults = true;
    
    rec.onresult = (e) => {
        if (state.isSuccessProcessed) return;
        
        const transcript = e.results[e.results.length-1][0].transcript.toLowerCase().trim();
        const target = document.getElementById('speech-target').innerText.toLowerCase();
        
        // Visual Bar Feedback (Simulated from transcript length for responsiveness)
        const vol = Math.min(100, transcript.length * 15);
        document.getElementById('audio-bar').style.width = vol + '%';
        
        document.getElementById('speech-feedback').innerText = `ได้ยิน: "${transcript}"`;
        
        // FUZZY MATCH LOGIC
        const isMatch = transcript.includes(target) || 
                        (transcript.length >= 3 && target.startsWith(transcript.substring(0,3))) ||
                        fuzzyCheck(transcript, target);

        if (isMatch) handleSpeechSuccess();
    };

    rec.onend = () => { if (state.isSpeech && !state.isSuccessProcessed) try { rec.start(); } catch(e){} };
    return rec;
}

// Easy mode fuzzy helper
function fuzzyCheck(heard, target) {
    if (target === "goes" && (heard.includes("go") || heard.includes("ghost"))) return true;
    if (target === "teacher" && (heard.includes("teach") || heard.includes("shirt"))) return true;
    if (target === "on" && (heard.includes("own") || heard.includes("orn"))) return true;
    return false;
}

function startSpeechPhase(text) {
    state.active = false; state.isSpeech = true; state.isSuccessProcessed = false; state.items = [];
    document.getElementById('screen-speech').classList.remove('hidden');
    document.getElementById('speech-target').innerText = text;
    document.getElementById('speech-hint').innerText = `พูดให้ชัด: ${questions[state.qIndex]?.hint || ""}`;
    document.getElementById('audio-bar').style.width = '0%';
    if (state.recognition) try { state.recognition.start(); } catch(e){}
}

function handleSpeechSuccess() {
    if (state.isSuccessProcessed) return;
    state.isSuccessProcessed = true;
    
    document.getElementById('speech-feedback').innerText = "ถูกต้อง! ✨";
    document.getElementById('speech-feedback').classList.add('text-green-400');
    document.getElementById('audio-bar').style.width = '100%';
    document.getElementById('audio-bar').classList.replace('bg-purple-400', 'bg-green-400');
    
    if (state.recognition) try { state.recognition.stop(); } catch(e){}
    
    setTimeout(() => {
        document.getElementById('screen-speech').classList.add('hidden');
        document.getElementById('speech-feedback').classList.remove('text-green-400');
        document.getElementById('audio-bar').classList.replace('bg-green-400', 'bg-purple-400');
        state.isSpeech = false; state.active = true; state.qIndex++;
        if (state.qIndex >= questions.length) finish();
        updateUI();
    }, 1200);
}

function skipSpeech() { handleSpeechSuccess(); }

async function syncScore() {
    if (!window.fb || !state.playerName || !window.fbUser) return;
    const { db, appId, doc, setDoc } = window.fb;
    try { await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'scores', window.fbUser.uid), { name: state.playerName, score: state.score, updatedAt: Date.now() }, { merge: true }); } catch (e) {}
}

function loadTeacherData() {
    if (!window.fb) return;
    const { db, appId, collection, onSnapshot } = window.fb;
    onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'scores'), (snap) => {
        let scores = []; snap.forEach(d => scores.push(d.data()));
        scores.sort((a,b) => b.score - a.score);
        document.getElementById('teacher-list').innerHTML = scores.map((s,i) => `
            <div class="flex items-center justify-between bg-slate-800 p-4 rounded-2xl mb-2 border border-slate-700">
                <span class="font-bold">#${i+1} ${s.name}</span>
                <span class="text-purple-400 font-mono font-black text-xl">${s.score}</span>
            </div>`).join('');
    });
}

function updateUI() {
    document.getElementById('score-display').innerText = state.score;
    if(questions[state.qIndex]) document.getElementById('question-text').innerText = questions[state.qIndex].q;
}

function finish() {
    state.active = false; nav('screen-end');
    document.getElementById('final-score').innerText = state.score;
}

window.onresize = () => { ctx.canvas.width = window.innerWidth; ctx.canvas.height = window.innerHeight; };
</script>
</body>
</html>
