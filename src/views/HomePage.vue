<template>
  <ion-page class="app-page">
    <ion-content :fullscreen="true" :scroll-events="true">

      <!-- Header -->
      <div class="top-bar">
        <div class="top-bar__inner">
          <div class="wordmark">
            <span class="wordmark__mantap">Lezat</span>
            <span class="wordmark__benar">Resepku</span>
          </div>
          <button class="icon-btn" @click="openAddModal">
            <ion-icon :icon="addOutline"></ion-icon>
          </button>
        </div>
      </div>

      <!-- Filter Strip -->
      <div class="filter-track">
        <button
          v-for="cat in categories"
          :key="cat.id"
          class="fcat"
          :class="{ 'fcat--on': activeCategory === cat.id }"
          @click="activeCategory = cat.id"
        >
          <span class="fcat__icon">{{ cat.emoji }}</span>
          <span class="fcat__label">{{ cat.label }}</span>
        </button>
      </div>

      <!-- List header -->
      <div class="list-head">
        <p class="list-head__label">
          {{ activeCategory === 'all' ? 'Semua' : categories.find(c => c.id === activeCategory)?.label }}
          <em>·</em> {{ filteredRecipes.length }}
        </p>
      </div>

      <!-- Cards -->
      <div class="card-list" v-if="filteredRecipes.length > 0">
        <ion-item-sliding
          v-for="recipe in filteredRecipes"
          :key="recipe.id"
          class="slide-host"
        >
          <ion-item-options side="end">
            <ion-item-option color="danger" expandable @click="confirmDelete(recipe.id)">
              <ion-icon :icon="trashOutline" slot="icon-only"></ion-icon>
            </ion-item-option>
          </ion-item-options>
          <ion-item-options side="start">
            <ion-item-option class="opt-edit" expandable @click="editRecipe(recipe)">
              <ion-icon :icon="createOutline" slot="icon-only"></ion-icon>
            </ion-item-option>
          </ion-item-options>

          <ion-item lines="none" class="slide-item" @click="openDetail(recipe)">
            <div class="rcard">
              <div class="rcard__thumb">{{ recipe.emoji || '🍽️' }}</div>
              <div class="rcard__body">
                <p class="rcard__name">{{ recipe.title }}</p>
                <p v-if="recipe.description" class="rcard__desc">{{ recipe.description }}</p>
                <div class="rcard__meta">
                  <span class="meta-tag">{{ recipe.ingredients.length }} bahan</span>
                  <span class="meta-tag meta-tag--step">{{ recipe.steps.length }} langkah</span>
                  <span v-if="recipe.duration" class="meta-tag meta-tag--time">{{ recipe.duration }} mnt</span>
                  <span v-if="recipe.category" class="meta-tag meta-tag--cat">
                    {{ categories.find(c => c.id === recipe.category)?.label }}
                  </span>
                </div>
              </div>
              <div class="rcard__caret">›</div>
            </div>
          </ion-item>
        </ion-item-sliding>
      </div>

      <!-- Empty -->
      <div v-else class="empty">
        <div class="empty__plate">{{ activeCategory === 'all' ? '🥘' : categories.find(c=>c.id===activeCategory)?.emoji }}</div>
        <p class="empty__msg">{{ activeCategory === 'all' ? 'Dapur masih kosong.' : 'Belum ada di sini.' }}</p>
        <button class="empty__btn" @click="openAddModal">Tambah resep pertama</button>
      </div>

      <div style="height:80px"></div>
    </ion-content>

    <!-- ── Add/Edit Modal ── -->
    <ion-modal
      :is-open="isModalOpen"
      @didDismiss="closeModal"
      :breakpoints="[0,0.9,1]"
      :initial-breakpoint="0.9"
      class="sheet-modal"
    >
      <ion-header>
        <ion-toolbar class="sheet-toolbar">
          <ion-title class="sheet-title">{{ editingRecipe ? 'Edit Resep' : 'Resep Baru' }}</ion-title>
          <ion-buttons slot="end">
            <ion-button @click="closeModal" class="sheet-close">
              <ion-icon :icon="closeOutline" slot="icon-only"></ion-icon>
            </ion-button>
          </ion-buttons>
        </ion-toolbar>
      </ion-header>
      <ion-content class="sheet-body">

        <div class="sf-group sf-emoji-row">
          <div class="emoji-big">{{ formData.emoji || '🍽️' }}</div>
          <div class="emoji-grid">
            <button
              v-for="e in emojiOptions" :key="e"
              class="eg-btn"
              :class="{ 'eg-btn--on': formData.emoji === e }"
              @click="formData.emoji = e"
            >{{ e }}</button>
          </div>
        </div>

        <div class="sf-group">
          <label class="sf-label">Nama Resep <span class="sf-req">*</span></label>
          <ion-input class="sf-input" v-model="formData.title" placeholder="Nasi Goreng Spesial" clearInput></ion-input>
        </div>

        <div class="sf-group">
          <label class="sf-label">Kategori</label>
          <div class="cat-pills">
            <button
              v-for="cat in categories.filter(c=>c.id!=='all')" :key="cat.id"
              class="cat-pill"
              :class="{ 'cat-pill--on': formData.category === cat.id }"
              @click="formData.category = cat.id"
            >{{ cat.emoji }} {{ cat.label }}</button>
          </div>
        </div>

        <div class="sf-group">
          <label class="sf-label">Deskripsi <span class="sf-opt">opsional</span></label>
          <ion-textarea class="sf-input" v-model="formData.description" placeholder="Ceritakan sedikit…" :rows="2" autoGrow></ion-textarea>
        </div>

        <div class="sf-group sf-group--half">
          <label class="sf-label">Waktu <span class="sf-opt">menit</span></label>
          <ion-input class="sf-input" type="number" v-model="formData.duration" placeholder="30"></ion-input>
        </div>

        <div class="sf-group">
          <div class="sf-row-head">
            <label class="sf-label">Bahan-bahan</label>
            <button class="sf-add" @click="addIngredient">+ Tambah</button>
          </div>
          <div v-for="(ing, i) in formData.ingredients" :key="i" class="sf-list-row">
            <span class="sf-bullet">·</span>
            <ion-input class="sf-input sf-input--inline" v-model="formData.ingredients[i]" :placeholder="`Bahan ${i+1}`"></ion-input>
            <button class="sf-rm" @click="removeIngredient(i)">✕</button>
          </div>
          <p v-if="formData.ingredients.length===0" class="sf-hint">Belum ada bahan</p>
        </div>

        <div class="sf-group">
          <div class="sf-row-head">
            <label class="sf-label">Langkah Memasak</label>
            <button class="sf-add" @click="addStep">+ Tambah</button>
          </div>
          <div v-for="(step, i) in formData.steps" :key="i" class="sf-list-row sf-step-row">
            <span class="sf-step-n">{{ i+1 }}</span>
            <ion-textarea class="sf-input sf-input--inline" v-model="formData.steps[i]" :placeholder="`Langkah ${i+1}…`" :rows="2" autoGrow></ion-textarea>
            <button class="sf-rm" @click="removeStep(i)">✕</button>
          </div>
          <p v-if="formData.steps.length===0" class="sf-hint">Belum ada langkah</p>
        </div>

        <div class="sf-footer">
          <button class="sf-btn sf-btn--cancel" @click="closeModal">Batal</button>
          <button class="sf-btn sf-btn--save" :disabled="!formData.title.trim()" @click="saveRecipe">
            {{ editingRecipe ? 'Simpan' : 'Tambah Resep' }}
          </button>
        </div>
      </ion-content>
    </ion-modal>

    <!-- ── Detail Modal ── -->
    <ion-modal
      :is-open="isDetailOpen"
      @didDismiss="closeDetail"
      :breakpoints="[0,0.9,1]"
      :initial-breakpoint="0.9"
      class="sheet-modal"
    >
      <template v-if="detailRecipe">
        <ion-header>
          <ion-toolbar class="sheet-toolbar">
            <ion-buttons slot="start">
              <ion-button @click="closeDetail" class="sheet-close">
                <ion-icon :icon="closeOutline" slot="icon-only"></ion-icon>
              </ion-button>
            </ion-buttons>
            <ion-buttons slot="end">
              <ion-button @click="editFromDetail" class="sheet-close">
                <ion-icon :icon="createOutline" slot="icon-only"></ion-icon>
              </ion-button>
            </ion-buttons>
          </ion-toolbar>
        </ion-header>
        <ion-content class="sheet-body">
          <div class="det-hero">
            <div class="det-emoji">{{ detailRecipe.emoji || '🍽️' }}</div>
            <h1 class="det-title">{{ detailRecipe.title }}</h1>
            <p v-if="detailRecipe.description" class="det-desc">{{ detailRecipe.description }}</p>
            <div class="det-chips">
              <span class="dchip">{{ detailRecipe.ingredients.length }} bahan</span>
              <span class="dchip">{{ detailRecipe.steps.length }} langkah</span>
              <span v-if="detailRecipe.duration" class="dchip dchip--time">⏱ {{ detailRecipe.duration }} mnt</span>
              <span v-if="detailRecipe.category" class="dchip dchip--cat">
                {{ categories.find(c => c.id === detailRecipe!.category)?.emoji }}
                {{ categories.find(c => c.id === detailRecipe!.category)?.label }}
              </span>
            </div>
          </div>

          <div class="det-section" v-if="detailRecipe.ingredients.length > 0">
            <p class="det-sec-title">Bahan-bahan</p>
            <ul class="ing-list">
              <li v-for="(ing,i) in detailRecipe.ingredients" :key="i" class="ing-item">
                <span class="ing-mark"></span>{{ ing }}
              </li>
            </ul>
          </div>

          <div class="det-section" v-if="detailRecipe.steps.length > 0">
            <p class="det-sec-title">Cara Membuat</p>
            <ol class="step-list">
              <li v-for="(step,i) in detailRecipe.steps" :key="i" class="step-item">
                <span class="step-n">{{ i+1 }}</span>
                <p class="step-txt">{{ step }}</p>
              </li>
            </ol>
          </div>

          <div style="height:60px"></div>
        </ion-content>
      </template>
    </ion-modal>

    <ion-alert
      :is-open="deleteTargetId !== null"
      header="Hapus Resep?"
      message="Resep ini akan dihapus permanen."
      :buttons="deleteAlertButtons"
      @didDismiss="deleteTargetId = null"
    ></ion-alert>
  </ion-page>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import {
  IonPage, IonContent, IonButtons, IonButton, IonIcon,
  IonItem, IonItemSliding, IonItemOptions, IonItemOption,
  IonModal, IonHeader, IonToolbar, IonTitle,
  IonInput, IonTextarea, IonAlert,
} from '@ionic/vue';
import { addOutline, createOutline, trashOutline, closeOutline } from 'ionicons/icons';

