<template>
  <div class="root">
    <!-- GAME FRAME -->
    <div class="frame">
      <canvas ref="gc"></canvas>

      <!-- START -->
      <div v-if="state === 'start'" class="screen">
        <div class="screen-inner">
          <div class="badge">ARCADE RACER</div>
          <h1 class="title-main">Halang<br><span class="title-accent">HINDAR</span></h1>
          <p class="subtitle">Hindari rintangan · Kumpulkan NOS<br>Berani pakai NOS saat berbahaya = BONUS!</p>
          <button class="btn-start" @click="beginGame">MULAI BALAPAN</button>
          <div class="hint-row">
            <span>◀ GESER</span>
            <span>NOS = RISIKO + REWARD</span>
            <span>GESER ▶</span>
          </div>
        </div>
      </div>

      <!-- GAME OVER -->
      <div v-if="state === 'over'" class="screen">
        <div class="screen-inner">
          <div class="badge danger">CRASH!</div>
          <div class="over-score-label">SKOR AKHIR</div>
          <div class="over-score">{{ score }}</div>
          <div class="hs-line" :class="{ gold: score >= highScore }">
            <template v-if="score >= highScore">★ REKOR BARU ★</template>
            <template v-else>REKOR: {{ highScore }}</template>
          </div>
          <div class="stat-row">
            <div class="stat">
              <div class="sv">{{ level }}</div>
              <div class="sk">STAGE</div>
            </div>
            <div class="stat">
              <div class="sv">{{ Math.floor(elapsed) }}s</div>
              <div class="sk">WAKTU</div>
            </div>
            <div class="stat">
              <div class="sv">{{ nosBonus }}</div>
              <div class="sk">NOS BONUS</div>
            </div>
            <div class="stat">
              <div class="sv">{{ dodgeCount }}</div>
              <div class="sk">DODGE</div>
            </div>
          </div>
          <button class="btn-start" @click="beginGame">COBA LAGI</button>
        </div>
      </div>

      <!-- IN-GAME HUD overlay -->
      <div v-if="state === 'play'" class="hud-overlay">
        <div class="hud-top">
          <div class="hud-cell">
            <div class="hk">SKOR</div>
            <div class="hv">{{ score }}</div>
          </div>
          <div class="hud-cell center">
            <div class="stage-pip" v-for="i in 6" :key="i" :class="{ lit: i <= level }"></div>
          </div>
          <div class="hud-cell right">
            <div class="hk">STAGE</div>
            <div class="hv">{{ level }}</div>
          </div>
        </div>
      </div>
    </div>

    <!-- NOS STATUS BAR -->
    <div class="nos-panel" :class="{ active: nosActive, ready: nosStack >= NOS_MAX && !nosActive }">
      <div class="nos-pips">
        <div
          v-for="i in NOS_MAX" :key="i"
          class="np"
          :class="{ f: i <= nosStack, a: nosActive, r: nosStack >= NOS_MAX && !nosActive }"
        ></div>
      </div>
      <div class="nos-info">
        <span class="nos-tag">NOS</span>
        <span class="nos-status">
          <template v-if="nosActive">⚡ AKTIF — BONUS x{{ nosMultiplier.toFixed(1) }}</template>
          <template v-else-if="nosStack >= NOS_MAX">SIAP! TEKAN BOOST</template>
          <template v-else>{{ nosStack }}/{{ NOS_MAX }} pip</template>
        </span>
        <span v-if="nosActive" class="nos-risk">⚠ BAHAYA GANDA</span>
      </div>
    </div>

    <!-- CONTROLS -->
    <div class="controls">
      <button class="cb left" @touchstart.prevent="moveLeft" @mousedown.prevent="moveLeft">
        <svg width="20" height="20" viewBox="0 0 20 20"><path d="M13 4L7 10l6 6" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round"/></svg>
      </button>

      <button
        class="cb nos"
        :class="{ 'nos-r': nosStack >= NOS_MAX && !nosActive, 'nos-a': nosActive }"
        :disabled="nosStack < NOS_MAX || nosActive"
        @touchstart.prevent="activateNos"
        @mousedown.prevent="activateNos"
      >
        <div class="nos-icon">⚡</div>
        <div class="nos-label">{{ nosActive ? 'AKTIF' : nosStack >= NOS_MAX ? 'BOOST!' : 'NOS' }}</div>
      </button>

      <button class="cb right" @touchstart.prevent="moveRight" @mousedown.prevent="moveRight">
        <svg width="20" height="20" viewBox="0 0 20 20"><path d="M7 4l6 6-6 6" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round"/></svg>
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const gc = ref(null)
let ctx = null
let animId = null

