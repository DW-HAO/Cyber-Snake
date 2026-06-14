<script setup lang="ts">
import {
  Activity,
  Gauge,
  Pause,
  Play,
  RotateCcw,
  Sparkles,
  Trophy,
  UserRound,
} from 'lucide-vue-next';
import { computed, nextTick, onMounted, onUnmounted, ref } from 'vue';

type GameState = 'ready' | 'running' | 'paused' | 'over';

type Point = {
  x: number;
  y: number;
};

type Direction = Point & {
  name: 'up' | 'down' | 'left' | 'right';
};

type RenderPoint = Point & {
  radius: number;
};

const config = {
  boardWidth: 28,
  boardHeight: 28,
  initialTickMs: 155,
  minTickMs: 62,
  speedupEveryPoints: 50,
  speedupStepMs: 7,
  pointsPerFood: 10,
};

const directions: Record<string, Direction> = {
  ArrowUp: { x: 0, y: -1, name: 'up' },
  ArrowDown: { x: 0, y: 1, name: 'down' },
  ArrowLeft: { x: -1, y: 0, name: 'left' },
  ArrowRight: { x: 1, y: 0, name: 'right' },
  w: { x: 0, y: -1, name: 'up' },
  s: { x: 0, y: 1, name: 'down' },
  a: { x: -1, y: 0, name: 'left' },
  d: { x: 1, y: 0, name: 'right' },
};

const canvasRef = ref<HTMLCanvasElement | null>(null);
const snake = ref<Point[]>([]);
const previousSnake = ref<Point[]>([]);
const food = ref<Point>({ x: 10, y: 10 });
const direction = ref<Direction>({ x: 1, y: 0, name: 'right' });
const queuedDirection = ref<Direction>({ x: 1, y: 0, name: 'right' });
const gameState = ref<GameState>('ready');
const score = ref(0);
const tickMs = ref(config.initialTickMs);
const elapsedMs = ref(0);
const playerName = ref(localStorage.getItem('cyber-snake-player') || 'NEON');
const highScore = ref(Number(localStorage.getItem('cyber-snake-high-score') || '0'));

let animationFrame = 0;
let lastTickAt = 0;
let lastMoveAt = 0;
let runStartedAt = 0;
let accumulatedMs = 0;

const overlayTitle = computed(() => {
  if (gameState.value === 'over') return '连接断开';
  if (gameState.value === 'paused') return '脉冲冻结';
  return '霓虹核心待命';
});

const overlayKicker = computed(() => {
  if (gameState.value === 'over') return `最终分数 ${score.value}`;
  if (gameState.value === 'paused') return 'PAUSE SIGNAL';
  return 'CYBER SNAKE';
});

const primaryActionLabel = computed(() => {
  if (gameState.value === 'paused') return '继续';
  if (gameState.value === 'over') return '再来一局';
  return '开始';
});

const currentSpeed = computed(() => `${Math.round(1000 / tickMs.value)} Hz`);
const snakeLength = computed(() => snake.value.length);
const localBest = computed(() => Math.max(highScore.value, score.value));

function normalizeName() {
  const cleaned = playerName.value.trim().replace(/\s+/g, ' ').slice(0, 16);
  playerName.value = cleaned || 'NEON';
  localStorage.setItem('cyber-snake-player', playerName.value);
  return playerName.value;
}

function cloneSnake(source = snake.value) {
  return source.map((segment) => ({ ...segment }));
}

function createInitialSnake() {
  const centerX = Math.floor(config.boardWidth / 2);
  const centerY = Math.floor(config.boardHeight / 2);
  return [
    { x: centerX, y: centerY },
    { x: centerX - 1, y: centerY },
    { x: centerX - 2, y: centerY },
    { x: centerX - 3, y: centerY },
  ];
}

function samePoint(a: Point, b: Point) {
  return a.x === b.x && a.y === b.y;
}

function randomFood(occupied: Point[] = snake.value) {
  const occupiedKeys = new Set(occupied.map((point) => `${point.x}:${point.y}`));
  const totalCells = config.boardWidth * config.boardHeight;

  for (let attempt = 0; attempt < totalCells; attempt += 1) {
    const point = {
      x: Math.floor(Math.random() * config.boardWidth),
      y: Math.floor(Math.random() * config.boardHeight),
    };
    if (!occupiedKeys.has(`${point.x}:${point.y}`)) return point;
  }

  for (let y = 0; y < config.boardHeight; y += 1) {
    for (let x = 0; x < config.boardWidth; x += 1) {
      if (!occupiedKeys.has(`${x}:${y}`)) return { x, y };
    }
  }

  return { x: 0, y: 0 };
}

