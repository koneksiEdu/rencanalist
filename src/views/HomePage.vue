<template>
  <div class="bsp-root">
    <!-- Animated starfield background -->
    <div class="starfield" aria-hidden="true">
      <div v-for="i in 40" :key="i" class="star" :style="starStyle(i)" />
    </div>

    <div class="game-wrapper">
      <!-- Title Bar -->
      <div class="title-bar">
        <span class="title-logo">✦ BALL</span>
        <span class="title-logo accent">SHINING</span>
        <span class="title-logo">POPPER ✦</span>
      </div>

      <!-- Canvas Frame -->
      <div class="canvas-frame">
        <canvas ref="gameCanvas" width="320" height="480" class="game-canvas" />

        <!-- START SCREEN -->
        <Transition name="screen-fade">
          <div v-if="state === 'start'" class="screen-overlay">
            <div class="screen-card">
              <div class="orb-deco">
                <div class="orb orb-1" />
                <div class="orb orb-2" />
                <div class="orb orb-3" />
              </div>
              <p class="chip-label">⚡ ARCADE EDITION</p>
              <h1 class="game-title">Ball<br><em>Shining</em><br>Popper</h1>
              <p class="game-desc">Cocokkan 3+ bola bercahaya<br>Jangan biarkan melewati garis bahaya!</p>
              <button class="btn-primary" @click="beginGame">
                <span class="btn-shine" />
                <span class="btn-text">🌟 MULAI MAIN</span>
              </button>
              <div class="hint-text">TAP atau KLIK untuk tembak</div>
            </div>
          </div>
        </Transition>

        <!-- GAME OVER SCREEN -->
        <Transition name="screen-fade">
          <div v-if="state === 'over'" class="screen-overlay">
            <div class="screen-card over-card">
              <p class="over-emoji">💥✨💥</p>
              <p class="chip-label danger-chip">GAME OVER</p>
              <div class="score-display">
                <span class="score-tiny">SKOR KAMU</span>
                <span class="score-big">{{ score }}</span>
              </div>
              <div class="hs-badge" :class="{ newrecord: score >= highScore && score > 0 }">
                <template v-if="score >= highScore && score > 0">⭐ REKOR BARU ⭐</template>
                <template v-else>REKOR: {{ highScore }}</template>
              </div>
              <div class="stats-grid">
                <div class="stat-item">
                  <span class="stat-val">{{ level }}</span>
                  <span class="stat-key">LEVEL</span>
                </div>
                <div class="stat-item">
                  <span class="stat-val">{{ popCount }}</span>
                  <span class="stat-key">POP</span>
                </div>
                <div class="stat-item">
                  <span class="stat-val">{{ shotsFired }}</span>
                  <span class="stat-key">TEMBAK</span>
                </div>
              </div>
              <button class="btn-primary" @click="beginGame">
                <span class="btn-shine" />
                <span class="btn-text">🔄 COBA LAGI</span>
              </button>
            </div>
          </div>
        </Transition>

        <!-- HUD -->
        <div v-if="state === 'play'" class="hud">
          <div class="hud-left">
            <span class="hud-label">SKOR</span>
            <span class="hud-value">{{ score }}</span>
          </div>
          <div class="hud-center">
            <span class="hud-label">NEXT</span>
            <div class="next-ball" :style="{ background: ballColors[nextBallColor], boxShadow: `0 0 10px ${ballColors[nextBallColor]}` }" />
          </div>
          <div class="hud-right">
            <span class="hud-label">LEVEL</span>
            <span class="hud-value">{{ level }}</span>
          </div>
        </div>

        <!-- Danger Bar -->
        <div v-if="state === 'play'" class="danger-strip" :style="{ opacity: dangerOpacity, transform: `scaleX(${dangerOpacity})` }" />
      </div>

      <!-- Controls -->
      <div v-if="state === 'play'" class="controls-bar">
        <button class="ctrl-btn"
          @touchstart.prevent="startRotate(-1)" @touchend.prevent="stopRotate"
          @mousedown.prevent="startRotate(-1)" @mouseup.prevent="stopRotate" @mouseleave.prevent="stopRotate">
          <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
            <path d="M13 4L7 10l6 6" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </button>

        <button class="ctrl-btn fire-btn" @touchstart.prevent="fireMarble" @mousedown.prevent="fireMarble">
          <span class="fire-glow" />
          <span class="fire-icon">⚡</span>
          <span class="fire-label">TEMBAK</span>
        </button>

        <button class="ctrl-btn"
          @touchstart.prevent="startRotate(1)" @touchend.prevent="stopRotate"
          @mousedown.prevent="startRotate(1)" @mouseup.prevent="stopRotate" @mouseleave.prevent="stopRotate">
          <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
            <path d="M7 4l6 6-6 6" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

