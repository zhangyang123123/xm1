<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'

const FOCUS_TIME = 25 * 60
const BREAK_TIME = 5 * 60

const mode = ref('focus')
const timeLeft = ref(FOCUS_TIME)
const isRunning = ref(false)
const completedToday = ref(0)

let timer = null
let audioContext = null

const totalTime = computed(() => mode.value === 'focus' ? FOCUS_TIME : BREAK_TIME)

const progress = computed(() => {
  return ((totalTime.value - timeLeft.value) / totalTime.value) * 100
})

const displayTime = computed(() => {
  const minutes = Math.floor(timeLeft.value / 60)
  const seconds = timeLeft.value % 60
  return `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`
})

const modeLabel = computed(() => mode.value === 'focus' ? '专注时间' : '休息时间')

const circumference = 2 * Math.PI * 140
const strokeDashoffset = computed(() => {
  return circumference - (progress.value / 100) * circumference
})

const todayKey = computed(() => {
  const d = new Date()
  return `pomodoro_${d.getFullYear()}_${d.getMonth() + 1}_${d.getDate()}`
})

function loadCompletedCount() {
  const saved = localStorage.getItem(todayKey.value)
  completedToday.value = saved ? parseInt(saved, 10) : 0
}

function saveCompletedCount() {
  localStorage.setItem(todayKey.value, String(completedToday.value))
}

function tick() {
  if (timeLeft.value > 0) {
    timeLeft.value--
  } else {
    handleTimerComplete()
  }
}

function startTimer() {
  if (isRunning.value) return
  if (Notification.permission === 'default') {
    Notification.requestPermission()
  }
  isRunning.value = true
  timer = setInterval(tick, 1000)
}

function pauseTimer() {
  isRunning.value = false
  if (timer) {
    clearInterval(timer)
    timer = null
  }
}

function resetTimer() {
  pauseTimer()
  timeLeft.value = totalTime.value
}

function switchMode() {
  pauseTimer()
  mode.value = mode.value === 'focus' ? 'break' : 'focus'
  timeLeft.value = totalTime.value
}

function playSound() {
  if (!audioContext) {
    audioContext = new (window.AudioContext || window.webkitAudioContext)()
  }
  const oscillator = audioContext.createOscillator()
  const gainNode = audioContext.createGain()
  oscillator.connect(gainNode)
  gainNode.connect(audioContext.destination)
  oscillator.type = 'sine'
  oscillator.frequency.setValueAtTime(880, audioContext.currentTime)
  oscillator.frequency.setValueAtTime(660, audioContext.currentTime + 0.2)
  oscillator.frequency.setValueAtTime(880, audioContext.currentTime + 0.4)
  gainNode.gain.setValueAtTime(0.3, audioContext.currentTime)
  gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.8)
  oscillator.start(audioContext.currentTime)
  oscillator.stop(audioContext.currentTime + 0.8)
}

function sendNotification() {
  if (Notification.permission === 'granted') {
    const title = mode.value === 'focus' ? '专注完成！' : '休息结束！'
    const body = mode.value === 'focus' ? '休息一下吧 ☕' : '开始下一个番茄钟吧 🚀'
    new Notification(title, { body })
  }
}

function handleTimerComplete() {
  pauseTimer()
  playSound()
  sendNotification()
  if (mode.value === 'focus') {
    completedToday.value++
    saveCompletedCount()
    mode.value = 'break'
  } else {
    mode.value = 'focus'
  }
  timeLeft.value = totalTime.value
}

watch(todayKey, () => {
  loadCompletedCount()
})

onMounted(() => {
  loadCompletedCount()
  if ('Notification' in window && Notification.permission === 'default') {
    Notification.requestPermission()
  }
})

onUnmounted(() => {
  if (timer) clearInterval(timer)
})
</script>

<template>
  <div class="app" :class="mode">
    <div class="container">
      <h1 class="title">番茄钟</h1>

      <div class="mode-tabs">
        <button
          class="mode-btn"
          :class="{ active: mode === 'focus' }"
          @click="mode === 'break' && switchMode()"
        >
          专注
        </button>
        <button
          class="mode-btn"
          :class="{ active: mode === 'break' }"
          @click="mode === 'focus' && switchMode()"
        >
          休息
        </button>
      </div>

      <div class="timer-wrapper">
        <svg class="progress-ring" viewBox="0 0 320 320">
          <circle
            class="progress-ring-bg"
            cx="160"
            cy="160"
            r="140"
          />
          <circle
            class="progress-ring-bar"
            cx="160"
            cy="160"
            r="140"
            :stroke-dasharray="circumference"
            :stroke-dashoffset="strokeDashoffset"
          />
        </svg>
        <div class="timer-center">
          <div class="time-display">{{ displayTime }}</div>
          <div class="mode-label">{{ modeLabel }}</div>
        </div>
      </div>

      <div class="controls">
        <button v-if="!isRunning" class="btn btn-primary" @click="startTimer">
          开始
        </button>
        <button v-else class="btn btn-warning" @click="pauseTimer">
          暂停
        </button>
        <button class="btn btn-secondary" @click="resetTimer">
          重置
        </button>
      </div>

      <div class="stats">
        <div class="stat-item">
          <span class="stat-count">{{ completedToday }}</span>
          <span class="stat-label">今日完成</span>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background-color 0.5s ease;
}