interface Recipe {
  id: number;
  title: string;
  description?: string;
  emoji?: string;
  category?: string;
  duration?: number;
  ingredients: string[];
  steps: string[];
  createdAt?: number;
}

const categories = [
  { id: 'all',     emoji: '🍽️', label: 'Semua' },
  { id: 'nasi',    emoji: '🍚', label: 'Nasi' },
  { id: 'mie',     emoji: '🍜', label: 'Mie' },
  { id: 'ayam',    emoji: '🍗', label: 'Ayam' },
  { id: 'daging',  emoji: '🥩', label: 'Daging' },
  { id: 'sayur',   emoji: '🥗', label: 'Sayur' },
  { id: 'seafood', emoji: '🦐', label: 'Seafood' },
  { id: 'camilan', emoji: '🧁', label: 'Camilan' },
  { id: 'minuman', emoji: '☕', label: 'Minuman' },
];

const emojiOptions = ['🍚','🍜','🍲','🍗','🥩','🥗','🍳','🥘','🍱','🧆','🍝','🥞','🍰','🧁','☕','🦐','🐟','🥦'];

const recipes = ref<Recipe[]>([]);
const activeCategory = ref('all');
const isModalOpen = ref(false);
const isDetailOpen = ref(false);
const editingRecipe = ref<Recipe | null>(null);
const detailRecipe = ref<Recipe | null>(null);
const deleteTargetId = ref<number | null>(null);

