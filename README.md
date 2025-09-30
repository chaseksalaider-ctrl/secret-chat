# secret-chat[chat.html lol.txt](https://github.com/user-attachments/files/22616629/chat.html.lol.txt)
<!doctype html>
<html lang="th">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>แชทออนไลน์ (เวอร์ชันชื่อเอง)</title>
<style>
  body{font-family: "Noto Sans", Arial; background:#efefef; margin:0; padding:20px; color:#111;}
  .wrap{max-width:900px;margin:0 auto;}
  header{display:flex;justify-content:space-between;align-items:center;margin-bottom:12px}
  .logo{font-weight:700;font-size:20px}
  button {cursor:pointer}
  .panel{background:#fff;border:4px solid #222;padding:12px;margin-bottom:14px;border-radius:6px;box-shadow:2px 2px 0 rgba(0,0,0,0.25)}
  #messages{height:360px; overflow:auto; padding:8px; border:1px solid #ddd; background:#fff;}
  .msg{padding:8px;margin:8px 0;border-radius:6px; border:1px solid #ccc; position:relative;}
  .me{background:#e8ffe8;border-color:#7fcf7f}
  .meta{font-size:12px;color:#555;margin-bottom:6px}
  .right {position:absolute; right:8px; top:8px;}
  .controls{display:flex;gap:8px; margin-top:8px; align-items:center}
  textarea{width:100%; height:70px; padding:8px; box-sizing:border-box; font-size:14px}
  .btn {padding:8px 12px;border:2px solid #222;background:#f8f8f8;border-radius:4px}
  .muted{color:#888;font-size:13px}
</style>
</head>
<body>
<div class="wrap">
  <header>
    <div class="logo">Encrypted text chat — แบบตั้งชื่อเอง</div>
    <div>
      <input id="nameInput" placeholder="ใส่ชื่อก่อนแชท" />
    </div>
  </header>

  <div class="panel">
    <div id="messages" aria-live="polite"></div>

    <div style="display:flex;gap:12px;margin-top:10px">
      <div style="flex:1">
        <textarea id="inputMsg" placeholder="พิมพ์ข้อความที่นี่..."></textarea>
        <div class="controls">
          <button id="sendBtn" class="btn">ส่ง</button>
          <button id="clearBtn" class="btn">ลบข้อความทั้งหมด (รีเซ็ต)</button>
          <div class="muted" id="statusTxt">สถานะ: เชื่อมต่อ</div>
        </div>
      </div>
    </div>
  </div>

  <div class="muted">ข้อความจะรีเซ็ตทุกเดือนอัตโนมัติ</div>
</div>

<!-- Firebase -->
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-database-compat.js"></script>

<script>
// ====== ใส่คอนฟิก Firebase ของมึงที่นี่ ======
const firebaseConfig = {
  apiKey: "API_KEY",
  authDomain: "PROJECT.firebaseapp.com",
  databaseURL: "https://PROJECT-default-rtdb.firebaseio.com",
  projectId: "PROJECT",
  storageBucket: "PROJECT.appspot.com",
  messagingSenderId: "SENDER_ID",
  appId: "APP_ID"
};
firebase.initializeApp(firebaseConfig);
const db = firebase.database();

const messagesRef = db.ref('messages');
const metaRef = db.ref('meta');

// UI
const sendBtn = document.getElementById('sendBtn');
const inputMsg = document.getElementById('inputMsg');
const messagesEl = document.getElementById('messages');
const clearBtn = document.getElementById('clearBtn');
const statusTxt = document.getElementById('statusTxt');
const nameInput = document.getElementById('nameInput');

// ฟังก์ชันส่งข้อความ
sendBtn.addEventListener('click', async () => {
  const text = inputMsg.value.trim();
  const name = nameInput.value.trim() || "ไม่ระบุชื่อ";
  if (!text) return;

  const msg = {
    text: text,
    sender: name,
    ts: Date.now()
  };
  await messagesRef.push(msg);
  inputMsg.value = '';
});

// ลบข้อความทั้งหมด
clearBtn.addEventListener('click', async () => {
  if (!confirm('ลบข้อความทั้งหมดจริง?')) return;
  await messagesRef.remove();
  await metaRef.child('lastReset').set(Date.now());
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

// ฟังการเปลี่ยนแปลงข้อความ
messagesRef.on('value', snap => {
  const data = snap.val() || {};
  renderMessages(data);
});

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

    // ปุ่มลบข้อความเดี่ยว
    const del = document.createElement('button');
    del.className = 'right btn';
    del.textContent = 'ลบ';
    del.onclick = async () => {
      if (!confirm('ลบข้อความนี้ไหม?')) return;
      await messagesRef.child(item.key).remove();
    }
    div.appendChild(del);

    messagesEl.appendChild(div);
  });
  messagesEl.scrollTop = messagesEl.scrollHeight;
}

// connection state
const connectedRef = db.ref(".info/connected");
connectedRef.on("value", snap => {
  statusTxt.textContent = snap.val() ? 'เชื่อมต่อ' : 'ตัดการเชื่อมต่อ';
});
</script>
</body>
</html>
