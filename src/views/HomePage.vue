<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>
          <div class="brand">
            <span class="brand-icon">⬡</span>
            <span class="brand-text">Jadwalkeun</span>
          </div>
        </ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">rencanaku</ion-title>
        </ion-toolbar>
      </ion-header>

      <!-- Hero Stats -->
      <div class="hero-section">
        <div class="hero-bg-orb orb-1"></div>
        <div class="hero-bg-orb orb-2"></div>
        <div class="hero-date">{{ todayStr }}</div>
        <div class="hero-headline">
          <span class="hero-count">{{ pendingCount }}</span>
          <span class="hero-label">tugas<br>menunggu</span>
        </div>
        <div class="progress-track">
          <div class="progress-fill" :style="{ width: progressPct + '%' }">
            <span class="progress-glow"></span>
          </div>
        </div>
        <div class="progress-meta">
          <span class="meta-badge done">✓ {{ completedCount }} selesai</span>
          <span class="meta-badge total">{{ todos.length }} total</span>
        </div>
      </div>

      <!-- Filter Tabs + Add Button Row -->
      <div class="toolbar-row">
        <div class="filter-tabs">
          <button
            v-for="tab in tabs"
            :key="tab.value"
            class="tab-pill"
            :class="{ active: filterType === tab.value }"
            @click="filterType = tab.value"
          >
            {{ tab.label }}
          </button>
        </div>
        <button class="add-inline-btn" @click="openAddModal">
          <ion-icon :icon="addOutline"></ion-icon>
          Tambah
        </button>
      </div>

      <!-- Todo List -->
      <ion-list lines="none" class="todo-list" v-if="filteredTodos.length > 0">
        <ion-item-sliding v-for="(todo, idx) in filteredTodos" :key="todo.id">
          <ion-item-options side="end">
            <ion-item-option color="danger" @click="confirmDelete(todo.id)">
              <ion-icon :icon="trashOutline" slot="icon-only"></ion-icon>
            </ion-item-option>
          </ion-item-options>
          <ion-item-options side="start">
            <ion-item-option class="edit-option" @click="editTodo(todo)">
              <ion-icon :icon="createOutline" slot="icon-only"></ion-icon>
            </ion-item-option>
          </ion-item-options>

          <ion-item class="todo-item" :class="{ 'is-done': todo.completed }">
            <div class="item-index" slot="start">{{ String(idx + 1).padStart(2, '0') }}</div>

            <ion-label class="todo-label">
              <h2 class="todo-title">{{ todo.title }}</h2>
              <p v-if="todo.description" class="todo-desc">{{ todo.description }}</p>
              <p class="todo-date">
                <ion-icon :icon="calendarOutline"></ion-icon>
                {{ formatDate(todo.date) }}
              </p>
            </ion-label>

            <div slot="end" class="item-end">
              <div class="check-bubble" :class="{ checked: todo.completed }" @click="toggleTodo(todo)">
                <ion-icon v-if="todo.completed" :icon="checkmarkOutline"></ion-icon>
              </div>
              <div class="mini-actions">
                <ion-button fill="clear" size="small" @click="editTodo(todo)">
                  <ion-icon :icon="createOutline" slot="icon-only"></ion-icon>
                </ion-button>
                <ion-button fill="clear" size="small" color="danger" @click="confirmDelete(todo.id)">
                  <ion-icon :icon="trashOutline" slot="icon-only"></ion-icon>
                </ion-button>
              </div>
            </div>
          </ion-item>
        </ion-item-sliding>
      </ion-list>

      <!-- Empty State -->
      <div v-else class="empty-state">
        <div class="empty-glyph">
          <span class="empty-ring"></span>
          <span class="empty-dot"></span>
        </div>
        <p class="empty-title">Belum ada rencana</p>
        <p class="empty-sub">Mulai dengan menambahkan tugas pertamamu</p>
      </div>
    </ion-content>

    <!-- Add/Edit Modal -->
    <ion-modal
      :is-open="isModalOpen"
      @didDismiss="closeModal"
      :breakpoints="[0, 0.6, 0.9]"
      :initial-breakpoint="0.6"
    >
      <ion-header>
        <ion-toolbar>
          <ion-title class="modal-title">{{ editingTodo ? 'Edit Rencana' : 'Rencana Baru' }}</ion-title>
          <ion-buttons slot="end">
            <ion-button @click="closeModal">
              <ion-icon :icon="closeOutline" slot="icon-only"></ion-icon>
            </ion-button>
          </ion-buttons>
        </ion-toolbar>
      </ion-header>
      <ion-content class="modal-content">
        <div class="form-group">
          <label class="form-label">Judul <span class="req">*</span></label>
          <ion-input
            class="form-input"
            v-model="formData.title"
            placeholder="Apa yang ingin kamu lakukan?"
            @keyup.enter="saveTodo"
            clearInput
          ></ion-input>
        </div>

        <div class="form-group">
          <label class="form-label">Deskripsi <span class="opt">— opsional</span></label>
          <ion-textarea
            class="form-input"
            v-model="formData.description"
            placeholder="Tambahkan detail..."
            :rows="3"
            autoGrow
          ></ion-textarea>
        </div>

        <div class="form-group">
          <label class="form-label">Tanggal</label>
          <ion-input
            class="form-input"
            type="date"
            v-model="formData.dateInput"
          ></ion-input>
        </div>

        <div class="modal-footer">
          <ion-button expand="block" fill="clear" class="btn-cancel" @click="closeModal">
            Batal
          </ion-button>
          <ion-button
            expand="block"
            class="btn-save"
            :disabled="!formData.title.trim()"
            @click="saveTodo"
          >
            {{ editingTodo ? 'Simpan' : 'Tambah' }}
          </ion-button>
        </div>
      </ion-content>
    </ion-modal>

    <!-- Delete Alert -->
    <ion-alert
      :is-open="deleteTargetId !== null"
      header="Hapus?"
      message="Tugas ini akan dihapus permanen."
      :buttons="deleteAlertButtons"
      @didDismiss="deleteTargetId = null"
    ></ion-alert>
  </ion-page>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import {
  IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButtons, IonButton,
  IonIcon, IonList, IonItem, IonItemSliding, IonItemOptions, IonItemOption,
  IonLabel, IonModal, IonInput, IonTextarea, IonAlert,
} from '@ionic/vue';
import {
  addOutline, checkmarkOutline, createOutline, trashOutline,
  calendarOutline, closeOutline,
} from 'ionicons/icons';

