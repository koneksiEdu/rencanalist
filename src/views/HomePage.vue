<template>
  <div class="root">
    <div class="frame">
      <canvas ref="gameCanvas" width="320" height="480"></canvas>

      <!-- START SCREEN -->
      <div v-if="state === 'start'" class="screen">
        <div class="screen-inner">
          <div class="badge">Marble Shotter ARCADE</div>
          <h1 class="title-main">Guli<br><span class="title-accent">Shutter</span></h1>
          <p class="subtitle">Tembak marble · Cocokkan 3+<br>Jangan biarkan marble melewati batas!</p>
          <button class="btn-start" @click="beginGame">
            <span class="btn-icon">🍭</span> MULAI GAME
          </button>
          <div class="hint-row">
            <span>TAP/KLIK untuk arahkan &amp; tembak</span>
          </div>
        </div>
      </div>

      <!-- GAME OVER -->
      <div v-if="state === 'over'" class="screen">
        <div class="screen-inner over-inner">
          <div class="candy-deco">💔🍬💔</div>
          <div class="badge danger">GAME OVER</div>
          <div class="over-score-label">SKOR AKHIR</div>
          <div class="over-score">{{ score }}</div>
          <div class="hs-line" :class="{ gold: score >= highScore && score > 0 }">
            <template v-if="score >= highScore && score > 0">⭐ REKOR BARU ⭐</template>
            <template v-else>REKOR: {{ highScore }}</template>
          </div>
          <div class="stat-row">
            <div class="stat">
              <div class="sv">{{ level }}</div>
              <div class="sk">LEVEL</div>
            </div>
            <div class="stat">
              <div class="sv">{{ popCount }}</div>
              <div class="sk">POP</div>
            </div>
            <div class="stat">
              <div class="sv">{{ shotsFired }}</div>
              <div class="sk">TEMBAKAN</div>
            </div>
          </div>
          <button class="btn-start" @click="beginGame">
            <span class="btn-icon">🔄</span> COBA LAGI
          </button>
        </div>
      </div>

      <!-- HUD -->
      <div v-if="state === 'play'" class="hud-overlay">
        <div class="hud-top">
          <div class="hud-cell">
            <div class="hk">SKOR</div>
            <div class="hv">{{ score }}</div>
          </div>
          <div class="hud-cell center">
            <div class="next-label">NEXT</div>
            <div class="next-marble" :style="{ background: marbleColors[nextMarbleColor] }"></div>
          </div>
          <div class="hud-cell right">
            <div class="hk">LEVEL</div>
            <div class="hv">{{ level }}</div>
          </div>
        </div>
        <div class="danger-bar" :style="{ opacity: dangerOpacity, width: dangerOpacity * 100 + '%' }"></div>
      </div>
    </div>

    <!-- CONTROLS -->
    <div v-if="state === 'play'" class="controls">
      <button class="cb"
        @touchstart.prevent="startRotate(-1)" @touchend.prevent="stopRotate"
        @mousedown.prevent="startRotate(-1)" @mouseup.prevent="stopRotate" @mouseleave.prevent="stopRotate">
        <svg width="22" height="22" viewBox="0 0 20 20"><path d="M13 4L7 10l6 6" stroke="currentColor" stroke-width="2.5" fill="none" stroke-linecap="round" stroke-linejoin="round"/></svg>
      </button>
      <button class="cb fire-btn" @touchstart.prevent="fireMarble" @mousedown.prevent="fireMarble">
        <div class="fire-icon">🍭</div>
        <div class="fire-label">TEMBAK</div>
      </button>
      <button class="cb"
        @touchstart.prevent="startRotate(1)" @touchend.prevent="stopRotate"
        @mousedown.prevent="startRotate(1)" @mouseup.prevent="stopRotate" @mouseleave.prevent="stopRotate">
        <svg width="22" height="22" viewBox="0 0 20 20"><path d="M7 4l6 6-6 6" stroke="currentColor" stroke-width="2.5" fill="none" stroke-linecap="round" stroke-linejoin="round"/></svg>
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