function resetGame() {
  snake.value = createInitialSnake();
  previousSnake.value = cloneSnake(snake.value);
  direction.value = { x: 1, y: 0, name: 'right' };
  queuedDirection.value = { x: 1, y: 0, name: 'right' };
  food.value = randomFood(snake.value);
  score.value = 0;
  tickMs.value = config.initialTickMs;
  elapsedMs.value = 0;
  accumulatedMs = 0;
  runStartedAt = 0;
  lastTickAt = 0;
  lastMoveAt = 0;
  gameState.value = 'ready';
}

function startGame() {
  normalizeName();
  if (gameState.value === 'ready' || gameState.value === 'over') {
    resetGame();
  }
  gameState.value = 'running';
  runStartedAt = performance.now();
  lastTickAt = runStartedAt;
  lastMoveAt = runStartedAt;
}

function pauseGame() {
  if (gameState.value !== 'running') return;
  accumulatedMs += performance.now() - runStartedAt;
  elapsedMs.value = Math.round(accumulatedMs);
  gameState.value = 'paused';
}

function togglePause() {
  if (gameState.value === 'running') {
    pauseGame();
    return;
  }
  if (gameState.value === 'paused') {
    gameState.value = 'running';
    runStartedAt = performance.now();
    lastTickAt = runStartedAt;
    lastMoveAt = runStartedAt;
    previousSnake.value = cloneSnake(snake.value);
  }
}

function queueDirection(nextDirection: Direction) {
  const current = queuedDirection.value;
  if (nextDirection.x + current.x === 0 && nextDirection.y + current.y === 0) return;
  queuedDirection.value = nextDirection;
}

function updateSpeed() {
  const speedups = Math.floor(score.value / config.speedupEveryPoints);
  tickMs.value = Math.max(
    config.minTickMs,
    config.initialTickMs - speedups * config.speedupStepMs,
  );
}

function endGame() {
  accumulatedMs += performance.now() - runStartedAt;
  elapsedMs.value = Math.round(accumulatedMs);
  gameState.value = 'over';
  if (score.value > highScore.value) {
    highScore.value = score.value;
    localStorage.setItem('cyber-snake-high-score', String(score.value));
  }
}

function stepGame(timestamp: number) {
  previousSnake.value = cloneSnake(snake.value);
  direction.value = queuedDirection.value;
  const head = snake.value[0];
  const nextHead = {
    x: head.x + direction.value.x,
    y: head.y + direction.value.y,
  };

  const hitWall =
    nextHead.x < 0 ||
    nextHead.y < 0 ||
    nextHead.x >= config.boardWidth ||
    nextHead.y >= config.boardHeight;
  if (hitWall) {
    endGame();
    return;
  }

  const willEat = samePoint(nextHead, food.value);
  const bodyToCheck = willEat ? snake.value : snake.value.slice(0, -1);
  if (bodyToCheck.some((segment) => samePoint(segment, nextHead))) {
    endGame();
    return;
  }

  const nextSnake = [nextHead, ...snake.value];
  if (willEat) {
    score.value += config.pointsPerFood;
    updateSpeed();
    food.value = randomFood(nextSnake);
  } else {
    nextSnake.pop();
  }
  snake.value = nextSnake;
  lastMoveAt = timestamp;
}

function handleKeydown(event: KeyboardEvent) {
  const key = event.key.length === 1 ? event.key.toLowerCase() : event.key;
  const nextDirection = directions[key];
  if (!nextDirection) return;

  event.preventDefault();
  if (gameState.value === 'ready') {
    startGame();
  }
  queueDirection(nextDirection);
}

function clamp(value: number, min: number, max: number) {
  return Math.min(max, Math.max(min, value));
}

function interpolatePoint(from: Point, to: Point, progress: number) {
  return {
    x: from.x + (to.x - from.x) * progress,
    y: from.y + (to.y - from.y) * progress,
  };
}

