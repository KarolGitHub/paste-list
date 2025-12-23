<template>
  <section class="panel">
    <div class="manager">
      <div class="tab-bar" v-if="checklists.length">
        <div class="tabs">
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
        <button class="btn primary add-btn" type="button" aria-label="New checklist" @click="openModal">
          New checklist
        </button>
      </div>
      <div v-else class="empty-bar">
        <p>No checklists yet.</p>
        <button class="btn primary add-btn" type="button" aria-label="Create checklist" @click="openModal">
          Create one
        </button>
      </div>
    </div>

    <div v-if="showModal" class="modal-backdrop" @click.self="closeModal" @touchstart.self="closeModal">
      <div class="modal">
        <div class="modal-head">
          <h3>Create checklist</h3>
          <button class="icon-btn" type="button" @click="closeModal">✕</button>
        </div>
        <label class="label" for="title-input">Title</label>
        <input id="title-input" v-model="newTitle" class="text-input" type="text" placeholder="Sprint prep" />
        <div class="inline-inputs">
          <label class="label" for="seed-sep">Separator</label>
          <select id="seed-sep" v-model="newSeparator" class="select-input">
            <option value="newline">New line (default)</option>
            <option value="comma">Comma</option>
            <option value="semicolon">Semicolon</option>
            <option value="custom">Custom</option>
          </select>
          <input v-if="newSeparator === 'custom'" v-model="customNewSeparator" class="text-input compact" type="text"
            maxlength="5" placeholder="|" />
        </div>
        <label class="label" for="seed-area">Seed items (optional)</label>
        <textarea id="seed-area" v-model="newDraft" placeholder="Write agenda&#10;Send invites&#10;Book room"
          rows="3" />
        <div class="modal-actions">
          <button class="btn secondary" type="button" @click="closeModal">Cancel</button>
          <button class="btn primary" type="button" @click="createChecklist">
            Add checklist
          </button>
        </div>
      </div>
    </div>

    <div v-if="showCelebration" class="modal-backdrop celebration-backdrop" @click.self="closeCelebration"
      @touchstart.self="closeCelebration">
      <div class="celebration-card">
        <div class="confetti">
          <span v-for="n in 30" :key="n"></span>
        </div>
        <div class="celebration-content">
          <p class="eyebrow">All done!</p>
          <h2>Merry Christmas 🎄</h2>
          <p class="lede">You checked off every item in this checklist.</p>
          <button class="btn primary" type="button" @click="closeCelebration">Nice!</button>
        </div>
      </div>
    </div>

    <div v-if="activeChecklist" class="active-section" ref="activeSectionRef">
      <div class="input-card">
        <label class="label" for="active-title">Checklist title</label>
        <input id="active-title" v-model="activeTitle" class="text-input" type="text"
          placeholder="Untitled checklist" />

        <div class="inline-inputs">
          <label class="label" for="active-sep">Separator</label>
          <select id="active-sep" v-model="activeSeparator" class="select-input">
            <option value="newline">New line (default)</option>
            <option value="comma">Comma</option>
            <option value="semicolon">Semicolon</option>
            <option value="custom">Custom</option>
          </select>
          <input v-if="activeSeparator === 'custom'" v-model="customActiveSeparator" class="text-input compact"
            type="text" maxlength="5" placeholder="|" />
        </div>
        <label class="label" for="paste-area">Paste items</label>
        <textarea id="paste-area" v-model="activeDraft" placeholder="Milk&#10;Eggs&#10;Call the dentist" rows="4" />
        <div class="actions">
          <label class="btn ghost" for="file-import">
            Import .txt / .json
          </label>
          <input id="file-import" ref="fileInputRef" class="file-input" type="file"
            accept=".txt,text/plain,.json,application/json" @change="importFromFile" />
          <button class="btn ghost" type="button" @click="exportJson">
            Export JSON
          </button>
          <button class="btn secondary" type="button" @click="clearAll">
            Clear list
          </button>
          <button class="btn primary" type="button" @click="addItems">
            Add items
          </button>
        </div>
      </div>

      <div class="summary">
        <div>
          <p class="count">{{ completedCount }} done</p>
          <p class="count subtle">{{ remainingCount }} remaining</p>
        </div>
        <div class="summary-actions">
          <button class="link" type="button" :disabled="!hasUndo" @click="undoLastAction">
            Undo
          </button>
          <button class="link" type="button" @click="toggleAll(false)">
            Uncheck all
          </button>
        </div>
      </div>

      <div class="list-card" v-if="items.length">

        <ul class="list" role="list">
          <li v-for="item in items" :key="item.id" class="row" :data-id="item.id" :class="{
            dragging: dragSourceId === item.id,
            'drop-target': dropTargetId === item.id,
          }" @dragenter.prevent="onDragEnter(item.id)" @dragover.prevent @drop.prevent="finishDrop(item.id)">
            <button class="drag-handle" type="button" aria-label="Reorder item" draggable="true"
              @dragstart="startDrag(item.id, $event)" @dragend="cancelDrag"
              @touchstart.prevent.stop="touchStart(item.id, $event)" @touchmove.prevent.stop="touchMove"
              @touchend.prevent.stop="touchEnd">
              ≡
            </button>
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
import { computed, nextTick, onMounted, ref, watch } from "vue";

