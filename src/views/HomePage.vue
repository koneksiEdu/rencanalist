<template>
  <div class="root">
    <div class="frame">
      <canvas ref="gc"></canvas>

      <!-- START -->
      <div v-if="state === 'start'" class="screen">
        <div class="screen-inner">
          <div class="badge">SPACE ARCADE</div>
          <h1 class="title-main">To Space To<br><span class="title-accent">Dodge</span></h1>
          <p class="subtitle">Hindari asteroid · Tembak untuk poin bonus<br>Kumpulkan ammo dari puing yang lolos!</p>
          <button class="btn-start" @click="beginGame">MULAI MISI</button>
          <div class="hint-row">
            <span>◀ GESER</span>
            <span>🔫 TEMBAK ASTEROID</span>
            <span>GESER ▶</span>
          </div>
        </div>
      </div>

      <!-- GAME OVER -->
      <div v-if="state === 'over'" class="screen">
        <div class="screen-inner">
          <div class="badge danger">KAPAL HANCUR!</div>
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
              <div class="sv">{{ shootBonus }}</div>
              <div class="sk">TEMBAK</div>
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

    <!-- AMMO STATUS BAR -->
    <div class="ammo-panel" :class="{ ready: ammoStack >= 1 }">
      <div class="ammo-pips">
        <div
          v-for="i in AMMO_MAX" :key="i"
          class="ap"
          :class="{ f: i <= ammoStack, r: i <= ammoStack && ammoStack >= 1 }"
        ></div>
      </div>
      <div class="ammo-info">
        <span class="ammo-tag">AMMO</span>
        <span class="ammo-status">
          <template v-if="ammoStack >= AMMO_MAX">PENUH! TEMBAK!</template>
          <template v-else-if="ammoStack >= 1">{{ ammoStack }}/{{ AMMO_MAX }} — SIAP TEMBAK</template>
          <template v-else>Tunggu asteroid lewat</template>
        </span>
        <span class="ammo-bonus" v-if="lastShotBonus > 0">+{{ lastShotBonus }}</span>
      </div>
    </div>

    <!-- CONTROLS -->
    <div class="controls">
      <button class="cb left" @touchstart.prevent="moveLeft" @mousedown.prevent="moveLeft">
        <svg width="20" height="20" viewBox="0 0 20 20"><path d="M13 4L7 10l6 6" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round"/></svg>
      </button>

      <button
        class="cb shoot"
        :class="{ 'shoot-r': ammoStack >= 1, 'shoot-empty': ammoStack < 1 }"
        :disabled="ammoStack < 1"
        @touchstart.prevent="shootBullet"
        @mousedown.prevent="shootBullet"
      >
        <div class="shoot-icon">⚡</div>
        <div class="shoot-label">{{ ammoStack >= 1 ? 'TEMBAK!' : 'AMMO: ' + ammoStack + '/' + AMMO_MAX }}</div>
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
const shootBonus = ref(0)
const lastShotBonus = ref(0)

const AMMO_MAX   = 5
const ammoStack  = ref(0)

const W = 320
const H = 500

const LANES        = 3
const LANE_W       = W / LANES
const LANE_CENTERS = [LANE_W * 0.5, LANE_W * 1.5, LANE_W * 2.5]
const SHIP_Y       = H - 70

const LEVEL_THRESHOLDS = [0, 12, 28, 48, 72, 102]
const LEVEL_SPEEDS     = [150, 200, 270, 340, 420, 510]
const LEVEL_INTERVALS  = [1.7, 1.45, 1.2, 0.98, 0.8, 0.62]

let asteroids    = []
let bullets      = []
let particles    = []
let floatTexts   = []
let stars        = []    // far parallax
let nebulae      = []    // mid parallax - nebula clouds
let dustLines    = []    // near parallax - speed lines

let elapsedRaw   = 0
let lastTime     = 0
let spawnTimer   = 0
let spawnInterval = 1.7
let currentSpeed = 150
let warningFlash = 0
let playerLane   = 1
let targetX      = LANE_CENTERS[1]
let currentX     = LANE_CENTERS[1]
let dodges       = 0
let shakeAmt     = 0
let lastBonusTimer = 0

