<template>
  <div class="game-root">
    <div class="canvas-wrapper">
      <canvas ref="gc"></canvas>

      <!-- Start Screen -->
      <div v-if="state === 'start'" class="overlay">
        <div class="title">GESHER-IN</div>
        <button class="btn" @click="beginGame">MULAI</button>
      </div>

      <!-- Game Over Screen -->
      <div v-if="state === 'over'" class="overlay">
        <div class="tag">NABRAK</div>
        <div class="over-label">SKOR</div>
        <div class="over-score">{{ score }}</div>
        <div class="over-hs" :class="{ 'new-record': score >= highScore }">
          <template v-if="score >= highScore">★ REKOR BARU ★</template>
          <template v-else>BEST: {{ highScore }}</template>
        </div>
        <div class="stats">
          <div class="stat-box">
            <div class="stat-val">{{ level }}</div>
            <div class="stat-lbl">STAGE</div>
          </div>
          <div class="stat-box">
            <div class="stat-val">{{ Math.floor(elapsed) }}s</div>
            <div class="stat-lbl">WAKTU</div>
          </div>
          <div class="stat-box">
            <div class="stat-val">{{ dodgeCount }}</div>
            <div class="stat-lbl">DODGE</div>
          </div>
        </div>
        <button class="btn" @click="beginGame">COBA LAGI</button>
      </div>
    </div>

    <!-- HUD -->
    <div class="hud">
      <div>
        <div class="hud-lbl">SKOR</div>
        <div class="hud-val">{{ score }}</div>
      </div>
      <div class="hud-center">
        <div class="hud-lbl">STAGE</div>
        <div class="hud-val">{{ level }}</div>
      </div>
      <div>
        <div class="hud-lbl">DODGE</div>
        <div class="hud-val">{{ dodgeCount }}</div>
      </div>
    </div>

    <!-- Controls: 3 lane buttons -->
    <div class="controls">
      <button
        class="ctrl-btn"
        @touchstart.prevent="setLane(0)"
        @mousedown.prevent="setLane(0)"
      >◀</button>
      <button
        class="ctrl-btn mid"
        @touchstart.prevent="setLane(1)"
        @mousedown.prevent="setLane(1)"
      >▼</button>
      <button
        class="ctrl-btn"
        @touchstart.prevent="setLane(2)"
        @mousedown.prevent="setLane(2)"
      >▶</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

// ─── Canvas ref ───────────────────────────────────────────────
const gc = ref(null)
let ctx = null
let animId = null

// ─── Reactive UI state ────────────────────────────────────────
const state      = ref('start')
const score      = ref(0)
const highScore  = ref(0)
const level      = ref(1)
const elapsed    = ref(0)
const dodgeCount = ref(0)

// ─── Canvas dimensions ────────────────────────────────────────
const W = 300
const H = 480

// ─── Lane config ─────────────────────────────────────────────
const LANES        = 3
const LANE_W       = W / LANES
const LANE_CENTERS = [LANE_W * 0.5, LANE_W * 1.5, LANE_W * 2.5]
const GROUND       = H - 80

// ─── Level progression ────────────────────────────────────────
const LEVEL_THRESHOLDS = [0, 12, 28, 48, 72, 102]
const LEVEL_SPEEDS     = [160, 220, 290, 370, 460, 560]
const LEVEL_INTERVALS  = [1.7, 1.45, 1.2, 1.0, 0.82, 0.65]

// ─── Mutable game state ───────────────────────────────────────
let obstacles    = []
let particles    = []
let roadLines    = []
let elapsedRaw   = 0
let lastTime     = 0
let spawnTimer   = 0
let spawnInterval = 1.7
let currentSpeed = 160
let warningFlash = 0
let playerLane   = 1
let targetX      = LANE_CENTERS[1]
let currentX     = LANE_CENTERS[1]
let dodges       = 0
let shakeAmt     = 0

// ─── Road init ───────────────────────────────────────────────
function initRoadLines() {
  roadLines = []
  for (let i = 0; i < 7; i++) {
    roadLines.push((H / 7) * i)
  }
}

// ─── Lane control (direct lane select — fixes touchscreen bug) ─
function setLane(lane) {
  if (state.value !== 'play') return
  if (lane === playerLane) return
  playerLane = lane
  targetX = LANE_CENTERS[lane]
  dodges++
  dodgeCount.value = dodges
}

function onKeyDown(e) {
  if (e.key === 'ArrowLeft')  { e.preventDefault(); setLane(Math.max(0, playerLane - 1)) }
  if (e.key === 'ArrowRight') { e.preventDefault(); setLane(Math.min(2, playerLane + 1)) }
  if (e.key === '1') { e.preventDefault(); setLane(0) }
  if (e.key === '2') { e.preventDefault(); setLane(1) }
  if (e.key === '3') { e.preventDefault(); setLane(2) }
}

