<script setup lang="ts">
import colormap from 'colormap';
import { computed } from 'vue';

const props = defineProps<{
  lines: number[]
}>()

const colors = colormap({
  colormap: 'jet',
  nshades: 32,
  format: 'rgbaString',
  alpha: 0.3,
})

const styles = computed(() => {
  return props.lines.map((line) => {
    return {
      padding: 0,
      backgroundColor: colors[line],
    }
  })
})
</script>

<template>
  <table style="width: fit-content; border: none">
    <tr v-for="i in 8" :key="i" :style="styles[i - 1]" class="relative">
      <td v-for="j in 8" :key="j" style="padding: 0">
        <div class="w-3 h-3"></div>
      </td>
      <div class="absolute left-0 right-0 text-center text-xs" style="transform: translateY(-2px);">
        Line {{ props.lines[i - 1] }}
      </div>
    </tr>
  </table>
</template>