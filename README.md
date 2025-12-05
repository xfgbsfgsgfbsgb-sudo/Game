<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Логическая Аркада: Управление Трафиком (Бесконечность)</title>
    <style>
        body {
            margin: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background-color: #34495e;
            font-family: Arial, sans-serif;
            color: #ecf0f1;
        }
        canvas {
            border: 5px solid #2c3e50;
            background-color: #7f8c8d; /* Цвет асфальта */
        }
        #hud {
            position: absolute;
            top: 10px;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(44, 62, 80, 0.8);
            padding: 10px 20px;
            border-radius: 5px;
            font-size: 18px;
            text-align: center;
            z-index: 10;
        }
        #shop-button {
            background-color: #f39c12; /* Оранжевый */
            color: white;
            padding: 8px 15px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            margin-left: 15px;
            font-weight: bold;
            transition: background-color 0.2s;
        }
        #shop-button:hover {
            background-color: #e67e22;
        }
        #shop-container {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(44, 62, 80, 0.95);
            border: 3px solid #f39c12;
            border-radius: 10px;
            padding: 20px;
            display: none;
            flex-direction: column;
            width: 300px;
            z-index: 30;
        }
        .upgrade-item {
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 10px 0;
            border-bottom: 1px solid #555;
        }
        .buy-button {
            padding: 8px 10px;
            background-color: #2ecc71;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-weight: bold;
        }
        .buy-button:disabled {
            background-color: #e74c3c;
            cursor: not-allowed;
        }
        .close-shop {
            align-self: flex-end;
            background: none;
            border: none;
            color: white;
            font-size: 24px;
            cursor: pointer;
        }

        #game-over-screen {
            display: none; /* Убираем экран окончания, так как игра бесконечная */
        }
    </style>
