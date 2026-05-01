<template>
  <div class="app-shell">
    <div class="scanlines" />

    <!-- ═══════════════ HOME SCREEN ═══════════════ -->
    <transition name="fade">
      <div v-if="screen === 'home'" class="screen home-screen">
        <div class="logo-wrap">
          <p class="logo-eyebrow">◈ ARCADE ◈</p>
          <h1 class="logo-title">NEON<br />Brick BREAKER</h1>
          <p class="logo-tag">DESTROY ALL BRICKS</p>
        </div>

        <div class="preview-stage">
          <div
            v-for="(b, i) in previewBricks"
            :key="i"
            class="preview-brick"
            :style="{
              left: b.x + 'px', top: b.y + 'px', width: b.w + 'px',
              background: b.color + '22', borderColor: b.color,
              boxShadow: '0 0 8px ' + b.color,
            }"
          />
          <div class="preview-ball" />
        </div>

        <div class="btn-stack">
          <button class="btn-primary" @click="startGame">▶ MULAI GAME</button>
          <button class="btn-secondary" @click="screen = 'help'">CARA BERMAIN</button>
        </div>

        <div v-if="highScore > 0" class="score-panel">
          <p class="score-panel-title">◈ HIGH SCORE ◈</p>
          <div class="score-row">
            <span>BEST SCORE</span>
            <span class="val-yellow">{{ highScore }}</span>
          </div>
          <div class="score-row" style="border:none">
            <span>BEST LEVEL</span>
            <span class="val-pink">{{ bestLevel }}</span>
          </div>
        </div>

        <p class="hint-text">← → ARROW KEY ATAU TOMBOL DI BAWAH</p>
      </div>
    </transition>

    <!-- ═══════════════ HELP SCREEN ═══════════════ -->
    <transition name="slide">
      <div v-if="screen === 'help'" class="screen help-screen">
        <div class="help-header">
          <button class="back-btn" @click="screen = 'home'">←</button>
          <h2 class="help-title">CARA BERMAIN</h2>
        </div>

        <div class="help-card">
          <p class="help-card-title">◈ KONTROL</p>
          <div class="help-item"><span class="em">⌨️</span><span>Tekan ← → untuk gerakkan paddle di keyboard</span></div>
          <div class="help-item"><span class="em">👇</span><span>Tombol ← → di layar untuk mobile</span></div>
          <div class="help-item"><span class="em">▶️</span><span>Tap LAUNCH atau SPACE untuk luncurkan bola</span></div>
          <div class="help-item"><span class="em">⏸️</span><span>Tombol pause kanan atas untuk jeda</span></div>
        </div>

        <div class="help-card">
          <p class="help-card-title">◈ CARA MAIN</p>
          <div class="help-item"><span class="em">🎯</span><span>Hancurkan semua bata untuk naik level</span></div>
          <div class="help-item"><span class="em">❤️</span><span>3 nyawa — jangan sampai bola jatuh!</span></div>
          <div class="help-item"><span class="em">🧱</span><span>Bata merah butuh 2 pukulan untuk hancur</span></div>
          <div class="help-item"><span class="em">💥</span><span>Combo memberi bonus poin!</span></div>
        </div>

        <div class="help-card">
          <p class="help-card-title">◈ POWER-UPS</p>
          <div class="pu-row"><div class="pu-dot green" /><span><b style="color:var(--g)">WIDE</b> — Paddle melebar 8 detik</span></div>
          <div class="pu-row"><div class="pu-dot orange" /><span><b style="color:var(--o)">MULTI</b> — Menjadi 3 bola</span></div>
          <div class="pu-row"><div class="pu-dot purple" /><span><b style="color:var(--v)">SLOW</b> — Bola melambat 5 detik</span></div>
        </div>

        <button class="btn-primary" style="width:100%;max-width:300px;margin-top:8px" @click="startGame">▶ MULAI SEKARANG</button>
        <div style="height:24px" />
      </div>
    </transition>

    <!-- ═══════════════ GAME SCREEN ═══════════════ -->
    <transition name="fade">
      <div v-if="screen === 'game'" class="screen game-screen">
        <canvas ref="canvas" class="game-canvas" />

        <!-- HUD - FIXED: Best Score sekarang sinkron dengan localStorage -->
        <div class="hud">
          <div class="hud-block">
            <span class="hud-label">SCORE</span>
            <span class="hud-val">{{ score }}</span>
          </div>
          <div class="hud-block hud-center">
            <span class="level-badge">LVL {{ level }}</span>
            <div class="lives-row">
              <div v-for="i in 3" :key="i" class="life-dot" :class="{ dead: i > lives }" />
            </div>
          </div>
          <div class="hud-block" style="align-items:flex-end">
            <button class="pause-btn" @click="togglePause">{{ isPaused ? '▶' : '⏸' }}</button>
          </div>

        </div>

        <!-- Active power-up chips - FIXED: timer display sekarang realtime -->
        <div class="pu-bar">
          <span v-if="puWideActive" class="pu-chip green">WIDE {{ puWideTimer.toFixed(0) }}s</span>
          <span v-if="puMulti" class="pu-chip orange">MULTI BALL</span>
          <span v-if="puSlowActive" class="pu-chip purple">SLOW {{ puSlowTimer.toFixed(0) }}s</span>
        </div>

        <!-- Combo flashes -->
        <div
          v-for="cf in comboFlashes"
          :key="cf.id"
          class="combo-flash"
          :style="{ top: cf.y + 'px', color: cf.color }"
        >{{ cf.text }}</div>

        <!-- Launch hint -->
        <transition name="fade">
          <div v-if="waitingLaunch && !isPaused" class="launch-hint">TAP LAUNCH ATAU TEKAN SPACE</div>
        </transition>

        <!-- ── MOBILE CONTROLS ── -->
        <div class="mobile-controls">
          <button
            class="ctrl-btn"
            @touchstart.prevent="startMove('left')"
            @touchend.prevent="stopMove"
            @mousedown="startMove('left')"
            @mouseup="stopMove"
          >◀</button>

          <button
            class="ctrl-btn launch-btn"
            @touchstart.prevent="launchBall"
            @mousedown="launchBall"
          >LAUNCH</button>

          <button
            class="ctrl-btn"
            @touchstart.prevent="startMove('right')"
            @touchend.prevent="stopMove"
            @mousedown="startMove('right')"
            @mouseup="stopMove"
          >▶</button>
        </div>
      </div>
    </transition>

    <!-- ═══════════════ PAUSE MODAL ═══════════════ -->
    <transition name="fade">
      <div v-if="screen === 'game' && isPaused && !showModal" class="modal-overlay">
        <div class="modal-box info">
          <span class="modal-icon">⏸</span>
          <h3 class="modal-title">PAUSED</h3>
          <p class="modal-sub">GAME DIJEDA</p>
          <div class="modal-row"><span>SCORE</span><span>{{ score }}</span></div>
          <div class="modal-row"><span>LEVEL</span><span>{{ level }}</span></div>
          <div class="modal-row" style="border:none"><span>LIVES</span><span>{{ lives }}</span></div>
          <div class="modal-btns">
            <button class="modal-btn primary" @click="togglePause">▶ LANJUTKAN</button>
            <button class="modal-btn secondary" @click="goHome">MENU UTAMA</button>
          </div>
        </div>
      </div>
    </transition>

    <!-- ═══════════════ RESULT MODAL ═══════════════ -->
    <transition name="fade">
      <div v-if="showModal" class="modal-overlay">
        <div class="modal-box" :class="modalClass">
          <span class="modal-icon">{{ modalIcon }}</span>
          <h3 class="modal-title">{{ modalTitle }}</h3>
          <p class="modal-sub">{{ modalSub }}</p>
          <div class="modal-score-big">{{ score }}</div>
          <div class="modal-row"><span>LEVEL DICAPAI</span><span>{{ level }}</span></div>
          <div v-if="isNewHigh" class="modal-row" style="border:none">
            <span>🎉 HIGH SCORE BARU!</span>
            <span class="val-yellow">{{ score }}</span>
          </div>
          <div class="modal-row" style="border:none"><span>BATA HANCUR</span><span>{{ totalDestroyed }}</span></div>
          <div class="modal-btns">
            <button class="modal-btn primary" @click="modalAction">
              {{ modalType === 'levelup' ? '⚡ LEVEL BERIKUTNYA' : '↺ MAIN LAGI' }}
            </button>
            <button class="modal-btn secondary" @click="goHome">MENU UTAMA</button>
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick } from 'vue'

