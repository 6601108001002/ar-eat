<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">
    <title>AR English Eat - M.1 Adventure</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/face_mesh.js" crossorigin="anonymous"></script>
    <script src="https://cdn.sheetjs.com/xlsx-0.20.1/package/dist/xlsx.full.min.js"></script>
    <link href="https://fonts.googleapis.com/css2?family=Prompt:wght@300;400;600;800&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Prompt', sans-serif; background: #0f172a; overscroll-behavior: none; }
        canvas { width: 100%; height: 100%; object-fit: cover; } 
        .glass { background: rgba(17, 24, 39, 0.8); backdrop-filter: blur(8px); border: 1px solid rgba(255,255,255,0.1); }
        .btn-game { background: linear-gradient(to bottom, #a855f7, #7e22ce); box-shadow: 0 4px 0 #581c87; transition: all 0.1s; }
        .btn-game:active { transform: translateY(4px); box-shadow: 0 0 0 #581c87; }
        .loader { width: 40px; height: 40px; border: 4px solid #FFF; border-bottom-color: #a855f7; border-radius: 50%; animation: rot 1s linear infinite; }
        @keyframes rot { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
        @keyframes float { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-10px); } }
        .floating { animation: float 3s ease-in-out infinite; }
    </style>
</head>
<body class="h-screen w-screen overflow-hidden text-white select-none">

    <!-- Game Layer -->
    <div id="game-container" class="relative w-full h-full flex justify-center items-center">
        <video class="hidden" playsinline></video>
        <canvas id="output-canvas" class="absolute inset-0"></canvas>
        
        <!-- Quick Loader -->
        <div id="quick-loader" class="absolute z-20 flex flex-col items-center justify-center pointer-events-none hidden">
            <span class="loader mb-2"></span>
            <p class="text-white text-xs bg-black/50 px-3 py-1 rounded-full">เปิดกล้อง...</p>
        </div>

        <!-- HUD -->
        <div id="ui-layer" class="absolute inset-0 pointer-events-none flex flex-col justify-between p-4 z-10">
            <div id="hud" class="glass rounded-2xl p-3 flex flex-col gap-2 transition-opacity duration-300 hidden">
                <div class="flex justify-between items-center px-1">
                    <div class="flex items-center gap-3">
                        <div class="w-10 h-10 bg-purple-600 rounded-lg flex items-center justify-center font-bold text-xl shadow-lg" id="score-box">0</div>
                        <div class="text-xs text-gray-300"><div class="font-bold text-white">SCORE</div><div id="player-label">Guest</div></div>
                    </div>
                    <button onclick="readQuestion()" class="pointer-events-auto bg-white/10 p-2 rounded-full hover:bg-white/20 active:scale-95">🔊 โจทย์</button>
                </div>
                <div id="question-text" class="text-center text-yellow-300 text-lg md:text-xl font-bold py-1 bg-black/20 rounded-lg">Loading...</div>
            </div>
            <div id="toast" class="self-center glass px-6 py-2 rounded-full text-sm font-bold text-red-400 hidden animate-bounce">Wrong!</div>
        </div>

        <!-- Login Screen -->
        <div id="screen-login" class="absolute inset-0 bg-[#0f172a] z-50 flex flex-col justify-center items-center p-6">
            <button onclick="openAdmin()" class="absolute top-4 right-4 text-xs text-gray-500 pointer-events-auto">🔒 Admin</button>
            <div class="text-center mb-6 floating">
                <h1 class="text-4xl font-extrabold text-transparent bg-clip-text bg-gradient-to-r from-purple-400 to-pink-500">AR English Eat</h1>
                <p class="text-gray-400 text-sm">M.1 Adventure</p>
            </div>
            <div class="glass p-6 rounded-3xl w-full max-w-sm space-y-3 shadow-2xl">
                <input type="text" id="inp-name" class="w-full bg-gray-800/50 border border-gray-600 rounded-xl p-3 text-white focus:border-purple-500 outline-none" placeholder="ชื่อเล่น">
                <input type="number" id="inp-num" class="w-full bg-gray-800/50 border border-gray-600 rounded-xl p-3 text-white focus:border-purple-500 outline-none" placeholder="เลขที่">
                <button onclick="startGameFlow()" class="w-full btn-game py-3 rounded-xl font-bold text-lg pointer-events-auto mt-2">🚀 เริ่มเลย</button>
            </div>
        </div>

        <!-- Admin -->
        <div id="screen-admin" class="absolute inset-0 bg-black/95 z-[60] hidden flex-col justify-center items-center p-6">
            <div id="admin-login" class="glass p-6 rounded-2xl w-full max-w-xs text-center">
                <h3 class="font-bold mb-4">🔑 Admin</h3>
                <input type="password" id="admin-pass" class="w-full bg-gray-800 p-2 rounded mb-4 text-center border border-gray-600" placeholder="PIN">
                <div class="flex gap-2">
                    <button onclick="closeAdmin()" class="flex-1 py-2 bg-gray-700 rounded">Cancel</button>
                    <button onclick="checkAdmin()" class="flex-1 py-2 bg-blue-600 rounded font-bold">Login</button>
                </div>
            </div>
            <div id="admin-panel" class="hidden w-full max-w-md h-3/4 glass rounded-xl flex flex-col">
                <div class="p-3 border-b border-gray-700 flex justify-between bg-gray-800">
                    <span class="font-bold text-purple-400">Scoreboard</span>
                    <button onclick="closeAdmin()" class="text-xs bg-gray-700 px-2 py-1 rounded">Close</button>
                </div>
                <div class="flex-1 overflow-auto p-0">
                    <table class="w-full text-sm text-left text-gray-300">
                        <thead class="bg-gray-900 text-xs uppercase"><tr><th class="p-3">Time</th><th class="p-3">Name</th><th class="p-3 text-right">Score</th></tr></thead>
                        <tbody id="admin-list" class="divide-y divide-gray-700"></tbody>
                    </table>
                </div>
                <div class="p-3 bg-gray-800 flex justify-between">
                    <button onclick="loadAdminData()" class="text-blue-400 text-xs">↻ Refresh</button>
                    <button onclick="exportAll()" class="bg-green-600 px-3 py-1 rounded text-xs font-bold">📥 Excel</button>
                </div>
            </div>
        </div>

        <!-- Speech -->
        <div id="screen-speech" class="absolute inset-0 bg-black/90 z-30 hidden flex-col justify-center items-center text-center p-6 backdrop-blur-md">
            <h2 class="text-xl text-purple-300 font-bold mb-2">พูดคำนี้</h2>
            <div id="sp-word" class="text-5xl font-extrabold text-white mb-2 tracking-wide drop-shadow-lg">...</div>
            <div id="sp-meaning" class="text-xl text-yellow-400 font-bold bg-gray-800 px-4 py-2 rounded-full border border-yellow-500/30 mb-8">...</div>
            <div class="relative w-32 h-32 flex items-center justify-center">
                <div class="absolute inset-0 bg-purple-600 rounded-full opacity-20 animate-ping"></div>
                <div class="text-4xl">🎙️</div>
            </div>
            <p id="sp-status" class="mt-8 text-gray-400">กำลังฟัง...</p>
            <button onclick="endSpeech(false)" class="mt-4 text-xs text-gray-500 underline pointer-events-auto">ข้าม</button>
        </div>

        <!-- End Screen -->
        <div id="screen-end" class="absolute inset-0 bg-[#0f172a]/95 z-50 hidden flex-col justify-center items-center p-6 text-center">
            <h1 class="text-5xl font-black text-white mb-2">จบเกม!</h1>
            <div class="text-gray-400 mb-6"><span id="end-player" class="text-purple-400 font-bold text-xl">Guest</span></div>
            <div class="bg-gray-800/50 p-8 rounded-full border-4 border-yellow-400/50 mb-6 w-40 h-40 flex items-center justify-center">
                <div><div class="text-xs text-gray-400">SCORE</div><div id="final-score" class="text-5xl font-black text-yellow-400">0</div></div>
            </div>
            <div id="save-msg" class="h-6 text-sm text-green-400 font-bold mb-4"></div>
            <button onclick="exportSelf()" class="w-full max-w-xs bg-green-600 py-3 rounded-xl font-bold mb-3 pointer-events-auto">📥 โหลดคะแนน</button>
            <button onclick="location.reload()" class="w-full max-w-xs bg-gray-700 py-3 rounded-xl font-bold pointer-events-auto">🔄 เล่นอีกครั้ง</button>
        </div>
    </div>

<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
import { getAuth, signInAnonymously } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
import { getFirestore, collection, addDoc, getDocs, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

const app = initializeApp(JSON.parse(__firebase_config));
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

signInAnonymously(auth).catch(console.error);

window.saveScore = async () => {
    if (!window.G) return;
    const msg = document.getElementById('save-msg');
    msg.innerText = "กำลังบันทึก...";
    if (!auth.currentUser) { setTimeout(window.saveScore, 1000); return; }
    try {
        await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'scores'), {
            name: window.G.user.name, number: window.G.user.number, score: window.G.score,
            history: window.G.history, timestamp: serverTimestamp(), date: new Date().toLocaleString()
        });
        msg.innerText = "✅ บันทึกแล้ว";
    } catch (e) { msg.innerText = "⚠️ บันทึกไม่ได้ (ใช้ Excel แทน)"; }
};

window.loadAdminData = async () => {
    const list = document.getElementById('admin-list');
    list.innerHTML = '<tr><td colspan="3" class="p-4 text-center">Loading...</td></tr>';
    if (!auth.currentUser) return;
    try {
        const snap = await getDocs(collection(db, 'artifacts', appId, 'public', 'data', 'scores'));
        let docs = [];
        snap.forEach(d => docs.push(d.data()));
        docs.sort((a,b) => (b.timestamp?.seconds || 0) - (a.timestamp?.seconds || 0));
        window.adminCache = docs;
        list.innerHTML = docs.map(d => `<tr class="border-b border-gray-700"><td class="p-3 font-mono">${d.date?.split(',')[0]||'-'}</td><td class="p-3"><div class="font-bold text-white">${d.name}</div><div class="text-xs text-yellow-500">#${d.number}</div></td><td class="p-3 text-right font-bold text-green-400">${d.score}</td></tr>`).join('');
    } catch(e) { list.innerHTML = '<tr><td colspan="3" class="p-4 text-center text-red-400">Error</td></tr>'; }
};
</script>

<script>
const CFG = { gravity: 1.5, spawnRate: 110, eatDist: 100, mouthThresh: 0.04 };
window.G = { playing: false, score: 0, qIdx: 0, items: [], history: [], mouth: {x:0, y:0, open:false}, user: {name:'', number:''}, pendingItem: null };

// QUESTIONS: Cleaned up (No phonetic, just Meaning 'm')
const QUESTIONS = [
    { q: "Yesterday, I ___ to school.", c: "went", w: ["go","goes"], m: "ไป (อดีต)" },
    { q: "I don't have ___ money.", c: "any", w: ["some","a"], m: "บ้าง (ใช้ในประโยคปฏิเสธ)" },
    { q: "This book is ___.", c: "mine", w: ["my","me"], m: "ของฉัน" },
    { q: "Opposite of 'Small'", c: "Big", w: ["Short","Long"], m: "ใหญ่" },
    { q: "A ___ teaches students.", c: "Teacher", w: ["Doctor","Pilot"], m: "ครู" },
    { q: "___ you speak English?", c: "Do", w: ["Are","Is"], m: "ใช้นำหน้าประโยคคำถาม" },
    { q: "She is ___ than me.", c: "taller", w: ["tall","tallest"], m: "สูงกว่า" },
    { q: "See you ___ Monday.", c: "on", w: ["in","at"], m: "ใช้กับ วัน (จันทร์-อาทิตย์)" },
    { q: "We ___ eating now.", c: "are", w: ["is","do"], m: "เป็น/อยู่/คือ (ใช้กับ We)" },
    { q: "Fish ___ in water.", c: "swim", w: ["fly","run"], m: "ว่ายน้ำ" }
];

async function initCamera() {
    document.getElementById('quick-loader').classList.remove('hidden');
    const video = document.querySelector('video');
    const faceMesh = new FaceMesh({locateFile: f => `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${f}`});
    faceMesh.setOptions({maxNumFaces: 1, minDetectionConfidence: 0.5, minTrackingConfidence: 0.5});
    faceMesh.onResults(onFaceResults);
    const camera = new Camera(video, { onFrame: async () => await faceMesh.send({image: video}), width: 1280, height: 720 });
    await camera.start();
}

const ctx = document.getElementById('output-canvas').getContext('2d');
let camReady = false;

function onFaceResults(res) {
    const w = ctx.canvas.width = window.innerWidth;
    const h = ctx.canvas.height = window.innerHeight;
    ctx.save(); ctx.translate(w, 0); ctx.scale(-1, 1); ctx.clearRect(0, 0, w, h); ctx.drawImage(res.image, 0, 0, w, h); ctx.restore();

    if (res.multiFaceLandmarks && res.multiFaceLandmarks.length > 0) {
        const lm = res.multiFaceLandmarks[0];
        const uLip = lm[13], lLip = lm[14], head = lm[10], chin = lm[152];
        G.mouth.open = Math.abs(uLip.y - lLip.y) > (Math.abs(head.y - chin.y) * CFG.mouthThresh);
        G.mouth.x = (1 - ((uLip.x + lLip.x) / 2)) * w;
        G.mouth.y = ((uLip.y + lLip.y) / 2) * h;

        ctx.beginPath(); ctx.arc(G.mouth.x, G.mouth.y, G.mouth.open ? 40 : 20, 0, 2*Math.PI);
        ctx.lineWidth = 4; ctx.strokeStyle = G.mouth.open ? '#4ade80' : '#f472b6'; ctx.stroke();
        
        if (!camReady) { camReady = true; document.getElementById('quick-loader').classList.add('hidden'); realStart(); }
    }
}

function gameLoop() {
    if (!G.playing) return;
    if (Math.random() * 1000 < (1000/CFG.spawnRate)) spawnItem();

    ctx.font = "bold 28px 'Prompt'"; ctx.textAlign = "center"; ctx.textBaseline = "middle";
    for (let i = G.items.length - 1; i >= 0; i--) {
        let it = G.items[i]; it.y += it.vy; it.x += it.vx;
        const tw = ctx.measureText(it.text).width + 30, th = 46;
        
        ctx.fillStyle = it.type === 'c' ? "rgba(34, 197, 94, 0.9)" : "rgba(239, 68, 68, 0.9)";
        ctx.beginPath(); ctx.roundRect(it.x - tw/2, it.y - th/2, tw, th, 15); ctx.fill();
        ctx.fillStyle = "white"; ctx.fillText(it.text, it.x, it.y);

        if (Math.sqrt((it.x-G.mouth.x)**2 + (it.y-G.mouth.y)**2) < CFG.eatDist && G.mouth.open) handleEat(it, i);
        else if (it.y > ctx.canvas.height + 50) G.items.splice(i, 1);
    }
    requestAnimationFrame(gameLoop);
}

function spawnItem() {
    const q = QUESTIONS[G.qIdx];
    const isCorrect = Math.random() > 0.4;
    G.items.push({
        x: Math.random() * (ctx.canvas.width - 100) + 50, y: -50,
        text: isCorrect ? q.c : q.w[Math.floor(Math.random()*q.w.length)],
        type: isCorrect ? 'c' : 'w', vx: (Math.random()-0.5)*1.5, vy: Math.random()*1.5 + CFG.gravity
    });
}

function handleEat(item, idx) {
    if (item.type === 'c') {
        G.items = []; G.pendingItem = item; G.playing = false;
        const q = QUESTIONS[G.qIdx];
        document.getElementById('screen-speech').classList.remove('hidden');
        document.getElementById('screen-speech').classList.add('flex');
        document.getElementById('sp-word').innerText = item.text;
        document.getElementById('sp-meaning').innerText = q.m; // Just meaning
        startSpeech();
    } else {
        G.items.splice(idx, 1); G.score = Math.max(0, G.score - 5); updateHUD();
        showToast(`กินผิด! ${item.text} -5`, 'red');
        G.history.push({q:QUESTIONS[G.qIdx].q, ans:item.text, res:'Wrong'});
    }
}

const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
let recognition;

function startSpeech() {
    const status = document.getElementById('sp-status');
    status.innerText = "กำลังฟัง... (พูดเลย!)"; status.className = "mt-8 text-yellow-400 animate-pulse font-bold";
    if (SpeechRecognition) {
        recognition = new SpeechRecognition(); recognition.lang = 'en-US'; recognition.interimResults = true;
        recognition.onresult = (e) => {
            const t = e.results[e.results.length-1][0].transcript.toLowerCase(), target = G.pendingItem.text.toLowerCase();
            if (t.includes(target) || target.includes(t) || (t[0]===target[0] && Math.abs(t.length-target.length)<3)) {
                recognition.stop(); endSpeech(true);
            }
        };
        recognition.start();
    } else { status.innerText = "ไม่รองรับเสียง (รอสักครู่...)"; setTimeout(() => endSpeech(false), 2000); }
}

function endSpeech(success) {
    if (recognition) try{recognition.stop()}catch(e){};
    document.getElementById('screen-speech').classList.add('hidden');
    document.getElementById('screen-speech').classList.remove('flex');
    G.score += (10 + (success?5:0));
    showToast(success ? `เยี่ยม! +${15}` : `ผ่าน! +10`, 'green');
    G.history.push({q:QUESTIONS[G.qIdx].q, ans:G.pendingItem.text, res:'Correct'});
    if (++G.qIdx >= QUESTIONS.length) finishGame();
    else { G.playing = true; loadLevel(); gameLoop(); }
}

function startGameFlow() {
    const name = document.getElementById('inp-name').value.trim();
    if (!name) return alert("กรอกชื่อก่อน!");
    G.user = {name, number: document.getElementById('inp-num').value.trim()};
    document.getElementById('player-label').innerText = name;
    document.getElementById('end-player').innerText = G.user.name;
    document.getElementById('screen-login').classList.add('hidden');
    initCamera();
}

function realStart() {
    G.playing = true; G.score = 0; G.qIdx = 0;
    document.getElementById('hud').classList.remove('hidden'); document.getElementById('hud').classList.add('flex');
    loadLevel(); gameLoop();
}

function loadLevel() {
    document.getElementById('question-text').innerText = `${G.qIdx+1}. ${QUESTIONS[G.qIdx].q}`;
    updateHUD(); readQuestion();
}

function finishGame() {
    G.playing = false;
    document.getElementById('screen-end').classList.remove('hidden'); document.getElementById('screen-end').classList.add('flex');
    document.getElementById('final-score').innerText = G.score;
    if(window.saveScore) window.saveScore();
}

function updateHUD() { document.getElementById('score-box').innerText = G.score; }
function showToast(msg, c) { const t=document.getElementById('toast'); t.innerText=msg; t.className=`self-center glass px-6 py-2 rounded-full text-sm font-bold animate-bounce text-${c}-400`; t.classList.remove('hidden'); setTimeout(()=>t.classList.add('hidden'),2000); }
window.readQuestion = () => { if(G.playing) { const u = new SpeechSynthesisUtterance(QUESTIONS[G.qIdx].q); u.lang='en-US'; speechSynthesis.speak(u); }};

window.openAdmin = () => { document.getElementById('screen-admin').classList.remove('hidden'); document.getElementById('screen-admin').classList.add('flex'); };
window.closeAdmin = () => { document.getElementById('screen-admin').classList.add('hidden'); document.getElementById('screen-admin').classList.remove('flex'); };
window.checkAdmin = () => { if(document.getElementById('admin-pass').value==='1234'){document.getElementById('admin-login').classList.add('hidden');document.getElementById('admin-panel').classList.remove('hidden');document.getElementById('admin-panel').classList.add('flex');if(window.loadAdminData)window.loadAdminData();}else alert("ผิด!"); };

window.exportSelf = () => doExport([["Item","Result"],...G.history.map(h=>[h.ans,h.res])], `Score_${G.user.name}`);
window.exportAll = () => window.adminCache ? doExport([["Time","Name","Score"],...window.adminCache.map(d=>[d.date,d.name,d.score])], "Scores") : alert("No Data");
function doExport(data,f){const ws=XLSX.utils.aoa_to_sheet(data),wb=XLSX.utils.book_new();XLSX.utils.book_append_sheet(wb,ws,"S1");XLSX.writeFile(wb,`${f}.xlsx`);}
window.onresize = () => { if(G.playing){ctx.canvas.width=window.innerWidth;ctx.canvas.height=window.innerHeight;} };
</script>
</body>
</html>
