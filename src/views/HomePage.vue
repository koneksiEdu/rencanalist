<template>
  <div class="shell">
    <!-- ═══ PARALLAX STARFIELD ═══ -->
    <canvas ref="starCanvas" class="star-canvas" />

    <!-- ═══ HOME SCREEN - RETRO ARCADE STYLE ═══ -->
    <transition name="fade">
      <div v-if="screen === 'home'" class="screen home-screen">
        <!-- Top deco -->
        <div class="arcade-header">
          <div class="scanline"></div>
          <div class="crt-glow"></div>
          <div class="header-badge">
            <span>✦ ARCADIA ARCADE ✦</span>
            <span class="badge-flash">INSERT COIN</span>
          </div>
        </div>

        <!-- Main Logo Area -->
        <div class="logo-arena">
          <div class="logo-glitch" data-text="PADEL">PADEL</div>
          <div class="logo-buster">
            <span>B</span><span>U</span><span>S</span><span>T</span><span>E</span><span>R</span>
          </div>
          <div class="logo-sub">VER 2.0.77 • NEO TOKYO</div>
        </div>

        <!-- Preview Arena (different brick style) -->
        <div class="preview-arena">
          <div class="preview-stage">
          </div>
        </div>

        <!-- Menu Buttons -->
        <div class="menu-buttons">
          <button class="arcade-btn primary-btn" @click="startGame">
            <span class="btn-pixel">▶</span> START <span class="btn-pixel">◀</span>
          </button>
          <button class="arcade-btn secondary-btn" @click="screen='help'">
            <span class="btn-pixel">⍟</span> MANUAL <span class="btn-pixel">⍟</span>
          </button>
          <button class="arcade-btn secondary-btn" @click="screen='stats'">
            <span class="btn-pixel">⍟</span> RECORDS <span class="btn-pixel">⍟</span>
          </button>
        </div>

        <div class="credit-line">© 2084 • NEO ARCADIA CORP</div>
      </div>
    </transition>

    <!-- ═══ STATS SCREEN ═══ -->
    <transition name="slide">
      <div v-if="screen==='stats'" class="screen stats-screen">
        <div class="stats-container">
          <button class="back-btn-arcade" @click="screen='home'">⌂ BACK</button>
          
          <div class="stats-title-wrap">
            <div class="stats-glitch">RECORDS</div>
            <div class="stats-glitch-sub">HALL OF FAME</div>
          </div>

          <div class="stats-card">
            <div class="stats-row">
              <span class="stats-label">TOP SCORE</span>
              <span class="stats-value">{{ highScore }}</span>
            </div>
            <div class="stats-divider"></div>
            <div class="stats-row">
              <span class="stats-label">DEEPEST SECTOR</span>
              <span class="stats-value">{{ bestLevel }}</span>
            </div>
          </div>

          <button class="arcade-btn primary-btn reset-stats" @click="resetStats">
            <span class="btn-pixel">⟳</span> RESET RECORDS <span class="btn-pixel">⟳</span>
          </button>
        </div>
      </div>
    </transition>

    <!-- ═══ HELP SCREEN ═══ -->
    <transition name="slide">
      <div v-if="screen==='help'" class="screen help-screen">
        <div class="help-container">
          <button class="back-btn-arcade" @click="screen='home'">⌂ BACK</button>
          
          <div class="help-title-wrap">
            <div class="help-glitch">MISSION</div>
            <div class="help-glitch-sub">BRIEFING</div>
          </div>

          <div class="help-grid">
            <div class="help-card">
              <div class="help-card-header">◈ CONTROLS ◈</div>
              <div class="help-item"><span class="key">◀  ▶</span><span>MOVE PADEL</span></div>
              <div class="help-item"><span class="key">␣</span><span>SHOOT / PAUSE</span></div>
              <div class="help-item"><span class="key">⟳</span><span>RETRY</span></div>
            </div>

            <div class="help-card">
              <div class="help-card-header">◈ TARGETS ◈</div>
              <div class="help-item"><span class="badge-red">●</span><span>ARMORED (2 HITS)</span></div>
              <div class="help-item"><span class="badge-dark">■</span><span>OBSTACLE (INVINCIBLE)</span></div>
              <div class="help-item"><span class="badge-gold">◆</span><span>GOLDEN (50 PTS)</span></div>
            </div>

            <div class="help-card">
              <div class="help-card-header">◈ POWER CELLS ◈</div>
              <div class="help-item"><span class="pu-ico">🛡</span><span>SHIELD (WIDE PADEL)</span></div>
              <div class="help-item"><span class="pu-ico">✦</span><span>NOVA (TRIPLE BALL)</span></div>
              <div class="help-item"><span class="pu-ico">⌛</span><span>WARP (SLOW TIME)</span></div>
              <div class="help-item"><span class="pu-ico">⚡</span><span>LASER (PIERCE)</span></div>
              <div class="help-item"><span class="pu-ico">❤</span><span>REPAIR (+1 LIFE)</span></div>
            </div>
          </div>

          <button class="arcade-btn primary-btn launch-now" @click="startGame">
            <span class="btn-pixel">⚡</span> LAUNCH MISSION <span class="btn-pixel">⚡</span>
          </button>
        </div>
      </div>
    </transition>

    <!-- ═══ GAME SCREEN - MONOCHROME GAMEBOY STYLE ═══ -->
    <transition name="fade">
      <div v-if="screen==='game'" class="screen game-screen">
        <canvas ref="gameCanvas" class="game-canvas"/>

        <!-- New HUD Design - Gameboy Monochrome -->
        <div class="hud-new">
          <div class="hud-score">
            <span class="label">SCORE</span>
            <span class="value">{{ String(score).padStart(6,'0') }}</span>
          </div>
          <div class="hud-level">
            <span class="label">SECTOR</span>
            <span class="value">{{ level }}</span>
          </div>
          <div class="hud-lives">
            <span class="label">LIVES</span>
            <div class="lives-icons">
              <span v-for="i in 3" :key="i" class="life-icon" :class="{active:i<=lives}">❤</span>
            </div>
          </div>
        </div>

        <!-- Power-up bar -->
        <div class="pu-bar-new">
          <div v-if="puShieldActive" class="pu-tag shield">SHIELD</div>
          <div v-if="puNova" class="pu-tag nova">NOVA</div>
          <div v-if="puWarpActive" class="pu-tag warp">WARP</div>
          <div v-if="puLaserActive" class="pu-tag laser">LASER</div>
        </div>

        <!-- Combo flashes -->
        <div v-for="cf in comboFlashes" :key="cf.id" class="combo-flash-new"
          :style="{top:cf.y+'px'}">{{ cf.text }}</div>

        <!-- Shoot hint -->
        <transition name="fade">
          <div v-if="waitingLaunch && !isPaused" class="launch-hint-new">
            <span class="hint-arrow">▼▼▼</span> PRESS SHOOT <span class="hint-arrow">▼▼▼</span>
          </div>
        </transition>

        <!-- New Touch Controls with Pause button below Shoot -->
        <div class="touch-controls">
          <button class="touch-btn left" @touchstart.prevent="startMove('left')" @touchend.prevent="stopMove" @mousedown="startMove('left')" @mouseup="stopMove">
            <span>◀</span>
          </button>
          <div class="center-controls">
            <button class="touch-btn launch" @touchstart.prevent="launchBall" @mousedown="launchBall">
              <span>⚡ SHOOT ⚡</span>
            </button>
            <button class="touch-btn pause-bottom" @click="togglePause">
              <span>⏸ PAUSE</span>
            </button>
          </div>
          <button class="touch-btn right" @touchstart.prevent="startMove('right')" @touchend.prevent="stopMove" @mousedown="startMove('right')" @mouseup="stopMove">
            <span>▶</span>
          </button>
        </div>
      </div>
    </transition>

    <!-- ═══ PAUSE MODAL (Gameboy Style) ═══ -->
    <transition name="fade">
      <div v-if="screen==='game' && isPaused && !showModal" class="modal-overlay-new">
        <div class="modal-arcade">
          <div class="modal-header-pause">⏸ PAUSED ⏸</div>
          <div class="modal-stats">
            <div class="stat-line"><span>SCORE</span><span>{{ score }}</span></div>
            <div class="stat-line"><span>SECTOR</span><span>{{ level }}</span></div>
            <div class="stat-line"><span>LIVES</span><span>{{ lives }}/3</span></div>
          </div>
          <div class="modal-buttons">
            <button class="modal-arcade-btn resume" @click="togglePause">▶ RESUME</button>
            <button class="modal-arcade-btn abort" @click="goHome">◼ EXIT</button>
          </div>
        </div>
      </div>
    </transition>

    <!-- ═══ RESULT MODAL (Gameboy Style) ═══ -->
    <transition name="fade">
      <div v-if="showModal" class="modal-overlay-new">
        <div class="modal-result" :class="modalType">
          <div class="result-icon">{{ modalIcon }}</div>
          <div class="result-title">{{ modalTitle }}</div>
          <div class="result-sub">{{ modalSub }}</div>
          <div class="result-score">{{ score }}</div>
          <div v-if="isNewHigh" class="result-record">★ NEW RECORD! ★</div>
          <div class="result-detail">
            <span>SECTOR {{ level }}</span>
            <span>🔰 {{ totalDestroyed }} DESTROYED</span>
          </div>
          <div class="modal-buttons">
            <button class="modal-arcade-btn resume" @click="modalAction">
              {{ modalType==='levelup'?'▶ NEXT SECTOR':'⟳ RETRY' }}
            </button>
            <button class="modal-arcade-btn abort" @click="goHome">⌂ MENU</button>
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick } from 'vue'