// ─── CONSTANTS ───────────────────────────────────────────────────────────────
const COLS = 8
const ROWS = 6
const BRICK_PAD = 4
const BRICK_TOP = 72
const BRICK_H = 20
const COLORS = ['#00ffff', '#ff00aa', '#ffff00', '#00ff88', '#ff6600', '#aa00ff']

// ─── LEVEL GENERATOR ─────────────────────────────────────────────────────────
function generateLevel(lv) {
  const bricks = []
  const patterns = [
    (r, c) => ({ on: true, hp: 1, color: COLORS[r % COLORS.length] }),
    (r, c) => ({ on: (r + c) % 2 === 0, hp: 1, color: COLORS[(r + c) % COLORS.length] }),
    (r, c) => { const d = Math.abs(c - Math.floor(COLS / 2)) + r; return { on: d <= 5, hp: r < 2 ? 2 : 1, color: COLORS[r % COLORS.length] } },
    (r, c) => { const e = r === Math.floor(ROWS / 2) || c === Math.floor(COLS / 2); return { on: e, hp: 2, color: '#ff00aa' } },
    (r, c) => { const edge = r === 0 || r === ROWS - 1 || c === 0 || c === COLS - 1; const inner = r === 2 && c >= 2 && c <= COLS - 3; return { on: edge || inner, hp: edge ? 2 : 1, color: edge ? '#ffff00' : '#00ffff' } },
  ]
  const fn = patterns[Math.min(lv - 1, patterns.length - 1)]
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      const b = fn(r, c)
      if (b.on) bricks.push({ row: r, col: c, hp: b.hp, maxHp: b.hp, color: b.color, active: true, hasPU: Math.random() < 0.12, puType: ['wide', 'multi', 'slow'][Math.floor(Math.random() * 3)] })
    }
  }
  return bricks
}