interface Todo {
  id: number;
  title: string;
  description?: string;
  completed: boolean;
  date: string;
}

const todos = ref<Todo[]>([]);
const isModalOpen = ref(false);
const editingTodo = ref<Todo | null>(null);
const filterType = ref('all');
const deleteTargetId = ref<number | null>(null);

const formData = ref({
  title: '',
  description: '',
  dateInput: new Date().toISOString().split('T')[0],
});

const tabs = [
  { value: 'all', label: 'Semua' },
  { value: 'pending', label: 'Pending' },
  { value: 'completed', label: 'Selesai' },
];

const todayStr = computed(() => new Date().toLocaleDateString('id-ID', {
  weekday: 'long', day: 'numeric', month: 'long', year: 'numeric'
}));

const completedCount = computed(() => todos.value.filter(t => t.completed).length);
const pendingCount = computed(() => todos.value.filter(t => !t.completed).length);
const progressPct = computed(() =>
  todos.value.length === 0 ? 0 : Math.round((completedCount.value / todos.value.length) * 100)
);

const filteredTodos = computed(() => {
  if (filterType.value === 'completed') return todos.value.filter(t => t.completed);
  if (filterType.value === 'pending') return todos.value.filter(t => !t.completed);
  return todos.value;
});

const formatDate = (iso: string) => {
  const d = new Date(iso);
  if (isNaN(d.getTime())) return iso;
  return d.toLocaleDateString('id-ID', { day: 'numeric', month: 'short', year: 'numeric' });
};

