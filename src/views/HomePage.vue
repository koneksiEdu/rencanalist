<template>
  <div class="game-root">
    <!-- Scanline overlay for CRT effect -->
    <div class="scanlines"></div>

    <div class="canvas-wrapper">
      <canvas ref="gc"></canvas>

      <!-- Start Screen -->
      <div v-if="state === 'start'" class="overlay">
        <div class="be-tag">◈ ARCADE RUNNER ◈</div>
        <div class="game-title">
          <span class="title-be">BE</span>
          <span class="title-rusher">RUSHER</span>
        </div>
        <div class="tagline">DODGE. SURVIVE. DOMINATE.</div>
        <div class="key-hints">
          <span class="key">◀</span>
          <span class="key-text">MOVE</span>
          <span class="key">▶</span>
        </div>
        <button class="btn-start" @click="beginGame">
          <span class="btn-inner">RUSH IT</span>
          <span class="btn-glow"></span>
        </button>
      </div>

      <!-- Game Over Screen -->
      <div v-if="state === 'over'" class="overlay">
        <div class="over-header">
          <span class="over-dash"></span>
          <span class="over-title">CRASHED</span>
          <span class="over-dash"></span>
        </div>
        <div class="over-score-wrap">
          <div class="over-label-sm">FINAL SCORE</div>
          <div class="over-score">{{ score }}</div>
        </div>
        <div class="over-hs" :class="{ 'new-record': score >= highScore }">
          <template v-if="score >= highScore">
            ★ NEW RECORD ★
          </template>
          <template v-else>
            BEST: {{ highScore }}
          </template>
        </div>
        <div class="over-stats">
          <div class="stat-box">
            <div class="stat-val">{{ level }}</div>
            <div class="stat-lbl">STAGE</div>
          </div>
          <div class="stat-box">
            <div class="stat-val">{{ Math.floor(elapsed) }}s</div>
            <div class="stat-lbl">SURVIVED</div>
          </div>
          <div class="stat-box">
            <div class="stat-val">{{ dodgeCount }}</div>
            <div class="stat-lbl">DODGES</div>
          </div>
        </div>
        <button class="btn-start" @click="beginGame">
          <span class="btn-inner">TRY AGAIN</span>
          <span class="btn-glow"></span>
        </button>
      </div>
    </div>

    <!-- Controls Bar -->
    <div class="controls">
      <button
        class="ctrl-btn left"
        @touchstart.prevent="moveLeft"
        @mousedown.prevent="moveLeft"
      >
        <span class="ctrl-arrow">◀</span>
      </button>

      <div class="hud-center">
        <div class="hud-score-wrap">
          <div class="hud-lbl">SCORE</div>
          <div class="hud-score">{{ score }}</div>
        </div>
        <div class="hud-stage-wrap">
          <div class="hud-lbl">STAGE</div>
          <div class="hud-stage">{{ level }}</div>
        </div>
        <div class="hud-boost-wrap">
          <div class="hud-lbl">BOOST</div>
          <div class="boost-bar-outer">
            <div class="boost-bar-inner" :style="{ width: (boostCharge * 100) + '%' }"></div>
          </div>
        </div>
      </div>

      <button
        class="ctrl-btn right"
        @touchstart.prevent="moveRight"
        @mousedown.prevent="moveRight"
      >
        <span class="ctrl-arrow">▶</span>
      </button>
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
const state     = ref('start')
const score     = ref(0)
const highScore = ref(0)
const level     = ref(1)
const elapsed   = ref(0)
const dodgeCount = ref(0)
const boostCharge = ref(0)  // 0..1 boost meter

// ─── Canvas dimensions ────────────────────────────────────────
const W = 360
const H = 520

// ─── Lane config ─────────────────────────────────────────────
const LANES        = 3
const LANE_W       = W / LANES
const LANE_CENTERS = [LANE_W * 0.5, LANE_W * 1.5, LANE_W * 2.5]
const GROUND       = H - 90

