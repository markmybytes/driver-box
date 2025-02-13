<script setup lang="ts">
import FileExeIcon from './icons/FileExeIcon.vue'
import GearIcon from './icons/GearIcon.vue'
import HomeIcon from './icons/HomeIcon.vue'
</script>

<template>
  <aside
    class="fixed top-0 left-0 w-14 h-screen transition-transform -translate-x-full sm:translate-x-0"
  >
    <div class="h-full px-3 py-4 overflow-y-auto bg-gray-50">
      <ul class="space-y-2 font-medium">
        <li>
          <RouterLink
            to="/"
            class="flex items-center p-2 text-gray-900 rounded-lg hover:bg-gray-200 group"
          >
            <HomeIcon></HomeIcon>
          </RouterLink>
        </li>
        <li>
          <RouterLink
            to="/"
            class="flex items-center p-2 text-gray-900 rounded-lg hover:bg-gray-200 group"
          >
            <FileExeIcon></FileExeIcon>
          </RouterLink>
        </li>
        <li>
          <RouterLink
            to="/"
            class="flex items-center p-2 text-gray-900 rounded-lg hover:bg-gray-200 group"
          >
            <GearIcon></GearIcon>
          </RouterLink>
        </li>
      </ul>
    </div>
  </aside>

  <div class="p-4 sm:ml-64">
    <RouterView></RouterView>
  </div>
</template>