const loadTodos = () => {
  const saved = localStorage.getItem('ionic-todos');
  if (saved) todos.value = JSON.parse(saved);
};

const saveToStorage = () => {
  localStorage.setItem('ionic-todos', JSON.stringify(todos.value));
};

const openAddModal = () => {
  editingTodo.value = null;
  formData.value = { title: '', description: '', dateInput: new Date().toISOString().split('T')[0] };
  isModalOpen.value = true;
};

const editTodo = (todo: Todo) => {
  editingTodo.value = todo;
  formData.value = {
    title: todo.title,
    description: todo.description || '',
    dateInput: todo.date.split('T')[0],
  };
  isModalOpen.value = true;
};

const closeModal = () => { isModalOpen.value = false; };

const saveTodo = () => {
  if (!formData.value.title.trim()) return;
  const dateISO = new Date(formData.value.dateInput).toISOString();
  if (editingTodo.value) {
    const idx = todos.value.findIndex(t => t.id === editingTodo.value!.id);
    if (idx !== -1) {
      todos.value[idx] = {
        ...editingTodo.value,
        title: formData.value.title,
        description: formData.value.description,
        date: dateISO,
      };
    }
  } else {
    todos.value.push({
      id: Date.now(),
      title: formData.value.title,
      description: formData.value.description,
      completed: false,
      date: dateISO,
    });
  }
  saveToStorage();
  closeModal();
};

const toggleTodo = (todo: Todo) => {
  todo.completed = !todo.completed;
  saveToStorage();
};

const confirmDelete = (id: number) => { deleteTargetId.value = id; };

const deleteAlertButtons = [
  { text: 'Batal', role: 'cancel', handler: () => { deleteTargetId.value = null; } },
  {
    text: 'Hapus',
    role: 'destructive',
    handler: () => {
      if (deleteTargetId.value === null) return;
      todos.value = todos.value.filter(t => t.id !== deleteTargetId.value);
      saveToStorage();
      deleteTargetId.value = null;
    },
  },
];

onMounted(loadTodos);
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Space+Mono:ital,wght@0,400;0,700;1,400&family=DM+Sans:ital,opsz,wght@0,9..40,300;0,9..40,400;0,9..40,600;1,9..40,300&family=Syne:wght@400;600;800&display=swap');

/* ── CSS Variables ── */
:host {
  --bg-base: #080c14;
  --bg-surface: rgba(12, 18, 30, 0.85);
  --bg-card: rgba(15, 22, 38, 0.9);
  --accent-teal: #00e5c0;
  --accent-teal-dim: rgba(0, 229, 192, 0.15);
  --accent-teal-glow: rgba(0, 229, 192, 0.35);
  --accent-blue: #3d7fff;
  --accent-red: #ff4757;
  --text-primary: #e8edf5;
  --text-secondary: #6b7a99;
  --text-muted: #3a4560;
  --border: rgba(255, 255, 255, 0.06);
  --border-accent: rgba(0, 229, 192, 0.3);
}

/* ── Base ── */
ion-toolbar {
  --background: #080c14;
  --color: #e8edf5;
  --border-color: transparent;
  font-family: 'DM Sans', sans-serif;
}

ion-content {
  --background: #080c14;
  --color: #e8edf5;
  font-family: 'DM Sans', sans-serif;
}

/* Subtle grid background */
ion-content::part(background) {
  background: 
    linear-gradient(rgba(0, 229, 192, 0.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0, 229, 192, 0.03) 1px, transparent 1px),
    #080c14;
  background-size: 40px 40px;
}

/* ── Brand ── */
.brand {
  display: flex;
  align-items: center;
  gap: 9px;
  font-family: 'Syne', sans-serif;
  font-weight: 800;
  font-size: 1.1rem;
  color: #e8edf5;
  letter-spacing: -0.02em;
}

