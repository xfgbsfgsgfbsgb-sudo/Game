<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Crystal Swap v10.0: Финальная Версия с Исправлением Ошибки</title>
    <style>
        body {
            margin: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background-color: #2c3e50;
            font-family: 'Arial', sans-serif;
            color: #ecf0f1;
            touch-action: none; 
            overflow: hidden;
        }
        canvas {
            border: 5px solid #34495e;
            background-color: #1a1a1a; 
            box-shadow: 0 0 20px rgba(100, 100, 255, 0.5);
            width: 500px;
            height: 500px;
        }

        #game-container {
            position: relative;
        }

        /* HUD */
        #hud-container {
            position: absolute;
            top: 10px;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(0, 0, 0, 0.7);
            padding: 10px 15px;
            border-radius: 8px;
            font-size: 20px;
            border: 2px solid #f1c40f;
            z-index: 100;
            display: flex;
            gap: 20px;
        }

        /* Панель паузы */
        #pause-overlay {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.8);
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            z-index: 200;
            pointer-events: auto;
        }
        .pause-button, .menu-button {
            padding: 10px 20px;
            margin: 10px;
            font-size: 24px;
            cursor: pointer;
            border: 2px solid #f1c40f;
            background: #2ecc71;
            color: white;
            border-radius: 8px;
        }
    </style>
