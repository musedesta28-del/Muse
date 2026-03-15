# Muse
online exam system
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Personalized AI Exam - Unique per Student</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>
    <style>
        :root { --primary: #2563eb; --admin: #7c3aed; --bg: #f8fafc; }
        body { font-family: 'Inter', sans-serif; background: var(--bg); margin: 0; padding: 20px; }
        .container { max-width: 850px; margin: auto; background: white; padding: 30px; border-radius: 15px; box-shadow: 0 10px 25px rgba(0,0,0,0.1); }
        .tabs { display: flex; gap: 10px; margin-bottom: 25px; border-bottom: 2px solid #eee; }
        .tab { padding: 10px 20px; cursor: pointer; font-weight: bold; }
        .active-tab { border-bottom: 3px solid var(--primary); color: var(--primary); }
        .box { display: none; }
        .active-box { display: block; }
        input, select { width: 100%; padding: 12px; margin: 10px 0; border: 1px solid #ddd; border-radius: 8px; }
        .btn { width: 100%; padding: 15px; border: none; border-radius: 8px; font-weight: bold; cursor: pointer; color: white; }
        .btn-admin { background: var(--admin); }
        .btn-user { background: var(--primary); }
        .q-card { background: #f8fafc; padding: 20px; border-radius: 10px; margin-bottom: 20px; border-left: 5px solid var(--primary); }
        .opt { display: block; background: white; padding: 12px; margin: 8px 0; border-radius: 6px; cursor: pointer; border: 1px solid #e2e8f0; }
        #timer { font-size: 24px; color: #e11d48; text-align: center; font-weight: 800; }
        .loader { color: var(--primary); font-weight: bold; text-align: center; display: none; }
    </style>
</head>
<body>

<div class="container">
    <div class="tabs">
        <div id="tA" class="tab active-tab" onclick="switchTab('admin')">Admin (Setup Source)</div>
        <div id="tU" class="tab" onclick="switchTab('user')">Student (Personalized Exam)</div>
    </div>

    <div id="adminBox" class="box active-box">
        <div id="adminUi">
            <h3 style="color:var(--admin)">1. Register Students</h3>
            <div style="display:flex; gap:10px">
                <input type="text" id="sn" placeholder="Student Name">
                <input type="text" id="sid" placeholder="Manual ID (e.g. STU-101)">
            </div>
            <button class="btn btn-admin" onclick="reg()">Register Student</button>
            <div id="stList" style="margin-top:10px; font-size: 0.9rem;"></div>

            <h3 style="color:var(--admin); margin-top:30px">2. Source Material (Full Book)</h3>
            <input type="file" id="pdffile" accept=".pdf">
            <input type="number" id="mins" value="30" placeholder="Duration in Minutes">
            <button class="btn btn-admin" onclick="loadSource()">Prepare Source Material</button>
            <p id="sourceStatus" style="color:green; display:none;">✓ Source text loaded and ready!</p>
        </div>
    </div>

    <div id="userBox" class="box">
        <div id="uLogin">
            <h3>Login to your Unique Exam</h3>
            <input type="text" id="lID" placeholder="Enter Your Assigned ID">
            <button class="btn btn-user" onclick="startPersonalizedExam()">Generate My Exam & Start</button>
            <div id="loadMsg" class="loader">Generating a unique set of questions for you...</div>
        </div>
        <div id="quiz" style="display:none">
            <div id="timer">Time: <span id="clock">00:00</span></div>
            <div id="qs"></div>
            <button class="btn btn-user" onclick="end()">Submit My Exam</button>
        </div>
        <div id="res" class="q-card" style="display:none; text-align:center"></div>
    </div>
</div>

<script>
    const K = "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";
    pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.worker.min.js';

    let users = [], sourceText = "", dur = 30, currentExam = [], tInt;

    function switchTab(m) {
        document.getElementById('adminBox').classList.toggle('active-box', m==='admin');
        document.getElementById('userBox').classList.toggle('active-box', m==='user');
        document.getElementById('tA').classList.toggle('active-tab', m==='admin');
        document.getElementById('tU').classList.toggle('active-tab', m==='user');
    }

    function reg(){
        let n=document.getElementById('sn').value, i=document.getElementById('sid').value || "ID-"+Math.floor(1000+Math.random()*9000);
        if(!n) return alert("Name is required");
        users.push({n, i});
        document.getElementById('stList').innerHTML += `<div><b>${n}</b> (ID: <code>${i}</code>)</div>`;
        document.getElementById('sn').value=""; document.getElementById('sid').value="";
    }

    async function loadSource() {
        const file = document.getElementById('pdffile').files[0];
        if(!file) return alert("Select PDF");
        const reader = new FileReader();
        reader.onload = async function() {
            const pdf = await pdfjsLib.getDocument(new Uint8Array(this.result)).promise;
            sourceText = "";
            // Full Scan Logic: Sampling pages across the book
            let step = Math.max(1, Math.floor(pdf.numPages / 15)); 
            for (let i = 1; i <= pdf.numPages; i += step) {
                const page = await pdf.getPage(i);
                const content = await page.getTextContent();
                sourceText += content.items.map(s => s.str).join(" ") + " ";
            }
            document.getElementById('sourceStatus').style.display = "block";
            dur = document.getElementById('mins').value;
        };
        reader.readAsArrayBuffer(file);
    }

    async function startPersonalizedExam() {
        const id = document.getElementById('lID').value;
        const student = users.find(u => u.i === id);
        if(!student) return alert("ID not found! Admin must register you first.");
        if(!sourceText) return alert("Admin hasn't loaded the book yet!");

        document.getElementById('loadMsg').style.display = "block";
        
        try {
            // ተማሪው በገባ ቁጥር ለአይዲው የተለየ ጥያቄ እንዲመጣ ለ AIው IDውን እንልካለን
            const r = await fetch("https://models.inference.ai.azure.com/chat/completions", {
                method: "POST",
                headers: {"Content-Type":"application/json", "Authorization":`Bearer ${K}`},
                body: JSON.stringify({
                    messages: [
                        {role:"system", content: `You are an examiner. Generate 10 unique MCQs for Student ID: ${id}. 
                        Pick random topics from the text. Ensure options are never empty. 
                        Format: JSON array [{"q":"..","a":"..","b":"..","c":"..","d":"..","correct":"a"}]`},
                        {role:"user", content: sourceText.substring(0, 15000)}
                    ],
                    model: "gpt-4o"
                })
            });
            const d = await r.json();
            currentExam = JSON.parse(d.choices[0].message.content.replace(/```json|```/g, ""));
            
            document.getElementById('uLogin').style.display="none";
            document.getElementById('quiz').style.display="block";
            renderQuiz();
            startTimer(dur);
        } catch(e) { 
            alert("Connection error. Try again."); 
            document.getElementById('loadMsg').style.display = "none";
        }
    }

    function renderQuiz() {
        let h = "";
        currentExam.forEach((q,i) => {
            h += `<div class="q-card"><b>${i+1}. ${q.q}</b>
                <label class="opt"><input type="radio" name="ans${i}" value="a"> A) ${q.a}</label>
                <label class="opt"><input type="radio" name="ans${i}" value="b"> B) ${q.b}</label>
                <label class="opt"><input type="radio" name="ans${i}" value="c"> C) ${q.c}</label>
                <label class="opt"><input type="radio" name="ans${i}" value="d"> D) ${q.d}</label></div>`;
        });
        document.getElementById('qs').innerHTML = h;
    }

    function startTimer(m) {
        let s = m * 60;
        tInt = setInterval(() => {
            s--;
            document.getElementById('clock').innerText = Math.floor(s/60) + ":" + (s%60).toString().padStart(2,'0');
            if(s<=0) end();
        }, 1000);
    }

    function end() {
        clearInterval(tInt);
        let sc = 0;
        currentExam.forEach((q,i) => {
            const sel = document.querySelector(`input[name="ans${i}"]:checked`);
            if(sel && sel.value === q.correct) sc++;
        });
        document.getElementById('quiz').style.display="none";
        document.getElementById('res').style.display="block";
        document.getElementById('res').innerHTML = `<h2>Result</h2><h1>${sc} / ${currentExam.length}</h1><button class="btn btn-user" onclick="location.reload()">Finish</button>`;
    }
</script>
</body>
</html>