.brand-icon {
  color: #00e5c0;
  font-size: 1rem;
  filter: drop-shadow(0 0 6px rgba(0, 229, 192, 0.8));
  animation: pulse-icon 3s ease-in-out infinite;
}

@keyframes pulse-icon {
  0%, 100% { filter: drop-shadow(0 0 6px rgba(0, 229, 192, 0.8)); }
  50% { filter: drop-shadow(0 0 14px rgba(0, 229, 192, 1)); }
}

/* ── Hero Section ── */
.hero-section {
  position: relative;
  margin: 14px 14px 0;
  border-radius: 20px;
  padding: 26px 22px 22px;
  background: linear-gradient(135deg, #0d1628 0%, #0a1120 60%, #0d1a2e 100%);
  border: 1px solid rgba(0, 229, 192, 0.18);
  overflow: hidden;
  box-shadow: 0 0 0 1px rgba(0,0,0,0.5), 0 20px 60px rgba(0, 0, 0, 0.6), inset 0 1px 0 rgba(255,255,255,0.05);
}

.hero-bg-orb {
  position: absolute;
  border-radius: 50%;
  pointer-events: none;
  filter: blur(50px);
}

.orb-1 {
  width: 180px;
  height: 180px;
  background: radial-gradient(circle, rgba(0, 229, 192, 0.12) 0%, transparent 70%);
  top: -50px;
  right: -40px;
}

.orb-2 {
  width: 120px;
  height: 120px;
  background: radial-gradient(circle, rgba(61, 127, 255, 0.1) 0%, transparent 70%);
  bottom: -30px;
  left: 10px;
}

.hero-date {
  font-family: 'Space Mono', monospace;
  font-size: 0.58rem;
  color: #3a4560;
  text-transform: uppercase;
  letter-spacing: 0.15em;
  margin-bottom: 16px;
}

.hero-headline {
  display: flex;
  align-items: baseline;
  gap: 14px;
  margin-bottom: 22px;
}

.hero-count {
  font-family: 'Syne', sans-serif;
  font-weight: 800;
  font-size: 4.2rem;
  line-height: 1;
  color: #00e5c0;
  text-shadow: 0 0 40px rgba(0, 229, 192, 0.5);
  letter-spacing: -0.03em;
}

.hero-label {
  font-family: 'DM Sans', sans-serif;
  font-size: 0.78rem;
  color: #4a5a7a;
  font-weight: 300;
  line-height: 1.5;
}

.progress-track {
  height: 3px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 100px;
  overflow: visible;
  margin-bottom: 12px;
  position: relative;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #00e5c0, #3d7fff);
  border-radius: 100px;
  transition: width 0.8s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  box-shadow: 0 0 10px rgba(0, 229, 192, 0.6), 0 0 25px rgba(0, 229, 192, 0.3);
}

.progress-fill::after {
  content: '';
  position: absolute;
  right: -1px;
  top: 50%;
  transform: translateY(-50%);
  width: 7px;
  height: 7px;
  background: #00e5c0;
  border-radius: 50%;
  box-shadow: 0 0 8px rgba(0, 229, 192, 1), 0 0 16px rgba(0, 229, 192, 0.6);
}

.progress-meta {
  display: flex;
  justify-content: space-between;
}

.meta-badge {
  font-family: 'Space Mono', monospace;
  font-size: 0.58rem;
  letter-spacing: 0.05em;
  padding: 2px 8px;
  border-radius: 100px;
}

.meta-badge.done {
  color: #00e5c0;
  background: rgba(0, 229, 192, 0.08);
  border: 1px solid rgba(0, 229, 192, 0.2);
}

.meta-badge.total {
  color: #3a4560;
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.06);
}

/* ── Toolbar Row ── */
.toolbar-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 14px 6px;
  gap: 8px;
}

.filter-tabs {
  display: flex;
  gap: 5px;
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 100px;
  padding: 3px;
}

.tab-pill {
  font-family: 'DM Sans', sans-serif;
  font-size: 0.72rem;
  font-weight: 400;
  padding: 5px 13px;
  border-radius: 100px;
  border: none;
  background: transparent;
  color: #4a5a7a;
  cursor: pointer;
  transition: all 0.2s ease;
  letter-spacing: 0.01em;
}

