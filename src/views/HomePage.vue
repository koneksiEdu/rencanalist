<template>
  <div class="root">
    <div class="scanlines" aria-hidden="true" />

    <div class="game-wrapper">
      <div class="title-bar">
        <div class="title-logo">
          <span class="logo-n">N</span><span class="logo-e">E</span><span class="logo-o">O</span><span class="logo-n2">N</span>
          <span class="logo-sep">·</span>
          <span class="logo-w">W</span><span class="logo-o2">O</span><span class="logo-r">R</span><span class="logo-m">M</span>
        </div>
      </div>

      <div class="hud" v-if="state === 'play'">
        <div class="hud-cell">
          <span class="hud-label">SKOR</span>
          <span class="hud-val" :class="{ 'nos-active-text': nosActive }">{{ score }}</span>
        </div>
        <div class="hud-cell center">
          <span class="hud-label">NOS</span>
          <div class="nos-bar-wrap">
            <div class="nos-bar-fill" :style="{ width: nosBar + '%' }" :class="{ full: nosBar >= 100, active: nosActive }" />
            <div class="nos-bar-segments">
              <div v-for="i in 4" :key="i" class="nos-seg" />
            </div>
          </div>
          <span class="nos-hint" v-if="nosBar >= 100 && !nosActive">SIAP!</span>
          <span class="nos-hint active-hint" v-if="nosActive">2×</span>
        </div>
        <div class="hud-cell right">
          <span class="hud-label">REKOR</span>
          <span class="hud-val">{{ best }}</span>
        </div>
      </div>

      <div class="canvas-wrap">
        <canvas ref="cvs" class="game-canvas" />

        <Transition name="fade">
          <div v-if="state === 'start'" class="overlay">
            <div class="overlay-card">
              <div class="worm-preview">
                <div v-for="i in 6" :key="i" class="prev-seg" :style="prevSegStyle(i)" />
              </div>
              <div class="badge">ARCADE</div>
              <h1 class="big-title">SNAKE<br><em>THE Rusher</em></h1>
              <p class="desc">
                Makan piksel untuk tumbuh<br>
                Kumpulkan NOS untuk <strong>turbo</strong>!
              </p>
              <button class="btn-start" @touchstart.prevent="beginGame" @click="beginGame">
                MULAI
              </button>
            </div>
          </div>
        </Transition>

        <Transition name="fade">
          <div v-if="state === 'over'" class="overlay">
            <div class="overlay-card">
              <div class="badge danger-badge">GAME OVER</div>
              <div class="final-score" :class="{ 'record-glow': isNewRecord }">{{ score }}</div>
              <p class="new-record" v-if="isNewRecord">✦ REKOR BARU ✦</p>
              <p class="over-sub" v-else>Rekor: {{ best }}</p>
              <div class="stats-row">
                <div class="stat">
                  <span class="stat-val">{{ snacksEaten }}</span>
                  <span class="stat-key">PIKSEL</span>
                </div>
                <div class="stat">
                  <span class="stat-val">{{ nosUsed }}</span>
                  <span class="stat-key">NOS</span>
                </div>
              </div>
              <button class="btn-start" @touchstart.prevent="beginGame" @click="beginGame">
                LAGI
              </button>
            </div>
          </div>
        </Transition>
      </div>

      <!-- Simplified D-pad: compass ring layout, no PC hints -->
      <div class="dpad" v-if="state === 'play'">
        <div class="dpad-ring">
          <button class="dpad-btn up" @touchstart.prevent="setDir(0,-1)" @mousedown.prevent="setDir(0,-1)">
            <svg width="18" height="18" viewBox="0 0 18 18"><path d="M9 14V4M3 9l6-6 6 6" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" fill="none"/></svg>
          </button>
          <button class="dpad-btn left" @touchstart.prevent="setDir(-1,0)" @mousedown.prevent="setDir(-1,0)">
            <svg width="18" height="18" viewBox="0 0 18 18"><path d="M14 9H4M9 3l-6 6 6 6" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" fill="none"/></svg>
          </button>
          <button class="dpad-btn nos-btn" :class="{ 'nos-ready': nosBar >= 100, 'nos-on': nosActive }" @touchstart.prevent="activateNos" @mousedown.prevent="activateNos">
            <svg width="22" height="22" viewBox="0 0 22 22">
              <path d="M13 2L4 13h7l-2 7 9-11h-7l2-7z" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" fill="none"/>
            </svg>
          </button>
          <button class="dpad-btn right" @touchstart.prevent="setDir(1,0)" @mousedown.prevent="setDir(1,0)">
            <svg width="18" height="18" viewBox="0 0 18 18"><path d="M4 9h10M9 3l6 6-6 6" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" fill="none"/></svg>
          </button>
          <button class="dpad-btn down" @touchstart.prevent="setDir(0,1)" @mousedown.prevent="setDir(0,1)">
            <svg width="18" height="18" viewBox="0 0 18 18"><path d="M9 4v10M3 9l6 6 6-6" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" fill="none"/></svg>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue'

