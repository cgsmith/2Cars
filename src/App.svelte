<script>
  import { onMount } from 'svelte';

  let gameLoop;
  let score = 0;
  let bestScore = 0;
  let gameOver = false;
  let gameStartTime = Date.now();
  let elapsedTime = 0;
  let lastFrameTime = performance.now();
  let isPaused = false;
  let gameHeight;
  let gameContainer;

  $: speed = Math.min(7, 3 + (score * 0.05) + (elapsedTime / 20000));
  $: speedDisplay = speed.toFixed(1);

  // Initialize best score and get container height
  onMount(() => {
    bestScore = parseInt(localStorage.getItem('bestScore')) || 0;

    const updateHeight = () => {
      if (gameContainer) {
        // Get the viewport height and set game height
        const vh = window.innerHeight;
        // Leave some space for browser UI, use 90% of viewport
        gameHeight = Math.min(600, vh * 0.9);
        gameContainer.style.height = `${gameHeight}px`;
      }
    };

    updateHeight();
    window.addEventListener('resize', updateHeight);
    return () => window.removeEventListener('resize', updateHeight);
  });

  // Keyboard controls
  onMount(() => {
    window.addEventListener('keydown', handleKeydown);
    return () => window.removeEventListener('keydown', handleKeydown);
  });

  // Car state management
  const INITIAL_POSITIONS = {
    left: { start: 70, alt: 131 },
    right: { start: 220, alt: 282 }
  };

  let cars = {
    left: { lane: 'left', y: 0, visualX: 70 },
    right: { lane: 'left', y: 0, visualX: 220 }
  };

  // Reactive car positions
  $: {
    const targetLeftX = cars.left.lane === 'left' ? INITIAL_POSITIONS.left.start : INITIAL_POSITIONS.left.alt;
    const targetRightX = cars.right.lane === 'left' ? INITIAL_POSITIONS.right.start : INITIAL_POSITIONS.right.alt;
    cars.left.x = targetLeftX;
    cars.right.x = targetRightX;
    // Position cars relative to game height
    if (gameHeight) {
      cars.left.y = gameHeight - 100;
      cars.right.y = gameHeight - 100;
    }
  }

  let obstacles = [];

  // Handle car movement
  const toggleCarLane = (side) => (event) => {
    event.preventDefault();
    if (!isPaused) {
      cars[side].lane = cars[side].lane === 'left' ? 'right' : 'left';
    }
  };

  // Handle keyboard controls
  const handleKeydown = (event) => {
    if (event.key === 's') {
      if (!isPaused) cars.left.lane = cars.left.lane === 'left' ? 'right' : 'left';
    } else if (event.key === 'k') {
      if (!isPaused) cars.right.lane = cars.right.lane === 'left' ? 'right' : 'left';
    } else if (event.key === 'p') {
      togglePause();
    }
  }

  // Obstacle management
  $: activeObstacles = obstacles.filter(obs => obs.y < gameHeight);
  $: canSpawnNewObstacle = !obstacles.some(obs => obs.y < 100);

  const spawnObstacle = () => {
    if (!canSpawnNewObstacle) return;

    const baseSpawnChance = 0.02;
    const speedMultiplier = (speed - 3) * 0.004;
    const spawnChance = baseSpawnChance + speedMultiplier;

    if (Math.random() > spawnChance) return;

    const side = Math.random() < 0.5 ? 'left' : 'right';
    const type = Math.random() < 0.5 ? 'circle' : 'square';
    const lane = Math.random() < 0.5 ? 'left' : 'right';

    if (obstacles.some(obs => obs.side === side && Math.abs(obs.y) < 150)) return;

    const positions = INITIAL_POSITIONS[side];
    const x = lane === 'left' ? positions.start : positions.alt;

    obstacles = [...obstacles, { x, y: -20, type, side }];
  }

  const checkCollision = (car, obstacle) => Math.hypot(car.x - obstacle.x, car.y - obstacle.y) < 30;

  let roadDashOffset = 0;

  const togglePause = () => {
    isPaused = !isPaused;
    if (!isPaused) {
      lastFrameTime = performance.now();
      requestAnimationFrame(update);
    }
  };

  const update = (timestamp) => {
    if (gameOver || isPaused) return;

    // Calculate delta time for smooth animations
    const deltaTime = timestamp - lastFrameTime;
    lastFrameTime = timestamp;

    elapsedTime = Date.now() - gameStartTime;
    roadDashOffset -= speed * (deltaTime / 16);

    obstacles = obstacles.filter(obs => {
      obs.y += speed * (deltaTime / 16);
      const relevantCar = cars[obs.side];
      if (checkCollision(relevantCar, obs)) {
        if (obs.type === 'square') {
          if (score > bestScore) {
            bestScore = score;
            localStorage.setItem('bestScore', bestScore);
          }
          gameOver = true;
        } else {
          score++;
          return false;
        }
      }
      if (obs.y >= gameHeight && obs.type === 'circle') {
        if (score > bestScore) {
          bestScore = score;
          localStorage.setItem('bestScore', bestScore);
        }
        gameOver = true;
      }
      return obs.y < gameHeight;
    });
    spawnObstacle();

    if (!gameOver) {
      requestAnimationFrame(update);
    }
  }

  const restartGame = () => {
    lastFrameTime = performance.now();
    score = 0;
    gameOver = false;
    obstacles = [];
    gameStartTime = Date.now();
    elapsedTime = 0;
    roadDashOffset = 0;
    isPaused = false;
    cars = {
      left: { lane: 'left', y: gameHeight - 100, visualX: 70 },
      right: { lane: 'left', y: gameHeight - 100, visualX: 220 }
    };
    requestAnimationFrame(update);
  }

  onMount(() => {
    requestAnimationFrame(update);
    return () => { gameOver = true; };
  });
