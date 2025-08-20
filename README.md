<!DOCTYPE html>
<html lang="en">
<head>
  <script src="https://apis.google.com/js/api.js"></script>
<script>
  const CLIENT_ID = "513988292696-si4qkesgks11ohecii6o6frknsnjka71.apps.googleusercontent.com";
  const API_KEY = "YOUR_API_KEY_HERE"; // IMPORTANT: Replace with your Google API Key
  const SCOPES = "https://www.googleapis.com/auth/drive.file";
  const SYNC_FILE_NAME = "jee_planner_data.json";


  function handleClientLoad() {
    gapi.load("client:auth2", initClient);
  }


  function initClient() {
    gapi.client.init({
      apiKey: API_KEY,
      clientId: CLIENT_ID,
      discoveryDocs: ["https://www.googleapis.com/discovery/v1/apis/drive/v3/rest"],
      scope: SCOPES
    }).then(() => {
      const authInstance = gapi.auth2.getAuthInstance();
      authInstance.isSignedIn.listen(updateSigninStatus);
      updateSigninStatus(authInstance.isSignedIn.get());
      if (authInstance.isSignedIn.get()) {
        loadFromDrive();
      }
    });
  }


  function updateSigninStatus(isSignedIn) {
    if (isSignedIn) {
      console.log("Signed in!");
      $('#signInBtn').classList.add('hidden');
      $('#signOutBtn').classList.remove('hidden');
    } else {
      console.log("Not signed in.");
      $('#signInBtn').classList.remove('hidden');
      $('#signOutBtn').classList.add('hidden');
    }
  }


  function handleAuthClick() {
    gapi.auth2.getAuthInstance().signIn();
  }


  function handleSignoutClick() {
    gapi.auth2.getAuthInstance().signOut();
  }


  async function findFileInDrive() {
    try {
      const response = await gapi.client.drive.files.list({
        q: `'appDataFolder' in parents and name = '${SYNC_FILE_NAME}'`,
        spaces: 'appDataFolder',
        fields: 'files(id, name)',
      });
      return response.result.files.length > 0 ? response.result.files[0].id : null;
    } catch (error) {
      console.error("Error finding file:", error);
      return null;
    }
  }


  async function loadFromDrive() {
    const statusEl = $('#syncStatus');
    statusEl.textContent = 'Restoring from Drive...';
    try {
      const fileId = await findFileInDrive();
      if (fileId) {
        const response = await gapi.client.drive.files.get({
          fileId: fileId,
          alt: 'media'
        });
        const driveState = JSON.parse(response.body);
        Object.assign(state, driveState);
        saveStateToLocal(state); // Save to local storage as fallback/cache
        renderAll();
        statusEl.textContent = 'Synced with Drive ✅';
      } else {
        statusEl.textContent = 'No file found on Drive. Using local data.';
      }
    } catch (error) {
      console.error("Error loading from Drive:", error);
      statusEl.textContent = 'Sync failed. Check console.';
    }
  }


  async function saveToDrive() {
    const statusEl = $('#syncStatus');
    statusEl.textContent = 'Syncing...';
    try {
      const fileId = await findFileInDrive();
      const content = JSON.stringify(state, null, 2);
      const metadata = {
        'name': SYNC_FILE_NAME,
        'mimeType': 'application/json',
        'parents': ['appDataFolder']
      };
      
      const form = new FormData();
      form.append('metadata', new Blob([JSON.stringify(metadata)], { type: 'application/json' }));
      form.append('file', new Blob([content], { type: 'application/json' }));

      const method = fileId ? 'PATCH' : 'POST';
      const path = fileId ? `/upload/drive/v3/files/${fileId}` : '/upload/drive/v3/files';
      
      await gapi.client.request({
        path: path,
        method: method,
        params: { uploadType: 'multipart' },
        headers: { 'Content-Type': 'multipart/related;' },
        body: form
      });
      statusEl.textContent = 'Synced with Drive ✅';
    } catch (error) {
      console.error("Error saving to Drive:", error);
      statusEl.textContent = 'Sync failed. Check console.';
    }
  }
</script>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>JEE Planner — Modern</title>
  <meta name="description" content="Modern JEE planner with doubts, homework, planner, deadlines, analytics and interactive charts. Single-file and GitHub Pages ready.">
  <script src="https://cdn.tailwindcss.com"></script>
  <script>tailwind.config={theme:{extend:{colors:{brand:{50:'#f2fbff',100:'#e6f7ff',200:'#bfeaff',300:'#99dcff',400:'#4fc1ff',500:'#1aa8ff',600:'#0b8de6',700:'#0670b4',800:'#054f7f',900:'#053a5c'}}}}}</script>
  <script src="https://unpkg.com/lucide@latest"></script>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    /* small customizations to complement Tailwind */
    .glass { background: linear-gradient(135deg, rgba(255,255,255,0.6), rgba(255,255,255,0.45));
    backdrop-filter: blur(6px); }
    .card { transition: transform .12s ease, box-shadow .12s ease;
    }
    .card:hover { transform: translateY(-6px); box-shadow: 0 18px 40px rgba(2,6,23,.12); }
    .badge-overdue{ background:#ffe4e6; color:#9b111e;
    padding:.15rem .45rem; border-radius:999px; font-weight:600 }
    .badge-today{ background:#fff7ed; color:#92400e; padding:.15rem .45rem; border-radius:999px;
    font-weight:600 }
    .badge-upcoming{ background:#ecfeff; color:#075985; padding:.15rem .45rem; border-radius:999px;
    font-weight:600 }
    .hidden { display: none; }
  </style>