const state      = ref('start')
const score      = ref(0)
const highScore  = ref(0)
const level      = ref(1)
const elapsed    = ref(0)
const dodgeCount = ref(0)
const nosBonus   = ref(0)
const nosMultiplier = ref(1.0)

const NOS_MAX    = 5
const nosStack   = ref(0)
const nosActive  = ref(false)

const W = 320
const H = 500

const LANES        = 3
const LANE_W       = W / LANES
const LANE_CENTERS = [LANE_W * 0.5, LANE_W * 1.5, LANE_W * 2.5]
const GROUND       = H - 70

const LEVEL_THRESHOLDS = [0, 12, 28, 48, 72, 102]
const LEVEL_SPEEDS     = [170, 230, 300, 380, 470, 570]
const LEVEL_INTERVALS  = [1.65, 1.4, 1.15, 0.95, 0.78, 0.6]

let obstacles    = []
let particles    = []
let floatTexts   = []
let parallaxA    = [] // far — dots/stars
let parallaxB    = [] // mid — trees/poles
let parallaxC    = [] // near — road marks

let elapsedRaw   = 0
let lastTime     = 0
let spawnTimer   = 0
let spawnInterval = 1.65
let currentSpeed = 170
let warningFlash = 0
let playerLane   = 1
let targetX      = LANE_CENTERS[1]
let currentX     = LANE_CENTERS[1]
let dodges       = 0
let shakeAmt     = 0
let nosTimer     = 0
let nosSessionBonus = 0
const NOS_DURATION    = 3.5
const NOS_SPEED_MULT  = 2.5
const NOS_SCORE_MULT  = 3.0

function initParallax() {
  parallaxA = []
  parallaxB = []
  parallaxC = []
  for (let i = 0; i < 20; i++) parallaxA.push({ x: Math.random() * W, y: Math.random() * H, r: 1 + Math.random() })
  for (let i = 0; i < 8; i++) parallaxB.push({ x: Math.random() < 0.5 ? 5 + Math.random() * 18 : W - 5 - Math.random() * 18, y: Math.random() * H, h: 20 + Math.random() * 40 })
  for (let i = 0; i < 6; i++) parallaxC.push({ y: (H / 6) * i })
}

function moveLeft() {
  if (state.value !== 'play') return
  if (playerLane === 0) return
  playerLane--
  targetX = LANE_CENTERS[playerLane]
  dodges++
  dodgeCount.value = dodges
}

function moveRight() {
  if (state.value !== 'play') return
  if (playerLane === 2) return
  playerLane++
  targetX = LANE_CENTERS[playerLane]
  dodges++
  dodgeCount.value = dodges
}

function activateNos() {
  if (state.value !== 'play') return
  if (nosStack.value < NOS_MAX || nosActive.value) return
  nosActive.value = true
  nosStack.value  = 0
  nosTimer        = NOS_DURATION
  nosMultiplier.value = 1.0
  nosSessionBonus = 0
}

function onKeyDown(e) {
  if (e.key === 'ArrowLeft')  { e.preventDefault(); moveLeft() }
  if (e.key === 'ArrowRight') { e.preventDefault(); moveRight() }
  if (e.key === ' ' || e.key === 'n' || e.key === 'N') { e.preventDefault(); activateNos() }
}

function beginGame() {
  state.value      = 'play'
  score.value      = 0
  level.value      = 1
  elapsed.value    = 0
  dodgeCount.value = 0
  nosBonus.value   = 0
  nosStack.value   = 0
  nosActive.value  = false

  obstacles    = []
  particles    = []
  floatTexts   = []
  elapsedRaw   = 0
  lastTime     = 0
  spawnTimer   = 0
  spawnInterval = 1.65
  currentSpeed = 170
  warningFlash = 0
  playerLane   = 1
  targetX      = LANE_CENTERS[1]
  currentX     = LANE_CENTERS[1]
  dodges       = 0
  shakeAmt     = 0
  nosTimer     = 0
  nosSessionBonus = 0
  nosMultiplier.value = 1.0
  initParallax()
  if (!animId) animId = requestAnimationFrame(loop)
}

function endGame() {
  state.value      = 'over'
  elapsed.value    = elapsedRaw
  if (nosActive.value) {
    nosBonus.value += nosSessionBonus
  }
  if (score.value > highScore.value) highScore.value = score.value
  shakeAmt    = 1.0
  nosActive.value = false
}

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

