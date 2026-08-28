<script setup lang="ts">
import { useNav } from '@slidev/client'

const { currentSlideNo, go, slides } = useNav()

function jump(event: Event) {
  go(Number((event.target as HTMLSelectElement).value))
}
</script>

<template>
  <label class="xskill-jump" title="跳转到指定页">
    <span class="sr-only">跳转到指定页</span>
    <select :value="currentSlideNo" aria-label="跳转到指定页" @change="jump">
      <option v-for="slide in slides" :key="slide.no" :value="slide.no">
        {{ slide.no }}{{ slide.meta.slide.title ? ` · ${slide.meta.slide.title}` : '' }}
      </option>
    </select>
  </label>
</template>