// ── Constants ──────────────────────────────────────────────
const W = 320
const H = 480
const R = 12          // marble radius (sedikit lebih kecil)
const COLS = 10
const ROWS = 14

// Hex grid: even rows offset right by half a cell
const CELL_W = W / COLS  // 32px
const CELL_H = R * 2.1   // ~25px agar bola tidak terlalu rapat
const TOP_PAD = R + 30   // first row Y
const ROW_H  = R * 1.72

// Cannon sits near bottom of canvas
const CANNON_Y = H - 50

// Danger: game over when any marble reaches row >= DANGER_ROW
const DANGER_ROW = 11

// Drop new row every N seconds (scales with level)
const DROP_INTERVAL_BASE = 20

// Bullet speed (px/s)
const SPEED = 400

// Rotation step per frame when button held (radians) – smaller = finer control
const ROTATE_STEP = 0.028

const MIN_ANGLE = -Math.PI + 0.18
const MAX_ANGLE = -0.18

const marbleColors = [
  '#FF6EB4', '#FF9F43', '#54A0FF', '#5F27CD',
  '#00D2D3', '#1DD1A1', '#FF6B6B'
]

// ── Vue reactive state ────────────────────────────────────
const gameCanvas    = ref(null)
const state         = ref('start')
const score         = ref(0)
const highScore     = ref(0)
const level         = ref(1)
const popCount      = ref(0)
const shotsFired    = ref(0)
const nextMarbleColor = ref(0)

// ── Game internals (non-reactive) ─────────────────────────
let ctx         = null
let grid        = []          // grid[row][col] = { color } | null
let cannonAngle = -Math.PI / 2
let currentColor = 0
let bullet      = null
let particles   = []
let dropTimer   = 0
let dropInterval = DROP_INTERVAL_BASE
let pops        = 0
let shots       = 0
let maxRow      = 0           // furthest occupied row
let rotateDir   = 0
let animId      = null
let lastTs      = 0
let gameActive  = false

// ── Computed ──────────────────────────────────────────────
const dangerOpacity = computed(() => {
  if (state.value !== 'play') return 0
  const ratio = maxRow / DANGER_ROW
  return Math.max(0, (ratio - 0.6) * 2.5)
})

// ── Hex helpers ───────────────────────────────────────────
/**
 * Returns pixel center of grid cell (col, row).
 * Even rows (0,2,4…) are NOT offset; odd rows offset right by CELL_W/2.
 * This is "odd-r" offset layout.
 */
function hexCenter(col, row) {
  const x = col * CELL_W + CELL_W / 2
  const y = row * CELL_H + TOP_PAD
  return { x, y }
}

/**
 * Returns valid neighbors of (row, col) using odd-r offset rules.
 * Reference: https://www.redblobgames.com/grids/hexagons/
 */
function hexNeighbors(row, col) {
  // Untuk grid persegi, gunakan 4 arah (atas, bawah, kiri, kanan)
  // Bisa juga tambahkan diagonal jika ingin match 8 arah
  const dirs = [
    [-1, 0], [1, 0], [0, -1], [0, 1]  // 4 arah
  ]
  const result = []
  for (const [dr, dc] of dirs) {
    const r = row + dr
    const c = col + dc
    if (r >= 0 && r < ROWS && c >= 0 && c < COLS) {
      result.push({ row: r, col: c })
    }
  }
  return result
}

// ── Grid helpers ──────────────────────────────────────────
function makeGrid() {
  return Array.from({ length: ROWS }, () => Array(COLS).fill(null))
}

function colorCount() {
  return Math.min(3 + Math.floor(level.value / 2), marbleColors.length)
}

function randColor() {
  return Math.floor(Math.random() * colorCount())
}

function initGrid() {
  grid = makeGrid()
  const filled = 5
  for (let row = 0; row < filled; row++) {
    for (let col = 0; col < COLS; col++) {
      // Semua row bisa diisi semua kolom (tidak ada skip)
      if (Math.random() < 0.85) {
        grid[row][col] = { color: randColor() }
      }
    }
  }
  recalcMaxRow()
}