// ── Constants ─────────────────────────────────────────────
const W = 320, H = 480
const R = 12
const COLS = 10, ROWS = 14
const CELL_W = W / COLS
const CELL_H = R * 2.1
const TOP_PAD = R + 30
const ROW_H = R * 1.72
const CANNON_Y = H - 50
const DANGER_ROW = 11
const DROP_INTERVAL_BASE = 20
const SPEED = 400
const ROTATE_STEP = 0.028
const MIN_ANGLE = -Math.PI + 0.18
const MAX_ANGLE = -0.18

// Shining neon palette
const ballColors = [
  '#00F5FF', // cyan
  '#FF2D78', // hot pink
  '#A259FF', // violet
  '#FFD600', // gold
  '#00FF94', // mint
  '#FF6B35', // orange
  '#38EFFF', // sky
]

// ── Reactive state ────────────────────────────────────────
const gameCanvas = ref(null)
const state = ref('start')
const score = ref(0)
const highScore = ref(0)
const level = ref(1)
const popCount = ref(0)
const shotsFired = ref(0)
const nextBallColor = ref(0)

// ── Internal state ────────────────────────────────────────
let ctx = null
let grid = []
let cannonAngle = -Math.PI / 2
let currentColor = 0
let bullet = null
let particles = []
let dropTimer = 0
let dropInterval = DROP_INTERVAL_BASE
let pops = 0, shots = 0, maxRow = 0
let rotateDir = 0
let animId = null
let lastTs = 0
let gameActive = false

// ── Star decoration helper ────────────────────────────────
function starStyle(i) {
  const seed = i * 137.508
  const x = (seed * 31) % 100
  const y = (seed * 17) % 100
  const size = 1 + (i % 3)
  const delay = (i * 0.3) % 4
  const dur = 2 + (i % 3)
  return {
    left: x + '%',
    top: y + '%',
    width: size + 'px',
    height: size + 'px',
    animationDelay: delay + 's',
    animationDuration: dur + 's',
  }
}

// ── Computed ──────────────────────────────────────────────
const dangerOpacity = computed(() => {
  if (state.value !== 'play') return 0
  return Math.max(0, (maxRow / DANGER_ROW - 0.6) * 2.5)
})

// ── Grid helpers ──────────────────────────────────────────
function hexCenter(col, row) {
  return { x: col * CELL_W + CELL_W / 2, y: row * CELL_H + TOP_PAD }
}

function hexNeighbors(row, col) {
  const result = []
  for (const [dr, dc] of [[-1,0],[1,0],[0,-1],[0,1]]) {
    const r = row + dr, c = col + dc
    if (r >= 0 && r < ROWS && c >= 0 && c < COLS) result.push({ row: r, col: c })
  }
  return result
}

function makeGrid() {
  return Array.from({ length: ROWS }, () => Array(COLS).fill(null))
}

function colorCount() {
  return Math.min(3 + Math.floor(level.value / 2), ballColors.length)
}

function randColor() {
  return Math.floor(Math.random() * colorCount())
}

function initGrid() {
  grid = makeGrid()
  for (let row = 0; row < 5; row++)
    for (let col = 0; col < COLS; col++)
      if (Math.random() < 0.85) grid[row][col] = { color: randColor() }
  recalcMaxRow()
}

function recalcMaxRow() {
  maxRow = 0
  for (let r = ROWS - 1; r >= 0; r--)
    for (let c = 0; c < COLS; c++)
      if (grid[r][c]) { maxRow = r; return }
}

// ── Match logic ───────────────────────────────────────────
function floodFill(row, col, color, visited) {
  const key = row * COLS + col
  if (visited[key]) return []
  if (!grid[row]?.[col] || grid[row][col].color !== color) return []
  visited[key] = true
  const group = [{ row, col }]
  for (const n of hexNeighbors(row, col))
    group.push(...floodFill(n.row, n.col, color, visited))
  return group
}