// ─── REACTIVE STATE ──────────────────────────────────────────────────────────
const screen = ref('home')
const canvas = ref(null)
const score = ref(0)
const lives = ref(3)
const level = ref(1)
// FIX: HUD Best Score selalu sync dari localStorage
const highScore = ref(parseInt(localStorage.getItem('nb_hs') || '0'))
const bestLevel = ref(parseInt(localStorage.getItem('nb_bl') || '1'))
const isPaused = ref(false)
const waitingLaunch = ref(true)
const showModal = ref(false)
const modalType = ref('gameover')
const modalTitle = ref('')
const modalSub = ref('')
const isNewHigh = ref(false)
const totalDestroyed = ref(0)
const comboFlashes = ref([])

// FIX: Power-up states dengan proper timer management
const puWideActive = ref(false)
const puWideTimer = ref(0)
const puMulti = ref(false)
const puSlowActive = ref(false)
const puSlowTimer = ref(0)

// Preview home
const previewBricks = ref([])

// ─── COMPUTED ────────────────────────────────────────────────────────────────
const modalClass = computed(() => modalType.value === 'win' ? 'success' : modalType.value === 'levelup' ? 'info' : 'danger')
const modalIcon = computed(() => modalType.value === 'win' ? '🏆' : modalType.value === 'levelup' ? '⚡' : '💀')

// ─── GAME INTERNALS ──────────────────────────────────────────────────────────
let ctx = null, animId = null, lastTime = 0
let W = 0, H = 0, BRICK_W = 0
let balls = [], bricks = [], powerups = []
let combo = 0, flashId = 0
let lastTimestamp = 0

const paddle = { x: 0, y: 0, w: 0, h: 14, normalWidth: 0 }

// Keyboard + button state
const keys = { left: false, right: false }
const PADDLE_SPEED = 9

// Power-up timer references untuk cleanup
let wideTimerInterval = null
let slowTimerInterval = null

function buildPreview() {
  const vw = Math.min(window.innerWidth, 430)
  const bw = Math.floor(vw * 0.8 / 5) - 4
  const off = (vw - (5 * (bw + 4))) / 2
  previewBricks.value = Array.from({ length: 5 }, (_, i) => ({
    x: off + i * (bw + 4), y: 8, w: bw, color: COLORS[i],
  }))
}

function initCanvas() {
  const el = canvas.value
  const rect = el.parentElement.getBoundingClientRect()
  W = rect.width; H = rect.height
  el.width = W; el.height = H
  ctx = el.getContext('2d')
  BRICK_W = Math.floor((W - 20) / COLS) - BRICK_PAD
  paddle.normalWidth = W * 0.22
  paddle.w = puWideActive.value ? W * 0.40 : paddle.normalWidth
  paddle.x = W / 2 - paddle.w / 2
  paddle.y = H - 120
}

function makeBall(extra = false) {
  const spd = 5 + level.value * 0.45
  return {
    x: paddle.x + paddle.w / 2,
    y: paddle.y - 10,
    r: 8, vx: 0, vy: 0, speed: spd,
    attached: !extra,
    dead: false,
    ...(extra ? { vx: (Math.random() > 0.5 ? 1 : -1) * spd * 0.7, vy: -spd, attached: false } : {}),
  }
}

function loadLevel() {
  bricks = generateLevel(level.value)
  powerups = []; combo = 0
  balls = [makeBall()]
  waitingLaunch.value = true
}

// FIX: Reset power-ups dengan clean timer
function resetPowerups() {
  if (wideTimerInterval) clearInterval(wideTimerInterval)
  if (slowTimerInterval) clearInterval(slowTimerInterval)
  
  puWideActive.value = false
  puWideTimer.value = 0
  puSlowActive.value = false
  puSlowTimer.value = 0
  puMulti.value = false
  
  if (paddle.normalWidth) {
    paddle.w = paddle.normalWidth
  }
}

function startWideTimer(duration) {
  if (wideTimerInterval) clearInterval(wideTimerInterval)
  puWideActive.value = true
  puWideTimer.value = duration
  paddle.w = W * 0.40
  
  wideTimerInterval = setInterval(() => {
    if (puWideTimer.value > 0) {
      puWideTimer.value -= 0.1
      if (puWideTimer.value <= 0) {
        puWideActive.value = false
        paddle.w = paddle.normalWidth
        if (wideTimerInterval) clearInterval(wideTimerInterval)
        wideTimerInterval = null
      }
    }
  }, 100)
}

function startSlowTimer(duration) {
  if (slowTimerInterval) clearInterval(slowTimerInterval)
  puSlowActive.value = true
  puSlowTimer.value = duration
  
  slowTimerInterval = setInterval(() => {
    if (puSlowTimer.value > 0) {
      puSlowTimer.value -= 0.1
      if (puSlowTimer.value <= 0) {
        puSlowActive.value = false
        if (slowTimerInterval) clearInterval(slowTimerInterval)
        slowTimerInterval = null
      }
    }
  }, 100)
}

// ─── CONTROLS ────────────────────────────────────────────────────────────────
function startMove(dir) { keys[dir] = true }
function stopMove() { keys.left = false; keys.right = false }

function launchBall() {
  if (!waitingLaunch.value) return
  waitingLaunch.value = false
  balls.forEach(b => {
    if (!b.attached) return
    b.attached = false
    b.vx = (Math.random() * 0.5 - 0.25) * b.speed
    b.vy = -b.speed
  })
}

function onKeyDown(e) {
  if (screen.value !== 'game') return
  if (e.code === 'ArrowLeft') { e.preventDefault(); keys.left = true }
  if (e.code === 'ArrowRight') { e.preventDefault(); keys.right = true }
  if (e.code === 'Space') { e.preventDefault(); if (waitingLaunch.value) launchBall(); else togglePause() }
  if (e.code === 'Escape') togglePause()
}