// ─── Level progression (fixed: progress bar tied to real level, not speed) ──
// Each level requires N seconds of survival (cumulative)
const LEVEL_THRESHOLDS = [0, 12, 26, 44, 66, 92, 122, 158, 200, 250]
const LEVEL_SPEEDS     = [180, 240, 310, 390, 480, 570, 670, 780, 900, 1030]
const LEVEL_INTERVALS  = [1.6, 1.4, 1.2, 1.0, 0.88, 0.76, 0.65, 0.56, 0.48, 0.40]

// ─── Mutable game state ───────────────────────────────────────
let obstacles    = []
let particles    = []
let roadLines    = []
let elapsedRaw   = 0
let lastTime     = 0
let spawnTimer   = 0
let spawnInterval = 1.6
let currentSpeed = 180
let warningFlash = 0
let playerLane   = 1
let targetX      = LANE_CENTERS[1]
let currentX     = LANE_CENTERS[1]
let moving       = false
let prevLane     = 1
let boostActive  = false
let boostTimer   = 0
let boostChargeRaw = 0
let dodges       = 0
let shakeAmt     = 0

// ─── Init road lines ──────────────────────────────────────────
function initRoadLines () {
  roadLines = []
  for (let i = 0; i < 8; i++) {
    roadLines.push({ y: (H / 8) * i, speed: 1 })
  }
}

// ─── Controls ─────────────────────────────────────────────────
function moveLeft () {
  if (state.value !== 'play') return
  if (playerLane > 0) {
    prevLane = playerLane
    playerLane--
    targetX = LANE_CENTERS[playerLane]
    moving = true
    dodges++
    dodgeCount.value = dodges
    // Charge boost meter on dodge
    boostChargeRaw = Math.min(1, boostChargeRaw + 0.2)
    boostCharge.value = boostChargeRaw
    if (boostChargeRaw >= 1 && !boostActive) activateBoost()
  }
}
function moveRight () {
  if (state.value !== 'play') return
  if (playerLane < 2) {
    prevLane = playerLane
    playerLane++
    targetX = LANE_CENTERS[playerLane]
    moving = true
    dodges++
    dodgeCount.value = dodges
    boostChargeRaw = Math.min(1, boostChargeRaw + 0.2)
    boostCharge.value = boostChargeRaw
    if (boostChargeRaw >= 1 && !boostActive) activateBoost()
  }
}

function activateBoost () {
  boostActive = true
  boostTimer  = 3.0   // 3 sec boost window
  boostChargeRaw = 0
  boostCharge.value = 0
}

function onKeyDown (e) {
  if (e.key === 'ArrowLeft')  { e.preventDefault(); moveLeft()  }
  if (e.key === 'ArrowRight') { e.preventDefault(); moveRight() }
}

// ─── Game lifecycle ───────────────────────────────────────────
function beginGame () {
  state.value  = 'play'
  score.value  = 0
  level.value  = 1
  elapsed.value = 0
  dodgeCount.value = 0
  boostCharge.value = 0

  obstacles    = []
  particles    = []
  elapsedRaw   = 0
  lastTime     = 0
  spawnTimer   = 0
  spawnInterval = 1.6
  currentSpeed = 180
  warningFlash = 0
  playerLane   = 1
  prevLane     = 1
  targetX      = LANE_CENTERS[1]
  currentX     = LANE_CENTERS[1]
  moving       = false
  boostActive  = false
  boostTimer   = 0
  boostChargeRaw = 0
  dodges       = 0
  shakeAmt     = 0
  initRoadLines()

  if (!animId) requestAnimationFrame(loop)
}

function endGame () {
  state.value = 'over'
  elapsed.value = elapsedRaw
  if (score.value > highScore.value) highScore.value = score.value
  shakeAmt = 0.5
}

// ─── Level helper ─────────────────────────────────────────────
function getLevel (secs) {
  for (let i = LEVEL_THRESHOLDS.length - 1; i >= 0; i--) {
    if (secs >= LEVEL_THRESHOLDS[i]) return i + 1
  }
  return 1
}

// progress within current level (0..1) — fixed to per-level duration
function getLevelProgress (secs) {
  const lv = getLevel(secs)
  const idx = lv - 1
  const start = LEVEL_THRESHOLDS[idx] ?? 0
  const next  = LEVEL_THRESHOLDS[idx + 1] ?? (start + 60)
  return Math.min((secs - start) / (next - start), 1)
}