function findFloating() {
  const attached = new Uint8Array(ROWS * COLS)
  const queue = []
  for (let c = 0; c < COLS; c++)
    if (grid[0][c] && !attached[c]) { attached[c] = 1; queue.push({ row: 0, col: c }) }
  let head = 0
  while (head < queue.length) {
    const { row, col } = queue[head++]
    for (const n of hexNeighbors(row, col)) {
      const key = n.row * COLS + n.col
      if (grid[n.row][n.col] && !attached[key]) { attached[key] = 1; queue.push(n) }
    }
  }
  const floating = []
  for (let r = 0; r < ROWS; r++)
    for (let c = 0; c < COLS; c++)
      if (grid[r][c] && !attached[r * COLS + c]) floating.push({ row: r, col: c })
  return floating
}

function emitParticles(x, y, color, count = 12) {
  for (let i = 0; i < count; i++) {
    const angle = Math.random() * Math.PI * 2
    const spd = 60 + Math.random() * 180
    particles.push({ x, y, vx: Math.cos(angle) * spd, vy: Math.sin(angle) * spd,
      r: 2 + Math.random() * 4, color, life: 0.8 + Math.random() * 0.5 })
  }
}

function tryMatch(row, col) {
  if (!grid[row]?.[col]) return
  const visited = new Uint8Array(ROWS * COLS)
  const group = floodFill(row, col, grid[row][col].color, visited)
  if (group.length >= 3) {
    for (const { row: r, col: c } of group) {
      if (grid[r]?.[c]) { const { x, y } = hexCenter(c, r); emitParticles(x, y, ballColors[grid[r][c].color], 10); grid[r][c] = null }
    }
    pops += group.length
    popCount.value = pops
    score.value += group.length * 10 * level.value
    for (const { row: r, col: c } of findFloating()) {
      if (grid[r]?.[c]) { const { x, y } = hexCenter(c, r); emitParticles(x, y, ballColors[grid[r][c].color], 6); score.value += 5 * level.value; grid[r][c] = null; pops++ }
    }
    popCount.value = pops
    if (score.value >= level.value * 300) { level.value++; dropInterval = Math.max(8, DROP_INTERVAL_BASE - level.value * 1.2) }
  }
  recalcMaxRow()
  if (maxRow >= DANGER_ROW) endGame()
}

function dropRow() {
  for (let r = ROWS - 1; r > 0; r--) {
    grid[r] = []
    for (let c = 0; c < COLS; c++)
      grid[r][c] = grid[r-1][c] ? { color: grid[r-1][c].color } : null
  }
  grid[0] = Array(COLS).fill(null)
  for (let c = 0; c < COLS; c++)
    if (Math.random() < 0.75) grid[0][c] = { color: randColor() }
  recalcMaxRow()
  if (maxRow >= DANGER_ROW) endGame()
}

function placeMarble(bx, by, color) {
  let bestRow = -1, bestCol = -1, minDist = Infinity
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      if (grid[r][c]) continue
      const { x, y } = hexCenter(c, r)
      const d = Math.hypot(bx - x, by - y)
      if (d < minDist && d < R * 2.2) {
        const hasNeighbor = r === 0 || hexNeighbors(r, c).some(n => grid[n.row]?.[n.col])
        if (hasNeighbor) { minDist = d; bestRow = r; bestCol = c }
      }
    }
  }
  if (bestRow === -1) {
    for (let r = 0; r < ROWS; r++)
      for (let c = 0; c < COLS; c++) {
        if (grid[r][c]) continue
        const { x, y } = hexCenter(c, r)
        const d = Math.hypot(bx - x, by - y)
        if (d < minDist && d < R * 2.5) { minDist = d; bestRow = r; bestCol = c }
      }
  }
  if (bestRow === -1) { endGame(); return }
  grid[bestRow][bestCol] = { color }
  tryMatch(bestRow, bestCol)
  if (maxRow >= DANGER_ROW) endGame()
}

