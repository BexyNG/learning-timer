//src/App.vue
<template>
  <div class="app-card">
    <header class="header">
      <h1>🎯 CIA Type Learning Timer</h1>
      <button @click="isDarkMode = !isDarkMode" class="theme-toggle">
        {{ isDarkMode ? '☀️' : '🌙' }}
      </button>
    </header>

    <!-- Timer + Queue side-by-side -->
    <div class="app-content">
      <section class="timer-section">
        <div class="timer-holder">
          <TimerDisplay
            :key="key"
            :session="currentSession"
            :onComplete="nextSession"
            :pause="pauseSignal"
            :stopped="stopSignal"
          />
        </div>
      </section>

      <section class="queue-section">
        <div class="queue-scroll">
          <SessionQueue ref="queueRef" />
        </div>
      </section>
    </div>

    <section class="controls-section">
      <Controls
        :isPaused="pauseSignal"
        @start="startQueue"
        @clear="clearQueue"
        @pause="pause"
        @resume="resume"
        @skip="skip"
        @stop="stop"
      />
    </section>
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'
import TimerDisplay from './components/TimerDisplay.vue'
import SessionQueue from './components/SessionQueue.vue'
import Controls from './components/Controls.vue'

const queueRef = ref(null)
const currentSession = ref(null)
const key = ref(0)

const pauseSignal = ref(false)
const stopSignal = ref(false)
let skipImmediate = false

// 🌙 Dark Mode
const isDarkMode = ref(localStorage.getItem('dark') === 'true')
watch(isDarkMode, (val) => {
  localStorage.setItem('dark', val)
  document.documentElement.setAttribute('data-theme', val ? 'dark' : 'light')
})
document.documentElement.setAttribute('data-theme', isDarkMode.value ? 'dark' : 'light')

// ============== Controls ==============
function startQueue() {
  const next = queueRef.value?.popNext?.()
  if (next) {
    pauseSignal.value = false
    stopSignal.value = false
    currentSession.value = next
    key.value++
  } else {
    alert('⚠️ Your session queue is empty!')
  }
}

function nextSession() {
  currentSession.value = null

  const next = queueRef.value?.popNext?.()
  if (!next) return

  const loadNext = () => {
    pauseSignal.value = false
    stopSignal.value = false
    currentSession.value = next
    key.value++
  }

  // Skip = load immediately
  if (skipImmediate) {
    skipImmediate = false
    loadNext()
  } else {
    // Wait a few seconds after alarm before auto-start
    setTimeout(loadNext, 3000) // 3s gap after beeps
  }
}

function pause() {
  pauseSignal.value = true
}
function resume() {
  pauseSignal.value = false
}
function stop() {
  pauseSignal.value = false
  stopSignal.value = true
  currentSession.value = null
  key.value++
}
function skip() {
  skipImmediate = true
  nextSession()
}
function clearQueue() {
  currentSession.value = null
  pauseSignal.value = false
  stopSignal.value = true
  queueRef.value?.clearQueue?.()
  key.value++
}
</script>

<style scoped>
.app-card {
  width: 100%;
  max-width: 960px;
  background-color: var(--card-bg);
  border-radius: 16px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.05);
  padding: 2rem;
  margin: 1rem;
  position: relative;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.theme-toggle {
  background: transparent;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
}

/* Layout */
.app-content {
  display: flex;
  flex-direction: row;
  gap: 2rem;
  flex-wrap: wrap;
  justify-content: center;
  align-items: flex-start;
}

.timer-section,
.queue-section {
  flex: 1 1 420px;
  min-width: 300px;
}

.timer-holder {
  min-height: 300px;
  display: grid;
  place-items: center;
}

.queue-scroll {
  max-height: 420px;
  overflow-y: auto;
  padding-right: 0.25rem;
}

.queue-scroll::-webkit-scrollbar {
  width: 8px;
}
.queue-scroll::-webkit-scrollbar-thumb {
  background: var(--border-color);
  border-radius: 8px;
}

.controls-section {
  margin-top: 1.5rem;
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  gap: 1rem;
  border-top: 1px solid var(--border-color);
  padding-top: 1rem;
}

@media (max-width: 640px) {
  .timer-holder { min-height: 260px; }
  .queue-scroll { max-height: 360px; }
}
</style>