// ─── Obstacle types (neon street hazards) ────────────────────
const OBS_TYPES = [
  { w: 38, h: 22, c: '#ff4500', shape: 'truck'   },  // Delivery truck
  { w: 30, h: 36, c: '#00e5ff', shape: 'barrel'  },  // Oil barrel
  { w: 44, h: 18, c: '#ffea00', shape: 'spike'   },  // Road spike strip
  { w: 28, h: 28, c: '#ff00aa', shape: 'cone'    },  // Traffic cone
]

function spawnObs () {
  const usedLanes = new Set()
  // At higher levels, occasionally spawn 2 obstacles at once
  const count = level.value >= 5 && Math.random() < 0.35 ? 2 : 1

  for (let c = 0; c < count; c++) {
    let lane, tries = 0
    do { lane = Math.floor(Math.random() * LANES); tries++ }
    while (usedLanes.has(lane) && tries < 10)
    usedLanes.add(lane)

    const t = OBS_TYPES[Math.floor(Math.random() * OBS_TYPES.length)]
    // Boost mode makes obstacles spawn faster but they give more score on dodge
    const spd = boostActive ? currentSpeed * 0.7 : currentSpeed
    obstacles.push({
      lane, x: LANE_CENTERS[lane], y: -50,
      w: t.w, h: t.h, c: t.c, shape: t.shape,
      speed: spd,
    })
  }
}

// ─── Particles ────────────────────────────────────────────────
function emitParticles (x, y, c, count = 14) {
  for (let i = 0; i < count; i++) {
    const a   = Math.random() * Math.PI * 2
    const spd = 80 + Math.random() * 160
    particles.push({
      x, y,
      vx: Math.cos(a) * spd,
      vy: Math.sin(a) * spd,
      r:  2 + Math.random() * 5,
      c,
      life: 1,
      square: Math.random() < 0.4,
    })
  }
}

function emitBoostTrail (x, y) {
  for (let i = 0; i < 3; i++) {
    particles.push({
      x: x + (Math.random() - 0.5) * 10,
      y: y + 10 + Math.random() * 8,
      vx: (Math.random() - 0.5) * 30,
      vy: 60 + Math.random() * 80,
      r:  3 + Math.random() * 3,
      c:  '#ffea00',
      life: 0.6,
      square: false,
      trail: true,
    })
  }
}

// ─── Update ───────────────────────────────────────────────────
function update (dt) {
  elapsedRaw += dt
  spawnTimer += dt

  // Boost timer
  if (boostActive) {
    boostTimer -= dt
    if (boostTimer <= 0) { boostActive = false; boostTimer = 0 }
  }

  // Level up
  const newLevel = getLevel(elapsedRaw)
  if (newLevel !== level.value) {
    level.value    = newLevel
    warningFlash   = 1.8
    currentSpeed   = LEVEL_SPEEDS[Math.min(newLevel - 1, LEVEL_SPEEDS.length - 1)]
    spawnInterval  = LEVEL_INTERVALS[Math.min(newLevel - 1, LEVEL_INTERVALS.length - 1)]
  }
  if (warningFlash > 0) warningFlash = Math.max(0, warningFlash - dt)

  // Spawn
  if (spawnTimer >= spawnInterval) { spawnTimer = 0; spawnObs() }

  // Player lerp (faster response)
  currentX += (targetX - currentX) * Math.min(1, 18 * dt)
  if (Math.abs(currentX - targetX) < 0.5) { currentX = targetX; moving = false }

  // Score — boost gives 2x
  const scoreRate = boostActive ? 20 : 10
  score.value = Math.floor(elapsedRaw * scoreRate + dodges * (boostActive ? 10 : 5))

  // Road lines
  const roadSpd = currentSpeed * 0.4
  for (const ln of roadLines) {
    ln.y += roadSpd * dt
    if (ln.y > H) ln.y -= H
  }

  // Screen shake decay
  if (shakeAmt > 0) shakeAmt = Math.max(0, shakeAmt - dt * 3)

  // Obstacles
  for (let i = obstacles.length - 1; i >= 0; i--) {
    obstacles[i].y += obstacles[i].speed * dt
    if (obstacles[i].y > H + 60) { obstacles.splice(i, 1); continue }

    // Collision
    const px = currentX, py = GROUND - 18
    const ox = obstacles[i].x, oy = obstacles[i].y
    const pw = 24, ph = 28
    const ow = obstacles[i].w * 0.75, oh = obstacles[i].h * 0.75
    if (
      px - pw/2 < ox + ow/2 && px + pw/2 > ox - ow/2 &&
      py - ph/2 < oy + oh/2 && py + ph/2 > oy - oh/2
    ) {
      emitParticles(px, py, '#ff4500', 20)
      emitParticles(px, py, '#ffea00', 10)
      shakeAmt = 1
      endGame()
      return
    }
  }

  // Boost trail
  if (boostActive) emitBoostTrail(currentX, GROUND - 18)

  // Particles
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]
    p.x   += p.vx * dt
    p.y   += p.vy * dt
    if (!p.trail) p.vy += 250 * dt
    p.life -= dt * (p.trail ? 3 : 1.4)
    if (p.life <= 0) particles.splice(i, 1)
  }
}