function initParallax() {
  stars   = []
  nebulae = []
  dustLines = []
  // Stars — various sizes and brightness
  for (let i = 0; i < 60; i++) {
    stars.push({
      x: Math.random() * W,
      y: Math.random() * H,
      r: 0.4 + Math.random() * 1.8,
      twinkle: Math.random() * Math.PI * 2,
      speed: 0.05 + Math.random() * 0.15
    })
  }
  // Nebulae — large soft blobs for depth
  for (let i = 0; i < 5; i++) {
    nebulae.push({
      x: 20 + Math.random() * (W - 40),
      y: Math.random() * H,
      rx: 30 + Math.random() * 50,
      ry: 20 + Math.random() * 35,
      hue: [240, 280, 200, 320, 260][i],
      alpha: 0.04 + Math.random() * 0.06,
      speed: 0.3 + Math.random() * 0.4
    })
  }
  // Dust/speed streaks
  for (let i = 0; i < 8; i++) {
    dustLines.push({
      x: Math.random() * W,
      y: Math.random() * H,
      len: 8 + Math.random() * 20
    })
  }
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

function shootBullet() {
  if (state.value !== 'play') return
  if (ammoStack.value < 1) return
  ammoStack.value--
  bullets.push({
    x: currentX,
    y: SHIP_Y - 20,
    vy: -480,
    life: 1,
    w: 4,
    h: 14
  })
  // Muzzle flash particles
  emitParticles(currentX, SHIP_Y - 25, '#7dd3fc', 6)
}

function onKeyDown(e) {
  if (e.key === 'ArrowLeft')  { e.preventDefault(); moveLeft() }
  if (e.key === 'ArrowRight') { e.preventDefault(); moveRight() }
  if (e.key === ' ' || e.key === 'z' || e.key === 'Z') { e.preventDefault(); shootBullet() }
}

function beginGame() {
  state.value      = 'play'
  score.value      = 0
  level.value      = 1
  elapsed.value    = 0
  dodgeCount.value = 0
  shootBonus.value = 0
  ammoStack.value  = 0
  lastShotBonus.value = 0

  asteroids    = []
  bullets      = []
  particles    = []
  floatTexts   = []
  elapsedRaw   = 0
  lastTime     = 0
  spawnTimer   = 0
  spawnInterval = 1.7
  currentSpeed = 150
  warningFlash = 0
  playerLane   = 1
  targetX      = LANE_CENTERS[1]
  currentX     = LANE_CENTERS[1]
  dodges       = 0
  shakeAmt     = 0
  lastBonusTimer = 0
  initParallax()
  if (!animId) animId = requestAnimationFrame(loop)
}

function endGame() {
  state.value  = 'over'
  elapsed.value = elapsedRaw
  if (score.value > highScore.value) highScore.value = score.value
  shakeAmt = 1.0
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

// Asteroid shapes: different rock silhouettes
const ASTEROID_TYPES = [
  { type: 'small',  w: 20, h: 20, pts: 7, col: '#6b7280', col2: '#9ca3af', hp: 1 },
  { type: 'medium', w: 30, h: 28, pts: 8, col: '#4b5563', col2: '#6b7280', hp: 2 },
  { type: 'large',  w: 42, h: 38, pts: 9, col: '#374151', col2: '#4b5563', hp: 3 },
  { type: 'iron',   w: 24, h: 24, pts: 6, col: '#7c3aed', col2: '#a78bfa', hp: 1 }, // purple = special
  { type: 'ice',    w: 26, h: 22, pts: 8, col: '#0e7490', col2: '#67e8f9', hp: 1 }, // cyan = ice
]

// Pre-generate asteroid polygon points
function genAsteroidPts(cx, cy, r, count) {
  const pts = []
  for (let i = 0; i < count; i++) {
    const angle = (i / count) * Math.PI * 2
    const dist = r * (0.65 + Math.random() * 0.35)
    pts.push([cx + Math.cos(angle) * dist, cy + Math.sin(angle) * dist])
  }
  return pts
}

function spawnAsteroid() {
  const used = new Set()
  let count = 1
  if (level.value >= 3 && Math.random() < 0.25) count = 2
  if (level.value >= 5 && Math.random() < 0.2)  count = Math.min(count + 1, LANES)

  for (let c = 0; c < count; c++) {
    let lane, tries = 0
    do { lane = Math.floor(Math.random() * LANES); tries++ } while (used.has(lane) && tries < 8)
    used.add(lane)

    const t = ASTEROID_TYPES[Math.floor(Math.random() * ASTEROID_TYPES.length)]
    const rx = Math.max(t.w, t.h) / 2
    const pts = genAsteroidPts(0, 0, rx, t.pts)
    const rot = Math.random() * Math.PI * 2
    const rotSpeed = (Math.random() - 0.5) * 1.2

    asteroids.push({
      lane,
      x: LANE_CENTERS[lane],
      y: -60,
      w: t.w, h: t.h,
      col: t.col, col2: t.col2,
      type: t.type,
      hp: t.hp,
      maxHp: t.hp,
      speed: currentSpeed,
      pts,
      rot,
      rotSpeed,
      dodged: false,
      hitFlash: 0
    })
  }
}

function emitParticles(x, y, col, n = 14) {
  for (let i = 0; i < n; i++) {
    const a = Math.random() * Math.PI * 2
    const s = 40 + Math.random() * 120
    particles.push({ x, y, vx: Math.cos(a) * s, vy: Math.sin(a) * s, r: 1.5 + Math.random() * 3.5, col, life: 1 })
  }
}

function emitAsteroidDebris(x, y, col) {
  for (let i = 0; i < 10; i++) {
    const a = Math.random() * Math.PI * 2
    const s = 30 + Math.random() * 80
    const pts = genAsteroidPts(0, 0, 3 + Math.random() * 6, 5)
    particles.push({ x, y, vx: Math.cos(a) * s, vy: Math.sin(a) * s, r: 0, col, life: 1, pts, rot: Math.random() * Math.PI })
  }
}

function spawnFloatText(x, y, text, col) {
  floatTexts.push({ x, y, vy: -55, text, col, life: 1 })
}

function update(dt) {
  elapsedRaw += dt
  spawnTimer += dt
  if (lastBonusTimer > 0) lastBonusTimer -= dt

  const newLevel = getLevel(elapsedRaw)
  if (newLevel !== level.value) {
    level.value   = newLevel
    warningFlash  = 1.5
    currentSpeed  = LEVEL_SPEEDS[Math.min(newLevel - 1, LEVEL_SPEEDS.length - 1)]
    spawnInterval = LEVEL_INTERVALS[Math.min(newLevel - 1, LEVEL_INTERVALS.length - 1)]
  }
  if (warningFlash > 0) warningFlash = Math.max(0, warningFlash - dt)
  if (spawnTimer >= spawnInterval) { spawnTimer = 0; spawnAsteroid() }

  currentX += (targetX - currentX) * Math.min(1, 18 * dt)
  if (Math.abs(currentX - targetX) < 0.5) currentX = targetX

  // Parallax
  const t = elapsedRaw
  for (const s of stars) {
    s.y += s.speed * currentSpeed * 0.04 * dt
    s.twinkle += dt * 2
    if (s.y > H) s.y -= H
  }
  for (const n of nebulae) {
    n.y += n.speed * dt
    if (n.y > H + 60) n.y -= H + 80
  }
  for (const d of dustLines) {
    d.y += currentSpeed * 0.6 * dt
    if (d.y > H) d.y -= H
  }

  score.value = Math.floor(elapsedRaw * 8) + shootBonus.value

  if (shakeAmt > 0) shakeAmt = Math.max(0, shakeAmt - dt * 4)

  // Update bullets
  for (let i = bullets.length - 1; i >= 0; i--) {
    const b = bullets[i]
    b.y += b.vy * dt
    if (b.y < -20) { bullets.splice(i, 1); continue }

    // Check bullet vs asteroids
    let hit = false
    for (let j = asteroids.length - 1; j >= 0; j--) {
      const a = asteroids[j]
      const dx = Math.abs(b.x - a.x), dy = Math.abs(b.y - a.y)
      if (dx < a.w / 2 + 4 && dy < a.h / 2 + 4) {
        a.hp--
        a.hitFlash = 0.18
        hit = true
        if (a.hp <= 0) {
          const bonus = a.maxHp * 50 + (a.type === 'iron' ? 30 : 0) + (a.type === 'ice' ? 20 : 0)
          shootBonus.value += bonus
          lastShotBonus.value = bonus
          lastBonusTimer = 1.5
          score.value = Math.floor(elapsedRaw * 8) + shootBonus.value
          emitAsteroidDebris(a.x, a.y, a.col2)
          spawnFloatText(a.x, a.y - 10, '+' + bonus, '#fde68a')
          asteroids.splice(j, 1)
        }
        break
      }
    }
    if (hit) { bullets.splice(i, 1); continue }
  }

  // Update asteroids
  for (let i = asteroids.length - 1; i >= 0; i--) {
    const o = asteroids[i]
    o.y += o.speed * dt
    o.rot += o.rotSpeed * dt
    if (o.hitFlash > 0) o.hitFlash -= dt

    if (o.y > H + 60) {
      if (!o.dodged) {
        // Give ammo when asteroid passes
        if (ammoStack.value < AMMO_MAX) ammoStack.value++
      }
      asteroids.splice(i, 1)
      continue
    }

    // Collision with ship
    const px = currentX, py = SHIP_Y
    const pw = 12, ph = 20
    const ow = o.w * 0.6, oh = o.h * 0.6
    if (
      px - pw < o.x + ow / 2 && px + pw > o.x - ow / 2 &&
      py - ph < o.y + oh / 2 && py + ph > o.y - oh / 2
    ) {
      emitParticles(px, py, '#f97316', 20)
      emitParticles(px, py, '#fbbf24', 10)
      endGame()
      return
    }
  }

  // Update particles
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]
    p.x += p.vx * dt; p.y += p.vy * dt
    p.vy += 40 * dt // slight gravity
    p.life -= dt * 1.6
    if (p.pts) p.rot += dt * 3
    if (p.life <= 0) particles.splice(i, 1)
  }

  for (let i = floatTexts.length - 1; i >= 0; i--) {
    const f = floatTexts[i]
    f.y += f.vy * dt; f.life -= dt * 1.4
    if (f.life <= 0) floatTexts.splice(i, 1)
  }
}

