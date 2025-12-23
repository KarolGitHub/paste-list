<template>
  <section class="panel">
    <div class="manager">
      <div class="tabs" v-if="checklists.length">
        <div v-for="list in checklists" :key="list.id" class="tab" :class="{ active: list.id === activeId }"
          role="button" tabindex="0" @click="setActive(list.id)" @keydown.enter.prevent="setActive(list.id)"
          @keydown.space.prevent="setActive(list.id)">
          <span class="tab-title">{{ list.title || "Untitled" }}</span>
          <span class="tab-count">{{ doneCount(list) }}/{{ list.items.length }}</span>
          <button class="tab-remove" type="button" @click.stop="removeChecklist(list.id)">
            ✕
          </button>
        </div>
      </div>
      <div class="new-card">
        <label class="label" for="title-input">New checklist title</label>
        <input id="title-input" v-model="newTitle" class="text-input" type="text" placeholder="Sprint prep" />
        <label class="label" for="seed-area">Seed items (optional, one per line)</label>
        <textarea id="seed-area" v-model="newDraft" placeholder="Write agenda&#10;Send invites&#10;Book room"
          rows="3" />
        <button class="btn primary full" type="button" @click="createChecklist">
          Add checklist
        </button>
      </div>
    </div>

    <div v-if="activeChecklist" class="active-section">
      <div class="input-card">
        <label class="label" for="active-title">Checklist title</label>
        <input id="active-title" v-model="activeTitle" class="text-input" type="text"
          placeholder="Untitled checklist" />

        <label class="label" for="paste-area">Paste items (one per line)</label>
        <textarea id="paste-area" v-model="activeDraft" placeholder="Milk&#10;Eggs&#10;Call the dentist" rows="4" />
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
    </div>

    <div v-else class="empty">
      <p>No checklists yet. Create one to begin.</p>
    </div>
  </section>
</template>

<script setup>
import { computed, onMounted, ref, watch } from "vue";

const STORAGE_KEY = "paste-checklists";
const LEGACY_KEY = "paste-checklist-items";

const checklists = ref([]);
const activeId = ref(null);

const newTitle = ref("");
const newDraft = ref("");
const activeDraft = ref("");

const parsedNewDraft = computed(() =>
  newDraft.value
    .split(/\r?\n/)
    .map((line) => line.trim())
    .filter(Boolean)
);

const parsedActiveDraft = computed(() =>
  activeDraft.value
    .split(/\r?\n/)
    .map((line) => line.trim())
    .filter(Boolean)
);

const activeChecklist = computed(() =>
  checklists.value.find((list) => list.id === activeId.value)
);

const items = computed(() => activeChecklist.value?.items ?? []);

const completedCount = computed(
  () => items.value.filter((item) => item.done).length
);
const remainingCount = computed(() => items.value.length - completedCount.value);

const activeTitle = computed({
  get: () => activeChecklist.value?.title ?? "",
  set: (nextTitle) => {
    if (!activeChecklist.value) return;
    const value = nextTitle.trim();
    updateChecklist(activeChecklist.value.id, (list) => ({
      ...list,
      title: value,
    }));
  },
});

const doneCount = (list) => list.items.filter((item) => item.done).length;

const setActive = (id) => {
  activeId.value = id;
};

const updateChecklist = (id, updater) => {
  checklists.value = checklists.value.map((list) =>
    list.id === id ? updater(list) : list
  );
};

const createChecklist = () => {
  const title =
    newTitle.value.trim() || `Checklist ${checklists.value.length + 1}`;
  const seededItems = parsedNewDraft.value.map((text) => ({
    id: crypto.randomUUID(),
    text,
    done: false,
  }));

  const id = crypto.randomUUID();
  checklists.value = [
    ...checklists.value,
    { id, title, items: seededItems },
  ];
  activeId.value = id;
  newTitle.value = "";
  newDraft.value = "";
  activeDraft.value = "";
};

const addItems = () => {
  if (!activeChecklist.value || !parsedActiveDraft.value.length) return;

  const newItems = parsedActiveDraft.value.map((text) => ({
    id: crypto.randomUUID(),
    text,
    done: false,
  }));

  updateChecklist(activeChecklist.value.id, (list) => ({
    ...list,
    items: [...list.items, ...newItems],
  }));

  activeDraft.value = "";
};