function onKeyUp(e) {
  if (e.code === 'ArrowLeft') keys.left = false
  if (e.code === 'ArrowRight') keys.right = false
}

// ─── GAME LOOP ────────────────────────────────────────────────────────────────
function loop(ts) {
  if (!lastTimestamp) lastTimestamp = ts
  const dt = Math.min((ts - lastTimestamp) / 16.67, 3)
  lastTimestamp = ts
  if (!isPaused.value && screen.value === 'game' && !showModal.value) update(dt)
  draw()
  animId = requestAnimationFrame(loop)
}

function update(dt) {
  // Paddle movement
  const spd = PADDLE_SPEED * dt
  if (keys.left) paddle.x = Math.max(0, paddle.x - spd)
  if (keys.right) paddle.x = Math.min(W - paddle.w, paddle.x + spd)

  // Balls update
  balls.forEach(b => {
    if (b.attached) { b.x = paddle.x + paddle.w / 2; b.y = paddle.y - b.r; return }
    const slow = puSlowActive.value ? 0.5 : 1
    b.x += b.vx * dt * slow
    b.y += b.vy * dt * slow
    
    if (b.x - b.r < 0) { b.x = b.r; b.vx = Math.abs(b.vx) }
    if (b.x + b.r > W) { b.x = W - b.r; b.vx = -Math.abs(b.vx) }
    if (b.y - b.r < 0) { b.y = b.r; b.vy = Math.abs(b.vy) }
    
    // Paddle collision
    if (b.vy > 0 && b.y + b.r >= paddle.y && b.y + b.r <= paddle.y + paddle.h + 4 && b.x >= paddle.x && b.x <= paddle.x + paddle.w) {
      const hit = (b.x - (paddle.x + paddle.w / 2)) / (paddle.w / 2)
      const angle = hit * 65 * Math.PI / 180
      b.vx = Math.sin(angle) * b.speed
      b.vy = -Math.abs(Math.cos(angle) * b.speed)
      b.y = paddle.y - b.r
      combo = 0
    }
    
    if (b.y - b.r > H) b.dead = true
  })

  const alive = balls.filter(b => !b.dead)
  if (alive.length < balls.length && alive.length === 0) {
    lives.value--; combo = 0; balls = []
    if (lives.value <= 0) { gameOver(); return }
    balls = [makeBall()]; waitingLaunch.value = true; return
  }
  balls = alive

  // Bricks collision
  bricks.forEach(bk => {
    if (!bk.active) return
    const bx = 10 + bk.col * (BRICK_W + BRICK_PAD)
    const by = BRICK_TOP + bk.row * (BRICK_H + BRICK_PAD)
    balls.forEach(b => {
      if (b.attached) return
      if (b.x + b.r < bx || b.x - b.r > bx + BRICK_W || b.y + b.r < by || b.y - b.r > by + BRICK_H) return
      bk.hp--; combo++
      if (bk.hp <= 0) {
        bk.active = false; totalDestroyed.value++
        const pts = bk.color === '#ffff00' ? 50 : bk.maxHp > 1 ? 20 : 10
        score.value += pts * Math.max(1, Math.floor(combo / 3))
        if (bk.hasPU) powerups.push({ x: bx + BRICK_W / 2, y: by, type: bk.puType, dead: false })
        if (combo >= 3) addComboFlash(bx + BRICK_W / 2, by, combo)
      }
      const ox = Math.abs(b.x - (bx + BRICK_W / 2)) / (BRICK_W / 2)
      const oy = Math.abs(b.y - (by + BRICK_H / 2)) / (BRICK_H / 2)
      if (ox > oy) b.vx = -b.vx; else b.vy = -b.vy
    })
  })

  if (bricks.every(bk => !bk.active)) { nextLevel(); return }

  // Power-up drops
  powerups.forEach(pu => {
    pu.y += 2.5 * dt
    if (pu.y >= paddle.y && pu.y <= paddle.y + paddle.h && pu.x >= paddle.x && pu.x <= paddle.x + paddle.w) { 
      collectPU(pu.type); pu.dead = true 
    }
    if (pu.y > H) pu.dead = true
  })
  powerups = powerups.filter(p => !p.dead)
}

function collectPU(type) {
  if (type === 'wide') { startWideTimer(8) }
  else if (type === 'multi') { 
    puMulti.value = true; 
    balls.push(makeBall(true)); 
    balls.push(makeBall(true)); 
    setTimeout(() => { puMulti.value = false }, 200) 
  }
  else if (type === 'slow') { startSlowTimer(5) }
}

function addComboFlash(x, y, c) {
  const id = flashId++
  comboFlashes.value.push({ id, y: y + 30, text: `x${c} COMBO!`, color: COLORS[c % COLORS.length] })
  setTimeout(() => { comboFlashes.value = comboFlashes.value.filter(f => f.id !== id) }, 1000)
}

// ─── GAME EVENTS ─────────────────────────────────────────────────────────────
function saveScore() {
  isNewHigh.value = score.value > highScore.value
  if (isNewHigh.value) { 
    highScore.value = score.value
    localStorage.setItem('nb_hs', score.value)
  }
  if (level.value > bestLevel.value) { 
    bestLevel.value = level.value
    localStorage.setItem('nb_bl', level.value)
  }
}

