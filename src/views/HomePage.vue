<template>
  <ion-page class="app-page">
    <ion-content :fullscreen="true" :scroll-events="true">

      <!-- Ambient blobs -->
      <div class="blob blob-1"></div>
      <div class="blob blob-2"></div>
      <div class="blob blob-3"></div>

      <!-- Header -->
      <div class="glass-header">
        <div class="brand">
          <span class="brand-name">My<em>RodaPutar</em></span>
        </div>
        <div class="header-avatar" @click="openSettings">
          <ion-icon :icon="settingsOutline" style="font-size: 18px;"></ion-icon>
        </div>
      </div>

      <!-- Hero Glass Card -->
      <div class="hero-card glass-card">
        <div class="hero-count-row">
          <span class="hero-number">{{ segments.length }}</span>
          <div class="hero-text-group">
            <span class="hero-big-label">Pilihan</span>
            <span class="hero-sub-label">tersimpan</span>
          </div>
        </div>
        <div class="stats-row text-white">
          <div class="stat-pill" @click="addSegment">
            <ion-icon :icon="addCircleOutline" style="font-size: 20px; margin-bottom: 4px;"></ion-icon>
            <span class="stat-label">Tambah</span>
          </div>
          <div class="stat-pill" @click="resetSegments">
            <ion-icon :icon="refreshOutline" style="font-size: 20px; margin-bottom: 4px;"></ion-icon>
            <span class="stat-label">Reset</span>
          </div>
          <div class="stat-pill accent-pill" @click="randomizeColors">
            <ion-icon :icon="colorPaletteOutline" style="font-size: 20px; margin-bottom: 4px;"></ion-icon>
            <span class="stat-label accent-label">Warna</span>
          </div>
        </div>
      </div>

      <!-- Wheel Canvas -->
      <div class="wheel-container">
        <canvas ref="wheelCanvas" width="500" height="500" class="wheel-canvas"></canvas>
        <div class="pointer" @click="spinWheel">
          <div class="pointer-inner">
            <ion-icon :icon="playOutline" style="font-size: 28px;"></ion-icon>
          </div>
        </div>
      </div>

      <!-- Result Section -->
      <div class="result-card glass-card" v-if="lastResult">
        <div class="result-label">Hasil Putaran</div>
        <div class="result-value">{{ lastResult }}</div>
      </div>
      <div class="result-card glass-card placeholder-card" v-else>
        <div class="result-label">Tekan tombol tengah</div>
        <div class="result-value">—</div>
      </div>

      <!-- Segment List -->
      <div class="section-row">
        <span class="section-title">Daftar Pilihan</span>
        <button class="add-pill" @click="addSegment">
          <ion-icon :icon="addOutline"></ion-icon>
          Tambah
        </button>
      </div>

      <!-- Segment Items -->
      <div class="segment-scroll" v-if="segments.length > 0">
        <ion-item-sliding
          v-for="(segment, idx) in segments"
          :key="idx"
          class="sliding-wrap"
        >
          <ion-item-options side="end">
            <ion-item-option color="danger" @click="deleteSegment(idx)">
              <ion-icon :icon="trashOutline" slot="icon-only"></ion-icon>
            </ion-item-option>
          </ion-item-options>
          <ion-item-options side="start">
            <ion-item-option class="edit-swipe" @click="editSegment(idx)">
              <ion-icon :icon="createOutline" slot="icon-only"></ion-icon>
            </ion-item-option>
          </ion-item-options>

          <ion-item lines="none" class="segment-item-wrap">
            <div class="segment-card glass-card">
              <div class="segment-color" :style="{ backgroundColor: segment.color }"></div>
              <div class="segment-info">
                <div class="segment-name">{{ segment.label }}</div>
                <div class="segment-weight">Bobot: {{ segment.weight }}</div>
              </div>
              <div class="segment-arrow" @click="editSegment(idx)">✎</div>
            </div>
          </ion-item>
        </ion-item-sliding>
      </div>

      <!-- Empty State -->
      <div v-else class="empty-state">
        <div class="empty-blob"></div>
        <div class="empty-icon">🎡</div>
        <p class="empty-title">Belum ada pilihan</p>
        <p class="empty-sub">Tambahkan pilihan untuk mulai memutar roda</p>
        <button class="empty-cta" @click="addSegment">+ Tambah Pilihan</button>
      </div>

      <!-- Bottom spacer -->
      <div style="height: 40px;"></div>

    </ion-content>

    <!-- ── Add/Edit Segment Modal ── -->
    <ion-modal
      :is-open="isModalOpen"
      @didDismiss="closeModal"
      :breakpoints="[0, 0.6]"
      :initial-breakpoint="0.6"
      class="glass-modal"
    >
      <ion-header class="modal-header">
        <ion-toolbar class="modal-toolbar">
          <ion-title class="modal-title">{{ editingIndex !== null ? 'Edit Pilihan' : 'Pilihan Baru' }}</ion-title>
          <ion-buttons slot="end">
            <ion-button @click="closeModal" class="close-btn">
              <ion-icon :icon="closeOutline" slot="icon-only"></ion-icon>
            </ion-button>
          </ion-buttons>
        </ion-toolbar>
      </ion-header>
      <ion-content class="modal-body">
        <div class="form-group">
          <label class="form-label">Nama Pilihan <span class="req">*</span></label>
          <ion-input class="glass-input" v-model="formData.label" placeholder="Contoh: Makan Malam, Hadiah, Challenge" clearInput></ion-input>
        </div>

        <div class="form-group">
          <label class="form-label">Bobot / Peluang <span class="opt">— semakin besar, semakin sering muncul</span></label>
          <ion-input class="glass-input" type="number" v-model="formData.weight" placeholder="1"></ion-input>
        </div>

        <div class="form-group">
          <label class="form-label">Warna Segment</label>
          <div class="color-options">
            <button
              v-for="color in colorOptions"
              :key="color"
              class="color-opt"
              :style="{ backgroundColor: color }"
              :class="{ selected: formData.color === color }"
              @click="formData.color = color"
            ></button>
            <button class="color-opt random-color" @click="randomColor" :style="{ background: 'linear-gradient(135deg, #a78bfa, #ec4899)' }">🎲</button>
          </div>
        </div>

        <div class="modal-footer">
          <ion-button expand="block" fill="clear" class="btn-cancel" @click="closeModal">Batal</ion-button>
          <ion-button expand="block" class="btn-save" :disabled="!formData.label.trim()" @click="saveSegment">
            {{ editingIndex !== null ? 'Simpan' : 'Tambah' }}
          </ion-button>
        </div>
      </ion-content>
    </ion-modal>

    <!-- ── Settings Modal ── -->
    <ion-modal
      :is-open="isSettingsOpen"
      @didDismiss="closeSettings"
      :breakpoints="[0, 0.5]"
      :initial-breakpoint="0.5"
      class="glass-modal"
    >
      <ion-header class="modal-header">
        <ion-toolbar class="modal-toolbar">
          <ion-title class="modal-title">Pengaturan</ion-title>
          <ion-buttons slot="end">
            <ion-button @click="closeSettings" class="close-btn">
              <ion-icon :icon="closeOutline" slot="icon-only"></ion-icon>
            </ion-button>
          </ion-buttons>
        </ion-toolbar>
      </ion-header>
      <ion-content class="modal-body">
        <div class="form-group">
          <label class="form-label">Durasi Animasi (ms)</label>
          <ion-input class="glass-input" type="number" v-model="settings.duration" placeholder="3000"></ion-input>
        </div>

        <div class="form-group">
          <label class="form-label">Putaran Minimal</label>
          <ion-input class="glass-input" type="number" v-model="settings.minSpins" placeholder="5"></ion-input>
        </div>

        <div class="form-group">
          <label class="form-label">Efek Suara</label>
          <div class="toggle-switch" @click="settings.sound = !settings.sound">
            <span>{{ settings.sound ? '🔊 ON' : '🔇 OFF' }}</span>
          </div>
        </div>

        <div class="modal-footer single">
          <ion-button expand="block" class="btn-save" @click="saveSettings">Simpan Pengaturan</ion-button>
        </div>
      </ion-content>
    </ion-modal>

    <!-- Delete Confirmation Alert -->
    <ion-alert
      :is-open="deleteTargetIndex !== null"
      header="Hapus Pilihan?"
      message="Pilihan ini akan dihapus dari roda."
      :buttons="deleteAlertButtons"
      @didDismiss="deleteTargetIndex = null"
    ></ion-alert>
  </ion-page>