const COLS = 18, ROWS = 22
let CELL = 18

const cvs = ref(null)
const state = ref('start')
const score = ref(0)
const best = ref(0)
const isNewRecord = ref(false)
const snacksEaten = ref(0)
const nosUsed = ref(0)
const nosBar = ref(0)
const nosActive = ref(false)
const snake = ref([])

let ctx = null
let dir = { x: 1, y: 0 }
let nextDir = { x: 1, y: 0 }
let foods = []
let particles = []
let loopId = null
let nosStartTime = 0
let nosDuration = 0 // Duration in ms that NOS should last
const BASE_SPEED = 175
const NOS_DURATION_MS = 3000 // NOS lasts 3 seconds
let speed = BASE_SPEED
let resizeObserver = null

const LS_KEY = 'neonworm_best'

const FOOD_TYPES = [
  { color: '#00E5FF', pts: 10, prob: 0.50, size: 0.38 },
  { color: '#E040FB', pts: 25, prob: 0.30, size: 0.42 },
  { color: '#FFEA00', pts: 50, prob: 0.15, size: 0.46 },
  { color: '#FF4081', pts: 5,  prob: 0.05, size: 0.34, isNos: true },
]

function prevSegStyle(i) {
  const hues = [180, 175, 170, 165, 160, 155]
  return {
    background: `hsl(${hues[i-1]}, 100%, 55%)`,
    width: '0',
    height: '0',
    marginLeft: i === 1 ? '0' : '-4px',
    borderLeft: `${10 - i}px solid transparent`,
    borderRight: `${10 - i}px solid transparent`,
    borderBottom: `${(22 - i * 2)}px solid hsl(${hues[i-1]}, 100%, 55%)`,
    transform: `rotate(${(i-1)*6}deg)`,
    display: 'inline-block',
  }
}

function pickFood() {
  const r = Math.random()
  let acc = 0
  for (const t of FOOD_TYPES) { acc += t.prob; if (r < acc) return t }
  return FOOD_TYPES[0]
}

function placeFood() {
  const body = new Set(snake.value.map(s => s.x + ',' + s.y))
  const existing = new Set(foods.map(f => f.x + ',' + f.y))
  const free = []
  for (let x = 0; x < COLS; x++)
    for (let y = 0; y < ROWS; y++)
      if (!body.has(x + ',' + y) && !existing.has(x + ',' + y)) free.push({ x, y })
  if (!free.length) return null
  const pos = free[Math.floor(Math.random() * free.length)]
  const t = pickFood()
  return { ...pos, ...t, pulse: Math.random() * Math.PI * 2 }
}

function spawnParticle(x, y, color) {
  for (let i = 0; i < 6; i++) {
    const angle = (Math.PI * 2 / 6) * i + Math.random() * 0.5
    particles.push({
      x: x * CELL + CELL / 2, y: y * CELL + CELL / 2,
      vx: Math.cos(angle) * (1 + Math.random() * 2),
      vy: Math.sin(angle) * (1 + Math.random() * 2),
      life: 1, color, size: 2 + Math.random() * 2,
    })
  }
}