.tab-pill.active {
  background: #00e5c0;
  color: #080c14;
  font-weight: 600;
  box-shadow: 0 0 12px rgba(0, 229, 192, 0.4);
}

.add-inline-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  font-family: 'DM Sans', sans-serif;
  font-size: 0.75rem;
  font-weight: 600;
  padding: 7px 15px;
  border-radius: 12px;
  border: 1px solid rgba(0, 229, 192, 0.4);
  background: rgba(0, 229, 192, 0.08);
  color: #00e5c0;
  cursor: pointer;
  transition: all 0.18s ease;
  white-space: nowrap;
  flex-shrink: 0;
  letter-spacing: 0.02em;
}

.add-inline-btn:active {
  transform: scale(0.96);
  background: rgba(0, 229, 192, 0.18);
}

.add-inline-btn ion-icon {
  font-size: 14px;
}

/* ── Todo List ── */
.todo-list {
  background: transparent;
  padding: 6px 14px 80px;
}

ion-item-sliding {
  margin-bottom: 8px;
  border-radius: 16px;
  overflow: hidden;
}

.todo-item {
  --background: rgba(12, 18, 32, 0.9);
  --border-color: transparent;
  --padding-start: 0;
  --inner-padding-end: 0;
  --min-height: 68px;
  border-radius: 16px;
  border: 1px solid rgba(255, 255, 255, 0.06);
  padding: 12px 12px;
  transition: border-color 0.2s, opacity 0.2s;
  backdrop-filter: blur(10px);
}

.todo-item:not(.is-done):hover {
  border-color: rgba(0, 229, 192, 0.2);
}

.todo-item.is-done {
  --background: rgba(8, 12, 22, 0.7);
  border-color: rgba(255, 255, 255, 0.03);
  opacity: 0.5;
}

.item-index {
  font-family: 'Space Mono', monospace;
  font-size: 0.55rem;
  color: #1e2a42;
  width: 24px;
  text-align: right;
  padding-right: 10px;
  flex-shrink: 0;
  align-self: flex-start;
  margin-top: 3px;
}

.todo-label {
  --color: #e8edf5;
}

.todo-title {
  font-family: 'DM Sans', sans-serif !important;
  font-size: 0.9rem !important;
  font-weight: 400 !important;
  color: #d0d8ea !important;
  line-height: 1.35 !important;
  margin-bottom: 4px;
  letter-spacing: -0.01em;
}

.is-done .todo-title {
  text-decoration: line-through;
  color: #2a3550 !important;
}

.todo-desc {
  font-size: 0.73rem !important;
  color: #3a4d6a !important;
  line-height: 1.4 !important;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin-bottom: 5px;
}

.todo-date {
  display: flex;
  align-items: center;
  gap: 4px;
  font-family: 'Space Mono', monospace !important;
  font-size: 0.56rem !important;
  color: #2a3550 !important;
  letter-spacing: 0.03em;
}

.todo-date ion-icon {
  font-size: 9px;
  color: #00e5c0;
  opacity: 0.5;
}

/* ── Item End ── */
.item-end {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 5px;
}

.check-bubble {
  width: 26px;
  height: 26px;
  border-radius: 8px;
  border: 1.5px solid #1e2a42;
  background: transparent;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
  font-size: 13px;
  color: #080c14;
  flex-shrink: 0;
}

.check-bubble.checked {
  background: #00e5c0;
  border-color: #00e5c0;
  box-shadow: 0 0 14px rgba(0, 229, 192, 0.5);
}

.mini-actions {
  display: flex;
}

.mini-actions ion-button {
  --color: #1e2a42;
  --padding-start: 3px;
  --padding-end: 3px;
  font-size: 12px;
}

/* ── Swipe Options ── */
ion-item-option {
  font-size: 18px;
}