</template>

<script setup lang="ts">
import { ref, onMounted, watch, nextTick } from 'vue';
import {
  IonPage, IonContent, IonButtons, IonButton, IonIcon,
  IonItem, IonItemSliding, IonItemOptions, IonItemOption,
  IonModal, IonHeader, IonToolbar, IonTitle,
  IonInput, IonAlert,
} from '@ionic/vue';
import {
  addOutline, createOutline, trashOutline, closeOutline,
  playOutline, refreshOutline, addCircleOutline,
  colorPaletteOutline, settingsOutline,
} from 'ionicons/icons';

interface Segment {
  label: string;
  weight: number;
  color: string;
}

const colorOptions = [
  '#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#FFEAA7',
  '#DDA0DD', '#98D8C8', '#F7D794', '#786FA6', '#F19066',
  '#3B9AE1', '#E15A5A', '#5D9B9B', '#FFB347', '#B5EAD7'
];

const defaultSegments: Segment[] = [
  { label: 'Hadiah 1', weight: 1, color: '#FF6B6B' },
  { label: 'Hadiah 2', weight: 1, color: '#4ECDC4' },
  { label: 'Hadiah 3', weight: 1, color: '#45B7D1' },
  { label: 'Zonk!', weight: 2, color: '#96CEB4' },
  { label: 'Bonus', weight: 1, color: '#FFEAA7' },
];

