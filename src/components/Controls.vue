<template>
  <div class="controls">
    <button class="btn-primary" @click="startQueue">▶ Start</button>
    <button @click="togglePause">{{ isPaused ? '⏵ Resume' : '⏸ Pause' }}</button>
    <button @click="skipSession">⏭ Skip</button>
    <button class="btn-danger" @click="stopTimer">⏹ Stop</button>
    <button class="btn-danger" @click="clearQueue">🗑 Clear</button>
  </div>
</template>

<script setup>
const props = defineProps({ isPaused: Boolean })
const emit = defineEmits(['start', 'clear', 'pause', 'resume', 'skip', 'stop'])

function startQueue() {
  emit('start')
}

function togglePause() {
  emit(props.isPaused ? 'resume' : 'pause')
}

function skipSession() {
  emit('skip')
}

function stopTimer() {
  emit('stop')
}

function clearQueue() {
  emit('clear')
}
</script>

<style scoped>
.controls {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 1rem;
  margin-top: 1.5rem;
}
button {
  background: var(--timer-bg); /* default bg */
  color: var(--text);
  padding: 0.75rem 1.5rem;
  border-radius: 8px; /* slightly less rounded */
  border: 1px solid var(--border-color);
  font-weight: bold;
  cursor: pointer;
  transition: all 0.2s ease;
}

button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0,0,0,0.1);
}

button:active {
  transform: translateY(0);
  box-shadow: none;
}

.btn-primary {
  background: var(--accent);
  color: var(--button-text);
  border-color: var(--accent);
}

.btn-danger {
  background: var(--danger);
  color: var(--button-text);
  border-color: var(--danger);
}
</style>
