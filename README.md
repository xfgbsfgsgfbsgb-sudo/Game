<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Crystal Swap v14.0: Зимняя Машина</title>
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
            /* Важно для мобильных устройств, чтобы предотвратить масштабирование при перетаскивании */
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
            margin-top: 60px; 
        }

        /* HUD */
        #hud-container {
            position: absolute;
            top: -50px; 
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

        /* Панель паузы и проигрыша */
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
        .pause-button, .menu-button, .restart-button {
            padding: 10px 20px;
            margin: 10px;
            font-size: 24px;
            cursor: pointer;
            border: 2px solid #f1c40f;
            background: #2ecc71;
            color: white;
            border-radius: 8px;
        }
        .restart-button {
             background: #e74c3c;
        }
    </style>
</head>
<body>
    <div id="game-container">
        <canvas id="gameCanvas" width="500" height="500"></canvas>
        <div id="hud-container" style="display: none;">
            🏃 Ходы: <span id="moves-left">0</span>
            🎯 Цель: 📦 <span id="boxes-remaining">0</span>
            ✨ Очки: <span id="score-value">0</span>
            <button id="pause-button">⏸️</button>
        </div>

        <div id="pause-overlay" style="display: none;">
            </div>
    </div>

    <script>
        // ГЛОБАЛЬНЫЕ ПЕРЕМЕННЫЕ
        let canvas, ctx, scoreValue, pauseOverlay, pauseButton, movesLeftValue, boxesRemainingValue;

        // --- КОНФИГУРАЦИЯ ИГРЫ ---
        const GRID_SIZE = 8;
        const CANVAS_SIZE = 500;
        const CELL_SIZE = CANVAS_SIZE / GRID_SIZE;
        const CRYSTAL_TYPES = 6; 
        const PROGRESS_KEY = 'crystal_swap_progress'; 
        
        // Цвета
        const COLORS = ['#e74c3c', '#3498db', '#2ecc71', '#f1c40f', '#9b59b6', '#34495e'];
        const ELECTRIC_COLOR = '#ecf0f1'; // Тип 6 (4 в ряд)
        const ENERGY_COLOR = '#ff00ff';  // Тип 7 (5+ в ряд)
        const BOOM_COLOR = '#ffff00'; 
        const BOX_COLOR = '#8e44ad'; // Тип -1
        const BACKGROUND_COLOR = '#1a1a1a';

        // Состояния игры
        let gameState = 'MENU';
        
        let grid = []; 
        let currentLevel = 1;
        let score = 0;
        let movesLeft = 0;
        let boxesRemaining = 0;
        let isProcessing = false; 

        // --- ДАННЫЕ УРОВНЕЙ (15 УРОВНЕЙ) ---
        let LEVELS = [
            { id: 1, moves: 25, color: '#2ecc71', unlocked: true, 
              boxMap: [
                "........",
                ".x....x.",
                "........",
                "........",
                "........",
                "........",
                ".x....x.",
                "........"
              ] 
            },
            { 
              id: 2, 
              moves: 30, 
              color: '#3498db', 
              unlocked: false,
              boxMap: [
                "........", 
                "........",
                "x...x...", 
                ".x......",
                "........",
                "........",
                "....x...",
                "x...x..."
              ] 
            },
            { id: 3, moves: 35, color: '#f1c40f', unlocked: false,
              boxMap: [
                "x.x.x.x.",
                ".x.x.x.x",
                "........",
                "........",
                "........",
                "........",
                ".x.x.x.x",
                "x.x.x.x."
              ]
            },
            { id: 4, moves: 40, color: '#e74c3c', unlocked: false,
              boxMap: [
                "x.......",
                ".x......",
                "..x.....",
                "...x....",
                "....x...",
                ".....x..",
                "......x.",
                ".......x"
              ]
            },
            // Уровень 5: Сетка в форме Машины!
            { id: 5, moves: 40, color: '#9b59b6', unlocked: false,
              boxMap: [
                "........",
                ".xxxxx..",
                "x......x",
                "x......x",
                "x......x",
                "x......x",
                "x.x..x.x", // Колеса
                ".x....x."  // Колеса
              ]
            },
            // ЗИМНИЕ УРОВНИ
            { id: 6, moves: 45, color: '#00bcd4', unlocked: false,
              boxMap: [
                "x.x.x.x.",
                "........",
                "x.x.x.x.",
                "........",
                "x.x.x.x.",
                "........",
                "x.x.x.x.",
                "........"
              ]
            },
            { id: 7, moves: 45, color: '#1abc9c', unlocked: false,
              boxMap: [
                "..x..x..",
                "..x..x..",
                "x......x",
                ".x....x.",
                ".x....x.",
                "x......x",
                "..x..x..",
                "..x..x.."
              ]
            },
            { id: 8, moves: 50, color: '#f39c12', unlocked: false,
              boxMap: [
                "x.x.x.x.",
                ".x.x.x.x",
                "x.x.x.x.",
                ".x.x.x.x",
                "x.x.x.x.",
                ".x.x.x.x",
                "x.x.x.x.",
                ".x.x.x.x"
              ]
            },
            { id: 9, moves: 50, color: '#8e44ad', unlocked: false,
              boxMap: [
                "x.......",
                "x.......",
                "x.x.x.x.",
                ".x.x.x.x",
                "x.x.x.x.",
                ".x.x.x.x",
                ".......x",
                ".......x"
              ]
            },
            { id: 10, moves: 60, color: '#c0392b', unlocked: false,
              boxMap: [
                "x.x.x.x.",
                ".x.x.x.x",
                "x.x.x.x.",
                ".x.x.x.x",
                "x.x.x.x.",
                ".x.x.x.x",
                "x.x.x.x.",
                ".x.x.x.x"
              ] 
            },
            // ДОПОЛНИТЕЛЬНЫЕ УРОВНИ
            { id: 11, moves: 65, color: '#3498db', unlocked: false,
              boxMap: [
                "x...x.x.",
                ".x.x.x.x",
                "x.x.x.x.",
                "........",
                "........",
                ".x.x.x.x",
                "x.x.x.x.",
                ".x...x.x"
              ] 
            },
            { id: 12, moves: 70, color: '#2ecc71', unlocked: false,
              boxMap: [
                "x.x.x.x.x.x",
                ".x.x.x.x.x.",
                "x.x.x.x.x.x",
                "x.x.x.x.x.x",
                "x.x.x.x.x.x",
                "x.x.x.x.x.x",
                "x.x.x.x.x.x",
                "x.x.x.x.x.x"
              ] 
            },
            { id: 13, moves: 75, color: '#9b59b6', unlocked: false,
              boxMap: [
                "........",
                ".xxxxxx.",
                ".x....x.",
                ".x.xx.x.",
                ".x.xx.x.",
                ".x....x.",
                ".xxxxxx.",
                "........"
              ] 
            },
            { id: 14, moves: 80, color: '#e74c3c', unlocked: false,
              boxMap: [
                "x.......",
                ".x......",
                "..x.....",
                "...x....",
                "....x...",
                ".....x..",
                "......x.",
                ".......x"
              ] 
            },
            { id: 15, moves: 85, color: '#f1c40f', unlocked: false,
              boxMap: [
                "xxxxxxxx",
                "x......x",
                "x......x",
                "x......x",
                "x......x",
                "x......x",
                "x......x",
                "xxxxxxxx"
              ] 
            }
        ];
        
        // --- КЛАСС КРИСТАЛЛА/ЯЧЕЙКИ ---
        class Crystal {
            constructor(r, c, type, originalType, isBox = false) {
                this.r = r; 
                this.c = c; 
                this.type = type; 
                this.originalType = originalType; 
                this.yOffset = 0; 
                this.rotation = 0; 
                this.scale = 1; 
                this.isBooming = false;
                this.isBox = isBox;
                
                this.targetR = r; 
                this.targetC = c; 
            }
        }

        // --- DRAG AND DROP ПЕРЕМЕННЫЕ ---
        let dragInfo = null; 
        const SWAP_THRESHOLD = CELL_SIZE * 0.4; 
        let animationFrame = null;
        
        // --- ЛОГИКА СОХРАНЕНИЯ/ЗАГРУЗКИ ---

        function saveProgress() {
            try {
                const saveState = LEVELS.map(level => ({
                    id: level.id,
                    unlocked: level.unlocked
                }));
                localStorage.setItem(PROGRESS_KEY, JSON.stringify(saveState));
            } catch (e) {
                console.error("Не удалось сохранить прогресс:", e);
            }
        }

        function loadProgress() {
            try {
                const savedData = localStorage.getItem(PROGRESS_KEY);
                if (savedData) {
                    const loadedLevels = JSON.parse(savedData);
                    
                    // Обновление старых данных новыми уровнями
                    LEVELS = LEVELS.map(defaultLevel => {
                        const loaded = loadedLevels.find(l => l.id === defaultLevel.id);
                        if (loaded) {
                            return {
                                ...defaultLevel,
                                unlocked: loaded.unlocked
                            };
                        }
                        return defaultLevel;
                    });
                    
                    // Ищем максимально разблокированный уровень
                    currentLevel = LEVELS.filter(l => l.unlocked).pop()?.id || 1;
                }
            } catch (e) {
                console.error("Не удалось загрузить прогресс:", e);
            }
        }


        // --- ИНИЦИАЛИЗАЦИЯ И ЗАПУСК УРОВНЯ ---

        function initLevel(levelId) {
            const levelData = LEVELS.find(l => l.id === levelId);
            if (!levelData || !levelData.unlocked) return;
            
            currentLevel = levelId;
            score = 0;
            isProcessing = false;
            dragInfo = null;
            boxesRemaining = 0;
            
            movesLeft = levelData.moves;
            
            grid = [];
            for (let r = 0; r < GRID_SIZE; r++) {
                grid[r] = [];
                for (let c = 0; c < GRID_SIZE; c++) {
                    const isBox = levelData.boxMap[r][c] === 'x';
                    let type;

                    if (isBox) {
                        type = -1; 
                        grid[r][c] = new Crystal(r, c, type, type, true);
                        boxesRemaining++;
                    } else {
                        // Убеждаемся, что на старте нет совпадений
                        do {
                            type = Math.floor(Math.random() * CRYSTAL_TYPES);
                        } while (checkMatchAtStart(r, c, type));
                        grid[r][c] = new Crystal(r, c, type, type, false);
                    }
                    grid[r][c].yOffset = -r * CELL_SIZE; 
                }
            }
            
            updateHUD();
            gameState = 'PLAYING';
            pauseOverlay.style.display = 'none';
            document.getElementById('hud-container').style.display = 'flex'; 
            if (!animationFrame) {
                animationFrame = requestAnimationFrame(drawGame);
            }
        }
        
        function checkMatchAtStart(r, c, type) {
            if (c >= 2 && grid[r][c - 1] && !grid[r][c - 1].isBox && grid[r][c - 1].type === type && grid[r][c - 2] && !grid[r][c - 2].isBox && grid[r][c - 2].type === type) return true;
            if (r >= 2 && grid[r - 1][c] && !grid[r - 1][c].isBox && grid[r - 1][c].type === type && grid[r - 2][c] && !grid[r - 2][c].isBox && grid[r - 2][c].type === type) return true;
            return false;
        }


        // --- ФУНКЦИИ УПРАВЛЕНИЯ ИГРОЙ ---

        function startGame() {
            canvas = document.getElementById('gameCanvas');
            ctx = canvas.getContext('2d');
            scoreValue = document.getElementById('score-value');
            movesLeftValue = document.getElementById('moves-left');
            boxesRemainingValue = document.getElementById('boxes-remaining');
            pauseOverlay = document.getElementById('pause-overlay');
            pauseButton = document.getElementById('pause-button');
            
            setupEventListeners();
            
            loadProgress(); 
            
            gameState = 'MENU'; 

            if (!animationFrame) {
                animationFrame = requestAnimationFrame(drawGame);
            }

            window.onbeforeunload = function() {
                saveProgress(); 
            };
        }

        function togglePause() {
            if (gameState === 'PLAYING') {
                gameState = 'PAUSED';
                showPauseMenu();
            } else if (gameState === 'PAUSED') {
                gameState = 'PLAYING';
                pauseOverlay.style.display = 'none';
            }
        }
        
        function showPauseMenu() {
            pauseOverlay.innerHTML = `
                <h2>Пауза</h2>
                <button class="pause-button" onclick="togglePause()">Продолжить</button>
                <button class="menu-button" onclick="goToMenu(true)">В меню</button>
            `;
            pauseOverlay.style.display = 'flex';
        }
        
        function showGameOverMenu() {
             gameState = 'GAME_OVER';
             pauseOverlay.innerHTML = `
                <h2>GAME OVER</h2>
                <p>Ходы закончились! Вы не убрали все коробки. Осталось: ${boxesRemaining}</p>
                <button class="restart-button" onclick="initLevel(${currentLevel})">Начать Снова</button>
                <button class="menu-button" onclick="goToMenu(false)">В Меню</button>
            `;
            pauseOverlay.style.display = 'flex';
            document.getElementById('hud-container').style.display = 'none';
        }

        function goToMenu(confirmNeeded) {
            if (confirmNeeded && gameState === 'PLAYING' && !confirm("Вы уверены? Текущая игра будет сброшена.")) {
                return;
            }
            
            gameState = 'MENU';
            pauseOverlay.style.display = 'none';
            document.getElementById('hud-container').style.display = 'none'; 
            
            score = 0;
            grid = [];
            updateHUD(); 
        }

        // --- ОТРИСОВКА ИГРЫ (ОСНОВНОЙ ЦИКЛ) ---

        function drawGame() {
            if (!ctx) return; 
            
            ctx.clearRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);

            if (gameState === 'MENU') {
                drawMap();
            } else if (gameState === 'PLAYING' || gameState === 'PAUSED' || gameState === 'GAME_OVER') {
                ctx.fillStyle = BACKGROUND_COLOR; 
                ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);

                if (gameState === 'PLAYING') {
                    updateAnimations();
                }

                for (let r = 0; r < GRID_SIZE; r++) {
                    for (let c = 0; c < GRID_SIZE; c++) {
                        const crystal = grid[r][c];
                        const x = c * CELL_SIZE;
                        const y = r * CELL_SIZE;

                        
                        if (crystal && crystal.isBox && !crystal.isBooming) { 
                            drawBox(x, y);
                            continue;
                        }
                        
                        if (crystal) {
                            // Использование crystal.r и crystal.c для анимации сдвига/отката
                            const drawX = crystal.c * CELL_SIZE + (crystal.targetC - crystal.c) * CELL_SIZE;
                            const drawY = crystal.r * CELL_SIZE + (crystal.targetR - crystal.r) * CELL_SIZE + crystal.yOffset;
    
                            if (dragInfo && dragInfo.startR === r && dragInfo.startC === c) {
                                 // Отрисовка перетаскиваемого кристалла отдельно
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
                }

                if (dragInfo) {
                    ctx.strokeStyle = '#ecf0f1'; 
                    ctx.lineWidth = 4;
                    ctx.strokeRect(dragInfo.startC * CELL_SIZE + 2, dragInfo.startR * CELL_SIZE + 2, CELL_SIZE - 4, CELL_SIZE - 4);
                }
            }

            animationFrame = requestAnimationFrame(drawGame);
        }

        // --- ФУНКЦИИ ОТРИСОВКИ КРИСТАЛЛОВ И КОРОБОК ---

        function drawBox(x, y) {
            ctx.fillStyle = BOX_COLOR;
            ctx.strokeStyle = '#2c3e50';
            ctx.lineWidth = 4;
            ctx.fillRect(x + 5, y + 5, CELL_SIZE - 10, CELL_SIZE - 10);
            ctx.strokeRect(x + 5, y + 5, CELL_SIZE - 10, CELL_SIZE - 10);
            
            ctx.strokeStyle = '#ecf0f1';
            ctx.lineWidth = 3;
            ctx.beginPath();
            ctx.moveTo(x + 10, y + 10);
            ctx.lineTo(x + CELL_SIZE - 10, y + CELL_SIZE - 10);
            ctx.moveTo(x + CELL_SIZE - 10, y + 10);
            ctx.lineTo(x + 10, y + CELL_SIZE - 10);
            ctx.stroke();
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
            } else if (type === -1) { 
                if (isBooming) {
                    color = BOOM_COLOR;
                } else {
                    return; 
                }
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

        function drawMap() {
            // ЗИМНЕЕ МЕНЮ
            ctx.fillStyle = '#3498db'; 
            ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);
            
            ctx.fillStyle = '#ecf0f1';
            ctx.font = '36px Arial';
            ctx.textAlign = 'center';
            ctx.fillText('❄️ Карта Уровней', CANVAS_SIZE / 2, 50);
            
            // 15 позиций уровней (зигзагом)
            const positions = [
                { x: 100, y: 100 }, { x: 250, y: 100 }, { x: 400, y: 100 }, // Ряд 1
                { x: 400, y: 170 }, { x: 250, y: 170 }, { x: 100, y: 170 }, // Ряд 2
                { x: 100, y: 240 }, { x: 250, y: 240 }, { x: 400, y: 240 }, // Ряд 3
                { x: 400, y: 310 }, { x: 250, y: 310 }, { x: 100, y: 310 }, // Ряд 4
                { x: 100, y: 380 }, { x: 250, y: 380 }, { x: 400, y: 380 }  // Ряд 5
            ];
            
            const levelCount = Math.min(LEVELS.length, positions.length);

            for(let index = 0; index < levelCount; index++) {
                const level = LEVELS[index];
                const pos = positions[index];
                const radius = 22;
                const isUnlocked = level.unlocked;
                const isCurrent = level.id === currentLevel && gameState !== 'PLAYING';

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

                if (isCurrent) {
                    ctx.strokeStyle = '#f1c40f';
                    ctx.lineWidth = 5;
                    ctx.stroke();
                }

                ctx.fillStyle = 'white';
                ctx.font = 'bold 16px Arial';
                ctx.textAlign = 'center';
                ctx.fillText(level.id, pos.x, pos.y + 5);
            }
        }


        // --- ФУНКЦИЯ АНИМАЦИИ ---

        function updateAnimations() {
            if (gameState !== 'PLAYING') return;

            const speed = CELL_SIZE / 8; 
            let processingComplete = true;

            for (let r = 0; r < GRID_SIZE; r++) {
                for (let c = 0; c < GRID_SIZE; c++) {
                    const crystal = grid[r][c];
                    if (!crystal) continue;

                    // 1. Анимация отката/сдвига
                    if (crystal.r !== crystal.targetR || crystal.c !== crystal.targetC) {
                        processingComplete = false;
                        
                        const startR = Math.round(crystal.r); 
                        const startC = Math.round(crystal.c);

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

                        if (Math.abs(crystal.targetR - crystal.r) < speed) {
                            crystal.r = crystal.targetR;
                        }
                        if (Math.abs(crystal.targetC - crystal.c) < speed) {
                            crystal.c = crystal.targetC;
                        }

                        if (crystal.r === crystal.targetR && crystal.c === crystal.targetC) {
                            
                            if (startR !== crystal.targetR || startC !== crystal.targetC) {
                                
                                if (startR >= 0 && startR < GRID_SIZE && startC >= 0 && startC < GRID_SIZE && grid[startR][startC] === crystal) {
                                    grid[startR][startC] = null; 
                                }
                                
                                grid[crystal.targetR][crystal.targetC] = crystal; 
                            }
                        }
                    }

                    // 2. Анимация падения
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

                    // 3. Анимация исчезновения
                    if (crystal.isBooming) {
                        processingComplete = false;
                        crystal.scale -= 0.1;
                        crystal.rotation += Math.PI / 10;
                        if (crystal.scale <= 0) {
                            if (grid[r] && grid[r][c] === crystal) {
                                grid[r][c] = null;
                                if (crystal.isBox) {
                                    boxesRemaining--; // Уменьшаем счетчик только после исчезновения
                                    updateHUD(); 
                                    checkWinCondition(); 
                                }
                            }
                        }
                    }
                }
            }
            
            if (processingComplete) {
                 isProcessing = false;
                 // Проверяем условие Game Over только после завершения всех анимаций и падений
                 if (gameState === 'PLAYING' && movesLeft <= 0 && boxesRemaining > 0) {
                     checkGameOver();
                 }
            } else {
                 isProcessing = true;
            }
        }


        // --- ЛОГИКА DRAG & DROP и ОБМЕНА ---

        function setupEventListeners() {
            if (!canvas || !pauseButton) return; 

            // Мышь
            canvas.addEventListener('mousedown', handleClickOrDrag);
            window.addEventListener('mousemove', doDrag);
            window.addEventListener('mouseup', endDrag);

            // Тач (мобильные)
            canvas.addEventListener('touchstart', handleClickOrDrag);
            window.addEventListener('touchmove', doDrag);
            window.addEventListener('touchend', endDrag);

            pauseButton.onclick = togglePause;
        }

        function getMousePos(e) {
            const rect = canvas.getBoundingClientRect();
            // Получаем координаты для мыши или первого касания
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
            if (gameState === 'PAUSED' || isProcessing || gameState === 'GAME_OVER') return; 

            const pos = getMousePos(e);
            
            if (gameState === 'MENU') {
                handleMenuClick(pos);
                return;
            }

            if (pos.row >= 0 && pos.row < GRID_SIZE && pos.col >= 0 && pos.col < GRID_SIZE && grid[pos.row] && grid[pos.row][pos.col] !== null) {
                 if (grid[pos.row][pos.col].isBox) return; 

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
            // 15 позиций уровней (зигзагом)
            const positions = [
                { x: 100, y: 100 }, { x: 250, y: 100 }, { x: 400, y: 100 }, 
                { x: 400, y: 170 }, { x: 250, y: 170 }, { x: 100, y: 170 }, 
                { x: 100, y: 240 }, { x: 250, y: 240 }, { x: 400, y: 240 }, 
                { x: 400, y: 310 }, { x: 250, y: 310 }, { x: 100, y: 310 }, 
                { x: 100, y: 380 }, { x: 250, y: 380 }, { x: 400, y: 380 }  
            ];
            
            LEVELS.forEach((level, index) => {
                const mapPos = positions[index];
                const radius = 22;
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

            // Определяем направление итогового сдвига
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

        function attemptSwap(c1, c2) {
            const crystal1 = grid[c1.row][c1.col];
            const crystal2 = grid[c2.row][c2.col];

            // Нельзя двигать, если одна из клеток - коробка
            if (crystal1.isBox || crystal2.isBox) return;

            if (!crystal1 || !crystal2) return;
            isProcessing = true; 

            const startR1 = crystal1.r; const startC1 = crystal1.c;
            const startR2 = crystal2.r; const startC2 = crystal2.c;

            // Виртуальный обмен для проверки
            crystal1.targetR = c2.row; crystal1.targetC = c2.col;
            crystal2.targetR = c1.row; crystal2.targetC = c1.col;

            [grid[c1.row][c1.col], grid[c2.row][c2.col]] = [crystal2, crystal1];
            
            const matches = findAllMatches();
            
            if (matches.length > 0) {
                movesLeft--; 
                updateHUD();
                handleMatches(matches);
            } else {
                // Если нет совпадений, отменяем ход и запускаем анимацию отката
                movesLeft--; 
                updateHUD();
                
                // Восстанавливаем позиции в массиве
                [grid[c1.row][c1.col], grid[c2.row][c2.col]] = [crystal1, crystal2];
                
                // Устанавливаем целевые позиции обратно для анимации отката
                crystal1.targetR = startR1; crystal1.targetC = startC1;
                crystal2.targetR = startR2; crystal2.targetC = startC2;
                
                // Устанавливаем текущие позиции для начала анимации отката
                crystal1.r = startR2; crystal1.c = startC2;
                crystal2.r = startR1; crystal2.c = startC1;
            }
        }
        
        // --- ЛОГИКА СОВПАДЕНИЙ И УДАЛЕНИЯ ---

        function findAllMatches() {
            const matches = []; 
            const ignoredTypes = [-1]; 
            
            // Горизонтальная проверка
            for (let r = 0; r < GRID_SIZE; r++) {
                for (let c = 0; c <= GRID_SIZE - 3; c++) {
                    const type = grid[r][c]?.type;
                    if (ignoredTypes.includes(type) || !grid[r][c + 1] || !grid[r][c + 2]) continue;

                    if (grid[r][c + 1] && grid[r][c + 2] && type === grid[r][c + 1].type && type === grid[r][c + 2].type) {
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
                    
                    if (grid[r + 1][c] && grid[r + 2][c] && type === grid[r + 1][c].type && type === grid[r + 2][c].type) {
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
            
            createSpecialCrystals(matches, cellsToClear);
            
            const matchedCoords = Array.from(cellsToClear).map(coord => coord.split(',').map(Number));

            // Логика уничтожения соседних коробок (строго 4 соседа)
            matchedCoords.forEach(([R, C]) => {
                const neighbors = [[R - 1, C], [R + 1, C], [R, C - 1], [R, C + 1]];
                
                neighbors.forEach(([r, c]) => {
                    if (r >= 0 && r < GRID_SIZE && c >= 0 && c < GRID_SIZE) {
                        const neighbor = grid[r][c];
                        if (neighbor && neighbor.isBox && !neighbor.isBooming) { 
                            cellsToClear.add(`${r},${c}`); 
                        }
                    }
                });
            });


            cellsToClear.forEach(coord => {
                const [r, c] = coord.split(',').map(Number);
                const crystal = grid[r][c];
                if (crystal) {
                    crystal.isBooming = true;
                    // Убедимся, что координаты для анимации сброшены
                    crystal.targetR = r;
                    crystal.targetC = c;
                    crystal.r = r;
                    crystal.c = c;
                }
            });
            
            score += cellsToClear.size * 10;
            updateHUD();

            setTimeout(() => {
                dropCrystals();
            }, 300); 
        }
        
        // --- ПРОВЕРКИ УСЛОВИЙ ---

        function checkWinCondition() {
            // Условие победы - все коробки уничтожены.
            // isProcessing должно быть false, чтобы удостовериться, что все анимации завершены.
            if (boxesRemaining <= 0 && !isProcessing) {
                alert(`Уровень ${currentLevel} пройден!`);
                const nextLevel = LEVELS.find(l => l.id === currentLevel + 1);
                
                if (nextLevel) {
                    nextLevel.unlocked = true;
                    saveProgress(); 
                    currentLevel++;
                    goToMenu(false);
                } else {
                    alert("Поздравляем! Вы прошли всю игру!");
                    saveProgress(); 
                    goToMenu(false);
                }
            }
        }
        
        function checkGameOver() {
             // Game Over наступает, если ходы закончились И коробки ЕЩЕ остались
             if (movesLeft <= 0 && gameState === 'PLAYING' && !isProcessing && boxesRemaining > 0) {
                 showGameOverMenu();
             }
        }

        // --- ЛОГИКА СОЗДАНИЯ СПЕЦ. КРИСТАЛЛОВ И ПАДЕНИЯ ---

        function createSpecialCrystals(matches, cellsToClear) {
            matches.forEach(match => {
                const type = match.type;
                let typeToCreate = null;

                if (type >= 0 && type < CRYSTAL_TYPES) {
                    // 6 - Электрический (4 в ряд), 7 - Энергетический (5+ в ряд)
                    typeToCreate = match.length >= 5 ? 7 : (match.length === 4 ? 6 : null);
                } 

                for (let i = 0; i < match.length; i++) {
                     let r = match.r + (match.direction === 'vertical' ? i : 0);
                     let c = match.c + (match.direction === 'horizontal' ? i : 0);
                     cellsToClear.add(`${r},${c}`);
                }

                if (typeToCreate) {
                    const newR = match.r;
                    const newC = match.c;

                    // Убеждаемся, что не заменяем уже созданный спец. кристалл
                    if (grid[newR][newC]?.type !== 6 && grid[newR][newC]?.type !== 7) {
                        grid[newR][newC] = new Crystal(newR, newC, typeToCreate, type); 
                        cellsToClear.delete(`${newR},${newC}`); 
                    }
                }
            });
        }

        function dropCrystals() {
            // 1. Сдвиг существующих кристаллов вниз
            for (let c = 0; c < GRID_SIZE; c++) {
                let emptyRow = GRID_SIZE - 1;
                for (let r = GRID_SIZE - 1; r >= 0; r--) {
                    const crystal = grid[r][c];
                    
                    // Кристаллы (НЕ коробки) падают
                    if (crystal && !crystal.isBox) { 
                        if (r !== emptyRow) {
                            grid[emptyRow][c] = crystal;
                            grid[r][c] = null; 
                            
                            crystal.targetR = emptyRow; 
                            crystal.targetC = c;
                            
                            // Расчет смещения для анимации падения
                            crystal.yOffset = (r - emptyRow) * CELL_SIZE; 
                            
                            crystal.r = emptyRow;
                            crystal.c = c;
                        }
                        emptyRow--;
                    } else if (crystal && crystal.isBox) {
                        // Если натыкаемся на коробку, она становится новой "полкой"
                        emptyRow = r - 1;
                    }
                }
            }
            
            // 2. Генерация новых кристаллов СВЕРХУ для заполнения всех пустых мест
            for (let r = 0; r < GRID_SIZE; r++) {
                for (let c = 0; c < GRID_SIZE; c++) {
                    // Если клетка пуста 
                    if (grid[r][c] === null) {
                        
                        const newType = Math.floor(Math.random() * CRYSTAL_TYPES);
                        const newCrystal = new Crystal(r, c, newType, newType);
                        
                        // Задаем начальное смещение над полем
                        newCrystal.yOffset = -(r + 1) * CELL_SIZE; 
                        newCrystal.targetR = r;
                        newCrystal.targetC = c;
                        
                        grid[r][c] = newCrystal;
                    }
                }
            }
            
            // Задержка для проверки новых совпадений
            setTimeout(() => {
                const newMatches = findAllMatches();
                
                if (newMatches.length > 0) {
                    handleMatches(newMatches);
                }
            }, 500); 
        }

        function updateHUD() {
            if (scoreValue && movesLeftValue && boxesRemainingValue) {
                scoreValue.textContent = score;
                movesLeftValue.textContent = movesLeft;
                boxesRemainingValue.textContent = boxesRemaining;
            }
        }

        // --- ЗАПУСК ИГРЫ ---
        
        document.addEventListener('DOMContentLoaded', startGame); 
    </script>
</body>
</html>