function getRenderSnake(progress: number, cell: number): RenderPoint[] {
  return snake.value.map((segment, index) => {
    const previous = previousSnake.value[index] ?? segment;
    const interpolated = interpolatePoint(previous, segment, progress);
    const taper = 1 - index / Math.max(1, snake.value.length - 1);
    return {
      x: (interpolated.x + 0.5) * cell,
      y: (interpolated.y + 0.5) * cell,
      radius: cell * (0.22 + taper * 0.18),
    };
  });
}

function drawRoundedRect(
  context: CanvasRenderingContext2D,
  x: number,
  y: number,
  width: number,
  height: number,
  radius: number,
) {
  context.beginPath();
  context.moveTo(x + radius, y);
  context.lineTo(x + width - radius, y);
  context.quadraticCurveTo(x + width, y, x + width, y + radius);
  context.lineTo(x + width, y + height - radius);
  context.quadraticCurveTo(x + width, y + height, x + width - radius, y + height);
  context.lineTo(x + radius, y + height);
  context.quadraticCurveTo(x, y + height, x, y + height - radius);
  context.lineTo(x, y + radius);
  context.quadraticCurveTo(x, y, x + radius, y);
  context.closePath();
}

function drawSoftBackground(
  context: CanvasRenderingContext2D,
  width: number,
  height: number,
  timestamp: number,
) {
  const drift = Math.sin(timestamp / 900);
  const baseGradient = context.createLinearGradient(0, 0, width, height);
  baseGradient.addColorStop(0, '#05070d');
  baseGradient.addColorStop(0.48, '#090a12');
  baseGradient.addColorStop(1, '#150b13');
  context.fillStyle = baseGradient;
  context.fillRect(0, 0, width, height);

  const cyanGlow = context.createRadialGradient(
    width * (0.18 + drift * 0.04),
    height * 0.24,
    0,
    width * 0.24,
    height * 0.24,
    width * 0.58,
  );
  cyanGlow.addColorStop(0, 'rgba(42, 245, 255, 0.17)');
  cyanGlow.addColorStop(1, 'rgba(42, 245, 255, 0)');
  context.fillStyle = cyanGlow;
  context.fillRect(0, 0, width, height);

  const magentaGlow = context.createRadialGradient(
    width * 0.82,
    height * (0.8 - drift * 0.04),
    0,
    width * 0.82,
    height * 0.8,
    width * 0.52,
  );
  magentaGlow.addColorStop(0, 'rgba(255, 43, 214, 0.16)');
  magentaGlow.addColorStop(1, 'rgba(255, 43, 214, 0)');
  context.fillStyle = magentaGlow;
  context.fillRect(0, 0, width, height);
}

function drawSnakeBody(context: CanvasRenderingContext2D, points: RenderPoint[], cell: number) {
  if (points.length < 2) return;

  const drawPath = () => {
    context.beginPath();
    context.moveTo(points[0].x, points[0].y);
    for (let index = 1; index < points.length - 1; index += 1) {
      const current = points[index];
      const next = points[index + 1];
      context.quadraticCurveTo(
        current.x,
        current.y,
        (current.x + next.x) / 2,
        (current.y + next.y) / 2,
      );
    }
    const tail = points[points.length - 1];
    context.lineTo(tail.x, tail.y);
  };

  context.lineCap = 'round';
  context.lineJoin = 'round';
  context.shadowColor = '#27f58a';
  context.shadowBlur = cell * 1.25;
  context.strokeStyle = 'rgba(39, 245, 138, 0.28)';
  context.lineWidth = cell * 0.92;
  drawPath();
  context.stroke();

  context.shadowColor = '#23b8ff';
  context.shadowBlur = cell * 0.58;
  context.strokeStyle = '#21d9c7';
  context.lineWidth = cell * 0.58;
  drawPath();
  context.stroke();

  context.shadowBlur = 0;
  context.strokeStyle = '#27f58a';
  context.lineWidth = cell * 0.34;
  drawPath();
  context.stroke();

  context.strokeStyle = 'rgba(255, 255, 255, 0.72)';
  context.lineWidth = Math.max(2, cell * 0.08);
  drawPath();
  context.stroke();
}

