<template>
  <div class="game-root">
    <div class="canvas-wrapper">
      <canvas ref="gc"></canvas>

      <!-- Start Screen -->
      <div v-if="state === 'start'" class="overlay">
        <div class="label">Endless Runner</div>
        <div class="title">The Dodge Rusher</div>
        <div class="subtitle">Press START to play</div>
        <button class="btn-primary" @click="beginGame">START</button>
      </div>

      <!-- Game Over Screen -->
      <div v-if="state === 'over'" class="overlay">
        <div class="over-label">GAME OVER</div>
        <div class="over-score">{{ score }}</div>
        <div class="hs-line" :class="{ new: score >= highScore }">
          {{ score >= highScore ? 'New Record!' : 'High Score: ' + highScore }}
        </div>
        <div class="level-reached">Level Reached: {{ level }}</div>
        <div class="divider"></div>
        <button class="btn-primary" @click="beginGame">Play Again</button>
      </div>
    </div>

    <!-- Controls -->
    <div class="controls">
      <button
        class="ctrl-btn"
        @touchstart.prevent="moveLeft"
        @mousedown.prevent="moveLeft"
      >&#8592;</button>

      <div class="hud-center">
        <div class="hud-label">SCORE</div>
        <div class="hud-score">{{ score }}</div>
        <div class="hud-level">{{ levelText }}</div>
      </div>

      <button
        class="ctrl-btn"
        @touchstart.prevent="moveRight"
        @mousedown.prevent="moveRight"
      >&#8594;</button>
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
const state     = ref('start')   // 'start' | 'play' | 'over'
const score     = ref(0)
const highScore = ref(0)
const level     = ref(1)
const levelText = ref('Level 1')

// ─── Canvas dimensions ────────────────────────────────────────
const W = 360
const H = 500

// ─── Lane config ─────────────────────────────────────────────
const LANES        = 3
const LANE_W       = W / LANES
const LANE_CENTERS = [LANE_W * 0.5, LANE_W * 1.5, LANE_W * 2.5]
const GROUND       = H - 80

// ─── Level progression ────────────────────────────────────────
const LEVEL_THRESHOLDS = [0, 10, 20, 35, 55, 80, 110, 150, 200]
const LEVEL_SPEEDS     = [200, 270, 340, 420, 510, 600, 700, 810, 930]
const LEVEL_INTERVALS  = [1.5, 1.3, 1.1, 0.95, 0.82, 0.7, 0.6, 0.52, 0.45]

// ─── Mutable game state (not reactive — updated every frame) ──
let obstacles    = []
let particles    = []
let elapsed      = 0
let lastTime     = 0
let spawnTimer   = 0
let spawnInterval = 1.5
let currentSpeed = 200
let warningFlash = 0
let playerLane   = 1
let targetX      = LANE_CENTERS[1]
let currentX     = LANE_CENTERS[1]
let moving       = false

// ─── Controls ─────────────────────────────────────────────────
function moveLeft () {
  if (state.value !== 'play') return
  if (playerLane > 0) { playerLane--; targetX = LANE_CENTERS[playerLane]; moving = true }
}
function moveRight () {
  if (state.value !== 'play') return
  if (playerLane < 2) { playerLane++; targetX = LANE_CENTERS[playerLane]; moving = true }
}

function onKeyDown (e) {
  if (e.key === 'ArrowLeft')  { e.preventDefault(); moveLeft()  }
  if (e.key === 'ArrowRight') { e.preventDefault(); moveRight() }
}

// ─── Game lifecycle ───────────────────────────────────────────
function beginGame () {
  state.value    = 'play'
  score.value    = 0
  level.value    = 1
  levelText.value = 'Level 1'

  obstacles    = []
  particles    = []
  elapsed      = 0
  lastTime     = 0
  spawnTimer   = 0
  spawnInterval = 1.5
  currentSpeed = 200
  warningFlash = 0
  playerLane   = 1
  targetX      = LANE_CENTERS[1]
  currentX     = LANE_CENTERS[1]
  moving       = false

  if (!animId) requestAnimationFrame(loop)
}

