<script setup lang="ts">
import type { store } from '@/wailsjs/go/models'

defineProps<{ type: store.DriverType | 'default' }>()

const colors = {
  network: 'bg-blue-100',
  display: 'bg-green-100',
  miscellaneous: 'bg-gray-100',
  default: 'bg-orange-200'
}
</script>

<template>
  <span class="content-center me-1 px-1.5 h-5 text-xs text-nowrap rounded-sm" :class="colors[type]">
    {{ $t(`driverForm.${type}`) }}
  </span>
</template>
