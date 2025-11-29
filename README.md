<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8" />
<title>통합 체스 플랫폼 - 완전 수정판</title>

<!-- Chessboard.js / Chess.js -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/chessboard-js/1.0.0/chessboard-1.0.0.min.css" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/chess.js/0.12.0/chess.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/chessboard-js/1.0.0/chessboard-1.0.0.min.js"></script>

<!-- Stockfish CDN (자동 선택) -->
<script>
window.STOCKFISH = "https://cdn.jsdelivr.net/gh/niklasf/stockfish.js/stockfish.js";
</script>

<style>
body { display:flex; gap:40px; margin:40px; font-family:Arial; }
#eval-container { width:40px; height:480px; background:#333; position:relative; }
#eval-bar { position:absolute; width:100%; height:50%; background:white; bottom:0; transition:height 0.2s; }

#controls button { margin-bottom:10px; padding:10px; width:200px; }

#profile-modal {
    position:fixed; top:0; left:0; width:100%; height:100%;
    background:rgba(0,0,0,0.5);
    display:none; justify-content:center; align-items:center;
}
#profile-box {
    width:400px; background:white; padding:20px;
    border-radius:10px;
}
#profile-box img {
    width:100px; height:100px; border-radius:50%; margin-bottom:10px;
}
.recent-item.win { color:green; }
.recent-item.lose { color:red; }
</style>
</head>
<body>

<!-- 좌측 UI ------------------------------------------------------------- -->
<div>

<h3>유저 시스템</h3>
<div id="user-info"></div>
<div id="elo-info">ELO: -</div>

<button id="profile-btn" style="display:none;">프로필 보기</button>

<!-- 로그인 -->
<div id="login-box">
<h3>로그인</h3>
<input id="login-user" placeholder="아이디"><br>
<input id="login-pass" type="password" placeholder="비밀번호"><br><br>
<button id="login-btn">로그인</button>
</div>

<!-- 회원가입 -->
<div id="signup-box">
<h3>회원가입</h3>
<input id="signup-user" placeholder="아이디"><br>
<input id="signup-pass" type="password" placeholder="비밀번호"><br><br>
<button id="signup-btn">가입</button>
</div>

<button id="logout-btn">로그아웃</button>

<hr>

<h3>AI 난이도</h3>
<select id="ai-depth">
<option value="5">쉬움 (5)</option>
<option value="10" selected>보통 (10)</option>
<option value="15">어려움 (15)</option>
<option value="20">최강 (20)</option>
</select>

<hr>

<h3>게임 모드</h3>
<div id="controls">
<button id="human-ai-btn">사람 vs AI</button>
<button id="ai-ai-btn">AI vs AI</button>
<button id="online-btn" disabled>온라인 멀티</button>
<button id="reset-btn">게임 초기화</button>
</div>

<div id="game-info">
현재 턴: <span id="turn-info">-</span><br>
상대: <span id="opponent-info">-</span>
</div>

<div id="status"></div>
</div>

<!-- 체스판 ------------------------------------------------------------- -->
<div id="board" style="width:480px;"></div>

<!-- 평가바 ------------------------------------------------------------- -->
<div id="eval-container">
<div id="eval-bar"></div>
</div>


<!-- 프로필 모달 ------------------------------------------------------------- -->
<div id="profile-modal">
    <div id="profile-box">
        <h2>프로필</h2>

        <img id="profile-avatar" src="" alt="avatar">
        <br>

        <strong id="profile-username"></strong><br>
        <strong id="profile-elo"></strong><br>

        <hr>
        <div>
            <strong>전적</strong><br>
            승: <span id="profile-wins"></span><br>
            패: <span id="profile-losses"></span><br>
            무: <span id="profile-draws"></span><br>
            승률: <span id="profile-winrate"></span>%<br>
        </div>

        <hr>
        <strong>최근 5경기</strong>
        <div id="profile-recent"></div>

        <hr>
        <strong>아바타 업로드</strong><br>
        <input type="file" id="avatar-file">
        <button id="avatar-upload-btn">업로드</button>

        <hr>
        <button id="profile-close-btn">닫기</button>
    </div>
</div>


<script>
/*************************************************************
 * 0) DOM 요소 수집
 *************************************************************/
const el = {
    loginUser: document.getElementById("login-user"),
    loginPass: document.getElementById("login-pass"),
    signupUser: document.getElementById("signup-user"),
    signupPass: document.getElementById("signup-pass"),

    userInfo: document.getElementById("user-info"),
    eloInfo: document.getElementById("elo-info"),

    profileBtn: document.getElementById("profile-btn"),
    onlineBtn: document.getElementById("online-btn"),

    aiDepth: document.getElementById("ai-depth"),

    statusBox: document.getElementById("status"),
    turnInfo: document.getElementById("turn-info"),
    opponentInfo: document.getElementById("opponent-info"),

    profileModal: document.getElementById("profile-modal"),
    profileAvatar: document.getElementById("profile-avatar"),
    profileUsername: document.getElementById("profile-username"),
    profileElo: document.getElementById("profile-elo"),
    profileWins: document.getElementById("profile-wins"),
    profileLosses: document.getElementById("profile-losses"),
    profileDraws: document.getElementById("profile-draws"),
    profileWinrate: document.getElementById("profile-winrate"),
    profileRecent: document.getElementById("profile-recent"),
    avatarFile: document.getElementById("avatar-file")
};