// ── Drawing ───────────────────────────────────────────────
function drawBackground() {
  const grad = ctx.createLinearGradient(0, 0, 0, H)
  grad.addColorStop(0, '#03001C')
  grad.addColorStop(0.5, '#0A0628')
  grad.addColorStop(1, '#130042')
  ctx.fillStyle = grad
  ctx.fillRect(0, 0, W, H)

  // Grid dots
  ctx.fillStyle = 'rgba(100,120,255,0.06)'
  for (let r = 0; r < ROWS; r++)
    for (let c = 0; c < COLS; c++) {
      const { x, y } = hexCenter(c, r)
      ctx.beginPath(); ctx.arc(x, y, 1.2, 0, Math.PI * 2); ctx.fill()
    }

  // Danger line
  const dangerY = DANGER_ROW * CELL_H + TOP_PAD - R
  ctx.save()
  ctx.strokeStyle = 'rgba(255,45,120,0.9)'
  ctx.lineWidth = 2
  ctx.setLineDash([6, 5])
  ctx.shadowColor = '#FF2D78'
  ctx.shadowBlur = 8
  ctx.beginPath(); ctx.moveTo(0, dangerY); ctx.lineTo(W, dangerY); ctx.stroke()
  ctx.setLineDash([])
  ctx.restore()
}

function drawBall(x, y, colorIdx, radius) {
  radius = radius || R
  const color = ballColors[colorIdx]

  ctx.save()
  ctx.shadowColor = color
  ctx.shadowBlur = 14

  // Main sphere gradient
  const g = ctx.createRadialGradient(x - radius * 0.3, y - radius * 0.35, radius * 0.05, x, y, radius)
  g.addColorStop(0, lighten(color, 60))
  g.addColorStop(0.45, color)
  g.addColorStop(1, darken(color, 40))
  ctx.fillStyle = g
  ctx.beginPath(); ctx.arc(x, y, radius, 0, Math.PI * 2); ctx.fill()
  ctx.restore()

  // Ring
  ctx.strokeStyle = `${color}55`
  ctx.lineWidth = 1.5
  ctx.beginPath(); ctx.arc(x, y, radius + 2, 0, Math.PI * 2); ctx.stroke()

  // Specular highlight
  ctx.fillStyle = 'rgba(255,255,255,0.5)'
  ctx.beginPath(); ctx.arc(x - radius * 0.28, y - radius * 0.3, radius * 0.2, 0, Math.PI * 2); ctx.fill()

  // Inner shine
  ctx.fillStyle = 'rgba(255,255,255,0.15)'
  ctx.beginPath(); ctx.arc(x + radius * 0.1, y + radius * 0.1, radius * 0.45, 0, Math.PI * 2); ctx.fill()
}

function lighten(hex, amt) {
  let r = parseInt(hex.slice(1,3),16), g = parseInt(hex.slice(3,5),16), b = parseInt(hex.slice(5,7),16)
  return `rgb(${Math.min(255,r+amt)},${Math.min(255,g+amt)},${Math.min(255,b+amt)})`
}
function darken(hex, amt) {
  let r = parseInt(hex.slice(1,3),16), g = parseInt(hex.slice(3,5),16), b = parseInt(hex.slice(5,7),16)
  return `rgb(${Math.max(0,r-amt)},${Math.max(0,g-amt)},${Math.max(0,b-amt)})`
}

function drawCannon() {
  const cx = W / 2, cy = CANNON_Y

  // Base platform glow
  ctx.save()
  ctx.shadowColor = '#A259FF'
  ctx.shadowBlur = 18
  const baseGrad = ctx.createRadialGradient(cx, cy, 0, cx, cy, 24)
  baseGrad.addColorStop(0, '#C77DFF')
  baseGrad.addColorStop(1, '#7B2FBE')
  ctx.fillStyle = baseGrad
  ctx.beginPath(); ctx.ellipse(cx, cy + 5, 24, 13, 0, 0, Math.PI * 2); ctx.fill()
  ctx.restore()

  ctx.save()
  ctx.translate(cx, cy)
  ctx.rotate(cannonAngle)

  // Barrel
  const barrelGrad = ctx.createLinearGradient(-6, -34, 6, -34)
  barrelGrad.addColorStop(0, '#E0C3FC')
  barrelGrad.addColorStop(0.5, '#8EC5FC')
  barrelGrad.addColorStop(1, '#A259FF')
  ctx.fillStyle = barrelGrad
  ctx.shadowColor = '#A259FF'
  ctx.shadowBlur = 12
  ctx.beginPath(); ctx.roundRect(-5, -32, 10, 32, 4); ctx.fill()

  // Barrel tip
  ctx.fillStyle = '#00F5FF'
  ctx.shadowColor = '#00F5FF'
  ctx.shadowBlur = 16
  ctx.beginPath(); ctx.roundRect(-5, -36, 10, 8, 3); ctx.fill()
  ctx.restore()

  // Aim guide
  for (let i = 1; i <= 10; i++) {
    const t = i / 10
    const gx = cx + Math.cos(cannonAngle) * (28 + t * 130)
    const gy = cy + Math.sin(cannonAngle) * (28 + t * 130)
    ctx.fillStyle = `rgba(0,245,255,${0.22 * (1 - t)})`
    ctx.beginPath(); ctx.arc(gx, gy, 2, 0, Math.PI * 2); ctx.fill()
  }

  drawBall(cx, cy - 4, currentColor, R - 1)
}