// ─── Draw helpers ─────────────────────────────────────────────
function roundRect (x, y, w, h, r) {
  ctx.beginPath()
  ctx.moveTo(x + r, y)
  ctx.lineTo(x + w - r, y);    ctx.arcTo(x + w, y,     x + w, y + r,     r)
  ctx.lineTo(x + w, y + h - r); ctx.arcTo(x + w, y + h, x + w - r, y + h, r)
  ctx.lineTo(x + r, y + h);    ctx.arcTo(x,     y + h, x,     y + h - r, r)
  ctx.lineTo(x, y + r);        ctx.arcTo(x,     y,     x + r, y,         r)
  ctx.closePath()
}

function drawBg () {
  // Road surface
  ctx.fillStyle = '#111118'
  ctx.fillRect(0, 0, W, H)

  // Subtle grid / asphalt texture using horizontal lines
  ctx.strokeStyle = 'rgba(255,255,255,0.03)'
  ctx.lineWidth = 1
  for (let y = 0; y < H; y += 16) {
    ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke()
  }

  // Moving dashed center lines
  ctx.strokeStyle = 'rgba(255,234,0,0.18)'
  ctx.lineWidth = 2
  ctx.setLineDash([20, 20])
  for (const ln of roadLines) {
    for (let li = 1; li < LANES; li++) {
      const x = LANE_W * li
      ctx.beginPath()
      ctx.moveTo(x, ln.y - 10)
      ctx.lineTo(x, ln.y + 10)
      ctx.stroke()
    }
  }
  ctx.setLineDash([])

  // Road edges (solid neon lines)
  ctx.strokeStyle = '#ff4500'
  ctx.lineWidth = 3
  ctx.beginPath(); ctx.moveTo(2, 0); ctx.lineTo(2, H); ctx.stroke()
  ctx.beginPath(); ctx.moveTo(W - 2, 0); ctx.lineTo(W - 2, H); ctx.stroke()

  // Level progress bar (bottom) — fixed per-level progress
  const pct = getLevelProgress(elapsedRaw)
  ctx.fillStyle = 'rgba(255,255,255,0.05)'
  ctx.fillRect(0, H - 5, W, 5)
  const barGrad = ctx.createLinearGradient(0, 0, W, 0)
  barGrad.addColorStop(0, '#ff4500')
  barGrad.addColorStop(1, '#ffea00')
  ctx.fillStyle = barGrad
  ctx.fillRect(0, H - 5, W * pct, 5)

  // Stage label on bar
  ctx.fillStyle = 'rgba(255,234,0,0.6)'
  ctx.font = 'bold 9px monospace'
  ctx.textAlign = 'right'
  ctx.fillText('STG ' + level.value, W - 4, H - 8)
  ctx.textAlign = 'left'

  // Level-up flash
  if (warningFlash > 0) {
    const t     = warningFlash / 1.8
    const pulse = Math.abs(Math.sin(warningFlash * 10))
    ctx.fillStyle = `rgba(255,100,0,${pulse * t * 0.25})`
    ctx.fillRect(0, 0, W, H)

    // Flash text
    const alpha = Math.min(pulse * t, 0.95)
    ctx.fillStyle = `rgba(255,234,0,${alpha})`
    ctx.font = 'bold 22px monospace'
    ctx.textAlign = 'center'
    ctx.fillText('◈  STAGE ' + level.value + '  ◈', W / 2, 36)
    ctx.font = '11px monospace'
    ctx.fillStyle = `rgba(255,80,0,${alpha})`
    ctx.fillText('SPEED UP!', W / 2, 54)
    ctx.textAlign = 'left'
  }

  // Boost overlay
  if (boostActive) {
    const bp = boostTimer / 3.0
    ctx.fillStyle = `rgba(255,234,0,${0.04 + Math.sin(Date.now() * 0.01) * 0.02})`
    ctx.fillRect(0, 0, W, H)
    // Side speed lines
    ctx.strokeStyle = `rgba(255,234,0,0.3)`
    ctx.lineWidth = 1.5
    for (let i = 0; i < 6; i++) {
      const lx = 4 + i * 10 + Math.sin(elapsedRaw * 5 + i) * 2
      ctx.beginPath()
      ctx.moveTo(lx, 0)
      ctx.lineTo(lx, H * (0.3 + Math.random() * 0.4))
      ctx.stroke()
      const rx = W - 4 - i * 10 + Math.sin(elapsedRaw * 5 + i) * 2
      ctx.beginPath()
      ctx.moveTo(rx, 0)
      ctx.lineTo(rx, H * (0.3 + Math.random() * 0.4))
      ctx.stroke()
    }
  }
}