function initGame() {
  const cx = Math.floor(COLS / 2), cy = Math.floor(ROWS / 2)
  snake.value = [{ x: cx, y: cy }, { x: cx - 1, y: cy }, { x: cx - 2, y: cy }]
  dir = { x: 1, y: 0 }; nextDir = { x: 1, y: 0 }
  foods = []; particles = []
  for (let i = 0; i < 3; i++) { const f = placeFood(); if (f) foods.push(f) }
  score.value = 0; snacksEaten.value = 0; nosUsed.value = 0
  nosBar.value = 0; nosActive.value = false
  speed = BASE_SPEED; isNewRecord.value = false

  const saved = localStorage.getItem(LS_KEY)
  if (saved) best.value = parseInt(saved) || 0
}

function activateNos() {
  if (nosBar.value < 100 || nosActive.value) return
  nosActive.value = true
  nosUsed.value++
  nosBar.value =0
  speed = Math.floor(BASE_SPEED * 0.45)
  nosStartTime = performance.now()
  nosDuration = NOS_DURATION_MS
}

let tickAnim = 0

// Draw a single snake segment as a blade/arrow shape
function drawBlade(cx, cy, segDir, i, len, isNos) {
  const t = i / len
  const half = CELL / 2
  const tip = 0.72  // how sharp the tip is (0=flat, 1=full point)
  const notch = 0.28 // tail notch depth

  // Determine which direction the blade faces
  // For head: faces movement dir. For body: faces previous segment direction
  const dx = segDir.x, dy = segDir.y

  // Build arrow polygon centered at cx,cy
  // Arrow points in +dx/+dy direction
  const pts = []

  if (dx === 1 || dx === -1) {
    // Horizontal blade
    const sx = dx
    pts.push(
      [cx + sx * half * tip,        cy],                         // tip
      [cx - sx * half * notch,      cy - half + 2],             // top-back
      [cx - sx * half * (1 - 0.1), cy - half * 0.45],          // top-notch
      [cx - sx * half * (1 - 0.1), cy + half * 0.45],          // bottom-notch  
      [cx - sx * half * notch,      cy + half - 2],             // bottom-back
    )
  } else {
    // Vertical blade
    const sy = dy
    pts.push(
      [cx,                          cy + sy * half * tip],       // tip
      [cx - half + 2,               cy - sy * half * notch],    // left-back
      [cx - half * 0.45,            cy - sy * half * (1-0.1)],  // left-notch
      [cx + half * 0.45,            cy - sy * half * (1-0.1)],  // right-notch
      [cx + half - 2,               cy - sy * half * notch],    // right-back
    )
  }

  ctx.beginPath()
  ctx.moveTo(pts[0][0], pts[0][1])
  for (let k = 1; k < pts.length; k++) ctx.lineTo(pts[k][0], pts[k][1])
  ctx.closePath()

  // Color
  if (isNos) {
    const brightness = i === 0 ? 1 : Math.max(0.28, 1 - t * 0.65)
    if (i === 0) {
      ctx.fillStyle = `rgba(180,255,255,${brightness})`
    } else {
      ctx.fillStyle = `rgba(0,${Math.round(220 - t * 80)},${Math.round(255 - t * 60)},${brightness})`
    }
    if (i < 3) { ctx.shadowColor = '#00E5FF'; ctx.shadowBlur = 10 }
  } else {
    if (i === 0) {
      ctx.fillStyle = '#39FF14'
    } else {
      const g = Math.round(225 - t * 90)
      const b = Math.round(140 - t * 60)
      ctx.fillStyle = `rgb(0,${g},${b < 50 ? 50 : b})`
    }
    ctx.shadowColor = 'transparent'
    ctx.shadowBlur = 0
  }

  ctx.fill()

  // Stroke outline for definition
  ctx.strokeStyle = isNos ? 'rgba(0,229,255,0.25)' : 'rgba(57,255,20,0.18)'
  ctx.lineWidth = 0.7
  ctx.stroke()
  ctx.shadowBlur = 0

  // Head details
  if (i === 0) {
    // Eye dots — positioned at the tip side
    const eyeOff = 0.22
    let ex1, ey1, ex2, ey2
    if (dx === 1 || dx === -1) {
      ex1 = cx + dx * half * 0.35; ey1 = cy - CELL * eyeOff
      ex2 = cx + dx * half * 0.35; ey2 = cy + CELL * eyeOff
    } else {
      ex1 = cx - CELL * eyeOff; ey1 = cy + dy * half * 0.35
      ex2 = cx + CELL * eyeOff; ey2 = cy + dy * half * 0.35
    }
    ctx.fillStyle = '#0a0a14'
    ctx.beginPath(); ctx.arc(ex1, ey1, 1.7, 0, Math.PI * 2); ctx.fill()
    ctx.beginPath(); ctx.arc(ex2, ey2, 1.7, 0, Math.PI * 2); ctx.fill()

    // NOS flash ring
    if (isNos && tickAnim % 3 === 0) {
      ctx.strokeStyle = 'rgba(255,255,255,0.55)'
      ctx.lineWidth = 1
      ctx.beginPath()
      ctx.moveTo(pts[0][0], pts[0][1])
      for (let k = 1; k < pts.length; k++) ctx.lineTo(pts[k][0], pts[k][1])
      ctx.closePath()
      ctx.stroke()
    }
  }
}