const OBS_TYPES = [
  { type: 'barrel',  w: 22, h: 22, col: '#c0392b', col2: '#e74c3c' },
  { type: 'cone',    w: 18, h: 26, col: '#e67e22', col2: '#f39c12' },
  { type: 'truck',   w: 34, h: 20, col: '#2c3e50', col2: '#34495e' },
  { type: 'oil',     w: 38, h: 12, col: '#1a1a2e', col2: '#16213e' },
  { type: 'barrier', w: 44, h: 10, col: '#c0392b', col2: '#e74c3c' },
]

function spawnObs() {
  const used = new Set()
  let count = 1
  if (level.value >= 4 && Math.random() < 0.3) count = 2
  if (nosActive.value) count = Math.min(count + (Math.random() < 0.6 ? 1 : 0), LANES)

  for (let c = 0; c < count; c++) {
    let lane, tries = 0
    do { lane = Math.floor(Math.random() * LANES); tries++ } while (used.has(lane) && tries < 8)
    used.add(lane)
    const t   = OBS_TYPES[Math.floor(Math.random() * OBS_TYPES.length)]
    const spd = nosActive.value ? currentSpeed / NOS_SPEED_MULT : currentSpeed
    obstacles.push({ lane, x: LANE_CENTERS[lane], y: -50, w: t.w, h: t.h, col: t.col, col2: t.col2, type: t.type, speed: spd, dodged: false })
  }
}

function emitParticles(x, y, col, n = 14) {
  for (let i = 0; i < n; i++) {
    const a = Math.random() * Math.PI * 2
    const s = 50 + Math.random() * 130
    particles.push({ x, y, vx: Math.cos(a) * s, vy: Math.sin(a) * s, r: 2 + Math.random() * 4, col, life: 1 })
  }
}

function spawnFloatText(x, y, text, col) {
  floatTexts.push({ x, y, vy: -60, text, col, life: 1 })
}

function update(dt) {
  elapsedRaw += dt
  spawnTimer += dt

  if (nosActive.value) {
    nosTimer -= dt
    nosMultiplier.value = Math.min(nosMultiplier.value + dt * 1.2, NOS_SCORE_MULT)
    if (nosTimer <= 0) {
      nosActive.value = false
      nosTimer = 0
      nosBonus.value += nosSessionBonus
      nosSessionBonus = 0
    }
  }

  const effectiveSpeed = nosActive.value ? currentSpeed * NOS_SPEED_MULT : currentSpeed

  const newLevel = getLevel(elapsedRaw)
  if (newLevel !== level.value) {
    level.value   = newLevel
    warningFlash  = 1.5
    currentSpeed  = LEVEL_SPEEDS[Math.min(newLevel - 1, LEVEL_SPEEDS.length - 1)]
    spawnInterval = LEVEL_INTERVALS[Math.min(newLevel - 1, LEVEL_INTERVALS.length - 1)]
  }
  if (warningFlash > 0) warningFlash = Math.max(0, warningFlash - dt)
  if (spawnTimer >= spawnInterval) { spawnTimer = 0; spawnObs() }

  currentX += (targetX - currentX) * Math.min(1, 16 * dt)
  if (Math.abs(currentX - targetX) < 0.5) currentX = targetX

  // Parallax update
  const spA = effectiveSpeed * 0.08
  const spB = effectiveSpeed * 0.22
  const spC = effectiveSpeed * 0.5
  for (const p of parallaxA) { p.y += spA * dt; if (p.y > H) p.y -= H }
  for (const p of parallaxB) { p.y += spB * dt; if (p.y > H + 50) p.y -= H + 80 }
  for (const p of parallaxC) { p.y += spC * dt; if (p.y > H) p.y -= H }

  const base = Math.floor(elapsedRaw * 8)
  score.value = base + (nosActive.value ? Math.floor(nosSessionBonus) : 0) + nosBonus.value

  if (shakeAmt > 0) shakeAmt = Math.max(0, shakeAmt - dt * 4)

  for (let i = obstacles.length - 1; i >= 0; i--) {
    const o = obstacles[i]
    o.y += o.speed * dt
    if (nosActive.value) o.y += (effectiveSpeed - o.speed) * dt

    if (o.y > H + 60) {
      if (!o.dodged) {
        if (nosStack.value < NOS_MAX && !nosActive.value) nosStack.value++
        if (nosActive.value) {
          const bonus = Math.floor(50 * nosMultiplier.value)
          nosSessionBonus += bonus
          spawnFloatText(o.x, GROUND - 40, '+' + bonus, '#f1c40f')
        }
      }
      obstacles.splice(i, 1)
      continue
    }

    const px = currentX, py = GROUND - 10
    const pw = 14, ph = 22
    const ow = o.w * 0.65, oh = o.h * 0.65

    if (
      px - pw < o.x + ow / 2 && px + pw > o.x - ow / 2 &&
      py - ph < o.y + oh / 2 && py + ph > o.y - oh / 2
    ) {
      emitParticles(px, py, '#e74c3c', 18)
      endGame()
      return
    }
  }

  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]
    p.x += p.vx * dt; p.y += p.vy * dt; p.vy += 220 * dt; p.life -= dt * 1.8
    if (p.life <= 0) particles.splice(i, 1)
  }

  for (let i = floatTexts.length - 1; i >= 0; i--) {
    const f = floatTexts[i]
    f.y += f.vy * dt; f.life -= dt * 1.5
    if (f.life <= 0) floatTexts.splice(i, 1)
  }
}

