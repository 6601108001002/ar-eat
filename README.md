<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>AR Eat - Compact</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/face_mesh.js" crossorigin="anonymous"></script>
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, doc, setDoc, onSnapshot } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        const cfg = JSON.parse(__firebase_config || "{}"), app = initializeApp(cfg), auth = getAuth(app), db = getFirestore(app);
        window.fb = { db, auth, appId: typeof __app_id !== 'undefined' ? __app_id : 'eng-eat-compact', setDoc, doc, collection, onSnapshot };
        onAuthStateChanged(auth, u => window.u = u);
        (async () => { (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) ? await signInWithCustomToken(auth, __initial_auth_token) : await signInAnonymously(auth); })();
    </script>
    <style>
        body { background:#0f172a; overscroll-behavior:none; }
        canvas { width:100%; height:100%; object-fit:cover; }
        .glass { background:rgba(15,23,42,0.8); backdrop-filter:blur(10px); }
        .mic-on { animation: pulse 1s infinite; }
        @keyframes pulse { 0%,100% { transform: scale(1); } 50% { transform: scale(1.1); box-shadow: 0 0 20px #a855f7; } }
    </style>
</head>
<body class="h-screen w-screen overflow-hidden select-none text-white font-sans">
    <div id="loader" class="fixed inset-0 z-[200] bg-slate-900 flex flex-col justify-center items-center opacity-0 transition-opacity pointer-events-none">
        <div class="w-10 h-10 border-4 border-purple-500 border-t-transparent rounded-full animate-spin mb-2"></div>
        <p class="text-purple-300 text-xs">กำลังเริ่ม...</p>
    </div>
    <div id="scr-menu" class="absolute inset-0 z-50 bg-slate-900 flex flex-col justify-center items-center p-6 text-center">
        <h1 class="text-5xl font-black mb-10 bg-gradient-to-r from-purple-400 to-pink-500 bg-clip-text text-transparent">AR Eat</h1>
        <div class="space-y-4 w-full max-w-xs">
            <button onclick="nav('scr-login')" class="w-full bg-purple-600 p-6 rounded-2xl font-bold">โหมดนักเรียน</button>
            <button onclick="p_modal.classList.remove('hidden')" class="w-full bg-slate-800 p-6 rounded-2xl font-bold">โหมดคุณครู</button>
        </div>
    </div>
    <div id="p_modal" class="fixed inset-0 z-[60] flex items-center justify-center p-6 hidden">
        <div class="absolute inset-0 bg-black/60" onclick="this.parentElement.classList.add('hidden')"></div>
        <div class="glass p-6 rounded-3xl relative w-full max-w-xs">
            <input id="pin" type="password" placeholder="Passcode" class="w-full bg-slate-800 p-4 rounded-xl mb-4 text-center outline-none">
            <button onclick="chkPin()" class="w-full bg-purple-600 py-3 rounded-xl font-bold">ตกลง</button>
        </div>
    </div>
    <div id="scr-login" class="absolute inset-0 z-40 bg-slate-900 hidden flex flex-col justify-center items-center p-6">
        <input id="s-name" type="text" placeholder="ชื่อของคุณ" class="w-full max-w-xs p-4 rounded-xl bg-slate-800 border-2 border-purple-500 text-center mb-6 outline-none">
        <button onclick="play()" class="bg-purple-600 px-12 py-4 rounded-full font-black">เริ่มเล่น</button>
    </div>
    <div id="scr-teacher" class="absolute inset-0 z-[45] bg-slate-900 hidden flex flex-col p-6">
        <div class="flex justify-between mb-4"><h2 class="text-xl font-bold">Leaderboard</h2><button onclick="location.reload()">ออก</button></div>
        <div id="s-list" class="flex-1 overflow-y-auto space-y-2"></div>
    </div>
    <div id="g-view" class="relative w-full h-full hidden">
        <video id="vid" class="hidden" playsinline muted autoplay></video>
        <canvas id="cvs"></canvas>
        <div id="hud" class="absolute inset-x-0 top-10 flex flex-col items-center pointer-events-none opacity-0 transition-opacity">
            <div id="q-box" class="glass px-6 py-2 rounded-full text-yellow-400 font-bold mb-2">...</div>
            <div class="text-2xl font-black">Score: <span id="s-num">0</span></div>
        </div>
        <div id="scr-sp" class="absolute inset-0 z-[100] bg-black/90 hidden flex flex-col justify-center items-center p-6 text-center">
            <div id="sp-t" class="text-6xl font-black mb-2 uppercase">...</div>
            <div id="sp-h" class="text-slate-500 mb-8 italic text-sm">...</div>
            <div class="w-20 h-20 bg-purple-600 rounded-full flex items-center justify-center mic-on text-3xl mb-6">🎙️</div>
            <div class="w-48 h-1 bg-slate-800 rounded-full overflow-hidden mb-4"><div id="v-bar" class="h-full bg-purple-400 w-0 transition-all"></div></div>
            <button onclick="spOk()" class="text-slate-600 underline text-xs">Skip</button>
        </div>
    </div>
<script>
const CFG={thr:10,dst:85,g:2.5,pin:"1234"}, qs=[{q:"She ___ to school.",c:"Goes",w:["Go"],h:"โกส์"},{q:"Apple is ___ the table.",c:"On",w:["In"],h:"ออน"},{q:"He is a ___.",c:"Teacher",w:["Nurse"],h:"ที-เชอร์"}];
let st={on:0,s:0,idx:0,it:[],m:{x:0,y:0,o:0},name:"",sp:0,rec:null}, ctx=cvs.getContext('2d');
const nav=id=>{ document.querySelectorAll('body>div:not(#loader)').forEach(d=>d.id!=='g-view'&&d.classList.add('hidden')); if(id)document.getElementById(id).classList.remove('hidden'); };
const chkPin=()=>{ pin.value===CFG.pin?(nav('scr-teacher'),loadS()):alert("Error"); };
const play=async()=>{ st.name=document.getElementById('s-name').value.trim(); if(!st.name)return; loader.style.opacity=1; st.rec=initR(); nav('g-view'); g_view.classList.remove('hidden'); await initAR(); loader.style.opacity=0; };
async function initAR(){
    const fm=new FaceMesh({locateFile:f=>`https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${f}`});
    fm.setOptions({maxNumFaces:1,refineLandmarks:true,minDetectionConfidence:0.5});
    fm.onResults(r=>{
        ctx.save(); ctx.clearRect(0,0,cvs.width,cvs.height); ctx.translate(cvs.width,0); ctx.scale(-1,1); ctx.drawImage(r.image,0,0,cvs.width,cvs.height); ctx.restore();
        if(r.multiFaceLandmarks?.[0]){
            const l=r.multiFaceLandmarks[0]; st.m.x=(1-l[13].x)*cvs.width; st.m.y=l[13].y*cvs.height;
            st.m.o=Math.abs(l[13].y-l[14].y)*cvs.height>CFG.thr;
            ctx.fillStyle=st.m.o?"#a855f7":"#fff"; ctx.beginPath(); ctx.arc(st.m.x,st.m.y,8,0,7); ctx.fill();
        }
        if(st.on&&!st.sp) upd();
    });
    const cam=new Camera(vid,{onFrame:async()=>await fm.send({image:vid}),width:480,height:360});
    cvs.width=innerWidth; cvs.height=innerHeight;
    return cam.start().then(()=>{st.on=1;hud.style.opacity=1;updUI();});
}
function upd(){
    if(Math.random()<0.04&&st.it.length<3) spawn();
    st.it.forEach((it,i)=>{
        it.y+=CFG.g; ctx.fillStyle="rgba(15,23,42,0.8)"; ctx.beginPath(); ctx.roundRect(it.x-60,it.y-25,120,50,15); ctx.fill();
        ctx.fillStyle="#fff"; ctx.font="bold 20px sans-serif"; ctx.textAlign="center"; ctx.fillText(it.t,it.x,it.y+7);
        if(Math.hypot(it.x-st.m.x,it.y-st.m.y)<CFG.dst&&st.m.o){
            st.it.splice(i,1); if(it.ok){st.s+=10;sync();startSp(it.t);} else{st.s=Math.max(0,st.s-5);sync();} updUI();
        }
        if(it.y>cvs.height+50)st.it.splice(i,1);
    });
}
const spawn=()=>{ const q=qs[st.idx]; if(!q)return; const p=[q.c,...q.w],t=p[Math.floor(Math.random()*p.length)]; st.it.push({x:60+Math.random()*(cvs.width-120),y:-50,t,ok:t===q.c}); };
const initR=()=>{
    const R=window.SpeechRecognition||window.webkitSpeechRecognition; if(!R)return null; const r=new R(); r.lang='en-US'; r.continuous=true;
    r.onresult=e=>{ const t=e.results[e.results.length-1][0].transcript.toLowerCase(); v_bar.style.width=Math.min(100,t.length*20)+'%'; if(t.includes(sp_t.innerText.toLowerCase()))spOk(); };
    r.onend=()=>st.sp&&r.start(); return r;
};
const startSp=t=>{ st.on=0;st.sp=1;st.it=[]; scr_sp.classList.remove('hidden'); sp_t.innerText=t; sp_h.innerText=qs[st.idx].h; if(st.rec)try{st.rec.start();}catch(e){} };
const spOk=()=>{ st.sp=0;st.on=1;st.idx++; scr_sp.classList.add('hidden'); if(st.rec)st.rec.stop(); if(st.idx>=qs.length){st.on=0;nav('scr-menu');alert("Score: "+st.s);location.reload();} updUI(); };
const sync=async()=>{ if(!window.u)return; await window.fb.setDoc(window.fb.doc(window.fb.db,'artifacts',window.fb.appId,'public','data','scores',window.u.uid),{name:st.name,score:st.s,time:Date.now()},{merge:true}); };
const loadS=()=>{ window.fb.onSnapshot(window.fb.collection(window.fb.db,'artifacts',window.fb.appId,'public','data','scores'),s=>{ let r=[]; s.forEach(d=>r.push(d.data())); s_list.innerHTML=r.sort((a,b)=>b.score-a.score).map(x=>`<div class="bg-slate-800 p-4 rounded-xl flex justify-between"><span>${x.name}</span><b>${x.score}</b></div>`).join(''); }); };
const updUI=()=>{ s_num.innerText=st.s; if(qs[st.idx])q_box.innerText=qs[st.idx].q; };
window.onresize=()=>{cvs.width=innerWidth;cvs.height=innerHeight;};
</script>
</body>
</html>
