<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>DSMAC - Combate Drone Visível</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #0b131a;
            font-family: 'Courier New', Courier, monospace;
            color: #39ff14;
        }

        #canvas-container {
            width: 100vw;
            height: 100vh;
            position: absolute;
            top: 0;
            left: 0;
            z-index: 1;
        }

        #hud {
            position: absolute;
            top: 20px;
            left: 20px;
            background: rgba(6, 12, 18, 0.9);
            padding: 20px;
            border: 1px solid #39ff1433;
            border-radius: 4px;
            z-index: 10;
            width: 340px;
            pointer-events: none;
            box-shadow: 0 0 20px rgba(57, 255, 20, 0.1);
        }

        h2 {
            margin: 0 0 5px 0;
            font-size: 15px;
            letter-spacing: 1px;
            color: #ffffff;
        }

        .stat {
            font-size: 11px;
            margin: 8px 0;
            display: flex;
            justify-content: space-between;
        }

        .label {
            color: #a0b0a0;
        }

        .value {
            font-weight: bold;
            text-shadow: 0 0 5px rgba(57, 255, 20, 0.5);
        }

        .alert-active {
            color: #ff3333;
            animation: blink 1s infinite;
        }

        @keyframes blink {
            50% { opacity: 0.3; }
        }

        #instructions {
            position: absolute;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(0, 0, 0, 0.8);
            padding: 8px 16px;
            border-radius: 4px;
            font-size: 11px;
            z-index: 10;
            pointer-events: none;
            border: 1px solid #39ff1422;
            color: #ffffff;
        }
    </style>
</head>
<body>

<div id="canvas-container"></div>

<div id="hud">
    <h2>SISTEMA INTERCEPTADOR ATR</h2>
    <hr style="border-color: #39ff1433; margin-bottom: 12px;">
    <div class="stat"><span class="label">INTERFERÊNCIA EW (JAMMING):</span> <span class="value alert-active">MÁXIMA (SINAL SAT OUT)</span></div>
    <div class="stat"><span class="label">SISTEMA ANTI-BLOQUEIO:</span> <span class="value" style="color: #00ffff;">DSMAC LOCAL ATIVO</span></div>
    <div class="stat"><span class="label">ALVOS DESTRUÍDOS:</span> <span class="value" id="hud-score" style="color:#ffffff;">0</span></div>
    <div class="stat"><span class="label">AUTO-GUIA (HÉLICES):</span> <span class="value" style="color: #ff9900;">VETOR DE COLISÃO</span></div>
</div>

<div id="instructions">Use o mouse (clique e arraste) para girar a câmera e ver os drones voando</div>

<script src="https://cloudflare.com"></script>
<script src="https://jsdelivr.net"></script>