/*************************************************************
 * 1) STOCKFISH
 *************************************************************/
const engine = new Worker(window.STOCKFISH);
let bestMoveCallback = null;

engine.onmessage = (e) => {
    if (typeof e.data !== "string") return;

    if (e.data.startsWith("info")) {
        const m = e.data.match(/score (cp|mate) (-?\d+)/);
        if (!m) return;
        let val = parseInt(m[2]);
        if (m[1] === "mate") val = val > 0 ? 2000 : -2000;

        const clamp = Math.max(-1000, Math.min(1000, val));
        const percent = ((clamp + 1000) / 2000) * 100;
        document.getElementById("eval-bar").style.height = percent + "%";
    }

    if (e.data.startsWith("bestmove")) {
        const move = e.data.split(" ")[1];
        if (bestMoveCallback) bestMoveCallback(move);
        bestMoveCallback = null;
    }
};

function getBestMove(fen, callback) {
    const depth = parseInt(el.aiDepth.value);
    bestMoveCallback = callback;
    engine.postMessage("position fen " + fen);
    engine.postMessage("go depth " + depth);
}

/*************************************************************
 * 2) UTIL
 *************************************************************/
function safe(t){
    return String(t).replace(/[<>&"'`]/g,
        c => ({'<':'&lt;','>':'&gt;','&':'&amp;','"':'&quot;',"'":"&#39;'}[c]));
}

/*************************************************************
 * 3) 프로필 시스템
 *************************************************************/
const Profile = (() => {

    async function load() {
        const data = await fetch("/me").then(r=>r.json());
        if (!data.loggedIn) return;

        el.profileAvatar.src = data.avatarUrl || "/default-avatar.png";
        el.profileUsername.innerText = data.username;
        el.profileElo.innerText = "ELO : " + data.elo;

        el.profileWins.innerText = data.stats.wins;
        el.profileLosses.innerText = data.stats.losses;
        el.profileDraws.innerText = data.stats.draws;

        const total = data.stats.wins + data.stats.losses + data.stats.draws;
        el.profileWinrate.innerText = total ? ((data.stats.wins / total) * 100).toFixed(1) : 0;

        el.profileRecent.innerHTML = "";
        data.recent.slice(0,5).forEach(r => {
            const div = document.createElement("div");
            div.className = "recent-item " + (r === "win" ? "win" : "lose");
            div.innerText = r === "win" ? "승" : "패";
            el.profileRecent.appendChild(div);
        });
    }

    function open() {
        load();
        el.profileModal.style.display = "flex";
    }

    function close() {
        el.profileModal.style.display = "none";
    }

    async function uploadAvatar() {
        const file = el.avatarFile.files[0];
        if (!file) return alert("파일 선택하세요");

        const fd = new FormData();
        fd.append("avatar", file);

        const res = await fetch("/profile/upload-avatar", {
            method:"POST",
            body: fd
        }).then(r=>r.json());

        if (res.success) {
            alert("업로드 성공");
            el.profileAvatar.src = res.avatarUrl;
        } else {
            alert("실패: " + res.msg);
        }
    }

    return { open, close, uploadAvatar };
})();

/*************************************************************
 * 4) 게임 모듈
 *************************************************************/
const Game = (() => {
    let game = new Chess();
    let board = null;

    let mode = "local";
    let ws = null;
    let aiLoop = null;

    let myColor = null;
    let opponent = "-";
    let myElo = 1200;

    function updateUI() {
        let msg = "";
        if (game.in_checkmate()) msg = "체크메이트!";
        else if (game.in_stalemate()) msg = "스테일메이트!";
        else if (game.in_check()) msg = (game.turn()==="w"?"백":"흑") + " 체크!";
        else msg = (game.turn()==="w"?"백":"흑") + " 차례";

        el.statusBox.innerText = msg;
        el.turnInfo.innerText = game.turn()==="w"?"백":"흑";
        el.opponentInfo.innerText = opponent;
        el.eloInfo.innerText = "ELO: " + myElo;
    }

    function onDragStart(src, piece) {
        if (game.game_over()) return false;

        if (mode === "online") {
            if (myColor==="white" && piece.startsWith("b")) return false;
            if (myColor==="black" && piece.startsWith("w")) return false;
            if ((myColor==="white" && game.turn()!=="w") ||
                (myColor==="black" && game.turn()!=="b"))
                return false;
        }

        if (mode === "human-ai" && game.turn()==="b") return false;
    }

    function onDrop(src, dst) {
        const move = game.move({from:src,to:dst,promotion:"q"});
        if (!move) return "snapback";

        board.position(game.fen());
        updateUI();

        if (mode === "human-ai") {
            getBestMove(game.fen(), best => {
                game.move(best);
                board.position(game.fen());
                updateUI();
            });
        }

        if (mode === "online" && ws) {
            ws.send(JSON.stringify({type:"move",move}));
        }
    }

    function initBoard() {
        if (aiLoop) { clearTimeout(aiLoop); aiLoop = null; }

        game.reset();
        board = Chessboard("board", {
            position:"start",
            draggable:true,
            onDragStart,onDrop,
            onSnapEnd:()=>board.position(game.fen())
        });

        updateUI();
    }

    async function updateElo(result) {
        const res = await fetch("/update-elo", {
            method:"POST",
            headers:{"Content-Type":"application/json"},
            body: JSON.stringify({ result })
        }).then(r=>r.json());

        if (res.success) {
            myElo = res.newElo;
            el.eloInfo.innerText = "ELO: " + myElo;
        }
    }

    function startHumanAI() {
        mode="human-ai";
        opponent="AI";
        initBoard();
    }

    function startAiVsAi() {
        mode="ai-ai";
        opponent="AI";
        initBoard();

        function loop() {
            if (game.game_over()) return;
            getBestMove(game.fen(), best => {
                game.move(best);
                board.position(game.fen());
                updateUI();
                aiLoop = setTimeout(loop,200);
            });
        }
        loop();
    }

    function startOnline() {
        mode="online";
        opponent="대기중...";
        initBoard();

        ws = new WebSocket("wss://" + window.location.host + "/ws");

        ws.onmessage = async e => {
            const data = JSON.parse(e.data);

            if (data.type==="waiting") {
                el.statusBox.innerText = "상대 찾는 중...";
            }

            if (data.type==="start") {
                myColor = data.color;          // "white" or "black"
                opponent = safe(data.opponent);
                board.orientation(myColor);
                updateUI();
            }

            if (data.type==="move") {
                game.move(data.move);
                board.position(game.fen());
                updateUI();

                if (game.in_checkmate() || game.in_stalemate()) {
                    const r = game.in_checkmate()
                        ? (game.turn() === (myColor==="white"?"w":"b") ? "lose" : "win")
                        : "draw";
                    await updateElo(r);
                }
            }

            if (data.type==="end") {
                alert("상대가 나갔습니다");
                opponent="-";
                updateUI();
            }
        };
    }

    function reset() {
        if (ws) ws.close();
        ws = null;
        myColor = null;
        opponent = "-";
        initBoard();
    }

    return { startHumanAI, startAiVsAi, startOnline, reset };
})();

/*************************************************************
 * 5) 로그인 & 회원가입
 *************************************************************/
document.getElementById("signup-btn").onclick = async () => {
    const username = el.signupUser.value;
    const password = el.signupPass.value;

    const res = await fetch("/signup",{
        method:"POST",
        headers:{"Content-Type":"application/json"},
        body: JSON.stringify({ username, password })
    }).then(r=>r.json());

    alert(res.success ? "가입 성공!" : res.msg);
};

document.getElementById("login-btn").onclick = async () => {
    const username = el.loginUser.value;
    const password = el.loginPass.value;

    const res = await fetch("/login",{
        method:"POST",
        headers:{"Content-Type":"application/json"},
        body: JSON.stringify({ username, password })
    }).then(r=>r.json());

    if (res.success) {
        el.userInfo.innerText = "로그인됨: " + res.username;
        el.eloInfo.innerText = "ELO: " + res.elo;
        el.profileBtn.style.display = "block";
        el.onlineBtn.disabled = false;

        document.getElementById("login-box").style.display = "none";
        document.getElementById("signup-box").style.display = "none";
    } else {
        alert(res.msg);
    }
};

document.getElementById("logout-btn").onclick = async () => {
    await fetch("/logout",{method:"POST"});
    location.reload();
};

/*************************************************************
 * 6) 프로필 버튼
 *************************************************************/
el.profileBtn.onclick = () => Profile.open();
document.getElementById("avatar-upload-btn").onclick = () => Profile.uploadAvatar();
document.getElementById("profile-close-btn").onclick = () => Profile.close();

/*************************************************************
 * 7) 자동 로그인 체크
 *************************************************************/
fetch("/me").then(r=>r.json()).then(d => {
    if (d.loggedIn) {
        el.userInfo.innerText = "로그인됨: " + d.username;
        el.eloInfo.innerText = "ELO: " + d.elo;
        el.profileBtn.style.display = "block";
        el.onlineBtn.disabled = false;

        document.getElementById("login-box").style.display = "none";
        document.getElementById("signup-box").style.display = "none";
    }
});

/*************************************************************
 * 8) 게임 컨트롤 버튼
 *************************************************************/
document.getElementById("human-ai-btn").onclick = () => Game.startHumanAI();
document.getElementById("ai-ai-btn").onclick = () => Game.startAiVsAi();
el.onlineBtn.onclick = () => Game.startOnline();
document.getElementById("reset-btn").onclick = () => Game.reset();

/*************************************************************
 * 9) 시작
 *************************************************************/
Game.reset();

</script>

</body>
</html>