// ─── CONSTANTS – MONOCHROME GAMEBOY PALETTE ─────────────────────────────────
const GAMEBOY_PALETTE = {
  bg: '#9bbc0f',      // Green typical Gameboy
  dark: '#306230',    // Dark green
  mid: '#8bac0f',     // Medium green  
  light: '#e0f8d0',   // Light green/cream
  white: '#f8f8f8'    // White
}

const COLS = 8, ROWS = 6
const BRICK_PAD = 4, BRICK_TOP = 78, BRICK_H = 20

const BRICK_SHADES = [GAMEBOY_PALETTE.dark, GAMEBOY_PALETTE.mid, GAMEBOY_PALETTE.light, GAMEBOY_PALETTE.bg]

const PU_TYPES = ['shield','nova','warp','laser','repair']

// ─── STARS (parallax) – simplified for monochrome ───────────────────────────
const starCanvas = ref(null)
let starCtx = null, starW = 0, starH = 0
const STAR_LAYERS = [
  { count:80, speed:0.05, size:0.8, opacity:0.4 },
  { count:50, speed:0.12, size:1.2, opacity:0.6 },
  { count:25, speed:0.25, size:1.8, opacity:0.8 },
]
let stars = []

function initStars() {
  const el = starCanvas.value
  starW = el.width = window.innerWidth
  starH = el.height = window.innerHeight
  starCtx = el.getContext('2d')
  stars = []
  STAR_LAYERS.forEach((layer, li) => {
    for (let i = 0; i < layer.count; i++) {
      stars.push({ 
        x: Math.random()*starW, 
        y: Math.random()*starH, 
        layer: li,
        speed: layer.speed * (0.7 + Math.random()*0.6), 
        size: layer.size * (0.6 + Math.random()*0.8),
        opacity: layer.opacity * (0.4 + Math.random()*0.6)
      })
    }
  })
}