<script>
    // --- CONFIGURAÇÃO ATMOSFÉRICA DA CENA ---
    const container = document.getElementById('canvas-container');
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0f14);
    scene.fog = new THREE.FogExp2(0x0a0f14, 0.015);

    // Câmera posicionada mais perto para garantir visibilidade imediata
    const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(0, 15, 40);

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    container.appendChild(renderer.domElement);

    const controls = new THREE.OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;

    // --- ILUMINAÇÃO FORTE ---
    const ambientLight = new THREE.AmbientLight(0xffffff, 1.2);
    scene.add(ambientLight);

    const dirLight = new THREE.DirectionalLight(0x00f0ff, 1.5);
    dirLight.position.set(20, 40, 20);
    scene.add(dirLight);

    // --- TERRENO DE COMBATE ---
    const grid = new THREE.GridHelper(100, 40, 0x1f3d52, 0x102030);
    grid.position.y = -10;
    scene.add(grid);

    // --- MODELAGEM GRANDE: DRONE INTERCEPTADOR (NOSSO DRONE) ---
    const droneGroup = new THREE.Group();
    
    // Corpo robusto e escuro
    const bodyGeo = new THREE.CylinderGeometry(2, 2.5, 1, 6);
    const bodyMat = new THREE.MeshStandardMaterial({ color: 0x1a232c, metalness: 0.8, roughness: 0.2 });
    const droneBody = new THREE.Mesh(bodyGeo, bodyMat);
    droneGroup.add(droneBody);

    // Câmera/Lente DSMAC frontal brilhante
    const eyeGeo = new THREE.SphereGeometry(0.6, 16, 16);
    const eyeMat = new THREE.MeshBasicMaterial({ color: 0x39ff14 });
    const droneEye = new THREE.Mesh(eyeGeo, eyeMat);
    droneEye.position.set(0, 0, 2.2);
    droneGroup.add(droneEye);

    // Grandes braços das hélices
    const armGeo = new THREE.BoxGeometry(9, 0.3, 0.4);
    const armMat = new THREE.MeshStandardMaterial({ color: 0x2c3e50 });
    const arm1 = new THREE.Mesh(armGeo, armMat);
    arm1.rotation.y = Math.PI / 4;
    droneGroup.add(arm1);
    const arm2 = new THREE.Mesh(armGeo, armMat);
    arm2.rotation.y = -Math.PI / 4;
    droneGroup.add(arm2);

    // Hélices visíveis
    const propGeo = new THREE.BoxGeometry(3.5, 0.05, 0.3);
    const propMat = new THREE.MeshStandardMaterial({ color: 0x000000 });
    const propellers = [];
    const propPositions = [[3.2, 0.6, 3.2], [-3.2, 0.6, 3.2], [3.2, 0.6, -3.2], [-3.2, 0.6, -3.2]];

    propPositions.forEach((pos) => {
        const prop = new THREE.Mesh(propGeo, propMat);
        prop.position.set(pos[0], pos[1], pos[2]);
        droneGroup.add(prop);
        propellers.push(prop);
    });

    scene.add(droneGroup);

    // --- MODELAGEM GRANDE: DRONE INIMIGO (ALVO MÓVEL) ---
    const enemyGroup = new THREE.Group();
    
    // Fuselagem esférica blindada inimiga
    const enemyBodyGeo = new THREE.SphereGeometry(2, 12, 12);
    const enemyBodyMat = new THREE.MeshStandardMaterial({ color: 0x222222, metalness: 0.9 });
    const enemyBody = new THREE.Mesh(enemyBodyGeo, enemyBodyMat);
    enemyGroup.add(enemyBody);

    // Assinatura de travamento digital e térmica ATR (Wireframe Laranja Neon bem grande)
    const signatureGeo = new THREE.BoxGeometry(4.5, 4.5, 4.5);
    const signatureMat = new THREE.MeshBasicMaterial({ color: 0xff4500, wireframe: true, transparent: true, opacity: 0.6 });
    const signatureBox = new THREE.Mesh(signatureGeo, signatureMat);
    enemyGroup.add(signatureBox);

    scene.add(enemyGroup);

    // --- RAIO LASER ÓPTICO DE RASTREAMENTO (DSMAC ANTI-JAMMING) ---
    const lineGeo = new THREE.BufferGeometry().setFromPoints([new THREE.Vector3(), new THREE.Vector3()]);
    const lineMat = new THREE.LineBasicMaterial({ color: 0x39ff14, linewidth: 3 }); // Linha verde forte
    const lockLine = new THREE.Line(lineGeo, lineMat);
    scene.add(lockLine);

    // --- POSIÇÕES INICIAIS ---
    let dronePos = new THREE.Vector3(-18, 5, -5);
    let enemyPos = new THREE.Vector3(15, 2, 10);
    let droneSpeed = 16.0; 
    let score = 0;

    droneGroup.position.copy(dronePos);
    enemyGroup.position.copy(enemyPos);

    const clock = new THREE.Clock();
    const hudScore = document.getElementById('hud-score');

    function respawnInimigo() {
        // Teleporta o inimigo para um ponto distante no ar aleatório para recomeçar a caça
        enemyPos.set(
            (Math.random() - 0.5) * 35 + 10,
            (Math.random() * 10) + 1,
            (Math.random() - 0.5) * 35
        );
        enemyGroup.position.copy(enemyPos);
        score++;
        hudScore.innerText = score;
    }

    // --- LOOP DE ANIMAÇÃO ---
    function animate() {
        requestAnimationFrame(animate);
        const dt = clock.getDelta();
        const time = clock.getElapsedTime();

        // 1. Voo caótico e evasivo do Drone Inimigo pelo ar
        enemyPos.x = 18 * Math.sin(time * 0.5);
        enemyPos.z = 15 * Math.cos(time * 0.3);
        enemyPos.y = 4 * Math.sin(time * 1.2) + 2; // Oscilação de altitude no ar
        enemyGroup.position.copy(enemyPos);

        // Giro da caixa de assinatura digital do alvo
        signatureBox.rotation.y += 0.02;

        // 2. Perseguição Autônoma DSMAC (Cálculo de aproximação sem rádio)
        const vectorToTarget = new THREE.Vector3().subVectors(enemyPos, dronePos);
        const distance = vectorToTarget.length();

        if (distance > 2.5) {
            const direction = vectorToTarget.clone().normalize();
            
            // Computador de bordo altera rotação física das hélices e corrige a rota
            dronePos.addScaledVector(direction, droneSpeed * dt);
            droneGroup.position.copy(dronePos);
            
            // O drone aponta fisicamente o nariz/lente para o alvo móvel
            droneGroup.lookAt(enemyPos);

            // Gira as hélices rapidamente
            propellers.forEach((prop) => {
                prop.rotation.y += 40 * dt;
            });
        } else {
            // Se aproximou o suficiente, considera alvo interceptado e gera outro
            respawnInimigo();
        }

        // 3. Mantém a linha de travamento óptico atualizada entre os dois drones
        const points = [droneGroup.position.clone(), target = enemyGroup.position.clone()];
        lockLine.geometry.setFromPoints(points);

        controls.update();
        renderer.render(scene, camera);
    }

    // AJUSTE DE TELA RESPONSIVO
    window.addEventListener('resize', () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    });

    animate();
</script>

</body>