const toggleItem = (id) => {
  if (!activeChecklist.value) return;
  updateChecklist(activeChecklist.value.id, (list) => ({
    ...list,
    items: list.items.map((item) =>
      item.id === id ? { ...item, done: !item.done } : item
    ),
  }));
};

const removeItem = (id) => {
  if (!activeChecklist.value) return;
  updateChecklist(activeChecklist.value.id, (list) => ({
    ...list,
    items: list.items.filter((item) => item.id !== id),
  }));
};

const toggleAll = (state) => {
  if (!activeChecklist.value) return;
  updateChecklist(activeChecklist.value.id, (list) => ({
    ...list,
    items: list.items.map((item) => ({ ...item, done: state })),
  }));
};

const clearAll = () => {
  if (!activeChecklist.value) return;
  updateChecklist(activeChecklist.value.id, (list) => ({
    ...list,
    items: [],
  }));
  activeDraft.value = "";
};

const removeChecklist = (id) => {
  checklists.value = checklists.value.filter((list) => list.id !== id);
  if (activeId.value === id) {
    activeId.value = checklists.value[0]?.id ?? null;
  }
};

const ensureDefault = () => {
  if (checklists.value.length) return;
  const id = crypto.randomUUID();
  checklists.value = [{ id, title: "Checklist 1", items: [] }];
  activeId.value = id;
};

const migrateLegacy = () => {
  const legacy = localStorage.getItem(LEGACY_KEY);
  if (!legacy) return false;

  try {
    const parsed = JSON.parse(legacy);
    if (Array.isArray(parsed)) {
      const id = crypto.randomUUID();
      checklists.value = [{ id, title: "Checklist 1", items: parsed }];
      activeId.value = id;
      localStorage.removeItem(LEGACY_KEY);
      return true;
    }
  } catch (err) {
    console.error("Failed to migrate legacy checklist", err);
  }
  return false;
};

onMounted(() => {
  try {
    const stored = localStorage.getItem(STORAGE_KEY);
    if (stored) {
      const parsed = JSON.parse(stored);
      if (Array.isArray(parsed?.checklists)) {
        checklists.value = parsed.checklists;
        activeId.value = parsed.activeId ?? parsed.checklists[0]?.id ?? null;
        return;
      }
      // fallback to array shape from early versions
      if (Array.isArray(parsed)) {
        const id = crypto.randomUUID();
        checklists.value = [{ id, title: "Checklist 1", items: parsed }];
        activeId.value = id;
        return;
      }
    }
  } catch (err) {
    console.error("Failed to read saved checklists", err);
  }

  const migrated = migrateLegacy();
  if (!migrated) {
    ensureDefault();
  }
});

watch(
  [checklists, activeId],
  ([nextChecklists, nextActive]) => {
    localStorage.setItem(
      STORAGE_KEY,
      JSON.stringify({ checklists: nextChecklists, activeId: nextActive })
    );
  },
  { deep: true }
);
</script>

<style scoped>
.panel {
  display: grid;
  gap: 18px;
}

.manager {
  display: grid;
  gap: 12px;
}

.tabs {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.tab {
  position: relative;
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  border-radius: 12px;
  border: 1px solid #e2e8f0;
  background: #fff;
  cursor: pointer;
  transition: border-color 120ms ease, box-shadow 120ms ease;
}

.tab.active {
  border-color: #2563eb;
  box-shadow: 0 6px 18px rgba(37, 99, 235, 0.18);
}

.tab-title {
  font-weight: 700;
}

.tab-count {
  font-size: 13px;
  color: #475569;
}

.tab-remove {
  border: none;
  background: transparent;
  color: #0f172a;
  cursor: pointer;
  padding: 4px;
  font-size: 14px;
}

.new-card {
  border: 1px dashed #cbd5e1;
  border-radius: 12px;
  padding: 12px;
  background: #f8fafc;
  display: grid;
  gap: 8px;
}

.input-card,
.list-card {
  border: 1px solid #e2e8f0;
  border-radius: 14px;
  padding: 16px;
  background: #f8fafc;
}

.active-section {
  display: grid;
  gap: 14px;
}

.label {
  display: block;
  font-weight: 600;
  margin-bottom: 6px;
}

.text-input,
textarea {
  width: 100%;
  border: 1px solid #cbd5e1;
  border-radius: 10px;
  padding: 10px 12px;
  font: inherit;
  background: #fff;
}

textarea {
  resize: vertical;
  min-height: 120px;
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

.btn.full {
  width: 100%;
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
