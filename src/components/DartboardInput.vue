<script setup>
const emit = defineEmits(['hit'])

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

function handleClick(event) {
  const svg = event.currentTarget
  const rect = svg.getBoundingClientRect()
  const x = event.clientX - rect.left
  const y = event.clientY - rect.top
  const cx = rect.width / 2
  const cy = rect.height / 2
  const R = Math.min(rect.width, rect.height) / 2

  const type = getHitType(x, y, cx, cy, R)
  const number = type === 'IB' || type === 'OB' || type === 'MISS' ? null : getSectorNumber(x, y, cx, cy)

  emit('hit', { type, number })
}
</script>

<template>
  <div class="dartboard-wrapper">
    <svg viewBox="0 0 400 400" class="dartboard" @click="handleClick">
      <circle cx="200" cy="200" r="170" class="ring outer" />
      <circle cx="200" cy="200" r="152" class="ring double" />
      <circle cx="200" cy="200" r="124" class="ring outer" />
      <circle cx="200" cy="200" r="112" class="ring triple" />
      <circle cx="200" cy="200" r="88" class="ring inner" />
      <circle cx="200" cy="200" r="48" class="ring single" />
      <circle cx="200" cy="200" r="24" class="ring outer-bull" />
      <circle cx="200" cy="200" r="12" class="ring inner-bull" />
      <line
        v-for="idx in 20"
        :key="idx"
        :x1="200"
        :y1="200"
        :x2="200 + 180 * Math.sin(((idx - 1) * 18 * Math.PI) / 180)"
        :y2="200 - 180 * Math.cos(((idx - 1) * 18 * Math.PI) / 180)"
        class="divider"
      />
    </svg>
    <p class="hint">盤面をタップして入力（外周で MISS）</p>
  </div>
</template>

<style scoped>
.dartboard-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.dartboard {
  width: 100%;
  max-width: 360px;
  aspect-ratio: 1 / 1;
  background: radial-gradient(circle at 50% 50%, #0f172a 0%, #0f172a 52%, #1f2937 52%, #1f2937 57%, #0f172a 57%, #0f172a 62%, #1f2937 62%, #1f2937 80%, #0f172a 80%, #0f172a 85%, #111827 85%);
  border-radius: 50%;
  border: 4px solid #111827;
  box-shadow: inset 0 0 8px rgba(0, 0, 0, 0.4), 0 8px 16px rgba(0, 0, 0, 0.2);
  cursor: pointer;
}

.ring {
  fill: transparent;
  stroke-width: 0;
}

.outer { stroke: #111827; }
.double { stroke: #4338ca; }
.triple { stroke: #ef4444; }
.inner { stroke: #111827; }
.single { stroke: #0f172a; }
.outer-bull { fill: #dc2626; }
.inner-bull { fill: #22c55e; }

.divider {
  stroke: rgba(255, 255, 255, 0.2);
  stroke-width: 1;
}

.hint {
  font-size: 12px;
  color: #6b7280;
  text-align: center;
}
</style>