const segments = ref<Segment[]>([...defaultSegments]);
const isModalOpen = ref(false);
const isSettingsOpen = ref(false);
const editingIndex = ref<number | null>(null);
const deleteTargetIndex = ref<number | null>(null);
const lastResult = ref<string>('');
const isSpinning = ref(false);
const wheelCanvas = ref<HTMLCanvasElement | null>(null);
let animationId: number | null = null;
let currentRotation = 0;
let targetRotation = 0;
let spinStartTime = 0;
let spinDuration = 3000;
let spinStartRotation = 0;

const settings = ref({
  duration: 3000,
  minSpins: 5,
  sound: true,
});

const formData = ref({
  label: '',
  weight: 1,
  color: colorOptions[0],
});

const emptyForm = () => ({
  label: '',
  weight: 1,
  color: colorOptions[Math.floor(Math.random() * colorOptions.length)],
});

const randomColor = () => {
  formData.value.color = colorOptions[Math.floor(Math.random() * colorOptions.length)];
};

const randomizeColors = () => {
  segments.value.forEach(seg => {
    seg.color = colorOptions[Math.floor(Math.random() * colorOptions.length)];
  });
  saveToLocal();
  drawWheel();
};

// Wheel drawing
const drawWheel = () => {
  const canvas = wheelCanvas.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;
  
  const totalWeight = segments.value.reduce((sum, seg) => sum + seg.weight, 0);
  if (totalWeight === 0) return;
  
  const size = canvas.width;
  const center = size / 2;
  const radius = size / 2 - 10;
  
  ctx.clearRect(0, 0, size, size);
  
  let startAngle = currentRotation;
  
  for (let i = 0; i < segments.value.length; i++) {
    const seg = segments.value[i];
    const angle = (seg.weight / totalWeight) * Math.PI * 2;
    const endAngle = startAngle + angle;
    
    // Draw segment
    ctx.beginPath();
    ctx.moveTo(center, center);
    ctx.arc(center, center, radius, startAngle, endAngle);
    ctx.closePath();
    
    ctx.fillStyle = seg.color;
    ctx.fill();
    ctx.strokeStyle = 'rgba(255,255,255,0.3)';
    ctx.lineWidth = 2;
    ctx.stroke();
    
    // Draw text
    ctx.save();
    ctx.translate(center, center);
    ctx.rotate(startAngle + angle / 2);
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    const textRadius = radius * 0.65;
    ctx.font = `bold ${Math.max(10, Math.min(14, 200 / segments.value.length))}px "Plus Jakarta Sans"`;
    ctx.fillStyle = '#ffffff';
    ctx.shadowBlur = 4;
    ctx.shadowColor = 'rgba(0,0,0,0.3)';
    
    let displayText = seg.label;
    if (displayText.length > 12) displayText = displayText.slice(0, 10) + '..';
    ctx.fillText(displayText, textRadius, 6);
    ctx.restore();
    
    startAngle = endAngle;
  }
  
  // Draw inner circle
  ctx.beginPath();
  ctx.arc(center, center, 28, 0, Math.PI * 2);
  ctx.fillStyle = 'rgba(15, 17, 30, 0.9)';
  ctx.fill();
  ctx.strokeStyle = 'rgba(255,255,255,0.2)';
  ctx.lineWidth = 2;
  ctx.stroke();
  
  // Draw center dot
  ctx.beginPath();
  ctx.arc(center, center, 12, 0, Math.PI * 2);
  ctx.fillStyle = 'rgba(167, 139, 250, 0.8)';
  ctx.fill();
};

