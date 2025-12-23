<template>
  <main class="shell">
    <header class="header">
      <div>
        <p class="eyebrow">Vue 3 · Interactive Checklists</p>
        <h1>Paste lists, title them, and start checking off items</h1>
        <p class="lede">
          Each line becomes a checklist item. Everything is saved locally so you
          can come back later.
        </p>
      </div>
      <div class="badges">
        <span class="badge">Vue 3</span>
        <span class="badge">LocalStorage</span>
      </div>
    </header>

    <Checklist />
  </main>
</template>

<script setup>
import Checklist from "./components/Checklist.vue";
</script>

<style scoped>
.shell {
  width: min(1100px, 100%);
  background: #ffffff;
  padding: 32px;
  border-radius: 18px;
  box-shadow: 0 25px 70px rgba(15, 23, 42, 0.08);
}

.header {
  display: flex;
  gap: 16px;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 24px;
}

.eyebrow {
  letter-spacing: 0.08em;
  text-transform: uppercase;
  font-weight: 600;
  color: #475569;
  margin: 0 0 6px 0;
}

h1 {
  font-size: clamp(26px, 3vw, 32px);
  margin: 0 0 8px 0;
  color: #0f172a;
}

.lede {
  margin: 0;
  color: #475569;
}

.badges {
  display: flex;
  gap: 8px;
}

.badge {
  padding: 6px 10px;
  background: #e2e8f0;
  border-radius: 12px;
  font-size: 13px;
  color: #0f172a;
  font-weight: 600;
}

@media (max-width: 720px) {
  .shell {
    padding: 24px;
  }

  .header {
    flex-direction: column;
    align-items: flex-start;
  }
}
</style>

