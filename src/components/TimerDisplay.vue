//src/components/TimerDisplay.vue
<template>
  <div class="timer-container">
    <div class="session-title">{{ session?.name || 'No Session' }}</div>

    <div class="countdown-wrap">
      <div class="countdown">
        {{ time.hours }}:{{ time.minutes }}:{{ time.seconds }}:{{ time.milliseconds }}
      </div>
    </div>

    <div class="progress-bar">
      <div class="fill" :style="{ width: progress + '%' }"></div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'

const props = defineProps({
  session: Object,          // { name, duration (minutes) }
  onComplete: Function,     // callback for natural completion
  pause: Boolean,
  stopped: Boolean
})

/* ============== State ============== */
const remaining = ref(0)
const progress  = ref(0)
const running   = ref(false)

let intervalId = null     // UI updates
let endTimerId = null     // hard deadline
let endTime = null

let currentRunId = 0
let invalidRunIds = new Set()
let audioCtx = null
let audioUnlocked = false

/* ============== Derived Time ============== */
const time = computed(() => {
  const ms   = Math.max(0, remaining.value) % 1000
  const tSec = Math.max(0, Math.floor(remaining.value / 1000))
  const sec  = tSec % 60
  const min  = Math.floor(tSec / 60) % 60
  const hrs  = Math.floor(tSec / 3600)
  return {
    hours: String(hrs).padStart(2, '0'),
    minutes: String(min).padStart(2, '0'),
    seconds: String(sec).padStart(2, '0'),
    milliseconds: String(Math.floor(ms / 10)).padStart(2, '0')
  }
})

/* ============== Audio Unlock ============== */
function ensureAudioUnlocked() {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)()
  if (audioCtx.state === 'suspended') {
    return audioCtx.resume().then(() => (audioUnlocked = true))
  }
  audioUnlocked = true
  return Promise.resolve()
}

function gestureUnlock() {
  ensureAudioUnlocked()
}

onMounted(() => {
  window.addEventListener('pointerdown', gestureUnlock, { once: true })
  window.addEventListener('keydown', gestureUnlock, { once: true })
  window.addEventListener('touchstart', gestureUnlock, { once: true })
})

/* ============== Timer Control ============== */
function startSession(s) {
  clearTimers()
  if (!s || !s.duration) {
    running.value = false
    remaining.value = 0
    progress.value = 0
    return
  }

  const durationMs = s.duration * 60 * 1000
  endTime = Date.now() + durationMs
  remaining.value = durationMs
  progress.value = 0
  running.value = true

  currentRunId++
  const runId = currentRunId

  // UI updater (can be throttled in background)
  intervalId = setInterval(() => {
    updateRemaining(durationMs)
  }, 100)

  // Hard deadline: guarantees completion even if backgrounded
  endTimerId = setTimeout(() => {
    forceComplete(runId)
  }, durationMs + 100)
}

function updateRemaining(durationMs) {
  if (!endTime) return
  const left = Math.max(0, endTime - Date.now())
  remaining.value = left
  progress.value = durationMs > 0 ? ((durationMs - left) / durationMs) * 100 : 0

  if (left <= 0) {
    forceComplete(currentRunId)
  }
}

function forceComplete(runId) {
  clearTimers()
  if (!running.value) return
  running.value = false
  remaining.value = 0
  progress.value = 100

  if (!invalidRunIds.has(runId)) {
    playAlarmSequence(runId)
    if (typeof props.onComplete === 'function') props.onComplete()
  }
}

function clearTimers() {
  if (intervalId) clearInterval(intervalId)
  if (endTimerId) clearTimeout(endTimerId)
  intervalId = null
  endTimerId = null
}

/* ============== Invalidation ============== */
function invalidateRun() {
  if (currentRunId) invalidRunIds.add(currentRunId)
  clearTimers()
}