const getWinner = (finalRotation: number): string => {
  const totalWeight = segments.value.reduce((sum, seg) => sum + seg.weight, 0);
  if (totalWeight === 0) return '';
  
  // Normalize rotation to 0-360
  let rotation = finalRotation % (Math.PI * 2);
  if (rotation < 0) rotation += Math.PI * 2;
  
  // The pointer is at top (270 degrees or -90 degrees)
  // We need to find which segment is at that position
  const pointerAngle = (Math.PI * 3) / 2; // 270 degrees
  let targetAngle = (pointerAngle - rotation) % (Math.PI * 2);
  if (targetAngle < 0) targetAngle += Math.PI * 2;
  
  let accumulated = 0;
  for (const seg of segments.value) {
    const angle = (seg.weight / totalWeight) * Math.PI * 2;
    if (targetAngle >= accumulated && targetAngle < accumulated + angle) {
      return seg.label;
    }
    accumulated += angle;
  }
  return segments.value[0]?.label || '';
};

const playSound = () => {
  if (!settings.value.sound) return;
  // Simple beep sound using Web Audio API
  try {
    const audioContext = new (window.AudioContext || (window as any).webkitAudioContext)();
    const oscillator = audioContext.createOscillator();
    const gainNode = audioContext.createGain();
    oscillator.connect(gainNode);
    gainNode.connect(audioContext.destination);
    oscillator.frequency.value = 880;
    gainNode.gain.value = 0.1;
    oscillator.start();
    gainNode.gain.exponentialRampToValueAtTime(0.00001, audioContext.currentTime + 0.3);
    oscillator.stop(audioContext.currentTime + 0.3);
    // Close context after sound to avoid memory issues
    setTimeout(() => audioContext.close(), 400);
  } catch (e) {
    console.log('Audio not supported');
  }
};

const spinWheel = () => {
  if (isSpinning.value) return;
  if (segments.value.length === 0) return;
  
  isSpinning.value = true;
  lastResult.value = '';
  
  const minSpins = settings.value.minSpins;
  const extraSpins = Math.random() * 5 + 2;
  const totalSpins = minSpins + extraSpins;
  const targetAngleRad = Math.random() * Math.PI * 2;
  const newTargetRotation = currentRotation + (Math.PI * 2 * totalSpins) + targetAngleRad;
  
  targetRotation = newTargetRotation;
  spinStartRotation = currentRotation;
  spinStartTime = performance.now();
  spinDuration = settings.value.duration;
  
  playSound();
  
  const animate = (now: number) => {
    const elapsed = now - spinStartTime;
    let progress = Math.min(1, elapsed / spinDuration);
    
    // Easing out cubic
    const easeOut = 1 - Math.pow(1 - progress, 3);
    
    currentRotation = spinStartRotation + (targetRotation - spinStartRotation) * easeOut;
    drawWheel();
    
    if (progress < 1) {
      animationId = requestAnimationFrame(animate);
    } else {
      currentRotation = targetRotation % (Math.PI * 2);
      drawWheel();
      const winner = getWinner(currentRotation);
      lastResult.value = winner;
      isSpinning.value = false;
      animationId = null;
      
      // Flash effect for winner
      setTimeout(() => {
        if (winner) playSound();
      }, 100);
    }
  };
  
  if (animationId) cancelAnimationFrame(animationId);
  animationId = requestAnimationFrame(animate);
};

