<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">
    <title>AR English Eat - M.1 (Easy Speech Mode)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/face_mesh.js" crossorigin="anonymous"></script>
    
    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, onSnapshot, query, orderBy, deleteDoc, getDocs, limit } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        let firebaseConfig = {};
        try {
            firebaseConfig = JSON.parse(__firebase_config);
        } catch (e) {
            console.error("Firebase config error:", e);
        }
        
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'eng-eat-m1-cloud';

        window.fb = { db, auth, appId, collection, doc, setDoc, onSnapshot, query, orderBy, deleteDoc, getDocs, limit };
        window.fbUser = null;

        const initAuth = async () => {
            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                await signInWithCustomToken(auth, __initial_auth_token);
            } else {
                await signInAnonymously(auth);
            }
        };

        onAuthStateChanged(auth, (user) => {
            window.fbUser = user;
        });

        initAuth();
    </script>

    <style>
        body { overscroll-behavior: none; background-color: #0f172a; color: white; }
        canvas { width: 100%; height: 100%; }
        .custom-scrollbar::-webkit-scrollbar { width: 6px; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: #475569; border-radius: 10px; }
        .glass { background: rgba(15, 23, 42, 0.8); backdrop-filter: blur(12px); border: 1px solid rgba(255,255,255,0.1); }
        
        .mic-active { animation: pulse-purple 1.5s infinite; }
        @keyframes pulse-purple {
            0% { box-shadow: 0 0 0 0 rgba(168, 85, 247, 0.7); transform: scale(1); }
            70% { box-shadow: 0 0 0 20px rgba(168, 85, 247, 0); transform: scale(1.05); }
            100% { box-shadow: 0 0 0 0 rgba(168, 85, 247, 0); transform: scale(1); }
        }

        .success-pop { animation: success-pop 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275); }
        @keyframes success-pop {
            0% { transform: scale(0.5); opacity: 0; }
            100% { transform: scale(1.1); opacity: 1; }
        }
    </style>
</head>
<body class="h-screen w-screen overflow-hidden font-sans select-none">

    <!-- Screen: Main Menu -->
    <div id="screen-menu" class="absolute inset-0 z-[100] bg-slate-900 flex flex-col justify-center items-center p-6 text-center">
        <h1 class="text-5xl font-black mb-10 text-transparent bg-clip-text bg-gradient-to-r from-purple-400 to-pink-500">
            AR English Eat
        </h1>
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6 w-full max-w-2xl">
            <button onclick="showLogin()" class="bg-purple-600 p-8 rounded-3xl shadow-xl hover:bg-purple-500 transition active:scale-95 border-b-4 border-purple-800">
                <div class="text-5xl mb-3">🎮</div>
                <div class="text-xl font-bold text-white">โหมดนักเรียน</div>
                <p class="text-xs opacity-70 mt-2">สะสมคะแนนจากการกินคำศัพท์และฝึกพูด</p>
            </button>
            <button onclick="openPasscodeModal()" class="bg-slate-800 p-8 rounded-3xl shadow-xl hover:bg-slate-700 transition active:scale-95 border-b-4 border-slate-950">
                <div class="text-5xl mb-3">👨‍🏫</div>
                <div class="text-xl font-bold text-slate-200">โหมดคุณครู</div>
                <p class="text-xs opacity-60 mt-2">ดูตารางอันดับคะแนน</p>
            </button>
        </div>
    </div>

    <!-- Passcode Modal -->
    <div id="modal-passcode" class="fixed inset-0 z-[150] flex items-center justify-center p-6 hidden">
        <div class="absolute inset-0 bg-black/60 backdrop-blur-sm" onclick="closePasscodeModal()"></div>
        <div class="glass w-full max-w-sm p-8 rounded-[2rem] relative shadow-2xl">
            <h3 class="text-2xl font-black text-center mb-2">ระบุรหัสผ่าน</h3>
            <input id="teacher-pass-input" type="password" placeholder="••••" class="w-full bg-slate-800 border-2 border-slate-700 rounded-2xl p-4 text-center text-3xl font-mono tracking-[0.5em] focus:border-purple-500 outline-none mb-6">
            <div class="grid grid-cols-2 gap-3">
                <button onclick="closePasscodeModal()" class="bg-slate-800 py-3 rounded-xl font-bold text-slate-400">ยกเลิก</button>
                <button onclick="verifyTeacherPass()" class="bg-purple-600 py-3 rounded-xl font-bold text-white">ตกลง</button>
            </div>
        </div>
    </div>

    <!-- Screen: Student Login -->
    <div id="screen-login" class="absolute inset-0 bg-slate-900 z-[90] hidden flex flex-col justify-center items-center p-6 text-center">
        <h2 class="text-3xl font-bold mb-6 text-purple-300">ยินดีต้อนรับ!</h2>
        <input id="student-name" type="text" maxlength="15" placeholder="ชื่อเล่นของคุณ..." class="w-full max-w-xs p-4 rounded-xl bg-slate-800 border-2 border-purple-500 text-center text-xl mb-6 outline-none text-white">
        <button id="btn-start" class="bg-gradient-to-r from-purple-600 to-pink-600 px-12 py-4 rounded-full font-black text-lg shadow-lg active:scale-95 transition-all">
            เริ่มภารกิจ!
        </button>
        <button onclick="location.reload()" class="mt-8 text-slate-500 underline text-sm">กลับหน้าหลัก</button>
    </div>

    <!-- Screen: Teacher Dashboard -->
    <div id="screen-teacher" class="absolute inset-0 bg-slate-900 z-[95] hidden flex flex-col p-6 overflow-hidden">
        <div class="flex justify-between items-center mb-6">
            <h2 class="text-3xl font-black text-white">👨‍🏫 Score Board</h2>
            <div class="flex gap-2">
                <button onclick="deleteLatestScore()" class="bg-orange-600/20 text-orange-400 px-4 py-2 rounded-xl text-xs border border-orange-600/50">ลบรายชื่อล่าสุด</button>
                <button onclick="location.reload()" class="bg-slate-800 px-4 py-2 rounded-xl text-xs">ออก</button>
            </div>
        </div>
        <div id="teacher-list" class="flex-1 overflow-y-auto space-y-2 custom-scrollbar pr-2">
            <!-- Data will be loaded here -->
        </div>
    </div>

    <!-- AR Game View -->
    <div id="game-view" class="relative w-full h-full hidden">
        <video id="input-video" class="hidden" playsinline></video>
        <canvas id="output-canvas" class="absolute inset-0 w-full h-full object-cover"></canvas>
        
        <div class="absolute inset-0 pointer-events-none p-4">
            <div id="hud" class="bg-black/60 backdrop-blur-md rounded-2xl p-4 border border-white/10 opacity-0 transition-opacity">
                <div id="question-text" class="text-yellow-400 text-lg font-bold text-center mb-2 italic">...</div>
                <div class="flex justify-between items-center px-4">
                    <span id="player-display" class="text-xs bg-purple-600 px-3 py-1 rounded-full text-white font-bold">👤 -</span>
                    <span id="score-display" class="text-white font-mono text-2xl font-bold">0</span>
                </div>
            </div>
        </div>

        <!-- Speech Detection Layer (EASY MODE) -->
        <div id="screen-speech" class="absolute inset-0 bg-black/90 z-[120] hidden flex flex-col justify-center items-center p-6 text-center">
            <div class="text-purple-400 text-sm font-bold uppercase tracking-widest mb-2">พูดคำนี้ออกเสียงดังๆ:</div>
            <div id="speech-target" class="text-7xl font-black mb-4 text-white uppercase tracking-wider scale-110">...</div>
            <div id="speech-hint" class="text-slate-400 text-sm italic mb-10">...</div>
            
            <div id="mic-indicator" class="w-28 h-28 bg-purple-600 rounded-full flex items-center justify-center mic-active mb-6 shadow-2xl">
                <span id="mic-emoji" class="text-5xl">🎙️</span>
            </div>
            
            <div class="w-full max-w-xs bg-slate-800/50 rounded-full h-2 mb-4 overflow-hidden border border-slate-700">
                <div id="audio-bar" class="h-full bg-purple-400 w-0 transition-all duration-100"></div>
            </div>

            <div id="speech-feedback" class="h-8 text-xl font-bold text-slate-300 mb-2 italic">กำลังรอฟัง...</div>
            <p id="speech-instruction" class="text-slate-500 text-sm">ไม่ต้องกังวลเรื่องสำเนียง แค่ออกเสียงให้ชัดที่สุด!</p>
            
            <button onclick="skipSpeech()" class="mt-12 text-slate-600 text-xs hover:text-slate-400 underline">กดเพื่อข้ามหากไมโครโฟนไม่ทำงาน</button>
        </div>
    </div>

    <!-- Screen: End Game -->
    <div id="screen-end" class="absolute inset-0 bg-slate-900/95 z-[90] hidden flex flex-col justify-center items-center p-6 text-center">
        <div class="bg-slate-800 p-8 rounded-3xl border border-slate-700 max-w-sm w-full">
            <h2 class="text-4xl font-black mb-2 text-white">เยี่ยมมาก!</h2>
            <div class="bg-slate-900 rounded-2xl p-6 my-8 border border-purple-500/30">
                <div class="text-xs text-purple-400 font-bold mb-1">คะแนนที่คุณทำได้</div>
                <div id="final-score" class="text-6xl font-black text-yellow-400">0</div>
            </div>
            <button onclick="location.reload()" class="w-full bg-purple-600 py-4 rounded-xl font-bold">🏠 จบการเรียนรู้</button>
        </div>
    </div>

<script>
const CONFIG = { mouthThreshold: 12, eatDist: 75, gravity: 2.3, teacherPass: "1234" };
const questions = [
    { q: "She ___ to school every day.", c: "Goes", w: ["Go", "Going"], hint: "ออกเสียงคล้าย 'โกส์'" },
    { q: "The apple is ___ the table.", c: "On", w: ["In", "Under"], hint: "ออกเสียงสั้นๆ 'ออน'" },
    { q: "A person who teaches is a ___.", c: "Teacher", w: ["Doctor", "Nurse"], hint: "ออกเสียง 'ที-เชอร์'" },
    { q: "What is the capital of Thailand?", c: "Bangkok", w: ["Phuket", "Chiangmai"], hint: "ออกเสียง 'แบง-ค็อก'" },
    { q: "I have a cat. ___ name is Kitty.", c: "Its", w: ["It", "His"], hint: "ออกเสียง 'อิทส์'" }
];

let state = {
    active: false, score: 0, qIndex: 0, items: [], 
    mouth: { x: 0, y: 0, open: false },
    playerName: "", isSpeech: false,
    recognition: null, isSuccessProcessed: false
};

const ctx = document.getElementById('output-canvas').getContext('2d');

// --- Enhanced Speech Logic (Easy Mode) ---
function initSpeechRecognition() {
    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
    if (!SpeechRecognition) return null;

    const rec = new SpeechRecognition();
    rec.lang = 'en-US';
    rec.continuous = true;
    rec.interimResults = true;
    
    rec.onresult = (event) => {
        if (state.isSuccessProcessed) return;

        const transcript = event.results[event.results.length - 1][0].transcript.trim().toLowerCase();
        const target = document.getElementById('speech-target').innerText.toLowerCase();
        
        // Simulating audio bar feedback
        const vol = Math.min(100, transcript.length * 10);
        document.getElementById('audio-bar').style.width = `${vol}%`;
        
        document.getElementById('speech-feedback').innerText = `ได้ยินว่า: "${transcript}"`;
        
        // EASY MATCHING LOGIC
        const isMatch = transcript.includes(target) || 
                        fuzzyCheck(transcript, target) ||
                        (transcript.length > 2 && target.toLowerCase().startsWith(transcript.substring(0, 3)));

        if (isMatch) handleSpeechSuccess();
    };

    rec.onend = () => {
        if (state.isSpeech && !state.isSuccessProcessed) {
            try { rec.start(); } catch(e) {}
        }
    };

    return rec;
}

// Simple fuzzy check to make it easier for kids
function fuzzyCheck(heard, target) {
    if (heard === target) return true;
    // Allow for common mishearing (e.g., 'Goes' might be heard as 'Go' or 'Ghost')
    if (target === "goes" && (heard.includes("go") || heard.includes("ghost"))) return true;
    if (target === "teacher" && (heard.includes("teach") || heard.includes("shirt"))) return true;
    if (target === "its" && (heard.includes("it") || heard.includes("is"))) return true;
    return false;
}

function handleSpeechSuccess() {
    if (state.isSuccessProcessed) return;
    state.isSuccessProcessed = true;

    const feedback = document.getElementById('speech-feedback');
    feedback.innerText = "ถูกต้อง! เก่งมาก ✨";
    feedback.classList.add('text-green-400', 'success-pop');
    
    document.getElementById('mic-indicator').classList.replace('bg-purple-600', 'bg-green-600');
    document.getElementById('mic-emoji').innerText = "👏";
    document.getElementById('audio-bar').style.width = "100%";
    document.getElementById('audio-bar').classList.replace('bg-purple-400', 'bg-green-400');
    
    if (state.recognition) try { state.recognition.stop(); } catch(e) {}
    
    setTimeout(() => {
        document.getElementById('screen-speech').classList.add('hidden');
        resetSpeechUI();
        state.isSpeech = false;
        state.active = true;
        state.qIndex++;
        if (state.qIndex >= questions.length) finish();
        updateUI();
    }, 1500);
}

function resetSpeechUI() {
    const feedback = document.getElementById('speech-feedback');
    feedback.classList.remove('text-green-400', 'success-pop');
    document.getElementById('mic-indicator').classList.replace('bg-green-600', 'bg-purple-600');
    document.getElementById('mic-emoji').innerText = "🎙️";
    document.getElementById('audio-bar').style.width = "0%";
    document.getElementById('audio-bar').classList.replace('bg-green-400', 'bg-purple-400');
}

function skipSpeech() {
    handleSpeechSuccess();
}

// --- App Flow ---
window.showLogin = () => {
    document.getElementById('screen-menu').style.display = 'none';
    document.getElementById('screen-login').classList.remove('hidden');
};

window.openPasscodeModal = () => document.getElementById('modal-passcode').classList.remove('hidden');
window.closePasscodeModal = () => document.getElementById('modal-passcode').classList.add('hidden');

window.verifyTeacherPass = () => {
    if (document.getElementById('teacher-pass-input').value === CONFIG.teacherPass) {
        closePasscodeModal();
        document.getElementById('screen-menu').style.display = 'none';
        document.getElementById('screen-teacher').classList.remove('hidden');
        loadTeacherData();
    } else {
        alert("รหัสผ่านผิด!");
    }
};

document.getElementById('btn-start').onclick = () => {
    const name = document.getElementById('student-name').value.trim();
    if (!name) return alert("กรุณาพิมพ์ชื่อเล่นก่อนจ้า");
    state.playerName = name;
    state.recognition = initSpeechRecognition();
    document.getElementById('player-display').innerText = `👤 ${name}`;
    document.getElementById('screen-login').classList.add('hidden');
    document.getElementById('game-view').classList.remove('hidden');
    initAR();
};

// --- AR Logic ---
function initAR() {
    const faceMesh = new FaceMesh({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}` });
    faceMesh.setOptions({ maxNumFaces: 1, refineLandmarks: true, minDetectionConfidence: 0.7 });
    faceMesh.onResults(onResults);
    
    const camera = new Camera(document.getElementById('input-video'), {
        onFrame: async () => { await faceMesh.send({image: document.getElementById('input-video')}); },
        width: 1280, height: 720
    });
    
    const canv = document.getElementById('output-canvas');
    canv.width = window.innerWidth;
    canv.height = window.innerHeight;
    
    camera.start().then(() => {
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
        const top = lm[13]; const bottom = lm[14];
        const dist = Math.abs(top.y - bottom.y) * ctx.canvas.height;
        state.mouth.x = (1 - top.x) * ctx.canvas.width;
        state.mouth.y = top.y * ctx.canvas.height;
        state.mouth.open = dist > CONFIG.mouthThreshold;
        
        ctx.beginPath();
        ctx.arc(state.mouth.x, state.mouth.y, 14, 0, Math.PI*2);
        ctx.fillStyle = state.mouth.open ? "#a855f7" : "rgba(255,255,255,0.8)";
        ctx.fill();
        if (state.mouth.open) {
            ctx.strokeStyle = "white"; ctx.lineWidth = 2; ctx.stroke();
        }
    }
    if (state.active && !state.isSpeech) updateGame();
}

function updateGame() {
    if (Math.random() < 0.035 && state.items.length < 4) spawnItem();
    state.items.forEach((item, idx) => {
        item.y += CONFIG.gravity;
        
        ctx.save();
        ctx.fillStyle = item.isCorrect ? "rgba(88, 28, 135, 0.9)" : "rgba(30, 41, 59, 0.9)";
        ctx.beginPath(); ctx.roundRect(item.x - 75, item.y - 35, 150, 70, 20); ctx.fill();
        ctx.strokeStyle = item.isCorrect ? "#d8b4fe" : "#475569";
        ctx.lineWidth = 3; ctx.stroke();
        ctx.fillStyle = "white"; ctx.font = "bold 24px sans-serif"; ctx.textAlign = "center";
        ctx.fillText(item.text, item.x, item.y + 10);
        ctx.restore();

        const d = Math.hypot(item.x - state.mouth.x, item.y - state.mouth.y);
        if (d < CONFIG.eatDist && state.mouth.open) {
            state.items.splice(idx, 1);
            if (item.isCorrect) {
                state.score += 10;
                syncScore();
                startSpeechPhase(item.text);
            } else {
                state.score = Math.max(0, state.score - 5);
                syncScore();
            }
            updateUI();
        }
        if (item.y > ctx.canvas.height + 100) state.items.splice(idx, 1);
    });
}

function spawnItem() {
    const q = questions[state.qIndex]; if(!q) return;
    const pool = [q.c, ...q.w];
    const text = pool[Math.floor(Math.random() * pool.length)];
    state.items.push({ x: 120 + Math.random()*(ctx.canvas.width-240), y: -60, text, isCorrect: text === q.c });
}

function startSpeechPhase(text) {
    state.active = false;
    state.isSpeech = true;
    state.isSuccessProcessed = false;
    state.items = [];
    
    document.getElementById('screen-speech').classList.remove('hidden');
    document.getElementById('speech-target').innerText = text;
    document.getElementById('speech-hint').innerText = questions[state.qIndex]?.hint || "";
    document.getElementById('speech-feedback').innerText = "รอฟังเสียง...";
    
    if (state.recognition) {
        try { state.recognition.start(); } catch(e) {}
    } else {
        setTimeout(skipSpeech, 4000);
    }
}

async function syncScore() {
    if (!window.fb || !state.playerName || !window.fbUser) return;
    const { db, appId, doc, setDoc } = window.fb;
    const scoreDoc = doc(db, 'artifacts', appId, 'public', 'data', 'scores', window.fbUser.uid);
    try {
        await setDoc(scoreDoc, { name: state.playerName, score: state.score, updatedAt: Date.now() }, { merge: true });
    } catch (e) {}
}

function loadTeacherData() {
    if (!window.fb) return;
    const { db, appId, collection, onSnapshot } = window.fb;
    onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'scores'), (snap) => {
        let scores = []; snap.forEach(d => scores.push(d.data()));
        scores.sort((a,b) => b.score - a.score);
        document.getElementById('teacher-list').innerHTML = scores.map((s,i) => `
            <div class="flex items-center justify-between bg-slate-800 p-4 rounded-2xl border border-slate-700">
                <div class="flex items-center gap-4">
                    <span class="text-xl font-black ${i<3?'text-yellow-400':'text-slate-500'}">#${i+1}</span>
                    <span class="font-bold text-white">${s.name}</span>
                </div>
                <span class="text-2xl font-mono font-black text-purple-400">${s.score}</span>
            </div>
        `).join('');
    });
}

window.deleteLatestScore = async () => {
    if (!window.fb) return;
    const { db, appId, collection, getDocs, deleteDoc, doc } = window.fb;
    const snapshot = await getDocs(collection(db, 'artifacts', appId, 'public', 'data', 'scores'));
    if (snapshot.empty) return;
    let scores = []; snapshot.forEach(d => scores.push({id: d.id, ...d.data()}));
    scores.sort((a,b) => b.updatedAt - a.updatedAt);
    if (confirm(`ลบคะแนนของ "${scores[0].name}" ?`)) {
        await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'scores', scores[0].id));
    }
};

function updateUI() {
    document.getElementById('score-display').innerText = state.score;
    if(questions[state.qIndex]) document.getElementById('question-text').innerText = `ข้อที่ ${state.qIndex+1}: ${questions[state.qIndex].q}`;
}

function finish() {
    state.active = false;
    document.getElementById('game-view').classList.add('hidden');
    document.getElementById('screen-end').classList.remove('hidden');
    document.getElementById('final-score').innerText = state.score;
}

window.onresize = () => {
    ctx.canvas.width = window.innerWidth;
    ctx.canvas.height = window.innerHeight;
};
</script>
</body>
</html>
