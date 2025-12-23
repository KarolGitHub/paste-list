<template>
  <section class="panel">
    <div class="input-card">
      <label class="label" for="paste-area">Paste items (one per line)</label>
      <textarea id="paste-area" v-model="draft" placeholder="Milk&#10;Eggs&#10;Call the dentist" rows="4" />
      <div class="actions">
        <button class="btn secondary" type="button" @click="clearAll">
          Clear list
        </button>
        <button class="btn primary" type="button" @click="addItems">
          Add items
        </button>
      </div>
    </div>

    <div class="list-card" v-if="items.length">
      <div class="summary">
        <div>
          <p class="count">{{ completedCount }} done</p>
          <p class="count subtle">{{ remainingCount }} remaining</p>
        </div>
        <button class="link" type="button" @click="toggleAll(false)">
          Uncheck all
        </button>
      </div>

      <ul class="list" role="list">
        <li v-for="item in items" :key="item.id" class="row">
          <label class="row-content">
            <input type="checkbox" :checked="item.done" @change="toggleItem(item.id)" />
            <span :class="['text', { done: item.done }]">{{ item.text }}</span>
          </label>
          <button class="icon-btn" type="button" @click="removeItem(item.id)">
            ✕
          </button>
        </li>
      </ul>
    </div>

    <div v-else class="empty">
      <p>No items yet. Paste a few lines to start your checklist.</p>
    </div>
  </section>
</template>

<script setup>
import { computed, onMounted, ref, watch } from "vue";

const STORAGE_KEY = "paste-checklist-items";
const draft = ref("");
const items = ref([]);

const parsedDraft = computed(() =>
  draft.value
    .split(/\r?\n/)
    .map((line) => line.trim())
    .filter(Boolean)
);

const completedCount = computed(
  () => items.value.filter((item) => item.done).length
);
const remainingCount = computed(() => items.value.length - completedCount.value);

const addItems = () => {
  if (!parsedDraft.value.length) return;

  const newItems = parsedDraft.value.map((text) => ({
    id: crypto.randomUUID(),
    text,
    done: false,
  }));

  items.value = [...items.value, ...newItems];
  draft.value = "";
};

const toggleItem = (id) => {
  items.value = items.value.map((item) =>
    item.id === id ? { ...item, done: !item.done } : item
  );
};

const removeItem = (id) => {
  items.value = items.value.filter((item) => item.id !== id);
};

const toggleAll = (state) => {
  items.value = items.value.map((item) => ({ ...item, done: state }));
};

const clearAll = () => {
  items.value = [];
  draft.value = "";
};

onMounted(() => {
  try {
    const stored = localStorage.getItem(STORAGE_KEY);
    if (stored) {
      const parsed = JSON.parse(stored);
      if (Array.isArray(parsed)) {
        items.value = parsed;
      }
    }
  } catch (err) {
    console.error("Failed to read saved checklist", err);
  }
});

watch(
  items,
  (next) => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(next));
  },
  { deep: true }
);
</script>

<style scoped>
.panel {
  display: grid;
  gap: 18px;
}

.input-card,
.list-card {
  border: 1px solid #e2e8f0;
  border-radius: 14px;
  padding: 16px;
  background: #f8fafc;
}

.label {
  display: block;
  font-weight: 600;
  margin-bottom: 8px;
}

textarea {
  width: 100%;
  border: 1px solid #cbd5e1;
  border-radius: 10px;
  padding: 12px;
  font: inherit;
  resize: vertical;
  min-height: 120px;
  background: #fff;
}

.actions {
  margin-top: 10px;
  display: flex;
  gap: 8px;
  justify-content: flex-end;
}

.btn {
  border: 1px solid transparent;
  border-radius: 10px;
  padding: 10px 14px;
  font-weight: 700;
  cursor: pointer;
  font-size: 14px;
  transition: transform 120ms ease, box-shadow 120ms ease;
}

.btn:active {
  transform: translateY(1px);
}

.primary {
  background: linear-gradient(135deg, #2563eb, #1d4ed8);
  color: white;
  box-shadow: 0 10px 30px rgba(37, 99, 235, 0.25);
}

.secondary {
  background: #e2e8f0;
  color: #0f172a;
}

.list-card {
  background: #ffffff;
}

.summary {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.count {
  margin: 0;
  font-weight: 700;
}

.count.subtle {
  color: #64748b;
  font-weight: 600;
}

.list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: grid;
  gap: 8px;
}

.row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 10px 12px;
}

.row-content {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  cursor: pointer;
}

input[type="checkbox"] {
  width: 18px;
  height: 18px;
  accent-color: #2563eb;
}

.text {
  flex: 1;
}

.text.done {
  text-decoration: line-through;
  color: #94a3b8;
}

.icon-btn {
  border: none;
  background: transparent;
  color: #0f172a;
  cursor: pointer;
  font-size: 14px;
}

.link {
  background: transparent;
  border: none;
  color: #2563eb;
  font-weight: 700;
  cursor: pointer;
}

.empty {
  text-align: center;
  color: #475569;
  border: 1px dashed #cbd5e1;
  border-radius: 14px;
  padding: 20px;
  background: #f8fafc;
}

@media (max-width: 640px) {
  .actions {
    flex-direction: column;
    align-items: stretch;
  }

  .row {
    align-items: flex-start;
  }
}
</style>
