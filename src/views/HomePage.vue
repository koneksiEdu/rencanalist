<template>
  <div class="root">
    <div class="starfield" aria-hidden="true">
      <div v-for="i in 35" :key="i" class="star" :style="starStyle(i)" />
    </div>

    <div class="game-wrapper">
      <div class="title-bar">
        <span class="title-accent">THE</span>
        <span class="title-main">SNACK DIGGER</span>
      </div>

      <div class="hud" v-if="state === 'play'">
        <div class="hud-cell">
          <span class="hud-label">SKOR</span>
          <span class="hud-val">{{ score }}</span>
        </div>
        <!-- <div class="hud-cell center">
          <span class="hud-label">NEXT</span>
          <div class="next-preview" :style="nextPreviewStyle" />
        </div> -->
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
              <div class="worm-deco">
                <div v-for="i in 5" :key="i" class="worm-seg" :style="wormSegStyle(i)" />
              </div>
              <p class="chip">ARCADE</p>
              <h1 class="big-title">The Snack<br><em>Digger</em></h1>
              <p class="desc">Makan snack untuk tumbuh<br>Jangan tabrak dinding atau tubuhmu!</p>
              <button class="btn-start" @touchstart.prevent="beginGame" @click="beginGame">
                MULAI MAIN
              </button>
            </div>
          </div>
        </Transition>

        <Transition name="fade">
          <div v-if="state === 'over'" class="overlay">
            <div class="overlay-card">
              <p class="chip danger-chip">GAME OVER</p>
              <div class="final-score">{{ score }}</div>
              <p class="new-record" v-if="isNewRecord">REKOR BARU!</p>
              <p class="over-sub" v-else>Rekor: {{ best }}</p>
              <div class="stats-row">
                <div class="stat">
                  <span class="stat-val">{{ snacksEaten }}</span>
                  <span class="stat-key">SNACK</span>
                </div>
              </div>
              <button class="btn-start" @touchstart.prevent="beginGame" @click="beginGame">
                COBA LAGI
              </button>
            </div>
          </div>
        </Transition>
      </div>

      <div class="dpad" v-if="state === 'play'">
        <div class="dpad-row">
          <button class="dpad-btn"
            @touchstart.prevent="setDir(0,-1)" @mousedown.prevent="setDir(0,-1)">
            <svg width="22" height="22" viewBox="0 0 22 22"><path d="M11 17V5M5 11l6-6 6 6" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" fill="none"/></svg>
          </button>
        </div>
        <div class="dpad-row">
          <button class="dpad-btn"
            @touchstart.prevent="setDir(-1,0)" @mousedown.prevent="setDir(-1,0)">
            <svg width="22" height="22" viewBox="0 0 22 22"><path d="M17 11H5M11 5l-6 6 6 6" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" fill="none"/></svg>
          </button>
          <button class="dpad-btn center-btn" @touchstart.prevent="fireBoost" @mousedown.prevent="fireBoost">
            <svg width="18" height="18" viewBox="0 0 18 18"><circle cx="9" cy="9" r="6" fill="currentColor" opacity="0.3"/><circle cx="9" cy="9" r="3" fill="currentColor"/></svg>
          </button>
          <button class="dpad-btn"
            @touchstart.prevent="setDir(1,0)" @mousedown.prevent="setDir(1,0)">
            <svg width="22" height="22" viewBox="0 0 22 22"><path d="M5 11h12M11 5l6 6-6 6" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" fill="none"/></svg>
          </button>
        </div>
        <div class="dpad-row">
          <button class="dpad-btn"
            @touchstart.prevent="setDir(0,1)" @mousedown.prevent="setDir(0,1)">
            <svg width="22" height="22" viewBox="0 0 22 22"><path d="M11 5v12M5 11l6 6 6-6" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" fill="none"/></svg>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick } from 'vue'

const COLS = 18, ROWS = 22
let CELL = 18