function drawAsteroid(o) {
  ctx.save()
  ctx.translate(o.x, o.y)
  ctx.rotate(o.rot)

  // Hit flash
  if (o.hitFlash > 0) {
    ctx.shadowColor = '#ffffff'
    ctx.shadowBlur = 12
  }

  // Main rock shape
  ctx.beginPath()
  const pts = o.pts
  ctx.moveTo(pts[0][0], pts[0][1])
  for (let i = 1; i < pts.length; i++) ctx.lineTo(pts[i][0], pts[i][1])
  ctx.closePath()

  // Fill with gradient-like coloring
  ctx.fillStyle = o.hitFlash > 0 ? '#ffffff' : o.col
  ctx.fill()

  // Rock texture lines
  ctx.strokeStyle = o.col2
  ctx.lineWidth = 0.8
  ctx.globalAlpha = 0.5
  ctx.stroke()

  // Crater detail
  ctx.globalAlpha = 0.3
  ctx.fillStyle = o.col2
  const r = Math.max(o.w, o.h) * 0.08
  ctx.beginPath()
  ctx.arc(-r * 2, -r, r, 0, Math.PI * 2)
  ctx.fill()
  ctx.beginPath()
  ctx.arc(r * 1.5, r * 1.5, r * 0.7, 0, Math.PI * 2)
  ctx.fill()

  // Special type glow
  if (o.type === 'iron') {
    ctx.globalAlpha = 0.25
    ctx.fillStyle = '#a78bfa'
    ctx.beginPath()
    ctx.arc(0, 0, Math.max(o.w, o.h) * 0.4, 0, Math.PI * 2)
    ctx.fill()
  } else if (o.type === 'ice') {
    ctx.globalAlpha = 0.2
    ctx.fillStyle = '#67e8f9'
    ctx.beginPath()
    ctx.arc(0, 0, Math.max(o.w, o.h) * 0.45, 0, Math.PI * 2)
    ctx.fill()
  }

  ctx.restore()
}