/* ============== Watchers ============== */
watch(() => props.session, (s) => {
  if (s) startSession(s)
  else {
    clearTimers()
    running.value = false
    remaining.value = 0
    progress.value = 0
  }
}, { immediate: true })

watch(() => props.pause, (paused) => {
  if (!running.value) return
  if (paused) {
    clearTimers()
  } else {
    const durationMs = props.session.duration * 60 * 1000
    const left = Math.max(0, endTime - Date.now())
    remaining.value = left
    progress.value = ((durationMs - left) / durationMs) * 100
    // resume hard deadline
    endTimerId = setTimeout(() => {
      forceComplete(currentRunId)
    }, left + 100)
    // resume UI updates
    intervalId = setInterval(() => updateRemaining(durationMs), 100)
  }
})

watch(() => props.stopped, (stopped) => {
  if (stopped) {
    invalidateRun()
    running.value = false
  }
})

onBeforeUnmount(() => {
  invalidateRun()
})

/* ============== Alarm Sequence ============== */
function playAlarmSequence(runId) {
  ensureAudioUnlocked().finally(() => {
    if (invalidRunIds.has(runId)) return

    const sequence = [
      { f: 520, d: 500, g: 0.55 },
      { f: 700, d: 600, g: 0.75 },
      { f: 920, d: 700, g: 0.92 }
    ]

    let delay = 0
    sequence.forEach(({ f, d, g }) => {
      setTimeout(() => {
        if (!invalidRunIds.has(runId)) playBeep(f, d, g)
      }, delay)
      delay += d + 200 // sequential spacing
    })
  })
}

function playBeep(freq = 660, duration = 400, peak = 0.7) {
  try {
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)()
    if (audioCtx.state === 'suspended') audioCtx.resume()

    const osc   = audioCtx.createOscillator()
    const gain  = audioCtx.createGain()
    const filter = audioCtx.createBiquadFilter()

    filter.type = 'lowpass'
    filter.frequency.value = Math.min(14000, freq * 10)

    osc.connect(filter)
    filter.connect(gain)
    gain.connect(audioCtx.destination)

    osc.type = 'square'
    osc.frequency.setValueAtTime(freq, audioCtx.currentTime)

    const now = audioCtx.currentTime
    const attack = 0.03
    const hold   = Math.max(0, (duration / 1000) - (attack + 0.15))
    const release = 0.15

    const safePeak = Math.min(0.98, Math.max(0.05, peak))
    gain.gain.setValueAtTime(0.0001, now)
    gain.gain.linearRampToValueAtTime(safePeak, now + attack)
    gain.gain.setValueAtTime(safePeak, now + attack + hold)
    gain.gain.linearRampToValueAtTime(0.0001, now + attack + hold + release)

    osc.start(now)
    osc.stop(now + attack + hold + release + 0.02)
  } catch (e) {
    console.warn('Audio error', e)
  }
}
</script>

<style scoped>
.timer-container {
  text-align: center;
  background-color: var(--timer-bg);
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: 0 0 15px rgba(0,0,0,0.06);
}

.session-title {
  font-size: 1.1rem;
  margin-bottom: 0.6rem;
  opacity: 0.9;
}

.countdown-wrap {
  display: grid;
  place-items: center;
  min-height: 140px;
}

.countdown {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', 'Courier New', monospace;
  font-size: 3.2rem;
  letter-spacing: 0.04em;
  text-shadow: 0 0 10px var(--accent-glow);
}

.progress-bar {
  height: 12px;
  border-radius: 8px;
  background: var(--progress-bg);
  overflow: hidden;
  margin-top: 1rem;
  position: relative;
}

.progress-bar .fill {
  height: 100%;
  width: 0%;
  transition: width 0.12s linear;
  background: linear-gradient(90deg, var(--accent-light), var(--accent));
  background-size: 200% 100%;
  animation: gradient-shift 4s linear infinite;
}

@keyframes gradient-shift {
  0% { background-position: 0% 50%; }
  100% { background-position: 200% 50%; }
}
</style>