// ─── Game lifecycle ───────────────────────────────────────────
function beginGame() {
  state.value      = 'play'
  score.value      = 0
  level.value      = 1
  elapsed.value    = 0
  dodgeCount.value = 0

  obstacles    = []
  particles    = []
  elapsedRaw   = 0
  lastTime     = 0
  spawnTimer   = 0
  spawnInterval = 1.7
  currentSpeed = 160
  warningFlash = 0
  playerLane   = 1
  targetX      = LANE_CENTERS[1]
  currentX     = LANE_CENTERS[1]
  dodges       = 0
  shakeAmt     = 0
  initRoadLines()

  if (!animId) animId = requestAnimationFrame(loop)
}

function endGame() {
  state.value   = 'over'
  elapsed.value = elapsedRaw
  if (score.value > highScore.value) highScore.value = score.value
  shakeAmt = 0.8
}

// ─── Level helpers ────────────────────────────────────────────
function getLevel(secs) {
  for (let i = LEVEL_THRESHOLDS.length - 1; i >= 0; i--) {
    if (secs >= LEVEL_THRESHOLDS[i]) return i + 1
  }
  return 1
}

function getLevelProgress(secs) {
  const lv  = getLevel(secs)
  const idx = lv - 1
  const start = LEVEL_THRESHOLDS[idx] ?? 0
  const next  = LEVEL_THRESHOLDS[idx + 1] ?? (start + 60)
  return Math.min((secs - start) / (next - start), 1)
}

// ─── Obstacle types ───────────────────────────────────────────
const OBS_TYPES = [
  { w: 36, h: 20, col: '#e05020' },
  { w: 26, h: 26, col: '#20a0d0' },
  { w: 40, h: 14, col: '#d0c020' },
  { w: 24, h: 30, col: '#d040a0' },
]

function spawnObs() {
  const used = new Set()
  const count = level.value >= 5 && Math.random() < 0.3 ? 2 : 1

  for (let c = 0; c < count; c++) {
    let lane, tries = 0
    do { lane = Math.floor(Math.random() * LANES); tries++ }
    while (used.has(lane) && tries < 8)
    used.add(lane)

    const t = OBS_TYPES[Math.floor(Math.random() * OBS_TYPES.length)]
    obstacles.push({
      lane, x: LANE_CENTERS[lane], y: -40,
      w: t.w, h: t.h, col: t.col,
      speed: currentSpeed,
    })
  }
}

// ─── Particles ────────────────────────────────────────────────
function emitParticles(x, y, col, count = 12) {
  for (let i = 0; i < count; i++) {
    const a   = Math.random() * Math.PI * 2
    const spd = 60 + Math.random() * 120
    particles.push({
      x, y,
      vx: Math.cos(a) * spd,
      vy: Math.sin(a) * spd,
      r:  2 + Math.random() * 4,
      col,
      life: 1,
    })
  }
}

// ─── Update ───────────────────────────────────────────────────
function update(dt) {
  elapsedRaw += dt
  spawnTimer += dt

  // Level up
  const newLevel = getLevel(elapsedRaw)
  if (newLevel !== level.value) {
    level.value   = newLevel
    warningFlash  = 1.5
    currentSpeed  = LEVEL_SPEEDS[Math.min(newLevel - 1, LEVEL_SPEEDS.length - 1)]
    spawnInterval = LEVEL_INTERVALS[Math.min(newLevel - 1, LEVEL_INTERVALS.length - 1)]
  }
  if (warningFlash > 0) warningFlash = Math.max(0, warningFlash - dt)

  // Spawn
  if (spawnTimer >= spawnInterval) { spawnTimer = 0; spawnObs() }

  // Player lerp
  currentX += (targetX - currentX) * Math.min(1, 16 * dt)
  if (Math.abs(currentX - targetX) < 0.5) currentX = targetX

  // Score
  score.value = Math.floor(elapsedRaw * 10 + dodges * 5)

  // Road lines
  const roadSpd = currentSpeed * 0.35
  for (let i = 0; i < roadLines.length; i++) {
    roadLines[i] += roadSpd * dt
    if (roadLines[i] > H) roadLines[i] -= H
  }

  // Shake decay
  if (shakeAmt > 0) shakeAmt = Math.max(0, shakeAmt - dt * 3)

  // Obstacles
  for (let i = obstacles.length - 1; i >= 0; i--) {
    obstacles[i].y += obstacles[i].speed * dt
    if (obstacles[i].y > H + 50) { obstacles.splice(i, 1); continue }

    // Collision
    const px = currentX, py = GROUND - 14
    const ox = obstacles[i].x, oy = obstacles[i].y
    const pw = 22, ph = 28
    const ow = obstacles[i].w * 0.7, oh = obstacles[i].h * 0.7

    if (
      px - pw / 2 < ox + ow / 2 && px + pw / 2 > ox - ow / 2 &&
      py - ph / 2 < oy + oh / 2 && py + ph / 2 > oy - oh / 2
    ) {
      emitParticles(px, py, '#e04020', 16)
      endGame()
      return
    }
  }

  // Particles
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]
    p.x   += p.vx * dt
    p.y   += p.vy * dt
    p.vy  += 200 * dt
    p.life -= dt * 1.5
    if (p.life <= 0) particles.splice(i, 1)
  }
}