const filteredRecipes = computed(() =>
  activeCategory.value === 'all'
    ? recipes.value
    : recipes.value.filter(r => r.category === activeCategory.value)
);

const emptyForm = () => ({
  title: '', description: '', emoji: '', category: '' as string,
  duration: undefined as number | undefined,
  ingredients: [] as string[], steps: [] as string[],
});

const formData = ref(emptyForm());

const load = () => {
  const s = localStorage.getItem('mb-recipes-v2');
  if (s) recipes.value = JSON.parse(s);
};
const persist = () => localStorage.setItem('mb-recipes-v2', JSON.stringify(recipes.value));

const openAddModal = () => { editingRecipe.value = null; formData.value = emptyForm(); isModalOpen.value = true; };
const editRecipe = (r: Recipe) => {
  editingRecipe.value = r;
  formData.value = {
    title: r.title, description: r.description || '', emoji: r.emoji || '',
    category: r.category || '', duration: r.duration,
    ingredients: [...r.ingredients], steps: [...r.steps],
  };
  isModalOpen.value = true;
};
const editFromDetail = () => { if (!detailRecipe.value) return; closeDetail(); editRecipe(detailRecipe.value); };
const closeModal = () => { isModalOpen.value = false; };
const saveRecipe = () => {
  if (!formData.value.title.trim()) return;
  const p: Recipe = {
    id: editingRecipe.value?.id ?? Date.now(),
    title: formData.value.title,
    description: formData.value.description,
    emoji: formData.value.emoji,
    category: formData.value.category || undefined,
    duration: formData.value.duration ? Number(formData.value.duration) : undefined,
    ingredients: formData.value.ingredients.filter(i => i.trim()),
    steps: formData.value.steps.filter(s => s.trim()),
    createdAt: editingRecipe.value?.createdAt ?? Date.now(),
  };
  if (editingRecipe.value) {
    const idx = recipes.value.findIndex(r => r.id === editingRecipe.value!.id);
    if (idx !== -1) recipes.value[idx] = p;
    if (detailRecipe.value?.id === p.id) detailRecipe.value = p;
  } else { recipes.value.unshift(p); }
  persist(); closeModal();
};
const openDetail = (r: Recipe) => { detailRecipe.value = r; isDetailOpen.value = true; };
const closeDetail = () => { isDetailOpen.value = false; };
const addIngredient = () => formData.value.ingredients.push('');
const removeIngredient = (i: number) => formData.value.ingredients.splice(i, 1);
const addStep = () => formData.value.steps.push('');
const removeStep = (i: number) => formData.value.steps.splice(i, 1);
const confirmDelete = (id: number) => { deleteTargetId.value = id; };
const deleteAlertButtons = [
  { text: 'Batal', role: 'cancel', handler: () => { deleteTargetId.value = null; } },
  { text: 'Hapus', role: 'destructive', handler: () => {
    if (deleteTargetId.value === null) return;
    recipes.value = recipes.value.filter(r => r.id !== deleteTargetId.value);
    persist(); deleteTargetId.value = null;
  }},
];