function recalcMaxRow() {
  maxRow = 0
  for (let r = ROWS - 1; r >= 0; r--) {
    for (let c = 0; c < COLS; c++) {
      if (grid[r][c]) { 
        maxRow = r
        return 
      }
    }
  }
}

// ── Match / pop logic ─────────────────────────────────────
function floodFill(row, col, color, visited) {
  const key = row * COLS + col
  if (visited[key]) return []
  if (!grid[row]?.[col] || grid[row][col].color !== color) return []
  visited[key] = true
  const group = [{ row, col }]
  for (const n of hexNeighbors(row, col)) {
    group.push(...floodFill(n.row, n.col, color, visited))
  }
  return group
}

// ── Update fungsi findFloating untuk grid persegi ──
function findFloating() {
  const attached = new Uint8Array(ROWS * COLS)
  const queue = []

  // Mulai dari semua marble di row 0
  for (let c = 0; c < COLS; c++) {
    if (grid[0][c]) {
      const key = c
      if (!attached[key]) {
        attached[key] = 1
        queue.push({ row: 0, col: c })
      }
    }
  }

  // BFS untuk menemukan semua marble yang terhubung ke atas
  let head = 0
  while (head < queue.length) {
    const { row, col } = queue[head++]
    for (const n of hexNeighbors(row, col)) {
      const key = n.row * COLS + n.col
      if (grid[n.row][n.col] && !attached[key]) {
        attached[key] = 1
        queue.push(n)
      }
    }
  }

  // Marble yang tidak terhubung ke row 0 adalah floating
  const floating = []
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      if (grid[r][c] && !attached[r * COLS + c]) {
        floating.push({ row: r, col: c })
      }
    }
  }
  return floating
}

function emitParticles(x, y, color, count = 12) {
  for (let i = 0; i < count; i++) {
    const angle = Math.random() * Math.PI * 2
    const spd = 60 + Math.random() * 160
    particles.push({
      x, y,
      vx: Math.cos(angle) * spd,
      vy: Math.sin(angle) * spd,
      r: 2 + Math.random() * 4,
      color,
      life: 0.7 + Math.random() * 0.5
    })
  }
}

function tryMatch(row, col) {
  if (!grid[row]?.[col]) return;
  
  const visited = new Uint8Array(ROWS * COLS)
  const group = floodFill(row, col, grid[row][col].color, visited)

  if (group.length >= 3) {
    for (const { row: r, col: c } of group) {
      const cell = grid[r]?.[c]
      if (cell) {
        const { x, y } = hexCenter(c, r)
        emitParticles(x, y, marbleColors[cell.color], 10)
        grid[r][c] = null
      }
    }
    pops += group.length
    popCount.value = pops
    score.value += group.length * 10 * level.value

    // Remove floating marbles after clearing group
    const floating = findFloating()
    for (const { row: r, col: c } of floating) {
      const cell = grid[r]?.[c]
      if (cell) {
        const { x, y } = hexCenter(c, r)
        emitParticles(x, y, marbleColors[cell.color], 6)
        score.value += 5 * level.value
        grid[r][c] = null
        pops++
      }
    }
    popCount.value = pops

    // Level up
    if (score.value >= level.value * 300) {
      level.value++
      dropInterval = Math.max(8, DROP_INTERVAL_BASE - level.value * 1.2)
    }
  }

  recalcMaxRow()
  
  // Cek game over setelah match
  if (maxRow >= DANGER_ROW) {
    endGame()
  }
}

// ── Drop a new row from top ───────────────────────────────
function dropRow() {
  // Geser semua row ke bawah
  for (let r = ROWS - 1; r > 0; r--) {
    grid[r] = []
    for (let c = 0; c < COLS; c++) {
      if (grid[r-1][c]) {
        grid[r][c] = { color: grid[r-1][c].color }
      } else {
        grid[r][c] = null
      }
    }
  }
  
  // Buat row baru di atas (row 0)
  grid[0] = Array(COLS).fill(null)
  for (let c = 0; c < COLS; c++) {
    if (Math.random() < 0.75) {
      grid[0][c] = { color: randColor() }
    }
  }
  
  recalcMaxRow()
  
  // Cek game over setelah drop row
  if (maxRow >= DANGER_ROW) {
    endGame()
  }
}