let starAnimId = null
function drawStars() {
  if (!starCtx) return
  starCtx.clearRect(0,0,starW,starH)
  
  starCtx.fillStyle = GAMEBOY_PALETTE.bg
  starCtx.fillRect(0,0,starW,starH)
  
  stars.forEach(s => {
    s.y += s.speed
    if (s.y > starH + 10) {
      s.y = -10
      s.x = Math.random() * starW
    }
    
    starCtx.save()
    starCtx.globalAlpha = s.opacity
    starCtx.fillStyle = GAMEBOY_PALETTE.mid
    starCtx.beginPath()
    starCtx.arc(s.x, s.y, s.size, 0, Math.PI * 2)
    starCtx.fill()
    starCtx.restore()
  })
  
  starAnimId = requestAnimationFrame(drawStars)
}

// ─── LEVEL GENERATOR ─────────────────────────────────────────────────────────
function generateLevel(lv) {
  const bricks = [], walls = []
  
  const patterns = [
    (r,c) => ({ on: true, hp: 1 }),
    (r,c) => ({ on: (r + c) % 2 === 0, hp: r === 2 ? 2 : 1 }),
    (r,c) => ({ on: Math.abs(r - 2.5) + Math.abs(c - 3.5) * 0.8 < 3, hp: 1 }),
    (r,c) => ({ on: r === 0 || r === ROWS-1 || c === 0 || c === COLS-1, hp: 2 }),
    (r,c) => ({ on: (r * 2 + c) % 3 < 2, hp: r < 2 ? 2 : 1 }),
    (r,c) => ({ on: Math.random() > 0.2, hp: Math.random() < 0.4 ? 2 : 1 }),
  ]
  
  const pfn = patterns[Math.min(lv - 1, patterns.length - 1)]
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      const b = pfn(r, c)
      if (b.on) {
        bricks.push({
          row: r, col: c, hp: b.hp, maxHp: b.hp, active: true,
          hasPU: Math.random() < 0.12,
          puType: PU_TYPES[Math.floor(Math.random() * PU_TYPES.length)]
        })
      }
    }
  }
  
  const wallCount = Math.min(lv + 1, 5)
  const used = new Set()
  let tries = 0
  while (walls.length < wallCount && tries < 150) {
    tries++
    const row = Math.floor(Math.random() * (ROWS + 1)) + ROWS
    const col = Math.floor(Math.random() * COLS)
    const key = `${row}-${col}`
    if (used.has(key)) continue
    used.add(key)
    walls.push({ row, col, active: true })
  }
  
  return { bricks, walls }
}

// ─── STATE ───────────────────────────────────────────────────────────────────
const screen = ref('home')
const gameCanvas = ref(null)
const score = ref(0), lives = ref(3), level = ref(1)
const highScore = ref(parseInt(localStorage.getItem('spb_hs') || '0'))
const bestLevel = ref(parseInt(localStorage.getItem('spb_bl') || '1'))
const isPaused = ref(false), waitingLaunch = ref(true)
const showModal = ref(false), modalType = ref('gameover')
const modalTitle = ref(''), modalSub = ref('')
const isNewHigh = ref(false), totalDestroyed = ref(0)
const comboFlashes = ref([])

const puShieldActive = ref(false), puShieldTimer = ref(0)
const puNova = ref(false)
const puWarpActive = ref(false), puWarpTimer = ref(0)
const puLaserActive = ref(false), puLaserTimer = ref(0)

// ─── COMPUTED ────────────────────────────────────────────────────────────────
const modalIcon = computed(() => {
  if (modalType.value === 'win') return '🏆'
  if (modalType.value === 'levelup') return '⚡'
  return '💀'
})

// ─── RESET STATS ─────────────────────────────────────────────────────────────
function resetStats() {
  highScore.value = 0
  bestLevel.value = 1
  localStorage.setItem('spb_hs', '0')
  localStorage.setItem('spb_bl', '1')
}

// ─── GAME INTERNALS ──────────────────────────────────────────────────────────
let ctx = null, animId = null
let W = 0, H = 0, BRICK_W = 0
let balls = [], bricks = [], walls = [], powerups = []
let combo = 0, flashId = 0, lastTs = 0
const paddle = { x: 0, y: 0, w: 0, h: 18, normalWidth: 0 }
const keys = { left: false, right: false }
const PADDLE_SPEED = 9

let shieldIv = null, warpIv = null, laserIv = null

function initCanvas() {
  const el = gameCanvas.value
  const rect = el.parentElement.getBoundingClientRect()
  W = rect.width
  H = rect.height
  el.width = W
  el.height = H
  ctx = el.getContext('2d')
  BRICK_W = Math.floor((W - 20) / COLS) - BRICK_PAD
  paddle.normalWidth = Math.max(70, W * 0.22)
  paddle.w = puShieldActive.value ? W * 0.4 : paddle.normalWidth
  paddle.x = W / 2 - paddle.w / 2
  paddle.y = H - 100
}

function makeBall(extra = false) {
  const spd = 5.0 + level.value * 0.2
  return {
    x: paddle.x + paddle.w / 2, y: paddle.y - 10,
    r: 6, vx: 0, vy: 0, speed: spd,
    attached: !extra, dead: false,
    trail: [],
    ...(extra ? { vx: (Math.random() > 0.5 ? 1 : -1) * spd * 0.7, vy: -spd, attached: false } : {})
  }
}

function loadLevel() {
  const data = generateLevel(level.value)
  bricks = data.bricks
  walls = data.walls
  powerups = []
  combo = 0
  balls = [makeBall()]
  waitingLaunch.value = true
}

function resetPU() {
  if (shieldIv) clearInterval(shieldIv)
  if (warpIv) clearInterval(warpIv)
  if (laserIv) clearInterval(laserIv)
  puShieldActive.value = false
  puShieldTimer.value = 0
  puNova.value = false
  puWarpActive.value = false
  puWarpTimer.value = 0
  puLaserActive.value = false
  puLaserTimer.value = 0
  if (paddle.normalWidth) paddle.w = paddle.normalWidth
}

