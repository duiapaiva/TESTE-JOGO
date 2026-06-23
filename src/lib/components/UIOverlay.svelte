<script>
  import { Info, Sparkles, Activity } from 'lucide-svelte';
  import { createEventDispatcher } from 'svelte';

  const dispatch = createEventDispatcher();

  export let activeScene = 'fan';
  export let fanSpeed = 0;       // telemetria (velocidade atual real)
  export let fanPower = true;    // telemetria (estado atual real)
  export let fanBlades = 3;      // telemetria
  export let ballCollisions = 0;
  export let lastImpactSpeed = 0;

  // Controles locais — valores que o usuário define (Fan)
  export let controlPower = true;
  export let controlSpeed = 8.0;

  // Controles locais — valores que o usuário define (Ball)
  export let controlBallCount = 1;

  const SPEED_STEP = 1;
  const SPEED_MIN  = 0;
  const SPEED_MAX  = 20;

  const BALL_MIN = 1;
  const BALL_MAX = 5;

  $: rpm = fanPower ? Math.round(fanSpeed * 135) : 0;

  function togglePower() {
    controlPower = !controlPower;
    dispatch('fanControl', { power: controlPower, speed: controlSpeed });
  }

  function decreaseSpeed() {
    controlSpeed = Math.max(SPEED_MIN, +(controlSpeed - SPEED_STEP).toFixed(1));
    dispatch('fanControl', { power: controlPower, speed: controlSpeed });
  }

  function increaseSpeed() {
    controlSpeed = Math.min(SPEED_MAX, +(controlSpeed + SPEED_STEP).toFixed(1));
    dispatch('fanControl', { power: controlPower, speed: controlSpeed });
  }

  function decreaseBallCount() {
    controlBallCount = Math.max(BALL_MIN, controlBallCount - 1);
    dispatch('ballControl', { ballsPerThrow: controlBallCount });
  }

  function increaseBallCount() {
    controlBallCount = Math.min(BALL_MAX, controlBallCount + 1);
    dispatch('ballControl', { ballsPerThrow: controlBallCount });
  }
</script>