// ── Place bullet into grid ────────────────────────────────
function placeMarble(bx, by, color) {
  let bestRow = -1, bestCol = -1, minDist = Infinity

  // Cari cell kosong terdekat
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      if (grid[r][c]) continue
      
      const { x, y } = hexCenter(c, r)
      const d = Math.hypot(bx - x, by - y)
      
      if (d < minDist && d < R * 2.2) {
        // Cek apakah cell ini adjacent ke marble lain atau row 0
        const hasNeighbor = r === 0 || hexNeighbors(r, c).some(n => {
          return grid[n.row]?.[n.col]
        })
        if (hasNeighbor) {
          minDist = d
          bestRow = r
          bestCol = c
        }
      }
    }
  }

  // Fallback: cari cell kosong terdekat tanpa syarat neighbor
  if (bestRow === -1) {
    for (let r = 0; r < ROWS; r++) {
      for (let c = 0; c < COLS; c++) {
        if (grid[r][c]) continue
        const { x, y } = hexCenter(c, r)
        const d = Math.hypot(bx - x, by - y)
        if (d < minDist && d < R * 2.5) {
          minDist = d
          bestRow = r
          bestCol = c
        }
      }
    }
  }

  if (bestRow === -1) {
    // Tidak ada tempat untuk menempatkan marble, game over
    endGame()
    return
  }

  grid[bestRow][bestCol] = { color }
  tryMatch(bestRow, bestCol)
  
  // Cek lagi game over setelah match
  if (maxRow >= DANGER_ROW) {
    endGame()
  }
}

// ── Drawing ───────────────────────────────────────────────
function drawBackground() {
  const grad = ctx.createLinearGradient(0, 0, 0, H)
  grad.addColorStop(0, '#12082a')
  grad.addColorStop(1, '#270d4e')
  ctx.fillStyle = grad
  ctx.fillRect(0, 0, W, H)

  // Subtle grid dots
  ctx.fillStyle = 'rgba(255,255,255,0.04)'
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      const { x, y } = hexCenter(c, r)
      ctx.beginPath()
      ctx.arc(x, y, 1.5, 0, Math.PI * 2)
      ctx.fill()
    }
  }

  // Danger line - perbaiki posisi Y
  const dangerY = DANGER_ROW * CELL_H + TOP_PAD - R
  ctx.save()
  ctx.strokeStyle = 'rgba(255,71,87,0.85)'
  ctx.lineWidth = 3
  ctx.setLineDash([8, 6])
  ctx.beginPath()
  ctx.moveTo(0, dangerY)
  ctx.lineTo(W, dangerY)
  ctx.stroke()
  ctx.setLineDash([])
  ctx.restore()
}

function drawMarble(x, y, colorIdx, radius) {
  radius = radius || R
  const color = marbleColors[colorIdx]

  // Shadow
  ctx.save()
  ctx.shadowColor = color
  ctx.shadowBlur = 8

  // Gradient sphere
  const g = ctx.createRadialGradient(x - radius * 0.3, y - radius * 0.3, radius * 0.1, x, y, radius)
  g.addColorStop(0, shiftLightness(color, 40))
  g.addColorStop(0.55, color)
  g.addColorStop(1, shiftLightness(color, -30))

  ctx.fillStyle = g
  ctx.beginPath()
  ctx.arc(x, y, radius, 0, Math.PI * 2)
  ctx.fill()
  ctx.restore()

  // Rim
  ctx.strokeStyle = 'rgba(255,255,255,0.2)'
  ctx.lineWidth = 1
  ctx.beginPath()
  ctx.arc(x, y, radius, 0, Math.PI * 2)
  ctx.stroke()

  // Specular
  ctx.fillStyle = 'rgba(255,255,255,0.35)'
  ctx.beginPath()
  ctx.arc(x - radius * 0.28, y - radius * 0.28, radius * 0.22, 0, Math.PI * 2)
  ctx.fill()
}

