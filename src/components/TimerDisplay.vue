<template>
  <div class="timer-container">
    <div class="session-title">{{ session?.name || 'No Session' }}</div>

    <!-- fixed-size wrap so the panel never shifts -->
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
import { ref, computed, watch, onBeforeUnmount } from 'vue'

const props = defineProps({
  session: Object,
  onComplete: Function,
  pause: Boolean,
  stopped: Boolean
})

/* ---------- state ---------- */
const remaining = ref(0)
const progress  = ref(0)
const running   = ref(false)

let intervalId = null
let currentRunId = 0              // increments per session start
let alarmTimeouts = []            // all scheduled alarm setTimeout ids
let audioCtx = null

/* ---------- derived time ---------- */
const time = computed(() => {
  const ms = remaining.value % 1000
  const totalSec = Math.floor(remaining.value / 1000)
  const sec = totalSec % 60
  const min = Math.floor(totalSec / 60) % 60
  const hrs = Math.floor(totalSec / 3600)
  return {
    hours: String(hrs).padStart(2, '0'),
    minutes: String(min).padStart(2, '0'),
    seconds: String(sec).padStart(2, '0'),
    milliseconds: String(Math.floor(ms / 10)).padStart(2, '0')
  }
})

/* ---------- watchers ---------- */
watch(() => props.session, (s, prev) => {
  // cancel any previous run when session changes
  invalidateRun()
  if (s && s.duration > 0) {
    startSession(s)
  } else {
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
    // resume current run
    runTick(currentRunId, Date.now(), props.session.duration * 60 * 1000 - remaining.value)
  }
})

watch(() => props.stopped, (shouldStop) => {
  if (!shouldStop) return
  // external stop/clear: invalidate run and reset visuals
  invalidateRun()
  running.value = false
  remaining.value = 0
  progress.value = 0
})

onBeforeUnmount(() => {
  invalidateRun()
})

/* ---------- helpers ---------- */
function startSession(s) {
  const durationMs = s.duration * 60 * 1000
  running.value = true
  progress.value = 0
  remaining.value = durationMs

  // new run id
  currentRunId += 1
  const runId = currentRunId

  // start ticking
  runTick(runId, Date.now(), 0)
}

function runTick(runId, startAt, elapsedOffset) {
  clearTick() // ensure single interval

  const baseStart = startAt - elapsedOffset

  intervalId = setInterval(() => {
    // if run was invalidated (skip/stop/new session), bail out
    if (runId !== currentRunId) {
      clearTick()
      return
    }

    const total = props.session.duration * 60 * 1000
    const elapsed = Date.now() - baseStart
    const left = Math.max(0, total - elapsed)

    remaining.value = left
    progress.value = Math.min(100, (elapsed / total) * 100)

    if (left <= 0) {
      // natural completion for this exact run
      clearTick()
      if (runId === currentRunId) {
        playAlarmForRun(runId)     // alarm only on natural completion
        props.onComplete && props.onComplete()
      }
    }
  }, 10)
}

function clearTick() {
  if (intervalId) {
    clearInterval(intervalId)
    intervalId = null
  }
}

function invalidateRun() {
  // bump run id so any pending tick or alarm callbacks are ignored
  currentRunId += 1
  clearTick()
  cancelAllAlarms()
}

/* ---------- alarm (guarded by runId) ---------- */
function playAlarmForRun(runId) {
  // safety: if run already invalidated, do nothing
  if (runId !== currentRunId) return

  cancelAllAlarms()

  const tones = [800, 1200, 1600]
  const durations = [500, 700, 900]

  // ensure audio context is created after a user gesture (browser policy)
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)()

  tones.forEach((freq, i) => {
    const tId = setTimeout(() => {
      // check again: only play if this run is still current
      if (runId !== currentRunId) return
      playBeep(freq, durations[i])
    }, i * 1000)
    alarmTimeouts.push(tId)
  })
}

function cancelAllAlarms() {
  alarmTimeouts.forEach(id => clearTimeout(id))
  alarmTimeouts = []
}

function playBeep(freq, duration) {
  try {
    const osc = audioCtx.createOscillator()
    const gain = audioCtx.createGain()
    osc.connect(gain)
    gain.connect(audioCtx.destination)
    osc.type = 'sine'
    osc.frequency.value = freq
    // subtle fade to reduce clicks
    const now = audioCtx.currentTime
    gain.gain.setValueAtTime(0.0001, now)
    gain.gain.exponentialRampToValueAtTime(0.4, now + 0.02)
    gain.gain.exponentialRampToValueAtTime(0.0001, now + duration / 1000)
    osc.start(now)
    osc.stop(now + duration / 1000 + 0.02)
  } catch {}
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
  font-size: 1.2rem;
  margin-bottom: 1rem;
  color: var(--text);
  font-family: 'Roboto Mono', ui-monospace, Menlo, Monaco, Consolas,
               'Liberation Mono', 'Courier New', monospace;
  font-weight: 700;
  letter-spacing: 1px;
  text-shadow:
    0 1px 3px rgba(0,0,0,0.12), 0 0 8px var(--accent-glow);
}

/* keep panel stable */
.countdown-wrap {
  display: grid;
  place-items: center;
  min-height: 78px;
}

.countdown {
  font-family: 'Roboto Mono', ui-monospace, Menlo, Monaco, Consolas,
               'Liberation Mono', 'Courier New', monospace;
  font-variant-numeric: tabular-nums;
  font-feature-settings: "tnum" 1, "zero" 1;
  font-size: 3rem;
  line-height: 1;
  letter-spacing: 2px;
  color: var(--text);
  /* subtle glow */
  text-shadow:
    0 2px 8px rgba(0,0,0,0.15), 0 0 12px var(--accent-glow);
  display: inline-block;
  min-width: 11ch; /* 00:00:00:00 */
}

.progress-bar {
  margin-top: 0.9rem;
  /* The track is now see-through to allow the fill to blend with the container bg */
  background: transparent;
  border: 2px solid var(--progress-bg); /* Thicker border for visibility */
  box-shadow: inset 0 1px 2px rgba(0,0,0,0.15); /* Inset shadow for depth */
  height: 14px;
  border-radius: 8px;
  overflow: hidden;
}
.fill {
  height: 100%;
  /* A vibrant, shifting gradient creates a cool "negative" effect with difference blend mode */
  background: linear-gradient(90deg, #ff00ff, #00ffff, #ffff00, #ff00ff);
  background-size: 200% 100%;
  mix-blend-mode: difference;
  width: 0%;
  transition: width 0.12s linear;
  animation: gradient-shift 4s linear infinite;
}

@keyframes gradient-shift {
  0% {
    background-position: 0% 50%;
  }
  100% {
    background-position: 200% 50%;
  }
}
</style>