</head>
<body>
    <div id="game-container">
        <canvas id="gameCanvas" width="500" height="500"></canvas>
        <div id="hud-container">
            🎯 Цель: <span id="target-value">0</span>
            ✨ Очки: <span id="score-value">0</span>
            <button id="pause-button">⏸️</button>
        </div>

        <div id="pause-overlay" style="display: none;">
            <h2>Пауза</h2>
            <button class="pause-button" onclick="togglePause()">Продолжить</button>
            <button class="menu-button" onclick="goToMenu()">В меню (Сброс прогресса)</button>
        </div>
    </div>

    <script>
        // ГЛОБАЛЬНЫЕ ПЕРЕМЕННЫЕ
        let canvas, ctx, scoreValue, targetValue, pauseOverlay, pauseButton;

        // --- КОНФИГУРАЦИЯ ИГРЫ ---
        const GRID_SIZE = 8;
        const CANVAS_SIZE = 500;
        const CELL_SIZE = CANVAS_SIZE / GRID_SIZE;
        const CRYSTAL_TYPES = 6; 
        
        // Цвета
        const COLORS = ['#e74c3c', '#3498db', '#2ecc71', '#f1c40f', '#9b59b6', '#34495e'];
        const ELECTRIC_COLOR = '#ecf0f1'; 
        const ENERGY_COLOR = '#ff00ff'; 
        const BOOM_COLOR = '#ffff00'; 

        // Состояния игры
        let gameState = 'MENU';
        
        let grid = []; 
        let currentLevel = 1;
        let score = 0;
        let isProcessing = false; 

        // --- ДАННЫЕ УРОВНЕЙ ---
        const LEVELS = [
            { id: 1, target: 1000, color: '#2ecc71', unlocked: true },
            { id: 2, target: 2500, color: '#3498db', unlocked: false },
            { id: 3, target: 4000, color: '#f1c40f', unlocked: false },
            { id: 4, target: 6000, color: '#e74c3c', unlocked: false },
            { id: 5, target: 8500, color: '#9b59b6', unlocked: false },
            { id: 6, target: 11000, color: '#34495e', unlocked: false },
            { id: 7, target: 14000, color: '#2ecc71', unlocked: false },
            { id: 8, target: 17000, color: '#3498db', unlocked: false },
            { id: 9, target: 20000, color: '#f1c40f', unlocked: false },
            { id: 10, target: 25000, color: '#e74c3c', unlocked: false },
        ];
        
        // --- КЛАСС КРИСТАЛЛА ДЛЯ АНИМАЦИИ ---
        class Crystal {
            constructor(r, c, type, originalType) {
                this.r = r; 
                this.c = c; 
                this.type = type; 
                this.originalType = originalType; 
                this.yOffset = 0; 
                this.rotation = 0; 
                this.scale = 1; 
                this.isBooming = false;
                
                this.targetR = r; 
                this.targetC = c; 
            }
        }

        // --- DRAG AND DROP ПЕРЕМЕННЫЕ ---
        let dragInfo = null; 
        const SWAP_THRESHOLD = CELL_SIZE * 0.4; 
        let animationFrame = null;

        // --- ИНИЦИАЛИЗАЦИЯ И ЗАПУСК УРОВНЯ ---

        function initLevel(levelId) {
            currentLevel = levelId;
            score = 0;
            isProcessing = false;
            dragInfo = null;

            const target = LEVELS.find(l => l.id === levelId).target;
            targetValue.textContent = target;

            grid = [];
            for (let r = 0; r < GRID_SIZE; r++) {
                grid[r] = [];
                for (let c = 0; c < GRID_SIZE; c++) {
                    let type;
                    do {
                        type = Math.floor(Math.random() * CRYSTAL_TYPES);
                    } while (checkMatchAtStart(r, c, type));
                    grid[r][c] = new Crystal(r, c, type, type);
                    grid[r][c].yOffset = -r * CELL_SIZE; 
                }
            }
            
            updateHUD();
            gameState = 'PLAYING';
            document.getElementById('hud-container').style.display = 'flex';
            if (!animationFrame) {
                animationFrame = requestAnimationFrame(drawGame);
            }
        }
        
        function checkMatchAtStart(r, c, type) {
            if (c >= 2 && grid[r][c - 1] && grid[r][c - 1].type === type && grid[r][c - 2] && grid[r][c - 2].type === type) return true;
            if (r >= 2 && grid[r - 1][c] && grid[r - 1][c].type === type && grid[r - 2][c] && grid[r - 2][c].type === type) return true;
            return false;
        }

        // --- ИСПРАВЛЕННАЯ ФУНКЦИЯ ЗАПУСКА ИГРЫ (ГАРАНТИРУЕТ ЭКРАН) ---

        function startGame() {
            canvas = document.getElementById('gameCanvas');
            ctx = canvas.getContext('2d');
            scoreValue = document.getElementById('score-value');
            targetValue = document.getElementById('target-value');
            pauseOverlay = document.getElementById('pause-overlay');
            pauseButton = document.getElementById('pause-button');
            
            setupEventListeners();
            
            // Запускаем сразу первый уровень, чтобы экран появился
            initLevel(1); 
            
            if (!animationFrame) {
                animationFrame = requestAnimationFrame(drawGame);
            }

            window.onbeforeunload = function() {
                if (gameState === 'PLAYING' || gameState === 'PAUSED') {
                    return "Ваш текущий прогресс будет потерян, если вы покинете игру.";
                }
            };
        }

        // --- ОСТАЛЬНЫЕ ФУНКЦИИ УПРАВЛЕНИЯ ИГРОЙ (Без изменений) ---

        function togglePause() {
            if (gameState === 'PLAYING') {
                gameState = 'PAUSED';
                pauseOverlay.style.display = 'flex';
            } else if (gameState === 'PAUSED') {
                gameState = 'PLAYING';
                pauseOverlay.style.display = 'none';
            }
        }

        function goToMenu() {
            if (gameState === 'PAUSED') {
                 if (confirm("Вы уверены? Весь текущий прогресс будет потерян.")) {
                    gameState = 'MENU';
                    pauseOverlay.style.display = 'none';
                    score = 0;
                    grid = [];
                    currentLevel = 1;
                    LEVELS.forEach((l, i) => l.unlocked = (i === 0));
                    updateHUD();
                    document.getElementById('hud-container').style.display = 'none';
                }
            }
        }

        // --- ОТРИСОВКА ИГРЫ (Без изменений) ---

        function drawGame() {
            if (!ctx) return; 
            
            ctx.clearRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);

            if (gameState === 'MENU') {
                drawMap();
            } else {
                ctx.fillStyle = '#1a1a1a'; 
                ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);

                if (gameState === 'PLAYING') {
                    updateAnimations();
                }

                for (let r = 0; r < GRID_SIZE; r++) {
                    for (let c = 0; c < GRID_SIZE; c++) {
                        const crystal = grid[r][c];

                        if (!crystal) continue;

                        const drawX = crystal.c * CELL_SIZE + (crystal.targetC - crystal.c) * CELL_SIZE;
                        const drawY = crystal.r * CELL_SIZE + (crystal.targetR - crystal.r) * CELL_SIZE + crystal.yOffset;

                        if (dragInfo && dragInfo.startR === r && dragInfo.startC === c) {
                             const dx = dragInfo.currentX - dragInfo.startX;
                             const dy = dragInfo.currentY - dragInfo.startY;
                             const maxOffset = CELL_SIZE * 0.5;
                             const limitedDx = Math.max(-maxOffset, Math.min(maxOffset, dx));
                             const limitedDy = Math.max(-maxOffset, Math.min(maxOffset, dy));
                             
                             drawCrystal(dragInfo.startC * CELL_SIZE + limitedDx, dragInfo.startR * CELL_SIZE + limitedDy, crystal.type, crystal.originalType, 1, 0, false);
                        } else {
                            drawCrystal(drawX, drawY, crystal.type, crystal.originalType, crystal.scale, crystal.rotation, crystal.isBooming);
                        }
                    }
                }

                if (dragInfo) {
                    ctx.strokeStyle = '#ecf0f1'; 
                    ctx.lineWidth = 4;
                    ctx.strokeRect(dragInfo.startC * CELL_SIZE + 2, dragInfo.startR * CELL_SIZE + 2, CELL_SIZE - 4, CELL_SIZE - 4);
                }
            }

            animationFrame = requestAnimationFrame(drawGame);
        }

        function drawMap() {
            ctx.fillStyle = '#27ae60'; 
            ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);
            
            ctx.fillStyle = '#e67e22'; 
            ctx.fillRect(50, 400, 20, 50);
            ctx.fillRect(430, 380, 30, 70);
            ctx.fillStyle = '#2ecc71'; 
            ctx.beginPath();
            ctx.arc(60, 400, 30, 0, Math.PI * 2);
            ctx.fill();
            ctx.beginPath();
            ctx.arc(445, 380, 40, 0, Math.PI * 2);
            ctx.fill();

            ctx.fillStyle = '#ecf0f1';
            ctx.font = '36px Arial';
            ctx.textAlign = 'center';
            ctx.fillText('Карта Уровней', CANVAS_SIZE / 2, 50);
            
            const positions = [
                { x: 100, y: 150 }, { x: 250, y: 120 }, { x: 400, y: 150 },
                { x: 400, y: 300 }, { x: 250, y: 350 }, { x: 100, y: 300 },
                { x: 100, y: 450 }, { x: 250, y: 420 }, { x: 400, y: 450 },
                { x: 250, y: 230 }
            ];

            LEVELS.forEach((level, index) => {
                const pos = positions[index];
                const radius = 25;
                const isUnlocked = level.unlocked;
                const isCurrent = level.id === currentLevel;

                if (index > 0) {
                    const prevPos = positions[index - 1];
                    ctx.strokeStyle = isUnlocked ? '#f1c40f' : '#7f8c8d';
                    ctx.lineWidth = 3;
                    ctx.beginPath();
                    ctx.moveTo(prevPos.x, prevPos.y);
                    ctx.lineTo(pos.x, pos.y);
                    ctx.stroke();
                }

                ctx.beginPath();
                ctx.arc(pos.x, pos.y, radius, 0, Math.PI * 2);

                if (isUnlocked) {
                    ctx.fillStyle = level.color;
                } else {
                    ctx.fillStyle = '#95a5a6';
                }

                ctx.fill();

                if (isCurrent && gameState !== 'PLAYING') {
                    ctx.strokeStyle = '#f1c40f';
                    ctx.lineWidth = 5;
                    ctx.stroke();
                }

                ctx.fillStyle = 'white';
                ctx.font = 'bold 18px Arial';
                ctx.textAlign = 'center';
                ctx.fillText(level.id, pos.x, pos.y + 6);
            });
        }

        function drawCrystal(x, y, type, originalType, scale, rotation, isBooming) {
            let color;
            let glow = false;
            let shadowColor = 'transparent';
            
            if (isBooming) {
                color = BOOM_COLOR;
            } else if (type === 6) {
                color = ELECTRIC_COLOR;
                glow = true;
                shadowColor = originalType >= 0 && originalType < COLORS.length ? COLORS[originalType] : ELECTRIC_COLOR;
            } else if (type === 7) {
                color = ENERGY_COLOR;
                glow = true;
                shadowColor = ENERGY_COLOR;
            } else {
                color = COLORS[type];
            }

            const size = CELL_SIZE * 0.7 * scale;
            const cx = x + CELL_SIZE / 2;
            const cy = y + CELL_SIZE / 2;
            
            ctx.save();
            ctx.translate(cx, cy);
            ctx.rotate(rotation);

            if (glow && !isBooming) {
                ctx.shadowBlur = 15;
                ctx.shadowColor = shadowColor;
            }

            ctx.fillStyle = color;
            ctx.beginPath();
            ctx.moveTo(0, -size / 2); 
            ctx.lineTo(size / 3, -size / 4);
            ctx.lineTo(size / 2, 0); 
            ctx.lineTo(size / 3, size / 4);
            ctx.lineTo(0, size / 2); 
            ctx.lineTo(-size / 3, size / 4);
            ctx.lineTo(-size / 2, 0); 
            ctx.lineTo(-size / 3, -size / 4);
            ctx.closePath();
            ctx.fill();
            
            ctx.shadowBlur = 0; 
            ctx.fillStyle = 'rgba(255, 255, 255, 0.5)';
            ctx.beginPath();
            ctx.moveTo(0, -size / 2);
            ctx.lineTo(size / 3, -size / 4);
            ctx.lineTo(0, 0);
            ctx.closePath();
            ctx.fill();
            
            ctx.restore();
            ctx.shadowColor = 'transparent';
        }


        // --- ФУНКЦИЯ АНИМАЦИИ: БЛОК B (Строгий контроль ссылок) ---

        function updateAnimations() {
            if (gameState !== 'PLAYING') return;

            const speed = CELL_SIZE / 8; 
            let processingComplete = true;

            for (let r = 0; r < GRID_SIZE; r++) {
                for (let c = 0; c < GRID_SIZE; c++) {
                    const crystal = grid[r][c];
                    if (!crystal) continue;

                    // 1. Анимация отката/сдвига к целевой позиции
                    if (crystal.r !== crystal.targetR || crystal.c !== crystal.targetC) {
                        processingComplete = false;
                        
                        // Сохраняем координаты ячейки, из которой мы начинаем анимацию
                        // ВАЖНО: startR/startC - это место, где ссылка на кристалл находится в grid
                        const startR = Math.round(crystal.r); 
                        const startC = Math.round(crystal.c);

                        // Сдвиг по R и C
                        const dr = crystal.targetR - crystal.r;
                        if (Math.abs(dr) > 0) {
                            const moveR = Math.min(speed, Math.abs(dr));
                            crystal.r += dr > 0 ? moveR : -moveR;
                        }
                        
                        const dc = crystal.targetC - crystal.c;
                        if (Math.abs(dc) > 0) {
                            const moveC = Math.min(speed, Math.abs(dc));
                            crystal.c += dc > 0 ? moveC : -moveC;
                        }

                        // Проверка, что мы почти достигли цели, округление
                        if (Math.abs(crystal.targetR - crystal.r) < speed) {
                            crystal.r = crystal.targetR;
                        }
                        if (Math.abs(crystal.targetC - crystal.c) < speed) {
                            crystal.c = crystal.targetC;
                        }

                        // !!! БЛОК B: СТРОГИЙ КОНТРОЛЬ ССЫЛОК ПРИ ЗАВЕРШЕНИИ ДВИЖЕНИЯ
                        if (crystal.r === crystal.targetR && crystal.c === crystal.targetC) {
                            // Кристалл достиг своей цели (targetR/targetC)
                            
                            // Мы перемещаем кристалл в grid только если он пришел из ДРУГОЙ ячейки.
                            if (startR !== crystal.targetR || startC !== crystal.targetC) {
                                
                                // 1. Если в ячейке, откуда он пришел (startR/startC), все еще лежит ссылка на этот кристалл, удаляем ее
                                if (startR >= 0 && startR < GRID_SIZE && startC >= 0 && startC < GRID_SIZE && grid[startR][startC] === crystal) {
                                    grid[startR][startC] = null; 
                                }
                                
                                // 2. Убеждаемся, что в целевой ячейке (targetR/targetC) стоит именно этот кристалл
                                grid[crystal.targetR][crystal.targetC] = crystal; 
                            }
                        }
                        // КОНЕЦ БЛОКА B
                    }

                    // 2. Анимация падения (Без изменений)
                    if (crystal.yOffset !== 0) {
                        processingComplete = false;
                        const distance = Math.abs(crystal.yOffset);
                        const move = Math.min(speed, distance);
                        
                        if (crystal.yOffset > 0) {
                            crystal.yOffset -= move;
                        } else {
                            crystal.yOffset += move;
                        }
                        
                        crystal.rotation += Math.PI / 16;
                        if (Math.abs(crystal.yOffset) < speed) {
                            crystal.yOffset = 0;
                            crystal.rotation = 0;
                        }
                    } 

                    // 3. Анимация исчезновения (Без изменений)
                    if (crystal.isBooming) {
                        processingComplete = false;
                        crystal.scale -= 0.1;
                        crystal.rotation += Math.PI / 10;
                        if (crystal.scale <= 0) {
                            if (grid[r] && grid[r][c] === crystal) {
                                grid[r][c] = null;
                            }
                        }
                    }
                }
            }
            
            if (processingComplete) {
                 isProcessing = false;
            } else {
                 isProcessing = true;
            }
        }

        // --- ЛОГИКА DRAG & DROP (Без изменений) ---

        function setupEventListeners() {
            if (!canvas || !pauseButton) return; 

            canvas.addEventListener('mousedown', handleClickOrDrag);
            window.addEventListener('mousemove', doDrag);
            window.addEventListener('mouseup', endDrag);

            canvas.addEventListener('touchstart', handleClickOrDrag);
            window.addEventListener('touchmove', doDrag);
            window.addEventListener('touchend', endDrag);

            pauseButton.onclick = togglePause;
        }

        function getMousePos(e) {
            const rect = canvas.getBoundingClientRect();
            const clientX = e.clientX || (e.touches ? e.touches[0].clientX : 0);
            const clientY = e.clientY || (e.touches ? e.touches[0].clientY : 0);

            const x = clientX - rect.left;
            const y = clientY - rect.top;

            return {
                x: x,
                y: y,
                col: Math.floor(x / CELL_SIZE),
                row: Math.floor(y / CELL_SIZE)
            };
        }

        function handleClickOrDrag(e) {
            e.preventDefault();
            if (gameState === 'PAUSED' || isProcessing) return; 

            const pos = getMousePos(e);
            
            if (gameState === 'MENU') {
                handleMenuClick(pos);
                return;
            }

            if (pos.row >= 0 && pos.row < GRID_SIZE && pos.col >= 0 && pos.col < GRID_SIZE && grid[pos.row] && grid[pos.row][pos.col] !== null) {
                dragInfo = {
                    startR: pos.row,
                    startC: pos.col,
                    startX: pos.x,
                    startY: pos.y,
                    currentX: pos.x,
                    currentY: pos.y
                };
            }
        }
        
        function handleMenuClick(pos) {
            const positions = [
                { x: 100, y: 150 }, { x: 250, y: 120 }, { x: 400, y: 150 },
                { x: 400, y: 300 }, { x: 250, y: 350 }, { x: 100, y: 300 },
                { x: 100, y: 450 }, { x: 250, y: 420 }, { x: 400, y: 450 },
                { x: 250, y: 230 }
            ];
            
            LEVELS.forEach((level, index) => {
                const mapPos = positions[index];
                const radius = 25;
                const dist = Math.sqrt(Math.pow(pos.x - mapPos.x, 2) + Math.pow(pos.y - mapPos.y, 2));

                if (dist < radius) {
                    if (level.unlocked) {
                        initLevel(level.id);
                    } else {
                        alert("Уровень заблокирован! Пройдите предыдущий уровень, чтобы открыть этот.");
                    }
                }
            });
        }

        function doDrag(e) {
            if (!dragInfo) return;
            e.preventDefault();
            
            const pos = getMousePos(e);
            dragInfo.currentX = pos.x;
            dragInfo.currentY = pos.y;
        }

        function endDrag(e) {
            if (!dragInfo || gameState !== 'PLAYING' || isProcessing) return;
            e.preventDefault();

            const { startR, startC, startX, startY, currentX, currentY } = dragInfo;

            const deltaX = currentX - startX;
            const deltaY = currentY - startY;

            let endR = startR;
            let endC = startC;

            if (Math.abs(deltaX) > Math.abs(deltaY) && Math.abs(deltaX) > SWAP_THRESHOLD) {
                endC += (deltaX > 0 ? 1 : -1);
            } else if (Math.abs(deltaY) > Math.abs(deltaX) && Math.abs(deltaY) > SWAP_THRESHOLD) {
                endR += (deltaY > 0 ? 1 : -1);
            }

            if (endR >= 0 && endR < GRID_SIZE && endC >= 0 && endC < GRID_SIZE) {
                const targetCell = { row: endR, col: endC };
                const startCell = { row: startR, col: startC };

                if (isAdjacent(startCell, targetCell)) {
                    attemptSwap(startCell, targetCell);
                }
            }
            
            dragInfo = null; 
        }

        function isAdjacent(c1, c2) {
            const rowDiff = Math.abs(c1.row - c2.row);
            const colDiff = Math.abs(c1.col - c2.col);
            return (rowDiff === 1 && colDiff === 0) || (rowDiff === 0 && colDiff === 1);
        }

        // --- ФУНКЦИЯ ОБМЕНА: БЛОК А (Гарантированный откат) ---
        function attemptSwap(c1, c2) {
            const crystal1 = grid[c1.row][c1.col];
            const crystal2 = grid[c2.row][c2.col];

            if (!crystal1 || !crystal2) return;
            isProcessing = true; 

            // Сохраняем исходные координаты для отката
            const startR1 = crystal1.r; const startC1 = crystal1.c;
            const startR2 = crystal2.r; const startC2 = crystal2.c;

            // 1. Устанавливаем целевые координаты для анимации обмена (targetR/targetC)
            crystal1.targetR = c2.row; crystal1.targetC = c2.col;
            crystal2.targetR = c1.row; crystal2.targetC = c1.col;

            // 2. Временно меняем ссылки в сетке для ПРОВЕРКИ совпадения
            [grid[c1.row][c1.col], grid[c2.row][c2.col]] = [crystal2, crystal1];
            
            const matches = findAllMatches();
            
            if (matches.length > 0) {
                // Успешный обмен: Ссылки остаются на новых местах в grid. 
                handleMatches(matches);
            } else {
                // БЛОК A: НЕУДАЧНЫЙ ОБМЕН (ОТКАТ)
                
                // 3. Возвращаем ссылки в сетке на исходные места НЕМЕДЛЕННО
                [grid[c1.row][c1.col], grid[c2.row][c2.col]] = [crystal1, crystal2];
                
                // 4. Сбрасываем целевые координаты на ИСХОДНЫЕ
                crystal1.targetR = startR1; crystal1.targetC = startC1;
                crystal2.targetR = startR2; crystal2.targetC = startC2;
                
                // 5. Устанавливаем r/c в *промежуточное* положение для начала обратной анимации
                crystal1.r = startR2; crystal1.c = startC2;
                crystal2.r = startR1; crystal2.c = startC1;
                
                // КОНЕЦ БЛОКА A
            }
        }
        
        // --- ЛОГИКА СОВПАДЕНИЙ И ПАДЕНИЯ (Без изменений) ---

        function findAllMatches() {
            const matches = []; 
            const ignoredTypes = [-1]; 
            
            // Горизонтальная проверка
            for (let r = 0; r < GRID_SIZE; r++) {
                for (let c = 0; c <= GRID_SIZE - 3; c++) {
                    const type = grid[r][c]?.type;
                    if (ignoredTypes.includes(type) || !grid[r][c + 1] || !grid[r][c + 2]) continue;

                    if (type === grid[r][c + 1].type && type === grid[r][c + 2].type) {
                        let length = 3;
                        let nextC = c + 3;
                        while (nextC < GRID_SIZE && grid[r][nextC] && grid[r][nextC].type === type) {
                            length++;
                            nextC++;
                        }
                        matches.push({ r, c, length, direction: 'horizontal', type });
                        c = nextC - 1; 
                    }
                }
            }

            // Вертикальная проверка
            for (let c = 0; c < GRID_SIZE; c++) {
                for (let r = 0; r <= GRID_SIZE - 3; r++) {
                    const type = grid[r][c]?.type;
                    if (ignoredTypes.includes(type) || !grid[r + 1][c] || !grid[r + 2][c]) continue;
                    
                    if (type === grid[r + 1][c].type && type === grid[r + 2][c].type) {
                        let length = 3;
                        let nextR = r + 3;
                        while (nextR < GRID_SIZE && grid[nextR][c] && grid[nextR][c].type === type) {
                            length++;
                            nextR++;
                        }
                        matches.push({ r, c, length, direction: 'vertical', type });
                        r = nextR - 1; 
                    }
                }
            }
            return matches;
        }

        function handleMatches(matches) {
            const cellsToClear = new Set();
            
            const energyMatches = matches.filter(m => m.type === 7 && m.length >= 3);
            
            if (energyMatches.length > 0) {
                energyMatches.forEach(match => {
                    for (let i = 0; i < match.length; i++) {
                        let r = match.r + (match.direction === 'vertical' ? i : 0);
                        let c = match.c + (match.direction === 'horizontal' ? i : 0);
                        cellsToClear.add(`${r},${c}`); 
                    }
                    const centerR = match.r + (match.direction === 'vertical' ? 1 : 0);
                    const centerC = match.c + (match.direction === 'horizontal' ? 1 : 0);
                    activateEnergyCrystal({r: centerR, c: centerC}, cellsToClear);
                });
            } else {
                createSpecialCrystals(matches, cellsToClear);
            }
            
            const activatedCells = Array.from(cellsToClear).filter(coord => {
                const [r, c] = coord.split(',').map(Number);
                const type = grid[r][c]?.type;
                return type === 6 || type === 7;
            });

            activatedCells.forEach(coord => {
                const [r, c] = coord.split(',').map(Number);
                const crystal = grid[r][c];
                if (crystal && (crystal.type === 6 || crystal.type === 7)) {
                    activateEnergyCrystal({r, c}, cellsToClear);
                }
            });

            cellsToClear.forEach(coord => {
                const [r, c] = coord.split(',').map(Number);
                const crystal = grid[r][c];
                if (crystal) {
                    crystal.isBooming = true;
                    crystal.targetR = r;
                    crystal.targetC = c;
                    crystal.r = r;
                    crystal.c = c;
                }
            });
            
            score += cellsToClear.size * 10;
            updateHUD();

            setTimeout(() => {
                checkLevelCompletion();
                dropCrystals();
            }, 300); 
        }
        
        function checkLevelCompletion() {
            const current = LEVELS.find(l => l.id === currentLevel);
            if (score >= current.target) {
                alert(`Уровень ${currentLevel} пройден!`);
                const nextLevel = LEVELS.find(l => l.id === currentLevel + 1);
                if (nextLevel) {
                    nextLevel.unlocked = true;
                    gameState = 'MENU';
                    currentLevel++;
                } else {
                    alert("Поздравляем! Вы прошли всю игру!");
                    gameState = 'MENU';
                }
            }
        }

        function createSpecialCrystals(matches, cellsToClear) {
            matches.forEach(match => {
                const type = match.type;
                let typeToCreate = null;

                if (type >= 0 && type < CRYSTAL_TYPES) {
                    typeToCreate = match.length >= 4 ? 6 : 7; 
                } 

                for (let i = 0; i < match.length; i++) {
                     let r = match.r + (match.direction === 'vertical' ? i : 0);
                     let c = match.c + (match.direction === 'horizontal' ? i : 0);
                     cellsToClear.add(`${r},${c}`);
                }

                if (typeToCreate) {
                    const newR = match.r;
                    const newC = match.c;

                    if (grid[newR][newC]?.type !== 6 && grid[newR][newC]?.type !== 7) {
                        grid[newR][newC] = new Crystal(newR, newC, typeToCreate, type); 
                        cellsToClear.delete(`${newR},${newC}`); 
                    }
                }
            });
        }

        function activateEnergyCrystal(activatedCell, cellsToClear) {
            const R = activatedCell.r;
            const C = activatedCell.c;
            
            for (let r = R - 1; r <= R + 1; r++) {
                for (let c = C - 1; c <= C + 1; c++) {
                    if (r >= 0 && r < GRID_SIZE && c >= 0 && c < GRID_SIZE) {
                        cellsToClear.add(`${r},${c}`); 
                    }
                }
            }
        }

        function dropCrystals() {
            for (let c = 0; c < GRID_SIZE; c++) {
                let emptyRow = GRID_SIZE - 1;
                for (let r = GRID_SIZE - 1; r >= 0; r--) {
                    const crystal = grid[r][c];
                    if (crystal) {
                        if (r !== emptyRow) {
                            grid[emptyRow][c] = crystal;
                            grid[r][c] = null; 
                            
                            crystal.targetR = emptyRow; 
                            crystal.targetC = c;
                            
                            crystal.yOffset = (r - emptyRow) * CELL_SIZE; 
                            
                            crystal.r = emptyRow;
                            crystal.c = c;
                        }
                        emptyRow--;
                    }
                }
            }
            
            for (let r = 0; r < GRID_SIZE; r++) {
                for (let c = 0; c < GRID_SIZE; c++) {
                    if (grid[r][c] === null) {
                        const newType = Math.floor(Math.random() * CRYSTAL_TYPES);
                        const newCrystal = new Crystal(r, c, newType, newType);
                        
                        newCrystal.yOffset = -(r + 1) * CELL_SIZE;
                        newCrystal.targetR = r;
                        newCrystal.targetC = c;
                        
                        grid[r][c] = newCrystal;
                    }
                }
            }
            
            setTimeout(() => {
                const newMatches = findAllMatches();
                
                if (newMatches.length > 0) {
                    handleMatches(newMatches);
                } 
            }, 500); 
        }

        function updateHUD() {
            if (scoreValue && targetValue) {
                scoreValue.textContent = score;
                const current = LEVELS.find(l => l.id === currentLevel);
                targetValue.textContent = current ? current.target : '—';
            }
        }

        // --- ЗАПУСК ИГРЫ ---
        
        document.addEventListener('DOMContentLoaded', startGame); 
    </script>
</body>
</html>
