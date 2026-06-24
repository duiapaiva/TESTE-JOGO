<script>
  import { onMount, onDestroy } from "svelte";
  import { createEventDispatcher } from "svelte";
  import * as THREE from "three";
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
  import { Pane } from "tweakpane";

  const dispatch = createEventDispatcher();

  // Prop externa: quantidade de bolas por lançamento (controlado pelo UIOverlay)
  export let controlBallCount = 1;
  $: settings.ballsPerThrow = controlBallCount;

  let canvasContainer;
  let paneContainer;

  // Three.js variables
  let scene, camera, renderer, controls;
  let animationFrameId;
  let pane;

  // Physics params
  const settings = {
    gravity: 9.8,
    restitution: 0.72,
    ballSize: 0.28,
    throwForce: 14.0,
    spawnHeight: 1.2,
    ballsPerThrow: 1,
    breakable: true,
    wallColor: "#ea580c",
    ballColor: "#ffff00",
  };

  // Ball states
  let balls = [];
  let bricks = [];
  let particles = [];
  let isDestroyed = false;

  // Clock must live outside animate() so getDelta() accumulates time correctly
  const clock = new THREE.Clock();

  // Raycaster for click-on-ball detection
  const raycaster = new THREE.Raycaster();
  const mouse = new THREE.Vector2();

  // Preview ball — always visible at spawn position
  let previewBallMesh = null;
  let previewBallLight = null;
  let previewPulseTime = 0;

  // Stats to send to parent
  let collisionCount = 0;
  let lastImpactSpeed = 0;

  // Dimensions of the brick wall (larger, denser wall)
  const wallConfig = {
    rows: 12,
    cols: 10,
    brickW: 0.42,
    brickH: 0.18,
    brickD: 0.22,
    spacing: 0.004,
    zPos: -1.5,
  };

  // Brick class
  class Brick {
    constructor(mesh, initialPos, row, col) {
      this.mesh = mesh;
      this.initialPos = initialPos.clone();
      this.row = row;
      this.col = col;

      // Physics state
      this.isStatic = true;
      this.pos = initialPos.clone();
      this.vel = new THREE.Vector3();
      this.rot = new THREE.Vector3();
      this.rotVel = new THREE.Vector3();

      // Bounding box for collision detection
      this.halfW = wallConfig.brickW / 2;
      this.halfH = wallConfig.brickH / 2;
      this.halfD = wallConfig.brickD / 2;
    }

    reset() {
      this.isStatic = true;
      this.pos.copy(this.initialPos);
      this.vel.set(0, 0, 0);
      this.rot.set(0, 0, 0);
      this.rotVel.set(0, 0, 0);

      this.mesh.position.copy(this.initialPos);
      this.mesh.rotation.set(0, 0, 0);
      this.mesh.castShadow = true;
    }

    activate(impactVel) {
      if (!this.isStatic) return;
      this.isStatic = false;

      // Inherit some velocity from the impact, directed slightly outward
      this.vel.copy(impactVel).multiplyScalar(0.25);
      this.vel.y += 1.5 + Math.random() * 2.0; // pop upwards
      this.vel.x += (Math.random() - 0.5) * 2;

      // Set random rotation velocity
      this.rotVel.set(
        (Math.random() - 0.5) * 5,
        (Math.random() - 0.5) * 5,
        (Math.random() - 0.5) * 5,
      );
    }

    update(dt) {
      if (this.isStatic) return;

      // Apply gravity
      this.vel.y -= settings.gravity * dt;

      // Apply drag
      this.vel.multiplyScalar(1.0 - 0.3 * dt);
      this.rotVel.multiplyScalar(1.0 - 0.5 * dt);

      // Update position & rotation
      this.pos.addScaledVector(this.vel, dt);
      this.rot.addScaledVector(this.rotVel, dt);

      // Floor collision
      const floorLevel = this.halfH;
      if (this.pos.y < floorLevel) {
        this.pos.y = floorLevel;
        this.vel.y = -this.vel.y * 0.25; // bounce a bit

        // Horizontal friction
        this.vel.x *= 0.75;
        this.vel.z *= 0.75;
        this.rotVel.multiplyScalar(0.7);
      }

      // Sync to 3D mesh
      this.mesh.position.copy(this.pos);
      this.mesh.rotation.set(this.rot.x, this.rot.y, this.rot.z);
    }
  }

  // Ball class
  class Ball {
    constructor(mesh, pos, vel) {
      this.mesh = mesh;
      this.pos = pos.clone();
      this.vel = vel.clone();
      this.radius = settings.ballSize;

      // Visual Squash & Stretch effects
      this.scale = new THREE.Vector3(1, 1, 1);
      this.targetScale = new THREE.Vector3(1, 1, 1);
      this.life = 0;
    }

    update(dt) {
      this.life += dt;

      // Apply gravity
      this.vel.y -= settings.gravity * dt;

      // Apply slight air resistance
      this.vel.multiplyScalar(1.0 - 0.08 * dt);

      // Update position
      this.pos.addScaledVector(this.vel, dt);

      // Floor collision
      const floorLevel = this.radius;
      if (this.pos.y < floorLevel) {
        this.pos.y = floorLevel;
        this.vel.y = -this.vel.y * settings.restitution;

        // Friction on floor
        this.vel.x *= 0.96;
        this.vel.z *= 0.96;

        // Squash factor on Y axis based on landing velocity
        const squash = Math.max(0.4, 1.0 - Math.abs(this.vel.y) * 0.02);
        this.scale.set(1.2, squash, 1.2);

        if (Math.abs(this.vel.y) > 1.5) {
          spawnImpactParticles(this.pos, 0x555555, 8);
        }
      }

      // Smoothly recover from squash/stretch
      this.scale.lerp(this.targetScale, 10 * dt);

      // Sync to 3D mesh
      this.mesh.position.copy(this.pos);
      this.mesh.scale.copy(this.scale);
    }
  }

  // Setup the Wall dynamically
  function buildWall() {
    // Clear existing brick meshes and structures
    bricks.forEach((b) => scene.remove(b.mesh));
    bricks = [];

    const { rows, cols, brickW, brickH, brickD, spacing, zPos } = wallConfig;
    const brickMat = new THREE.MeshStandardMaterial({
      color: new THREE.Color(settings.wallColor),
      roughness: 0.75,
      metalness: 0.15,
    });

    const halfWallW = (cols * (brickW + spacing)) / 2;

    for (let r = 0; r < rows; r++) {
      // Row offset for running bond pattern
      const isOffsetRow = r % 2 !== 0;
      const rowCols = isOffsetRow ? cols + 1 : cols;
      const rowOffset = isOffsetRow ? -brickW / 2 : 0;

      for (let c = 0; c < rowCols; c++) {
        // Handle edge half-bricks for perfect running bond
        let currentW = brickW;
        let xOffset = 0;

        if (isOffsetRow) {
          if (c === 0) {
            currentW = brickW / 2;
            xOffset = brickW / 4;
          } else if (c === rowCols - 1) {
            currentW = brickW / 2;
            xOffset = -brickW / 4;
          }
        }

        const geom = new THREE.BoxGeometry(currentW, brickH, brickD);
        const mesh = new THREE.Mesh(geom, brickMat.clone());
        mesh.castShadow = true;
        mesh.receiveShadow = true;

        const posX =
          -halfWallW +
          c * (brickW + spacing) +
          rowOffset +
          xOffset +
          brickW / 2;
        const posY = r * (brickH + spacing) + brickH / 2;
        const posZ = zPos;

        const initialPos = new THREE.Vector3(posX, posY, posZ);
        mesh.position.copy(initialPos);
        scene.add(mesh);

        const brickObj = new Brick(mesh, initialPos, r, c);
        // Save bounds for custom half-bricks
        if (isOffsetRow && (c === 0 || c === rowCols - 1)) {
          brickObj.halfW = currentW / 2;
        }

        bricks.push(brickObj);
      }
    }
  }

  // Reset the entire scene state
  function resetScene() {
    // Clear active balls
    balls.forEach((b) => scene.remove(b.mesh));
    balls = [];

    // Clear particles
    particles.forEach((p) => scene.remove(p.mesh));
    particles = [];

    // Reset wall bricks
    bricks.forEach((b) => b.reset());

    // Restore preview ball
    if (previewBallMesh) previewBallMesh.visible = true;

    // Reset metrics
    collisionCount = 0;
    lastImpactSpeed = 0;
    dispatchTelemetry();
  }

  // Build the always-visible preview ball at the spawn position (sitting on the floor)
  function buildPreviewBall() {
    if (previewBallMesh) scene.remove(previewBallMesh);
    if (previewBallLight) scene.remove(previewBallLight);

    const radius = settings.ballSize;
    const geom = new THREE.SphereGeometry(radius, 32, 32);
    const mat = new THREE.MeshStandardMaterial({
      color: new THREE.Color(settings.ballColor),
      roughness: 0.08,
      metalness: 0.85,
      emissive: new THREE.Color(settings.ballColor),
      emissiveIntensity: 0.5,
    });
    previewBallMesh = new THREE.Mesh(geom, mat);
    // Sit exactly on the floor: y = radius
    previewBallMesh.position.set(0, radius, 2.2);
    previewBallMesh.castShadow = true;
    scene.add(previewBallMesh);

    previewBallLight = new THREE.PointLight(
      new THREE.Color(settings.ballColor),
      1.5,
      5,
    );
    previewBallLight.position.copy(previewBallMesh.position);
    scene.add(previewBallLight);
  }

  // Throw N balls towards the wall from the preview ball position
  function throwBall() {
    const MAX_BALLS = 5;
    const requestedCount = Math.max(1, Math.round(settings.ballsPerThrow));
    const availableSlots = Math.max(0, MAX_BALLS - balls.length);
    const count = Math.min(requestedCount, availableSlots);

    console.log(
      "Bolas selecionadas:",
      settings.ballsPerThrow,
      "| Bolas que serão criadas:",
      count,
    );

    if (count === 0) return;

    for (let i = 0; i < count; i++) {
      const geom = new THREE.SphereGeometry(settings.ballSize, 32, 32);
      const mat = new THREE.MeshStandardMaterial({
        color: new THREE.Color(settings.ballColor),
        roughness: 0.12,
        metalness: 0.7,
        emissive: new THREE.Color(settings.ballColor),
        emissiveIntensity: 0.2,
      });

      const mesh = new THREE.Mesh(geom, mat);
      mesh.castShadow = true;
      mesh.receiveShadow = true;
      scene.add(mesh);

      // Spread spawn positions horizontally when multiple balls
      const spreadX =
        count > 1 ? (i / (count - 1) - 0.5) * 1.5 : (Math.random() - 0.5) * 0.2;

      // Start at floor level (y = radius), same as preview ball
      const initialPos = new THREE.Vector3(
        spreadX,
        settings.ballSize + Math.random() * 0.2,
        2.2,
      );

      // Aim at center of wall with slight random spread
      // Target a point at the middle height of the wall
      const wallHeight =
        wallConfig.rows * (wallConfig.brickH + wallConfig.spacing);
      const targetY =
        wallHeight * 0.45 + (Math.random() - 0.5) * wallHeight * 0.3;
      const targetX = spreadX * 0.1 + (Math.random() - 0.5) * 1.0;
      const targetZ = wallConfig.zPos;

      const dir = new THREE.Vector3(
        targetX - initialPos.x,
        targetY - initialPos.y,
        targetZ - initialPos.z,
      ).normalize();

      const velocity = dir.multiplyScalar(settings.throwForce);
      const ball = new Ball(mesh, initialPos, velocity);
      balls.push(ball);

      if (balls.length > MAX_BALLS) {
        const oldest = balls.shift();
        scene.remove(oldest.mesh);
      }
    }
  }

  // Click on the preview ball in 3D scene → throw!
  function onCanvasClick(event) {
    if (!previewBallMesh || !camera || !renderer) return;

    const rect = renderer.domElement.getBoundingClientRect();
    mouse.set(
      ((event.clientX - rect.left) / rect.width) * 2 - 1,
      -((event.clientY - rect.top) / rect.height) * 2 + 1,
    );

    raycaster.setFromCamera(mouse, camera);
    const hits = raycaster.intersectObject(previewBallMesh);
    if (hits.length > 0) {
      throwBall();
    }
  }

  // Spawns dust particles on impacts
  function spawnImpactParticles(position, colorHex, count = 12) {
    const pGroup = [];
    const geom = new THREE.BoxGeometry(0.04, 0.04, 0.04);
    const mat = new THREE.MeshBasicMaterial({
      color: colorHex,
      transparent: true,
      opacity: 0.8,
    });

    for (let i = 0; i < count; i++) {
      const pMesh = new THREE.Mesh(geom, mat.clone());
      pMesh.position.copy(position);
      scene.add(pMesh);

      const pVel = new THREE.Vector3(
        (Math.random() - 0.5) * 3,
        Math.random() * 3 + 1, // bounce up
        (Math.random() - 0.5) * 3,
      );

      pGroup.push({
        mesh: pMesh,
        vel: pVel,
        life: 0.6 + Math.random() * 0.4, // seconds
      });
    }

    particles = [...particles, ...pGroup];
  }

  // Custom Physics Collision Solver: Ball vs Bricks
  function checkBallBrickCollisions(ball, dt) {
    bricks.forEach((brick) => {
      // Skip check if the brick is blown far away
      if (!brick.isStatic && brick.pos.y < -5) return;

      // AABB sphere collision check
      // Find the closest point to the ball sphere center inside the brick AABB box
      const minX = brick.pos.x - brick.halfW;
      const maxX = brick.pos.x + brick.halfW;
      const minY = brick.pos.y - brick.halfH;
      const maxY = brick.pos.y + brick.halfH;
      const minZ = brick.pos.z - brick.halfD;
      const maxZ = brick.pos.z + brick.halfD;

      const closestX = Math.max(minX, Math.min(ball.pos.x, maxX));
      const closestY = Math.max(minY, Math.min(ball.pos.y, maxY));
      const closestZ = Math.max(minZ, Math.min(ball.pos.z, maxZ));

      // Distance from closest point to sphere center
      const diffX = ball.pos.x - closestX;
      const diffY = ball.pos.y - closestY;
      const diffZ = ball.pos.z - closestZ;
      const sqDist = diffX * diffX + diffY * diffY + diffZ * diffZ;

      if (sqDist < ball.radius * ball.radius) {
        // Collision detected!
        const dist = Math.sqrt(sqDist);

        // Calculate collision normal (pointing from brick to ball)
        const normal = new THREE.Vector3();
        if (dist > 0.001) {
          normal.set(diffX / dist, diffY / dist, diffZ / dist);
        } else {
          // If perfectly overlapping, default normal along Z axis
          normal.set(0, 0, 1);
        }

        // 1. Solve Penetration (Push sphere out of the AABB)
        const penetration = ball.radius - dist;
        ball.pos.addScaledVector(normal, penetration);

        // 2. Calculate impact speed along collision normal
        const relativeVel = new THREE.Vector3().copy(ball.vel);
        if (!brick.isStatic) {
          relativeVel.sub(brick.vel);
        }

        const impactSpeed = relativeVel.dot(normal);

        // Perform collision response only if moving towards each other
        if (impactSpeed < 0) {
          // Reflection vector: V_new = V - (1 + e) * (V . N) * N
          const impulse = -(1 + settings.restitution) * impactSpeed;
          ball.vel.addScaledVector(normal, impulse);

          // Apply slight squash/stretch effect
          ball.scale.set(1.2, 0.8, 1.2);

          // Trigger brick activation if breakable
          if (settings.breakable) {
            if (brick.isStatic) {
              brick.activate(ball.vel);
              collisionCount++;
              lastImpactSpeed = Math.abs(impactSpeed);
              dispatchTelemetry();
              // Spawn dust particles
              spawnImpactParticles(
                new THREE.Vector3(closestX, closestY, closestZ),
                settings.wallColor,
                6,
              );
            } else {
              // Push already-activated brick
              brick.vel.addScaledVector(normal, -impulse * 0.4);
              brick.rotVel.add(
                new THREE.Vector3(
                  Math.random() - 0.5,
                  Math.random() - 0.5,
                  Math.random() - 0.5,
                ).multiplyScalar(5),
              );
            }
          } else {
            // Indestructible wall impact
            collisionCount++;
            lastImpactSpeed = Math.abs(impactSpeed);
            dispatchTelemetry();
            spawnImpactParticles(
              new THREE.Vector3(closestX, closestY, closestZ),
              settings.wallColor,
              10,
            );
          }
        }
      }
    });
  }

  // Standard boundary limits (prevents balls leaving workspace)
  function checkBoundaryCollisions(ball) {
    const limits = { x: 5, zStart: 5, zEnd: -5 };

    // Side walls
    if (Math.abs(ball.pos.x) > limits.x - ball.radius) {
      ball.pos.x = Math.sign(ball.pos.x) * (limits.x - ball.radius);
      ball.vel.x = -ball.vel.x * settings.restitution;
    }

    // Front/Back barriers
    if (ball.pos.z > limits.zStart - ball.radius) {
      ball.pos.z = limits.zStart - ball.radius;
      ball.vel.z = -ball.vel.z * settings.restitution;
    }

    // If ball flies too deep past the wall
    if (ball.pos.z < limits.zEnd) {
      // Remove the ball to prevent resource leak
      const idx = balls.indexOf(ball);
      if (idx > -1) {
        balls.splice(idx, 1);
        scene.remove(ball.mesh);
      }
    }
  }

  // Update particles animation
  function updateParticles(dt) {
    const active = [];

    particles.forEach((p) => {
      p.life -= dt;
      if (p.life > 0) {
        // Gravity
        p.vel.y -= settings.gravity * dt;
        p.mesh.position.addScaledVector(p.vel, dt);

        // Fade out
        p.mesh.material.opacity = p.life;

        // Spin
        p.mesh.rotation.x += 2 * dt;
        p.mesh.rotation.y += 3 * dt;

        active.push(p);
      } else {
        scene.remove(p.mesh);
        p.mesh.geometry.dispose();
        p.mesh.material.dispose();
      }
    });

    particles = active;
  }

  function dispatchTelemetry() {
    dispatch("telemetry", {
      collisions: collisionCount,
      impactSpeed: lastImpactSpeed,
    });
  }

  // Animation Loop
  function animate() {
    if (isDestroyed) return;
    animationFrameId = requestAnimationFrame(animate);

    let deltaTime = clock.getDelta();
    if (deltaTime > 0.1) deltaTime = 0.1;
    if (deltaTime <= 0) deltaTime = 0.016;

    // 0. Pulse the preview ball
    if (previewBallMesh && previewBallLight) {
      previewPulseTime += deltaTime;
      const pulse = 0.88 + Math.sin(previewPulseTime * 2.8) * 0.12;
      previewBallMesh.scale.setScalar(pulse);
      previewBallMesh.material.emissiveIntensity =
        0.25 + Math.sin(previewPulseTime * 2.8) * 0.2;
      previewBallLight.intensity = 0.8 + Math.sin(previewPulseTime * 2.8) * 0.5;
    }

    // 1. Update balls physics and collisions
    balls.forEach((ball) => {
      ball.update(deltaTime);
      checkBallBrickCollisions(ball, deltaTime);
      checkBoundaryCollisions(ball);
    });

    // 2. Update dynamic bricks physics
    bricks.forEach((brick) => {
      brick.update(deltaTime);
    });

    // 3. Update active dust particles
    updateParticles(deltaTime);

    // 4. Controls update & rendering
    controls.update();
    renderer.render(scene, camera);
  }

  function initTweakpane() {
    pane = new Pane({
      container: paneContainer,
      title: "Configurações de Física",
    });

    pane.addButton({ title: "🏀 LANÇAR BOLA(S)" }).on("click", () => {
      throwBall();
    });

    pane.addButton({ title: "🧱 Reconstruir Muro" }).on("click", () => {
      resetScene();
    });

    pane.addBinding(settings, "ballsPerThrow", {
      label: "Bolas por Lançamento",
      min: 1,
      max: 5,
      step: 1,
    });

    pane.addBinding(settings, "throwForce", {
      label: "Força do Lançamento",
      min: 5.0,
      max: 30.0,
      step: 1.0,
    });

    pane.addBinding(settings, "gravity", {
      label: "Gravidade (m/s²)",
      min: 0,
      max: 25,
      step: 0.1,
    });

    pane.addBinding(settings, "restitution", {
      label: "Elasticidade",
      min: 0.1,
      max: 1.0,
      step: 0.05,
    });

    pane
      .addBinding(settings, "ballSize", {
        label: "Tamanho da Bola",
        min: 0.1,
        max: 0.6,
        step: 0.02,
      })
      .on("change", (ev) => {
        // Update existing balls
        balls.forEach((b) => {
          b.radius = ev.value;
        });
        // Rebuild preview ball with new size
        buildPreviewBall();
      });

    pane
      .addBinding(settings, "breakable", { label: "Parede Quebrável" })
      .on("change", () => {
        resetScene();
      });

    const folderColors = pane.addFolder({ title: "Estilo Visual" });

    folderColors
      .addBinding(settings, "wallColor", { label: "Cor dos Tijolos" })
      .on("change", (ev) => {
        bricks.forEach((b) => b.mesh.material.color.set(ev.value));
      });

    folderColors
      .addBinding(settings, "ballColor", { label: "Cor da Bola" })
      .on("change", (ev) => {
        balls.forEach((b) => b.mesh.material.color.set(ev.value));
        // Update preview ball color too
        if (previewBallMesh) {
          previewBallMesh.material.color.set(ev.value);
          previewBallMesh.material.emissive.set(ev.value);
        }
        if (previewBallLight) {
          previewBallLight.color.set(ev.value);
        }
      });
  }

  onMount(() => {
    // 1. Scene Setup
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0b10);
    scene.fog = new THREE.FogExp2(0x0a0b10, 0.06);

    // 2. Camera Setup
    camera = new THREE.PerspectiveCamera(
      45,
      canvasContainer.clientWidth / canvasContainer.clientHeight,
      0.1,
      100,
    );
    camera.position.set(3.5, 2.8, 6.0); // pulled back to frame the larger wall

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
    controls.maxPolarAngle = Math.PI / 2 - 0.02; // prevent camera looking below floor
    controls.minDistance = 2.0;
    controls.maxDistance = 10;
    controls.target.set(0, 1.2, -0.8);

    // 5. Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.15);
    scene.add(ambientLight);

    // Soft overhead light
    const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
    dirLight.position.set(2, 5, 2);
    dirLight.castShadow = true;
    dirLight.shadow.mapSize.width = 1024;
    dirLight.shadow.mapSize.height = 1024;
    dirLight.shadow.bias = -0.0005;
    scene.add(dirLight);

    // Cyan Neon Accent Light pointing from left side
    const cyanLight = new THREE.DirectionalLight(0x00f2fe, 1.5);
    cyanLight.position.set(-4, 2, -1);
    scene.add(cyanLight);

    // Violet Spotlight shining directly on the wall
    const spotLight = new THREE.SpotLight(0xd946ef, 6, 12, Math.PI / 4, 0.2, 1);
    spotLight.position.set(0, 4, 2);
    spotLight.target.position.set(0, 0.8, -1.5);
    scene.add(spotLight);
    scene.add(spotLight.target);

    // 6. Grid floor
    const floorSize = 40;
    const gridHelper = new THREE.GridHelper(floorSize, 40, 0xd946ef, 0x1f2937);
    gridHelper.position.y = 0;
    gridHelper.material.opacity = 0.2;
    gridHelper.material.transparent = true;
    scene.add(gridHelper);

    // Floor shadow catcher
    const floorGeom = new THREE.PlaneGeometry(floorSize, floorSize);
    const floorMat = new THREE.ShadowMaterial({ opacity: 0.35 });
    const floor = new THREE.Mesh(floorGeom, floorMat);
    floor.rotation.x = -Math.PI / 2;
    floor.position.y = 0;
    floor.receiveShadow = true;
    scene.add(floor);

    // 7. Assemble the wall
    buildWall();

    // 7b. Build the preview ball (sits on floor)
    buildPreviewBall();

    // 8. Event listeners & panel
    window.addEventListener("resize", handleResize);
    // Click on the 3D preview ball to throw
    renderer.domElement.addEventListener("click", onCanvasClick);
    initTweakpane();

    // 9. Start loop
    animate();
    dispatchTelemetry();
  });

  function handleResize() {
    if (!canvasContainer || !camera || !renderer) return;
    camera.aspect = canvasContainer.clientWidth / canvasContainer.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(canvasContainer.clientWidth, canvasContainer.clientHeight);
  }

  onDestroy(() => {
    isDestroyed = true;
    window.removeEventListener("resize", handleResize);
    if (renderer)
      renderer.domElement.removeEventListener("click", onCanvasClick);
    cancelAnimationFrame(animationFrameId);

    if (pane) pane.dispose();
    if (renderer) renderer.dispose();

    scene?.traverse((object) => {
      if (object.geometry) object.geometry.dispose();
      if (object.material) {
        if (Array.isArray(object.material)) {
          object.material.forEach((mat) => mat.dispose());
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
