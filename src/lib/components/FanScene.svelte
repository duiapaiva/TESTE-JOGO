<script>
  import { onMount, onDestroy } from 'svelte';
  import { createEventDispatcher } from 'svelte';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
  import { Pane } from 'tweakpane';

  const dispatch = createEventDispatcher();

  // Props externas: controlados pelo UIOverlay / App.svelte
  export let controlPower = true;
  export let controlSpeed = 8.0;


  let canvasContainer;
  let paneContainer;

  // Three.js variables
  let scene, camera, renderer, controls;
  let animationFrameId;
  let pane;

  // 3D Objects
  let fanHeadGroup;      // The parts that rotate during oscillation (motor, hub, blades, cage)
  let rotorGroup;        // The parts that spin (hub, blades)
  let bladeMeshes = [];  // Array of blade meshes to easily update their color/material
  let windParticles;     // Particle system for wind

  // Settings object bound to Tweakpane
  const settings = {
    power: true,
    speed: 8.0,
    blades: 3,
    oscillation: true,
    oscillationRange: 45, // degrees
    oscillationSpeed: 1.5,
    fanColor: '#00f2fe',
    showWind: true,
    wireframeCage: true
  };

  // Sincroniza props externas → settings (reatividade Svelte)
  $: settings.power = controlPower;
  $: settings.speed  = controlSpeed;

  // State variables for animation loop
  let currentRotationSpeed = 0;
  let targetRotationSpeed = 0;
  let rotationAngle = 0;
  let oscillationTime = 0;
  let isDestroyed = false;

  // Clock must live outside animate() so getDelta() accumulates time correctly
  const clock = new THREE.Clock();

  // Dispatches telemetry back to parent
  function updateTelemetry() {
    dispatch('telemetry', {
      speed: currentRotationSpeed,
      power: settings.power,
      blades: settings.blades
    });
  }

  // Build the fan model
  function buildFan() {
    // Clear old elements if rebuilding
    if (fanHeadGroup) {
      scene.remove(fanHeadGroup);
    }
    bladeMeshes = [];

    // Create main head group (oscillates)
    fanHeadGroup = new THREE.Group();
    fanHeadGroup.position.set(0, 0.6, 0);
    scene.add(fanHeadGroup);

    // 1. Motor Casing
    const motorGeom = new THREE.CylinderGeometry(0.25, 0.25, 0.45, 32);
    motorGeom.rotateX(Math.PI / 2); // align along Z-axis
    const motorMat = new THREE.MeshStandardMaterial({
      color: 0x1f2937,
      roughness: 0.4,
      metalness: 0.8
    });
    const motorMesh = new THREE.Mesh(motorGeom, motorMat);
    motorMesh.position.set(0, 0, -0.15);
    motorMesh.castShadow = true;
    motorMesh.receiveShadow = true;
    fanHeadGroup.add(motorMesh);

    // Rear Motor cap
    const capGeom = new THREE.SphereGeometry(0.25, 32, 16, 0, Math.PI * 2, 0, Math.PI / 2);
    capGeom.rotateX(-Math.PI / 2);
    const capMesh = new THREE.Mesh(capGeom, motorMat);
    capMesh.position.set(0, 0, -0.37);
    capMesh.castShadow = true;
    fanHeadGroup.add(capMesh);

    // 2. Rotor Group (Spins)
    rotorGroup = new THREE.Group();
    rotorGroup.position.set(0, 0, 0.1);
    fanHeadGroup.add(rotorGroup);

    // Center hub
    const hubGeom = new THREE.SphereGeometry(0.15, 32, 16);
    const hubMat = new THREE.MeshStandardMaterial({
      color: 0x4b5563,
      roughness: 0.3,
      metalness: 0.9
    });
    const hubMesh = new THREE.Mesh(hubGeom, hubMat);
    hubMesh.scale.set(1, 1, 0.7); // flatten slightly
    hubMesh.castShadow = true;
    rotorGroup.add(hubMesh);

    // Blades
    rebuildBlades();

    // 3. Protective Cage
    const cageGroup = new THREE.Group();
    cageGroup.position.set(0, 0, 0.08);
    fanHeadGroup.add(cageGroup);

    const cageRadius = 0.95;
    
    // Front/Rear rims (torus)
    const rimGeom = new THREE.TorusGeometry(cageRadius, 0.012, 8, 64);
    const rimMat = new THREE.MeshStandardMaterial({
      color: 0x6b7280,
      wireframe: settings.wireframeCage,
      roughness: 0.2,
      metalness: 0.9
    });
    
    const rimFront = new THREE.Mesh(rimGeom, rimMat);
    rimFront.position.set(0, 0, 0.05);
    cageGroup.add(rimFront);

    const rimBack = new THREE.Mesh(rimGeom, rimMat);
    rimBack.position.set(0, 0, -0.15);
    cageGroup.add(rimBack);

    // Cage profiles (spherical wireframe look)
    const cageSphereGeom = new THREE.SphereGeometry(cageRadius, 16, 8);
    const cageSphereMat = new THREE.MeshBasicMaterial({
      color: 0x4b5563,
      wireframe: true,
      transparent: true,
      opacity: 0.15
    });
    const cageSphere = new THREE.Mesh(cageSphereGeom, cageSphereMat);
    cageSphere.scale.set(1, 1, 0.3); // squash it along Z
    cageSphere.position.set(0, 0, -0.05);
    cageGroup.add(cageSphere);

    // Center logo plate
    const plateGeom = new THREE.CylinderGeometry(0.12, 0.12, 0.02, 32);
    plateGeom.rotateX(Math.PI / 2);
    const plateMat = new THREE.MeshStandardMaterial({
      color: 0x111827,
      roughness: 0.2,
      metalness: 0.8
    });
    const plate = new THREE.Mesh(plateGeom, plateMat);
    plate.position.set(0, 0, 0.06);
    cageGroup.add(plate);

    // Small glowing dot in the center of the plate
    const dotGeom = new THREE.SphereGeometry(0.02, 16, 16);
    const dotMat = new THREE.MeshBasicMaterial({ color: settings.fanColor });
    const dot = new THREE.Mesh(dotGeom, dotMat);
    dot.position.set(0, 0, 0.075);
    cageGroup.add(dot);
  }

  // Rebuild blades dynamically when blade count changes
  function rebuildBlades() {
    // Clear old blade meshes
    if (rotorGroup) {
      // Remove all children except the hub (first child)
      while (rotorGroup.children.length > 1) {
        rotorGroup.remove(rotorGroup.children[1]);
      }
    }
    bladeMeshes = [];

    const count = settings.blades;
    const angleStep = (Math.PI * 2) / count;
    
    // Blade material
    const bladeMat = new THREE.MeshStandardMaterial({
      color: new THREE.Color(settings.fanColor),
      roughness: 0.15,
      metalness: 0.5,
      transparent: true,
      opacity: 0.85,
      side: THREE.DoubleSide
    });

    for (let i = 0; i < count; i++) {
      const bladeAngle = i * angleStep;

      // Custom shape for organic aerodynamic look
      const shape = new THREE.Shape();
      shape.moveTo(0, 0.1);
      shape.quadraticCurveTo(0.18, 0.35, 0.15, 0.7);
      shape.quadraticCurveTo(0.08, 0.85, 0, 0.85);
      shape.quadraticCurveTo(-0.08, 0.85, -0.15, 0.7);
      shape.quadraticCurveTo(-0.18, 0.35, 0, 0.1);

      // Extrude shape with twist-like scaling
      const extrudeSettings = {
        steps: 1,
        depth: 0.015,
        bevelEnabled: true,
        bevelThickness: 0.01,
        bevelSize: 0.01,
        bevelSegments: 3
      };

      const bladeGeom = new THREE.ExtrudeGeometry(shape, extrudeSettings);
      
      // Rotate geometry around Y-axis to give it pitch/angle
      bladeGeom.rotateY(-0.25);
      // Center the geometry origin at the hub connection point
      bladeGeom.translate(0, 0, -0.01);

      const bladeMesh = new THREE.Mesh(bladeGeom, bladeMat);
      
      // Position and rotate around the rotor hub
      bladeMesh.rotation.z = bladeAngle;
      bladeMesh.castShadow = true;
      bladeMesh.receiveShadow = true;
      
      rotorGroup.add(bladeMesh);
      bladeMeshes.push(bladeMesh);
    }
  }

  // Create Wind Particles
  function buildWind() {
    if (windParticles) {
      scene.remove(windParticles);
    }

    const particleCount = 80;
    const geometry = new THREE.BufferGeometry();
    const positions = new Float32Array(particleCount * 3);
    const velocities = new Float32Array(particleCount);
    const opacities = new Float32Array(particleCount);

    for (let i = 0; i < particleCount; i++) {
      // Random coordinates in a cone in front of the fan (Z > 0)
      const r = Math.random() * 0.7;
      const theta = Math.random() * Math.PI * 2;
      positions[i * 3] = Math.cos(theta) * r;     // X
      positions[i * 3 + 1] = Math.sin(theta) * r; // Y
      positions[i * 3 + 2] = Math.random() * 3.5; // Z (distance in front)
      
      velocities[i] = 1.0 + Math.random() * 2.0; // speed factor
      opacities[i] = Math.random();
    }

    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute('opacity', new THREE.BufferAttribute(opacities, 1));

    // Custom shader material for fading wind lines
    const material = new THREE.ShaderMaterial({
      transparent: true,
      depthWrite: false,
      blending: THREE.AdditiveBlending,
      vertexShader: `
        attribute float opacity;
        varying float vOpacity;
        void main() {
          vOpacity = opacity;
          vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
          gl_Position = projectionMatrix * mvPosition;
          gl_PointSize = 3.5 * (300.0 / -mvPosition.z);
        }
      `,
      fragmentShader: `
        varying float vOpacity;
        uniform vec3 color;
        void main() {
          // Circular particle shape
          float dist = length(gl_PointCoord - vec2(0.5));
          if (dist > 0.5) discard;
          float strength = 1.0 - (dist * 2.0);
          gl_FragColor = vec4(color, vOpacity * strength * 0.4);
        }
      `,
      uniforms: {
        color: { value: new THREE.Color(settings.fanColor) }
      }
    });

    windParticles = new THREE.Points(geometry, material);
    // Align with fan head position initially, but we update in animation loop
    scene.add(windParticles);
    
    // Save velocities as custom property
    windParticles.userData = { velocities };
  }

  // Update wind particles positioning and simulation
  function updateWind(deltaTime) {
    if (!windParticles || !settings.showWind) return;

    const positions = windParticles.geometry.attributes.position.array;
    const opacities = windParticles.geometry.attributes.opacity.array;
    const velocities = windParticles.userData.velocities;
    const count = velocities.length;

    // Get current fan position and orientation vector pointing forward
    const fanPos = new THREE.Vector3();
    fanHeadGroup.getWorldPosition(fanPos);
    
    const fanDir = new THREE.Vector3(0, 0, 1);
    fanDir.applyQuaternion(fanHeadGroup.quaternion);

    // Right and Up vectors for spawning offset
    const right = new THREE.Vector3(1, 0, 0).applyQuaternion(fanHeadGroup.quaternion);
    const up = new THREE.Vector3(0, 1, 0).applyQuaternion(fanHeadGroup.quaternion);

    for (let i = 0; i < count; i++) {
      // Progress the particle forward along the fan's direction
      // Speed scales with current rotation speed
      const speed = velocities[i] * (currentRotationSpeed + 0.1) * 0.4;
      
      // Calculate local Z progress
      let pZ = positions[i * 3 + 2] + speed * deltaTime;
      
      // Reset if too far or speed is 0
      if (pZ > 4.5 || currentRotationSpeed < 0.1) {
        pZ = 0.05 + Math.random() * 0.1; // reset close to the blades
        const r = Math.random() * 0.7;
        const theta = Math.random() * Math.PI * 2;
        positions[i * 3] = Math.cos(theta) * r;     // X local offset
        positions[i * 3 + 1] = Math.sin(theta) * r; // Y local offset
      }
      positions[i * 3 + 2] = pZ;

      // Calculate final world position
      const worldPos = fanPos.clone()
        .addScaledVector(fanDir, pZ)
        .addScaledVector(right, positions[i * 3])
        .addScaledVector(up, positions[i * 3 + 1]);

      // Write back values in world coordinates for rendering, or keep relative to fanHeadGroup
      // It's easier if the windParticles system is a child of fanHeadGroup!
      // Let's attach windParticles to fanHeadGroup to make physics local
    }

    windParticles.geometry.attributes.position.needsUpdate = true;
    
    // Fade out as they get further away
    for (let i = 0; i < count; i++) {
      const z = positions[i * 3 + 2];
      opacities[i] = Math.max(0, 1.0 - z / 4.0) * (currentRotationSpeed / settings.speed);
    }
    windParticles.geometry.attributes.opacity.needsUpdate = true;
  }

  // Animation Frame Loop
  function animate() {
    if (isDestroyed) return;
    animationFrameId = requestAnimationFrame(animate);

    let deltaTime = clock.getDelta();
    // Clamp delta time to prevent physics jumping when tab is inactive
    if (deltaTime > 0.1) deltaTime = 0.1;
    if (deltaTime <= 0) deltaTime = 0.016;

    // 1. Blade speed interpolation (power on/off transition)
    targetRotationSpeed = settings.power ? settings.speed : 0;
    const accelRate = settings.power ? 2.5 : 1.2; // spin up/down rates
    currentRotationSpeed += (targetRotationSpeed - currentRotationSpeed) * deltaTime * accelRate;

    // 2. Rotate blades
    rotationAngle += currentRotationSpeed * 10 * deltaTime;
    if (rotorGroup) {
      rotorGroup.rotation.z = rotationAngle;
    }

    // 3. Head Oscillation
    if (settings.oscillation && currentRotationSpeed > 0.2) {
      oscillationTime += deltaTime * settings.oscillationSpeed * (currentRotationSpeed / settings.speed);
      const angleRad = THREE.MathUtils.degToRad(settings.oscillationRange);
      fanHeadGroup.rotation.y = Math.sin(oscillationTime) * angleRad;
    }

    // 4. Update wind particles
    updateWind(deltaTime);

    // 5. Update controls
    controls.update();

    // 6. Render
    renderer.render(scene, camera);

    // Send telemetry data
    updateTelemetry();
  }

  // Set up Tweakpane
  function initTweakpane() {
    pane = new Pane({
      container: paneContainer,
      title: 'Configurações do Ventilador'
    });

    pane.addBinding(settings, 'power', { label: 'Energia' }).on('change', () => {
      // update color indicator if needed
    });
    
    pane.addBinding(settings, 'speed', {
      label: 'Velocidade',
      min: 0,
      max: 20,
      step: 0.5
    });

    pane.addBinding(settings, 'blades', {
      label: 'Nº de Pás',
      min: 2,
      max: 6,
      step: 1
    }).on('change', () => {
      rebuildBlades();
    });

    const folderOsc = pane.addFolder({ title: 'Oscilação' });
    folderOsc.addBinding(settings, 'oscillation', { label: 'Ativa' });
    folderOsc.addBinding(settings, 'oscillationRange', {
      label: 'Amplitude (Graus)',
      min: 10,
      max: 90,
      step: 5
    });
    folderOsc.addBinding(settings, 'oscillationSpeed', {
      label: 'Velocidade Osc.',
      min: 0.5,
      max: 3.0,
      step: 0.1
    });

    const folderVisual = pane.addFolder({ title: 'Visual & Efeitos' });
    folderVisual.addBinding(settings, 'fanColor', { label: 'Cor das Pás' }).on('change', (ev) => {
      // Update color on existing meshes
      bladeMeshes.forEach(mesh => {
        mesh.material.color.set(ev.value);
      });
      // Update particle uniforms
      if (windParticles) {
        windParticles.material.uniforms.color.value.set(ev.value);
      }
    });

    folderVisual.addBinding(settings, 'showWind', { label: 'Mostrar Vento' }).on('change', (ev) => {
      if (windParticles) {
        windParticles.visible = ev.value;
      }
    });

    folderVisual.addBinding(settings, 'wireframeCage', { label: 'Grade Aramada' }).on('change', () => {
      buildFan(); // Rebuild with new cage wireframe setting
      // Re-attach wind system to new head
      if (windParticles) {
        fanHeadGroup.add(windParticles);
      }
    });
  }

  onMount(() => {
    // 1. Scene Setup
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0b10);
    scene.fog = new THREE.FogExp2(0x0a0b10, 0.08);

    // 2. Camera Setup
    camera = new THREE.PerspectiveCamera(
      45,
      canvasContainer.clientWidth / canvasContainer.clientHeight,
      0.1,
      100
    );
    camera.position.set(0, 1.2, 3.2);

    // 3. Renderer Setup
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(canvasContainer.clientWidth, canvasContainer.clientHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    canvasContainer.appendChild(renderer.domElement);

    // 4. Orbit Controls
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.maxPolarAngle = Math.PI / 2 + 0.1; // limit camera looking under floor
    controls.minDistance = 1.5;
    controls.maxDistance = 8;
    controls.target.set(0, 0.6, 0);

    // 5. Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.2);
    scene.add(ambientLight);

    // Neon Accent Spotlight
    const spotlight = new THREE.SpotLight(0x00f2fe, 8, 15, Math.PI / 3, 0.5, 1);
    spotlight.position.set(2, 4, 3);
    spotlight.castShadow = true;
    spotlight.shadow.mapSize.width = 1024;
    spotlight.shadow.mapSize.height = 1024;
    spotlight.shadow.bias = -0.001;
    scene.add(spotlight);

    // Subtle Rim Light
    const rimLight = new THREE.DirectionalLight(0xd946ef, 3);
    rimLight.position.set(-3, 2, -2);
    scene.add(rimLight);

    // Soft fill light
    const fillLight = new THREE.DirectionalLight(0xffffff, 0.4);
    fillLight.position.set(0, 2, 4);
    scene.add(fillLight);

    // 6. Base / Pole Structure (Non-moving part of fan)
    const fanStandGroup = new THREE.Group();
    scene.add(fanStandGroup);

    // Base plate
    const standBaseGeom = new THREE.CylinderGeometry(0.45, 0.48, 0.08, 32);
    const standBaseMat = new THREE.MeshStandardMaterial({
      color: 0x111827,
      roughness: 0.5,
      metalness: 0.7
    });
    const standBase = new THREE.Mesh(standBaseGeom, standBaseMat);
    standBase.position.set(0, 0.04, 0);
    standBase.receiveShadow = true;
    fanStandGroup.add(standBase);

    // Vertical metal pole
    const poleGeom = new THREE.CylinderGeometry(0.04, 0.045, 1.2, 16);
    const poleMat = new THREE.MeshStandardMaterial({
      color: 0xe5e7eb,
      roughness: 0.15,
      metalness: 0.95
    });
    const pole = new THREE.Mesh(poleGeom, poleMat);
    pole.position.set(0, 0.6, 0);
    pole.castShadow = true;
    pole.receiveShadow = true;
    fanStandGroup.add(pole);

    // Ring collar
    const collarGeom = new THREE.CylinderGeometry(0.06, 0.06, 0.06, 16);
    const collar = new THREE.Mesh(collarGeom, standBaseMat);
    collar.position.set(0, 0.9, 0);
    fanStandGroup.add(collar);

    // 7. Grid Floor
    const floorSize = 40;
    const gridHelper = new THREE.GridHelper(floorSize, 40, 0x00f2fe, 0x1f2937);
    gridHelper.position.y = 0;
    gridHelper.material.opacity = 0.25;
    gridHelper.material.transparent = true;
    scene.add(gridHelper);

    // Floor physics receiver plane (invisible shadow catcher)
    const floorGeom = new THREE.PlaneGeometry(floorSize, floorSize);
    const floorMat = new THREE.ShadowMaterial({ opacity: 0.4 });
    const floor = new THREE.Mesh(floorGeom, floorMat);
    floor.rotation.x = -Math.PI / 2;
    floor.position.y = 0;
    floor.receiveShadow = true;
    scene.add(floor);

    // 8. Build moving parts
    buildFan();

    // 9. Build wind particle system & attach to the fan head
    buildWind();
    fanHeadGroup.add(windParticles);

    // 10. Controls init & resize listener
    window.addEventListener('resize', handleResize);

    // 11. Initial Tweakpane setup
    initTweakpane();

    // 12. Run loop
    animate();
  });

  function handleResize() {
    if (!canvasContainer || !camera || !renderer) return;
    camera.aspect = canvasContainer.clientWidth / canvasContainer.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(canvasContainer.clientWidth, canvasContainer.clientHeight);
  }

  onDestroy(() => {
    isDestroyed = true;
    window.removeEventListener('resize', handleResize);
    cancelAnimationFrame(animationFrameId);

    // Dispose Tweakpane
    if (pane) {
      pane.dispose();
    }

    // Dispose WebGL context & geometries
    if (renderer) {
      renderer.dispose();
    }

    // Recursively dispose scene geometries and materials
    scene?.traverse((object) => {
      if (object.geometry) object.geometry.dispose();
      if (object.material) {
        if (Array.isArray(object.material)) {
          object.material.forEach(mat => mat.dispose());
        } else {
          object.material.dispose();
        }
      }
    });
  });
</script>

<div class="scene-wrapper">
  <div bind:this={canvasContainer} class="canvas-container"></div>
  <div bind:this={paneContainer} class="pane-wrapper ui-element"></div>
</div>

<style>
  .scene-wrapper {
    width: 100%;
    height: 100%;
    position: relative;
  }
</style>
