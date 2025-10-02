<!doctype html>
<html lang="th">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>แชทออนไลน์ชื่อเอง</title>
<style>
body{font-family:sans-serif;background:#8da99b;margin:0;padding:4px;color:#111;}
.wrap{max-width:900px;margin:0 auto;}
header{display:flex;justify-content:space-between;align-items:center;margin-bottom:12px}
.logo{font-weight:700;font-size:20px}
button{cursor:pointer}
.panel{background:#fff;border:2px solid #fff;padding:12px;margin-bottom:14px;border-radius:11px;box-shadow:2px 2px 0 rgba(0,0,0,0.25)}
#messages{height:360px; overflow:auto; padding:8px; border:1px solid #ddd; background:#fff;}
.msg{padding:8px;margin:8px 0;border-radius:6px; border:1px solid #ccc; position:relative;}
.meta{font-size:12px;color:#610a10;margin-bottom:6px}
.controls{display:flex;gap:8px; margin-top:8px; align-items:center; justify-content:flex-end;}
textarea{width:100%; height:70px; padding:8px; box-sizing:border-box; font-size:14px}
.btn{padding:8px 12px;border:2px solid #fab3ad;background:#fab3ad;border-radius:300px}
.muted{color:#2c2827;font-size:13px}
.msg button{margin-left:6px;padding:4px 8px;border-radius:4px;border:1px solid #888;background:#f0f0f0;}
</style>
</head>
<body>
<div class="wrap">
<header>
  <div class="logo">แชทออนไลน์ — โมดูลโจเซฟ (ทดลอง)</div>
  <div>
    <input id="nameInput" placeholder="ใส่ชื่อก่อนแชท" />
  </div>
</header>

<div class="panel">
  <div id="messages" aria-live="polite"></div>
  <div style="margin-top:10px; display:flex; flex-direction:column;">
    <textarea id="inputMsg" placeholder="พิมพ์ข้อความที่นี่..."></textarea>
    <div class="controls">
      <button id="sendBtn" class="btn">ส่ง</button>
    </div>
    <div class="muted" id="statusTxt">สถานะ: เชื่อมต่อ</div>
  </div>
</div>
<div class="muted">ข้อความจะรีเซ็ตทุกเดือนอัตโนมัติ</div>
</div>

<!-- Firebase -->
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-database-compat.js"></script>

<script>
// ====== แก้ config ให้ตรงกับ Project Firebase ของมึง ======
const firebaseConfig = {
  apiKey: "AIzaSyDe23wzWDc5yh1GMKuo8eCpHE1m7_967X0",
  authDomain: "messagesel-3bfce.firebaseapp.com",
  databaseURL: "https://messagesel-3bfce-default-rtdb.firebaseio.com",
  projectId: "messagesel-3bfce",
  storageBucket: "messagesel-3bfce.firebasestorage.app",
  messagingSenderId: "1013439867079",
  appId: "1:1013439867079:web:c6146c9124703d6fc876e2"
};
firebase.initializeApp(firebaseConfig);
const db = firebase.database();
const messagesRef = db.ref('messages');
const metaRef = db.ref('meta');

// UI
const sendBtn = document.getElementById('sendBtn');
const inputMsg = document.getElementById('inputMsg');
const messagesEl = document.getElementById('messages');
const statusTxt = document.getElementById('statusTxt');
const nameInput = document.getElementById('nameInput');

// ส่งข้อความ
sendBtn.addEventListener('click', async () => {
  const text = inputMsg.value.trim();
  const name = nameInput.value.trim() || "ไม่ระบุชื่อ";
  if (!text) return;
  const msg = { text, sender: name, ts: Date.now() };
  await messagesRef.push(msg);
  inputMsg.value = '';
});

// รีเซ็ตอัตโนมัติรายเดือน
async function checkMonthlyReset() {
  const metaSnap = await metaRef.child('lastReset').once('value');
  const last = metaSnap.val();
  const now = new Date();
  const nowKey = now.getFullYear() + '-' + (now.getMonth()+1);
  let needReset = false;
  if (!last) needReset = true;
  else {
    const lastDate = new Date(last);
    const lastKey = lastDate.getFullYear() + '-' + (lastDate.getMonth()+1);
    if (lastKey !== nowKey) needReset = true;
  }
  if (needReset) {
    await messagesRef.remove();
    await metaRef.child('lastReset').set(Date.now());
  }
}
checkMonthlyReset();
setInterval(checkMonthlyReset, 30000);

// ฟังการเปลี่ยนแปลงข้อความแบบ realtime
messagesRef.on('value', snap => {
  const data = snap.val() || {};
  renderMessages(data);
});

// แสดงข้อความและปุ่มคัดลอก/ยกเลิก
function renderMessages(data) {
  const arr = Object.keys(data).map(k => ({ key:k, ...data[k] }));
  arr.sort((a,b) => (a.ts||0) - (b.ts||0));
  messagesEl.innerHTML = '';
  arr.forEach(item => {
    const div = document.createElement('div');
    div.className = 'msg';

    const meta = document.createElement('div');
    meta.className = 'meta';
    meta.textContent = `ส่งโดย: ${item.sender} — ${new Date(item.ts).toLocaleString()}`;
    div.appendChild(meta);

    const txt = document.createElement('div');
    txt.textContent = item.text;
    div.appendChild(txt);

    // ปุ่มยกเลิก
    const del = document.createElement('button');
    del.textContent = 'ยกเลิก';
    del.onclick = async () => {
      if (!confirm('ลบข้อความนี้ไหม?')) return;
      await messagesRef.child(item.key).remove();
    };
    div.appendChild(del);

    // ปุ่มคัดลอก
    const copyBtn = document.createElement('button');
    copyBtn.textContent = 'คัดลอก';
    copyBtn.onclick = () => {
      navigator.clipboard.writeText(item.text)
        .then(() => alert('คัดลอกเรียบร้อย!'))
        .catch(() => alert('คัดลอกไม่สำเร็จ'));
    };
    div.appendChild(copyBtn);

    messagesEl.appendChild(div);
  });
  messagesEl.scrollTop = messagesEl.scrollHeight;
}

// ตรวจ connection state
const connectedRef = db.ref(".info/connected");
connectedRef.on("value", snap => {
  statusTxt.textContent = snap.val() ? 'เชื่อมต่อ' : 'ตัดการเชื่อมต่อ';
});
</script>
</body>
</html>