function drawParticles(dt) {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]
    p.x += p.vx * dt; p.y += p.vy * dt; p.vy += 240 * dt; p.life -= dt * 1.6
    if (p.life <= 0) { particles.splice(i, 1); continue }
    ctx.save()
    ctx.globalAlpha = Math.min(1, p.life)
    ctx.shadowColor = p.color; ctx.shadowBlur = 6
    ctx.fillStyle = p.color
    ctx.beginPath(); ctx.arc(p.x, p.y, p.r * p.life, 0, Math.PI * 2); ctx.fill()
    ctx.restore()
  }
}

function render() {
  if (!ctx) return
  drawBackground()
  for (let r = 0; r < ROWS; r++)
    for (let c = 0; c < COLS; c++) {
      const cell = grid[r]?.[c]
      if (cell && cell.color !== undefined) { const { x, y } = hexCenter(c, r); drawBall(x, y, cell.color) }
    }
  if (bullet && bullet.color !== undefined) drawBall(bullet.x, bullet.y, bullet.color)
  drawCannon()
  drawParticles(1 / 60)
}

// ── Game loop ─────────────────────────────────────────────
function update(dt) {
  if (!gameActive) return
  if (rotateDir !== 0) cannonAngle = Math.max(MIN_ANGLE, Math.min(MAX_ANGLE, cannonAngle + rotateDir * ROTATE_STEP))
  dropTimer += dt
  if (dropTimer >= dropInterval) { dropTimer = 0; dropRow() }
  if (bullet) {
    bullet.x += bullet.vx * dt; bullet.y += bullet.vy * dt
    if (bullet.x - R < 0) { bullet.x = R; bullet.vx = Math.abs(bullet.vx) }
    if (bullet.x + R > W) { bullet.x = W - R; bullet.vx = -Math.abs(bullet.vx) }
    if (bullet.y - R <= TOP_PAD - R) { placeMarble(bullet.x, bullet.y, bullet.color); bullet = null; return }
    for (let r = 0; r <= Math.min(maxRow + 2, ROWS - 1); r++)
      for (let c = 0; c < COLS; c++) {
        const cell = grid[r]?.[c]
        if (!cell) continue
        const { x, y } = hexCenter(c, r)
        if (Math.hypot(bullet.x - x, bullet.y - y) < R * 1.85) { placeMarble(bullet.x, bullet.y, bullet.color); bullet = null; return }
      }
    if (bullet.y + R > H) bullet = null
  }
}

function gameLoop(ts) {
  const dt = lastTs ? Math.min((ts - lastTs) / 1000, 0.033) : 0
  lastTs = ts
  update(dt); render()
  animId = requestAnimationFrame(gameLoop)
}

// ── Input ─────────────────────────────────────────────────
function onCanvasClick(e) {
  if (state.value !== 'play') return
  const rect = gameCanvas.value.getBoundingClientRect()
  const sx = W / rect.width, sy = H / rect.height
  const cx = e.clientX ?? e.touches?.[0]?.clientX
  const cy = e.clientY ?? e.touches?.[0]?.clientY
  if (cx == null) return
  const mx = (cx - rect.left) * sx, my = (cy - rect.top) * sy
  cannonAngle = Math.max(MIN_ANGLE, Math.min(MAX_ANGLE, Math.atan2(my - CANNON_Y, mx - W / 2)))
  fireMarble()
}

function onKeyDown(e) {
  if (state.value !== 'play') return
  if (e.key === 'ArrowLeft')  { e.preventDefault(); cannonAngle = Math.max(MIN_ANGLE, cannonAngle - ROTATE_STEP * 2) }
  if (e.key === 'ArrowRight') { e.preventDefault(); cannonAngle = Math.min(MAX_ANGLE, cannonAngle + ROTATE_STEP * 2) }
  if (e.key === ' ' || e.key === 'ArrowUp') { e.preventDefault(); fireMarble() }
}