onMounted(load);
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Instrument+Serif:ital@0;1&family=Instrument+Sans:wght@400;500;600&display=swap');

/* ── Tokens ── */
:host {
  --ink:      #1a1208;
  --ink-2:    #5a4e3a;
  --ink-3:    #a89880;
  --paper:    #faf7f2;
  --paper-2:  #f2ede4;
  --paper-3:  #e8e0d2;
  --rust:     #c0472b;
  --rust-bg:  #fbede9;
  --sage:     #4a6741;
  --warm:     #c07a2b;
  --warm-bg:  #fdf3e3;
  --r:        14px;
  --r-sm:     8px;
  --font-serif: 'Instrument Serif', Georgia, serif;
  --font-sans:  'Instrument Sans', system-ui, sans-serif;
}

/* ── Reset & Base ── */
ion-content {
  --background: #faf7f2;
  --color: #1a1208;
  font-family: 'Instrument Sans', system-ui, sans-serif;
  color: #1a1208;
}

ion-input {
  --color: #1a1208 !important;
  --placeholder-color: #c8bfb0 !important;
  --background: #ffffff !important;
  color: #1a1208 !important;
}

ion-textarea {
  --color: #1a1208 !important;
  --placeholder-color: #c8bfb0 !important;
  --background: #ffffff !important;
  color: #1a1208 !important;
}

ion-item {
  --background: transparent;
  --color: #1a1208;
}