// Quick HSL lightness shift (approximate, works on hex colors)
function shiftLightness(hex, amount) {
  let r = parseInt(hex.slice(1, 3), 16)
  let g = parseInt(hex.slice(3, 5), 16)
  let b = parseInt(hex.slice(5, 7), 16)
  r = Math.max(0, Math.min(255, r + amount * 2))
  g = Math.max(0, Math.min(255, g + amount * 2))
  b = Math.max(0, Math.min(255, b + amount * 2))
  return `rgb(${r},${g},${b})`
}

function drawCannon() {
  const cx = W / 2
  const cy = CANNON_Y

  // Platform base
  const baseGrad = ctx.createRadialGradient(cx, cy, 0, cx, cy, 22)
  baseGrad.addColorStop(0, '#FF9DE2')
  baseGrad.addColorStop(1, '#FF3D9A')
  ctx.fillStyle = baseGrad
  ctx.beginPath()
  ctx.ellipse(cx, cy + 4, 22, 12, 0, 0, Math.PI * 2)
  ctx.fill()

  // Barrel
  ctx.save()
  ctx.translate(cx, cy)
  ctx.rotate(cannonAngle)

  // Barrel body
  ctx.fillStyle = '#FF9F43'
  ctx.beginPath()
  ctx.roundRect(-6, -32, 12, 32, 4)
  ctx.fill()

  // Barrel tip
  ctx.fillStyle = '#FF6EB4'
  ctx.beginPath()
  ctx.roundRect(-5, -34, 10, 8, 3)
  ctx.fill()

  ctx.restore()

  // Aim guide (dotted)
  const steps = 8
  for (let i = 1; i <= steps; i++) {
    const t = i / steps
    const gx = cx + Math.cos(cannonAngle) * (30 + t * 120)
    const gy = cy + Math.sin(cannonAngle) * (30 + t * 120)
    ctx.fillStyle = `rgba(255,255,255,${0.18 * (1 - t)})`
    ctx.beginPath()
    ctx.arc(gx, gy, 2, 0, Math.PI * 2)
    ctx.fill()
  }

  // Current marble in cannon
  drawMarble(cx, cy - 4, currentColor, R - 1)
}

function drawParticles(dt) {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]
    p.x += p.vx * dt
    p.y += p.vy * dt
    p.vy += 220 * dt
    p.life -= dt * 1.8

    if (p.life <= 0) { particles.splice(i, 1); continue }

    ctx.save()
    ctx.globalAlpha = Math.min(1, p.life)
    ctx.fillStyle = p.color
    ctx.beginPath()
    ctx.arc(p.x, p.y, p.r * p.life, 0, Math.PI * 2)
    ctx.fill()
    ctx.restore()
  }
}

function render() {
  if (!ctx) return
  drawBackground()

  // Grid marbles - FIXED: Add null check
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      const cell = grid[r]?.[c]
      if (cell && cell.color !== undefined) {
        const { x, y } = hexCenter(c, r)
        drawMarble(x, y, cell.color)
      }
    }
  }

  // Bullet in flight
  if (bullet && bullet.color !== undefined) {
    drawMarble(bullet.x, bullet.y, bullet.color)
  }

  drawCannon()
  drawParticles(1 / 60)
}