function drawObstacle(o) {
  const x = o.x, y = o.y
  ctx.save()
  if (nosActive.value) { ctx.shadowColor = 'rgba(241,196,15,0.5)'; ctx.shadowBlur = 6 }

  if (o.type === 'barrel') {
    ctx.fillStyle = o.col
    ctx.beginPath(); ctx.ellipse(x, y, o.w / 2, o.h / 2, 0, 0, Math.PI * 2); ctx.fill()
    ctx.strokeStyle = o.col2; ctx.lineWidth = 2
    ctx.beginPath(); ctx.ellipse(x, y - o.h * 0.15, o.w * 0.45, o.h * 0.18, 0, 0, Math.PI * 2); ctx.stroke()

  } else if (o.type === 'cone') {
    ctx.fillStyle = o.col
    ctx.beginPath()
    ctx.moveTo(x, y - o.h / 2)
    ctx.lineTo(x - o.w / 2, y + o.h / 2)
    ctx.lineTo(x + o.w / 2, y + o.h / 2)
    ctx.closePath(); ctx.fill()
    ctx.fillStyle = '#fff'
    ctx.fillRect(x - o.w / 2, y + o.h / 6, o.w, 3)

  } else if (o.type === 'truck') {
    ctx.fillStyle = o.col2
    ctx.fillRect(x - o.w / 2, y - o.h / 2, o.w * 0.65, o.h)
    ctx.fillStyle = o.col
    ctx.fillRect(x - o.w / 2 + o.w * 0.65, y - o.h / 2, o.w * 0.35, o.h)
    ctx.fillStyle = 'rgba(100,200,255,0.5)'
    ctx.fillRect(x - o.w / 2 + o.w * 0.67, y - o.h / 2 + 2, o.w * 0.28, o.h * 0.45)
    ctx.fillStyle = '#e74c3c'
    ctx.fillRect(x - o.w / 2, y - o.h / 2, 4, 4)
    ctx.fillRect(x - o.w / 2, y + o.h / 2 - 4, 4, 4)

  } else if (o.type === 'oil') {
    ctx.fillStyle = o.col
    ctx.beginPath(); ctx.ellipse(x, y, o.w / 2, o.h / 2, 0, 0, Math.PI * 2); ctx.fill()
    ctx.fillStyle = 'rgba(80,80,200,0.3)'
    ctx.beginPath(); ctx.ellipse(x - 4, y - 2, o.w * 0.22, o.h * 0.3, -0.4, 0, Math.PI * 2); ctx.fill()
    ctx.fillStyle = 'rgba(200,80,200,0.3)'
    ctx.beginPath(); ctx.ellipse(x + 4, y + 2, o.w * 0.18, o.h * 0.28, 0.5, 0, Math.PI * 2); ctx.fill()

  } else if (o.type === 'barrier') {
    const bw = o.w, bh = o.h
    ctx.fillStyle = o.col
    ctx.fillRect(x - bw / 2, y - bh / 2, bw, bh)
    ctx.fillStyle = '#fff'
    for (let s = 0; s < 4; s++) ctx.fillRect(x - bw / 2 + s * (bw / 4), y - bh / 2, bw / 8, bh)
  }

  ctx.shadowBlur = 0
  ctx.restore()
}

