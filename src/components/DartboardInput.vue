<script setup>
import { computed } from 'vue'

const sectors = [20, 1, 18, 4, 13, 6, 10, 15, 2, 17, 3, 19, 7, 16, 8, 11, 14, 9, 12, 5]

const props = defineProps({
  disabled: {
    type: Boolean,
    default: false
  }
})

const emit = defineEmits(['hit'])

const rings = [
  { label: 'IB', ratio: 0.06 },
  { label: 'OB', ratio: 0.12 },
  { label: 'S', ratio: 0.52 },
  { label: 'T', ratio: 0.57 },
  { label: 'S', ratio: 0.62 },
  { label: 'D', ratio: 0.8 },
  { label: 'S', ratio: 0.9 }
]

const ticks = computed(() => {
  const angle = (Math.PI * 2) / 20
  return sectors.map((number, index) => {
    const startAngle = -Math.PI / 2 + angle * index
    const endAngle = startAngle + angle
    return {
      number,
      index,
      startAngle,
      endAngle,
      midAngle: startAngle + angle / 2
    }
  })
})

function decideRingType(ratio) {
  if (ratio <= 0.06) return 'IB'
  if (ratio <= 0.12) return 'OB'
  if (ratio <= 0.52) return 'S'
  if (ratio <= 0.57) return 'T'
  if (ratio <= 0.62) return 'S'
  if (ratio <= 0.8) return 'D'
  if (ratio <= 0.9) return 'S'
  return 'MISS'
}

function getSectorNumber(x, y, cx, cy) {
  const dx = x - cx
  const dy = cy - y

  let angle = Math.atan2(dy, dx)
  let cw = Math.PI / 2 - angle
  if (cw < 0) cw += Math.PI * 2

  const sectorAngle = (Math.PI * 2) / 20
  const sectorIndex = Math.floor(cw / sectorAngle)
  return sectors[sectorIndex]
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

  const dx = x - cx
  const dy = cy - y
  const r = Math.sqrt(dx * dx + dy * dy)
  const ratio = r / R

  const hitType = decideRingType(ratio)
  const number = ['OB', 'IB', 'MISS'].includes(hitType)
    ? null
    : getSectorNumber(x, y, cx, cy)

  emit('hit', { type: hitType, number })
}
</script>

<template>
  <div class="dartboard-wrapper">
    <svg
      class="dartboard"
      viewBox="0 0 400 400"
      role="button"
      tabindex="0"
      :aria-disabled="props.disabled"
      @click="handleClick"
    >
      <defs>
        <radialGradient id="board-bg" cx="50%" cy="50%" r="50%">
          <stop offset="0%" stop-color="#111827" stop-opacity="0.95" />
          <stop offset="100%" stop-color="#1f2937" stop-opacity="0.95" />
        </radialGradient>
      </defs>

      <circle cx="200" cy="200" r="180" fill="url(#board-bg)" />

      <g v-for="ring in rings" :key="ring.label">
        <circle
          cx="200"
          cy="200"
          :r="200 * ring.ratio"
          :class="['ring', ring.label.toLowerCase()]"
          fill="none"
        />
      </g>

      <g class="sectors">
        <g v-for="tick in ticks" :key="tick.index">
          <path
            :d="`M200 200 L${200 + 200 * Math.cos(tick.startAngle)} ${200 + 200 * Math.sin(tick.startAngle)} `
              + `A200 200 0 0 1 ${200 + 200 * Math.cos(tick.endAngle)} ${200 + 200 * Math.sin(tick.endAngle)} Z`"
            :class="['sector', tick.index % 2 === 0 ? 'even' : 'odd']"
          />
          <text
            :x="200 + 190 * Math.cos(tick.midAngle)"
            :y="200 + 190 * Math.sin(tick.midAngle)"
            class="number"
            dominant-baseline="middle"
            text-anchor="middle"
          >
            {{ tick.number }}
          </text>
        </g>
      </g>

      <circle cx="200" cy="200" r="48" class="bull ob" />
      <circle cx="200" cy="200" r="24" class="bull ib" />
    </svg>
  </div>
</template>

<style scoped>
.dartboard-wrapper {
  width: 100%;
  display: flex;
  justify-content: center;
}

.dartboard {
  width: 100%;
  max-width: 360px;
  aspect-ratio: 1 / 1;
  cursor: pointer;
  touch-action: manipulation;
  border-radius: 12px;
  box-shadow: 0 12px 30px rgba(0, 0, 0, 0.2);
}

.ring {
  stroke: rgba(255, 255, 255, 0.08);
  stroke-width: 2;
}

.ring.ib {
  stroke: #c71f37;
  stroke-width: 8;
}

.ring.ob {
  stroke: #1f7a8c;
  stroke-width: 8;
}

.ring.t {
  stroke: rgba(255, 255, 255, 0.12);
}

.ring.d {
  stroke: rgba(255, 255, 255, 0.12);
}

.sector {
  fill: none;
  stroke: rgba(255, 255, 255, 0.04);
  stroke-width: 1;
}

.sector.even {
  fill: rgba(255, 255, 255, 0.04);
}

.sector.odd {
  fill: rgba(255, 255, 255, 0.02);
}

.number {
  fill: #f9fafb;
  font-size: 14px;
  font-weight: 700;
}

.bull.ob {
  fill: #1f7a8c;
}

.bull.ib {
  fill: #c71f37;
}
</style>