<aside class="glass-panel overlay-panel ui-element">
  {#if activeScene === 'fan'}
    <div class="panel-section">
      <div class="header-with-icon">
        <span class="accent-color"><Sparkles size={20} /></span>
        <h2>Ventilador Dinâmico</h2>
      </div>
      <p class="description">
        Explore a simulação tridimensional de um ventilador físico. Modifique parâmetros em tempo real como velocidade, número de pás e oscilação.
      </p>
    </div>

    <hr class="divider" />

    <div class="panel-section">
      <div class="header-with-icon">
        <span class="cyan-color"><Activity size={18} /></span>
        <h3>Telemetria em Tempo Real</h3>
      </div>
      <!-- Estado — toggle de energia -->
      <div class="control-card">
        <span class="stat-label">Estado</span>
        <div class="control-row">
          <span class="stat-value {controlPower ? 'active-green' : 'inactive-red'}">
            {controlPower ? 'LIGADO' : 'DESLIGADO'}
          </span>
          <button
            class="ctrl-btn power-btn {controlPower ? 'btn-on' : 'btn-off'}"
            on:click={togglePower}
            title="Ligar / Desligar"
          >
            {controlPower ? '⏸' : '▶'}
          </button>
        </div>
      </div>

      <!-- Rotação — controle de velocidade -->
      <div class="control-card">
        <span class="stat-label">Velocidade</span>
        <div class="control-row">
          <button class="ctrl-btn minus-btn" on:click={decreaseSpeed} disabled={controlSpeed <= 0} title="Diminuir">−</button>
          <div class="speed-display">
            <span class="stat-value cyan-text">{controlSpeed}</span>
            <span class="speed-unit">{rpm} RPM</span>
          </div>
          <button class="ctrl-btn plus-btn" on:click={increaseSpeed} disabled={controlSpeed >= 20} title="Aumentar">+</button>
        </div>
      </div>

      <!-- Leitura somente -->
      <div class="stats-grid">
        <div class="stat-card">
          <span class="stat-label">Nº de Pás</span>
          <span class="stat-value">{fanBlades}</span>
        </div>
        <div class="stat-card">
          <span class="stat-label">Vel. Atual</span>
          <span class="stat-value">{fanSpeed.toFixed(1)} m/s</span>
        </div>
      </div>
    </div>
  {:else}
    <div class="panel-section">
      <div class="header-with-icon">
        <span class="accent-color"><Sparkles size={20} /></span>
        <h2>Colisão & Física</h2>
      </div>
      <p class="description">
        Simule o lançamento de projéteis elásticos contra uma parede estruturada. Ajuste a gravidade e o coeficiente de restituição para ver a conservação de energia.
      </p>
    </div>

    <hr class="divider" />

    <div class="panel-section">
      <div class="header-with-icon">
        <span class="magenta-color"><Activity size={18} /></span>
        <h3>Controles de Lançamento</h3>
      </div>

      <!-- Quantidade de bolas por lançamento -->
      <div class="control-card">
        <span class="stat-label">Bolas por lançamento</span>
        <div class="control-row">
          <button class="ctrl-btn minus-btn" on:click={decreaseBallCount} disabled={controlBallCount <= 1} title="Menos bolas">−</button>
          <div class="speed-display">
            <span class="stat-value magenta-text">{controlBallCount}</span>
            <span class="speed-unit">{controlBallCount === 1 ? 'bola' : 'bolas'}</span>
          </div>
          <button class="ctrl-btn plus-btn" on:click={increaseBallCount} disabled={controlBallCount >= 5} title="Mais bolas">+</button>
        </div>
      </div>

      <!-- Stats somente leitura -->
      <div class="stats-grid">
        <div class="stat-card">
          <span class="stat-label">Impactos na Parede</span>
          <span class="stat-value magenta-text">{ballCollisions}</span>
        </div>
        <div class="stat-card">
          <span class="stat-label">Vel. Último Impacto</span>
          <span class="stat-value">{lastImpactSpeed > 0 ? `${lastImpactSpeed.toFixed(1)} m/s` : 'N/A'}</span>
        </div>
      </div>

      <!-- Dica de interação -->
      <div class="info-badge">
        <Info size={13} />
        <span>Clique na bola amarela para arremessar!</span>
      </div>
    </div>
  {/if}

  <hr class="divider" />

  <div class="panel-section footer">
    <div class="info-badge">
      <Info size={14} />
      <span>Use o mouse para rotacionar e zoom</span>
    </div>
  </div>
</aside>

<style>
  .overlay-panel {
    grid-column: 1;
    grid-row: 2;
    display: flex;
    flex-direction: column;
    padding: 24px;
    height: 100%;
    color: var(--text-main);
  }

  .panel-section {
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .header-with-icon {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  h2 {
    font-size: 1.4rem;
    font-weight: 600;
    letter-spacing: -0.5px;
    color: var(--text-main);
  }

  h3 {
    font-size: 1rem;
    font-weight: 600;
    color: var(--text-muted);
    text-transform: uppercase;
    letter-spacing: 1px;
  }

  .description {
    font-size: 0.9rem;
    line-height: 1.5;
    color: var(--text-muted);
  }

  .divider {
    border: none;
    border-top: 1px solid rgba(255, 255, 255, 0.08);
    margin: 20px 0;
  }

  /* ── Grids e Cards ─────────────────────────────────── */
  .stats-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 10px;
  }

  .stat-card {
    background: rgba(0, 0, 0, 0.2);
    border: 1px solid rgba(255, 255, 255, 0.04);
    padding: 12px;
    border-radius: 8px;
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  /* Card interativo (ocupa linha inteira) */
  .control-card {
    background: rgba(0, 0, 0, 0.25);
    border: 1px solid rgba(255, 255, 255, 0.08);
    padding: 12px 14px;
    border-radius: 10px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .control-row {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  /* ── Labels e Valores ───────────────────────────────── */
  .stat-label {
    font-size: 0.72rem;
    color: var(--text-muted);
    text-transform: uppercase;
    letter-spacing: 0.6px;
  }

  .stat-value {
    font-size: 1.15rem;
    font-weight: 700;
    font-family: var(--font-mono);
    flex: 1;
  }

  .speed-display {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1px;
  }

  .speed-unit {
    font-size: 0.7rem;
    color: var(--text-muted);
    font-family: var(--font-mono);
  }

  /* ── Botões de Controle ─────────────────────────────── */
  .ctrl-btn {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 36px;
    height: 36px;
    border-radius: 8px;
    border: 1px solid rgba(255, 255, 255, 0.12);
    background: rgba(255, 255, 255, 0.06);
    color: var(--text-main);
    font-size: 1.2rem;
    font-weight: 700;
    cursor: pointer;
    transition: background 0.15s, border-color 0.15s, transform 0.1s;
    line-height: 1;
    padding: 0;
    flex-shrink: 0;
  }

  .ctrl-btn:hover:not(:disabled) {
    background: rgba(255, 255, 255, 0.14);
    border-color: rgba(255, 255, 255, 0.25);
    transform: scale(1.08);
  }

  .ctrl-btn:active:not(:disabled) {
    transform: scale(0.95);
  }

  .ctrl-btn:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }

  /* Botão + em ciano */
  .plus-btn {
    border-color: rgba(0, 242, 254, 0.35);
    color: var(--primary);
  }
  .plus-btn:hover:not(:disabled) {
    background: rgba(0, 242, 254, 0.12);
    border-color: rgba(0, 242, 254, 0.6);
    box-shadow: 0 0 10px rgba(0, 242, 254, 0.2);
  }

  /* Botão − em magenta */
  .minus-btn {
    border-color: rgba(217, 70, 239, 0.35);
    color: var(--accent);
  }
  .minus-btn:hover:not(:disabled) {
    background: rgba(217, 70, 239, 0.12);
    border-color: rgba(217, 70, 239, 0.6);
    box-shadow: 0 0 10px rgba(217, 70, 239, 0.2);
  }

  /* Botão de Power */
  .power-btn {
    width: 42px;
    height: 42px;
    border-radius: 50%;
    font-size: 1rem;
  }
  .btn-on {
    border-color: rgba(74, 222, 128, 0.5);
    color: #4ade80;
  }
  .btn-on:hover {
    background: rgba(74, 222, 128, 0.1);
    box-shadow: 0 0 12px rgba(74, 222, 128, 0.25);
  }
  .btn-off {
    border-color: rgba(248, 113, 113, 0.5);
    color: #f87171;
  }
  .btn-off:hover {
    background: rgba(248, 113, 113, 0.1);
    box-shadow: 0 0 12px rgba(248, 113, 113, 0.25);
  }

  /* ── Cores de Texto ─────────────────────────────────── */
  .active-green  { color: #4ade80; }
  .inactive-red  { color: #f87171; }
  .cyan-text     { color: var(--primary); }
  .magenta-text  { color: var(--accent); }

  .accent-color  { color: var(--primary); display: inline-flex; }
  .cyan-color    { color: var(--primary); display: inline-flex; }
  .magenta-color { color: var(--accent);  display: inline-flex; }

  .footer {
    margin-top: auto;
  }

  .info-badge {
    display: flex;
    align-items: center;
    gap: 8px;
    background: rgba(0, 242, 254, 0.05);
    border: 1px dashed rgba(0, 242, 254, 0.15);
    padding: 8px 12px;
    border-radius: 8px;
    font-size: 0.8rem;
    color: var(--primary);
  }
</style>
