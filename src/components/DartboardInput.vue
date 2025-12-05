<script setup>
const sectors = [
  20, 1, 18, 4, 13, 6, 10, 15, 2, 17,
  3, 19, 7, 16, 8, 11, 14, 9, 12, 5
]

const props = defineProps({
  disabled: {
    type: Boolean,
    default: false
  }
})

const emit = defineEmits(['hit'])

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
  const rect = svg.getBoundingClientRect()
  const x = evt.clientX - rect.left
  const y = evt.clientY - rect.top
  const cx = rect.width / 2
  const cy = rect.height / 2
  const R = Math.min(rect.width, rect.height) / 2

  const type = getHitType(x, y, cx, cy, R)
  let number = null

  if (type !== 'IB' && type !== 'OB' && type !== 'MISS') {
    number = getSectorNumber(x, y, cx, cy)
  }

  emit('hit', { type, number })
}
</script>

<template>
  <div class="dartboard" :class="{ disabled }">
    <svg viewBox="0 0 400 400" @click="handleClick">
      <circle class="ring outer" cx="200" cy="200" r="170" />
      <circle class="ring double" cx="200" cy="200" r="160" />
      <circle class="ring single" cx="200" cy="200" r="140" />
      <circle class="ring triple" cx="200" cy="200" r="120" />
      <circle class="ring inner-single" cx="200" cy="200" r="105" />
      <circle class="ring bull-outer" cx="200" cy="200" r="24" />
      <circle class="ring bull-inner" cx="200" cy="200" r="12" />

      <g class="separators">
        <line
          v-for="index in 20"
          :key="index"
          x1="200"
          y1="200"
          :x2="200 + 180 * Math.sin(((index - 1) * 2 * Math.PI) / 20)"
          :y2="200 - 180 * Math.cos(((index - 1) * 2 * Math.PI) / 20)"
        />
      </g>

      <g class="labels">
        <text
          v-for="(number, idx) in sectors"
          :key="number"
          :x="200 + 185 * Math.sin((idx * 2 * Math.PI) / 20)"
          :y="206 - 185 * Math.cos((idx * 2 * Math.PI) / 20)"
          text-anchor="middle"
          dominant-baseline="middle"
        >
          {{ number }}
        </text>
      </g>
    </svg>
  </div>
</template>

<style scoped>
.dartboard {
  width: 100%;
  max-width: 420px;
  margin: 0 auto;
  user-select: none;
}

.dartboard.disabled {
  opacity: 0.6;
  pointer-events: none;
}

svg {
  width: 100%;
  height: auto;
  display: block;
  background: radial-gradient(circle at 50% 50%, #111827, #0f172a);
  border-radius: 50%;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.35);
}

.ring {
  fill: none;
  stroke-width: 10;
}

.ring.outer {
  stroke: #0b1222;
  stroke-width: 20;
}

.ring.double {
  stroke: #e11d48;
}

.ring.single {
  stroke: #f8fafc;
}

.ring.triple {
  stroke: #10b981;
}

.ring.inner-single {
  stroke: #cbd5e1;
  stroke-width: 16;
}

.ring.bull-outer {
  fill: #f59e0b;
  stroke: #f59e0b;
}

.ring.bull-inner {
  fill: #ef4444;
  stroke: #ef4444;
}

.separators line {
  stroke: rgba(255, 255, 255, 0.5);
  stroke-width: 2;
}

.labels text {
  fill: #e5e7eb;
  font-size: 14px;
  font-weight: 700;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.6);
}
</style>