function drawBike(x, y) {
  ctx.save()
  const nos = nosActive.value
  // exhaust flames
  if (nos) {
    const fl = 4 + 5 * Math.random()
    const grad = ctx.createLinearGradient(x, y + 10, x, y + 10 + fl)
    grad.addColorStop(0, '#f1c40f')
    grad.addColorStop(1, 'rgba(241,196,15,0)')
    ctx.fillStyle = grad
    ctx.fillRect(x - 5, y + 10, 10, fl)
  }

  // rear wheel
  ctx.fillStyle = nos ? '#f1c40f' : '#2c3e50'
  ctx.beginPath(); ctx.ellipse(x, y + 10, 7, 4, 0, 0, Math.PI * 2); ctx.fill()
  ctx.strokeStyle = '#95a5a6'; ctx.lineWidth = 1.5
  ctx.beginPath(); ctx.ellipse(x, y + 10, 7, 4, 0, 0, Math.PI * 2); ctx.stroke()

  // front wheel
  ctx.fillStyle = nos ? '#f1c40f' : '#2c3e50'
  ctx.beginPath(); ctx.ellipse(x, y - 14, 5, 3, 0, 0, Math.PI * 2); ctx.fill()
  ctx.strokeStyle = '#95a5a6'; ctx.lineWidth = 1.5
  ctx.beginPath(); ctx.ellipse(x, y - 14, 5, 3, 0, 0, Math.PI * 2); ctx.stroke()

  // frame / body
  ctx.fillStyle = nos ? '#f39c12' : '#e74c3c'
  ctx.beginPath()
  ctx.moveTo(x - 4, y + 8)
  ctx.lineTo(x - 6, y - 4)
  ctx.lineTo(x - 2, y - 12)
  ctx.lineTo(x + 2, y - 12)
  ctx.lineTo(x + 6, y - 4)
  ctx.lineTo(x + 4, y + 8)
  ctx.closePath(); ctx.fill()

  // seat/tank
  ctx.fillStyle = nos ? '#e67e22' : '#c0392b'
  ctx.fillRect(x - 5, y - 6, 10, 6)

  // rider body
  ctx.fillStyle = nos ? '#f1c40f' : '#ecf0f1'
  ctx.fillRect(x - 4, y - 18, 8, 10)

  // helmet
  ctx.fillStyle = nos ? '#e67e22' : '#2c3e50'
  ctx.beginPath(); ctx.arc(x, y - 21, 5, 0, Math.PI * 2); ctx.fill()
  ctx.fillStyle = 'rgba(100,200,255,0.6)'
  ctx.fillRect(x - 3, y - 24, 6, 3)

  // handlebar
  ctx.strokeStyle = '#7f8c8d'; ctx.lineWidth = 2
  ctx.beginPath(); ctx.moveTo(x - 8, y - 10); ctx.lineTo(x + 8, y - 10); ctx.stroke()

  ctx.restore()
}