function draw() {
  if (!ctx) return
  const W = COLS * CELL, H = ROWS * CELL

  ctx.fillStyle = '#0a0a14'
  ctx.fillRect(0, 0, W, H)

  // Subtle grid
  ctx.strokeStyle = 'rgba(100,100,200,0.05)'
  ctx.lineWidth = 0.5
  for (let x = 0; x <= COLS; x++) {
    ctx.beginPath(); ctx.moveTo(x * CELL, 0); ctx.lineTo(x * CELL, H); ctx.stroke()
  }
  for (let y = 0; y <= ROWS; y++) {
    ctx.beginPath(); ctx.moveTo(0, y * CELL); ctx.lineTo(W, y * CELL); ctx.stroke()
  }

  // Food
  foods.forEach(f => {
    const px = f.x * CELL + CELL / 2, py = f.y * CELL + CELL / 2
    const r = CELL * f.size
    const pulse = 0.85 + Math.sin(f.pulse) * 0.15

    if (f.isNos) {
      ctx.save()
      ctx.translate(px, py)
      ctx.scale(pulse, pulse)
      ctx.strokeStyle = f.color
      ctx.lineWidth = 1.5
      ctx.lineJoin = 'round'
      ctx.lineCap = 'round'
      const s = r * 0.8
      ctx.beginPath()
      ctx.moveTo(s * 0.3, -s)
      ctx.lineTo(-s * 0.1, -s * 0.1)
      ctx.lineTo(s * 0.3, -s * 0.1)
      ctx.lineTo(-s * 0.3, s)
      ctx.stroke()
      ctx.restore()
    } else {
      ctx.save()
      ctx.translate(px, py)
      ctx.scale(pulse, pulse)
      ctx.beginPath()
      ctx.moveTo(0, -r)
      ctx.lineTo(r * 0.7, 0)
      ctx.lineTo(0, r)
      ctx.lineTo(-r * 0.7, 0)
      ctx.closePath()
      ctx.fillStyle = f.color + '22'
      ctx.fill()
      ctx.strokeStyle = f.color
      ctx.lineWidth = 1.5
      ctx.stroke()
      ctx.beginPath()
      ctx.moveTo(0, -r * 0.55)
      ctx.lineTo(r * 0.35, 0)
      ctx.lineTo(0, r * 0.55)
      ctx.lineTo(-r * 0.35, 0)
      ctx.closePath()
      ctx.fillStyle = f.color + '55'
      ctx.fill()
      ctx.restore()
    }
  })

  // Particles
  particles.forEach(p => {
    ctx.globalAlpha = p.life
    ctx.fillStyle = p.color
    ctx.beginPath()
    ctx.arc(p.x, p.y, p.size * p.life, 0, Math.PI * 2)
    ctx.fill()
  })
  ctx.globalAlpha = 1

  // Snake — blade/arrow segments
  const len = snake.value.length
  snake.value.forEach((seg, i) => {
    const cx = seg.x * CELL + CELL / 2
    const cy = seg.y * CELL + CELL / 2

    // Each segment faces the direction toward the previous segment (i-1)
    let segDir
    if (i === 0) {
      segDir = dir
    } else {
      const prev = snake.value[i - 1]
      const ddx = prev.x - seg.x
      const ddy = prev.y - seg.y
      // Clamp to unit direction
      segDir = {
        x: ddx === 0 ? 0 : ddx / Math.abs(ddx),
        y: ddy === 0 ? 0 : ddy / Math.abs(ddy),
      }
    }

    drawBlade(cx, cy, segDir, i, len, nosActive.value)
  })
}