function nextLevel() {
  saveScore()
  if (level.value >= 5) { 
    modalType.value = 'win'
    modalTitle.value = 'VICTORY!'
    modalSub.value = 'SEMUA LEVEL SELESAI'
  } else { 
    modalType.value = 'levelup'
    modalTitle.value = `LEVEL ${level.value} CLEAR!`
    modalSub.value = 'LANJUT KE LEVEL BERIKUTNYA'
  }
  showModal.value = true
}

function gameOver() {
  saveScore()
  modalType.value = 'gameover'
  modalTitle.value = 'GAME OVER'
  modalSub.value = 'NYAWA HABIS'
  showModal.value = true
}

function modalAction() {
  showModal.value = false
  if (modalType.value === 'levelup') {
    level.value++
    resetPowerups()
    if (!puWideActive.value && paddle.normalWidth) paddle.w = paddle.normalWidth
    loadLevel()
  } else {
    score.value = 0; lives.value = 3; level.value = 1; totalDestroyed.value = 0
    resetPowerups()
    loadLevel()
  }
  isPaused.value = false
}

function startGame() {
  screen.value = 'game'
  score.value = 0; lives.value = 3; level.value = 1; totalDestroyed.value = 0
  showModal.value = false; isPaused.value = false
  resetPowerups()
  comboFlashes.value = []
  nextTick(() => { 
    initCanvas()
    loadLevel()
    if (animId) cancelAnimationFrame(animId)
    lastTimestamp = 0
    animId = requestAnimationFrame(loop)
  })
}

function goHome() {
  if (animId) cancelAnimationFrame(animId)
  resetPowerups()
  showModal.value = false
  isPaused.value = false
  screen.value = 'home'
}

function togglePause() { if (!showModal.value) isPaused.value = !isPaused.value }

// ─── DRAW ─────────────────────────────────────────────────────────────────────
function draw() {
  if (!ctx) return
  ctx.clearRect(0, 0, W, H)

  // Background
  const bg = ctx.createRadialGradient(W / 2, H / 2, 0, W / 2, H / 2, H * 0.7)
  bg.addColorStop(0, 'rgba(0,40,60,0.25)'); bg.addColorStop(1, 'transparent')
  ctx.fillStyle = bg; ctx.fillRect(0, 0, W, H)

  // Bricks
  bricks.forEach(bk => {
    if (!bk.active) return
    const bx = 10 + bk.col * (BRICK_W + BRICK_PAD)
    const by = BRICK_TOP + bk.row * (BRICK_H + BRICK_PAD)
    ctx.save()
    ctx.shadowColor = bk.color; ctx.shadowBlur = 10
    ctx.fillStyle = bk.color + '22'
    ctx.beginPath(); ctx.roundRect(bx, by, BRICK_W, BRICK_H, 3); ctx.fill()
    ctx.strokeStyle = bk.color; ctx.lineWidth = 1.5
    ctx.stroke()
    if (bk.maxHp > 1 && bk.hp < bk.maxHp) {
      ctx.fillStyle = bk.color; ctx.globalAlpha = 0.35
      ctx.fillRect(bx + 4, by + BRICK_H - 4, (BRICK_W - 8) * (bk.hp / bk.maxHp), 2)
    }
    if (bk.hasPU) {
      const pc = { wide: '#00ff88', multi: '#ff6600', slow: '#aa00ff' }[bk.puType]
      ctx.globalAlpha = 0.85; ctx.fillStyle = pc
      ctx.beginPath(); ctx.arc(bx + BRICK_W - 8, by + BRICK_H / 2, 3, 0, Math.PI * 2); ctx.fill()
    }
    ctx.restore()
  })

  // Power-ups
  powerups.forEach(pu => {
    const pc = { wide: '#00ff88', multi: '#ff6600', slow: '#aa00ff' }[pu.type]
    ctx.save(); ctx.shadowColor = pc; ctx.shadowBlur = 14
    ctx.fillStyle = pc; ctx.beginPath(); ctx.arc(pu.x, pu.y, 7, 0, Math.PI * 2); ctx.fill()
    ctx.fillStyle = '#000'; ctx.font = 'bold 7px monospace'; ctx.textAlign = 'center'; ctx.textBaseline = 'middle'
    ctx.fillText({ wide: 'W', multi: 'M', slow: 'S' }[pu.type], pu.x, pu.y)
    ctx.restore()
  })

  // Paddle
  ctx.save()
  ctx.shadowColor = '#00ffff'; ctx.shadowBlur = 22
  const pg = ctx.createLinearGradient(paddle.x, 0, paddle.x + paddle.w, 0)
  pg.addColorStop(0, '#00aaff'); pg.addColorStop(0.5, '#00ffff'); pg.addColorStop(1, '#00aaff')
  ctx.fillStyle = pg; ctx.beginPath(); ctx.roundRect(paddle.x, paddle.y, paddle.w, paddle.h, 7); ctx.fill()
  ctx.fillStyle = 'rgba(255,255,255,0.28)'; ctx.beginPath(); ctx.roundRect(paddle.x + 6, paddle.y + 2, paddle.w - 12, 3, 2); ctx.fill()
  ctx.restore()

  // Balls
  balls.forEach(b => {
    ctx.save()
    ctx.shadowColor = '#00ffff'; ctx.shadowBlur = 22
    const g = ctx.createRadialGradient(b.x - b.r * 0.3, b.y - b.r * 0.3, 0, b.x, b.y, b.r)
    g.addColorStop(0, '#fff'); g.addColorStop(0.4, '#00ffff'); g.addColorStop(1, '#0066ff')
    ctx.fillStyle = g; ctx.beginPath(); ctx.arc(b.x, b.y, b.r, 0, Math.PI * 2); ctx.fill()
    ctx.globalAlpha = 0.18; ctx.fillStyle = '#00ffff'
    ctx.beginPath(); ctx.arc(b.x - b.vx * 2, b.y - b.vy * 2, b.r * 0.65, 0, Math.PI * 2); ctx.fill()
    ctx.restore()
  })
}