</head>
<body>
    <canvas id="gameCanvas" width="600" height="600"></canvas>

    <div id="hud">
        Монеты: <span id="hud-money" style="color:#f1c40f;">0</span> |
        Пропущено: <span id="hud-passed" style="color:#2ecc71;">0</span> |
        Аварии: <span id="hud-crashes" style="color:#e74c3c;">0</span>
        <button id="shop-button">🛒 Магазин</button>
    </div>
    
    <div id="shop-container">
        <button class="close-shop">X</button>
        <h3>Магазин Улучшений</h3>
        <div class="upgrade-item" id="upgrade-road">
            <span>Больше дорог (Две полосы!)</span>
            <button class="buy-button" data-cost="50" data-upgrade="road">Купить (50 💰)</button>
        </div>
        <div class="upgrade-item" id="upgrade-speed">
            <span>Ускоренный спавн машин</span>
            <button class="buy-button" data-cost="100" data-upgrade="speed" disabled>Купить (100 💰) - скоро</button>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const shopButton = document.getElementById('shop-button');
        const shopContainer = document.getElementById('shop-container');
        const closeShopButton = document.querySelector('.close-shop');
        const upgradeRoadButton = document.querySelector('.buy-button[data-upgrade="road"]');

        let game;
        
        // --- 🚥 КЛАССЫ СУЩНОСТЕЙ ---

        class Car {
            constructor(road, lane) {
                this.road = road;
                this.lane = lane;
                this.size = 30;
                this.width = road === 'horizontal' ? this.size * 1.5 : this.size;
                this.height = road === 'horizontal' ? this.size : this.size * 1.5;
                this.color = this.getRandomColor();
                this.speed = 1 + Math.random() * 1.5; 
                this.isCrashed = false;
                
                if (road === 'horizontal') {
                    this.x = Math.random() < 0.5 ? -this.width : canvas.width;
                    this.direction = this.x < 0 ? 1 : -1;
                } else {
                    this.y = Math.random() < 0.5 ? -this.height : canvas.height;
                    this.direction = this.y < 0 ? 1 : -1;
                }

                this.setInitialPosition();
            }

            getRandomColor() {
                const colors = ['#e74c3c', '#3498db', '#f1c40f', '#2ecc71', '#9b59b6', '#34495e'];
                return colors[Math.floor(Math.random() * colors.length)];
            }
            
            setInitialPosition() {
                const roadWidth = game.upgrades.roadUpgrade ? 150 : 100;
                const roadStart = (canvas.width - roadWidth) / 2;
                
                if (this.road === 'horizontal') {
                    if (!game.upgrades.roadUpgrade) {
                        this.y = this.direction === 1 ? roadStart + 50 : roadStart + 0;
                    } else {
                        const y_offset = this.lane === 1 ? 50 : 20;
                        this.y = this.direction === 1 ? roadStart + roadWidth - y_offset : roadStart + y_offset - this.height;
                    }

                } else { // vertical
                    if (!game.upgrades.roadUpgrade) {
                        this.x = this.direction === 1 ? roadStart + 0 : roadStart + 50;
                    } else {
                        const x_offset = this.lane === 1 ? 50 : 20;
                        this.x = this.direction === 1 ? roadStart + x_offset - this.width : roadStart + roadWidth - x_offset;
                    }
                }
            }

            getRect() {
                return { x: this.x, y: this.y, width: this.width, height: this.height };
            }

            update(trafficLight) {
                if (this.isCrashed) return;

                const isMoving = trafficLight.isGreen || !this.isApproachingStop(trafficLight);
                
                if (isMoving) {
                    if (this.road === 'horizontal') {
                        this.x += this.speed * this.direction;
                    } else {
                        this.y += this.speed * this.direction;
                    }
                }
            }
            
            isApproachingStop(trafficLight) {
                if (trafficLight.isGreen) return false;
                
                const roadWidth = game.upgrades.roadUpgrade ? 150 : 100;
                const roadStart = (canvas.width - roadWidth) / 2;
                const crossEnd = roadStart + roadWidth;

                if (this.road === 'horizontal') {
                    if (this.direction === 1) {
                        if (this.x > crossEnd) return false; 
                        if (this.x + this.width > roadStart - 5) return true;
                    } else {
                         if (this.x < roadStart) return false;
                         if (this.x < crossEnd + 5) return true; 
                    }
                } else { // Vertical road
                    if (this.direction === 1) {
                         if (this.y > crossEnd) return false; 
                         if (this.y + this.height > roadStart - 5) return true;
                    } else {
                         if (this.y < roadStart) return false;
                         if (this.y < crossEnd + 5) return true; 
                    }
                }
                return false;
            }

            // --- НОВАЯ ФУНКЦИЯ: РИСОВАНИЕ С УЛУЧШЕННОЙ ГРАФИКОЙ ---
            draw() {
                // Основной кузов
                ctx.fillStyle = this.color;
                ctx.fillRect(this.x, this.y, this.width, this.height);
                
                const isHorizontal = this.road === 'horizontal';
                
                // 1. Колеса (Черные круги)
                ctx.fillStyle = '#333';
                if (isHorizontal) {
                    const wheelY = this.y + this.height - 5;
                    ctx.fillRect(this.x + 5, wheelY, 5, 5); // Переднее
                    ctx.fillRect(this.x + this.width - 10, wheelY, 5, 5); // Заднее
                } else {
                    const wheelX = this.x + this.width - 5;
                    ctx.fillRect(wheelX, this.y + 5, 5, 5); // Верхнее
                    ctx.fillRect(wheelX, this.y + this.height - 10, 5, 5); // Нижнее
                }

                // 2. Окна (Светло-голубой)
                ctx.fillStyle = '#ADD8E6';
                if (isHorizontal) {
                    ctx.fillRect(this.x + 5, this.y + 5, this.width - 10, 10);
                } else {
                    ctx.fillRect(this.x + 5, this.y + 5, 10, this.height - 10);
                }

                // 3. Фары (Желтый/Красный)
                if (!this.isCrashed) {
                    const lightColor = this.direction === 1 ? '#FFFF00' : '#FF0000'; // Вперед: желтый, Назад: красный
                    ctx.fillStyle = lightColor;
                    if (isHorizontal) {
                        if (this.direction === 1) ctx.fillRect(this.x + this.width - 5, this.y + 5, 5, 5); // Передние (вправо)
                        else ctx.fillRect(this.x, this.y + 5, 5, 5); // Передние (влево)
                    } else {
                        if (this.direction === 1) ctx.fillRect(this.x + 5, this.y + this.height - 5, 5, 5); // Передние (вниз)
                        else ctx.fillRect(this.x + 5, this.y, 5, 5); // Передние (вверх)
                    }
                }

                // Авария
                if (this.isCrashed) {
                    ctx.fillStyle = 'rgba(255, 0, 0, 0.8)';
                    ctx.font = '20px Arial';
                    ctx.fillText('💥', this.x + 5, this.y + this.height / 2 + 5);
                }
            }
            // --- КОНЕЦ ФУНКЦИИ РИСОВАНИЯ ---
        }

        class TrafficLight {
            constructor(road, x, y) {
                this.road = road;
                this.x = x;
                this.y = y;
                this.size = 20;
                this.isGreen = road === 'vertical' ? true : false;
            }

            draw() {
                ctx.fillStyle = '#333';
                ctx.fillRect(this.x, this.y, this.size, this.size * 3);
                
                const color = this.isGreen ? '#2ecc71' : '#e74c3c';
                const yOffset = this.isGreen ? this.size * 2 : 0;

                ctx.fillStyle = color;
                ctx.beginPath();
                ctx.arc(this.x + this.size / 2, this.y + this.size / 2 + yOffset, this.size / 3, 0, Math.PI * 2);
                ctx.fill();
            }

            toggle() {
                this.isGreen = !this.isGreen;
            }
        }

        // --- 🕹️ ГЛАВНЫЙ КЛАСС ИГРЫ ---

        class Game {
            constructor() {
                this.cars = [];
                this.trafficLights = [
                    new TrafficLight('horizontal', 240, 200),
                    new TrafficLight('vertical', 350, 200)
                ];
                this.money = 0;
                this.passedCount = 0;
                this.crashCount = 0;
                this.isRunning = true;
                this.spawnTimer = 0;
                this.lastToggleTime = Date.now();
                
                this.upgrades = {
                    roadUpgrade: false,
                };
            }
            
            // --- ЛОГИКА МАГАЗИНА ---
            buyUpgrade(upgradeType, cost) {
                if (this.money < cost) return false;
                
                if (upgradeType === 'road' && !this.upgrades.roadUpgrade) {
                    this.money -= cost;
                    this.upgrades.roadUpgrade = true;
                    upgradeRoadButton.disabled = true;
                    upgradeRoadButton.textContent = 'КУПЛЕНО!';
                    this.updateTrafficLightPositions();
                    return true;
                }
                return false;
            }
            
            updateTrafficLightPositions() {
                 const roadWidth = 150;
                 const roadStart = (canvas.width - roadWidth) / 2;
                 this.trafficLights.find(l => l.road === 'horizontal').x = roadStart - 10 - 20;
                 this.trafficLights.find(l => l.road === 'vertical').x = roadStart + roadWidth + 10;
            }
            // --- КОНЕЦ ЛОГИКИ МАГАЗИНА ---

            handleInput(mouseX, mouseY) {
                if (!this.isRunning || shopContainer.style.display === 'flex') return;

                const now = Date.now();
                if (now - this.lastToggleTime < 500) return;

                for (const light of this.trafficLights) {
                    if (mouseX > light.x && mouseX < light.x + light.size &&
                        mouseY > light.y && mouseY < light.y + light.size * 3) {
                        
                        const otherLight = this.trafficLights.find(l => l !== light);
                        if (!light.isGreen && otherLight.isGreen) {
                             otherLight.toggle();
                             light.toggle();
                             this.lastToggleTime = now;
                             return;
                        }
                    }
                }
            }

            spawnCar() {
                const roadType = Math.random() < 0.5 ? 'horizontal' : 'vertical';
                let lane = 1; 

                if (this.upgrades.roadUpgrade) {
                    lane = Math.random() < 0.5 ? 1 : 2; 
                }

                this.cars.push(new Car(roadType, lane));
            }

            checkCollisions() {
                for (let i = 0; i < this.cars.length; i++) {
                    const car1 = this.cars[i];
                    if (car1.isCrashed) continue;

                    for (let j = i + 1; j < this.cars.length; j++) {
                        const car2 = this.cars[j];
                        if (car2.isCrashed) continue;
                        
                        if (car1.road !== car2.road) {
                            if (this.isColliding(car1.getRect(), car2.getRect())) {
                                this.handleCrash(car1, car2);
                                return;
                            }
                        }
                    }
                }
            }

            isColliding(r1, r2) {
                const collisionPadding = 5;
                return r1.x + collisionPadding < r2.x + r2.width - collisionPadding &&
                       r1.x + r1.width - collisionPadding > r2.x + collisionPadding &&
                       r1.y + collisionPadding < r2.y + r2.height - collisionPadding &&
                       r1.y + r1.height - collisionPadding > r2.y + collisionPadding;
            }

            handleCrash(car1, car2) {
                car1.isCrashed = true;
                car2.isCrashed = true;
                this.crashCount++;
                
                this.money = Math.max(0, this.money - 5);
                
                setTimeout(() => {
                    this.cars = this.cars.filter(c => !c.isCrashed);
                }, 1000); 
            }
            
            // Убрана функция checkBonus

            update() {
                if (!this.isRunning || shopContainer.style.display === 'flex') return;

                this.spawnTimer++;
                if (this.spawnTimer > 40) {
                    this.spawnCar();
                    this.spawnTimer = 0;
                }
                
                const lightH = this.trafficLights.find(l => l.road === 'horizontal');
                const lightV = this.trafficLights.find(l => l.road === 'vertical');

                for (let i = this.cars.length - 1; i >= 0; i--) {
                    const car = this.cars[i];
                    const currentLight = car.road === 'horizontal' ? lightH : lightV;
                    car.update(currentLight);

                    if (!car.isCrashed) {
                        // Проверка, проехала ли машина (доход 1 монета)
                        if (car.road === 'horizontal' && ((car.direction === 1 && car.x > canvas.width) || (car.direction === -1 && car.x < -car.width))) {
                            this.passedCount++;
                            this.money += 1;
                            this.cars.splice(i, 1);
                        } else if (car.road === 'vertical' && ((car.direction === 1 && car.y > canvas.height) || (car.direction === -1 && car.y < -car.height))) {
                            this.passedCount++;
                            this.money += 1;
                            this.cars.splice(i, 1);
                        }
                    }
                }
                
                this.checkCollisions();
                this.drawHUD();
                
                // Убрана проверка на окончание игры
            }

            draw() {
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                
                const roadWidth = this.upgrades.roadUpgrade ? 150 : 100;
                const roadStart = (canvas.width - roadWidth) / 2;
                
                // Рисуем перекресток (дороги)
                ctx.fillStyle = '#607d8b';
                ctx.fillRect(roadStart, 0, roadWidth, canvas.height); 
                ctx.fillRect(0, roadStart, canvas.width, roadWidth);  
                
                // Разметка (Центр)
                ctx.strokeStyle = '#f9f9f9';
                ctx.lineWidth = 4;
                ctx.setLineDash([10, 10]);
                ctx.strokeRect(roadStart, roadStart, roadWidth, roadWidth);
                ctx.setLineDash([]); 

                // Рисуем сущности
                this.trafficLights.forEach(light => light.draw());
                this.cars.forEach(car => car.draw());
            }

            drawHUD() {
                document.getElementById('hud-money').textContent = this.money;
                document.getElementById('hud-passed').textContent = this.passedCount;
                document.getElementById('hud-crashes').textContent = this.crashCount;
                
                if (!this.upgrades.roadUpgrade) {
                    upgradeRoadButton.disabled = this.money < parseInt(upgradeRoadButton.dataset.cost);
                }
            }
            
            // Убрана функция endGame
            
            reset() {
                game = new Game();
                game.money = 10;
                gameLoop();
            }
        }

        // --- 🎧 ЦИКЛ ИГРЫ И УПРАВЛЕНИЕ ВВОДОМ ---

        function gameLoop() {
            if (game.isRunning) {
                game.update();
                game.draw();
                requestAnimationFrame(gameLoop);
            }
        }

        function handleKeyDown(e) {
             if (e.key.toLowerCase() === 'r' && game && !game.isRunning) {
                 game.reset();
             }
        }
        
        document.addEventListener('keydown', handleKeyDown);

        // Открытие/закрытие магазина
        shopButton.addEventListener('click', () => {
            shopContainer.style.display = 'flex';
        });
        closeShopButton.addEventListener('click', () => {
            shopContainer.style.display = 'none';
        });

        // Покупка улучшений
        upgradeRoadButton.addEventListener('click', (e) => {
            const cost = parseInt(e.target.dataset.cost);
            const upgradeType = e.target.dataset.upgrade;
            
            if (game.buyUpgrade(upgradeType, cost)) {
                game.cars.forEach(car => car.setInitialPosition());
                alert(`Куплено улучшение: ${upgradeType}! Дороги расширены!`);
                shopContainer.style.display = 'none';
            }
        });


        canvas.addEventListener('mousedown', (e) => {
            const rect = canvas.getBoundingClientRect();
            const mouseX = e.clientX - rect.left;
            const mouseY = e.clientY - rect.top;
            game.handleInput(mouseX, mouseY);
        });


        // --- ЗАПУСК ---
        window.onload = () => {
            game = new Game();
            game.money = 10; 
            game.spawnCar();
            gameLoop();
            
            game.updateTrafficLightPositions();
        };

    </script>
</body>
</html>