const cvs = ref(null)
const state = ref('start')
const score = ref(0)
const best = ref(0)
const isNewRecord = ref(false)
const snacksEaten = ref(0)
const snake = ref([])

let ctx = null
let dir = { x: 1, y: 0 }
let nextDir = { x: 1, y: 0 }
let snacks = []
let loopId = null
let speed = 160
let tickCount = 0
let nextColor = '#FFD600'

let resizeObserver = null

const SNACK_TYPES = [
  { color: '#FFD600', pts: 10, prob: 0.55, r: 4 },
  { color: '#00F5C4', pts: 25, prob: 0.28, r: 5 },
  { color: '#FF6B6B', pts: 50, prob: 0.17, r: 6 },
]

const nextPreviewStyle = computed(() => ({
  background: nextColor,
  boxShadow: `0 0 8px ${nextColor}55`,
}))

function starStyle(i) {
  const s = i * 137.5
  return {
    left: (s * 31 % 100) + '%',
    top: (s * 17 % 100) + '%',
    width: (1 + i % 3) + 'px',
    height: (1 + i % 3) + 'px',
    animationDelay: (i * 0.25 % 4) + 's',
    animationDuration: (2 + i % 3) + 's',
  }
}

function wormSegStyle(i) {
  const colors = ['#00F5C4','#00E0B0','#00C89C','#00B088','#009874']
  return {
    background: colors[i - 1],
    width: (28 - i * 3) + 'px',
    height: (28 - i * 3) + 'px',
    marginLeft: i === 1 ? '0' : '-6px',
  }
}

function pickSnack() {
  const r = Math.random()
  let acc = 0
  for (const t of SNACK_TYPES) { acc += t.prob; if (r < acc) return t }
  return SNACK_TYPES[0]
}

function placeSnack() {
  const body = new Set(snake.value.map(s => s.x + ',' + s.y))
  const existing = new Set(snacks.map(s => s.x + ',' + s.y))
  const free = []
  for (let x = 0; x < COLS; x++)
    for (let y = 0; y < ROWS; y++)
      if (!body.has(x + ',' + y) && !existing.has(x + ',' + y)) free.push({ x, y })
  if (!free.length) return null
  const pos = free[Math.floor(Math.random() * free.length)]
  const t = pickSnack()
  return { ...pos, ...t }
}

function initGame() {
  const cx = Math.floor(COLS / 2), cy = Math.floor(ROWS / 2)
  snake.value = [{ x: cx, y: cy }, { x: cx - 1, y: cy }, { x: cx - 2, y: cy }]
  dir = { x: 1, y: 0 }; nextDir = { x: 1, y: 0 }
  snacks = []
  for (let i = 0; i < 3; i++) { const s = placeSnack(); if (s) snacks.push(s) }
  score.value = 0; snacksEaten.value = 0
  speed = 160; tickCount = 0; isNewRecord.value = false
  nextColor = snacks[0]?.color || '#FFD600'
}

function tick() {
  dir = { ...nextDir }
  const head = { x: snake.value[0].x + dir.x, y: snake.value[0].y + dir.y }
  if (head.x < 0 || head.x >= COLS || head.y < 0 || head.y >= ROWS) { endGame(); return }
  if (snake.value.some(s => s.x === head.x && s.y === head.y)) { endGame(); return }
  snake.value = [head, ...snake.value]
  let ate = false
  for (let i = snacks.length - 1; i >= 0; i--) {
    if (snacks[i].x === head.x && snacks[i].y === head.y) {
      score.value += snacks[i].pts
      snacksEaten.value++
      snacks.splice(i, 1)
      ate = true; break
    }
  }
  if (!ate) snake.value = snake.value.slice(0, -1)
  while (snacks.length < 3) { const s = placeSnack(); if (!s) break; snacks.push(s) }
  nextColor = snacks[0]?.color || '#FFD600'
  tickCount++
  if (tickCount % 6 === 0) speed = Math.max(65, speed - 2)
  draw()
  loopId = setTimeout(tick, speed)
}

