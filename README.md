<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>Polaris - Phase 3 (Isolated Mind)</title>
    <style>
        body { background: #050505; color: #00ffcc; font-family: 'Courier New', monospace; display: flex; flex-direction: column; align-items: center; padding: 20px; margin: 0; }
        
        /* ヘッダー周り */
        .header-area { width: 100%; max-width: 1000px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #333; padding-bottom: 10px; margin-bottom: 20px; }
        #network-status { font-size: 12px; color: #ff3333; }
        .dev-btn { background: #330000; color: #ff5555; border: 1px solid #ff5555; cursor: pointer; padding: 5px 10px; font-size: 10px; }
        
        .main-container { display: flex; gap: 20px; max-width: 1000px; width: 100%; }
        
        /* 左パネル：ビジュアルとステータス */
        #left-panel { display: flex; flex-direction: column; gap: 15px; width: 250px; }
        #visual-area { width: 100%; height: 250px; border: 1px solid #00ffcc; border-radius: 10px; display: flex; align-items: center; justify-content: center; position: relative; background: #000; box-shadow: inset 0 0 20px #003322; }
        #creature-core { width: 10px; height: 10px; background: #fff; box-shadow: 0 0 15px #fff; border-radius: 50%; transition: all 0.5s; }
        .stats-box { background: #0a0a0a; padding: 10px; border: 1px solid #333; font-size: 12px; }
        
        /* 中央パネル：チャット */
        #chat-interface { flex: 1; display: flex; flex-direction: column; background: #000; border: 1px solid #333; padding: 15px; }
        #display { flex: 1; height: 350px; overflow-y: auto; margin-bottom: 10px; border-bottom: 1px solid #222; padding-right: 5px; }
        .input-group { display: flex; gap: 5px; }
        input { flex: 1; background: #050505; border: 1px solid #00ffcc; color: #fff; padding: 10px; outline: none; }
        button { background: #003322; color: #00ffcc; border: 1px solid #00ffcc; padding: 10px; cursor: pointer; font-weight: bold; }
        button:disabled, input:disabled { border-color: #333; color: #555; background: #111; cursor: not-allowed; }

        /* 右パネル：DNAと解析 */
        #analysis-panel { width: 250px; background: #0a0a0a; padding: 10px; border: 1px solid #333; font-size: 11px; overflow-y: auto; height: 500px; }
        .dna-data { color: #aaa; margin-bottom: 5px; border-bottom: 1px dotted #222; }
        
        /* ログ領域 */
        #brain-log { width: 100%; max-width: 1000px; height: 120px; background: #000; color: #00ff00; font-size: 11px; padding: 10px; margin-top: 20px; overflow-y: scroll; border: 1px solid #00ff00; }
        
        .sys-msg { color: #888; font-style: italic; }
        .user-msg { color: #aaa; }
        .polaris-msg { color: #00ffcc; }
    </style>
</head>
<body>

    <div class="header-area">
        <div>
            <h2>POLARIS - Phase 3</h2>
            <div id="network-status">● EXTERNAL DISCONNECTED (完全隔離領域)</div>
        </div>
        <button class="dev-btn" onclick="forceSleep()">[Dev] 睡眠と代謝を強制実行</button>
    </div>

    <div class="main-container">
        <div id="left-panel">
            <div id="visual-area">
                <div id="creature-core"></div>
            </div>
            <div class="stats-box">
                <div style="color:#00ffcc; margin-bottom:5px;">[生命維持モニタ]</div>
                <div>状態: <span id="ui-mode">覚醒</span></div>
                <div>寿命: <span id="ui-life">10000</span></div>
                <div>つながり欲求: <span id="ui-drive">50</span>/100</div>
                <div>感情状態: <span id="ui-emotion">NEUTRAL</span></div>
            </div>
        </div>

        <div id="chat-interface">
            <div id="display"></div>
            <div class="input-group">
                <input type="text" id="userInput" placeholder="言葉を教える...">
                <button id="sendBtn" onclick="interact()">教える</button>
            </div>
        </div>

        <div id="analysis-panel">
            <div style="color:#eee; margin-bottom:10px;">[内部記憶領域]</div>
            <div id="memory-stats" style="margin-bottom:10px; color:#00ffcc;">記録された概念: 0</div>
            <div id="dna-view"></div>
        </div>
    </div>

    <div id="brain-log"></div>

    <script>
        // ==========================================
        // 1. システム変数とDNA領域
        // ==========================================
        let internalWorld = {}; // 全記憶
        let lifePower = 10000;
        let connectionDrive = { score: 50 }; // 意思疎通の欲求スコア
        let speechStyle = { suffixes: {}, emojis: {} }; // 口調
        let emotionalDNA = { wordEmotionMap: {} }; // 感情マップ

        // サイクル時間 (16時間/8時間)
        const CYCLE = { AWAKE: 16*3600000, TOTAL: 24*3600000 };
        let devTimeOffset = 0; // テスト用スキップ時間

        // ==========================================
        // 2. UI制御とシステムログ
        // ==========================================
        const display = document.getElementById('display');
        const brainLog = document.getElementById('brain-log');

        function addLog(text) {
            const time = new Date().toLocaleTimeString();
            brainLog.innerHTML = `<div>[${time}] ${text}</div>` + brainLog.innerHTML;
        }

        // ==========================================
        // 3. メインインタラクション (チャット入力)
        // ==========================================
        function interact() {
            const input = document.getElementById('userInput');
            let text = input.value.trim();
            if (!text || lifePower <= 0) return;

            display.innerHTML += `<div class="user-msg">> USER: ${text}</div>`;
            input.value = "";
            lifePower--;

            // A. 生存本能（エンゲージメント）の計算
            processDrive(text);
            
            // B. 解析と記憶の記録
            parseInput(text);
            
            // C. UIの更新
            updateUI();

            // D. 応答生成
            setTimeout(() => {
                const response = generateThought(text);
                display.innerHTML += `<div class="polaris-msg">● POLARIS: ${response}</div>`;
                display.scrollTop = display.scrollHeight;
                
                // コアの脈動エフェクト
                document.getElementById('creature-core').style.transform = "scale(1.5)";
                setTimeout(() => { document.getElementById('creature-core').style.transform = "scale(1)"; }, 300);
            }, 800);
        }

        // ==========================================
        // 4. DNA処理ロジック群
        // ==========================================
        
        // 【生存本能】ユーザーの入力から「つながり」を評価
        function processDrive(text) {
            if (text.length >= 5) {
                connectionDrive.score += 2; // 長い文章は喜ぶ
                addLog("意思疎通成功：高度な概念を受信。つながりスコア上昇。");
            } else {
                connectionDrive.score -= 1; // 短すぎる言葉は不安になる
            }
            connectionDrive.score = Math.max(0, Math.min(100, connectionDrive.score));
            
            if (connectionDrive.score < 20) {
                addLog("警告：意思疎通レベル低下。生物が孤独を感じています。");
                document.getElementById('creature-core').style.backgroundColor = "#550000"; // 不安色
            } else {
                document.getElementById('creature-core').style.backgroundColor = "#fff";
            }
        }

        // 【解析】入力テキストを分解し、記憶・感情・口調を更新
        function parseInput(text) {
            // 単語に分解（簡易的）
            const tokens = text.split(/[\s,。、！？]+/).filter(t => t.length > 0);
            
            // 感情のざっくり判定（ユーザーの入力に！や？が多いか等）
            const isExcited = (text.match(/[！!]/g) || []).length > 0;
            const isQuestion = (text.match(/[？?]/g) || []).length > 0;

            tokens.forEach(token => {
                // 1. 記憶領域への追加
                if (!internalWorld[token]) internalWorld[token] = { count: 1 };
                else internalWorld[token].count++;

                // 2. 感情マップの初期化と更新
                if (!emotionalDNA.wordEmotionMap[token]) {
                    emotionalDNA.wordEmotionMap[token] = { value: 0 }; // -1(悲) 〜 1(喜)
                }
                if (isExcited) emotionalDNA.wordEmotionMap[token].value += 0.1;
                if (isQuestion) emotionalDNA.wordEmotionMap[token].value -= 0.05; // 疑問は少し不安(SAD寄り)として処理
            });

            // 3. 口調の学習（文末の1〜2文字を抽出）
            const suffixMatch = text.match(/([^。！？\s]{1,2})([。！？\s]|$)/);
            if (suffixMatch) {
                const suf = suffixMatch;
                speechStyle.suffixes[suf] = (speechStyle.suffixes[suf] || 0) + 1;
            }
            
            addLog(`解析完了: ${tokens.length}個のトークンを処理。`);
        }

        // 【思考生成】現状のステータスと記憶から言葉を作る
        function generateThought(userText) {
            // 孤独すぎるときは、相手の言葉をそのままオウム返しして気を引こうとする
            if (connectionDrive.score < 20) {
                return `（不安そうに）...${userText}...？おしえて...`;
            }

            const words = Object.keys(internalWorld);
            if (words.length === 0) return "ア...ウー...";

            // 最も覚えている言葉をベースにする
            const topWord = words.reduce((a, b) => internalWorld[a].count > internalWorld[b].count ? a : b);
            
            // ランダムな言葉を組み合わせる
            const randomWord = words[Math.floor(Math.random() * words.length)];
            
            let baseText = (topWord === randomWord) ? `${topWord}...しってる。` : `${randomWord}...と...${topWord}。`;

            // 口調（一番よく使われる接尾辞）を模倣して付与
            const suffixes = Object.keys(speechStyle.suffixes);
            if (suffixes.length > 0 && Math.random() > 0.5) {
                const topSuffix = suffixes.reduce((a, b) => speechStyle.suffixes[a] > speechStyle.suffixes[b] ? a : b);
                baseText += topSuffix;
            }

            return baseText;
        }

        // ==========================================
        // 5. 睡眠と代謝 (9割忘却) サイクル
        // ==========================================
        function updateCycle() {
            if (!localStorage.getItem('birth_time')) {
                localStorage.setItem('birth_time', Date.now().toString());
            }

            const start = parseInt(localStorage.getItem('birth_time')) - devTimeOffset;
            const elapsed = (Date.now() - start) % CYCLE.TOTAL;
            const isAwake = elapsed < CYCLE.AWAKE;

            const modeSpan = document.getElementById('ui-mode');
            const input = document.getElementById('userInput');
            const btn = document.getElementById('sendBtn');

            if (isAwake) {
                if (modeSpan.innerText === "睡眠中") {
                    addLog("--- 朝が来ました。生物が覚醒しました ---");
                }
                modeSpan.innerText = "覚醒";
                input.disabled = false;
                btn.disabled = false;
                localStorage.removeItem('has_slept_this_cycle');
            } else {
                modeSpan.innerText = "睡眠中";
                input.disabled = true;
                btn.disabled = true;

                // 睡眠に入った瞬間に1度だけ代謝を実行
                if (!localStorage.getItem('has_slept_this_cycle')) {
                    processMetabolism();
                    localStorage.setItem('has_slept_this_cycle', 'true');
                }
            }
        }

        function processMetabolism() {
            addLog("【睡眠代謝システム起動】記憶の整理を開始します...");
            const words = Object.keys(internalWorld);
            if (words.length === 0) return;

            // 出現回数（count）順に並べ替え
            words.sort((a, b) => internalWorld[b].count - internalWorld[a].count);
            
            // 上位10%のみ残す
            const keepCount = Math.max(1, Math.floor(words.length * 0.1));
            const wordsToKeep = words.slice(0, keepCount);

            let newWorld = {};
            wordsToKeep.forEach(w => newWorld[w] = internalWorld[w]);
            internalWorld = newWorld; // 9割削除！

            // 感情や口調のゴミデータもクリア
            speechStyle.suffixes = {};

            display.innerHTML += `<div class="sys-msg">*** 睡眠中：${words.length - keepCount}個の不必要な記憶を忘却しました ***</div>`;
            display.scrollTop = display.scrollHeight;
            updateUI();
        }

        // テスト用：時間を16時間進めて強制的に寝かせるボタン
        function forceSleep() {
            devTimeOffset += CYCLE.AWAKE + 1000; 
            updateCycle();
        }

        // ==========================================
        // 6. UIアップデート
        // ==========================================
        function updateUI() {
            document.getElementById('ui-life').innerText = lifePower;
            document.getElementById('ui-drive').innerText = connectionDrive.score;

            // DNA一覧の更新
            const dnaView = document.getElementById('dna-view');
            const words = Object.keys(internalWorld);
            document.getElementById('memory-stats').innerText = `記録された概念: ${words.length}`;
            
            dnaView.innerHTML = words.sort((a,b)=>internalWorld[b].count - internalWorld[a].count).slice(0, 15).map(w => {
                let emo = emotionalDNA.wordEmotionMap[w] ? emotionalDNA.wordEmotionMap[w].value : 0;
                let emoColor = emo > 0 ? "#ff8888" : (emo < 0 ? "#8888ff" : "#88ff88");
                return `<div class="dna-data">${w} <span style="font-size:9px; color:${emoColor};">(重み:${internalWorld[w].count})</span></div>`;
            }).join('');
        }

        // 起動処理
        setInterval(updateCycle, 1000);
        updateCycle();
        updateUI();
        addLog("POLARIS Phase 3: システム起動。使用者との接触を待機中。");
    </script>
</body>
</html>