function drawSpaceship(x, y) {
  ctx.save()

  // Engine glow / thrust flame
  const thrustLen = 14 + Math.random() * 8
  const grad = ctx.createLinearGradient(x, y + 18, x, y + 18 + thrustLen)
  grad.addColorStop(0, 'rgba(99,179,237,0.9)')
  grad.addColorStop(0.4, 'rgba(59,130,246,0.6)')
  grad.addColorStop(1, 'rgba(59,130,246,0)')
  ctx.fillStyle = grad
  // Left thruster
  ctx.beginPath()
  ctx.moveTo(x - 7, y + 14)
  ctx.lineTo(x - 10, y + 14 + thrustLen)
  ctx.lineTo(x - 4, y + 14 + thrustLen * 0.6)
  ctx.closePath()
  ctx.fill()
  // Right thruster
  ctx.beginPath()
  ctx.moveTo(x + 7, y + 14)
  ctx.lineTo(x + 10, y + 14 + thrustLen)
  ctx.lineTo(x + 4, y + 14 + thrustLen * 0.6)
  ctx.closePath()
  ctx.fill()

  // Main hull — futuristic triangular fighter
  // Fuselage
  ctx.fillStyle = '#1e3a5f'
  ctx.beginPath()
  ctx.moveTo(x, y - 22)          // nose tip
  ctx.lineTo(x - 10, y + 10)     // left body
  ctx.lineTo(x - 6, y + 16)      // left engine mount
  ctx.lineTo(x + 6, y + 16)      // right engine mount
  ctx.lineTo(x + 10, y + 10)     // right body
  ctx.closePath()
  ctx.fill()

  // Cockpit glass
  ctx.fillStyle = '#7dd3fc'
  ctx.globalAlpha = 0.8
  ctx.beginPath()
  ctx.moveTo(x, y - 18)
  ctx.lineTo(x - 5, y - 6)
  ctx.lineTo(x + 5, y - 6)
  ctx.closePath()
  ctx.fill()
  ctx.globalAlpha = 1

  // Wing left
  ctx.fillStyle = '#1e40af'
  ctx.beginPath()
  ctx.moveTo(x - 8, y)
  ctx.lineTo(x - 20, y + 12)
  ctx.lineTo(x - 20, y + 16)
  ctx.lineTo(x - 6, y + 12)
  ctx.closePath()
  ctx.fill()

  // Wing right
  ctx.beginPath()
  ctx.moveTo(x + 8, y)
  ctx.lineTo(x + 20, y + 12)
  ctx.lineTo(x + 20, y + 16)
  ctx.lineTo(x + 6, y + 12)
  ctx.closePath()
  ctx.fill()

  // Wing accent stripe left
  ctx.fillStyle = '#3b82f6'
  ctx.globalAlpha = 0.7
  ctx.fillRect(x - 17, y + 10, 10, 2)
  // Wing accent stripe right
  ctx.fillRect(x + 7, y + 10, 10, 2)
  ctx.globalAlpha = 1

  // Hull center stripe
  ctx.fillStyle = '#3b82f6'
  ctx.fillRect(x - 1.5, y - 16, 3, 28)

  // Engine mounts
  ctx.fillStyle = '#374151'
  ctx.fillRect(x - 10, y + 12, 6, 6)
  ctx.fillRect(x + 4, y + 12, 6, 6)

  // Engine glow rings
  ctx.strokeStyle = '#60a5fa'
  ctx.lineWidth = 1
  ctx.globalAlpha = 0.7
  ctx.beginPath(); ctx.arc(x - 7, y + 15, 3, 0, Math.PI * 2); ctx.stroke()
  ctx.beginPath(); ctx.arc(x + 7, y + 15, 3, 0, Math.PI * 2); ctx.stroke()
  ctx.globalAlpha = 1

  // Cannon tips
  ctx.fillStyle = '#9ca3af'
  ctx.fillRect(x - 2, y - 24, 4, 4)

  ctx.restore()
}