function startTimer(activeRef, timerRef, ivRef, duration, onEnd) {
  if (ivRef) clearInterval(ivRef)
  activeRef.value = true
  timerRef.value = duration
  const iv = setInterval(() => {
    if (timerRef.value > 0) {
      timerRef.value -= 0.1
      if (timerRef.value <= 0) {
        activeRef.value = false
        onEnd()
        clearInterval(iv)
      }
    }
  }, 100)
  return iv
}

// ─── CONTROLS ────────────────────────────────────────────────────────────────
function startMove(d) { keys[d] = true }
function stopMove() { keys.left = false; keys.right = false }

function launchBall() {
  if (!waitingLaunch.value) return
  waitingLaunch.value = false
  balls.forEach(b => {
    if (!b.attached) return
    b.attached = false
    b.vx = (Math.random() * 0.6 - 0.3) * b.speed
    b.vy = -b.speed
  })
}

function onKeyDown(e) {
  if (screen.value !== 'game') return
  if (e.code === 'ArrowLeft') { e.preventDefault(); keys.left = true }
  if (e.code === 'ArrowRight') { e.preventDefault(); keys.right = true }
  if (e.code === 'Space') { e.preventDefault(); waitingLaunch.value ? launchBall() : togglePause() }
  if (e.code === 'Escape') togglePause()
}

function onKeyUp(e) {
  if (e.code === 'ArrowLeft') keys.left = false
  if (e.code === 'ArrowRight') keys.right = false
}

// ─── GAME LOOP ───────────────────────────────────────────────────────────────
function loop(ts) {
  if (!lastTs) lastTs = ts
  const dt = Math.min((ts - lastTs) / 16.67, 3)
  lastTs = ts
  if (!isPaused.value && screen.value === 'game' && !showModal.value) updateGame(dt)
  drawGame()
  animId = requestAnimationFrame(loop)
}

function updateGame(dt) {
  const spd = PADDLE_SPEED * dt
  if (keys.left) paddle.x = Math.max(0, paddle.x - spd)
  if (keys.right) paddle.x = Math.min(W - paddle.w, paddle.x + spd)
  
  balls.forEach(b => {
    if (b.attached) {
      b.x = paddle.x + paddle.w / 2
      b.y = paddle.y - b.r
      return
    }
    const slow = puWarpActive.value ? 0.45 : 1
    b.trail.push({ x: b.x, y: b.y })
    if (b.trail.length > 8) b.trail.shift()
    b.x += b.vx * dt * slow
    b.y += b.vy * dt * slow
    
    if (b.x - b.r < 0) { b.x = b.r; b.vx = Math.abs(b.vx) }
    if (b.x + b.r > W) { b.x = W - b.r; b.vx = -Math.abs(b.vx) }
    if (b.y - b.r < 0) { b.y = b.r; b.vy = Math.abs(b.vy) }
    
    if (b.vy > 0 && b.y + b.r >= paddle.y && b.y + b.r <= paddle.y + paddle.h + 4 && 
        b.x >= paddle.x && b.x <= paddle.x + paddle.w) {
      const hit = (b.x - (paddle.x + paddle.w / 2)) / (paddle.w / 2)
      const angle = hit * 70 * Math.PI / 180
      b.vx = Math.sin(angle) * b.speed
      b.vy = -Math.abs(Math.cos(angle) * b.speed)
      b.y = paddle.y - b.r
      combo = 0
    }
    
    if (b.y - b.r > H) b.dead = true
  })
  
  const alive = balls.filter(b => !b.dead)
  if (alive.length < balls.length && alive.length === 0) {
    lives.value--
    combo = 0
    balls = []
    if (lives.value <= 0) {
      gameOver()
      return
    }
    balls = [makeBall()]
    waitingLaunch.value = true
    return
  }
  balls = alive
  
  walls.forEach(wl => {
    if (!wl.active) return
    const wx = 10 + wl.col * (BRICK_W + BRICK_PAD)
    const wy = BRICK_TOP + wl.row * (BRICK_H + BRICK_PAD)
    balls.forEach(b => {
      if (b.attached) return
      if (b.x + b.r < wx || b.x - b.r > wx + BRICK_W || b.y + b.r < wy || b.y - b.r > wy + BRICK_H) return
      const ox = Math.abs(b.x - (wx + BRICK_W / 2)) / (BRICK_W / 2)
      const oy = Math.abs(b.y - (wy + BRICK_H / 2)) / (BRICK_H / 2)
      if (ox > oy) b.vx = -b.vx
      else b.vy = -b.vy
    })
  })
  
  bricks.forEach(bk => {
    if (!bk.active) return
    const bx = 10 + bk.col * (BRICK_W + BRICK_PAD)
    const by = BRICK_TOP + bk.row * (BRICK_H + BRICK_PAD)
    balls.forEach(b => {
      if (b.attached) return
      if (b.x + b.r < bx || b.x - b.r > bx + BRICK_W || b.y + b.r < by || b.y - b.r > by + BRICK_H) return
      bk.hp--
      combo++
      
      if (bk.hp <= 0) {
        bk.active = false
        totalDestroyed.value++
        const pts = bk.maxHp > 2 ? 35 : (bk.maxHp > 1 ? 20 : 10)
        score.value += pts * Math.max(1, Math.floor(combo / 3))
        if (bk.hasPU) powerups.push({ x: bx + BRICK_W / 2, y: by, type: bk.puType, dead: false, vy: 2.5 })
        if (combo >= 3) addComboFlash(bx + BRICK_W / 2, by, combo)
      }
      
      if (!puLaserActive.value) {
        const ox = Math.abs(b.x - (bx + BRICK_W / 2)) / (BRICK_W / 2)
        const oy = Math.abs(b.y - (by + BRICK_H / 2)) / (BRICK_H / 2)
        if (ox > oy) b.vx = -b.vx
        else b.vy = -b.vy
      }
    })
  })
  
  if (bricks.every(bk => !bk.active)) {
    nextLevel()
    return
  }
  
  powerups.forEach(pu => {
    pu.y += pu.vy * dt
    if (pu.y >= paddle.y && pu.y <= paddle.y + paddle.h && pu.x >= paddle.x && pu.x <= paddle.x + paddle.w) {
      collectPU(pu.type)
      pu.dead = true
    }
    if (pu.y > H) pu.dead = true
  })
  powerups = powerups.filter(p => !p.dead)
}