// ─── Draw ─────────────────────────────────────────────────────
function draw() {
  ctx.save()
  if (shakeAmt > 0 && state.value === 'over') {
    ctx.translate(
      (Math.random() - 0.5) * shakeAmt * 6,
      (Math.random() - 0.5) * shakeAmt * 6
    )
  }

  // Background
  ctx.fillStyle = '#0a0a0a'
  ctx.fillRect(0, 0, W, H)

  // Subtle grid
  ctx.strokeStyle = 'rgba(255,255,255,0.04)'
  ctx.lineWidth = 1
  for (let y = 0; y < H; y += 20) {
    ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke()
  }

  // Moving dashed lane dividers
  ctx.strokeStyle = 'rgba(255,255,255,0.10)'
  ctx.lineWidth = 1
  ctx.setLineDash([14, 14])
  for (const ry of roadLines) {
    for (let li = 1; li < LANES; li++) {
      const x = LANE_W * li
      ctx.beginPath(); ctx.moveTo(x, ry - 8); ctx.lineTo(x, ry + 8); ctx.stroke()
    }
  }
  ctx.setLineDash([])

  // Road edges
  ctx.strokeStyle = 'rgba(255,255,255,0.15)'
  ctx.lineWidth = 2
  ctx.beginPath(); ctx.moveTo(1, 0); ctx.lineTo(1, H); ctx.stroke()
  ctx.beginPath(); ctx.moveTo(W - 1, 0); ctx.lineTo(W - 1, H); ctx.stroke()

  // Level progress bar
  const pct = getLevelProgress(elapsedRaw)
  ctx.fillStyle = 'rgba(255,255,255,0.06)'
  ctx.fillRect(0, H - 3, W, 3)
  ctx.fillStyle = 'rgba(255,255,255,0.4)'
  ctx.fillRect(0, H - 3, W * pct, 3)

  // Level-up flash
  if (warningFlash > 0) {
    const t     = warningFlash / 1.5
    const pulse = Math.abs(Math.sin(warningFlash * 8))
    ctx.fillStyle = `rgba(255,255,255,${pulse * t * 0.12})`
    ctx.fillRect(0, 0, W, H)
    ctx.fillStyle = `rgba(255,255,255,${Math.min(pulse * t, 0.9)})`
    ctx.font = '500 13px monospace'
    ctx.textAlign = 'center'
    ctx.fillText('STAGE ' + level.value, W / 2, 28)
    ctx.textAlign = 'left'
  }

  // Obstacles
  for (const o of obstacles) {
    ctx.fillStyle = o.col
    ctx.fillRect(o.x - o.w / 2, o.y - o.h / 2, o.w, o.h)
    ctx.fillStyle = 'rgba(0,0,0,0.3)'
    ctx.fillRect(o.x - o.w / 2 + 2, o.y - o.h / 2 + 2, 6, 4)
  }

  // Particles
  for (const p of particles) {
    ctx.save()
    ctx.globalAlpha = Math.max(0, p.life)
    ctx.fillStyle   = p.col
    ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2); ctx.fill()
    ctx.restore()
  }

  // Player car
  if (state.value === 'play' || state.value === 'over') {
    const px = currentX, py = GROUND - 14
    ctx.save()
    // Shadow
    ctx.fillStyle = 'rgba(255,255,255,0.06)'
    ctx.fillRect(px - 13, py - 18, 26, 36)
    // Body
    ctx.fillStyle = '#e8e8e8'
    ctx.fillRect(px - 12, py - 18, 24, 34)
    // Windshield
    ctx.fillStyle = 'rgba(0,20,40,0.8)'
    ctx.fillRect(px - 9, py - 15, 18, 12)
    // Headlights
    ctx.fillStyle = '#fffde0'
    ctx.fillRect(px - 10, py + 10, 5, 3)
    ctx.fillRect(px + 5, py + 10, 5, 3)
    // Taillights
    ctx.fillStyle = '#c02020'
    ctx.fillRect(px - 10, py - 18, 4, 3)
    ctx.fillRect(px + 6, py - 18, 4, 3)
    ctx.restore()
  }

  ctx.restore()
}

