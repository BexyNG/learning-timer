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
  pause: Boolean,           // pause/resume flag from parent
  stopped: Boolean          // stop/skip flag from parent (invalidates beeps)
})

/* ============== State ============== */
const remaining = ref(0)         // ms left
const progress  = ref(0)         // 0..100
const running   = ref(false)

let intervalId = null
let currentRunId = 0             // increases every time a new session starts
let invalidRunIds = new Set()    // runs that were stopped/skipped (no alarm)
let alarmTimeouts = []           // timeouts scheduled for current alarm sequence
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

/* ============== Audio Unlock (crucial for mobile/Chrome) ============== */
function ensureAudioUnlocked() {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)()
  if (audioCtx.state === 'suspended') {
    return audioCtx.resume().then(() => (audioUnlocked = true))
  }
  audioUnlocked = true
  return Promise.resolve()
}

function gestureUnlock() {
  ensureAudioUnlocked().finally(() => {
    if (audioUnlocked) {
      window.removeEventListener('pointerdown', gestureUnlock, { capture: true })
      window.removeEventListener('keydown', gestureUnlock, { capture: true })
      window.removeEventListener('touchstart', gestureUnlock, { capture: true })
    }
  })
}

onMounted(() => {
  // unlock as soon as the user interacts with the page
  window.addEventListener('pointerdown', gestureUnlock, { capture: true })
  window.addEventListener('keydown', gestureUnlock, { capture: true })
  window.addEventListener('touchstart', gestureUnlock, { capture: true })
})

/* ============== Core: Start / Tick / Complete ============== */
function startSession(s) {
  clearTick()
  cancelAllAlarms()

  if (!s || !s.duration) {
    running.value = false
    remaining.value = 0
    progress.value = 0
    return
  }

  const durationMs = Math.max(0, s.duration * 60 * 1000)
  running.value = true
  remaining.value = durationMs
  progress.value = 0

  currentRunId += 1
  const runId = currentRunId

  const startedAt = Date.now()
  runTick(runId, startedAt, 0)
}

function runTick(runId, startAt, carriedElapsed) {
  const tick = () => {
    if (invalidRunIds.has(runId)) return clearTick()

    const now = Date.now()
    const durationMs = props.session ? props.session.duration * 60 * 1000 : 0
    const elapsed = (now - startAt) + carriedElapsed
    const left = Math.max(0, durationMs - elapsed)
    remaining.value = left
    progress.value = durationMs > 0 ? Math.min(100, (elapsed / durationMs) * 100) : 0

    if (left <= 0) {
      // natural completion
      clearTick()
      running.value = false

      if (!invalidRunIds.has(runId)) {
        playAlarmForRun(runId)
        if (typeof props.onComplete === 'function') props.onComplete()
      }
      return
    }
  }

  clearTick()
  intervalId = setInterval(tick, 10)
}

function clearTick() {
  if (intervalId) {
    clearInterval(intervalId)
    intervalId = null
  }
}

/* ============== Invalidation (Skip/Stop) ============== */
function invalidateRun() {
  if (currentRunId) invalidRunIds.add(currentRunId)
  cancelAllAlarms()
}

/* ============== Watchers ============== */
watch(() => props.session, (s) => {
  if (s) startSession(s)
  else {
    clearTick()
    running.value = false
    remaining.value = 0
    progress.value = 0
  }
}, { immediate: true })

watch(() => props.pause, (paused) => {
  if (!running.value) return
  if (paused) {
    clearTick()
  } else {
    const durationMs = props.session.duration * 60 * 1000
    const elapsed = durationMs - remaining.value
    runTick(currentRunId, Date.now(), elapsed)
  }
})

watch(() => props.stopped, (stopped) => {
  if (stopped) {
    invalidateRun()
    clearTick()
    running.value = false
  }
})

onBeforeUnmount(() => {
  invalidateRun()
  clearTick()
  cancelAllAlarms()
})

/* ============== Alarm Logic (slower, louder, crescendo) ============== */
function playAlarmForRun(runId) {
  cancelAllAlarms()

  ensureAudioUnlocked().finally(() => {
    // Slower cadence + obvious crescendo (each beep louder and a bit longer)
    // Each entry: start at t (ms), frequency f (Hz), duration d (ms), peak gain g (0..1)
    const plan = [
      { t: 0,    f: 520, d: 520, g: 0.55 }, // 1st: clear & audible
      { t: 800,  f: 700, d: 600, g: 0.75 }, // 2nd: louder
      { t: 1700, f: 920, d: 680, g: 0.92 }, // 3rd: loudest
    ]
    for (const { t, f, d, g } of plan) {
      const id = setTimeout(() => {
        if (invalidRunIds.has(runId)) return
        playBeep(f, d, g)
      }, t)
      alarmTimeouts.push(id)
    }
  })
}

function cancelAllAlarms() {
  alarmTimeouts.forEach(id => clearTimeout(id))
  alarmTimeouts.length = 0
}

// Punchy tone with filter and envelope; 'peak' controls loudness safely.
function playBeep(freq = 660, duration = 400, peak = 0.7) {
  try {
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)()
    if (audioCtx.state === 'suspended') audioCtx.resume()

    const osc   = audioCtx.createOscillator()
    const gain  = audioCtx.createGain()
    const filter = audioCtx.createBiquadFilter()

    // Slightly tame harshness while keeping presence
    filter.type = 'lowpass'
    filter.frequency.value = Math.min(14000, freq * 10)

    // Chain: osc -> filter -> gain -> destination
    osc.connect(filter)
    filter.connect(gain)
    gain.connect(audioCtx.destination)

    // Square wave is naturally louder; switch to 'triangle' if too sharp
    osc.type = 'square'
    osc.frequency.setValueAtTime(freq, audioCtx.currentTime)

    // ADSR envelope for punch + smooth tail
    const now = audioCtx.currentTime
    const attack = 0.03
    const hold   = Math.max(0, (duration / 1000) - (attack + 0.15))
    const release = 0.15

    const safePeak = Math.min(0.98, Math.max(0.05, peak)) // safety cap against clipping
    gain.gain.cancelScheduledValues(now)
    gain.gain.setValueAtTime(0.0001, now)
    gain.gain.linearRampToValueAtTime(safePeak, now + attack)                 // attack
    gain.gain.setValueAtTime(safePeak, now + attack + hold)                   // hold
    gain.gain.linearRampToValueAtTime(0.0001, now + attack + hold + release)  // release

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