function draw() {
  if (!ctx) return
  const W = COLS * CELL, H = ROWS * CELL
  ctx.fillStyle = '#060d0a'
  ctx.fillRect(0, 0, W, H)
  for (let x = 0; x < COLS; x++)
    for (let y = 0; y < ROWS; y++) {
      if ((x + y) % 2 === 0) {
        ctx.fillStyle = 'rgba(255,255,255,0.018)'
        ctx.fillRect(x * CELL, y * CELL, CELL, CELL)
      }
    }
  snacks.forEach(s => {
    const px = s.x * CELL + CELL / 2, py = s.y * CELL + CELL / 2
    ctx.beginPath(); ctx.arc(px, py, s.r, 0, Math.PI * 2)
    ctx.fillStyle = s.color; ctx.fill()
    ctx.beginPath(); ctx.arc(px - s.r * 0.3, py - s.r * 0.3, s.r * 0.28, 0, Math.PI * 2)
    ctx.fillStyle = 'rgba(255,255,255,0.55)'; ctx.fill()
  })
  snake.value.forEach((seg, i) => {
    const t = i / snake.value.length
    const px = seg.x * CELL, py = seg.y * CELL
    const pad = 1.5
    if (i === 0) {
      ctx.fillStyle = '#00F5C4'
    } else {
      const g = Math.round(180 - t * 60)
      ctx.fillStyle = `rgb(0,${g},${Math.round(g * 0.6)})`
    }
    ctx.beginPath()
    ctx.roundRect(px + pad, py + pad, CELL - pad * 2, CELL - pad * 2, 3)
    ctx.fill()
    if (i === 0) {
      const ex = dir.x === 0 ? CELL * 0.32 : (dir.x > 0 ? CELL * 0.62 : CELL * 0.28)
      const ey = dir.y === 0 ? CELL * 0.32 : (dir.y > 0 ? CELL * 0.62 : CELL * 0.28)
      const ex2 = dir.x === 0 ? CELL * 0.68 : CELL * ex / CELL
      const ey2 = dir.y === 0 ? CELL * 0.68 : CELL * ey / CELL
      ctx.fillStyle = '#060d0a'
      ctx.beginPath(); ctx.arc(px + ex, py + ey, 2, 0, Math.PI * 2); ctx.fill()
      if (dir.x === 0) { ctx.beginPath(); ctx.arc(px + ex2, py + ey, 2, 0, Math.PI * 2); ctx.fill() }
      else { ctx.beginPath(); ctx.arc(px + ex, py + ey2, 2, 0, Math.PI * 2); ctx.fill() }
    }
  })
}

async function beginGame() {
  state.value = 'play'
  initGame()
  await nextTick()
  // Force canvas setup with proper dimensions
  setupCanvas()
  if (!ctx) {
    console.error('Canvas context not available')
    return
  }
  draw()
  if (loopId) clearTimeout(loopId)
  loopId = setTimeout(tick, speed)
}

function endGame() {
  clearTimeout(loopId)
  if (score.value >= best.value && score.value > 0) {
    best.value = score.value; isNewRecord.value = true
  }
  state.value = 'over'
  draw()
}

function setDir(dx, dy) {
  if (dx === -dir.x && dy === -dir.y) return
  nextDir = { x: dx, y: dy }
}

function fireBoost() {}