function drawSnakeHead(
  context: CanvasRenderingContext2D,
  head: RenderPoint,
  cell: number,
  timestamp: number,
) {
  const angle = Math.atan2(direction.value.y, direction.value.x);
  const pulse = (Math.sin(timestamp / 180) + 1) / 2;
  const headLength = cell * 0.98;
  const headWidth = cell * 0.78;

  context.save();
  context.translate(head.x, head.y);
  context.rotate(angle);
  context.shadowColor = '#3fffea';
  context.shadowBlur = cell * (0.92 + pulse * 0.18);
  context.fillStyle = '#3fffea';
  context.strokeStyle = 'rgba(255, 255, 255, 0.9)';
  context.lineWidth = Math.max(1.5, cell * 0.07);
  context.beginPath();
  context.moveTo(headLength * 0.55, 0);
  context.quadraticCurveTo(headLength * 0.18, -headWidth * 0.55, -headLength * 0.42, -headWidth * 0.38);
  context.quadraticCurveTo(-headLength * 0.72, 0, -headLength * 0.42, headWidth * 0.38);
  context.quadraticCurveTo(headLength * 0.18, headWidth * 0.55, headLength * 0.55, 0);
  context.closePath();
  context.fill();
  context.stroke();

  context.shadowBlur = 0;
  context.fillStyle = '#06100e';
  context.beginPath();
  context.arc(headLength * 0.18, -headWidth * 0.18, cell * 0.06, 0, Math.PI * 2);
  context.arc(headLength * 0.18, headWidth * 0.18, cell * 0.06, 0, Math.PI * 2);
  context.fill();

  context.strokeStyle = '#ff2bd6';
  context.lineWidth = Math.max(1.5, cell * 0.045);
  context.lineCap = 'round';
  context.beginPath();
  context.moveTo(headLength * 0.55, 0);
  context.lineTo(headLength * (0.78 + pulse * 0.08), 0);
  context.lineTo(headLength * 0.94, -cell * 0.12);
  context.moveTo(headLength * (0.78 + pulse * 0.08), 0);
  context.lineTo(headLength * 0.94, cell * 0.12);
  context.stroke();
  context.restore();
}

function drawGame(timestamp: number) {
  const canvas = canvasRef.value;
  const context = canvas?.getContext('2d');
  if (!canvas || !context) return;

  const bounds = canvas.getBoundingClientRect();
  const dpr = window.devicePixelRatio || 1;
  const targetWidth = Math.max(1, Math.floor(bounds.width * dpr));
  const targetHeight = Math.max(1, Math.floor(bounds.height * dpr));

  if (canvas.width !== targetWidth || canvas.height !== targetHeight) {
    canvas.width = targetWidth;
    canvas.height = targetHeight;
  }

  context.setTransform(dpr, 0, 0, dpr, 0, 0);
  context.clearRect(0, 0, bounds.width, bounds.height);

  const cell = Math.min(bounds.width / config.boardWidth, bounds.height / config.boardHeight);
  const boardWidth = cell * config.boardWidth;
  const boardHeight = cell * config.boardHeight;
  const offsetX = (bounds.width - boardWidth) / 2;
  const offsetY = (bounds.height - boardHeight) / 2;
  const pulse = (Math.sin(timestamp / 240) + 1) / 2;
  const progress =
    gameState.value === 'running' && lastMoveAt > 0
      ? clamp((timestamp - lastMoveAt) / tickMs.value, 0, 1)
      : 1;

  drawSoftBackground(context, bounds.width, bounds.height, timestamp);
  context.save();
  context.translate(offsetX, offsetY);
  drawSoftBackground(context, boardWidth, boardHeight, timestamp + 500);

  context.strokeStyle = 'rgba(63, 255, 234, 0.38)';
  context.lineWidth = 2;
  context.strokeRect(1, 1, boardWidth - 2, boardHeight - 2);

  const foodCenterX = food.value.x * cell + cell / 2;
  const foodCenterY = food.value.y * cell + cell / 2;
  const foodGlow = context.createRadialGradient(
    foodCenterX,
    foodCenterY,
    1,
    foodCenterX,
    foodCenterY,
    cell * 1.45,
  );
  foodGlow.addColorStop(0, `rgba(255, 213, 74, ${0.9 + pulse * 0.1})`);
  foodGlow.addColorStop(0.5, 'rgba(255, 68, 214, 0.38)');
  foodGlow.addColorStop(1, 'rgba(255, 68, 214, 0)');
  context.fillStyle = foodGlow;
  context.beginPath();
  context.arc(foodCenterX, foodCenterY, cell * 1.25, 0, Math.PI * 2);
  context.fill();

  context.shadowColor = '#ffd54a';
  context.shadowBlur = 18 + pulse * 10;
  context.fillStyle = '#ffd54a';
  drawRoundedRect(
    context,
    food.value.x * cell + cell * 0.27,
    food.value.y * cell + cell * 0.27,
    cell * 0.46,
    cell * 0.46,
    Math.max(3, cell * 0.1),
  );
  context.fill();
  context.shadowBlur = 0;

  const renderSnake = getRenderSnake(progress, cell);
  drawSnakeBody(context, renderSnake, cell);
  if (renderSnake[0]) {
    drawSnakeHead(context, renderSnake[0], cell, timestamp);
  }

  if (gameState.value === 'over') {
    context.strokeStyle = `rgba(255, 43, 92, ${0.45 + pulse * 0.25})`;
    context.lineWidth = 5;
    context.strokeRect(2, 2, boardWidth - 4, boardHeight - 4);
  }
  context.restore();
}

