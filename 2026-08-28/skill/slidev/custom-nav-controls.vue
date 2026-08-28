<script setup lang="ts">
import { useNav } from '@slidev/client'

const { currentSlideNo, go, total } = useNav()

function jump(event: Event) {
  go(Number((event.target as HTMLSelectElement).value))
}
</script>

<template>
  <label class="xskill-jump" title="跳转到指定页">
    <span class="sr-only">跳转到指定页</span>
    <select :value="currentSlideNo" aria-label="跳转到指定页" @change="jump">
      <option v-for="page in total" :key="page" :value="page">
        {{ page }}
      </option>
    </select>
  </label>
</template>