ion-item-option[color="danger"] {
  --background: #ff4757;
  --color: #fff;
}

.edit-option {
  --background: #0a1628;
  --color: #00e5c0;
  border: 1px solid rgba(0, 229, 192, 0.2);
}

/* ── Empty State ── */
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 70px 24px;
  text-align: center;
}

.empty-glyph {
  position: relative;
  width: 60px;
  height: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 24px;
}

.empty-ring {
  position: absolute;
  width: 50px;
  height: 50px;
  border-radius: 50%;
  border: 1.5px solid rgba(0, 229, 192, 0.2);
  animation: spin-ring 8s linear infinite;
}

.empty-ring::before {
  content: '';
  position: absolute;
  top: -2px;
  left: 50%;
  transform: translateX(-50%);
  width: 5px;
  height: 5px;
  background: #00e5c0;
  border-radius: 50%;
  box-shadow: 0 0 8px rgba(0, 229, 192, 0.8);
}

.empty-dot {
  width: 8px;
  height: 8px;
  background: rgba(0, 229, 192, 0.3);
  border-radius: 50%;
  border: 1px solid rgba(0, 229, 192, 0.5);
}

@keyframes spin-ring {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.empty-title {
  font-family: 'Syne', sans-serif;
  font-weight: 600;
  font-size: 1rem;
  color: #2a3550;
  margin-bottom: 6px;
  letter-spacing: 0.02em;
}

.empty-sub {
  font-family: 'DM Sans', sans-serif;
  font-size: 0.78rem;
  color: #1e2a3e;
  font-weight: 300;
}

/* ── Modal ── */
ion-modal ion-toolbar {
  --background: #080f1e;
  --border-color: rgba(0, 229, 192, 0.1);
}

.modal-title {
  font-family: 'Syne', sans-serif;
  font-weight: 600;
  font-size: 1rem;
  color: #e8edf5;
  letter-spacing: 0.02em;
}

.modal-content {
  --background: #080f1e;
  padding: 8px 0;
}

.form-group {
  padding: 0 18px;
  margin-bottom: 18px;
}

.form-label {
  display: block;
  font-family: 'Space Mono', monospace;
  font-size: 0.58rem;
  text-transform: uppercase;
  letter-spacing: 0.12em;
  color: #3a4d6a;
  margin-bottom: 8px;
}

.req { color: #00e5c0; }
.opt {
  color: #1e2a42;
  text-transform: none;
  letter-spacing: 0;
  font-size: 0.56rem;
}

.form-input {
  --background: rgba(10, 16, 30, 0.8);
  --color: #d0d8ea;
  --placeholder-color: #1e2a42;
  --padding-start: 14px;
  --padding-end: 14px;
  --padding-top: 12px;
  --padding-bottom: 12px;
  border: 1px solid rgba(255, 255, 255, 0.07);
  border-radius: 12px;
  font-family: 'DM Sans', sans-serif;
  font-size: 0.88rem;
  transition: border-color 0.2s;
}

.form-input:focus-within {
  border-color: rgba(0, 229, 192, 0.4);
  box-shadow: 0 0 0 3px rgba(0, 229, 192, 0.06);
}

.modal-footer {
  display: grid;
  grid-template-columns: 1fr 2fr;
  gap: 10px;
  padding: 10px 18px 36px;
}

.btn-cancel {
  --color: #3a4d6a;
  font-family: 'DM Sans', sans-serif;
  font-weight: 500;
  border: 1px solid rgba(255, 255, 255, 0.07);
  border-radius: 12px;
}

.btn-save {
  --background: linear-gradient(135deg, #00e5c0, #00bfa8);
  --color: #080c14;
  font-family: 'DM Sans', sans-serif;
  font-weight: 700;
  border-radius: 12px;
  letter-spacing: 0.01em;
  box-shadow: 0 0 20px rgba(0, 229, 192, 0.3);
}

.btn-save[disabled] {
  --background: rgba(255, 255, 255, 0.05);
  --color: #1e2a42;
  box-shadow: none;
}
</style>