const addSegment = () => {
  editingIndex.value = null;
  formData.value = emptyForm();
  isModalOpen.value = true;
};

const editSegment = (idx: number) => {
  editingIndex.value = idx;
  const seg = segments.value[idx];
  formData.value = {
    label: seg.label,
    weight: seg.weight,
    color: seg.color,
  };
  isModalOpen.value = true;
};

const deleteSegment = (idx: number) => {
  deleteTargetIndex.value = idx;
};

const deleteAlertButtons = [
  { text: 'Batal', role: 'cancel', handler: () => { deleteTargetIndex.value = null; } },
  {
    text: 'Hapus', role: 'destructive',
    handler: () => {
      if (deleteTargetIndex.value !== null) {
        segments.value.splice(deleteTargetIndex.value, 1);
        deleteTargetIndex.value = null;
        saveToLocal();
        drawWheel();
        if (segments.value.length === 0) lastResult.value = '';
      }
    },
  },
];

const saveSegment = () => {
  if (!formData.value.label.trim()) return;
  
  const newSegment: Segment = {
    label: formData.value.label.trim(),
    weight: Math.max(1, Number(formData.value.weight) || 1),
    color: formData.value.color,
  };
  
  if (editingIndex.value !== null) {
    segments.value[editingIndex.value] = newSegment;
  } else {
    segments.value.push(newSegment);
  }
  
  saveToLocal();
  closeModal();
  drawWheel();
};

const resetSegments = () => {
  segments.value = JSON.parse(JSON.stringify(defaultSegments));
  saveToLocal();
  drawWheel();
  lastResult.value = '';
};

const openSettings = () => {
  isSettingsOpen.value = true;
};

const closeSettings = () => {
  isSettingsOpen.value = false;
};

const saveSettings = () => {
  saveSettingsToLocal();
  closeSettings();
};

const closeModal = () => {
  isModalOpen.value = false;
};

const saveToLocal = () => {
  localStorage.setItem('myrodaputar-segments', JSON.stringify(segments.value));
};

const loadFromLocal = () => {
  const saved = localStorage.getItem('myrodaputar-segments');
  if (saved) {
    try {
      const parsed = JSON.parse(saved);
      if (parsed.length > 0) segments.value = parsed;
    } catch (e) {}
  }
  loadSettingsFromLocal();
};

const saveSettingsToLocal = () => {
  localStorage.setItem('myrodaputar-settings', JSON.stringify(settings.value));
};

const loadSettingsFromLocal = () => {
  const saved = localStorage.getItem('myrodaputar-settings');
  if (saved) {
    try {
      const parsed = JSON.parse(saved);
      settings.value = { ...settings.value, ...parsed };
    } catch (e) {}
  }
};

// Redraw when segments change
watch(segments, () => {
  if (!isSpinning.value) {
    drawWheel();
  }
}, { deep: true });

onMounted(() => {
  loadFromLocal();
  nextTick(() => {
    drawWheel();
  });
});
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:ital,wght@0,300;0,400;0,500;0,600;0,700;1,400&family=Syne:wght@700;800&display=swap');

/* ─── Variables (same as original) ─── */
:host {
  --bg: #0d0f1a;
  --glass-bg: rgba(255, 255, 255, 0.07);
  --glass-bg-hover: rgba(255, 255, 255, 0.11);
  --glass-border: rgba(255, 255, 255, 0.11);
  --glass-border-hover: rgba(255, 255, 255, 0.22);
  --shimmer: rgba(255, 255, 255, 0.18);
  --text-primary: #ffffff;
  --text-secondary: rgba(255, 255, 255, 0.6);
  --text-muted: rgba(255, 255, 255, 0.35);
  --accent-purple: #a78bfa;
  --accent-pink: #ec4899;
  --accent-blue: #60a5fa;
  --font-display: 'Syne', sans-serif;
  --font-body: 'Plus Jakarta Sans', sans-serif;
  --blur: blur(28px);
}

ion-content {
  --background: #0d0f1a;
  font-family: var(--font-body);
}

.app-page {
  background: #0d0f1a;
}