// roundRect helper
if (!CanvasRenderingContext2D.prototype.roundRect) {
  CanvasRenderingContext2D.prototype.roundRect = function(x, y, w, h, r) {
    if (w < 2 * r) r = w / 2;
    if (h < 2 * r) r = h / 2;
    this.moveTo(x+r, y);
    this.lineTo(x+w-r, y);
    this.quadraticCurveTo(x+w, y, x+w, y+r);
    this.lineTo(x+w, y+h-r);
    this.quadraticCurveTo(x+w, y+h, x+w-r, y+h);
    this.lineTo(x+r, y+h);
    this.quadraticCurveTo(x, y+h, x, y+h-r);
    this.lineTo(x, y+r);
    this.quadraticCurveTo(x, y, x+r, y);
    return this;
  }
}

// ─── LIFECYCLE ────────────────────────────────────────────────────────────────
onMounted(() => {
  buildPreview()
  window.addEventListener('keydown', onKeyDown)
  window.addEventListener('keyup', onKeyUp)
  window.addEventListener('mouseup', stopMove)
  window.addEventListener('resize', buildPreview)
})

onUnmounted(() => {
  if (animId) cancelAnimationFrame(animId)
  if (wideTimerInterval) clearInterval(wideTimerInterval)
  if (slowTimerInterval) clearInterval(slowTimerInterval)
  window.removeEventListener('keydown', onKeyDown)
  window.removeEventListener('keyup', onKeyUp)
  window.removeEventListener('mouseup', stopMove)
  window.removeEventListener('resize', buildPreview)
})
</script>

<style scoped>
/* (Sisakan style sama seperti kode asli Anda, tidak diubah) */
/* ─── CSS VARIABLES ─── */
.app-shell {
  --c: #00ffff;
  --p: #ff00aa;
  --y: #ffff00;
  --g: #00ff88;
  --o: #ff6600;
  --v: #aa00ff;
  --dark: #050510;
  --panel: #0a0a1f;
  --grid: rgba(0,255,255,0.04);

  position: relative;
  width: 100%;
  max-width: 430px;
  height: 100dvh;
  margin: 0 auto;
  background: var(--dark);
  overflow: hidden;
  font-family: 'Share Tech Mono', 'Courier New', monospace;
  color: var(--c);
}

.app-shell::before {
  content: '';
  position: absolute; inset: 0;
  background-image: linear-gradient(var(--grid) 1px, transparent 1px), linear-gradient(90deg, var(--grid) 1px, transparent 1px);
  background-size: 30px 30px;
  pointer-events: none; z-index: 0;
}

.scanlines {
  position: absolute; inset: 0; z-index: 5; pointer-events: none;
  background: repeating-linear-gradient(0deg, transparent, transparent 2px, rgba(0,0,0,0.025) 2px, rgba(0,0,0,0.025) 4px);
}

.screen {
  position: absolute; inset: 0; z-index: 10;
  display: flex; flex-direction: column; align-items: center;
  overflow-y: auto;
}

.home-screen { justify-content: center; padding: 24px 20px; gap: 0; }
.logo-wrap { text-align: center; margin-bottom: 24px; }
.logo-eyebrow { font-size: 10px; letter-spacing: 6px; color: var(--p); text-shadow: 0 0 10px var(--p); margin-bottom: 6px; }
.logo-title {
  font-family: 'Orbitron', 'Courier New', monospace;
  font-size: 44px; font-weight: 900; line-height: 1;
  background: linear-gradient(135deg, var(--c), var(--p));
  -webkit-background-clip: text; -webkit-text-fill-color: transparent; background-clip: text;
  filter: drop-shadow(0 0 18px rgba(0,255,255,0.5));
  animation: pulseTitle 3s ease-in-out infinite;
}
.logo-tag { font-size: 10px; letter-spacing: 3px; color: rgba(0,255,255,0.4); margin-top: 6px; }

@keyframes pulseTitle {
  0%,100% { filter: drop-shadow(0 0 18px rgba(0,255,255,0.5)); }
  50%      { filter: drop-shadow(0 0 30px rgba(255,0,170,0.7)); }
}

.preview-stage { position: relative; width: 100%; height: 100px; margin-bottom: 20px; overflow: hidden; }
.preview-brick { position: absolute; height: 14px; border-radius: 3px; border: 1px solid; }
.preview-ball {
  position: absolute; width: 14px; height: 14px; border-radius: 50%;
  background: var(--c); box-shadow: 0 0 14px var(--c), 0 0 28px var(--c);
  bottom: 10px; left: 50%; transform: translateX(-50%);
  animation: floatBall 2s ease-in-out infinite;
}
@keyframes floatBall { 0%,100% { bottom: 10px; } 50% { bottom: 40px; } }

