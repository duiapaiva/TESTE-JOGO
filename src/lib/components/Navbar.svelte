<script>
  import { createEventDispatcher } from 'svelte';
  import { Wind, CircleDot } from 'lucide-svelte';

  export let activeScene = 'fan'; // 'fan' or 'ball'

  const dispatch = createEventDispatcher();

  function selectScene(scene) {
    activeScene = scene;
    dispatch('select', { scene });
  }
</script>

<header class="glass-panel navbar ui-element">
  <div class="brand">
    <div class="glow-indicator"></div>
    <span class="logo-text">THREE.JS LAB</span>
  </div>

  <div class="tabs">
    <button 
      class="tab-btn {activeScene === 'fan' ? 'active' : ''}" 
      on:click={() => selectScene('fan')}
    >
      <Wind size={18} class="icon {activeScene === 'fan' ? 'spinning' : ''}" />
      <span>Ventilador 3D</span>
    </button>

    <button 
      class="tab-btn {activeScene === 'ball' ? 'active' : ''}" 
      on:click={() => selectScene('ball')}
    >
      <CircleDot size={18} class="icon {activeScene === 'ball' ? 'pulsing' : ''}" />
      <span>Bola e Muro</span>
    </button>
  </div>
</header>

<style>
  .navbar {
    grid-column: 1 / span 3;
    grid-row: 1;
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0 24px;
    height: 100%;
    pointer-events: auto;
    border-radius: 12px;
  }

  .brand {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .glow-indicator {
    width: 10px;
    height: 10px;
    background-color: var(--primary);
    border-radius: 50%;
    box-shadow: var(--glow-shadow);
    animation: pulse-cyan 2s infinite ease-in-out;
  }

  .logo-text {
    font-size: 1.15rem;
    font-weight: 700;
    letter-spacing: 2px;
    background: linear-gradient(90deg, var(--primary), var(--secondary));
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
  }

  .tabs {
    display: flex;
    gap: 12px;
    background: rgba(0, 0, 0, 0.25);
    padding: 4px;
    border-radius: 30px;
    border: 1px solid rgba(255, 255, 255, 0.05);
  }

  .tab-btn {
    display: flex;
    align-items: center;
    gap: 8px;
    background: transparent;
    border: none;
    color: var(--text-muted);
    padding: 8px 18px;
    border-radius: 25px;
    font-family: var(--font-sans);
    font-size: 0.9rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }

  .tab-btn:hover {
    color: var(--text-main);
    background: rgba(255, 255, 255, 0.04);
  }

  .tab-btn.active {
    color: #000;
    background: linear-gradient(135deg, var(--primary), var(--secondary));
    box-shadow: var(--glow-shadow);
    font-weight: 600;
  }

  :global(.icon.spinning) {
    animation: spin-slow 8s infinite linear;
  }

  :global(.icon.pulsing) {
    animation: pulse-cyan 1.5s infinite ease-in-out;
  }

  @keyframes spin-slow {
    from { transform: rotate(0deg); }
    to { transform: rotate(360deg); }
  }
</style>
