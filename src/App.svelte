<script>
  import Navbar from './lib/components/Navbar.svelte';
  import UIOverlay from './lib/components/UIOverlay.svelte';
  import FanScene from './lib/components/FanScene.svelte';
  import BallScene from './lib/components/BallScene.svelte';

  let activeScene = 'fan'; // 'fan' or 'ball'

  // Fan scene telemetry (leitura — vem da FanScene)
  let fanSpeed = 0;
  let fanPower = true;
  let fanBlades = 3;

  // Controles do usuário (escrita — vão para a FanScene)
  let controlPower = true;
  let controlSpeed = 8.0;

  // Ball scene telemetry
  let ballCollisions = 0;
  let lastImpactSpeed = 0;

  // Handle scene change
  function handleSceneSelect(event) {
    activeScene = event.detail.scene;
  }

  // Handle telemetry updates
  function handleFanTelemetry(event) {
    fanSpeed = event.detail.speed;
    fanPower = event.detail.power;
    fanBlades = event.detail.blades;
  }

  function handleBallTelemetry(event) {
    ballCollisions = event.detail.collisions;
    lastImpactSpeed = event.detail.impactSpeed;
  }

  // Recebe controles do UIOverlay e repassa para a FanScene
  function handleFanControl(event) {
    controlPower = event.detail.power;
    controlSpeed = event.detail.speed;
  }
</script>

<div class="ui-container">
  <!-- Top Navbar -->
  <Navbar {activeScene} on:select={handleSceneSelect} />

  <!-- Left Sidebar Info Card -->
  <UIOverlay
    {activeScene}
    {fanSpeed}
    {fanPower}
    {fanBlades}
    {ballCollisions}
    {lastImpactSpeed}
    {controlPower}
    {controlSpeed}
    on:fanControl={handleFanControl}
  />
</div>

<!-- Active 3D Scene Viewport -->
{#if activeScene === 'fan'}
  <FanScene
    {controlPower}
    {controlSpeed}
    on:telemetry={handleFanTelemetry}
  />
{:else if activeScene === 'ball'}
  <BallScene on:telemetry={handleBallTelemetry} />
{/if}

<style>
  /* Local layout overrides can go here if needed */
</style>
