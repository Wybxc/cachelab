<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps({
  hits: Number,
  misses: Number,
})

const missRate = computed(() => {
  const { hits = 1, misses = 1 } = props
  return ((misses / (hits + misses)) * 100).toFixed(2)
})
</script>

<template>
  <span> {{ missRate }}% </span>
</template>