function collectPU(type) {
  if (type === 'shield') {
    startTimer(puShieldActive, puShieldTimer, shieldIv, 8, () => { paddle.w = paddle.normalWidth })
    paddle.w = W * 0.4
  } else if (type === 'nova') {
    puNova.value = true
    balls.push(makeBall(true))
    balls.push(makeBall(true))
    setTimeout(() => puNova.value = false, 200)
  } else if (type === 'warp') {
    startTimer(puWarpActive, puWarpTimer, warpIv, 5, () => {})
  } else if (type === 'laser') {
    startTimer(puLaserActive, puLaserTimer, laserIv, 4, () => {})
  } else if (type === 'repair') {
    lives.value = Math.min(3, lives.value + 1)
  }
}

function addComboFlash(x, y, c) {
  const id = flashId++
  comboFlashes.value.push({ id, y: y + 30, text: `×${c} COMBO!` })
  setTimeout(() => { comboFlashes.value = comboFlashes.value.filter(f => f.id !== id) }, 800)
}

// ─── EVENTS ──────────────────────────────────────────────────────────────────
function saveScore() {
  isNewHigh.value = score.value > highScore.value
  if (isNewHigh.value) {
    highScore.value = score.value
    localStorage.setItem('spb_hs', score.value)
  }
  if (level.value > bestLevel.value) {
    bestLevel.value = level.value
    localStorage.setItem('spb_bl', level.value)
  }
}

function nextLevel() {
  saveScore()
  if (level.value >= 7) {
    modalType.value = 'win'
    modalTitle.value = 'VICTORY!'
    modalSub.value = 'ALL SECTORS COMPLETE'
  } else {
    modalType.value = 'levelup'
    modalTitle.value = `SECTOR ${level.value} CLEARED`
    modalSub.value = 'ADVANCING TO NEXT SECTOR'
  }
  showModal.value = true
}

function gameOver() {
  saveScore()
  modalType.value = 'gameover'
  modalTitle.value = 'GAME OVER'
  modalSub.value = 'MISSION FAILED'
  showModal.value = true
}

function modalAction() {
  showModal.value = false
  if (modalType.value === 'levelup') {
    level.value++
    resetPU()
    loadLevel()
  } else {
    score.value = 0
    lives.value = 3
    level.value = 1
    totalDestroyed.value = 0
    resetPU()
    loadLevel()
  }
  isPaused.value = false
}

function startGame() {
  screen.value = 'game'
  score.value = 0
  lives.value = 3
  level.value = 1
  totalDestroyed.value = 0
  showModal.value = false
  isPaused.value = false
  resetPU()
  comboFlashes.value = []
  nextTick(() => {
    initCanvas()
    loadLevel()
    if (animId) cancelAnimationFrame(animId)
    lastTs = 0
    animId = requestAnimationFrame(loop)
  })
}

function goHome() {
  if (animId) cancelAnimationFrame(animId)
  resetPU()
  showModal.value = false
  isPaused.value = false
  screen.value = 'home'
}

function togglePause() {
  if (!showModal.value) isPaused.value = !isPaused.value
}

