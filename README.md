<!DOCTYPE html>
<html lang="uk">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, orientation=landscape">
    <title>Car Container Tycoon 3D</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; user-select: none; -webkit-user-select: none; }
        body, html { width: 100%; height: 100%; overflow: hidden; background: #000; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
        
        /* Landscape enforce hint */
        @media screen and (orientation: portrait) {
            #portrait-warning { display: flex !important; }
        }
        #portrait-warning {
            display: none; position: fixed; top:0; left:0; width:100vw; height:100vh; background:#111; color:#fff;
            z-index: 9999; justify-content: center; align-items: center; text-align: center; font-size: 20px; padding: 20px;
        }

        #canvas-container { width: 100vw; height: 100vh; display: block; touch-action: none; }

        /* UI overlay */
        .hud { position: absolute; top: 15px; left: 15px; z-index: 100; color: #fff; text-shadow: 0 2px 4px rgba(0,0,0,0.8); }
        .hud-balance { font-size: 24px; font-weight: bold; background: rgba(0,0,0,0.6); padding: 8px 16px; border-radius: 12px; border: 1px solid #444; }

        .btn-menu { position: absolute; top: 15px; right: 15px; z-index: 100; padding: 10px 20px; background: #28a745; color: white; border: none; border-radius: 10px; font-weight: bold; font-size: 16px; cursor: pointer; box-shadow: 0 4px 6px rgba(0,0,0,0.3); }

        /* Garage Modal */
        #garage-modal {
            display: none; position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%);
            width: 80vw; height: 80vh; background: rgba(20, 20, 20, 0.95); border-radius: 20px; border: 2px solid #555;
            z-index: 200; padding: 20px; color: white; box-shadow: 0 10px 30px rgba(0,0,0,0.8); flex-direction: column;
        }
        .garage-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; border-bottom: 1px solid #444; padding-bottom: 10px; }
        .garage-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(180px, 1fr)); gap: 15px; overflow-y: auto; flex: 1; padding-right: 5px; }
        .car-card { background: #2a2a2a; border-radius: 12px; padding: 12px; text-align: center; border: 1px solid #444; display: flex; flex-direction: column; align-items: center; justify-content: space-between; }
        .car-icon { width: 100%; height: 90px; background: #333; border-radius: 8px; display: flex; justify-content: center; align-items: center; margin-bottom: 10px; font-size: 40px; }
        .btn-action { padding: 8px 12px; background: #007bff; color: white; border: none; border-radius: 6px; cursor: pointer; margin-top: 5px; width: 100%; font-weight: bold; }
        .btn-sell { background: #dc3545; }
        .btn-close { background: #6c757d; padding: 8px 16px; border-radius: 8px; border: none; color: white; cursor: pointer; }

        /* Controls */
        .controls-container { position: absolute; bottom: 20px; left: 20px; z-index: 100; display: flex; gap: 15px; }
        .btn-drive { width: 65px; height: 65px; background: rgba(255,255,255,0.2); border: 2px solid #fff; border-radius: 50%; color: white; font-size: 24px; display: flex; justify-content: center; align-items: center; }
        .btn-drive:active { background: rgba(255,255,255,0.5); }
        #exit-car-btn { display: none; position: absolute; bottom: 20px; right: 20px; z-index: 100; padding: 15px 25px; background: #ffc107; color: black; font-weight: bold; border-radius: 12px; border: none; }
    </style>
    <!-- Three.js Library -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
</head>
<body>

    <div id="portrait-warning">🔄 Будь ласка, поверніть телефон у горизонтальне положення!</div>

    <div class="hud">
        <div class="hud-balance">Баланс: <span id="balance-display">150,000</span> ₴</div>
    </div>

    <button class="btn-menu" onclick="openGarage()">🏎️ Гараж</button>

    <div id="garage-modal">
        <div class="garage-header">
            <h2>Мої Автомобілі</h2>
            <button class="btn-close" onclick="closeGarage()">Закрити ✖</button>
        </div>
        <div class="garage-grid" id="garage-list"></div>
    </div>

    <button id="exit-car-btn" onclick="exitCar()">Вийти з авто 🚪</button>

    <div id="canvas-container"></div>

    <script>
        // --- ЗВУКОВА СИСТЕМА (Web Audio API) ---
        const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        
        function playSound(type) {
            if (audioCtx.state === 'suspended') audioCtx.resume();
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.connect(gain);
            gain.connect(audioCtx.destination);

            if (type === 'container') {
                // Звук відкриття контейнера (важкий металевий скрип + гул)
                osc.type = 'sawtooth';
                osc.frequency.setValueAtTime(100, audioCtx.currentTime);
                osc.frequency.exponentialRampToValueAtTime(30, audioCtx.currentTime + 0.8);
                gain.gain.setValueAtTime(0.3, audioCtx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.8);
                osc.start();
                osc.stop(audioCtx.currentTime + 0.8);
            } else if (type === 'engine') {
                // Звук мотора
                osc.type = 'triangle';
                osc.frequency.setValueAtTime(80, audioCtx.currentTime);
                osc.frequency.linearRampToValueAtTime(160, audioCtx.currentTime + 0.2);
                gain.gain.setValueAtTime(0.2, audioCtx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.2);
                osc.start();
                osc.stop(audioCtx.currentTime + 0.2);
            }
        }

        // --- ДАНІ ГРИ ---
        let balance = 150000;
        let garage = [];
        let inCar = false;
        let currentDrivenCar = null;

        const CAR_TYPES = [
            { name: "Sedan Standard", price: 40000, color: 0x3366cc, icon: "🚗", speed: 0.3, soundPitch: 80 },
            { name: "Sport Coupe", price: 120000, color: 0xcc0000, icon: "🏎️", speed: 0.5, soundPitch: 120 },
            { name: "Offroad 4x4", price: 250000, color: 0x228b22, icon: "🛻", speed: 0.35, soundPitch: 60 },
            { name: "Hypercar V12", price: 1500000, color: 0xffd700, icon: "⚡", speed: 0.7, soundPitch: 180 }
        ];

        // --- THREE.JS СЦЕНА ---
        const container = document.getElementById('canvas-container');
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87ceeb); // Небо
        scene.fog = new THREE.FogExp2(0x87ceeb, 0.015);

        const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true;
        container.appendChild(renderer.domElement);

        // Освітлення
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
        scene.add(ambientLight);

        const sun = new THREE.DirectionalLight(0xffffff, 0.8);
        sun.position.set(50, 80, 50);
        sun.castShadow = true;
        sun.shadow.mapSize.width = 2048;
        sun.shadow.mapSize.height = 2048;
        scene.add(sun);

        // --- ЛАНДШАФТ (Трава, Дорога, Дерева) ---
        // Земля з травою
        const groundGeo = new THREE.PlaneGeometry(300, 300);
        const groundMat = new THREE.MeshStandardMaterial({ color: 0x4e8d46, roughness: 0.9 });
        const ground = new THREE.Mesh(groundGeo, groundMat);
        ground.rotation.x = -Math.PI / 2;
        ground.receiveShadow = true;
        scene.add(ground);

        // Асфальтова майданчик
        const roadGeo = new THREE.PlaneGeometry(60, 60);
        const roadMat = new THREE.MeshStandardMaterial({ color: 0x333333, roughness: 0.6 });
        const road = new THREE.Mesh(roadGeo, roadMat);
        road.rotation.x = -Math.PI / 2;
        road.position.y = 0.01;
        road.receiveShadow = true;
        scene.add(road);

        // Генерація дерев навколо
        function createTree(x, z) {
            const group = new THREE.Group();
            const trunkMat = new THREE.MeshStandardMaterial({ color: 0x5c4033 });
            const trunk = new THREE.Mesh(new THREE.CylinderGeometry(0.3, 0.5, 3), trunkMat);
            trunk.position.y = 1.5;
            trunk.castShadow = true;
            group.add(trunk);

            const leavesMat = new THREE.MeshStandardMaterial({ color: 0x2e8b57, roughness: 0.6 });
            const leaves = new THREE.Mesh(new THREE.ConeGeometry(2, 5, 5), leavesMat);
            leaves.position.y = 4.5;
            leaves.castShadow = true;
            group.add(leaves);

            group.position.set(x, 0, z);
            scene.add(group);
        }

        for (let i = 0; i < 40; i++) {
            let rx = (Math.random() - 0.5) * 200;
            let rz = (Math.random() - 0.5) * 200;
            if (Math.abs(rx) > 35 || Math.abs(rz) > 35) createTree(rx, rz);
        }

        // --- ГРАВЕЦЬ (ЛЮДИНА) ---
        const playerGroup = new THREE.Group();
        const bodyGeo = new THREE.CylinderGeometry(0.4, 0.4, 1.6);
        const bodyMat = new THREE.MeshStandardMaterial({ color: 0x007bff });
        const playerBody = new THREE.Mesh(bodyGeo, bodyMat);
        playerBody.position.y = 0.8;
        playerBody.castShadow = true;
        playerGroup.add(playerBody);

        const headGeo = new THREE.SphereGeometry(0.3);
        const headMat = new THREE.MeshStandardMaterial({ color: 0xffcc99 });
        const playerHead = new THREE.Mesh(headGeo, headMat);
        playerHead.position.y = 1.8;
        playerGroup.add(playerHead);

        playerGroup.position.set(0, 0, 15);
        scene.add(playerGroup);

        // --- СТВОРЕННЯ КОНТЕЙНЕРІВ С ТЕКСТУРАМИ ---
        const containers = [];
        const containerPositions = [-20, -10, 0, 10, 20];

        containerPositions.forEach((x, i) => {
            const cGroup = new THREE.Group();
            
            // Корпус контейнера
            const boxGeo = new THREE.BoxGeometry(4, 4, 8);
            const boxMat = new THREE.MeshStandardMaterial({ color: 0xd9534f, roughness: 0.4, metalness: 0.3 });
            const box = new THREE.Mesh(boxGeo, boxMat);
            box.position.y = 2;
            box.castShadow = true;
            cGroup.add(box);

            // Двері / Ворота
            const doorGeo = new THREE.BoxGeometry(3.8, 3.8, 0.2);
            const doorMat = new THREE.MeshStandardMaterial({ color: 0xaaaaaa, metalness: 0.8 });
            const door = new THREE.Mesh(doorGeo, doorMat);
            door.position.set(0, 2, 4.01);
            cGroup.add(door);

            cGroup.position.set(x, 0, -15);
            scene.add(cGroup);

            containers.push({ mesh: cGroup, price: (i + 1) * 50000, opened: false });
        });

        // --- ОБЕРТАННЯ КАМЕРИ ТА КЕРУВАННЯ ТАЧЕМ ---
        let isDragging = false;
        let previousTouchX = 0;
        let previousTouchY = 0;
        let cameraYaw = 0;
        let cameraPitch = 0.3;

        window.addEventListener('touchstart', (e) => {
            if (e.touches.length === 1 && e.target.tagName === 'CANVAS') {
                isDragging = true;
                previousTouchX = e.touches[0].clientX;
                previousTouchY = e.touches[0].clientY;
            }
        });

        window.addEventListener('touchmove', (e) => {
            if (isDragging && e.touches.length === 1) {
                const deltaX = e.touches[0].clientX - previousTouchX;
                const deltaY = e.touches[0].clientY - previousTouchY;

                cameraYaw -= deltaX * 0.005;
                cameraPitch = Math.max(0.1, Math.min(Math.PI / 3, cameraPitch + deltaY * 0.005));

                previousTouchX = e.touches[0].clientX;
                previousTouchY = e.touches[0].clientY;
            }
        });

        window.addEventListener('touchend', () => isDragging = false);

        // --- ГОЛОВНИЙ ЦИКЛ ОНОВЛЕННЯ ---
        function animate() {
            requestAnimationFrame(animate);

            // Оновлення камери навколо гравця/авто
            const target = inCar && currentDrivenCar ? currentDrivenCar.position : playerGroup.position;
            const dist = inCar ? 10 : 7;
            
            camera.position.x = target.x + dist * Math.sin(cameraYaw) * Math.cos(cameraPitch);
            camera.position.z = target.z + dist * Math.cos(cameraYaw) * Math.cos(cameraPitch);
            camera.position.y = target.y + dist * Math.sin(cameraPitch);
            camera.lookAt(target.x, target.y + 1, target.z);

            // Перевірка наближення до контейнерів
            containers.forEach(c => {
                if (!c.opened && playerGroup.position.distanceTo(c.mesh.position) < 5) {
                    if (confirm(`Відкрити контейнер за ${c.price.toLocaleString()} ₴?`)) {
                        if (balance >= c.price) {
                            balance -= c.price;
                            c.opened = true;
                            playSound('container');
                            document.getElementById('balance-display').innerText = balance.toLocaleString();
                            
                            // Выпадание авто
                            const reward = CAR_TYPES[Math.floor(Math.random() * CAR_TYPES.length)];
                            garage.push(reward);
                            alert(`Вітаємо! Ви виграли ${reward.name} (${reward.price.toLocaleString()} ₴)!`);
                            c.mesh.position.y = -10; // Сховати контейнер
                        } else {
                            alert("Недостатньо коштів!");
                        }
                    }
                }
            });

            renderer.render(scene, camera);
        }
        animate();

        // --- ГАРАЖ ТА ІКОНКИ ---
        function openGarage() {
            const list = document.getElementById('garage-list');
            list.innerHTML = '';
            if (garage.length === 0) {
                list.innerHTML = '<p style="grid-column: 1/-1; text-align:center;">Гараж порожній. Купуйте контейнери!</p>';
            } else {
                garage.forEach((car, index) => {
                    const card = document.createElement('div');
                    card.className = 'car-card';
                    card.innerHTML = `
                        <div class="car-icon">${car.icon}</div>
                        <h3>${car.name}</h3>
                        <p>Ціна: ${car.price.toLocaleString()} ₴</p>
                        <button class="btn-action" onclick="spawnCar(${index})">Кататися 🔑</button>
                        <button class="btn-action btn-sell" onclick="sellCar(${index})">Продати 💰</button>
                    `;
                    list.appendChild(card);
                });
            }
            document.getElementById('garage-modal').style.display = 'flex';
        }

        function closeGarage() { document.getElementById('garage-modal').style.display = 'none'; }

        function sellCar(index) {
            balance += garage[index].price;
            document.getElementById('balance-display').innerText = balance.toLocaleString();
            garage.splice(index, 1);
            openGarage();
        }

        function spawnCar(index) {
            closeGarage();
            const carData = garage[index];
            
            // Створення 3D машини
            const carGroup = new THREE.Group();
            const body = new THREE.Mesh(
                new THREE.BoxGeometry(2, 1, 4),
                new THREE.MeshStandardMaterial({ color: carData.color, metalness: 0.6, roughness: 0.2 })
            );
            body.position.y = 0.6;
            body.castShadow = true;
            carGroup.add(body);

            // Колеса
            const wheelGeo = new THREE.CylinderGeometry(0.4, 0.4, 0.3, 16);
            const wheelMat = new THREE.MeshStandardMaterial({ color: 0x111111 });
            [[-1, 0.4, 1.2], [1, 0.4, 1.2], [-1, 0.4, -1.2], [1, 0.4, -1.2]].forEach(p => {
                const w = new THREE.Mesh(wheelGeo, wheelMat);
                w.rotation.z = Math.PI / 2;
                w.position.set(...p);
                carGroup.add(w);
            });

            carGroup.position.copy(playerGroup.position);
            scene.add(carGroup);

            playerGroup.visible = false;
            inCar = true;
            currentDrivenCar = carGroup;
            document.getElementById('exit-car-btn').style.display = 'block';

            playSound('engine');
        }

        function exitCar() {
            if (currentDrivenCar) {
                playerGroup.position.copy(currentDrivenCar.position);
                playerGroup.position.x += 3;
                playerGroup.visible = true;
                scene.remove(currentDrivenCar);
                currentDrivenCar = null;
                inCar = false;
                document.getElementById('exit-car-btn').style.display = 'none';
            }
        }

        // Авто-ресайз
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