async function beginGame() {
  state.value = 'play'
  initGame()
  await nextTick()
  setupCanvas()
  if (!ctx) return
  draw()
  if (loopId) clearTimeout(loopId)
  loopId = setTimeout(tick, speed)
}

function tick() {
  dir = { ...nextDir }
  const head = { x: snake.value[0].x + dir.x, y: snake.value[0].y + dir.y }

  if (head.x < 0 || head.x >= COLS || head.y < 0 || head.y >= ROWS) { endGame(); return }
  if (snake.value.some(s => s.x === head.x && s.y === head.y)) { endGame(); return }

  snake.value = [head, ...snake.value]

  let ate = false
  for (let i = foods.length - 1; i >= 0; i--) {
    const f = foods[i]
    if (f.x === head.x && f.y === head.y) {
      const mult = nosActive.value ? 2 : 1
      if (f.isNos) {
        nosBar.value = Math.min(100, nosBar.value + 35)
      } else {
        score.value += f.pts * mult
        nosBar.value = Math.min(100, nosBar.value + 8)
      }
      snacksEaten.value++
      spawnParticle(f.x, f.y, f.color)
      foods.splice(i, 1)
      ate = true; break
    }
  }
  if (!ate) snake.value = snake.value.slice(0, -1)
  while (foods.length < 3) { const f = placeFood(); if (!f) break; foods.push(f) }

  // NOS: active until bar depletes gradually
  if (nosActive.value) {
    const now = performance.now()
    const elapsed = now - nosStartTime
    const remainingPercent = Math.max(0, 1 - (elapsed / nosDuration))
    nosBar.value = remainingPercent * 100
    
    if (elapsed >= nosDuration) {
      nosActive.value = false
      speed = BASE_SPEED
      nosBar.value = 0
    }
  }

  tickAnim++
  foods.forEach(f => { f.pulse = (f.pulse || 0) + 0.12 })

  particles = particles
    .map(p => ({ ...p, x: p.x + p.vx, y: p.y + p.vy, life: p.life - 0.12, vx: p.vx * 0.85, vy: p.vy * 0.85 }))
    .filter(p => p.life > 0)

  draw()
  loopId = setTimeout(tick, speed)
}

function endGame() {
  clearTimeout(loopId)
  if (nosActive.value) { nosActive.value = false }
  if (score.value > best.value && score.value > 0) {
    best.value = score.value
    isNewRecord.value = true
    localStorage.setItem(LS_KEY, String(best.value))
  }
  state.value = 'over'
  draw()
}

function setDir(dx, dy) {
  if (dx === -dir.x && dy === -dir.y) return
  nextDir = { x: dx, y: dy }
}