function draw() {
  ctx.save()
  if (shakeAmt > 0) {
    ctx.translate((Math.random() - 0.5) * shakeAmt * 8, (Math.random() - 0.5) * shakeAmt * 8)
  }

  // Sky gradient
  const sky = ctx.createLinearGradient(0, 0, 0, H)
  if (nosActive.value) {
    sky.addColorStop(0, '#0a0510')
    sky.addColorStop(1, '#1a0820')
  } else {
    sky.addColorStop(0, '#06060f')
    sky.addColorStop(1, '#0d0d1a')
  }
  ctx.fillStyle = sky
  ctx.fillRect(0, 0, W, H)

  // Parallax A — stars/dots (far)
  ctx.fillStyle = nosActive.value ? 'rgba(241,196,15,0.5)' : 'rgba(255,255,255,0.35)'
  for (const p of parallaxA) {
    ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2); ctx.fill()
  }

  // Road surface
  ctx.fillStyle = '#0f1014'
  ctx.fillRect(LANE_W * 0.05, 0, W - LANE_W * 0.1, H)

  // Parallax B — roadside trees / poles (mid)
  for (const p of parallaxB) {
    const isLeft = p.x < W / 2
    if (nosActive.value) {
      ctx.fillStyle = 'rgba(241,196,15,0.15)'
    } else {
      ctx.fillStyle = 'rgba(46,139,87,0.35)'
    }
    // trunk
    ctx.fillRect(p.x - 2, p.y - p.h, 4, p.h)
    // foliage
    ctx.beginPath()
    ctx.arc(p.x, p.y - p.h, 8 + p.h * 0.2, 0, Math.PI * 2)
    ctx.fill()
    // glow light on pole
    ctx.fillStyle = nosActive.value ? 'rgba(241,196,15,0.4)' : 'rgba(255,200,100,0.25)'
    ctx.beginPath(); ctx.arc(isLeft ? p.x + 6 : p.x - 6, p.y - p.h + 4, 3, 0, Math.PI * 2); ctx.fill()
  }

  // Road edge lines
  ctx.strokeStyle = nosActive.value ? 'rgba(241,196,15,0.4)' : 'rgba(255,255,255,0.18)'
  ctx.lineWidth = 2
  ctx.beginPath(); ctx.moveTo(LANE_W * 0.05, 0); ctx.lineTo(LANE_W * 0.05, H); ctx.stroke()
  ctx.beginPath(); ctx.moveTo(W - LANE_W * 0.05, 0); ctx.lineTo(W - LANE_W * 0.05, H); ctx.stroke()

  // Parallax C — dashed lane marks (near)
  ctx.strokeStyle = nosActive.value ? 'rgba(241,196,15,0.25)' : 'rgba(255,255,255,0.12)'
  ctx.lineWidth   = 1.5
  ctx.setLineDash([16, 16])
  for (const p of parallaxC) {
    for (let li = 1; li < LANES; li++) {
      const x = LANE_W * li
      ctx.beginPath(); ctx.moveTo(x, p.y - 10); ctx.lineTo(x, p.y + 10); ctx.stroke()
    }
  }
  ctx.setLineDash([])

  // NOS speed lines
  if (nosActive.value) {
    const t = Date.now() / 1000
    for (let i = 0; i < 6; i++) {
      const sx = LANE_W * 0.15 + (i / 6) * (W - LANE_W * 0.3)
      const alpha = 0.05 + 0.05 * Math.sin(t * 8 + i)
      ctx.strokeStyle = `rgba(241,196,15,${alpha})`
      ctx.lineWidth = 1
      ctx.beginPath(); ctx.moveTo(sx, 0); ctx.lineTo(sx + 8, H); ctx.stroke()
    }
    // NOS timer bar top
    const frac = nosTimer / NOS_DURATION
    ctx.fillStyle = 'rgba(241,196,15,0.18)'
    ctx.fillRect(0, 0, W * frac, 5)
    ctx.fillStyle = 'rgba(241,196,15,0.8)'
    ctx.fillRect(0, 0, W * frac, 2)
  }

  // Level progress bar — bottom
  const pct = getLevelProgress(elapsedRaw)
  ctx.fillStyle = 'rgba(255,255,255,0.05)'
  ctx.fillRect(0, H - 3, W, 3)
  ctx.fillStyle = nosActive.value ? 'rgba(241,196,15,0.7)' : 'rgba(255,255,255,0.35)'
  ctx.fillRect(0, H - 3, W * pct, 3)

  // Level flash
  if (warningFlash > 0) {
    const pulse = Math.abs(Math.sin(warningFlash * 9))
    ctx.fillStyle = `rgba(255,255,255,${pulse * (warningFlash / 1.5) * 0.1})`
    ctx.fillRect(0, 0, W, H)
    ctx.fillStyle = `rgba(255,255,255,${Math.min(pulse * (warningFlash / 1.5), 0.9)})`
    ctx.font = 'bold 13px monospace'
    ctx.textAlign = 'center'
    ctx.fillText('STAGE ' + level.value, W / 2, 24)
    ctx.textAlign = 'left'
  }

  // Obstacles
  for (const o of obstacles) drawObstacle(o)

  // Particles
  for (const p of particles) {
    ctx.save()
    ctx.globalAlpha = Math.max(0, p.life)
    ctx.fillStyle   = p.col
    ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2); ctx.fill()
    ctx.restore()
  }

  // Float texts (NOS bonus)
  for (const f of floatTexts) {
    ctx.save()
    ctx.globalAlpha = Math.max(0, f.life)
    ctx.fillStyle   = f.col
    ctx.font        = 'bold 14px monospace'
    ctx.textAlign   = 'center'
    ctx.fillText(f.text, f.x, f.y)
    ctx.restore()
  }

  // Player bike
  if (state.value === 'play' || state.value === 'over') {
    drawBike(currentX, GROUND - 10)
  }

  ctx.restore()
}