function draw() {
  ctx.save()
  if (shakeAmt > 0) {
    ctx.translate((Math.random() - 0.5) * shakeAmt * 8, (Math.random() - 0.5) * shakeAmt * 8)
  }

  // Deep space background
  ctx.fillStyle = '#020408'
  ctx.fillRect(0, 0, W, H)

  // Nebula blobs (far parallax)
  for (const n of nebulae) {
    ctx.save()
    ctx.globalAlpha = n.alpha
    const ng = ctx.createRadialGradient(n.x, n.y, 0, n.x, n.y, n.rx)
    ng.addColorStop(0, `hsl(${n.hue},80%,40%)`)
    ng.addColorStop(1, 'transparent')
    ctx.fillStyle = ng
    ctx.beginPath()
    ctx.ellipse(n.x, n.y, n.rx, n.ry, 0, 0, Math.PI * 2)
    ctx.fill()
    ctx.restore()
  }

  // Stars (mid parallax)
  for (const s of stars) {
    const twinkle = 0.5 + 0.5 * Math.sin(s.twinkle)
    ctx.globalAlpha = 0.2 + twinkle * 0.6
    ctx.fillStyle = s.r > 1.4 ? '#e0f2fe' : '#ffffff'
    ctx.beginPath()
    ctx.arc(s.x, s.y, s.r, 0, Math.PI * 2)
    ctx.fill()
  }
  ctx.globalAlpha = 1

  // Speed dust streaks (near parallax)
  ctx.strokeStyle = 'rgba(148,163,184,0.12)'
  ctx.lineWidth = 0.8
  for (const d of dustLines) {
    ctx.beginPath()
    ctx.moveTo(d.x, d.y - d.len)
    ctx.lineTo(d.x, d.y)
    ctx.stroke()
  }

  // Lane edge subtle glow
  ctx.strokeStyle = 'rgba(59,130,246,0.08)'
  ctx.lineWidth = 1
  ctx.setLineDash([12, 24])
  for (let li = 1; li < LANES; li++) {
    const lx = LANE_W * li
    ctx.beginPath(); ctx.moveTo(lx, 0); ctx.lineTo(lx, H); ctx.stroke()
  }
  ctx.setLineDash([])

  // Level progress bar
  const pct = getLevelProgress(elapsedRaw)
  ctx.fillStyle = 'rgba(255,255,255,0.04)'
  ctx.fillRect(0, H - 3, W, 3)
  ctx.fillStyle = 'rgba(96,165,250,0.6)'
  ctx.fillRect(0, H - 3, W * pct, 3)

  // Stage flash
  if (warningFlash > 0) {
    const pulse = Math.abs(Math.sin(warningFlash * 9))
    ctx.fillStyle = `rgba(96,165,250,${pulse * (warningFlash / 1.5) * 0.08})`
    ctx.fillRect(0, 0, W, H)
    ctx.fillStyle = `rgba(255,255,255,${Math.min(pulse * (warningFlash / 1.5), 0.9)})`
    ctx.font = 'bold 13px monospace'
    ctx.textAlign = 'center'
    ctx.fillText('STAGE ' + level.value, W / 2, 24)
    ctx.textAlign = 'left'
  }

  // Bullets
  for (const b of bullets) {
    ctx.save()
    ctx.shadowColor = '#7dd3fc'
    ctx.shadowBlur  = 8
    const bg = ctx.createLinearGradient(b.x, b.y, b.x, b.y - b.h)
    bg.addColorStop(0, '#7dd3fc')
    bg.addColorStop(1, 'rgba(125,211,252,0)')
    ctx.fillStyle = bg
    ctx.fillRect(b.x - b.w / 2, b.y - b.h, b.w, b.h)
    ctx.restore()
  }

  // Asteroids
  for (const o of asteroids) drawAsteroid(o)

  // Particles (rock debris + explosion)
  for (const p of particles) {
    ctx.save()
    ctx.globalAlpha = Math.max(0, p.life)
    if (p.pts) {
      // Debris chunk
      ctx.translate(p.x, p.y)
      ctx.rotate(p.rot)
      ctx.fillStyle = p.col
      ctx.beginPath()
      ctx.moveTo(p.pts[0][0], p.pts[0][1])
      for (let i = 1; i < p.pts.length; i++) ctx.lineTo(p.pts[i][0], p.pts[i][1])
      ctx.closePath()
      ctx.fill()
    } else {
      ctx.fillStyle = p.col
      ctx.beginPath()
      ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2)
      ctx.fill()
    }
    ctx.restore()
  }

  // Float texts
  for (const f of floatTexts) {
    ctx.save()
    ctx.globalAlpha = Math.max(0, f.life)
    ctx.fillStyle   = f.col
    ctx.font        = 'bold 14px monospace'
    ctx.textAlign   = 'center'
    ctx.shadowColor = f.col
    ctx.shadowBlur  = 4
    ctx.fillText(f.text, f.x, f.y)
    ctx.restore()
  }

  // Ship
  if (state.value === 'play' || state.value === 'over') {
    drawSpaceship(currentX, SHIP_Y)
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
  background: #020408;
  font-family: 'Share Tech Mono', 'Courier New', monospace;
  padding-bottom: 12px;
}

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
  border-left: 1px solid rgba(59,130,246,0.12);
  border-right: 1px solid rgba(59,130,246,0.12);
}