ion-modal { --color: #1a1208; }

.app-page { background: #faf7f2; }

/* ── Top Bar ── */
.top-bar {
  padding-top: max(env(safe-area-inset-top), 48px);
  background: #faf7f2;
  position: relative;
  z-index: 20;
}

.top-bar__inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 20px 14px;
  border-bottom: 1.5px solid #1a1208;
}

.wordmark {
  font-family: 'Instrument Serif', Georgia, serif;
  line-height: 1;
}
.wordmark__mantap {
  font-size: 28px;
  font-style: italic;
  color: #1a1208;
  letter-spacing: -0.5px;
}
.wordmark__benar {
  font-size: 28px;
  font-style: normal;
  color: #c0472b;
  letter-spacing: -0.5px;
  margin-left: 3px;
}

.icon-btn {
  width: 38px; height: 38px;
  border-radius: 50%;
  background: #1a1208;
  border: none;
  display: flex; align-items: center; justify-content: center;
  color: #faf7f2;
  font-size: 18px;
  cursor: pointer;
  transition: transform 0.15s;
}
.icon-btn:active { transform: scale(0.9); }

/* ── Filter Track ── */
.filter-track {
  display: flex;
  overflow-x: auto;
  scrollbar-width: none;
  border-bottom: 1.5px solid #1a1208;
  background: #faf7f2;
  position: sticky;
  top: 0;
  z-index: 10;
}
.filter-track::-webkit-scrollbar { display: none; }

.fcat {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
  padding: 10px 14px 8px;
  border: none;
  border-right: 1px solid #e8e0d2;
  background: transparent;
  cursor: pointer;
  flex-shrink: 0;
  transition: background 0.15s;
  position: relative;
}
.fcat:last-child { border-right: none; }
.fcat__icon { font-size: 16px; line-height: 1; }
.fcat__label {
  font-family: 'Instrument Sans', system-ui, sans-serif;
  font-size: 9px;
  font-weight: 600;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: #a89880;
}
.fcat--on { background: #1a1208; }
.fcat--on .fcat__label { color: #faf7f2; }
.fcat--on::after {
  content: '';
  position: absolute;
  bottom: -1.5px; left: 0; right: 0;
  height: 2.5px;
  background: #c0472b;
}

/* ── List Head ── */
.list-head { padding: 14px 20px 4px; }
.list-head__label {
  font-family: 'Instrument Serif', Georgia, serif;
  font-style: italic;
  font-size: 15px;
  color: #a89880;
  margin: 0;
}
.list-head__label em { font-style: normal; margin: 0 4px; }

/* ── Card List ── */
.card-list { padding: 8px 16px 0; }

.slide-host {
  margin-bottom: 8px;
  border-radius: 14px;
  overflow: hidden;
}

.slide-item {
  --background: transparent;
  --inner-padding-start: 0;
  --inner-padding-end: 0;
  --padding-start: 0;
  --padding-end: 0;
  --min-height: auto;
}

.rcard {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px;
  background: #ffffff;
  border: 1.5px solid #e8e0d2;
  border-radius: 14px;
  cursor: pointer;
  width: 100%;
  transition: background 0.15s, border-color 0.15s;
}
.rcard:active { background: #f2ede4; border-color: #c8bfb0; }

.rcard__thumb {
  width: 48px; height: 48px;
  border-radius: 10px;
  background: #f2ede4;
  border: 1.5px solid #e8e0d2;
  display: flex; align-items: center; justify-content: center;
  font-size: 22px; flex-shrink: 0;
}

.rcard__body { flex: 1; overflow: hidden; }

.rcard__name {
  font-family: 'Instrument Serif', Georgia, serif;
  font-size: 17px;
  font-style: italic;
  color: #1a1208;
  margin: 0 0 2px;
  white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
}

.rcard__desc {
  font-size: 12px; color: #a89880;
  margin: 0 0 8px;
  white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
}

.rcard__meta { display: flex; gap: 5px; flex-wrap: wrap; }

.meta-tag {
  font-family: 'Instrument Sans', system-ui, sans-serif;
  font-size: 10px; font-weight: 600;
  letter-spacing: 0.04em; text-transform: uppercase;
  padding: 2px 8px; border-radius: 4px;
  background: #f2ede4; color: #a89880; border: 1px solid #e8e0d2;
}
.meta-tag--step { background: #eef4ee; color: #4a6741; border-color: #d4e4d4; }
.meta-tag--time { background: #fdf3e3; color: #c07a2b; border-color: #f0ddb8; }
.meta-tag--cat  { background: #fbede9; color: #c0472b; border-color: #f0cfc7; }

.rcard__caret { font-size: 20px; color: #c8bfb0; flex-shrink: 0; }

/* Swipe */
ion-item-option[color="danger"] { --background: #c0472b; --color: #fff; }
.opt-edit { --background: #1a1208; --color: #faf7f2; }

/* ── Empty ── */
.empty {
  display: flex; flex-direction: column;
  align-items: center;
  padding: 80px 24px 40px; text-align: center;
}
.empty__plate { font-size: 3.5rem; margin-bottom: 12px; opacity: 0.5; }
.empty__msg {
  font-family: 'Instrument Serif', Georgia, serif;
  font-size: 20px; font-style: italic;
  color: #a89880; margin-bottom: 24px;
}
.empty__btn {
  font-family: 'Instrument Sans', system-ui, sans-serif;
  font-size: 13px; font-weight: 600;
  padding: 12px 28px; border-radius: 8px;
  background: #1a1208; color: #faf7f2;
  border: none; cursor: pointer;
}

/* ── Sheet Modal ── */
ion-modal.sheet-modal {
  --background: #faf7f2;
  --backdrop-opacity: 0.35;
}
ion-modal.sheet-modal::part(content) {
  border-radius: 20px 20px 0 0;
  border-top: 1.5px solid #1a1208;
}

.sheet-toolbar {
  --background: #faf7f2;
  --border-color: #e8e0d2;
  --color: #1a1208;
}
.sheet-title {
  font-family: 'Instrument Serif', Georgia, serif !important;
  font-size: 18px !important; font-style: italic !important;
  color: #1a1208 !important; font-weight: 400 !important;
}
.sheet-close { --color: #1a1208; }

.sheet-body {
  --background: #faf7f2;
  --color: #1a1208;
  color: #1a1208;
  font-family: 'Instrument Sans', system-ui, sans-serif;
}

/* ── Form ── */
.sf-group { padding: 0 20px; margin-bottom: 24px; }
.sf-group--half { max-width: 52%; }

.sf-label {
  display: block;
  font-size: 10px; font-weight: 600;
  text-transform: uppercase; letter-spacing: 0.12em;
  color: #a89880; margin-bottom: 8px;
}
.sf-req { color: #c0472b; }
.sf-opt {
  text-transform: none; letter-spacing: 0;
  font-size: 9px; font-weight: 400;
  color: #c8bfb0; margin-left: 4px;
}

.sf-input {
  --color: #1a1208 !important;
  --placeholder-color: #c8bfb0 !important;
  --background: #ffffff !important;
  --padding-start: 13px; --padding-end: 13px;
  --padding-top: 11px; --padding-bottom: 11px;
  border: 1.5px solid #e8e0d2;
  border-radius: 8px;
  font-family: 'Instrument Sans', system-ui, sans-serif;
  font-size: 14px; color: #1a1208;
  transition: border-color 0.15s;
}
.sf-input:focus-within { border-color: #1a1208; }
.sf-input--inline { flex: 1; }

.sf-emoji-row { display: flex; align-items: center; gap: 14px; }
.emoji-big {
  font-size: 2.8rem; width: 60px; height: 60px;
  display: flex; align-items: center; justify-content: center;
  background: #f2ede4; border: 1.5px solid #e8e0d2;
  border-radius: 14px; flex-shrink: 0;
}
.emoji-grid { display: flex; flex-wrap: wrap; gap: 6px; flex: 1; }
.eg-btn {
  font-size: 1.2rem; width: 34px; height: 34px;
  display: flex; align-items: center; justify-content: center;
  border-radius: 8px; border: 1.5px solid transparent;
  background: #f2ede4; cursor: pointer; transition: all 0.12s;
}
.eg-btn--on { background: #fff; border-color: #1a1208; }

.cat-pills { display: flex; flex-wrap: wrap; gap: 7px; }
.cat-pill {
  font-family: 'Instrument Sans', system-ui, sans-serif;
  font-size: 11px; font-weight: 600;
  padding: 6px 12px; border-radius: 6px;
  background: #f2ede4; border: 1.5px solid #e8e0d2;
  color: #5a4e3a; cursor: pointer; transition: all 0.12s;
}
.cat-pill--on { background: #1a1208; border-color: #1a1208; color: #faf7f2; }

.sf-row-head { display: flex; align-items: center; justify-content: space-between; margin-bottom: 8px; }
.sf-add {
  font-family: 'Instrument Sans', system-ui, sans-serif;
  font-size: 11px; font-weight: 600;
  color: #c0472b; background: none; border: none; cursor: pointer;
}
.sf-list-row { display: flex; align-items: flex-start; gap: 8px; margin-bottom: 8px; }
.sf-step-row { align-items: center; }
.sf-bullet { font-size: 22px; color: #c8bfb0; margin-top: 8px; flex-shrink: 0; }
.sf-step-n {
  font-size: 10px; font-weight: 700; color: #faf7f2;
  background: #1a1208; width: 22px; height: 22px;
  border-radius: 50%;
  display: flex; align-items: center; justify-content: center;
  flex-shrink: 0; margin-top: 11px;
}
.sf-rm { font-size: 11px; color: #c8bfb0; background: none; border: none; cursor: pointer; padding: 4px; margin-top: 10px; transition: color 0.12s; }
.sf-rm:hover { color: #c0472b; }
.sf-hint {
  font-size: 12px; color: #c8bfb0; text-align: center;
  padding: 14px 0; border: 1.5px dashed #e8e0d2;
  border-radius: 8px; margin: 0;
}

.sf-footer { display: grid; grid-template-columns: auto 1fr; gap: 10px; padding: 12px 20px 48px; }
.sf-btn {
  font-family: 'Instrument Sans', system-ui, sans-serif;
  font-size: 13px; font-weight: 600;
  padding: 13px 20px; border-radius: 8px;
  border: none; cursor: pointer; transition: all 0.15s;
}
.sf-btn--cancel { background: #f2ede4; color: #a89880; border: 1.5px solid #e8e0d2; }
.sf-btn--save { background: #1a1208; color: #faf7f2; }
.sf-btn--save:disabled { background: #e8e0d2; color: #c8bfb0; }
.sf-btn--save:active:not(:disabled) { background: #2d1e0f; }

/* ── Detail ── */
.det-hero { padding: 24px 20px 20px; border-bottom: 1.5px solid #e8e0d2; }
.det-emoji { font-size: 3.5rem; line-height: 1; margin-bottom: 12px; }
.det-title {
  font-family: 'Instrument Serif', Georgia, serif;
  font-size: 28px; font-style: italic; color: #1a1208;
  margin: 0 0 6px; line-height: 1.2;
}
.det-desc { font-size: 14px; color: #a89880; line-height: 1.6; margin: 0 0 14px; }
.det-chips { display: flex; gap: 6px; flex-wrap: wrap; }
.dchip {
  font-family: 'Instrument Sans', system-ui, sans-serif;
  font-size: 10px; font-weight: 600;
  text-transform: uppercase; letter-spacing: 0.05em;
  padding: 4px 10px; border-radius: 5px;
  background: #f2ede4; border: 1px solid #e8e0d2; color: #a89880;
}
.dchip--time { background: #fdf3e3; color: #c07a2b; border-color: #f0ddb8; }
.dchip--cat  { background: #fbede9; color: #c0472b; border-color: #f0cfc7; }

.det-section { padding: 22px 20px 0; }
.det-sec-title {
  font-family: 'Instrument Sans', system-ui, sans-serif;
  font-size: 10px; font-weight: 600;
  text-transform: uppercase; letter-spacing: 0.14em; color: #c8bfb0;
  margin: 0 0 14px;
  display: flex; align-items: center; gap: 10px;
}
.det-sec-title::after { content: ''; flex: 1; height: 1px; background: #e8e0d2; }

.ing-list { list-style: none; padding: 0; margin: 0; }
.ing-item {
  display: flex; align-items: center; gap: 12px;
  font-size: 15px; color: #1a1208; padding: 10px 0;
  border-bottom: 1px solid #f0ece4;
}
.ing-item:last-child { border-bottom: none; }
.ing-mark { width: 6px; height: 6px; background: #c0472b; border-radius: 50%; flex-shrink: 0; }

.step-list { list-style: none; padding: 0; margin: 0; }
.step-item { display: flex; gap: 12px; margin-bottom: 18px; align-items: flex-start; }
.step-n {
  font-size: 11px; font-weight: 700; color: #faf7f2; background: #1a1208;
  min-width: 26px; height: 26px; border-radius: 50%;
  display: flex; align-items: center; justify-content: center;
  flex-shrink: 0; margin-top: 1px;
}
.step-txt { font-size: 15px; color: #1a1208; line-height: 1.65; font-weight: 400; margin: 0; padding-top: 3px; }
</style>