// ─── DRAW GAME – MONOCHROME GAMEBOY STYLE ───────────────────────────────────
function drawGame() {
  if (!ctx) return
  ctx.clearRect(0, 0, W, H)
  
  ctx.fillStyle = GAMEBOY_PALETTE.bg
  ctx.fillRect(0, 0, W, H)
  
  bricks.forEach(bk => {
    if (!bk.active) return
    const bx = 10 + bk.col * (BRICK_W + BRICK_PAD)
    const by = BRICK_TOP + bk.row * (BRICK_H + BRICK_PAD)
    
    const shadeIndex = (bk.maxHp - bk.hp + bk.row) % BRICK_SHADES.length
    const baseColor = BRICK_SHADES[shadeIndex]
    
    ctx.fillStyle = baseColor
    ctx.fillRect(bx, by, BRICK_W, BRICK_H)
    
    ctx.strokeStyle = GAMEBOY_PALETTE.dark
    ctx.lineWidth = 1
    ctx.strokeRect(bx, by, BRICK_W, BRICK_H)
    
    if (bk.maxHp > 1 && bk.hp < bk.maxHp) {
      ctx.fillStyle = GAMEBOY_PALETTE.light
      ctx.fillRect(bx + 2, by + BRICK_H - 4, (BRICK_W - 4) * (bk.hp / bk.maxHp), 2)
    }
    
    if (bk.hasPU) {
      ctx.fillStyle = GAMEBOY_PALETTE.light
      ctx.font = 'bold 10px monospace'
      ctx.textAlign = 'center'
      ctx.textBaseline = 'middle'
      ctx.fillText('?', bx + BRICK_W - 8, by + BRICK_H / 2)
    }
  })
  
  walls.forEach(wl => {
    if (!wl.active) return
    const wx = 10 + wl.col * (BRICK_W + BRICK_PAD)
    const wy = BRICK_TOP + wl.row * (BRICK_H + BRICK_PAD)
    
    ctx.fillStyle = GAMEBOY_PALETTE.dark
    ctx.fillRect(wx, wy, BRICK_W, BRICK_H)
    
    ctx.strokeStyle = GAMEBOY_PALETTE.mid
    ctx.lineWidth = 1
    ctx.strokeRect(wx + 2, wy + 2, BRICK_W - 4, BRICK_H - 4)
    
    ctx.fillStyle = GAMEBOY_PALETTE.light
    ctx.font = 'bold 10px monospace'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText('█', wx + BRICK_W / 2, wy + BRICK_H / 2)
  })
  
  const puSymbols = { shield: '🛡', nova: '✦', warp: '⌛', laser: '⚡', repair: '❤' }
  
  powerups.forEach(pu => {
    ctx.fillStyle = GAMEBOY_PALETTE.light
    ctx.font = '14px monospace'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(puSymbols[pu.type], pu.x, pu.y)
    ctx.strokeStyle = GAMEBOY_PALETTE.dark
    ctx.beginPath()
    ctx.arc(pu.x, pu.y, 10, 0, Math.PI * 2)
    ctx.stroke()
  })
  
  ctx.fillStyle = GAMEBOY_PALETTE.dark
  ctx.fillRect(paddle.x, paddle.y, paddle.w, paddle.h)
  ctx.fillStyle = GAMEBOY_PALETTE.light
  ctx.fillRect(paddle.x + 4, paddle.y + 3, paddle.w - 8, paddle.h - 6)
  ctx.fillStyle = GAMEBOY_PALETTE.bg
  ctx.fillRect(paddle.x + paddle.w/2 - 6, paddle.y + 5, 12, 4)
  
  balls.forEach(b => {
    b.trail.forEach((t, i) => {
      ctx.globalAlpha = 0.3 - i * 0.04
      ctx.fillStyle = GAMEBOY_PALETTE.mid
      ctx.beginPath()
      ctx.arc(t.x, t.y, b.r * 0.5, 0, Math.PI * 2)
      ctx.fill()
      ctx.globalAlpha = 1
    })
    
    ctx.fillStyle = GAMEBOY_PALETTE.light
    ctx.beginPath()
    ctx.arc(b.x, b.y, b.r, 0, Math.PI * 2)
    ctx.fill()
    ctx.fillStyle = GAMEBOY_PALETTE.white
    ctx.beginPath()
    ctx.arc(b.x - 2, b.y - 2, 2, 0, Math.PI * 2)
    ctx.fill()
  })
}

// ─── LIFECYCLE ───────────────────────────────────────────────────────────────
onMounted(() => {
  initStars()
  starAnimId = requestAnimationFrame(drawStars)
  window.addEventListener('keydown', onKeyDown)
  window.addEventListener('keyup', onKeyUp)
  window.addEventListener('mouseup', stopMove)
  window.addEventListener('resize', () => {
    if (starCanvas.value) {
      starW = starCanvas.value.width = window.innerWidth
      starH = starCanvas.value.height = window.innerHeight
      initStars()
    }
    if (screen.value === 'game' && ctx) initCanvas()
  })
})

