<template>
  <div class="dartboard-wrapper">
    <svg
      class="dartboard"
      viewBox="0 0 400 400"
      role="presentation"
      aria-label="dartboard input"
      @click="handleClick"
    >
      <defs>
        <radialGradient id="board-bg" cx="50%" cy="50%" r="70%">
          <stop offset="0%" stop-color="#111827" stop-opacity="0.9" />
          <stop offset="100%" stop-color="#111827" stop-opacity="0.6" />
        </radialGradient>
      </defs>
      <circle cx="200" cy="200" r="190" fill="url(#board-bg)" stroke="#1f2937" stroke-width="2" />
      <circle cx="200" cy="200" r="152" fill="none" stroke="#d1d5db" stroke-width="2" stroke-dasharray="2 6" />
      <circle cx="200" cy="200" r="182" fill="none" stroke="#d1d5db" stroke-width="3" />
      <circle cx="200" cy="200" r="112" fill="none" stroke="#9ca3af" stroke-width="2" />
      <circle cx="200" cy="200" r="24" fill="#ef4444" stroke="#fca5a5" stroke-width="2" />
      <circle cx="200" cy="200" r="48" fill="#fcd34d" stroke="#f59e0b" stroke-width="2" />
      <g v-for="index in 20" :key="index">
        <line
          :x1="200"
          :y1="200"
          :x2="200 + 190 * Math.cos(((index - 1) * 18 - 90) * Math.PI / 180)"
          :y2="200 + 190 * Math.sin(((index - 1) * 18 - 90) * Math.PI / 180)"
          stroke="#374151"
          stroke-width="2"
        />
      </g>
      <g class="numbers">
        <text
          v-for="(value, index) in sectors"
          :key="value"
          :x="200 + 155 * Math.cos((index * 18 - 90) * Math.PI / 180)"
          :y="206 + 155 * Math.sin((index * 18 - 90) * Math.PI / 180)"
        >
          {{ value }}
        </text>
      </g>
    </svg>
    <p class="hint">ボードをタップして入力（中央ほど BULL / 外側は MISS）</p>
    <p v-if="disabled" class="disabled-overlay">入力できません</p>
  </div>
</template>

<script setup>
const emit = defineEmits(['hit'])
const props = defineProps({
  disabled: { type: Boolean, default: false }
})

const sectors = [
  20, 1, 18, 4, 13, 6, 10, 15, 2, 17,
  3, 19, 7, 16, 8, 11, 14, 9, 12, 5
]

function getSectorNumber(x, y, cx, cy) {
  const dx = x - cx
  const dy = y - cy
  let angle = Math.atan2(dy, dx)
  angle += Math.PI / 2
  if (angle < 0) angle += Math.PI * 2
  const sectorAngle = (Math.PI * 2) / 20
  const index = Math.floor(angle / sectorAngle)
  return sectors[index]
}

function getHitType(x, y, cx, cy, R) {
  const dx = x - cx
  const dy = y - cy
  const r = Math.sqrt(dx * dx + dy * dy)
  const ratio = r / R

  if (ratio <= 0.06) return 'IB'
  if (ratio <= 0.12) return 'OB'
  if (ratio <= 0.52) return 'S'
  if (ratio <= 0.57) return 'T'
  if (ratio <= 0.62) return 'S'
  if (ratio <= 0.8) return 'D'
  if (ratio <= 0.85) return 'S'
  return 'MISS'
}

function handleClick(evt) {
  if (props.disabled) return
  const svg = evt.currentTarget
  if (!svg) return
  const rect = svg.getBoundingClientRect()
  const x = evt.clientX - rect.left
  const y = evt.clientY - rect.top
  const cx = rect.width / 2
  const cy = rect.height / 2
  const R = Math.min(rect.width, rect.height) / 2

  const type = getHitType(x, y, cx, cy, R)
  const number = type === 'IB' || type === 'OB' || type === 'MISS' ? null : getSectorNumber(x, y, cx, cy)

  emit('hit', { type, number })
}
</script>

<style scoped>
.dartboard-wrapper {
  position: relative;
  width: 100%;
  max-width: 360px;
  margin: 0 auto;
  text-align: center;
}

.dartboard {
  width: 100%;
  height: auto;
  cursor: pointer;
  border-radius: 50%;
  background: radial-gradient(circle, #111827 0%, #0f172a 100%);
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.25);
}

.numbers text {
  fill: #e5e7eb;
  font-size: 12px;
  font-weight: 700;
  text-anchor: middle;
}

.hint {
  margin-top: 8px;
  font-size: 12px;
  color: #6b7280;
}

.disabled-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.75);
  color: #9ca3af;
  font-weight: 700;
}
</style>