function loop(ts) {
  animId = requestAnimationFrame(loop)
  const dt = lastTime ? Math.min((ts - lastTime) / 1000, 0.05) : 0
  lastTime = ts
  if (state.value === 'play') update(dt)
  draw()
}

onMounted(() => {
  ctx = gc.value.getContext('2d')
  gc.value.width  = W
  gc.value.height = H
  document.addEventListener('keydown', onKeyDown)
  initParallax()
  animId = requestAnimationFrame(loop)
})

onUnmounted(() => {
  document.removeEventListener('keydown', onKeyDown)
  if (animId) cancelAnimationFrame(animId)
})
</script>

<style scoped>
* { box-sizing: border-box; -webkit-tap-highlight-color: transparent; user-select: none; touch-action: manipulation; }

.root {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  min-height: 100svh;
  background: #06060f;
  font-family: 'Share Tech Mono', 'Courier New', monospace;
  padding-bottom: 12px;
}

/* FRAME */
.frame {
  position: relative;
  width: 320px;
  margin-top: 12px;
  flex-shrink: 0;
}

canvas {
  display: block;
  width: 320px;
  height: 500px;
  border-left: 1.5px solid rgba(255,255,255,0.08);
  border-right: 1.5px solid rgba(255,255,255,0.08);
}

/* SCREENS */
.screen {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  padding-bottom: 28px;
  background: linear-gradient(to bottom, rgba(6,6,15,0.1) 0%, rgba(6,6,15,0.97) 55%);
}

.screen-inner {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0;
  width: 100%;
}

.badge {
  font-size: 10px;
  letter-spacing: 0.3em;
  color: rgba(241,196,15,0.7);
  padding: 3px 10px;
  border: 0.5px solid rgba(241,196,15,0.25);
  margin-bottom: 14px;
}

.badge.danger { color: rgba(231,76,60,0.8); border-color: rgba(231,76,60,0.3); }

.title-main {
  font-size: 62px;
  font-weight: 700;
  color: #fff;
  text-align: center;
  line-height: 0.88;
  letter-spacing: -0.02em;
  margin: 0 0 18px;
}