function startRotate(dir) { rotateDir = dir }
function stopRotate() { rotateDir = 0 }

function fireMarble() {
  if (!gameActive || bullet) return
  bullet = { x: W / 2, y: CANNON_Y - 4, vx: Math.cos(cannonAngle) * SPEED, vy: Math.sin(cannonAngle) * SPEED, color: currentColor }
  currentColor = nextBallColor.value
  nextBallColor.value = randColor()
  shots++; shotsFired.value = shots
}

function beginGame() {
  state.value = 'play'; score.value = 0; level.value = 1; popCount.value = 0; shotsFired.value = 0
  pops = 0; shots = 0; bullet = null; particles = []
  cannonAngle = -Math.PI / 2; dropTimer = 0; dropInterval = DROP_INTERVAL_BASE
  rotateDir = 0; lastTs = 0; gameActive = true
  initGrid(); currentColor = randColor(); nextBallColor.value = randColor()
}

function endGame() {
  gameActive = false; state.value = 'over'
  if (score.value > highScore.value) highScore.value = score.value
  bullet = null
}

onMounted(() => {
  ctx = gameCanvas.value.getContext('2d')
  render()
  gameCanvas.value.addEventListener('click', onCanvasClick)
  gameCanvas.value.addEventListener('touchstart', e => { e.preventDefault(); onCanvasClick(e) }, { passive: false })
  document.addEventListener('keydown', onKeyDown)
  animId = requestAnimationFrame(gameLoop)
})

onUnmounted(() => {
  if (animId) cancelAnimationFrame(animId)
  document.removeEventListener('keydown', onKeyDown)
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Exo+2:ital,wght@0,400;0,700;0,900;1,900&family=Rajdhani:wght@500;600;700&display=swap');

*, *::before, *::after {
  box-sizing: border-box;
  -webkit-tap-highlight-color: transparent;
  user-select: none;
  touch-action: manipulation;
}

/* ── Root ── */
.bsp-root {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100svh;
  width: 100%;
  background: #03001C;
  overflow: hidden;
  font-family: 'Rajdhani', 'Segoe UI', sans-serif;
}

/* ── Starfield ── */
.starfield {
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: 0;
}
.star {
  position: absolute;
  border-radius: 50%;
  background: white;
  animation: twinkle var(--dur, 3s) ease-in-out infinite;
  opacity: 0.6;
}
@keyframes twinkle {
  0%, 100% { opacity: 0.15; transform: scale(1); }
  50% { opacity: 0.9; transform: scale(1.4); }
}

/* ── Game wrapper ── */
.game-wrapper {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  max-width: 400px;
  padding: 0 8px 16px;
  gap: 0;
}

/* ── Title bar ── */
.title-bar {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 10px 0 6px;
  font-family: 'Exo 2', sans-serif;
  font-weight: 900;
  font-size: clamp(11px, 3vw, 14px);
  letter-spacing: 0.15em;
  color: rgba(162, 89, 255, 0.7);
}
.title-logo.accent {
  color: #00F5FF;
  text-shadow: 0 0 12px #00F5FF;
}

/* ── Canvas frame ── */
.canvas-frame {
  position: relative;
  width: 100%;
  max-width: 340px;
  aspect-ratio: 320 / 480;
  border-radius: 20px;
  overflow: hidden;
  box-shadow:
    0 0 0 1px rgba(162,89,255,0.3),
    0 0 40px rgba(0,245,255,0.12),
    0 0 80px rgba(162,89,255,0.08);
}

.game-canvas {
  display: block;
  width: 100%;
  height: 100%;
  cursor: crosshair;
}

/* ── Screens ── */
.screen-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  padding-bottom: 24px;
  background: linear-gradient(to bottom, transparent 15%, rgba(3,0,28,0.97) 50%);
  border-radius: 20px;
}

.screen-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  padding: 0 20px;
  gap: 8px;
}
.over-card { gap: 6px; }

