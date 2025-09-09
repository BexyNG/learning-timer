//src/components/SessionQueue.vue
<template>
  <div class="queue-container">
    <h2>📁 Session Queue</h2>

    <!-- Quick Presets (store minutes, display seconds) -->
    <div class="quick-buttons">
      <button @click="quickAdd('Main Topic', 25)">Main Topic</button>
      <button @click="quickAdd('Misc Topic', 15)">Misc Topic</button>
      <button @click="quickAdd('Break', 30)">Break</button>
    </div>

    <!-- Add Custom Session (enter minutes; queue displays seconds) -->
    <form @submit.prevent="addSession" class="add-form">
      <input v-model="name" placeholder="Custom session name" />
      <input v-model="durationMinutes" type="text" placeholder="Minutes" />
      <button type="submit">＋ Add</button>
    </form>

    <!-- Queue List -->
    <ul v-if="queue.length > 0">
      <li v-for="(s, i) in queue" :key="i">
        {{ s.name }} – {{ formatDuration(s.duration) }}
      </li>
    </ul>
    <p v-else class="empty">Your queue is empty.</p>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'

/*
  IMPORTANT:
  - We store durations in MINUTES (number).
  - We DISPLAY them in SECONDS (e.g., 5m -> 300s), per your request.
*/

const queue = ref([])
const name = ref('')
const durationMinutes = ref('25') // default minutes

function formatDuration(minutes) {
  const secs = Math.round((Number(minutes) || 0) * 60)
  return `${secs}s`
}

function addSession() {
  const mins = parseFloat(durationMinutes.value)
  if (!name.value || isNaN(mins) || mins <= 0) return
  queue.value.push({ name: name.value.trim(), duration: mins })
  name.value = ''
  durationMinutes.value = '25'
}

function quickAdd(label, minutes) {
  queue.value.push({ name: label, duration: minutes })
}

// 💾 Load + Save queue from localStorage
onMounted(() => {
  const saved = localStorage.getItem('sessionQueue')
  if (saved) queue.value = JSON.parse(saved)
})
watch(queue, (val) => {
  localStorage.setItem('sessionQueue', JSON.stringify(val))
}, { deep: true })

// Expose to App.vue
defineExpose({
  popNext: () => queue.value.shift(),
  clearQueue: () => (queue.value = []),
  queue
})
</script>

<style scoped>
.queue-container {
  background: var(--card-bg);
  padding: 1.5rem;
  border-radius: 12px;
  box-shadow: 0 0 8px rgba(0, 0, 0, 0.04);
}

.quick-buttons {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
  margin-bottom: 1rem;
}

.quick-buttons button {
  background: var(--accent-light);
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 8px;
  font-weight: bold;
  cursor: pointer;
  color: #000;
}

/* 🔧 Inline input layout */
.add-form {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
  margin-bottom: 1rem;
}

.add-form input {
  flex: 1 1 auto;
  padding: 0.5rem;
  border: 1px solid var(--border-color);
  border-radius: 8px;
  font-size: 1rem;
}

.add-form button {
  background: var(--accent);
  border: none;
  padding: 0.5rem 1.2rem;
  border-radius: 8px;
  color: var(--button-text);
  font-weight: bold;
  cursor: pointer;
}

ul {
  list-style: none;
  padding: 0;
  margin: 0.5rem 0;
}

li {
  padding: 0.4rem 0;
  border-bottom: 1px solid var(--border-color);
  font-size: 0.95rem;
}

.empty {
  color: var(--text);
  opacity: 0.6;
  font-style: italic;
  font-size: 0.9rem;
}
</style>