function endGame () {
  state.value = 'over'
  if (score.value > highScore.value) highScore.value = score.value
}

// ─── Level helper ─────────────────────────────────────────────
function getLevel (secs) {
  for (let i = LEVEL_THRESHOLDS.length - 1; i >= 0; i--) {
    if (secs >= LEVEL_THRESHOLDS[i]) return i + 1
  }
  return 1
}

// ─── Spawn obstacle ───────────────────────────────────────────
const OBS_TYPES = [
  { w: 36, h: 34, c: '#f43f5e', shape: 'rect'    },
  { w: 34, h: 34, c: '#f97316', shape: 'diamond'  },
  { w: 32, h: 32, c: '#8b5cf6', shape: 'rect'    },
  { w: 38, h: 28, c: '#06b6d4', shape: 'rect'    },
]

function spawnObs () {
  const usedLanes = new Set()
  const count = level.value >= 6 && Math.random() < 0.3 ? 2 : 1

  for (let c = 0; c < count; c++) {
    let lane, tries = 0
    do { lane = Math.floor(Math.random() * LANES); tries++ }
    while (usedLanes.has(lane) && tries < 10)
    usedLanes.add(lane)

    const t = OBS_TYPES[Math.floor(Math.random() * OBS_TYPES.length)]
    obstacles.push({
      lane, x: LANE_CENTERS[lane], y: -40,
      w: t.w, h: t.h, c: t.c, shape: t.shape,
      speed: currentSpeed,
    })
  }
}

// ─── Particles ────────────────────────────────────────────────
function emitParticles (x, y, c) {
  for (let i = 0; i < 12; i++) {
    const a   = Math.random() * Math.PI * 2
    const spd = 60 + Math.random() * 120
    particles.push({ x, y, vx: Math.cos(a) * spd, vy: Math.sin(a) * spd, r: 3 + Math.random() * 4, c, life: 1 })
  }
}

// ─── Update ───────────────────────────────────────────────────
function update (dt) {
  elapsed   += dt
  spawnTimer += dt

  // Level up
  const newLevel = getLevel(elapsed)
  if (newLevel !== level.value) {
    level.value     = newLevel
    levelText.value = 'Level ' + newLevel
    warningFlash    = 1.5
    currentSpeed    = LEVEL_SPEEDS[Math.min(newLevel - 1, LEVEL_SPEEDS.length - 1)]
    spawnInterval   = LEVEL_INTERVALS[Math.min(newLevel - 1, LEVEL_INTERVALS.length - 1)]
  }
  if (warningFlash > 0) warningFlash = Math.max(0, warningFlash - dt)

  // Spawn
  if (spawnTimer >= spawnInterval) { spawnTimer = 0; spawnObs() }

  // Player lerp
  currentX += (targetX - currentX) * Math.min(1, 14 * dt)
  if (Math.abs(currentX - targetX) < 0.5) { currentX = targetX; moving = false }

  // Score
  score.value = Math.floor(elapsed * 10)

  // Obstacles
  for (let i = obstacles.length - 1; i >= 0; i--) {
    obstacles[i].y += obstacles[i].speed * dt
    if (obstacles[i].y > H + 50) { obstacles.splice(i, 1); continue }

    // Collision (AABB shrunk slightly)
    const px = currentX, py = GROUND - 16
    const ox = obstacles[i].x, oy = obstacles[i].y
    const pw = 26, ph = 28
    const ow = obstacles[i].w * 0.8, oh = obstacles[i].h * 0.8
    if (px - pw/2 < ox + ow/2 && px + pw/2 > ox - ow/2 &&
        py - ph/2 < oy + oh/2 && py + ph/2 > oy - oh/2) {
      emitParticles(px, py, '#ec4899')
      endGame()
      return
    }
  }

  // Particles
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]
    p.x    += p.vx * dt
    p.y    += p.vy * dt
    p.vy   += 200 * dt
    p.life -= dt * 1.5
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
  ctx.fillStyle = '#08080f'
  ctx.fillRect(0, 0, W, H)

  // Dashed lane dividers
  ctx.strokeStyle = 'rgba(255,255,255,0.06)'
  ctx.lineWidth = 2
  ctx.setLineDash([12, 18])
  for (let i = 1; i < LANES; i++) {
    const x = LANE_W * i
    ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, H); ctx.stroke()
  }
  ctx.setLineDash([])

  // Level progress bar
  const pct = Math.min((elapsed % 10) / 10, 1)
  ctx.fillStyle = 'rgba(255,255,255,0.04)'
  ctx.fillRect(0, H - 4, W, 4)
  ctx.fillStyle = 'rgba(167,139,250,0.5)'
  ctx.fillRect(0, H - 4, W * pct, 4)

  // Level-up flash
  if (warningFlash > 0) {
    const alpha = Math.abs(Math.sin(warningFlash * 8)) * warningFlash * 0.3
    ctx.fillStyle = `rgba(255,120,0,${alpha})`
    ctx.fillRect(0, 0, W, H)
    ctx.fillStyle = `rgba(255,180,0,${Math.min(alpha * 2, 0.9)})`
    ctx.font = 'bold 15px system-ui'
    ctx.textAlign = 'center'
    ctx.fillText('LEVEL UP!  x' + level.value, W / 2, 30)
    ctx.textAlign = 'left'
  }
}