// ── Game loop ─────────────────────────────────────────────
// In update function - collision detection
function update(dt) {
  if (!gameActive) return

  // Smooth cannon rotation
  if (rotateDir !== 0) {
    cannonAngle += rotateDir * ROTATE_STEP
    cannonAngle = Math.max(MIN_ANGLE, Math.min(MAX_ANGLE, cannonAngle))
  }

  // Drop timer
  dropTimer += dt
  if (dropTimer >= dropInterval) {
    dropTimer = 0
    dropRow()
  }

  // Bullet physics
  if (bullet) {
    bullet.x += bullet.vx * dt
    bullet.y += bullet.vy * dt

    // Wall bounces
    if (bullet.x - R < 0)  { bullet.x = R;     bullet.vx =  Math.abs(bullet.vx) }
    if (bullet.x + R > W)  { bullet.x = W - R; bullet.vx = -Math.abs(bullet.vx) }

    // Hit ceiling → place
    if (bullet.y - R <= TOP_PAD - R) {
      placeMarble(bullet.x, bullet.y, bullet.color)
      bullet = null
      return
    }

    // Collision with grid marbles - FIXED: Add null checks
    for (let r = 0; r <= Math.min(maxRow + 2, ROWS - 1); r++) {
      for (let c = 0; c < COLS; c++) {
        const cell = grid[r]?.[c]
        if (!cell) continue
        const { x, y } = hexCenter(c, r)
        if (Math.hypot(bullet.x - x, bullet.y - y) < R * 1.85) {
          placeMarble(bullet.x, bullet.y, bullet.color)
          bullet = null
          return
        }
      }
    }

    // Out of bounds bottom (shouldn't normally happen)
    if (bullet.y + R > H) {
      bullet = null
    }
  }
}

function gameLoop(ts) {
  const dt = lastTs ? Math.min((ts - lastTs) / 1000, 0.033) : 0
  lastTs = ts

  update(dt)
  render()
  animId = requestAnimationFrame(gameLoop)
}

// ── Input ─────────────────────────────────────────────────
function onCanvasClick(e) {
  if (state.value !== 'play') return

  const rect = gameCanvas.value.getBoundingClientRect()
  const sx = W / rect.width
  const sy = H / rect.height

  const cx = (e.clientX ?? e.touches?.[0]?.clientX)
  const cy = (e.clientY ?? e.touches?.[0]?.clientY)
  if (cx == null) return

  const mx = (cx - rect.left) * sx
  const my = (cy - rect.top)  * sy

  const dx = mx - W / 2
  const dy = my - CANNON_Y
  cannonAngle = Math.atan2(dy, dx)
  cannonAngle = Math.max(MIN_ANGLE, Math.min(MAX_ANGLE, cannonAngle))

  fireMarble()
}

function onKeyDown(e) {
  if (state.value !== 'play') return
  if (e.key === 'ArrowLeft')  { e.preventDefault(); cannonAngle = Math.max(MIN_ANGLE, cannonAngle - ROTATE_STEP * 2) }
  if (e.key === 'ArrowRight') { e.preventDefault(); cannonAngle = Math.min(MAX_ANGLE, cannonAngle + ROTATE_STEP * 2) }
  if (e.key === ' ' || e.key === 'ArrowUp') { e.preventDefault(); fireMarble() }
}

function startRotate(dir) { rotateDir = dir }
function stopRotate()      { rotateDir = 0 }

function fireMarble() {
  if (!gameActive || bullet) return

  bullet = {
    x: W / 2,
    y: CANNON_Y - 4,
    vx: Math.cos(cannonAngle) * SPEED,
    vy: Math.sin(cannonAngle) * SPEED,
    color: currentColor
  }

  currentColor = nextMarbleColor.value
  pickNext()
  shots++
  shotsFired.value = shots
}

function pickNext() {
  nextMarbleColor.value = randColor()
}

// ── Game lifecycle ────────────────────────────────────────
function beginGame() {
  state.value     = 'play'
  score.value     = 0
  level.value     = 1
  popCount.value  = 0
  shotsFired.value = 0
  pops    = 0
  shots   = 0
  bullet  = null
  particles = []
  cannonAngle  = -Math.PI / 2
  dropTimer    = 0
  dropInterval = DROP_INTERVAL_BASE
  rotateDir    = 0
  lastTs       = 0
  gameActive   = true

  initGrid()
  currentColor = randColor()
  pickNext()
}

function endGame() {
  gameActive = false
  state.value = 'over'
  if (score.value > highScore.value) highScore.value = score.value
  bullet = null
}

