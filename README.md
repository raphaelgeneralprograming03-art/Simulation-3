<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Visor Tático Aéreo 3D - Visão de Satélite & Câmera EO/IR</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { overflow: hidden; background-color: #000; font-family: 'Courier New', Courier, monospace; color: #00ffcc; }
    #canvas-container { width: 100vw; height: 100vh; }

    /* HUD ESTÁTICO (IMAGEM VIRTUAL DE TELA) */
    #hud-overlay {
      position: absolute;
      top: 0; left: 0;
      width: 100vw; height: 100vh;
      pointer-events: none; /* Mantém a interação 3D do mouse ativa abaixo */
      border: 20px solid rgba(0, 255, 204, 0.15);
      box-sizing: border-box;
    }

    /* Retícula Central de Mira */
    .reticle-box {
      position: absolute;
      top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      width: 160px; height: 160px;
      border: 2px solid #00ffcc;
      border-radius: 50%;
      box-shadow: 0 0 10px rgba(0, 255, 204, 0.5);
    }
    .reticle-box::before {
      content: '';
      position: absolute;
      top: 50%; left: -20px; right: -20px;
      height: 2px; background: #00ffcc;
    }
    .reticle-box::after {
      content: '';
      position: absolute;
      left: 50%; top: -20px; bottom: -20px;
      width: 2px; background: #00ffcc;
    }
    .center-dot {
      position: absolute;
      top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      width: 8px; height: 8px;
      background: #ff3366;
      border-radius: 50%;
      box-shadow: 0 0 8px #ff3366;
    }

    /* Caixas de Telemetria Virtual */
    .data-panel {
      position: absolute;
      background: rgba(5, 12, 20, 0.85);
      border: 1px solid #00ffcc;
      padding: 12px 18px;
      font-size: 13px;
      line-height: 1.6;
    }
    .top-left { top: 30px; left: 30px; }
    .top-right { top: 30px; right: 30px; text-align: right; }
    .bottom-left { bottom: 30px; left: 30px; }
    
    .txt-red { color: #ff3366; font-weight: bold; }
    .txt-green { color: #00ffcc; font-weight: bold; }
    .txt-yellow { color: #ffcc00; font-weight: bold; }
  </style>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js"></script>
</head>
<body>

  <div id="canvas-container"></div>

  <!-- HUD E CAMADA VIRTUAL 2D SOBREPOSTA -->
  <div id="hud-overlay">
    <div class="reticle-box">
      <div class="center-dot"></div>
    </div>

    <div class="data-panel top-left">
      <div>SYS: <span class="txt-green">EO/IR CAMERA [ACTIVE]</span></div>
      <div>MODE: <span class="txt-yellow">SATELLITE/AIR LOCK</span></div>
      <div>LAT: <span class="txt-green">48°26'19.2"N</span></div>
      <div>LON: <span class="txt-green">37°48'12.4"E</span></div>
    </div>

    <div class="data-panel top-right">
      <div>JAMMING EVISION: <span class="txt-red">98.5 dB</span></div>
      <div>STATUS: <span class="txt-green">HARDWARE ISOLATED</span></div>
      <div>SYSTEM EFFICIENCY: <span class="txt-green">94.0%</span></div>
    </div>

    <div class="data-panel bottom-left">
      <div>TARGET ID: <span class="txt-red">ARMOURED_VEHICLE_T72</span></div>
      <div>OPTICAL ERROR E(t): <span class="txt-yellow">Ex: 0.00px | Ey: 0.00px</span></div>
      <div>TRACKING: <span class="txt-green">LOCKED ON TARGET</span></div>
    </div>
  </div>

  <script>
    // --- 1. CENA E CÂMERA AÉREA DE VISÃO TÁTICA ---
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x050a12);

    // Câmera posicionada no alto simulando visão aérea/satélite de ataque
    const camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(0, 35, 25);

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.getElementById('canvas-container').appendChild(renderer.domElement);

    const controls = new THREE.OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.target.set(0, 0, 0);

    // Luzes da Cena
    scene.add(new THREE.AmbientLight(0xffffff, 0.9));
    const sunLight = new THREE.DirectionalLight(0xffddaa, 1.2);
    sunLight.position.set(30, 50, 20);
    scene.add(sunLight);

    // --- 2. IMAGEM DE SATÉLITE / MAPA NO SOLO ---
    // Criando a textura do mapa/satélite procedimentalmente (estilo Google Maps Satélite)
    const canvasMap = document.createElement('canvas');
    canvasMap.width = 1024; canvasMap.height = 1024;
    const ctx = canvasMap.getContext('2d');

    // Fundo de Vegetação/Campo Agrícola (Estilo Leste Europeu)
    ctx.fillStyle = '#2d3a22';
    ctx.fillRect(0, 0, 1024, 1024);

    // Detalhes do Terreno (Campos de cultivo)
    ctx.fillStyle = '#3a4a2c';
    ctx.fillRect(50, 50, 400, 924);
    ctx.fillStyle = '#222c19';
    ctx.fillRect(480, 50, 480, 400);

    // Estradas Rurais de Terra (Cruzamento Tático)
    ctx.strokeStyle = '#7a684d';
    ctx.lineWidth = 40;
    ctx.beginPath();
    ctx.moveTo(0, 512); ctx.lineTo(1024, 512); // Estrada horizontal
    ctx.moveTo(512, 0); ctx.lineTo(512, 1024); // Estrada vertical
    ctx.stroke();

    const mapTexture = new THREE.CanvasTexture(canvasMap);
    const groundGeo = new THREE.PlaneGeometry(80, 80);
    const groundMat = new THREE.MeshStandardMaterial({ map: mapTexture, roughness: 0.8 });
    const ground = new THREE.Mesh(groundGeo, groundMat);
    ground.rotation.x = -Math.PI / 2;
    scene.add(ground);

    // --- 3. MODELO 3D VIRTUAL DO TANQUE DE GUERRA (BLINDADO T-72) ---
    const tankGroup = new THREE.Group();

    // Chassi / Lagartas
    const chassisMat = new THREE.MeshStandardMaterial({ color: 0x1c2419, roughness: 0.6, metalness: 0.4 });
    const chassis = new THREE.Mesh(new THREE.BoxGeometry(4, 1.2, 7), chassisMat);
    chassis.position.y = 0.6;
    tankGroup.add(chassis);

    // Torre do Tanque
    const turretMat = new THREE.MeshStandardMaterial({ color: 0x273323, roughness: 0.5, metalness: 0.5 });
    const turret = new THREE.Mesh(new THREE.CylinderGeometry(1.6, 1.8, 1.0, 12), turretMat);
    turret.position.set(0, 1.5, -0.2);
    tankGroup.add(turret);

    // Canhão Principal
    const barrelMat = new THREE.MeshStandardMaterial({ color: 0x111610, metalness: 0.8 });
    const barrel = new THREE.Mesh(new THREE.CylinderGeometry(0.18, 0.18, 5), barrelMat);
    barrel.rotation.x = Math.PI / 2;
    barrel.position.set(0, 1.6, 3);
    tankGroup.add(barrel);

    // Posiciona o tanque exatamente no centro da retícula (P_alvo)
    tankGroup.position.set(0, 0, 0);
    scene.add(tankGroup);

    // --- 4. RENDERIZAÇÃO ESTÁTICA DEDICADA ---
    function renderScene() {
      requestAnimationFrame(renderScene);
      controls.update();
      renderer.render(scene, camera);
    }

    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });

    renderScene();
  </script>
</body>
</html>