/* SCREENS */
.screen {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  padding-bottom: 28px;
  background: linear-gradient(to bottom, rgba(2,4,8,0.05) 0%, rgba(2,4,8,0.96) 52%);
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
  color: rgba(96,165,250,0.75);
  padding: 3px 10px;
  border: 0.5px solid rgba(96,165,250,0.25);
  margin-bottom: 14px;
}

.badge.danger { color: rgba(251,113,133,0.85); border-color: rgba(251,113,133,0.3); }

.title-main {
  font-size: 62px;
  font-weight: 700;
  color: #fff;
  text-align: center;
  line-height: 0.88;
  letter-spacing: -0.02em;
  margin: 0 0 18px;
}

.title-accent { color: #60a5fa; }

.subtitle {
  font-size: 10px;
  letter-spacing: 0.1em;
  color: rgba(148,163,184,0.6);
  text-align: center;
  line-height: 2;
  margin: 0 0 26px;
}

.over-score-label { font-size: 9px; letter-spacing: 0.2em; color: rgba(148,163,184,0.4); margin-bottom: 6px; }
.over-score { font-size: 76px; font-weight: 700; color: #fff; line-height: 1; margin-bottom: 8px; }

.hs-line { font-size: 11px; color: rgba(148,163,184,0.35); margin-bottom: 20px; letter-spacing: 0.08em; }
.hs-line.gold { color: #fbbf24; }

.stat-row {
  display: flex;
  border: 0.5px solid rgba(59,130,246,0.15);
  margin-bottom: 24px;
}

.stat { padding: 10px 14px; text-align: center; border-right: 0.5px solid rgba(59,130,246,0.15); }
.stat:last-child { border-right: none; }
.sv { font-size: 22px; font-weight: 600; color: #fff; }
.sk { font-size: 8px; color: rgba(148,163,184,0.4); letter-spacing: 0.12em; margin-top: 2px; }

.btn-start {
  background: transparent;
  border: 0.5px solid rgba(96,165,250,0.4);
  padding: 13px 44px;
  font-family: inherit;
  font-size: 13px;
  letter-spacing: 0.18em;
  color: #60a5fa;
  cursor: pointer;
  transition: background 0.15s, border-color 0.15s;
  animation: btn-pulse 1.8s ease-in-out infinite;
}

@keyframes btn-pulse {
  0%, 100% { border-color: rgba(96,165,250,0.4); }
  50%       { border-color: rgba(96,165,250,0.9); box-shadow: 0 0 14px rgba(96,165,250,0.18); }
}

.btn-start:hover  { background: rgba(96,165,250,0.07); }
.btn-start:active { opacity: 0.7; }

.hint-row {
  display: flex;
  justify-content: space-between;
  width: 290px;
  margin-top: 16px;
  font-size: 9px;
  letter-spacing: 0.08em;
  color: rgba(148,163,184,0.2);
}

/* HUD */
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
  background: linear-gradient(to bottom, rgba(2,4,8,0.9) 0%, rgba(2,4,8,0) 100%);
}

.hud-cell { min-width: 60px; }
.hud-cell.center { display: flex; gap: 4px; align-items: center; justify-content: center; }
.hud-cell.right { text-align: right; }
.hk { font-size: 8px; color: rgba(148,163,184,0.35); letter-spacing: 0.2em; }
.hv { font-size: 20px; font-weight: 600; color: #fff; line-height: 1; }

.stage-pip {
  width: 8px; height: 8px;
  border: 0.5px solid rgba(59,130,246,0.18);
  background: rgba(59,130,246,0.05);
  transition: background 0.2s;
}
.stage-pip.lit { background: #3b82f6; border-color: rgba(59,130,246,0.7); }

/* AMMO PANEL */
.ammo-panel {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 320px;
  padding: 7px 12px;
  background: #070b11;
  border: 1px solid rgba(59,130,246,0.1);
  border-top: none;
  transition: border-color 0.2s;
}

.ammo-panel.ready { border-color: rgba(96,165,250,0.4); }

.ammo-pips { display: flex; gap: 4px; }

.ap {
  width: 16px; height: 8px;
  background: rgba(59,130,246,0.06);
  border: 0.5px solid rgba(59,130,246,0.12);
  transition: background 0.12s;
}

.ap.f { background: rgba(96,165,250,0.45); border-color: rgba(96,165,250,0.6); }

@keyframes ap-r { 0%,100% { background: rgba(96,165,250,0.45); } 50% { background: rgba(96,165,250,0.95); } }
.ap.r { animation: ap-r 1s infinite; }

.ammo-info { display: flex; align-items: center; gap: 8px; flex: 1; }
.ammo-tag  { font-size: 9px; letter-spacing: 0.2em; color: rgba(96,165,250,0.55); }
.ammo-status { font-size: 10px; color: rgba(148,163,184,0.4); letter-spacing: 0.06em; }
.ammo-panel.ready .ammo-status { color: #7dd3fc; }
.ammo-bonus {
  font-size: 11px; color: #fde68a; margin-left: auto;
  animation: bonus-pop 0.3s ease-out;
}

@keyframes bonus-pop {
  0% { transform: scale(1.4); opacity: 0; }
  100% { transform: scale(1); opacity: 1; }
}

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
  background: rgba(59,130,246,0.03);
  border: 0.5px solid rgba(59,130,246,0.12);
  color: rgba(148,163,184,0.7);
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

.cb:active { background: rgba(59,130,246,0.1); }

.shoot {
  flex: 1.4;
  gap: 3px;
  background: rgba(96,165,250,0.04);
  border-color: rgba(96,165,250,0.15);
  cursor: not-allowed;
  opacity: 0.35;
}
.shoot:disabled { cursor: not-allowed; }

.shoot-icon { font-size: 18px; }
.shoot-label { font-size: 9px; letter-spacing: 0.12em; color: rgba(96,165,250,0.7); }

.shoot.shoot-r {
  cursor: pointer;
  opacity: 1;
  background: rgba(96,165,250,0.08);
  border-color: rgba(96,165,250,0.55);
  animation: shoot-pulse 0.85s ease-in-out infinite;
}

@keyframes shoot-pulse {
  0%,100% { box-shadow: 0 0 0 0 rgba(96,165,250,0); }
  50%      { box-shadow: 0 0 0 3px rgba(96,165,250,0.22); }
}

.shoot.shoot-r .shoot-label { color: #7dd3fc; }
.shoot.shoot-empty { opacity: 0.25; cursor: not-allowed; }
</style>