.btn-stack { display: flex; flex-direction: column; align-items: center; gap: 12px; width: 100%; }

.btn-primary {
  font-family: 'Orbitron', monospace; font-size: 15px; font-weight: 700; letter-spacing: 3px;
  color: var(--dark); background: linear-gradient(135deg, var(--c), #00aaff);
  border: none; border-radius: 6px; padding: 16px 0; cursor: pointer;
  box-shadow: 0 0 18px rgba(0,255,255,0.35);
  width: 100%; max-width: 300px; transition: transform .15s, box-shadow .15s;
  position: relative; overflow: hidden;
}
.btn-primary:active { transform: scale(0.97); box-shadow: 0 0 8px rgba(0,255,255,0.2); }

.btn-secondary {
  font-family: 'Orbitron', monospace; font-size: 12px; font-weight: 700; letter-spacing: 2px;
  color: var(--p); background: transparent; border: 1px solid var(--p);
  border-radius: 6px; padding: 12px 0; cursor: pointer;
  box-shadow: 0 0 8px rgba(255,0,170,0.15);
  width: 100%; max-width: 300px; transition: all .15s;
}
.btn-secondary:active { background: rgba(255,0,170,0.08); transform: scale(0.97); }

.score-panel { background: rgba(0,255,255,0.03); border: 1px solid rgba(0,255,255,0.1); border-radius: 8px; padding: 14px 16px; width: 100%; max-width: 300px; margin-top: 20px; }
.score-panel-title { font-family: 'Orbitron', monospace; font-size: 9px; letter-spacing: 3px; color: var(--y); text-align: center; margin-bottom: 10px; }
.score-row { display: flex; justify-content: space-between; font-size: 12px; padding: 5px 0; border-bottom: 1px solid rgba(0,255,255,0.06); color: rgba(0,255,255,0.6); }
.val-yellow { color: var(--y); font-family: 'Orbitron', monospace; }
.val-pink   { color: var(--p); font-family: 'Orbitron', monospace; }

.hint-text { font-size: 9px; letter-spacing: 2px; color: rgba(0,255,255,0.2); margin-top: 16px; text-align: center; }
.help-screen { padding: 16px 16px 24px; gap: 12px; justify-content: flex-start; }
.help-header { display: flex; align-items: center; gap: 12px; width: 100%; margin-top: 8px; }
.back-btn { background: transparent; border: none; color: var(--c); font-size: 22px; cursor: pointer; padding: 0; }
.help-title { font-family: 'Orbitron', monospace; font-size: 17px; font-weight: 900; letter-spacing: 3px; }
.help-card { background: rgba(0,255,255,0.03); border: 1px solid rgba(0,255,255,0.1); border-radius: 8px; padding: 14px; width: 100%; }
.help-card-title { font-family: 'Orbitron', monospace; font-size: 9px; letter-spacing: 3px; color: var(--p); margin-bottom: 10px; }
.help-item { display: flex; gap: 8px; margin-bottom: 6px; font-size: 12px; color: rgba(0,255,255,0.7); line-height: 1.5; }
.em { font-size: 16px; flex-shrink: 0; }
.pu-row { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; font-size: 12px; color: rgba(0,255,255,0.7); }
.pu-dot { width: 14px; height: 14px; border-radius: 50%; flex-shrink: 0; }
.pu-dot.green  { background: var(--g); box-shadow: 0 0 6px var(--g); }
.pu-dot.orange { background: var(--o); box-shadow: 0 0 6px var(--o); }
.pu-dot.purple { background: var(--v); box-shadow: 0 0 6px var(--v); }
.game-screen { padding: 0; }
.game-canvas { position: absolute; inset: 0; width: 100%; height: 100%; }
.hud {
  position: absolute; top: 0; left: 0; right: 0; z-index: 20;
  padding: 10px 14px 6px;
  background: linear-gradient(to bottom, rgba(5,5,16,0.92), transparent);
  display: flex; justify-content: space-between; align-items: flex-start;
}
.hud-block { display: flex; flex-direction: column; }
.hud-center { align-items: center; }
.hud-label { font-size: 8px; letter-spacing: 2px; color: rgba(0,255,255,0.35); }
.hud-val { font-family: 'Orbitron', monospace; font-size: 18px; font-weight: 700; color: var(--c); text-shadow: 0 0 10px var(--c); line-height: 1.1; }
.level-badge { font-family: 'Orbitron', monospace; font-size: 10px; padding: 2px 10px; border: 1px solid var(--y); border-radius: 20px; color: var(--y); text-shadow: 0 0 8px var(--y); }
.lives-row { display: flex; gap: 4px; margin-top: 4px; }
.life-dot { width: 8px; height: 8px; border-radius: 50%; background: var(--p); box-shadow: 0 0 6px var(--p); transition: all .3s; }
.life-dot.dead { background: rgba(255,0,170,0.1); box-shadow: none; }
.pause-btn {
  position: absolute; top: 10px; right: 14px; z-index: 25;
  background: transparent; border: 1px solid rgba(0,255,255,0.25); border-radius: 6px;
  color: var(--c); font-size: 14px; width: 32px; height: 32px; cursor: pointer; display: flex; align-items: center; justify-content: center;
}
.pu-bar { position: absolute; top: 56px; left: 50%; transform: translateX(-50%); display: flex; gap: 6px; z-index: 20; }
.pu-chip {
  font-family: 'Orbitron', monospace; font-size: 9px; padding: 3px 8px;
  border-radius: 20px; letter-spacing: 1px; animation: chipPulse 1s ease-in-out infinite;
}
.pu-chip.green  { background: rgba(0,255,136,0.12); border: 1px solid var(--g); color: var(--g); }
.pu-chip.orange { background: rgba(255,102,0,0.12); border: 1px solid var(--o); color: var(--o); }
.pu-chip.purple { background: rgba(170,0,255,0.12); border: 1px solid var(--v); color: var(--v); }
@keyframes chipPulse { 0%,100% { opacity:1; } 50% { opacity:.55; } }
.combo-flash {
  position: absolute; left: 50%; transform: translateX(-50%);
  font-family: 'Orbitron', monospace; font-size: 18px; font-weight: 900;
  pointer-events: none; z-index: 30; white-space: nowrap;
  text-shadow: 0 0 16px currentColor;
  animation: comboFloat 1s ease-out forwards;
}
@keyframes comboFloat {
  0%   { opacity: 1; transform: translateX(-50%) translateY(0) scale(1); }
  100% { opacity: 0; transform: translateX(-50%) translateY(-55px) scale(1.35); }
}
.launch-hint {
  position: absolute; bottom: 130px; left: 50%; transform: translateX(-50%);
  font-size: 9px; letter-spacing: 2px; color: rgba(0,255,255,0.45);
  white-space: nowrap; z-index: 20;
  animation: chipPulse 1.2s infinite;
}
.mobile-controls {
  position: absolute; bottom: 14px; left: 0; right: 0; z-index: 20;
  display: flex; justify-content: center; align-items: center; gap: 12px;
  padding: 0 20px;
}
.ctrl-btn {
  font-family: 'Orbitron', monospace; font-size: 18px; font-weight: 700;
  background: rgba(0,255,255,0.07); border: 1px solid rgba(0,255,255,0.25);
  border-radius: 10px; color: var(--c); cursor: pointer;
  width: 72px; height: 56px; display: flex; align-items: center; justify-content: center;
  box-shadow: 0 0 10px rgba(0,255,255,0.1);
  transition: background .1s, transform .1s;
  -webkit-tap-highlight-color: transparent;
  user-select: none;
}
.ctrl-btn:active { background: rgba(0,255,255,0.18); transform: scale(0.94); }
.launch-btn {
  font-size: 10px; letter-spacing: 1px; width: 100px;
  background: rgba(0,255,255,0.1); border-color: var(--c);
  color: var(--c); box-shadow: 0 0 12px rgba(0,255,255,0.2);
}
.modal-overlay {
  position: absolute; inset: 0; z-index: 50;
  background: rgba(5,5,16,0.86); backdrop-filter: blur(4px);
  display: flex; align-items: center; justify-content: center; padding: 24px;
}
.modal-box {
  background: var(--panel); border-radius: 12px; padding: 28px 22px;
  width: 100%; max-width: 310px; text-align: center;
  position: relative; overflow: hidden; border: 1px solid;
}
.modal-box::before { content: ''; position: absolute; top: 0; left: 0; right: 0; height: 2px; background: linear-gradient(90deg, transparent, currentColor, transparent); }
.modal-box.danger  { border-color: var(--p); color: var(--p); }
.modal-box.success { border-color: var(--g); color: var(--g); }
.modal-box.info    { border-color: var(--c); color: var(--c); }
.modal-icon   { font-size: 44px; display: block; margin-bottom: 10px; }
.modal-title  { font-family: 'Orbitron', monospace; font-size: 20px; font-weight: 900; letter-spacing: 3px; margin-bottom: 4px; }
.modal-sub    { font-size: 11px; opacity: .55; margin-bottom: 14px; letter-spacing: 1px; }
.modal-score-big { font-family: 'Orbitron', monospace; font-size: 34px; font-weight: 900; margin: 8px 0 14px; text-shadow: 0 0 18px currentColor; }
.modal-row    { display: flex; justify-content: space-between; font-size: 12px; padding: 5px 0; border-bottom: 1px solid rgba(255,255,255,0.05); color: rgba(255,255,255,0.55); }
.modal-row span:last-child { color: #fff; }
.modal-btns   { display: flex; flex-direction: column; gap: 10px; margin-top: 18px; }
.modal-btn    { font-family: 'Orbitron', monospace; font-size: 12px; font-weight: 700; letter-spacing: 2px; border: none; border-radius: 6px; padding: 13px; cursor: pointer; transition: transform .15s; }
.modal-btn:active { transform: scale(0.97); }
.modal-btn.primary   { background: linear-gradient(135deg, var(--c), #00aaff); color: var(--dark); box-shadow: 0 0 14px rgba(0,255,255,0.3); }
.modal-btn.secondary { background: transparent; border: 1px solid rgba(255,255,255,0.15); color: rgba(255,255,255,0.5); }
.fade-enter-active, .fade-leave-active { transition: opacity .3s; }
.fade-enter-from, .fade-leave-to       { opacity: 0; }
.slide-enter-active, .slide-leave-active { transition: transform .35s ease, opacity .35s; }
.slide-enter-from { transform: translateX(100%); opacity: 0; }
.slide-leave-to   { transform: translateX(-100%); opacity: 0; }
</style>