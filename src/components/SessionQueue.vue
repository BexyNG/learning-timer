<template>
  <div class="queue-container">
    <h2>📁 Session Queue</h2>

    <!-- Quick Presets -->
    <div class="quick-buttons">
      <button @click="quickAdd('Main Topic', 1500)">Main Topic</button>
      <button @click="quickAdd('Misc Topic', 900)">Misc Topic</button>
      <button @click="quickAdd('Gaming Break', 1800)">Gaming Break</button>
    </div>

    <!-- Add Custom Session -->
    <form @submit.prevent="addSession" class="add-form">
      <input v-model="name" placeholder="Custom session name" />
      <input v-model="duration" type="text" placeholder="Seconds" />
      <button type="submit">＋ Add</button>
    </form>

    <!-- Queue List -->
    <ul v-if="queue.length > 0">
      <li v-for="(s, i) in queue" :key="i">
        {{ s.name }} – {{ s.duration }}s
      </li>
    </ul>
    <p v-else class="empty">Your queue is empty.</p>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'

const queue = ref([])
const name = ref('')
const duration = ref('1500')

function addSession() {
  const seconds = parseInt(duration.value)
  if (!name.value || isNaN(seconds) || seconds <= 0) return

  queue.value.push({ name: name.value.trim(), duration: seconds })
  name.value = ''
  duration.value = '1500'
}

function quickAdd(label, seconds) {
  queue.value.push({ name: label, duration: seconds })
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