/* Orb decorations */
.orb-deco {
  position: relative;
  width: 80px;
  height: 40px;
  margin-bottom: 4px;
}
.orb {
  position: absolute;
  border-radius: 50%;
  filter: blur(8px);
  opacity: 0.7;
  animation: float 3s ease-in-out infinite;
}
.orb-1 { width: 30px; height: 30px; background: #00F5FF; top: 5px; left: 10px; animation-delay: 0s; }
.orb-2 { width: 24px; height: 24px; background: #FF2D78; top: 12px; left: 30px; animation-delay: 0.8s; }
.orb-3 { width: 20px; height: 20px; background: #A259FF; top: 2px; left: 50px; animation-delay: 1.5s; }
@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-6px); }
}

.chip-label {
  font-size: 9px;
  font-weight: 700;
  letter-spacing: 0.3em;
  color: #A259FF;
  border: 1px solid rgba(162,89,255,0.4);
  border-radius: 99px;
  padding: 3px 14px;
  background: rgba(162,89,255,0.08);
  margin: 0;
}
.danger-chip { color: #FF2D78; border-color: rgba(255,45,120,0.4); background: rgba(255,45,120,0.08); }

.game-title {
  font-family: 'Exo 2', sans-serif;
  font-style: italic;
  font-weight: 900;
  font-size: clamp(36px, 12vw, 46px);
  line-height: 0.9;
  text-align: center;
  margin: 0;
  color: #fff;
  text-shadow: 0 0 30px rgba(0,245,255,0.4), 0 0 60px rgba(162,89,255,0.2);
}
.game-title em {
  font-style: italic;
  color: #00F5FF;
  text-shadow: 0 0 20px #00F5FF, 0 0 40px rgba(0,245,255,0.5);
}

.game-desc {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.04em;
  color: rgba(255,255,255,0.28);
  text-align: center;
  line-height: 1.8;
  margin: 4px 0 8px;
}

.over-emoji {
  font-size: 24px;
  margin-bottom: 4px;
  animation: float 1.5s ease-in-out infinite;
}

.score-display {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
}
.score-tiny {
  font-size: 9px;
  font-weight: 700;
  letter-spacing: 0.22em;
  color: rgba(255,255,255,0.25);
}
.score-big {
  font-family: 'Exo 2', sans-serif;
  font-weight: 900;
  font-size: clamp(54px, 18vw, 68px);
  line-height: 1;
  color: #fff;
  text-shadow: 0 0 30px rgba(0,245,255,0.4);
}

.hs-badge {
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.12em;
  color: rgba(255,255,255,0.25);
  margin-bottom: 4px;
}
.hs-badge.newrecord {
  color: #FFD600;
  text-shadow: 0 0 12px rgba(255,214,0,0.6);
}

.stats-grid {
  display: flex;
  border: 1px solid rgba(0,245,255,0.15);
  border-radius: 14px;
  overflow: hidden;
  margin-bottom: 8px;
  background: rgba(0,245,255,0.03);
}
.stat-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 10px 20px;
  border-right: 1px solid rgba(0,245,255,0.1);
}
.stat-item:last-child { border-right: none; }
.stat-val {
  font-family: 'Exo 2', sans-serif;
  font-weight: 900;
  font-size: 22px;
  color: #fff;
}
.stat-key {
  font-size: 8px;
  font-weight: 700;
  letter-spacing: 0.16em;
  color: rgba(0,245,255,0.4);
  margin-top: 2px;
}

/* ── Button ── */
.btn-primary {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  border: none;
  border-radius: 99px;
  padding: 13px 48px;
  background: linear-gradient(135deg, #00c4ff, #7B2FBE, #FF2D78);
  background-size: 200% 200%;
  animation: gradShift 4s ease infinite;
  cursor: pointer;
  box-shadow: 0 4px 28px rgba(0,245,255,0.35), 0 0 0 1px rgba(255,255,255,0.1);
  transition: transform 0.12s, box-shadow 0.12s;
  margin-bottom: 4px;
}
@keyframes gradShift {
  0%, 100% { background-position: 0% 50%; }
  50% { background-position: 100% 50%; }
}
.btn-shine {
  position: absolute;
  top: 0; left: -60%;
  width: 40%; height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255,255,255,0.3), transparent);
  animation: shine 2.4s ease-in-out infinite;
}
@keyframes shine {
  0% { left: -60%; }
  100% { left: 140%; }
}
.btn-text {
  font-family: 'Exo 2', sans-serif;
  font-weight: 900;
  font-size: 15px;
  letter-spacing: 0.1em;
  color: #fff;
  position: relative;
  z-index: 1;
}
.btn-primary:hover { transform: translateY(-2px); box-shadow: 0 6px 36px rgba(0,245,255,0.45); }
.btn-primary:active { transform: scale(0.95); }