.app.focus {
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
}

.app.break {
  background: linear-gradient(135deg, #1a2e1a 0%, #1e3a1e 50%, #2d4a2d 100%);
}

.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 40px 20px;
  width: 100%;
  max-width: 480px;
}

.title {
  font-size: 2rem;
  font-weight: 300;
  color: #e8e8e8;
  margin: 0 0 32px 0;
  letter-spacing: 4px;
}

.mode-tabs {
  display: flex;
  gap: 8px;
  margin-bottom: 40px;
  background: rgba(255, 255, 255, 0.06);
  padding: 6px;
  border-radius: 12px;
}

.mode-btn {
  padding: 10px 28px;
  border: none;
  background: transparent;
  color: #888;
  font-size: 0.95rem;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-family: inherit;
}

.mode-btn.active {
  background: rgba(255, 255, 255, 0.12);
  color: #fff;
}

.mode-btn:hover:not(.active) {
  color: #ccc;
}

.timer-wrapper {
  position: relative;
  width: 320px;
  height: 320px;
  margin-bottom: 48px;
}

.progress-ring {
  width: 100%;
  height: 100%;
  transform: rotate(-90deg);
}

.progress-ring-bg {
  fill: none;
  stroke: rgba(255, 255, 255, 0.06);
  stroke-width: 10;
}

.progress-ring-bar {
  fill: none;
  stroke: url(#gradient);
  stroke-width: 10;
  stroke-linecap: round;
  transition: stroke-dashoffset 1s linear;
}

.app.focus .progress-ring-bar {
  stroke: #e94560;
}

.app.break .progress-ring-bar {
  stroke: #52c58f;
}

.timer-center {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
}

.time-display {
  font-size: 5rem;
  font-weight: 200;
  color: #fff;
  font-variant-numeric: tabular-nums;
  letter-spacing: 2px;
  line-height: 1;
  margin-bottom: 12px;
}

.mode-label {
  font-size: 0.95rem;
  color: #888;
  letter-spacing: 6px;
  text-transform: uppercase;
}

.controls {
  display: flex;
  gap: 16px;
  margin-bottom: 48px;
}

.btn {
  padding: 14px 36px;
  border: none;
  border-radius: 12px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.25s ease;
  font-family: inherit;
  min-width: 120px;
}

.btn-primary {
  background: #e94560;
  color: #fff;
}

.btn-primary:hover {
  background: #d13a54;
  transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(233, 69, 96, 0.3);
}

.app.break .btn-primary {
  background: #52c58f;
}

.app.break .btn-primary:hover {
  background: #46b07e;
  box-shadow: 0 8px 24px rgba(82, 197, 143, 0.3);
}

.btn-warning {
  background: #f5a623;
  color: #1a1a2e;
}

.btn-warning:hover {
  background: #e69512;
  transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(245, 166, 35, 0.3);
}

.btn-secondary {
  background: rgba(255, 255, 255, 0.1);
  color: #ddd;
}

.btn-secondary:hover {
  background: rgba(255, 255, 255, 0.18);
  transform: translateY(-2px);
}

.stats {
  display: flex;
  justify-content: center;
}

.stat-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px 48px;
  background: rgba(255, 255, 255, 0.04);
  border-radius: 16px;
}

.stat-count {
  font-size: 2.5rem;
  font-weight: 300;
  color: #fff;
  line-height: 1;
  margin-bottom: 8px;
}

.stat-label {
  font-size: 0.85rem;
  color: #888;
  letter-spacing: 2px;
}

@media (max-width: 480px) {
  .timer-wrapper {
    width: 280px;
    height: 280px;
  }
  .time-display {
    font-size: 4rem;
  }
  .btn {
    min-width: 100px;
    padding: 12px 24px;
  }
  .title {
    font-size: 1.5rem;
  }
}
</style>