const STORAGE_KEY = "paste-checklists";
const LEGACY_KEY = "paste-checklist-items";

const checklists = ref([]);
const activeSectionRef = ref(null);
const showModal = ref(false);
const fileInputRef = ref(null);
const dragSourceId = ref(null);
const dropTargetId = ref(null);
const touchDragId = ref(null);
const actionStack = ref([]);
const ACTION_LIMIT = 10;
const activeId = ref(null);
const showCelebration = ref(false);
const celebratedMap = ref({});

const newTitle = ref("");
const newDraft = ref("");
const newSeparator = ref("newline");
const customNewSeparator = ref("");

const activeDraft = ref("");
const activeSeparator = ref("newline");
const customActiveSeparator = ref("");

const parseBySeparator = (text, separator, custom) => {
  const trimmedCustom = custom.trim();
  const sep =
    separator === "custom" && trimmedCustom
      ? trimmedCustom
      : separator === "comma"
        ? ","
        : separator === "semicolon"
          ? ";"
          : "\n";

  const parts =
    sep === "\n"
      ? text.split(/\r?\n/)
      : text.split(sep);

  return parts.map((line) => line.trim()).filter(Boolean);
};

const parsedNewDraft = computed(() =>
  parseBySeparator(newDraft.value, newSeparator.value, customNewSeparator.value)
);