/* Blobs (same) */
.blob {
  position: fixed;
  border-radius: 50%;
  pointer-events: none;
  z-index: 0;
}
.blob-1 {
  width: 340px; height: 340px;
  background: radial-gradient(circle, rgba(139, 92, 246, 0.38) 0%, transparent 70%);
  top: -80px; right: -80px;
  animation: blobFloat1 12s ease-in-out infinite;
}
.blob-2 {
  width: 260px; height: 260px;
  background: radial-gradient(circle, rgba(236, 72, 153, 0.28) 0%, transparent 70%);
  top: 240px; left: -100px;
  animation: blobFloat2 15s ease-in-out infinite;
}
.blob-3 {
  width: 220px; height: 220px;
  background: radial-gradient(circle, rgba(59, 130, 246, 0.22) 0%, transparent 70%);
  bottom: 280px; right: -50px;
  animation: blobFloat3 10s ease-in-out infinite;
}

@keyframes blobFloat1 {
  0%, 100% { transform: translate(0, 0) scale(1); }
  50% { transform: translate(-20px, 30px) scale(1.06); }
}
@keyframes blobFloat2 {
  0%, 100% { transform: translate(0, 0) scale(1); }
  50% { transform: translate(30px, -20px) scale(0.95); }
}
@keyframes blobFloat3 {
  0%, 100% { transform: translate(0, 0) scale(1); }
  50% { transform: translate(-15px, 25px) scale(1.04); }
}

/* Glass Card */
.glass-card {
  background: var(--glass-bg);
  backdrop-filter: var(--blur);
  -webkit-backdrop-filter: var(--blur);
  border: 1px solid var(--glass-border);
  position: relative;
  overflow: hidden;
}
.glass-card::before {
  content: '';
  position: absolute;
  top: 0; left: 0; right: 0;
  height: 1px;
  background: linear-gradient(90deg, transparent, var(--shimmer), transparent);
  pointer-events: none;
}

/* Header */
.glass-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px 12px;
  position: relative;
  z-index: 10;
}
.brand {
  font-family: var(--font-display);
  font-size: 22px;
  font-weight: 800;
  color: #fff;
  letter-spacing: -0.5px;
}
.brand em {
  font-style: normal;
  color: var(--accent-purple);
}
.header-avatar {
  width: 40px; height: 40px;
  border-radius: 50%;
  background: rgba(255,255,255,0.1);
  backdrop-filter: blur(10px);
  display: flex; align-items: center; justify-content: center;
  cursor: pointer;
  transition: all 0.2s;
  color: white;
}
.header-avatar:active { transform: scale(0.95); }

/* Hero Card */
.hero-card {
  margin: 0 16px 20px;
  border-radius: 28px;
  padding: 24px;
  position: relative;
  z-index: 10;
}
.hero-eyebrow {
  font-family: var(--font-body);
  font-size: 10px;
  font-weight: 600;
  letter-spacing: 0.16em;
  text-transform: uppercase;
  color: var(--text-muted);
  margin-bottom: 10px;
}
.hero-count-row {
  display: flex;
  align-items: flex-end;
  gap: 12px;
  margin-bottom: 20px;
}
.hero-number {
  font-family: var(--font-display);
  font-size: 80px;
  font-weight: 800;
  line-height: 0.9;
  color: #fff;
  letter-spacing: -5px;
}
.hero-text-group {
  padding-bottom: 10px;
  display: flex;
  flex-direction: column;
}
.hero-big-label {
  font-family: var(--font-body);
  font-size: 13px; font-weight: 700;
  color: rgba(255,255,255,0.7);
  text-transform: uppercase;
  letter-spacing: 0.1em;
}
.hero-sub-label {
  font-size: 11px; font-weight: 400;
  color: #fff;
  margin-top: 2px;
}
.stats-row {
  display: flex;
  gap: 8px;
  color: #fff;
}
.stat-pill {
  flex: 1;
  background: rgba(255, 255, 255, 0.07);
  border: 1px solid rgba(255, 255, 255, 0.09);
  border-radius: 16px;
  padding: 10px 12px;
  text-align: center;
  display: flex; flex-direction: column;
  align-items: center;
  cursor: pointer;
  transition: all 0.15s;
}
.stat-pill:active { transform: scale(0.96); background: rgba(255,255,255,0.12); }
.accent-pill {
  background: rgba(167, 139, 250, 0.14);
  border-color: rgba(167, 139, 250, 0.28);
}
.accent-label { color: rgba(167, 139, 250, 0.6) !important; }

