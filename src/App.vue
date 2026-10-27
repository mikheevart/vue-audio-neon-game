<template>
  <div id="game-canvas" @mousedown="spawnExplosion">
    <!-- Панель управления -->
    <div class="ui-panel" @mousedown.stop>
      <h1>Аудио-неоновый салют на Vue 3</h1>
      <p>Кликните для запуска взрывной волны и аккорда</p>
      <p>Фигур: <strong>{{ shapes.length }}</strong> | Искр: <strong>{{ trails.length }}</strong></p>
    </div>

    <!-- Рендеринг следов (искр) -->
    <div
      v-for="trail in trails"
      :key="trail.id"
      class="trail-particle"
      :style="{
        left: trail.x + 'px',
        top: trail.y + 'px',
        width: trail.size + 'px',
        height: trail.size + 'px',
        backgroundColor: trail.color,
        opacity: trail.alpha,
        borderRadius: trail.type === 'circle' ? '50%' : '4px',
        boxShadow: `0 0 10px ${trail.color}`
      }"
    ></div>

    <!-- Рендеринг основных фигур -->
    <div
      v-for="shape in shapes"
      :key="shape.id"
      class="shape"
      :style="{
        left: shape.x + 'px',
        top: shape.y + 'px',
        width: shape.size + 'px',
        height: shape.size + 'px',
        backgroundColor: shape.color,
        borderRadius: shape.type === 'circle' ? '50%' : '4px',
        boxShadow: `0 0 20px ${shape.color}, 0 0 40px ${shape.color}`
      }"
    ></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const shapes = ref([])
const trails = ref([])
const gravity = 0.5
const colors = ['#38bdf8', '#a855f7', '#ec4899', '#f43f5e', '#10b981', '#fbbf24']

// --- НАСТРОЙКА WEB AUDIO API ---
let audioCtx = null

// Пентатоническая шкала (в Гц) для благозвучных отскоков (Ля-минор пентатоника)
const pentatonicScale = [110, 130.81, 146.83, 164.81, 196.00, 220, 261.63, 293.66, 329.63, 392.00, 440, 523.25, 587.33, 659.25, 783.99, 880]

// Готовые красивые аккорды для взрывов (массивы частот в Гц)
const chords = [
  [130.81, 164.81, 196.00, 261.63], // До-мажор (C)
  [146.83, 174.61, 220.00, 293.66], // Ре-минор (Dm)
  [164.81, 196.00, 246.94, 329.63], // Ми-минор (Em)
  [174.61, 220.00, 261.63, 349.23], // Фа-мажор (F)
  [220.00, 261.63, 293.66, 440.00]  // Ля-минор (Am)
]

// Инициализация аудиоконтекста (браузеры требуют, чтобы это происходило по клику пользователя)
const initAudio = () => {
  if (!audioCtx) {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)()
  }
}

// Функция генерации одиночной ноты (для отскоков)
const playNote = (frequency, duration = 0.3, type = 'sine') => {
  if (!audioCtx) return

  const osc = audioCtx.createOscillator()
  const gainNode = audioCtx.createGain()

  osc.type = type
  osc.frequency.setValueAtTime(frequency, audioCtx.currentTime)

  // Плавное затухание звука (чтобы не было щелчков)
  gainNode.gain.setValueAtTime(0.15, audioCtx.currentTime) // Громкость ноты
  gainNode.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime + duration)

  osc.connect(gainNode)
  gainNode.connect(audioCtx.destination)

  osc.start()
  osc.stop(audioCtx.currentTime + duration)
}

// Функция генерации целого аккорда (для взрывов)
const playChord = () => {
  if (!audioCtx) return
  const randomChord = chords[Math.floor(Math.random() * chords.length)]

  // Проигрываем каждую ноту из аккорда одновременно с мягким типом волны 'triangle'
  randomChord.forEach(freq => {
    playNote(freq, 0.8, 'triangle')
  })
}
// -------------------------------