// ─── Main loop ────────────────────────────────────────────────
function loop(ts) {
  animId = requestAnimationFrame(loop)
  const dt = lastTime ? Math.min((ts - lastTime) / 1000, 0.05) : 0
  lastTime = ts

  if (state.value === 'play') update(dt)
  draw()
}

// ─── Lifecycle ────────────────────────────────────────────────
onMounted(() => {
  ctx = gc.value.getContext('2d')
  gc.value.width  = W
  gc.value.height = H
  document.addEventListener('keydown', onKeyDown)
  initRoadLines()
  animId = requestAnimationFrame(loop)
})

onUnmounted(() => {
  document.removeEventListener('keydown', onKeyDown)
  if (animId) cancelAnimationFrame(animId)
})
</script>

<style scoped>
* {
  box-sizing: border-box;
  -webkit-tap-highlight-color: transparent;
  user-select: none;
  touch-action: manipulation;
}

.game-root {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  min-height: 100svh;
  background: #111;
  font-family: 'Share Tech Mono', 'Courier New', monospace;
}

/* ── Canvas wrapper ── */
.canvas-wrapper {
  position: relative;
  width: 300px;
  margin-top: 16px;
}

canvas {
  display: block;
  width: 300px;
  height: 480px;
  border: 1px solid rgba(255, 255, 255, 0.15);
}

/* ── Overlay (start / game over) ── */
.overlay {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: rgba(10, 10, 10, 0.92);
  color: #fff;
  gap: 0;
}

.tag {
  font-size: 10px;
  letter-spacing: 0.25em;
  color: rgba(255, 255, 255, 0.4);
  margin-bottom: 8px;
}

.title {
  font-size: 52px;
  font-weight: 700;
  color: #fff;
  letter-spacing: -0.02em;
  margin-bottom: 4px;
  line-height: 1;
}

.tagline {
  font-size: 11px;
  letter-spacing: 0.15em;
  color: rgba(255, 255, 255, 0.35);
  margin-bottom: 28px;
}

/* Game Over */
.over-label {
  font-size: 10px;
  letter-spacing: 0.2em;
  color: rgba(255, 255, 255, 0.35);
  margin-bottom: 4px;
}

.over-score {
  font-size: 64px;
  font-weight: 700;
  color: #fff;
  line-height: 1;
  margin-bottom: 8px;
}

.over-hs {
  font-size: 11px;
  color: rgba(255, 255, 255, 0.4);
  margin-bottom: 20px;
  letter-spacing: 0.08em;
}

.over-hs.new-record {
  color: #fff;
}

.stats {
  display: flex;
  border: 0.5px solid rgba(255, 255, 255, 0.2);
  margin-bottom: 24px;
}

.stat-box {
  padding: 10px 18px;
  text-align: center;
  border-right: 0.5px solid rgba(255, 255, 255, 0.2);
}

.stat-box:last-child {
  border-right: none;
}

.stat-val {
  font-size: 24px;
  font-weight: 600;
  color: #fff;
}

.stat-lbl {
  font-size: 9px;
  color: rgba(255, 255, 255, 0.35);
  letter-spacing: 0.12em;
  margin-top: 2px;
}

/* ── Button ── */
.btn {
  background: transparent;
  border: 0.5px solid rgba(255, 255, 255, 0.4);
  padding: 12px 44px;
  font-family: inherit;
  font-size: 14px;
  letter-spacing: 0.1em;
  color: #fff;
  cursor: pointer;
  transition: background 0.15s;
}

.btn:hover {
  background: rgba(255, 255, 255, 0.08);
}

.btn:active {
  opacity: 0.7;
}

/* ── HUD ── */
.hud {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 300px;
  padding: 8px 14px;
  background: #0d0d0d;
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-top: none;
}

.hud-lbl {
  font-size: 9px;
  color: rgba(255, 255, 255, 0.3);
  letter-spacing: 0.18em;
}

.hud-val {
  font-size: 22px;
  font-weight: 600;
  color: #fff;
  line-height: 1;
}

.hud-center {
  text-align: center;
}

/* ── Controls ── */
.controls {
  display: flex;
  gap: 6px;
  width: 300px;
  padding: 8px 0 4px;
}

.ctrl-btn {
  flex: 1;
  height: 64px;
  background: rgba(255, 255, 255, 0.04);
  border: 0.5px solid rgba(255, 255, 255, 0.15);
  border-radius: 4px;
  color: #fff;
  font-size: 20px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  touch-action: manipulation;
  transition: background 0.1s;
}

.ctrl-btn:active {
  background: rgba(255, 255, 255, 0.12);
}

.ctrl-btn.mid {
  font-size: 14px;
  letter-spacing: 0.05em;
}
</style>