function setupCanvas() {
  if (!cvs.value) return false
  const wrap = cvs.value.parentElement
  if (!wrap) return false
  const containerWidth = wrap.clientWidth
  const containerHeight = wrap.clientHeight || 400
  const maxCellByWidth = Math.floor(containerWidth / COLS)
  const maxCellByHeight = Math.floor(containerHeight / ROWS)
  CELL = Math.min(maxCellByWidth, maxCellByHeight, 22)
  if (CELL < 10) CELL = 14
  cvs.value.width = COLS * CELL
  cvs.value.height = ROWS * CELL
  ctx = cvs.value.getContext('2d')
  cvs.value.style.width = `${cvs.value.width}px`
  cvs.value.style.height = `${cvs.value.height}px`
  return true
}

function onKey(e) {
  if (state.value !== 'play') return
  const map = {
    ArrowUp: [0,-1], ArrowDown: [0,1], ArrowLeft: [-1,0], ArrowRight: [1,0],
    w: [0,-1], s: [0,1], a: [-1,0], d: [1,0]
  }
  if (map[e.key]) { e.preventDefault(); setDir(...map[e.key]); return }
  if (e.key === ' ') { e.preventDefault(); activateNos() }
}

let touchStartX = 0, touchStartY = 0
function onTouchStart(e) { touchStartX = e.touches[0].clientX; touchStartY = e.touches[0].clientY }
function onTouchEnd(e) {
  if (state.value !== 'play') return
  const dx = e.changedTouches[0].clientX - touchStartX
  const dy = e.changedTouches[0].clientY - touchStartY
  if (Math.abs(dx) < 10 && Math.abs(dy) < 10) return
  if (Math.abs(dx) > Math.abs(dy)) setDir(dx > 0 ? 1 : -1, 0)
  else setDir(0, dy > 0 ? 1 : -1)
}

onMounted(() => {
  const saved = localStorage.getItem(LS_KEY)
  if (saved) best.value = parseInt(saved) || 0
  setupCanvas()
  if (ctx && cvs.value) {
    ctx.fillStyle = '#0a0a14'
    ctx.fillRect(0, 0, cvs.value.width, cvs.value.height)
  }
  if (cvs.value?.parentElement) {
    resizeObserver = new ResizeObserver(() => {
      if (state.value === 'play') { setupCanvas(); draw() }
    })
    resizeObserver.observe(cvs.value.parentElement)
  }
  document.addEventListener('keydown', onKey)
  document.addEventListener('touchstart', onTouchStart, { passive: true })
  document.addEventListener('touchend', onTouchEnd, { passive: true })
})