const spawnExplosion = (event) => {
  initAudio() // Активируем звук при первом же клике
  playChord() // Запускаем взрывной аккорд

  const types = ['circle', 'box']
  const count = Math.floor(Math.random() * 6) + 5

  for (let i = 0; i < count; i++) {
    const angle = Math.random() * Math.PI * 2
    const speed = Math.random() * 8 + 4

    shapes.value.push({
      id: Date.now() + Math.random(),
      x: event.clientX,
      y: event.clientY,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed - 2,
      size: Math.random() * 20 + 15,
      color: colors[Math.floor(Math.random() * colors.length)],
      type: types[Math.floor(Math.random() * types.length)],
      hasHitFloor: false // Флаг, чтобы избежать спама звука при трении о пол
    })
  }
}

const updatePhysics = () => {
  const floor = window.innerHeight
  const wallRight = window.innerWidth

  shapes.value.forEach(shape => {
    shape.vy += gravity
    shape.y += shape.vy
    shape.x += shape.vx

    if (Math.abs(shape.vy) > 0.1 || Math.abs(shape.vx) > 0.1) {
      trails.value.push({
        id: Math.random(),
        x: shape.x,
        y: shape.y,
        size: shape.size * 0.6,
        color: shape.color,
        type: shape.type,
        alpha: 0.8
      })
    }

    // Отскок от пола + Звук
    const halfSize = shape.size / 2
    if (shape.y + halfSize >= floor) {
      shape.y = floor - halfSize

      // Генерируем звук отскока только при значительной вертикальной скорости
      if (Math.abs(shape.vy) > 1.5) {
        // Подбираем индекс ноты в зависимости от размера: мелкие фигуры (size ~15) получат высокий индекс, крупные — низкий
        const sizePercentage = (shape.size - 15) / 20 // Нормализация от 0 до 1
        const scaleIndex = Math.floor((1 - sizePercentage) * (pentatonicScale.length - 1))
        const safeIndex = Math.max(0, Math.min(scaleIndex, pentatonicScale.length - 1))

        playNote(pentatonicScale[safeIndex], 0.2, 'sine')
      }

      shape.vy = -shape.vy * 0.55
      shape.vx *= 0.9
    }

    // Отскок от стен
    if (shape.x - halfSize <= 0) {
      shape.x = halfSize
      shape.vx = -shape.vx * 0.7
    } else if (shape.x + halfSize >= wallRight) {
      shape.x = wallRight - halfSize
      shape.vx = -shape.vx * 0.7
    }
  })

  // Таяние искр
  trails.value.forEach(trail => {
    trail.alpha -= 0.04
    trail.size *= 0.94
  })

  // Очистка
  trails.value = trails.value.filter(trail => trail.alpha > 0 && trail.size > 1)

  if (shapes.value.length > 60) {
    shapes.value.splice(0, shapes.value.length - 60) // Снизим лимит до 60 для аудио-стабильности
  }

  requestAnimationFrame(updatePhysics)
}

onMounted(() => {
  updatePhysics()
})
</script>

<style>
* {
  box-sizing: border-box;
}

body, html {
  margin: 0;
  padding: 0;
  overflow: hidden;
  background: #090d16;
  font-family: sans-serif;
  width: 100%;
  height: 100%;
}

#game-canvas {
  width: 100vw;
  height: 100vh;
  position: absolute;
  top: 0;
  left: 0;
}

.shape, .trail-particle {
  position: absolute;
  transform: translate(-50%, -50%);
  user-select: none;
  pointer-events: none;
}

.trail-particle {
  mix-blend-mode: screen;
}

.ui-panel {
  position: absolute;
  top: 20px;
  left: 20px;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(10px);
  padding: 15px 25px;
  border-radius: 12px;
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.1);
  z-index: 10;
}

h1 { margin: 0 0 5px 0; font-size: 1.2rem; text-shadow: 0 0 10px rgba(255,255,255,0.3); }
p { margin: 0; font-size: 0.9rem; opacity: 0.8; }
</style>