.hint-text {
  font-size: 9px;
  font-weight: 600;
  letter-spacing: 0.06em;
  color: rgba(255,255,255,0.14);
}

/* ── HUD ── */
.hud {
  position: absolute;
  top: 0; left: 0; right: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 14px;
  background: linear-gradient(to bottom, rgba(3,0,28,0.88) 0%, transparent);
  pointer-events: none;
  z-index: 5;
}

.hud-left, .hud-right { min-width: 60px; }
.hud-right { text-align: right; }
.hud-center { display: flex; flex-direction: column; align-items: center; }

.hud-label {
  display: block;
  font-size: 8px;
  font-weight: 700;
  letter-spacing: 0.2em;
  color: rgba(0,245,255,0.45);
}
.hud-value {
  display: block;
  font-family: 'Exo 2', sans-serif;
  font-weight: 900;
  font-size: 22px;
  color: #fff;
  text-shadow: 0 0 10px rgba(0,245,255,0.4);
  line-height: 1.1;
}

.next-ball {
  width: 22px;
  height: 22px;
  border-radius: 50%;
  border: 2px solid rgba(255,255,255,0.25);
  margin-top: 4px;
  transition: background 0.2s, box-shadow 0.2s;
}

/* ── Danger strip ── */
.danger-strip {
  position: absolute;
  bottom: 0; left: 0;
  width: 100%;
  height: 3px;
  background: linear-gradient(90deg, #FF2D78, #A259FF, #00F5FF, #A259FF, #FF2D78);
  background-size: 300% 100%;
  animation: dangerScroll 0.6s linear infinite;
  transform-origin: left;
  transition: opacity 0.3s, transform 0.3s;
}
@keyframes dangerScroll {
  0% { background-position: 0% 0%; }
  100% { background-position: 300% 0%; }
}

/* ── Controls ── */
.controls-bar {
  display: flex;
  gap: 6px;
  width: 100%;
  max-width: 340px;
  padding: 6px 0 0;
}

.ctrl-btn {
  flex: 1;
  height: 60px;
  background: rgba(0,245,255,0.05);
  border: 1px solid rgba(0,245,255,0.15);
  border-radius: 16px;
  color: rgba(255,255,255,0.55);
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  transition: transform 0.1s, background 0.1s, box-shadow 0.1s;
  gap: 2px;
}
.ctrl-btn:active {
  transform: scale(0.9);
  background: rgba(0,245,255,0.12);
  box-shadow: 0 0 12px rgba(0,245,255,0.2);
}

.fire-btn {
  position: relative;
  flex: 1.8;
  overflow: hidden;
  background: linear-gradient(135deg, rgba(0,245,255,0.1), rgba(162,89,255,0.08));
  border-color: rgba(0,245,255,0.3);
}
.fire-btn:active {
  background: linear-gradient(135deg, rgba(0,245,255,0.2), rgba(162,89,255,0.15));
  box-shadow: 0 0 20px rgba(0,245,255,0.25);
}

.fire-glow {
  position: absolute;
  inset: -2px;
  border-radius: 16px;
  background: conic-gradient(from 0deg, #00F5FF, #A259FF, #FF2D78, #00F5FF);
  opacity: 0;
  transition: opacity 0.2s;
  animation: rotGlow 2s linear infinite;
  mask: linear-gradient(#fff 0 0) content-box, linear-gradient(#fff 0 0);
  mask-composite: exclude;
  padding: 1px;
}
@keyframes rotGlow {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}
.fire-btn:active .fire-glow { opacity: 0.6; }

.fire-icon {
  font-size: 22px;
  line-height: 1;
  filter: drop-shadow(0 0 6px #00F5FF);
}
.fire-label {
  font-size: 9px;
  font-weight: 700;
  letter-spacing: 0.18em;
  color: rgba(0,245,255,0.65);
}

/* ── Screen transitions ── */
.screen-fade-enter-active, .screen-fade-leave-active {
  transition: opacity 0.3s ease;
}
.screen-fade-enter-from, .screen-fade-leave-to {
  opacity: 0;
}
</style>