// ── Lifecycle hooks ───────────────────────────────────────
onMounted(() => {
  ctx = gameCanvas.value.getContext('2d')

  // initial idle render (background)
  render()

  gameCanvas.value.addEventListener('click', onCanvasClick)
  gameCanvas.value.addEventListener('touchstart', e => {
    e.preventDefault()
    onCanvasClick(e)
  }, { passive: false })

  document.addEventListener('keydown', onKeyDown)
  animId = requestAnimationFrame(gameLoop)
})

onUnmounted(() => {
  if (animId) cancelAnimationFrame(animId)
  document.removeEventListener('keydown', onKeyDown)
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Fredoka+One&family=Nunito:wght@400;600;700;800;900&display=swap');

*, *::before, *::after {
  box-sizing: border-box;
  -webkit-tap-highlight-color: transparent;
  user-select: none;
  touch-action: manipulation;
}

.root {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  min-height: 100svh;
  background: #12082a;
  font-family: 'Nunito', 'Segoe UI', sans-serif;
  padding-bottom: 12px;
}

.frame {
  position: relative;
  width: 320px;
  flex-shrink: 0;
}

canvas {
  display: block;
  width: 320px;
  height: 480px;
  cursor: crosshair;
  border-radius: 14px;
  box-shadow: 0 0 40px rgba(255,61,154,0.25);
}

/* ── Screens ── */
.screen {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  padding-bottom: 28px;
  background: linear-gradient(to bottom, rgba(18,8,42,0) 20%, rgba(18,8,42,0.97) 52%);
  border-radius: 14px;
  pointer-events: all;
}

.screen-inner {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  gap: 0;
}

.over-inner { gap: 5px; }

.candy-deco {
  font-size: 22px;
  margin-bottom: 10px;
  animation: bounce 1.6s ease-in-out infinite;
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50%       { transform: translateY(-7px); }
}

.badge {
  font-size: 9px;
  font-weight: 800;
  letter-spacing: 0.3em;
  color: #FF9DE2;
  padding: 4px 14px;
  border: 1.5px solid rgba(255,157,226,0.3);
  border-radius: 99px;
  margin-bottom: 12px;
  background: rgba(255,110,180,0.07);
}

.badge.danger {
  color: #FF6B81;
  border-color: rgba(255,107,129,0.4);
  background: rgba(255,107,129,0.07);
}

.title-main {
  font-family: 'Fredoka One', cursive;
  font-size: 52px;
  font-weight: 400;
  color: #fff;
  text-align: center;
  line-height: 0.92;
  margin: 0 0 14px;
  text-shadow: 0 0 32px rgba(255,110,180,0.4);
}

.title-accent {
  color: #FF6EB4;
  text-shadow: 0 0 28px rgba(255,110,180,0.7);
}

.subtitle {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.06em;
  color: rgba(255,255,255,0.32);
  text-align: center;
  line-height: 2;
  margin: 0 0 24px;
}

.over-score-label {
  font-size: 9px;
  font-weight: 800;
  letter-spacing: 0.22em;
  color: rgba(255,255,255,0.28);
  margin: 6px 0 4px;
}

.over-score {
  font-family: 'Fredoka One', cursive;
  font-size: 72px;
  color: #fff;
  line-height: 1;
  text-shadow: 0 0 40px rgba(255,110,180,0.5);
  margin-bottom: 4px;
}

.hs-line {
  font-size: 11px;
  font-weight: 700;
  color: rgba(255,255,255,0.26);
  letter-spacing: 0.1em;
  margin-bottom: 14px;
}

.hs-line.gold { color: #FFD32A; text-shadow: 0 0 12px rgba(255,211,42,0.5); }

.stat-row {
  display: flex;
  border: 1.5px solid rgba(255,110,180,0.18);
  border-radius: 14px;
  overflow: hidden;
  margin-bottom: 20px;
  background: rgba(255,110,180,0.04);
}

.stat {
  padding: 10px 18px;
  text-align: center;
  border-right: 1px solid rgba(255,110,180,0.12);
}
.stat:last-child { border-right: none; }

.sv { font-family: 'Fredoka One', cursive; font-size: 22px; color: #fff; }
.sk {
  font-size: 8px;
  font-weight: 800;
  color: rgba(255,255,255,0.26);
  letter-spacing: 0.14em;
  margin-top: 2px;
}

/* ── Buttons ── */
.btn-start {
  display: flex;
  align-items: center;
  gap: 8px;
  background: linear-gradient(135deg, #FF6EB4, #FF3D9A);
  border: none;
  border-radius: 99px;
  padding: 14px 44px;
  font-family: 'Fredoka One', cursive;
  font-size: 17px;
  letter-spacing: 0.06em;
  color: #fff;
  cursor: pointer;
  box-shadow: 0 4px 26px rgba(255,61,154,0.45);
  transition: transform 0.12s, box-shadow 0.12s;
  margin-bottom: 10px;
}
.btn-start:hover { transform: translateY(-2px); box-shadow: 0 6px 34px rgba(255,61,154,0.6); }
.btn-start:active { transform: scale(0.95); }

.btn-icon { font-size: 18px; }

.hint-row {
  font-size: 9px;
  font-weight: 700;
  letter-spacing: 0.06em;
  color: rgba(255,255,255,0.16);
}

/* ── HUD ── */
.hud-overlay {
  position: absolute;
  top: 0; left: 0; right: 0;
  pointer-events: none;
  z-index: 5;
}

.hud-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 16px;
  background: linear-gradient(to bottom, rgba(18,8,42,0.9) 0%, transparent 100%);
}

.hud-cell { min-width: 64px; }
.hud-cell.center { display: flex; flex-direction: column; align-items: center; }
.hud-cell.right  { text-align: right; }

.hk {
  font-size: 8px;
  font-weight: 800;
  color: rgba(255,157,226,0.55);
  letter-spacing: 0.22em;
}

.hv {
  font-family: 'Fredoka One', cursive;
  font-size: 22px;
  color: #fff;
  line-height: 1;
  text-shadow: 0 0 12px rgba(255,110,180,0.4);
}

.next-label {
  font-size: 8px;
  font-weight: 800;
  color: rgba(255,157,226,0.45);
  letter-spacing: 0.16em;
  margin-bottom: 4px;
}

.next-marble {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  border: 2px solid rgba(255,255,255,0.28);
  box-shadow: 0 0 8px currentColor;
}

.danger-bar {
  height: 3px;
  background: linear-gradient(90deg, #FF4757, #FF6EB4, #FF4757);
  background-size: 200% 100%;
  animation: dbar 0.5s linear infinite;
  transition: width 0.3s, opacity 0.3s;
}

@keyframes dbar {
  0%   { background-position: 0% 0%; }
  100% { background-position: 200% 0%; }
}

/* ── Controls ── */
.controls {
  display: flex;
  gap: 6px;
  width: 320px;
  padding: 8px 0 4px;
}

.cb {
  flex: 1;
  height: 62px;
  background: rgba(255,110,180,0.07);
  border: 1.5px solid rgba(255,110,180,0.18);
  border-radius: 16px;
  color: rgba(255,255,255,0.65);
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  transition: transform 0.1s, background 0.1s;
}

.cb:active {
  transform: scale(0.91);
  background: rgba(255,110,180,0.18);
}

.fire-btn {
  flex: 1.8;
  background: linear-gradient(135deg, rgba(255,110,180,0.16), rgba(255,61,154,0.10));
  border-color: rgba(255,110,180,0.38);
  gap: 2px;
}

.fire-btn:active {
  background: linear-gradient(135deg, rgba(255,110,180,0.28), rgba(255,61,154,0.20));
}

.fire-icon  { font-size: 22px; line-height: 1; }
.fire-label {
  font-size: 10px;
  font-weight: 800;
  letter-spacing: 0.14em;
  color: rgba(255,157,226,0.75);
}
</style>