onUnmounted(() => {
  if (animId) cancelAnimationFrame(animId)
  if (starAnimId) cancelAnimationFrame(starAnimId)
  if (shieldIv) clearInterval(shieldIv)
  if (warpIv) clearInterval(warpIv)
  if (laserIv) clearInterval(laserIv)
  window.removeEventListener('keydown', onKeyDown)
  window.removeEventListener('keyup', onKeyUp)
  window.removeEventListener('mouseup', stopMove)
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&family=VT323&display=swap');

* {
  user-select: none;
  -webkit-tap-highlight-color: transparent;
}

.shell {
  --gb-bg: #9bbc0f;
  --gb-dark: #306230;
  --gb-mid: #8bac0f;
  --gb-light: #e0f8d0;
  --gb-white: #f8f8f8;
  
  position: relative;
  width: 100%;
  max-width: 430px;
  height: 100dvh;
  margin: 0 auto;
  background: #000;
  overflow: hidden;
  font-family: 'VT323', 'Courier New', monospace;
}

.star-canvas {
  position: absolute;
  inset: 0;
  z-index: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

.screen {
  position: absolute;
  inset: 0;
  z-index: 10;
  display: flex;
  flex-direction: column;
  align-items: center;
  overflow-y: auto;
  overflow-x: hidden;
}

/* ========== HOME SCREEN - GAMEBOY STYLE ========== */
.home-screen {
  padding: 20px 16px;
  background: var(--gb-bg);
}

.arcade-header {
  width: 100%;
  text-align: center;
  margin-bottom: 20px;
}

.header-badge {
  display: inline-flex;
  flex-direction: column;
  gap: 4px;
  padding: 6px 12px;
  background: var(--gb-dark);
  border: 2px solid var(--gb-light);
  font-size: 7px;
  letter-spacing: 2px;
  font-family: 'Press Start 2P', monospace;
  color: var(--gb-light);
}

.badge-flash {
  animation: blink 1s step-end infinite;
  color: var(--gb-white);
}

@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.3; }
}

.logo-arena {
  text-align: center;
  margin: 10px 0 20px;
}

.logo-glitch {
  font-family: 'Press Start 2P', monospace;
  font-size: 42px;
  font-weight: 900;
  color: var(--gb-dark);
  text-shadow: 3px 3px 0 var(--gb-light);
  letter-spacing: 2px;
}

.logo-buster {
  display: flex;
  justify-content: center;
  gap: 4px;
  margin: 8px 0;
}

.logo-buster span {
  font-family: 'Press Start 2P', monospace;
  font-size: 24px;
  font-weight: 900;
  color: var(--gb-dark);
  background: var(--gb-light);
  padding: 2px 4px;
}

.logo-sub {
  font-size: 7px;
  letter-spacing: 2px;
  color: var(--gb-dark);
  font-family: 'Press Start 2P', monospace;
  opacity: 0.7;
}

.preview-arena {
  margin: 16px 0;
  width: 100%;
}

.arena-label {
  text-align: center;
  font-size: 7px;
  letter-spacing: 2px;
  color: var(--gb-dark);
  margin-bottom: 6px;
  font-family: 'Press Start 2P', monospace;
}

.preview-stage {
  position: relative;
  width: 100%;
  height: 36px;
  background: var(--gb-dark);
  border-radius: 4px;
}

.menu-buttons {
  display: flex;
  flex-direction: column;
  gap: 12px;
  width: 100%;
  max-width: 260px;
  margin: 10px 0;
}

.arcade-btn {
  font-family: 'Press Start 2P', monospace;
  font-size: 10px;
  padding: 14px 0;
  border: none;
  cursor: pointer;
  transition: all 0.1s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
}

.primary-btn {
  background: var(--gb-dark);
  border: 2px solid var(--gb-light);
  color: var(--gb-light);
  box-shadow: 4px 4px 0 var(--gb-dark);
}

.primary-btn:active {
  transform: translate(2px, 2px);
  box-shadow: 2px 2px 0 var(--gb-dark);
}

.secondary-btn {
  background: transparent;
  border: 2px solid var(--gb-dark);
  color: var(--gb-dark);
}

.secondary-btn:active {
  transform: scale(0.97);
}

.btn-pixel {
  font-size: 12px;
}

.credit-line {
  margin-top: 16px;
  font-size: 6px;
  letter-spacing: 2px;
  color: var(--gb-dark);
  opacity: 0.6;
  font-family: monospace;
}

/* ========== STATS SCREEN ========== */
.stats-screen {
  background: var(--gb-bg);
  padding: 20px 16px;
}

.stats-container {
  width: 100%;
  max-width: 300px;
}

.back-btn-arcade {
  background: var(--gb-dark);
  border: 2px solid var(--gb-light);
  color: var(--gb-light);
  padding: 6px 12px;
  font-family: 'Press Start 2P', monospace;
  font-size: 8px;
  cursor: pointer;
  margin-bottom: 20px;
}

.stats-title-wrap {
  text-align: center;
  margin-bottom: 30px;
}

.stats-glitch {
  font-family: 'Press Start 2P', monospace;
  font-size: 28px;
  font-weight: 900;
  color: var(--gb-dark);
  text-shadow: 2px 2px 0 var(--gb-light);
}

.stats-glitch-sub {
  font-size: 8px;
  letter-spacing: 3px;
  color: var(--gb-dark);
  font-family: monospace;
}

.stats-card {
  background: var(--gb-dark);
  border: 2px solid var(--gb-light);
  padding: 24px;
  margin-bottom: 24px;
}

.stats-row {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  padding: 12px 0;
}

.stats-label {
  font-family: 'Press Start 2P', monospace;
  font-size: 8px;
  color: var(--gb-light);
}

.stats-value {
  font-family: 'Press Start 2P', monospace;
  font-size: 18px;
  font-weight: bold;
  color: var(--gb-white);
}

.stats-divider {
  height: 1px;
  background: var(--gb-mid);
  margin: 4px 0;
}

.reset-stats {
  width: 100%;
  margin-top: 8px;
  background: transparent;
  border: 2px solid var(--gb-dark);
  color: var(--gb-dark);
  box-shadow: none;
}

/* ========== HELP SCREEN ========== */
.help-screen {
  background: var(--gb-bg);
  padding: 16px;
}

.help-container {
  width: 100%;
  max-width: 360px;
}

.help-title-wrap {
  text-align: center;
  margin-bottom: 20px;
}

.help-glitch {
  font-family: 'Press Start 2P', monospace;
  font-size: 28px;
  font-weight: 900;
  color: var(--gb-dark);
  text-shadow: 2px 2px 0 var(--gb-light);
}

.help-glitch-sub {
  font-size: 8px;
  letter-spacing: 3px;
  color: var(--gb-dark);
  font-family: monospace;
}

.help-grid {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 24px;
}

.help-card {
  background: var(--gb-dark);
  padding: 12px;
  border-radius: 4px;
  border: 2px solid var(--gb-light);
}

.help-card-header {
  font-family: 'Press Start 2P', monospace;
  font-size: 7px;
  color: var(--gb-light);
  margin-bottom: 10px;
  text-align: center;
}

.help-item {
  display: flex;
  gap: 12px;
  margin-bottom: 6px;
  font-size: 10px;
  color: var(--gb-light);
  font-family: monospace;
}

.key {
  min-width: 45px;
  color: var(--gb-white);
  font-family: 'Press Start 2P', monospace;
  font-size: 7px;
}

.badge-red, .badge-dark, .badge-gold {
  min-width: 20px;
  font-size: 12px;
}

.badge-red { color: #ff3366; }
.badge-dark { color: #ff6600; }
.badge-gold { color: #ffcc00; }

.pu-ico {
  min-width: 25px;
  font-size: 12px;
}

.launch-now {
  width: 100%;
  margin-top: 8px;
}

/* ========== GAME SCREEN ========== */
.game-screen {
  padding: 0;
}

.game-canvas {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
}

/* HUD - Gameboy Style */
.hud-new {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  z-index: 20;
  display: flex;
  justify-content: space-between;
  padding: 10px 16px;
  background: var(--gb-dark);
  border-bottom: 2px solid var(--gb-light);
  font-family: 'Press Start 2P', monospace;
}

.hud-score, .hud-level, .hud-lives {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.label {
  font-size: 6px;
  letter-spacing: 2px;
  color: var(--gb-light);
}

.value {
  font-size: 14px;
  font-weight: bold;
  color: var(--gb-white);
}

.lives-icons {
  display: flex;
  gap: 4px;
}

.life-icon {
  font-size: 12px;
  color: var(--gb-mid);
}

.life-icon.active {
  color: var(--gb-light);
}

/* Power-up bar */
.pu-bar-new {
  position: absolute;
  top: 55px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 6px;
  z-index: 20;
}

.pu-tag {
  font-family: 'Press Start 2P', monospace;
  font-size: 6px;
  padding: 3px 8px;
  background: var(--gb-dark);
  border: 1px solid var(--gb-light);
  color: var(--gb-light);
}

/* Combo flash */
.combo-flash-new {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
  font-family: 'Press Start 2P', monospace;
  font-size: 14px;
  font-weight: 900;
  pointer-events: none;
  z-index: 30;
  white-space: nowrap;
  color: var(--gb-white);
  text-shadow: 2px 2px 0 var(--gb-dark);
  animation: comboFloatNew 0.8s ease-out forwards;
}

@keyframes comboFloatNew {
  0% { opacity: 1; transform: translateX(-50%) translateY(0) scale(1); }
  100% { opacity: 0; transform: translateX(-50%) translateY(-40px) scale(1.2); }
}

/* Shoot hint */
.launch-hint-new {
  position: absolute;
  bottom: 130px;
  left: 50%;
  transform: translateX(-50%);
  font-family: 'Press Start 2P', monospace;
  font-size: 7px;
  color: var(--gb-white);
  background: var(--gb-dark);
  padding: 4px 8px;
  text-align: center;
  white-space: nowrap;
  z-index: 20;
  animation: hintBlink 1s step-end infinite;
  border: 1px solid var(--gb-light);
}

@keyframes hintBlink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}

.hint-arrow {
  font-size: 8px;
  letter-spacing: 4px;
}

/* Touch Controls with Pause below Shoot */
.touch-controls {
  position: absolute;
  bottom: 16px;
  left: 0;
  right: 0;
  z-index: 20;
  display: flex;
  justify-content: center;
  gap: 12px;
  padding: 0 12px;
}

.touch-btn {
  font-family: 'Press Start 2P', monospace;
  background: var(--gb-dark);
  border: 2px solid var(--gb-light);
  color: var(--gb-light);
  padding: 12px 16px;
  font-size: 16px;
  cursor: pointer;
  transition: all 0.05s linear;
  border-radius: 0px;
}

.touch-btn:active {
  transform: scale(0.95);
  background: var(--gb-mid);
}

.touch-btn.launch {
  padding: 8px 20px;
  font-size: 10px;
}

.touch-btn.pause-bottom {
  padding: 8px 16px;
  font-size: 9px;
  margin-top: 6px;
}

.center-controls {
  display: flex;
  flex-direction: column;
  gap: 6px;
  align-items: center;
}

/* ========== MODALS ========== */
.modal-overlay-new {
  position: absolute;
  inset: 0;
  z-index: 50;
  background: rgba(0,0,0,0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}

.modal-arcade {
  background: var(--gb-dark);
  border: 3px solid var(--gb-light);
  padding: 24px;
  text-align: center;
  max-width: 280px;
  width: 100%;
}

.modal-header-pause {
  font-family: 'Press Start 2P', monospace;
  font-size: 11px;
  color: var(--gb-light);
  margin-bottom: 20px;
}

.modal-stats {
  margin: 20px 0;
}

.stat-line {
  display: flex;
  justify-content: space-between;
  padding: 8px 0;
  border-bottom: 1px solid var(--gb-mid);
  font-size: 10px;
  color: var(--gb-light);
  font-family: monospace;
}

.stat-line span:last-child {
  color: var(--gb-white);
  font-weight: bold;
}

.modal-buttons {
  display: flex;
  gap: 12px;
  margin-top: 20px;
}

.modal-arcade-btn {
  flex: 1;
  padding: 10px;
  font-family: 'Press Start 2P', monospace;
  font-size: 8px;
  cursor: pointer;
  transition: all 0.1s;
  border: none;
}

.modal-arcade-btn.resume {
  background: var(--gb-mid);
  color: var(--gb-dark);
  border: 1px solid var(--gb-light);
}

.modal-arcade-btn.abort {
  background: transparent;
  color: var(--gb-light);
  border: 1px solid var(--gb-light);
}

.modal-arcade-btn:active {
  transform: scale(0.97);
}

.modal-result {
  background: var(--gb-dark);
  border: 3px solid var(--gb-light);
  padding: 28px 20px;
  text-align: center;
  max-width: 300px;
  width: 100%;
}

.result-icon {
  font-size: 40px;
  margin-bottom: 8px;
}

.result-title {
  font-family: 'Press Start 2P', monospace;
  font-size: 16px;
  font-weight: 900;
  margin-bottom: 4px;
  color: var(--gb-light);
}

.result-sub {
  font-size: 8px;
  color: var(--gb-mid);
  margin-bottom: 16px;
}

.result-score {
  font-family: 'Press Start 2P', monospace;
  font-size: 32px;
  font-weight: 900;
  color: var(--gb-white);
  margin: 12px 0;
}

.result-record {
  font-size: 8px;
  color: var(--gb-light);
  margin: 8px 0;
  animation: recordPulse 1s ease-in-out infinite;
}

@keyframes recordPulse {
  0%, 100% { transform: scale(1); letter-spacing: 1px; }
  50% { transform: scale(1.05); letter-spacing: 2px; }
}

.result-detail {
  display: flex;
  justify-content: center;
  gap: 20px;
  font-size: 8px;
  color: var(--gb-mid);
  margin: 16px 0;
  font-family: monospace;
}

/* Transitions */
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
.slide-enter-active, .slide-leave-active {
  transition: transform 0.35s ease, opacity 0.35s;
}
.slide-enter-from {
  transform: translateX(100%);
  opacity: 0;
}
.slide-leave-to {
  transform: translateX(-100%);
  opacity: 0;
}
</style>