.title-accent { color: #f1c40f; }

.subtitle {
  font-size: 10px;
  letter-spacing: 0.12em;
  color: rgba(255,255,255,0.35);
  text-align: center;
  line-height: 1.9;
  margin: 0 0 26px;
}

.over-score-label { font-size: 9px; letter-spacing: 0.2em; color: rgba(255,255,255,0.3); margin-bottom: 6px; }
.over-score { font-size: 76px; font-weight: 700; color: #fff; line-height: 1; margin-bottom: 8px; }

.hs-line { font-size: 11px; color: rgba(255,255,255,0.3); margin-bottom: 20px; letter-spacing: 0.08em; }
.hs-line.gold { color: #f1c40f; }

.stat-row {
  display: flex;
  border: 0.5px solid rgba(255,255,255,0.12);
  margin-bottom: 24px;
}

.stat { padding: 10px 14px; text-align: center; border-right: 0.5px solid rgba(255,255,255,0.12); }
.stat:last-child { border-right: none; }
.sv { font-size: 22px; font-weight: 600; color: #fff; }
.sk { font-size: 8px; color: rgba(255,255,255,0.28); letter-spacing: 0.12em; margin-top: 2px; }

.btn-start {
  background: transparent;
  border: 0.5px solid rgba(241,196,15,0.4);
  padding: 13px 44px;
  font-family: inherit;
  font-size: 13px;
  letter-spacing: 0.18em;
  color: #f1c40f;
  cursor: pointer;
  transition: background 0.15s, border-color 0.15s;
  animation: btn-pulse 1.8s ease-in-out infinite;
}

@keyframes btn-pulse {
  0%, 100% { border-color: rgba(241,196,15,0.4); }
  50%       { border-color: rgba(241,196,15,0.9); box-shadow: 0 0 12px rgba(241,196,15,0.15); }
}

.btn-start:hover  { background: rgba(241,196,15,0.08); }
.btn-start:active { opacity: 0.7; }

.hint-row {
  display: flex;
  justify-content: space-between;
  width: 290px;
  margin-top: 16px;
  font-size: 9px;
  letter-spacing: 0.1em;
  color: rgba(255,255,255,0.18);
}

/* HUD OVERLAY */
.hud-overlay {
  position: absolute;
  top: 0; left: 0; right: 0;
  pointer-events: none;
}

.hud-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 12px 6px;
  background: linear-gradient(to bottom, rgba(6,6,15,0.85) 0%, rgba(6,6,15,0) 100%);
}

.hud-cell { min-width: 60px; }
.hud-cell.center { display: flex; gap: 4px; align-items: center; justify-content: center; }
.hud-cell.right { text-align: right; }
.hk { font-size: 8px; color: rgba(255,255,255,0.28); letter-spacing: 0.2em; }
.hv { font-size: 20px; font-weight: 600; color: #fff; line-height: 1; }

.stage-pip {
  width: 8px;
  height: 8px;
  border: 0.5px solid rgba(255,255,255,0.15);
  background: rgba(255,255,255,0.05);
  transition: background 0.2s;
}
.stage-pip.lit { background: #f1c40f; border-color: rgba(241,196,15,0.6); }

/* NOS PANEL */
.nos-panel {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 320px;
  padding: 7px 12px;
  background: #09090f;
  border: 1px solid rgba(255,255,255,0.07);
  border-top: none;
  transition: border-color 0.2s;
}

.nos-panel.ready  { border-color: rgba(241,196,15,0.35); }
.nos-panel.active { border-color: rgba(241,196,15,0.7); background: rgba(241,196,15,0.06); }

.nos-pips { display: flex; gap: 4px; }

.np {
  width: 16px;
  height: 8px;
  background: rgba(255,255,255,0.06);
  border: 0.5px solid rgba(255,255,255,0.1);
  transition: background 0.12s;
}

.np.f { background: rgba(241,196,15,0.5); border-color: rgba(241,196,15,0.7); }
@keyframes np-r { 0%,100% { background: rgba(241,196,15,0.5); } 50% { background: rgba(241,196,15,1); } }
.np.r { animation: np-r 0.8s infinite; border-color: #f1c40f; }
@keyframes np-a { 0%,100% { background: rgba(241,196,15,0.8); } 50% { background: #fff; } }
.np.a { animation: np-a 0.18s infinite; border-color: #f1c40f; }

.nos-info { display: flex; align-items: center; gap: 8px; flex: 1; }
.nos-tag  { font-size: 9px; letter-spacing: 0.2em; color: rgba(241,196,15,0.6); }
.nos-status { font-size: 10px; color: rgba(255,255,255,0.45); letter-spacing: 0.06em; }
.nos-panel.ready  .nos-status { color: #f1c40f; }
.nos-panel.active .nos-status { color: #f1c40f; font-weight: 600; }
.nos-risk { font-size: 9px; color: rgba(231,76,60,0.8); letter-spacing: 0.05em; margin-left: auto; }

/* CONTROLS */
.controls {
  display: flex;
  gap: 5px;
  width: 320px;
  padding: 8px 0 6px;
}

.cb {
  flex: 1;
  height: 62px;
  background: rgba(255,255,255,0.02);
  border: 0.5px solid rgba(255,255,255,0.1);
  color: rgba(255,255,255,0.7);
  font-size: 18px;
  font-family: inherit;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  touch-action: manipulation;
  transition: background 0.1s;
}

.cb:active { background: rgba(255,255,255,0.08); }

/* NOS CONTROL BUTTON */
.nos {
  flex: 1.4;
  gap: 3px;
  background: rgba(241,196,15,0.03);
  border-color: rgba(241,196,15,0.15);
  cursor: not-allowed;
  opacity: 0.35;
}
.nos:disabled { cursor: not-allowed; }

.nos-icon { font-size: 20px; }
.nos-label { font-size: 10px; letter-spacing: 0.14em; color: rgba(241,196,15,0.8); }

.nos.nos-r {
  cursor: pointer;
  opacity: 1;
  background: rgba(241,196,15,0.07);
  border-color: rgba(241,196,15,0.6);
  animation: nos-btn-pulse 0.9s ease-in-out infinite;
}

@keyframes nos-btn-pulse {
  0%,100% { box-shadow: 0 0 0 0 rgba(241,196,15,0); }
  50%      { box-shadow: 0 0 0 3px rgba(241,196,15,0.25); }
}

.nos.nos-a {
  cursor: default;
  opacity: 1;
  background: rgba(241,196,15,0.14);
  border-color: rgba(241,196,15,0.85);
  animation: nos-active-flash 0.2s infinite;
}

@keyframes nos-active-flash {
  0%,100% { background: rgba(241,196,15,0.1); }
  50%      { background: rgba(241,196,15,0.22); }
}

.nos.nos-r .nos-label,
.nos.nos-a .nos-label { color: #f1c40f; }
</style>