function setupCanvas() {
  if (!cvs.value) return false
  
  // Get the actual container size
  const wrap = cvs.value.parentElement
  if (!wrap) return false
  
  // Calculate cell size based on parent width
  const containerWidth = wrap.clientWidth
  const containerHeight = wrap.clientHeight || 400
  
  // Use the smaller dimension to ensure square cells
  const maxCellByWidth = Math.floor(containerWidth / COLS)
  const maxCellByHeight = Math.floor(containerHeight / ROWS)
  CELL = Math.min(maxCellByWidth, maxCellByHeight, 24) // Cap at 24px max
  
  if (CELL < 1) CELL = 12 // fallback
  
  cvs.value.width = COLS * CELL
  cvs.value.height = ROWS * CELL
  ctx = cvs.value.getContext('2d')
  
  // Set canvas CSS dimensions to match actual size
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
  if (map[e.key]) { e.preventDefault(); setDir(...map[e.key]) }
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

// Update onMounted:
onMounted(() => {
  setupCanvas()
  if (ctx && cvs.value) {
    ctx.fillStyle = '#060d0a'
    ctx.fillRect(0, 0, cvs.value.width, cvs.value.height)
  }
  
  // Watch for container size changes
  if (cvs.value && cvs.value.parentElement) {
    resizeObserver = new ResizeObserver(() => {
      if (state.value === 'play') {
        setupCanvas()
        draw()
      }
    })
    resizeObserver.observe(cvs.value.parentElement)
  }
  
  document.addEventListener('keydown', onKey)
  document.addEventListener('touchstart', onTouchStart, { passive: true })
  document.addEventListener('touchend', onTouchEnd, { passive: true })
})

// Update onUnmounted:
onUnmounted(() => {
  clearTimeout(loopId)
  if (resizeObserver) resizeObserver.disconnect()
  document.removeEventListener('keydown', onKey)
  document.removeEventListener('touchstart', onTouchStart)
  document.removeEventListener('touchend', onTouchEnd)
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Exo+2:ital,wght@0,700;0,900;1,900&family=Rajdhani:wght@500;600;700&display=swap');

*, *::before, *::after { box-sizing: border-box; -webkit-tap-highlight-color: transparent; user-select: none; touch-action: manipulation; }

.root {
  position: relative; min-height: 100svh; width: 100%;
  background: #060d0a; display: flex; align-items: center; justify-content: center;
  overflow: hidden; font-family: 'Rajdhani', sans-serif;
}

.starfield { position: fixed; inset: 0; pointer-events: none; z-index: 0; }
.star { position: absolute; border-radius: 50%; background: white; animation: twinkle var(--dur,3s) ease-in-out infinite; opacity: 0.5; }
@keyframes twinkle { 0%,100% { opacity: 0.1; } 50% { opacity: 0.8; } }

.game-wrapper { position: relative; z-index: 1; display: flex; flex-direction: column; align-items: center; width: 100%; max-width: 420px; padding: 12px 12px 20px; gap: 10px; }

.title-bar { display: flex; align-items: baseline; gap: 8px; }
.title-accent { font-family: 'Rajdhani', sans-serif; font-weight: 600; font-size: 13px; letter-spacing: 0.25em; color: #00b88a; }
.title-main { font-family: 'Exo 2', sans-serif; font-weight: 900; font-size: 18px; letter-spacing: 0.12em; color: #00F5C4; }

.hud { width: 100%; display: flex; justify-content: space-between; align-items: center; padding: 0 4px; }
.hud-cell { display: flex; flex-direction: column; align-items: flex-start; min-width: 64px; }
.hud-cell.center { align-items: center; }
.hud-cell.right { align-items: flex-end; }
.hud-label { font-size: 9px; font-weight: 700; letter-spacing: 0.22em; color: rgba(0,245,196,0.4); }
.hud-val { font-family: 'Exo 2', sans-serif; font-weight: 900; font-size: 22px; color: #fff; line-height: 1.1; }
.next-preview { width: 18px; height: 18px; border-radius: 50%; margin-top: 4px; transition: background 0.2s; }

/* Add to your <style> section */
.canvas-wrap {
  position: relative;
  width: 100%;
  aspect-ratio: 18 / 22; /* Match COLS:ROWS ratio */
  border-radius: 16px;
  overflow: hidden;
  border: 0.5px solid rgba(0,245,196,0.18);
}

.game-canvas {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: contain;
}
.game-canvas {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.overlay { position: absolute; inset: 0; display: flex; align-items: flex-end; justify-content: center; padding-bottom: 20px; background: linear-gradient(to bottom, transparent 10%, rgba(6,13,10,0.97) 48%); border-radius: 16px; }
.overlay-card { display: flex; flex-direction: column; align-items: center; width: 100%; padding: 0 24px; gap: 8px; }

.worm-deco { display: flex; align-items: center; margin-bottom: 4px; }
.worm-seg { border-radius: 50%; animation: wfloat 2s ease-in-out infinite; }
.worm-seg:nth-child(1) { animation-delay: 0s; }
.worm-seg:nth-child(2) { animation-delay: 0.15s; }
.worm-seg:nth-child(3) { animation-delay: 0.3s; }
.worm-seg:nth-child(4) { animation-delay: 0.45s; }
.worm-seg:nth-child(5) { animation-delay: 0.6s; }
@keyframes wfloat { 0%,100% { transform: translateY(0); } 50% { transform: translateY(-5px); } }

.chip { font-size: 9px; font-weight: 700; letter-spacing: 0.28em; color: #00b88a; border: 0.5px solid rgba(0,184,138,0.4); border-radius: 99px; padding: 3px 14px; background: rgba(0,245,196,0.06); margin: 0; }
.danger-chip { color: #ff6b6b; border-color: rgba(255,107,107,0.4); background: rgba(255,107,107,0.07); }

.big-title { font-family: 'Exo 2', sans-serif; font-weight: 900; font-size: 40px; line-height: 0.95; text-align: center; margin: 0; color: #fff; }
.big-title em { font-style: italic; color: #00F5C4; }

.desc { font-size: 12px; font-weight: 600; letter-spacing: 0.04em; color: rgba(255,255,255,0.3); text-align: center; line-height: 1.9; margin: 4px 0 6px; }

.final-score { font-family: 'Exo 2', sans-serif; font-weight: 900; font-size: 62px; line-height: 1; color: #fff; margin: 4px 0; }
.new-record { font-size: 12px; font-weight: 700; letter-spacing: 0.2em; color: #FFD600; margin: 0; }
.over-sub { font-size: 12px; color: rgba(255,255,255,0.28); letter-spacing: 0.1em; margin: 0; }

.stats-row { display: flex; gap: 0; border: 0.5px solid rgba(0,245,196,0.15); border-radius: 12px; overflow: hidden; margin-bottom: 4px; }
.stat { display: flex; flex-direction: column; align-items: center; padding: 10px 28px; }
.stat + .stat { border-left: 0.5px solid rgba(0,245,196,0.1); }
.stat-val { font-family: 'Exo 2', sans-serif; font-weight: 900; font-size: 24px; color: #fff; }
.stat-key { font-size: 8px; font-weight: 700; letter-spacing: 0.18em; color: rgba(0,245,196,0.4); margin-top: 2px; }

.btn-start { border: 0.5px solid rgba(0,245,196,0.5); border-radius: 99px; padding: 13px 44px; background: rgba(0,245,196,0.08); color: #00F5C4; font-family: 'Exo 2', sans-serif; font-weight: 900; font-size: 15px; letter-spacing: 0.12em; cursor: pointer; transition: background 0.15s, transform 0.1s; }
.btn-start:active { background: rgba(0,245,196,0.18); transform: scale(0.96); }

.dpad { display: flex; flex-direction: column; align-items: center; gap: 4px; }
.dpad-row { display: flex; gap: 4px; justify-content: center; }
.dpad-btn { width: 64px; height: 64px; background: rgba(0,245,196,0.05); border: 0.5px solid rgba(0,245,196,0.18); border-radius: 14px; color: rgba(0,245,196,0.8); cursor: pointer; display: flex; align-items: center; justify-content: center; transition: background 0.1s, transform 0.1s; }
.dpad-btn:active { background: rgba(0,245,196,0.15); transform: scale(0.92); }
.center-btn { color: rgba(0,245,196,0.45); }

.fade-enter-active, .fade-leave-active { transition: opacity 0.25s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>