function drawPlayer () {
  const px = currentX, py = GROUND - 18
  ctx.save()

  if (shakeAmt > 0) {
    ctx.translate(
      (Math.random() - 0.5) * shakeAmt * 8,
      (Math.random() - 0.5) * shakeAmt * 8
    )
  }

  // Motion ghost trail
  if (moving) {
    for (let t = 1; t <= 3; t++) {
      const tx = currentX + (targetX - currentX) * (-t * 0.12)
      ctx.fillStyle = `rgba(255,234,0,${0.08 - t * 0.02})`
      roundRect(tx - 15, py - 22, 30, 40, 5)
      ctx.fill()
    }
  }

  // Car body
  const carColor = boostActive ? '#ffea00' : '#ff6a00'
  const carGlow  = boostActive ? '#fff200' : '#ff4500'

  // Glow effect
  ctx.shadowColor  = carGlow
  ctx.shadowBlur   = boostActive ? 22 : 12

  // Main body
  ctx.fillStyle = carColor
  roundRect(px - 14, py - 22, 28, 42, 6)
  ctx.fill()

  // Roof
  ctx.fillStyle = boostActive ? '#fff176' : '#ff8c00'
  roundRect(px - 10, py - 20, 20, 16, 4)
  ctx.fill()

  // Windows
  ctx.fillStyle = 'rgba(0,30,60,0.85)'
  roundRect(px - 8, py - 18, 16, 11, 3)
  ctx.fill()

  ctx.shadowBlur = 0

  // Headlights
  ctx.fillStyle = '#fffde7'
  ctx.shadowColor = '#ffea00'
  ctx.shadowBlur  = 8
  ctx.fillRect(px - 12, py + 12, 6, 4)
  ctx.fillRect(px + 6, py + 12, 6, 4)

  // Taillights
  ctx.fillStyle = '#ff1744'
  ctx.shadowColor = '#ff1744'
  ctx.shadowBlur  = 6
  ctx.fillRect(px - 12, py - 22, 5, 4)
  ctx.fillRect(px + 7, py - 22, 5, 4)

  ctx.shadowBlur = 0
  ctx.restore()
}