function drawPlayer () {
  const px = currentX, py = GROUND - 16
  ctx.save()

  // Ghost trail when moving
  if (moving) {
    ctx.fillStyle = 'rgba(167,139,250,0.12)'
    for (let t = 1; t <= 3; t++) {
      const tx = currentX + (targetX - currentX) * (-t * 0.15)
      roundRect(tx - 14 + t, py - 14 + t * 2, 28 - t * 2, 28 - t * 2, 7)
      ctx.fill()
    }
  }

  // Body
  const g = ctx.createLinearGradient(px - 16, py - 16, px + 16, py + 16)
  g.addColorStop(0, '#a78bfa'); g.addColorStop(1, '#ec4899')
  ctx.fillStyle = g
  roundRect(px - 14, py - 14, 28, 28, 7)
  ctx.fill()

  // Eyes
  ctx.fillStyle = 'rgba(255,255,255,0.9)'
  ctx.fillRect(px - 8, py - 8,  6, 6)
  ctx.fillRect(px + 2, py - 8,  6, 6)

  // Mouth
  ctx.fillStyle = 'rgba(255,255,255,0.5)'
  ctx.fillRect(px - 7, py + 4, 14, 3)

  ctx.restore()
}

function drawObs (o) {
  ctx.save()
  ctx.fillStyle = o.c
  if (o.shape === 'diamond') {
    ctx.beginPath()
    ctx.moveTo(o.x,           o.y - o.h / 2)
    ctx.lineTo(o.x + o.w / 2, o.y)
    ctx.lineTo(o.x,           o.y + o.h / 2)
    ctx.lineTo(o.x - o.w / 2, o.y)
    ctx.closePath(); ctx.fill()
  } else {
    roundRect(o.x - o.w / 2, o.y - o.h / 2, o.w, o.h, 5)
    ctx.fill()
    ctx.fillStyle = 'rgba(255,255,255,0.6)'
    ctx.fillRect(o.x - o.w / 2 + 4, o.y - o.h / 2 + 5, 5, 5)
    ctx.fillRect(o.x + o.w / 2 - 9, o.y - o.h / 2 + 5, 5, 5)
  }
  ctx.restore()
}

function drawParticles () {
  for (const p of particles) {
    ctx.save()
    ctx.globalAlpha = Math.max(0, p.life)
    ctx.fillStyle   = p.c
    ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2); ctx.fill()
    ctx.restore()
  }
}

function drawHUD () {
  ctx.fillStyle = 'rgba(255,255,255,0.25)'
  ctx.font      = '500 12px system-ui'
  ctx.textAlign = 'right'
  ctx.fillText(Math.floor(elapsed) + 's', W - 10, 20)
  ctx.textAlign = 'left'
}