</script>

<div class="game-container" bind:this={gameContainer}>
  <button class="pause-button" on:click={togglePause}>
    {isPaused ? '▶' : '❚ ❚'}
  </button>

  <div class="absolute score-container">
    <div class="score">{score} / {speedDisplay}</div>
  </div>

  {#if gameOver}
    <div class="absolute overlay"></div>
    <div class="absolute game-over">
      GAME OVER
      <div class="final-scores">
        <div>Score: {score}</div>
        <div>Best: {bestScore}</div>
      </div>
      <button class="restart-button" on:click={restartGame}>Restart</button>
    </div>
  {/if}

  <svg width="400" height={gameHeight}>
    <!-- Lanes -->
    {#each ['left', 'right'] as side}
    <rect x={side === 'left' ? 40 : 190} y="0" width="120" height={gameHeight} fill="#253479" />
    <!-- Left border -->
      <line x1={side === 'left' ? 40 : 190} y1="0" x2={side === 'left' ? 40 : 190} y2={gameHeight} stroke="#8297ee" stroke-width="2" />
      <!-- Right border -->
      <line x1={side === 'left' ? 160 : 310} y1="0" x2={side === 'left' ? 160 : 310} y2={gameHeight} stroke="#8297ee" stroke-width="2" />
      <line x1={side === 'left' ? 100 : 250} y1="0" x2={side === 'left' ? 100 : 250} y2={gameHeight} stroke="#7a95ec" stroke-dasharray="20,20" stroke-dashoffset={roadDashOffset} stroke-width="2" />
    {/each}

    <!-- Cars -->
    {#each ['left', 'right'] as side}
      <g>
        <!-- Wheels -->
        {#each [[-15, -15], [15, -15], [-15, 15], [15, 15]] as [xOffset, yOffset]}
          <circle
            class="car"
            cx={cars[side].x + xOffset}
            cy={cars[side].y + yOffset}
            r="5"
            fill="#000"
          />
        {/each}
        <!-- Car body -->
        <rect class="car" x={cars[side].x - 15} y={cars[side].y - 25} width="30" height="50" rx="5" ry="5" fill={side === 'left' ? '#f3416b' : '#1eadc2'} />
        <!-- Front windshield -->
        <rect class="car" x={cars[side].x - 10} y={cars[side].y - 20} width="20" height="15" rx="2" ry="2" fill="#333" opacity="0.7" />
        <!-- Back windshield -->
        <rect class="car" x={cars[side].x - 10} y={cars[side].y + 5} width="20" height="15" rx="2" ry="2" fill="#333" opacity="0.7" />
      </g>
    {/each}

    <!-- Obstacles -->
    {#each obstacles as obstacle}
      {#if obstacle.type === 'circle'}
        <g>
          <circle cx={obstacle.x} cy={obstacle.y} r="15" fill={obstacle.side === 'left' ? '#f23a63' : '#05aac1'} />
          <circle cx={obstacle.x} cy={obstacle.y} r="13" fill="none" stroke="white" stroke-width="2" />
        </g>
      {:else}
        <g>
          <rect x={obstacle.x - 15} y={obstacle.y - 15} width="30" height="30" rx="3" ry="3" fill={obstacle.side === 'left' ? '#f23a63' : '#05aac1'} />
          <rect x={obstacle.x - 13} y={obstacle.y - 13} width="26" height="26" rx="2" ry="2" fill="none" stroke="white" stroke-width="2" />
        </g>
      {/if}
    {/each}
  </svg>

  <!-- Touch areas -->
  <div class="absolute touch-left" on:mousedown={toggleCarLane('left')} on:touchstart={toggleCarLane('left')}></div>
  <div class="absolute touch-right" on:mousedown={toggleCarLane('right')} on:touchstart={toggleCarLane('right')}></div>
</div>

<style>
  .absolute { position: absolute; }
  .game-container {
    position: relative;
    width: 350px;
    margin: 0 auto;
    overflow: hidden;
    background-color: #1a1b26;
  }
  .score-container {
    top: 20px;
    right: 20px;
    z-index: 6;
    color: white;
    font-size: 24px;
    text-shadow: 2px 2px 2px rgba(0,0,0,0.5);
  }
  .pause-button {
    position: absolute;
    top: 20px;
    left: 20px;
    z-index: 999;
    padding: 8px;
    font-size: 20px;
    background: transparent;
    color: white;
    border: none;
  }
  .overlay {
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.7);
    z-index: 10;
  }
  .game-over {
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 30px;
    color: white;
    font-weight: bold;
    text-align: center;
    z-index: 9999;
  }
  .final-scores { margin: 15px 0; font-size: 24px; font-weight: normal; }
  .restart-button {
    margin: 20px auto;
    padding: 10px 20px;
    font-size: 24px;
    background-color: #4CAF50;
    color: white;
    border: none;
    border-radius: 5px;
  }
  .car { transition: x 0.2s ease-out, cx 0.2s ease-out; }
  .touch-left, .touch-right { top: 0; width: 50%; height: 100%; z-index: 5; }
  .touch-left { left: 0; }
  .touch-right { right: 0; }
</style>