function drawObs (o) {
  ctx.save()
  ctx.fillStyle = o.c
  ctx.shadowColor = o.c
  ctx.shadowBlur  = 10

  if (o.shape === 'truck') {
    // Box truck
    roundRect(o.x - o.w/2, o.y - o.h/2, o.w, o.h, 4)
    ctx.fill()
    ctx.fillStyle = 'rgba(0,0,0,0.4)'
    ctx.fillRect(o.x - o.w/2 + 3, o.y - o.h/2 + 3, 10, 8)
    ctx.fillRect(o.x + o.w/2 - 13, o.y - o.h/2 + 3, 10, 8)
    ctx.fillStyle = '#fff3'
    ctx.fillRect(o.x - o.w/2 + 4, o.y + o.h/2 - 6, 5, 3)
    ctx.fillRect(o.x + o.w/2 - 9, o.y + o.h/2 - 6, 5, 3)

  } else if (o.shape === 'barrel') {
    // Cylinder barrel
    ctx.beginPath()
    ctx.ellipse(o.x, o.y, o.w/2, o.h/2, 0, 0, Math.PI * 2)
    ctx.fill()
    // Rings
    ctx.strokeStyle = 'rgba(0,0,0,0.35)'
    ctx.lineWidth = 2
    ctx.beginPath(); ctx.ellipse(o.x, o.y - o.h * 0.2, o.w/2, 4, 0, 0, Math.PI * 2); ctx.stroke()
    ctx.beginPath(); ctx.ellipse(o.x, o.y + o.h * 0.2, o.w/2, 4, 0, 0, Math.PI * 2); ctx.stroke()
    // Biohazard ☠
    ctx.fillStyle = 'rgba(0,0,0,0.5)'
    ctx.font = 'bold 14px monospace'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText('☣', o.x, o.y)
    ctx.textBaseline = 'alphabetic'

  } else if (o.shape === 'spike') {
    // Spike strip
    ctx.fillStyle = o.c
    ctx.fillRect(o.x - o.w/2, o.y - 4, o.w, 8)
    // Spikes
    ctx.fillStyle = '#ffffffcc'
    for (let s = 0; s < 5; s++) {
      const sx = o.x - o.w/2 + 4 + s * 8
      ctx.beginPath()
      ctx.moveTo(sx, o.y - 4)
      ctx.lineTo(sx + 4, o.y - 12)
      ctx.lineTo(sx + 8, o.y - 4)
      ctx.closePath()
      ctx.fill()
    }

  } else if (o.shape === 'cone') {
    // Traffic cone
    ctx.beginPath()
    ctx.moveTo(o.x, o.y - o.h/2)
    ctx.lineTo(o.x + o.w/2, o.y + o.h/2)
    ctx.lineTo(o.x - o.w/2, o.y + o.h/2)
    ctx.closePath()
    ctx.fill()
    // Stripes
    ctx.fillStyle = 'rgba(255,255,255,0.7)'
    ctx.fillRect(o.x - 8, o.y - 2, 16, 4)
    // Base
    ctx.fillStyle = '#fff6'
    ctx.fillRect(o.x - o.w/2, o.y + o.h/2 - 5, o.w, 5)
  }

  ctx.shadowBlur = 0
  ctx.restore()
}

function drawParticles () {
  for (const p of particles) {
    ctx.save()
    ctx.globalAlpha = Math.max(0, p.life)
    ctx.fillStyle   = p.c
    ctx.shadowColor = p.c
    ctx.shadowBlur  = 6
    if (p.square) {
      const s = p.r * 2
      ctx.fillRect(p.x - p.r, p.y - p.r, s, s)
    } else {
      ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2); ctx.fill()
    }
    ctx.restore()
  }
}

function drawHUD () {
  // Boost indicator (top right corner of canvas)
  if (boostActive) {
    ctx.save()
    ctx.fillStyle = '#ffea00'
    ctx.font = 'bold 11px monospace'
    ctx.textAlign = 'right'
    ctx.shadowColor = '#ffea00'
    ctx.shadowBlur  = 8
    const blink = Math.floor(elapsedRaw * 4) % 2 === 0
    if (blink) ctx.fillText('⚡ BOOST x2', W - 8, 20)
    ctx.restore()
  }
}