/* Wheel Container */
.wheel-container {
  position: relative;
  display: flex;
  justify-content: center;
  margin: 16px 0 20px;
  z-index: 10;
}
.wheel-canvas {
  width: min(85vw, 85vh, 380px);
  height: min(85vw, 85vh, 380px);
  border-radius: 50%;
  box-shadow: 0 20px 40px rgba(0,0,0,0.4), 0 0 0 8px rgba(255,255,255,0.05);
  transition: box-shadow 0.2s;
}
.pointer {
  position: absolute;
  top: -20px;
  left: 50%;
  transform: translateX(-50%);
  cursor: pointer;
  z-index: 20;
}
.pointer-inner {
  width: 60px;
  height: 60px;
  background: linear-gradient(135deg, var(--accent-purple), var(--accent-pink));
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  box-shadow: 0 8px 24px rgba(167, 139, 250, 0.5);
  transition: transform 0.1s ease;
  cursor: pointer;
}
.pointer:active .pointer-inner {
  transform: scale(0.92);
}
/* Pointer triangle tip */
.pointer::before {
  content: '';
  position: absolute;
  top: 48px;
  left: 50%;
  transform: translateX(-50%);
  width: 0;
  height: 0;
  border-left: 14px solid transparent;
  border-right: 14px solid transparent;
  border-top: 28px solid var(--accent-purple);
  filter: drop-shadow(0 4px 6px rgba(0,0,0,0.3));
}

/* Result Card */
.result-card {
  margin: 8px 16px 20px;
  border-radius: 20px;
  padding: 16px 20px;
  text-align: center;
  z-index: 10;
  position: relative;
}
.result-label {
  font-size: 9px;
  font-weight: 600;
  letter-spacing: 0.2em;
  text-transform: uppercase;
  color: var(--text-muted);
  margin-bottom: 8px;
}
.result-value {
  font-family: var(--font-display);
  font-size: 20px;
  font-weight: 800;
  color: #fff;
  word-break: break-word;
}
.placeholder-card .result-value {
  color: var(--text-muted);
  font-size: 24px;
}

/* Section Row */
.section-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 4px 20px 14px;
  position: relative;
  z-index: 10;
}
.section-title {
  font-family: var(--font-display);
  font-size: 16px; font-weight: 700;
  color: #fff;
}
.add-pill {
  display: flex;
  align-items: center;
  gap: 5px;
  background: linear-gradient(135deg, var(--accent-purple), var(--accent-pink));
  border: none;
  border-radius: 100px;
  padding: 8px 16px;
  font-family: var(--font-body);
  font-size: 11px; font-weight: 700;
  color: #fff;
  cursor: pointer;
  transition: all 0.18s ease;
}
.add-pill:active { transform: scale(0.96); }

/* Segment List */
.segment-scroll {
  padding: 0 16px;
  position: relative;
  z-index: 10;
}
.sliding-wrap {
  margin-bottom: 10px;
  border-radius: 20px;
  overflow: hidden;
}
.segment-item-wrap {
  --background: transparent;
  --inner-padding-start: 0;
  --inner-padding-end: 0;
  --padding-start: 0;
  --padding-end: 0;
  --min-height: auto;
}
.segment-card {
  border-radius: 20px;
  padding: 12px 14px;
  display: flex;
  align-items: center;
  gap: 12px;
  width: 100%;
}
.segment-color {
  width: 40px;
  height: 40px;
  border-radius: 12px;
  flex-shrink: 0;
  border: 1px solid rgba(255,255,255,0.2);
}
.segment-info {
  flex: 1;
}
.segment-name {
  font-family: var(--font-display);
  font-size: 14px;
  font-weight: 700;
  color: #fff;
  margin-bottom: 2px;
}
.segment-weight {
  font-size: 10px;
  color: var(--text-muted);
}
.segment-arrow {
  color: rgba(255,255,255,0.3);
  font-size: 16px;
  padding: 8px;
  cursor: pointer;
  transition: color 0.15s;
}
.segment-arrow:active { color: var(--accent-purple); }

