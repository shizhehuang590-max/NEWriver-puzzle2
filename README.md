[index (7).html](https://github.com/user-attachments/files/26318804/index.7.html)
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>A* vs BFS 搜尋樹視覺化 - 過河謎題</title>
    <style>
        body { font-family: 'Segoe UI', Tahoma, sans-serif; background-color: #1e272e; color: #ecf0f1; margin: 0; padding: 20px; text-align: center; }
        h1 { color: #0fb9b1; margin-bottom: 5px; }
        .subtitle { color: #808e9b; margin-bottom: 20px; font-size: 14px; }
        .container { display: flex; justify-content: space-between; gap: 20px; height: 75vh; }
        .panel { flex: 1; background: #2d3436; border-radius: 12px; display: flex; flex-direction: column; overflow: hidden; box-shadow: 0 10px 20px rgba(0,0,0,0.5); }
        .header { padding: 15px; background: #222f3e; border-bottom: 2px solid #576574; }
        .header h2 { margin: 0; font-size: 20px; color: #feca57; }
        .stats { display: flex; justify-content: space-around; font-size: 13px; margin-top: 10px; color: #c8d6e5; }
        .stats span { font-weight: bold; color: #48dbfb; }
        canvas { flex: 1; width: 100%; height: 100%; background: #2d3436; cursor: grab; }
        .controls { margin-top: 20px; padding: 15px; background: #2d3436; border-radius: 12px; }
        button { background: #0fb9b1; color: white; border: none; padding: 12px 25px; font-size: 16px; border-radius: 8px; cursor: pointer; font-weight: bold; transition: 0.2s; margin: 0 10px; }
        button:hover { background: #0097e6; transform: scale(1.05); }
        button:disabled { background: #576574; cursor: not-allowed; transform: none; }
        .legend { margin-top: 15px; display: flex; justify-content: center; gap: 20px; font-size: 14px; }
        .legend-item { display: flex; align-items: center; gap: 8px; }
        .dot { width: 12px; height: 12px; border-radius: 50%; }
    </style>
</head>
<body>

<h1>🌳 搜尋樹同步推演：A* 演算法 vs BFS 演算法</h1>
<div class="subtitle">即時運算過河謎題的狀態空間，觀察兩者的決策差異</div>

<div class="container">
    <div class="panel">
        <div class="header">
            <h2>BFS (廣度優先) - 穩紮穩打平推流</h2>
            <div class="stats">
                <div>已探索節點: <span id="bfs-explored">0</span></div>
                <div>最大深度: <span id="bfs-depth">0</span></div>
                <div>佇列大小: <span id="bfs-queue">0</span></div>
            </div>
        </div>
        <canvas id="canvas-bfs"></canvas>
    </div>

    <div class="panel">
        <div class="header">
            <h2>A* Search - 啟發式特攻隊</h2>
            <div class="stats">
                <div>已探索節點: <span id="astar-explored">0</span></div>
                <div>最大深度: <span id="astar-depth">0</span></div>
                <div>佇列大小: <span id="astar-queue">0</span></div>
            </div>
        </div>
        <canvas id="canvas-astar"></canvas>
    </div>
</div>

<div class="controls">
    <button id="startBtn" onclick="startSearch()">▶ 雙引擎同步推演</button>
    <button onclick="location.reload()">🔄 重設</button>
    <div class="legend">
        <div class="legend-item"><div class="dot" style="background: #3498db;"></div> 邊界 (Frontier)</div>
        <div class="legend-item"><div class="dot" style="background: #2ecc71;"></div> 已探索 (Explored)</div>
        <div class="legend-item"><div class="dot" style="background: #e74c3c;"></div> 違規/剪枝 (Invalid)</div>
        <div class="legend-item"><div class="dot" style="background: #95a5a6;"></div> 重複狀態 (Visited)</div>
        <div class="legend-item"><div class="dot" style="background: #f1c40f; box-shadow: 0 0 10px #f1c40f;"></div> 目標 (Goal)</div>
    </div>
</div>

<script>
    // 角色對應：0:Boat, 1:F, 2:M, 3:S1, 4:S2, 5:D1, 6:D2, 7:P, 8:Dog
    const START_STATE = [0,0,0,0,0,0,0,0,0];
    const GOAL_STATE = [1,1,1,1,1,1,1,1,1];

    // 檢查狀態是否合法 (根據題目規則)
    function isValid(s) {
        const [B, F, M, S1, S2, D1, D2, P, G] = s;
        // 規則1：僕人不在，狗咬全家
        if (G !== P) {
            // 如果狗跟任何家人在一起，就會咬人
            if (G === 0 && (F===0 || M===0 || S1===0 || S2===0 || D1===0 || D2===0)) return false;
            if (G === 1 && (F===1 || M===1 || S1===1 || S2===1 || D1===1 || D2===1)) return false;
        }
        // 規則2：媽媽不在，爸爸打女兒
        if (F !== M) {
            if (F === 0 && (D1===0 || D2===0)) return false;
            if (F === 1 && (D1===1 || D2===1)) return false;
        }
        return true;
    }

    // 取得所有可能的下一步
    function getSuccessors(s) {
        let successors = [];
        let B = s[0];
        // 會開船的人的索引：1(F), 2(M), 7(P)
        let drivers = [1, 2, 7];
        
        // 枚舉所有1人或2人的組合
        for (let i = 1; i <= 8; i++) {
            for (let j = i; j <= 8; j++) {
                // 如果兩個人都在船的那一岸
                if (s[i] === B && s[j] === B) {
                    // 必須至少有一人會開船
                    if (drivers.includes(i) || drivers.includes(j)) {
                        let newState = [...s];
                        newState[0] = 1 - B; // 船過河
                        newState[i] = 1 - B;
                        newState[j] = 1 - B;
                        successors.push(newState);
                    }
                }
            }
        }
        return successors;
    }

    // A* 的啟發函數 h(n)：左岸剩下的人數 / 2 (因為一次最多載兩人)
    function heuristic(s) {
        let leftCount = 0;
        for(let i=1; i<=8; i++) { if(s[i] === 0) leftCount++; }
        return Math.ceil(leftCount / 2);
    }

    class SearchEngine {
        constructor(type, canvasId) {
            this.type = type; // 'BFS' or 'ASTAR'
            this.canvas = document.getElementById(canvasId);
            this.ctx = this.canvas.getContext('2d');
            this.resizeCanvas();
            
            this.nodes = []; // 所有節點供渲染
            this.visited = new Set();
            this.queue = []; // Frontier
            this.isDone = false;
            
            this.stats = { explored: 0, maxDepth: 0 };
            this.levelCounts = {}; // 用於排版
            
            // 初始化根節點
            let root = this.createNode(START_STATE, null, 0);
            this.queue.push(root);
            this.nodes.push(root);
            this.visited.add(root.id);
        }

        resizeCanvas() {
            this.canvas.width = this.canvas.clientWidth * 2;
            this.canvas.height = this.canvas.clientHeight * 2;
            this.ctx.scale(2, 2);
        }

        createNode(state, parent, depth) {
            let id = state.join('');
            if (!this.levelCounts[depth]) this.levelCounts[depth] = 0;
            
            let node = {
                id: id, state: state, parent: parent, depth: depth,
                status: 'frontier', // frontier, explored, invalid, visited, goal
                g: depth,
                h: heuristic(state),
                x: 0, y: depth * 40 + 30, // 初始位置
                levelIndex: this.levelCounts[depth]++
            };
            node.f = node.g + node.h;
            return node;
        }

        step() {
            if (this.isDone || this.queue.length === 0) return;

            // 取出節點
            let current;
            if (this.type === 'BFS') {
                current = this.queue.shift(); // FIFO
            } else {
                // A* 取 f 值最小的 (若 f 相同則取 g 最大的，即深入優先)
                this.queue.sort((a, b) => a.f === b.f ? b.g - a.g : a.f - b.f);
                current = this.queue.shift();
            }

            current.status = 'explored';
            this.stats.explored++;
            if (current.depth > this.stats.maxDepth) this.stats.maxDepth = current.depth;

            // 檢查是否為目標
            if (current.id === GOAL_STATE.join('')) {
                current.status = 'goal';
                this.isDone = true;
                this.highlightPath(current);
                this.updateUI();
                this.render();
                return;
            }

            // 展開子節點
            let successors = getSuccessors(current.state);
            for (let succ of successors) {
                let isVal = isValid(succ);
                let succId = succ.join('');
                let child = this.createNode(succ, current, current.depth + 1);
                
                if (!isVal) {
                    child.status = 'invalid';
                    this.nodes.push(child);
                } else if (this.visited.has(succId)) {
                    child.status = 'visited';
                    this.nodes.push(child);
                } else {
                    this.visited.add(succId);
                    this.queue.push(child);
                    this.nodes.push(child);
                }
            }

            this.updateUI();
            this.render();
        }

        highlightPath(node) {
            let curr = node;
            while(curr) {
                if(curr.status === 'explored') curr.status = 'goal';
                curr = curr.parent;
            }
        }

        updateUI() {
            let prefix = this.type === 'BFS' ? 'bfs' : 'astar';
            document.getElementById(`${prefix}-explored`).innerText = this.stats.explored;
            document.getElementById(`${prefix}-depth`).innerText = this.stats.maxDepth;
            document.getElementById(`${prefix}-queue`).innerText = this.queue.length;
        }

        render() {
            let w = this.canvas.width / 2;
            let h = this.canvas.height / 2;
            this.ctx.clearRect(0, 0, w, h);
            
            // 自動縮放與置中計算
            let maxDepthStr = Math.max(...Object.keys(this.levelCounts));
            let maxNodesInLevel = Math.max(...Object.values(this.levelCounts));
            
            // 計算每個節點的動態 X 座標
            this.nodes.forEach(n => {
                let totalInLevel = this.levelCounts[n.depth];
                let spacing = w / (totalInLevel + 1);
                // 為了讓 A* 的直線看起來更集中，增加向中心靠攏的權重
                n.x = spacing * (n.levelIndex + 1);
            });

            // 繪製連線
            this.ctx.lineWidth = 1.5;
            this.nodes.forEach(n => {
                if (n.parent) {
                    this.ctx.beginPath();
                    this.ctx.moveTo(n.parent.x, n.parent.y);
                    this.ctx.lineTo(n.x, n.y);
                    // 根據狀態給線條顏色
                    if (n.status === 'goal') { this.ctx.strokeStyle = '#f1c40f'; this.ctx.lineWidth = 3; }
                    else if (n.status === 'invalid') this.ctx.strokeStyle = 'rgba(231, 76, 60, 0.2)';
                    else this.ctx.strokeStyle = 'rgba(189, 195, 199, 0.4)';
                    this.ctx.stroke();
                    this.ctx.lineWidth = 1.5;
                }
            });

            // 繪製節點
            const colors = {
                'frontier': '#3498db',
                'explored': '#2ecc71',
                'invalid': '#e74c3c',
                'visited': '#95a5a6',
                'goal': '#f1c40f'
            };

            this.nodes.forEach(n => {
                this.ctx.beginPath();
                this.ctx.arc(n.x, n.y, n.status === 'goal' ? 6 : 4, 0, Math.PI * 2);
                this.ctx.fillStyle = colors[n.status];
                this.ctx.fill();
                if(n.status === 'goal') {
                    this.ctx.shadowBlur = 10;
                    this.ctx.shadowColor = '#f1c40f';
                    this.ctx.fill();
                    this.ctx.shadowBlur = 0;
                }
            });
        }
    }

    let bfsEngine, astarEngine;
    let timer;

    function startSearch() {
        document.getElementById('startBtn').disabled = true;
        bfsEngine = new SearchEngine('BFS', 'canvas-bfs');
        astarEngine = new SearchEngine('ASTAR', 'canvas-astar');
        
        // 使用定時器同步推演，產生動畫效果
        timer = setInterval(() => {
            if (bfsEngine.isDone && astarEngine.isDone) {
                clearInterval(timer);
                return;
            }
            // 每 Tick 各走 2 步，加速渲染
            for(let i=0; i<2; i++) {
                if(!bfsEngine.isDone) bfsEngine.step();
                if(!astarEngine.isDone) astarEngine.step();
            }
        }, 50); // 50 毫秒一幀
    }
</script>
</body>
</html>
