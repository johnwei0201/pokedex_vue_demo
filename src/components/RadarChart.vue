<script setup>
import { computed } from 'vue'

const props = defineProps({
  stats: {
    type: Array,
    required: true,
  },
})

const SIZE = 260
const CX = 130
const CY = 132
const RADIUS = 78
const LEVELS = 4
const MAX_STAT = 180

const axes = computed(() => {
  const count = props.stats.length || 6
  return props.stats.map((stat, index) => {
    const angle = -Math.PI / 2 + (index * 2 * Math.PI) / count
    const ratio = Math.min(stat.value / MAX_STAT, 1)
    return {
      label: stat.label,
      value: stat.value,
      x: CX + Math.cos(angle) * RADIUS * ratio,
      y: CY + Math.sin(angle) * RADIUS * ratio,
      axisX: CX + Math.cos(angle) * RADIUS,
      axisY: CY + Math.sin(angle) * RADIUS,
      labelX: CX + Math.cos(angle) * (RADIUS + 24),
      labelY: CY + Math.sin(angle) * (RADIUS + 24),
      anchor:
        Math.cos(angle) > 0.2 ? 'start' : Math.cos(angle) < -0.2 ? 'end' : 'middle',
    }
  })
})

function gridPoints(level) {
  const count = props.stats.length || 6
  const radius = RADIUS * (level / LEVELS)
  return Array.from({ length: count }, (_, index) => {
    const angle = -Math.PI / 2 + (index * 2 * Math.PI) / count
    return `${CX + Math.cos(angle) * radius},${CY + Math.sin(angle) * radius}`
  }).join(' ')
}

const dataPoints = computed(() =>
  axes.value.map((item) => `${item.x},${item.y}`).join(' '),
)
</script>

<template>
  <svg
    class="radar"
    :viewBox="`0 0 ${SIZE} ${SIZE}`"
    role="img"
    aria-label="能力雷達圖"
  >
    <polygon
      v-for="level in LEVELS"
      :key="level"
      class="radar-grid"
      :points="gridPoints(level)"
    />
    <line
      v-for="(axis, index) in axes"
      :key="`axis-${index}`"
      class="radar-axis"
      :x1="CX"
      :y1="CY"
      :x2="axis.axisX"
      :y2="axis.axisY"
    />
    <polygon class="radar-area" :points="dataPoints" />
    <circle
      v-for="(axis, index) in axes"
      :key="`dot-${index}`"
      class="radar-dot"
      :cx="axis.x"
      :cy="axis.y"
      r="3.5"
    />
    <text
      v-for="(axis, index) in axes"
      :key="`label-${index}`"
      class="radar-label"
      :x="axis.labelX"
      :y="axis.labelY"
      :text-anchor="axis.anchor"
      dominant-baseline="middle"
    >
      {{ axis.label }}
    </text>
  </svg>
</template>