onUnmounted(() => {
  clearTimeout(loopId)
  if (resizeObserver) resizeObserver.disconnect()
  document.removeEventListener('keydown', onKey)
  document.removeEventListener('touchstart', onTouchStart)
  document.removeEventListener('touchend', onTouchEnd)
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Share+Tech+Mono&family=Orbitron:wght@700;900&display=swap');

*, *::before, *::after {
  box-sizing: border-box;
  -webkit-tap-highlight-color: transparent;
  user-select: none;
  touch-action: manipulation;
}

.root {
  position: relative;
  min-height: 100svh;
  width: 100%;
  background: #0a0a14;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  font-family: 'Share Tech Mono', monospace;
}

.scanlines {
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: 0;
  background: repeating-linear-gradient(
    to bottom,
    transparent 0px, transparent 3px,
    rgba(0,0,0,0.07) 3px, rgba(0,0,0,0.07) 4px
  );
}

.game-wrapper {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  max-width: 400px;
  padding: 10px 10px 16px;
  gap: 8px;
}

.title-bar {
  display: flex;
  align-items: center;
  justify-content: center;
}
.title-logo {
  font-family: 'Orbitron', sans-serif;
  font-weight: 900;
  font-size: 20px;
  letter-spacing: 0.15em;
  color: #39FF14;
  text-shadow: 0 0 12px rgba(57,255,20,0.4);
}
.logo-sep { color: rgba(57,255,20,0.3); margin: 0 4px; }
.logo-e, .logo-o, .logo-n2 { color: #00E5FF; text-shadow: 0 0 12px rgba(0,229,255,0.4); }
.logo-w, .logo-o2, .logo-r, .logo-m { color: rgba(255,255,255,0.85); text-shadow: none; }

/* HUD */
.hud {
  width: 100%;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 10px;
  background: rgba(255,255,255,0.03);
  border: 0.5px solid rgba(57,255,20,0.15);
  border-radius: 8px;
}
.hud-cell { display: flex; flex-direction: column; align-items: flex-start; min-width: 60px; }
.hud-cell.center { align-items: center; flex: 1; padding: 0 12px; }
.hud-cell.right { align-items: flex-end; }
.hud-label { font-size: 9px; letter-spacing: 0.22em; color: rgba(57,255,20,0.4); margin-bottom: 2px; }
.hud-val {
  font-family: 'Orbitron', sans-serif;
  font-weight: 700; font-size: 20px; color: #fff; line-height: 1; transition: color 0.2s;
}
.nos-active-text { color: #00E5FF; text-shadow: 0 0 8px rgba(0,229,255,0.5); }

/* NOS Bar */
.nos-bar-wrap {
  width: 100%; height: 10px;
  background: rgba(255,255,255,0.06);
  border-radius: 99px; overflow: hidden; position: relative;
  border: 0.5px solid rgba(255,255,255,0.08); margin-bottom: 2px;
}
.nos-bar-fill {
  height: 100%;
  background: linear-gradient(90deg, #E040FB, #FF4081);
  border-radius: 99px;
  transition: width 0.12s ease, background 0.3s;
}
.nos-bar-fill.full {
  background: linear-gradient(90deg, #FF4081, #FFEA00);
  animation: nospulse 0.6s ease-in-out infinite alternate;
}
.nos-bar-fill.active {
  background: linear-gradient(90deg, #00E5FF, #39FF14);
}
@keyframes nospulse { from { opacity: 0.7; } to { opacity: 1; } }
.nos-bar-segments { position: absolute; inset: 0; display: flex; }
.nos-seg { flex: 1; border-right: 1px solid rgba(10,10,20,0.5); }
.nos-seg:last-child { border-right: none; }
.nos-hint {
  font-size: 9px; letter-spacing: 0.15em; color: rgba(255,64,129,0.7);
  height: 10px; line-height: 10px;
  animation: nosblip 0.5s ease-in-out infinite alternate;
}
.active-hint { color: #00E5FF; animation: none; }
@keyframes nosblip { from { opacity: 0.5; } to { opacity: 1; } }

/* Canvas */
.canvas-wrap {
  position: relative; width: 100%; aspect-ratio: 18 / 22;
  border-radius: 10px; overflow: hidden;
  border: 0.5px solid rgba(57,255,20,0.2);
  box-shadow: 0 0 30px rgba(57,255,20,0.05);
}
.game-canvas { display: block; width: 100%; height: 100%; object-fit: contain; }

/* Overlay */
.overlay {
  position: absolute; inset: 0;
  display: flex; align-items: flex-end; justify-content: center; padding-bottom: 24px;
  background: linear-gradient(to bottom, transparent 5%, rgba(10,10,20,0.96) 45%);
  border-radius: 10px;
}
.overlay-card {
  display: flex; flex-direction: column; align-items: center;
  width: 100%; padding: 0 28px; gap: 10px;
}
.worm-preview {
  display: flex; align-items: flex-end; margin-bottom: 6px; height: 28px;
}
.badge {
  font-size: 9px; font-family: 'Orbitron', sans-serif; letter-spacing: 0.3em; color: #39FF14;
  border: 0.5px solid rgba(57,255,20,0.4); border-radius: 99px; padding: 3px 14px;
  background: rgba(57,255,20,0.06);
}
.danger-badge { color: #FF4081; border-color: rgba(255,64,129,0.4); background: rgba(255,64,129,0.07); }
.big-title {
  font-family: 'Orbitron', sans-serif; font-weight: 900; font-size: 44px;
  line-height: 0.9; text-align: center; margin: 0; color: #fff; letter-spacing: 0.05em;
}
.big-title em { font-style: normal; color: #39FF14; text-shadow: 0 0 16px rgba(57,255,20,0.4); }
.desc {
  font-size: 12px; color: rgba(255,255,255,0.3); text-align: center; line-height: 1.8; margin: 2px 0;
}
.desc strong { color: rgba(0,229,255,0.7); font-weight: normal; }
.final-score {
  font-family: 'Orbitron', sans-serif; font-weight: 900; font-size: 64px;
  line-height: 1; color: #fff; margin: 2px 0; letter-spacing: 0.05em;
}
.record-glow { color: #FFEA00; text-shadow: 0 0 20px rgba(255,234,0,0.4); }
.new-record { font-size: 11px; letter-spacing: 0.2em; color: #FFEA00; margin: 0; }
.over-sub { font-size: 11px; color: rgba(255,255,255,0.25); letter-spacing: 0.1em; margin: 0; }
.stats-row {
  display: flex; border: 0.5px solid rgba(255,255,255,0.08);
  border-radius: 10px; overflow: hidden; width: 100%;
}
.stat { display: flex; flex-direction: column; align-items: center; padding: 10px 0; flex: 1; }
.stat + .stat { border-left: 0.5px solid rgba(255,255,255,0.06); }
.stat-val { font-family: 'Orbitron', sans-serif; font-weight: 700; font-size: 22px; color: #fff; }
.stat-key { font-size: 8px; letter-spacing: 0.18em; color: rgba(57,255,20,0.4); margin-top: 2px; }
.btn-start {
  border: 0.5px solid rgba(57,255,20,0.5); border-radius: 99px; padding: 13px 48px;
  background: rgba(57,255,20,0.07); color: #39FF14;
  font-family: 'Orbitron', sans-serif; font-weight: 700; font-size: 14px;
  letter-spacing: 0.15em; cursor: pointer; transition: background 0.15s, transform 0.1s;
}
.btn-start:active { background: rgba(57,255,20,0.18); transform: scale(0.96); }

/* D-pad ring layout */
.dpad {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 4px 0;
}
.dpad-ring {
  display: grid;
  grid-template-areas:
    ". up ."
    "left nos right"
    ". down .";
  grid-template-columns: 64px 64px 64px;
  grid-template-rows: 64px 64px 64px;
  gap: 5px;
}
.dpad-btn {
  background: rgba(255,255,255,0.03);
  border: 0.5px solid rgba(255,255,255,0.1);
  border-radius: 14px;
  color: rgba(255,255,255,0.5);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.1s, transform 0.1s;
  width: 100%; height: 100%;
}
.dpad-btn:active { background: rgba(255,255,255,0.1); transform: scale(0.9); }
.up    { grid-area: up; }
.down  { grid-area: down; }
.left  { grid-area: left; }
.right { grid-area: right; }
.nos-btn {
  grid-area: nos;
  color: rgba(224,64,251,0.5);
  border-color: rgba(224,64,251,0.2);
  border-radius: 50%;
}
.nos-btn.nos-ready {
  color: #FF4081; border-color: rgba(255,64,129,0.5);
  background: rgba(255,64,129,0.08);
  animation: nosready 0.7s ease-in-out infinite alternate;
}
.nos-btn.nos-on {
  color: #00E5FF; border-color: rgba(0,229,255,0.5);
  background: rgba(0,229,255,0.12);
}
@keyframes nosready {
  from { box-shadow: none; }
  to { box-shadow: 0 0 14px rgba(255,64,129,0.35); }
}

.fade-enter-active, .fade-leave-active { transition: opacity 0.2s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>