</head>
<body class="bg-slate-50 text-slate-900">
  <div class="max-w-6xl mx-auto p-4 md:p-8">
    <header class="flex items-center justify-between gap-4 mb-6">
      <div class="flex items-center gap-3">
        <div class="h-12 w-12 rounded-2xl bg-brand-600 grid place-items-center text-white text-2xl font-bold">J</div>
        <div>
          <h1 class="text-2xl md:text-3xl font-extrabold">JEE Planner — Study Companion</h1>
          <p class="text-sm text-slate-500">Doubts • Homework • Planner • Deadlines • Analytics</p>
        </div>
 
      </div>
      <div class="flex items-center gap-2">
        <button id="exportBtn" class="px-3 py-2 rounded-xl bg-slate-800 text-white">Export</button>
        <label class="px-3 py-2 rounded-xl bg-slate-100 cursor-pointer"><input id="importInput" type="file" accept="application/json" class="hidden">Import</label>
        <button id="themeBtn" class="px-3 py-2 rounded-xl bg-white border">Toggle Theme</button>
        <button id="signInBtn" class="px-3 py-2 rounded-xl bg-brand-600 text-white" onclick="handleAuthClick()">Sign in with Google</button>
        <button id="signOutBtn" class="px-3 py-2 rounded-xl bg-slate-100 hidden" onclick="handleSignoutClick()">Sign out</button>
      </div>
    </header>


    <nav class="grid grid-cols-4 gap-3 mb-6">
      <button data-tab="dashboard" class="tab card p-3 rounded-2xl bg-white">🏠 Dashboard</button>
      <button data-tab="doubts" class="tab 
 card p-3 rounded-2xl bg-white">❓ Doubts</button>
      <button data-tab="homework" class="tab card p-3 rounded-2xl bg-white">📝 Homework</button>
      <button data-tab="planner" class="tab card p-3 rounded-2xl bg-white">✅ Planner</button>
    </nav>


    <main>
      <section id="dashboard" class="tab-pane">
        <div class="grid md:grid-cols-3 gap-4">
          <div class="md:col-span-2 bg-white p-4 rounded-2xl card glass">
            <div class="flex items-start justify-between">
    
              <h2 class="text-lg font-semibold">Today & Reminders</h2>
              <div class="text-sm text-slate-500">Updated: <span id="lastUpdated">-</span></div>
            </div>
            <div class="text-sm text-slate-500 mt-2" id="syncStatus">Not synced. Sign in to enable.</div>
            <div class="mt-4 grid grid-cols-1 md:grid-cols-2 gap-4">
              <div class="p-3 rounded-lg bg-slate-50">
                <h3 class="font-medium">Due / Overdue</h3>
   
              <ul id="dueList" class="mt-2 text-sm space-y-2"></ul>
              </div>
              <div class="p-3 rounded-lg bg-slate-50">
                <h3 class="font-medium">Revisit Doubts</h3>
                <ul id="revisitList" class="mt-2 text-sm space-y-2"></ul>
              
 </div>
            </div>
            <div class="mt-4">
              <h3 class="font-medium">Notes</h3>
              <p class="text-sm text-slate-500 mt-2">Click any chart to filter the lists below.
 Use export/import to back up.</p>
            </div>
            <div class="mt-4 grid md:grid-cols-3 gap-3">
              <div class="p-3 rounded-lg bg-slate-50 text-center">
                <div class="text-xs text-slate-500">Pending Doubts</div>
                <div id="statDoubts" class="text-2xl font-bold">0</div>
            
   </div>
              <div class="p-3 rounded-lg bg-slate-50 text-center">
                <div class="text-xs text-slate-500">Open Homework</div>
                <div id="statHw" class="text-2xl font-bold">0</div>
              </div>
              <div class="p-3 rounded-lg bg-slate-50 text-center">
          
                <div class="text-xs text-slate-500">To-dos</div>
                <div id="statTodo" class="text-2xl font-bold">0</div>
              </div>
            </div>
          </div>


          <div class="bg-white p-4 rounded-2xl card glass">
            <h3 class="font-semibold">Analytics</h3>
         
            <div class="mt-3">
              <canvas id="pieChart" height="220"></canvas>
            </div>
            <div class="mt-3">
              <canvas id="barChart" height="160"></canvas>
            </div>
          </div>
        </div>


        <div 
 class="mt-6 grid md:grid-cols-2 gap-4">
          <div class="bg-white p-4 rounded-2xl card">
            <h3 class="font-semibold mb-3">Recent Items</h3>
            <div id="recentList" class="text-sm space-y-2 max-h-64 overflow-auto"></div>
          </div>
          <div class="bg-white p-4 rounded-2xl card">
            <h3 class="font-semibold mb-3">Deadlines (next 14 days)</h3>
          
            <div id="upcomingDeadlines" class="text-sm space-y-2 max-h-64 overflow-auto"></div>
          </div>
        </div>
      </section>


      <section id="doubts" class="tab-pane hidden">
        <div class="grid md:grid-cols-3 gap-4">
          <div class="md:col-span-2 bg-white p-4 rounded-2xl card">
            <h2 class="text-lg font-semibold">Add a Doubt</h2>
          
            <form id="doubtForm" class="mt-3 grid grid-cols-1 gap-2">
              <div class="grid md:grid-cols-3 gap-2">
                <select id="doubtSubject" class="p-2 rounded-lg border">
                  <option>Physics</option>
                  <option>Chemistry</option>
                  <option>Maths</option>
  
                </select>
                <input id="doubtTopic" placeholder="Topic / Chapter" class="p-2 rounded-lg border" />
                <input id="doubtRevisit" type="date" class="p-2 rounded-lg border" />
              </div>
              <textarea id="doubtDetail" rows="5" placeholder="Describe the doubt in detail (what you tried, where 
 stuck)" class="p-3 rounded-lg border"></textarea>
              <div class="flex gap-2 mt-2">
                <button class="px-3 py-2 rounded-xl bg-brand-600 text-white" type="submit">Add Doubt</button>
                <button id="clearDoubt" type="button" class="px-3 py-2 rounded-xl bg-slate-100">Clear</button>
              </div>
            </form>


         
            <div class="mt-4">
              <div class="flex items-center justify-between">
                <h3 class="font-semibold">All Doubts</h3>
                <div class="flex items-center gap-2">
                  <select id="filterDoubtSubj" class="p-2 rounded-lg border text-sm"><option value="">All</option><option>Physics</option><option>Chemistry</option><option>Maths</option></select>
                  
 <select id="filterDoubtStatus" class="p-2 rounded-lg border text-sm"><option value="">Any</option><option>Unresolved</option><option>Resolved</option></select>
                  <input id="searchDoubt" placeholder="Search..." class="p-2 rounded-lg border text-sm" />
                </div>
              </div>
              <div id="doubtList" class="mt-3 text-sm space-y-2 max-h-96 overflow-auto"></div>
            </div>
       
          </div>


          <div class="bg-white p-4 rounded-2xl card">
            <h3 class="font-semibold">Revisit Today</h3>
            <ul id="revisitToday" class="mt-3 text-sm space-y-2"></ul>
            <hr class="my-3">
            <h3 class="font-semibold">Tips</h3>
            <p class="text-sm text-slate-500 mt-2">Write exact steps you tried, mark the revisit date, and resolve when 
 understood. Use revisit reminders to practice spaced repetition.</p>
          </div>
        </div>
      </section>


      <section id="homework" class="tab-pane hidden">
        <div class="grid md:grid-cols-3 gap-4">
          <div class="md:col-span-2 bg-white p-4 rounded-2xl card">
            <h2 class="text-lg font-semibold">Add Homework</h2>
           
            <form id="hwForm" class="mt-3 grid grid-cols-1 gap-2">
              <input id="hwTitle" placeholder="Title (short)" class="p-2 rounded-lg border" />
              <textarea id="hwDesc" rows="5" placeholder="Detailed description (what to solve, pages, constraints)" class="p-3 rounded-lg border"></textarea>
              <div class="grid md:grid-cols-3 gap-2">
                <select id="hwSubject" class="p-2 rounded-lg border"><option>General</option><option>Physics</option><option>Chemistry</option><option>Maths</option></select>
          
                <input id="hwDate" type="date" class="p-2 rounded-lg border" />
                <input id="hwDeadline" type="date" class="p-2 rounded-lg border" />
              </div>
              <div class="flex gap-2 mt-2">
                <button class="px-3 py-2 rounded-xl bg-brand-600 text-white" type="submit">Add Homework</button>
            
            <button id="clearHw" type="button" class="px-3 py-2 rounded-xl bg-slate-100">Clear</button>
              </div>
            </form>


            <div class="mt-4">
              <h3 class="font-semibold">All Homework</h3>
              <div id="hwList" class="mt-3 text-sm space-y-2 max-h-96 overflow-auto"></div>
            </div>
   
          </div>


          <div class="bg-white p-4 rounded-2xl card">
            <h3 class="font-semibold">Homework Analytics</h3>
            <canvas id="hwPie" height="200"></canvas>
            <div class="mt-3 text-sm text-slate-500">Click a slice to filter homework by status.</div>
          </div>
        </div>
      </section>


     
 <section id="planner" class="tab-pane hidden">
        <div class="grid md:grid-cols-3 gap-4">
          <div class="md:col-span-2 bg-white p-4 rounded-2xl card">
            <h2 class="text-lg font-semibold">Add Task / To‑do</h2>
            <form id="todoForm" class="mt-3 grid grid-cols-1 gap-2">
              <input id="todoTitle" placeholder="Task title" class="p-2 rounded-lg border" />
       
        <textarea id="todoDesc" rows="3" placeholder="Details / expected time" class="p-3 rounded-lg border"></textarea>
              <div class="grid md:grid-cols-3 gap-2">
                <select id="todoPriority" class="p-2 rounded-lg border"><option>High</option><option>Medium</option><option>Low</option></select>
                <input id="todoDate" type="date" class="p-2 rounded-lg border" />
                <input id="todoDeadline" type="date" class="p-2 rounded-lg border" />
   
            </div>
              <div class="flex gap-2 mt-2">
                <button class="px-3 py-2 rounded-xl bg-brand-600 text-white" type="submit">Add Task</button>
                <button id="clearTodo" type="button" class="px-3 py-2 rounded-xl bg-slate-100">Clear</button>
              </div>
            </form>


 
            <div class="mt-4">
              <h3 class="font-semibold">All Tasks</h3>
              <div id="taskList" class="mt-3 text-sm space-y-2 max-h-96 overflow-auto"></div>
            </div>
          </div>


          <div class="bg-white p-4 rounded-2xl card">
            <h3 class="font-semibold">Focus Tracking</h3>
  
            <canvas id="focusLine" height="160"></canvas>
            <div class="mt-3 text-sm text-slate-500">Pomodoro minutes get added to Focus analytics when you complete a session.</div>
            <div class="mt-3">
              <label class="text-sm">Focus min<input id="pomoFocus" type="number" value="25" class="w-full mt-1 p-2 rounded-lg border" /></label>
              <label class="text-sm mt-2">Break min<input id="pomoBreak" type="number" value="5" class="w-full mt-1 p-2 
 rounded-lg border" /></label>
              <div class="mt-3 flex gap-2"><button id="pomoStart" class="px-3 py-2 rounded-xl bg-brand-600 text-white">Start</button><button id="pomoStop" class="px-3 py-2 rounded-xl bg-slate-100">Stop</button></div>
            </div>
          </div>
        </div>
      </section>


    </main>


    <footer class="mt-8 text-center text-sm text-slate-500">Made for JEE aspirants • Single file • Deploy on GitHub Pages</footer>
  </div>


  <script>
    // State (persisted)
 
    const saveStateToLocal = s => localStorage.setItem('jee.v2', JSON.stringify(s));
    const loadStateFromLocal = () => JSON.parse(localStorage.getItem('jee.v2') || '{}');
    const state = Object.assign({ doubts:[], homework:[], todos:[], analytics:{focusByDay:{}}, lastUpdated:null }, loadStateFromLocal());

    const saveState = () => {
      saveStateToLocal(state);
      if (gapi.auth2.getAuthInstance().isSignedIn.get()) {
        saveToDrive();
      }
    };


    // Helpers
    const $ = (sel,el=document)=>el.querySelector(sel);
    const $$ = (sel,el=document)=>Array.from(el.querySelectorAll(sel));
    const uid = ()=>Math.random().toString(36).slice(2,9);
    const todayStr = d=> (d||new Date()).toISOString().slice(0,10);
    // Tabs
    $$('.tab').forEach(b=>b.addEventListener('click', ()=>{ $$('.tab').forEach(t=>t.classList.remove('bg-brand-600','text-white')); b.classList.add('bg-brand-600','text-white'); const id=b.dataset.tab; $$('.tab-pane').forEach(p=>p.classList.add('hidden')); $('#'+id).classList.remove('hidden'); renderAll(); }));
    // activate default
    document.querySelector('[data-tab="dashboard"]').classList.add('bg-brand-600','text-white');


    // Theme toggle
    $('#themeBtn').addEventListener('click', ()=>{ document.documentElement.classList.toggle('dark'); });
    // Export/Import
    $('#exportBtn').addEventListener('click', ()=>{ const blob=new Blob([JSON.stringify(state,null,2)],{type:'application/json'}); const url=URL.createObjectURL(blob); const a=document.createElement('a'); a.href=url; a.download='jee_planner_backup.json'; document.body.appendChild(a); a.click(); a.remove(); URL.revokeObjectURL(url); });
    $('#importInput').addEventListener('change', async e=>{ const f=e.target.files[0]; if(!f) return; try{ const txt=await f.text(); const obj=JSON.parse(txt); Object.assign(state, obj); saveState(state); renderAll(); alert('Imported'); }catch(err){ alert('Invalid file'); } e.target.value=''; });
    // ---------------- Data operations ----------------
    function addDoubtEntry(o){ o.id = uid(); o.status = o.status||'Unresolved'; o.date = todayStr(); state.doubts.unshift(o);
    state.lastUpdated = new Date().toISOString(); saveState(); renderDoubts(); renderDashboard(); }
    function addHwEntry(o){ o.id = uid(); o.done = false;
    o.date = todayStr(); state.homework.unshift(o); state.lastUpdated = new Date().toISOString(); saveState(); renderHW(); renderDashboard(); }
    function addTodoEntry(o){ o.id = uid();
    o.done = false; o.date = todayStr(); state.todos.unshift(o); state.lastUpdated = new Date().toISOString(); saveState(); renderTodos(); renderDashboard();
    }


    // Form handlers
    $('#doubtForm').addEventListener('submit', e=>{ e.preventDefault(); const subj=$('#doubtSubject').value; const topic=$('#doubtTopic').value.trim(); const revisit=$('#doubtRevisit').value||(()=>{ const dt=new Date(); dt.setDate(dt.getDate()+2); return dt.toISOString().slice(0,10); })(); const detail=$('#doubtDetail').value.trim(); addDoubtEntry({subject:subj, topic, revisit, detail}); e.target.reset(); });
    $('#hwForm').addEventListener('submit', e=>{ e.preventDefault(); const title=$('#hwTitle').value.trim(); const desc=$('#hwDesc').value.trim(); const subject=$('#hwSubject').value; const date=$('#hwDate').value||todayStr(); const deadline=$('#hwDeadline').value||null; if(!title) return alert('Give a title'); addHwEntry({title, desc, subject, date, deadline}); e.target.reset(); });
    $('#todoForm').addEventListener('submit', e=>{ e.preventDefault(); const title=$('#todoTitle').value.trim(); const desc=$('#todoDesc').value.trim(); const priority=$('#todoPriority').value; const date=$('#todoDate').value||todayStr(); const deadline=$('#todoDeadline').value||null; if(!title) return alert('Enter title'); addTodoEntry({title, desc, priority, date, deadline}); e.target.reset(); });
    // Renderers
    function renderAll(){ renderDashboard(); renderDoubts(); renderHW(); renderTodos(); renderAnalytics();
    }


    // Dashboard
    function renderDashboard(){ $('#lastUpdated').textContent = state.lastUpdated? new Date(state.lastUpdated).toLocaleString() : '—';
    // due / overdue: any item with deadline <= today and not done/resolved
      const dueItems = [];
    state.doubts.forEach(d=>{ if(d.revisit && d.revisit <= todayStr() && d.status!=='Resolved') dueItems.push({type:'Doubt', title:d.topic||d.subject, when:d.revisit, id:d.id}); });
    state.homework.forEach(h=>{ if(h.deadline && h.deadline <= todayStr() && !h.done) dueItems.push({type:'HW', title:h.title, when:h.deadline, id:h.id}); });
    state.todos.forEach(t=>{ if(t.deadline && t.deadline <= todayStr() && !t.done) dueItems.push({type:'Todo', title:t.title, when:t.deadline, id:t.id}); });
      const dl = $('#dueList'); dl.innerHTML = dueItems.length?
    dueItems.map(i=>`<li class="flex justify-between items-center"><div><strong>${i.type}</strong> • ${escapeHtml(i.title)}</div><div class="text-xs">${i.when}</div></li>`).join('') : '<li class="text-slate-400">No due items</li>';
    // revisit list
      const revisit = state.doubts.filter(d=>d.revisit===todayStr() && d.status==='Unresolved'); $('#revisitList').innerHTML = revisit.length?
    revisit.map(d=>`<li><strong>${escapeHtml(d.subject)}</strong> • ${escapeHtml(d.topic || '')} — ${escapeHtml(d.detail.slice(0,80))} <button data-id="${d.id}" class="ml-2 px-2 py-1 rounded bg-emerald-100 text-emerald-700 mark-resolve">Mark Resolved</button></li>`).join('') : '<li class="text-slate-400">None</li>';
    $$('.mark-resolve').forEach(b=>b.addEventListener('click', e=>{ const id=e.target.dataset.id; const d=state.doubts.find(x=>x.id===id); if(d){ d.status='Resolved'; saveState(); renderAll(); } }));
    // stats
      $('#statDoubts').textContent = state.doubts.filter(d=>d.status!=='Resolved').length;
      $('#statHw').textContent = state.homework.filter(h=>!h.done).length;
      $('#statTodo').textContent = state.todos.filter(t=>!t.done).length;
    // recent
      const recent = [].concat(state.doubts.slice(0,5).map(d=>({t:'Doubt',txt:`${d.subject}: ${d.topic||''}`})), state.homework.slice(0,5).map(h=>({t:'HW',txt:h.title})), state.todos.slice(0,5).map(t=>({t:'Todo',txt:t.title}))).slice(0,8);
      $('#recentList').innerHTML = recent.length?
    recent.map(r=>`<div class="p-2 rounded bg-slate-50">[${r.t}] ${escapeHtml(r.txt)}</div>`).join('') : '<div class="text-slate-400">No recent items</div>';
    // upcoming deadlines 14 days
      const upcoming = [];
      const now = new Date();
    const limit = new Date(); limit.setDate(limit.getDate()+14);
      state.homework.forEach(h=>{ if(h.deadline){ const dd=new Date(h.deadline); if(dd>=now && dd<=limit) upcoming.push({type:'HW',title:h.title,when:h.deadline}); }});
    state.todos.forEach(t=>{ if(t.deadline){ const dd=new Date(t.deadline); if(dd>=now && dd<=limit) upcoming.push({type:'Todo',title:t.title,when:t.deadline}); }});
    state.doubts.forEach(d=>{ if(d.revisit){ const dd=new Date(d.revisit); if(dd>=now && dd<=limit) upcoming.push({type:'Doubt',title:d.topic||d.subject,when:d.revisit}); }});
    $('#upcomingDeadlines').innerHTML = upcoming.length? upcoming.sort((a,b)=>a.when>b.when?1:-1).map(u=>`<div class="p-2 rounded bg-slate-50"><strong>${u.type}</strong> • ${escapeHtml(u.title)} <div class="text-xs text-slate-500">${u.when}</div></div>`).join('') : '<div class="text-slate-400">No upcoming deadlines</div>';
    }


    // Doubts render
    function renderDoubts(){ const container = $('#doubtList'); const filterSubj = $('#filterDoubtSubj').value;
    const filterStatus = $('#filterDoubtStatus').value; const search = $('#searchDoubt').value.trim().toLowerCase(); let list = state.doubts.slice(); if(filterSubj) list = list.filter(d=>d.subject===filterSubj); if(filterStatus) list = list.filter(d=>d.status===filterStatus);
    if(search) list = list.filter(d=> (d.topic + ' ' + d.detail).toLowerCase().includes(search)); if(!list.length){ container.innerHTML = '<div class="text-slate-400">No doubts</div>'; return;
    } container.innerHTML = list.map(d=>{
      const overdue = d.revisit && d.revisit < todayStr();
      const badge = overdue? '<span class="badge-overdue">Overdue</span>' : (d.revisit===todayStr()? '<span class="badge-today">Revisit</span>' : '<span class="badge-upcoming">Scheduled</span>');
      return `<div class="p-3 rounded-lg bg-slate-50 flex justify-between items-start"><div><div class="font-semibold">${escapeHtml(d.subject)} • ${escapeHtml(d.topic||'')}</div><div class="text-sm text-slate-600 mt-1">${escapeHtml(d.detail)}</div><div class="text-xs text-slate-500 mt-2">Added: ${d.date} • Revisit: ${d.revisit}</div></div><div class="text-right space-y-2"><div>${badge}</div><div><button data-id="${d.id}" class="px-2 py-1 rounded bg-emerald-100 mark-resolve">Toggle</button></div><div><button data-id="${d.id}" class="px-2 py-1 rounded bg-red-100 del-doubt">Delete</button></div></div></div>`; }).join('');
    $$('.mark-resolve').forEach(b=>b.addEventListener('click', e=>{ const id=e.target.dataset.id; const d=state.doubts.find(x=>x.id===id); if(d){ d.status = d.status==='Resolved'?'Unresolved':'Resolved'; saveState(); renderDoubts(); renderDashboard(); } }));
    $$('.del-doubt').forEach(b=>b.addEventListener('click', e=>{ if(confirm('Delete doubt?')){ state.doubts = state.doubts.filter(x=>x.id!==e.target.dataset.id); saveState(); renderDoubts(); renderDashboard(); } }));
    }


    // HW render
    function renderHW(){ const c = $('#hwList'); const data = state.homework.slice();
    if(!data.length){ c.innerHTML = '<div class="text-slate-400">No homework</div>'; return; } c.innerHTML = data.map(h=>{ const overdue = h.deadline && h.deadline < todayStr(); const badge = overdue? '<span class="badge-overdue">Overdue</span>' : (h.deadline===todayStr()? '<span class="badge-today">Due Today</span>' : (h.deadline? '<span class="badge-upcoming">Due</span>': '')); return `<div class="p-3 rounded-lg bg-slate-50 flex justify-between"><div><div class="font-semibold">${escapeHtml(h.title)}</div><div class="text-sm text-slate-600 mt-1">${escapeHtml(h.desc)}</div><div class="text-xs text-slate-500 mt-2">Subject: ${escapeHtml(h.subject)} • Added: ${h.date}</div></div><div class="text-right space-y-2"><div>${badge}</div><div><button data-id="${h.id}" class="px-2 py-1 rounded bg-emerald-100 hw-done">${h.done? 'Undo':'Mark Done'}</button></div><div><button data-id="${h.id}" class="px-2 py-1 rounded bg-red-100 del-hw">Delete</button></div></div></div>`; }).join('');
    $$('.hw-done').forEach(b=>b.addEventListener('click', e=>{ const id=e.target.dataset.id; const h=state.homework.find(x=>x.id===id); if(h){ h.done=!h.done; saveState(); renderHW(); renderDashboard(); } }));
    $$('.del-hw').forEach(b=>b.addEventListener('click', e=>{ if(confirm('Delete?')){ state.homework = state.homework.filter(x=>x.id!==e.target.dataset.id); saveState(); renderHW(); renderDashboard(); } }));
    }


    // Todos
    function renderTodos(){ const c = $('#taskList'); const data = state.todos.slice();
    if(!data.length){ c.innerHTML = '<div class="text-slate-400">No tasks</div>'; return; } c.innerHTML = data.map(t=>{ const overdue = t.deadline && t.deadline < todayStr(); const badge = overdue? '<span class="badge-overdue">Overdue</span>' : (t.deadline===todayStr()? '<span class="badge-today">Today</span>' : (t.deadline? '<span class="badge-upcoming">Due</span>': '')); return `<div class="p-3 rounded-lg bg-slate-50 flex justify-between"><div><div class="font-semibold">${escapeHtml(t.title)}</div><div class="text-sm text-slate-600 mt-1">${escapeHtml(t.desc)}</div><div class="text-xs text-slate-500 mt-2">Priority: ${t.priority}</div></div><div class="text-right space-y-2"><div>${badge}</div><div><button data-id="${t.id}" class="px-2 py-1 rounded bg-emerald-100 task-done">${t.done? 'Undo':'Done'}</button></div><div><button data-id="${t.id}" class="px-2 py-1 rounded bg-red-100 del-task">Delete</button></div></div></div>`; }).join('');
    $$('.task-done').forEach(b=>b.addEventListener('click', e=>{ const id=e.target.dataset.id; const t=state.todos.find(x=>x.id===id); if(t){ t.done=!t.done; saveState(); renderTodos(); renderDashboard(); } }));
    $$('.del-task').forEach(b=>b.addEventListener('click', e=>{ if(confirm('Delete?')){ state.todos = state.todos.filter(x=>x.id!==e.target.dataset.id); saveState(); renderTodos(); renderDashboard(); } }));
    }


    // ---------------- Analytics: Charts ----------------
    let pieChart, barChart, hwPie, focusLine;
    function renderAnalytics(){ // pie: status breakdown (Doubts unresolved/resolved, HW done/pending, Todos done/pending)
      const unresolvedD = state.doubts.filter(d=>d.status!=='Resolved').length;
    const resolvedD = state.doubts.length - unresolvedD;
      const pendingH = state.homework.filter(h=>!h.done).length;
      const doneH = state.homework.length - pendingH;
      const pendingT = state.todos.filter(t=>!t.done).length;
    const doneT = state.todos.length - pendingT;
      const pieCtx = document.getElementById('pieChart').getContext('2d');
      if(pieChart) pieChart.destroy();
    pieChart = new Chart(pieCtx, { type:'pie', data:{ labels:['Doubts Unresolved','Doubts Resolved','HW Pending','HW Done','Tasks Pending','Tasks Done'], datasets:[{ data:[unresolvedD,resolvedD,pendingH,doneH,pendingT,doneT], backgroundColor:['#ef4444','#10b981','#f59e0b','#60a5fa','#f97316','#34d399'] }] }, options:{ responsive:true, onClick:(e,i)=>{ if(i.length){ const idx = i[0].index; handlePieClick(idx); } } } });
    // bar chart: deadlines count in next 7 days
      const days = [...Array(7)].map((_,i)=>{ const d = new Date(); d.setDate(d.getDate()+i); return d.toISOString().slice(0,10); });
    const counts = days.map(day=>{ let c=0; state.homework.forEach(h=>{ if(h.deadline===day && !h.done) c++; }); state.todos.forEach(t=>{ if(t.deadline===day && !t.done) c++; }); state.doubts.forEach(d=>{ if(d.revisit===day && d.status!=='Resolved') c++; }); return c; });
    const barCtx = document.getElementById('barChart').getContext('2d'); if(barChart) barChart.destroy(); barChart = new Chart(barCtx, { type:'bar', data:{ labels:days, datasets:[{ label:'Pending items', data:counts, backgroundColor:counts.map(c=> c>0? '#fb923c':'#bfdbfe') }] }, options:{ responsive:true, onClick:(e,i)=>{ if(i.length){ const idx=i[0].index; filterByDate(days[idx]); } } } });
    // hwPie in homework tab
      const hwCtx = document.getElementById('hwPie').getContext('2d'); if(hwPie) hwPie.destroy();
    hwPie = new Chart(hwCtx, { type:'doughnut', data:{ labels:['Pending','Done'], datasets:[{ data:[pendingH,doneH], backgroundColor:['#f97316','#10b981'] }] , }, options:{ responsive:true, onClick:(e,i)=>{ if(i.length){ const idx=i[0].index; if(idx===0) filterHW('pending'); else filterHW('done'); } } } });
    // focus line
      const last7 = [...Array(7)].map((_,i)=>{ const d=new Date(); d.setDate(d.getDate()-6+i); return d.toISOString().slice(0,10); });
    const mins = last7.map(d=> state.analytics.focusByDay && state.analytics.focusByDay[d] ? state.analytics.focusByDay[d] : 0 ); const flCtx = document.getElementById('focusLine').getContext('2d'); if(focusLine) focusLine.destroy();
    focusLine = new Chart(flCtx, { type:'line', data:{ labels:last7, datasets:[{ label:'Focus minutes', data:mins, borderColor:'#0ea5e9', backgroundColor:'rgba(14,165,233,0.12)', fill:true }] }, options:{ responsive:true } });
    }


    function handlePieClick(idx){ // map idx ranges to filters
      if(idx===0) { // doubts unresolved
        document.querySelector('[data-tab="doubts"]').click();
    $('#filterDoubtSubj').value=''; $('#filterDoubtStatus').value='Unresolved'; renderDoubts(); }
      if(idx===2) { document.querySelector('[data-tab="homework"]').click(); filterHW('pending');
    }
      if(idx===4) { document.querySelector('[data-tab="planner"]').click(); filterTasks('pending');
    }
    }


    function filterByDate(date){ // show items for that date
      document.querySelector('[data-tab="dashboard"]').click();
    // highlight upcoming
      // temporarily scroll to upcomingDeadlines and highlight matching elements (quick approach)
      window.location.hash='';
    // no-op
      alert('Filtering items for date: '+date+' — use search/filter in respective tabs for deeper actions');
    }


    // Filter helpers used by charts
    function filterHW(mode){ if(mode==='pending'){ // show only pending
        state.homework = state.homework.sort((a,b)=> (a.done?1:-1));
    renderHW(); } else if(mode==='done'){ state.homework = state.homework.sort((a,b)=> (a.done?-1:1)); renderHW(); } }
    function filterTasks(mode){ if(mode==='pending'){ state.todos = state.todos.sort((a,b)=> (a.done?1:-1));
    renderTodos(); } }


    // Mark done / delete handlers managed in render functions


    // Pomodoro (simple)
    let pomoTimer=null, pomoRemaining=0, pomoIsBreak=false;
    function startPomo(){ clearInterval(pomoTimer); pomoIsBreak=false; let sec=(parseInt($('#pomoFocus').value)||25)*60; pomoRemaining=sec; updatePomoDisplay(); pomoTimer=setInterval(()=>{ pomoRemaining--; updatePomoDisplay(); if(pomoRemaining<=0){ clearInterval(pomoTimer); // add focus minutes
          const mins = parseInt($('#pomoFocus').value)||25; const key=todayStr(); state.analytics.focusByDay = state.analytics.focusByDay||{}; state.analytics.focusByDay[key] = (state.analytics.focusByDay[key]||0)+mins; saveState(); renderAnalytics(); notify('Pomodoro completed','Focus minutes logged.'); } },1000);
    }
    function updatePomoDisplay(){ const m=Math.floor(pomoRemaining/60).toString().padStart(2,'0'); const s=(pomoRemaining%60).toString().padStart(2,'0'); $('#pomoStart').textContent = 'Running'; $('#pomoStop').textContent='Stop'; $('#pomoStart').disabled=false;
    }
    $('#pomoStart').addEventListener('click', startPomo); $('#pomoStop').addEventListener('click', ()=>{ clearInterval(pomoTimer); $('#pomoStart').textContent='Start'; });
    // Utility: escape
    function escapeHtml(s){ return String(s||'').replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
    }


    // ---------------- filters listeners ----------------
    $('#filterDoubtSubj').addEventListener('change', renderDoubts); $('#filterDoubtStatus').addEventListener('change', renderDoubts); $('#searchDoubt').addEventListener('input', renderDoubts);
    // ----------------- Render init functions call ----------------
    function renderAll(){ renderDashboard(); renderDoubts(); renderHW(); renderTodos(); renderAnalytics();
    }


    // Initial render functions reuse home section renderers
    function init(){ renderAll();
    }
    init();
  </script>
<script async defer onload="handleClientLoad()" src="https://apis.google.com/js/api.js"></script>
</body>
</html>