function animationLoop(timestamp: number) {
  if (gameState.value === 'running') {
    elapsedMs.value = Math.round(accumulatedMs + timestamp - runStartedAt);
    if (timestamp - lastTickAt >= tickMs.value) {
      lastTickAt = timestamp;
      stepGame(timestamp);
    }
  }

  drawGame(timestamp);
  animationFrame = window.requestAnimationFrame(animationLoop);
}

onMounted(async () => {
  resetGame();
  window.addEventListener('keydown', handleKeydown, { passive: false });
  animationFrame = window.requestAnimationFrame(animationLoop);
  await nextTick();
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown);
  window.cancelAnimationFrame(animationFrame);
});
</script>

<template>
  <main class="app-shell">
    <section class="game-zone" aria-label="赛博贪吃蛇游戏区域">
      <header class="hero-bar">
        <div>
          <p class="eyebrow">CYBER SNAKE</p>
          <h1>赛博贪吃蛇</h1>
        </div>
        <div class="status-chip online">
          <Sparkles :size="18" />
          <span>纯前端版</span>
        </div>
      </header>

      <div class="arena-frame" :class="gameState">
        <canvas ref="canvasRef" class="game-canvas" aria-label="贪吃蛇画布"></canvas>
        <div v-if="gameState !== 'running'" class="game-overlay">
          <span>{{ overlayKicker }}</span>
          <strong>{{ overlayTitle }}</strong>
          <button class="primary-action" type="button" @click="startGame">
            <Play :size="18" />
            {{ primaryActionLabel }}
          </button>
        </div>
      </div>
    </section>

    <aside class="control-deck" aria-label="控制面板">
      <section class="pilot-panel">
        <div class="panel-heading">
          <UserRound :size="18" />
          <span>玩家档案</span>
        </div>
        <label class="input-field" for="player-name">
          <span>代号</span>
          <input
            id="player-name"
            v-model="playerName"
            maxlength="16"
            autocomplete="off"
            @blur="normalizeName"
          />
        </label>
        <div class="button-row">
          <button type="button" :disabled="gameState === 'ready' || gameState === 'over'" @click="togglePause">
            <Pause v-if="gameState === 'running'" :size="17" />
            <Play v-else :size="17" />
            {{ gameState === 'running' ? '暂停' : '继续' }}
          </button>
          <button type="button" @click="resetGame">
            <RotateCcw :size="17" />
            重置
          </button>
        </div>
      </section>

      <section class="stats-grid" aria-label="游戏数据">
        <div class="stat-cell">
          <Sparkles :size="18" />
          <span>分数</span>
          <strong>{{ score }}</strong>
        </div>
        <div class="stat-cell">
          <Trophy :size="18" />
          <span>本地最高</span>
          <strong>{{ localBest }}</strong>
        </div>
        <div class="stat-cell">
          <Activity :size="18" />
          <span>长度</span>
          <strong>{{ snakeLength }}</strong>
        </div>
        <div class="stat-cell">
          <Gauge :size="18" />
          <span>速度</span>
          <strong>{{ currentSpeed }}</strong>
        </div>
      </section>
    </aside>
  </main>
</template>