// ─── Main loop ────────────────────────────────────────────────
function loop (ts) {
  animId = requestAnimationFrame(loop)
  const dt = lastTime ? Math.min((ts - lastTime) / 1000, 0.05) : 0
  lastTime = ts

  if (state.value === 'play') update(dt)

  drawBg()
  obstacles.forEach(drawObs)
  drawParticles()
  if (state.value === 'play' || state.value === 'over') drawPlayer()
  if (state.value === 'play') drawHUD()
}

// ─── Lifecycle ────────────────────────────────────────────────
onMounted(() => {
  ctx = gc.value.getContext('2d')
  gc.value.width  = W
  gc.value.height = H
  document.addEventListener('keydown', onKeyDown)
  requestAnimationFrame(loop)
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
}

.game-root {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  min-height: 100svh;
  background: linear-gradient(135deg, #1a0533 0%, #0d0d1f 60%, #001a33 100%);
  padding: 16px 0 8px;
}

.canvas-wrapper {
  position: relative;
  width: 100%;
  max-width: 400px;
}

canvas {
  display: block;
  width: 100%;
  height: auto;
  border-radius: 12px;
  border: 2px solid rgba(255, 255, 255, 0.1);
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
  border-radius: 12px;
  background: rgba(8, 8, 18, 0.93);
  color: #fff;
}

.label {
  font-size: 11px;
  letter-spacing: 0.2em;
  color: rgba(167, 139, 250, 0.85);
  text-transform: uppercase;
  margin-bottom: 10px;
}

.title {
  font-size: 36px;
  font-weight: 700;
  letter-spacing: -0.02em;
  margin-bottom: 4px;
}

.subtitle {
  font-size: 13px;
  color: rgba(255, 255, 255, 0.45);
  margin-bottom: 32px;
}

.features {
  display: flex;
  gap: 28px;
  margin-bottom: 32px;
}

.feature {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 6px;
  text-align: center;
}

.feature span {
  font-size: 22px;
  line-height: 1;
}

.feature small {
  font-size: 11px;
  color: rgba(255, 255, 255, 0.4);
  line-height: 1.5;
}

/* Game Over */
.over-label {
  font-size: 13px;
  letter-spacing: 0.1em;
  color: rgba(255, 255, 255, 0.45);
  margin-bottom: 6px;
}

.over-score {
  font-size: 52px;
  font-weight: 700;
  line-height: 1;
}

.hs-line {
  font-size: 13px;
  color: rgba(255, 255, 255, 0.5);
  margin-top: 8px;
  margin-bottom: 4px;
}

.hs-line.new {
  color: #a78bfa;
}

.level-reached {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.3);
  margin-bottom: 10px;
}

.divider {
  width: 40px;
  height: 1.5px;
  background: rgba(255, 255, 255, 0.1);
  margin-bottom: 28px;
}

/* ── Buttons ── */
.btn-primary {
  background: linear-gradient(135deg, #a78bfa, #ec4899);
  border: none;
  padding: 14px 48px;
  font-size: 18px;
  font-weight: 600;
  color: #fff;
  border-radius: 999px;
  cursor: pointer;
  font-family: inherit;
  letter-spacing: 0.02em;
  transition: transform 0.1s;
}

.btn-primary:active {
  transform: scale(0.96);
}

/* ── Controls bar ── */
.controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  max-width: 400px;
  padding: 12px 20px;
  margin-top: 6px;
}

.ctrl-btn {
  width: 80px;
  height: 80px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 30px;
  background: rgba(255, 255, 255, 0.08);
  border: 1.5px solid rgba(255, 255, 255, 0.18);
  border-radius: 50%;
  color: #fff;
  cursor: pointer;
  font-family: inherit;
  transition: background 0.1s;
  touch-action: manipulation;
}

.ctrl-btn:active {
  background: rgba(255, 255, 255, 0.22);
}

/* ── HUD center ── */
.hud-center {
  text-align: center;
  line-height: 1.3;
  color: #fff;
}

.hud-label {
  font-size: 10px;
  color: rgba(255, 255, 255, 0.3);
  letter-spacing: 0.1em;
}

.hud-score {
  font-size: 24px;
  font-weight: 600;
}

.hud-level {
  font-size: 10px;
  color: rgba(167, 139, 250, 0.7);
  margin-top: 1px;
}
</style>