const parsedActiveDraft = computed(() =>
  parseBySeparator(activeDraft.value, activeSeparator.value, customActiveSeparator.value)
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
const hasUndo = computed(() => actionStack.value.length > 0);
const allDone = computed(
  () =>
    !!items.value.length &&
    items.value.every((item) => item.done)
);

const pushAction = (action) => {
  actionStack.value = [...actionStack.value.slice(-(ACTION_LIMIT - 1)), action];
};

const cloneItems = (itemsToClone) =>
  itemsToClone.map((item) => ({ ...item }));

const scrollToActive = () => {
  nextTick(() => {
    const el = activeSectionRef.value;
    if (el) {
      el.scrollIntoView({ behavior: "smooth", block: "start" });
    } else {
      window.scrollTo({ top: 0, behavior: "smooth" });
    }
  });
};

const setActive = (id) => {
  activeId.value = id;
  scrollToActive();
};

const openModal = () => {
  showModal.value = true;
};

const closeModal = () => {
  showModal.value = false;
};

const closeCelebration = () => {
  showCelebration.value = false;
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
  newSeparator.value = "newline";
  customNewSeparator.value = "";
  activeDraft.value = "";
  closeModal();
};

const addItems = () => {
  if (!activeChecklist.value || !parsedActiveDraft.value.length) return;

  const newItems = parsedActiveDraft.value.map((text) => ({
    id: crypto.randomUUID(),
    text,
    done: false,
  }));

  const targetId = activeChecklist.value.id;
  const newCount = newItems.length;

  if (newCount === 1) {
    pushAction({
      type: "add",
      checklistId: targetId,
      itemId: newItems[0].id,
    });
  } else {
    pushAction({
      type: "snapshot",
      checklistId: targetId,
      items: cloneItems(activeChecklist.value.items),
    });
  }

  updateChecklist(targetId, (list) => ({
    ...list,
    items: [...list.items, ...newItems],
  }));

  activeDraft.value = "";
  activeSeparator.value = "newline";
  customActiveSeparator.value = "";
};

const importFromFile = (event) => {
  const [file] = event.target.files || [];
  if (!file || !activeChecklist.value) {
    event.target.value = "";
    return;
  }

  const reader = new FileReader();
  reader.onload = () => {
    const raw = reader.result ?? "";
    let parsedLines = [];
    const isJsonFile =
      file.type === "application/json" || file.name.toLowerCase().endsWith(".json");

    if (isJsonFile) {
      try {
        const data = JSON.parse(raw);
        const itemsArray = Array.isArray(data)
          ? data
          : Array.isArray(data?.items)
            ? data.items
            : [];
        parsedLines = itemsArray
          .map((item) =>
            typeof item === "string"
              ? { text: item, done: false }
              : item && typeof item.text === "string"
                ? { text: item.text, done: !!item.done }
                : null
          )
          .filter(Boolean);
      } catch (err) {
        console.error("Failed to parse JSON import", err);
      }
    }

    if (!parsedLines.length) {
      const parsedText = parseBySeparator(
        String(raw),
        activeSeparator.value,
        customActiveSeparator.value
      );
      parsedLines = parsedText.map((line) => ({ text: line, done: false }));
    }

    if (!parsedLines.length) {
      event.target.value = "";
      return;
    }

    const newItems = parsedLines.map((entry) => ({
      id: crypto.randomUUID(),
      text: entry.text,
      done: !!entry.done,
    }));

    const targetId = activeChecklist.value.id;
    const newCount = newItems.length;

    if (newCount === 1) {
      pushAction({
        type: "add",
        checklistId: targetId,
        itemId: newItems[0].id,
      });
    } else {
      pushAction({
        type: "snapshot",
        checklistId: targetId,
        items: cloneItems(activeChecklist.value.items),
      });
    }

    updateChecklist(targetId, (list) => ({
      ...list,
      items: [...list.items, ...newItems],
    }));

    activeSeparator.value = "newline";
    customActiveSeparator.value = "";
    event.target.value = "";
  };
  reader.onerror = () => {
    event.target.value = "";
  };
  reader.readAsText(file);
};

const exportJson = () => {
  if (!activeChecklist.value) return;
  const payload = {
    title: activeChecklist.value.title,
    items: activeChecklist.value.items,
  };
  const blob = new Blob([JSON.stringify(payload, null, 2)], {
    type: "application/json",
  });
  const url = URL.createObjectURL(blob);
  const title = (activeChecklist.value.title || "checklist")
    .trim()
    .replace(/\s+/g, "-")
    .replace(/[^a-zA-Z0-9-_]/g, "")
    .toLowerCase();
  const a = document.createElement("a");
  a.href = url;
  a.download = `${title || "checklist"}.json`;
  a.click();
  URL.revokeObjectURL(url);
};

const toggleItem = (id) => {
  if (!activeChecklist.value) return;
  const current = activeChecklist.value.items.find((item) => item.id === id);
  if (!current) return;
  pushAction({
    type: "toggle",
    checklistId: activeChecklist.value.id,
    itemId: id,
    prevDone: current.done,
  });
  updateChecklist(activeChecklist.value.id, (list) => ({
    ...list,
    items: list.items.map((item) =>
      item.id === id ? { ...item, done: !item.done } : item
    ),
  }));
};

const removeItem = (id) => {
  if (!activeChecklist.value) return;
  const list = activeChecklist.value.items;
  const index = list.findIndex((item) => item.id === id);
  if (index === -1) return;
  const removed = list[index];
  pushAction({
    type: "remove",
    checklistId: activeChecklist.value.id,
    item: { ...removed },
    index,
  });
  updateChecklist(activeChecklist.value.id, (list) => ({
    ...list,
    items: list.items.filter((item) => item.id !== id),
  }));
};

const toggleAll = (state) => {
  if (!activeChecklist.value) return;
  pushAction({
    type: "snapshot",
    checklistId: activeChecklist.value.id,
    items: cloneItems(activeChecklist.value.items),
  });
  updateChecklist(activeChecklist.value.id, (list) => ({
    ...list,
    items: list.items.map((item) => ({ ...item, done: state })),
  }));
};

const clearAll = () => {
  if (!activeChecklist.value) return;
  if (!activeChecklist.value.items.length) return;
  pushAction({
    type: "snapshot",
    checklistId: activeChecklist.value.id,
    items: cloneItems(activeChecklist.value.items),
  });
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

const reorderItems = (sourceId, targetId) => {
  if (!activeChecklist.value || sourceId === targetId) return;
  const list = activeChecklist.value.items;
  const from = list.findIndex((item) => item.id === sourceId);
  const to = list.findIndex((item) => item.id === targetId);
  if (from === -1 || to === -1) return;

  pushAction({
    type: "snapshot",
    checklistId: activeChecklist.value.id,
    items: cloneItems(list),
  });

  const next = [...list];
  const [moved] = next.splice(from, 1);
  const insertIndex = from < to ? to - 1 : to;
  next.splice(insertIndex, 0, moved);

  updateChecklist(activeChecklist.value.id, (current) => ({
    ...current,
    items: next,
  }));
};

const startDrag = (id, event) => {
  dragSourceId.value = id;
  dropTargetId.value = null;
  event.dataTransfer?.setData("text/plain", id);
  event.dataTransfer?.setDragImage?.(event.target, 10, 10);
};

const touchStart = (id, event) => {
  touchDragId.value = id;
  dragSourceId.value = id;
  dropTargetId.value = null;
  if (event.target instanceof HTMLElement) {
    event.target.style.touchAction = "none";
  }
};

const touchMove = (event) => {
  if (!touchDragId.value) return;
  const touch = event.touches?.[0];
  if (!touch) return;
  const el = document.elementFromPoint(touch.clientX, touch.clientY);
  const targetLi = el?.closest?.("li.row[data-id]");
  if (targetLi) {
    const targetId = targetLi.getAttribute("data-id");
    if (targetId && targetId !== touchDragId.value) {
      dropTargetId.value = targetId;
    }
  }
};

const touchEnd = () => {
  if (touchDragId.value && dropTargetId.value) {
    reorderItems(touchDragId.value, dropTargetId.value);
  }
  touchDragId.value = null;
  dragSourceId.value = null;
  dropTargetId.value = null;
};

const onDragEnter = (id) => {
  if (!dragSourceId.value || dragSourceId.value === id) {
    dropTargetId.value = null;
    return;
  }
  dropTargetId.value = id;
};

const finishDrop = (id) => {
  if (!dragSourceId.value) return;
  reorderItems(dragSourceId.value, id);
  dragSourceId.value = null;
  dropTargetId.value = null;
};

const cancelDrag = () => {
  dragSourceId.value = null;
  dropTargetId.value = null;
};

const undoLastAction = () => {
  const action = actionStack.value[actionStack.value.length - 1];
  if (!action) return;
  actionStack.value = actionStack.value.slice(0, -1);

  const checklist = checklists.value.find(
    (list) => list.id === action.checklistId
  );
  if (!checklist) return;

  if (action.type === "snapshot") {
    updateChecklist(action.checklistId, (list) => ({
      ...list,
      items: cloneItems(action.items),
    }));
    return;
  }

  if (action.type === "add") {
    updateChecklist(action.checklistId, (list) => ({
      ...list,
      items: list.items.filter((item) => item.id !== action.itemId),
    }));
    return;
  }

  if (action.type === "remove") {
    updateChecklist(action.checklistId, (list) => {
      const next = [...list.items];
      next.splice(action.index, 0, action.item);
      return { ...list, items: next };
    });
    return;
  }

  if (action.type === "toggle") {
    updateChecklist(action.checklistId, (list) => ({
      ...list,
      items: list.items.map((item) =>
        item.id === action.itemId ? { ...item, done: action.prevDone } : item
      ),
    }));
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

watch(
  [items, activeId],
  () => {
    const id = activeId.value;
    if (!id) return;
    const completed = allDone.value;
    const map = { ...celebratedMap.value };

    if (completed) {
      if (!map[id]) {
        map[id] = true;
        celebratedMap.value = map;
        showCelebration.value = true;
      }
    } else {
      if (map[id]) {
        map[id] = false;
        celebratedMap.value = map;
      }
    }
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

.tab-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
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

.empty-bar {
  display: flex;
  align-items: center;
  gap: 12px;
  background: #f8fafc;
  border: 1px dashed #cbd5e1;
  border-radius: 12px;
  padding: 10px 12px;
}

.add-btn {
  white-space: nowrap;
}

.add-btn::after {
  content: "";
}

.inline-inputs {
  display: flex;
  align-items: center;
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
.select-input,
textarea {
  width: 100%;
  border: 1px solid #cbd5e1;
  border-radius: 10px;
  margin-bottom: 10px;
  padding: 10px 12px;
  font: inherit;
  background: #fff;
}

.text-input.compact {
  max-width: 90px;
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

.ghost {
  background: #fff;
  border-color: #cbd5e1;
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

.summary-actions {
  display: flex;
  gap: 10px;
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

.row.dragging {
  opacity: 0.5;
}

.row.drop-target {
  outline: 2px dashed #2563eb;
}

.row-content {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  cursor: pointer;
}

.drag-handle {
  border: 1px solid #cbd5e1;
  background: #fff;
  color: #475569;
  border-radius: 8px;
  width: 32px;
  height: 32px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: grab;
  touch-action: none;
}

.drag-handle:active {
  cursor: grabbing;
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

.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.28);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 16px;
  z-index: 30;
}

.modal {
  width: min(500px, 100%);
  background: #fff;
  border-radius: 16px;
  padding: 18px;
  box-shadow: 0 25px 60px rgba(15, 23, 42, 0.25);
  display: grid;
  gap: 10px;
}

.modal-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 4px;
}

.modal-head h3 {
  margin: 0;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}

.celebration-backdrop {
  background: radial-gradient(circle at 20% 20%, rgba(236, 72, 153, 0.25), transparent),
    radial-gradient(circle at 80% 10%, rgba(59, 130, 246, 0.25), transparent),
    radial-gradient(circle at 50% 80%, rgba(16, 185, 129, 0.25), transparent),
    rgba(15, 23, 42, 0.35);
  padding: 0;
}

.celebration-card {
  position: relative;
  overflow: hidden;
  width: 100%;
  height: 100%;
  background: linear-gradient(135deg, #0f172a, #1e293b);
  color: #fff;
  border-radius: 0;
  padding: 32px;
  box-shadow: 0 25px 60px rgba(15, 23, 42, 0.35);
  display: flex;
  align-items: center;
  justify-content: center;
}

.celebration-content h2 {
  margin: 0 0 8px 0;
}

.celebration-content .lede {
  color: #cbd5e1;
}

.confetti {
  position: absolute;
  inset: 0;
  pointer-events: none;
}

.confetti span {
  position: absolute;
  width: 10px;
  height: 14px;
  background: var(--confetti-color, #fbbf24);
  top: -20px;
  left: 50%;
  animation: fall 1.8s linear infinite;
  transform: translateX(-50%);
  opacity: 0.9;
}

.confetti span:nth-child(4n) {
  --confetti-color: #f472b6;
}

.confetti span:nth-child(4n + 1) {
  --confetti-color: #38bdf8;
}

.confetti span:nth-child(4n + 2) {
  --confetti-color: #a3e635;
}

.confetti span:nth-child(4n + 3) {
  --confetti-color: #fcd34d;
}

.confetti span:nth-child(n) {
  left: calc(3% * var(--i, 1));
  animation-delay: calc(0.05s * var(--i, 1));
}

.confetti span {
  --i: 1;
}

.confetti span:nth-child(1) {
  --i: 1;
}

.confetti span:nth-child(2) {
  --i: 2;
}

.confetti span:nth-child(3) {
  --i: 3;
}

.confetti span:nth-child(4) {
  --i: 4;
}

.confetti span:nth-child(5) {
  --i: 5;
}

.confetti span:nth-child(6) {
  --i: 6;
}

.confetti span:nth-child(7) {
  --i: 7;
}

.confetti span:nth-child(8) {
  --i: 8;
}

.confetti span:nth-child(9) {
  --i: 9;
}

.confetti span:nth-child(10) {
  --i: 10;
}

.confetti span:nth-child(11) {
  --i: 11;
}

.confetti span:nth-child(12) {
  --i: 12;
}

.confetti span:nth-child(13) {
  --i: 13;
}

.confetti span:nth-child(14) {
  --i: 14;
}

.confetti span:nth-child(15) {
  --i: 15;
}

.confetti span:nth-child(16) {
  --i: 16;
}

.confetti span:nth-child(17) {
  --i: 17;
}

.confetti span:nth-child(18) {
  --i: 18;
}

.confetti span:nth-child(19) {
  --i: 19;
}

.confetti span:nth-child(20) {
  --i: 20;
}

.confetti span:nth-child(21) {
  --i: 21;
}

.confetti span:nth-child(22) {
  --i: 22;
}

.confetti span:nth-child(23) {
  --i: 23;
}

.confetti span:nth-child(24) {
  --i: 24;
}

.confetti span:nth-child(25) {
  --i: 25;
}

.confetti span:nth-child(26) {
  --i: 26;
}

.confetti span:nth-child(27) {
  --i: 27;
}

.confetti span:nth-child(28) {
  --i: 28;
}

.confetti span:nth-child(29) {
  --i: 29;
}

.confetti span:nth-child(30) {
  --i: 30;
}

@keyframes fall {
  0% {
    transform: translate3d(var(--x, 0), -10px, 0) rotate(0deg);
  }

  100% {
    transform: translate3d(var(--x, 0), 110vh, 0) rotate(360deg);
  }
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

.file-input {
  display: none;
}

@media (max-width: 640px) {
  .actions {
    flex-direction: column;
    align-items: stretch;
  }

  .row {
    align-items: flex-start;
  }

  .drag-handle {
    width: 28px;
    height: 28px;
  }
}

@media (max-width: 640px) {
  .tab-bar {
    gap: 8px;
  }

  .add-btn {
    position: fixed;
    right: 18px;
    bottom: 18px;
    width: 56px;
    height: 56px;
    border-radius: 50%;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 0;
    box-shadow: 0 14px 30px rgba(37, 99, 235, 0.35);
    z-index: 40;
    font-size: 0;
    line-height: 1;
  }

  .add-btn::after {
    content: "+";
    font-size: 28px;
  }

  .add-btn {
    color: white;
  }
}
</style>
