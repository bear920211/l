<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>校園功德榜 - 實境失物尋寶</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;700&display=swap');
        
        body {
            font-family: 'Noto Sans TC', sans-serif;
            background: #f8fafc;
        }

        .map-container {
            position: relative;
            width: 100%;
            height: 600px;
            background-color: #e2e8f0;
            background-image: url('https://i.ibb.co/DPQm11wf/1.png');
            background-size: contain;
            background-position: center;
            background-repeat: no-repeat;
            border-radius: 2.5rem;
            cursor: crosshair;
            overflow: hidden;
            border: 8px solid white;
            box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
            user-select: none;
        }

        /* 核心形狀容器 */
        .shape-box {
            position: absolute;
            transform-origin: center center;
            display: flex;
            align-items: center;
            justify-content: center;
            pointer-events: none; 
        }

        .shape-box::after {
            content: '';
            position: absolute;
            inset: 0;
            box-sizing: border-box;
            transition: all 0.2s;
            background: transparent;
        }

        /* 管理模式：現有區域顯示紫色虛線 */
        .existing-area-admin::after {
            border: 2px dashed rgba(139, 92, 246, 0.7) !important;
            background: rgba(139, 92, 246, 0.15) !important;
            pointer-events: auto;
        }

        /* 編輯中區域：藍色實線 */
        .edit-box::after {
            border: 3px solid #4f46e5 !important;
            background: rgba(79, 70, 229, 0.2) !important;
            pointer-events: auto;
        }

        /* 暫存區域：黃色虛線 */
        .temp-box::after {
            border: 2px dashed #fbbf24 !important;
            background: rgba(251, 191, 36, 0.1) !important;
            pointer-events: auto;
        }

        /* 形狀類型樣式 */
        .s-rect::after { border-radius: 0; }
        .s-capsule::after { border-radius: 9999px; } 
        .s-ellipse::after { border-radius: 50%; }
        
        /* 三角形特別處理：使用 clip-path */
        .s-triangle::after {
            background: rgba(79, 70, 229, 0.1);
            clip-path: polygon(0% 100%, 100% 100%, 0% 0%); /* 左下直角三角形 */
            border: none !important; /* clip-path 下 border 不管用 */
        }
        /* 管理模式下的三角形邊框模擬 */
        .existing-area-admin.s-triangle::before,
        .edit-box.s-triangle::before,
        .temp-box.s-triangle::before {
            content: '';
            position: absolute;
            inset: 0;
            background: currentColor;
            clip-path: polygon(0% 100%, 100% 100%, 0% 0%, 0% 100%, 2% 98%, 2% 2%, 98% 98%, 2% 98%);
            opacity: 0.5;
            pointer-events: none;
        }

        /* 控制點 */
        .handle { position: absolute; pointer-events: auto !important; z-index: 100; }
        .rotate-handle {
            width: 18px; height: 18px; background: #4f46e5; border: 2px solid white;
            border-radius: 50%; top: -30px; left: 50%; transform: translateX(-50%); cursor: grab;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }
        .rotate-line { width: 2px; height: 12px; background: #4f46e5; top: -12px; left: 50%; transform: translateX(-50%); pointer-events: none; }
        .resize-handle {
            width: 14px; height: 14px; background: white; border: 2px solid #4f46e5;
            border-radius: 4px; bottom: -7px; right: -7px; cursor: nwse-resize;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }

        .tool-btn {
            @apply px-3 py-2 bg-slate-800 rounded-xl text-[12px] hover:bg-indigo-600 transition-all flex items-center gap-1 shadow-md text-white;
        }
        .tool-btn.active {
            @apply bg-indigo-600 ring-2 ring-white font-bold scale-105;
        }

        @keyframes float {
            0%, 100% { transform: translate(-50%, -100%) translateY(0px); }
            50% { transform: translate(-50%, -100%) translateY(-10px); }
        }
        .floating-pin { animation: float 2s ease-in-out infinite; }
    </style>
</head>
<body class="p-4 md:p-8 bg-slate-50">

    <div class="max-w-6xl mx-auto">
        <header class="flex flex-col md:flex-row justify-between items-start md:items-center mb-8 gap-6">
            <div>
                <div class="flex items-center gap-3">
                    <div class="w-12 h-12 bg-indigo-600 rounded-2xl flex items-center justify-center shadow-lg shadow-indigo-200">
                        <span class="text-2xl">💎</span>
                    </div>
                    <h1 class="text-3xl font-extrabold text-slate-800 tracking-tight">
                        NTCU 實境功德地圖
                        <button id="admin-trigger" class="ml-1 text-slate-200 hover:text-indigo-400 text-xs transition-colors">⚙️</button>
                    </h1>
                </div>
                <p class="text-slate-500 mt-2 font-medium">目前的區域資料與雲端即時同步中。</p>
            </div>
            
            <div class="flex items-center gap-4 bg-white p-2 px-6 rounded-3xl shadow-sm border border-slate-100">
                <div class="flex flex-col">
                    <span class="text-[10px] text-slate-400 uppercase font-black tracking-widest">目前貢獻者</span>
                    <span id="display-nickname" class="text-base font-bold text-slate-700">載入中...</span>
                </div>
                <div class="h-8 w-px bg-slate-100 mx-2"></div>
                <div class="flex flex-col items-end">
                    <span class="text-[10px] text-slate-400 uppercase font-black tracking-widest">功德值</span>
                    <span id="my-merit" class="text-xl font-black text-indigo-600">0</span>
                </div>
            </div>
        </header>

        <!-- 管理工具欄 -->
        <div id="admin-tools" class="hidden mb-6 p-5 bg-slate-900 text-white rounded-3xl shadow-2xl transition-all border border-slate-800">
            <div class="flex flex-wrap items-center justify-between gap-6">
                <div class="flex items-center gap-3">
                    <span class="bg-indigo-500 text-[10px] px-2 py-0.5 rounded font-black uppercase">管理模式</span>
                    <div class="flex gap-2">
                        <button id="btn-select" class="tool-btn active">👆 選取</button>
                        <button id="btn-rect" class="tool-btn">矩形</button>
                        <button id="btn-capsule" class="tool-btn">膠囊</button>
                        <button id="btn-ellipse" class="tool-btn">橢圓</button>
                        <button id="btn-triangle" class="tool-btn">三角形</button>
                    </div>
                </div>
                
                <div id="save-area-tool" class="hidden flex items-center gap-3 bg-indigo-500/20 p-2 px-4 rounded-2xl border border-indigo-500/30">
                    <button id="btn-open-save-modal" class="bg-indigo-500 hover:bg-indigo-400 px-4 py-1.5 rounded-xl text-sm font-bold transition-all">儲存新區域</button>
                    <button id="btn-cancel-temp" class="text-slate-400 text-xs hover:text-white">放棄</button>
                </div>

                <div class="flex items-center gap-3">
                    <button id="btn-delete-selected" class="text-slate-400 hover:text-red-400 text-xs font-bold px-3 py-1.5 rounded-xl transition-all">刪除</button>
                    <button id="btn-exit-admin" class="bg-red-500/10 text-red-400 text-xs px-3 py-1.5 rounded-xl hover:bg-red-500/20 transition-all">退出</button>
                </div>
            </div>
        </div>

        <div class="grid grid-cols-1 lg:grid-cols-12 gap-8">
            <div class="lg:col-span-8">
                <div id="school-map" class="map-container">
                    <div id="edit-layer" class="absolute inset-0 z-20"></div>
                    <div id="temp-layer" class="absolute inset-0 z-30"></div>
                    <div id="pins-layer" class="absolute inset-0 z-10 pointer-events-none"></div>
                    <canvas id="drawing-canvas" class="absolute inset-0 pointer-events-none z-40"></canvas>
                </div>
            </div>

            <div class="lg:col-span-4 flex flex-col gap-6">
                <div class="bg-indigo-600 rounded-[2.5rem] p-6 text-white shadow-xl">
                    <h3 class="text-lg font-bold mb-4 flex items-center gap-2">🏆 功德排行榜</h3>
                    <div id="leaderboard" class="space-y-3 opacity-90"></div>
                </div>

                <div class="bg-white rounded-[2.5rem] shadow-sm border border-slate-100 h-[400px] flex flex-col overflow-hidden">
                    <div class="p-6 border-b bg-slate-50/50 font-black text-slate-700">🔍 待認領物品 (<span id="items-count">0</span>)</div>
                    <div id="items-list" class="flex-1 overflow-y-auto p-4 space-y-3"></div>
                </div>
            </div>
        </div>
    </div>

    <!-- 儲存區域彈窗 -->
    <div id="save-building-modal" class="fixed inset-0 bg-slate-900/60 hidden flex items-center justify-center z-[110] p-4 backdrop-blur-sm">
        <div class="bg-white rounded-[2.5rem] shadow-2xl max-w-sm w-full p-8 border border-white">
            <h3 class="text-xl font-extrabold mb-2 text-slate-800">💾 命名區域</h3>
            <input type="text" id="input-new-building-name" placeholder="例如：美研大樓" class="w-full px-5 py-4 bg-slate-50 border border-slate-200 rounded-2xl outline-none focus:border-indigo-500 mb-4 font-bold">
            <button id="btn-final-save" class="w-full py-4 bg-indigo-600 text-white font-extrabold rounded-2xl shadow-lg hover:bg-indigo-700 transition-colors">確認建立</button>
            <button id="btn-close-save-modal" class="w-full text-slate-400 font-bold py-2 mt-2 text-sm hover:text-slate-600">取消</button>
        </div>
    </div>

    <!-- 報失彈窗 -->
    <div id="report-modal" class="fixed inset-0 bg-slate-900/60 hidden flex items-center justify-center z-[100] p-4 backdrop-blur-sm">
        <div class="bg-white rounded-[2.5rem] shadow-2xl max-w-md w-full p-8 border border-white">
            <h3 class="text-2xl font-extrabold mb-1 text-slate-800">📍 發現寶藏</h3>
            <div id="location-hint" class="inline-block bg-indigo-50 text-indigo-700 text-xs font-bold px-3 py-1.5 rounded-full mb-6">位置確認中...</div>
            <div class="space-y-4">
                <input type="text" id="input-finder" placeholder="您的英雄暱稱" class="w-full px-5 py-3 bg-slate-50 border border-slate-200 rounded-2xl font-bold">
                <input type="text" id="input-name" placeholder="遺失物品名稱" class="w-full px-5 py-3 bg-slate-50 border border-slate-200 rounded-2xl font-bold">
                <textarea id="input-detail-location" placeholder="具體描述（如：靠近飲水機旁）" class="w-full px-5 py-3 bg-slate-50 border border-slate-200 rounded-2xl font-bold h-24 resize-none"></textarea>
                <button id="btn-confirm-report" class="w-full py-4 bg-indigo-600 text-white font-extrabold rounded-2xl shadow-lg mt-2 hover:bg-indigo-700 transition-colors">送出功德報</button>
                <button id="btn-cancel-report" class="w-full text-slate-400 font-bold py-2 text-sm hover:text-slate-600">取消</button>
            </div>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, setDoc, collection, onSnapshot, increment } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'campus-merit-v4';

        let isAdminMode = false;
        let adminClickCount = 0;
        let activeTool = 'select'; 
        let isDrawing = false, isDragging = false, isResizing = false, isRotating = false;
        let selectedId = null, tempShape = null;
        let startPos = { x: 0, y: 0 }, offset = { x: 0, y: 0 };
        let buildings = [], lostItems = [];

        onAuthStateChanged(auth, async (user) => {
            if (user) {
                const nick = localStorage.getItem('last_nickname') || "訪客";
                document.getElementById('display-nickname').innerText = nick;
                syncUserMerit(nick);
                listenToItems(); 
                listenToBuildings(); 
                listenToLeaderboard();
            }
        });

        const initAuth = async () => {
            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                await signInWithCustomToken(auth, __initial_auth_token);
            } else { await signInAnonymously(auth); }
        };
        initAuth();

        const mapEl = document.getElementById('school-map');
        mapEl.addEventListener('mousedown', handleMouseDown);
        window.addEventListener('mousemove', handleMouseMove);
        window.addEventListener('mouseup', handleMouseUp);

        document.getElementById('admin-trigger').addEventListener('click', () => {
            adminClickCount++;
            if (adminClickCount >= 5) {
                toggleAdmin(true);
                adminClickCount = 0;
            }
            setTimeout(() => { adminClickCount = 0; }, 3000);
        });

        document.getElementById('btn-exit-admin').addEventListener('click', () => toggleAdmin(false));

        ['select', 'rect', 'capsule', 'ellipse', 'triangle'].forEach(t => {
            const btn = document.getElementById(`btn-${t}`);
            if(btn) btn.addEventListener('click', () => setTool(t));
        });

        document.getElementById('btn-delete-selected').addEventListener('click', deleteSelected);
        document.getElementById('btn-cancel-temp').addEventListener('click', () => { tempShape = null; renderTemp(); setTool('select'); });
        document.getElementById('btn-open-save-modal').addEventListener('click', () => document.getElementById('save-building-modal').classList.remove('hidden'));
        document.getElementById('btn-close-save-modal').addEventListener('click', () => document.getElementById('save-building-modal').classList.add('hidden'));
        document.getElementById('btn-final-save').addEventListener('click', finalizeBuildingSave);
        document.getElementById('btn-confirm-report').addEventListener('click', submitReport);
        document.getElementById('btn-cancel-report').addEventListener('click', () => document.getElementById('report-modal').classList.add('hidden'));

        function toggleAdmin(state) {
            isAdminMode = state;
            document.getElementById('admin-tools').classList.toggle('hidden', !state);
            selectedId = null;
            tempShape = null;
            renderBuildings();
            renderTemp();
        }

        function setTool(tool) {
            activeTool = tool;
            document.querySelectorAll('.tool-btn').forEach(btn => btn.classList.toggle('active', btn.id === `btn-${tool}`));
            if (tool !== 'select') { selectedId = null; renderBuildings(); }
        }

        function getRotatedPoint(px, py, cx, cy, angle) {
            const radians = (Math.PI / 180) * angle;
            const cos = Math.cos(radians), sin = Math.sin(radians);
            const nx = (cos * (px - cx)) + (sin * (py - cy)) + cx;
            const ny = (cos * (py - cy)) - (sin * (px - cx)) + cy;
            return { x: nx, y: ny };
        }

        function isInsidePrecise(px, py, box) {
            let lp = { x: px, y: py };
            if (box.rotation) {
                const cx = box.x + box.w/2, cy = box.y + box.h/2;
                lp = getRotatedPoint(px, py, cx, cy, -box.rotation);
            }
            if (!(lp.x >= box.x && lp.x <= box.x + box.w && lp.y >= box.y && lp.y <= box.y + box.h)) return false;
            
            if (box.type === 'ellipse' || box.type === 'capsule') {
                const cx = box.x + box.w/2, cy = box.y + box.h/2, rx = box.w/2, ry = box.h/2;
                return (Math.pow(lp.x-cx, 2)/Math.pow(rx, 2)) + (Math.pow(lp.y-cy, 2)/Math.pow(ry, 2)) <= 1;
            }
            
            if (box.type === 'triangle') {
                // 直角三角形點擊判定 (左下直角: (0,1), (1,1), (0,0) 相對於 box)
                const relX = (lp.x - box.x) / box.w;
                const relY = (lp.y - box.y) / box.h;
                return (relX >= 0 && relY >= 0 && relX <= 1 && relY <= 1 && (relX + (1-relY)) <= 1);
            }

            return true;
        }

        function handleMouseDown(e) {
            const rect = mapEl.getBoundingClientRect();
            const curX = ((e.clientX - rect.left) / rect.width) * 100;
            const curY = ((e.clientY - rect.top) / rect.height) * 100;
            startPos = { x: curX, y: curY };

            if (!isAdminMode) {
                let idx = findBuildingAt(curX, curY);
                document.getElementById('location-hint').innerText = `定位區域：${idx !== -1 ? buildings[idx].name : "校園開放區域"}`;
                document.getElementById('report-modal').classList.remove('hidden');
                return;
            }

            if (activeTool !== 'select') { isDrawing = true; return; }

            if (tempShape && isInsidePrecise(curX, curY, tempShape)) {
                offset = { x: curX - tempShape.x, y: curY - tempShape.y };
                isDragging = 'temp'; return;
            }

            let idx = findBuildingAt(curX, curY);
            if (idx !== -1) {
                selectedId = idx;
                offset = { x: curX - buildings[idx].x, y: curY - buildings[idx].y };
                isDragging = 'existing'; renderBuildings();
            } else {
                selectedId = null; renderBuildings();
            }
        }

        function handleMouseMove(e) {
            if (!isAdminMode) return;
            const rect = mapEl.getBoundingClientRect();
            const curX = ((e.clientX - rect.left) / rect.width) * 100;
            const curY = ((e.clientY - rect.top) / rect.height) * 100;

            if (isDrawing) {
                drawPreview(startPos.x, startPos.y, curX, curY);
            } else if (isDragging) {
                const t = isDragging === 'temp' ? tempShape : buildings[selectedId];
                if(t) { t.x = curX - offset.x; t.y = curY - offset.y; updateView(); }
            } else if (isResizing) {
                const t = isResizing === 'temp' ? tempShape : buildings[selectedId];
                if(t) { t.w = Math.max(1, curX - t.x); t.h = Math.max(1, curY - t.y); updateView(); }
            } else if (isRotating) {
                const t = isRotating === 'temp' ? tempShape : buildings[selectedId];
                if(t) {
                    const cx = t.x + t.w/2, cy = t.y + t.h/2;
                    t.rotation = Math.atan2(curY - cy, curX - cx) * (180/Math.PI) + 90;
                    updateView();
                }
            }
            function updateView() { isResizing === 'temp' || isDragging === 'temp' || isRotating === 'temp' ? renderTemp() : renderBuildings(); }
        }

        async function handleMouseUp(e) {
            if (isDrawing) {
                const rect = mapEl.getBoundingClientRect();
                const curX = ((e.clientX - rect.left) / rect.width) * 100;
                const curY = ((e.clientY - rect.top) / rect.height) * 100;
                const w = Math.abs(curX - startPos.x), h = Math.abs(curY - startPos.y);
                if (w > 0.5 && h > 0.5) {
                    tempShape = { type: activeTool, x: Math.min(startPos.x, curX), y: Math.min(startPos.y, curY), w, h, rotation: 0 };
                    setTool('select'); renderTemp();
                }
                isDrawing = false;
                clearCanvas();
            } else if (isDragging === 'existing' || isResizing === 'existing' || isRotating === 'existing') {
                await updateCloudConfig();
            }
            isDragging = isResizing = isRotating = false;
        }

        function findBuildingAt(px, py) {
            if (!buildings) return -1;
            for (let i = buildings.length - 1; i >= 0; i--) {
                if (isInsidePrecise(px, py, buildings[i])) return i;
            }
            return -1;
        }

        function createShapeDOM(b, i, mode) {
            const isSelected = (mode === 'temp') || (selectedId === i);
            const el = document.createElement('div');
            
            let stateClass = '';
            if (isAdminMode) {
                if (mode === 'temp') stateClass = 'temp-box';
                else if (isSelected) stateClass = 'edit-box';
                else stateClass = 'existing-area-admin';
            }
            
            el.className = `shape-box ${stateClass} s-${b.type}`;
            el.style.cssText = `left:${b.x}%; top:${b.y}%; width:${b.w}%; height:${b.h}%; transform:rotate(${b.rotation||0}deg);`;

            if (isAdminMode && isSelected) {
                const line = document.createElement('div'); line.className = 'handle rotate-line'; el.appendChild(line);
                const rot = document.createElement('div'); rot.className = 'handle rotate-handle';
                rot.addEventListener('mousedown', (e) => { e.stopPropagation(); isRotating = mode; });
                el.appendChild(rot);
                const res = document.createElement('div'); res.className = 'handle resize-handle';
                res.addEventListener('mousedown', (e) => { e.stopPropagation(); isResizing = mode; });
                el.appendChild(res);
            }
            return el;
        }

        function renderBuildings() {
            const l = document.getElementById('edit-layer');
            if (!l) return;
            l.innerHTML = '';
            buildings.forEach((b, i) => l.appendChild(createShapeDOM(b, i, 'existing')));
        }
        function renderTemp() {
            const l = document.getElementById('temp-layer');
            l.innerHTML = '';
            document.getElementById('save-area-tool').classList.toggle('hidden', !tempShape);
            if (tempShape) l.appendChild(createShapeDOM(tempShape, null, 'temp'));
        }

        function drawPreview(sx, sy, ex, ey) {
            const canvas = document.getElementById('drawing-canvas');
            canvas.width = mapEl.clientWidth; canvas.height = mapEl.clientHeight;
            const ctx = canvas.getContext('2d');
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.strokeStyle = "#4f46e5"; ctx.setLineDash([4, 4]); ctx.lineWidth = 2;
            const vx = (sx/100)*canvas.width, vy = (sy/100)*canvas.height;
            const vw = ((ex-sx)/100)*canvas.width, vh = ((ey-sy)/100)*canvas.height;
            
            if (activeTool === 'triangle') {
                ctx.beginPath();
                ctx.moveTo(vx, vy);
                ctx.lineTo(vx + vw, vy + vh);
                ctx.lineTo(vx, vy + vh);
                ctx.closePath();
                ctx.stroke();
            } else {
                ctx.strokeRect(vx, vy, vw, vh);
            }
        }

        async function finalizeBuildingSave() {
            const name = document.getElementById('input-new-building-name').value.trim();
            if (!name) return;
            const newList = [...buildings, { ...tempShape, name }];
            tempShape = null;
            document.getElementById('input-new-building-name').value = '';
            document.getElementById('save-building-modal').classList.add('hidden');
            
            buildings = newList;
            renderBuildings();
            renderTemp();
            
            await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'buildings'), { list: buildings }, { merge: true });
        }

        async function updateCloudConfig() {
            if (!auth.currentUser) return;
            await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'buildings'), { list: buildings }, { merge: true });
        }
        
        function listenToBuildings() {
            onSnapshot(doc(db, 'artifacts', appId, 'public', 'data', 'config', 'buildings'), (snap) => {
                if (snap.exists()) { 
                    const data = snap.data();
                    buildings = Array.isArray(data.list) ? data.list : [];
                    renderBuildings(); 
                } else {
                    buildings = [];
                    renderBuildings();
                }
            }, (err) => console.error("Sync Error:", err));
        }

        async function deleteSelected() {
            if (selectedId !== null) { 
                buildings.splice(selectedId, 1); 
                selectedId = null; 
                await updateCloudConfig(); 
                renderBuildings(); 
            }
        }

        function listenToItems() {
            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'lost_items'), (s) => {
                lostItems = s.docs.map(d => ({ id: d.id, ...d.data() }));
                document.getElementById('items-count').innerText = lostItems.length;
                renderPins(); renderList();
            });
        }
        function renderPins() {
            const l = document.getElementById('pins-layer'); l.innerHTML = '';
            lostItems.forEach(item => {
                const p = document.createElement('div'); p.className = 'absolute transform -translate-x-1/2 -translate-y-full floating-pin';
                p.style.left = `${item.x}%`; p.style.top = `${item.y}%`;
                p.innerHTML = `<div class="bg-indigo-600 text-white text-[8px] px-1 py-0.5 rounded shadow mb-1 whitespace-nowrap font-bold">${item.name}</div><div class="w-3 h-3 bg-white rounded-full border-2 border-indigo-600 mx-auto shadow-sm"></div>`;
                l.appendChild(p);
            });
        }
        function renderList() {
            const l = document.getElementById('items-list');
            l.innerHTML = lostItems.map(item => `
                <div class="p-4 bg-white rounded-2xl border border-slate-100 flex justify-between items-center shadow-sm">
                    <div>
                        <div class="font-bold text-slate-800 text-sm">${item.name}</div>
                        <div class="text-[10px] text-indigo-500 font-bold uppercase">📍 ${item.location}</div>
                    </div>
                    <div class="text-right">
                        <div class="text-[9px] text-slate-300">${item.finder}</div>
                        <div class="text-xs font-black text-amber-500">+50</div>
                    </div>
                </div>
            `).join('') || '<div class="text-center py-12 text-slate-300 text-sm font-medium">目前沒人掉東西</div>';
        }

        function listenToLeaderboard() {
            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'users'), (s) => {
                const list = s.docs.map(d => ({ name: d.id, merit: d.data().merit || 0 })).sort((a,b)=>b.merit-a.merit);
                document.getElementById('leaderboard').innerHTML = list.slice(0, 5).map((u, i) => `
                    <div class="flex justify-between items-center text-xs p-2 ${i===0?'bg-white/20 rounded-xl ring-1 ring-white/30 font-bold':''}"><span>${i+1}. ${u.name}</span><span>${u.merit} pts</span></div>
                `).join('');
            });
        }

        async function submitReport() {
            const nick = document.getElementById('input-finder').value.trim() || "匿名善士";
            const name = document.getElementById('input-name').value.trim();
            const detail = document.getElementById('input-detail-location').value.trim();
            if(!name) return;
            const idx = findBuildingAt(startPos.x, startPos.y);
            const locName = idx !== -1 ? buildings[idx].name : "校園開放區域";
            await setDoc(doc(collection(db, 'artifacts', appId, 'public', 'data', 'lost_items')), { name, finder: nick, location: locName, detail, x: startPos.x, y: startPos.y, createdAt: Date.now() });
            await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', nick), { merit: increment(50) }, { merge: true });
            localStorage.setItem('last_nickname', nick); syncUserMerit(nick); 
            document.getElementById('report-modal').classList.add('hidden');
        }
        
        async function syncUserMerit(n) {
            const s = await getDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', n));
            if(s.exists()) document.getElementById('my-merit').innerText = s.data().merit;
        }
        function clearCanvas() { const c = document.getElementById('drawing-canvas'); if(c) c.getContext('2d').clearRect(0,0,c.width,c.height); }
    </script>
</body>
</html>