/* Swipe options */
ion-item-option[color="danger"] {
  --background: rgba(239, 68, 68, 0.85);
}
.edit-swipe {
  --background: rgba(167, 139, 250, 0.2);
  --color: var(--accent-purple);
}

/* Empty State */
.empty-state {
  display: flex; flex-direction: column;
  align-items: center;
  padding: 60px 24px;
  text-align: center;
  position: relative; z-index: 10;
}
.empty-blob {
  position: absolute;
  width: 200px; height: 200px;
  border-radius: 50%;
  background: radial-gradient(circle, rgba(167, 139, 250, 0.15) 0%, transparent 70%);
  top: 0; left: 50%; transform: translateX(-50%);
}
.empty-icon { font-size: 3.5rem; margin-bottom: 16px; opacity: 0.5; }
.empty-title {
  font-family: var(--font-display);
  font-size: 18px; font-weight: 700;
  color: rgba(255,255,255,0.6);
  margin-bottom: 8px;
}
.empty-sub {
  font-size: 13px; color: var(--text-muted);
  margin-bottom: 28px;
}
.empty-cta {
  font-family: var(--font-body);
  font-size: 13px; font-weight: 700;
  padding: 12px 28px; border-radius: 100px;
  background: linear-gradient(135deg, var(--accent-purple), var(--accent-pink));
  border: none; color: #fff; cursor: pointer;
}

/* Modal */
ion-modal.glass-modal {
  --background: rgba(15, 17, 30, 0.85);
  --backdrop-opacity: 0.5;
}
ion-modal.glass-modal::part(content) {
  backdrop-filter: blur(40px);
  border-top: 1px solid rgba(255,255,255,0.1);
  border-radius: 28px 28px 0 0;
}
.modal-toolbar {
  --background: transparent;
  --border-color: rgba(255,255,255,0.08);
}
.modal-title {
  font-family: var(--font-display) !important;
  font-size: 16px !important; font-weight: 700 !important;
  color: #fff !important;
}
.modal-body {
  --background: transparent;
}
.form-group {
  padding: 0 18px;
  margin-bottom: 20px;
}
.form-label {
  display: block;
  font-size: 10px; font-weight: 600;
  text-transform: uppercase; letter-spacing: 0.14em;
  color: var(--text-muted);
  margin-bottom: 8px;
}
.req { color: var(--accent-purple); }
.opt { color: rgba(255,255,255,0.2); text-transform: none; font-size: 9px; }

.glass-input {
  --background: rgba(255,255,255,0.07);
  --color: #fff;
  --placeholder-color: rgba(255,255,255,0.2);
  --padding-start: 14px;
  --padding-end: 14px;
  --padding-top: 12px;
  --padding-bottom: 12px;
  border: 1px solid rgba(255,255,255,0.1);
  border-radius: 14px;
  font-size: 14px;
}

.color-options {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-top: 8px;
}
.color-opt {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  border: 2px solid transparent;
  cursor: pointer;
  transition: all 0.15s;
}
.color-opt.selected {
  border-color: white;
  transform: scale(1.1);
  box-shadow: 0 0 0 2px rgba(167, 139, 250, 0.5);
}
.random-color {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
}

.modal-footer {
  display: grid;
  grid-template-columns: 1fr 2fr;
  gap: 10px;
  padding: 12px 18px 44px;
}
.modal-footer.single {
  grid-template-columns: 1fr;
}
.btn-cancel {
  --color: rgba(255,255,255,0.3);
  border: 1px solid rgba(255,255,255,0.1);
  border-radius: 14px;
}
.btn-save {
  --background: linear-gradient(135deg, #a78bfa, #ec4899);
  --color: #fff;
  font-weight: 700;
  border-radius: 14px;
}
.btn-save[disabled] {
  --background: rgba(255,255,255,0.07);
  --color: rgba(255,255,255,0.2);
}

.toggle-switch {
  background: rgba(255,255,255,0.1);
  padding: 12px;
  border-radius: 14px;
  text-align: center;
  cursor: pointer;
  font-weight: 600;
  color: white;
  transition: all 0.15s;
}
.toggle-switch:active { transform: scale(0.98); background: rgba(255,255,255,0.15); }
</style>