// ─── Main loop ────────────────────────────────────────────────
function loop (ts) {
  animId = requestAnimationFrame(loop)
  const dt = lastTime ? Math.min((ts - lastTime) / 1000, 0.05) : 0
  lastTime = ts

  if (state.value === 'play') update(dt)

  ctx.save()
  if (shakeAmt > 0 && state.value === 'over') {
    ctx.translate((Math.random() - 0.5) * shakeAmt * 10, (Math.random() - 0.5) * shakeAmt * 10)
  }
  drawBg()
  obstacles.forEach(drawObs)
  drawParticles()
  if (state.value === 'play' || state.value === 'over') drawPlayer()
  if (state.value === 'play') drawHUD()
  ctx.restore()
}

// ─── Lifecycle ────────────────────────────────────────────────
onMounted(() => {
  ctx = gc.value.getContext('2d')
  gc.value.width  = W
  gc.value.height = H
  document.addEventListener('keydown', onKeyDown)
  initRoadLines()
  requestAnimationFrame(loop)
})

onUnmounted(() => {
  document.removeEventListener('keydown', onKeyDown)
  if (animId) cancelAnimationFrame(animId)
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Barlow+Condensed:wght@400;600;700;900&family=Share+Tech+Mono&display=swap');

* {
  box-sizing: border-box;
  -webkit-tap-highlight-color: transparent;
  user-select: none;
}

.game-root {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  min-height: 100svh;
  background: #07070e;
  font-family: 'Barlow Condensed', sans-serif;
  overflow: hidden;
}

/* CRT scanlines */
.scanlines {
  position: fixed;
  inset: 0;
  background: repeating-linear-gradient(
    0deg,
    transparent,
    transparent 2px,
    rgba(0, 0, 0, 0.08) 2px,
    rgba(0, 0, 0, 0.08) 4px
  );
  pointer-events: none;
  z-index: 100;
}

.canvas-wrapper {
  position: relative;
  width: 100%;
  max-width: 400px;
  margin-top: 12px;
}

canvas {
  display: block;
  width: 100%;
  height: auto;
  border-left: 3px solid #ff4500;
  border-right: 3px solid #ff4500;
  border-top: 1px solid #333;
  touch-action: none;
}

/* ── Overlays ── */
.overlay {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: rgba(7, 7, 14, 0.94);
  color: #fff;
  gap: 0;
}

.be-tag {
  font-family: 'Share Tech Mono', monospace;
  font-size: 10px;
  letter-spacing: 0.25em;
  color: #ff4500;
  margin-bottom: 12px;
}

.game-title {
  line-height: 0.9;
  text-align: center;
  margin-bottom: 6px;
}

.title-be {
  display: block;
  font-size: 28px;
  font-weight: 600;
  color: #ffea00;
  letter-spacing: 0.3em;
}

.title-rusher {
  display: block;
  font-size: 72px;
  font-weight: 900;
  color: #ff4500;
  letter-spacing: -0.02em;
  text-shadow: 0 0 30px rgba(255,69,0,0.6), 4px 4px 0 #7a1d00;
}

.tagline {
  font-family: 'Share Tech Mono', monospace;
  font-size: 11px;
  letter-spacing: 0.15em;
  color: rgba(255,255,255,0.4);
  margin-bottom: 28px;
}

.key-hints {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 28px;
}

.key {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  border: 1.5px solid rgba(255,69,0,0.5);
  color: #ff4500;
  font-size: 14px;
  border-radius: 4px;
}

.key-text {
  font-family: 'Share Tech Mono', monospace;
  font-size: 11px;
  color: rgba(255,255,255,0.3);
  letter-spacing: 0.1em;
}

/* Game Over */
.over-header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 16px;
}

.over-dash {
  width: 36px;
  height: 2px;
  background: #ff4500;
}

.over-title {
  font-size: 20px;
  font-weight: 700;
  letter-spacing: 0.2em;
  color: #ff4500;
}

.over-score-wrap {
  text-align: center;
  margin-bottom: 8px;
}

.over-label-sm {
  font-family: 'Share Tech Mono', monospace;
  font-size: 10px;
  color: rgba(255,255,255,0.35);
  letter-spacing: 0.2em;
  margin-bottom: 2px;
}

.over-score {
  font-size: 72px;
  font-weight: 900;
  color: #ffea00;
  line-height: 1;
  text-shadow: 0 0 24px rgba(255,234,0,0.4);
}

.over-hs {
  font-family: 'Share Tech Mono', monospace;
  font-size: 12px;
  color: rgba(255,255,255,0.4);
  margin-bottom: 20px;
  letter-spacing: 0.08em;
}

.over-hs.new-record {
  color: #ffea00;
  text-shadow: 0 0 12px rgba(255,234,0,0.5);
}

.over-stats {
  display: flex;
  gap: 0;
  border: 1px solid rgba(255,69,0,0.3);
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 28px;
}

.stat-box {
  padding: 10px 18px;
  text-align: center;
  border-right: 1px solid rgba(255,69,0,0.3);
}

.stat-box:last-child { border-right: none; }

.stat-val {
  font-size: 26px;
  font-weight: 700;
  color: #ff6a00;
}

.stat-lbl {
  font-family: 'Share Tech Mono', monospace;
  font-size: 9px;
  color: rgba(255,255,255,0.3);
  letter-spacing: 0.12em;
  margin-top: 2px;
}

/* ── Button ── */
.btn-start {
  position: relative;
  background: transparent;
  border: 2px solid #ff4500;
  padding: 0;
  cursor: pointer;
  font-family: 'Barlow Condensed', sans-serif;
  overflow: hidden;
}

.btn-inner {
  display: block;
  padding: 14px 52px;
  font-size: 22px;
  font-weight: 700;
  letter-spacing: 0.12em;
  color: #ffea00;
  position: relative;
  z-index: 1;
  transition: color 0.15s;
}

.btn-glow {
  position: absolute;
  inset: 0;
  background: linear-gradient(135deg, #ff4500, #ff8c00);
  opacity: 0;
  transition: opacity 0.15s;
}

.btn-start:hover .btn-glow,
.btn-start:active .btn-glow { opacity: 1; }
.btn-start:hover .btn-inner  { color: #fff; }
.btn-start:active { transform: scale(0.97); }

/* ── Controls bar ── */
.controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  max-width: 400px;
  padding: 8px 16px 16px;
  gap: 8px;
}

.ctrl-btn {
  width: 88px;
  height: 88px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(255,69,0,0.08);
  border: 2px solid rgba(255,69,0,0.4);
  border-radius: 4px;
  color: #ff4500;
  cursor: pointer;
  font-family: inherit;
  transition: background 0.1s, border-color 0.1s;
  touch-action: manipulation;
}

.ctrl-btn:active {
  background: rgba(255,69,0,0.25);
  border-color: #ff4500;
}

.ctrl-arrow {
  font-size: 28px;
}

/* ── HUD Center ── */
.hud-center {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 6px;
}

.hud-score-wrap,
.hud-stage-wrap {
  text-align: center;
}

.hud-lbl {
  font-family: 'Share Tech Mono', monospace;
  font-size: 9px;
  color: rgba(255,255,255,0.3);
  letter-spacing: 0.18em;
}

.hud-score {
  font-size: 32px;
  font-weight: 900;
  color: #ffea00;
  line-height: 1;
  text-shadow: 0 0 12px rgba(255,234,0,0.4);
}

.hud-stage {
  font-size: 20px;
  font-weight: 700;
  color: #ff6a00;
  line-height: 1;
}

.hud-boost-wrap {
  width: 100%;
  text-align: center;
}

.boost-bar-outer {
  width: 100%;
  height: 5px;
  background: rgba(255,255,255,0.08);
  border-radius: 999px;
  margin-top: 3px;
  overflow: hidden;
}

.boost-bar-inner {
  height: 100%;
  background: linear-gradient(90deg, #ff4500, #ffea00);
  border-radius: 999px;
  transition: width 0.15s;
  box-shadow: 0 0 8px rgba(255,234,0,0.5);
}
</style>